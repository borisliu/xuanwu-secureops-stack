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