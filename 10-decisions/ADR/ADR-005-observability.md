# ADR-005：Observability 架构 —— Prometheus + Alertmanager + Grafana + Fluent Bit + Loki（单栈）

| 项 | 内容 |
|---|---|
| **状态** | Accepted（原则接受，本次裁决确认具体实现与"单栈"落点） |
| **相关** | ADR-001、ADR-006、ADR-007（Agent 读取观测数据）、ADR-002 |

---

## 1. 决策

> **玄武云盾 V0.1 只有一套指标栈和一套日志栈：**
>
> - **指标：** `kube-prometheus-stack`（Prometheus Operator + Prometheus + Alertmanager + Grafana + node-exporter + kube-state-metrics）作为**唯一指标实现**。
> - **日志：** `Fluent Bit`（DaemonSet）→ `Loki` 作为**唯一日志实现**；Grafana 作为两者唯一的可视化入口。
> - **告警：** Alertmanager → 平台 n8n → DingTalk（玄武云盾运维群）。**业务日报群与运维告警群严格分离。**
>
> **明确不做：** Elasticsearch / OpenSearch / ELK、Kibana、第二套 Prometheus、KubeSphere 自带监控与日志组件、Traces、SIEM。

同时裁定：

1. **KubeSphere 自带监控（`monitoring`）与日志（`logging`）组件必须关闭**，避免与集群侧实现形成双栈。KubeSphere 只作为**管理面 UI / RBAC / 用户管理入口**。
2. **日志保留：** 容器与节点日志 30 天；**Kubernetes Audit 日志 90 天**。
3. **指标保留：** Prometheus 15 天（V0.1 明确声明，不足以做长期趋势分析，属有意取舍）。
4. **Agent 通过只读凭据直接查询 Prometheus / Loki / Alertmanager / Kubernetes API**，不引入中间聚合层（见 ADR-008 与 baseline §16）。
5. **业务指标（DA-SOC 链路）必须纳入监控**，且**不得由 LLM 生成数字**（只做引用）。

---

## 2. 背景

### 2.1 候选方案的分歧（本次裁决的焦点之一）

| 方案 | 指标 | 日志 | 判定 |
|---|---|---|---|
| codebuddy | Prometheus（KubeSphere 可插拔）+ Grafana + Alertmanager | **Loki + Promtail** | 单栈 ✅（日志用 Promtail） |
| codex | KubeSphere / Prometheus / Grafana / Alertmanager | **Fluent Bit + Loki** | 单栈 ✅（与 ADR-005 一致） |
| cursor | 优先复用 KubeSphere Monitoring | KubeSphere Logging 或 Loki（二选一） | 单栈但依赖 KubeSphere 组件 ⚠️ |
| dsh | kube-prometheus-stack | Loki + Promtail | 单栈 ✅ |
| kimi | **KubeSphere 内置** Prometheus/Alertmanager（明确不用 kube-prometheus-stack） | **Fluent Bit → Elasticsearch 单节点** | 指标单栈 ✅；**日志用 ES，与 ADR-005"避免 ELK"冲突** ❌ |

**分歧点有两个：**

**分歧 A：指标栈用 KubeSphere 内置，还是独立 `kube-prometheus-stack`？**

| 维度 | KubeSphere 内置监控 | kube-prometheus-stack |
|---|---|---|
| 组件数量 | 少（复用 KS 安装） | 多（Prometheus Operator + CRD） |
| 与 KubeSphere 解耦 | ❌ 强耦合（关掉 KS 监控就没有监控） | ✅ 独立 |
| 告警规则管理 | 依赖 KubeSphere 界面/配置 | PrometheusRule CRD，**Git 友好** |
| Agent 可操作性 | 需经 KubeSphere API | 直接 Prometheus HTTP API + CRD |
| 长期可维护性 | 中（升级路径受 KS 约束） | 高（生态事实标准） |
| 迁移成本 | 若将来弃用 KubeSphere，需重建监控 | 无 |

