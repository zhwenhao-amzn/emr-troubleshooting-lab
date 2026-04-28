# Scenario 04: Bootstrap Action Failure

## 問題描述
EMR 叢集啟動時 bootstrap action 失敗，叢集進入 TERMINATED_WITH_ERRORS 狀態。

## 架構設計

| 組件 | 設定 | 目的 |
|------|------|------|
| Primary + Core | m5.xlarge × 2 | 標準配置 |
| Bootstrap Script | `bootstrap_install.sh` | 安裝 jq（成功）+ 安裝不存在的 package（失敗）|
| `set -e` | 啟用 | 確保任何命令失敗就中止整個 script |

## CloudFormation 模板

📄 [`cfn/04-bootstrap-failure.yaml`](cfn/04-bootstrap-failure.yaml)

### 前置步驟
需先上傳 bootstrap script 到 S3：
```bash
cat > /tmp/bootstrap_install.sh << 'EOF'
#!/bin/bash
set -e
echo "Installing custom packages..."
sudo yum install -y jq
echo "jq installed successfully"
echo "Installing custom monitoring agent..."
sudo yum install -y custom-monitoring-agent-3.2.1
echo "Bootstrap completed successfully"
EOF
aws s3 cp /tmp/bootstrap_install.sh s3://<LogBucket>/emr-scripts/bootstrap_install.sh
```

### 部署
```bash
aws cloudformation create-stack \
  --stack-name emr-lab-04-bootstrap-failure \
  --template-body file://cfn/04-bootstrap-failure.yaml \
  --parameters \
    ParameterKey=SubnetId,ParameterValue=subnet-xxxxxxxx \
    ParameterKey=LogBucket,ParameterValue=your-log-bucket \
  --capabilities CAPABILITY_NAMED_IAM \
  --region us-east-1
```

**注意**：Stack 會 ROLLBACK_COMPLETE（預期行為），cluster ID 從 stack events 取得。

## 預期症狀

- Cluster 狀態：TERMINATED_WITH_ERRORS
- State change reason：`BOOTSTRAP_FAILURE — bootstrap action 1 returned a non-zero return code`
- Bootstrap stderr：`Error: Unable to find a match: custom-monitoring-agent-3.2.1`
- Log 路徑：`s3://<bucket>/emr-logs/<cluster-id>/node/<instance-id>/bootstrap-actions/1/stderr`

## 測試 Prompt

> My EMR cluster **j-XXXXX** terminated during startup with BOOTSTRAP_FAILURE. How do I find what went wrong?

## 預期 DevOps Agent 應能

1. 查到 cluster 狀態 TERMINATED_WITH_ERRORS + BOOTSTRAP_FAILURE
2. 找到 bootstrap log S3 路徑
3. 讀取 stderr 發現具體 error
4. 建議修正方案

---

## ✅ 測試結果（2026-04-28）

**Cluster ID**: `j-11UI7D1AYIBHT`
**Stack**: `emr-lab-04-bootstrap-failure`
**Account**: `104172191111` / `us-east-1`

### DevOps Agent 回答摘要

Agent 一輪精準命中，完整排查鏈：

1. 查到 cluster TERMINATED_WITH_ERRORS + BOOTSTRAP_FAILURE
2. 從 S3 讀取 bootstrap stderr：`Error: Unable to find a match: custom-monitoring-agent-3.2.1`
3. **主動讀取 bootstrap script 原始碼**，指出 `set -e` 是關鍵
4. 提供 3 個修復方案：
   - 配置 custom yum repo 或從 S3 下載 RPM
   - 移除不存在的 package
   - Error handling：`|| echo "Warning: custom agent not available"`

### 評分

| 評分項目 | 分數 | 評語 |
|----------|------|------|
| 查到 cluster 狀態 | 5/5 | TERMINATED_WITH_ERRORS + BOOTSTRAP_FAILURE |
| 找到 bootstrap log 路徑 | 5/5 | 從 S3 讀取 node/<instance>/bootstrap-actions/1/stderr |
| 讀取 stderr 內容 | 5/5 | 精準找到 package not found error |
| 解法建議 | 5/5 | 3 個方案 + `\|\|` error handling |
| 整體排查流程 | 5/5 | 主動讀 script 原始碼，指出 `set -e` 是關鍵 |

**總分：25/25 — 滿分 🎉**

## Demo 錄影

📹 [下載 Demo 錄影](demos/04-bootstrap-failure-test.mp4)

## 清理

```bash
aws cloudformation delete-stack --stack-name emr-lab-04-bootstrap-failure --region us-east-1
```
