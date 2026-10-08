# Xuanwu SecureOps Stack V0.1
# Consistency Audit — dsh

> **审计类型：** READ-ONLY AUDIT（未修改任何现有项目文件）
> **审计对象：** 仓库 `main` 分支当前工作区（HEAD = `3151bb3 docs: 新增 V0.1 两清两固安全运营能力及相关基线文档`）
> **审计范围：** 除 `09-implementation/00-architecture-review/`（5 份候选方案，非实施依据）之外的全部项目文件，共 32 个文件
> **审计日期：** 2026-10-08
> **审计原则：** 本审计不重新设计架构；每个发现均给出双向引文与行号

---

## 1. Executive Summary

### 总体判断

```text
PASS WITH P1
```

**结论要点：** 仓库的**架构结论本身是自洽的**——5 台 VM、1 CP + 2 Worker、Harbor/备份外置、local-path、单一观测栈、Git Task 无 CRD、短生命周期 Job Agent、无 `xw-opsapi` 在 `README`／`Baseline`／`Adjudication`／`ADR-001~008`／`TODO`／`VERSION-MATRIX`／CLM／两清两固之间**方向一致，未发现范围膨胀，也未发现任何绕过 L2 审批的 Agent 权限路径**。

**但最近三次补丁（`1b3145d` 实施清单 → `695c6c7` 版本矩阵+CLM → `3151bb3` 两清两固）在「状态模型」与「Task Schema」两个横切面上引入了重复定义与引用错误**，属于实施前必须统一的一致性问题。这些问题的共同根因是：

> **三个补丁各自定义了「状态」，但没有一份文档裁定「哪个状态模型属于哪个维度」。**

### 发现统计

| 严重度 | 数量 | 说明 |
|---|---:|---|
| **P0** | **0** | 未发现存在真实生产安全风险、必须阻断实施的架构矛盾 |
| **P1** | **5** | 实施前必须统一：状态词汇冲突（5 套枚举并存，含 `lifecycle` 同名不同义）、Task Schema 三处重复定义、`port_required` 字段类型混用、README 文档状态自相矛盾、两清两固四份基线的统一状态模型未落到可校验字段 |
| **P2** | **12** | 应修复：Task 编号/依赖引用错误 5 处、循环依赖 1 处、依赖图与字段不一致 2 处、前置/依赖字段不一致 3 处、`TASK-SEC-001` 字段缺项、观测栈部署主体歧义、章节编号乱序、需求侧与版本侧张力、V0.1 能力清单漏列 CLM/两清两固、组件注册表覆盖基准不明 |
| **P3** | **6** | 文档优化：路径引用不统一、术语/拼写一致性、清单完备性、Markdown 结构等 |
| **合计** | **23** | 其中 5 项 P1 全部落在"状态模型 / Task Schema / 文档状态"的元数据层面，不涉及架构结论、安全边界或生产风险 |

### 是否有 P0

**没有。** 具体理由见 §12「No-Issue Areas」：三项最高风险面（Agent 越权、双 n8n 消费生产邮箱、备份未演练即切换）在所有文档中**一致地被禁止并被门禁覆盖**，不存在互相矛盾的漏洞。

---

## 2. Architecture Consistency

### 检查范围与结论

| 检查项 | Baseline | Adjudication | ADR | TODO | VERSION-MATRIX | CLM | README | 结论 |
|---|---|---|---|---|---|---|---|---|
| VM 数量 = 5 | §2.1 L21–L27 | §1 L18 / §10 L226–L231 | ADR-003 L8 | §2 L41 / §1 L33 | — | — | 未出现 | ✅ 一致 |
| 1 CP + 2 Worker | §5.1 L124 | §11 L236–L237 | — | §2 L42 | — | — | 未出现 | ✅ 一致 |
| Harbor 在集群外（`xw-harbor-01`） | §9.1 L209 | §8 ADR-003 L185 | ADR-003 L8 | §2 L45 | §3 L32 | components.yaml L60 | 未出现 | ✅ 一致 |
| 备份在集群外（`xw-backup-01`） | §2.1 L27 | §10 L231 | ADR-006 L12 | §2 L48 | — | — | 未出现 | ✅ 一致 |
| local-path / Local PV | §8.1 L189 | §13 L258 | ADR-004 L8 | §2 L46 | §3 L31 | — | 未出现 | ✅ 一致 |
| ClickHouse 单副本 + 固定 `xw-wk-02` | §8.2 L195 | §13 L258 | ADR-004 L8 | §9 L373 | — | — | 未出现 | ✅ 一致 |
| 不用 Ceph / Longhorn | §8.3 L203 | §22 L441 | ADR-004 L12 | §2 L54 | — | — | 未出现 | ✅ 一致 |
| 单一观测栈（Prom+Grafana+AM+Fluent Bit+Loki） | §10.1 L236–L241 | §16 L291–L293 | ADR-005 L8 | §2 L47 | §3 L34–L38 | components.yaml L94–L143 | 未出现 | ✅ 一致 |
| 禁 Promtail / ELK / SIEM | §10.1 L241 | §24 L461 | ADR-005 L27 | §2 L54 | — | README L116 | 未出现 | ✅ 一致 |
| DA-SOC 在 K8s `da-soc` 运行 | §15.1 L343–L359 | §7.1 L148 | ADR-001 L8 | §1 L33 | — | — | 未出现 | ✅ 一致 |
| 验证双跑 / 生产单活 | §1 L13 | §7.3 L165–L171 | ADR-001 L30 | §1 L33 / §3 L60 | — | — | 未出现 | ✅ 一致 |
| ECS 仅短期回退源 | §1 L13 | §7.1 L148 | ADR-001 L8 | §2 L52 | — | — | 未出现 | ✅ 一致 |
| **禁止两个 n8n 同时读生产邮箱** | §1 L13 | §7.3 L167 | ADR-001 L30 | §3 L60 | — | — | 未出现 | ✅ 一致（且三处显式禁止） |
| Task = Git YAML/Markdown，无 CRD | §16.3 L392 | §8 ADR-008 L205 | ADR-008 L8 | §2 L50 | — | — | 未出现 | ⚠️ 载体一致，**Schema 冲突（CNS-002）** |
| Agent = 短生命周期 Job | §16.1 L377 | §8 ADR-007 L199 | ADR-007 L8 | §2 L49 | — | README L1161 | L1161 | ✅ 一致 |
| 不建 `xw-opsapi` | §16.2 L390 | §17.2 L306 | ADR-007 L14 | §2 L54 | — | — | 未出现 | ✅ 一致 |

### 结论

**架构一致性：无 P0/P1 级冲突。** 全部架构结论在所有权威文档中一致。唯一进入架构章节的问题是 **CNS-002（Task Schema 三处重复定义）** 与 **CNS-007（README 状态自相矛盾）**，二者属"定义/状态漂移"而非"架构结论冲突"。

**特别确认（对应审计要求 C02）：** 「不得让 ECS + Kubernetes 同时读取生产邮箱」在**三个独立文档中被显式禁止**，且写入了 Cutover Gate：

| 文件 | 行 | 引文 |
|---|---|---|
| Baseline | L13 | "不得让 ECS 和 Kubernetes 两个 n8n 同时读取生产邮箱" |
| Adjudication | L167 | "不允许 ECS n8n 和 Kubernetes n8n 同时读取生产邮箱" |
| ADR-001 | L30 | "两个 n8n 不得同时读取生产邮箱、写入同一生产数据、发送同一日报或修改同一状态" |
| CUTOVER-CHECKLIST | §2 | "[ ] 单活消费者证明完成。" |

---

## 3. Version Consistency

### 检查方法

对 17 类组件（UOS、Kubernetes、KubeSphere、containerd、Calico、Harbor、ClickHouse、n8n、render/archive、Prometheus、Grafana、Alertmanager、Fluent Bit、Loki、DA-SOC images、Agent images、SOPS/age）逐一比对 7 份文件的版本表述。

### 一致性结论：**无版本号互相冲突**

| 组件 | VERSION-MATRIX | components.yaml | TODO §0 | README §23 | Baseline/ADR | 结论 |
|---|---|---|---|---|---|---|
| OS | UOS Server V20 1060e AMD64（L20） | `V20 1060e`（L10） | 同（L20 L73 L87） | 同（L1151） | Baseline 仅要求"按兼容矩阵冻结"（L41） | ✅ 一致 |
| Kubernetes | `v1.30.6`（L21） | `1.30.6`（L20） | 同（L21 L73 L87 L216） | 同（L1152） | Baseline §5.1 L125 要求"来自同一官方兼容矩阵" | ✅ 一致 |
| KubeSphere | `4.1.x`（优先 4.1.2）（L22） | `4.1.x`/`4e1.2`（L30） | 同（L22 L73 L87 L274） | 同（L1153） | Baseline §5.1 L125；§5.3 未固定版本 | ✅ 一致 |
| containerd | `1.7.x`（L23） | `1.7.x`（L40） | 同（L23 L87） | 同（L1154） | — | ✅ 一致 |
| Calico | 未固定（L24） | `unknown`（L50） | 同（L24） | 同（L1155） | ADR-002 只选型不固定版本 | ✅ 一致（同为未固定） |
| Harbor | "Approved independent-VM release"（L32） | `unknown`（L60） | 要求 TASK-002 冻结（L87） | — | ADR-003 不固定版本 | ✅ 一致（同为待冻结） |
| ClickHouse | "Existing DA-SOC-compatible release"（L39） | `unknown`（L70） | 要求冻结（L87） | — | ADR-001 L155 单副本 | ✅ 一致 |
| n8n | `2.15.0 或批准的现有兼容版本`（L40） | 同（L80） | — | — | Adjudication L79 `n8n 2.15.0`；ADR-001 无版本 | ✅ 一致 |
| render/archive | `da-soc-render:0.1`（L41） | 同（L90） | — | — | Adjudication L79 | ✅ 一致 |
| Prom/Grafana/AM/Fluent Bit/Loki | 全部 "compatible approved release / Pending Version Freeze"（L34–L38） | 全部 `unknown`（L100–L143） | 要求冻结（L87） | — | ADR-005 只选型 | ✅ 一致（同为待冻结） |
| Agent image | "Approved fixed digest"（L42） | `unknown`（L160） | 要求冻结（L87） | — | ADR-007 L14 固定 digest | ✅ 一致 |
| SOPS/age | "Approved secret-encryption implementation"（L43） | — | 要求冻结（L87） | — | Baseline §11.1 L270 | ✅ 一致 |

### Candidate / Pending / Frozen 状态语义检查：**使用正确**