**裁决：采用 `kube-prometheus-stack`，关闭 KubeSphere 自带监控。**

理由：① 与 ADR-005 的"Prometheus + Grafana + Alertmanager"表述一致；② 告警规则以 `PrometheusRule` CRD 表达，可进 Git、可被 Argo CD 管理、可被 Agent 读取——直接服务于"Git 即事实源"与"AI 可维护"两项原则；③ 监控是平台的核心能力，不应与 KubeSphere 的存续强耦合；④ 4/5 方案最终都指向"集群侧独立 Prometheus 栈"或等价物。

**代价：** 失去 KubeSphere 控制台的内建监控视图；改由 Grafana 承担（在 baseline 中明确 KubeSphere 的原生监控面板不可用）。

**分歧 B：日志栈用 Fluent Bit + Loki，还是 Fluent Bit + Elasticsearch？**

**裁决：采用 Fluent Bit + Loki。** 理由：

1. ADR-005 明确规定"避免 Elasticsearch/ELK"、"避免多套日志系统"；
2. ES 单节点在 V0.1 的资源占用与运维负担（JVM 堆、索引生命周期、分片、磁盘水位）远高于 Loki，而 V0.1 的检索需求（按命名空间/Pod/时间/关键词、审计按 SA/verb 检索）用 LogQL 完全覆盖；
3. Loki 的标签模型天然契合"按 namespace/pod/container/job 检索"，而审计检索（按 user/verb）可通过结构化日志字段 + LogQL 过滤实现；
4. kimi 自己也把"Fluent Bit → ES"列为**需要在资源不足时降级**的方案，说明其团队对 ES 资源消耗本就有顾虑——本裁决直接把降级方案升级为主方案。

**关于采集器的说明：** 4 份方案中 3 份用 Promtail、2 份用 Fluent Bit。本裁决采用 **Fluent Bit**，理由：① ADR-005 指定 Fluent Bit；② Fluent Bit 是 CNCF 毕业项目，资源占用低、插件生态广，且能同时采集 **journald + 容器 stdout + 节点文件（审计日志）**；③ Promtail 已进入维护/替代阶段（Grafana 官方转向 Alloy），采用 Fluent Bit 可避免 V0.1 就选一个生命周期末期的采集器。**代价：** 配置语法（Lua/过滤器）学习成本略高于 Promtail — 通过 Git 化的 ConfigMap 与 Runbook 消化。

### 2.2 必须遵守的一条纪律

> **同一能力域只允许存在一套实现。** 出现第二套监控或第二套日志，即为架构违规，必须在评审中修正（不许"两套都留着，看哪套好用"）。

---

## 3. 指标架构

### 3.1 组件与部署

| 组件 | 形态 | 部署位置 | 说明 |
|---|---|---|---|
| Prometheus Operator | Deployment | `xw-obs` | 管理 Prometheus/Alertmanager/Rule 的 CRD |
| Prometheus | StatefulSet（1 副本） | `xw-obs`（固定 `xw-wk-02`） | 保留 15 天，PVC 100 GiB |
| Alertmanager | StatefulSet（1 副本） | `xw-obs` | 分组、抑制、静默；webhook → 平台 n8n |
| Grafana | Deployment | `xw-obs` | 唯一可视化入口（指标 + 日志） |
| node-exporter | DaemonSet | `kube-system`/`xw-obs` | 节点指标 |
| kube-state-metrics | Deployment | `xw-obs` | Kubernetes 对象状态 |
| blackbox-exporter | Deployment | `xw-obs` | HTTP/TCP 探测（DA-SOC 链路健康、管理面可用性） |
| Pushgateway | Deployment | `xw-obs` | 接收批处理类指标（**备份 Job、DA-SOC 日报执行结果**） |

### 3.2 监控内容（六层，全部必须有）

