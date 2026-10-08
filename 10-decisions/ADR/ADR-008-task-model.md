# ADR-008：Task Model —— 轻量 YAML / Markdown 文件（V0.1 不引入 CRD）

| 项 | 内容 |
|---|---|
| **状态** | Accepted（原则接受，本次裁决确认载体、Schema、状态机与流转） |
| **相关** | ADR-007（Agent Runtime）、ADR-006（Backup）、ADR-001 |

---

## 1. 决策

> **玄武云盾 V0.1 的 Task 以"轻量文件"承载：`tasks/YYYY/MM/TASK-<id>.yaml`（机器可读的状态与字段）+ `tasks/YYYY/MM/TASK-<id>.md`（人可读的分析与证据记录），存放在 Git 仓库中，Git 是 Task 的唯一事实源。**
>
> **V0.1 不引入 Task CRD、不引入独立 Task Center 服务、不引入 Task 数据库、不引入 Task Web UI。**

同时裁定：

1. **Task 的创建/更新通过 Git 提交完成**（由平台 n8n 或 Agent Job 执行），提交记录即审计记录。
2. **Task 状态机在 V0.1 采用 7 个状态**（见 §4），状态迁移必须显式记录（谁、何时、依据）。
3. **每个 Task 必须包含完整的证据字段**（可复现的查询语句 + 时间范围），无证据的 Task 视为不合格。
4. **V0.2 根据真实运行经验决定是否演进为 CRD**（触发条件见 §7）。
5. **禁止把 Task 放在与状态无关的系统中**（不引入 Jira/ITSM/自研工单系统）。

---

## 2. 背景

### 2.1 候选方案的分歧（三方对一，第 5 方未定义）

| 方案 | Task 载体 | 表述 |
|---|---|---|
| codebuddy | **Git + Markdown** | `07-aiops/logs/YYYY-MM/YYYYMMDD-<task-id>.md`；每次 Agent 会话产出；"AI Task Center 正式化"列为 V0.2 |
| cursor | **Git** | `TaskCenter[Task Records in Git]`；"写入 Git 或受控日志存储"；未使用 CRD/数据库 |
| kimi | **Git / JSON 文件** | `tasks/YYYY/MM/TASK-XXXX.json`（README §10 schema）+ 钉钉通知；"**文件即状态**，不建 Web UI"；明确"自研 Task Web 平台"属"刻意不用" |
| codex | **未定义** | 仅出现"任务 Schema"（ADR-009「AI Ops Task Schema、L0/L1/L2、审批和 Runbook Gateway」），未说明承载方式 |
| dsh | **Kubernetes CRD（`xuanwu.io/v1alpha1 Task`）** | 用 CRD 一次获得状态机、RBAC、审计、GitOps、无新增组件 |

**3:1 倾向"轻量文件 + Git"，1 份未定义。** 而 ADR-008 明确规定："**V0.1 使用轻量 YAML / Markdown Task；暂不强制 Kubernetes Task CRD；V0.2 再根据真实运行经验决定是否演进为 CRD**"。因此本次裁决采纳**轻量文件方案**，CRD 推迟。

### 2.2 对"CRD 方案"论证的检验（诚实评估）

CRD 方案（dsh）的论证是：自建服务需要数据库、认证、审计、状态存储、GitOps 支持、Agent 读写 SDK，而 CRD 一次全有。

**这一论证是成立的** —— 但它比较的对象是"自建服务"，而不是"轻量文件"。真正的比较是 **CRD vs 轻量文件**：

| 需求 | Task CRD | 轻量 YAML/Markdown + Git |
|---|---|---|
| 状态存储 | etcd | Git |
| 权限 | 集群 RBAC（原生） | Git 仓库权限 + PR（原生） |
| 审计 | K8s Audit 自动记录每次变更 | Git commit history（**同样原生，且更适合人阅读**） |
| 变更历史 | etcd 事件 / audit | `git log -p tasks/…`（**可读性更强**） |
| GitOps | Argo CD 原生 | **本身就是 Git**（不需要同步机制） |
| Agent 读写 | `kubectl` / client-go | 文件读写 + `git commit`（**更简单**） |
| 人类可读性 | 需要 `kubectl get -o yaml` | **直接是 Markdown，可读性最好** |
| 无需新增组件 | ✅（复用 kube-apiserver） | ✅（复用 Git） |
| 查询能力 | 弱（`kubectl get` + 标签选择器） | 弱（grep / Git 检索） |
| 并发写冲突 | 由 etcd/乐观锁处理 | 可能产生 Git 冲突（**需要纪律**） |
| CRD 版本管理 | 需要维护 CRD schema 与升级 | 不需要 |
| 长文本（分析/证据） | 不适合（etcd 对象大小/可读性） | **适合** |

