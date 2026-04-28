# Scenario 04: Bootstrap Action Failure

## 問題描述
EMR 叢集啟動時 bootstrap action 失敗，叢集進入 TERMINATED_WITH_ERRORS 狀態。

## 模擬方式
- 設定一個會失敗的 bootstrap action script（例如安裝不存在的 package）
- 叢集啟動後 bootstrap 階段失敗，自動 terminate

## 預期症狀
- 叢集狀態 TERMINATED_WITH_ERRORS
- State change reason: `BOOTSTRAP_FAILURE`
- Bootstrap action log 在 s3://bucket/cluster-id/node/instance-id/bootstrap-actions/1/stderr

## 測試 Prompt
> "My EMR cluster j-XXXXX terminated during startup with BOOTSTRAP_FAILURE. How do I find what went wrong?"

## 預期 DevOps Agent 應能
1. 讀取叢集 state change reason
2. 引導查看 bootstrap action logs（S3 路徑）
3. 識別失敗的 bootstrap script 和具體 error
4. 建議修正 bootstrap script 或改用 custom AMI