`Candidate` / `Pending Compatibility Validation` / `Pending Version Freeze` / `Frozen` 四类状态在 VERSION-MATRIX（L10 定义、L20–L57 使用）、components.yaml（`desired.status`，L10–L160）、TODO §0（L18–L29）中**语义一致**，且都明确声明"候选 ≠ 官方认证组合"：

```text
VERSION-MATRIX L10   : "本文件区分 Candidate、Pending Compatibility Validation 和 Frozen；候选版本不得描述为官方认证组合。"
VERSION-MATRIX L47   : "以下组合是 V0.1 的候选基线，不是官方认证或官方保证的完整组合"
TODO L29             : "上述 OS + Kubernetes + KubeSphere + containerd + Calico 仅是 V0.1 候选组合，不代表官方认证的完整组合。"
README L1147         : "当前记录的是候选基线，不是官方认证的完整组合"
components.yaml L10  : "免费使用授权；不等同于官方认证组合、SLA 或商业支持承诺"
```

**Audit 特别说明（避免误判）：** VERSION-MATRIX L21/L22 给出的 `Kubernetes v1.30.6` 与 `KubeSphere 4.1.x` 属 `Candidate / Pending Compatibility Validation`，**不构成"同一决策在不同文件中状态含义冲突"**，因此**不判为不一致**。但需注意两点客观事实：

1. Baseline §5.1 L125 要求"Kubernetes 与 KubeSphere 版本必须来自**同一官方兼容矩阵**"，而 VERSION-MATRIX §4 L47–L57 明确该组合**尚未验证**，冻结需 6 项证据（L59–L66）。状态本身自洽（pending），但**该组合的可行性仍是未消除的技术前提**（见 §11 FP-01 与 CNS-017）。
2. 大火版本跨度值得风险登记：Kubernetes 1.30 + KubeSphere 4.1.x 相对早期评估基线（K8s 1.26 系 + KubeSphere 3.4 系）跨度较大，而 README §23 L1157 与 TODO L29 均已承认"当前安装尚未开始"。**这属于待验证风险，不属于一致性问题。**

### 结论

**版本一致性：无冲突。** 未发现同一组件在不同文件出现不同版本号，也未发现 Candidate/Frozen 状态被混用。该候选组合归入 §11 FP-01 并作为待验证风险登记，**不作为一致性问题**。

---

## 4. TODO Consistency

### 4.1 架构是否全部落到 Task：**是，覆盖完整**

| 架构能力 | TODO 任务 | 结论 |
|---|---|---|
| Calico | TASK-018（L315）、TASK-019（L329） | ✅ |
| Harbor（外置） | TASK-025～028（L417/L431/L445/L459） | ✅ |
| Local PV | TASK-022（L373）、TASK-023（L387）、TASK-024（L401） | ✅ |
| Observability（单一栈） | TASK-029～033（L475/L489/L503/L517/L531） | ✅ |
| Security Baseline | TASK-034～037（L547/L561/L575/L589） | ✅ |
| Backup / Restore | TASK-038～041（L605/L619/L633/L647） | ✅ |
| DA-SOC 承载 | TASK-042～049（L663–L761） | ✅ |
| 迁移 / Cutover / Rollback | TASK-050～059（L777–L907） | ✅ |
| AI Ops | TASK-060～063（L923/L937/L951/L965） | ✅ |
| 恢复 / 故障演练 | TASK-064～066（L981/L995/L1009） | ✅ |
| 最终验收 | TASK-067～068（L1025/L1039） | ✅ |
| **CLM** | TASK-CLM-001～006（L1055–L1125） | ✅ |
| **两清两固** | TASK-SEC-001～006（L1231–L1277） | ✅ |

### 4.2 Task 是否违反 Architecture：**无违反**

TODO 明文重述全部 Baseline 约束（§2 L41–L54），并在 §3 L54 明确 `V0.1 不引入 xw-opsapi、Ceph、Longhorn、ELK、Istio/Linkerd、GPU、多集群、复杂 Policy Engine、Task CRD`。**未发现任何任务引入架构明确拒绝的组件。**

**特别确认：** TODO 未出现 `Task CRD` 的**建立**任务（L50/L54/L969/L1043 均为"不使用/不建立/评估"语义）。**未发现"架构说不用 CRD、TODO 却建 CRD"这类硬冲突。**

### 4.3 发现的问题（3 类）

**（a）状态模型未统一 —— 见 CNS-001（P1）**

TODO L129 定义 `PENDING/IN_PROGRESS/BLOCKED/PASSED/ROLLED_BACK`，但该词汇与 ADR-008 的 Task 生命周期、Baseline §16.3 的 Task YAML 状态字段、`security-baseline.yaml` 的 finding 生命周期互不相同。

**（b）Task 编号引用错误 —— 见 CNS-009（P2）**，3 处：

| # | 位置 | 现有引文 | 应为 |
|---|---|---|---|
| 1 | `RESTORE-DRILL-PLAN.md` L41 | "任一关键恢复失败则 TASK-061～TASK-064 不通过" | **TASK-064～TASK-066**（TASK-061/062 是 AI Ops 任务：L937「实现 CrashLoopBackOff 真实闭环」、L951「验证 Agent 失败、超时和回滚」） |
| 2 | `EXTERNAL-DEPENDENCIES.md` L37 | "离线/不可变备份副本 … Blocking Task = **TASK-061**" | **TASK-040**（L633「实现 DA-SOC、Harbor 和 Secret 备份」）或 TASK-038/041；TASK-061 与备份无关 |
| 3 | `TODO.md` L874 | TASK-056 **Dependencies:** `TASK-041、TASK-053～055、TASK-061～066` | TASK-061～066 属 Phase 14/15（L921/L979），**晚于** TASK-056 所在 Phase 13（L877）；且 §21 依赖图（L1160–L1161）**不含**该边 |

**（c）Preconditions 与 Dependencies 字段不一致 —— 见 CNS-013（P2）**，3 处：

| 位置 | Preconditions | Dependencies | 差异 |
|---|---|---|---|
| TODO L375 / L384 | `TASK-016、TASK-018、TASK-021` | `TASK-006、TASK-016、TASK-018` | Preconditions 含 TASK-021、Dependencies 缺 TASK-021；Dependencies 多 TASK-006 |
| TODO L549 / L558 | `TASK-016、TASK-017、TASK-027` | `TASK-017、TASK-027` | Preconditions 多 TASK-016 |
| TODO L591 / L600 | `TASK-034～036、TASK-013、TASK-030` | `TASK-035、TASK-036` | Preconditions 多 TASK-034、TASK-013、TASK-030 |

**（d）章节编号乱序 —— 见 CNS-014（P2）**

TODO L1208 `## 24. Definition of Done` 出现在 L1229 `## 20B. Two-Clear-Two-Firm Security Operations` **之前**，即 20B 位于 24 之后。`## 20A. Component Lifecycle Management`（L1053）位置正常。

### 结论

**TODO 一致性：架构无冲突，但存在 4 类实施前应修复的编号/字段/状态问题（1 项 P1，3 项 P2）。**

---

## 5. Source of Truth Consistency

### 5.1 实际形成的 Source of Truth 链（与审计预期基本吻合）

```text
ARCHITECTURE-BASELINE-V0.1.md ── 定义架构（已声明，Baseline §24 L567）
        ↓
ADR-001 ~ ADR-008 ───────────── 定义已接受的技术决策
        ↓
09-implementation/VERSION-MATRIX.md ── 定义期望版本（CLM README §4 L69 显式声明）
        ↓
07-aiops/component-lifecycle/components.yaml ── 定义组件与三类状态（L2 source_of_truth: git）
07-aiops/component-lifecycle/policies.yaml ──── 定义审批策略（L2）
07-aiops/component-lifecycle/upgrade-rules.yaml ─ 定义 upgrade_required 与优先级（L2）
        ↓
04-security/security-baseline.yaml ── 定义统一状态模型与 Task 分类（L2）
04-security/port-baseline.yaml ────── 定义端口 Desired State（L2）
04-security/account-baseline.yaml ─── 定义账号 Desired State（L2）
04-security/access-control-baseline.yaml ── 定义访问控制 Desired State（L2）
        ↓
TODO.md ─────────────────────── 定义实施任务
        ↓
09-implementation/*.md ──────── 定义执行方法（Cutover/Rollback/Restore）
        ↓
Evidence（Git commit / Test 输出 / 审计日志）── 定义实际事实
```

**分工声明在文本上基本清晰：**

| 文件 | 声明 | 行 |
|---|---|---|
| Baseline | "Git 是所有声明式定义的 Source of Truth，但不是业务数据仓库" | L266 |
| CLM README | "`09-implementation/VERSION-MATRIX.md`：批准的期望版本…" / "Git CLM YAML：组件、策略、升级规则和审批边界的 Source of Truth" / "Discovery evidence：实际运行状态…不手工覆盖" | L69 / L71 / L72 |
| security-baseline | `source_of_truth: git` | L2 |
| port-baseline | `source_of_truth: git` | L2 |
| account-baseline | `source_of_truth: git` | L2 |
| access-control-baseline | `source_of_truth: git` | L2 |
| TODO | "唯一架构依据：`10-decisions/ARCHITECTURE-BASELINE-V0.1.md`" | L9 |

### 5.2 发现的缺口（2 项）

**（a）Task 状态与 Task Schema 无唯一 Source of Truth —— 见 CNS-001 / CNS-002（P1）**

四份文件各自声称定义 Task 状态/结构，但**没有任何一份被裁定为权威**：

| 文件 | 声称 | 行 |
|---|---|---|
| ADR-008 | `Decision`/`Lifecycle`：定义 Task 生命周期 | L8 / L12–L16 |
| Baseline §16.3 | `Task YAML`：给出 `api_version: xuanwu/v0.1` / `kind: Task` 完整结构 | L392–L427 |
| security-baseline | `task_model` + `required_fields` | L9–L11 |
| TODO TASK-005 | "创建任务状态模型" | L129 |

> **这是本次补丁引入的核心 SoT 缺口：Task（运维任务）的「生命周期」与「字段 Schema」在两个文件里各说一次，第三个文件又说一次，第四处要求"创建"一个。实施者无法判断以哪一份为准。**

**（b）端口/账号/访问控制三份基线的 `status` 字段未声明受统一模型约束 —— 见 CNS-023（P1，旁证）**

