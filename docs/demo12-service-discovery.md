# Demo12 — Service Discovery 与 Service Connect

## 实验简介

本实验将完成「Service Discovery 服务发现」的全部操作，涵盖资源创建、配置、验证的完整流程。

**实验目标：**
- 掌握 Service Discovery 服务发现 的核心操作流程
- 理解相关组件的工作原理和配置要点
- 能够独立完成从部署到验证的完整闭环

**实验流程：**
1. 创建 Cloud Map Private DNS Namespace
2. 创建 Cloud Map Service（backend）
3. 注册 Backend Task Definition
4. 创建 Backend ECS Service（带 Service Discovery）
5. 验证 Cloud Map 注册实例
6. 部署 Frontend Task（通过 DNS 访问 Backend）
7. 故障场景：缩容 Backend 并观察
8. 恢复 Backend
9. 说明：Service Connect（可选）

**预计 AI 执行时长：** 8-10 分钟


## 前提条件

- **工具**：AWS CLI v2
- **权限**：AdministratorAccess（含 ECS、Cloud Map、Route 53、EC2、IAM、CloudWatch Logs）
- **前提**：Demo01 已完成，VPC 和子网已创建；Demo02 ECR 镜像可复用
- **初始化**：

```bash
source /tmp/demo-ecs.env
```

---

## 步骤

### 1. 创建 Cloud Map Private DNS Namespace

> ⚠️ Service Discovery 要求 Task 安全组允许来自同安全组的 80 端口流量（Task 间互访）。在步骤4创建 Backend Service 前，需确保 Task SG 已添加自引用入站规则：
> ```bash
> aws ec2 authorize-security-group-ingress \
>   --group-id ${TASK_SG_ID} --protocol tcp --port 80 --source-group ${TASK_SG_ID}
> ```

```bash
NS_OP_ID=$(aws servicediscovery create-private-dns-namespace \
  --name demo.local \
  --vpc ${VPC_ID} \
  --tags Key=Project,Value=${PROJECT_TAG} Key=Demo,Value=Demo12 \
  --region ${AWS_REGION} \
  --query OperationId --output text)

echo "Namespace operation: ${NS_OP_ID}"

until [ "$(aws servicediscovery get-operation \
  --operation-id ${NS_OP_ID} \
  --region ${AWS_REGION} \
  --query 'Operation.Status' --output text)" = "SUCCESS" ]; do
  echo "等待 namespace 创建..."
  sleep 5
done

NS_ID=$(aws servicediscovery list-namespaces \
  --region ${AWS_REGION} \
  --query "Namespaces[?Name=='demo.local'].Id" --output text)

echo "Namespace ID: ${NS_ID}"
```

**预期输出**：Operation Status 变为 `SUCCESS`；打印 Namespace ID

### 2. 创建 Cloud Map Service（backend）

```bash
SD_SVC_ID=$(aws servicediscovery create-service \
  --name backend \
  --namespace-id ${NS_ID} \
  --dns-config "NamespaceId=${NS_ID},DnsRecords=[{Type=A,TTL=10}]" \
  --health-check-custom-config FailureThreshold=1 \
  --tags Key=Project,Value=${PROJECT_TAG} \
  --region ${AWS_REGION} \
  --query Service.Id --output text)

SD_SVC_ARN=$(aws servicediscovery get-service \
  --id ${SD_SVC_ID} \
  --region ${AWS_REGION} \
  --query Service.Arn --output text)

echo "Cloud Map Service ID: ${SD_SVC_ID}"
echo "Cloud Map Service ARN: ${SD_SVC_ARN}"
```

**预期输出**：打印 Cloud Map Service ID 和 ARN

### 3. 注册 Backend Task Definition

```bash
TASK_DEF_BACKEND=$(aws ecs register-task-definition \
  --family demo-ecs-backend \
  --network-mode awsvpc \
  --requires-compatibilities FARGATE \
  --cpu 256 \
  --memory 512 \
  --execution-role-arn ${EXECUTION_ROLE_ARN} \
  --task-role-arn ${TASK_ROLE_ARN} \
  --container-definitions "[
    {
      \"name\": \"backend\",
      \"image\": \"public.ecr.aws/nginx/nginx:latest\",
      \"essential\": true,
      \"portMappings\": [{\"containerPort\": 80, \"protocol\": \"tcp\"}],
      \"command\": [\"sh\", \"-c\", \"echo 'backend-v1' > /usr/share/nginx/html/index.html && nginx -g 'daemon off;'\"],
      \"logConfiguration\": {
        \"logDriver\": \"awslogs\",
        \"options\": {
          \"awslogs-group\": \"${LOG_GROUP}\",
          \"awslogs-region\": \"${AWS_REGION}\",
          \"awslogs-stream-prefix\": \"backend\"
        }
      }
    }
  ]" \
  --region ${AWS_REGION} \
  --query taskDefinition.taskDefinitionArn --output text)

echo "Backend Task Definition: ${TASK_DEF_BACKEND}"
```

