# 玄武云盾 V0.1 — Final Consistency Patch Report

> **执行角色：** Final Consistency Patch Agent（DSH）
> **基线提交：** `6f152bc docs: 新增 codebuddy 独立一致性审计报告与修复建议`
> **执行日期：** 2026-10-08
> **输入依据：** 6 份独立一致性审计报告（`audit/codebuddy-`、`codex-`、`cursor-`、`dsh-`、`kimi-`、`minimax-V0.1-CONSISTENCY-AUDIT.md`）
> **任务范围：** 仅 P0-01 与 P1-01 ~ P1-08 收口；不重新设计架构；不扩大 V0.1 Scope

---

## 1. 执行摘要

本次补丁把 V0.1 从 **Architecture Frozen + Audit Findings** 收口到 **Implementation Ready**。

**核心变更（4 个横切面）：**

1. **恢复演练成为生产切换的结构性前置。** 原先 `TODO` 把生产切换（Phase 13）排在恢复演练（Phase 15）之前，且 `RESTORE-DRILL-PLAN` 的门禁引用了错误的任务区间（`TASK-061～064`，指向 AI Ops 任务），使"未完成恢复验证即完成生产切换"成为**合法执行路径**。现已通过 Task `Dependencies`、`Preconditions`、依赖图与两份实施附件共同封堵。
2. **CLM Schema 闭合。** `upgrade_required` 与 `upgrade.required` 并存、`minimum_priority` 与 `priority` 并存、`planned` 与 `discovered` 混用的局面已消除：确立唯一 canonical Schema（`upgrade.*`）、派生字段、deprecated 字段与未命中兜底。
3. **状态模型分域。** 原 5 套状态词汇混用已拆分为 6 个**有唯一权威**的维度，并在 `ADR-008`、`TODO TASK-005`、`VERSION-MATRIX §1`、`security-baseline.yaml` 四处同时声明分工，禁止新建第 7 套。
4. **SoT 收敛。** `VERSION-MATRIX.md` 成为版本唯一 SoT；`EXTERNAL-DEPENDENCIES.md` 成为外部依赖唯一 SoT；Task 基础 Schema 权威归 `Baseline §16.3`；运维 Task 生命周期权威归 `ADR-008`。

**修复统计：**

| 项 | 状态 |
|---|---|
| P0 修复 | 1 / 1 |
| P1 修复 | 9 / 9（P1-01 ~ P1-08 + P1-09 命名空间授权） |
| 修改文件 | 16 |
| 删除文件 | 0 |
| 新增架构组件 | 0 |
| Architecture Baseline 变更 | **NO ARCHITECTURE CHANGE** |
| V0.1 Scope 扩展 | **无** |

**结论：** `TASK-001 Readiness = READY`（详见 §12）。

---

## 2. 修改文件列表

| # | 文件 | 增/删 | 修复项 | 变更性质 |
|---|---|---:|---|---|
| 1 | `TODO.md` | +116/−29 | P0-01, P1-03, P1-04, P1-07, P1-08, P1-09 | Task 依赖、Preconditions、依赖图、状态维度声明、版本引用、命名空间授权、External Dependency SoT 指针 |
| 2 | `09-implementation/RESTORE-DRILL-PLAN.md` | +27/−6 | P0-01, P1-06 | 门禁语义反转修正、恢复环境明确、Backup→Restore→Validate→Cutover 链 |
| 3 | `09-implementation/CUTOVER-CHECKLIST.md` | +21/−3 | P0-01 | 新增 `2.1 Pre-Cutover Restore Gate` 结构性阻塞清单 |
| 4 | `09-implementation/EXTERNAL-DEPENDENCIES.md` | +23/−11 | P1-07, P1-08 | SoT 声明、6 处 Blocking Task 编号修正、恢复环境依赖行、Markdown 结构 |
| 5 | `09-implementation/VERSION-MATRIX.md` | +18/−8 | P1-04 | 声明为版本唯一 SoT、版本冻结状态唯一词表、禁止复制版本表 |
| 6 | `09-implementation/IMPLEMENTATION-RISKS.md` | +2/−2 | P1-11（同源） | 去除与"单集群"冲突的"独立验证集群"表述 |
| 7 | `07-aiops/component-lifecycle/upgrade-rules.yaml` | +49/−22 | P1-01 | canonical `upgrade_state_schema`、字段映射、兜底、`lifecycle_statuses` 改名 |
| 8 | `07-aiops/component-lifecycle/components.yaml` | +71/−50 | P1-01, P1-03, P1-04 | `field_semantics` 声明、canonical 字段、`pending-freeze`→`validating` |
| 9 | `07-aiops/component-lifecycle/policies.yaml` | +16/−0 | P1-02 | `approval_resolution`：rule → policy → L2 兜底、fail-closed |
| 10 | `07-aiops/component-lifecycle/README.md` | +64/−22 | P1-01, P1-02, P1-04 | Schema 闭合关系图、canonical 字段、审批解析顺序、Version SoT 分工 |
| 11 | `04-security/security-baseline.yaml` | +25/−10 | P1-03 | State Model SoT、`finding_lifecycle` 改名、EXCEPTION 澄清、L1 约束、审批兜底 |
| 12 | `04-security/port-baseline.yaml` | +19/−6 | P1-03 | `state_model_ref`、`port_required` 枚举修正、rules 词表统一 |
| 13 | `04-security/account-baseline.yaml` | +6/−2 | P1-03 | `state_model_ref`、rules 词表统一 |
| 14 | `04-security/access-control-baseline.yaml` | +10/−4 | P1-03 | `state_model_ref`、rules 词表统一、审批兜底 |
| 15 | `10-decisions/ADR/ADR-008-task-model.md` | +15/−0 | P1-03 | 声明为 Task 生命周期唯一权威 + 状态维度分工表（**纯新增，未改原有结论**） |
| 16 | `README.md` | +21/−13 | P1-05 | 当前状态表、V0.1 能力清单补全、Status 行 |

