# Scenario 10: Spark Data Skew — Straggler Task

## 問題描述
Spark job 在 `groupBy` 聚合階段出現嚴重的 data skew，99% 的資料集中在單一 key（`hot_key`），導致少數 task 處理時間遠超其他 task（straggler），整體 job 效能極差。AQE（Adaptive Query Execution）被刻意關閉，無法自動緩解 skew。

## 架構設計

| 組件 | 設定 | 目的 |
|------|------|------|
| Primary Node | m7g.2xlarge (8 vCPU, 32 GB) | ResourceManager |
| Core Nodes | m7g.2xlarge × 3 | 跑 executor |
| Total Records | 500,000,000 (5 億) | 足夠大的資料量觸發明顯 skew |
| Hot Key 比例 | `hot_key` 佔 99%（4.95 億筆） | 極端 skew |
| Normal Keys | `key_001` ~ `key_009` 佔 1%（500 萬筆） | 均勻分佈 |
| Payload | 每筆 ~1KB | 增加 shuffle 資料量 |
| AQE | **disabled** | 確保不會自動 skew join optimization |
| shuffle.partitions | **10** | 少量 partition 放大 skew 效果 |

## CloudFormation 模板

📄 [`cfn/10-spark-data-skew.yaml`](cfn/10-spark-data-skew.yaml)

### 關鍵設計
- **Lambda + Custom Resource**：透過 Lambda 將 base64 編碼的 PySpark script 上傳到 S3
- **PySpark Script**：`spark_skew_demo.py` — 產生 5 億筆資料，99% 集中在 `hot_key`，執行 `groupBy` + 多重聚合 + 寫入 HDFS
- **Spark Config**：`spark.sql.adaptive.enabled=false` + `spark.sql.shuffle.partitions=10`，雙重確保 skew 不被自動緩解
- **Spark History Server**：啟用 `spark.eventLog.enabled=true`，方便事後在 Spark UI 觀察 task 分佈
- **Service Role**：使用帳號既有的 `EMR_DefaultRole`
- **EC2 Role**：使用 `EMR_EC2_DefaultRole`

### PySpark Script 邏輯（`spark_skew_demo.py`）

```python
# 資料分佈
TOTAL_RECORDS = 500_000_000
HOT_KEY_RATIO = 0.99  # hot_key 佔 99%

# hot_key: 4.95 億筆，每筆 ~1KB payload
hot_rdd = sc.range(0, hot_count).map(
    lambda x: ("hot_key", x, "payload_" + "X" * 1000)
)
# key_001~009: 500 萬筆，均勻分佈
normal_rdd = sc.range(0, normal_count).map(
    lambda x: (f"key_{(x % 9) + 1:03d}", x, "payload_" + "Y" * 1000)
)

# groupBy 聚合 → 觸發 shuffle → hot_key partition 承受 99% 資料
skewed_df.groupBy("join_key").agg(
    count("*"), sum("record_id"), avg("record_id"), max("record_id"), min("record_id")
).write.mode("overwrite").parquet("hdfs:///tmp/skew_output")
```

## 部署方式

```bash
# 前置：帳號需有 EMR_DefaultRole（執行 aws emr create-default-roles 建立）
# 前置：需有 EC2 Key Pair 和既有的 S3 bucket
aws cloudformation create-stack \
  --stack-name emr-lab-10-spark-data-skew \
  --template-body file://cfn/10-spark-data-skew.yaml \
  --parameters \
    ParameterKey=S3BucketName,ParameterValue=your-emr-bucket \
    ParameterKey=KeyName,ParameterValue=your-key-pair \
    ParameterKey=SubnetId,ParameterValue=subnet-xxxxxxxx \
  --capabilities CAPABILITY_NAMED_IAM \
  --region ap-northeast-1
```

部署後取得 Cluster ID：
```bash
aws cloudformation describe-stacks \
  --stack-name emr-lab-10-spark-data-skew \
  --query 'Stacks[0].Outputs[?OutputKey==`ClusterId`].OutputValue' \
  --output text
```

查看 Spark History Server（需 SSH tunnel）：
```bash
ssh -i <key.pem> -N -L 18080:localhost:18080 hadoop@<master-dns>
# 瀏覽器開啟 http://localhost:18080
# -> App: DataskewDemo -> Stages -> GroupBy+Write stage
# -> Tasks -> Duration: 1 straggler >> others
# -> Shuffle Read: hot_key task holds ~99% of data
```

## 預期症狀

