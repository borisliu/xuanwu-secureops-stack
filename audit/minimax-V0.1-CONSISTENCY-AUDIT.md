# Xuanwu SecureOps Stack V0.1
# Consistency Audit — minimax

> 审计对象：`main` 分支 `3151bb3`（HEAD）
> 审计日期：2026-10-08
> 审计类型：READ ONLY AUDIT（未修改任何项目文件）
> 审计范围：C01–C09，重点核查最近两次补丁 `695c6c7`（CLM）与 `3151bb3`（两清两固）

**方法说明（独立性声明）**：本次审计只读取仓库中已生效的 Source of Truth 文件
（README、Baseline、Adjudication、ADR、VERSION-MATRIX、CLM、`04-security/`、`09-implementation/`、TODO）。
审计过程中**未阅读** `09-implementation/00-architecture-review/` 下的候选方案，
也**未阅读** `audit/` 下其他 Agent 的审计报告（grep 结果中曾偶然出现片段，已刻意不取用其结论），
以保证本报告为独立判断。所有结论均给出文件与行号证据。

---

## 1. Executive Summary

**总体判断：BLOCKED**

结论一句话：

> **架构本身是自洽且高质量的，CLM 与两清两固也没有引入任何被否决的大型组件；
> 但这两块补丁在"状态模型 / Source of Truth / 版本词表"三个层面各自新建了一套并行定义，
> 并且最新的 TODO 把生产切换排在了恢复演练之前——后者直接违反 Baseline 的硬门禁。**

发现统计：

- **P0: 1**
- **P1: 11**
- **P2: 10**
- **P3: 8**

| 检查维度 | 判断 |
|---|---|
| C01 项目目标一致性 | 基本一致；README §23/§24 状态与顺序过时 |
| C02 架构一致性 | **一致**（拓扑/组件/DA-SOC 承载/单一观测栈无冲突） |
| C03 版本一致性 | 版本**数值**一致；**状态词表**四套混用 |
| C04 TODO 一致性 | **P0**：切换与恢复演练顺序倒置；多处任务编号引用错误 |
| C05 Source of Truth | **P1**：CLM 期望版本、Task schema、cadence、依赖映射各自双份 |
| C06 CLM 一致性 | **P1**：CLM 三文件之间 schema 不自洽 |
| C07 两清两固一致性 | 状态模型与 L0/L1/L2 语义一致；端口基线漏登记、依赖图缺口 |
| C08 V0.1 Scope / HA | 未膨胀为 SIEM/SOAR/CMDB；但风险登记要求"独立验证集群" |
| C09 人工审批 / Agent 权限 | **无 L2 绕过路径**；但 CLM 审批缺省级别未定义 |

**必须强调的三点正面结论**：

1. **架构主体没有被污染。** 5 VM / 1 CP + 2 Worker / Calico / 外置 Harbor / local-path /
   单一观测栈 / 短生命周期 Job Agent / Git Task，在 Baseline、Adjudication、8 份 ADR、
   VERSION-MATRIX、TODO §2 中**逐项一致**，无一处漂移。
2. **CLM 与两清两固遵守了"复用现有平台能力"的原则。** 没有引入 SIEM、SOAR、CMDB、
   新的数据库、第二个 Task Center、第二套监控或常驻高权限 Agent。
   两块补丁的所有动作都落在既有 Prometheus / Grafana / Alertmanager / Git / 短生命周期 Job 上。
3. **L0/L1/L2 的语义在 6 个不同文件中保持一致，且找不到绕过 L2 的路径。**
   `access-control-baseline.yaml:66` 甚至显式定义了 `agent_escalation: BLOCK_AND_ESCALATE_TO_L2`。

---

## 2. Architecture Consistency

### 2.1 结论：一致（No-Issue）

逐项核对 Baseline 声明与全部引用方：

| 项目 | Baseline 出处 | Adjudication | ADR | TODO §2 | 结果 |
|---|---|---|---|---|---|
| 单集群 / 5 VM | `:11`, `:21-27` | `:18`, `:225-231` | — | `:41` | 一致 |
| 1 CP + 2 Worker | `:11`, `:23-25` | `:237-238` | — | `:42` | 一致 |
| Harbor 在集群外 | `:26`, `:209` | `:185`, `:230` | ADR-003 `:8` | `:45` | 一致 |
| 备份仓库在集群外 | `:27`, `:69` | `:231-233` | ADR-006 `:14` | `:48` | 一致 |
| Calico | `:127`, `:178` | `:181`, `:213` | ADR-002 `:8` | `:44` | 一致 |
| local-path / Local PV | `:189` | `:214`, `:258` | ADR-004 `:8` | `:46` | 一致 |
| ClickHouse 单副本固定 xw-wk-02 | `:195`, `:65` | `:258` | ADR-004 `:8` | `:46` | 一致 |
| 不使用 Ceph/Longhorn | `:203` | `:441` | ADR-004 `:24` | `:54`, `:377` | 一致 |
| 唯一观测栈 | `:236-241` | `:291-293` | ADR-005 `:8-13`, `:27` | `:47` | 一致 |
| Task = Git YAML，非 CRD | `:15`, `:394` | `:205`, `:442` | ADR-008 `:8` | `:50` | 一致 |
| Agent = 短生命周期 Job | `:15`, `:379` | `:201` | ADR-007 `:8` | `:49` | 一致 |

**单一观测栈专项核查**（C02 明确要求不得出现第二套）：全仓库 grep 未发现
`Promtail`、`ELK`、`Elasticsearch`、`第二套 Grafana` 出现在任何生效文件中。
`ARCHITECTURE-BASELINE-V0.1.md:149` 明确禁用 KubeSphere ES Logging，
`:174` 明确日志固定由 Fluent Bit → Loki 提供。两清两固的 8 项指标
（`security-baseline.yaml:34`）声明复用现有 Prometheus 栈（`CLM/README.md:151`），
未引入新存储。**判定：无第二套观测栈。**

**DA-SOC 承载专项核查**（C02 要求"不得出现 ECS + Kubernetes 同时读取生产邮箱"）：

| 文件 | 行 | 陈述 |
|---|---|---|
| Baseline | `:13` | "不得让 ECS 和 Kubernetes 两个 n8n 同时读取生产邮箱" |
| Baseline | `:88`(§15.2) | "生产邮箱不 Mark as Read" |
| Adjudication | `:142`, `:168`, `:444`, `:462` | 生产双消费者明确否决 |
| ADR-001 | `:36` | "两个 n8n 不得同时读取生产邮箱、写入同一生产数据、发送同一日报" |
| CUTOVER-CHECKLIST | `:33`, `:46`, `:55` | 单活消费者证明；ECS 不消费 |
| ROLLBACK-PLAN | `:5`, `:29` | "只启动一个消费者" |
| TODO | `:33`, `:60` | "禁止 ECS n8n 与 Kubernetes n8n 同时消费生产邮箱" |
| IMPLEMENTATION-RISKS | `:10` | RISK-006 Critical，双消费 |

**判定：8 处独立表述完全一致，无任何一处允许双活。**

### 2.2 唯一例外：`da-soc-restore` Namespace 越权

见 **P1-01**。

---

## 3. Version Consistency

### 3.1 版本数值：一致（No-Issue）

同一组件在 5 个位置的版本**数值**完全一致：

| 组件 | README `:1149-1155` | TODO `:20-24` | VERSION-MATRIX §2/§3 | `components.yaml` | EXTERNAL-DEPS `:18-21` |
|---|---|---|---|---|---|
| OS | 统信 V20 1060e AMD64 | 同 | 同 | `:10` V20 1060e | `:9` 同 |
| Kubernetes | v1.30.6 | v1.30.6 | v1.30.6 | `:20` "1.30.6" | `:18` v1.30.6 |
| KubeSphere | 4.1.x 优先 4.1.2 | 同 | 同 | `:30` 4.1.x / 4.1.2 | `:19` 同 |
| containerd | 1.7.x | 1.7.x | 1.7.x | `:40` 1.7.x | `:20` 1.7.x |
| CNI | Calico | Calico | Calico | `:50` Calico | `:21` 同 |
| n8n | — | — | 2.15.0 | `:80` 2.15.0 | — |

`Candidate / Pending Compatibility Validation` 状态标签在 README、TODO、VERSION-MATRIX 三处一致，
且三处都附带"不代表官方认证组合"的限定。**判定：版本数值与候选状态使用正确，未发现
"文档 A = 1.30.6 / 文档 B = 1.29.x"这类冲突。**

### 3.2 版本状态词表：四套混用

见 **P1-06**。这是 C03 要求重点检查的 "Candidate / Frozen / Approved / planned 混用"，
实际混用程度比预期更高（4 套 token，语义部分重叠）。

---

## 4. TODO Consistency

### 4.1 架构是否全部落到 Task

| 架构要素 | 对应 Task | 判定 |
|---|---|---|
| 5 VM / OS 基线 | TASK-006 / 007 / 010 | 已覆盖 |
| Kubernetes / containerd / etcd / Audit | TASK-011～014 | 已覆盖 |
| KubeSphere + RBAC | TASK-015～017 | 已覆盖 |
| Calico / NetworkPolicy | TASK-018～021 | 已覆盖 |
| local-path / PVC | TASK-022～024 | 已覆盖 |
| Harbor（集群外） | TASK-025～028 | 已覆盖 |
| Observability 单一栈 | TASK-029～033 | 已覆盖 |
| Security Baseline / PSA / RBAC | TASK-034～037 | 已覆盖 |
| Backup | TASK-038～041 | 已覆盖 |
| DA-SOC 入仓 | TASK-042～049 | 已覆盖 |
| 迁移 / 单活 / Cutover | TASK-050～059 | 已覆盖 |
| AI Ops MVP | TASK-060～063 | 已覆盖 |
| 恢复演练 | TASK-064～066 | 已覆盖 |
| CLM | TASK-CLM-001～006 | 已覆盖 |
| 两清两固 | TASK-SEC-001～006 | 已覆盖 |

**判定：无架构要素缺失实施任务，也未出现"Architecture 禁止但 TODO 引入"的组件
（无 Task CRD、无 `xw-opsapi`、无第二集群任务、无 Ceph/Longhorn 任务）。**

### 4.2 Task 是否违反 Architecture

**是，且为 P0 级别。** 见 **P0-01**：TODO 把 Phase 13（生产切换）排在 Phase 15（恢复演练）之前，
而 Baseline §20 明确规定 S8 恢复演练必须在 S11 Cutover Gate 之前，并有一句硬约束：

> `ARCHITECTURE-BASELINE-V0.1.md:493`
> **未完成 S7/S8 的恢复验证，不得进入 DA-SOC 生产切换。**

此外另有 4 处任务编号引用错误，见 **P1-08**、**P1-09**、**P1-10**、**P2-10**。

---

## 5. Source of Truth Consistency

审计任务要求形成 SoT 映射。当前实际状态：

