# Scenario 03: Iceberg Small Files Problem

## 問題描述
頻繁小批量寫入 Iceberg table 產生大量小檔案，導致 query 效能嚴重劣化、S3 ListObjects 請求爆量。

## 模擬方式
- 建立 Iceberg table（Glue Catalog + S3）
- 用 Spark job 執行 100+ 次小批量 append（每次寫入少量 rows）
- 最後執行 query job 讀取整張 table，因大量小檔案而極慢或超時

## 預期症狀
- Query job 執行時間異常長
- S3 ListObjects 請求量暴增
- Iceberg metadata/manifest 檔案數量膨脹
- Spark task 數量極多但每個 task 處理資料量極小

## 測試 Prompt
> "My Spark query on Iceberg table is extremely slow on EMR cluster j-XXXXX. It used to take 30 seconds but now takes over 10 minutes. Help me troubleshoot."

## 預期 DevOps Agent 應能
1. 檢查 Iceberg table metadata 發現大量 manifest/data files
2. 識別 small files problem
3. 建議執行 Iceberg compaction（rewrite_data_files）
4. 建議設定 write.target-file-size-bytes 和定期 maintenance