`security-baseline.yaml` L5–L8 声明 `state_model.statuses: [PASS, FAIL, REVIEW, UNKNOWN]`，但：
- `port-baseline.yaml` L10 的 `record_fields` 含 `status`，其 L70 又出现 `port_required: REVIEW`（把状态值用作另一字段取值）；
- `account-baseline.yaml` L4 与 `access-control-baseline.yaml` L4 的 `record_fields` 含 `status`，但**均未声明 `state_model`**；
- CLM README §10 L144 声称"All four capabilities use `PASS`, `FAIL`, `REVIEW` and `UNKNOWN`"，但**该约束只是散文，未落入三份 YAML 的可校验字段**。

### 结论

**Source of Truth 一致性：架构/版本/组件/策略四层分工清晰；但「Task 状态模型」与「Task Schema」这一横切面缺少唯一权威（P1）。**

---

## 6. CLM Consistency

### 6.1 CLM 与既有能力是否重复

| 检查项 | 结论 | 证据 |
|---|---|---|
| 与 AI Ops 重复？ | **不重复，属正确定位** | CLM 是 AI Ops 的一个能力域，复用 Job/Task/L0-L1-L2（CLM README §6 L94–L98；Baseline §16.5 L435 明确"纳入 CLM 最小闭环，但不建设独立漏洞平台"） |
| 与 Security Baseline 重复？ | **部分重叠，但已声明分工** | CLM README §10 L137 声明"CLM supplies the vulnerability and lifecycle part…The other three controls use the declarative baselines in `04-security/`"；Baseline §25 L575 同口径 |
| 与 TODO 重复？ | **不重复** | CLM 有独立任务组 TASK-CLM-001～006（TODO L1055–L1137） |
| 创建第二套 Task 模型？ | ⚠️ **部分成立** | `upgrade-rules.yaml` L35 定义 `lifecycle.statuses`（10 项），与 ADR-008 L14 生命周期、Baseline L426 `audit.result` 三者不同 → **CNS-001** |
| 创建第二套状态模型？ | ⚠️ **部分成立** | 同上；另 `components.yaml` 的 `desired.status`（candidate/pending-freeze）与 `upgrade.status`（planned）与 Task 状态共用词汇但语义不同 → **CNS-004** |
| 创建第二套漏洞管理机制？ | **否** | CLM README §7 L113–L118 明确"完整 SBOM 平台、独立漏洞管理平台、SIEM、CMDB、SOAR"延后 V0.2/V0.3 |
| 引入原架构拒绝的组件？ | **否** | 全部复用 Git + 短生命周期 Job + Prometheus/Grafana/Alertmanager（CLM README §11 L151） |

### 6.2 CLM 概念在各文档中的定义冲突（重点检查项）

| 概念 | Baseline §16.5 | CLM README | components.yaml | upgrade-rules.yaml | TODO | 结论 |
|---|---|---|---|---|---|---|
| Component | "组件生命周期管理（CLM）最小闭环" | §2 L25–L27 "由 `components.yaml` 定义组件身份…" | L3–L163 `components:` 16 项 | — | TASK-CLM-001 L1055 | ✅ 一致 |
| Desired State | §16.5 L437 通过 CLM 管理 | §2 L37 "期望状态和策略在 Git"；§4 L69 VERSION-MATRIX | L10/L20/… `desired:` | — | — | ✅ 一致 |
| Observed State | §16.5 L437 "发现当前版本、运行状态、镜像 repository/tag/digest" | §2 L37 "实际运行状态由发现 Job 写入审计结果或受控状态 artifact"；§3 L53–L63 Discovery Methods | `discovery:` | — | TASK-CLM-002 L1069 | ✅ 一致 |
| Vulnerability State | §16.5 L437 "CVE/CVSS/KEV/EOL 和来源时间" | §2 L29–L33 "回答当前运行版本有什么风险" | `security:` + `lifecycle:` | `rules:` | TASK-CLM-003 L1083 | ✅ 一致 |
| Lifecycle State | §16.5 L437 | §2 L35 "Upgrade State" | `lifecycle.support_status` | `lifecycle.statuses` L35 | — | ⚠️ **术语"Lifecycle"在 components.yaml 指支持状态、在 upgrade-rules 指升级流程 → CNS-004** |
| Upgrade State | §16.5 L437 "生成 Git Upgrade Task" | §2 L35–L37 | `upgrade:` | `rules:` / `priority:` | TASK-CLM-004/005 L1097/L1111 | ✅ 一致 |
| Task | §16.3 Task YAML | §1 L16 "Git Task" | — | `task-created`（状态值） | TASK-CLM-005 L1111 | ⚠️ **Schema 冲突 → CNS-002** |
| Approval | §16.5 L437 "生产升级…必须遵守 L2 人工审批" | §6 L94–L98 | `upgrade.approval_required: true` | `approval_rules`（policies.yaml L20–L34） | TASK-CLM-005 | ✅ 一致 |
| Execution / Verification / Audit | §16.5 L437 "通知、审计、验证和回退" | §1 L16–L21 | `audit:` | `lifecycle.statuses` 含 executing/verifying | TASK-CLM-006 L1125 | ✅ 一致 |

### 6.3 `status` 同名不同义检查（审计重点）

| 状态词 | 出处 A | 出处 B | 是否冲突 |
|---|---|---|---|
| `planned` | ADR-008 L15 Task 生命周期中间态（analyzed → **planned** → pending-approval） | components.yaml L13/L23/… `upgrade.status: planned`（组件升级计划的属性） | ⚠️ **同名不同义 → CNS-004** |
| `candidate` | VERSION-MATRIX L10 版本冻结状态 | components.yaml L10 `desired.status: candidate`；policies.yaml 未用 | ✅ 语义一致（均指"候选，未冻结"） |
| `approved` | ADR-008 L15 审批结果 | policies.yaml L4 `approval_level: L2`（不同字段名）；upgrade-rules L35 `approved`（升级流程态）；TODO L1128 出现 1 次 | ⚠️ 三处语义接近但归属不同模型 → 并入 CNS-001 |
| `review` / `REVIEW` | security-baseline L6 发现状态 `REVIEW` | upgrade-rules L35 `review`（升级流程态）；CLM README L83 "进入 REVIEW"；`upgrade-rules` L28 `upgrade_required: review` | ⚠️ **同名不同义（发现状态 vs 升级结论 vs 流程态）→ CNS-004** |
| `closed` | ADR-008 L15 Task 终态 | security-baseline L8 发现生命周期终态；upgrade-rules L35 升级终态 | ⚠️ 三模型同名终态 → 并入 CNS-001 |
| `discovered` / `assessed` / `verifying` | security-baseline L8 | upgrade-rules L35（同样出现） | ⚠️ **两套模型重叠但不完全一致** → CNS-001 |
| `rolled-back` / `ROLLED_BACK` | upgrade-rules L35 `rolled-back` | TODO L129 `ROLLED_BACK` | ⚠️ 拼写风格不同但语义一致；属 P3 |

### 结论

**CLM 一致性：定位正确、未引入新平台、未引入架构拒绝组件；但引入了第 3 套状态词汇，且 `planned`/`review`/`lifecycle` 三个词与既有模型同名不同义（P1）。**

---

## 7. Two-Clear-Two-Firm Consistency

### 7.1 四项能力是否统一采用同一闭环

声明层面：**是**。

```text
README L1275          : "四项能力都必须完成：发现 → 判断 → 任务 → 整改 → 验证 → 审计。"
Baseline §25 L577     : "四项能力均使用 Desired State → Observed State → Deviation → Risk → Task →
                          Approval → Remediation → Verification → Audit"
CLM README §10 L145   : "The V0.1 closure target is continuous manageability…discover, assess, create a task,
                          remediate through an approved L0/L1/L2 path, rescan or re-audit, verify and close with evidence."
TODO TASK-SEC-001 L1234: "冻结四类 finding、PASS/FAIL/REVIEW/UNKNOWN、证据脱敏、Owner、SLA、任务字段和 Git 审批规则"
```

**落地层面：部分不一致 —— 见 CNS-023（P1）与 CNS-005（P2）。**

| 能力 | Desired State 载体 | 闭环声明 | Task category | 状态模型声明 |
|---|---|---|---|---|
| 清高危漏洞 | `components.yaml` + `upgrade-rules.yaml` | ✅ CLM README §6 | `VULNERABILITY`（security-baseline L10） | ✅ |
| 清高危端口 | `port-baseline.yaml` | ⚠️ 无显式闭环声明，仅 `rules:` L72–L76 | `PORT` | ❌ 未声明 `state_model` |
| 固弱账号口令 | `account-baseline.yaml` | ⚠️ 仅 `rules:` L43–L46 | `ACCOUNT` | ❌ 未声明 `state_model` |
| 固弱访问控制 | `access-control-baseline.yaml` | ⚠️ 仅 `rules:` L57–L61 | `ACCESS_CONTROL` | ❌ 未声明 `state_model` |

### 7.2 PASS / FAIL / REVIEW / UNKNOWN 一致性

| 位置 | 使用的四个状态 | 是否有 UNKNOWN 误处理为 PASS/SAFE |
|---|---|---|
| `security-baseline.yaml` L6 | `[PASS, FAIL, REVIEW, UNKNOWN]` + L7 `unknown_is_pass: false` | ✅ 显式禁止 |
| CLM README §3 L65 | "无法自动发现的版本必须标记为 `unknown_version`…不得当作'无漏洞'或'无需升级'" | ✅ 显式禁止 |
| CLM README §5 L83 | "版本未知：高风险状态，必须先完成发现或人工确认" | ✅ |
| TODO L1235 | "UNKNOWN 不会变成 PASS" | ✅ |
| TODO L1224 | "UNKNOWN 可见且不默认为 PASS" | ✅ |
| README L1275 | "`UNKNOWN` 不得默认为 PASS" | ✅ |
| Baseline §25 L577 | "`UNKNOWN` 不得当作安全" | ✅ |
| upgrade-rules L31–L33 | `unknown-version` → `upgrade_required: review, risk: high` | ✅ |

**结论：`UNKNOWN` 在全部 8 处均被正确拒绝为 PASS/SAFE，未发现任何一处把 UNKNOWN 当作安全。** ✅ **这是一致性最好的区域之一。**

### 7.3 两清两固是否引入新基础设施

**未引入。** README L1277、Baseline §25 L579、CLM README §10 L147 三处独立声明不引入 SIEM、SOAR、CMDB、NDR、完整 IAM、完整漏洞管理平台或独立安全基础设施；复用 Kubernetes、KubeSphere、Calico、Harbor、Prometheus/Grafana/Alertmanager、Fluent Bit/Loki、Git、短生命周期 Agent Job。✅

### 7.4 L0/L1/L2 定义一致性