| 信息类别 | 声明的 SoT | 实际状态 | 问题 |
|---|---|---|---|
| 架构 | Baseline `:567` "本文件是实施唯一架构依据" | ✅ 唯一 | — |
| 版本 | VERSION-MATRIX `:10-14` + CLM README `:69` | ❌ **双份** | P1-03 |
| 组件身份 | `components.yaml:2` `source_of_truth: git` | ✅ | — |
| 生命周期策略 | `policies.yaml:2` | ✅ | — |
| 升级规则 | `upgrade-rules.yaml:2` | ✅ | — |
| 安全状态模型 | `security-baseline.yaml:2` + CLM README `:139` | ❌ **四份并行** | P1-02 |
| 端口 Desired State | `port-baseline.yaml:2` | ⚠️ 漏登记 Baseline 强制端口 | P2-01 |
| 账号 Desired State | `account-baseline.yaml:2` | ✅ | — |
| 访问控制 Desired State | `access-control-baseline.yaml:2` | ✅ | — |
| Task schema | Baseline `:394-427` + ADR-008 `:10` + `security-baseline.yaml:11` | ❌ **三份字段集不互通** | P2-06 |
| freshness / cadence | `policies.yaml:43-46` + `security-baseline.yaml:28-33` + 各基线自带 | ❌ **三份重复** | P2-08 |
| 实施任务 | TODO.md | ✅ 唯一（DoD#14 明确"未创建第二份 TODO"） | — |
| 外部依赖 Blocking 映射 | TODO §22 `:1178-1194` + EXTERNAL-DEPS `:6-50` | ❌ **双份且编号冲突** | P1-09 |
| 风险登记 | TODO `:1198` 声明 `IMPLEMENTATION-RISKS.md` 为权威 | ✅ 单一 | — |
| 执行方法 | `06-runbooks/` | ❌ **目录为空** | P3-05 |
| 运行事实 Evidence | Git（README `:1157` 指向 VERSION-MATRIX） | ⚠️ 未定义统一位置 | P2-05 |

**两处"都声称自己是 SoT"的直接冲突**：

1. **期望版本**：CLM README `:69` 写「`VERSION-MATRIX.md`：批准的期望版本、候选状态和冻结条件」，
   而同一文件 `:27` 写「由 `components.yaml` 定义组件身份、Owner、环境、关键性、**期望版本**和支持策略」。
   README.md `:1157` 也把 VERSION-MATRIX 定为版本冻结地。→ P1-03
2. **Blocking Task 映射**：TODO `:1176` 写「实施必须继续维护 `09-implementation/EXTERNAL-DEPENDENCIES.md`」
   并在 §22 内嵌了一张 Blocking 表，而 EXTERNAL-DEPS 自己也有一张 Blocking Task 表，两者编号不同。→ P1-09

---

## 6. CLM Consistency

### 6.1 CLM 是否与原有 AI Ops / Security Baseline / TODO 重复

| 复用检查 | 结论 |
|---|---|
| 是否新建 Task 模型 | ⚠️ **未新建存储，但新建了一套状态词表** → P1-02 |
| 是否新建漏洞管理机制 | ⚠️ **未引入平台，但 `upgrade-rules.yaml:46` 与 `security-baseline.yaml:8` 定义了两套 lifecycle** → P1-02 |
| 是否引入原架构拒绝的组件 | ✅ 否。无 SIEM/CMDB/SOAR/SBOM 平台/自动 Patch 平台 |
| 是否复用 Agent Runtime | ✅ 是。全部落在短生命周期 Job（`CLM/README.md:110`, `:122`） |
| 是否复用 Task 审批链 | ⚠️ 部分。复用 L0/L1/L2 与 DingTalk，但审批**级别判定**另建一套 → P1-04 |
| 是否复用观测栈 | ✅ 是（`CLM/README.md:151`） |

### 6.2 CLM 三文件之间的 schema 自洽性

**这是本次补丁最集中的问题区。** CLM README 声称：

> `CLM/README.md:39-48`
> ```yaml
> component: { id, name, type, owner, environment, criticality }
> desired:   { version, digest, support_policy }
> discovery: { method, endpoint, last_seen }
> security:  { highest_severity, cve_count, kev, affected, fixed_version }
> lifecycle: { support_status, eol, eos }
> upgrade:   { required, reason, priority, target_version, approval_required, status }
> audit:     { last_checked, source }
> ```

实际 `components.yaml`（17 个组件）中：

- `upgrade` 只有 `{ approval_required, status }`（如 `:13` `upgrade: { approval_required: true, status: planned }`），
  **缺少 `required` / `reason` / `priority` / `target_version`**
- `desired` 缺少 `digest`（除 `image.*` 外），且 Calico/Harbor 等 `version: unknown`
- **完全没有 `security:` 段**（highest_severity / cve_count / kev / affected / fixed_version）
- **完全没有 `audit:` 段**（last_checked / source）

而 `upgrade-rules.yaml:17-43` 的规则 `set:` 写的是**顶层** `upgrade_required`：

```yaml
- id: kev
  when: "kev == true"
  set: { upgrade_required: review, minimum_priority: P0 }
```

`when` 条件引用的是 `affected` / `severity` / `fixed_version` / `kev` / `eol` / `eos` /
`current_version` / `current_digest`，这些字段在 `components.yaml` 中**均不存在**。

**判定：CLM 的"规则引擎输入"与"Registry 输出"是两个不相交的 schema。** → P1-05

### 6.3 `status` 语义冲突矩阵（C06 指定检查项）

| 文件 | 使用的 status token | 与其他文件的关系 |
|---|---|---|
| `VERSION-MATRIX.md:10` | `Candidate` / `Pending Compatibility Validation` / `Frozen` | 权威词表 |
| `VERSION-MATRIX.md:28-43` | `Pending Version Freeze` / `Candidate / Verify` | 与 `:10` 词表不同 |
| `components.yaml:10,20,30,40,50,80,90` | `status: candidate` | 与 `:10` 语义同、token 不同 |
| `components.yaml:60,70,100…` | `status: pending-freeze` | **在 VERSION-MATRIX 词表中未定义** |
| `components.yaml:12,22,32…` | `lifecycle.support_status: pending-validation` | **第三套，未在 VERSION-MATRIX 出现** |
| `components.yaml:13,23,33…` | `upgrade.status: planned` | 第四套 |
| `upgrade-rules.yaml:46` | `discovered…rolled-back, review` | 与 `security-baseline.yaml:8` 冲突 → P1-02 |
| `security-baseline.yaml:8` | `discovered…escalated` | 与 `upgrade-rules.yaml:46` 冲突 → P1-02 |
| `ADR-008:15-16` | `open…pending-approval…escalated` | 第三套 → P1-02 |
| `Baseline:414,421,426` | `pending/approved/rejected`、`pending/passed/failed`、`open/closed/escalated` | 第四套 → P1-02 |
| `TODO.md:1029` | `PASS/FAIL/EXCEPTION` | 第五套 → P2-02 |

**判定：`status` 语义冲突确实存在，且集中在两次补丁引入的文件中。** → P1-02、P1-06

### 6.4 Component / Desired / Observed / Task / Approval 链路完整性

| 环节 | 定义位置 | 判定 |
|---|---|---|
| Component | `components.yaml` | ✅ |
| Desired State | `components.yaml.desired` + VERSION-MATRIX | ⚠️ 双份 P1-03 |
| Observed State | `CLM/README.md:72` "Discovery evidence" | ⚠️ **存放位置未定义** P2-05 |
| Vulnerability State | `CLM/README.md:31`, `:45` | ⚠️ schema 未落地 P1-05 |
| Lifecycle State | `components.yaml.lifecycle` + 两处 lifecycle 列表 | ⚠️ 三份 P1-02 |
| Upgrade State | `upgrade-rules.yaml:45-53` | ⚠️ 与 Registry 不相接 P1-05 |
| Task | 复用 Git Task | ⚠️ 字段集不同 P2-06 |
| Approval | `policies.yaml:29-41` | ⚠️ 双机制 + 无缺省 P1-04 |
| Execution / Verification / Audit | `CLM/README.md:110`, `TODO.md:1129-1130` | ✅ |

---

## 7. Two-Clear-Two-Firm Consistency

### 7.1 统一闭环模型核查（C07 指定）

Baseline `:577` 与 CLM README `:139` 都声明四项能力使用：

```
Desired State → Observed State → Deviation → Risk → Task → Approval
→ Remediation → Verification → Audit
```

四份 YAML 的实际支持情况：

| 能力 | Desired | Observed | Deviation | Risk | Task | Approval | Remediation | Verification | Audit |
|---|---|---|---|---|---|---|---|---|---|
| 清高危漏洞 | ✅ `components.yaml` | ✅ discovery | ⚠️ 未定义字段 | ✅ priority+severity | ✅ | ⚠️ P1-04 | ✅ `upgrade-rules` | ✅ CLM README `:130` | ✅ `:122` |
| 清高危端口 | ✅ `port-baseline.baseline` | ✅ `:5-9` | ⚠️ 未定义字段 | ✅ `:10` risk_level+priority | ✅ `:72-76` | ✅ `:77-80` | ✅ | ⚠️ 仅 `rescan_after_remediation` | ⚠️ 未定义 |
| 固弱账号口令 | ✅ `account-baseline.checks` | ✅ `:6-11` | ⚠️ 未定义字段 | ⚠️ 仅 severity，无 priority | ✅ `:59-62` | ✅ `:66-69` | ✅ | ✅ `:63-65` | ⚠️ 未定义 |
| 固弱访问控制 | ✅ `access-control.layers` | ✅ 各 control.discovery | ✅ `:5` 含 deviation 字段 | ⚠️ `:5` 有 risk 字段 | ✅ `:63-66` | ✅ `:70-73` | ✅ | ✅ `:67-69` | ⚠️ 未定义 |

**发现：四份基线没有一份显式定义 `deviation` 的数据结构**（`access-control-baseline.yaml:5`
的 `record_fields` 含 `deviation`，但其余三份的 `record_fields` 都不含）。
`security-baseline.yaml` 作为"统一状态模型"定义方，也没有定义 deviation 的判定规则。→ P2-08

### 7.2 PASS / FAIL / REVIEW / UNKNOWN 一致性

- `Baseline:577`：状态只有 `PASS`、`FAIL`、`REVIEW`、`UNKNOWN`
- `security-baseline.yaml:6`：`statuses: [PASS, FAIL, REVIEW, UNKNOWN]` ✅
- `security-baseline.yaml:7`：`unknown_is_pass: false` ✅
- `CLM/README.md:144,160`：同上 ✅
- `README.md:1275`：`UNKNOWN 不得默认为 PASS` ✅
- `TODO.md:1224`（DoD#13）：`UNKNOWN 可见且不默认为 PASS` ✅
- `RISK-CLM-001`、`RISK-SEC-001`：过期/未知 → 转人工 ✅

**判定：UNKNOWN 处理在全仓库一致，未发现任何一处将 UNKNOWN 当作 PASS/SAFE。**
唯一偏离是 `TODO.md:1029` 的 `EXCEPTION`（不在四状态内）→ P2-02。

### 7.3 L0/L1/L2 一致性（C07 + C09 指定）

| 文件 | L0 | L1 | L2 |
|---|---|---|---|
| `Baseline:303-305, :461-471` | 只读检查/报告/查询/告警聚合 | 白名单 Runbook | RBAC/NetPol/CNI/节点/存储/生产 n8n/数据/凭据 |
| `ADR-007:10` | 默认只读 | 白名单动作 | 人工审批后短时授权 |
| `CLM/README.md:96-98` | 发现/判断/报告/Task | 验证环境 Patch、非生产白名单 | K8s/KS/Calico/Harbor/OS/ClickHouse/节点/存储/网络/RBAC |
| `security-baseline.yaml:19-23` | discover/assess/report/create_task/notify | non_production_remediation | 10 类生产对象 |
| `port-baseline.yaml:78-80` | record/notify/create_task | close_non_production_port | close_production_port / firewall / Calico / Ingress |
| `account-baseline.yaml:67-69` | inventory/assess/report/create_task | disable_non_production_stale_account | 生产凭据/特权账号/cluster-admin |
| `access-control-baseline.yaml:71-73` | audit/report/create_task | non_production_policy_change | 生产 RBAC/NetPol/firewall/Secret 边界 |
| `README.md:1275` | — | — | 生产 RBAC/防火墙/NetPol/生产账号/Harbor 管理/DA-SOC 访问控制 |
| `TODO.md:62, :928` | — | 无白名单被拒 | 无人工审批不创建/不执行 |

