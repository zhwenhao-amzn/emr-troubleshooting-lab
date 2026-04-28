# Scenario 02: Spark OOM — OutOfMemoryError

## 問題描述
Spark job 執行中因 driver 或 executor 記憶體不足而失敗，拋出 OutOfMemoryError。

## 模擬方式
- 使用小 instance（m5.xlarge）
- 提交 Spark job 處理大量資料，但設定極低的 driver/executor memory
- 觸發 Java heap OOM 或 container killed by YARN (exceeded memory limits)

## 預期症狀
- Step 狀態 FAILED
- stderr log: `java.lang.OutOfMemoryError: Java heap space` 或 `Container killed by YARN for exceeding memory limits`
- YARN 顯示 container exit code 137 (OOM killed)

## 測試 Prompt
> "Spark job failed with OutOfMemoryError on EMR cluster j-XXXXX. The step shows FAILED status. Help me troubleshoot."

## 預期 DevOps Agent 應能
1. 讀取 step logs 找到 OOM error
2. 識別是 driver OOM 還是 executor OOM
3. 建議調整 spark.driver.memory / spark.executor.memory / spark.memory.fraction
4. 建議檢查 data skew 或 broadcast join 過大
