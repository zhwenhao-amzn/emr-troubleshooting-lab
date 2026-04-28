# Scenario 07: Iceberg Snapshot Expiration OOM

## 問題描述
Iceberg table 累積過多 snapshot 未清理，執行 expire_snapshots 操作時因 metadata 過大導致 OOM。

## 模擬方式
- 建立 Iceberg table 並執行大量寫入操作累積 snapshot
- 設定極低的 driver memory 執行 expire_snapshots
- 觸發 metadata 載入時的 OOM

## 預期症狀
- expire_snapshots 操作 OOM 失敗
- Iceberg metadata.json 檔案過大
- S3 上累積大量 snapshot 相關檔案

## 測試 Prompt
> "Iceberg expire_snapshots operation failed with OOM on EMR cluster j-XXXXX. The table has been running for months without maintenance."

## 預期 DevOps Agent 應能
1. 識別 snapshot 數量過多
2. 建議增加 driver memory 執行 expire_snapshots
3. 建議分批清理（設定 older_than 逐步縮小範圍）
4. 建議設定定期 snapshot 清理排程
