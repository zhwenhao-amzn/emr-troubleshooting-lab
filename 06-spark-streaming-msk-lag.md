# Scenario 06: Spark Streaming + MSK — Consumer Lag

## 問題描述
Spark Structured Streaming 消費 MSK topic，出現嚴重 consumer lag，處理速度跟不上生產速度。

## 模擬方式
- 建立 MSK cluster（kafka.m5.large, 2 brokers）
- 建立 EMR cluster 執行 Spark Structured Streaming 消費 MSK
- Producer 以高速寫入（每秒數千筆），但 Spark 端設定極低的 maxOffsetsPerTrigger 和小 batch interval
- 造成 consumer lag 持續增長

## 預期症狀
- MSK CloudWatch: `EstimatedMaxTimeLag` 持續上升
- MSK CloudWatch: `SumOffsetLag` 持續增長
- Spark Streaming UI: batch processing time > batch interval
- Spark Streaming UI: scheduling delay 持續增加

## 測試 Prompt
> "My Spark Streaming job on EMR cluster j-XXXXX is consuming from MSK but the consumer lag keeps growing. EstimatedMaxTimeLag is over 30 minutes. How do I fix this?"

## 預期 DevOps Agent 應能
1. 檢查 MSK CloudWatch metrics（SumOffsetLag, EstimatedMaxTimeLag）
2. 檢查 Spark Streaming batch processing time vs interval
3. 識別瓶頸：maxOffsetsPerTrigger 太低、executor 數量不足、或 processing logic 太慢
4. 建議：增加 maxOffsetsPerTrigger、增加 executor/partition 數、優化 processing logic