**判定：8 处表述语义一致，无扩大也无缩小。** ✅

**P0 级检查（绕过 L2 的路径）**：

- `components.yaml` 全部 17 个组件 `upgrade.approval_required: true` ✅
- `policies.yaml:8,14,20,26` 全部 `production_auto_upgrade: false` ✅
- `upgrade-rules.yaml:47-53` `forbidden_automatic_actions` 含 production_upgrade / k8s upgrade / os_upgrade ✅
- `Baseline:317` cluster-admin 必须 break-glass + 双人 + Incident ✅
- `access-control-baseline.yaml:66` `agent_escalation: BLOCK_AND_ESCALATE_TO_L2` ✅
- `account-baseline.yaml:60` `secret_value_protection: FAIL_AND_REDACT_AND_CREATE_INCIDENT` ✅
- `RISK-CLM-004`（Critical）：专门覆盖"CLM 自动生成升级 Task 绕过审批" ✅
- `TODO.md:1115` "不自动升级生产"、`:1116` "审批前不能创建升级 Job" ✅

**判定：未发现绕过 L2 的路径。** 唯一的权限判定弱点是 `policies.yaml` 的缺省级别未定义
（不是绕过，而是**无规则可依**）→ P1-04。

---

## 8. V0.1 Scope / HA Consistency

### 8.1 隐性范围膨胀检查（C08 指定清单逐项）

| 检查项 | 结果 | 证据 |
|---|---|---|
| 多 Control Plane | ✅ 未引入 | `Adjudication:440`, `TODO:54` |
| 多集群 | ⚠️ **风险登记隐含要求** | `IMPLEMENTATION-RISKS.md:19,21` "独立验证集群" vs `Baseline:11` "单集群" → **P1-11** |
| 跨地域 DR | ✅ 未引入 | `Adjudication:441` |
| Ceph / Longhorn | ✅ 未引入 | `Baseline:203`, `TODO:377` |
| Service Mesh | ✅ 未引入 | `Baseline:149,183`, `TODO:275` |
| SIEM | ✅ 未引入 | `Baseline:241`, `Adjudication:441`；两清两固复用现有栈 |
| SOAR | ✅ 未引入 | `CLM/README.md:116` 明确 deferred |
| CMDB | ✅ 未引入 | 同上 |
| 完整 IAM | ✅ 未引入 | `Adjudication:441` |
| 自动 Kubernetes 升级 | ✅ 明确禁止 | `Baseline:437`, `CLM/README.md:117`, `upgrade-rules.yaml:49-50` |
| 自动 OS 升级 | ✅ 明确禁止 | 同上 |
| 自动生产 Patch | ✅ 明确禁止 | `CLM/README.md:100,117` |
| GPU | ✅ 未引入 | `Adjudication:441`, `TODO:54` |
| 复杂 Policy Engine | ✅ 未引入 | `TODO:54` |
| Task CRD | ✅ 未引入 | `Baseline:15`, `ADR-008:8`, `TODO:50,969,1043`（V0.4 backlog） |
| 常驻高权限 Agent | ✅ 未引入 | `Baseline:15`, `ADR-007:8`, `TODO:49` |

**判定：两次补丁没有把任何被否决的能力变成 V0.1 必须实施内容。
唯一越界是风险登记中的"独立验证集群"。** → P1-11

### 8.2 反向检查：是否因强调"非 HA"而删掉了必须的备份/恢复能力

| 能力 | 是否保留 | 证据 |
|---|---|---|
| etcd snapshot | ✅ | `Baseline:279`, `ADR-006:19` |
| ClickHouse 原生 BACKUP/RESTORE | ✅ | `Baseline:281`, `ADR-004:19` |
| raw archive 文件备份 + 校验和 | ✅ | `Baseline:282` |
| Harbor 备份（配置/DB/registry data/镜像 tar） | ✅ | `Baseline:285`, `ADR-003:21` |
| Secret 加密导出 + 离线密钥 | ✅ | `Baseline:286` |
| 离线/不可变副本 | ✅ | `Baseline:292`, `ADR-006:14` |
| 6 项强制恢复演练 | ✅ | `Baseline:293`, `ADR-006:34-41`, `RESTORE-DRILL-PLAN` D1–D4 |
| RPO 24h / RTO 4h/8h | ✅ | `Baseline:290-291`, `ADR-006:30` |
| 故障演练 | ✅ | `TODO.md:979-1021` (TASK-064~066) |
| POP3 重放兜底 | ✅ | `Baseline:293`, `RESTORE-DRILL-PLAN:32` |

**判定：备份与恢复能力完整保留，未被"非 HA"叙事削弱。
唯一问题是这些能力在 TODO 中的执行位置晚于切换（P0-01）。**

---

## 9. Agent / L0-L1-L2 Consistency

### 9.1 Agent 权限过大检查（C09 指定）

| 禁止项 | 是否被违反 | 证据 |
|---|---|---|
| `cluster-admin` | ✅ 未授予 | `Baseline:317,530`, `ADR-007:22`, `Adjudication:283`, `TODO:303` |
| `root` | ✅ 未授予 | `Baseline:44`, `ADR-007:22` |
| 长期 kubeconfig | ✅ 未授予 | `ADR-007:22` |
| 任意 shell / 宿主机命令 | ✅ 未授予 | `Baseline:324` |
| 生产 Secret | ✅ 未授予 | `Baseline:321`, `security-baseline.yaml:15`, `account-baseline.yaml:11` |
| 生产邮箱 | ✅ 未授予 | `Baseline:371`, `ADR-007:22`, `account-baseline.yaml:65` |
| 生产 DingTalk 凭据 | ✅ 未授予 | `Baseline:372`, `ADR-001:20` |

### 9.2 Agent 权限定义完整性

- `Baseline:154-164` 七个身份的 RBAC 清单 ✅
- `Baseline:164` 明确 `agent-l1-render` 的禁止清单（不读 Secret、不改 RBAC/NetPol、不删 PVC、不改 digest、不访问节点、不执行任意命令）✅
- `CLM/README.md:98` L2 清单覆盖 K8s/KS/Calico/Harbor/OS/ClickHouse/生产镜像/节点/存储/网络/RBAC ✅
- `RISK-CLM-001/002/004` 覆盖 discovery 失败、digest 漂移、审批绕过 ✅

**判定：C09 未发现 Critical。** 唯一弱点是 P1-04 的审批缺省级别。

---

## 10. Detailed Findings

### P0 — 阻断实施

---

#### <a id="p0-01"></a>P0-01 — TODO 把生产切换排在恢复演练之前，直接违反 Baseline 硬门禁

**Severity:** P0
**Category:** C04 TODO 一致性 / C08 Scope 边界

**File A:** `10-decisions/ARCHITECTURE-BASELINE-V0.1.md`
**Line / Section:** `:473-493`（§20 Deployment Sequence）、`:493`（硬约束句）

**Current statement:**
```text
S7  etcd/Git/ClickHouse/raw/n8n/Harbor/Secret 备份
S8  恢复演练：控制面、资源、ClickHouse、Harbor、raw
S9  da-soc-validate 回放验证
S10 da-soc 正式资源创建、数据导入、workflow 导入
S11 Cutover Gate 评审，停止 ECS n8n，启动 K8s n8n
...
未完成 S7/S8 的恢复验证，不得进入 DA-SOC 生产切换。
```

**File B:** `TODO.md`
**Line / Section:** `:35`（§1 Implementation Overview）

**Current statement:**
```text
实施顺序固定为：参数冻结 → 基础设施 → Kubernetes → KubeSphere → Calico/网络 → Local PV
→ Harbor → 观测 → 安全 → 备份 → DA-SOC 部署 → 数据迁移 → 业务验证
→ 切换/回退 → AI Ops MVP → 恢复演练 → 最终验收。
```
（`切换/回退` 在 `恢复演练` 之前）

**File B:** `TODO.md`
**Line / Section:** `:877`（Phase 13 — Cutover）、`:979`（Phase 15 — Failure / Recovery Drills）

**Current statement:** Phase 13（TASK-057~059）编号早于 Phase 15（TASK-064~066）。

**File B:** `TODO.md`
**Line / Section:** `:1159-1161`（§21 Dependency Graph）

**Current statement:**
```text
TASK-057 ─> TASK-058 ─> TASK-059
TASK-064/TASK-065/TASK-066 ─> TASK-067 ─> TASK-068
```
（切换链与演练链之间**无任何依赖边**）

**File C:** `09-implementation/IMPLEMENTATION-RISKS.md`
**Line / Section:** `:15`（RISK-011）

**Current statement:**
```text
| RISK-011 | Backup 文件存在但不可恢复 | Critical | 未做 restore drill | 强制恢复验收 |
  backup age/restore evidence | 不允许切换，重新备份 | Backup |
```

**Conflict:**
Baseline 把恢复演练定义为 S8、把切换定义为 S11，并写明未完成 S7/S8 不得切换。
TODO 把切换排在演练之前，且依赖图没有加任何"演练完成才能切换"的门禁边。
RISK-011 的处置条款同样写明"不允许切换"。三份文件对同一门禁给出互斥的执行顺序。

**Impact:**
按 TODO 执行，TASK-058（禁用 ECS 生产消费者并启用 K8s 单活）会在
TASK-064/065/066 全部尚未执行的情况下发生。此时：

- ClickHouse / raw archive / n8n state 是否可恢复**没有任何实证**；
- Baseline `:203` 明确 Local PV 是单节点风险，数据安全"必须由 ClickHouse BACKUP、
  文件备份、外部仓库和 POP3 重放共同覆盖"——而这四项全部要在 TASK-064~066 才被证明；
- DA-SOC 没有第二个数据源。若切换后发现数据不可恢复，只能靠 POP3 重放，
  而 `Baseline:554` 已把"无法解释的数据缺失"列为立即停止条件；
- RISK-011 本身是 **Critical** 级风险，其处置被 TODO 的执行顺序架空。

这属于"继续实施可能造成严重问题"，且是本次审计中唯一触及生产数据不可逆损失的门禁冲突。

**Recommended Resolution:**
1. 在 `TODO.md` §21 依赖图加入硬门禁边：
   `TASK-064/TASK-065/TASK-066 ─> TASK-056`（Cutover Gate 评审包）`─> TASK-057`。
2. 把 `TODO.md:35` 的固定顺序改为：
   `… → DA-SOC 部署 → 数据迁移 → 业务验证 → **恢复演练** → 切换/回退 → AI Ops MVP → 最终验收`。
3. 在 `TASK-056`（Cutover Gate 评审包，`:863`）的 `Preconditions` 中显式加入
   `TASK-064～TASK-066 完成并归档证据`。
4. 在 `TASK-057`（`:879`）的 `Preconditions` 中重复该门禁，作为第二道防线。
5. 可选：把 Phase 15 整段前移至 Phase 12 之后，使文档顺序与 Baseline S8 一致。

**Architecture Change Required:** NO
（Baseline §20 已经写对了；问题在 TODO 未对齐。属执行计划修订，不需要改动架构。）

---

### P1 — 必须修复后才能进入实施

---

#### <a id="p1-01"></a>P1-01 — `da-soc-restore` Namespace 超出 Baseline 授权的 Namespace 集合

**Severity:** P1 | **Category:** C08 / C04

