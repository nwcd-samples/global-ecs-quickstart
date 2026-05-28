# Demo06 — 任务配置 Secrets 与 Task Role

## 实验简介

本实验将完成「Secrets 管理与 Task Role」的全部操作，涵盖资源创建、配置、验证的完整流程。

**实验目标：**
- 掌握 Secrets 管理与 Task Role 的核心操作流程
- 理解相关组件的工作原理和配置要点
- 能够独立完成从部署到验证的完整闭环

**实验流程：**
1. 创建 Secrets Manager Secret
2. 授权 Execution Role 读取 Secret
3. 为 Task Role 添加最小业务权限
4. 注册新 Task Definition（注入 Secret）
5. 更新 Service 并等待稳定
6. 验证 Secret 注入成功（不打印完整值）

**预计 AI 执行时长：** 5-8 分钟


## 前提条件

- **工具**：AWS CLI v2
- **权限**：AdministratorAccess（含 ECS、IAM、Secrets Manager、CloudWatch Logs）
- **前提**：Demo04 或 Demo05 已完成，`demo-ecs-web` service 运行中
- **初始化**：

```bash
source /tmp/demo-ecs.env
```

---

## 步骤

### 1. 创建 Secrets Manager Secret

```bash
SECRET_ARN=$(aws secretsmanager create-secret \
  --name demo-ecs/app/password \
  --secret-string '{"api_key":"demo-secret-12345","db_password":"demo-db-pass"}' \
  --tags Key=Project,Value=${PROJECT_TAG} Key=Demo,Value=Demo06 \
  --region ${AWS_REGION} \
  --query ARN --output text)

echo "Secret 已创建（ARN 仅供内部使用）"
echo "Secret 名称: demo-ecs/app/password"
```

**预期输出**：打印"Secret 已创建"（不打印完整 ARN 或 Secret 内容）

### 2. 授权 Execution Role 读取 Secret

```bash
cat > /tmp/secret-policy.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": ["secretsmanager:GetSecretValue"],
    "Resource": "${SECRET_ARN}"
  }]
}
EOF

aws iam put-role-policy \
  --role-name demo-ecs-task-execution-role \
  --policy-name demo-ecs-secret-access \
  --policy-document file:///tmp/secret-policy.json

echo "Execution Role 已获得 Secret 读取权限"
```

**预期输出**：打印"Execution Role 已获得 Secret 读取权限"

### 3. 为 Task Role 添加最小业务权限

```bash
cat > /tmp/task-role-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": ["sts:GetCallerIdentity"],
    "Resource": "*"
  }]
}
EOF

aws iam put-role-policy \
  --role-name demo-ecs-task-role \
  --policy-name demo-ecs-task-minimal \
  --policy-document file:///tmp/task-role-policy.json

echo "Task Role 最小权限已配置"
```

**预期输出**：打印"Task Role 最小权限已配置"

### 4. 注册新 Task Definition（注入 Secret）

```bash
TASK_DEF_SEC=$(aws ecs register-task-definition \
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
      \"secrets\": [
        {
          \"name\": \"APP_API_KEY\",
          \"valueFrom\": \"${SECRET_ARN}:api_key::\"
        }
      ],
      \"environment\": [
        {\"name\": \"APP_VERSION\", \"value\": \"with-secrets\"}
      ],
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

TASK_DEF_SEC_REV=$(echo ${TASK_DEF_SEC} | awk -F: '{print $NF}')
echo "含 Secret 的 Task Definition Revision: ${TASK_DEF_SEC_REV}"
```

**预期输出**：打印新 revision 号

### 5. 更新 Service 并等待稳定

```bash
aws ecs update-service \
  --cluster ${CLUSTER_NAME} \
  --service ${SERVICE_NAME} \
  --task-definition ${TASK_FAMILY}:${TASK_DEF_SEC_REV} \
  --region ${AWS_REGION}

aws ecs wait services-stable \
  --cluster ${CLUSTER_NAME} \
  --services ${SERVICE_NAME} \
  --region ${AWS_REGION}

echo "Service 已更新到含 Secret 的版本"
```

**预期输出**：打印"Service 已更新到含 Secret 的版本"

### 6. 验证 Secret 注入成功（不打印完整值）

```bash
echo "=== 检查 Secret 是否注入（通过日志确认，不打印完整值）==="
sleep 30
aws logs tail ${LOG_GROUP} --since 3m --region ${AWS_REGION} 2>/dev/null | head -5 || \
  echo "日志正在生成..."

echo "=== 验证 Task Role（容器内 AWS 调用使用 task role）==="
TASK_ARN=$(aws ecs list-tasks \
  --cluster ${CLUSTER_NAME} \
  --service-name ${SERVICE_NAME} \
  --region ${AWS_REGION} \
  --query 'taskArns[0]' --output text)

echo "当前 Task: ${TASK_ARN}"

aws ecs describe-tasks \
  --cluster ${CLUSTER_NAME} \
  --tasks ${TASK_ARN} \
  --region ${AWS_REGION} \
  --query 'tasks[0].{TaskRole:overrides.taskRoleArn,Status:lastStatus}' \
  --output table
```

**预期输出**：Task 使用 task role，Secret 已注入（不输出完整 Secret 内容）

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
| 1 | `aws secretsmanager describe-secret --secret-id demo-ecs/app/password --region us-east-1 --query 'Name' --output text` | `demo-ecs/app/password` |
| 2 | `aws ecs describe-services --cluster demo-ecs --services demo-ecs-web --region us-east-1 --query 'services[0].runningCount' --output text` | `2` |
| 3 | `aws iam get-role-policy --role-name demo-ecs-task-execution-role --policy-name demo-ecs-secret-access --query 'PolicyName' --output text` | `demo-ecs-secret-access` |
| 4 | `aws iam get-role-policy --role-name demo-ecs-task-role --policy-name demo-ecs-task-minimal --query 'PolicyName' --output text` | `demo-ecs-task-minimal` |

---

## 实验总结

本实验完成了「Secrets 管理与 Task Role」的全部操作，从资源创建到功能验证形成了完整闭环。通过动手实践，你已掌握了相关组件的核心配置方法和最佳实践。Demo07 将学习 Service Auto Scaling。

---

## 清理

```bash
source /tmp/demo-ecs.env

aws secretsmanager delete-secret \
  --secret-id demo-ecs/app/password \
  --force-delete-without-recovery \
  --region ${AWS_REGION} 2>/dev/null || true

aws iam delete-role-policy \
  --role-name demo-ecs-task-execution-role \
  --policy-name demo-ecs-secret-access 2>/dev/null || true

aws iam delete-role-policy \
  --role-name demo-ecs-task-role \
  --policy-name demo-ecs-task-minimal 2>/dev/null || true

echo "清理完成"
```
