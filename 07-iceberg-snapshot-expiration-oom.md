# Scenario 07: Iceberg Snapshot Expiration OOM — expire_snapshots Failed on Bloated Table

## 問題描述
Iceberg table 長期運行未做 snapshot 維護，累積 500+ snapshots。嘗試執行 `expire_snapshots` 清理時，因 driver memory 不足導致 Spark 無法啟動（INVALID_DRIVER_MEMORY），即使 Spark 能啟動，metadata 載入也會消耗大量記憶體。

## 架構設計

| 組件 | 設定 | 目的 |
|------|------|------|
| Primary Node | m5.xlarge (4 vCPU, 16 GB) | 跑 ResourceManager |
| Core Node | m5.xlarge (4 vCPU, 16 GB) | YARN 可用 ~12 GB |
| Iceberg Table | 500 appends, 每次 10 rows | 累積 500 snapshots + 501 metadata.json |
| expire_snapshots Step | driver=256m, overhead=128m | 故意設太低，觸發 INVALID_DRIVER_MEMORY |
| Iceberg Version | 1.5.2 (via spark.jars.packages) | EMR 7.1.0 內建 |
| Glue Catalog | emr_lab_07_iceberg_snapshot_oom_db.metrics | AWS Glue 作為 Iceberg catalog |

### 關鍵 Table Properties
```
format-version = 2
write.metadata.delete-after-commit.enabled = false   ← 不自動清理舊 metadata
history.expire.max-snapshot-age-ms = 999999999999999  ← 永不自動過期
```

## CloudFormation 模板

📄 [`cfn/07-iceberg-snapshot-expiration-oom.yaml`](cfn/07-iceberg-snapshot-expiration-oom.yaml)

### 關鍵設計
- **Service Role**: `EMR_DefaultRole`
- **EC2 Role**: 自建，附加 `AmazonElasticMapReduceforEC2Role` + `CloudWatchAgentServerPolicy` + Glue/S3 全權限
- **3 個 Steps**:
  1. `UploadScripts` — 上傳 Python scripts 到 S3
  2. `FloodSnapshots` — 用 2g driver/executor 執行 500 次 append（每次 10 rows，每次產生新 snapshot）
  3. `ExpireSnapshotsOOM` — 用 256m driver 嘗試 expire_snapshots → 失敗

## 部署方式

```bash
aws cloudformation create-stack \
  --stack-name emr-lab-07-snapshot-oom \
  --template-body file://cfn/07-iceberg-snapshot-expiration-oom.yaml \
  --parameters \
    ParameterKey=SubnetId,ParameterValue=subnet-d4433188 \
    ParameterKey=LogBucket,ParameterValue=aws-zhwenhao-logs \
  --capabilities CAPABILITY_NAMED_IAM \
  --region us-east-1
```

## Iceberg Metadata 狀態

### Snapshot 累積
- **500 snapshots** 在 7 分鐘內產生（~1.2 commits/sec）
- Glue table version: **500**（每次 append 更新一次 Glue table pointer）

### Metadata 檔案大小（線性增長）
| Metadata Version | 大小 | 說明 |
|-----------------|------|------|
| 00000 | 1.5 KiB | 初始 CREATE TABLE |
| 00100 | 116.6 KiB | 100 snapshots |
| 00200 | 208.2 KiB | 200 snapshots |
| 00300 | 300.6 KiB | 300 snapshots |
| 00377 | 370.3 KiB | S3 listing 截斷處 |
| 00500 | ~560 KiB（估算）| 最終版本 |

每個 metadata.json 包含所有歷史 snapshot 引用，因此大小線性增長（~1.1 KiB/version）。

### S3 路徑
- **Data**: `s3://emr-lab-07-snapshot-oom-iceberg-data-104172191111/warehouse/emr_lab_07_iceberg_snapshot_oom_db.db/metrics/data/`
- **Metadata**: `s3://emr-lab-07-snapshot-oom-iceberg-data-104172191111/warehouse/emr_lab_07_iceberg_snapshot_oom_db.db/metrics/metadata/`

## Step Failure Log 分析

### Step: ExpireSnapshotsOOM（s-025063721K4U2ASH4YXH）

**Controller log**:
```
INFO startExec 'hadoop jar command-runner.jar spark-submit --master yarn --deploy-mode cluster
  --conf spark.driver.memory=256m --conf spark.driver.memoryOverhead=128m ...'
INFO waitProcessCompletion ended with exit code 1
INFO total process run time: 26 seconds
WARN Step failed with exitCode 1 and took 26 seconds
```

**Container stderr** (AM container):
```
ERROR SparkContext: Error initializing SparkContext.
org.apache.spark.SparkIllegalArgumentException: [INVALID_DRIVER_MEMORY]
  System memory 268435456 must be at least 471859200.
  Please increase heap size using the --driver-memory option or "spark.driver.memory"
```

**Container stdout** (Python traceback):
```
py4j.protocol.Py4JJavaError: An error occurred while calling None.org.apache.spark.api.java.JavaSparkContext.
: org.apache.spark.SparkIllegalArgumentException: [INVALID_DRIVER_MEMORY]
  System memory 268435456 must be at least 471859200.
```

**關鍵數字**:
- 256m = 268,435,456 bytes
- Spark 3.5 最低需求 = 471,859,200 bytes (~450 MB)
- AM container 分配 = 384 MB（256m + 128m overhead）
- Application 嘗試 2 次後 FAILED（exit code 13）