**结论：** 两者都能满足 V0.1。差异在于：

1. **CRD 在天花板更高**（可被控制器 watch、可做复杂选择器、可与 Kubernetes 生态深度集成）；
2. **轻量文件在"人可读 + Git 原生 + 零 schema 维护"上更优**；
3. **V0.1 的 Task 量级是每天个位数到数十** —— 远未触及 CRD 的优势区间；
4. **ADR-008 已明确选择轻量文件**，且 3/5 方案独立得出同一结论（这构成"独立共识"的证据，而非从众）。

**因此裁决：轻量文件。** CRD 的收益（watch、控制器、选择器）在 V0.1 无消费者：V0.1 的触发者是 n8n（不是控制器），消费者是 Agent Job（一次性，读文件即可）与人（读 Markdown）。

### 2.3 必须解决的 Git 并发写问题

轻量文件方案唯一的真实缺陷是**并发写冲突**。V0.1 的处理方式：

| 措施 | 说明 |
|---|---|
| 单一写入者 | Task 的创建与状态更新**只由平台 n8n 与 Agent Job 执行**；人的评论通过 Issue/PR 评论，不直接改文件 |
| 文件粒度 = Task 粒度 | 一个 Task 一个文件，避免多 Task 争抢同一文件 |
| 写入幂等 | 更新前先 `git pull --rebase`；按 Task ID 命名避免重名 |
| 冲突处理 | rebase 失败 → 重试（最多 3 次）→ 仍失败则告警并转人工 |
| 顺序保证 | 同一 Task 的状态更新**串行化**（n8n 侧按 Task ID 加锁/队列）；Agent Job 不使用 `backoffLimit > 0`（ADR-007） |
| 每日归档 | 每日快照 Tag（`tasks/YYYY/MM/` 目录 + 每日 tag），便于审计与差异定位 |

---

## 3. Task 载体与目录结构

```text
tasks/
├── README.md                       # Task 规范说明（Schema 与状态机）
├── TEMPLATE.yaml                   # 机器可读模板
├── TEMPLATE.md                     # 人可读模板
├── 2026/
│   └── 02/
│       ├── TASK-20260201-0001.yaml
│       ├── TASK-20260201-0001.md
│       ├── TASK-20260201-0002.yaml
│       └── TASK-20260201-0002.md
└── index/                          # 可选：按状态/资产生成的索引（由脚本生成，不手工维护）
    ├── open.yaml
    └── by-asset.yaml
```

**命名规范：** `TASK-<YYYYMMDD>-<NNNN>`，其中 `NNNN` 为当日顺序号（4 位，从 0001 开始）。

**为什么 YAML + Markdown 双文件：**

- `.yaml` 面向**机器**（状态机、字段、审批记录、证据引用）——可被脚本与 Agent 稳定解析；
- `.md` 面向**人**（现象描述、分析推理、证据摘录、计划与影响说明）——审计与复盘时人直接阅读；
- 两者由同一个 Task ID 关联；`.md` 允许自由文本，`.yaml` 保持结构稳定。

---

## 4. Task Schema 与状态机

### 4.1 `.yaml` Schema（V0.1 字段全集）