| 层次 | 监控项 |
|---|---|
| **Infrastructure** | 节点 Up/Down、CPU 使用与饱和、内存可用、**磁盘使用率与 24h 预测耗尽**、inode、磁盘 I/O 延迟、网络吞吐与丢包、**NTP 时间偏移**、系统负载 |
| **Kubernetes** | 节点 Ready/NotReady、控制面组件健康（apiserver/etcd/scheduler/controller-manager）、etcd leader/DB 大小/fsync/快照成功、apiserver 延迟与 5xx、**证书到期**、Pod Pending、调度失败、PVC/PV 状态、Deployment 可用副本 |
| **Pod / Application** | CrashLoopBackOff、重启次数、OOMKilled、ImagePullBackOff、readiness 失败、资源使用 vs requests/limits、ResourceQuota 使用率、驱逐事件 |
| **Platform** | Harbor 可用与磁盘、Loki 写入成功率、Prometheus 自身、**备份 Job 成功状态与时效**、Velero/etcd 快照状态、Argo CD 同步状态与漂移 |
| **Business（DA-SOC）** | **数据新鲜度**（ClickHouse 最新数据日期 / 最近一次 `/archive` 成功时间）、日报按日产出结果、`/archive` 成功率、`/render` 成功率、DingTalk 发送结果、ClickHouse 可用与表行数、**IMAP 未读计数（验证"未标已读"）** |
| **Security / Audit** | 审计日志管道存活、**异常 privileged / hostNetwork 对象检测**、非 Harbor 来源镜像引用检测、Secret 读取异常、apiserver 认证失败率、Harbor 高危漏洞计数 |

### 3.3 业务指标如何进入监控（关键设计，不得违反 DA-SOC 纪律）

**约束：** 平台**不得**修改 DA-SOC 镜像，**不得**让 LLM 生成业务数字。

**方案：** 在 `da-soc` 命名空间部署一个**只读 metrics adapter（CronJob，每 5 分钟）**：

```
只读查询 ClickHouse（da_soc_ro 账号）：
  SELECT max(data_date), count(*) FROM ...      → 数据新鲜度、行数
读取 n8n API：
  最近执行状态、最近成功时间                     → 链路健康
推送到 Pushgateway                              → 平台侧抓取
```

**边界纪律（必须写进 Runbook 与验收）：**

1. adapter **只读**（使用 `da_soc_ro` 只读账号，无写权限）；
2. adapter **不参与业务链路**，失败不影响日报；
3. adapter **不产生**任何业务数字，只上报"是否存在数据、最近成功时间、行数"等**观测元数据**；
4. adapter 输出的指标**不得**用于生成日报图或替代 SQL 出数。

### 3.4 告警规则（V0.1 共 12 条）

| # | 告警 | 条件 | 等级 |
|---|---|---|---|
| A1 | NodeNotReady | Ready=False > 5 分钟 | critical |
| A2 | DiskWillFillIn24h | `predict_linear(node_filesystem_avail_bytes[6h], 86400) < 0` | critical |
| A3 | DiskSpaceLow | 可用 < 15% | warning |
| A4 | EtcdUnhealthy / SnapshotStale | etcd 无 leader > 1 分钟；或最新快照 > 2 小时 | critical |
| A5 | ApiServerUnavailable / 5xx | 不可用 > 1 分钟；或 5xx 率 > 1% | critical |
| A6 | PodCrashLoop | 重启增长 > 3 次/15 分钟 | warning |
| A7 | **BackupFailed / BackupStale** | 任一备份失败；或最新备份 > 26 小时 | critical |
| A8 | CertExpiringIn30d | 证书剩余 < 30 天 | warning |
| A9 | NtpOffsetHigh | 时间偏移 > 1 秒 | warning |
| A10 | **DASOCNoFreshData** | 业务时限前无当日数据；或最近 archive 成功 > 26 小时 | critical |
| A11 | **DASOCChainDegraded** | `/archive` 或 `/render` 连续失败；或 n8n 执行失败 | critical |
| A12 | PlatformComponentDown | Harbor / Loki / Prometheus / Argo CD 不可用 | warning |

**纪律：**