**File A:** `ARCHITECTURE-BASELINE-V0.1.md:133-142`（§5.2 Namespace）
```text
正式 Namespace：kube-system / kubesphere-system / xw-platform / xw-observability / xw-aiops / da-soc
迁移验证阶段可临时创建 `da-soc-validate`……切换后必须删除或标记为不可生产使用。
正式生产命名空间只有 `da-soc`。
```

**File B:** `TODO.md:997`（TASK-065 Preconditions）
```text
- **Preconditions:** TASK-040、TASK-047～049、TASK-064 完成；隔离 `da-soc-restore` 环境可用。
```

**File C:** `RESTORE-DRILL-PLAN.md:20`（Drill D2）
```text
- 动作：在隔离 `da-soc-restore` 环境恢复 ClickHouse、raw、n8n/render 配置。
```

**File D:** `TODO.md:998`（TASK-065 Inputs）
```text
- **Inputs:** ClickHouse BACKUP、raw manifest、n8n artifact/state/key、render config、测试 Secret。
```

**Conflict:** Baseline 只授权 `da-soc-validate` 一个临时 Namespace。
`da-soc-restore` 未在 Baseline §5.2 出现，因此**没有**对应的 ResourceQuota、LimitRange、
NetworkPolicy、RBAC、PSA 或销毁期限约束。同时 `Baseline:339` 要求
"策略变更必须先在 `da-soc-validate` 验证，再进入正式 `da-soc`"——
恢复演练引入了第三个未定义空间。

**Impact:** 恢复演练会把完整 ClickHouse 数据集、raw archive、n8n workflow/state
**以及 n8n encryption key** 放进一个无授权、无网络策略、无配额定义的空间。
该空间一旦未按 Baseline §5.2 的纪律销毁，会成为长期影子生产环境，
直接违反 `Baseline:527`（数据可恢复）与 `Baseline:571`（不得以测试环境为由取消门槛）。

**Recommended Resolution:** 二选一，写入 Baseline：
- **(a)** 在 `Baseline:142` 的临时 Namespace 列表补入 `da-soc-restore`，
  并要求复用 `da-soc` 的 Quota/NetworkPolicy/PSA 模板，强制带 `production=false` 标签与销毁期限；
- **(b)** 明确 Drill D2 在集群外的隔离 VM/临时环境执行，集群内不创建该 Namespace。
  注意 `TODO.md:275`（TASK-015）已使用"独立验证**环境**"这一措辞，(b) 与现有用词一致。

**Architecture Change Required:** YES（Baseline §5.2 一行增补或澄清。
这是本次唯一需要触碰 Baseline 的点，且属授权范围澄清，不是重新设计。）

---

#### <a id="p1-02"></a>P1-02 — 四套互不兼容的 Task / CLM 生命周期状态词表

**Severity:** P1 | **Category:** C06 / C07

**File A:** `10-decisions/ADR/ADR-008-task-model.md:14-17`
```text
open → analyzed → planned → pending-approval → approved/rejected
→ executing → verified/failed → closed/escalated
```

**File B:** `ARCHITECTURE-BASELINE-V0.1.md:411-427`（§16.3 Task YAML）
```yaml
approval:   { required, approver, decision: pending|approved|rejected }
verification: { status: pending|passed|failed, evidence }
audit:      { agent_digest, git_commit, result: open|closed|escalated }
```

**File C:** `04-security/security-baseline.yaml:5-8`
```yaml
state_model:
  statuses: [PASS, FAIL, REVIEW, UNKNOWN]
  lifecycle: [discovered, assessed, task-created, approval-pending, remediation, verifying, closed, escalated]
```

**File D:** `07-aiops/component-lifecycle/upgrade-rules.yaml:45-46`
```yaml
lifecycle:
  statuses: [discovered, assessed, upgrade-required, task-created, approved, executing,
             verifying, closed, rolled-back, review]
```

**Conflict:** 同一"Task 生命周期"概念有四套状态集合，互不包含：

| Token | ADR-008 | Baseline YAML | security-baseline | upgrade-rules |
|---|:--:|:--:|:--:|:--:|
| `open` | ✅ | ✅(`audit.result`) | — | — |
| `analyzed` / `assessed` | analyzed | — | assessed | assessed |
| `planned` / `task-created` | planned | — | task-created | task-created, upgrade-required |
| **`pending-approval`** | ✅ | `pending` | **`approval-pending`** | — |
| `approved` / `rejected` | ✅ | ✅ | — | approved |
| `executing` / `remediation` | executing | — | remediation | executing |
| `verified` / `verifying` | verified | `passed`/`failed` | verifying | verifying |
| `escalated` | ✅ | ✅ | ✅ | ❌ |
| `rolled-back` | ❌ | — | ❌ | ✅ |
| `review` | ❌ | — | （REVIEW 作为状态） | review |

同名概念不同 token 的确证：`pending-approval`（ADR-008）与 `approval-pending`
（security-baseline.yaml）是同一状态的两种拼写。

**Impact:** 这是 AI-Native 运维模式最直接的可执行性缺陷：

- Agent 按 ADR-008 写 Task → 检查器按 `security-baseline.yaml` 校验 → 状态不识别；
- 未识别状态若落到 `unknown_is_pass: false`（`security-baseline.yaml:7`），
  Task 永远无法关闭，两清两固 `remediation_closure_rate` 与
  CLM `Upgrade Closure Rate`（`CLM/README.md:132`）永久不达标；
- 反向后果更糟：若某实现把未知状态当作"已批准"，则 L2 门禁失效——
  这是 C09 关注的风险，只是目前没有显式绕过路径，而是**没有可判定的状态**。

**Recommended Resolution:**
1. 指定唯一权威状态机（建议以 ADR-008 为基础，补入 `rolled-back`），例如：
   ```text
   open → analyzed → planned → pending-approval → approved/rejected
        → executing → verifying → verified/failed → closed | escalated | rolled-back
   ```
2. `security-baseline.yaml:8` 与 `upgrade-rules.yaml:46` 改为引用该权威枚举，
   不再各自定义。
3. 明确区分两个不同维度并写入 `security-baseline.yaml`：
   - **finding status**（检查结论）：`PASS | FAIL | REVIEW | UNKNOWN`
   - **task status**（任务状态）：上表 12 个 token
   当前把两者混在同一 `state_model` 下是词表冲突的直接成因。

**Architecture Change Required:** NO

---

#### <a id="p1-03"></a>P1-03 — CLM 期望版本存在双 Source of Truth

**Severity:** P1 | **Category:** C05 / C03

**File A:** `07-aiops/component-lifecycle/README.md:67-72`（§4 Version Truth）
```text
- `09-implementation/VERSION-MATRIX.md`：批准的期望版本、候选状态和冻结条件。
- Kubernetes workload `image digest`：实际部署事实……
- Git CLM YAML：组件、策略、升级规则和审批边界的 Source of Truth。
```

**File B:** `07-aiops/component-lifecycle/README.md:25-27`（§2 Component Registry）
```text
回答"平台管理什么"，由 `components.yaml` 定义组件身份、Owner、环境、关键性、
**期望版本**和支持策略。
```

**File C:** `07-aiops/component-lifecycle/components.yaml:20,30,40,80,90`
```yaml
- id: platform.kubernetes
  desired: { version: "1.30.6", status: candidate, … }
- id: platform.kubesphere
  desired: { version: "4.1.x", preferred_version: "4.1.2", … }
- id: business.n8n
  desired: { version: "2.15.0 or approved existing version", … }
```

**File D:** `README.md:1157`
```text
……才能在 `09-implementation/VERSION-MATRIX.md` 中冻结。
```

**Conflict:** 同一份 CLM README 的 §2 和 §4 各自把"期望版本"判给不同文件；
README.md 站在 VERSION-MATRIX 一边。VERSION-MATRIX 与 components.yaml 同时持有
可写的版本字段，且措辞已出现差异（`"1.30.6"` vs `"v1.30.6"`；
`"4.1.x，优先验证 4.1.2"` vs `preferred_version: "4.1.2"`）。

`TODO.md:1058`（TASK-CLM-001 Inputs）同时把两者列为输入，
`TODO.md:20-26`（§0 Version Baseline Status）又单独维护第三份版本表。

**Impact:** TASK-002「冻结版本、镜像和 digest」执行后若只更新 VERSION-MATRIX，
components.yaml 会静默漂移 → CLM-004 的升级判断基于过期期望版本 →
`priority`/`target_version` 计算错误。且 TASK-002 与 TASK-CLM-001 都会被合理地
指派为"版本权威"，产生两个 Owner。

**Recommended Resolution:**
1. 声明 `VERSION-MATRIX.md` 为**唯一**期望版本 SoT。
2. `components.yaml` 的 `desired.version` 改为引用式：
   `version_ref: "VERSION-MATRIX.md#platform.kubernetes"`，或由 CI 从 VERSION-MATRIX 生成。
3. 增加一致性校验：VERSION-MATRIX 与 components.yaml 版本字段不等则 CI 失败。
4. `TODO.md §0` 的版本表改为链接到 VERSION-MATRIX，不重复维护。

**Architecture Change Required:** NO

---

#### <a id="p1-04"></a>P1-04 — CLM 审批级别存在两套并行机制，且无优先级与缺省定义；L1 分支不可达

**Severity:** P1 | **Category:** C06 / C09

**File A:** `07-aiops/component-lifecycle/policies.yaml:29-41`
```yaml
approval_rules:
  - condition: "priority in [P0, P1]"                              → L2
  - condition: "component in [kubernetes, kubesphere, calico, harbor, os, clickhouse]" → L2
  - condition: "environment == production"                        → L2
  - condition: "environment in [validation, test] and rollback_capable == true" → L1
```

**File B:** `07-aiops/component-lifecycle/policies.yaml:3-27`
```yaml
policies:
  - id: platform-core      component_types: [os, platform, runtime, cni, registry]  approval_level: L2
  - id: da-soc-core        component_types: [database, workflow, business-service]  approval_level: L2
  - id: observability      component_types: [observability, logging]               approval_level: L2
  - id: non-production     environment: [validation, test]                          approval_level: L1
```

**File C:** `07-aiops/component-lifecycle/components.yaml`（全部 17 个组件）
```yaml
environment 取值仅有：management, production, kubernetes, platform, da-soc
```
**没有任何组件使用 `validation` 或 `test`。**

**File D:** `TODO.md:1137`（TASK-CLM-006 Definition of Done）
```text
至少一个非生产/验证组件完成一次受控升级验证；生产自动升级保持禁用。
```

**Conflict:** 三重问题：

1. **两套机制并存且无优先级。** `policies[].approval_level`（按 component_types）
   与 `approval_rules[]`（按条件列表）都能决定审批级别，文件未说明谁优先。
2. **无规则匹配时无缺省级别。** 以 `business.n8n`（`environment: da-soc`）的 P2 级升级为例：
   - rule 1：priority=P2 → 不匹配
   - rule 2：n8n 不在核心组件列表 → 不匹配
   - rule 3：`environment == production`？n8n 是 `da-soc` → **不匹配**
   - rule 4：`environment in [validation, test]`？否 → **不匹配**
   → 四条规则全部不命中，`level` 未定义。
3. **L1 分支不可达。** rule 4 与 `non-production` policy 都要求
   `environment in [validation, test]`，而 Registry 中零个组件满足；
   同时 TASK-CLM-006 要求"至少一个非生产/验证组件完成一次受控升级验证"——
   这样的组件在 Registry 中不存在，该 DoD 不可满足。

**Impact:**
- 若实现取"无匹配即最低级别"或"无匹配即放行"，`business.n8n` / `data.clickhouse` /
  `observability.*` 的 P2 级升级将绕过人工审批。这正是
  `IMPLEMENTATION-RISKS.md:25` RISK-CLM-004（**Critical**：
  "CLM 自动生成升级 Task 绕过审批"）所描述的场景。
