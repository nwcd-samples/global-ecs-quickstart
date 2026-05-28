# Demo04 — 使用 ALB 暴露服务

## 实验简介

本实验将完成「通过 ALB 暴露服务」的全部操作，涵盖资源创建、配置、验证的完整流程。

**实验目标：**
- 掌握 通过 ALB 暴露服务 的核心操作流程
- 理解相关组件的工作原理和配置要点
- 能够独立完成从部署到验证的完整闭环

**实验流程：**
1. 创建 ALB
2. 创建 Target Group
3. 创建 Listener
4. 将 ECS Service 接入 ALB
5. 等待 Target 健康
6. 访问应用
7. 演练健康检查失败与恢复
8. 保存 ALB 变量

**预计 AI 执行时长：** 5-8 分钟


## 前提条件

- **工具**：AWS CLI v2、curl
- **权限**：AdministratorAccess（含 ELBv2、ECS、EC2、IAM PassRole）
- **前提**：Demo03 已完成，`demo-ecs-web` service 正在运行
- **初始化**：

```bash
source /tmp/demo-ecs.env
```

---

## 步骤

### 1. 创建 ALB

```bash
ALB_ARN=$(aws elbv2 create-load-balancer \
  --name demo-ecs-alb \
  --subnets ${PUBLIC_SUBNET_1} ${PUBLIC_SUBNET_2} \
  --security-groups ${ALB_SG_ID} \
  --scheme internet-facing \
  --type application \
  --ip-address-type ipv4 \
  --tags Key=Project,Value=${PROJECT_TAG} Key=Demo,Value=Demo04 Key=Owner,Value=${OWNER_TAG} \
  --region ${AWS_REGION} \
  --query 'LoadBalancers[0].LoadBalancerArn' --output text)

ALB_DNS=$(aws elbv2 describe-load-balancers \
  --load-balancer-arns ${ALB_ARN} \
  --region ${AWS_REGION} \
  --query 'LoadBalancers[0].DNSName' --output text)

echo "ALB ARN: ${ALB_ARN}"
echo "ALB DNS: ${ALB_DNS}"
```

**预期输出**：打印 ALB ARN 和 DNS 名称

### 2. 创建 Target Group

```bash
TG_ARN=$(aws elbv2 create-target-group \
  --name demo-ecs-tg \
  --protocol HTTP \
  --port 80 \
  --target-type ip \
  --vpc-id ${VPC_ID} \
  --health-check-protocol HTTP \
  --health-check-path / \
  --health-check-interval-seconds 15 \
  --health-check-timeout-seconds 5 \
  --healthy-threshold-count 2 \
  --unhealthy-threshold-count 3 \
  --tags Key=Project,Value=${PROJECT_TAG} Key=Demo,Value=Demo04 \
  --region ${AWS_REGION} \
  --query 'TargetGroups[0].TargetGroupArn' --output text)

echo "Target Group ARN: ${TG_ARN}"
```

**预期输出**：打印 Target Group ARN

### 3. 创建 Listener

```bash
LISTENER_ARN=$(aws elbv2 create-listener \
  --load-balancer-arn ${ALB_ARN} \
  --protocol HTTP \
  --port 8080 \
  --default-actions Type=forward,TargetGroupArn=${TG_ARN} \
  --tags Key=Project,Value=${PROJECT_TAG} \
  --region ${AWS_REGION} \
  --query 'Listeners[0].ListenerArn' --output text)

echo "Listener ARN: ${LISTENER_ARN}"
```

**预期输出**：打印 Listener ARN

### 4. 将 ECS Service 接入 ALB

```bash
aws ecs update-service \
  --cluster ${CLUSTER_NAME} \
  --service ${SERVICE_NAME} \
  --load-balancers "targetGroupArn=${TG_ARN},containerName=${CONTAINER_NAME},containerPort=80" \
  --region ${AWS_REGION}

aws ecs wait services-stable \
  --cluster ${CLUSTER_NAME} \
  --services ${SERVICE_NAME} \
  --region ${AWS_REGION}

echo "ECS Service 已接入 ALB"
```

**预期输出**：打印"ECS Service 已接入 ALB"

### 5. 等待 Target 健康

```bash
echo "等待 target 健康（约 1-2 分钟）..."
for i in $(seq 1 20); do
  HEALTHY=$(aws elbv2 describe-target-health \
    --target-group-arn ${TG_ARN} \
    --region ${AWS_REGION} \
    --query 'TargetHealthDescriptions[?TargetHealth.State==`healthy`] | length(@)' \
    --output text)
  echo "健康 target 数: ${HEALTHY}"
  [ "${HEALTHY}" -ge 2 ] 2>/dev/null && break
  sleep 15
done
```