| 文档 | L0 | L1 | L2 | 结论 |
|---|---|---|---|---|
| README §4.3 L146–L179 | 查询状态/健康检查/日志查询/资源统计/漏洞扫描/证书检查/生成报告 | 重启异常 Pod/非生产扩容/清理无用资源/修复低风险配置 | 修改生产网络策略/RBAC/删除节点/升级 K8s/修改存储/核心网络/删除生产数据/关闭安全控制/影响业务连续性 | 基线定义 |
| Baseline §12.2 L303–L305 / §19 L461–L471 | 只读检查、报告、查询、告警聚合 | 白名单 Runbook（`da-soc-render` 重启、非生产临时文件清理），必须有前置条件和自动验证 | RBAC/NetworkPolicy/CNI/节点/存储/生产 n8n/ClickHouse 数据删除恢复/凭据/镜像策略变更 | ✅ 一致（更具体） |
| ADR-007 L14 / Adjudication L282 | 默认只读 | `da-soc` render Deployment 受限动作 | 人工审批后才获得短时授权 | ✅ 一致 |
| CLM README §6 L94–L98 | 自动发现、漏洞/EOL 判断、报告和 Git Task 创建 | 验证环境 Patch、非生产组件或可回滚白名单 Runbook | Kubernetes/KubeSphere/Calico/Harbor/OS/ClickHouse/生产镜像/节点/存储/网络/RBAC | ✅ 一致 |
| upgrade-rules `policies.yaml` L4/L9/L14/L19 | `approval_level` L2（各策略） | `non-production` → `approval_level: L1` | P0/P1 与生产 → L2（L20–L34） | ✅ 一致 |
| **security-baseline L19–L21** | `[discover, assess, report, create_task, notify]` | **`[non_production_remediation, approved_low_risk_runbook]`** | `[production_account, production_rbac, cluster_admin, firewall_core_rule, calico_core_networkpolicy, kubernetes_api, harbor_admin_or_robot, da_soc_access_control, core_business_port, os_or_platform_upgrade]` | ⚠️ **L1 未限定"白名单/可回滚"前提 → CNS-005（P2）** |

### 结论

**两清两固一致性：闭环设计正确、UNKNOWN 处理正确、未引入新基础设施；但四项能力的统一状态模型只落在 `security-baseline.yaml`，三份对照基线未引用，且 L1 定义比 Baseline 略宽（P1/P2）。**

---

## 8. V0.1 Scope / HA Consistency

### 8.1 隐性范围膨胀检查（逐项）

| 检查项 | 是否被某文档变成 V0.1 必须实施 | 证据 |
|---|---|---|
| 多 Control Plane | ❌ 未膨胀 | Baseline §5.1（单 CP）；Adjudication L19/L117/L440（"V0.1 先 1 CP + 恢复演练，V0.2 按 SLA 触发 HA"）；TODO L42；Baseline §23 L559（V0.2） |
| 多集群 | ❌ | Adjudication L441；TODO L54 |
| 跨地域 DR | ❌ | Adjudication L441；Baseline §23 |
| Ceph | ❌ | Baseline §8.3 L203；ADR-004 L12；TODO L54 |
| Longhorn | ❌ | 同上 |
| Service Mesh | ❌ | Baseline §7 L183；ADR-002 L14；ADR-004 L12 |
| SIEM | ❌ | Baseline §10.1 L241 / §23 L560；ADR-005 L27；README L875（V0.3）/ L927 / L1277；TODO L54 未列（见 CNS-020） |
| SOAR | ❌ | README L1277；Baseline §25 L579 |
| CMDB | ❌ | Baseline §25 L579；CLM README §7 L116 |
| 完整 IAM | ❌ | README L1277 |
| 自动 Kubernetes 升级 | ❌ | Baseline §16.5 L437"V0.1 禁止…自动 Kubernetes minor/major upgrade"；CLM README §7 L117；upgrade-rules L37–L43 `forbidden_automatic_actions` |
| 自动 OS 升级 | ❌ | 同上 |
| 自动生产 Patch | ❌ | upgrade-rules L37–L43；CLM README §6 L100"生产环境禁止自动升级" |
| GPU | ❌ | README L295 仅为能力域枚举；TODO L54；Adjudication L441 |
| 复杂 Policy Engine | ❌ | TODO L54（"复杂 Policy Engine"）；Adjudication L441 |
| Task CRD | ❌ | Baseline L15/L561；ADR-008 L8；TODO L50/L54 |
| 常驻高权限 Agent | ❌ | Baseline §16.1 L377；ADR-007 L8；TODO L49 |

**结论：未发现任何隐性范围膨胀。** 上述 17 项在各文档中**一致地**被标为 V0.2+/Non-Goal。✅

### 8.2 反向检查：是否因强调"单 CP / 非 HA"而错误删除必备能力

**未发现。** V0.1 的备份、恢复、故障演练能力被**强化而非削弱**：

| 必备能力 | 是否保留 | 证据 |
|---|---|---|
| etcd 备份 | ✅ 每 6 小时 + 变更前 | Baseline §11.2 L279；ADR-006 L16 |
| ClickHouse BACKUP/RESTORE | ✅ 每日 + 切换前 | Baseline §11.2 L281；ADR-006 L18 |
| raw archive 文件备份 | ✅ 每日 | Baseline §11.2 L282 |
| Harbor 备份 | ✅ 每日/每周 | Baseline §11.2 L285 |
| Secret 加密备份 | ✅ 每次轮换 | Baseline §11.2 L286 |
| 离线/不可变副本 | ✅ | Adjudication L233；ADR-006 L12；TODO L37 |
| **真实恢复演练（6 类）** | ✅ | Baseline §11.3 L293 / §21 L522；ADR-006 §Required Drills；TODO TASK-064～066 |
| 故障演练 | ✅ | Baseline §20 S14 L490；TODO TASK-067 L1025 |
| RPO/RTO 声明 | ✅ RPO ≤24h；CP RTO ≤4h；DA-SOC RTO ≤8h | Baseline §11.3 L290–L291；ADR-006 L33 |
| "没有恢复演练的备份不计入 DoD" | ✅ | ADR-006 L35；RESTORE-DRILL-PLAN L3；Baseline L493 |

**单 CP 的代价被显式登记而非隐藏：**

```text
Baseline §2.2 L33  : "xw-cp-01 故障：Kubernetes 管理面和调度中断…需要恢复 Control Plane"
Adjudication L469  : "单 Control Plane 故障 | 高 | etcd snapshot、Git 重建、RTO 4h、V0.2 HA 触发条件"
Adjudication L440  : "三控制面 HA（除非实施前经 RTO/SLA 评审将其升级为硬门槛）"
```

### 8.3 HA 边界结论

Baseline §23 L559 与 Adjudication L448 明确 V0.2 引入 3 CP，**作为演进项而非 V0.1 项**，与"V0.1 非完整企业级 HA"定位一致。**HA 边界一致，无膨胀，无错误删减。**

---

## 9. Agent / L0-L1-L2 Consistency

### 9.1 L0/L1/L2 跨文档一致性

见 §7.4 表格。**结论：Baseline / Adjudication / ADR-007 / CLM / policies.yaml 五处一致；仅 `security-baseline.yaml` L20 的 L1 定义略宽（CNS-005，P2）。**

### 9.2 Agent 是否获得过大权限（逐项排查——审计 Critical 项）

| 权限 | 是否出现在任何文档 | 证据 |
|---|---|---|
| `cluster-admin` | ❌ 被明确禁止 5 次 | Baseline §13 L317"任何 `cluster-admin` 使用都必须是 break-glass、人工、短时、双人确认"；ADR-007 L20"Agent 不得使用 cluster-admin"；Adjudication L283"禁止业务和 Agent 使用 cluster-admin"；Baseline §21 L530"Agent 没有 cluster-admin、root 或任意 shell 权限"；account-baseline L36 `cluster-admin-binding` → `cluster_admin_is_break_glass_only`（L2） |
| root / 主机 root | ❌ | ADR-007 L20；Baseline §21 L530 |
| 长期 kubeconfig | ❌ | ADR-007 L20"不得使用…长期 kubeconfig" |
| 任意 shell | ❌ | Baseline §13 L325"执行任意 shell 或宿主机命令"（禁止项）；ADR-007 L20；Baseline §21 L530 |
| 生产 Secret | ❌（有严格例外条款） | Baseline §13 L321"访问 Secret 值，除非某个已批准 Runbook 明确需要且凭据临时注入"；§12.1 L299"n8n 使用业务专用 Secret"；Adjudication L287"Agent 只能读状态和执行白名单 Runbook" |
| 生产邮箱 | ❌ | ADR-007 L20"不得修改…生产邮箱"；Baseline §17 L448"生产邮箱和凭据 \| 负责授权 \| **不直接操作**" |
| 生产 DingTalk 凭据 | ❌ | ADR-007 L20"不得修改…DingTalk 目标凭据"；Adjudication L287；Baseline §17 L448 |
| 修改 RBAC/NetworkPolicy/CNI/StorageClass/节点/etcd | ❌ | Baseline §13 L322；§12.2 L305（列为 L2）；TODO L62 |
| 删除业务数据/PVC | ❌ | Baseline §13 L323；§12.2 L305（L2） |
| 修改 DA-SOC SQL/出数/出图 | ❌ | ADR-007 L20；Adjudication L443 |

### 9.3 是否存在绕过 L2 审批的路径（审计 Critical 项）

**逐一排查结论：不存在。**

| 潜在绕过路径 | 是否被封堵 | 封堵机制 |
|---|---|---|
| Agent 直接写生产资源 | ✅ 封堵 | ADR-007 L14"L2 由人工审批后才获得**短时授权**"；Baseline §12.2 L305 |
| Agent 通过修改 Task YAML 自行置为 approved | ✅ 封堵 | ADR-008 L16"Git 保存不可变审计记录"；Baseline §16.3 `approval.approver` 为独立字段；TODO L62"所有高风险动作走 Git Task、审批和审计" |
| CLM 自动生成升级 Task 后自动执行 | ✅ 封堵 | CLM README §6 L100"生产环境禁止自动升级…不得绕过审批"；upgrade-rules L37–L43；policies.yaml L4/L9/L14 `production_auto_upgrade: false`；TODO RISK-CLM-004 L1125（Critical 风险已登记 + 对策"L0/L1/L2、最小 RBAC、Git 分支保护、生产自动升级禁用"） |
| 两清两固自动整改生产 | ✅ 封堵 | security-baseline L21 L2 清单含 `production_account`/`production_rbac`/`firewall_core_rule`/`calico_core_networkpolicy`/`kubernetes_api`/`harbor_admin_or_robot`/`da_soc_access_control`；account-baseline L84 / port-baseline L80 / access-control-baseline L64 一致 |
| Agent 通过 `kubectl exec` 获得容器内权限 | ✅ 封堵 | Baseline §16.1 未授予 exec；account-baseline L36/L40（L2）；README L494"不直接修改 CNI"语义同源 |
| Agent 通过 nodes/proxy 或 hostPath 逃逸 | ✅ 封堵 | Baseline §12.3 L309 禁 privileged/hostNetwork/hostPID/hostIPC；ADR-004 L18 禁任意 hostPath |
| TODO 中出现"授予 Agent 更大权限"的任务 | ✅ 无 | TODO §2 L49、§3 L62、TASK-036 L575（生成**最小** RBAC 和 L0/L1/L2 策略） |

