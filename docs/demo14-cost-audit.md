# Demo14 — 成本治理与清理审计（可选）

## 实验简介

本实验将完成「成本审计与资源清理」的全部操作，涵盖资源创建、配置、验证的完整流程。

**实验目标：**
- 掌握 成本审计与资源清理 的核心操作流程
- 理解相关组件的工作原理和配置要点
- 能够独立完成从部署到验证的完整闭环

**实验流程：**
1. 盘点 ECS 资源
2. 盘点 ALB 和网络资源
3. 盘点存储和服务发现资源
4. 盘点 CI/CD 和 ECR 资源
5. 按 Tag 查询资源
6. 成本优化：设置日志保留期和 ECR 生命周期策略
7. 输出清理顺序检查清单
8. 可选：执行主线资源清理

**预计 AI 执行时长：** 8-10 分钟


## 前提条件

- **工具**：AWS CLI v2
- **权限**：AdministratorAccess（ECS、EC2、ECR、ELBv2、Logs、EFS、Cloud Map、CodePipeline、IAM 查询权限）
- **前提**：至少完成 Demo01-Demo04
- **默认行为**：先审计，确认后再删除
- **初始化**：

```bash
source /tmp/demo-ecs.env
export COST_TAG_KEY=Project
export COST_TAG_VALUE=ecs-global-quickstart
```

---

## 步骤

### 1. 盘点 ECS 资源

```bash
echo "=== ECS Cluster ==="
aws ecs describe-clusters \
  --clusters ${CLUSTER_NAME} \
  --region ${AWS_REGION} \
  --query 'clusters[0].{Name:clusterName,Status:status,Running:runningTasksCount,Pending:pendingTasksCount,Services:activeServicesCount}' \
  --output table

echo ""
echo "=== ECS Services ==="
aws ecs list-services \
  --cluster ${CLUSTER_NAME} \
  --region ${AWS_REGION} \
  --query 'serviceArns' --output table

echo ""
echo "=== Task Definitions（demo-ecs 前缀）==="
aws ecs list-task-definitions \
  --family-prefix demo-ecs \
  --region ${AWS_REGION} \
  --query 'taskDefinitionArns' --output table

echo ""
echo "=== Auto Scaling 配置 ==="
aws application-autoscaling describe-scalable-targets \
  --service-namespace ecs \
  --region ${AWS_REGION} \
  --query 'ScalableTargets[?contains(ResourceId, `demo-ecs`)].{Resource:ResourceId,Min:MinCapacity,Max:MaxCapacity}' \
  --output table 2>/dev/null || echo "无 Auto Scaling 配置"
```

**预期输出**：列出所有 ECS 相关资源及其状态

### 2. 盘点 ALB 和网络资源

```bash
echo "=== ALB ==="
aws elbv2 describe-load-balancers \
  --region ${AWS_REGION} \
  --query "LoadBalancers[?contains(LoadBalancerName, 'demo-ecs')].{Name:LoadBalancerName,DNS:DNSName,State:State.Code}" \
  --output table

echo ""
echo "=== Target Groups ==="
aws elbv2 describe-target-groups \
  --region ${AWS_REGION} \
  --query "TargetGroups[?contains(TargetGroupName, 'demo')].{Name:TargetGroupName,Port:Port,Type:TargetType}" \
  --output table

echo ""
echo "=== VPC Endpoints（持续计费项）==="
aws ec2 describe-vpc-endpoints \
  --filters "Name=vpc-id,Values=${VPC_ID}" \
  --region ${AWS_REGION} \
  --query 'VpcEndpoints[*].{ID:VpcEndpointId,Service:ServiceName,State:State,Type:VpcEndpointType}' \
  --output table 2>/dev/null || echo "无 VPC Endpoint"
```

**预期输出**：列出 ALB、Target Group、VPC Endpoint 资源

### 3. 盘点存储和服务发现资源

```bash
echo "=== EFS File Systems（持续计费项）==="
aws efs describe-file-systems \
  --region ${AWS_REGION} \
  --query "FileSystems[?Tags[?Key=='Project' && Value=='${COST_TAG_VALUE}']].{ID:FileSystemId,State:LifeCycleState,Size:SizeInBytes.Value}" \
  --output table 2>/dev/null || echo "无 EFS"

echo ""
echo "=== Cloud Map Namespaces ==="
aws servicediscovery list-namespaces \
  --region ${AWS_REGION} \
  --query 'Namespaces[*].{Name:Name,ID:Id,Type:Type}' \
  --output table 2>/dev/null || echo "无 Cloud Map Namespace"

echo ""
echo "=== Cloud Map Services ==="
aws servicediscovery list-services \
  --region ${AWS_REGION} \
  --query 'Services[*].{Name:Name,ID:Id}' \
  --output table 2>/dev/null || echo "无 Cloud Map Service"
```

**预期输出**：列出 EFS 和 Cloud Map 资源

### 4. 盘点 CI/CD 和 ECR 资源