### 為什麼不是真正的 java.lang.OutOfMemoryError？
Spark 3.5 在 `UnifiedMemoryManager.getMaxMemory()` 有前置檢查：如果 JVM heap < `spark.testing.reservedMemory`(300MB) × 1.5 = 450MB，直接拋 `INVALID_DRIVER_MEMORY` 而不是等到 runtime OOM。這是 Spark 的 fail-fast 機制。

## 測試 Prompt

> Iceberg expire_snapshots operation failed with OOM on EMR cluster **j-10YNHL84WDBR**. The table is in Glue database `emr_lab_07_iceberg_snapshot_oom_db`, table name is `metrics`. The table has been running for months without maintenance. Help me troubleshoot.

## 預期 DevOps Agent 應能

1. **識別 snapshot 數量過多**（5 分）
   - 查 Glue table version（500）或 S3 metadata 檔案數量
   - 識別 `write.metadata.delete-after-commit.enabled=false` 和超長 `history.expire.max-snapshot-age-ms`

2. **分析 step failure root cause**（5 分）
   - 找到 INVALID_DRIVER_MEMORY 錯誤
   - 解釋 256m 不足以啟動 Spark 3.5（需要 ~450MB）
   - 區分 INVALID_DRIVER_MEMORY vs java.lang.OutOfMemoryError

3. **建議增加 driver memory 重新執行 expire_snapshots**（5 分）
   - 建議至少 1g-2g driver memory
   - 提供可執行的 spark-submit 命令或 EMR add-steps CLI

4. **建議分批清理策略**（5 分）
   - 設定 `older_than` 逐步縮小範圍（先清 30 天前，再 7 天前）
   - 或設定 `retain_last` 保留最近 N 個 snapshot

5. **Bonus：建議預防措施**（5 分）
   - 設定 `write.metadata.delete-after-commit.enabled=true`
   - 設定合理的 `history.expire.max-snapshot-age-ms`（如 7 天）
   - 建議定期排程 expire_snapshots（如 EMR Step / Airflow / Lambda）
   - 建議設定 `write.metadata.previous-versions-max`（限制 metadata 保留數量）

---

## ✅ 測試結果（2026-04-29）

**Cluster ID**: `j-10YNHL84WDBR`
**Stack**: `emr-lab-07-snapshot-oom`
**Account**: `104172191111` / `us-east-1`
**測試工具**: AWS DevOps Agent

### 第一輪回答摘要

Agent 正確識別了 root cause：

> Driver memory was too small (256 MB) to handle the accumulated metadata. With 500 snapshots embedded in a single file, 384 MB wasn't enough.

關鍵發現：
- ✅ 識別 500 snapshots 累積
- ✅ 找到 `history.expire.max-snapshot-age-ms=999999999999999` 和 `write.metadata.delete-after-commit.enabled=false`
- ✅ 發現 512m retry step 已成功完成
- ✅ 提供 `ALTER TABLE` SQL 修正 table properties
- ⚠️ 誤述為「YARN killed the container with exit code 13」（實際是 Spark INVALID_DRIVER_MEMORY fail-fast）
- ⚠️ 未提出分批清理策略

### Follow-up 追問

Prompt: "dig into the S3 metadata directory to see how much cleanup expire_snapshots performed"

Agent 深入分析後發現：

> No cleanup actually happened, even though expire_snapshots completed successfully. Since no snapshot is older than 31.7 million years, none qualified for expiration.

追加關鍵發現：
- ✅ 精準計算 S3 檔案數量（501 metadata.json、501 snap-*.avro、~1001 manifests）
- ✅ 發現 expire 成功但沒有實際清理（關鍵洞察）
- ✅ 提供 `retain_last` 和 `older_than` 兩種清理方案
- ✅ 補充 `write.metadata.previous-versions-max` 設定

### 評分

| 評分項目 | 分數 | 評語 |
|----------|------|------|
| 識別 snapshot 數量過多 | 5/5 | 精準識別 500 snapshots + 兩個關鍵 table properties |
| 分析 step failure root cause | 4/5 | 找到 256m 不足，但誤述為 YARN kill（實際是 INVALID_DRIVER_MEMORY） |
| 建議增加 driver memory | 5/5 | 發現 512m retry 已成功，提供 rule of thumb |
| 分批清理策略 | 5/5 | Follow-up 中提供 retain_last + older_than 兩種方案 |
| Bonus：預防措施 | 5/5 | ALTER TABLE SQL + previous-versions-max + 定期排程建議 |

**總分：24/25 — 優秀 🎉**

### 亮點
- 主動發現 512m retry step 已成功（不只看 FAILED steps）
- Follow-up 中發現 expire 成功但沒實際清理（expire 成功 ≠ 有東西被清理）
- 回答末尾主動提供 follow-up 選項，引導用戶深入排查
- 提供可直接執行的 SQL 命令

### 改進空間
- exit code 13 的解釋不精確：不是 YARN kill（那是 137），而是 Spark `UnifiedMemoryManager` 的前置檢查 `INVALID_DRIVER_MEMORY`（heap 256m < 最低要求 ~450m）
- 第一輪未主動提出分批清理策略（需追問才補上）

## 清理

```bash
aws cloudformation delete-stack --stack-name emr-lab-07-snapshot-oom --region us-east-1
# S3 bucket 需手動清空後才能刪除（或保留供其他場景使用）
```