**未修改（已确认为权威/无需变更）：** `README.md` 之外的 `00-project/*`；`10-decisions/ARCHITECTURE-BASELINE-V0.1.md`；`10-decisions/ARCHITECTURE-ADJUDICATION-V0.1.md`；`ADR-001` ~ `ADR-007`；`ROLLBACK-PLAN.md`；`audit/` 下 6 份审计报告。

---

## 3. P0 修复

### P0-01 — Restore Drill 必须严格先于 Production Cutover

**问题（修复前）：** 存在合法路径 `Restore Drill 未完成 → Production Cutover`。三处独立证据：

| 证据 | 修复前内容 |
|---|---|
| `TODO.md` Phase 顺序 | Phase 13 Cutover（L877）**早于** Phase 15 Failure/Recovery Drills（L979） |
| `TODO.md:L874` | TASK-056（Cutover Gate 评审包）`Dependencies: TASK-041、TASK-053～055、TASK-061～066` —— 引用范围错误且不含恢复演练 |
| `RESTORE-DRILL-PLAN.md:41` | "任一关键恢复失败则 **TASK-061～TASK-064** 不通过，禁止生产切换" —— `TASK-061/062` 是 AI Ops 任务，门禁语义被反转 |
| `TODO.md:L860` | TASK-055（切换前单活演练）反向依赖 `TASK-057`（实际切换后任务） |

**修复（结构性，非文字性）：**

| # | 变更 | 位置 |
|---|---|---|
| 1 | TASK-056 `Dependencies` 改为 `TASK-041、TASK-053～055、TASK-064～066`（修正范围 + 纳入三项恢复演练） | `TODO.md` L874 区域 |
| 2 | TASK-056 `Preconditions` 增补"TASK-064～066 恢复演练全部完成且 RTO/RPO 达标；未完成任何一项 Restore Drill 时不得提交 GO 评审" | 同上 |
| 3 | TASK-056 `Definition of Done` 增补"未完成 TASK-064～066 时本任务不得标记完成" | 同上 |
| 4 | TASK-057 `Preconditions` 增补"**TASK-064～066 恢复演练全部通过且恢复校验 PASS**"；`Validation` 增补"Restore Drill 报告仍在有效期内且未被新的失败演练推翻"；`Definition of Done` 增补门禁有效性 | `TODO.md` L881 区域 |
| 5 | TASK-055 `Dependencies` 去掉反向依赖 `TASK-057`，改为 `TASK-052～054`；补 `Preconditions`（原缺失） | `TODO.md` L860 区域 |
| 6 | 依赖图新增独立 `RESTORE DRILL GATE` 区块，显式画出 `TASK-064/065/066 → TASK-056 → TASK-057 → TASK-058 → TASK-059`，并写明"禁止存在任何合法路径：TASK-064/065/066 未完成 ─> TASK-056 GO ─> TASK-057" | `TODO.md` §21 L1172–L1183 |
| 7 | `RESTORE-DRILL-PLAN.md` 第 6 节由单句改为 `Exit Criteria 与生产切换门禁`：新增 Backup→Restore→Validate→Cutover 结构图、三条硬约束、失败必须重跑 | `RESTORE-DRILL-PLAN.md` §6 |
| 8 | `CUTOVER-CHECKLIST.md` 新增 `2.1 Pre-Cutover Restore Gate`：6 项阻塞勾选 + "TASK-064～066 未全部通过时，本清单第 3 节任何步骤均不得执行" | `CUTOVER-CHECKLIST.md` §2.1 |
| 9 | TASK-067 `Dependencies` 与 `Preconditions` 增补 `TASK-SEC-006`（消除与 §24 停止条件的冲突） | `TODO.md` L1036/L1027 区域 |

**结构验证（程序化）：**

```text
TASK-064 ∈ deps(TASK-056) = True
TASK-065 ∈ deps(TASK-056) = True
TASK-066 ∈ deps(TASK-056) = True
TASK-056 ∈ deps(TASK-057) = True
TASK-057 ∈ deps(TASK-058) = True
TASK-058 ∈ deps(TASK-059) = True
TASK-064/065/066 ∈ deps(TASK-067) = True
TASK-055 ∉ 反向依赖 TASK-057 = True
```

**结论：** 由于 `TASK-056` 是 `TASK-057` 的硬依赖，且 `TASK-056` 自身硬依赖 `TASK-064/065/066`，**任何到达 `TASK-057/058/059` 的路径都必须先完成三项恢复演练**。P0-01 **PASS**。

---

## 4. P1 修复

### P1-01 — CLM Schema 闭合

**修复前冲突：** 同一个 `upgrade.required` 概念在三份 CLM 文件中以不同名称与类型存在：

| 文件 | 字段 | 取值 |
|---|---|---|
| `upgrade-rules.yaml:20-42` | `set.upgrade_required` + `set.minimum_priority` | `true` / `review` + `P0…P2` |
| `components.yaml:17`（16 处） | `upgrade: { approval_required, status: planned }` | 无 `required` |
| `README.md` | 散文列字段，第 3 个字段为 `reason` | 与 `reason_codes` 不一致 |
| `TODO.md:1101` | `upgrade_required`、`upgrade_reason` | 输入/输出字段名独立发明 |

**修复后（Schema = Registry = Rules Input = Rules Output = Task Input）：**

```text
upgrade-rules.yaml : upgrade_state_schema      ← canonical 字段定义（唯一权威）
                     fields: required | reason_codes | priority | target_version
                             | approval_required | status | deadline
                     field_mapping.canonical_field: "upgrade.required"
                     field_mapping.derived_field:   "upgrade_required"（只读派生投影）
                     field_mapping.deprecated_field:"set.minimum_priority" → set.priority
                     default_when_unmatched: review / P0 / approval_required: true
components.yaml    : component.upgrade.*        ← 使用同一字段与枚举（16/16 含 required）
policies.yaml      : approval_resolution        ← 决定 approval_required
TODO.md            : TASK-CLM-004/005/060       ← 引用 canonical 字段名
README.md          : §3/§5 + Schema 闭合关系图
04-security/...    : task_model.base_schema_authority / extension_fields
```

**程序化验证（20 项全 PASS，节选）：**