**预期输出**：健康 target 数达到 2。

### 6. 访问应用

```bash
echo "等待 ALB DNS 传播..."
sleep 10

curl -s -o /dev/null -w "HTTP Status: %{http_code}\n" --max-time 10 http://${ALB_DNS}:8080/
curl -s --max-time 10 http://${ALB_DNS}:8080/ | grep -o 'demo-ecs-web v[0-9]'
```

**预期输出**：`HTTP Status: 200`；显示 `demo-ecs-web v1`

### 7. 演练健康检查失败与恢复

```bash
echo "=== 改错 health check path（触发 unhealthy）==="
aws elbv2 modify-target-group \
  --target-group-arn ${TG_ARN} \
  --health-check-path /notexist \
  --region ${AWS_REGION}

sleep 60
aws elbv2 describe-target-health \
  --target-group-arn ${TG_ARN} \
  --region ${AWS_REGION} \
  --query 'TargetHealthDescriptions[*].TargetHealth.State' \
  --output text

echo "=== 恢复 health check path ==="
aws elbv2 modify-target-group \
  --target-group-arn ${TG_ARN} \
  --health-check-path / \
  --region ${AWS_REGION}

sleep 30
aws elbv2 describe-target-health \
  --target-group-arn ${TG_ARN} \
  --region ${AWS_REGION} \
  --query 'TargetHealthDescriptions[*].TargetHealth.State' \
  --output text
```

**预期输出**：改错后状态变为 `unhealthy`；恢复后重回 `healthy`

### 8. 保存 ALB 变量

```bash
cat >> /tmp/demo-ecs.env << EOF
export ALB_ARN=${ALB_ARN}
export ALB_DNS=${ALB_DNS}
export TG_ARN=${TG_ARN}
export LISTENER_ARN=${LISTENER_ARN}
EOF

echo "ALB 变量已追加到 /tmp/demo-ecs.env"
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
| 1 | `aws elbv2 describe-load-balancers --names demo-ecs-alb --region us-east-1 --query 'LoadBalancers[0].State.Code' --output text` | `active` |
| 2 | `aws elbv2 describe-target-health --target-group-arn $(aws elbv2 describe-target-groups --names demo-ecs-tg --region us-east-1 --query 'TargetGroups[0].TargetGroupArn' --output text) --region us-east-1 --query 'TargetHealthDescriptions[?TargetHealth.State==\`healthy\`] | length(@)' --output text` | `2` |
| 3 | `curl -s -o /dev/null -w "%{http_code}" --max-time 10 http://$(aws elbv2 describe-load-balancers --names demo-ecs-alb --region us-east-1 --query 'LoadBalancers[0].DNSName' --output text):8080/` | `200` |
| 4 | `aws ecs describe-services --cluster demo-ecs --services demo-ecs-web --region us-east-1 --query 'services[0].runningCount' --output text` | `2` |

---

## 实验总结

本实验完成了「通过 ALB 暴露服务」的全部操作，从资源创建到功能验证形成了完整闭环。通过动手实践，你已掌握了相关组件的核心配置方法和最佳实践。Demo05 将学习滚动部署与回滚。

---

## 清理

```bash
source /tmp/demo-ecs.env

aws ecs update-service \
  --cluster ${CLUSTER_NAME} \
  --service ${SERVICE_NAME} \
  --desired-count 0 \
  --region ${AWS_REGION} 2>/dev/null || true

aws ecs wait services-stable \
  --cluster ${CLUSTER_NAME} \
  --services ${SERVICE_NAME} \
  --region ${AWS_REGION} 2>/dev/null || true

aws ecs delete-service \
  --cluster ${CLUSTER_NAME} \
  --service ${SERVICE_NAME} \
  --region ${AWS_REGION} 2>/dev/null || true

aws elbv2 delete-listener --listener-arn ${LISTENER_ARN} --region ${AWS_REGION} 2>/dev/null || true
aws elbv2 delete-target-group --target-group-arn ${TG_ARN} --region ${AWS_REGION} 2>/dev/null || true
aws elbv2 delete-load-balancer --load-balancer-arn ${ALB_ARN} --region ${AWS_REGION} 2>/dev/null || true

echo "清理完成"
```
