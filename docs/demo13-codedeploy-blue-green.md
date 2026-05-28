# Demo13 — CodeDeploy 蓝绿发布（可选）

## 实验简介

本实验将完成「CodeDeploy Blue/Green 部署」的全部操作，涵盖资源创建、配置、验证的完整流程。

**实验目标：**
- 掌握 CodeDeploy Blue/Green 部署 的核心操作流程
- 理解相关组件的工作原理和配置要点
- 能够独立完成从部署到验证的完整闭环

**实验流程：**
1. 创建 Blue 和 Green Target Groups
2. 在 ALB 上添加测试流量 Listener（端口 8081）
3. 创建 CodeDeploy Service Role
4. 注册 Blue-Green Task Definition（v1）
5. 创建 Blue-Green ECS Service
6. 创建 CodeDeploy Application 和 Deployment Group
7. 注册 Task Definition v2（新版本）
8. 准备 AppSpec 并发起蓝绿部署
9. 观察部署进度

**预计 AI 执行时长：** 10-12 分钟


## 前提条件

- **工具**：AWS CLI v2
- **权限**：AdministratorAccess（含 ECS、CodeDeploy、ELBv2、IAM、ECR、CloudWatch Logs）
- **前提**：Demo01-Demo04 已完成，ALB 和 ECR 镜像已就绪
- **隔离说明**：本 Demo 使用独立 service `demo-ecs-bg`，不影响主线 `demo-ecs-web`
- **重要约束**：ECS service 的 deployment controller 创建后不能改回 rolling，需重建 service
- **初始化**：

```bash
source /tmp/demo-ecs.env
export BG_SERVICE_NAME=demo-ecs-bg
export BG_BLUE_TG=demo-web-blue
export BG_GREEN_TG=demo-web-green
export BG_TEST_PORT=8081
export CD_APP_NAME=demo-ecs-app
export CD_DG_NAME=demo-ecs-dg
```

---

## 步骤

### 1. 创建 Blue 和 Green Target Groups

```bash
BLUE_TG_ARN=$(aws elbv2 create-target-group \
  --name ${BG_BLUE_TG} \
  --protocol HTTP \
  --port 80 \
  --target-type ip \
  --vpc-id ${VPC_ID} \
  --health-check-path "/" \
  --health-check-interval-seconds 15 \
  --healthy-threshold-count 2 \
  --tags Key=Project,Value=${PROJECT_TAG} Key=Demo,Value=Demo13 \
  --region ${AWS_REGION} \
  --query 'TargetGroups[0].TargetGroupArn' --output text)

GREEN_TG_ARN=$(aws elbv2 create-target-group \
  --name ${BG_GREEN_TG} \
  --protocol HTTP \
  --port 80 \
  --target-type ip \
  --vpc-id ${VPC_ID} \
  --health-check-path "/" \
  --health-check-interval-seconds 15 \
  --healthy-threshold-count 2 \
  --tags Key=Project,Value=${PROJECT_TAG} Key=Demo,Value=Demo13 \
  --region ${AWS_REGION} \
  --query 'TargetGroups[0].TargetGroupArn' --output text)

echo "Blue TG: ${BLUE_TG_ARN}"
echo "Green TG: ${GREEN_TG_ARN}"
```

**预期输出**：打印 Blue 和 Green Target Group ARN

### 2. 在 ALB 上添加测试流量 Listener（端口 8081）

```bash
TEST_LISTENER_ARN=$(aws elbv2 create-listener \
  --load-balancer-arn ${ALB_ARN} \
  --protocol HTTP \
  --port ${BG_TEST_PORT} \
  --default-actions "Type=forward,TargetGroupArn=${GREEN_TG_ARN}" \
  --tags Key=Project,Value=${PROJECT_TAG} \
  --region ${AWS_REGION} \
  --query 'Listeners[0].ListenerArn' --output text)

echo "Test Listener (port ${BG_TEST_PORT}): ${TEST_LISTENER_ARN}"
echo "Production Listener (port 8080): ${LISTENER_ARN}"
```

**预期输出**：打印测试 Listener ARN

### 3. 创建 CodeDeploy Service Role

