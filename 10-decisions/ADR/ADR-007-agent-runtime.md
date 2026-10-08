# ADR-007：AI Agent Runtime —— 短生命周期 Kubernetes Job

| 项 | 内容 |
|---|---|
| **状态** | Accepted（原则接受，本次裁决确认运行形态、权限模型、触发与执行通道） |
| **相关** | ADR-001、ADR-005、ADR-006、ADR-008（Task Model） |

---

## 1. 决策

> **玄武云盾 V0.1 的 Agent 以"短生命周期 Kubernetes Job"形态运行：由平台 n8n（或 Alertmanager 经 n8n）触发，Pod 启动 → 读取 Task 与上下文 → 分析/规划 → （必要时）等待人工审批 → 执行 → 验证 → 写回 Task 与审计 → Job 结束即销毁。**
>
> **明确不做：** 常驻 Agent Pod、常驻 Agent 平台（多 Agent 编排框架）、Agent 通用 SSH、Agent 常驻高权限凭据、Agent 直接读业务 Secret、Agent 触碰业务数据或业务数字。

同时裁定：

1. **Agent 身份为 3 个 ServiceAccount**（`sa-agent-readonly` / `sa-agent-executor` / `sa-agent-auditor`），禁止共用人类账号。
2. **只读是默认**；写操作只允许在**白名单命名空间 + 白名单资源/动词**内，且必须可验证、可回滚。
3. **L1 自动执行、L2 人工审批**；不可逆操作一律 L2，且默认由人执行。
4. **Job 必须设置 `activeDeadlineSeconds`、`ttlSecondsAfterFinished`、非 root、只读根文件系统、资源 limits、`automountServiceAccountToken: true` 但 SA 权限最小。**
5. **不引入 `xw-opsapi` 或任何自研只读聚合层**（见 §6）。
6. **Agent 不参与 DA-SOC 业务数字的生成**（ADR-001 §6 的硬约束）。

---

## 2. 背景

### 2.1 候选方案的分歧（5 份方案 5 种形态）

| 方案 | Agent 形态 | 权限模型 | 评价 |
|---|---|---|---|
| codebuddy | **IDE/CLI 型 Agent（人工触发）** + `xw-opsapi` + 受限 kubeconfig | read / exec 两个 SA；exec 只能调 opsapi 白名单动作 | 形态合理；但引入自研中间层，且"人工触发"难以形成可重复的自动化闭环 |
| codex | **未明确形态**；"单 Agent Manager 的逻辑模块" + Runbook Gateway | ReadOnly / Executor 分离；Executor 只能调 Runbook Gateway | 权限设计优秀（白名单网关思想）；形态缺失 |
| cursor | **未明确形态**（Ops Agent = LLM + Tools） | `xuanwu-agent-l0`（多 NS 只读）+ `xuanwu-agent-l1`（限定 NS 写） | 分级清晰；形态缺失 |
| dsh | **短生命周期 Job + Task CRD** | 3 个 SA；无 SSH | 形态明确、可审计、可复现 |
| kimi | **AI Agent Runner**（部署在 `xw-ops`，workload 形态未提及） | 集群只读 + `xw-ops` 内 L1 白名单写 + 无 secret 读 + 无 SSH | 权限边界最严格（含"通道不存在"思想） |

**共识（5/5）：** ① 只读与执行权限分离；② 无 SSH；③ 白名单动作；④ L2 人工审批；⑤ 不可逆操作禁止自动执行；⑥ 不建常驻多 Agent 平台。

**分歧：** 运行形态（Job / IDE-CLI / 未定义）、是否引入自研中间层（仅 codebuddy 提出 `xw-opsapi`）。

### 2.2 为什么选"短生命周期 Job"

| 需求 | Job 形态如何满足 |
|---|---|
| 可审计 | Job 的 SA、镜像 digest、启动/结束时间、容器日志全部由平台原生记录；配合 K8s Audit，写操作可归因 |
| 最小权限 | 每个 Job 绑定一个 SA，权限随 Job 生命周期存在；**Job 结束权限即消失**，不存在长期高权限进程 |
| 可复现 | 镜像以 digest 固定；提示词模板与白名单表进 Git，版本化 |
| 可终止 | `activeDeadlineSeconds` 强制超时终止；不存在"Agent 卡死并持有权限" |
| 无状态泄漏 | 容器销毁即无残留；不需要治理"Agent 内存里的凭据" |
| 并发控制 | Job 的 `parallelism`/标签互斥天然限制同一资源的并发操作 |
| AI 友好 | `kubectl create job` 即可触发；状态用 `kubectl get job -o json` 读取 |
| 不引入新组件 | 复用 Kubernetes，无需 Agent 平台 |

