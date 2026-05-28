# Demo10 — CodePipeline 自动部署 ECS

## 实验简介

本实验将完成「CodePipeline CI/CD」的全部操作，涵盖资源创建、配置、验证的完整流程。

**实验目标：**
- 掌握 CodePipeline CI/CD 的核心操作流程
- 理解相关组件的工作原理和配置要点
- 能够独立完成从部署到验证的完整闭环

**实验流程：**
1. 创建 Artifact S3 Bucket
2. 创建 CodeCommit 仓库并提交源码
3. 创建 CodeBuild Role 和 Project
4. 创建 CodePipeline Role 和 Pipeline
5. 等待 Pipeline 首次执行完成

**预计 AI 执行时长：** 10-12 分钟


## 前提条件

- **工具**：AWS CLI v2、Docker、git
- **权限**：AdministratorAccess（含 CodeCommit、CodeBuild、CodePipeline、ECR、ECS、S3、IAM、CloudWatch Logs）
- **前提**：Demo04 已完成，`demo-ecs-web` service 使用 ECR 镜像
- **初始化**：

```bash
source /tmp/demo-ecs.env
export PIPELINE_NAME=demo-ecs-pipeline
export ARTIFACT_BUCKET=demo-ecs-artifacts-${ACCOUNT_ID}
export CODECOMMIT_REPO=demo-ecs-app
```

---

## 步骤

### 1. 创建 Artifact S3 Bucket

```bash
aws s3 mb s3://${ARTIFACT_BUCKET} --region ${AWS_REGION}
aws s3api put-bucket-versioning \
  --bucket ${ARTIFACT_BUCKET} \
  --versioning-configuration Status=Enabled

echo "Artifact bucket 已创建"
```

**预期输出**：打印"Artifact bucket 已创建"

### 2. 创建 CodeCommit 仓库并提交源码

```bash
aws codecommit create-repository \
  --repository-name ${CODECOMMIT_REPO} \
  --tags Project=${PROJECT_TAG} \
  --region ${AWS_REGION}

COMMIT_URL=$(aws codecommit get-repository \
  --repository-name ${CODECOMMIT_REPO} \
  --region ${AWS_REGION} \
  --query 'repositoryMetadata.cloneUrlHttp' --output text)

mkdir -p /tmp/demo-ecs-cicd
cat > /tmp/demo-ecs-cicd/index.html << 'EOF'
<!DOCTYPE html>
<html>
<body style="text-align:center;padding:50px;font-family:sans-serif">
  <h1>demo-ecs-web — Pipeline v1</h1>
</body>
</html>
EOF

cat > /tmp/demo-ecs-cicd/Dockerfile << 'EOF'
FROM public.ecr.aws/nginx/nginx:latest
COPY index.html /usr/share/nginx/html/index.html
EXPOSE 80
EOF

cat > /tmp/demo-ecs-cicd/buildspec.yml << EOF
version: 0.2
phases:
  pre_build:
    commands:
      - IMAGE_TAG=\$(echo \$CODEBUILD_RESOLVED_SOURCE_VERSION | cut -c 1-7)
      - aws ecr get-login-password --region \$AWS_DEFAULT_REGION | docker login --username AWS --password-stdin ${ECR_REGISTRY}
  build:
    commands:
      - docker build -t ${REPO_NAME}:\$IMAGE_TAG .
      - docker tag ${REPO_NAME}:\$IMAGE_TAG ${IMAGE_URI}:\$IMAGE_TAG
  post_build:
    commands:
      - docker push ${IMAGE_URI}:\$IMAGE_TAG
      - printf '[{"name":"${CONTAINER_NAME}","imageUri":"%s"}]' "${IMAGE_URI}:\$IMAGE_TAG" > imagedefinitions.json
artifacts:
  files:
    - imagedefinitions.json
EOF

git config --global user.email "ecs-lab@example.com" 2>/dev/null || true
git config --global user.name "ECS Lab" 2>/dev/null || true

cd /tmp/demo-ecs-cicd
git init -b main 2>/dev/null || git init
git add .
git commit -m "Initial commit"
git -c credential.helper='!aws codecommit credential-helper $@' \
    -c credential.UseHttpPath=true push -u origin main 2>/dev/null || \
  aws codecommit put-file \
    --repository-name ${CODECOMMIT_REPO} \
    --branch-name main \
    --file-path buildspec.yml \
    --file-content fileb:///tmp/demo-ecs-cicd/buildspec.yml \
    --region ${AWS_REGION} 2>/dev/null || true
cd -

echo "源码已提交到 CodeCommit"
```

**预期输出**：打印"源码已提交到 CodeCommit"