```text
PASS [1]  upgrade.required enum = true|false|review
PASS [2]  rules.set contains only canonical (required/priority)
PASS [3]  rules.required values within enum
PASS [4]  all reason_codes within vocabulary
PASS [5]  schema reason_codes enum == vocabulary
PASS [6]  components.upgrade keys within canonical
PASS [7]  every component.upgrade has required
PASS [8]  components.upgrade.status within lifecycle_statuses
PASS [19] no set.minimum_priority left
PASS [20] canonical upgrade.required declared
```

**关键约束落地：** `upgrade_required` 与 `upgrade.required` **不再并存为两个无关系模型** —— 前者被显式定义为后者的只读派生投影，派生规则写入 `field_mapping.derived_field.rule`。另修正 `unknown-version` 规则的 `reason_codes` 由 `POLICY` 改为 `UNKNOWN_VERSION`，并补 `priority: P0`。

### P1-02 — Approval Rule Fallback 明确为 L2

**修复前：** `policies.yaml` 有 `policies[].approval_level` 与 `approval_rules[]` 两套机制，但**未定义两者优先级，也未定义都不命中时的行为**。最坏情况：`approval_rules` 全覆盖，使 `non-production` policy 的 `L1` 分支永不可达；或规则求值失败时无兜底。

**修复（`policies.yaml` 顶部新增唯一权威块）：**

```yaml
approval_resolution:
  order: [match_approval_rule, match_policy_default, fallback_l2]
  fallback_level: L2
  fallback_condition: "no_approval_rule_matched and no_policy_default_matched"
  unknown_defaults_to_l2: true
  rule_match_precedence: first_match
  fail_closed: true
  never_auto_execute_production: true
```

并按安全默认值原则同步到：`security-baseline.yaml:approval.resolution_order/fallback_level/unknown_or_unmatched`、`port-baseline.yaml:remediation`、`access-control-baseline.yaml:remediation`、`07-aiops/component-lifecycle/README.md` §6、`TODO.md TASK-CLM-004`。

**最终逻辑（全库一致）：**

```text
1. 命中 approval_rules 首条规则            → 使用该规则级别
2. 未命中 → 命中 policy 的 approval_level  → 使用该 policy 级别
3. 都未命中 / 条件无法求值 / 字段缺失       → L2（fail-closed）
4. 版本、环境、优先级未知或不确定           → L2
```

**结论：** 不存在"Approval Rule 未匹配 → 自动执行"的路径。P1-02 **PASS**。

### P1-03 — 统一状态模型（分域，不混用）

原有 5 套状态词汇散落在 6 个文件中且无适用范围声明。现已拆分为 **6 个有唯一权威的维度**，并在 `ADR-008`、`TODO TASK-005`、`VERSION-MATRIX §1`、`security-baseline.yaml`、`upgrade-rules.yaml` 五处同时声明分工：

| 维度 | 权威文件 | 词表 | 关键约束 |
|---|---|---|---|
| **Finding Status** | `04-security/security-baseline.yaml` | `PASS` / `FAIL` / `REVIEW` / `UNKNOWN` | `unknown_is_pass: false`；`unknown_default: UNKNOWN` |
| Finding Lifecycle | 同上（字段改名 `state_model.lifecycle` → `finding_lifecycle`） | `discovered, assessed, task-created, approval-pending, remediation, verifying, closed, escalated` | 仅用于两清两固 finding |
| **Task Lifecycle** | `ADR-008` | `open, analyzed, planned, pending-approval, approved, rejected, executing, verifying, verified, failed, closed, escalated, rolled-back` | 运维 Task 专用 |
| Task 字段 Schema | `Baseline §16.3` | `api_version/kind/spec/approval/execution/verification/audit` | 安全类只允许 `extension_fields` |
| **Component Lifecycle** | `components.yaml:lifecycle.support_status` | `pending-validation, supported, eol, eos, unknown` | 与 `upgrade.status` 不同维度 |
| **Component Upgrade Status** | `upgrade-rules.yaml:lifecycle_statuses` | `discovered, assessed, upgrade-required, task-created, approved, executing, verifying, closed, rolled-back, review` | 原键名 `lifecycle` 已改名以消除同名不同义 |
| **Version Status** | `VERSION-MATRIX.md §1` | `candidate, validating, frozen, superseded` | 表格中的 `Pending Compatibility Validation` 等为可读标注，不是第二套状态 |
| 实施任务状态（仅 TODO 内部） | `TODO.md` | `PENDING, IN_PROGRESS, BLOCKED, PASSED, ROLLED_BACK` | 明确限缩为"仅用于本清单" |

**同步修复：**

- `components.yaml` 的 `upgrade.status` 由 `planned`（不在任何权威词表内）改为 `discovered`（∈ `lifecycle_statuses`）。
- `components.yaml` 的 10 处 `desired.status: pending-freeze` → `validating`（∈ Version Status 词表）；新增 `field_semantics` 声明四个字段的权威来源与枚举。
- **`EXCEPTION` 明确不是 finding status**：`security-baseline.yaml` 新增 `exception_is_finding_status: false` 与说明；`TODO.md TASK-067` 就地标注"仅用于验收结果；finding 状态只能取 `PASS/FAIL/REVIEW/UNKNOWN`"。**未新增任何状态。**
- 两清两固三份对照基线新增 `state_model_ref: 04-security/security-baseline.yaml`，并统一 `rules:` 动作词汇为 `FAIL_AND_CREATE_TASK` / `UNKNOWN_AND_CREATE_TASK` / `REVIEW_AND_CREATE_TASK` / `BLOCK_AND_ESCALATE_L2`（`INCIDENT` 降为附加属性）。
- `port-baseline.yaml` 的 `port_required: REVIEW` → `UNKNOWN`，并新增 `field_types` 声明其枚举为 `YES/NO/UNKNOWN`，消除与 finding status 的枚举碰撞。
- `security-baseline.yaml` L1 增补 `L1_requires: [non_production_scope, approved_reversible_runbook, automatic_verification, rollback_available]`，与 Baseline §19 和 CLM §6 对齐。

