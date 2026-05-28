# Demo09 — CloudWatch 日志指标与 ECS Exec

## 实验简介

本实验将完成「CloudWatch 日志与 ECS Exec」的全部操作，涵盖资源创建、配置、验证的完整流程。

**实验目标：**
- 掌握 CloudWatch 日志与 ECS Exec 的核心操作流程
- 理解相关组件的工作原理和配置要点
- 能够独立完成从部署到验证的完整闭环

**实验流程：**
1. 查看 CloudWatch 日志
2. 查看 CloudWatch 指标
3. 为 Task Role 添加 ECS Exec 权限
4. 启用 ECS Exec 并强制新部署
5. 进入容器执行命令

**预计 AI 执行时长：** 5-8 分钟


## 前提条件

- **工具**：AWS CLI v2、Session Manager plugin（`session-manager-plugin`）
- **权限**：AdministratorAccess（含 ECS、SSM Messages、CloudWatch Logs、CloudWatch Metrics、IAM）
- **前提**：Demo03 以后任一 service 已运行；如在私有子网则需要 Demo08 的 SSM Endpoint
- **初始化**：

```bash
source /tmp/demo-ecs.env
```

---

## 步骤

### 1. 查看 CloudWatch 日志

```bash
echo "=== 应用日志（最近 5 分钟）==="
aws logs tail ${LOG_GROUP} --since 5m --region ${AWS_REGION} 2>/dev/null | head -20 || \
  echo "日志尚未生成或 log group 不存在"

echo "=== Log Streams ==="
aws logs describe-log-streams \
  --log-group-name ${LOG_GROUP} \
  --order-by LastEventTime \
  --descending \
  --limit 5 \
  --region ${AWS_REGION} \
  --query 'logStreams[*].{Stream:logStreamName,Last:lastEventTimestamp}' \
  --output table 2>/dev/null || true
```

**预期输出**：显示应用容器日志；列出最近的 log stream。

### 2. 查看 CloudWatch 指标

```bash
END_TIME=$(date --utc +%s)
START_TIME=$((END_TIME - 600))

echo "=== ECS Service CPU 指标（最近 10 分钟）==="
aws cloudwatch get-metric-statistics \
  --namespace AWS/ECS \
  --metric-name CPUUtilization \
  --dimensions Name=ClusterName,Value=${CLUSTER_NAME} Name=ServiceName,Value=${SERVICE_NAME} \
  --start-time ${START_TIME} \
  --end-time ${END_TIME} \
  --period 300 \
  --statistics Average \
  --region ${AWS_REGION} \
  --query 'Datapoints[*].{Time:Timestamp,CPU:Average}' \
  --output table 2>/dev/null || echo "指标数据积累中..."

echo "=== ECS Service Memory 指标 ==="
aws cloudwatch get-metric-statistics \
  --namespace AWS/ECS \
  --metric-name MemoryUtilization \
  --dimensions Name=ClusterName,Value=${CLUSTER_NAME} Name=ServiceName,Value=${SERVICE_NAME} \
  --start-time ${START_TIME} \
  --end-time ${END_TIME} \
  --period 300 \
  --statistics Average \
  --region ${AWS_REGION} \
  --query 'Datapoints[*].{Time:Timestamp,Mem:Average}' \
  --output table 2>/dev/null || echo "指标数据积累中..."
```

**预期输出**：显示 CPU/Memory 指标数据点（可能需等待几分钟生效）

### 3. 为 Task Role 添加 ECS Exec 权限

```bash
cat > /tmp/exec-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": [
      "ssmmessages:CreateControlChannel",
      "ssmmessages:CreateDataChannel",
      "ssmmessages:OpenControlChannel",
      "ssmmessages:OpenDataChannel"
    ],
    "Resource": "*"
  }]
}
EOF

aws iam put-role-policy \
  --role-name demo-ecs-task-role \
  --policy-name demo-ecs-exec-policy \
  --policy-document file:///tmp/exec-policy.json

echo "ECS Exec 权限已添加到 Task Role"
```

**预期输出**：打印"ECS Exec 权限已添加到 Task Role"

### 4. 启用 ECS Exec 并强制新部署

```bash
aws ecs update-service \
  --cluster ${CLUSTER_NAME} \
  --service ${SERVICE_NAME} \
  --enable-execute-command \
  --force-new-deployment \
  --region ${AWS_REGION}

aws ecs wait services-stable \
  --cluster ${CLUSTER_NAME} \
  --services ${SERVICE_NAME} \
  --region ${AWS_REGION}

echo "ECS Exec 已启用，新 Task 已部署"
```

**预期输出**：打印"ECS Exec 已启用，新 Task 已部署"

### 5. 进入容器执行命令

```bash
TASK_ARN=$(aws ecs list-tasks \
  --cluster ${CLUSTER_NAME} \
  --service-name ${SERVICE_NAME} \
  --region ${AWS_REGION} \
  --query 'taskArns[0]' --output text)

echo "=== 目标 Task: ${TASK_ARN} ==="

echo "=== 验证 execute-command 状态 ==="
aws ecs describe-tasks \
  --cluster ${CLUSTER_NAME} \
  --tasks ${TASK_ARN} \
  --region ${AWS_REGION} \
  --query 'tasks[0].containers[0].managedAgents[?name==`ExecuteCommandAgent`].lastStatus' \
  --output text

echo ""
echo "=== 进入容器（执行 ECS Exec）==="
aws ecs execute-command \
  --cluster ${CLUSTER_NAME} \
  --task ${TASK_ARN} \
  --container ${CONTAINER_NAME} \
  --interactive \
  --command "/bin/sh -c 'echo === Container Info ===; hostname; id; env | grep -v SECRET; echo === Process ===; ps aux; echo === Done ===; exit 0'" \
  --region ${AWS_REGION}
```

**预期输出**：显示容器内主机名、进程和环境变量（Secret 值不输出）

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
| 1 | `aws ecs describe-services --cluster demo-ecs --services demo-ecs-web --region us-east-1 --query 'services[0].enableExecuteCommand' --output text` | `True` |
| 2 | `aws iam get-role-policy --role-name demo-ecs-task-role --policy-name demo-ecs-exec-policy --query 'PolicyName' --output text` | `demo-ecs-exec-policy` |
| 3 | `aws logs describe-log-groups --log-group-name-prefix /ecs/demo-ecs --region us-east-1 --query 'length(logGroups)' --output text` | `1` |
| 4 | `aws ecs describe-services --cluster demo-ecs --services demo-ecs-web --region us-east-1 --query 'services[0].runningCount' --output text` | `2` |

---

## 实验总结

本实验完成了「CloudWatch 日志与 ECS Exec」的全部操作，从资源创建到功能验证形成了完整闭环。通过动手实践，你已掌握了相关组件的核心配置方法和最佳实践。Demo10 将学习 CodePipeline CI/CD。

---

## 清理

```bash
source /tmp/demo-ecs.env

aws ecs update-service \
  --cluster ${CLUSTER_NAME} \
  --service ${SERVICE_NAME} \
  --no-enable-execute-command \
  --region ${AWS_REGION} 2>/dev/null || true

aws iam delete-role-policy \
  --role-name demo-ecs-task-role \
  --policy-name demo-ecs-exec-policy 2>/dev/null || true

echo "ECS Exec 权限已移除"
```
