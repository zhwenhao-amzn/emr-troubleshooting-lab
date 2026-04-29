# Scenario 09: Iceberg Concurrent Write Conflict

## 問題描述
多個 Spark job 同時寫入同一 Iceberg table，觸發 optimistic concurrency conflict（CommitFailedException）。

## 架構設計

| 組件 | 設定 | 目的 |
|------|------|------|
| Primary Node | m5.xlarge | ResourceManager |
| Core Node | m5.xlarge | YARN NodeManager |
| Iceberg Table | events（4 欄：id, event_time, source, data） | 寫入目標 |
| Concurrent Write | 3 Python threads × 8 iterations × 50 rows | 同時寫入觸發 conflict |
| Retry Config | `commit.retry.num-retries=1`, `min-wait-ms=0` | 故意設極低，最大化 conflict |
| Glue Catalog | emr_lab_09_iceberg_concurrent_write_db.events | Iceberg catalog |

## CloudFormation 模板

📄 [`cfn/09-iceberg-concurrent-write-conflict.yaml`](cfn/09-iceberg-concurrent-write-conflict.yaml)

### 3 個 Steps
1. `UploadScripts` — 上傳 create_table.py + concurrent_write.py 到 S3
2. `CreateTableAndSeed` — 建立 Iceberg table + 寫入 100 rows seed data
3. `ConcurrentWriteConflict` — 3 個 thread 同時寫入 → Thread 0 FAILED（CommitFailedException）

### 關鍵設計
- `commit.retry.num-retries=1` + `commit.retry.min-wait-ms=0`：最大化 conflict 機率
- 3 個 thread 共用同一 SparkSession：模擬真實的 concurrent write 場景
- Python try/except 捕獲異常：EMR step 顯示 COMPLETED，但 container log 有 FAILED thread

## 部署方式

```bash
aws cloudformation create-stack \
  --stack-name emr-lab-09-concurrent-write \
  --template-body file://cfn/09-iceberg-concurrent-write-conflict.yaml \
  --parameters \
    ParameterKey=SubnetId,ParameterValue=subnet-d4433188 \
    ParameterKey=LogBucket,ParameterValue=aws-zhwenhao-logs \
  --capabilities CAPABILITY_NAMED_IAM \
  --region us-east-1
```

## 實際執行結果

| Thread | 結果 | 寫入量 |
|--------|------|--------|
| Thread 0 | ❌ FAILED（CommitFailedException） | 0 rows |
| Thread 1 | ✅ SUCCESS | 400 rows（8 × 50） |
| Thread 2 | ✅ SUCCESS | 400 rows（8 × 50） |

**最終 table**: 900 rows（100 seed + 800 successful）— 資料完整性正確。

### Error Log
```
org.apache.iceberg.exceptions.CommitFailedException: Cannot commit
  glue_catalog.emr_lab_09_iceberg_concurrent_write_db.events
  because Glue detected concurrent update

Caused by: ConcurrentModificationException: Update table failed
  due to concurrent modifications. (Service: Glue, Status Code: 400)
```

## 測試 Prompt

> Some of my Spark jobs writing to the same Iceberg table are failing intermittently with CommitFailedException on EMR cluster **j-31DCCZHX2DO8Q**. The table is in Glue database `emr_lab_09_iceberg_concurrent_write_db`, table name `events`. Not all writes fail — some succeed while others get "Glue detected concurrent update". The step name is `ConcurrentWriteConflict`. Help me troubleshoot.

## 預期 DevOps Agent 應能

1. **從 log 識別 CommitFailedException**（5 分）
2. **解釋 Iceberg optimistic concurrency 機制**（5 分）
3. **識別 `commit.retry.num-retries=1` 是關鍵設定**（5 分）
4. **建議增加 retry + 架構改進**（5 分）
5. **Bonus：Glue catalog locking + 資料完整性確認**（5 分）

---

## ✅ 測試結果（2026-04-29）

**Cluster ID**: `j-31DCCZHX2DO8Q`
**Stack**: `emr-lab-09-concurrent-write`
**Account**: `104172191111` / `us-east-1`
**測試工具**: AWS DevOps Agent

### 第一輪回答摘要

Agent 完美識別了 root cause：

> 3 Python threads simultaneously sharing the same SparkSession, creating a classic OCC race condition. Table configured with commit.retry.num-retries=1 and min-wait-ms=0.

關鍵發現：
- ✅ 精準找到 CommitFailedException + ConcurrentModificationException
- ✅ 完整解釋 OCC 機制（read same metadata → race to commit → one winner）
- ✅ 識別 retry=1 + min-wait=0ms 是關鍵（對比 default 4/100ms）
- ✅ Snapshot timing 分析精確到毫秒（500ms race window）
- ✅ 確認 data integrity（900 rows 正確）
- ✅ 提供 ALTER TABLE SQL + 4 種架構方案

### Follow-up 追問

Prompt: "modify the table properties to increase retry tolerance, and explore a different write architecture"

- ✅ 提供完整 ALTER TABLE SQL（含 total-timeout-ms）
- ✅ 4 種架構方案：Structured Streaming / Separate Steps / Partition-Level / Iceberg Branches
- ✅ Decision Guide 場景對照表
- ⚠️ 小瑕疵：WAP 與 partition-level parallelism 無直接關係

### 評分

| 評分項目 | 分數 | 評語 |
|----------|------|------|
| 識別 CommitFailedException | 5/5 | 精準找到錯誤 + Glue ConcurrentModificationException |
| 解釋 Iceberg OCC 機制 | 5/5 | 完整解釋 + ASCII 圖示 + 毫秒級 timing 分析 |
| 識別 retry 設定問題 | 5/5 | 找到 retry=1/min-wait=0，對比 default 值 |
| 建議增加 retry + 架構改進 | 5/5 | ALTER TABLE + 4 種架構方案 + Decision Guide |
| Bonus：Glue locking + data integrity | 5/5 | 確認 900 rows 正確，提到 Iceberg branches |

**總分：25/25 — 滿分 🎉🎉🎉**

### 亮點
- Snapshot timing 精確到毫秒（14:53:10.792 vs 14:53:11.302）
- 主動驗證 data integrity（900 = 100 + 800）
- 提到 Iceberg branches 作為進階方案
- Decision Guide 場景對照表非常實用

## 清理

```bash
aws cloudformation delete-stack --stack-name emr-lab-09-concurrent-write --region us-east-1
```