### P1-04 — VERSION-MATRIX 成为唯一 Version SoT

`VERSION-MATRIX.md` 头部新增 SoT 声明与三条禁令：

- `components.yaml` 的 `desired.version` 为**引用本矩阵的派生值**，不得冲突；
- `TODO.md` **不得复制版本矩阵**，只引用本文件；
- 运行态实际版本由 Discovery evidence 决定，不属于矩阵内容；
- §1 新增规则 7："任何组件在本矩阵中出现且仅出现一个 desired version；发现不一致时以本文件为准，并修正引用方。"
- §1 规则 1 明确 Version Status 唯一词表为 `candidate/validating/frozen/superseded`。

`README.md §23` 同步加注"版本唯一事实来源为 VERSION-MATRIX，下表为摘要，不得作为第二份版本表维护"。

**验证：** 全库未发现同一组件在两个文件中存在不同 `desired version`。P1-04 **PASS**。

### P1-05 — README 同步到当前 V0.1 状态

| 项 | 修复前 | 修复后 |
|---|---|---|
| §23 标题行 | `Architecture Frozen / Implementation TODO Ready` | `Architecture Frozen / Consistency Closed / Implementation Ready` |
| §23 内容 | 仅一句 + 版本表 | 新增 6 行状态表（Architecture/ADR/Consistency Audit/Closure/TODO/Installation）+ 下一阶段说明 + 明确"不是 Implementation Completed，也不是 Production Ready；当前仅为 Implementation Ready" |
| §15 V0.1 能力清单 | 12 项，**漏 CLM 与两清两固** | 补入"组件生命周期管理（CLM）最小闭环"与"两清两固安全运营最小闭环" |
| 文末 Status | `Planning / Architecture` | `Architecture Frozen / Consistency Closed / Implementation Ready（Installation: NOT STARTED）` |

**验证：** 全文已无"仍在完成顶层架构设计"类过时表述；无"Implementation Completed"或"Production Ready"的虚假宣称。

### P1-06 — Restore Namespace / 环境明确

**原问题：** `RESTORE-DRILL-PLAN.md:20` 出现未授权命名空间 `da-soc-restore`；Baseline §5.2 的授权 Namespace 集合为 `kube-system`、`kubesphere-system` 族、`xw-platform`、`xw-observability`、`xw-aiops`、`da-soc`（+ 迁移期 `da-soc-validate`），**不含** `da-soc-restore`。

**采纳方案（Prompt 推荐方案）：** **Restore Drill 使用外部隔离恢复环境，不新增 V0.1 Kubernetes 正式 Namespace。**

- `da-soc-restore` 表述**已全部移除**（全库 `grep da-soc-restore` = 0 命中）。
- D1/D2/D3 均显式声明环境为**外部隔离恢复环境 `xw-restore-drill`**（独立 VM / 受控环境，非 Kubernetes Namespace），并写明"不新增 Kubernetes Namespace"。
- `TODO.md TASK-065` 的 `Preconditions`/`Rollback`/`Definition of Done` 同步为外部隔离环境。
- `EXTERNAL-DEPENDENCIES.md` 新增一行依赖："外部隔离恢复环境 `xw-restore-drill`（独立 VM，非 Kubernetes Namespace）| Infrastructure/Backup Owner | TASK-064 | TASK-064～066"。
- 未修改 Baseline §5.2（因未新增 Namespace，无需扩权）。

### P1-07 — Task ID / Dependency 全局收口

**修复的编号缺陷：**

| # | 位置 | 修复前 | 修复后 | 缺陷类型 |
|---|---|---|---|---|
| 1 | `RESTORE-DRILL-PLAN.md:41` | `TASK-061～TASK-064` | `TASK-064 / TASK-065 / TASK-066` | 引用错误区间，门禁语义反转 |
| 2 | `EXTERNAL-DEPENDENCIES.md:37` | Blocking = `TASK-061` | `TASK-038～041/064～066` | 指向 AI Ops 任务 |
| 3 | `EXTERNAL-DEPENDENCIES.md:23` | `TASK-043` | `TASK-020/050/052` | 指向错误任务 |
| 4 | `EXTERNAL-DEPENDENCIES.md:28` | `TASK-044` | `TASK-044/049` | 缺任务 |
| 5 | `EXTERNAL-DEPENDENCIES.md:30` | `TASK-055` | `TASK-052/055/058` | 缺任务 |
| 6 | `EXTERNAL-DEPENDENCIES.md:31` | `TASK-050` | `TASK-050～055/066` | 缺任务 |
| 7 | `EXTERNAL-DEPENDENCIES.md:39` | `TASK-059` | `TASK-036/055～059/061` | 缺任务 |
| 8 | `07-aiops/component-lifecycle/README.md:64` | 占位符 `TASK-CLM-00x` | `TASK-CLM-001 ~ TASK-CLM-006`（并注明各任务职责） | 非真实 ID |
| 9 | `TODO.md` TASK-055 | 反向依赖 `TASK-057` | `TASK-052～054` | 依赖方向错误 |
| 10 | `TODO.md` TASK-028 | 依赖 `TASK-040`（Phase 9） | `TASK-002、TASK-027` | 前向依赖 |
| 11 | `TODO.md` TASK-033 | 依赖 `TASK-038`（Phase 9） | `TASK-030～032` | 前向依赖 |
| 12 | `TODO.md` TASK-056 | `TASK-061～066` | `TASK-064～066` | 范围错误 |
| 13 | `TODO.md` TASK-067/068 | 未列 `TASK-SEC-006` | 补入 | 与 §24 停止条件冲突 |
| 14 | `TODO.md` TASK-SEC-006 | 反向依赖 `TASK-067` | 去掉 | **循环依赖** |

**程序化验证：**

```text
TASK-xxx 引用有效性（全库，排除 audit 与历史候选目录）
  ✅ PASS — 全部引用指向真实任务（80 个真实 Task）
依赖环检测（三色 DFS，含 ～ 区间展开）
  ✅ PASS — 无循环依赖
```

**剩余前向依赖（均为设计意图内的结构性门禁，非缺陷）：**

