# Demo11 — EFS 持久化存储

## 实验简介

本实验将完成「EFS 持久化存储」的全部操作，涵盖资源创建、配置、验证的完整流程。

**实验目标：**
- 掌握 EFS 持久化存储 的核心操作流程
- 理解相关组件的工作原理和配置要点
- 能够独立完成从部署到验证的完整闭环

**实验流程：**
1. 创建 EFS 安全组
2. 创建 EFS File System
3. 创建 Mount Targets（2 个 AZ）
4. 注册带 EFS Volume 的 Task Definition
5. 运行 Writer Task（写入文件）
6. 运行 Reader Task（读取同一文件）

**预计 AI 执行时长：** 8-10 分钟


## 前提条件

- **工具**：AWS CLI v2
- **权限**：AdministratorAccess（含 EFS、ECS、EC2、IAM、CloudWatch Logs）
- **前提**：Demo01 已完成，VPC 和子网已创建
- **成本提示**：EFS 持续计费，演示后及时删除
- **初始化**：

```bash
source /tmp/demo-ecs.env
```

---

## 步骤

### 1. 创建 EFS 安全组

```bash
EFS_SG_ID=$(aws ec2 create-security-group \
  --group-name demo-ecs-efs-sg \
  --description "EFS security group for demo-ecs" \
  --vpc-id ${VPC_ID} \
  --tag-specifications "ResourceType=security-group,Tags=[{Key=Name,Value=demo-ecs-efs-sg},{Key=Project,Value=${PROJECT_TAG}}]" \
  --query GroupId --output text)

aws ec2 authorize-security-group-ingress \
  --group-id ${EFS_SG_ID} \
  --protocol tcp \
  --port 2049 \
  --source-group ${TASK_SG_ID}

echo "EFS SG: ${EFS_SG_ID}"
```

**预期输出**：打印 EFS 安全组 ID

### 2. 创建 EFS File System

```bash
EFS_ID=$(aws efs create-file-system \
  --encrypted \
  --performance-mode generalPurpose \
  --throughput-mode bursting \
  --tags Key=Name,Value=demo-ecs-efs Key=Project,Value=${PROJECT_TAG} Key=Demo,Value=Demo11 \
  --region ${AWS_REGION} \
  --query FileSystemId --output text)

echo "EFS File System ID: ${EFS_ID}"

until [ "$(aws efs describe-file-systems \
  --file-system-id ${EFS_ID} \
  --region ${AWS_REGION} \
  --query 'FileSystems[0].LifeCycleState' --output text)" = "available" ]; do
  echo "等待 EFS available..."
  sleep 5
done
echo "EFS available"
```

**预期输出**：打印 EFS ID；状态变为 `available`

### 3. 创建 Mount Targets（2 个 AZ）

```bash
MT_1=$(aws efs create-mount-target \
  --file-system-id ${EFS_ID} \
  --subnet-id ${PUBLIC_SUBNET_1} \
  --security-groups ${EFS_SG_ID} \
  --region ${AWS_REGION} \
  --query MountTargetId --output text)

MT_2=$(aws efs create-mount-target \
  --file-system-id ${EFS_ID} \
  --subnet-id ${PUBLIC_SUBNET_2} \
  --security-groups ${EFS_SG_ID} \
  --region ${AWS_REGION} \
  --query MountTargetId --output text)

echo "等待 Mount Targets available..."
for i in $(seq 1 20); do
  MT_STATES=$(aws efs describe-mount-targets \
    --file-system-id ${EFS_ID} \
    --region ${AWS_REGION} \
    --query 'MountTargets[*].LifeCycleState' \
    --output text)
  echo "Mount Target 状态: ${MT_STATES}"
  echo "${MT_STATES}" | grep -v available | wc -w | grep -q "^0$" && break
  sleep 15
done

echo "Mount Targets: ${MT_1}, ${MT_2}"
```

**预期输出**：两个 Mount Target 均为 `available`

### 4. 注册带 EFS Volume 的 Task Definition

```bash
TASK_DEF_EFS=$(aws ecs register-task-definition \
  --family demo-ecs-efs-demo \
  --network-mode awsvpc \
  --requires-compatibilities FARGATE \
  --cpu 256 \
  --memory 512 \
  --execution-role-arn ${EXECUTION_ROLE_ARN} \
  --task-role-arn ${TASK_ROLE_ARN} \
  --volumes "[
    {
      \"name\": \"shared-data\",
      \"efsVolumeConfiguration\": {
        \"fileSystemId\": \"${EFS_ID}\",
        \"rootDirectory\": \"/\",
        \"transitEncryption\": \"ENABLED\"
      }
    }
  ]" \
  --container-definitions "[
    {
      \"name\": \"efs-demo\",
      \"image\": \"public.ecr.aws/docker/library/busybox:latest\",
      \"essential\": true,
      \"mountPoints\": [{
        \"sourceVolume\": \"shared-data\",
        \"containerPath\": \"/mnt/data\",
        \"readOnly\": false
      }],
      \"logConfiguration\": {
        \"logDriver\": \"awslogs\",
        \"options\": {
          \"awslogs-group\": \"${LOG_GROUP}\",
          \"awslogs-region\": \"${AWS_REGION}\",
          \"awslogs-stream-prefix\": \"efs\"
        }
      }
    }
  ]" \
  --region ${AWS_REGION} \
  --query taskDefinition.taskDefinitionArn --output text)

echo "EFS Task Definition: ${TASK_DEF_EFS}"
```

