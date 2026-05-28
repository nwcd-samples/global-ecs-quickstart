# Demo01 — 准备实验环境与基础资源

## 实验简介

本实验将完成「环境准备与 VPC/集群创建」的全部操作，涵盖资源创建、配置、验证的完整流程。

**实验目标：**
- 掌握 环境准备与 VPC/集群创建 的核心操作流程
- 理解相关组件的工作原理和配置要点
- 能够独立完成从部署到验证的完整闭环

**实验流程：**
1. 验证工具和 AWS 连通
2. 创建 VPC
3. 创建 Internet Gateway
4. 创建公有子网（2 个 AZ）
5. 创建私有子网（2 个 AZ）
6. 配置公有路由表
7. 创建安全组
8. 创建 ECS Cluster 和 Log Group
9. 创建 IAM Role
10. 保存环境变量

**预计 AI 执行时长：** 15-20 分钟


## 前提条件

- **工具**：AWS CLI v2、Docker、jq、git、Session Manager plugin
- **权限**：AdministratorAccess（含 EC2、ECS、IAM、ECR、CloudWatch Logs、ELB 创建权限）
- **说明**：本 Demo 创建所有后续 Demo 共用的 VPC、子网、安全组、ECS Cluster 和 IAM Role
- **初始化**：

```bash
export AWS_REGION=us-east-1
export AWS_DEFAULT_REGION=us-east-1
export CLUSTER_NAME=demo-ecs
export PROJECT_TAG=ecs-global-quickstart
export OWNER_TAG=${USER:-ecs-lab}
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export ECR_REGISTRY=${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com
export LOG_GROUP=/ecs/demo-ecs
```

---

## 步骤

### 1. 验证工具和 AWS 连通

```bash
aws --version
docker --version
jq --version
git --version
session-manager-plugin --version 2>/dev/null || echo "Session Manager plugin 未安装，Demo09 ECS Exec 需要"

aws sts get-caller-identity --query 'Account' --output text
echo "AWS 连通正常"
```

**预期输出**：工具版本号；Account ID（12位数字）；"AWS 连通正常"

### 2. 创建 VPC

```bash
VPC_ID=$(aws ec2 create-vpc \
  --cidr-block 10.42.0.0/16 \
  --tag-specifications "ResourceType=vpc,Tags=[{Key=Name,Value=demo-ecs-vpc},{Key=Project,Value=${PROJECT_TAG}},{Key=Owner,Value=${OWNER_TAG}}]" \
  --query Vpc.VpcId --output text)

aws ec2 modify-vpc-attribute --vpc-id ${VPC_ID} --enable-dns-hostnames
aws ec2 modify-vpc-attribute --vpc-id ${VPC_ID} --enable-dns-support

echo "VPC ID: ${VPC_ID}"
```

**预期输出**：打印 VPC ID（`vpc-xxxxxxxxx`）

### 3. 创建 Internet Gateway

```bash
IGW_ID=$(aws ec2 create-internet-gateway \
  --tag-specifications "ResourceType=internet-gateway,Tags=[{Key=Name,Value=demo-ecs-igw},{Key=Project,Value=${PROJECT_TAG}}]" \
  --query InternetGateway.InternetGatewayId --output text)

aws ec2 attach-internet-gateway --internet-gateway-id ${IGW_ID} --vpc-id ${VPC_ID}
echo "IGW ID: ${IGW_ID}"
```

**预期输出**：打印 IGW ID

### 4. 创建公有子网（2 个 AZ）

```bash
PUBLIC_SUBNET_1=$(aws ec2 create-subnet \
  --vpc-id ${VPC_ID} \
  --cidr-block 10.42.1.0/24 \
  --availability-zone ${AWS_REGION}a \
  --tag-specifications "ResourceType=subnet,Tags=[{Key=Name,Value=demo-ecs-public-1a},{Key=Project,Value=${PROJECT_TAG}}]" \
  --query Subnet.SubnetId --output text)

PUBLIC_SUBNET_2=$(aws ec2 create-subnet \
  --vpc-id ${VPC_ID} \
  --cidr-block 10.42.2.0/24 \
  --availability-zone ${AWS_REGION}b \
  --tag-specifications "ResourceType=subnet,Tags=[{Key=Name,Value=demo-ecs-public-1b},{Key=Project,Value=${PROJECT_TAG}}]" \
  --query Subnet.SubnetId --output text)

aws ec2 modify-subnet-attribute --subnet-id ${PUBLIC_SUBNET_1} --map-public-ip-on-launch
aws ec2 modify-subnet-attribute --subnet-id ${PUBLIC_SUBNET_2} --map-public-ip-on-launch

echo "Public Subnet 1: ${PUBLIC_SUBNET_1}"
echo "Public Subnet 2: ${PUBLIC_SUBNET_2}"
```

**预期输出**：打印两个公有子网 ID

### 5. 创建私有子网（2 个 AZ）