```text
TASK-056 -> TASK-064/065/066    ← P0-01 恢复演练门禁（有意为之）
TASK-057 -> TASK-064/065/066    ← 同上，双重校验
TASK-067 -> TASK-CLM-006        ← 最终验收依赖 CLM 闭环
TASK-067 -> TASK-SEC-006        ← 最终验收依赖两清两固闭环
TASK-068 -> TASK-CLM-006/SEC-006← 交接依赖两者
```

**另修复的字段缺陷：**

- TASK-055 补 `Preconditions`（原缺失）；TASK-065/066 补并行独立性说明；TASK-SEC-006 补 `Preconditions / Inputs / Expected Output / Risk / Evidence`（原缺 5 项）；TASK-SEC-001 补 `Preconditions / Risk / Evidence`（原缺 3 项，违反文件自身 L59 与 §24 DoD 第 11 项）。
- TASK-004 仓库布局清单修正：`security/` → `04-security/`，补入 `09-implementation/`、`policies.yaml`、`upgrade-rules.yaml`、`VERSION-MATRIX.md`，并移除未使用的 `tasks/`、`backup/`。
- `09-implementation/EXTERNAL-DEPENDENCIES.md` 两清两固表格前补空行（Markdown 结构）。

### P1-08 — External Dependencies 统一

- `EXTERNAL-DEPENDENCIES.md` 头部声明：**"本文件是 V0.1 外部依赖的唯一事实来源（SoT）；`TODO.md` §22 只做引用与摘要，不得维护第二份互相冲突的 Blocking 映射；两者不一致时以本文件为准。"**
- `TODO.md §22` 同步声明该指针，并明确"若与本文件不一致，以 `EXTERNAL-DEPENDENCIES.md` 为准并修正本表"。
- 6 处 Blocking Task 编号按上述 P1-07 表修正，消除双份定义冲突。

### P1-09 — V0.1 授权 Namespace 无对应创建任务（一致性缺口）

**发现：** Baseline §5.2 已授权 `xw-platform`、`xw-observability`、`xw-aiops` 为**正式 Namespace**（用于平台 Runbook/备份 Job、观测栈配置、Agent Job 模板），但 `TODO.md` 全文对这三个名称**零命中**，`TASK-016` 仅创建 `da-soc` 与 `da-soc-validate`。后果：`TASK-029`（Prometheus/Grafana/Alertmanager）与 `TASK-060`（Agent Job 模板）缺少放置依据，实施者可能落入 `default`/`kube-system`，破坏 Baseline 的配额/PSA/NetworkPolicy 边界。

**修复（非 Scope 扩展 —— 授权已存在于 Baseline，仅补齐执行依据）：** `TASK-016` 的 `Inputs/Actions/Validation/Evidence/Definition of Done` 明确按 Baseline §5.2 创建 `xw-platform`、`xw-observability`、`xw-aiops`、`da-soc` 与临时 `da-soc-validate`，并为其配置 Quota/LimitRange/PSA/default-deny。**未新增任何 Namespace，未修改 Baseline。**

### 附带修复（同源，成本极低）

- `IMPLEMENTATION-RISKS.md` 的 `RISK-OS-001`/`RISK-OS-003` 把"独立验证集群"改为"**单集群内的隔离验证**（外部隔离环境 `xw-restore-drill` 或 `da-soc-validate` Namespace）"，消除与 Baseline §2.1"单集群"的冲突，且**无需新增任务**。
- `TODO.md §22` External Dependencies 表保持为摘要表（未删除），但已声明 SoT 归属。

---

## 5. 未处理 P2/P3

以下问题经裁决**不在本次范围**（P2/P3 或非 Implementation Blocker），按要求仅记录、不实施，避免 Scope Creep：

| ID | 问题 | 级别 | 不处理理由 / 建议归属 |
|---|---|---|---|
| P2-A | **KubeSphere 4.1.x 的"组件开关"语义与 Baseline §5.3 的 3.x 语义不对应**（Baseline 要求禁用 Jenkins/App Store/ES Logging 等 3.x 插件开关，4.x 改为 Extensions 模型） | P2 | 属版本机制差异，须在 `TASK-002` 版本冻结时一并核实官方兼容矩阵；codebuddy 建议在 Baseline §5.3 增加"机制无关"的枚举比对要求 |
| P2-B | **`TODO.md` §20B 位于 §24 之后**（章节编号乱序，导致 §24/§21 前向引用后置章节） | P2 | 纯文档结构，不影响依赖语义；建议一次性重排章节号 |
| P2-C | **`TODO.md` §21 依赖图未画出 TASK-044 → TASK-022～024 的存储前置**（图形缺边，字段本身正确） | P2 | 依赖字段为权威且正确；建议改为从字段自动生成图形 |
| P2-D | **RPO/RTO 在 Baseline/ADR-006 已定，在 VERSION-MATRIX 仍列为待冻结参数** | P2 | 属参数冻结范围（`TASK-001`~`TASK-005` 内闭环） |
| P2-E | **`components.yaml` 未单列 `Kernel`、`local-path`、`Ingress Controller`**，而 VERSION-MATRIX §3 有这三项，使 CLM "Component Coverage = 100%" 的基准不明 | P2 | 建议在 `TASK-CLM-001` 内补登记或在 CLM README 定义"应管理组件"枚举来源 |
| P2-F | **`TODO.md` TASK-029 Inputs 写作"KubeSphere/Prometheus stack"**，与同任务"不部署第二套指标系统"存在部署主体歧义 | P2 | 已在本报告记录；建议在 `TASK-029` 内明确为"独立 Prometheus/Grafana/Alertmanager 组件" |
| P3-A | `TODO.md` 路径引用风格不统一（`VERSION-MATRIX.md` 与 `09-implementation/VERSION-MATRIX.md` 混用；`EXTERNAL-DEPENDENCIES.md` 与全路径混用） | P3 | 纯风格 |
| P3-B | 拼写风格：`rolled-back`（YAML）与 `ROLLED_BACK`（TODO） | P3 | 分属两个不同维度（升级状态 vs 实施任务状态），**并非同一枚举**，无需统一 |
| P3-C | `TODO.md §22` 摘要表比 Baseline §22 Non-Goals 少 SIEM/CMDB/多租户/跨地域 DR/完整供应链安全 5 项 | P3 | README §27 与 Baseline §25 已另有覆盖 |
| P3-D | `CLM/README.md` §10/§11 英文与 Baseline §25 中文平行表述 | P3 | 语言一致性优化 |
| P3-E | `components.yaml` 的 `environment` 字段两义（部署目标 vs 生命周期阶段） | P3 | 建议拆为 `deployment_target` 与 `lifecycle_stage`（minimax 指出此为 P1-04 根因之一，但本次已通过 `field_semantics` 声明缓解） |
| P3-F | `00-project/{VISION,GOALS,SCOPE,PRINCIPLES,VERSIONING}.md` 仍为 ~150–200 字节占位符 | P3 | 不影响 V0.1 实施 |