1. **每条告警必须能回答三个问题**：谁处理、怎么处理（Runbook 链接）、能否自动化。**没有 Runbook 的告警不允许上线。**
2. **A10/A11 是业务告警**，其价值高于任何平台告警——它们直接保护"确定性日报"这一业务价值，且**不需要 LLM 参与**。
3. 告警必须携带：对象、影响、证据链接（Grafana/Loki）、Runbook、风险等级、是否需要审批。

---

## 4. 日志架构

### 4.1 组件与采集范围

| 日志源 | 采集方式 | Loki 标签 | 保留 |
|---|---|---|---|
| 节点 journald（ssh/kernel/systemd/chrony/auditd） | Fluent Bit `systemd` 输入 | `{job="journal", host=…}` | 30 天 |
| 容器 stdout/stderr | Fluent Bit `tail` + Kubernetes 元数据 | `{namespace, pod, container}` | 30 天 |
| **Kubernetes Audit** | Fluent Bit `tail` 读控制面 `/var/log/kubernetes/audit/audit.log` | `{job="k8s-audit"}` | **90 天** |
| KubeSphere 控制台操作审计 | 定期导出 + Fluent Bit tail | `{job="ks-audit"}` | 90 天 |
| 集群外组件（Harbor / MinIO） | 宿主机 Fluent Bit/journald | `{job="harbor"}` / `{job="minio"}` | 30 天 |
| n8n 执行日志 | 容器 stdout（已覆盖） | `{namespace="da-soc", app="n8n"}` | 30 天 |

### 4.2 Kubernetes Audit 策略

| 项 | 裁定 |
|---|---|
| 默认级别 | `Metadata`（记录 who/what/when/result，不记录请求体） |
| 提升级别 | 对 `Secret`、`RBAC`、`NetworkPolicy`、`ResourceQuota`、`Pod/exec`、`Pod/portforward` 等敏感资源记 `RequestResponse` |
| 输出 | 控制面节点文件（`--audit-log-path`），带轮转（`maxsize`/`maxbackup`/`maxage`） |
| 汇聚 | Fluent Bit → Loki，保留 90 天 |
| **禁止** | 记录 Secret 的明文内容（`RequestResponse` 对 Secret 只记元数据；避免审计日志本身成为泄露源） |
| 用途 | ① 审计（谁改了什么）；② **Agent 操作审计**（按 ServiceAccount 过滤）；③ 安全验证（越权测试证据） |

### 4.3 脱敏纪律

**日志中不得出现：** 生产邮箱原文、DingTalk token 与群 ID、Secret 值、ClickHouse 密码、LLM API Key、业务数据明细。

**实现：** ① 应用侧不打印凭据（DA-SOC 工作流与 adapter 必须遵守）；② Fluent Bit 侧配置过滤规则对疑似 token/邮箱/密码模式做替换；③ 定期抽样检查；④ **明确禁止把生产邮箱原文送入 AI 上下文**。

### 4.4 明确不引入

Elasticsearch / OpenSearch / Kibana / Logstash、Fluentd（避免双采集器）、第二套 Loki、独立日志平台、Traces（DA-SOC 链路短，V0.1 无收益）。

---

## 5. 告警到人的通路

```text
Prometheus Rule 触发
   ↓
Alertmanager（分组 / 抑制 / 静默）
   ↓  webhook（集群内可达平台 n8n）
平台 n8n（xw-ops）
   ├─ critical → DingTalk「玄武云盾运维群」（立即 @负责人）
   ├─ warning  → DingTalk（15 分钟批量聚合）
   └─ info     → 仅入 Grafana 与 Task 记录（不打扰人）
   ↓
创建 Task 记录（Git）+ 携带证据链接（Grafana/Loki）
   ↓
（如需）触发 Agent Job 分析
```

**纪律：**

1. **业务日报群与运维告警群严格分离。** 平台告警**绝不**发往 DA-SOC 业务日报群。
2. 告警消息必须包含：对象、影响、证据链接、Runbook 链接、是否需审批。
3. **n8n 只做传输与触发，不代替 Agent 判断根因**，也不承担分析职责。
4. 告警风暴抑制：同类告警 15 分钟内聚合；维护窗口用静默（silence），并记录原因。

