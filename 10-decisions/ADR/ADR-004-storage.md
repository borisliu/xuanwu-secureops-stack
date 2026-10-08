# ADR-004: Storage 采用 local-path / Local PV

- 状态：Accepted
- 日期：2026-10-08

## Decision

V0.1 使用 local-path/Local PV 和节点专用数据盘。ClickHouse、raw archive、n8n 状态使用独立 PVC；ClickHouse 单副本固定在 `xw-wk-02`。

## Rationale

DA-SOC v0.1 的首要风险是数据正确恢复，而不是跨节点在线复制。Ceph、Longhorn、分布式 ClickHouse 和跨节点同步数据库会增加节点、网络、故障模式和 AI 运维复杂度。

## Required Controls

- PVC 必须声明用途、节点亲和、容量和备份级别。
- 不用 emptyDir 保存持久业务数据。
- 不用任意 hostPath 暴露主机目录。
- ClickHouse 使用原生 BACKUP/RESTORE，不能只复制正在写入的 PVC。
- raw archive 使用文件备份和校验和。
- 节点磁盘设置 70%/80%/90% 水位告警。
- 必须完成节点故障、数据盘故障、ClickHouse 恢复和 raw 恢复演练。

## Deferred

分布式存储和分布式 ClickHouse 仅在 V0.2+ 根据容量、RPO/RTO 和业务规模触发。