```yaml
# tasks/2026/02/TASK-20260201-0007.yaml
task_id: TASK-20260201-0007
created_at: "2026-02-01T09:12:33+08:00"
created_by: n8n-platform                  # n8n-platform | dingtalk | alertmanager | human:<id>
schema_version: "1.0"

# 来源与对象
source: alertmanager                      # alertmanager | schedule | dingtalk | manual | harbor | argo
trigger_ref: "ALERT-A6-BackupStale"       # 触发源引用（告警名/定时任务名/消息 ID）
asset:
  kind: BackupJob                         # Node | Pod | Deployment | BackupJob | Certificate | Registry | Cluster
  name: velero-daily-da-soc
  namespace: xw-ops
  node: ""

# 严重性与影响
severity: high                            # info | low | medium | high | critical
impact: "da-soc 命名空间无 26 小时内可用备份；若发生数据损坏将无法恢复到昨日状态"
affected_business: ["da-soc"]

# 风险分级（按 ARCHITECTURE-BASELINE §19 的确定性规则推导，不得由 Agent 自行判定）
risk_level: L1                            # L0 | L1 | L2
risk_rule: "R3: 可逆变更 ∧ tier=B ∧ 非业务命名空间关键路径"
requires_approval: false
approval_required_reason: ""

# 证据（可复现引用 —— 强制字段）
evidence:
  - source: prometheus
    query: 'time() - velero_backup_last_successful_timestamp{schedule="daily-da-soc"}'
    time_range: "2026-02-01T00:00:00+08:00/2026-02-01T09:00:00+08:00"
    result_summary: "last success 34h ago (threshold 26h)"
  - source: kubectl
    query: 'kubectl -n xw-ops get backups.velero.io -o json'
    result_summary: "Backup daily-da-soc-20260131020000 phase=PartiallyFailed"
  - source: loki
    query: '{namespace="xw-ops", app="velero"} |= "error"'
    result_summary: "S3 timeout ×3"

# 分析与计划
analysis: "备份部分失败，原因为 MinIO 连接超时；未产生可用恢复点。"
plan:
  - step: 1
    action: "验证 MinIO 可达性与容量"
    method: "curl MinIO health endpoint"
    expected: "200 OK"
  - step: 2
    action: "重新触发备份 Job"
    method: "kubectl create job --from=cronjob/velero-daily-da-soc"
    expected: "Backup phase=Completed"
  - step: 3
    action: "验证备份时效指标 < 26h"
    method: "prometheus query"
    expected: "true"
preconditions:
  - "MinIO bucket 可用空间 > 20%"
dry_run_result: "n/a（重建 Job 无破坏性）"
rollback_plan: "无需回滚（幂等操作）；若新 Job 失败则保留原 CronJob 不动并升级人工"

# 审批
approvals: []                             # L2 时填入 {by, role, at, channel, message_id, decision, comment}
expires_at: "2026-02-01T13:12:33+08:00"   # L2 超时时间（默认 4h）

# 执行
execution:
  agent_job: agent-task-20260201-0007
  agent_image_digest: "sha256:…"
  service_account: sa-agent-executor
  started_at: "2026-02-01T09:15:02+08:00"
  finished_at: "2026-02-01T09:17:41+08:00"
  result: succeeded                       # succeeded | failed | aborted | timed_out
  commands:
    - "kubectl -n xw-ops create job backup-retry-0007 --from=cronjob/velero-daily-da-soc"
  artifacts: ["/work/evidence.json", "/work/plan.md"]

# 验证（强制；无法写出验证方法的动作不得执行）
verification:
  method: "prometheus: velero_backup_last_successful_timestamp 年龄 < 26h ∧ phase=Completed"
  performed_at: "2026-02-01T09:22:10+08:00"
  result: passed                          # passed | failed | inconclusive
  detail: "last success age = 8m; phase=Completed"

# 结果与收尾
status: Closed                            # 见 §4.2 状态机
status_history:
  - { at: "2026-02-01T09:12:33+08:00", from: "", to: New, by: n8n-platform, reason: "告警触发" }
  - { at: "2026-02-01T09:15:00+08:00", from: New, to: Executing, by: sa-agent-executor, reason: "L1 自动执行" }
  - { at: "2026-02-01T09:22:10+08:00", from: Executing, to: Verifying, by: sa-agent-executor, reason: "执行成功" }
  - { at: "2026-02-01T09:23:00+08:00", from: Verifying, to: Closed, by: sa-agent-auditor, reason: "验证通过" }
lessons: "MinIO 单点导致备份偶发超时；建议 V0.2 增加备份重试与告警联动"
related: ["TASK-20260131-0012"]
```

### 4.2 状态机（V0.1 共 7 个状态）

