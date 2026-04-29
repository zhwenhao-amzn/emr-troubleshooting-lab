# Scenario 08: Iceberg Schema Evolution 衝突

## 問題描述
寫入端的 DataFrame schema 與 Iceberg table schema 不匹配，導致 Spark job 失敗。

## 模擬方式
- 建立 Iceberg table 定義特定 schema
- 用 Spark job 寫入不同 schema 的資料（新增欄位或型別不匹配）
- 未啟用 schema evolution 導致寫入失敗

## 預期症狀
- Spark job 失敗，error log 顯示 schema mismatch
- AnalysisException 或 ValidationException

## 測試 Prompt
> "Spark job writing to Iceberg table failed with schema error on EMR cluster j-XXXXX. The job was working fine yesterday."

## 預期 DevOps Agent 應能
1. 從 error log 識別 schema mismatch 細節（5 分）
2. 比對 source DataFrame 和 target table schema（5 分）
3. 建議啟用 schema evolution（mergeSchema / accept-any-schema）（5 分）
4. 說明 Iceberg schema evolution 機制（5 分）
5. Bonus：提供可執行的修復命令（5 分）

## 架構設計

| 組件 | 設定 | 目的 |
|------|------|------|
| Primary Node | m5.xlarge | ResourceManager |
| Core Node | m5.xlarge | YARN NodeManager |
| Iceberg Table | orders（4 欄：order_id, customer_name, amount, order_date） | 原始 schema |
| Mismatch Write | 7 欄（+category, status, priority） | 模擬 pipeline 升級後 schema drift |
| Glue Catalog | emr_lab_08_schema_db.orders | Iceberg catalog |

## CloudFormation 模板

📄 [`cfn/08-iceberg-schema-evolution-conflict.yaml`](cfn/08-iceberg-schema-evolution-conflict.yaml)

### 3 個 Steps
1. `UploadScripts` — 上傳 schema_setup.py + schema_mismatch.py 到 S3
2. `CreateTableAndSeedData` — 建立 4 欄 Iceberg table + 寫入 100 rows
3. `SchemaMismatchWrite` — 嘗試寫入 7 欄 DataFrame → FAILED（預期行為）

## 部署方式

```bash
aws cloudformation create-stack \
  --stack-name emr-lab-08-schema-conflict \
  --template-body file://cfn/08-iceberg-schema-evolution-conflict.yaml \
  --parameters \
    ParameterKey=SubnetId,ParameterValue=subnet-d4433188 \
    ParameterKey=LogBucket,ParameterValue=aws-zhwenhao-logs \
  --capabilities CAPABILITY_NAMED_IAM \
  --region us-east-1
```

## 測試 Prompt

> Spark job writing to Iceberg table failed on EMR cluster **j-4B3GXY7HHB75**. The table is in Glue database `emr_lab_08_schema_db`, table name `orders`. The initial data load worked fine, but a new version of our pipeline deployed today started failing. The step name is `SchemaMismatchWrite`. Help me troubleshoot.

---

## ✅ 測試結果（2026-04-29）

**Cluster ID**: `j-4B3GXY7HHB75`
**Stack**: `emr-lab-08-schema-conflict`
**Account**: `104172191111` / `us-east-1`
**測試工具**: AWS DevOps Agent

### 第一輪回答摘要

Agent 正確識別了 root cause：

> INSERT_COLUMN_ARITY_MISMATCH.TOO_MANY_DATA_COLUMNS — Table has 4 columns, DataFrame has 7 columns.

關鍵發現：
- ✅ 精準找到 AnalysisException 錯誤，用 `^^^^^^^^` 視覺標記新增欄位
- ✅ 識別 category/status/priority 是新增的 3 個欄位
- ✅ 提供 3 個方案：ALTER TABLE / mergeSchema / drop columns
- ✅ 正確推薦 schema evolution（Option 1）
- ⚠️ mergeSchema 的 API 寫法不正確（Iceberg ≠ Delta Lake）

### Follow-up 追問

Prompt: "verify the table schema after making changes"

- ✅ 確認 table 仍為 4 欄，展示 Field ID（1-4）
- ✅ 提供 EMR add-steps CLI（用 `spark-sql -e` 直接跑 SQL）
- ✅ 提供 PySpark script 方案

### 評分

| 評分項目 | 分數 | 評語 |
|----------|------|------|
| 識別 schema mismatch | 5/5 | 精準找到 INSERT_COLUMN_ARITY_MISMATCH，列出 4 vs 7 欄位 |
| 比對 source vs target schema | 5/5 | 清楚標示新增欄位，視覺呈現優秀 |
| 建議 schema evolution | 4/5 | 3 個方案但 mergeSchema API 寫法有誤 |
| 說明 Iceberg schema evolution 機制 | 5/5 | Follow-up 展示 Field ID tracking |
| Bonus：可執行修復命令 | 5/5 | ALTER TABLE SQL + spark-sql -e EMR step |

**總分：24/25 — 優秀 🎉**

### 亮點
- 錯誤訊息視覺呈現清楚（`^^^^^^^^` 標記）
- 提供 3 個方案適合不同需求
- `spark-sql -e` 直接跑 SQL 是實用技巧

### 改進空間
- Iceberg 的 mergeSchema 寫法與 Delta Lake 不同，應使用 `spark.sql.catalog.glue_catalog.accept-any-schema=true`

## 清理

```bash
aws cloudformation delete-stack --stack-name emr-lab-08-schema-conflict --region us-east-1
```
