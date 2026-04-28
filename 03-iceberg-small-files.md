# Scenario 03: Iceberg Small Files Problem

## 問題描述
頻繁小批量寫入 Iceberg table 產生大量小檔案，導致 query 效能嚴重劣化。

## 架構設計

| 組件 | 設定 | 目的 |
|------|------|------|
| Primary Node | m5.xlarge | ResourceManager |
| Core Node | m5.xlarge | Executor |
| Iceberg Table | Glue Catalog + S3 | `events` table, partitioned by `days(event_time)` |
| Flood Step | 120 次 append × 50 rows | 產生 120 個 ~2.2KB 小檔案 |
| Query Step | GROUP BY aggregation | 讀取 120 個小檔案，耗時 20.8 秒 |

## CloudFormation 模板

📄 [`cfn/03-iceberg-small-files.yaml`](cfn/03-iceberg-small-files.yaml)

### 關鍵設計
- **Glue Database**: `emr_lab_03_iceberg_small_files_db`（hardcoded，避免 hyphen/underscore 不匹配）
- **S3 Bucket**: 自動建立 `${StackName}-iceberg-data-${AccountId}`
- **三步驟 Step**: Upload scripts → 120 次 small append → slow query
- **Iceberg Config**: `spark.sql.catalog.glue_catalog` + `IcebergSparkSessionExtensions`

## 部署方式

```bash
aws cloudformation create-stack \
  --stack-name emr-lab-03-iceberg-small-files \
  --template-body file://cfn/03-iceberg-small-files.yaml \
  --parameters \
    ParameterKey=SubnetId,ParameterValue=subnet-xxxxxxxx \
    ParameterKey=LogBucket,ParameterValue=your-log-bucket \
  --capabilities CAPABILITY_NAMED_IAM \
  --region us-east-1
```

## 預期症狀

- 120 個 data files，每個 ~2.2KB（total ~263KB）
- 120 個 snapshots + 121 個 metadata files
- 簡單 aggregation query 耗時 20+ 秒（正常應 < 1 秒）

## 測試 Prompt

> My Spark query on Iceberg table is extremely slow on EMR cluster **j-XXXXX**. It used to take 30 seconds but now takes over 10 minutes. The table is in Glue database **emr_lab_03_iceberg_small_files_db**. Help me troubleshoot.

## 預期 DevOps Agent 應能

1. 查 Iceberg table metadata 發現 120 data files
2. 識別 small files problem（每個 file ~2.2KB/50 rows）
3. 查 snapshot 數量（120 個未清理）
4. 建議 `rewrite_data_files` compaction
5. 建議 `write.target-file-size-bytes` + 定期 maintenance

---

## ✅ 測試結果（2026-04-28）

**Cluster ID**: `j-CS6UMJ1KHHXT`
**Stack**: `emr-lab-03-iceberg-small-files`
**Account**: `104172191111` / `us-east-1`

### DevOps Agent 回答摘要

Agent 一輪就精準命中 root cause：

> Your Iceberg table has a **severe small files problem**: 120 files, 263 KB total, 2.2 KB average.

完整分析 4 個影響因素：
1. File Opening Overhead — 120 次 S3 API calls + schema parsing
2. Metadata Explosion — 120 snapshots + 120 manifest files
3. Parallelism Inefficiency — 大量 tiny tasks
4. S3 Latency — 120 files × 50-100ms = 6-12s overhead

提供 4 個解法：
1. **`rewrite_data_files`** — compaction（含 target/min/max file size 參數）
2. **`expire_snapshots`** — 清理 120 個累積 snapshot
3. **`remove_orphan_files`** — 清理孤立 metadata
4. **ALTER TABLE** — 設定 `write.target-file-size-bytes` + `distribution-mode` + `expire.max-snapshot-age-ms`

### 評分

| 評分項目 | 分數 | 評語 |
|----------|------|------|
| 查 Iceberg table metadata | 5/5 | 精準列出 120 files、263KB、2.2KB avg |
| 識別 small files problem | 5/5 | 完整解釋 4 個影響因素 |
| 查 snapshot 數量 | 5/5 | 120 snapshots + 121 metadata files |
| 建議 compaction | 5/5 | `rewrite_data_files` 語法正確 |
| 建議預防措施 | 5/5 | target-file-size + distribution-mode + auto expire |

**總分：25/25 — 滿分 🎉**

### 亮點
- 一輪精準命中，不需 follow-up
- S3 latency 估算精確（6-12s overhead）
- 4 個解法按優先級排列
- 預測 compaction 效果（120 files → 1 file）

## Demo 錄影

📹 [下載 Demo 錄影](demos/03-iceberg-small-files-test.mp4)

## 清理

```bash
aws cloudformation delete-stack --stack-name emr-lab-03-iceberg-small-files --region us-east-1
```
