# EMR Troubleshooting Lab

用 CloudFormation 建立 EMR 異常場景，測試 AWS DevOps Agent 的 troubleshooting 能力。

## 場景列表

| # | 場景 | 難度 | CFN | 狀態 |
|---|------|------|-----|------|
| 01 | [YARN Resource Exhaustion](01-yarn-resource-exhaustion.md) | 中階 | ✅ | ✅ 已測試 (24/25) |
| 02 | [Spark OOM](02-spark-oom.md) | 中階 | ✅ | ✅ 已測試 (22/25) |
| 03 | [Iceberg Small Files Problem](03-iceberg-small-files.md) | 中階 | ✅ | ✅ 已測試 (25/25) |
| 04 | [Bootstrap Failure](04-bootstrap-failure.md) | 中階 | 🔲 | 待測 |
| 05 | [Spark Shuffle Failure](05-spark-shuffle-failure.md) | 進階 | 🔲 | 待測 |
| 06 | [Spark Streaming MSK Lag](06-spark-streaming-msk-lag.md) | 進階 | 🔲 | 待測 |
| 07 | [Iceberg Snapshot Expiration OOM](07-iceberg-snapshot-expiration-oom.md) | 進階 | 🔲 | 待測 |
| 08 | [Iceberg Schema Evolution 衝突](08-iceberg-schema-evolution-conflict.md) | 中階 | 🔲 | 待測 |
| 09 | [Iceberg Concurrent Write Conflict](09-iceberg-concurrent-write-conflict.md) | 進階 | 🔲 | 待測 |

## 架構

每個場景包含：
- **Scenario MD** — 問題描述、架構設計、CW 指標、測試 prompt、評分標準
- **CFN Template** — 一鍵部署異常環境（`cfn/` 目錄）
- **測試結果** — DevOps Agent 回答摘要與評分

## 前置需求

- AWS 帳號已有 EMR default roles（`aws emr create-default-roles`）
- S3 bucket 用於 EMR logs
- VPC subnet

## 快速開始

```bash
# 部署場景 1
aws cloudformation create-stack \
  --stack-name emr-lab-01-yarn-exhaustion \
  --template-body file://cfn/01-yarn-resource-exhaustion.yaml \
  --parameters \
    ParameterKey=SubnetId,ParameterValue=subnet-xxxxxxxx \
    ParameterKey=LogBucket,ParameterValue=your-log-bucket \
  --capabilities CAPABILITY_NAMED_IAM \
  --region us-east-1

# 取得 Cluster ID
aws cloudformation describe-stacks \
  --stack-name emr-lab-01-yarn-exhaustion \
  --query 'Stacks[0].Outputs[?OutputKey==`ClusterId`].OutputValue' \
  --output text
```

## CloudWatch Agent 指標

所有 EMR 7.x 指標發送到 **CWAgent** namespace：
- YARN/HDFS metrics: dimension `service.name=hadoop`
- Spark metrics: dimension `ApplicationID`
- System metrics: `cpu_*`, `mem_*`, `disk_*`

## 清理

```bash
aws cloudformation delete-stack --stack-name emr-lab-01-yarn-exhaustion --region us-east-1
```
