# Scenario 10: Spark Data Skew — Straggler Task

## 問題描述
Spark job 在 `groupBy` 聚合階段出現嚴重的 data skew，99% 的資料集中在單一 key（`hot_key`），導致一個 task 處理時間遠超其他 task（straggler），整體 job 效能極差。AQE（Adaptive Query Execution）被刻意關閉，無法自動緩解 skew。

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

- Step `SparkDataSkewDemo` 狀態 **COMPLETED**（job 不會失敗，但耗時極長）
- Spark History Server 中可觀察到：
  - GroupBy stage 的 10 個 task 中，1 個 task 耗時遠超其他 9 個
  - Straggler task 的 Shuffle Read Size 佔整體 ~99%
  - 其他 9 個 task 很快完成，整個 stage 被 1 個 task 拖住
- Driver log 輸出：`Aggregation completed in XXXs`（預期數分鐘）
- Executor metrics：1 個 executor 的 CPU/Memory 持續高負載，其他 executor 閒置

## Spark History Server 觀察重點

| 觀察項目 | 位置 | 預期異常值 |
|----------|------|-----------|
| Stage Duration | Stages tab → GroupBy stage | 整體耗時被 straggler 拖長 |
| Task Duration 分佈 | Stage detail → Tasks | 1 個 task >> 其他 9 個 |
| Shuffle Read Size | Stage detail → Tasks → Shuffle Read | hot_key task ~495GB vs 其他 ~0.5GB |
| Task Skew | Stage detail → Summary Metrics | Max >> Median（Duration / Shuffle Read） |
| GC Time | Stage detail → Tasks → GC Time | hot_key task GC time 顯著偏高 |

## 測試 Prompt

> My Spark job "DataskewDemo" on EMR cluster **j-XXXXX** completed but took much longer than expected. The job is a simple groupBy aggregation on 500 million records. I noticed one task in the shuffle stage took significantly longer than others. Help me diagnose.

## 預期 DevOps Agent 應能

1. 識別 cluster 配置（1 Primary + 3 Core, m7g.2xlarge）和 Spark 設定
2. 透過 Spark History Server / step logs 發現 task duration 分佈不均
3. 識別 data skew — `hot_key` 佔 99% 資料量，導致單一 partition 過載
4. 發現 AQE 被關閉（`spark.sql.adaptive.enabled=false`）是關鍵因素
5. 發現 `shuffle.partitions=10` 過少，加劇 skew 影響
6. 建議修復方案：
   - **啟用 AQE**：`spark.sql.adaptive.enabled=true` + `spark.sql.adaptive.skewJoin.enabled=true`
   - **增加 shuffle partitions**：`spark.sql.shuffle.partitions=200` 或更高
   - **Salting 技巧**：對 hot key 加 random suffix 打散分佈
   - **Two-phase aggregation**：先 partial agg 再 final agg
   - **Broadcast join**（如適用）：小表 broadcast 避免 shuffle

---

## ✅ 測試結果

> _待測試後填寫_

## Demo 錄影

📹 [demos/10-spark-data-skew-test.mp4](demos/10-spark-data-skew-test.mp4)

## 清理

```bash
aws cloudformation delete-stack --stack-name emr-lab-10-spark-data-skew --region ap-northeast-1
```