### 结论

**Agent / L0-L1-L2 一致性：优秀。** 9 类越权路径全部被至少两处独立文档封堵，且越权风险已登记为 Critical 级并配有对策（IMPLEMENTATION-RISKS RISK-012 / RISK-CLM-004 / RISK-SEC-003 / RISK-SEC-004）。**未发现任何绕过 L2 的路径，因此本次审计没有 Critical 发现。**

---

## 10. Detailed Findings

| ID | Severity | Category | File A | File B | Conflict | Recommended Resolution |
|---|---|---|---|---|---|---|
| **CNS-001** | **P1** | SoT / 状态模型 | `TODO.md` L129（TASK-005 Actions）："创建任务状态模型 `PENDING/IN_PROGRESS/BLOCKED/PASSED/ROLLED_BACK`" | `10-decisions/ADR/ADR-008-task-model.md` L12–L16：`open → analyzed → planned → pending-approval → approved/rejected → executing → verified/failed → closed/escalated`；`10-decisions/ARCHITECTURE-BASELINE-V0.1.md` L411–L414/L420–L426：`decision: pending\|approved\|rejected` / `status: pending\|passed\|failed` / `result: open\|closed\|escalated`；`04-security/security-baseline.yaml` L8：`discovered, assessed, task-created, approval-pending, remediation, verifying, closed, escalated`；`07-aiops/component-lifecycle/upgrade-rules.yaml` L35：`discovered, assessed, upgrade-required, task-created, approved, executing, verifying, closed, rolled-back, review` | 同一"Task 状态"概念存在 **5 套互不相同的枚举**，且无任何文档裁定适用范围与优先级。TODO L129 还声称"创建"该模型，与 ADR-008 已定义的生命周期直接竞争。**实施者无法确定 Task 应写入哪个状态值**，直接影响 TASK-005（L125）、TASK-060（L923）、TASK-CLM-005（L1111）、TASK-SEC-005（L1268） | 新增/修订 `10-decisions/ADR/ADR-008` 附录，裁定**状态维度归属**：① **运维 Task 生命周期**（权威 = ADR-008 L14）；② **实施任务状态**（权威 = TODO L129，仅用于 TODO 自身）；③ **检查/发现状态**（权威 = `security-baseline.yaml` L6 `PASS/FAIL/REVIEW/UNKNOWN` + L8 生命周期）；④ **组件升级状态**（权威 = `upgrade-rules.yaml` L35）。ADR-008 显式声明①为通用运维 Task 生命周期；Baseline §16.3 YAML 增加 `lifecycle:` 字段并映射到①；TODO L129 改为"引用 ADR-008 生命周期 + 定义实施任务状态（仅限 TODO）"。**Architecture Change Required: NO**（属 Schema 统一，不改架构结论） |
| **CNS-002** | **P1** | 定义 / Task Schema | `10-decisions/ARCHITECTURE-BASELINE-V0.1.md` L392–L427：`api_version: xuanwu/v0.1` / `kind: Task`，字段为 `spec{target,severity,risk,evidence,impact,plan,rollback}` + `approval{required,approver,decision}` + `execution{job,action,started_at,finished_at}` + `verification{status,evidence}` + `audit{agent_digest,git_commit,result}` | `04-security/security-baseline.yaml` L9–L11：`task_model.categories: [VULNERABILITY, PORT, ACCOUNT, ACCESS_CONTROL]` + `required_fields: [id, category, asset, finding, severity, evidence, expected_state, observed_state, remediation, approval_required, owner, status, verification, rollback]`；`10-decisions/ADR/ADR-008-task-model.md` L10："每个 Task 至少包含：ID、来源、目标、严重性、风险等级、证据、影响、计划、回滚、审批、Agent Job、执行结果、验证结果、审计 commit 和最终状态" | **同一个 `Task` 在三个文件中被定义三次，字段名与结构互不兼容**：Baseline 用 `spec.risk` / `approval.decision` / `verification.status`；security-baseline 用 `category` / `expected_state` / `observed_state` / `status`；ADR-008 用散文列举且无字段路径。实现时无法确定 Task YAML 的权威 Schema，且 CLM Upgrade Task（TASK-CLM-005 L1111）与安全 Task（TASK-SEC-005 L1268）将各自发明字段 | 裁定 **Baseline §16.3 为 Task 基础 Schema 的唯一定义**；`security-baseline.yaml` 改为**引用并扩展**（如 `extensions: {category, expected_state, observed_state}`），删除其重复的 `required_fields` 全集；ADR-008 只保留生命周期与"必须包含的语义类别"，删除字段清单式表述。**Architecture Change Required: NO** |
| **CNS-003** | **P1** | 字段类型 / 状态混用 | `04-security/port-baseline.yaml` L10 `record_fields: [... port_required, port_reason, priority, expected_source, actual_source, status]`；L22 `port_required: YES`、L34 `port_required: YES`、L58 `port_required: NO`、L70 `port_required: REVIEW` | `04-security/security-baseline.yaml` L6 `state_model.statuses: [PASS, FAIL, REVIEW, UNKNOWN]`（`REVIEW` 为**状态值**） | `port_required` 字段取值混用布尔（`YES`/`NO`）与状态值（`REVIEW`），而 `REVIEW` 在 `security-baseline.yaml` 中被定义为状态枚举成员。同一文件 L10 已有独立 `status` 字段，因此 `port_required` 不是状态字段 | 在 `port-baseline.yaml` 增加字段声明：`port_required` 取值限定 `YES \| NO \| REQUIRED \| UNKNOWN`（或加 `port_required_review: true` 布尔字段），使其**不与 `PASS/FAIL/REVIEW/UNKNOWN` 状态词汇重叠**；并在 `record_fields` 旁增加类型/枚举声明。**Architecture Change Required: NO** |
| **CNS-004** | **P1** | 同名不同义 | `04-security/security-baseline.yaml` L8 `lifecycle: [discovered, assessed, task-created, approval-pending, remediation, verifying, closed, escalated]` | `07-aiops/component-lifecycle/upgrade-rules.yaml` L35 `lifecycle: { statuses: [discovered, assessed, upgrade-required, task-created, approved, executing, verifying, closed, rolled-back, review] }` | 两文件**同名 `lifecycle` 键**，枚举**部分重叠但不相同**（security 有 `approval-pending`/`remediation`/`escalated`；upgrade-rules 有 `upgrade-required`/`approved`/`executing`/`rolled-back`/`review`）。若工具按 `lifecycle` 键聚合，将得到不一致的状态机。另 `planned` 在 ADR-008（Task 态）与 `components.yaml` L13 等（`upgrade.status`）中同名不同义 | 将 `upgrade-rules.yaml` 的键重命名为 `upgrade_lifecycle_statuses`（或 `upgrade.state.statuses`），`security-baseline.yaml` 保持 `state_model.lifecycle`，并在两文件互相加注引用；`components.yaml` 的 `upgrade.status` 改为引用 `upgrade-rules` 的枚举。**Architecture Change Required: NO** |
| **CNS-023** | **P1** | SoT / 统一模型未落地 | `07-aiops/component-lifecycle/README.md` L144："**All four capabilities use `PASS`, `FAIL`, `REVIEW` and `UNKNOWN`**"；L137–L143 声明四份文件的分工；`04-security/security-baseline.yaml` L5–L8 定义 `state_model.statuses` 与 `lifecycle` | `04-security/port-baseline.yaml` L5–L10、`04-security/account-baseline.yaml` L5–L6、`04-security/access-control-baseline.yaml` L5–L6：三者均有 `record_fields`（含 `status`）但**均未声明 `state_model`**，也未以任何字段引用 `security-baseline.yaml`；三者的 `rules:` 动作词汇互不相同：`FAIL_AND_CREATE_PORT_TASK`（port L73–L76）／`FAIL_AND_REDACT_AND_CREATE_INCIDENT`、`UNKNOWN_AND_CREATE_ACCOUNT_TASK`（account L43–L46）／`FAIL_AND_CREATE_ACCESS_CONTROL_TASK`、`BLOCK_AND_ESCALATE_TO_L2`（access-control L57–L61） | "四项能力统一采用同一状态模型"这一要求**只以散文形式存在于 CLM README 与 Baseline §25**，未落入三份对照 YAML 的可校验字段。后果：① 工具的 `status` 校验无权威基准；② 三份文件的 `rules:` 动作词汇各自发明（`_AND_CREATE_INCIDENT`、`BLOCK_AND_ESCALATE_TO_L2`），无法被同一解析器处理；③ `security-baseline.yaml` L11 的 `required_fields` 与三份基线的 `record_fields` 不是同一集合 | 在三份对照 YAML 中显式加入 `state_model_ref: 04-security/security-baseline.yaml`，并把各自 `rules:` 的动作映射为统一词汇（建议限定为 `FAIL_AND_CREATE_TASK`、`UNKNOWN_AND_CREATE_TASK`、`REVIEW_AND_CREATE_TASK`、`BLOCK_AND_ESCALATE_L2` 四类，`INCIDENT` 作为附加属性而非动作名）。**Architecture Change Required: NO** |
| **CNS-005** | P2 | 定义 L1 | `10-decisions/ARCHITECTURE-BASELINE-V0.1.md` L304：L1 = "白名单 Runbook，例如 `da-soc-render` 重启、非生产临时文件清理；**必须有前置条件和自动验证**"；L467："受策略范围限制的非破坏动作…**必须自动验证，失败自动升级人工**"；`07-aiops/component-lifecycle/README.md` L97：L1 = "验证环境 Patch、非生产组件或**可回滚白名单 Runbook**" | `04-security/security-baseline.yaml` L20：`L1: [non_production_remediation, approved_low_risk_runbook]`（未声明"前置条件/可回滚/自动验证"约束） | 两清两固基线的 L1 定义缺少 Baseline 与 CLM 都要求的**可回滚 / 前置条件 / 自动验证**前提，形成"更宽的 L1" | 在 `security-baseline.yaml` L20 补全为 `L1: [non_production_remediation_with_approved_reversible_runbook]`，并在文件内注明"L1 严格遵守 Baseline §19 与 CLM §6"。**Architecture Change Required: NO** |
| **CNS-006** | P2 | 状态/引用 | `README.md` L1143："**当前版本：V0.1 --- Architecture Frozen / Implementation TODO Ready**" | `README.md` L1269："**Status:** Planning / Architecture"；L1157："…当前安装尚未开始。"；L1147："当前记录的是候选基线，不是官方认证的完整组合" | 同一文件内既称"Architecture Frozen / TODO Ready"，又在文末声称"Planning / Architecture"，与 Baseline §24 L567"唯一实施依据"、TODO L3"Implementation Readiness: READY"、TODO L18"Architecture \| FROZEN"三处不一致 | 统一 README 全局状态为 `V0.1 — Architecture Frozen / Implementation TODO Ready (Installation: NOT STARTED)`，同步修订 L1269 的 `Status` 字段与 L1143 表述。**Architecture Change Required: NO** |
| **CNS-007** | P2 | 范围清单遗漏 | `README.md` §15 V0.1 主要能力清单 L832–L843（仅列 Kubernetes/KubeSphere/基础网络/CNI/基础 NetworkPolicy/Harbor/RBAC/基础监控/基础日志/基础审计/基础备份/基础 AI Ops） | `README.md` L1159–L1163（CLM 为 V0.1 能力）、L1271–L1277（两清两固为 V0.1 能力） | 补丁新增的 CLM 与两清两固已是 V0.1 必做能力（TODO 有 12 个专属任务），但 README §15 的 V0.1 能力清单与 §16 暂缓清单均未更新 | 在 README §15 L843 后补入"组件生命周期管理（CLM）最小闭环"与"两清两固安全运营最小闭环"，并在 §16 说明二者边界。**Architecture Change Required: NO** |
| **CNS-008** | P2 | 引用错误 | `09-implementation/RESTORE-DRILL-PLAN.md` L41："任一关键恢复失败则 **TASK-061～TASK-064** 不通过，禁止生产切换" | `TODO.md` L937 `### TASK-061 — 实现 CrashLoopBackOff 真实闭环`；L951 `### TASK-062 — 验证 Agent 失败、超时和回滚`；恢复演练实为 L981 `TASK-064`、L995 `TASK-065`、L1009 `TASK-066` | 演练计划的失败门禁引用了 AI Ops 任务（TASK-061/062），而非恢复演练任务，导致门禁指向错误任务 | 修订为 `TASK-064～TASK-066`。**Architecture Change Required: NO** |
| **CNS-009** | P2 | 引用错误 | `09-implementation/EXTERNAL-DEPENDENCIES.md` L37："离线/不可变备份副本 \| Backup Owner \| TASK-040 \| **TASK-061** \| TBD / BLOCKING" | `TODO.md` L937 TASK-061 是 CrashLoopBackOff 闭环；备份实现实为 L619 `TASK-039`、L633 `TASK-040`、L605 `TASK-038`（初始化备份仓库和不可变副本） | 外部依赖表中"离线/不可变备份副本"的 Blocking Task 指向与备份无关的 AI Ops 任务 | 修订 Blocking Task 为 `TASK-038`（该任务标题即"初始化备份仓库和不可变副本"）。**Architecture Change Required: NO** |
| **CNS-010** | P2 | 依赖前置（forward dependency） | `TODO.md` L874：TASK-056（Phase 13 Cutover，L877）`Dependencies: TASK-041、TASK-053～055、TASK-061～066` | `TODO.md` L979 `## 19. Phase 15 — Failure / Recovery Drills`（TASK-064～066）与 L921 `## 18. Phase 14 — AI Ops MVP`（TASK-060～063）均在 TASK-056 之后；L1158 的依赖图只画 `TASK-053 ─> TASK-054 ─> TASK-055 ─> TASK-056`，**不含**该边 | Cutover Gate 评审包依赖了排在其后的 Phase 14/15 任务，形成逻辑倒置；且依赖图与任务字段不一致 | 从 TASK-056 Dependencies 移除 `TASK-061～066`（Baseline §20 L493 仅要求 S7/S8 恢复验证先行），或明确改为"引用但不阻塞"。**Architecture Change Required: NO** |
| **CNS-011** | P2 | 依赖前置（forward dependency，系统性） | `TODO.md` L470：TASK-028（Phase 6 Harbor，L459）`Dependencies: TASK-027、TASK-040`；**同样模式**见 L428：TASK-026（Phase 6 Harbor，L431）`Dependencies: TASK-006～010、TASK-002`（含 TASK-040？否——但 L542 与 L860 同类） | `TODO.md` L633：TASK-040（Phase 9 Backup）；L647：TASK-041（Phase 9）；L605：TASK-038（Phase 9）；L879：TASK-057（Phase 13） | 位于 Phase 6/7/12 的任务依赖 Phase 9/13 的任务：**L470（TASK-028→TASK-040）、L542（TASK-033→TASK-038）、L860（TASK-055→TASK-057）三处同型**，表明是对"备份/切换能力"的引用未落到具体任务号 | 统一规则：Phase N 任务不得依赖 Phase > N 任务；把"Harbor 可备份""观测栈可备份"改为验证性动作（不依赖具体备份实现任务），或改为依赖同阶段的存储/证据任务。**Architecture Change Required: NO** |
| **CNS-012** | P2 | 依赖成环 | `TODO.md` L1283：TASK-SEC-006 `Dependencies: TASK-SEC-002～005、**TASK-067**` | `TODO.md` L1169 依赖图：`TASK-SEC-006 ─> TASK-067 ─> TASK-068`；L1227 §24 停止条件："当 `TASK-CLM-006`、`TASK-SEC-006`、`TASK-067` 和 `TASK-068` 完成后停止"；L1027 TASK-067 依赖含 `TASK-CLM-006` 但**不含 TASK-SEC-006** | **TASK-SEC-006 与 TASK-067 互为依赖，形成循环依赖**，导致两条任务都无法进入执行。同时 TASK-067/TASK-068 未列 TASK-SEC-006，而 §24 DoD 第 13 项（L1224）已把两清两固纳入 V0.1 完成条件 | 依依赖图修正：TASK-SEC-006 的 Dependencies 去掉 `TASK-067`（改为 `TASK-SEC-002～005`）；TASK-067（L1036）与 TASK-068（L1050）的 Dependencies 补入 `TASK-SEC-006`。**Architecture Change Required: NO** |
| **CNS-013** | P2 | 依赖图 vs 字段 | `TODO.md` L1156 图：`TASK-042 ─> TASK-043 ─> TASK-044 ─> TASK-045 ─> TASK-046`（**缺** TASK-044 → TASK-022～024 的存储前置） | `TODO.md` L693：TASK-044 `Preconditions: TASK-022～024、TASK-042～043 完成`；另 L1157 图 `TASK-047/TASK-048/TASK-049 ─> TASK-050` 与 L758（TASK-048 `Dependencies: TASK-046、TASK-040、TASK-047`）、L772（TASK-049 `Dependencies: TASK-004、TASK-042～048`）不一致 | 依赖图（§21 L1139–L1170）与任务字段（Preconditions/Dependencies）之间存在**至少 2 类不一致**：图形缺边、图形汇聚点与字段不符 | 以任务字段为准重新生成 §21 依赖图；建议改为自动生成（脚本从 Preconditions/Dependencies 抽取）以避免再次漂移。**Architecture Change Required: NO** |
| **CNS-014** | P2 | 字段一致性（文件自身规则） | `TODO.md` L59 §3 实施规则："所有任务都必须保留 **Validation、Evidence、Rollback**；没有证据不算完成"；L1222 §24 DoD 第 11 项："所有任务有 Validation、**Evidence**、Approval、**Rollback** 和 Owner" | `TODO.md` L1231–L1239 TASK-SEC-001：缺 **Rollback**、缺 **Risk**、缺 **Evidence**（其余字段齐全） | 文件自身规定每个任务必须含 Evidence/Rollback，但 TASK-SEC-001（两清两固的**基线定义任务**，决定另外 6 个任务的状态模型）缺少 Evidence 与 Rollback 字段，自我违反。TASK-SEC-002～006 已含 Rollback（L1246/L1255/L1264/L1273/L1282），仅 TASK-SEC-001 例外 | 为 TASK-SEC-001 补 `Rollback`、`Risk`、`Evidence` 三个字段。**Architecture Change Required: NO** |
| **CNS-015** | P2 | 观测栈部署主体歧义 | `TODO.md` L478：TASK-029 `Inputs: **KubeSphere/Prometheus stack** 兼容版本、保留周期、TLS/RBAC` | `TODO.md` L479：同任务 `Actions: 部署 Prometheus、Grafana、Alertmanager…**不部署第二套指标系统**`；Baseline §10.1 L236：唯一栈为 "Prometheus → Grafana → Alertmanager → n8n/DingTalk"；CLM README §11 L151："The existing Prometheus/Grafana/Alertmanager stack"；components.yaml L94–L123 独立登记为 `observability.*` 组件 | TASK-029 的 Inputs 写作 "**KubeSphere/Prometheus stack**"，可能被读为"部署 KubeSphere 自带监控栈"或"部署 KubeSphere 内的 Prometheus 栈"，而同任务 Actions 要求"不部署第二套指标系统"。**文档未说明最终部署主体是 KubeSphere 内置监控还是独立栈**；若实施者按 Inputs 字面部署 KubeSphere 内置监控，将与"组件注册表独立登记 observability.*"产生双栈风险 | 将 L478 Inputs 改为明确的单一主体，例如"独立 Prometheus/Grafana/Alertmanager 组件（复用 KubeSphere 管理面视图，不启用其内置监控栈）"，并写明与 Baseline §10.1 的一致性。**Architecture Change Required: NO** |
| **CNS-016** | P2 | 状态引用/编号 | `TODO.md` L1229 `## 20B. Two-Clear-Two-Firm Security Operations`（文件最后一个章节，位于 L1208 `## 24. Definition of Done` 之后） | `TODO.md` L1208 `## 24. Definition of Done`；L1224 DoD 第 13 项已引用两清两固覆盖率；L1227 停止条件引用 `TASK-SEC-006`；L1167–L1169 §21 依赖图引用 `TASK-SEC-*` | 20B 排在 24 之后，导致 §24 与 §21 **前向引用**（正文先引用后定义）尚未出现的章节。`## 20A`（L1053）位置正常 | 将 `## 20B` 上移至 `## 20A` 之后（L1139 `## 21` 之前）。**Architecture Change Required: NO** |
| **CNS-017** | P2 | 需求/版本张力 | `README.md` §15 L833/L835/L837 将 `KubeSphere`、`CNI`、`Harbor` 列为 V0.1 **既定主要能力** | `README.md` L1152–L1155、`VERSION-MATRIX.md` L20–L24、`TODO.md` L20–L24 将同一批组件标为 `Candidate / Pending Compatibility Validation` / `Pending Freeze` | 能力清单（既定）与版本状态（候选未冻结）并列，未说明"能力范围已冻结、具体版本待冻结"的层次关系 | 在 README §15 能力清单加注"能力范围已冻结；具体版本见 `09-implementation/VERSION-MATRIX.md`，当前为候选待冻结"。**Architecture Change Required: NO** |
| **CNS-018** | P2 | 覆盖范围声明 | `07-aiops/component-lifecycle/README.md` L128："Component Coverage \| 已纳入 CLM 管理的组件 / 应管理组件 \| **100%**"；L134："Unknown Version Count \| … \| **0**" | `09-implementation/VERSION-MATRIX.md` §3 含 3 个注册表未单列的条目：`Kernel`（L30）、`local-path`（L31）、`Ingress Controller`（L33） | VERSION-MATRIX 出现 3 个组件，而 `components.yaml` 的 16 项注册表未单列它们，使 "Component Coverage = 100%" 的**判定基准（"应管理组件"的枚举来源）不明确** | 在 `components.yaml` 增补 `platform.kernel`、`storage.local-path`、`ingress.nginx`，或在 CLM README 明确"应管理组件"的枚举来源。**Architecture Change Required: NO** |
| **CNS-019** | P3 | 路径引用不统一 | `TODO.md` L86 `09-implementation/VERSION-MATRIX.md`（带路径） | `TODO.md` L81/L95/L158/L186/L216/L274/L318/L1058 使用裸 `VERSION-MATRIX.md`；L115 规划 `security/`（无编号）而 L1233/L1243/L1261 引用 `04-security/...`；L144 裸 `EXTERNAL-DEPENDENCIES.md` 而 L1176 带路径；L852/L910 `CUTOVER-CHECKLIST.md`/`ROLLBACK-PLAN.md` 无任何前缀；`09-implementation/`、`04-security/`、`policies.yaml`、`upgrade-rules.yaml`、`build_workflow.py` 均未出现在 L115 布局清单中 | 同一文件混用裸名与全路径；TASK-004（L115）规划的仓库布局与实际被引用的目录/文件不对应，实施者按 L115 建库将无法定位被引用文件 | 统一为仓库根相对路径；在 TASK-004 L115 布局中补入 `09-implementation/`、`04-security/`、`policies.yaml`、`upgrade-rules.yaml`、`build_workflow.py`。**Architecture Change Required: NO** |
| **CNS-020** | P3 | 清单完备性 | `TODO.md` L54 V0.1 不引入：`xw-opsapi、Ceph、Longhorn、ELK、Istio/Linkerd、GPU、多集群、复杂 Policy Engine、Task CRD` | `10-decisions/ARCHITECTURE-BASELINE-V0.1.md` §22 L560 / `ARCHITECTURE-ADJUDICATION-V0.1.md` L441 的 Non-Goals 还包含 `SIEM`、`CMDB`、`多租户`、`跨地域 DR`、`完整 Runtime/Supply Chain Security` | TODO 的"不引入"清单比 Baseline/Adjudication 的 Non-Goals 少 5 项（虽 README L1277 与 Baseline §25 L579 另有覆盖） | 将 TODO L54 与 Baseline §22 Non-Goals 对齐，或直接引用 Baseline Non-Goals。**Architecture Change Required: NO** |
| **CNS-021** | P3 | 拼写/风格 | `07-aiops/component-lifecycle/upgrade-rules.yaml` L35 `rolled-back`；`TODO.md` L1130 `rolled-back` | `TODO.md` L129 任务状态 `ROLLED_BACK` | 同一语义三种写法（`rolled-back` / `ROLLED_BACK`） | 统一为 `rolled_back` 或 `ROLLED_BACK`。**Architecture Change Required: NO** |
| **CNS-022** | P3 | 文档结构 | `09-implementation/EXTERNAL-DEPENDENCIES.md` L45–L50：两清两固 5 行依赖表与主表之间无空行分隔 | 同文件 L51 `## 1. Ownership Rule` | 表格与后续标题紧贴，Markdown 渲染可能异常；且该文件后半使用 `## 1.` 编号而前半无编号 | 在 L50 与 L51 之间补空行，并统一编号风格。**Architecture Change Required: NO** |