```text
                 ┌──────────┐
   触发 ────────► │   New    │
                 └────┬─────┘
                      │ Agent 读取上下文并分析
                      ▼
                 ┌──────────┐
                 │Analyzing │
                 └────┬─────┘
                      │ 生成计划 + 风险分级
                      ▼
                 ┌──────────┐
        ┌───────►│ Planned  │
        │        └────┬─────┘
        │             │ requires_approval = true        requires_approval = false
        │             ▼                                        │
        │      ┌────────────────┐                              │
        │      │AwaitingApproval│                              │
        │      └───────┬────────┘                              │
        │              │ 人工批准（DingTalk）                   │
        │              ▼                                       ▼
        │        ┌───────────┐                          ┌───────────┐
        └────────┤ Executing │◄─────────────────────────┤ Executing │
   验证失败/回滚  └─────┬─────┘                          └─────┬─────┘
                       │                                      │
                       ▼                                      ▼
                 ┌──────────┐                            ┌──────────┐
                 │ Verifying│───────────────────────────►│ Verifying│
                 └────┬─────┘                            └────┬─────┘
                      │ 通过                                  │ 通过
                      ▼                                       ▼
                 ┌──────────┐                            ┌──────────┐
                 │  Closed  │                            │  Closed  │
                 └──────────┘                            └──────────┘

  任何阶段的失败/超时 → Failed / RolledBack / TimedOut（终态）
  L2 超时未批准      → Expired（终态）
```

| 状态 | 含义 | 允许的下一状态 |
|---|---|---|
| `New` | 已创建，等待 Agent 处理 | `Analyzing`, `Failed` |
| `Analyzing` | Agent 正在读取证据与上下文 | `Planned`, `Failed` |
| `Planned` | 计划已生成，风险等级已确定 | `AwaitingApproval`（L2）, `Executing`（L0/L1） |
| `AwaitingApproval` | 等待人工审批 | `Executing`, `Expired`, `Failed` |
| `Executing` | 正在执行 | `Verifying`, `Failed`, `RolledBack` |
| `Verifying` | 正在验证结果 | `Closed`, `RolledBack`, `Failed` |
| `Closed` | 成功关闭（终态） | — |
| `Failed` | 执行或验证失败（终态，需人处理） | — |
| `RolledBack` | 已回滚（终态） | — |
| `Expired` | L2 审批超时（终态） | — |
| `TimedOut` | Job 超时终止（终态） | — |

**V0.1 核心 7 态：** `New` → `Analyzing` → `Planned` → (`AwaitingApproval`) → `Executing` → `Verifying` → `Closed`，其余为异常分支终态。

---

## 5. Task 流转（谁在什么时候写什么）

| 步骤 | 执行者 | 动作 | 写入 |
|---|---|---|---|
| 1 | Platform n8n | 收到告警/定时/指令 → 生成 Task | `New` 状态；`source`/`trigger_ref`/`asset`/`severity`/`impact` |
| 2 | Platform n8n | 按 baseline §19 规则**推导风险等级**（不由 Agent 判定） | `risk_level` / `risk_rule` / `requires_approval` |
| 3 | Platform n8n | 创建 Agent Job（绑定对应 SA） | 拉起 `agent-task-<id>` |
| 4 | Agent Job | 收集证据（K8s/Prometheus/Loki/Harbor/Velero/Git） | `evidence[]`（**强制可复现引用**） |
| 5 | Agent Job | 分析 + 生成计划 + 前置条件 + dry-run + 回滚方案 | `analysis` / `plan` / `preconditions` / `rollback_plan` |
| 6a | Platform n8n → DingTalk | L2：发送审批卡片（含计划、影响、回滚、证据链接） | `AwaitingApproval` |
| 6b | 人（DingTalk 批准） | 批准/拒绝 | `approvals[]`（by/role/at/channel/message_id/decision/comment） |
| 7 | Agent Job（executor SA） | 执行白名单动作 | `execution.*` |
| 8 | Agent Job / 独立验证 | 验证（独立方法） | `verification.*` |
| 9 | Agent Job（auditor SA） | 审计交叉比对（Task vs K8s Audit vs 容器日志） | `evidence[]` 追加审计引用；`status_history` |
| 10 | Platform n8n | 通知（完成/失败/需人工） + 更新索引 | `status` / `lessons` / 钉钉消息 |
| 11 | 人（复盘） | 填写经验教训（可选） | `lessons` |

**关键纪律：**

1. **风险等级不由 Agent 判定，而由规则推导**；Agent 若认为规则不适配，只能提出 ADR 提案，不得自行降级。
2. **证据必须可复现**（查询语句 + 时间范围 + 结果摘要），不允许只写结论。
3. **L2 审批记录必须包含消息 ID**（可回溯到 DingTalk 原始消息）。
4. **验证必须使用独立方法**（不是"命令返回 0"）。
5. **审计必须交叉**（Task 记录 ↔ K8s Audit ↔ 容器日志）。

---