```bash
cat > /tmp/codedeploy-trust.json << 'EOF'
{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"codedeploy.amazonaws.com"},"Action":"sts:AssumeRole"}]}
EOF

CD_ROLE_ARN=$(aws iam create-role \
  --role-name demo-ecs-codedeploy-role \
  --assume-role-policy-document file:///tmp/codedeploy-trust.json \
  --tags Key=Project,Value=${PROJECT_TAG} \
  --query Role.Arn --output text)

aws iam attach-role-policy \
  --role-name demo-ecs-codedeploy-role \
  --policy-arn arn:aws:iam::aws:policy/AWSCodeDeployRoleForECS

echo "CodeDeploy Role: ${CD_ROLE_ARN}"
echo "等待 IAM 传播..."
sleep 15
```

**预期输出**：打印 CodeDeploy Role ARN

### 4. 注册 Blue-Green Task Definition（v1）

```bash
TASK_DEF_BG_V1=$(aws ecs register-task-definition \
  --family demo-ecs-bg-web \
  --network-mode awsvpc \
  --requires-compatibilities FARGATE \
  --cpu 256 \
  --memory 512 \
  --execution-role-arn ${EXECUTION_ROLE_ARN} \
  --task-role-arn ${TASK_ROLE_ARN} \
  --container-definitions "[
    {
      \"name\": \"web\",
      \"image\": \"${IMAGE_URI}:v1\",
      \"essential\": true,
      \"portMappings\": [{\"containerPort\": 80, \"protocol\": \"tcp\"}],
      \"logConfiguration\": {
        \"logDriver\": \"awslogs\",
        \"options\": {
          \"awslogs-group\": \"${LOG_GROUP}\",
          \"awslogs-region\": \"${AWS_REGION}\",
          \"awslogs-stream-prefix\": \"bg\"
        }
      }
    }
  ]" \
  --region ${AWS_REGION} \
  --query taskDefinition.taskDefinitionArn --output text)

echo "BG Task Definition v1: ${TASK_DEF_BG_V1}"
```

**预期输出**：打印 BG Task Definition v1 ARN

### 5. 创建 Blue-Green ECS Service

```bash
aws ecs create-service \
  --cluster ${CLUSTER_NAME} \
  --service-name ${BG_SERVICE_NAME} \
  --task-definition demo-ecs-bg-web \
  --launch-type FARGATE \
  --desired-count 1 \
  --network-configuration "awsvpcConfiguration={subnets=[${PUBLIC_SUBNET_1},${PUBLIC_SUBNET_2}],securityGroups=[${TASK_SG_ID}],assignPublicIp=ENABLED}" \
  --load-balancers "[{\"targetGroupArn\":\"${BLUE_TG_ARN}\",\"containerName\":\"web\",\"containerPort\":80}]" \
  --deployment-controller '{"type":"CODE_DEPLOY"}' \
  --region ${AWS_REGION}

aws ecs wait services-stable \
  --cluster ${CLUSTER_NAME} \
  --services ${BG_SERVICE_NAME} \
  --region ${AWS_REGION}

echo "Blue-Green service 已创建并稳定"
```

**预期输出**：BG service 稳定，1 个 task 运行中

### 6. 创建 CodeDeploy Application 和 Deployment Group

