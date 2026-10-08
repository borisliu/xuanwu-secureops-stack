# 玄武云盾 V0.1 Implementation Risks

| ID | 风险 | 等级 | 触发条件 | 预防 | 监测 | 回退/处置 | Owner |
|---|---|---|---|---|---|---|---|
| RISK-001 | 版本不兼容 | High | K8s/KubeSphere/CNI 安装失败 | VERSION-MATRIX 兼容矩阵冻结 | 安装前 PoC/健康检查 | 不安装，回到参数冻结 | Platform |
| RISK-002 | VM/IP/路由不足 | High | 节点无法互通 | 外部依赖确认、端口矩阵 | preflight | 修复网络参数，不进入安装 | Infrastructure |
| RISK-003 | Harbor 失效 | High | 镜像无法拉取 | 独立 VM、备份、离线 tar | registry pull/backup alert | 使用已验证离线包，暂停发布 | Registry |
| RISK-004 | Local PV 节点故障 | High | ClickHouse/PVC 所在节点故障 | 业务备份、恢复演练 | disk/node/PVC alert | 新节点 + 数据恢复或 POP3 重放 | DA-SOC |
| RISK-005 | 生产邮箱被标记已读 | Critical | cutover/rollback 行为异常 | 专用流程、测试回放、单活 | mailbox audit | 立即停止流程，人工调查 | DA-SOC |
| RISK-006 | 两个 n8n 同时消费 | Critical | ECS/K8s trigger 同时启用 | Cutover lock/checklist | trigger/status audit | 停止新旧双方，人工决定恢复 | Platform/DA-SOC |
| RISK-007 | 重复 DingTalk | High | 重试或回退重复发送 | 单活、幂等校验、发送记录 | delivery log | 冻结发送并对账 | DA-SOC |
| RISK-008 | archive 失败后仍出图 | Critical | workflow error branch 错误 | golden test、显式熔断 | business SLI | 禁止切换，回滚 workflow | DA-SOC |
| RISK-009 | Secret 丢失 | High | 加密密钥/备份未验证 | recovery key 双人管理 | backup/restore drill | 按 Secret 恢复 Runbook | Security |
| RISK-010 | Audit/日志泄露敏感信息 | High | token/邮箱正文进入日志 | 脱敏、最小采集 | Loki review | 删除/隔离日志，轮换凭据 | Security |
| RISK-011 | Backup 文件存在但不可恢复 | Critical | 未做 restore drill | 强制恢复验收 | backup age/restore evidence | 不允许切换，重新备份 | Backup |
| RISK-012 | Agent 越权 | Critical | Job SA 权限过大 | 最小 RBAC、Job TTL、L2 | audit/Role review | revoke SA、暂停 Agent | Security/AIOps |
| RISK-013 | 资源/磁盘耗尽 | High | ClickHouse/Loki/Harbor 增长 | quota/retention/watermark | metrics/alert | 清理受控数据或扩盘，禁止盲删 | Platform |
| RISK-014 | 生产切换后不可回退 | High | 未保留 ECS 状态/凭据 | cutover gate | checklist | 切换前置条件不满足即阻塞 | Platform/DA-SOC |
| RISK-OS-001 | UOS 1060e 与 Kubernetes/KubeSphere 组合兼容性未知 | High | 节点初始化、加入或 KubeSphere 安装失败 | 将组合标为 Candidate；先完成独立验证集群 | TASK-010/011/015 健康检查和安装报告 | 不冻结、不进入 DA-SOC 切换；回到版本冻结 | Platform |
| RISK-OS-002 | OS 软件源或离线安装介质不可获得 | High | 无法安装补丁、containerd 或 Kubernetes 前置包 | 提前确认 UOS 软件源、离线 RPM/包、ISO 和 SHA256 | TASK-006/007/009 preflight | 停止安装，使用已批准介质或重新排期 | Infrastructure |
| RISK-OS-003 | UOS 安全基线与 Kubernetes/containerd 网络配置冲突 | High | firewall、SELinux/安全模块、iptables/nftables、sysctl、cgroup 或时间同步异常 | 记录为待验证项，不预设必然冲突；在 preflight 和独立验证集群测试 | TASK-007/009/010/011/018 报告 | 保留最小安全基线，修订兼容参数并重新验证；不得无审批关闭安全控制 | Security/Platform |
| RISK-CLM-001 | 组件版本无法自动发现 | High | API、节点命令或镜像 metadata 不可用 | 组件专用 discovery method；unknown_version 进入高风险 | discovery 失败告警，暂停升级判断 | 冻结升级判断，转人工确认 | CLM/Platform |
| RISK-CLM-002 | 镜像 tag 与实际 digest 不一致 | High | tag 漂移或手工 manifest | digest 作为部署事实，Harbor/Git 双重记录 | drift 检查、镜像拉取审计 | 冻结发布，恢复批准 digest | Registry/Platform |
| RISK-CLM-003 | 漏洞或 EOL 数据过期/误判 | High | Source/Last Updated 超过 freshness 或 affected 判断缺失 | 记录来源、时间和 affected；过期状态为 REVIEW | freshness、KEV、EOL 和 unknown 指标 | 不自动升级，转人工复核 | Security |
| RISK-CLM-004 | CLM 自动生成升级 Task 绕过审批 | Critical | Job 获得写权限或生产 trigger 被启用 | L0/L1/L2、最小 RBAC、Git 分支保护、生产自动升级禁用 | Audit、Role review、Task/Job 状态 | 撤销 SA/Role，关闭 Job，创建 Incident | Security/AIOps |
| RISK-CLM-005 | 升级后验证或回退不完整 | High | 无 backup/checkpoint、兼容性或业务 golden test 证据 | 升级前备份，组件 Runbook，自动 post-discovery/verify | Upgrade Closure Rate、rollback evidence | 冻结后续升级，按 Runbook 恢复 | Platform/Component |