**预期输出**：打印 EFS Task Definition ARN

### 5. 运行 Writer Task（写入文件）

```bash
WRITE_CMD="echo 'Hello from ECS Fargate EFS Demo' > /mnt/data/demo.txt && cat /mnt/data/demo.txt && echo 'Write complete'"

WRITER_TASK=$(aws ecs run-task \
  --cluster ${CLUSTER_NAME} \
  --task-definition demo-ecs-efs-demo \
  --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={subnets=[${PUBLIC_SUBNET_1}],securityGroups=[${TASK_SG_ID}],assignPublicIp=ENABLED}" \
  --overrides "{\"containerOverrides\":[{\"name\":\"efs-demo\",\"command\":[\"sh\",\"-c\",\"${WRITE_CMD}\"]}]}" \
  --region ${AWS_REGION} \
  --query 'tasks[0].taskArn' --output text)

echo "Writer task: ${WRITER_TASK}"

aws ecs wait tasks-stopped \
  --cluster ${CLUSTER_NAME} \
  --tasks ${WRITER_TASK} \
  --region ${AWS_REGION}

echo "=== Writer Task 日志 ==="
sleep 5
aws logs tail ${LOG_GROUP} --since 3m --region ${AWS_REGION} 2>/dev/null | grep "efs/" | head -5
```

**预期输出**：日志中显示 `Hello from ECS Fargate EFS Demo` 和 `Write complete`

### 6. 运行 Reader Task（读取同一文件）

```bash
READ_CMD="cat /mnt/data/demo.txt && echo 'Read complete'"

READER_TASK=$(aws ecs run-task \
  --cluster ${CLUSTER_NAME} \
  --task-definition demo-ecs-efs-demo \
  --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={subnets=[${PUBLIC_SUBNET_2}],securityGroups=[${TASK_SG_ID}],assignPublicIp=ENABLED}" \
  --overrides "{\"containerOverrides\":[{\"name\":\"efs-demo\",\"command\":[\"sh\",\"-c\",\"${READ_CMD}\"]}]}" \
  --region ${AWS_REGION} \
  --query 'tasks[0].taskArn' --output text)

echo "Reader task: ${READER_TASK}"

aws ecs wait tasks-stopped \
  --cluster ${CLUSTER_NAME} \
  --tasks ${READER_TASK} \
  --region ${AWS_REGION}

echo "=== Reader Task 日志（应包含写入内容）==="
sleep 5
aws logs tail ${LOG_GROUP} --since 3m --region ${AWS_REGION} 2>/dev/null | grep "efs/" | head -5
```

**预期输出**：Reader Task 日志显示 `Hello from ECS Fargate EFS Demo` 和 `Read complete`（跨 task 持久化成功）

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
| 1 | `aws efs describe-file-systems --file-system-id $(aws efs describe-file-systems --region us-east-1 --query 'FileSystems[?Tags[?Key==\`Demo\`&&Value==\`Demo11\`]].FileSystemId' --output text) --region us-east-1 --query 'FileSystems[0].LifeCycleState' --output text` | `available` |
| 2 | `aws efs describe-mount-targets --file-system-id $(aws efs describe-file-systems --region us-east-1 --query 'FileSystems[?Tags[?Key==\`Demo\`&&Value==\`Demo11\`]].FileSystemId' --output text) --region us-east-1 --query 'length(MountTargets[?LifeCycleState==\`available\`])' --output text` | `2` |
| 3 | `aws logs tail /ecs/demo-ecs --since 10m --region us-east-1 2>/dev/null \| grep -c 'Write complete'` | 大于 `0` |
| 4 | `aws logs tail /ecs/demo-ecs --since 10m --region us-east-1 2>/dev/null \| grep -c 'Read complete'` | 大于 `0` |

---

## 实验总结

本实验完成了「EFS 持久化存储」的全部操作，从资源创建到功能验证形成了完整闭环。通过动手实践，你已掌握了相关组件的核心配置方法和最佳实践。Demo12 将学习 Service Discovery 服务发现。

---

## 清理

```bash
source /tmp/demo-ecs.env

aws ecs deregister-task-definition \
  --task-definition demo-ecs-efs-demo:1 \
  --region ${AWS_REGION} 2>/dev/null || true

for MT in $(aws efs describe-mount-targets \
  --file-system-id ${EFS_ID} \
  --region ${AWS_REGION} \
  --query 'MountTargets[*].MountTargetId' --output text 2>/dev/null); do
  aws efs delete-mount-target --mount-target-id ${MT} --region ${AWS_REGION} 2>/dev/null || true
done

echo "等待 Mount Targets 删除（约 30 秒）..."
sleep 30

aws efs delete-file-system --file-system-id ${EFS_ID} --region ${AWS_REGION} 2>/dev/null || true
aws ec2 delete-security-group --group-id ${EFS_SG_ID} --region ${AWS_REGION} 2>/dev/null || true

echo "清理完成"
```