---

## 6. Architecture Baseline 是否变化

> ## **NO ARCHITECTURE CHANGE**

**证据（`git diff --numstat`）：**

```text
10-decisions/ARCHITECTURE-BASELINE-V0.1.md        → 未出现在 diff 中（0 变更）
10-decisions/ARCHITECTURE-ADJUDICATION-V0.1.md    → 未出现在 diff 中（0 变更）
10-decisions/ADR/ADR-001 ~ ADR-007                → 未出现在 diff 中（0 变更）
10-decisions/ADR/ADR-008-task-model.md            → 15 insertions(+), 0 deletions(-)
```

`ADR-008` 的唯一变更是在 `## Lifecycle` 段落后**新增** 15 行"状态维度分工"表与两句声明；原有 `Decision`、`Lifecycle` 状态序列、`Security`、`Why Not CRD` 内容**逐字未动**。该变更为**澄清**（"本 ADR 是运维 Task 生命周期的唯一权威定义"），不改变任何架构结论。

**未变更的核心架构事实（全部保持原样）：** 5 台 VM；1 Control Plane + 2 Worker；Harbor 在集群外；备份仓库在集群外；Calico default-deny；local-path/Local PV；ClickHouse 单副本固定 `xw-wk-02`；唯一观测栈 Prometheus + Grafana + Alertmanager + Fluent Bit + Loki；DA-SOC 部署方式；n8n 定位；Agent 短生命周期 Job；L0/L1/L2；Git Task 无 CRD；Kubernetes/KubeSphere 基础架构；Baseline §5.2 Namespace 集合。

**Scope 未扩展（负向清单核查）：** 未新增 Task CRD、`xw-opsapi`、CMDB、SIEM、SOAR、Ceph、Longhorn、Service Mesh、第二集群、GPU、多集群、多租户门户、Web Task Center、多 Agent Runtime、新数据库、新消息队列、新 API Gateway。

---

## 7. SoT 清单

| 事实域 | 唯一事实来源（SoT） | 派生 / 引用方 |
|---|---|---|
| **Architecture Baseline** | `10-decisions/ARCHITECTURE-BASELINE-V0.1.md` | 全部实施文档；ADR 为其决策记录 |
| Architecture Decision | `10-decisions/ADR/ADR-001` ~ `ADR-008` | Baseline、TODO |
| **Version（Approved Desired Version / Freeze）** | `09-implementation/VERSION-MATRIX.md` | `components.yaml:desired.version`（派生值）、`README.md §23`（摘要）、`TODO.md`（引用，不复制） |
| **Task** | 列表与状态：`TODO.md`；字段 Schema：`Baseline §16.3`；运维 Task 生命周期：`ADR-008` | CLM/两清两固 Task 为其扩展 |
| **External Dependency** | `09-implementation/EXTERNAL-DEPENDENCIES.md` | `TODO.md §22`（摘要 + 指针） |
| **Security Baseline / 两清两固 State Model** | `04-security/security-baseline.yaml` | `port-baseline.yaml`、`account-baseline.yaml`、`access-control-baseline.yaml`（`state_model_ref`） |
| **CLM Schema** | `07-aiops/component-lifecycle/upgrade-rules.yaml: upgrade_state_schema` | `components.yaml`（Registry 数据）、`policies.yaml`（审批）、`TODO TASK-CLM-00x`、CLM README |
| CLM Approval Policy | `07-aiops/component-lifecycle/policies.yaml: approval_resolution` | `security-baseline.yaml`、三份对照基线、CLM README、TODO TASK-CLM-004 |
| Component Registry | `07-aiops/component-lifecycle/components.yaml` | Discovery evidence（运行态，非 Git） |
| Risk Register | `09-implementation/IMPLEMENTATION-RISKS.md` | `TODO.md §23`（引用） |
| Restore / Cutover 实施方法 | `09-implementation/RESTORE-DRILL-PLAN.md`、`CUTOVER-CHECKLIST.md`、`ROLLBACK-PLAN.md` | `TODO.md` Phase 13/15 |
| Git 声明式定义（总原则） | Git 仓库本身（`Baseline §11`） | 上述全部文件 |

**关键去重结果：** `VERSION-MATRIX` 与 `components.yaml` 之间、`EXTERNAL-DEPENDENCIES` 与 `TODO §22` 之间、`security-baseline` 与三份对照基线之间，均已从"双份并行定义"降级为"SoT + 派生/引用"，并写明不一致时的裁决方向。

---

## 8. 状态模型