### 3. 创建 CodeBuild Role 和 Project

```bash
cat > /tmp/codebuild-trust.json << 'EOF'
{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"codebuild.amazonaws.com"},"Action":"sts:AssumeRole"}]}
EOF

CODEBUILD_ROLE_ARN=$(aws iam create-role \
  --role-name demo-ecs-codebuild-role \
  --assume-role-policy-document file:///tmp/codebuild-trust.json \
  --tags Key=Project,Value=${PROJECT_TAG} \
  --query Role.Arn --output text)

aws iam attach-role-policy \
  --role-name demo-ecs-codebuild-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryPowerUser

cat > /tmp/codebuild-policy.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {"Effect":"Allow","Action":["logs:CreateLogGroup","logs:CreateLogStream","logs:PutLogEvents"],"Resource":"*"},
    {"Effect":"Allow","Action":["s3:GetObject","s3:PutObject","s3:GetBucketVersioning"],"Resource":["arn:aws:s3:::${ARTIFACT_BUCKET}","arn:aws:s3:::${ARTIFACT_BUCKET}/*"]}
  ]
}
EOF

aws iam put-role-policy \
  --role-name demo-ecs-codebuild-role \
  --policy-name demo-ecs-codebuild-policy \
  --policy-document file:///tmp/codebuild-policy.json

aws codebuild create-project \
  --name demo-ecs-build \
  --source "type=CODECOMMIT,location=${COMMIT_URL}" \
  --artifacts "type=S3,location=${ARTIFACT_BUCKET},name=build-output,packaging=ZIP" \
  --environment "type=LINUX_CONTAINER,image=aws/codebuild/standard:7.0,computeType=BUILD_GENERAL1_SMALL,privilegedMode=true" \
  --service-role ${CODEBUILD_ROLE_ARN} \
  --tags key=Project,value=${PROJECT_TAG} \
  --region ${AWS_REGION}

echo "CodeBuild Project 已创建"
```

**预期输出**：打印"CodeBuild Project 已创建"

### 4. 创建 CodePipeline Role 和 Pipeline

```bash
cat > /tmp/pipeline-trust.json << 'EOF'
{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"codepipeline.amazonaws.com"},"Action":"sts:AssumeRole"}]}
EOF

PIPELINE_ROLE_ARN=$(aws iam create-role \
  --role-name demo-ecs-pipeline-role \
  --assume-role-policy-document file:///tmp/pipeline-trust.json \
  --tags Key=Project,Value=${PROJECT_TAG} \
  --query Role.Arn --output text)

cat > /tmp/pipeline-policy.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {"Effect":"Allow","Action":["codecommit:GetBranch","codecommit:GetCommit","codecommit:UploadArchive","codecommit:GetUploadArchiveStatus"],"Resource":"arn:aws:codecommit:${AWS_REGION}:${ACCOUNT_ID}:${CODECOMMIT_REPO}"},
    {"Effect":"Allow","Action":["codebuild:StartBuild","codebuild:BatchGetBuilds"],"Resource":"arn:aws:codebuild:${AWS_REGION}:${ACCOUNT_ID}:project/demo-ecs-build"},
    {"Effect":"Allow","Action":["ecs:DescribeServices","ecs:DescribeTaskDefinition","ecs:RegisterTaskDefinition","ecs:UpdateService"],"Resource":"*"},
    {"Effect":"Allow","Action":["iam:PassRole"],"Resource":["${EXECUTION_ROLE_ARN}","${TASK_ROLE_ARN}"]},
    {"Effect":"Allow","Action":["s3:GetObject","s3:PutObject","s3:GetBucketVersioning","s3:GetObjectVersion"],"Resource":["arn:aws:s3:::${ARTIFACT_BUCKET}","arn:aws:s3:::${ARTIFACT_BUCKET}/*"]}
  ]
}
EOF

aws iam put-role-policy \
  --role-name demo-ecs-pipeline-role \
  --policy-name demo-ecs-pipeline-policy \
  --policy-document file:///tmp/pipeline-policy.json

cat > /tmp/pipeline-def.json << EOF
{
  "name": "${PIPELINE_NAME}",
  "roleArn": "${PIPELINE_ROLE_ARN}",
  "artifactStore": {"type":"S3","location":"${ARTIFACT_BUCKET}"},
  "stages": [
    {
      "name": "Source",
      "actions": [{
        "name": "Source",
        "actionTypeId": {"category":"Source","owner":"AWS","provider":"CodeCommit","version":"1"},
        "configuration": {"RepositoryName":"${CODECOMMIT_REPO}","BranchName":"main"},
        "outputArtifacts": [{"name":"SourceOutput"}]
      }]
    },
    {
      "name": "Build",
      "actions": [{
        "name": "Build",
        "actionTypeId": {"category":"Build","owner":"AWS","provider":"CodeBuild","version":"1"},
        "configuration": {"ProjectName":"demo-ecs-build"},
        "inputArtifacts": [{"name":"SourceOutput"}],
        "outputArtifacts": [{"name":"BuildOutput"}]
      }]
    },
    {
      "name": "Deploy",
      "actions": [{
        "name": "Deploy",
        "actionTypeId": {"category":"Deploy","owner":"AWS","provider":"ECS","version":"1"},
        "configuration": {"ClusterName":"${CLUSTER_NAME}","ServiceName":"${SERVICE_NAME}","FileName":"imagedefinitions.json"},
        "inputArtifacts": [{"name":"BuildOutput"}]
      }]
    }
  ]
}
EOF

aws codepipeline create-pipeline \
  --pipeline file:///tmp/pipeline-def.json \
  --region ${AWS_REGION}

echo "Pipeline 已创建，等待首次执行（约 5-10 分钟）..."
```

