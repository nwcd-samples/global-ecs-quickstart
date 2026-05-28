# Demo05 — 滚动发布、回滚与故障排查

## 实验简介

本实验将完成「滚动部署与回滚」的全部操作，涵盖资源创建、配置、验证的完整流程。

**实验目标：**
- 掌握 滚动部署与回滚 的核心操作流程
- 理解相关组件的工作原理和配置要点
- 能够独立完成从部署到验证的完整闭环

**实验流程：**
1. 注册 v2 Task Definition
2. 发布 v2（Rolling Deployment）
3. 观察 Service Events 和 Target Health
4. 注册故障版本（演示排障）
5. 排查故障
6. 回滚到 v2

**预计 AI 执行时长：** 5-8 分钟


## 前提条件

- **工具**：AWS CLI v2、curl
- **权限**：AdministratorAccess（含 ECS、ECR、ELBv2、CloudWatch Logs、IAM PassRole）
- **前提**：Demo04 已完成，ALB target 当前 healthy
- **初始化**：

```bash
source /tmp/demo-ecs.env
```

---

## 步骤

### 1. 注册 v2 Task Definition

```bash
TASK_DEF_V2=$(aws ecs register-task-definition \
  --family ${TASK_FAMILY} \
  --network-mode awsvpc \
  --requires-compatibilities FARGATE \
  --cpu 256 \
  --memory 512 \
  --execution-role-arn ${EXECUTION_ROLE_ARN} \
  --task-role-arn ${TASK_ROLE_ARN} \
  --container-definitions "[
    {
      \"name\": \"${CONTAINER_NAME}\",
      \"image\": \"${IMAGE_URI_V2}\",
      \"portMappings\": [{\"containerPort\": 80, \"protocol\": \"tcp\"}],
      \"essential\": true,
      \"logConfiguration\": {
        \"logDriver\": \"awslogs\",
        \"options\": {
          \"awslogs-group\": \"${LOG_GROUP}\",
          \"awslogs-region\": \"${AWS_REGION}\",
          \"awslogs-stream-prefix\": \"ecs\"
        }
      }
    }
  ]" \
  --region ${AWS_REGION} \
  --query taskDefinition.taskDefinitionArn --output text)

TASK_DEF_V2_REV=$(echo ${TASK_DEF_V2} | awk -F: '{print $NF}')
echo "v2 Task Definition Revision: ${TASK_DEF_V2_REV}"
```

**预期输出**：打印 v2 revision 号

### 2. 发布 v2（Rolling Deployment）

```bash
echo "=== 发布前 ALB 返回版本 ==="
curl -s --max-time 5 http://${ALB_DNS}:8080/ | grep -o 'demo-ecs-web v[0-9]' || echo "v1"

aws ecs update-service \
  --cluster ${CLUSTER_NAME} \
  --service ${SERVICE_NAME} \
  --task-definition ${TASK_FAMILY}:${TASK_DEF_V2_REV} \
  --region ${AWS_REGION}

echo "等待滚动发布完成..."
aws ecs wait services-stable \
  --cluster ${CLUSTER_NAME} \
  --services ${SERVICE_NAME} \
  --region ${AWS_REGION}

echo "=== 发布后 ALB 返回版本 ==="
curl -s --max-time 10 http://${ALB_DNS}:8080/ | grep -o 'demo-ecs-web v[0-9]'
```

**预期输出**：发布后 ALB 返回 `demo-ecs-web v2`

### 3. 观察 Service Events 和 Target Health

```bash
echo "=== Service Events（最近 5 条）==="
aws ecs describe-services \
  --cluster ${CLUSTER_NAME} \
  --services ${SERVICE_NAME} \
  --region ${AWS_REGION} \
  --query 'services[0].events[:5].message' \
  --output table

echo "=== Target Health ==="
aws elbv2 describe-target-health \
  --target-group-arn ${TG_ARN} \
  --region ${AWS_REGION} \
  --query 'TargetHealthDescriptions[*].{IP:Target.Id,State:TargetHealth.State}' \
  --output table
```

**预期输出**：Events 显示 rollout 过程；Target health 全部 `healthy`

### 4. 注册故障版本（演示排障）

> ⚠️ 当 ECS Service 已绑定 ALB Target Group 时，update-service 要求新 Task Definition 必须包含 ALB 配置中指定的容器端口（此处为 80）。使用错误端口（如 9999）会被 ECS 直接拒绝（InvalidParameterException），不会实际部署。这本身就是 ECS 的保护机制演示。

