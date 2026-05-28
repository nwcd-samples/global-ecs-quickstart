# Demo08 — 私有子网与 VPC Endpoint

## 实验简介

本实验将完成「私有子网与 VPC Endpoint」的全部操作，涵盖资源创建、配置、验证的完整流程。

**实验目标：**
- 掌握 私有子网与 VPC Endpoint 的核心操作流程
- 理解相关组件的工作原理和配置要点
- 能够独立完成从部署到验证的完整闭环

**实验流程：**
1. 创建 Endpoint 安全组
2. 获取私有路由表 ID
3. 创建 S3 Gateway Endpoint
4. 创建 Interface Endpoints
5. 等待 Endpoints Available
6. 将 Service 迁移到私有子网
7. 验证 ALB 仍可访问

**预计 AI 执行时长：** 8-10 分钟


## 前提条件

- **工具**：AWS CLI v2
- **权限**：AdministratorAccess（含 EC2 VPC Endpoint、ECS、ELBv2、IAM、Secrets Manager、CloudWatch Logs）
- **前提**：Demo04-Demo06 已完成，`demo-ecs-web` service 当前运行在公有子网
- **成本提示**：Interface Endpoint 持续计费，演示后按需删除
- **初始化**：

```bash
source /tmp/demo-ecs.env
```

---

## 步骤

### 1. 创建 Endpoint 安全组

```bash
VPCE_SG_ID=$(aws ec2 create-security-group \
  --group-name demo-ecs-vpce-sg \
  --description "VPC Endpoint security group" \
  --vpc-id ${VPC_ID} \
  --tag-specifications "ResourceType=security-group,Tags=[{Key=Name,Value=demo-ecs-vpce-sg},{Key=Project,Value=${PROJECT_TAG}}]" \
  --query GroupId --output text)

aws ec2 authorize-security-group-ingress \
  --group-id ${VPCE_SG_ID} \
  --protocol tcp \
  --port 443 \
  --source-group ${TASK_SG_ID}

echo "Endpoint SG: ${VPCE_SG_ID}"
```

**预期输出**：打印 Endpoint 安全组 ID

### 2. 获取私有路由表 ID

```bash
PRIVATE_RT=$(aws ec2 describe-route-tables \
  --filters "Name=vpc-id,Values=${VPC_ID}" \
  --query "RouteTables[?!contains(Associations[].Main, \`true\`) && RouteTableId!='${PUBLIC_RT}'].RouteTableId" \
  --output text | awk '{print $1}')

if [ -z "${PRIVATE_RT}" ]; then
  PRIVATE_RT=$(aws ec2 create-route-table \
    --vpc-id ${VPC_ID} \
    --tag-specifications "ResourceType=route-table,Tags=[{Key=Name,Value=demo-ecs-private-rt},{Key=Project,Value=${PROJECT_TAG}}]" \
    --query RouteTable.RouteTableId --output text)
  aws ec2 associate-route-table --route-table-id ${PRIVATE_RT} --subnet-id ${PRIVATE_SUBNET_1}
  aws ec2 associate-route-table --route-table-id ${PRIVATE_RT} --subnet-id ${PRIVATE_SUBNET_2}
fi
echo "Private Route Table: ${PRIVATE_RT}"
```

**预期输出**：打印私有路由表 ID

### 3. 创建 S3 Gateway Endpoint

```bash
aws ec2 create-vpc-endpoint \
  --vpc-id ${VPC_ID} \
  --vpc-endpoint-type Gateway \
  --service-name com.amazonaws.${AWS_REGION}.s3 \
  --route-table-ids ${PRIVATE_RT} \
  --tag-specifications "ResourceType=vpc-endpoint,Tags=[{Key=Name,Value=demo-ecs-s3-ep},{Key=Project,Value=${PROJECT_TAG}}]" \
  --region ${AWS_REGION}

echo "S3 Gateway Endpoint 已创建"
```

**预期输出**：打印"S3 Gateway Endpoint 已创建"

### 4. 创建 Interface Endpoints

```bash
SERVICES=(
  "ecr.api"
  "ecr.dkr"
  "logs"
  "secretsmanager"
  "ssm"
  "ssmmessages"
  "ec2messages"
)

for SVC in "${SERVICES[@]}"; do
  FULL_SVC="com.amazonaws.${AWS_REGION}.${SVC}"
  EP_ID=$(aws ec2 create-vpc-endpoint \
    --vpc-id ${VPC_ID} \
    --vpc-endpoint-type Interface \
    --service-name ${FULL_SVC} \
    --subnet-ids ${PRIVATE_SUBNET_1} ${PRIVATE_SUBNET_2} \
    --security-group-ids ${VPCE_SG_ID} \
    --private-dns-enabled \
    --tag-specifications "ResourceType=vpc-endpoint,Tags=[{Key=Name,Value=demo-ecs-${SVC}-ep},{Key=Project,Value=${PROJECT_TAG}}]" \
    --region ${AWS_REGION} \
    --query VpcEndpoint.VpcEndpointId --output text 2>/dev/null)
  echo "Endpoint ${SVC}: ${EP_ID}"
done
```