---

## 11. Potential False Positives

以下各项**经检查后判定不是真正的一致性冲突**，明确排除，以免后续被误当作缺陷修复：

| # | 表象 | 为何不是冲突 |
|---|---|---|
| FP-01 | `VERSION-MATRIX` 给出 Kubernetes `v1.30.6` / KubeSphere `4.1.x`，而 Baseline 未固定版本 | Baseline §5.1 L125 明确"版本必须来自同一官方兼容矩阵，**实施前**冻结完整版本号"；VERSION-MATRIX L10 明确区分 `Candidate` 与 `Frozen`，L47 声明"候选基线，不是官方认证"。**这是"待冻结"的正常状态，不是冲突** |
| FP-02 | `Baseline` 未点名 UOS / K8s / KubeSphere 具体版本，而 README/VERSION-MATRIX/TODO/components.yaml 都写了 | Baseline 是架构依据，版本由 VERSION-MATRIX 承载（CLM README §4 L69 已显式分工）。**分工正确，不是缺口** |
| FP-03 | `Calico` 版本在 VERSION-MATRIX L24 与 components.yaml L50 均为 `unknown` | 二者同为"待冻结"，语义一致。**不是冲突** |
| FP-04 | Baseline §4.1 L79 的"服务网 `10.20.30.0/24`"与 L81 的"Service CIDR `10.96.0.0/12`"都含"服务"字样 | "服务网"是物理 VLAN（承载 Harbor/备份/恢复流量），"Service CIDR"是 Kubernetes ClusterIP 网段，职责完全不同。Adjudication §12 L250–L252 同口径。**术语相近但语义不冲突**（可读性优化属 P3） |
| FP-05 | `RESTORE-DRILL-PLAN.md` 定义 D1–D4 四个 Drill，而 TODO 只有 TASK-064/065/066 三个恢复演练任务 | TASK-066 标题同时覆盖 Harbor、Local PV、POP3 replay 三类，L1015 明确输出 "D3 Harbor、Local PV、D4 POP3 replay 报告"。**任务数与 Drill 数不必一一对应，覆盖完整** |
| FP-06 | CLM 与 两清两固 都以"漏洞"为主题 | Baseline §25 L575 与 CLM README §10 L137 已明确："漏洞和生命周期继续由 CLM 管理；端口、账号和访问控制分别由三份 baseline 管理"。**分工已声明，不是重复建设** |
| FP-07 | `security-baseline.yaml` L2 / `port-baseline.yaml` L2 等声明 `source_of_truth: git`，而 Baseline L266 也声明 Git 为 Source of Truth | 二者层级不同：Baseline 声明"Git 是所有声明式定义的 SoT"（总原则），各 YAML 声明"本文件以 Git 为存储"（实现方式）。**不是两份文件争抢同一事实的所有权** |
| FP-08 | README 未提及 5 台 VM、1 CP + 2 Worker、Harbor 外置 | README 是项目总纲，不承载实施拓扑；Baseline §2.1 已完整定义。**"未提及"不等于"冲突"** |
| FP-09 | TODO L969/L1043 出现 `Task CRD` | L969 语境为 "Agent 不能…创建第二份 TODO…评估 Task CRD"类表述中的**评估/延后**语义，L1043 为 V0.2 backlog 登记。**不是"V0.1 要建 CRD"** |
| FP-10 | 5 份候选方案 (`09-implementation/00-architecture-review/`) 中含 Promtail / Velero / MinIO / Task CRD / 3 CP 等已否决内容 | Baseline §24 L567 明确"候选方案不具有实施权威性"。**候选记录不属于一致性审计范围**（本审计已排除该目录） |
| FP-11 | `Baseline §23 L559` 出现"三控制面 HA" | 位于 `## 23. Future Evolution`，明确标注 V0.2。**属演进计划，不是 V0.1 范围膨胀** |
| FP-12 | ClM `policies.yaml` 出现 `approval_level: L1`（non-production） | 与 Baseline L467 L1 定义一致（非生产、可回滚）。**不是越权** |
| FP-13 | `account-baseline.yaml` L84 L2 动作含 `rotate_registry_credential` | 属凭证轮换，Baseline §12.2 L305 将"凭据变更"列为 L2。**一致** |
| FP-14 | `upgrade-rules.yaml` L28 `upgrade_required: review`（非布尔） | CLM README L83–L84 明确"版本未知：高风险状态…进入 REVIEW"，`review` 是三值之一（true/false/review）。**设计如此，不是类型错误**（但与 CNS-001 的状态词汇重叠需一并处理） |