```bash
echo "=== 注册故障 Task Definition（错误端口）==="
TASK_DEF_BAD=$(aws ecs register-task-definition \
  --family ${TASK_FAMILY} \
  --network-mode awsvpc \
  --requires-compatibilities FARGATE \
  --cpu 256 \
  --memory 512 \
  --execution-role-arn ${EXECUTION_ROLE_ARN} \
  --task-role-arn ${TASK_ROLE_ARN} \
  --container-definitions "[
    {
      \"name\": \"${CONTAINER_NAME}\",
      \"image\": \"${IMAGE_URI_V2}\",
      \"portMappings\": [{\"containerPort\": 9999, \"protocol\": \"tcp\"}],
      \"essential\": true,
      \"logConfiguration\": {
        \"logDriver\": \"awslogs\",
        \"options\": {
          \"awslogs-group\": \"${LOG_GROUP}\",
          \"awslogs-region\": \"${AWS_REGION}\",
          \"awslogs-stream-prefix\": \"ecs\"
        }
      }
    }
  ]" \
  --region ${AWS_REGION} \
  --query taskDefinition.taskDefinitionArn --output text)

TASK_DEF_BAD_REV=$(echo ${TASK_DEF_BAD} | awk -F: '{print $NF}')

aws ecs update-service \
  --cluster ${CLUSTER_NAME} \
  --service ${SERVICE_NAME} \
  --task-definition ${TASK_FAMILY}:${TASK_DEF_BAD_REV} \
  --region ${AWS_REGION}

echo "等待 60 秒观察故障..."
sleep 60
```

### 5. 排查故障

```bash
echo "=== Service Events（排查）==="
aws ecs describe-services \
  --cluster ${CLUSTER_NAME} \
  --services ${SERVICE_NAME} \
  --region ${AWS_REGION} \
  --query 'services[0].events[:5].message' \
  --output table

echo "=== Target Health（应出现 unhealthy）==="
aws elbv2 describe-target-health \
  --target-group-arn ${TG_ARN} \
  --region ${AWS_REGION} \
  --query 'TargetHealthDescriptions[*].{IP:Target.Id,State:TargetHealth.State,Reason:TargetHealth.Reason}' \
  --output table

echo "=== 故障原因：端口不匹配，health check 失败 ==="
```

**预期输出**：Target 状态出现 `unhealthy`，原因为 health check 失败

### 6. 回滚到 v2

```bash
echo "=== 回滚到 v2（revision: ${TASK_DEF_V2_REV}）==="
aws ecs update-service \
  --cluster ${CLUSTER_NAME} \
  --service ${SERVICE_NAME} \
  --task-definition ${TASK_FAMILY}:${TASK_DEF_V2_REV} \
  --region ${AWS_REGION}

aws ecs wait services-stable \
  --cluster ${CLUSTER_NAME} \
  --services ${SERVICE_NAME} \
  --region ${AWS_REGION}

echo "=== 回滚后 ALB 返回版本 ==="
curl -s --max-time 10 http://${ALB_DNS}:8080/ | grep -o 'demo-ecs-web v[0-9]'
echo "回滚完成"
```

**预期输出**：ALB 重新返回 `demo-ecs-web v2`，打印"回滚完成"

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
| 1 | `aws ecs describe-services --cluster demo-ecs --services demo-ecs-web --region us-east-1 --query 'services[0].deployments[0].status' --output text` | `PRIMARY` |
| 2 | `aws elbv2 describe-target-health --target-group-arn $(aws elbv2 describe-target-groups --names demo-ecs-tg --region us-east-1 --query 'TargetGroups[0].TargetGroupArn' --output text) --region us-east-1 --query 'TargetHealthDescriptions[?TargetHealth.State==\`healthy\`] | length(@)' --output text` | `2` |
| 3 | `curl -s --max-time 10 http://$(aws elbv2 describe-load-balancers --names demo-ecs-alb --region us-east-1 --query 'LoadBalancers[0].DNSName' --output text):8080/ \| grep -c 'v2'` | `1` |
| 4 | `aws ecs describe-services --cluster demo-ecs --services demo-ecs-web --region us-east-1 --query 'services[0].runningCount' --output text` | `2` |

---

## 实验总结

本实验完成了「滚动部署与回滚」的全部操作，从资源创建到功能验证形成了完整闭环。通过动手实践，你已掌握了相关组件的核心配置方法和最佳实践。Demo06 将学习 Secrets 管理与 Task Role。

---

## 清理

```bash
source /tmp/demo-ecs.env

for REV in $(aws ecs list-task-definitions \
  --family-prefix ${TASK_FAMILY} \
  --region ${AWS_REGION} \
  --query 'taskDefinitionArns[*]' --output text); do
  aws ecs deregister-task-definition --task-definition ${REV} --region ${AWS_REGION} 2>/dev/null || true
done

echo "清理完成（Service 和 ALB 在 Demo04 清理）"
```
