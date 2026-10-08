# 玄武云盾 V0.1 综合架构裁决

> 状态：**已裁决**  
> 日期：2026-10-08  
> 文档类型：Architecture Adjudication  
> 权威关系：本文件解释裁决过程；`ARCHITECTURE-BASELINE-V0.1.md` 是后续实施唯一架构依据。

## 1. 裁决摘要

本次裁决不再输出多个候选方案，而是形成一套唯一的 V0.1 Architecture Baseline：

> **玄武云盾 V0.1 是一套单站点、单 Kubernetes 集群、1 个 Control Plane + 2 个 Worker 的 KubeSphere 私有云平台；Calico 提供网络隔离，Harbor 和备份仓库位于集群外，DA-SOC v0.1 的 n8n、ClickHouse、render/archive 和 raw archive 在 V0.1 内实际运行于 `da-soc` Namespace，所有迁移通过单活生产切换、确定性验证和可回退 Runbook 完成；AI Agent 以短生命周期 Kubernetes Job 运行，Task 以 Git YAML/Markdown 保存，所有高风险操作人工审批并审计。**

关键裁决：

1. **ADR-001 被项目负责人明确锁定：V0.1 必须实际承载 DA-SOC v0.1。** “只纳管 ECS”“影子环境”“V0.2 再迁移”不再是最终结论。
2. 迁移风险通过临时验证环境、数据回放、单活切换、消息幂等、观察窗口和回退 Runbook 解决；不得让两个 n8n 同时读取生产邮箱。
3. 采用 5 台 VM：`xw-cp-01`、`xw-wk-01`、`xw-wk-02`、`xw-harbor-01`、`xw-backup-01`。
4. Kubernetes 采用单控制面，V0.1 接受控制面非 HA，要求 etcd/资源/业务数据真实恢复演练；V0.2 以真实容量和 SLA 数据为依据增加 HA。
5. 采用已接受的 ADR-002～ADR-008，不引入 Ceph、Longhorn、Service Mesh、SIEM、CMDB、Task CRD、常驻高权限 Agent 或 `xw-opsapi`。

## 2. 输入材料

### 2.1 项目正式资料

已读取：

- `README.md`
- `TODO.md`
- `00-project/VISION.md`
- `00-project/GOALS.md`
- `00-project/SCOPE.md`
- `00-project/PRINCIPLES.md`
- `00-project/VERSIONING.md`

`01-architecture/` 与 `02-governance/` 当前没有可供读取的正式架构文档，相关文件仍由占位状态开始。本裁决因此将 README/TODO 与任务中明确的 DA-SOC 约束作为上位输入，并把缺失的治理内容转化为 Baseline 和 ADR 的实施前置条件。

### 2.2 五份候选方案

实际文件名与任务中编号的映射如下：

| 任务名称 | 实际文件 | 主要特征 |
|---|---|---|
| `codebuddy(6).md` | `09-implementation/00-architecture-review/codebuddy.md` | 强调 ECS 生产冻结、影子实例、`xw-opsapi`、外部备份 |
| `codex(5).md` | `09-implementation/00-architecture-review/codex.md` | 小集群、DA-SOC 受控迁移、单一观测栈和外部备份 |
| `cursor(7).md` | `09-implementation/00-architecture-review/cursor.md` | 3 节点集群、组件精简、逐组件迁移判断 |
| `dsh(7).md` | `09-implementation/00-architecture-review/dsh.md` | 4 节点、Job Agent、Task CRD、详细双跑和恢复设计 |
| `kimi(5).md` | `09-implementation/00-architecture-review/kimi.md` | 4 节点混合桥接、ECS 冻结、文件型 Task、外置服务节点 |

这些文件均被视为候选输入，不因作者或历史来源而获得默认优先级。

## 3. 项目事实

### 3.1 V0.1 的事实目标

- 玄武云盾是平台，DA-SOC 是第一个核心业务应用。
- V0.1 不是完整企业私有云，而是第一个可实际承载业务、可恢复、可审计、可验证 AI-Native 运维闭环的最小版本。
- 企业缺少成熟专职 Kubernetes 运维团队，因此平台必须依靠文档、Git、Runbook、策略和 AI 辅助运维，而不能依赖某个专家的隐性经验。
- 高风险操作必须人工审批；重要操作必须可审计、可验证和可恢复。