---

## 6. Agent 如何读取观测数据

| 信息 | 通道 | 权限 |
|---|---|---|
| 集群对象状态 | Kubernetes API（`kubectl get -o json`） | `sa-agent-readonly`（cluster 只读，**无 Secret 读权限**） |
| 指标 | Prometheus HTTP API（`/api/v1/query`、`/query_range`） | 只读，集群内可达 |
| 日志与审计 | Loki HTTP API（LogQL） | 只读 |
| 当前告警 | Alertmanager API | 只读 |
| 备份状态 | Velero CR + MinIO 对象列表 | 只读 |
| 镜像与漏洞 | Harbor REST API | robot account 只读 |
| 定义与知识 | Git（Runbook / ADR / 架构 / 资产） | 只读 |

**关键纪律（来自 ADR-001 与 DA-SOC 约束）：**

1. Agent **只能读**观测数据，**不能**写 Prometheus/Loki/Alertmanager。
2. Agent **不得**直接连生产邮箱、**不得**直接改 ClickHouse 数据、**不得**直接发钉钉日报。
3. Agent 的每一次观测都必须写入 Task / Audit 记录（证据引用），使"Agent 当时看到了什么"可复盘。
4. **审计可检索性要求：** 事后任何人（或审计 Agent）仅凭 **Loki（审计日志）+ Git（Task 与定义）**，能够重建"Agent 为什么这么做"。

---

## 7. 被否决的方案及原因

| 被否决 | 原因 |
|---|---|
| Fluent Bit → Elasticsearch（kimi） | 违反 ADR-005"避免 ELK"；资源与运维负担显著高于 Loki，而 V0.1 检索需求 Loki 完全覆盖 |
| KubeSphere 自带监控作为唯一指标栈（kimi 主张） | 与 KubeSphere 强耦合（关掉 KS 就没有监控）；告警规则无法以 Git 友好的 CRD 表达；不利于 Agent 读取与长期可维护 |
| 使用 KubeSphere 自带日志组件 | 与自建 Fluent Bit 形成双采集器；且其存储后端为 ES |
| 使用 Promtail（3 份方案采用） | 生命周期末期，V0.1 就选一个即将被替代的采集器不划算；Fluent Bit 能力等价且更通用 |
| 双栈并存（"两套都留着看哪套好用"） | 违反单栈原则；直接导致告警污染、资源翻倍、Agent 无法判断以哪套为准 |
| 引入 Traces / SIEM | V0.1 无需求；DA-SOC 链路短；SIEM 属 V0.3 |
| 引入中间聚合层供 Agent 观测（`xw-opsapi`） | 见 ADR-008 与 baseline §16：5 份方案中仅 1 份提出；V0.1 用只读 RBAC + 直接 API 即可满足需求，新增层增加组件与单点 |

---

## 8. 后果与影响

**正面：** 单栈清晰（一套指标、一套日志、一个 Grafana 入口）；规则与看板全部 Git 化；Agent 可直接用标准 HTTP API 读取；与 KubeSphere 解耦；审计与业务指标统一在同一条管道。

**负面 / 代价：**

1. KubeSphere 控制台的原生监控视图不可用（需在文档与培训中明确）；
2. Prometheus 仅保留 15 天，趋势分析受限（V0.1 有意取舍；长期趋势属 V0.2+ 议题）；
3. Fluent Bit 配置复杂度高于 Promtail（由 Git 化 ConfigMap + Runbook 消化）；
4. 日志存储容量需人工规划（Loki 非 HA，单副本）。

**对后续版本的影响：** V0.2 评估 Prometheus 长期存储（Thanos/远端写）、日志保留延长、Grafana Alloy 迁移；V0.3 引入安全事件关联（SIEM-lite）。在此之前，**不得新增第二套指标或日志实现**。
