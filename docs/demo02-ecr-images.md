# Demo02 — ECR 镜像构建与发布

## 实验简介

本实验将完成「ECR 镜像构建与推送」的全部操作，涵盖资源创建、配置、验证的完整流程。

**实验目标：**
- 掌握 ECR 镜像构建与推送 的核心操作流程
- 理解相关组件的工作原理和配置要点
- 能够独立完成从部署到验证的完整闭环

**实验流程：**
1. 创建 ECR 仓库
2. 登录 ECR
3. 构建并推送 v1 镜像
4. 构建并推送 v2 镜像
5. 配置 ECR Lifecycle Policy
6. 查看镜像列表和扫描结果
7. 保存镜像变量

**预计 AI 执行时长：** 5-8 分钟


## 前提条件

- **工具**：AWS CLI v2、Docker
- **权限**：AdministratorAccess（含 ECR 创建权限）
- **前提**：Demo01 已完成，`/tmp/demo-ecs.env` 存在
- **初始化**：

```bash
source /tmp/demo-ecs.env
export REPO_NAME=demo-ecs-web
export IMAGE_URI=${ECR_REGISTRY}/${REPO_NAME}
```

---

## 步骤

### 1. 创建 ECR 仓库

```bash
aws ecr create-repository \
  --repository-name ${REPO_NAME} \
  --image-scanning-configuration scanOnPush=true \
  --encryption-configuration encryptionType=AES256 \
  --tags Key=Project,Value=${PROJECT_TAG} Key=Demo,Value=Demo02 Key=Owner,Value=${OWNER_TAG} \
  --region ${AWS_REGION}

echo "ECR 仓库已创建：${IMAGE_URI}"
```

**预期输出**：打印 ECR 仓库 URI

### 2. 登录 ECR

```bash
aws ecr get-login-password --region ${AWS_REGION} \
  | docker login --username AWS --password-stdin ${ECR_REGISTRY}

echo "ECR 登录成功"
```

**预期输出**：`Login Succeeded`，打印"ECR 登录成功"

### 3. 构建并推送 v1 镜像

```bash
mkdir -p /tmp/demo-ecs-app
cat > /tmp/demo-ecs-app/index.html << 'EOF'
<!DOCTYPE html>
<html>
<body style="font-family:sans-serif;text-align:center;padding:50px">
  <h1>demo-ecs-web v1</h1>
  <p>ECS Fargate Global Demo</p>
</body>
</html>
EOF

cat > /tmp/demo-ecs-app/Dockerfile << 'EOF'
FROM public.ecr.aws/nginx/nginx:latest
COPY index.html /usr/share/nginx/html/index.html
EXPOSE 80
EOF

cd /tmp/demo-ecs-app
docker build -t ${IMAGE_URI}:v1 .
docker push ${IMAGE_URI}:v1
echo "v1 推送完成"
```

**预期输出**：打印"v1 推送完成"

### 4. 构建并推送 v2 镜像

```bash
cat > /tmp/demo-ecs-app/index.html << 'EOF'
<!DOCTYPE html>
<html>
<body style="font-family:sans-serif;text-align:center;padding:50px;background:#f0f8ff">
  <h1>demo-ecs-web v2</h1>
  <p>ECS Fargate Global Demo — Updated!</p>
</body>
</html>
EOF

docker build -t ${IMAGE_URI}:v2 .
docker push ${IMAGE_URI}:v2

docker pull ${IMAGE_URI}:v1
docker tag ${IMAGE_URI}:v1 ${IMAGE_URI}:stable
docker push ${IMAGE_URI}:stable

echo "v2 和 stable 标签推送完成"
cd -
```

**预期输出**：打印"v2 和 stable 标签推送完成"

### 5. 配置 ECR Lifecycle Policy

```bash
aws ecr put-lifecycle-policy \
  --repository-name ${REPO_NAME} \
  --lifecycle-policy-text '{
    "rules": [
      {
        "rulePriority": 1,
        "description": "Keep last 5 tagged images",
        "selection": {
          "tagStatus": "tagged",
          "tagPrefixList": ["v"],
          "countType": "imageCountMoreThan",
          "countNumber": 5
        },
        "action": {"type": "expire"}
      },
      {
        "rulePriority": 2,
        "description": "Remove untagged images after 1 day",
        "selection": {
          "tagStatus": "untagged",
          "countType": "sinceImagePushed",
          "countUnit": "days",
          "countNumber": 1
        },
        "action": {"type": "expire"}
      }
    ]
  }' \
  --region ${AWS_REGION}

echo "Lifecycle policy 已配置"
```

**预期输出**：打印"Lifecycle policy 已配置"

### 6. 查看镜像列表和扫描结果

```bash
echo "=== ECR 中的镜像 ==="
aws ecr describe-images \
  --repository-name ${REPO_NAME} \
  --region ${AWS_REGION} \
  --query 'imageDetails[*].[imageTags[0],imagePushedAt]' \
  --output table

echo "=== 扫描结果 ==="
aws ecr describe-image-scan-findings \
  --repository-name ${REPO_NAME} \
  --image-id imageTag=v1 \
  --region ${AWS_REGION} \
  --query 'imageScanFindings.findingSeverityCounts' \
  --output json 2>/dev/null || echo "扫描仍在进行中"
```

**预期输出**：显示 v1、v2、stable 三个 tag；扫描结果（可能需等待几分钟）

### 7. 保存镜像变量

```bash
cat >> /tmp/demo-ecs.env << EOF
export REPO_NAME=${REPO_NAME}
export IMAGE_URI=${IMAGE_URI}
export IMAGE_URI_V1=${IMAGE_URI}:v1
export IMAGE_URI_V2=${IMAGE_URI}:v2
export IMAGE_URI_STABLE=${IMAGE_URI}:stable
export IMAGE_TAG=stable
EOF

echo "镜像变量已追加到 /tmp/demo-ecs.env"
```

**预期输出**：打印"镜像变量已追加到 /tmp/demo-ecs.env"

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
| 1 | `aws ecr describe-repositories --repository-names demo-ecs-web --region us-east-1 --query 'repositories[0].repositoryName' --output text` | `demo-ecs-web` |
| 2 | `aws ecr describe-images --repository-name demo-ecs-web --region us-east-1 --query 'length(imageDetails)' --output text` | `3` |
| 3 | `aws ecr describe-images --repository-name demo-ecs-web --region us-east-1 --query 'imageDetails[?imageTags[0]==\`stable\`].imageTags[0]' --output text` | `stable` |
| 4 | `test -f /tmp/demo-ecs.env && echo exists` | `exists` |

---

## 实验总结

本实验完成了「ECR 镜像构建与推送」的全部操作，从资源创建到功能验证形成了完整闭环。通过动手实践，你已掌握了相关组件的核心配置方法和最佳实践。Demo03 将学习创建第一个 Fargate Service。

---

## 清理

```bash
source /tmp/demo-ecs.env

aws ecr delete-repository \
  --repository-name ${REPO_NAME} \
  --force \
  --region ${AWS_REGION} 2>/dev/null || true

rm -rf /tmp/demo-ecs-app
echo "清理完成"
```