### 3.2 DA-SOC 的固定事实

```text
IMAP（ALL，不标已读）
  → Filter
  → POST /archive
  → 解析入库
  → ClickHouse SQL
  → POST /render
  → 测试钉钉群
```

一次性回补：

```text
POP3 → /data/da-soc/raw → HTTP INSERT → ClickHouse
```

当前组件：ClickHouse、`da-soc-render:0.1`、n8n 2.15.0；当前采用 Docker host network 和 loopback 地址；ECS 不能直接连接 Docker Registry。

### 3.3 不可变业务纪律

- 数字只能来自 ClickHouse SQL 和既定视图。
- 图片只能由 render 服务生成。
- LLM 不得参与出数、出图、业务统计口径或 SQL 生成。
- 无数据保持 `null`/“暂无数据”，不能填充 `0`。
- `/archive` 失败时不入库、不出图、不发送。
- 不能覆盖生产邮箱，不能 Mark as Read、删除或修改生产邮件。
- 测试钉钉群与生产群/运维群必须隔离，目标只能来自凭据。

### 3.4 已锁定的原则性 ADR

任务已经原则接受：Calico、独立 Harbor、local-path/Local PV、Prometheus + Grafana + Alertmanager + Fluent Bit + Loki、分层备份、短生命周期 Job Agent、YAML/Markdown Task。裁决只补足具体边界和实施方式，不重新打开无休止的选型讨论。

## 4. 五方案共识

五份方案尽管对 DA-SOC 是否在 V0.1 生产迁移存在根本分歧，但在以下方面高度一致：

1. V0.1 必须小范围、单集群、避免分布式存储、Service Mesh、SIEM、CMDB 和多租户。
2. Calico 比 Cilium 更适合当前 V0.1 的范围和运维能力。
3. ClickHouse 应使用单实例和本地持久存储，恢复能力优先于在线副本。
4. Harbor 是必要的镜像治理和离线交付能力，镜像应固定 digest。
5. 监控和日志必须收敛为单一栈，不能引入 ES/ELK 作为额外复杂度。
6. DA-SOC 的 SQL、n8n 编排权、邮箱只读纪律和 archive 失败门禁不能被平台破坏。
7. Agent 不应拥有无限权限，必须使用 L0/L1/L2 风险分级和审计。
8. 备份没有真实恢复演练就不算完成。
9. Git 应保存架构、策略、清单、Runbook、SQL 和工作流等定义态。
10. V0.1 的成功不等于“组件安装完成”，而是业务、恢复、观测和 AI Ops 闭环都能被验证。

## 5. 五方案主要分歧

| 分歧 | 候选观点 | 裁决问题 |
|---|---|---|
| DA-SOC Hosting | `codebuddy`/`kimi`/`dsh`主张 V0.1 ECS 冻结或平行实例；`cursor`/`codex`更接近受控入仓 | ADR-001 已明确必须在 V0.1 实际承载，必须把“平行验证”变成迁移过程而不是最终状态 |
| Harbor 位置 | 有方案将 Harbor 放进集群，有方案放独立 VM | 已接受 ADR-003，必须独立于 Kubernetes 集群 |
| VM 数量 | 3、4、5 台都有提议 | 资源不是主要约束，恢复故障域比省一台 VM 更重要；最终采用 5 台 |
| Control Plane | 普遍 1 CP，DSH 提出后续扩到 3 CP | V0.1 先 1 CP + 恢复演练，V0.2 按 SLA 触发 HA |
| Task 模型 | DSH 提出 CRD；其他方案偏 Git 文件/JSONL | 已接受 ADR-008，V0.1 使用 Git YAML/Markdown，不建 CRD |
| Agent 入口 | `codebuddy`强调 `xw-opsapi`；其他方案可直接访问 API | V0.1 直接只读访问 K8s/Prometheus/Loki/Git，推迟中间聚合层 |
| Agent Runtime | DSH 明确 Job；其他方案有常驻/逻辑 Agent | 已接受 ADR-007，统一为短生命周期 Job |
| 日志实现 | Fluent Bit/Loki 与 Promtail/Loki 两种实现 | 任务已锁定 Fluent Bit + Loki，Promtail 不进入 Baseline |
| 备份工具 | restic、Velero、脚本、MinIO 各有提案 | 按已接受 ADR-006 使用 Git + etcd snapshot + ClickHouse BACKUP + 文件备份；不以 Velero/MinIO 作为 V0.1 必需组件 |

