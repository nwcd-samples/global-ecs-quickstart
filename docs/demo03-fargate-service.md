# Demo03 — 部署第一个 Fargate 服务

## 实验简介

本实验将完成「创建第一个 Fargate Service」的全部操作，涵盖资源创建、配置、验证的完整流程。

**实验目标：**
- 掌握 创建第一个 Fargate Service 的核心操作流程
- 理解相关组件的工作原理和配置要点
- 能够独立完成从部署到验证的完整闭环

**实验流程：**
1. 注册 Fargate Task Definition
2. 运行 One-off Task 预验证
3. 创建 ECS Service
4. 查看 Service 状态和日志
5. 扩容到 2 个副本
6. 保存 Service 变量

**预计 AI 执行时长：** 5-8 分钟


## 前提条件

- **工具**：AWS CLI v2
- **权限**：AdministratorAccess（含 ECS、IAM PassRole、EC2 ENI、CloudWatch Logs）
- **前提**：Demo01、Demo02 已完成
- **初始化**：

```bash
source /tmp/demo-ecs.env
export SERVICE_NAME=demo-ecs-web
export TASK_FAMILY=demo-ecs-web
export CONTAINER_NAME=web
```

---

## 步骤

### 1. 注册 Fargate Task Definition

```bash
TASK_DEF_ARN=$(aws ecs register-task-definition \
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
      \"image\": \"${IMAGE_URI_STABLE}\",
      \"portMappings\": [{\"containerPort\": 80, \"protocol\": \"tcp\"}],
      \"essential\": true,
      \"logConfiguration\": {
        \"logDriver\": \"awslogs\",
        \"options\": {
          \"awslogs-group\": \"${LOG_GROUP}\",
          \"awslogs-region\": \"${AWS_REGION}\",
          \"awslogs-stream-prefix\": \"ecs\"
        }
      },
      \"healthCheck\": {
        \"command\": [\"CMD-SHELL\", \"curl -sf http://localhost/ || exit 1\"],
        \"interval\": 30,
        \"timeout\": 5,
        \"retries\": 3,
        \"startPeriod\": 10
      }
    }
  ]" \
  --tags key=Project,value=${PROJECT_TAG} key=Demo,value=Demo03 \
  --region ${AWS_REGION} \
  --query taskDefinition.taskDefinitionArn --output text)

echo "Task Definition 已注册: ${TASK_DEF_ARN}"
```

**预期输出**：打印 Task Definition ARN

### 2. 运行 One-off Task 预验证

```bash
TRIAL_TASK=$(aws ecs run-task \
  --cluster ${CLUSTER_NAME} \
  --task-definition ${TASK_FAMILY} \
  --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={subnets=[${PUBLIC_SUBNET_1}],securityGroups=[${TASK_SG_ID}],assignPublicIp=ENABLED}" \
  --region ${AWS_REGION} \
  --query 'tasks[0].taskArn' --output text)

echo "Trial task ARN: ${TRIAL_TASK}"

aws ecs wait tasks-stopped \
  --cluster ${CLUSTER_NAME} \
  --tasks ${TRIAL_TASK} \
  --region ${AWS_REGION}

STOP_REASON=$(aws ecs describe-tasks \
  --cluster ${CLUSTER_NAME} \
  --tasks ${TRIAL_TASK} \
  --region ${AWS_REGION} \
  --query 'tasks[0].stoppedReason' --output text)

echo "Trial task 停止原因: ${STOP_REASON}"
```

**预期输出**：Task 停止原因应为 `Essential container in task exited`（正常退出）或容器自然结束，不应有拉镜像失败等错误。

### 3. 创建 ECS Service