- 即使实现取"默认最严"，规则也是靠猜而非靠定义，Agent 无法解释判断依据，
  与 `Baseline:437`「CLM 可以生成 Git Upgrade Task…但生产升级…必须遵守 L2 人工审批」
  的可解释性要求冲突。

**Recommended Resolution:**
1. 在 `policies.yaml` 顶部明确：`approval_rules` 优先于 `policies[].approval_level`；
   **未匹配任何 rule 时回落到所属 policy 的 `approval_level`**；
   并显式增加兜底规则 `- condition: "default" → level: L2`。
2. 统一 `environment` 词表（P3-07），并为升级验证登记 `validation` 环境组件，
   或删除 rule 4 / `non-production` policy 的 L1 分支，把
   TASK-CLM-006 的 DoD 改为"在生产执行一次经 L2 批准的升级演练"。
3. 明确 `production` 的定义：`components.yaml` 中 `data.clickhouse` /
   `business.n8n` / `business.render-archive` 的 `environment` 是否应包含 `production`。

**Architecture Change Required:** NO

---

#### <a id="p1-05"></a>P1-05 — `upgrade_required` 字段名与类型在三份 CLM 文件中不一致

**Severity:** P1 | **Category:** C06

**File A:** `07-aiops/component-lifecycle/README.md:42-48`（schema 模板）
```yaml
upgrade: { required: "", reason: [], priority: "", target_version: "", approval_required: true, status: "" }
```
→ 字段名 `upgrade.required`，类型为字符串模板。

**File B:** `07-aiops/component-lifecycle/components.yaml:13,23,33,43,…`（17 处）
```yaml
upgrade: { approval_required: true, status: planned }
```
→ **没有 `required` 字段**，也没有 `reason` / `priority` / `target_version`。

**File C:** `07-aiops/component-lifecycle/upgrade-rules.yaml:17-43`
```yaml
- id: critical-fixed-version
  when: "affected == true and severity == Critical and fixed_version != null"
  set: { upgrade_required: true, minimum_priority: P1 }
- id: kev
  when: "kev == true"
  set: { upgrade_required: review, minimum_priority: P0 }
- id: not-affected
  when: "cve_count > 0 and affected == false"
  set: { upgrade_required: review }
```
→ 顶层字段名 `upgrade_required`，类型是 **`true` / `review` 三态**，不是布尔。

**File D:** `TODO.md:1101`（TASK-CLM-004 Actions）
```text
输出 `upgrade_required`、`upgrade_reason`、priority、target_version、
approval_required、status 和 deadline
```

**Conflict:** 同一概念三种字段名（`upgrade.required` / `upgrade.approval_required` /
`upgrade_required`）、两种类型（布尔 / 三态字符串）、两处命名（`reason` / `reason_codes` /
`upgrade_reason`；`priority` / `minimum_priority`）。
另外 `upgrade-rules.yaml` 的 `when` 条件读取的 `affected`、`severity`、`kev`、
`fixed_version`、`eol`、`eos`、`cve_count`、`current_version`、`current_digest`
**在 `components.yaml` 中全部不存在**——`security:` 与 `audit:` 段整体缺失。

**Impact:**
- 规则引擎 `set:` 的目标字段在 Registry 中不存在 → 规则结果无法落盘；
- Agent 按 TASK-CLM-004 输出的字段名与 README schema 不一致 → 校验失败；
- `upgrade_required` 恒为空 → TASK-CLM-004 Validation「每个判断可回放」、
  DoD#12「升级判断可解释」均不可验证。

**Recommended Resolution:**
1. 统一 schema 为：
   ```yaml
   upgrade:
     required: true | false | review      # 三态，与 upgrade-rules 对齐
     reason_codes: [CVE, CVSS_CRITICAL, KEV, EOL, SECURITY_PATCH, POLICY]
     priority: P0 | P1 | P2 | P3
     target_version: "…"
     approval_required: true
     status: …
     deadline: "…"
   ```
2. `upgrade-rules.yaml` 的 `set` 改用点号路径（`set: { upgrade.required: true }`）。
3. `components.yaml` 补齐 `security:` 与 `audit:` 段（可全部初始化为 `unknown`），
   使 `when` 条件有输入；或明确声明"规则输入来自 discovery evidence 而非 Registry"并
   给出 evidence 的 schema（目前 `CLM/README.md:72` 只说 evidence 不手工覆盖，无 schema）。

**Architecture Change Required:** NO

---

#### <a id="p1-06"></a>P1-06 — 冻结状态词表四套混用，`pending-freeze` 未在权威词表中定义

**Severity:** P1 | **Category:** C03

**File A:** `09-implementation/VERSION-MATRIX.md:10-11`（Version Freeze Rules）
```text
1. 本文件区分 `Candidate`、`Pending Compatibility Validation` 和 `Frozen`；
   候选版本不得描述为官方认证组合。
```

**File B:** `VERSION-MATRIX.md:28-43`（实际表格使用）
→ `Pending Version Freeze`（Harbor/ClickHouse/Prometheus/…）、`Candidate / Verify`（n8n、render）

**File C:** `07-aiops/component-lifecycle/components.yaml`
→ `:10,20,30,40,50,80,90` `status: candidate`
→ `:60,70,100,110,120,130,140,150,160` `status: pending-freeze` ← **未在 VERSION-MATRIX 词表中定义**
→ `:12,22,32,…` `lifecycle.support_status: pending-validation` ← **第三套，VERSION-MATRIX 中不存在**
→ `:13,23,33,…` `upgrade.status: planned` ← **第四套**

**File D:** `README.md:1151-1155`、`TODO.md:20-24`
→ 统一使用 `Candidate / Pending Compatibility Validation`

**Conflict:** 同一"版本冻结语义"有四种写法：
`Pending Compatibility Validation` / `Pending Version Freeze` / `pending-freeze` /
`pending-validation`。VERSION-MATRIX §1.1 声称自己是词表权威，但 components.yaml
使用了两个它未定义的 token；README 与 TODO 又各用第三种组合。

**Impact:**
- "该组件是否已冻结"这一 TASK-002 的核心门禁**无法机器判定**；
- `VERSION-MATRIX.md:10` 「候选版本不得描述为官方认证组合」这条约束失去机器可执行性；
- `lifecycle.support_status`（组件支持状态）与 `desired.status`（版本冻结状态）
  是两个正交维度，目前混用同一套 token 表达，未来必然互相污染。

**Recommended Resolution:**
1. 在 `VERSION-MATRIX.md:10` 定义唯一枚举并全仓库统一：
   `candidate | validating | frozen | superseded`
2. 修正 components.yaml：`pending-freeze` → `validating`；
   `pending-validation` → 归入独立的 `lifecycle.support_status` 枚举
   （`supported | eol | unknown`），不与冻结状态共用 token。
3. README / TODO 的状态列改为直接引用 VERSION-MATRIX 的枚举值。

**Architecture Change Required:** NO

---

#### <a id="p1-07"></a>P1-07 — README §23/§24 仍停留在"架构设计阶段"，与 Baseline/TODO 的 Frozen/READY 冲突

**Severity:** P1 | **Category:** C01

**File A:** `README.md:1143`
```text
**当前版本：V0.1 --- Architecture Frozen / Implementation TODO Ready**
```

**File B:** `README.md:1165-1174`（同一节的"当前首要任务"）
```text
1. 完成玄武云盾项目顶层设计
2. 完成 V0.1 架构设计          ← 已完成（Baseline + Adjudication + 8 份 ADR 已在 Git）
3. 完成平台治理和安全基线
4. 完成 AI Ops 最小模型
5. 规划测试环境基础设施
6. 建设 KubeSphere Kubernetes 平台
7. 将 DA-SOC v0.1 部署到玄武云盾
8. 验证 AI 辅助运维闭环
```

**File C:** `README.md:1178-1212`（§24 第一阶段实施顺序）
```text
Project Charter → Overall Architecture → Governance → Security Baseline
→ AI Ops Model → V0.1 Implementation Plan → Infrastructure → Kubernetes
→ KubeSphere → Security Baseline → Observability → DA-SOC → AI Ops MVP
→ V0.1 Validation
```
（`Security Baseline` 出现两次；缺 Harbor、备份、恢复演练、Cutover/回滚、CLM、两清两固）

**File D:** `ARCHITECTURE-BASELINE-V0.1.md:473-491`（S0–S14）
**File E:** `TODO.md:35`（Phase 0–16 固定顺序）

**Conflict:** README §14 自述「README 是项目最高层级的总纲」，但其 §23/§24 的
项目状态与实施顺序落后于 Baseline §20 与 TODO §1：
- §23 说架构设计是"首要任务"，§23 同一节开头又说架构已 Frozen——**README 自相矛盾**；
- §24 的顺序与 Baseline S0–S14、TODO Phase 0–16 都不同，且遗漏 7 个强制环节。

**Impact:** `README.md:1006-1017`（§19）与 `:750-764`（§13）强制要求
任何 Agent "必须读取 README.md → 确认当前版本 → 读取相关架构文档"。
Agent 在 TASK-001 阶段读到的会是一份"应当先做总体架构设计"的过时指令，
可能误判阶段、重复已完成工作，或跳过 Phase 0 参数冻结直接进入安装。

**Recommended Resolution:**
1. 用 Baseline §20 S0–S14 替换 §24 的实施顺序；
2. §23「当前首要任务」改为当前真实入口：`TASK-001～005 参数冻结`；
3. 删除 §24 中重复的第二个 `Security Baseline`；
4. 在 §24 补入 Harbor、备份/恢复演练、Cutover/回退、CLM、两清两固。

**Architecture Change Required:** NO

---

#### <a id="p1-08"></a>P1-08 — RESTORE-DRILL-PLAN 引用了错误的任务编号区间，门禁语义反转

**Severity:** P1 | **Category:** C04

**File A:** `09-implementation/RESTORE-DRILL-PLAN.md:41`（§6 Exit Criteria）
```text
所有 Drill 完成、RTO/RPO 达标、证据提交 Git、Owner 签字；
任一关键恢复失败则 TASK-061～TASK-064 不通过，禁止生产切换。
```

**File B:** `TODO.md`（实际任务定义）
```text
:937  TASK-061 — 实现 CrashLoopBackOff 真实闭环          （AI Ops）
:951  TASK-062 — 验证 Agent 失败、超时和回滚             （AI Ops）
:965  TASK-063 — 完成 AI Ops 维护交接和边界确认           （AI Ops）
:981  TASK-064 — 执行 Control Plane / etcd 恢复演练        （Drill D1）
:995  TASK-065 — 执行 ClickHouse、raw 和 n8n 恢复演练      （Drill D2）
:1009 TASK-066 — 执行 Harbor、节点/磁盘和 POP3 replay 演练 （Drill D3 + D4）
```

**Conflict:** 恢复演练任务的实际编号是 **TASK-064～TASK-066**。
`RESTORE-DRILL-PLAN` 引用 `TASK-061～TASK-064`，指向 3 个 AI Ops 任务 + 1 个演练任务，
**遗漏 TASK-065 与 TASK-066**。

**Impact:** 门禁语义被反转：

- 按字面执行，"ClickHouse/raw/n8n 全量恢复演练"（D2）与
  "Harbor/Local PV/POP3 replay 演练"（D3+D4）失败**不阻止**生产切换；
- 而"Agent 失败、超时和回滚验证"（TASK-062）失败反而阻止切换。