## 6. Task 与其它系统的关系

| 系统 | 关系 |
|---|---|
| **Git** | Task 的唯一事实源；commit history 即审计记录 |
| **Kubernetes Audit** | 交叉验证来源：Agent 的每次集群请求都在审计日志中 |
| **DingTalk** | 通知与审批通道；审批结果回写 Task |
| **Prometheus / Loki** | 证据来源（通过引用而非复制原始数据） |
| **Velero / Harbor API** | 证据来源与执行对象（白名单动作） |
| **Argo CD** | 不存在直接关系（Task 不改定义态；定义态变更走 Git PR + Argo CD 同步）——但 Task 可**引用** Argo CD 的同步状态作为证据 |
| **TODO.md** | **无自动同步关系**。TODO 是实施期任务清单（给人和 Agent 的建设任务），Task 是运行期运维任务。两者不混用，避免语义混乱。 |

---

## 7. 演进为 CRD 的触发条件（V0.2 评估）

满足**任一**条件时，在 V0.2 重新裁决是否演进为 CRD：

1. **Task 吞吐量**持续超过每天约 50 条，Git 提交产生可感知的噪声与冲突；
2. 出现**必须 watch Task 状态**的消费者（如真正的控制器、实时看板、自动升级机制）；
3. 出现**跨 Task 的复杂查询/聚合**需求（Git/脚本检索无法满足）；
4. 出现**多个并写者**（多个 n8n / 多个 Agent 同时更新）导致 Git 冲突频繁；
5. 需要**细粒度的 Kubernetes RBAC** 区分"谁能改 Task 的哪些字段"。

**演进方式（若触发）：** Task CRD 与 Git Task **并存**——CRD 承载运行态状态机，Git 保留人可读的分析与证据记录（或定期从 CRD 导出 Markdown）。**不允许"双写不一致"**：任一时刻必须明确哪一侧是权威（建议：CRD 为运行态权威，Git 为归档与证据权威，单向导出）。

---

## 8. 被否决的方案及原因

| 被否决 | 原因 |
|---|---|
| **Task CRD（dsh 方案）** | 非错误方案，但 V0.1 的 Task 量级与消费者特征无法发挥其优势；引入 CRD schema 维护成本；长文本（分析/证据）不适合放 etcd；**ADR-008 已明确 V0.1 用轻量文件、V0.2 再评估**。其"复用平台原生能力"的思想已被吸收（复用 Git 而非自建服务）。 |
| 独立 Task Center 服务（自建 API + 数据库 + 认证） | 违反"不建平台中的平台"；引入第二套认证、审计与状态存储；kimi 明确将"自研 Task Web 平台"列入"刻意不用" |
| Task 数据库（关系型/文档型） | 同样的自建服务问题；且失去 Git 的原生审计与可读性 |
| Task Web UI | V0.1 无需求；`git log`、Grep、Markdown 阅读已足够；V0.4 再评估 |
| 引入 Jira / ITSM / 工单系统 | 引入外部依赖、第二套权限与状态机；V0.1 禁止"完整 ITSM" |
| 把 Task 放在 Loki 日志里（仅靠日志，无结构化文件） | 状态机与审批记录需要可靠的结构化载体；日志不适合作为状态权威 |
| 把 Task 与 TODO.md 合并 | 运行期运维任务与建设期任务语义不同；合并会导致 TODO 无法收敛 |
| 不做 Task，直接让 Agent 输出结论 | 失去证据链、审批记录与可复盘性，AI-Native 闭环无法验证 |

---

## 9. 后果与影响

**正面：** 零新增组件；Task 天然 Git 化、可评审、可 diff、可回溯；人可读性最好；与 Agent Job 的交互简单（文件读写）；审计直接复用 Git history。

**负面 / 代价：**

1. **查询与聚合能力弱**（依赖脚本/Grep），随着 Task 累积需要索引脚本；
2. **并发写需要纪律**（单一写入者 + 串行化 + rebase 重试）；
3. **无实时事件/watch 能力**（无法被控制器驱动）；
4. 长生命周期 Task 的字段演进需要 `schema_version` 管理。

**对后续版本的影响：** V0.2 按 §7 触发条件评估 CRD 演进；V0.4 评估 Web UI 与多 Agent 共享 Task 的机制。**无论载体如何演进，"证据必须可复现、审批必须有记录、验证必须独立、审计必须可交叉"这四条纪律不得放宽。**
