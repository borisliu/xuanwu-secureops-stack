# ADR-003: Harbor 独立于 Kubernetes

- 状态：Accepted
- 日期：2026-10-08

## Decision

Harbor 部署在独立 VM `xw-harbor-01`，不进入 Kubernetes 集群，不由 Kubernetes 生命周期管理。

## Required Controls

- HTTPS、企业 CA、项目级权限、机器人账号最小权限和审计。
- `xuanwu/platform` 与 `xuanwu/da-soc` 项目分离。
- 工作负载使用不可变 tag 或 digest，禁止 `latest`。
- 开启基础漏洞扫描；扫描是 V0.1 基础门禁，不等于完整供应链安全。
- Worker 只通过 HTTPS 拉取，未授权 Registry 必须被拒绝。
- ECS 当前无法直连 Registry 时使用 `docker save`/离线传输/Harbor 导入。

## Recovery

Harbor 配置、数据库、registry data 和关键镜像 tar 进入 `xw-backup-01`。集群重建顺序必须是先恢复 Harbor，再恢复 Kubernetes 工作负载。Harbor 与备份仓库不共享 Kubernetes 故障域。

## Rejected

将 Harbor 作为 Kubernetes 业务 Pod 部署会让集群重建依赖集群自身，扩大恢复闭环，不符合 V0.1 的恢复优先原则。