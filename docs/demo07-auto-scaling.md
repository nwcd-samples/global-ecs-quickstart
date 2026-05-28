# Demo07 — Service Auto Scaling

## 实验简介

本实验将完成「Service Auto Scaling」的全部操作，涵盖资源创建、配置、验证的完整流程。

**实验目标：**
- 掌握 Service Auto Scaling 的核心操作流程
- 理解相关组件的工作原理和配置要点
- 能够独立完成从部署到验证的完整闭环

**实验流程：**
1. 注册 ECS Service 为 Scalable Target
2. 创建 CPU Target Tracking 策略
3. 查看当前 Auto Scaling 配置
4. 触发压测（制造 CPU 负载）
5. 观察扩容
6. 停止压测并观察缩容
7. 可选：Fargate Spot Capacity Provider

**预计 AI 执行时长：** 8-10 分钟


## 前提条件

- **工具**：AWS CLI v2
- **权限**：AdministratorAccess（含 ECS、Application Auto Scaling、CloudWatch、IAM）
- **前提**：Demo04 已完成，`demo-ecs-web` service 通过 ALB 对外暴露
- **初始化**：

```bash
source /tmp/demo-ecs.env
export RESOURCE_ID=service/${CLUSTER_NAME}/${SERVICE_NAME}
```

---

## 步骤

### 1. 注册 ECS Service 为 Scalable Target

```bash
aws application-autoscaling register-scalable-target \
  --service-namespace ecs \
  --resource-id ${RESOURCE_ID} \
  --scalable-dimension ecs:service:DesiredCount \
  --min-capacity 1 \
  --max-capacity 4 \
  --region ${AWS_REGION}

echo "Scalable target 已注册"
```

**预期输出**：打印"Scalable target 已注册"，无报错

### 2. 创建 CPU Target Tracking 策略

```bash
aws application-autoscaling put-scaling-policy \
  --service-namespace ecs \
  --resource-id ${RESOURCE_ID} \
  --scalable-dimension ecs:service:DesiredCount \
  --policy-name demo-web-cpu-50 \
  --policy-type TargetTrackingScaling \
  --target-tracking-scaling-policy-configuration '{
    "TargetValue": 50.0,
    "PredefinedMetricSpecification": {
      "PredefinedMetricType": "ECSServiceAverageCPUUtilization"
    },
    "ScaleInCooldown": 60,
    "ScaleOutCooldown": 60
  }' \
  --region ${AWS_REGION}

echo "CPU Target Tracking 策略已创建（目标 CPU 50%）"
```

**预期输出**：打印"CPU Target Tracking 策略已创建"

### 3. 查看当前 Auto Scaling 配置

```bash
echo "=== Scalable Targets ==="
aws application-autoscaling describe-scalable-targets \
  --service-namespace ecs \
  --resource-ids ${RESOURCE_ID} \
  --region ${AWS_REGION} \
  --query 'ScalableTargets[0].{Resource:ResourceId,Min:MinCapacity,Max:MaxCapacity}' \
  --output table

echo "=== Scaling Policies ==="
aws application-autoscaling describe-scaling-policies \
  --service-namespace ecs \
  --resource-id ${RESOURCE_ID} \
  --region ${AWS_REGION} \
  --query 'ScalingPolicies[*].{Name:PolicyName,Type:PolicyType}' \
  --output table
```

**预期输出**：显示 scalable target（min=1, max=4）和 scaling policy。

### 4. 触发压测（制造 CPU 负载）

> ⚠️ nginx 静态文件服务 CPU 消耗极低（通常 < 0.2%），无法通过 HTTP 请求压测触发 50% CPU 阈值。演示扩容有两种方式：
> 1. 使用 CPU 密集型应用（如 stress-ng）替代 nginx
> 2. 临时将 Auto Scaling 阈值降低（如 0.05%）来触发扩容演示，演示完成后恢复

```bash
echo "=== 启动压测 Task（对 ALB 发起连续请求）==="
STRESS_TASK=$(aws ecs run-task \
  --cluster ${CLUSTER_NAME} \
  --task-definition ${TASK_FAMILY} \
  --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={subnets=[${PUBLIC_SUBNET_1}],securityGroups=[${TASK_SG_ID}],assignPublicIp=ENABLED}" \
  --overrides '{
    "containerOverrides": [{
      "name": "'"${CONTAINER_NAME}"'",
      "command": [
        "sh", "-c",
        "apk add curl -q 2>/dev/null; while true; do curl -s http://'"${ALB_DNS}"':8080/ > /dev/null; done"
      ]
    }]
  }' \
  --region ${AWS_REGION} \
  --query 'tasks[0].taskArn' --output text 2>/dev/null || \
  echo "压测 task 启动失败，请手动从操作机发起 curl 循环")

echo "压测 Task ARN: ${STRESS_TASK}"
echo "等待 Auto Scaling 触发（约 2-3 分钟）..."
```