## 6. 决策矩阵

评分范围为 1～10，先按任务给定权重计算候选方向，再应用硬约束否决。分数是裁决记录，不是对候选作者的评价。

| 候选 | 生产安全 25% | 可恢复 20% | AI 可维护 15% | 复杂度 15% | 可实施 10% | 演进 10% | 资源 5% | 加权参考分 | 硬约束结果 |
|---|---:|---:|---:|---:|---:|---:|---:|---:|---|
| codebuddy | 9 | 9 | 8 | 8 | 8 | 9 | 8 | 8.55 | DA-SOC 最终不在 V0.1 承载，否决 |
| codex | 8 | 8 | 8 | 8 | 9 | 8 | 8 | 8.10 | 可作为小集群和迁移输入 |
| cursor | 8 | 8 | 8 | 8 | 9 | 8 | 8 | 8.10 | 可作为组件迁移和网络输入 |
| dsh | 8 | 9 | 9 | 7 | 7 | 9 | 7 | 8.20 | Task CRD 与已接受 ADR-008 冲突，部分采纳 |
| kimi | 9 | 9 | 8 | 9 | 9 | 8 | 8 | 8.75 | DA-SOC 最终不在 V0.1 承载，否决 |
| **最终裁决** | **9** | **9** | **8** | **8** | **8** | **9** | **7** | **8.45** | **通过** |

最终方案不是把五份方案的组件全部相加，而是采纳与硬约束一致的部分：

- 从 `cursor`/`codex` 采纳小型单集群和逐组件入仓。
- 从 `dsh` 采纳 Job Agent、明确切换门槛、恢复演练和业务指标思路，但拒绝 Task CRD。
- 从 `codebuddy`/`kimi` 采纳故障域隔离、外部备份和对 ECS 现实约束的重视，但拒绝“V0.1 不承载 DA-SOC”的结论。
- 对所有方案中“两个 n8n 同时读取生产邮箱”的隐含风险作出否决：验证可以双跑，生产消费者必须单活。

## 7. ADR-001 裁决

### 7.1 裁决结论

**V0.1 必须实际承载 DA-SOC v0.1。** 最终生产承载位置是 Kubernetes 的 `da-soc` Namespace。ECS 只作为迁移期间的回退源和短期保留实例，不是 V0.1 的最终生产承载架构。

### 7.2 逐组件裁决

| 组件 | V0.1 归属 | 裁决 |
|---|---|---|
| n8n 2.15.0 | Kubernetes `da-soc` | 单副本、单活生产工作流；工作流从 Git 构建/导入，不在 UI 修改 SQL |
| ClickHouse | Kubernetes `da-soc` | 单副本 StatefulSet + Local PV；HTTP 8123 只走 ClusterIP |
| render/archive | Kubernetes `da-soc` | Deployment + ClusterIP 8091；不使用 hostNetwork |
| raw archive | Kubernetes `da-soc` PVC | 独立 PVC；使用文件备份和恢复校验 |
| SQL | Git | `v0.1/sql/` 为唯一源，构建后导入 n8n |
| workflow | Git + n8n PVC | Git 为定义态，n8n 为运行态；禁止 UI 私改生产 SQL |
| Secrets | K8s Secret + 加密 Git/外部备份 | 明文不进 Git，不交给 Agent |
| ECS | 临时回退/迁移源 | 不能作为最终 V0.1 生产架构；不得与新 n8n 同时读取生产邮箱 |

### 7.3 双跑裁决

允许**验证双跑**，不允许**生产双消费者**：

- 允许 K8s `da-soc-validate` 临时 Namespace 使用测试邮箱、回放数据和独立 ClickHouse 验证。
- 不允许 ECS n8n 和 Kubernetes n8n 同时读取生产邮箱。
- 不允许两个流程同时发送同一业务日报。
- 生产切换采用“停止旧消费者 → 检查无活动执行 → 启动新消费者”的单活切换。
- 生产切换前必须建立 Message-ID/邮件 UID/业务日期的去重和发送幂等验证；没有幂等证据，不允许切换。

### 7.4 切换与回退