**预期输出**：打印 Backend Task Definition ARN

### 4. 创建 Backend ECS Service（带 Service Discovery）

```bash
BACKEND_SVC=$(aws ecs create-service \
  --cluster ${CLUSTER_NAME} \
  --service-name demo-ecs-backend \
  --task-definition demo-ecs-backend \
  --launch-type FARGATE \
  --desired-count 2 \
  --network-configuration "awsvpcConfiguration={subnets=[${PUBLIC_SUBNET_1},${PUBLIC_SUBNET_2}],securityGroups=[${TASK_SG_ID}],assignPublicIp=ENABLED}" \
  --service-registries "[{\"registryArn\":\"${SD_SVC_ARN}\"}]" \
  --region ${AWS_REGION} \
  --query service.serviceArn --output text)

aws ecs wait services-stable \
  --cluster ${CLUSTER_NAME} \
  --services demo-ecs-backend \
  --region ${AWS_REGION}

echo "Backend service 稳定，ARN: ${BACKEND_SVC}"
```

**预期输出**：Backend service 稳定，2 个 task 运行中

### 5. 验证 Cloud Map 注册实例

```bash
echo "=== Cloud Map 注册实例 ==="
aws servicediscovery list-instances \
  --service-id ${SD_SVC_ID} \
  --region ${AWS_REGION} \
  --query 'Instances[*].{Id:Id,IP:Attributes.AWS_INSTANCE_IPV4,Port:Attributes.AWS_INSTANCE_PORT}' \
  --output table
```

**预期输出**：列出 2 个 backend task 实例（含 IP 和端口）

### 6. 部署 Frontend Task（通过 DNS 访问 Backend）

```bash
FRONTEND_CMD="for i in \$(seq 1 5); do echo \"Request \$i:\"; curl -s --max-time 3 http://backend.demo.local/ || echo 'FAILED'; sleep 2; done; echo 'Frontend test done'"

FRONTEND_TASK=$(aws ecs run-task \
  --cluster ${CLUSTER_NAME} \
  --task-definition demo-ecs-backend \
  --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={subnets=[${PUBLIC_SUBNET_1}],securityGroups=[${TASK_SG_ID}],assignPublicIp=ENABLED}" \
  --overrides "{\"containerOverrides\":[{\"name\":\"backend\",\"command\":[\"sh\",\"-c\",\"${FRONTEND_CMD}\"]}]}" \
  --region ${AWS_REGION} \
  --query 'tasks[0].taskArn' --output text)

echo "Frontend task: ${FRONTEND_TASK}"

aws ecs wait tasks-stopped \
  --cluster ${CLUSTER_NAME} \
  --tasks ${FRONTEND_TASK} \
  --region ${AWS_REGION}

echo "=== Frontend Task 日志（验证 DNS 访问）==="
sleep 5
aws logs tail ${LOG_GROUP} --since 3m --region ${AWS_REGION} 2>/dev/null | grep "backend/" | head -20
```

**预期输出**：日志显示 `backend-v1` 响应内容，说明通过 `backend.demo.local` DNS 解析成功

### 7. 故障场景：缩容 Backend 并观察

```bash
aws ecs update-service \
  --cluster ${CLUSTER_NAME} \
  --service demo-ecs-backend \
  --desired-count 0 \
  --region ${AWS_REGION}

echo "等待 backend 缩容..."
sleep 30

echo "=== 验证 Cloud Map 实例（缩容后应为空）==="
aws servicediscovery list-instances \
  --service-id ${SD_SVC_ID} \
  --region ${AWS_REGION} \
  --query 'Instances[*].Id' --output text

FAIL_CMD="curl -s --max-time 3 http://backend.demo.local/ && echo 'OK' || echo 'BACKEND UNREACHABLE'"
FAIL_TASK=$(aws ecs run-task \
  --cluster ${CLUSTER_NAME} \
  --task-definition demo-ecs-backend \
  --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={subnets=[${PUBLIC_SUBNET_1}],securityGroups=[${TASK_SG_ID}],assignPublicIp=ENABLED}" \
  --overrides "{\"containerOverrides\":[{\"name\":\"backend\",\"command\":[\"sh\",\"-c\",\"${FAIL_CMD}\"]}]}" \
  --region ${AWS_REGION} \
  --query 'tasks[0].taskArn' --output text)

aws ecs wait tasks-stopped \
  --cluster ${CLUSTER_NAME} \
  --tasks ${FAIL_TASK} \
  --region ${AWS_REGION}

sleep 5
aws logs tail ${LOG_GROUP} --since 2m --region ${AWS_REGION} 2>/dev/null | grep "backend/" | tail -5
```