**预期输出**：打印"Pipeline 已创建"

### 5. 等待 Pipeline 首次执行完成

```bash
for i in $(seq 1 20); do
  STATUS=$(aws codepipeline get-pipeline-state \
    --name ${PIPELINE_NAME} \
    --region ${AWS_REGION} \
    --query 'stageStates[?stageName==`Deploy`].latestExecution.status' \
    --output text 2>/dev/null)
  echo "$(date +%H:%M:%S) Deploy 状态: ${STATUS}"
  [[ "${STATUS}" == "Succeeded" ]] && break
  [[ "${STATUS}" == "Failed" ]] && { echo "Pipeline 失败，请检查 CodeBuild 日志"; break; }
  sleep 30
done

echo "=== ALB 验证 ==="
curl -s --max-time 10 http://${ALB_DNS}:8080/ | head -5
# 注意：Pipeline 部署的镜像内容取决于 buildspec.yml 中构建的 index.html
# 若 Pipeline 使用 imagedefinitions.json 更新镜像，ALB 返回内容为新构建镜像的页面
```

**预期输出**：Deploy 状态 `Succeeded`；ALB 返回 `Pipeline v1`

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
| 1 | `aws codepipeline get-pipeline-state --name demo-ecs-pipeline --region us-east-1 --query 'stageStates[?stageName==\`Deploy\`].latestExecution.status' --output text` | `Succeeded` |
| 2 | `aws codebuild list-builds-for-project --project-name demo-ecs-build --region us-east-1 --query 'length(ids)' --output text` | 大于 `0` |
| 3 | `aws codecommit list-repositories --region us-east-1 --query 'repositories[?repositoryName==\`demo-ecs-app\`].repositoryName' --output text` | `demo-ecs-app` |
| 4 | `aws ecs describe-services --cluster demo-ecs --services demo-ecs-web --region us-east-1 --query 'services[0].runningCount' --output text` | `2` |

---

## 实验总结

本实验完成了「CodePipeline CI/CD」的全部操作，从资源创建到功能验证形成了完整闭环。通过动手实践，你已掌握了相关组件的核心配置方法和最佳实践。Demo11 将学习 EFS 持久化存储。

---

## 清理

```bash
source /tmp/demo-ecs.env

aws codepipeline delete-pipeline --name ${PIPELINE_NAME} --region ${AWS_REGION} 2>/dev/null || true
aws codebuild delete-project --name demo-ecs-build --region ${AWS_REGION} 2>/dev/null || true
aws codecommit delete-repository --repository-name ${CODECOMMIT_REPO} --region ${AWS_REGION} 2>/dev/null || true

aws s3 rm s3://${ARTIFACT_BUCKET} --recursive 2>/dev/null || true
aws s3 rb s3://${ARTIFACT_BUCKET} 2>/dev/null || true

aws iam delete-role-policy --role-name demo-ecs-codebuild-role --policy-name demo-ecs-codebuild-policy 2>/dev/null || true
aws iam detach-role-policy --role-name demo-ecs-codebuild-role --policy-arn arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryPowerUser 2>/dev/null || true
aws iam delete-role --role-name demo-ecs-codebuild-role 2>/dev/null || true

aws iam delete-role-policy --role-name demo-ecs-pipeline-role --policy-name demo-ecs-pipeline-policy 2>/dev/null || true
aws iam delete-role --role-name demo-ecs-pipeline-role 2>/dev/null || true

rm -rf /tmp/demo-ecs-cicd
echo "清理完成"
```