- Step `SparkDataSkewDemo` 狀態 **COMPLETED**（job 不會失敗，但耗時極長 ~9.4 分鐘）
- Spark History Server 中可觀察到：
  - GroupBy stage 有 4 個 task（shuffle.partitions=10，但 key 只有 10 個，hash 分佈到 4 個有效 partition）
  - Task 0/1 各處理 ~2.475 億筆 `hot_key` 資料，耗時 ~534 秒
  - Task 2/3 處理 ~250 萬筆 normal key 資料，耗時僅 ~10 秒
  - 任務時長傾斜比：**53.5 倍** 🔴
  - 2/3 的 Executor 核心閒置超過 8 分鐘
- Driver log 輸出：`Aggregation completed in ~566s`
- 集群資源利用率嚴重不足（75% 時間處於閒置）

## Spark History Server 觀察重點

| 觀察項目 | 位置 | 預期異常值 |
|----------|------|-----------|
| Stage Duration | Stages tab → Stage 0 (GroupBy) | 佔整體執行時間 94.3% |
| Task Duration 分佈 | Stage detail → Tasks | Task 0/1: ~534s vs Task 2/3: ~10s（53x 差距）|
| Shuffle Read Size | Stage detail → Tasks → Shuffle Read | Task 0/1 各 ~2.475 億筆 vs Task 2/3 各 ~250 萬筆 |
| Data Skew 比 | Summary Metrics | 100:1（hot_key vs normal keys）|
| GC Time | Stage detail → Tasks → GC Time | Task 0/1 GC 開銷 5-8%（正常範圍）|
| Shuffle Spill | Stage detail → Tasks → Spill | 0 bytes（記憶體充足，無溢出）|
| Executor 閒置 | Executors tab | 浪費的 Executor 時間 ~1,068 秒 |

## 關鍵指標摘要

| 指標 | 數值 | 嚴重程度 |
|------|------|----------|
| 數據傾斜比 | 100:1 | 🔴 極端 |
| 任務時長傾斜 | 53.5x | 🔴 極端 |
| 浪費的 Executor 時間 | 1,068 秒 | 🔴 嚴重 |
| GC 開銷 | 5-8% | 🟢 正常 |
| Shuffle 溢出 | 0 bytes | 🟢 優秀 |
| 任務失敗/重試 | 0 | 🟢 正常 |

## 測試 Prompt

> My Spark job "DataskewDemo" on EMR cluster **j-XXXXX** completed but took 9.4 minutes for a simple groupBy aggregation on 500 million records. I noticed some tasks in the shuffle stage took significantly longer than others. Help me diagnose.

## 預期 DevOps Agent 應能

1. 識別 cluster 配置（1 Primary + 3 Core, m7g.2xlarge）和 Spark 設定
2. 透過 Spark History Server / event log 發現 task duration 分佈不均（534s vs 10s）
3. 識別 data skew — `hot_key` 佔 99% 資料量，導致 Task 0/1 過載
4. 發現 AQE 被關閉（`spark.sql.adaptive.enabled=false`）是關鍵因素
5. 發現 `shuffle.partitions=10` 過少，加劇 skew 影響
6. 確認集群本身健康（無 OOM、無 spill、GC 正常、無網路瓶頸）
7. 建議修復方案（含預期效果）：

| 優化方案 | 預計耗時 | 速度提升 |
|----------|----------|----------|
| 當前（無優化） | 566 秒 | 基準 |
| 啟用 AQE | ~300 秒 | 1.9x |
| AQE + 200 分區 | ~180 秒 | 3.1x |
| AQE + 200 分區 + Salting | ~125 秒 | 4.5x ✨ |

具體建議：
- **啟用 AQE**：`spark.sql.adaptive.enabled=true` + `spark.sql.adaptive.skewJoin.enabled=true` + `spark.sql.adaptive.coalescePartitions.enabled=true`
- **增加 shuffle partitions**：`spark.sql.shuffle.partitions=200`（當前僅 10）
- **Salting 技巧**：對 hot key 加 random suffix 打散到多個 partition，先 partial agg 再 final agg
- **Two-phase aggregation**：先按 salted key 聚合，再合併回原始 key

---

## ✅ 測試結果

> _待測試後填寫_

## Demo 錄影

📹 [demos/10-spark-data-skew-test.mp4](demos/10-spark-data-skew-test.mp4)

## 清理

```bash
aws cloudformation delete-stack --stack-name emr-lab-10-spark-data-skew --region ap-northeast-1
```