**对比被否决的形态：**

- **常驻 Agent Pod：** 长期持有权限，与"最小权限"原则冲突；需要额外的会话/内存/凭据治理；故障时权限暴露窗口长。
- **IDE/CLI 型 Agent（人工触发）：** 优点是零新增组件；缺点是**不可重复、不可定时、不可被告警自动触发**，无法验证"闭环是否真的成立"，且 Agent 运行在人的工作站上，凭据管理更差。
- **外部 Agent 平台（Dify/LangGraph 等）：** 引入第二套认证/权限/审计体系，与"任务即平台内一等对象、审计统一"目标冲突。

---

## 3. 运行形态与作业配置

### 3.1 触发方式（V0.1 仅 3 种）

| 触发源 | 路径 | 用途 |
|---|---|---|
| **定时** | 平台 n8n Schedule → 创建 Task → 创建 Job | 巡检、日报状态摘要、证书/备份/磁盘检查 |
| **告警** | Alertmanager → webhook → 平台 n8n → 创建 Task → 创建 Job | 由**真实告警**触发的分析（V0.1 验收要求至少一次） |
| **人工** | DingTalk 指令 → 平台 n8n → 创建 Task → 创建 Job | 自然语言查询/汇总（只读）、L2 批准后触发执行 |

**禁止：** Agent 自我触发、Agent 创建新的 Agent Job、Agent 修改触发规则。

### 3.2 Job 规格（Baseline 强制）

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: agent-task-<task-id>
  namespace: xw-ops
  labels:
    xuanwu.io/component: agent
    xuanwu.io/task-id: "<task-id>"
    xuanwu.io/risk-level: "L0|L1|L2"
spec:
  backoffLimit: 0                 # 不自动重试（避免重复副作用）
  activeDeadlineSeconds: 900      # 强制超时
  ttlSecondsAfterFinished: 86400  # 保留 24h 供审计，之后回收
  template:
    spec:
      serviceAccountName: sa-agent-readonly   # 或 sa-agent-executor / sa-agent-auditor
      restartPolicy: Never
      automountServiceAccountToken: true
      securityContext:
        runAsNonRoot: true
        runAsUser: 10001
        fsGroup: 10001
        seccompProfile: { type: RuntimeDefault }
      containers:
        - name: agent
          image: harbor.xw.internal/xuanwu-platform/agent-runtime@sha256:<digest>
          imagePullPolicy: IfNotPresent
          envFrom:
            - secretRef: { name: agent-llm-cred }   # 仅含 LLM 网关凭据
          env:
            - name: TASK_ID
              value: "<task-id>"
            - name: TASK_RISK_LEVEL
              value: "L0"
            - name: PROM_URL
              value: "http://prometheus.xw-obs:9090"
            - name: LOKI_URL
              value: "http://loki.xw-obs:3100"
            - name: HARBOR_API
              value: "https://harbor.xw.internal"
          resources:
            requests: { cpu: 200m, memory: 512Mi }
            limits:   { cpu: "1",  memory: 2Gi }
          securityContext:
            readOnlyRootFilesystem: true
            allowPrivilegeEscalation: false
            capabilities: { drop: ["ALL"] }
          volumeMounts:
            - { name: work, mountPath: /work }        # emptyDir，用于写入证据/计划
            - { name: repo, mountPath: /repo, readOnly: true }  # Git 只读副本
      volumes:
        - name: work
          emptyDir: { sizeLimit: 1Gi }
        - name: repo
          emptyDir: {}                # 由 initContainer 克隆 Git（只读用途）