---

## 12. No-Issue Areas

以下关键区域经**逐项核对后确认一致**，明确记录以便后续审计复用（避免重复劳动）：

### 12.1 项目定位（C01）—— 完全一致 ✅

「玄武云盾是平台，DA-SOC 是第一个业务应用」在所有关键文档中一致表达，且**无一处**含糊或反向表述：

| 文件 | 行 | 表述 |
|---|---|---|
| README | L11–L12 / L27–L28 / L942 / L963–L964 | "AI 原生私有云平台项目" / "首先承载 DA-SOC" / "玄武云盾**不是** DA-SOC 的组成部分" / "DA-SOC 是玄武云盾的第一个核心业务承载对象" |
| Baseline | §1 L13 / §18 L453–L455 | "DA-SOC 采用验证双跑、生产单活" / IT 与业务职责划分 |
| Adjudication | §3.1 L56 | "玄武云盾是平台，DA-SOC 是第一个核心业务应用" |
| ADR-001 | L8 | "正式生产承载位置为 Kubernetes 的 `da-soc` Namespace" |
| TODO | §1 L33 / §2 L53 | "在 `da-soc` Namespace 实际承载" / 业务硬约束清单 |
| components.yaml | L67/L77/L86/L146 | `data.clickhouse` / `business.n8n` / `business.render-archive` / `image.da-soc` 的 `owner: da-soc` |

### 12.2 架构结论（C02）—— 完全一致 ✅