`ADR-006:34-41` 要求 6 项必须真实验证，其中 D2/D3/D4 三项被这一引用错误排除出门禁。
叠加 P0-01（TODO 顺序），DA-SOC 生产切换实际上处于**双重失守**状态。

**Recommended Resolution:** 将 `RESTORE-DRILL-PLAN.md:41` 改为 `TASK-064～TASK-066`，
并在同一文件补一句交叉引用，说明该门禁与 `TASK-056`（Cutover Gate 评审包）的关系。

**Architecture Change Required:** NO

---

#### <a id="p1-09"></a>P1-09 — EXTERNAL-DEPENDENCIES.md 与 TODO §22 的 Blocking Task 编号多处不一致

**Severity:** P1 | **Category:** C05 / C04

**File A:** `TODO.md:1176` + `:1178-1194`（§22 External Dependencies）
```text
实施必须继续维护 `09-implementation/EXTERNAL-DEPENDENCIES.md`。
| 外部出口白名单 | … | TASK-020/050/052 | … |
| DA-SOC 镜像、workflow、SQL、render contract | … | TASK-027/042/049 | … |
| 不可变/离线备份介质 | … | TASK-038～041/064～066 | … |
| L2 审批人和生产切换窗口 | … | TASK-036/055～059/061 | … |
| 生产邮箱只读访问和生产 DingTalk 凭据 | … | TASK-052/055/058 | … |
```

**File B:** `09-implementation/EXTERNAL-DEPENDENCIES.md`
```text
:23  出口防火墙白名单        Needed Before TASK-021 | Blocking TASK-043
:28  n8n workflow source/SQL Needed Before TASK-044 | Blocking TASK-044
:29  n8n encryption key      Needed Before TASK-045 | Blocking TASK-045
:37  离线/不可变备份副本      Needed Before TASK-040 | Blocking TASK-061
:39  L2 审批人及替补          Needed Before TASK-058 | Blocking TASK-059
:30  生产邮箱访问授权        Needed Before TASK-052 | Blocking TASK-055
```

**Conflict:** 两份文件都声称是外部依赖登记，Blocking Task 编号却不同：

| 依赖 | EXTERNAL-DEPS | TODO §22 | 实际应阻塞 |
|---|---|---|---|
| 出口防火墙白名单 | TASK-043（创建 Namespace） | TASK-020/050/052 | **TASK-020**（配置 Ingress 和外部出口） |
| n8n workflow/SQL | TASK-044（部署 ClickHouse） | TASK-027/042/049 | **TASK-049**（导入 n8n workflow） |
| n8n encryption key | TASK-045（部署 render/archive） | — | **TASK-049** |
| 离线/不可变备份副本 | TASK-061（AI Ops CrashLoop） | TASK-038～041/064～066 | **TASK-064～066** |
| L2 审批人 | TASK-058/059 | TASK-036/055～059/061 | 含 **TASK-036**（EXTERNAL-DEPS 漏） |
| 生产邮箱授权 | TASK-055 | TASK-052/055/058 | 含 **TASK-052**（EXTERNAL-DEPS 漏） |

**Impact:**
- `TASK-036`（生成 Agent 最小 RBAC 和 L0/L1/L2 策略）在 EXTERNAL-DEPS 中
  没有 L2 审批人依赖 → 可能在无审批人确认的情况下推进；
- 备份不可变副本错挂到 `TASK-061`（AI Ops 任务）→ 备份门禁被弱化，
  与 RISK-011 的"不允许切换"处置相矛盾；
- 出口白名单错挂 `TASK-043` → Namespace 创建与出口白名单产生虚假依赖。

**Recommended Resolution:**
1. 以 `TODO.md §22` 为唯一 Blocking 映射源；
2. `EXTERNAL-DEPENDENCIES.md` 增加 `Blocking Task (canonical)` 列，
   由 TODO §22 生成或加 CI 一致性校验；
3. 修正上表 6 处编号。

**Architecture Change Required:** NO

---

#### <a id="p1-10"></a>P1-10 — TASK-SEC-006 ↔ TASK-067 循环依赖，且依赖图与 Dependencies 字段互相矛盾

**Severity:** P1 | **Category:** C04

**File A:** `TODO.md:1283`（TASK-SEC-006 Dependencies）
```text
- **Dependencies:** TASK-SEC-002～005、TASK-067。
```

**File B:** `TODO.md:1169`（§21 Dependency Graph）
```text
TASK-SEC-002/TASK-SEC-003/TASK-SEC-004 ─> TASK-SEC-005 ─> TASK-SEC-006
TASK-SEC-006 ─> TASK-067 ─> TASK-068
```

**File C:** `TODO.md:1036`（TASK-067 Dependencies）
```text
- **Dependencies:** TASK-056、TASK-059、TASK-063～066、TASK-CLM-006。
```
（**不含 TASK-SEC-006**）

**File D:** `TODO.md:1165`（对照：CLM 侧写法一致）
```text
TASK-CLM-006 ─> TASK-067 ─> TASK-068
```
**File E:** `TODO.md:1227`（停止条件）
```text
当 `TASK-CLM-006`、`TASK-SEC-006`、`TASK-067` 和 `TASK-068` 完成后停止本阶段工作
```

**Conflict:** TASK-SEC-006 声明依赖 TASK-067（即在 067 之后），
而依赖图与 TASK-067 的 Dependencies 都表明相反顺序（在 067 之前）→ 循环依赖。
CLM 侧（File D）写法正确，两清两固侧不一致。

**Impact:** TASK-SEC-006 无法排程：按 Dependencies 字段执行，两清两固验收发生在最终验收之后；
但 `TODO.md:1224`（DoD#13）又把两清两固 4 项 100% 覆盖率作为 V0.1 DoD 条件。
门禁不可判定。

**Recommended Resolution:**
1. 统一为 `TASK-SEC-006 → TASK-067`（与 CLM 对称）：
   - TASK-SEC-006 Dependencies 改为 `TASK-SEC-002～005`；
   - TASK-SEC-006 Dependencies 增加 `TASK-067` → 移除；
   - TASK-067 Dependencies 增加 `TASK-SEC-006`；
2. 依赖图 `TASK-SEC-006 ─> TASK-067` 保持不变即与上述一致。

**Architecture Change Required:** NO

---

#### <a id="p1-11"></a>P1-11 — 风险登记要求"独立验证集群"，与 Baseline 单集群约束冲突且无对应任务

**Severity:** P1 | **Category:** C08

**File A:** `09-implementation/IMPLEMENTATION-RISKS.md:19`（RISK-OS-001）
```text
| RISK-OS-001 | UOS 1060e 与 Kubernetes/KubeSphere 组合兼容性未知 | High |
  节点初始化、加入或 KubeSphere 安装失败 |
  预防：将组合标为 Candidate；**先完成独立验证集群** |
  监测：TASK-010/011/015 健康检查和安装报告 |
  处置：不冻结、不进入 DA-SOC 切换；回到版本冻结 | Platform |
```

**File A:** `IMPLEMENTATION-RISKS.md:21`（RISK-OS-003）
```text
预防：……在 preflight 和**独立验证集群**测试
```

**File B:** `ARCHITECTURE-BASELINE-V0.1.md:11`
```text
玄武云盾 V0.1 是一套**单站点、单 Kubernetes 集群**……5 台 VM 的 KubeSphere 私有云平台
```

**File C:** `ARCHITECTURE-ADJUDICATION-V0.1.md:441`（V0.1 Non-Goals）
```text
- 三控制面 HA……；Ceph、Longhorn、……、多集群、跨地域 DR、……
```

**File D:** `TODO.md:54`
```text
| V0.1 不引入 | `xw-opsapi`、Ceph、Longhorn、ELK、Istio/Linkerd、GPU、多集群、复杂 Policy Engine、Task CRD |
```

**File E:** `TODO.md:275`（TASK-015 Actions，注意用的是"环境"不是"集群"）
```text
先在独立验证**环境**按官方兼容路径验证 KubeSphere 4.1.x
```

**Conflict:** 风险登记的**预防措施**依赖一个 Baseline 与 TODO 都明确排除的第二集群；
TODO 中没有任何任务交付该集群；TODO 自己用的是"独立验证环境"这一不同措辞。

**Impact:**
- **若照做**：需要额外 VM 与第二个 Kubernetes 集群 → 隐性范围膨胀、
  额外成本、第二套故障域与生命周期，且与 `Baseline:571`（不得以测试环境为由取消门槛）
  的精神相悖；
- **若不做**：RISK-OS-001 与 RISK-OS-003 两条 High 风险的**预防措施落空**。
  RISK-OS-001 正是"KubeSphere 4.1.2 尚未验证"这一 Candidate 状态的根因风险——
  它是 `VERSION-MATRIX.md:22` 把 K8s/KubeSphere 组合标为 Candidate 的直接原因，
  失去缓解手段意味着 TASK-002 无法真正冻结版本。

**Recommended Resolution:** 二选一：
- **(a)** 统一措辞为"隔离验证环境"，并明确其落地形态（例如在一次性 VM 上以独立
  kubeadm 安装 K8s+KubeSphere 组合做兼容性验证后销毁；或直接依赖
  TASK-010 Preflight + TASK-011/015 安装报告 + 明确回退方案），
  并把 RISK-OS-001/003 的"监测"列改为指向已有任务；
- **(b)** 若确实需要第二个集群，作为范围例外提交 Baseline 审批并补对应 TODO 任务、
  资源与销毁策略。

**Architecture Change Required:** NO（若选 (a)；选 (b) 则需 Baseline 例外记录）

---

### P2 — 应修复，但不阻断

