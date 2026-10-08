# ADR-006: 分层 Backup 与 Recovery

- 状态：Accepted
- 日期：2026-10-08

## Decision

V0.1 使用：

```text
Git + etcd snapshot + ClickHouse BACKUP/RESTORE + 文件备份
```

备份目标位于集群外的 `xw-backup-01` 或企业 NAS/S3，并保留至少一份离线/不可变副本。

## Coverage

- Git 定义态：架构、治理、安全策略、NetworkPolicy、RBAC、Runbook、Task、ADR、Incident、SQL、workflow、Kubernetes manifests、镜像 digest。
- etcd：每 6 小时、控制面变更前。
- Kubernetes/KubeSphere：Git 清单与定期资源导出。
- ClickHouse：原生 BACKUP/RESTORE。
- raw archive：文件备份和校验和。
- n8n：workflow JSON、状态 PVC/配置和加密密钥。
- render/archive：ConfigMap、镜像 digest、配置和必要 PVC。
- Harbor：配置、数据库、registry data 和关键镜像 tar。
- Secret：SOPS/age 等加密导出；明文不得进 Git。

## Targets

RPO 为 24 小时以内；普通 Pod/节点故障分钟级恢复；Control Plane 恢复目标 4 小时；完整 DA-SOC 恢复目标 8 小时。具体容量和保留策略必须在实施参数中冻结。

## Required Drills

V0.1 完成前必须真实验证：

1. etcd/Control Plane 恢复。
2. `da-soc` 资源和 Secret 恢复。
3. ClickHouse 数据恢复并校验 SQL 结果。
4. raw archive 恢复。
5. Harbor 恢复并成功拉取镜像。
6. POP3 重放作为业务数据兜底。

没有恢复演练的备份不计入 Definition of Done。