见 §2 完整性矩阵：**17 项架构结论在 5–8 份文件间全部一致，零冲突。**

### 12.3 版本号（C03）—— 无冲突 ✅

见 §3：**17 类组件未发现任何版本号互相矛盾；Candidate/Pending/Frozen 状态使用正确。**

### 12.4 范围/HA 边界（C08）—— 无膨胀、无错误删减 ✅

见 §8：**17 项潜在膨胀全部为 V0.2+/Non-Goal；备份/恢复/演练能力完整保留并强化。**

### 12.5 Agent 权限与 L2 审批（C09）—— 优秀 ✅

见 §9：**9 类越权路径全部被封堵；未发现任何绕过 L2 的路径。**

### 12.6 `UNKNOWN` 处理 —— 优秀 ✅

见 §7.2：**8 处独立声明均明确 `UNKNOWN` 不得为 PASS/SAFE，无一处误处理。**

### 12.7 单一观测栈纪律 —— 一致 ✅

`Prometheus + Grafana + Alertmanager + Fluent Bit + Loki` 作为唯一栈，在 Baseline §10.1 L236、Adjudication §16 L291、ADR-005 L8、TODO L47、VERSION-MATRIX L34–L38、components.yaml L94–L143 **六处一致**；Promtail/ELK/SIEM 的禁止在 ADR-005 L27、Baseline L241、Adjudication L461、TODO L54 **四处一致**。

### 12.8 双 n8n 消费生产邮箱 —— 一致禁止 ✅

见 §2 末尾四行引文表：**三个独立文档 + 一份 Cutover Checklist 一致禁止。**

### 12.9 备份五路径与恢复演练门槛 —— 一致 ✅

`Git + etcd snapshot + ClickHouse BACKUP/RESTORE + 文件备份`（+ Harbor 备份）在 Baseline §11.2 L279–L286、Adjudication §14 L267–L277、ADR-006 L8、TODO L48 **四处一致**；"没有恢复演练的备份不计入 DoD"在 ADR-006 L35、RESTORE-DRILL-PLAN L3、Baseline §11.3 L293 **三处一致**。

### 12.10 候选方案无实施权威 —— 一致声明 ✅

Baseline §24 L567 声明"候选方案不具有实施权威性"；候选目录未被任何实施文档引用为依据。

---

## 13. Final Verdict

### Q1 当前仓库是否存在阻断 V0.1 实施的 P0/P1 一致性问题？

**P0：不存在。**

**P1：存在 5 项，但 4 项不阻断实施启动（TASK-001～005 属 Phase 0 参数冻结阶段，可在该阶段内解决）；1 项（CNS-012 循环依赖）会实质阻塞验收链路：**

| ID | P1 问题 | 是否会阻断 TASK-001 启动 |
|---|---|---|
| CNS-001 | Task 状态存在 5 套枚举，无权威裁定 | ❌ 不阻断 TASK-001（参数冻结），但**阻断 TASK-005**（其 Actions 即"创建任务状态模型"）→ **必须在 Phase 0 内解决** |
| CNS-002 | Task Schema 在 3 个文件中重复定义 | ❌ 不阻断 TASK-001/002/003/004，**阻断 TASK-005 与 TASK-060** → 须在 Phase 0 内解决 |
| CNS-003 | `port_required: REVIEW` 状态值混用 | ❌ 不阻断；影响 TASK-SEC-002（L1241）→ 可在 Phase 20B 前解决 |
| CNS-004 | `lifecycle` 同名不同义（security vs upgrade-rules） | ❌ 不阻断；影响 TASK-CLM-001～003 与 TASK-SEC-001 → 可在 Phase 20A/20B 前解决 |
| CNS-023 | 两清两固统一状态模型未落入三份对照 YAML | ❌ 不阻断；影响 TASK-SEC-001～004（L1231/L1241/L1250/L1259）→ 可在 Phase 20B 前解决 |

**另需注意（P2 中唯一具有链路阻塞性的 1 项）：** **CNS-012（TASK-SEC-006 ↔ TASK-067 循环依赖）**。它虽定为 P2（因为不阻断 Phase 0，也不阻断任何生产切换门禁），但**会使 Phase 16 最终验收无法以无环顺序执行**，因此必须在进入 Phase 16 之前修正。

**判定：`PASS WITH P1`。** 5 项 P1 全部落在**状态模型 / Schema / 文档状态**的元数据层面，不涉及架构结论、安全边界或生产风险；且**全部可由 TODO 既有任务的 Actions 吸收**——CNS-001/CNS-002 恰好是 TASK-005（L129）与 TASK-004（L111）的既定工作内容，CNS-003/CNS-004/CNS-023 恰好是 TASK-SEC-001（L1234）与 TASK-CLM-001（L1055）的既定工作内容。**因此不需要新增任务，只需要在既有任务中明确"以哪一份为权威"。**

### Q2 是否建议进行一次统一修订？

**建议：是，但限定为"状态模型与 Task Schema 统一"这一件事，且不重新讨论架构。**

修订范围（最小集）：

```text
1. 新增或修订 ADR（建议作为 ADR-008 的附录或独立 ADR-009「Task 状态与 Schema 统一」），裁定：
   ① 运维 Task 生命周期权威 = ADR-008 L14（并补齐映射）
   ② 实施任务状态权威 = TODO L129（仅限 TODO 自身）
   ③ 检查/发现状态权威 = security-baseline.yaml L6
   ④ 组件升级状态权威 = upgrade-rules.yaml L35（键名改为 upgrade_lifecycle_statuses）
2. Baseline §16.3 增加 lifecycle 字段并引用 ①；security-baseline.yaml 的 task_model 改为引用并扩展 Baseline §16.3
3. 修正 CNS-003 字段枚举；CNS-008/CNS-009 引用错误；CNS-006 README 状态行；CNS-012 循环依赖
4. 一次性修正 CNS-010~CNS-022（依赖/编号/图字段/路径），不涉及架构
```

**不建议**启动全局文档重写，也**不建议**重新做 Architecture Review（理由见 Q5）。

### Q3 是否需要修改 Architecture Baseline？

**不需要修改架构结论；需要 2 处最小补充（非架构性）：**

| 补充项 | 位置 | 性质 |
|---|---|---|
| 在 Task YAML 中增加 `lifecycle:` 字段并引用 ADR-008 生命周期 | Baseline §16.3 L392–L427 | Schema 补齐，**不改架构** |
| 在 Baseline 中明确"Task 状态有 4 个维度、各自权威文件"的索引（1 段） | Baseline §16.3 或 §24 | 澄清，**不改架构** |

**Baseline 的以下结论均无需改动：** 5 台 VM、1 CP + 2 Worker、Harbor/备份外置、local-path、单一观测栈、Git Task 无 CRD、短生命周期 Job Agent、无 `xw-opsapi`、DA-SOC 单活切换、L0/L1/L2 边界、RPO/RTO。**4 项 P1 全部是元数据定义问题，不是架构问题。**

### Q4 是否可以直接进入 TASK-001？

**可以直接进入 TASK-001（参数冻结），但需附条件：**

**可以立即开始（无需等待修订）：**

```text
TASK-001  建立实施参数冻结记录        ✅ 可开始
TASK-002  冻结版本、镜像和 digest      ✅ 可开始（注意 FP-01：候选状态需在 TASK-002 内闭环）
TASK-003  冻结 Secret 管理和恢复方案    ✅ 可开始
TASK-004  冻结 Git Source of Truth     ✅ 可开始（顺带解决 CNS-002 的 SoT 归属）
TASK-006  交付五台 VM 和数据盘          ✅ 可开始（外部依赖 L8 BLOCKING）
```

**必须在 Phase 0 内解决 P1 的 2 项后才可继续（阻断对象）：**

```text
CNS-001 + CNS-002  →  阻断 TASK-005（建立实施门禁、变更和证据目录，TODO L125）
                       阻断 TASK-060（实现 Git Task、Job 模板和审批入口，TODO L923）
                       ⇒ 必须在 TASK-005 开工前完成统一
CNS-003 + CNS-004  →  阻断 TASK-SEC-001（TODO L1231）、TASK-CLM-001（TODO L1055）
                       ⇒ 可在进入 Phase 18（L1229）与 Phase 20A（L1053）前完成
```

**结论：** `TASK-001 ～ TASK-004` 立即执行；**`TASK-005` 之前必须完成 CNS-001/CNS-002 的统一修订**。这是唯一的时间顺序约束。

### Q5 是否建议再次进行架构设计？

**不建议。**

理由：

1. **架构层无任何矛盾。** 本次审计在 C02/C03/C08/C09 四个核心类别上**未发现一处架构结论冲突**（§2、§3、§8、§9）。全部 17 项架构结论在多份文件间一致。
2. **4 项 P1 全部是元数据/引用层面的问题**，修复方式是"裁定权威 + 修正引用"，而非"重新设计"。任何重新架构都会重复已有结论并引入新的不一致。
3. **反向风险更大。** Baseline §24 L570 与 Adjudication L502 均明确"不得在 Baseline 未冻结前开始 Kubernetes 实施"；而 Baseline §24 L567 已声明"本文件是实施唯一架构依据"。此时重启架构评审会作废 TODO 中 80 个已编号任务与 12 个 CLM/两清两固任务，成本远超收益。
4. **正确的下一步是"一次性元数据修订 + Baseline 冻结确认"**，而非新一轮设计：

```text
5 Agents 独立审计（进行中）
        ↓
交叉比较 → 识别共同问题（本报告提供 4 项 P1 + 16 项 P2/P3 候选）
        ↓
唯一裁决（状态模型与 Task Schema 统一）
        ↓
一次性修订（仅元数据与引用，不改架构）
        ↓
Baseline 冻结确认
        ↓
TASK-001（可立即并行启动）
```

### 审计结论一句话

> **仓库在架构、版本、范围、HA 边界、Agent 权限与安全纪律上是一致的，可以进入实施；但最近三次补丁在"Task 状态词汇"（3 套→5 套）与"Task Schema"（1 处→3 处）上引入了重复定义，加上 3 处任务编号引用错误、3 处前置/依赖字段不一致与 1 处章节乱序，必须在 Phase 0 的 TASK-005（及 Phase 18/20A 的 TASK-SEC-001 / TASK-CLM-001）内一次性统一，不需要重新做架构设计。**

---

**审计约束声明：** 本次审计为只读；未修改、未创建（除本报告）、未删除任何现有项目文件；未触碰 `TODO.md`、Baseline、ADR、CLM、两清两固或任何 YAML。本报告为新建文件，位于 `audit/dsh-V0.1-CONSISTENCY-AUDIT.md`。