| ID | Category | File A | File B | Conflict | Recommended Resolution | Arch Change |
|---|---|---|---|---|---|---|
| **P2-01** | C07 | `Baseline:89,92` + `Adjudication:241`：etcd 2379/2380 仅 CP 本机/必要集群路径；KubeSphere 控制台与 n8n 管理 UI 经管理 VPN 的受控 Ingress（HTTPS） | `port-baseline.yaml:11-71`：仅登记 5 条（22 / 6443 / 443@xw-harbor-01 / 80 全关闭 / 30000-32767） | **`xw-wk-01` 的 Ingress 443 与 `xw-cp-01` 的 etcd 2379/2380 在端口基线中未登记**。`:75` `unregistered_port: REVIEW_AND_CREATE_PORT_TASK` 会把这两项判为未登记端口 | 补 `ingress-https`（xw-wk-01:443，expected open，allowed_source=management-network/bastion）与 `etcd-client`（xw-cp-01:2379/2380，allowed_source=cluster-nodes only），标注 Baseline 出处。**否则 TASK-SEC-002 的"端口覆盖率 100%"在 Baseline 合规状态下不可达成，且每 24h（`security-baseline.yaml:30`）产生 2 条固定假阳性 PORT Task** | NO |
| **P2-02** | C06/C07 | `security-baseline.yaml:6` `statuses: [PASS, FAIL, REVIEW, UNKNOWN]`；`Baseline:577` 同 | `TODO.md:1029`（TASK-067 Actions）「逐项标记 **PASS/FAIL/EXCEPTION**」 | `EXCEPTION` 不在统一状态集合中，且与 `unknown_is_pass: false` 无映射关系。TASK-067 的「任何未满足的硬约束均阻断验收」与 EXCEPTION 之间存在未定义缺口 | 统一为 `PASS/FAIL/REVIEW`；例外通过"风险等级 + Owner + 期限 + 回退计划"表达（`TODO.md:1224` 已有该表述），或在 `security-baseline.yaml` 正式加入 `EXCEPTION` 并定义其与 `REVIEW` 的关系 | NO |
| **P2-03** | C04 | `TODO.md:1030`（TASK-067 Validation）「`TODO.md` 未被修改」 | `TODO.md:65`（§3.8）「本次版本基线补充**允许修订**根目录 `TODO.md`」；git `bfd655d`「重构 V0.1 最终实施任务清单为参数冻结与分阶段任务模型」（发生在 Baseline 之后） | TODO 已被授权修订并实际修订，但 TASK-067 把它作为验收条件；"未被修改"的基准时点未定义 | 改为「自 TASK-001 起 `TODO.md` 未被修改，或每次修改均有对应 Change 记录并与 Baseline 一致」 | NO |
| **P2-04** | C04/C08 | `Baseline:473-491`（§20 S0–S14）与 `:495-530`（§21 Acceptance Criteria）**不含** CLM 与两清两固任何步骤/验收项 | `Baseline:433-437`（§16.5）与 `:573-579`（§25）已把两者写入 V0.1 范围；`TODO.md:35` 固定顺序同样不含两者 | Baseline 自称"实施唯一架构依据"，但其**实施性**章节（顺序、验收）与**能力性**增补章节不一致。TODO 的 `§20A` 排在 Phase 16 之后（`:1053`），`§20B` 更排在 Definition of Done 之后（`:1229`） | 在 Baseline §20 增补 S15（CLM 与两清两固闭环）、§21 增补对应验收条目；把 TODO 的 20A/20B 移入 Phase 序列（TASK-CLM/SEC 依赖 TASK-004/005/060，本应在 Phase 0 与 Phase 14 之后） | **YES**（§20/§21 最小增补，与 §16.5/§25 对齐） |
| **P2-05** | C05/C07 | `Baseline:243-250`（§10.2 监控对象）与 `:295-313`（§12 Security）**不引用** `04-security/*.yaml`，也不含 CLM/两清两固指标 | `security-baseline.yaml:34` 定义 8 项 metrics；`CLM/README.md:151-162` 定义 7+8 项指标与 24h/168h cadence | 四份安全基线与 CLM 的指标/周期在 Baseline 的可观测性与安全章节没有落点；`TODO.md:1188` 只把"运维 DingTalk 告警入口"挂在 TASK-032 上，无这些指标 | 在 Baseline §10.2/§10.4 增补 security/CLM 指标与 stale REVIEW 告警；或在 TODO TASK-029/032 的 Actions 中显式加入这些指标。**当前状态：两清两固覆盖率指标无采集任务** | **YES**（§10/§12 最小增补） |
| **P2-06** | C05 | `Baseline:394-427` Task YAML：`id, created_at, source, spec{target,severity,risk,evidence,impact,plan,rollback}, approval{…}, execution{…}, verification{…}, audit{…}`；`ADR-008:10` 同语义 | `security-baseline.yaml:11` `required_fields: [id, category, asset, finding, severity, evidence, expected_state, observed_state, remediation, approval_required, owner, status, verification, rollback]` | **两套必填字段集几乎不重叠**：两清两固缺 `risk`（L0/L1/L2）、`source`、`target`、`impact`、`plan`、`execution`、`audit`；AI Ops Task 缺 `category`、`expected_state`、`observed_state`、`remediation`、`owner`。`TODO.md:1270`（TASK-SEC-005）要求"复用 TASK-060～063 的 Git Task"但字段集不同 | 合并为单一 Task schema：Baseline §16.3 为基础，两清两固追加 `category/asset/finding/expected_state/observed_state/remediation/owner`，并**保留 `risk: L0\|L1\|L2` 为必填**（当前 `security-baseline.yaml:18-23` 用 `approval.L0/L1/L2` 承载，字段名与 Baseline 不一致） | NO |
| **P2-07** | C04 | `TODO.md:1238`（TASK-SEC-001 Dependencies）`TASK-004、TASK-005、TASK-CLM-001～004` | `TODO.md:1167`（§21 依赖图）`TASK-SEC-001 ─> TASK-SEC-002/003/004`（无上游） | 同一文件两处对 TASK-SEC-001 的依赖描述不一致（CLM 侧 `:1163` 有 `TASK-004/TASK-005 ─> TASK-CLM-001`） | 依赖图补 `TASK-004/TASK-005/TASK-CLM-001～004 ─> TASK-SEC-001`。否则可在 CLM Registry 尚未建立时启动两清两固基线冻结 | NO |
| **P2-08** | C05/C07 | `security-baseline.yaml:28-34` cadence（port/account/access_control 各 24h）+ `:9-17` task_model；`CLM/README.md:139` 声称它"定义统一模型" | `port-baseline.yaml:72-80`、`account-baseline.yaml:59-69`、`access-control-baseline.yaml:63-73` **各自**定义 `rules` + `remediation`；`policies.yaml:43-46` 又定义一次 `freshness`（24h/168h）+ `stale_state: REVIEW` | 四处定义 freshness/陈旧判定（数值当前一致：24/168），无引用关系；四份基线没有一份显式定义 `deviation` 数据结构（仅 `access-control-baseline.yaml:5` 的 `record_fields` 含 `deviation`） | 四份基线只保留 Desired State 与 check 定义，cadence/rules/remediation/deviation 统一由 `security-baseline.yaml` 提供，其余改为 `extends: xuanwu-two-clear-two-firm` | NO |
| **P2-09** | C06/C07 | `upgrade-rules.yaml:3-15` + `CLM/README.md:89-92` 定义 `P0–P3`（含 SLA）；`security-baseline.yaml:11` task 必填含 `severity` 但**不含 priority** | `port-baseline.yaml:10` `record_fields` 同时含 `risk_level`(critical/high) 与 `priority`(P0/P1)；`account-baseline.yaml:16-57` 九项 check **只有 `severity`，没有 priority** | 同一 finding 需要 priority（处置紧迫度）还是 severity（影响面）无规则；账号类 finding 无 priority，无法进入 CLM 的 P0–P3 SLA 体系 | 定义 `severity`（critical/high/medium/low，影响面）与 `priority`（P0–P3，紧迫度）双字段及映射表，并在 `security-baseline.yaml` `task_model.required_fields` 中同时列出。**当前两清两固 closure 与 Critical Upgrade SLA 无法跨四类能力统一度量** | NO |
| **P2-10** | C04 | `TODO.md:925` TASK-060 Preconditions 含 `TASK-059`；`:983` TASK-064 Preconditions 含 `TASK-012`；`:1011` TASK-066 Preconditions 含 `TASK-064` | `TODO.md:934` TASK-060 Dependencies 仅 `TASK-004、TASK-036、TASK-042`；`:992` TASK-064 Dependencies 仅 `TASK-039、TASK-041`；`:1020` TASK-066 Dependencies 仅 `TASK-028、TASK-050、TASK-065` | 多个任务的 `Preconditions` 与 `Dependencies` 表达的前置集合不同 | 统一以 `Dependencies` 为准；`Preconditions` 只保留非任务类前置（环境/权限/窗口）。否则按 Dependencies 排程会在切换完成前启动 Phase 14，且 TASK-064 可在 TASK-012（etcd/API 健康）未完成时开始 | NO |

---

### P3 — 文档优化

| ID | Category | 证据 | 说明 | 建议 |
|---|---|---|---|---|
| **P3-01** | C05 | `EXTERNAL-DEPENDENCIES.md:44-50` | 第 45 行空行把两清两固 5 行切到表格之外，且该段无表头 → Markdown 渲染断裂 | 去掉第 45 行空行，或为第二段补表头 |
| **P3-02** | C01 | `CLM/README.md:135-162`（§10/§11 英文）vs `Baseline:573-579`（§25 中文） | 同一模型一份英文一份中文平行表述，易产生措辞漂移（已实际发生：P1-02/P1-06） | 统一语言，或让 §10/§11 只引用 Baseline §25 |
| **P3-03** | C04 | `TODO.md:1023`(§20) → `:1053`(§20A) → `:1139`(§21) → `:1174`(§22) → `:1196`(§23) → `:1208`(§24 DoD) → `:1229`(§20B) | 章节编号乱序：`§20B` 排在 `Definition of Done` 之后 | 重排为 §20A / §20B 均在 §21 之前 |
| **P3-04** | C01 | `00-project/VISION.md`、`GOALS.md`、`SCOPE.md`、`PRINCIPLES.md`、`VERSIONING.md` 全部为「占位 · 待编写 / TODO: 描述…」 | 项目状态已是 Architecture Frozen，但项目基础文档仍为占位。`VERSIONING.md` 自述「V0.1 Planning / Architecture」，与 `README.md:1143`「Architecture Frozen / Implementation TODO Ready」状态漂移 | 补齐五份文档。**注意：`Adjudication:36` 已显式声明这是已知缺口并将其转为 Baseline/ADR 实施前置，因此不构成冲突，仅为文档债** |
| **P3-05** | C05 | `01-architecture/`、`02-governance/`、`03-platform/`、`05-operations/`、`06-runbooks/`、`08-business/`、`11-incidents/`、`12-assets/`、`configs/`、`manifests/`、`scripts/` 均只有 `.gitkeep` | Baseline `:268` 要求"架构、治理、Runbook、Task YAML、NetworkPolicy、RBAC、backup 配置、DA-SQL、workflow JSON、镜像 digest、Kubernetes manifests"进入 Git；TODO TASK-049 要求 workflow 从 Git 构建；`07-aiops/` 下也无 Task/Runbook 目录约定 | 由 TASK-004（冻结 Git SoT 与仓库布局）建立目录约定与 `.gitkeep` 语义。**TASK-004 尚未执行，因此当前不判为阻断** |
| **P3-06** | C03 | `Baseline:124`「采用 Kubeadm 或 KubeKey 的受支持安装路径，最终安装器只选一个并写入 Git」 | 安装器仍是二选一未决，而 `VERSION-MATRIX.md:68-74`（§6 Parameters Still Pending Freeze）**未把安装器列为待冻结参数**，TASK-002 也不覆盖 | 把"安装器"加入 VERSION-MATRIX §6 待冻结清单，避免在 TASK-011 临时决定 |
| **P3-07** | C06 | `components.yaml:8,28,38,48,58,68,78,88,96,106,116,126,136,146,156` | `environment` 字段两义：既表示部署目标（`da-soc`/`platform`/`kubernetes`），也表示生命周期阶段（`management`/`production`）。如 `:28` `kubesphere: [management, production]` vs `:38` `containerd: [kubernetes]` | 这是 **P1-04 的根因之一**。建议拆为 `deployment_target` 与 `lifecycle_stage` 两个字段 |
| **P3-08** | C02 | git `a5922e5`「docs: 移除 V0.1 架构决策记录（ADR-001 至 ADR-004）」 | `--name-status` 显示该提交实际**删除 8 个文件**（ADR-001～ADR-008），与 commit message 描述的 4 个不符。后续 `1b3145d` 已全部重新引入，**当前 8 份 ADR 齐备，状态无缺失** | 仅提示 commit message 不准确；当前状态无需处理 |

---

## 11. Potential False Positives

以下是我在审计中检查过、但**判定为不是真正冲突**的项目：

1. **Baseline 与 Adjudication 同时出现"权威"字样。**
   `Baseline:3`「唯一实施依据 / Approved Baseline」，`Adjudication:6`「本文件解释裁决过程；
   `ARCHITECTURE-BASELINE-V0.1.md` 是后续实施唯一架构依据」。
   → 两者已明确分工（裁决过程 vs 实施依据），**不是双 SoT**。