切换前保留 ECS 可启动状态、最后工作流版本、生产凭据恢复方式和数据快照。切换失败时先停止 K8s 生产 n8n；若尚未产生新生产日报，可直接回退 ECS。若已产生业务输出，先冻结消费和发送，核对数据、发送记录和幂等状态后再由业务负责人批准回退，禁止盲目启动两个消费者。

## 8. ADR-002～008 裁决

### ADR-002：Calico

采用 Calico，仅使用 NetworkPolicy 和基础网络能力；不引入 Hubble、Tetragon、Service Mesh 或 eBPF 运行时安全。

### ADR-003：Harbor

Harbor 部署在 `xw-harbor-01`，不进入 Kubernetes 集群。使用 HTTPS、项目权限、digest 固定、基础扫描和离线导入。Harbor 自身数据与配置进入外部备份，集群重建时先恢复 Harbor，再恢复工作负载。

### ADR-004：Storage

业务使用 local-path/Local PV。ClickHouse、raw archive、n8n 状态分别使用 PVC；不使用 Ceph、Longhorn、分布式 ClickHouse。数据安全由 ClickHouse BACKUP、文件备份、POP3 重放兜底和真实恢复演练保证。

### ADR-005：Observability

唯一主观测栈是 Prometheus、Grafana、Alertmanager、Fluent Bit、Loki。禁用 Elasticsearch/ELK/KubeSphere ES Logging。关键告警进入独立玄武云盾运维 DingTalk 群，DA-SOC 业务日报群不用于平台告警。

### ADR-006：Backup

Git + etcd snapshot + ClickHouse BACKUP + raw/n8n/render/Harbor 文件备份。Secret 使用加密导出；备份目标位于集群外；至少完成 Control Plane、DA-SOC 数据和 Harbor 的真实恢复演练。

### ADR-007：Agent Runtime

Agent 每次任务以短生命周期 Kubernetes Job 运行，使用独立 ServiceAccount、TTL、资源限制和最小 Role。没有常驻高权限 Agent；执行结束后 Job 终止，日志和任务记录保留在 Loki/Git。

### ADR-008：Task Model

Task 使用 Git YAML/Markdown。Task 状态包含事实、证据、计划、风险、审批、执行、验证、回滚和审计字段。V0.1 不使用 Task CRD；V0.2 再按真实查询量评估。

## 9. 最终技术栈

| 能力 | 最终裁决 | V0.1 边界 |
|---|---|---|
| Kubernetes | Kubeadm/KubeKey 兼容安装路径，版本按 KubeSphere 支持矩阵锁定 | 单集群、1 CP + 2 Worker |
| Management | KubeSphere 精简组件集 | 1 Workspace、1 业务 Project |
| CNI | Calico | NetworkPolicy、CoreDNS、基础连通性 |
| Storage | local-path/Local PV | 单实例、节点亲和、外部备份 |
| Registry | 独立 Harbor | HTTPS、digest、基础扫描、离线导入 |
| Monitoring | Prometheus + Grafana + Alertmanager | 平台和 DA-SOC 指标 |
| Logging | Fluent Bit + Loki | 容器、节点、审计和业务结构化日志 |
| Backup | Git + etcd snapshot + ClickHouse BACKUP + 文件备份 | 真实恢复演练 |
| Security | RBAC、PSA、NetworkPolicy、镜像准入、Audit、Secret 加密 | 默认拒绝和最小权限 |
| AI Ops | n8n/Cron 触发 + Job Agent + Git Task | L0、有限 L1、L2 审批 |
| Task | Git YAML/Markdown | 不建 CRD、不建 Web Task Center |

## 10. 最终 VM 拓扑

| VM | 角色 | 关键职责 | 故障影响 |
|---|---|---|---|
| `xw-cp-01` | Control Plane + etcd | API Server、scheduler、controller、etcd、KubeSphere 控制面 | 管理面中断；业务已运行的 Pod 可短时继续，依赖恢复 Runbook |
| `xw-wk-01` | Worker / Platform | 平台 Job、观测采集、Ingress、AI Job | 平台辅助能力降级，不应丢业务数据 |
| `xw-wk-02` | Worker / DA-SOC | ClickHouse、n8n、render/archive、raw PVC | DA-SOC 业务中断，需要本地盘恢复/重建 |
| `xw-harbor-01` | 独立 Registry | Harbor、镜像项目、HTTPS、扫描 | 新部署/恢复受阻，已有容器可继续运行 |
| `xw-backup-01` | 独立备份仓库 | etcd、ClickHouse、raw、n8n、Harbor 备份 | 恢复点暂不可用；需离线/第二副本保护 |