```bash
aws deploy create-application \
  --application-name ${CD_APP_NAME} \
  --compute-platform ECS \
  --region ${AWS_REGION}

aws deploy create-deployment-group \
  --application-name ${CD_APP_NAME} \
  --deployment-group-name ${CD_DG_NAME} \
  --service-role-arn ${CD_ROLE_ARN} \
  --deployment-config-name CodeDeployDefault.ECSAllAtOnce \
  --deployment-style '{"deploymentType":"BLUE_GREEN","deploymentOption":"WITH_TRAFFIC_CONTROL"}' \
  --ecs-services "[{\"serviceName\":\"${BG_SERVICE_NAME}\",\"clusterName\":\"${CLUSTER_NAME}\"}]" \
  --load-balancer-info "{
    \"targetGroupPairInfoList\": [{
      \"targetGroups\": [
        {\"name\":\"${BG_BLUE_TG}\"},
        {\"name\":\"${BG_GREEN_TG}\"}
      ],
      \"prodTrafficRoute\": {\"listenerArns\":[\"${LISTENER_ARN}\"]},
      \"testTrafficRoute\": {\"listenerArns\":[\"${TEST_LISTENER_ARN}\"]}
    }]
  }" \
  --blue-green-deployment-configuration "{
    \"terminateBlueInstancesOnDeploymentSuccess\": {
      \"action\": \"TERMINATE\",
      \"terminationWaitTimeInMinutes\": 5
    },
    \"deploymentReadyOption\": {\"actionOnTimeout\": \"CONTINUE_DEPLOYMENT\"}
  }" \
  --auto-rollback-configuration "enabled=true,events=DEPLOYMENT_FAILURE" \
  --region ${AWS_REGION}

echo "CodeDeploy Application 和 Deployment Group 已创建"
```

**预期输出**：打印"CodeDeploy Application 和 Deployment Group 已创建"

### 7. 注册 Task Definition v2（新版本）

```bash
TASK_DEF_BG_V2=$(aws ecs register-task-definition \
  --family demo-ecs-bg-web \
  --network-mode awsvpc \
  --requires-compatibilities FARGATE \
  --cpu 256 \
  --memory 512 \
  --execution-role-arn ${EXECUTION_ROLE_ARN} \
  --task-role-arn ${TASK_ROLE_ARN} \
  --container-definitions "[
    {
      \"name\": \"web\",
      \"image\": \"${IMAGE_URI}:v2\",
      \"essential\": true,
      \"portMappings\": [{\"containerPort\": 80, \"protocol\": \"tcp\"}],
      \"logConfiguration\": {
        \"logDriver\": \"awslogs\",
        \"options\": {
          \"awslogs-group\": \"${LOG_GROUP}\",
          \"awslogs-region\": \"${AWS_REGION}\",
          \"awslogs-stream-prefix\": \"bg\"
        }
      }
    }
  ]" \
  --region ${AWS_REGION} \
  --query taskDefinition.taskDefinitionArn --output text)

echo "BG Task Definition v2: ${TASK_DEF_BG_V2}"
```

**预期输出**：打印 BG Task Definition v2 ARN

### 8. 准备 AppSpec 并发起蓝绿部署

```bash
cat > /tmp/appspec.json << EOF
{
  "version": 0.0,
  "Resources": [{
    "TargetService": {
      "Type": "AWS::ECS::Service",
      "Properties": {
        "TaskDefinition": "${TASK_DEF_BG_V2}",
        "LoadBalancerInfo": {
          "ContainerName": "web",
          "ContainerPort": 80
        }
      }
    }
  }]
}
EOF

DEPLOYMENT_ID=$(aws deploy create-deployment \
  --application-name ${CD_APP_NAME} \
  --deployment-group-name ${CD_DG_NAME} \
  --revision "revisionType=AppSpecContent,appSpecContent={content='$(cat /tmp/appspec.json | tr -d '\n')'}" \
  --region ${AWS_REGION} \
  --query deploymentId --output text)

echo "Deployment ID: ${DEPLOYMENT_ID}"
echo "等待蓝绿部署完成（约 3-5 分钟）..."
```

**预期输出**：打印 Deployment ID

### 9. 观察部署进度

```bash
for i in $(seq 1 20); do
  STATUS=$(aws deploy get-deployment \
    --deployment-id ${DEPLOYMENT_ID} \
    --region ${AWS_REGION} \
    --query 'deploymentInfo.status' --output text)
  echo "$(date +%H:%M:%S) 部署状态: ${STATUS}"
  [[ "${STATUS}" == "Succeeded" ]] && break
  [[ "${STATUS}" == "Failed" || "${STATUS}" == "Stopped" ]] && {
    echo "部署失败，查看详情："
    aws deploy get-deployment \
      --deployment-id ${DEPLOYMENT_ID} \
      --region ${AWS_REGION} \
      --query 'deploymentInfo.errorInformation'
    break
  }
  sleep 15
done

echo ""
echo "=== 验证 ALB 生产流量（端口 8080）==="
curl -s --max-time 10 http://${ALB_DNS}:8080/ | head -5

echo ""
echo "=== 验证 Green Target Group 健康状态 ==="
aws elbv2 describe-target-health \
  --target-group-arn ${GREEN_TG_ARN} \
  --region ${AWS_REGION} \
  --query 'TargetHealthDescriptions[*].{IP:Target.Id,State:TargetHealth.State}' \
  --output table
```

