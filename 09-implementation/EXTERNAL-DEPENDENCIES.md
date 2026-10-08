# 玄武云盾 V0.1 External Dependencies

> 用途：列出实施不能由 Kubernetes/Agent 自行完成的外部条件。
>
> **本文件是 V0.1 外部依赖（External Dependency）的唯一事实来源（SoT）。**
> `TODO.md` §22 只做引用与摘要，不得维护第二份互相冲突的 Blocking 映射；两者不一致时以本文件为准。
>
> 规则：任何 `BLOCKING` 项必须有 Owner、截止时间、证据和替代/降级路径。所有 Task 编号必须指向 `TODO.md` 中真实存在的任务。

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
| 出口防火墙白名单 | Security/Network Owner | TASK-020 | TASK-020/050/052 | TBD / BLOCKING | IMAPS/POP3S/DingTalk/API |
| Harbor 离线安装包 | Registry Owner | TASK-026 | TASK-027 | TBD / BLOCKING | checksum/digest |
| DA-SOC 镜像源和授权 | DA-SOC Owner | TASK-042 | TASK-042 | TBD / BLOCKING | n8n/ClickHouse/render |
| ClickHouse 历史备份 | DA-SOC Owner | TASK-047 | TASK-047 | TBD / BLOCKING | backup manifest |
| `/data/da-soc/raw` 历史归档 | DA-SOC Owner | TASK-048 | TASK-048 | TBD / BLOCKING | file manifest |
| n8n workflow source/SQL | DA-SOC Owner | TASK-044 | TASK-044/049 | TBD / BLOCKING | Git source |
| n8n encryption key recovery | DA-SOC Owner/Security Owner | TASK-045 | TASK-045 | TBD / BLOCKING | sealed/encrypted recovery |
| 生产邮箱访问授权 | DA-SOC Owner | TASK-052 | TASK-052/055/058 | TBD / BLOCKING | 不标已读证明 |
| 测试邮箱/回放数据 | DA-SOC Owner | TASK-050 | TASK-050～055/066 | TBD / BLOCKING | 禁止污染生产 |
| 测试 DingTalk 群 | DA-SOC Owner | TASK-052 | TASK-052 | TBD / BLOCKING | 与生产群隔离 |
| 生产 DingTalk 凭据 | DA-SOC Owner/Security Owner | TASK-054 | TASK-055 | TBD / BLOCKING | 凭据来源固定 |
| 运维 DingTalk 群 | Platform Owner | TASK-032 | TASK-032 | TBD / BLOCKING | 告警入口 |
| 内部 CA/证书 | Security Owner | TASK-026 | TASK-026 | TBD / BLOCKING | Harbor/Ingress |
| 集群外备份目标 | Backup Owner | TASK-038 | TASK-039 | TBD / BLOCKING | xw-backup-01/NAS/S3 |
| 离线/不可变备份副本 | Backup Owner | TASK-040 | TASK-038～041/064～066 | TBD / BLOCKING | 不能只有一份；恢复演练依赖其真实性 |
| 生产切换窗口 | Business Owner | TASK-054 | TASK-055 | TBD / BLOCKING | 人工确认 |
| L2 审批人及替补 | Platform/Business Owner | TASK-036 | TASK-036/055～059/061 | TBD / BLOCKING | DingTalk/人工 |
| 变更/事故记录入口 | Governance Owner | TASK-005 | TASK-068 | TBD / BLOCKING | Git/Incident |
| CLM 只读 Kubernetes/Harbor/节点/API 访问 | Platform/Security Owner | TASK-CLM-001 | TASK-CLM-002 | TBD / BLOCKING | ServiceAccount、Role、审计和 Job TTL |
| CVE/NVD/CNVD/CNNVD/CISA KEV/厂商公告或离线数据 | Security Owner | TASK-CLM-003 | TASK-CLM-003 | TBD / BLOCKING | Source、Last Updated、导入 hash |
| CLM Git 目录、分支保护和审批人 | Governance Owner | TASK-CLM-001 | TASK-CLM-005 | TBD / BLOCKING | YAML schema、CODEOWNERS、PR 审批 |
| 升级验证环境、备份点和组件 Runbook | Platform/Backup/Component Owner | TASK-CLM-005 | TASK-CLM-006 | TBD / BLOCKING | 兼容性、backup hash、验证/回退证据 |

| 两清两固 Git 安全基线和 CODEOWNERS | Security/Governance Owner | TASK-SEC-001 | TASK-SEC-001 | TBD / BLOCKING | `04-security/` 四份 YAML、审批人和字段校验 |
| 五台 VM、关键 Service、Ingress、NodePort 和 Harbor 端口清单 | Security/Network/Platform Owner | TASK-SEC-002 | TASK-SEC-002 | TBD / BLOCKING | ss/firewall/Kubernetes discovery evidence |
| UOS/Kubernetes/KubeSphere/Harbor 账号元数据和最近使用信息 | Security/Platform/DA-SOC Owner | TASK-SEC-003 | TASK-SEC-003 | TBD / BLOCKING | 仅元数据；禁止提供真实密码 |
| RBAC、NetworkPolicy、Firewall 和平台访问路径测试环境 | Security/Network/Platform Owner | TASK-SEC-004 | TASK-SEC-004 | TBD / BLOCKING | allow/deny 测试、diff、回退证据 |
| 两清两固样例 finding 和批准的整改 Runbook | Security/Component Owner | TASK-SEC-005 | TASK-SEC-006 | TBD / BLOCKING | 漏洞、端口、账号、访问控制各至少一个样例 |

| 外部隔离恢复环境 `xw-restore-drill`（独立 VM，非 Kubernetes Namespace） | Infrastructure/Backup Owner | TASK-064 | TASK-064～066 | TBD / BLOCKING | 不与生产 `da-soc` 共用资源；演练后销毁；不新增 V0.1 Namespace |

## 1. Ownership Rule

Owner 不能填写“Agent”。Agent 可以执行已批准任务，但不能批准外部依赖、生产窗口或安全例外。