```bash
aws ecs create-service \
  --cluster ${CLUSTER_NAME} \
  --service-name ${SERVICE_NAME} \
  --task-definition ${TASK_FAMILY} \
  --desired-count 1 \
  --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={subnets=[${PUBLIC_SUBNET_1},${PUBLIC_SUBNET_2}],securityGroups=[${TASK_SG_ID}],assignPublicIp=ENABLED}" \
  --tags key=Project,value=${PROJECT_TAG} key=Demo,value=Demo03 key=Owner,value=${OWNER_TAG} \
  --region ${AWS_REGION}

aws ecs wait services-stable \
  --cluster ${CLUSTER_NAME} \
  --services ${SERVICE_NAME} \
  --region ${AWS_REGION}

echo "ECS Service 已稳定运行"
```

**预期输出**：打印"ECS Service 已稳定运行"，约 1-2 分钟。

### 4. 查看 Service 状态和日志

```bash
echo "=== Service 状态 ==="
aws ecs describe-services \
  --cluster ${CLUSTER_NAME} \
  --services ${SERVICE_NAME} \
  --region ${AWS_REGION} \
  --query 'services[0].{Status:status,Running:runningCount,Desired:desiredCount}' \
  --output table

echo "=== 最近 events ==="
aws ecs describe-services \
  --cluster ${CLUSTER_NAME} \
  --services ${SERVICE_NAME} \
  --region ${AWS_REGION} \
  --query 'services[0].events[:5].[createdAt,message]' \
  --output table

echo "=== CloudWatch 日志（等待约 30 秒）==="
sleep 30
aws logs tail ${LOG_GROUP} --since 5m --region ${AWS_REGION} 2>/dev/null | head -10 || \
  echo "日志尚未出现，稍后重试"
```

**预期输出**：Service 状态 `ACTIVE`，`runningCount=1`；日志中可见容器启动信息。

### 5. 扩容到 2 个副本

```bash
aws ecs update-service \
  --cluster ${CLUSTER_NAME} \
  --service ${SERVICE_NAME} \
  --desired-count 2 \
  --region ${AWS_REGION}

aws ecs wait services-stable \
  --cluster ${CLUSTER_NAME} \
  --services ${SERVICE_NAME} \
  --region ${AWS_REGION}

aws ecs describe-services \
  --cluster ${CLUSTER_NAME} \
  --services ${SERVICE_NAME} \
  --region ${AWS_REGION} \
  --query 'services[0].{Running:runningCount,Desired:desiredCount}' \
  --output json

echo "扩容到 2 个副本完成"
```

**预期输出**：`runningCount=2`，打印"扩容到 2 个副本完成"

### 6. 保存 Service 变量

```bash
cat >> /tmp/demo-ecs.env << EOF
export SERVICE_NAME=${SERVICE_NAME}
export TASK_FAMILY=${TASK_FAMILY}
export CONTAINER_NAME=${CONTAINER_NAME}
EOF

echo "Service 变量已追加到 /tmp/demo-ecs.env"
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
| 1 | `aws ecs describe-services --cluster demo-ecs --services demo-ecs-web --region us-east-1 --query 'services[0].status' --output text` | `ACTIVE` |
| 2 | `aws ecs describe-services --cluster demo-ecs --services demo-ecs-web --region us-east-1 --query 'services[0].runningCount' --output text` | `2` |
| 3 | `aws ecs list-task-definitions --family-prefix demo-ecs-web --region us-east-1 --query 'length(taskDefinitionArns)' --output text` | 大于等于 `1` |
| 4 | `aws logs describe-log-groups --log-group-name-prefix /ecs/demo-ecs --region us-east-1 --query 'length(logGroups)' --output text` | `1` |

---

## 实验总结

本实验完成了「创建第一个 Fargate Service」的全部操作，从资源创建到功能验证形成了完整闭环。通过动手实践，你已掌握了相关组件的核心配置方法和最佳实践。Demo04 将学习通过 ALB 暴露服务。

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

aws ecs deregister-task-definition \
  --task-definition ${TASK_FAMILY}:1 \
  --region ${AWS_REGION} 2>/dev/null || true

echo "清理完成"
```
