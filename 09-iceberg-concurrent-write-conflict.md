# Scenario 09: Iceberg Concurrent Write Conflict

## 問題描述
多個 Spark job 同時寫入同一 Iceberg table，觸發 optimistic concurrency conflict（CommitFailedException）。

## 模擬方式
- 建立 Iceberg table
- 同時提交 2-3 個 Spark job 寫入同一 table
- 觸發 commit conflict 導致部分 job 失敗

## 預期症狀
- 部分 Step 失敗，error: CommitFailedException
- Iceberg metadata 顯示 commit retry 失敗
- 成功的 job 正常完成

## 測試 Prompt
> "Some of my Spark jobs writing to the same Iceberg table are failing with CommitFailedException on EMR cluster j-XXXXX. Not all jobs fail, just some."

## 預期 DevOps Agent 應能
1. 從 error log 識別 CommitFailedException
2. 解釋 Iceberg optimistic concurrency 機制
3. 建議增加 commit.retry.num-retries
4. 建議架構層面避免 concurrent write（分 partition 寫入或用 queue）