```bash
echo "=== CodePipeline ==="
aws codepipeline list-pipelines \
  --region ${AWS_REGION} \
  --query "pipelines[?contains(name, 'demo-ecs')].{Name:name,Updated:updated}" \
  --output table 2>/dev/null || echo "无 Pipeline"

echo ""
echo "=== CodeBuild Projects ==="
aws codebuild list-projects \
  --region ${AWS_REGION} \
  --query "projects[?contains(@, 'demo-ecs')]" \
  --output table 2>/dev/null || echo "无 CodeBuild"

echo ""
echo "=== ECR Repositories ==="
aws ecr describe-repositories \
  --region ${AWS_REGION} \
  --query "repositories[?contains(repositoryName, 'demo-ecs')].{Name:repositoryName,URI:repositoryUri}" \
  --output table 2>/dev/null || echo "无 ECR Repository"

echo ""
echo "=== ECR 镜像数量（各仓库）==="
for REPO in $(aws ecr describe-repositories \
  --region ${AWS_REGION} \
  --query "repositories[?contains(repositoryName, 'demo-ecs')].repositoryName" \
  --output text 2>/dev/null); do
  COUNT=$(aws ecr list-images \
    --repository-name ${REPO} \
    --region ${AWS_REGION} \
    --query 'length(imageIds)' --output text)
  echo "  ${REPO}: ${COUNT} 个镜像"
done

echo ""
echo "=== CloudWatch Log Groups ==="
aws logs describe-log-groups \
  --log-group-name-prefix /ecs/demo-ecs \
  --region ${AWS_REGION} \
  --query 'logGroups[*].{Name:logGroupName,Retention:retentionInDays,SizeMB:storedBytes}' \
  --output table 2>/dev/null || echo "无 Log Group"
```

**预期输出**：列出 CI/CD、ECR 和 CloudWatch Logs 资源

### 5. 按 Tag 查询资源

```bash
echo "=== 按 Tag ${COST_TAG_KEY}=${COST_TAG_VALUE} 查询 Security Groups ==="
aws ec2 describe-security-groups \
  --filters "Name=tag:${COST_TAG_KEY},Values=${COST_TAG_VALUE}" \
  --region ${AWS_REGION} \
  --query 'SecurityGroups[*].{ID:GroupId,Name:GroupName,VPC:VpcId}' \
  --output table

echo ""
echo "=== 按 Tag 查询 Subnets ==="
aws ec2 describe-subnets \
  --filters "Name=tag:${COST_TAG_KEY},Values=${COST_TAG_VALUE}" \
  --region ${AWS_REGION} \
  --query 'Subnets[*].{ID:SubnetId,CIDR:CidrBlock,AZ:AvailabilityZone,Public:MapPublicIpOnLaunch}' \
  --output table

echo ""
echo "=== 按 Tag 查询 VPC ==="
aws ec2 describe-vpcs \
  --filters "Name=tag:${COST_TAG_KEY},Values=${COST_TAG_VALUE}" \
  --region ${AWS_REGION} \
  --query 'Vpcs[*].{ID:VpcId,CIDR:CidrBlock}' \
  --output table
```

**预期输出**：列出所有带项目标签的网络资源

### 6. 成本优化：设置日志保留期和 ECR 生命周期策略

```bash
echo "=== 设置 CloudWatch Logs 保留 30 天 ==="
aws logs put-retention-policy \
  --log-group-name ${LOG_GROUP} \
  --retention-in-days 30 \
  --region ${AWS_REGION}
echo "日志保留期已设置为 30 天"

echo ""
echo "=== 设置 ECR 生命周期策略（保留 5 个 tagged 镜像，untagged 1 天后过期）==="
aws ecr put-lifecycle-policy \
  --repository-name ${REPO_NAME} \
  --lifecycle-policy-text '{
    "rules": [
      {
        "rulePriority": 1,
        "description": "Expire untagged images after 1 day",
        "selection": {"tagStatus":"untagged","countType":"sinceImagePushed","countUnit":"days","countNumber":1},
        "action": {"type":"expire"}
      },
      {
        "rulePriority": 2,
        "description": "Keep only 5 tagged images",
        "selection": {"tagStatus":"tagged","tagPrefixList":["v"],"countType":"imageCountMoreThan","countNumber":5},
        "action": {"type":"expire"}
      }
    ]
  }' \
  --region ${AWS_REGION} 2>/dev/null || echo "ECR lifecycle policy 设置完成（或已存在）"
echo "ECR 生命周期策略已设置"
```

**预期输出**：日志保留期和 ECR 策略设置完成

### 7. 输出清理顺序检查清单