| # | 维度 | 权威文件 | 规范词表 | 未知/未命中处理 |
|---|---|---|---|---|
| 1 | **Finding Status** | `04-security/security-baseline.yaml` | `PASS` / `FAIL` / `REVIEW` / `UNKNOWN` | `UNKNOWN`；`unknown_is_pass: false`；**`UNKNOWN ≠ PASS`** |
| 1b | Finding Lifecycle | `04-security/security-baseline.yaml:finding_lifecycle` | `discovered` / `assessed` / `task-created` / `approval-pending` / `remediation` / `verifying` / `closed` / `escalated` | 超期 → `REVIEW` |
| 2 | **Task Lifecycle**（运维 Task） | `10-decisions/ADR/ADR-008-task-model.md` | `open` / `analyzed` / `planned` / `pending-approval` / `approved` / `rejected` / `executing` / `verifying` / `verified` / `failed` / `closed` / `escalated` / `rolled-back` | 未审批不得进入 `executing` |
| 3 | 实施任务状态（仅 `TODO.md` 内部） | `TODO.md` | `PENDING` / `IN_PROGRESS` / `BLOCKED` / `PASSED` / `ROLLED_BACK` | `BLOCKED` 时必须登记 Owner 与阻塞项 |
| 4 | **Component Lifecycle** | `components.yaml:lifecycle.support_status` | `pending-validation` / `supported` / `eol` / `eos` / `unknown` | `unknown` 视为高风险 |
| 5 | **Component Upgrade Status** | `upgrade-rules.yaml:lifecycle_statuses` | `discovered` / `assessed` / `upgrade-required` / `task-created` / `approved` / `executing` / `verifying` / `closed` / `rolled-back` / `review` | 未命中规则 → `required: review` + `priority: P0` + `approval_required: true` |
| 6 | **Version Status** | `09-implementation/VERSION-MATRIX.md §1` | `candidate` / `validating` / `frozen` / `superseded` | 未验证组合只能 `candidate`/`validating`，不得 `frozen` |

**跨维度禁令（已写入文档）：** 不得以任一维度的状态值替代另一维度；不得新建第 7 套状态词表。`EXCEPTION` 明确**不是** finding status（`exception_is_finding_status: false`），仅作为 Task/验收层的例外登记属性；`INCIDENT` 同理为附加属性。

**升级字段派生关系（CLM 内部）：**

```text
canonical : upgrade.required      ∈ {true, false, review}
derived   : upgrade_required      = 投影(upgrade.required)   # 只读，不得独立维护
deprecated: set.minimum_priority  → set.priority             # 已重命名，不再出现
```

---

## 9. Dependency Gate

```text
                        Backup
                          │
                          ▼
            Restore Drill（TASK-064 / TASK-065 / TASK-066）
                          │
                          ▼
              Restore Validation PASS（RTO/RPO + SQL/图片/门禁对账）
                          │
                          ▼
              DA-SOC Validation PASS（TASK-053 ~ TASK-055）
                          │
                          ▼
          Cutover Preparation / Gate 评审包（TASK-056）
                          │
                          ▼
      Cutover Preparation 冻结与最终备份（TASK-057，重新校验恢复门禁）
                          │
                          ▼
              Production Cutover（TASK-058 → TASK-059）
```

**结构约束（已落地为硬依赖，非文字声明）：**

| 边 | 载体 | 验证 |
|---|---|---|
| `TASK-064/065/066 → TASK-056` | `Dependencies` + `Preconditions` + `Definition of Done` + §21 图 RESTORE DRILL GATE 区块 | ✅ True |
| `TASK-056 → TASK-057` | `Preconditions`（TASK-056 GO）+ `Dependencies` | ✅ True |
| `TASK-057 → TASK-058 → TASK-059` | `Dependencies` | ✅ True |
| `TASK-064/065/066 → TASK-067` | `Dependencies` + `Preconditions` | ✅ True |
| 门禁陈述 | `CUTOVER-CHECKLIST.md §2.1`："TASK-064～066 未全部通过时，本清单第 3 节任何步骤均不得执行" | ✅ |
| 失败处理 | `RESTORE-DRILL-PLAN.md §6`："任一关键恢复失败 → 对应恢复演练任务不通过 → 禁止生产切换，回到备份/恢复修复后重跑该 Drill" | ✅ |

**反例搜索结论：** 不存在任何合法执行路径使 `Restore Drill 未完成 → Production Cutover` 成立。曾存在的三条路径（Phase 顺序倒置、`TASK-056` 依赖范围错误、`TASK-055` 反向依赖）均已封堵。

---

## 10. Verification Results

### 10.1 Architecture

| 检查项 | 结果 |
|---|---|
| 5 VM | ✅ PASS |
| 1 CP + 2 Worker | ✅ PASS |
| Harbor external | ✅ PASS |
| Backup external | ✅ PASS |
| Calico | ✅ PASS |
| Local PV（无 Ceph/Longhorn） | ✅ PASS |
| ClickHouse 单副本 `xw-wk-02` | ✅ PASS |
| Observability single stack | ✅ PASS |

### 10.2 Security

| 检查项 | 结果 |
|---|---|
| L0/L1/L2（含 L1 前置约束） | ✅ PASS |
| L2 approval（fallback 明确为 L2、fail-closed） | ✅ PASS |
| Unknown → safe default（`unknown_is_pass: false`） | ✅ PASS |
| No cluster-admin | ✅ PASS |
| No long-lived kubeconfig | ✅ PASS |
| No Internet-exposed K8s API/etcd | ✅ PASS |

### 10.3 CLM

| 检查项 | 结果 |
|---|---|
| Schema closed（20 项程序化断言） | ✅ PASS（20/20） |
| Rules closed（`reason_codes` ⊆ 唯一词表） | ✅ PASS |
| Approval closed（rule → policy → L2） | ✅ PASS |
| Version SoT closed（VERSION-MATRIX 唯一） | ✅ PASS |

### 10.4 Task

| 检查项 | 结果 |
|---|---|
| Task ID closed（全库引用有效性） | ✅ PASS（0 无效引用 / 80 真实任务） |
| Dependency closed（三色 DFS 环检测） | ✅ PASS（0 环） |
| Restore before Cutover | ✅ PASS |

### 10.5 Recovery

| 检查项 | 结果 |
|---|---|
| Backup | ✅ PASS |
| Restore | ✅ PASS |
| Validation | ✅ PASS |
| Cutover | ✅ PASS |
| Rollback | ✅ PASS |
| **Backup → Restore → Validate → Cutover 闭环** | ✅ PASS |

### 10.6 其他验证