**预期输出**：每个 Endpoint 打印 ID（`vpce-xxxxxxxxx`）

### 5. 等待 Endpoints Available

```bash
echo "等待所有 Interface Endpoint 变为 available（约 2-3 分钟）..."
for i in $(seq 1 20); do
  PENDING=$(aws ec2 describe-vpc-endpoints \
    --filters "Name=vpc-id,Values=${VPC_ID}" \
    --query 'VpcEndpoints[?State==`pending`] | length(@)' --output text \
    --region ${AWS_REGION})
  echo "仍在 pending 的 endpoint 数: ${PENDING}"
  [ "${PENDING}" = "0" ] && break
  sleep 15
done

aws ec2 describe-vpc-endpoints \
  --filters "Name=vpc-id,Values=${VPC_ID}" \
  --query 'VpcEndpoints[*].{Service:ServiceName,State:State}' \
  --output table \
  --region ${AWS_REGION}
```

**预期输出**：所有 endpoint 状态为 `available`

### 6. 将 Service 迁移到私有子网

```bash
aws ecs update-service \
  --cluster ${CLUSTER_NAME} \
  --service ${SERVICE_NAME} \
  --network-configuration "awsvpcConfiguration={subnets=[${PRIVATE_SUBNET_1},${PRIVATE_SUBNET_2}],securityGroups=[${TASK_SG_ID}],assignPublicIp=DISABLED}" \
  --force-new-deployment \
  --region ${AWS_REGION}

aws ecs wait services-stable \
  --cluster ${CLUSTER_NAME} \
  --services ${SERVICE_NAME} \
  --region ${AWS_REGION}

echo "Service 已迁移到私有子网"
```

**预期输出**：打印"Service 已迁移到私有子网"

### 7. 验证 ALB 仍可访问

```bash
curl -s -o /dev/null -w "ALB HTTP Status: %{http_code}\n" \
  --max-time 15 http://${ALB_DNS}:8080/

echo "=== 验证 Task 使用私有子网 ==="
TASK_ARN=$(aws ecs list-tasks \
  --cluster ${CLUSTER_NAME} \
  --service-name ${SERVICE_NAME} \
  --region ${AWS_REGION} \
  --query 'taskArns[0]' --output text)

aws ecs describe-tasks \
  --cluster ${CLUSTER_NAME} \
  --tasks ${TASK_ARN} \
  --region ${AWS_REGION} \
  --query 'tasks[0].attachments[0].details[?name==`subnetId`].value' \
  --output text
```

**预期输出**：ALB 返回 `200`；Task 子网 ID 属于私有子网（`PRIVATE_SUBNET_1` 或 `PRIVATE_SUBNET_2`）

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
| 1 | `aws ec2 describe-vpc-endpoints --filters Name=vpc-id,Values=$(aws ec2 describe-vpcs --filters Name=tag:Project,Values=ecs-global-quickstart --query 'Vpcs[0].VpcId' --output text) Name=state,Values=available --region us-east-1 --query 'length(VpcEndpoints)' --output text` | 至少 `8` |
| 2 | `aws ecs describe-services --cluster demo-ecs --services demo-ecs-web --region us-east-1 --query 'services[0].networkConfiguration.awsvpcConfiguration.assignPublicIp' --output text` | `DISABLED` |
| 3 | `curl -s -o /dev/null -w "%{http_code}" --max-time 15 http://$(aws elbv2 describe-load-balancers --names demo-ecs-alb --region us-east-1 --query 'LoadBalancers[0].DNSName' --output text):8080/` | `200` |
| 4 | `aws ecs describe-services --cluster demo-ecs --services demo-ecs-web --region us-east-1 --query 'services[0].runningCount' --output text` | `2` |

---

## 实验总结

本实验完成了「私有子网与 VPC Endpoint」的全部操作，从资源创建到功能验证形成了完整闭环。通过动手实践，你已掌握了相关组件的核心配置方法和最佳实践。Demo09 将学习 CloudWatch 日志与 ECS Exec。

---

## 清理

```bash
source /tmp/demo-ecs.env

for EP_ID in $(aws ec2 describe-vpc-endpoints \
  --filters "Name=vpc-id,Values=${VPC_ID}" \
  --query 'VpcEndpoints[*].VpcEndpointId' \
  --output text --region ${AWS_REGION}); do
  aws ec2 delete-vpc-endpoints \
    --vpc-endpoint-ids ${EP_ID} \
    --region ${AWS_REGION} 2>/dev/null || true
done

aws ec2 delete-security-group --group-id ${VPCE_SG_ID} --region ${AWS_REGION} 2>/dev/null || true

echo "VPC Endpoints 和 Endpoint SG 已清理"
```