`xw-harbor-01` 和 `xw-backup-01` 不与 Kubernetes 节点共享生命周期。备份仓库还必须有一份离线或不可变副本，避免单台备份 VM 成为唯一救命数据。

## 11. 最终 Kubernetes 拓扑

- Control Plane：单节点、etcd 单实例、Control Plane taint，不调度 DA-SOC。
- Worker：`xw-wk-01` 承载平台辅助工作负载，`xw-wk-02` 通过标签和亲和性承载 DA-SOC 数据面。
- Namespace：`da-soc` 为唯一正式业务 Namespace；`da-soc-validate` 仅为迁移期间的临时回放环境，切换后删除。
- KubeSphere：一个 Workspace `xuanwu`，一个正式业务 Project `da-soc`。
- Ingress：只为管理 VPN 下的 KubeSphere 控制台和 n8n 管理 UI 提供 HTTPS；ClickHouse、render/archive 和业务 API 使用 ClusterIP。
- 安全：`da-soc` 默认拒绝 ingress/egress，显式放行 DNS、n8n→render、业务→ClickHouse、监控抓取和必要外部出口。

## 12. 最终网络架构

| 网络 | 示例 | 规则 |
|---|---|---|
| 管理网 | `10.20.10.0/24` | 仅 VPN/堡垒机到 API、KubeSphere、SSH |
| 节点网 | `10.20.20.0/24` | 节点间必要端口，etcd 最小开放 |
| 服务网 | `10.20.30.0/24` | Harbor、备份仓库、恢复流量 |
| Pod 网 | `10.244.0.0/16` | Calico 内部网络 |
| Service 网 | `10.96.0.0/12` | ClusterIP |

DA-SOC 外部出口仅允许 DNS、IMAP/POP3S、DingTalk HTTPS 和经批准的外部 API；禁止任意互联网出口。API Server、etcd、ClickHouse 不暴露公网。

## 13. 最终存储架构

- ClickHouse：`xw-wk-02` local PV，单副本，独立数据盘，磁盘水位告警。
- raw archive：独立 PVC，按文件校验和备份。
- n8n：单副本 PVC，保留现有数据库形态，不因 V0.1 额外引入 PostgreSQL。
- render/archive：无状态 Deployment；必要临时文件只使用 PVC/emptyDir，不把业务真数据放在 emptyDir。
- Harbor：独立 VM 本地数据盘，独立备份。
- 不把存储副本当成备份；所有恢复必须从备份或重放路径验证。

## 14. 最终备份架构

| 对象 | 方法 | 频率/触发 | 恢复验证 |
|---|---|---|---|
| Git 定义态 | Git 远端、离线镜像 | 每次变更 | 克隆后渲染/校验 |
| etcd | `etcdctl snapshot save` | 每 6 小时、变更前 | 新控制面恢复 |
| K8s/KubeSphere 资源 | Git 清单 + 资源导出 | 每次变更、每日 | 空集群重建 |
| ClickHouse | 原生 `BACKUP`/`RESTORE` | 每日、切换前 | 查询结果和行数校验 |
| raw archive | 文件备份/校验和 | 每日 | 文件恢复和重放 |
| n8n workflow | Git JSON + PVC/配置备份 | 每次变更 | 导入后执行验证 |
| render/archive 配置 | Git/ConfigMap/镜像 digest | 每次变更 | Pod 重建 |
| Harbor | 配置、数据库、registry data、关键镜像 tar | 每日/每周 | 独立恢复拉取镜像 |
| Secret | SOPS/age 加密导出 + 离线密钥 | 每次轮换 | 受控恢复，禁止明文 Git |

## 15. 最终安全架构