**预期输出**：压测 Task 启动，等待 Auto Scaling 触发扩容。

### 5. 观察扩容

```bash
for i in $(seq 1 10); do
  COUNT=$(aws ecs describe-services \
    --cluster ${CLUSTER_NAME} \
    --services ${SERVICE_NAME} \
    --region ${AWS_REGION} \
    --query 'services[0].{Running:runningCount,Desired:desiredCount}' \
    --output json)
  echo "$(date +%H:%M:%S) Service 状态: ${COUNT}"
  sleep 30
done
```

**预期输出**：desiredCount 从 2 增加到 3 或 4（取决于 CPU 负载）。

### 6. 停止压测并观察缩容

```bash
if [ -n "${STRESS_TASK}" ] && [ "${STRESS_TASK}" != "压测 task 启动失败，请手动从操作机发起 curl 循环" ]; then
  aws ecs stop-task \
    --cluster ${CLUSTER_NAME} \
    --task ${STRESS_TASK} \
    --region ${AWS_REGION}
  echo "压测 Task 已停止"
fi

echo "等待缩容冷却（约 2-3 分钟）..."
for i in $(seq 1 8); do
  COUNT=$(aws ecs describe-services \
    --cluster ${CLUSTER_NAME} \
    --services ${SERVICE_NAME} \
    --region ${AWS_REGION} \
    --query 'services[0].{Running:runningCount,Desired:desiredCount}' \
    --output json)
  echo "$(date +%H:%M:%S) Service 状态: ${COUNT}"
  sleep 30
done
```

**预期输出**：desiredCount 在冷却期后回落到 1 或 2。

### 7. 可选：Fargate Spot Capacity Provider

```bash
echo "=== 可选：演示 Fargate Spot Capacity Provider Strategy ==="
echo "以下配置仅说明，不实际执行（需要先在 Cluster 启用 Fargate Spot）"
cat << 'SCRIPT'
aws ecs update-service \
  --cluster demo-ecs \
  --service demo-ecs-web \
  --capacity-provider-strategy '[
    {"capacityProvider": "FARGATE", "weight": 1, "base": 1},
    {"capacityProvider": "FARGATE_SPOT", "weight": 2}
  ]'
SCRIPT
echo "说明：Fargate Spot 适合无状态可重试 workload，不适合对中断敏感的关键服务"
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
| 1 | `aws application-autoscaling describe-scalable-targets --service-namespace ecs --resource-ids service/demo-ecs/demo-ecs-web --region us-east-1 --query 'ScalableTargets[0].MaxCapacity' --output text` | `4` |
| 2 | `aws application-autoscaling describe-scaling-policies --service-namespace ecs --resource-id service/demo-ecs/demo-ecs-web --region us-east-1 --query 'ScalingPolicies[0].PolicyName' --output text` | `demo-web-cpu-50` |
| 3 | `aws ecs describe-services --cluster demo-ecs --services demo-ecs-web --region us-east-1 --query 'services[0].runningCount' --output text` | 大于等于 `1` |

---

## 实验总结

本实验完成了「Service Auto Scaling」的全部操作，从资源创建到功能验证形成了完整闭环。通过动手实践，你已掌握了相关组件的核心配置方法和最佳实践。Demo08 将学习私有子网与 VPC Endpoint。

---

## 清理

```bash
source /tmp/demo-ecs.env

aws application-autoscaling delete-scaling-policy \
  --service-namespace ecs \
  --resource-id ${RESOURCE_ID} \
  --scalable-dimension ecs:service:DesiredCount \
  --policy-name demo-web-cpu-50 \
  --region ${AWS_REGION} 2>/dev/null || true

aws application-autoscaling deregister-scalable-target \
  --service-namespace ecs \
  --resource-id ${RESOURCE_ID} \
  --scalable-dimension ecs:service:DesiredCount \
  --region ${AWS_REGION} 2>/dev/null || true

echo "Auto Scaling 策略和注册已清理"
```
