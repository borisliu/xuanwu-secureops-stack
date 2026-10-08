# ADR-007: Agent 使用短生命周期 Kubernetes Job

- 状态：Accepted
- 日期：2026-10-08

## Decision

每个 Agent 任务以短生命周期 Kubernetes Job 运行，不部署常驻高权限 Agent。

Job 必须具备固定镜像 digest、独立 ServiceAccount、最小 Role、资源限制、执行超时、TTL、Loki 日志和 Git Task 记录。默认只读，L1 仅绑定到明确的白名单动作，L2 由人工审批后才获得短时授权。

## Data Sources

V0.1 Agent Job 直接通过最小权限读取 Kubernetes API、Prometheus、Loki、Harbor API、Git 和 Runbook；不建设 `xw-opsapi` 中间层。V0.2 只有在重复适配或统一授权成为明确问题时才评估聚合服务。

## Example

`da-soc-render` CrashLoopBackOff：告警触发 n8n → 创建 Git Task → 启动 Agent Job → 读取证据 → 判断 L1/L2 → 白名单重启或人工审批 → 健康和业务连通性验证 → 失败 rollout undo/人工升级 → Task 记录审计。

## Prohibited

Agent 不得使用 cluster-admin、主机 root、任意 shell、长期 kubeconfig；不得修改 DA-SOC SQL、出数、出图、生产邮箱或 DingTalk 目标凭据。