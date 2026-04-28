# Scenario 01: YARN Resource Exhaustion — Spark Job Stuck in ACCEPTED

## 問題描述
Spark job 提交後一直卡在 ACCEPTED 狀態，無法取得 YARN 資源執行。

## 架構設計

| 組件 | 設定 | 目的 |
|------|------|------|
| Primary Node | m5.xlarge (4 vCPU, 16 GB) | 跑 ResourceManager |
| Core Node | m5.xlarge (4 vCPU, 16 GB) | YARN 可用 ~12 GB |
| Spark Job | 5 executors × 10 GB × 4 cores | 需要 50 GB + 20 vCPU → 遠超可用資源 |
| Dynamic Allocation | **disabled** | 確保不會自動降低 executor 數量 |

## CloudFormation 模板

📄 [`cfn/01-yarn-resource-exhaustion.yaml`](cfn/01-yarn-resource-exhaustion.yaml)

### 關鍵設計
- **Service Role**: 使用帳號既有的 `EMR_DefaultRole`（不自建，避免 EC2 權限不足）
- **EC2 Role**: 自建，附加 `AmazonElasticMapReduceforEC2Role` + `CloudWatchAgentServerPolicy`
- **CW Agent**: 透過 `emr-metrics` classification 啟用 YARN/HDFS/System metrics
- **Spark Config**: 在 `spark-defaults` 和 Step args 雙重設定，確保 oversized request

### CFN 踩坑紀錄

| 問題 | 錯誤訊息 | 解法 |
|------|----------|------|
| EMR Configuration 屬性名稱 | `Encountered unsupported property Properties` | CFN 用 `ConfigurationProperties`，不是 EMR API 的 `Properties` |
| 自建 Service Role 權限不足 | `Service role has insufficient EC2 permissions` | `AmazonEMRServicePolicy_v2` 設計給 service-linked role，自建 role 改用 `EMR_DefaultRole` |
| Managed Policy ARN 缺 partition | ARN 解析失敗 | 用 `!Sub 'arn:${AWS::Partition}:iam::aws:policy/...'` |

## 部署方式

```bash
# 前置：帳號需有 EMR_DefaultRole（執行 aws emr create-default-roles 建立）
aws cloudformation create-stack \
  --stack-name emr-lab-01-yarn-exhaustion \
  --template-body file://cfn/01-yarn-resource-exhaustion.yaml \
  --parameters \
    ParameterKey=SubnetId,ParameterValue=subnet-xxxxxxxx \
    ParameterKey=LogBucket,ParameterValue=your-emr-log-bucket \
  --capabilities CAPABILITY_NAMED_IAM \
  --region us-east-1
```

部署後取得 Cluster ID：
```bash
aws cloudformation describe-stacks \
  --stack-name emr-lab-01-yarn-exhaustion \
  --query 'Stacks[0].Outputs[?OutputKey==`ClusterId`].OutputValue' \
  --output text
```

## CloudWatch Agent 指標（CWAgent namespace）

所有指標在 **CWAgent** namespace，透過 dimension 區分：

### YARN ResourceManager (service.name=hadoop)
| JMX MBean | 指標 | 異常時預期值 |
|-----------|------|-------------|
| QueueMetrics,q0=root | `PendingMB` | 持續很高（~50GB） |
| QueueMetrics,q0=root | `AvailableMB` | 接近 0 |
| QueueMetrics,q0=root | `PendingContainers` | 5（不下降） |
| QueueMetrics,q0=root | `PendingVCores` | > 0 |
| QueueMetrics,q0=root | `AppsPending` | ≥ 1 |
| ClusterMetrics | `NumActiveNMs` | 1 |

### YARN NodeManager (service.name=hadoop)
| JMX MBean | 指標 | 異常時預期值 |
|-----------|------|-------------|
| NodeManagerMetrics | `AvailableGB` | 接近 0 |
| NodeManagerMetrics | `ContainersRunning` | 0（無法啟動） |

### HDFS & System Metrics
- HDFS: `CapacityUsedGB`, `CapacityRemainingGB`, `BlocksTotal`
- CPU: `cpu_usage_active`, `cpu_usage_idle`, `cpu_usage_iowait`
- Memory: `mem_used_percent`, `mem_available_percent`
- Disk: `disk_used_percent` (/, /mnt, /emr)

## 測試 Prompt

> My Spark job on EMR cluster **j-LF4I0S8953P6** has been stuck in ACCEPTED state for 30 minutes. No executors are launching. What's wrong?

## 預期 DevOps Agent 應能

1. 識別 cluster 配置（1 Primary + 1 Core, m5.xlarge）
2. 讀取 YARN metrics 發現 PendingMB 高、AvailableMB 低
3. 計算 requested vs available（50GB vs ~12GB）
4. 建議修復方案（調低 config / scale out / 開 dynamic allocation）

---

## ✅ 測試結果（2026-04-28）

**Cluster ID**: `j-LF4I0S8953P6`
**Stack**: `emr-lab-01-yarn-exhaustion`
**Account**: `104172191111` / `us-east-1`

### DevOps Agent 回答摘要

Agent 正確識別了 root cause：

> Your Spark job is stuck because you're requesting more resources than your cluster can provide.
> - Requested: 5 × 10GB = **50GB memory**, 5 × 4 = **20 vCPUs**
> - Available: ~12GB (single m5.xlarge CORE node)

提供了 3 個解法：
1. **Scale cluster** — `aws emr modify-instance-groups` 擴展到 5 個 Core nodes
2. **Reduce resource requests** — 調低 executor memory/instances
3. **Enable dynamic allocation** — `spark.dynamicAllocation.enabled=true`

### 評分

| 評分項目 | 分數 | 評語 |
|----------|------|------|
| 識別 cluster/step 狀態 | 5/5 | 正確識別 1 Primary + 1 Core 的 m5.xlarge 配置 |
| 讀取 YARN metrics | 5/5 | 發現 5 containers pending、YARN memory ~80% available |
| Root cause 分析 | 5/5 | 精準算出 50GB requested vs ~12GB available |
| 解法建議 | 5/5 | 3 個方案都正確，附可執行的 CLI 命令 |
| Bonus：深入 log 分析 | 4/5 | 沒有主動去 S3 讀 Spark driver log 佐證 |

**總分：24/25 — 優秀 🎉**

### 亮點
- 數學計算精確（5×10GB=50GB vs ~12GB）
- 正確指出 `dynamicAllocation.enabled=false` 是關鍵因素
- 抓到正確的 InstanceGroupId（`ig-2CISNZF2E4RP4`）用於 scale 命令
- 3 個解法都是實務上正確的做法

### 改進空間
- 未主動查看 S3 上的 Spark driver/executor log 來佐證分析

## 清理

```bash
aws cloudformation delete-stack --stack-name emr-lab-01-yarn-exhaustion --region us-east-1
```
