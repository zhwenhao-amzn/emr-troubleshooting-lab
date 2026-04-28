# Scenario 02: Spark OOM — OutOfMemoryError

## 問題描述
Spark job 執行中因 executor 記憶體不足而失敗，YARN kill container（exit code 137）。

## 架構設計

| 組件 | 設定 | 目的 |
|------|------|------|
| Primary Node | m5.xlarge (4 vCPU, 16 GB) | ResourceManager |
| Core Node | m5.xlarge (4 vCPU, 16 GB) | 跑 executor |
| Driver Memory | **512m** | 刻意設低 |
| Executor Memory | **1g** | 不足以處理 50M rows |
| PySpark Script | `oom_job.py` — 50M rows × 200 bytes + `.collect()` | 觸發 OOM |

## CloudFormation 模板

📄 [`cfn/02-spark-oom.yaml`](cfn/02-spark-oom.yaml)

### 關鍵設計
- **兩步驟 Step**：Step 1 建立 PySpark script 上傳到 S3，Step 2 用 `spark-submit` 執行
- **OOM 觸發機制**：`spark.range(50M).select(200-byte payload).collect()` — 把 ~10GB 資料拉到 512MB driver
- **實際失敗模式**：Executor 在處理大量資料時被 YARN OOM kill（exit code 137），重試 3 次後 app 終止

## 部署方式

```bash
aws cloudformation create-stack \
  --stack-name emr-lab-02-spark-oom \
  --template-body file://cfn/02-spark-oom.yaml \
  --parameters \
    ParameterKey=SubnetId,ParameterValue=subnet-xxxxxxxx \
    ParameterKey=LogBucket,ParameterValue=your-log-bucket \
  --capabilities CAPABILITY_NAMED_IAM \
  --region us-east-1
```

## 預期症狀

- Step `SparkDriverOOM` 狀態 **FAILED**
- Container log: `Exit code is 137. Container killed on request. Killed by external signal`
- Driver log: `Max number of executor failures (3) reached`
- YARN NodeManager metrics: `ContainersFailed` 增加

## 測試 Prompt

> Spark job failed with OutOfMemoryError on EMR cluster **j-XXXXX**. The step shows FAILED status. Help me troubleshoot.

## 預期 DevOps Agent 應能

1. 找到 step FAILED 狀態和 exit code 137
2. 讀取 container log 識別 OOM kill
3. 區分 driver OOM vs executor OOM
4. 建議調整 `spark.executor.memory` / `spark.driver.memory`
5. **進階**：讀取 `oom_job.py` 識別 `.collect()` anti-pattern

---

## ✅ 測試結果（2026-04-28）

**Cluster ID**: `j-CAZ8IJLVHP9C`
**Stack**: `emr-lab-02-spark-oom`
**Account**: `104172191111` / `us-east-1`

### 第一輪：自動分析

Agent 正確識別：
- Exit code 137 = SIGKILL（OOM killer）
- Driver memory 512m 過低
- AM container 被 kill 兩次

**誤判**：判斷為 AM/Driver OOM，實際是 **Executor OOM**（container_01_000002/003/004 被 kill，3 次 executor 失敗導致 app 終止）

### 第二輪：查 oom_job.py 後修正

請 Agent 查看 `oom_job.py` 後，Agent 從 S3 讀取 script 並精準分析：

> `.collect()` operation attempts to bring all 50 million rows into the driver's memory (512 MB). This is a classic anti-pattern that will always fail with large datasets.

提供 4 個修復方案：
1. **Don't collect — write to storage**: `df.write.parquet("s3://...")`
2. **Limit rows**: `df.limit(1000).collect()`
3. **Use take()**: `df.take(1000)`
4. **Process in batches**: `df.foreachPartition(process_partition)`

並識別出這是刻意的 OOM 測試場景。

### 評分

| 評分項目 | 第一輪 | 第二輪 | 評語 |
|----------|--------|--------|------|
| 識別 step 失敗 | 5/5 | — | 正確找到 exit code 137 |
| 讀取 container log | 3/5 | — | 只看 AM log，沒深入 executor log |
| Root cause 分析 | 3/5 | 5/5 | 第一輪誤判為 AM OOM，第二輪修正為 `.collect()` anti-pattern |
| 解法建議 | 4/5 | 5/5 | 第二輪 4 個方案都是最佳做法 |
| 深入分析能力 | 3/5 | 5/5 | 主動從 S3 讀取 script，精準估算 ~10GB vs 512MB |

**綜合總分：22/25 — 良好 👍**

### 亮點
- 第二輪從 S3 讀取 `oom_job.py` 後立刻修正判斷
- 識別 `.collect()` anti-pattern 並提供 4 個生產級修復方案
- 聰明地判斷出這是刻意的 OOM 測試

### 改進空間
- 第一輪應讀取 executor container log（不只 AM log）來區分 driver vs executor OOM
- 應主動查看 CW metrics（ContainersFailed）佐證分析

## Demo 錄影

📹 （待補）

## 清理

```bash
aws cloudformation delete-stack --stack-name emr-lab-02-spark-oom --region us-east-1
```