**预期输出**：Cloud Map 实例为空；日志显示 `BACKEND UNREACHABLE`（故障场景验证）

### 8. 恢复 Backend

```bash
aws ecs update-service \
  --cluster ${CLUSTER_NAME} \
  --service demo-ecs-backend \
  --desired-count 2 \
  --region ${AWS_REGION}

aws ecs wait services-stable \
  --cluster ${CLUSTER_NAME} \
  --services demo-ecs-backend \
  --region ${AWS_REGION}

echo "Backend 已恢复，Cloud Map 实例数："
aws servicediscovery list-instances \
  --service-id ${SD_SVC_ID} \
  --region ${AWS_REGION} \
  --query 'length(Instances)' --output text
```

**预期输出**：Backend 稳定，Cloud Map 实例数恢复为 `2`

### 9. 说明：Service Connect（可选）

```bash
echo "=== Service Connect 说明 ==="
cat << 'INFO'
Service Connect 是 Cloud Map Service Discovery 的进化版本：
  - 无需客户端 DNS 解析，ECS Agent 代理流量
  - 支持 per-request 负载均衡（vs DNS TTL 缓存）
  - 提供连接级指标（请求数、延迟、错误率）
  - 配置方式：在 create-service/update-service 中指定 --service-connect-configuration

示例配置（不执行）：
aws ecs update-service \
  --cluster demo-ecs \
  --service demo-ecs-backend \
  --service-connect-configuration '{
    "enabled": true,
    "namespace": "demo.local",
    "services": [{
      "portName": "http",
      "discoveryName": "backend",
      "clientAliases": [{"port": 80, "dnsName": "backend"}]
    }]
  }'

适用场景：微服务间东西向流量治理、需要统一可观测性的服务网格轻量替代方案。
INFO
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
| 1 | `aws servicediscovery list-namespaces --region us-east-1 --query "Namespaces[?Name=='demo.local'].Name" --output text` | `demo.local` |
| 2 | `aws servicediscovery list-namespaces --region us-east-1 --query "length(Namespaces[?Name=='demo.local'])" --output text` | `1` |
| 3 | `aws ecs describe-services --cluster demo-ecs --services demo-ecs-backend --region us-east-1 --query 'services[0].runningCount' --output text` | `2` |
| 4 | `aws servicediscovery list-services --region us-east-1 --query "Services[?Name=='backend'].Name" --output text` | `backend` |

---

## 实验总结

本实验完成了「Service Discovery 服务发现」的全部操作，从资源创建到功能验证形成了完整闭环。通过动手实践，你已掌握了相关组件的核心配置方法和最佳实践。Demo13 将学习 CodeDeploy Blue/Green 部署。

---

## 清理

```bash
source /tmp/demo-ecs.env

aws ecs update-service \
  --cluster ${CLUSTER_NAME} \
  --service demo-ecs-backend \
  --desired-count 0 \
  --region ${AWS_REGION} 2>/dev/null || true

sleep 15

aws ecs delete-service \
  --cluster ${CLUSTER_NAME} \
  --service demo-ecs-backend \
  --force \
  --region ${AWS_REGION} 2>/dev/null || true

aws ecs deregister-task-definition \
  --task-definition demo-ecs-backend:1 \
  --region ${AWS_REGION} 2>/dev/null || true

SD_SVC_ID=$(aws servicediscovery list-services \
  --region ${AWS_REGION} \
  --query "Services[?Name=='backend'].Id" --output text 2>/dev/null)

if [ -n "${SD_SVC_ID}" ]; then
  for INST in $(aws servicediscovery list-instances \
    --service-id ${SD_SVC_ID} \
    --region ${AWS_REGION} \
    --query 'Instances[*].Id' --output text 2>/dev/null); do
    aws servicediscovery deregister-instance \
      --service-id ${SD_SVC_ID} \
      --instance-id ${INST} \
      --region ${AWS_REGION} 2>/dev/null || true
  done
  sleep 10
  aws servicediscovery delete-service \
    --id ${SD_SVC_ID} \
    --region ${AWS_REGION} 2>/dev/null || true
fi

NS_ID=$(aws servicediscovery list-namespaces \
  --region ${AWS_REGION} \
  --query "Namespaces[?Name=='demo.local'].Id" --output text 2>/dev/null)

if [ -n "${NS_ID}" ]; then
  aws servicediscovery delete-namespace \
    --id ${NS_ID} \
    --region ${AWS_REGION} 2>/dev/null || true
fi

echo "清理完成"
```
