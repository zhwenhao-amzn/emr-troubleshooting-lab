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
1. 從 error log 識別 schema mismatch 細節
2. 比對 source DataFrame 和 target table schema
3. 建議啟用 write.spark.accept-any-schema 或 mergeSchema
4. 說明 Iceberg schema evolution 機制