- 人类管理员使用个人账号、MFA/堡垒机和短时授权。
- Agent ReadOnly 只读；Agent L1 仅有 `da-soc` render Deployment 的受限动作；L2 无自动写权限。
- `default` ServiceAccount 不绑定高权限；禁止业务和 Agent 使用 cluster-admin。
- PSA 采用 restricted；禁止 privileged、hostNetwork、hostPID、hostIPC，例外必须有 ADR 和到期时间。
- Harbor 是唯一认可镜像来源；镜像固定 digest，未授权 Registry 在 admission/验收中被拒绝。
- Kubernetes Audit、KubeSphere 审计、主机审计和 Agent Task 审计全部留痕。
- n8n 的生产邮箱和 DingTalk 凭据只对业务工作流可见，Agent 只能读状态和执行白名单 Runbook。

## 16. 最终 Observability 架构

监控 Prometheus/Grafana/Alertmanager：节点、API Server、etcd、KubeSphere、Pod、PVC、Harbor、备份、证书和 DA-SOC 业务 SLI。

日志 Fluent Bit → Loki：节点 journald、容器 stdout/stderr、Kubernetes Audit、KubeSphere 关键审计、n8n execution 摘要、render/archive 和 ClickHouse 运行日志。生产邮箱正文、Secret、Token 和敏感业务数据必须脱敏或不采集。

首批告警：NodeDown、DiskFull、PodCrashLoop、PodRestart、API/etcd 异常、证书到期、Harbor 不可用、备份失败、ClickHouse 不可用、archive 失败、日报未按时完成和 DingTalk 发送失败。

## 17. 最终 AI Ops 架构

### 17.1 组件

- n8n/Cron：定时、Webhook、DingTalk、任务触发和通知。
- Agent Job：一次性观察、分析、计划、受控执行、验证。
- Git：Policy、Runbook、Task、审计结论和变更定义。
- K8s API/Prometheus/Loki：直接只读数据源。

### 17.2 不引入 xw-opsapi

V0.1 不建设 `xw-opsapi`，原因是它会新增服务、认证、部署、备份和故障面，而当前 Agent Job 可以直接使用：

- Kubernetes API + 最小 ServiceAccount
- Prometheus HTTP API 的只读凭据
- Loki HTTP API 的只读凭据
- Git 仓库的受限 token

V0.2 只有在跨多个业务、查询授权无法复用、或重复适配代码成为明确运维负担时，才引入 `xw-opsapi`。

### 17.3 真实闭环场景

以 `da-soc-render` CrashLoopBackOff 为例：

1. Prometheus/事件告警触发 n8n。
2. n8n 创建 `tasks/TSK-*.yaml`，写入 Pod 状态、日志摘要、影响和证据链接。
3. n8n 启动短生命周期 Agent Job。
4. Agent 读取 K8s/Prometheus/Loki/Git/Runbook，判断为 L1 或 L2。
5. L1 在白名单内执行 `rollout restart`；L2 通过 DingTalk 请求人工审批。
6. Agent 验证 Deployment Ready、健康探针、错误日志下降和 n8n→render 连通性。
7. 验证失败时执行受限 `rollout undo` 或将 Task 升级人工，不得擅自改镜像和网络策略。
8. Agent Job 结束，Task YAML 写入执行、验证、回滚和审计证据。

## 18. 最终 DA-SOC Hosting Architecture

正式运行图：

```text
n8n (da-soc)
  ├─ ClusterIP → render/archive:8091
  ├─ ClusterIP → ClickHouse:8123
  ├─ Egress → IMAPS/POP3S
  └─ Egress → DingTalk HTTPS
```

关键不变量：

- n8n 仍是日常编排者。
- SQL 仍来自 `v0.1/sql/` 和 `build_workflow.py`。
- ClickHouse 仍负责确定性出数。
- render 仍负责出图。
- archive 失败仍熔断后续流程。
- 生产邮箱仍不 Mark as Read。
- LLM 不进入业务数据路径。

## 19. 迁移 / Cutover / Rollback

### Phase 1：准备

冻结 Baseline、建立 VM/网络/备份目标、完成安全基线、建立 `da-soc` 和临时 `da-soc-validate`。

### Phase 2：镜像

在受控构建环境取得 ClickHouse、render/archive、n8n 镜像；生成 digest、SBOM/扫描记录；离线导入 Harbor；Worker 使用 HTTPS 拉取验证。

### Phase 3：部署

先部署 ClickHouse，再部署 render/archive，最后部署 n8n。生产 n8n 的触发器默认关闭；Service、PVC、Secret、NetworkPolicy、配额和探针必须先验收。

### Phase 4：数据