```bash
PRIVATE_SUBNET_1=$(aws ec2 create-subnet \
  --vpc-id ${VPC_ID} \
  --cidr-block 10.42.101.0/24 \
  --availability-zone ${AWS_REGION}a \
  --tag-specifications "ResourceType=subnet,Tags=[{Key=Name,Value=demo-ecs-private-1a},{Key=Project,Value=${PROJECT_TAG}}]" \
  --query Subnet.SubnetId --output text)

PRIVATE_SUBNET_2=$(aws ec2 create-subnet \
  --vpc-id ${VPC_ID} \
  --cidr-block 10.42.102.0/24 \
  --availability-zone ${AWS_REGION}b \
  --tag-specifications "ResourceType=subnet,Tags=[{Key=Name,Value=demo-ecs-private-1b},{Key=Project,Value=${PROJECT_TAG}}]" \
  --query Subnet.SubnetId --output text)

echo "Private Subnet 1: ${PRIVATE_SUBNET_1}"
echo "Private Subnet 2: ${PRIVATE_SUBNET_2}"
```

**预期输出**：打印两个私有子网 ID

### 6. 配置公有路由表

```bash
PUBLIC_RT=$(aws ec2 create-route-table \
  --vpc-id ${VPC_ID} \
  --tag-specifications "ResourceType=route-table,Tags=[{Key=Name,Value=demo-ecs-public-rt},{Key=Project,Value=${PROJECT_TAG}}]" \
  --query RouteTable.RouteTableId --output text)

aws ec2 create-route \
  --route-table-id ${PUBLIC_RT} \
  --destination-cidr-block 0.0.0.0/0 \
  --gateway-id ${IGW_ID}

aws ec2 associate-route-table --route-table-id ${PUBLIC_RT} --subnet-id ${PUBLIC_SUBNET_1}
aws ec2 associate-route-table --route-table-id ${PUBLIC_RT} --subnet-id ${PUBLIC_SUBNET_2}
echo "公有路由表配置完成"
```

**预期输出**：打印"公有路由表配置完成"

### 7. 创建安全组

```bash
ALB_SG_ID=$(aws ec2 create-security-group \
  --group-name demo-ecs-alb-sg \
  --description "ALB security group for demo-ecs" \
  --vpc-id ${VPC_ID} \
  --tag-specifications "ResourceType=security-group,Tags=[{Key=Name,Value=demo-ecs-alb-sg},{Key=Project,Value=${PROJECT_TAG}}]" \
  --query GroupId --output text)

aws ec2 authorize-security-group-ingress \
  --group-id ${ALB_SG_ID} \
  --protocol tcp \
  --port 8080 \
  --cidr 0.0.0.0/0

TASK_SG_ID=$(aws ec2 create-security-group \
  --group-name demo-ecs-task-sg \
  --description "Task security group for demo-ecs" \
  --vpc-id ${VPC_ID} \
  --tag-specifications "ResourceType=security-group,Tags=[{Key=Name,Value=demo-ecs-task-sg},{Key=Project,Value=${PROJECT_TAG}}]" \
  --query GroupId --output text)

aws ec2 authorize-security-group-ingress \
  --group-id ${TASK_SG_ID} \
  --protocol tcp \
  --port 80 \
  --source-group ${ALB_SG_ID}

echo "ALB SG: ${ALB_SG_ID}"
echo "Task SG: ${TASK_SG_ID}"
```

**预期输出**：打印两个安全组 ID

### 8. 创建 ECS Cluster 和 Log Group

```bash
aws ecs create-cluster \
  --cluster-name ${CLUSTER_NAME} \
  --settings name=containerInsights,value=enabled \
  --tags key=Project,value=${PROJECT_TAG} key=Owner,value=${OWNER_TAG}

aws logs create-log-group \
  --log-group-name ${LOG_GROUP} \
  --tags Project=${PROJECT_TAG}

echo "ECS Cluster 和 Log Group 已创建"
```

**预期输出**：打印"ECS Cluster 和 Log Group 已创建"

### 9. 创建 IAM Role

```bash
cat > /tmp/ecs-trust.json << 'EOF'
{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"ecs-tasks.amazonaws.com"},"Action":"sts:AssumeRole"}]}
EOF

aws iam create-role \
  --role-name demo-ecs-task-execution-role \
  --assume-role-policy-document file:///tmp/ecs-trust.json \
  --tags Key=Project,Value=${PROJECT_TAG}

aws iam attach-role-policy \
  --role-name demo-ecs-task-execution-role \
  --policy-arn arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy

EXECUTION_ROLE_ARN=$(aws iam get-role \
  --role-name demo-ecs-task-execution-role \
  --query Role.Arn --output text)

aws iam create-role \
  --role-name demo-ecs-task-role \
  --assume-role-policy-document file:///tmp/ecs-trust.json \
  --tags Key=Project,Value=${PROJECT_TAG}

TASK_ROLE_ARN=$(aws iam get-role \
  --role-name demo-ecs-task-role \
  --query Role.Arn --output text)

echo "Execution Role: ${EXECUTION_ROLE_ARN}"
echo "Task Role: ${TASK_ROLE_ARN}"
```

