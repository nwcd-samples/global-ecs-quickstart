# Global ECS QuickStart
## AWS ECS Hands-on Lab Collection

基于 ECS Fargate Linux · 全球区（us-east-1）· Prompt 驱动执行

---

## Demo 列表

建议先完成 **基础主线**，再按需要选择 **进阶专题**。正式主线控制在 10 篇；额外生产专题作为可选 Demo。

### 基础主线

#### 1. 基础资源、镜像与第一个服务
- [Demo01 — 准备实验环境与基础资源](docs/demo01-env-setup.md)
- [Demo02 — ECR 镜像构建与发布](docs/demo02-ecr-images.md)
- [Demo03 — 部署第一个 Fargate 服务](docs/demo03-fargate-service.md)
- [Demo04 — 使用 ALB 暴露服务](docs/demo04-alb.md)

#### 2. 发布、权限与弹性
- [Demo05 — 滚动发布、回滚与故障排查](docs/demo05-rolling-deploy.md)
- [Demo06 — 任务配置 Secrets 与 Task Role](docs/demo06-secrets-task-role.md)
- [Demo07 — Service Auto Scaling](docs/demo07-auto-scaling.md)

#### 3. 私有网络、可观测性与 CI/CD
- [Demo08 — 私有子网与 VPC Endpoint](docs/demo08-private-subnet-vpc-endpoint.md)
- [Demo09 — CloudWatch 日志指标与 ECS Exec](docs/demo09-cloudwatch-ecs-exec.md)
- [Demo10 — CodePipeline 自动部署 ECS](docs/demo10-codepipeline-cicd.md)

### 进阶专题

#### 4. 存储与服务通信
- [Demo11 — EFS 持久化存储](docs/demo11-efs-storage.md)
- [Demo12 — Service Discovery 与 Service Connect](docs/demo12-service-discovery.md)

### 可选生产专题

#### 5. 发布治理与成本治理
- [Demo13 — CodeDeploy 蓝绿发布（可选）](docs/demo13-codedeploy-blue-green.md)
- [Demo14 — 成本治理与清理审计（可选）](docs/demo14-cost-audit.md)

---

## 使用方式

1. 先阅读 `demo-docs/DemoXX-*.md`，确认 Demo 目标、约束、主要步骤、验收标准和清理范围
2. 在此目录下打开 Claude Code 或 Kiro，`CLAUDE.md` / `kiro-system-prompt.md` 自动加载全球区 ECS 配置
3. 将对应 `demo-prompts/demoXX-*.md` 的内容粘贴到对话框，由 AI 自主执行
4. Demo01-Demo04 形成最小可访问服务；Demo05-Demo10 补齐发布、权限、伸缩、私有网络、可观测性和 CI/CD
5. Demo11-Demo14 按需执行
6. 每个 Demo 末尾均有清理要求，实验结束后执行以避免持续计费

与中国区版本（`china-ecs-quickstart/`）的差异：
- Region 默认使用 `us-east-1`
- IAM ARN 使用 `arn:aws:` 标准前缀
- ECR registry 为 `${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com`
- 官方镜像、工具和公共源可直接访问
- CodeCommit、CodeBuild、CodePipeline、Secrets Manager、CloudWatch Logs、EFS、Cloud Map 等服务使用全球区 endpoint
- 资源 tag 使用 `Project=ecs-global-quickstart`

---

## 环境要求

| 工具 | 版本 |
|------|------|
| AWS CLI | 2.x |
| Docker | 24.x+ |
| jq | latest |
| session-manager-plugin | ECS Exec Demo 前安装 |
| git | latest |

操作机建议使用 Amazon Linux 2023 EC2，绑定具备 ECS、EC2、IAM、ECR、Logs、ELB、Application Auto Scaling、Secrets Manager、SSM、CodeBuild、CodePipeline、CodeCommit、EFS、Service Discovery 操作权限的 IAM Role。

## License

MIT - see the [LICENSE](LICENSE) file for details.

## 免责声明

本 Workshop 仅供学习和测试用途。执行过程中会创建 AWS 资源并产生费用，请在实验完成后及时清理资源。作者不对因使用本 Workshop 产生的任何费用或损失承担责任。所有命令和配置仅作为示例参考，生产环境使用前请根据实际需求进行安全评估和调整。
