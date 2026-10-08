# 玄武云盾 V0.1 External Dependencies

> 用途：列出实施不能由 Kubernetes/Agent 自行完成的外部条件。  
> 规则：任何 `BLOCKING` 项必须有 Owner、截止时间、证据和替代/降级路径。

| 依赖 | Owner | Needed Before | Blocking Task | Status | Evidence / Notes |
|---|---|---|---|---|---|
| 5 台 VM 及数据盘 | Infrastructure Owner | TASK-006 | TASK-011 | TBD / BLOCKING | 规格按 Baseline |
| 管理网/VPN/堡垒机 | Network/Security Owner | TASK-007 | TASK-011 | TBD / BLOCKING | API/KubeSphere/SSH 入口 |
| 节点网与服务网 | Network Owner | TASK-008 | TASK-011 | TBD / BLOCKING | VLAN/CIDR/路由 |
| Pod/Service CIDR 不冲突 | Network Owner | TASK-008 | TASK-011 | TBD / BLOCKING | 书面确认 |
| 企业 DNS | Network Owner | TASK-008 | TASK-011 | TBD / BLOCKING | 正向/反向解析 |
| 企业 NTP | Infrastructure Owner | TASK-007 | TASK-011 | TBD / BLOCKING | chrony 验证 |
| 出口防火墙白名单 | Security/Network Owner | TASK-021 | TASK-043 | TBD / BLOCKING | IMAPS/POP3S/DingTalk/API |
| Harbor 离线安装包 | Registry Owner | TASK-026 | TASK-027 | TBD / BLOCKING | checksum/digest |
| DA-SOC 镜像源和授权 | DA-SOC Owner | TASK-042 | TASK-042 | TBD / BLOCKING | n8n/ClickHouse/render |
| ClickHouse 历史备份 | DA-SOC Owner | TASK-047 | TASK-047 | TBD / BLOCKING | backup manifest |
| `/data/da-soc/raw` 历史归档 | DA-SOC Owner | TASK-048 | TASK-048 | TBD / BLOCKING | file manifest |
| n8n workflow source/SQL | DA-SOC Owner | TASK-044 | TASK-044 | TBD / BLOCKING | Git source |
| n8n encryption key recovery | DA-SOC Owner/Security Owner | TASK-045 | TASK-045 | TBD / BLOCKING | sealed/encrypted recovery |
| 生产邮箱访问授权 | DA-SOC Owner | TASK-052 | TASK-055 | TBD / BLOCKING | 不标已读证明 |
| 测试邮箱/回放数据 | DA-SOC Owner | TASK-050 | TASK-050 | TBD / BLOCKING | 禁止污染生产 |
| 测试 DingTalk 群 | DA-SOC Owner | TASK-052 | TASK-052 | TBD / BLOCKING | 与生产群隔离 |
| 生产 DingTalk 凭据 | DA-SOC Owner/Security Owner | TASK-054 | TASK-055 | TBD / BLOCKING | 凭据来源固定 |
| 运维 DingTalk 群 | Platform Owner | TASK-032 | TASK-032 | TBD / BLOCKING | 告警入口 |
| 内部 CA/证书 | Security Owner | TASK-026 | TASK-026 | TBD / BLOCKING | Harbor/Ingress |
| 集群外备份目标 | Backup Owner | TASK-038 | TASK-039 | TBD / BLOCKING | xw-backup-01/NAS/S3 |
| 离线/不可变备份副本 | Backup Owner | TASK-040 | TASK-061 | TBD / BLOCKING | 不能只有一份 |
| 生产切换窗口 | Business Owner | TASK-054 | TASK-055 | TBD / BLOCKING | 人工确认 |
| L2 审批人及替补 | Platform/Business Owner | TASK-058 | TASK-059 | TBD / BLOCKING | DingTalk/人工 |
| 变更/事故记录入口 | Governance Owner | TASK-005 | TASK-068 | TBD / BLOCKING | Git/Incident |

## 1. Ownership Rule

Owner 不能填写“Agent”。Agent 可以执行已批准任务，但不能批准外部依赖、生产窗口或安全例外。