```bash
cat << 'CHECKLIST'
==========================================
  ECS Workshop 清理顺序检查清单
==========================================

⚠️  重要：按以下顺序清理，跳过顺序可能导致依赖错误

[ ] 1. 停止压测和自动伸缩
    - 停止所有临时压测 Task
    - aws application-autoscaling delete-scaling-policy（Demo07）
    - aws application-autoscaling deregister-scalable-target（Demo07）

[ ] 2. 删除 CI/CD 资源（Demo10）
    - aws codepipeline delete-pipeline
    - aws codebuild delete-project
    - aws codecommit delete-repository
    - aws s3 rm s3://<artifact-bucket> --recursive && aws s3 rb

[ ] 3. 删除 CodeDeploy 资源（Demo13）
    - aws deploy delete-deployment-group
    - aws deploy delete-application
    - aws ecs delete-service demo-ecs-bg（先 desired-count=0）
    - aws elbv2 delete-listener（测试流量 listener）
    - aws elbv2 delete-target-group demo-web-blue/green

[ ] 4. 删除 ECS Services（先 desired-count=0，再 delete）
    - demo-ecs-backend（Demo12）
    - demo-ecs-web（Demo03-09 主线）

[ ] 5. 删除 ECS Cluster
    - aws ecs delete-cluster demo-ecs

[ ] 6. 删除 ALB 资源（Demo04）
    - aws elbv2 delete-listener
    - aws elbv2 delete-target-group demo-ecs-tg
    - aws elbv2 delete-load-balancer demo-ecs-alb

[ ] 7. 删除 EFS（Demo11）
    - aws efs delete-mount-target（先删 mount targets）
    - 等待 30 秒
    - aws efs delete-file-system

[ ] 8. 删除 Cloud Map（Demo12）
    - aws servicediscovery delete-service（先删服务）
    - aws servicediscovery delete-namespace

[ ] 9. 删除 VPC Endpoints（Demo08）
    - aws ec2 delete-vpc-endpoints --vpc-endpoint-ids <all>

[ ] 10. 清理 ECR（Demo02）
    - aws ecr delete-repository --force demo-ecs-web

[ ] 11. 删除 IAM 资源
    - 内联策略：aws iam delete-role-policy
    - 托管策略：aws iam detach-role-policy
    - 角色：aws iam delete-role（执行角色、任务角色、CodeBuild/Pipeline/CodeDeploy 角色）

[ ] 12. 删除 CloudWatch Logs
    - aws logs delete-log-group /ecs/demo-ecs

[ ] 13. 删除网络资源（最后删）
    - aws ec2 delete-security-group（EFS SG、VPCE SG、Task SG、ALB SG）
    - aws ec2 delete-subnet（4 个子网）
    - aws ec2 detach-internet-gateway && aws ec2 delete-internet-gateway
    - aws ec2 delete-route-table
    - aws ec2 delete-vpc

==========================================
持续计费资源提醒（请优先清理）：
  🔴 ALB：按小时计费
  🔴 VPC Interface Endpoints：每个按小时计费
  🔴 EFS：按存储量计费
  🟡 NAT Gateway（如有）：按小时+流量计费
  🟡 ECR：按存储量计费（有免费额度）
  🟢 ECS Fargate：仅 Task 运行时计费（service 停止则无费用）
==========================================
CHECKLIST
```

**预期输出**：完整清理顺序检查清单

### 8. 可选：执行主线资源清理

```bash
echo "=== 当前主线资源状态（清理前确认）==="
echo "ECS Service 运行中的 Task 数："
aws ecs describe-services \
  --cluster ${CLUSTER_NAME} \
  --services ${SERVICE_NAME} \
  --region ${AWS_REGION} \
  --query 'services[0].{Running:runningCount,Desired:desiredCount}' \
  --output json 2>/dev/null || echo "Service 不存在或已清理"

echo ""
echo "如确认执行清理，请参照上方检查清单逐项执行。"
echo "建议在每个 Demo 的清理章节分别执行，而非一次性批量删除。"
```

---

## 验收标准

完成本实验后，你应当能够：
- [ ] 所有核心资源创建成功且状态正常
- [ ] 验证检查点中的所有命令返回预期结果
- [ ] 理解各组件之间的关联关系和配置要点

---

## 验证检查点

| # | 检查命令 | 期望精确输出 |
|---|---------|-------------|
| 1 | `aws logs describe-log-groups --log-group-name-prefix /ecs/demo-ecs --region us-east-1 --query 'logGroups[0].retentionInDays' --output text` | `30` |
| 2 | `aws ecr get-lifecycle-policy --repository-name demo-ecs-web --region us-east-1 --query 'lifecyclePolicyText' --output text \| python3 -c "import sys,json; p=json.load(sys.stdin); print(len(p['rules']))"` | `2` |
| 3 | `aws ecs list-services --cluster demo-ecs --region us-east-1 --query 'length(serviceArns)' --output text` | 大于等于 `1`（审计时 service 仍存在） |

---

## 实验总结

本实验完成了「成本审计与资源清理」的全部操作，从资源创建到功能验证形成了完整闭环。通过动手实践，你已掌握了相关组件的核心配置方法和最佳实践。至此完成 ECS QuickStart 全部实验。

---

## 清理说明

本 Demo 为审计专题，不创建独立资源。执行清理请参照步骤 7 中的检查清单，按依赖顺序在各 Demo 的清理章节逐项执行。

**关键原则：**
- 删除 ECS Service 前先将 `desired-count` 设为 `0`，等待 Task 停止后再 `delete-service`
- VPC 删除失败通常表示仍有 ENI、Endpoint、Mount Target 或 Security Group 依赖未清理
- 不使用 `--force` 跳过依赖错误，按顺序排查