**预期输出**：打印两个 Role ARN

### 10. 保存环境变量

```bash
cat > /tmp/demo-ecs.env << EOF
export AWS_REGION=${AWS_REGION}
export AWS_DEFAULT_REGION=${AWS_REGION}
export CLUSTER_NAME=${CLUSTER_NAME}
export PROJECT_TAG=${PROJECT_TAG}
export OWNER_TAG=${OWNER_TAG}
export ACCOUNT_ID=${ACCOUNT_ID}
export ECR_REGISTRY=${ECR_REGISTRY}
export LOG_GROUP=${LOG_GROUP}
export VPC_ID=${VPC_ID}
export IGW_ID=${IGW_ID}
export PUBLIC_SUBNET_1=${PUBLIC_SUBNET_1}
export PUBLIC_SUBNET_2=${PUBLIC_SUBNET_2}
export PRIVATE_SUBNET_1=${PRIVATE_SUBNET_1}
export PRIVATE_SUBNET_2=${PRIVATE_SUBNET_2}
export ALB_SG_ID=${ALB_SG_ID}
export TASK_SG_ID=${TASK_SG_ID}
export EXECUTION_ROLE_ARN=${EXECUTION_ROLE_ARN}
export TASK_ROLE_ARN=${TASK_ROLE_ARN}
export PUBLIC_RT=${PUBLIC_RT}
EOF

echo "环境变量已保存到 /tmp/demo-ecs.env"
echo "后续 Demo 开头执行：source /tmp/demo-ecs.env"
```

**预期输出**：打印"环境变量已保存到 /tmp/demo-ecs.env"

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
| 1 | `aws ecs describe-clusters --clusters demo-ecs --region us-east-1 --query 'clusters[0].status' --output text` | `ACTIVE` |
| 2 | `aws ec2 describe-subnets --filters "Name=vpc-id,Values=$(aws ec2 describe-vpcs --filters Name=tag:Project,Values=ecs-global-quickstart --query 'Vpcs[0].VpcId' --output text)" --query 'length(Subnets)' --output text --region us-east-1` | `4` |
| 3 | `aws iam get-role --role-name demo-ecs-task-execution-role --query 'Role.RoleName' --output text` | `demo-ecs-task-execution-role` |
| 4 | `aws logs describe-log-groups --log-group-name-prefix /ecs/demo-ecs --region us-east-1 --query 'length(logGroups)' --output text` | `1` |

---

## 实验总结

本实验完成了「环境准备与 VPC/集群创建」的全部操作，从资源创建到功能验证形成了完整闭环。通过动手实践，你已掌握了相关组件的核心配置方法和最佳实践。Demo02 将学习 ECR 镜像构建与推送。

---

## 清理

> ⚠️ Demo01 是基础资源，建议在所有 Demo 完成后最后清理。清理 VPC 前必须先删除 ECS service、ALB、target group、VPC endpoint、EFS、Cloud Map 等依赖资源。

```bash
source /tmp/demo-ecs.env

aws ecs delete-cluster --cluster ${CLUSTER_NAME} 2>/dev/null || true
aws logs delete-log-group --log-group-name ${LOG_GROUP} 2>/dev/null || true

aws iam detach-role-policy \
  --role-name demo-ecs-task-execution-role \
  --policy-arn arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy 2>/dev/null || true
aws iam delete-role --role-name demo-ecs-task-execution-role 2>/dev/null || true
aws iam delete-role --role-name demo-ecs-task-role 2>/dev/null || true

aws ec2 delete-security-group --group-id ${TASK_SG_ID} 2>/dev/null || true
aws ec2 delete-security-group --group-id ${ALB_SG_ID} 2>/dev/null || true

aws ec2 delete-subnet --subnet-id ${PUBLIC_SUBNET_1} 2>/dev/null || true
aws ec2 delete-subnet --subnet-id ${PUBLIC_SUBNET_2} 2>/dev/null || true
aws ec2 delete-subnet --subnet-id ${PRIVATE_SUBNET_1} 2>/dev/null || true
aws ec2 delete-subnet --subnet-id ${PRIVATE_SUBNET_2} 2>/dev/null || true

aws ec2 delete-route-table --route-table-id ${PUBLIC_RT} 2>/dev/null || true
aws ec2 detach-internet-gateway --internet-gateway-id ${IGW_ID} --vpc-id ${VPC_ID} 2>/dev/null || true
aws ec2 delete-internet-gateway --internet-gateway-id ${IGW_ID} 2>/dev/null || true
aws ec2 delete-vpc --vpc-id ${VPC_ID} 2>/dev/null || true

rm -f /tmp/demo-ecs.env
echo "清理完成"
```
