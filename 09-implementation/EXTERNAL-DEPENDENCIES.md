# 玄武云盾 V0.1 External Dependencies

> 用途：列出实施不能由 Kubernetes/Agent 自行完成的外部条件。  
> 规则：任何 `BLOCKING` 项必须有 Owner、截止时间、证据和替代/降级路径。

| 依赖 | Owner | Needed Before | Blocking Task | Status | Evidence / Notes |
|---|---|---|---|---|---|
| 5 台 VM 及数据盘 | Infrastructure Owner | TASK-006 | TASK-011 | TBD / BLOCKING | 规格按 Baseline |
| 统信服务器操作系统 V20 1060e AMD64 ISO/安装介质 | Infrastructure/Platform Owner | TASK-007 | TASK-007 | TBD / BLOCKING | ISO/介质 SHA256、来源和导入记录 |
| 统信 OS 免费使用授权确认 | Platform/Legal Owner | TASK-007 | TASK-007 | TBD / BLOCKING | 授权条款或内部确认记录；不得描述为认证或 SLA |
| OS 安装介质完整性校验 | Security/Infrastructure Owner | TASK-007 | TASK-007 | TBD / BLOCKING | SHA256、签名/校验输出 |
| UOS 软件源或离线 RPM/包获取路径 | Infrastructure Owner | TASK-007 | TASK-009 | TBD / BLOCKING | 在线源或离线介质、校验和、导入记录 |
| 管理网/VPN/堡垒机 | Network/Security Owner | TASK-007 | TASK-011 | TBD / BLOCKING | API/KubeSphere/SSH 入口 |
| 节点网与服务网 | Network Owner | TASK-008 | TASK-011 | TBD / BLOCKING | VLAN/CIDR/路由 |
| Pod/Service CIDR 不冲突 | Network Owner | TASK-008 | TASK-011 | TBD / BLOCKING | 书面确认 |
| 企业 DNS | Network Owner | TASK-008 | TASK-011 | TBD / BLOCKING | 正向/反向解析 |
| 企业 NTP | Infrastructure Owner | TASK-007 | TASK-011 | TBD / BLOCKING | chrony 验证 |
| Kubernetes v1.30.6 安装包/镜像 | Platform Owner | TASK-002 | TASK-011 | TBD / BLOCKING | 官方来源、digest、离线包校验 |
| KubeSphere 4.1.x（优先验证 4.1.2）安装包/兼容矩阵 | Platform Owner | TASK-002 | TASK-015 | TBD / BLOCKING | 官方矩阵快照、安装包校验 |
| containerd 1.7.x 安装包 | Platform Owner | TASK-002 | TASK-009 | TBD / BLOCKING | RPM/包来源、SHA256、CRI 验证 |
| Calico 安装清单/镜像 | Network Owner | TASK-002 | TASK-018 | TBD / BLOCKING | 版本、digest、离线导入和兼容性记录 |
| Kubernetes/KubeSphere/Calico 相关镜像 | Registry/Platform Owner | TASK-002 | TASK-011 | TBD / BLOCKING | Harbor/离线 tar、digest、扫描结果 |
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
| CLM 只读 Kubernetes/Harbor/节点/API 访问 | Platform/Security Owner | TASK-CLM-001 | TASK-CLM-002 | TBD / BLOCKING | ServiceAccount、Role、审计和 Job TTL |
| CVE/NVD/CNVD/CNNVD/CISA KEV/厂商公告或离线数据 | Security Owner | TASK-CLM-003 | TASK-CLM-003 | TBD / BLOCKING | Source、Last Updated、导入 hash |
| CLM Git 目录、分支保护和审批人 | Governance Owner | TASK-CLM-001 | TASK-CLM-005 | TBD / BLOCKING | YAML schema、CODEOWNERS、PR 审批 |
| 升级验证环境、备份点和组件 Runbook | Platform/Backup/Component Owner | TASK-CLM-005 | TASK-CLM-006 | TBD / BLOCKING | 兼容性、backup hash、验证/回退证据 |

## 1. Ownership Rule

Owner 不能填写“Agent”。Agent 可以执行已批准任务，但不能批准外部依赖、生产窗口或安全例外。
