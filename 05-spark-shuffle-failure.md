# Scenario 05: Spark Shuffle — FetchFailedException

## 問題描述
Spark job 在 shuffle 階段失敗，拋出 FetchFailedException，通常伴隨 executor lost。

## 模擬方式
- 使用小 instance（m5.xlarge）搭配極小的 EBS volume（10 GB）
- 提交需要大量 shuffle 的 Spark job（大表 join / groupBy）
- Shuffle 資料量超過磁碟空間，觸發 "No space left on device" → executor crash → FetchFailedException

## 預期症狀
- Step 狀態 FAILED
- stderr log: `org.apache.spark.shuffle.FetchFailedException`
- 伴隨 `ExecutorLostFailure` 或 `java.io.IOException: No space left on device`
- YARN 顯示多個 container failed

## 測試 Prompt
> "Spark job on cluster j-XXXXX failed with FetchFailedException during a large join operation. Multiple executors were lost. Help me diagnose."

## 預期 DevOps Agent 應能
1. 讀取 step logs 識別 FetchFailedException root cause
2. 檢查 EBS volume 使用率 / disk space
3. 識別 shuffle spill 過大導致磁碟滿
4. 建議：增加 EBS volume、啟用 spark.shuffle.compress、調整 spark.sql.shuffle.partitions