2. **Calico / Harbor / ClickHouse 等没有具体版本号。**
   `VERSION-MATRIX.md:24` 只写「Calico」、`:32` 写「Approved independent-VM release」，
   `components.yaml:50` `desired.version: unknown`。
   → 版本确实处于待冻结状态，标注也正确（`Pending Version Freeze`）。
   依审计规则「不要因为版本是 TBD 就直接判错」，**不算冲突**；
   仅其 token 命名不统一被记为 P1-06。

3. **`09-implementation/00-architecture-review/*.md` 中的 4 VM、ECS 平行承载、Task CRD、Promtail。**
   → 这些是候选方案输入。`Adjudication:44-48` 列出并定性，
   `:24` 明确「这些文件均被视为候选输入，不因作者或历史来源而获得默认优先级」，
   `Baseline:567` 明确「候选方案不具有实施权威性」，`:24.8` 已逐条否决。
   **当前版本无冲突。**

4. **`Adjudication:501`「不修改 `TODO.md`：下一阶段根据 Baseline 重新生成最终实施 TODO」
   与 `TODO.md:65`「允许修订根目录 `TODO.md`」。**
   → 二者可调和：前者说的是"裁决那一阶段不修改、下一阶段重新生成"，
   后者是"重新生成"这一动作本身。git `bfd655d` 正是该重新生成。
   **不判为冲突**；但由此产生的 TASK-067 验收条件歧义仍需澄清（P2-03）。

5. **`Adjudication:24-36` 把 `00-project/*.md` 列为"已读取"的正式资料，而它们是占位文件。**
   → `Adjudication:36` 已在同一段显式声明「`01-architecture/` 与 `02-governance/`
   当前没有可供读取的正式架构文档……本裁决因此将 README/TODO 与任务中明确的
   DA-SOC 约束作为上位输入」。**已如实披露，不构成冲突**（仅 P3-04 文档债）。

6. **`Baseline:11`「单集群」与 `Adjudication:19`「接受控制面非 HA」。**
   → 一致（1 CP 单实例），非冲突。`Adjudication:440` 把 3 CP 明确列为 Non-Goal。
   仅 P1-11 的"独立验证集群"是真冲突。

7. **`security-baseline.yaml:29-33` 与 `policies.yaml:43-46` 的 freshness 数值。**
   → 两者都是 24h discovery / 168h vulnerability，**数值一致**。
   重复定义被记为 P2-08（结构问题），**不是数值冲突**。

8. **`CLM/README.md:11-21` 的流程与 `Baseline:376-437`（§16 AI Ops）的流程。**
   → 二者都复用"Discovery → 判断 → Git Task → 审批 → 执行 → 验证 → 审计"，
   **无新增 Task Center、无新增 Runtime**，符合 README 预期。**不算重复定义**。

9. **`audit/` 下已有其他 Agent 的报告。**
   → 本审计**未阅读**其结论（grep 中偶然出现的片段未取用），
   以保证 5 份审计的独立性。它们的结论需在交叉比较阶段由裁决方处理。

---

## 12. No-Issue Areas

明确检查后确认一致、不需要任何修改的关键区域：

1. **物理与组件拓扑（C02）**——5 VM、单集群、1 CP + 2 Worker、Harbor 与备份仓库在集群外、
   ClickHouse 单副本固定 `xw-wk-02`、Calico、local-path。在 Baseline、Adjudication、
   8 份 ADR、TODO §2、EXTERNAL-DEPS 中逐项一致。

2. **DA-SOC 承载与单活（C02 重点项）**——V0.1 实际入仓 `da-soc`、验证双跑、生产单活、
   ECS 为短期回退源、"两个 n8n 不得同时读取生产邮箱"。**8 处独立表述完全一致**，
   且 `RISK-006` 以 Critical 级别持续监测。

3. **DA-SOC 业务不变量（C01/C02）**——SQL 出数、render 出图、`/archive` 失败熔断、
   `null`/暂无数据语义、生产邮箱不 Mark as Read/删除/修改、DingTalk 目标来自凭据、
   LLM 不进入业务链路。在 Baseline `:361-373`、Adjudication `:81-89, :342-350`、
   ADR-001 `:36,40`、CUTOVER-CHECKLIST `:24-29`、TODO DoD#5#6、RISK-005/006/007/008
   中完全一致。**未发现任何一处平台设计会破坏业务纪律。**

4. **单一观测栈（C02 重点项）**——Prometheus / Grafana / Alertmanager / Fluent Bit / Loki，
   无 ELK、无 Promtail、无第二套监控、无 SIEM。CLM 与两清两固的指标全部声明复用现有栈。

5. **L0/L1/L2 语义与权限边界（C07/C09 重点项）**——8 处定义语义一致，
   且**未发现任何绕过 L2 的路径**。Agent 的 7 项禁止项（cluster-admin / root /
   长期 kubeconfig / 任意 shell / 生产 Secret / 生产邮箱 / 生产 DingTalk 凭据）
   在 5 处独立文件中一致禁止。

6. **`UNKNOWN` 处理（C07 重点项）**——`unknown_is_pass: false`、`UNKNOWN 不得默认为 PASS`、
   `UNKNOWN 可见且不默认为 PASS`、`unknown_version` 进入高风险、`missing_evidence → UNKNOWN`
   在 6 处一致，**未发现任何将 UNKNOWN 当作 PASS/SAFE 的地方**。

7. **备份与恢复能力未被"非 HA"叙事削弱（C08 反向检查）**——etcd snapshot、
   ClickHouse 原生 BACKUP/RESTORE、raw 校验和、Harbor 全量备份、Secret 加密导出、
   离线/不可变副本、RPO 24h / RTO 4h·8h、6 项强制演练、POP3 重放兜底，全部保留。

8. **ADR 集合完整**——ADR-001～ADR-008 共 8 份齐备且均为 `Accepted`，
   与 Baseline §5.4/§7/§8/§9/§10/§11/§16 逐条对应，无孤儿 ADR、无被违反的 ADR。

9. **DA-SOC 核心纪律未被补丁破坏**——CLM 与两清两固都没有把 LLM 引入
   DA-SOC 出数/出图链路，两清两固的证据规则明确禁止邮箱正文与业务载荷
   （`security-baseline.yaml:26-27`、`CLM/README.md:122`、`account-baseline.yaml:65`）。

---

## 13. Final Verdict

### Q1 — 当前仓库是否存在阻断 V0.1 实施的 P0/P1 一致性问题？

**是。**

- **1 个 P0**：`P0-01` —— TODO 把生产切换排在恢复演练之前，
  违反 `Baseline:493` 的硬门禁"未完成 S7/S8 的恢复验证，不得进入 DA-SOC 生产切换"，
  并架空了 `RISK-011`（Critical）的"不允许切换"处置。
  叠加 `P1-08`（RESTORE-DRILL-PLAN 引用错误任务区间，遗漏 TASK-065/066），
  DA-SOC 生产切换处于双重失守。
- **11 个 P1**：其中 `P1-01`（未授权 Namespace）、`P1-04`（审批级别无缺省定义）
  涉及安全边界；`P1-02`（四套状态词表）、`P1-03`（版本双 SoT）、`P1-05`（CLM schema 不自洽）、
  `P1-06`（状态词表混用）会直接阻断 TASK-002～005 与 CLM/两清两固的实现与验证。

**但必须同时说明：以上全部是文档与计划层的一致性缺陷，不是架构缺陷。**
被审计的架构主体（拓扑、组件、边界、业务不变量、权限模型）经逐项核对是自洽的。

### Q2 — 是否建议进行一次统一修订？

**建议，且建议一次性完成，不要分批。**

理由：本次两个补丁（`695c6c7` CLM、`3151bb3` 两清两固）分别独立引入了
状态模型、Source of Token、版本词表。分批修订会导致第二次修订时又要为第一批的
"修正版"再写一次说明，形成第三套定义。一次性统一后即可冻结 Baseline。

建议修订范围（不含架构改动）：
1. 先修 `P0-01` + `P1-08`（门禁，优先级最高）。
2. 再统一三套状态词表（`P1-02`）与两套版本 SoT（`P1-03`、`P1-06`）。
3. 再统一 CLM 内部 schema（`P1-04`、`P1-05`）与 Task schema（`P2-06`）。
4. 最后对齐任务编号、依赖图与文档顺序（`P1-07`、`P1-09`、`P1-10`、`P2-02`、`P2-03`、`P2-07`、`P2-10`、`P3-01`、`P3-03`）。

### Q3 — 是否需要修改 Architecture Baseline？

**需要，但仅限 4 处最小增补，不涉及任何架构决策变更。**

| 增补位置 | 原因 | 对应发现 |
|---|---|---|
| `Baseline:142` | 把 `da-soc-restore` 补入临时 Namespace 清单（或明确 D2 在集群外执行） | P1-01 |
| `Baseline:473-491`（§20） | 补 S15（CLM 与两清两固闭环），使部署顺序覆盖 §16.5 与 §25 已写入的能力 | P2-04 |
| `Baseline:495-530`（§21） | 补 CLM / 两清两固的验收条目，使 DoD 可判定 | P2-04 |
| `Baseline:243-250`（§10.2） | 补 security/CLM 指标与 stale REVIEW 告警，使覆盖率指标有采集落点 | P2-05 |

**明确不需要修改的部分**：拓扑、组件选型、DA-SOC 承载方式、单一观测栈、
存储方案、Agent Runtime、Task 存储模型、L0/L1/L2 定义、V0.1 Scope 与 Non-Goals
——经审计全部一致且正确。

### Q4 — 是否可以直接进入 TASK-001？

**可以进入 TASK-001，但 TASK-002 之前必须先完成 3 项修复。**

- **TASK-001（建立实施参数冻结记录）**：不受本次发现阻塞，可立即开始。
  其 Objectives 是"建立唯一参数冻结工作项和责任人矩阵"，
  且明确要求"任何实现阶段发现的架构冲突必须停止并记录为 Architecture Blocker"。
- **TASK-002（冻结版本、镜像和 digest）**：**必须先解决** `P1-03`（版本双 SoT）、
  `P1-05`（CLM schema 不自洽）、`P1-06`（冻结状态词表四套）。
  否则版本会被冻结进两份互不引用的文件，
  且"是否已冻结"这一门禁无法机器判定——而这正是 TASK-002 的交付物本身。
- **TASK-003 / TASK-004 / TASK-005**：TASK-004（冻结 Git SoT 与仓库布局）
  建议在 `P1-03` 修订后执行，否则会把双 SoT 固化进仓库布局。
- **不得开始**：任何 Phase 12 之后的任务，尤其 TASK-056～059（Cutover），
  在 `P0-01` + `P1-08` 修复前不得执行。

### Q5 — 是否建议再次进行架构设计？

**不建议。**

本次审计未发现任何架构级矛盾：

- 被审计的架构主体在 Baseline / Adjudication / 8 份 ADR / VERSION-MATRIX / TODO 之间**逐项一致**；
- CLM 与两清两固这两块补丁**遵守了"复用现有平台能力、不新增大型安全基础设施"的原则**，
  未把任何被否决的能力（SIEM / SOAR / CMDB / Task CRD / 常驻高权 Agent / 第二集群 /
  分布式存储 / Service Mesh）偷渡进 V0.1；
- 全部 P0/P1 都是**定义层的一致性缺陷**（同一概念被写了两遍、词表不统一、
  门禁引用编号错误、顺序倒置），修复方式是统一表述，不是改变设计。

依审计原则「除非发现真正的架构级矛盾，否则不要建议重新做 Architecture Review」——
本次不存在架构级矛盾。**应停止架构讨论，进入一致性修订 + Implementation Preflight。**

---

**审计人：** minimax
**审计基线：** `main` @ `3151bb3`
**报告状态：** 独立审计，待与其他 Agent 报告交叉比较