```

**镜像内容要求：** `kubectl`、`curl`、`jq`、`git`、CA 证书、提示词模板、白名单表副本。**不得包含** `kubectl exec` 的利用路径、SSH 客户端、业务系统客户端、生产邮箱凭据。

### 3.3 关键纪律

1. **`backoffLimit: 0`** —— Agent 失败不自动重试（避免重复副作用）；重试由 n8n 显式决策并记录。
2. **`activeDeadlineSeconds`** —— 防止长时间持有权限。
3. **只读根文件系统** —— 证据/计划写入 `emptyDir`（`/work`），Job 结束后随 Pod 销毁（同时已写回 Task/Git）。
4. **不挂载任何业务 Secret**（`sa-agent-*` 无 Secret 读权限，RBAC 层面阻断）。
5. **不使用 hostPath / hostNetwork / privileged**（PSS `restricted` 强制）。

---

## 4. 权限模型（三个 ServiceAccount）

| SA | 范围 | 允许 | 明确禁止 |
|---|---|---|---|
| `sa-agent-readonly` | 集群级 ClusterRole（`view` 等价）+ 指定命名空间 Pod 日志 | `get/list/watch` 所有资源；`pods/log` 读取 | **Secret 读取**；任何写操作；`pods/exec`；`pods/portforward` |
| `sa-agent-executor` | 仅 `xw-obs`、`xw-ops`、`da-soc` 三个命名空间的 Role | 白名单动词（见下） | ClusterRole 绑定；`kube-system`/`platform-system` 写；Secret 读写；RBAC/NetworkPolicy/Quota/Node/PV 操作；`pods/exec` |
| `sa-agent-auditor` | 集群级只读 + Loki/Prometheus 只读 | 读取 Task、审计日志、集群对象 | 任何写操作（除创建/更新 Task 记录外） |

### 4.1 `sa-agent-executor` 白名单（V0.1 完整清单，不得私自扩展）

| # | 动作 | 目标 | 风险等级 |
|---|---|---|---|
| E1 | 删除 CrashLoop/Evicted 的 Pod（让其被控制器重建） | `xw-obs`、`xw-ops` | L1 |
| E2 | 重启 Deployment（`kubectl rollout restart`） | `xw-obs`、`xw-ops` | L1 |
| E3 | 扩缩容 Deployment（在 Quota 内） | `xw-obs`、`xw-ops` | L1 |
| E4 | 删除已完成的 Job（按标签选择器） | `xw-ops`、`xw-obs` | L1 |
| E5 | 触发 Harbor GC / Trivy 重扫 | Harbor API | L1 |
| E6 | 重新触发失败的备份 Job | `xw-ops` | L1 |
| E7 | 重启失败的前置检查 Job | `xw-ops` | L1 |
| **E8** | **重启 `da-soc` 中的 `da-soc-render`（无状态组件）** | `da-soc` | **L1 —— 仅当不在日报发送窗口内**；窗口内强制 L2 |
| E9 | 更新 Task 记录（状态/证据/结果/审批） | Git 仓库 | L0/L1 |

**绝对不在白名单内（任何情况都不允许 Agent 自动执行）：** 修改 RBAC / NetworkPolicy / Quota / CNI / StorageClass / PV / Node；删除 PVC/命名空间；重启 ClickHouse；删除或修改 DA-SOC 工作流与 SQL；修改 DingTalk 群配置；读写生产邮箱；`kubectl exec`；关闭任何安全控制。

### 4.2 试运行与幂等要求

每个白名单动作必须：

1. **先 dry-run**（`--dry-run=server` 或 API 侧校验），结果写入 Task；
2. **幂等**（重复执行不产生额外副作用）或**显式标记不可重复**；
3. **有前置条件校验**（如 E8 必须校验当前不在日报发送窗口内）；
4. **有验证方法**（见 §5）与**回滚路径**（见 §6）。

---

## 5. 执行通道（V0.1 仅 3 条）

| 通道 | 用途 | 权限载体 | 审计 |
|---|---|---|---|
| **Kubernetes API**（kubectl/client-go） | 平台与业务资源的白名单变更 | SA 的 Role（动词白名单） | K8s Audit + 容器日志 |
| **HTTP API** | 查询 Prometheus / Loki / Alertmanager / Harbor / Argo CD；触发 n8n webhook | 只读凭据（Harbor robot account、内部服务） | 容器日志 + 各服务自身日志 |
| **Git 提交（PR 或受限直推）** | 写入 Task 记录、审计记录、ADR 提案 | 细粒度 Git 凭据（仅限特定路径） | Git history |

**明确不用：** SSH 到节点、Docker socket、直接操作宿主机的任何脚本、MCP（V0.2 评估）、Agent 执行任意 shell 命令（除内置工具外的自由文本命令禁止）。

> **"通道不存在就不会被误用"** —— 这是本 ADR 最重要的设计思想：与其给 Agent 一条 SSH 通道再加白名单限制，不如**根本不建立该通道**。

---

## 6. 验证、回滚与审计

### 6.1 验证（必须独立于执行）

| 动作 | 验证方法 | 判据 |
|---|---|---|
| E1 删除异常 Pod | 查询新 Pod 状态 + 重启计数 | Ready 持续 5 分钟，重启计数不再增长 |
| E2 重启 Deployment | `rollout status` + 副本就绪 + 相关告警清除 | 全部副本 Ready；告警消除 |
| E3 扩缩容 | 实际副本数 + 无 Pending + 无资源紧张 | 双条件满足 |
| E4 清理 Job | 按标签选择器列出结果 | 目标 Job 不存在；非目标 Job 未被误删 |
| E5 Harbor 操作 | Harbor API 查询任务状态 | 任务成功 |
| E6 备份重触发 | Velero/Job 状态 + 备份时效指标 | 备份成功且年龄 < 26h |
| E8 重启 render | Pod Ready + 探针 + **业务链路后续步骤未被中断** | 日报链路正常 |
| 通用【业务】 | **数字比对**（不是"发送成功"） | 与基线一致 |

**硬规则：无法写出验证方法的动作，一律不得执行。** 验证失败 → 立即进入回滚流程。

### 6.2 回滚

| 变更类型 | 回滚机制 |
|---|---|
| 声明式配置变更 | Argo CD 回退到上一 Git revision（`argocd app rollback`）或 `git revert` + re-sync |
| Pod 类操作（E1/E2/E8） | 无需回滚（控制器自愈 / 无状态重建）；若引入故障则恢复上一镜像 digest |
| 扩缩容（E3） | 恢复变更前副本数（记录在 Task 中） |
| Job 清理（E4） | 从 Job 定义重建（定义在 Git） |
| 备份重触发（E6） | 无回滚需求（幂等） |
| 数据类变更 | **不在白名单内**（一律 L2，默认由人执行） |
| 不可回滚动作 | **禁止自动执行**；只能 L2，且必须在计划中显式声明"不可回滚" |

**回滚本身也是被审计的动作**：必须创建/更新 Task，记录回滚理由、命令、结果。

### 6.3 审计（每次 Job 必须产出）

| 产物 | 内容 |
|---|---|
| Task 记录（Git） | 请求人/触发源、任务 ID、风险等级、证据引用（可复现的查询与时间范围）、结论、计划、前置条件、dry-run 结果、审批记录、执行动作、执行结果、验证结果、回滚方式、关联的 Job 名与镜像 digest |
| 容器日志 | 完整执行日志（脱敏后） |
| K8s Audit | 该 SA 在集群上的所有请求（可按时段与 SA 检索） |
| 三者交叉 | **要求：仅凭 Git + Loki，事后任何人能完整重建"Agent 为什么这么做"** |

**可复盘性验证方法（V0.1 验收项）：** 随机抽取一次已完成的 Agent 操作，在**不看 Agent 会话记录**的前提下，仅用 Git 与 Loki 重建其决策链。

---

## 7. 被否决的方案及原因

| 被否决 | 原因 |
|---|---|
| 常驻 Agent Pod / 常驻 Agent 平台 | 长期持有权限，与最小权限原则冲突；需要会话/内存/凭据治理；故障时权限暴露窗口长 |
| IDE/CLI 型 Agent 人工触发（codebuddy） | 不可重复、不可定时、不可被告警自动触发 → 无法验证"闭环真的成立"；凭据在个人工作站上，审计与权限更差 |
| 外部 Agent 平台（Dify / LangGraph / 自研多 Agent 编排） | 引入第二套认证、权限、审计、状态存储体系，与"统一审计、Task 为平台一等对象"目标冲突；多 Agent 编排属 V0.4 |
| **`xw-opsapi`（codebuddy 提出的自研只读聚合层）** | **5 份方案中仅 1 份提出**；其四项理由均已有更轻量的替代（见 §7.1）；新增组件 + 单点 + 自研代码维护面，与"V0.1 克制"冲突 |
| Agent 直接持有 Prometheus/Loki 凭据并由平台侧下发 | 只读 SA + 集群内服务访问已足够；把凭据下发给 Agent 反而扩大暴露面 |
| 给 Agent 通用 SSH / 任意 shell | 权限无法用 RBAC 表达，审计无法收敛；直接违反最小权限 |
| Agent 常驻高权限 ServiceAccount | 违反 README 安全红线与 ADR-007 第 1 条 |
| 用 Agent 代替人执行 L2 操作 | V0.1 的目标是"验证闭环成立"，而非"最大化自动化率"；L2 执行权下放属 V0.4 |

### 7.1 对 `xw-opsapi` 的裁决（明确推迟）

codebuddy 提出 `xw-opsapi`（约 300–500 行 FastAPI 只读聚合服务）作为 Agent 与 K8s/Prometheus/Loki/Git 之间的唯一入口，理由是：① 最小权限（Agent 只拿一个 token）；② 可审计（记录 Agent 的"观察"）；③ AI 友好（统一 JSON）；④ 成本可控。

**逐条检验与替代方案：**

| codebuddy 的理由 | 替代方案（不需新增组件） |
|---|---|
| 最小权限：Agent 只拿一个 token | 用**独立的只读 ServiceAccount**（`sa-agent-readonly`）达到同等效果；Prometheus/Loki 在集群内网可达且仅需只读访问；**无需把平台凭据"聚合"后再下发** |
| 可审计："Agent 看了什么"要留证 | ① K8s Audit 已记录 Agent 对 apiserver 的每次读请求；② Prometheus/Loki 本身有查询日志；③ **Task 记录中强制要求写入"证据引用"（PromQL/LogQL 语句 + 时间范围）**，比记录 HTTP 调用更贴近"证据链"目标 |
| AI 友好：统一 JSON，避免解析表格 | 用 `kubectl get -o json`（原生 JSON）；Prometheus/Loki 本身就是 JSON API；**不需要中间层做转换** |
| 成本可控（300–500 行） | "300–500 行"是**新增自研代码的维护面**：需要镜像构建、漏洞修复、升级、测试、备份、监控——这些成本远大于 500 行的表面成本 |

**裁决：V0.1 不引入 `xw-opsapi`，推迟到至少 V0.2，且必须由真实需求触发。**

**触发条件（满足任一才重新评估）：** ① Agent 需要访问的观测源超过 6 个且凭据分发成为实际瓶颈；② 出现"必须记录 Agent 观察行为但 K8s Audit/查询日志无法覆盖"的审计缺口；③ 出现明确的统一查询接口需求（如多个 Agent/多个消费者）。

**同时保留其有价值的设计思想：** "Agent 的观察也要留证"这一要求被完整吸收为本 ADR §6.3 的"证据引用"强制项。

---

## 8. 后果与影响

**正面：** Agent 有明确身份、最小权限、完整审计、强制超时与终止；权限随 Job 生命周期消失；可复现（镜像 digest + Git 版本化的白名单与提示词）；不引入新的 Agent 平台组件。

**负面 / 代价：**

1. 冷启动延迟（每次分析都要拉镜像、起 Pod，数十秒量级）；
2. 无长期记忆 / 多轮对话能力（V0.1 不需要；V0.4 评估）；
3. 需要自建一个 Agent 运行时镜像并纳入 Harbor 与漏洞管理（属平台组件，可控）；
4. 并发与批次控制需在 n8n 侧实现。

**对后续版本的影响：** V0.2 评估 MCP 与更丰富的只读工具集；V0.3 增加运行时安全约束；**V0.4 才考虑多 Agent 编排与 L2 执行权下放**（前提是 V0.1–V0.3 已积累足够的 Task 样本与验证经验）。任何版本的 Agent 权力扩张都必须伴随**审计能力同步扩张**。