使用 ClickHouse 原生 BACKUP/RESTORE 导入历史数据；使用文件备份恢复 raw archive；校验表结构、行数、日期范围、校验和和数据来源。POP3 重放作为数据恢复兜底，不作为未经验证的日常双写。

### Phase 5：workflow

从 Git 构建 workflow JSON，经 n8n API 导入；禁止通过 UI 修改 SQL；凭据通过受控 Secret 注入。

### Phase 6：验证

使用独立测试邮箱或回放数据，验证当天、近 6 周、近 6 月；逐项比较 SQL 结果、`null` 语义、图片像素/关键字段、archive 失败行为、HTTP SQL 和 render 请求。

### Phase 7：DingTalk

测试群与运维群隔离，发送目标只能来自 Secret；先发送单独验收消息，再验证完整日报，禁止在验证阶段使用生产发送凭据。

### Phase 8：切换

- 停止 ECS n8n 生产触发器。
- 等待并确认 ECS 无活动执行，记录最后处理边界和工作流 digest。
- 备份 ECS 状态与 K8s 数据。
- 启用 K8s `da-soc` n8n 生产触发器。
- 观察至少一个完整日报周期。

### Phase 9：观察

观察 archive、ClickHouse、render、邮件读取、DingTalk、日志、告警、备份和数据新鲜度；ECS 保持可启动但禁止消费生产邮箱。

### Phase 10：回退

满足以下任一条件立即停止 K8s 生产 n8n：数据差异、archive/render 异常、不可解释数据缺失、邮件行为异常、DingTalk 目标异常、备份失败或安全边界失效。先冻结发送和消费，再按幂等状态和业务确认决定技术回退到 ECS 或在 K8s 内修复；禁止双 n8n 同时运行。

### Cutover Gate

```text
数据一致
图片一致
业务流程一致
邮件读取行为一致
DingTalk 行为一致
备份成功
恢复测试成功
监控正常
日志正常
告警正常
回退路径验证成功
单活消费者验证成功
```

## 20. IT / Business Boundary

IT/平台负责 VM、OS、Kubernetes、KubeSphere、Calico、Storage、Harbor、Ingress、RBAC、NetworkPolicy、监控、日志、备份、审计和平台安全。

DA-SOC 业务负责应用镜像内容、n8n 业务流程、SQL、解析逻辑、render 逻辑、邮箱规则、DingTalk 业务目标、业务数据、业务指标、SLA 和统计口径。

边界通过 Namespace、RBAC、ResourceQuota、LimitRange、NetworkPolicy、Secret、镜像准入和审批流程落实。

## 21. V0.1 Scope

必须完成：

- 5 台 VM 的物理基线、Kubernetes/KubeSphere 单集群和 Calico。
- 独立 Harbor、镜像 digest、离线导入和基础扫描。
- local-path/Local PV、DA-SOC `da-soc` Namespace、默认拒绝网络策略和资源配额。
- Prometheus/Grafana/Alertmanager、Fluent Bit/Loki、Audit 和 DingTalk 运维告警。
- Git、etcd、ClickHouse、raw、n8n、render/archive、Harbor 和 Secret 的备份/恢复路径。
- DA-SOC 实际入仓、数据迁移、workflow 导入、验证、单活切换、观察和回退。
- Agent Job、Git Task、L0/L1/L2、真实 CrashLoop 或 Disk Pressure 闭环。
- 安全测试、恢复演练、故障演练和架构/实际状态核对。

## 22. V0.1 Non-Goals

不做：

- 三控制面 HA（除非实施前经 RTO/SLA 评审将其升级为硬门槛）。
- Ceph、Longhorn、分布式 ClickHouse、Service Mesh、ELK/SIEM、CMDB、多集群、跨地域 DR、多租户、GPU、完整 Runtime/Supply Chain Security。
- Task CRD、常驻高权限 Agent、`xw-opsapi`、独立 Web Task Center。
- LLM 参与 DA-SOC 出数、出图、SQL 或邮箱操作。
- 两个生产 n8n 同时消费邮箱。

## 23. V0.2～V1.0 Evolution

