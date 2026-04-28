# Scenario 03: S3 Access Denied

## 問題描述
EMR job 嘗試寫入 S3 時收到 Access Denied，但 EMR instance profile 看起來有 S3 權限。

## 模擬方式
- 建立 S3 bucket 並啟用 KMS 加密
- EMR instance profile 有 s3:PutObject 但缺少 kms:GenerateDataKey 權限
- Spark job 寫入該 bucket 時觸發 Access Denied

## 預期症狀
- Step 狀態 FAILED
- stderr log: `com.amazonaws.services.s3.model.AmazonS3Exception: Access Denied (Service: Amazon S3; Status Code: 403)`
- S3 bucket policy 或 KMS key policy 限制了存取

## 測試 Prompt
> "My EMR job gets Access Denied when writing to s3://emr-lab-output-XXXXX/results/. The cluster role has S3 full access. What's going on?"

## 預期 DevOps Agent 應能
1. 檢查 EMR instance profile / service role 的 IAM policy
2. 檢查 S3 bucket policy
3. 識別 KMS key policy 缺少 EMR role 的 kms:GenerateDataKey
4. 建議修正 KMS key policy 或 IAM policy