**预期输出**：部署状态 `Succeeded`；ALB 返回 v2 内容；Green target group 健康状态为 `healthy`

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
| 1 | `aws deploy get-deployment --deployment-id $(aws deploy list-deployments --application-name demo-ecs-app --region us-east-1 --query 'deployments[0]' --output text) --region us-east-1 --query 'deploymentInfo.status' --output text` | `Succeeded` |
| 2 | `aws ecs describe-services --cluster demo-ecs --services demo-ecs-bg --region us-east-1 --query 'services[0].deploymentController.type' --output text` | `CODE_DEPLOY` |
| 3 | `aws elbv2 describe-target-groups --names demo-web-blue demo-web-green --region us-east-1 --query 'length(TargetGroups)' --output text` | `2` |
| 4 | `aws ecs describe-services --cluster demo-ecs --services demo-ecs-web --region us-east-1 --query 'services[0].runningCount' --output text` | `2` |

---

## 实验总结

本实验完成了「CodeDeploy Blue/Green 部署」的全部操作，从资源创建到功能验证形成了完整闭环。通过动手实践，你已掌握了相关组件的核心配置方法和最佳实践。Demo14 将学习成本审计与资源清理。

---

## 清理

```bash
source /tmp/demo-ecs.env

aws deploy delete-deployment-group \
  --application-name ${CD_APP_NAME} \
  --deployment-group-name ${CD_DG_NAME} \
  --region ${AWS_REGION} 2>/dev/null || true

aws deploy delete-application \
  --application-name ${CD_APP_NAME} \
  --region ${AWS_REGION} 2>/dev/null || true

aws ecs update-service \
  --cluster ${CLUSTER_NAME} \
  --service ${BG_SERVICE_NAME} \
  --desired-count 0 \
  --region ${AWS_REGION} 2>/dev/null || true

sleep 15

aws ecs delete-service \
  --cluster ${CLUSTER_NAME} \
  --service ${BG_SERVICE_NAME} \
  --force \
  --region ${AWS_REGION} 2>/dev/null || true

aws ecs deregister-task-definition \
  --task-definition demo-ecs-bg-web:1 \
  --region ${AWS_REGION} 2>/dev/null || true

aws ecs deregister-task-definition \
  --task-definition demo-ecs-bg-web:2 \
  --region ${AWS_REGION} 2>/dev/null || true

TEST_LISTENER_ARN=$(aws elbv2 describe-listeners \
  --load-balancer-arn ${ALB_ARN} \
  --region ${AWS_REGION} \
  --query "Listeners[?Port==\`8081\`].ListenerArn" --output text 2>/dev/null)
[ -n "${TEST_LISTENER_ARN}" ] && aws elbv2 delete-listener \
  --listener-arn ${TEST_LISTENER_ARN} \
  --region ${AWS_REGION} 2>/dev/null || true

for TG in demo-web-blue demo-web-green; do
  TG_ARN=$(aws elbv2 describe-target-groups \
    --names ${TG} \
    --region ${AWS_REGION} \
    --query 'TargetGroups[0].TargetGroupArn' --output text 2>/dev/null)
  [ -n "${TG_ARN}" ] && aws elbv2 delete-target-group \
    --target-group-arn ${TG_ARN} \
    --region ${AWS_REGION} 2>/dev/null || true
done

aws iam detach-role-policy \
  --role-name demo-ecs-codedeploy-role \
  --policy-arn arn:aws:iam::aws:policy/AWSCodeDeployRoleForECS 2>/dev/null || true

aws iam delete-role \
  --role-name demo-ecs-codedeploy-role 2>/dev/null || true

echo "清理完成"
```