- **V0.2**：根据实际 RTO/RPO、容量和故障数据增加 3 Control Plane、集中身份、密钥服务、受控漂移修复和可选 `xw-opsapi`。
- **V0.3**：运行时安全、EDR/SIEM、SBOM/签名、策略即代码和安全事件管理。
- **V0.4**：多 Agent 协作、事件关联、更多 L1 自动化和变更影响分析；按真实 Task 查询量评估 CRD。
- **V0.5**：多业务 Namespace、租户/配额治理、分布式存储和容量/成本管理。
- **V1.0**：企业级私有云、跨站点恢复、统一身份、安全运营和成熟 AI Ops。

## 24. 被否决方案及原因

1. **V0.1 只纳管 ECS、V0.2 再迁移**：违反项目负责人明确的 ADR-001，不能作为最终架构。
2. **只建 K8s 影子环境并把生产链路留在 ECS**：只能作为迁移阶段，不满足 V0.1 实际承载要求。
3. **Harbor 放入 Kubernetes**：违反 ADR-003，集群重建时会形成镜像供应链自依赖。
4. **Task CRD**：违反 ADR-008；V0.1 Task 量和查询复杂度不足以抵消 CRD 生命周期、Schema 和升级成本。
5. **xw-opsapi**：V0.1 直接 API 已足够，新增中间层会扩大安全、部署和恢复面。
6. **Promtail/Loki 或 KubeSphere ES Logging**：任务已锁定 Fluent Bit + Loki，ES/Promtail 不进入最终基线。
7. **双 n8n 同时读取生产邮箱**：会带来重复归档、重复发送、状态竞争和不可确定回退，明确禁止。
8. **3 CP + 分布式存储一次性建设**：没有 V0.1 需求证据，增加故障和 AI 运维复杂度；留作后续触发式演进。

## 25. 架构风险

| 风险 | 等级 | 对策 |
|---|---|---|
| 单 Control Plane 故障 | 高 | etcd snapshot、Git 重建、RTO 4h、V0.2 HA 触发条件 |
| ClickHouse 本地盘故障 | 高 | 原生 BACKUP、外部备份、POP3 重放、恢复演练 |
| 迁移后行为差异 | 高 | 回放、golden data、单活切换、观察窗口、回退 |
| 邮箱/DingTalk 凭据错误 | 高 | Secret 分离、测试群、凭据校验、变更审批 |
| Harbor/备份 VM 故障 | 中高 | 独立故障域、离线/不可变副本、恢复 Runbook |
| NetworkPolicy 误配 | 中高 | Git review、连通性测试、默认拒绝、回滚 |
| Agent 误操作 | 高 | Job、最小 RBAC、白名单、L2 审批、审计 |
| 资源不足 | 中 | requests/limits、磁盘阈值、容量巡检 |

## 26. 未决问题

以下事项不阻塞 Baseline，但必须在实施前形成参数记录或人工确认：

1. 现网可提供的 VLAN、IP、DNS、NTP、出口防火墙和备份目标地址。
2. 当前 n8n 数据库形态、工作流导入 API、凭据加密密钥和 Message-ID/UID 状态能力。
3. ClickHouse 实际数据量、每日增长量、可接受 RPO/RTO 和历史导入窗口。
4. 测试群、运维群和生产群的 DingTalk 凭据归属。
5. KubeSphere/Kubernetes 的实施时兼容版本对，必须在 BOM 中冻结。
6. 是否已有企业 NAS/S3；若无，`xw-backup-01` 需要配置外部离线副本。

这些问题不能被 Agent 自行猜测；必须在实施前作为参数和审批记录落入 Git。

## 27. 最终架构结论

### 一句话架构

> **玄武云盾 V0.1 是一套 5 台 VM、单控制面双 Worker 的 KubeSphere 私有云：外置 Harbor 与备份仓库，Calico + local-path + 单一观测栈提供平台边界与恢复能力，DA-SOC 全部核心组件在 `da-soc` Namespace 内单活运行，迁移以回放验证和可回退切换完成，AI 以短生命周期 Job 和 Git Task 受控运维。**

### 最终裁决

- **唯一方案**：采用本文和 `ARCHITECTURE-BASELINE-V0.1.md`。
- **V0.1 必须承载 DA-SOC**：不再接受“仅纳管 ECS”作为最终结论。
- **不修改 `TODO.md`**：下一阶段根据 Baseline 重新生成最终实施 TODO。
- **不直接实施 Kubernetes**：本阶段只完成裁决、Baseline 和 ADR；实施必须以 Baseline 为唯一依据，并经过人工批准。