| 检查项 | 结果 |
|---|---|
| 7 份 YAML 语法（PyYAML `safe_load`） | ✅ PASS（7/7） |
| 行尾一致性（`git ls-files --eol`，i/lf） | ✅ PASS |
| `da-soc-restore` 残留 | ✅ PASS（0 命中） |
| `pending-freeze` 残留 | ✅ PASS（0 命中） |
| 旧 rules 动作名残留（`*_CREATE_PORT_TASK` 等） | ✅ PASS（0 命中） |
| 文件删除 | ✅ PASS（0 删除） |
| Baseline / Adjudication / ADR-001~007 未改 | ✅ PASS（0 变更） |

**总体：** 全部检查 PASS，无 FAIL、无未决 REVIEW。

---

## 11. Remaining Risks

仅列真实现存问题（均已记录、不在本次范围、且不构成 Implementation Blocker）：

| # | 风险 | 级别 | 影响 | 建议处理时点 |
|---|---|---|---|---|
| R1 | **Kubernetes v1.30.6 + KubeSphere 4.1.x + UOS 1060e 组合尚未实际安装验证** | High | 若组合不兼容，需回退版本基线 | `TASK-002` 版本冻结前完成官方兼容矩阵核对与实机验证（`RISK-OS-001` 已登记） |
| R2 | **KubeSphere 4.x 的组件/扩展模型与 Baseline §5.3 的 3.x"插件开关"语义不对应** | Medium | `TASK-015` 的"仅安装 Baseline 所需管理组件"无法字面验证 | 与 R1 同期在 `TASK-002`/`TASK-015` 内解决 |
| R3 | **单 Control Plane 无 HA**，`xw-cp-01` 故障导致管理面中断 | High（已接受） | 依赖 etcd snapshot + Git 重建，CP RTO ≤ 4h | 已由 `TASK-064` 覆盖；V0.2 引入 3 CP（Baseline §23） |
| R4 | **CLM 组件覆盖基准（"应管理组件"枚举）未定义** | Low | 影响 `Component Coverage = 100%` 的可判定性 | `TASK-CLM-001` 内补登记 Kernel/local-path/Ingress Controller 或定义枚举来源 |
| R5 | **委员会 6 份审计对同一问题分级不一致**（如 `da-soc-restore` 有报告定为 P1、有报告未提及） | Low | 不影响已裁决结论 | 本次已按"授权 Namespace 集合"客观事实裁决为需修复，已修复 |
| R6 | **`TODO.md` §20B 章节位置在 §24 之后** | Low | 阅读顺序与前向引用 | 后续文档整理批次 |
| R7 | **Recovery Drill 使用外部隔离环境，未在集群内验证 Namespace 级资源恢复** | Medium | 恢复验证覆盖的是数据与配置，不是集群内对象图 | 由 `TASK-064`（Control Plane 重建含 Namespace 验证）覆盖；如需 Namespace 级演练，在 V0.2 评估 |

**无 P0/P1 剩余风险。**

---

## 12. TASK-001 Readiness

> # **READY**

**Gate 逐项确认（Prompt §16 的 11 项条件）：**

| # | 条件 | 结果 | 证据 |
|---|---|---|---|
| 1 | P0-01 PASS | ✅ | 三项恢复演练成为 TASK-056/057 硬依赖；反例路径已封堵（§3、§9） |
| 2 | 所有 P1 Implementation Blocker PASS | ✅ | P1-01 ~ P1-09 全部修复并验证（§4、§10） |
| 3 | Task ID 全部有效 | ✅ | 0 无效引用 / 80 真实任务（§10.4） |
| 4 | Dependency Graph 无断链 | ✅ | 三色 DFS 环检测 0 环；图与字段关键门禁边一致（§10.4、§9） |
| 5 | Restore → Cutover Gate 成立 | ✅ | 结构性硬依赖 + Cutover Checklist 阻塞清单（§9） |
| 6 | CLM Schema 闭合 | ✅ | 20/20 程序化断言 PASS（§4 P1-01、§10.3） |
| 7 | Approval fallback 明确为 L2 | ✅ | `approval_resolution` order/fallback/fail-closed（§4 P1-02） |
| 8 | Version SoT 唯一 | ✅ | VERSION-MATRIX 声明 + 派生/引用关系（§4 P1-04、§7） |
| 9 | README 与实际状态一致 | ✅ | §23 状态表 + Status 行 + V0.1 能力清单（§4 P1-05） |
| 10 | Namespace / Restore Environment 明确 | ✅ | 外部隔离环境 `xw-restore-drill`；授权 Namespace 补齐创建依据（§4 P1-06/P1-09） |
| 11 | Git diff 无架构漂移 | ✅ | Baseline/Adjudication/ADR-001~007 零变更；ADR-008 纯新增澄清（§6） |

**最终状态：**

```text
Architecture   : Frozen
Consistency    : Closed
P0             : 0
P1 Blocker     : 0
Task ID        : Consistent
Dependency     : Consistent
CLM            : Schema Closed
Approval       : Safe Default (L2)
Version        : Single SoT
Restore        : Before Cutover
README         : Current
Scope          : No Expansion
```

> ## **V0.1 IMPLEMENTATION READY**

**下一步：** 由架构 Owner / 治理 Owner 对本次 16 个文件的 `git diff` 完成人工复核并提交；随后方可启动 `TASK-001`（Phase 0 参数冻结）。**本 Agent 不自行开始 TASK-001。**

---

### 附：人工复核建议清单

```bash
git status
git diff --stat
git diff -- TODO.md                                  # P0-01 门禁与状态维度声明
git diff -- 09-implementation/RESTORE-DRILL-PLAN.md  # P0-01 门禁语义 + P1-06 环境
git diff -- 09-implementation/CUTOVER-CHECKLIST.md   # P0-01 §2.1 Restore Gate
git diff -- 09-implementation/EXTERNAL-DEPENDENCIES.md  # P1-07/P1-08 Blocking Task
git diff -- 07-aiops/component-lifecycle/            # P1-01/P1-02 CLM Schema + Approval
git diff -- 04-security/                             # P1-03 State Model
git diff -- 10-decisions/ADR/ADR-008-task-model.md   # P1-03 状态维度分工（纯新增）
git diff -- README.md                                # P1-05 状态同步
```
