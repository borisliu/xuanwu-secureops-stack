# Xuanwu SecureOps Stack V0.1
# Consistency Audit — codebuddy

> **Agent:** codebuddy（第 5 份独立审计）
> **审计对象：** GitHub 仓库 main 分支当前快照（含最近加入的 CLM 补丁与「两清两固」安全运营补丁）
> **审计性质：** READ ONLY AUDIT。本次未修改任何既有项目文件。
> **独立性声明：** 本审计未读取也未采用 `audit/` 下其他 Agent 的审计结论、`09-implementation/00-architecture-review/` 下的候选方案。跨目录关键字检索过程中工具输出曾附带其他审计文件的碎片行，本报告的**任何一条结论均未从中派生**；每条结论均附 `文件:行号` 自证。
> **裁决原则：** 不重新设计架构。若发现潜在设计问题，仅作为「潜在问题」记录，不自行修改 Kubernetes / KubeSphere / Calico / Harbor / Local PV / Observability / DA-SOC 承载 / Agent Runtime / Task Model / Backup Model / L0/L1/L2 / V0.1 Scope。

---

## 1. Executive Summary

### 总体判断

```
PASS WITH P1
```

仓库的**架构结论本身是自洽的**：5 台 VM、1 Control Plane + 2 Worker、单集群、Harbor 与备份仓库外置、local-path/Local PV、单一观测栈（Prometheus + Grafana + Alertmanager + Fluent Bit + Loki）、DA-SOC 实际承载于 `da-soc`、短生命周期 Job Agent、Git Task 无 CRD、无 `xw-opsapi`，在 README / Baseline / Adjudication / ADR-001～008 / TODO / VERSION-MATRIX / CLM / 两清两固之间方向一致。

**未发现 P0。** 未发现架构级矛盾，未发现隐性范围膨胀，未发现任何可绕过 L2 人工审批的 Agent 权限路径，`UNKNOWN ≠ PASS` 在全库一致。

但**补丁（CLM + 两清两固）与实施清单（TODO）是在既有 Baseline 之上叠加的**，叠加过程引入了若干状态模型重叠、Source of Truth 重叠、以及 1 处**真实的任务顺序风险（恢复演练排在切换之后）**。

### 发现统计

| 级别 | 数量 | 说明 |
|---|---:|---|
| **P0** | **0** | 无阻断级架构/安全/生产风险 |
| **P1** | **4** | F-01（恢复演练排在切换之后）、F-03（Git 目录布局二选一未定）、F-06（正式 Namespace 未落任务）、F-07（Task 状态枚举多套互斥） |
| **P2** | **15** | 状态令牌扩散、SoT 重叠、CLM 覆盖缺口、任务 ID 漂移、README 状态漂移等 |
| **P3** | **4** | 章节编号、占位文档、措辞链路等 |

### 一句话结论

> **可以进入 TASK-001，但必须在 Phase 0 内完成一次「一次性一致性修订」；不建议、也不需要再做一轮架构设计。**

---

## 2. Architecture Consistency

### 2.1 通过项（全库一致，附 Verified 证据）

| 架构命题 | 证据位置 | 结论 |
|---|---|---|
| 玄武云盾是平台，DA-SOC 是第一个业务应用 | Baseline `L11` / Adjudication `L12,L56` / README `§17 L940-965` / TODO `§1 L33` | ✅ 一致 |
| 单集群、单 Control Plane + 2 Worker、5 台 VM | Baseline `§2.1 L21-27` / Adjudication `L18` / TODO `§2 L41-42`、`TASK-006 L145` / EXTERNAL-DEPENDENCIES `L8` | ✅ 一致（含 VM 规格逐项对齐） |
| Harbor 在 Kubernetes 外 | Baseline `§9.1 L209` / ADR-003 `L8` / Adjudication `L115,L185` / TODO `§2 L45` | ✅ 一致 |
| 备份仓库在 Kubernetes 外 + 离线/不可变副本 | Baseline `§2.1 L27`、`§3.2 L69` / ADR-006 `L14` / TODO `TASK-038` / IMPLEMENTATION-RISKS `RISK-011` | ✅ 一致 |
| local-path / Local PV，不用 Ceph/Longhorn | Baseline `§8 L187-203` / ADR-004 `L12` / TODO `§2 L46`、`TASK-022 L377` | ✅ 一致 |
| ClickHouse 单副本 + 数据固定 `xw-wk-02` | Baseline `§3.1 L60-65`、`§8.2 L195` / ADR-004 / TODO `TASK-022 L377`、`TASK-044` | ✅ 一致 |
| 唯一观测栈 Prometheus/Grafana/Alertmanager/Fluent Bit/Loki | Baseline `§10.1 L236-241` / ADR-005 `L8` / TODO `§2 L47` / VERSION-MATRIX `L34-38` / components.yaml `L94-143` | ✅ 五处一致 |
| 无第二套监控/日志/SIEM | Baseline `L241`、`§25 L579` / ADR-005 `L27` / README `L1277` / TODO `§2 L54` | ✅ 一致；全库检索 `Promtail\|ELK\|SIEM` 仅出现在"禁用/否决/V0.3 延后"语境 |
| DA-SOC V0.1 实际运行于 `da-soc` Namespace | ADR-001 `L9` / Adjudication `L12,L148` / Baseline `§15.1 L345-359`、`§21 L512` / TODO `§1 L33`、`§2 L51`、`§24 L1214` | ✅ 一致 |
| 禁止 ECS + K8s 双 n8n 同时读生产邮箱 | Baseline `L13` / ADR-001 `L21,L36` / Adjudication `L168` / TODO `§3 Rule 3 L60`、`§24 L1220` / ROLLBACK-PLAN `L5,L29` / CUTOVER-CHECKLIST `L55` | ✅ 六处一致 |
| ECS 为短期回退源，不是最终生产架构 | ADR-001 `L9` / Baseline `L13` / TODO `§2 L52`、`TASK-059` | ✅ 一致 |
| Calico + default-deny + 显式放行 | ADR-002 / Baseline `§7 L176-183`、`§14 L328-339` / TODO `TASK-018/019` | ✅ 一致 |

### 2.2 一致性问题

#### F-06（P1）— Baseline 正式 Namespace 未落到任何实施任务

```
Issue ID: F-06
Severity: P1
Category: C02 架构一致性 / C04 TODO 一致性
```

**File A:** `10-decisions/ARCHITECTURE-BASELINE-V0.1.md`
**Line / Section:** §5.2 Kubernetes Architecture → Namespace，`L133-140`
**Current statement:**

> 正式 Namespace：
> | `kube-system` | Kubernetes、Calico、CoreDNS、local-path |
> | `kubesphere-system` 及其官方系统命名空间 | KubeSphere 控制面 |
> | `xw-platform` | 平台 Runbook、备份 Job、平台配置 |
> | `xw-observability` | Prometheus、Grafana、Alertmanager、Fluent Bit/Loki 配置 |
> | `xw-aiops` | Agent Job 模板、只读 ServiceAccount、Task 触发器 |
> | `da-soc` | DA-SOC 正式生产工作负载 |

**File B:** `TODO.md`
**Line / Section:** 全文件；`TASK-016`（`L285-297`）「创建 Workspace、Project 和资源配额」
**Current statement:**

> **Actions:** 创建 KubeSphere Workspace；创建生产 `da-soc` Project 和临时 `da-soc-validate` Project；配置 CPU/内存/PVC/对象数量配额、LimitRange 和 DA-SOC 节点选择策略。

补充事实：对 `TODO.md` 全文检索 `xw-platform` / `xw-observability` / `xw-aiops`，**命中 0 次**。`TASK-043`（`L677-689`）也仅创建 `da-soc` 与 `da-soc-validate`。

**Conflict:**
Baseline 声明 6 类正式 Namespace，其中 `xw-platform`、`xw-observability`、`xw-aiops` 是承载备份 Job、观测栈、Agent Job 模板与只读 ServiceAccount 的平台命名空间；TODO 只落地了 `da-soc` 与 `da-soc-validate`，另外三个平台命名空间既无创建任务、也无 ResourceQuota / PSA / NetworkPolicy / RBAC 归属任务，导致 Phase 7（观测）、Phase 9（备份）、Phase 14（AI Ops）缺少承载位置定义。

**Impact:**
`TASK-029`（部署 Prometheus/Grafana/Alertmanager）与 `TASK-060`（Agent Job 模板）在执行时没有 Namespace 依据，实施者可能自行放置（如置于 `default` 或 `kube-system`），直接破坏 Baseline §5.5 的配额/PSA/NetworkPolicy 边界与 access-control-baseline 的 `namespace-default-deny` 审计项（覆盖率目标 100%）。

**Recommended Resolution:**
在 Phase 3（`TASK-016`）或 Phase 4 前增加一条任务：创建并加固 `xw-platform` / `xw-observability` / `xw-aiops`，并明确其 quota、LimitRange、PSA、default-deny 与 Owner。不需要更改 Namespace 清单本身。

**Architecture Change Required:** **NO**

---

#### F-10（P2）— KubeSphere 组件禁用清单按 3.x 可插拔模型书写，而版本矩阵冻结 4.1.x

```
Issue ID: F-10
Severity: P2
Category: C02 架构一致性 / C03 版本一致性
```

**File A:** `10-decisions/ARCHITECTURE-BASELINE-V0.1.md`
**Line / Section:** §5.3 KubeSphere，`L149`
**Current statement:**

> 禁用 Jenkins/DevOps、Service Mesh、App Store、Multi-cluster、ES Logging 等 V0.1 非必要组件。

（同一表述复现于 Adjudication `L212`「KubeSphere 精简组件集」、`L241`、`L461`。）

**File B:** `09-implementation/VERSION-MATRIX.md`
**Line / Section:** §2 `L22`、§4 `L49-55`
**Current statement:**

> | KubeSphere | 4.1.x，优先验证 4.1.2 | Candidate / Pending Compatibility Validation | …
>
> `统信服务器操作系统 V20 1060e AMD64 + Kubernetes v1.30.6 + KubeSphere 4.1.x（优先验证 4.1.2） + containerd 1.7.x + Calico`

**File C:** `TODO.md`
**Line / Section:** `TASK-015` `L275`（及 `L274` 明确写「KubeSphere 4.1.x 候选安装包（优先验证 4.1.2）」）
**Current statement:**

> **Actions:** 先在独立验证环境按官方兼容路径验证 KubeSphere 4.1.x；仅安装 Baseline 所需管理组件；…不额外引入多集群、DevOps、Service Mesh 或扩展运行时能力。

**Conflict:**
Baseline §5.3 的合规对象是 KubeSphere **3.x 的"可插拔组件开关"（Jenkins/App Store/Logging 等内建插件）**；而 VERSION-MATRIX 与 TASK-015 冻结的是 **KubeSphere 4.1.x**，其附加能力采用 **Extensions（扩展市场）模型**，不再是同一套开关语义。两条约束描述的是不同机制，但 TODO `TASK-015` 的验收写的是「仅安装 Baseline 所需管理组件」，未说明在 4.1.x 下"组件清单"该如何表达与验证。

**Impact:**
`TASK-015` 无法字面执行：在 4.1.x 上找不到一一对应的"关闭 Jenkins/App Store/Logging"开关。若实施者按 KS 3.x 语义去核对，可能产生「因找不到对应开关而无法验证」或「反向误判为已通过」两种错误结果，导致 V0.1 非必要组件被无意引入，与管理面资源/攻击面约束冲突。

**Recommended Resolution:**
不改动架构结论（仍是"精简管理面、不含 DevOps/Mesh/App Store/ES Logging"）。建议在 Baseline §5.3 增加一句机制无关的要求：*"无论 KubeSphere 版本采用何种组件/扩展模型，V0.1 安装结果必须可被枚举并与本清单逐项比对，比对结果作为 TASK-015 证据。"* 并在 `TASK-015` 中把"组件开关"改为"安装后组件/扩展清单枚举比对"。本条需在 `TASK-002`（版本冻结）时一并核实 4.1.x 与 K8s 1.30.6 的官方兼容矩阵。

**Architecture Change Required:** **NO**

---

#### F-23（P2）— Baseline 内部对 Agent L1 边界的两种写法

```
Issue ID: F-23
Severity: P2
Category: C09 Agent / L0-L1-L2 一致性
```

**File A:** `10-decisions/ARCHITECTURE-BASELINE-V0.1.md`
**Line / Section:** §5.4 RBAC，`L161-164`
**Current statement:**

> | `agent-l1-render` | 只允许 `da-soc-render` 的限定 restart/rollout 操作 |
>
> `agent-l1-render` 不得读取 Secret、修改 RBAC/NetworkPolicy、删除 PVC、修改镜像 digest、访问节点或执行任意命令。

**File B:** 同文件 §19 Risk Levels，`L467`
**Current statement:**

> **L1**：受策略范围限制的非破坏动作，例如重启 `da-soc-render`、重启非关键平台 Pod、清理已确认的临时文件。必须自动验证，失败自动升级人工。

**Conflict:**
同一份 Baseline 内，§5.4 把 L1 的 RBAC 承载体 `agent-l1-render` 严格限定为**仅 `da-soc-render`**；§19 的 L1 定义却扩展到"重启非关键平台 Pod"和"清理已确认的临时文件"，且"非关键平台 Pod"没有任何定义或白名单来源。同文件 §12.2 `L304` 又写作"重启 `da-soc-render`、非生产临时文件清理"，三处措辞不一致。

**Impact:**
`TASK-036`（生成 Agent 最小 RBAC 和 L0/L1/L2 策略）缺少唯一依据。若按 §19 放宽，会创建超出 §5.4 设计的 L1 Role；若按 §5.4 收紧，"清理临时文件"类 Runbook 将无法归类。此为 **L1 白名单边界的定义歧义**，不构成 L2 绕过（因 §19 同句要求"必须自动验证，失败自动升级人工"），但会在 RBAC 评审（`TASK-037`）中形成争议。

**Recommended Resolution:**
在 Baseline §19 L1 增加限定语："上例动作必須逐条登记为已批准白名单 Runbook 并有自动验证与回滚；未登记动作默认 L2"。并把"非关键平台 Pod"替换为可枚举清单，或删除该短语。
**Architecture Change Required:** **NO**

---

## 3. Version Consistency

### 3.1 版本号本体一致（无误）

| 组件 | VERSION-MATRIX | components.yaml | README §23.1 | TODO |
|---|---|---|---|---|
| OS | UOS V20 1060e AMD64 `L20` | `V20 1060e` `L10` | 同 `L1151` | TASK-006/007 `L144,L156` |
| Kubernetes | v1.30.6 `L21` | `1.30.6` `L20` | 同 `L1152` | TASK-001/002 `L73,L87` |
| KubeSphere | 4.1.x / 优先 4.1.2 `L22` | `4.1.x` preferred `4.1.2` `L30` | 同 `L1153` | TASK-015 `L274` |
| containerd | 1.7.x `L23` | `1.7.x` `L40` | 同 `L1154` | TASK-009 |
| CNI | Calico `L24` | Calico（`version: unknown`）`L50` | 同 `L1155` | TASK-018 |
| n8n | 2.15.0 或批准兼容版本 `L40` | 同 `L80` | — | TASK-049 |
| render/archive | `da-soc-render:0.1` `L41` | 同 `L90` | — | TASK-042 |

未发现同一组件出现互斥版本号。

### 3.2 F-05（P2）— 状态令牌扩散，超出 VERSION-MATRIX 自身声明的三态

```
Issue ID: F-05
Severity: P2
Category: C03 版本一致性
```

**File A:** `09-implementation/VERSION-MATRIX.md`
**Line / Section:** §1 Version Freeze Rules，`L10`
**Current statement:**

> 本文件区分 `Candidate`、`Pending Compatibility Validation` 和 `Frozen`；候选版本不得描述为官方认证组合。

**File B:** 同一文件 §2 / §3
**Line / Section:** `L20-24`、`L30-43`、`L40-41`
**Current statement:**

> | Kernel | … | **Pending Version Freeze** | `L30`
> | Harbor | … | **Pending Version Freeze** | `L32`
> | Ingress Controller | … | **Pending Version Freeze** | `L33`
> | n8n | … | **Candidate / Verify** | `L40`
> | render/archive | … | **Candidate / Verify** | `L41`

**File C:** `07-aiops/component-lifecycle/components.yaml`
**Line / Section:** 全文件 `components[]`
**Current statement:**

> `desired: { version: "1.30.6", status: candidate, … }` `L20`
> `desired: { version: unknown, status: pending-freeze }` `L60`（Harbor / ClickHouse / Prometheus / Grafana / Alertmanager / Fluent Bit / Loki / image.da-soc / image.agent 同形）
> `lifecycle: { support_status: pending-validation, … }` 全部 16 个组件
> `upgrade: { approval_required: true, status: planned }` 全部 16 个组件

**File D:** `TODO.md` §0 Version Baseline Status `L18-26`
**Current statement:**

> | Architecture | **FROZEN** |
> | Version Matrix | **Pending Freeze** |
> | Installation | NOT STARTED |

**Conflict:**
VERSION-MATRIX 自声明只使用 3 个状态词，实际文件内出现 `Candidate / Pending Compatibility Validation`、`Pending Version Freeze`、`Candidate / Verify` 三种复合写法；components.yaml 另有 `candidate`、`pending-freeze`、`pending-validation`、`planned` 四种小写令牌。两文件对同一批组件（如 Harbor）标注不同令牌字符串（`Pending Version Freeze` vs `pending-freeze`），且无任何映射表。同时 TODO 用大写 `FROZEN` 描述**架构**，而 VERSION-MATRIX 用 `Frozen` 描述**版本**，同一单词在两个文件指向不同对象。

**Impact:**
四条 IUDFT (`Upgrade Closure Rate` / `EOL Component Count` / `Component Coverage` / `Discovery Freshness`) 与 `TASK-002` 的 DoD（"`VERSION-MATRIX.md` 中所有安装相关参数均为 Frozen，或有明确 Pending/Blocked 状态"）依赖可枚举状态。没有统一词表时，自动化校验脚本与 Agent 判断无法编写，"未经组合验证不得宣称官方认证组合"（`VERSION-MATRIX L47`、`README L1147`）也无法被机器检查。

**Recommended Resolution:**
新增一张状态字典（建议放在 VERSION-MATRIX §1 或 CLM README §4）：`candidate | pending-freeze | frozen | verify`（版本维度）与 `planned | discovered | assessed | upgrade-required | task-created | approved | executing | verifying | closed | rolled-back | review`（升级维度），并声明 VERSION-MATRIX 为**版本值**权威、components.yaml 为**组件身份与策略**权威。不改动任何已定版本号。
**Architecture Change Required:** **NO**

---

### 3.3 F-09（P2）— RPO/RTO 在 Baseline/ADR-006 已定，在 VERSION-MATRIX 仍列为待冻结

```
Issue ID: F-09
Severity: P2
Category: C03 版本一致性 / C05 Source of Truth
```

**File A:** `10-decisions/ARCHITECTURE-BASELINE-V0.1.md`
**Line / Section:** §11.3 目标与演练，`L290-292`
**Current statement:**

> - RPO：DA-SOC 数据不超过 24 小时；配置变更即时进入 Git。
> - RTO：普通 Pod/节点级故障分钟级；Control Plane 恢复不超过 4 小时；整套 DA-SOC 恢复不超过 8 小时。
> - 保留至少 7 个日备、4 个周备和 1 个离线/不可变副本。

**File B:** `10-decisions/ADR/ADR-006-backup.md`
**Line / Section:** `L30`
**Current statement:**

> RPO 为 24 小时以内；普通 Pod/节点故障分钟级恢复；Control Plane 恢复目标 4 小时；完整 DA-SOC 恢复目标 8 小时。**具体容量和保留策略必须在实施参数中冻结。**

**File C:** `09-implementation/VERSION-MATRIX.md`
**Line / Section:** §6 Parameters Still Pending Freeze，`L74`
**Current statement:**

> - Secret management implementation、backup target、**RPO/RTO**。

**Conflict:**
Baseline §11.3 与 ADR-006 已给出 RPO/RTO 的确定数值（且 ADR-006 明确把"待冻结"限定为**容量与保留策略**）；VERSION-MATRIX §6 却把 `RPO/RTO` 整体列入"仍待冻结"。三处对同一参数的决策状态不一致。

**Impact:**
`TASK-003`（Secret 与恢复方案冻结）与 `TASK-038`（初始化备份仓库）的验收口径分裂：按 Baseline 应作为**达标阈值**核验，按 VERSION-MATRIX 则仍属**待定参数**可重新协商。这会直接影响 `TASK-064/065` 的"达到批准 RTO/RPO"验收（`L986`）与 CUTOVER-CHECKLIST 的切换门禁。

**Recommended Resolution:**
把 VERSION-MATRIX §6 的 `RPO/RTO` 删除（保留 `backup target` 与容量/保留策略），并在该处引用 Baseline §11.3 作为取值来源。
**Architecture Change Required:** **NO**

---

### 3.4 F-21（P3）— 半开区间版本号与"禁止未锁定版本"的执行歧义

```
Issue ID: F-21
Severity: P3
Category: C03 版本一致性
```

**File A:** `TODO.md` §3 Implementation Rules，`L58`
**Current statement:**

> 所有版本、镜像、digest、证书和变更记录进入 Git；禁止 `latest`。

**File B:** `07-aiops/component-lifecycle/components.yaml`
**Line / Section:** `L30`、`L40`、`L50` 等
**Current statement:**

> `desired: { version: "4.1.x", preferred_version: "4.1.2", status: candidate, … }`
> `desired: { version: "1.7.x", status: candidate }`
> `desired: { version: "2.15.0 or approved existing version", status: candidate }`

**Conflict:**
非冲突，而是**歧义**：期望版本使用 `x` 通配符与"or approved existing version"自然语言值，与"所有版本必须精确冻结"的执行规则之间存在解释空间。当前所有条目状态均为 `candidate`，符合 VERSION-MATRIX `L14` 的「`TASK-001`～`TASK-005` 完成前不得安装对应组件；候选状态不等于安装放行」，因此**不构成违规**。

**Impact:**
`TASK-002` 若未在 DoD 中明确"所有 `x` 通配符必须在冻结时被精确补丁号替换"，可能出现部分组件冻结后仍带通配的环境。

**Recommended Resolution:**
在 `TASK-002` DoD 增加一句：`VERSION-MATRIX` 与 `components.yaml` 中不得残留 `*`、`x` 或 "or …" 形式的期望版本。
**Architecture Change Required:** **NO**

---

## 4. TODO Consistency

### 4.1 F-01（P1，本次最重要的一条）— 恢复演练被排在 DA-SOC 生产切换之后

```
Issue ID: F-01
Severity: P1
Category: C02 架构一致性 / C04 TODO 一致性 / C08 范围与可靠性边界
```

**File A:** `10-decisions/ARCHITECTURE-BASELINE-V0.1.md`
**Line / Section:** §20 Deployment Sequence，`L475-491`，以及紧随其后的强制性规则 `L493`
**Current statement:**

```text
S7  etcd/Git/ClickHouse/raw/n8n/Harbor/Secret 备份
S8  恢复演练：控制面、资源、ClickHouse、Harbor、raw
S9  da-soc-validate 回放验证
S10 da-soc 正式资源创建、数据导入、workflow 导入
S11 Cutover Gate 评审，停止 ECS n8n，启动 K8s n8n
S12 观察一个完整日报周期，确认回退路径
S13 Agent Job + Git Task + CrashLoop/Disk Pressure 闭环
S14 安全测试、故障演练、Baseline/Runtime 对照验收
```

> **`L493`：未完成 S7/S8 的恢复验证，不得进入 DA-SOC 生产切换。**

**File B:** `TODO.md`
**Line / Section:** §1 Implementation Overview，`L35`
**Current statement:**

> 实施顺序固定为：参数冻结 → 基础设施 → Kubernetes → KubeSphere → Calico/网络 → Local PV → Harbor → 观测 → 安全 → 备份 → DA-SOC 部署 → 数据迁移 → 业务验证 → **切换/回退 → AI Ops MVP → 恢复演练** → 最终验收。

同一顺序被固化在 §21 Dependency Graph `L1159-1161`：

```text
TASK-057 ─> TASK-058 ─> TASK-059
TASK-064/TASK-065/TASK-066 ─> TASK-067 ─> TASK-068
```

（`TASK-057/058/059` = Phase 13 Cutover；`TASK-064/065/066` = Phase 15 Failure / Recovery Drills。图中二者之间**无依赖边**，即 cutover 与 drills 完全解耦。）

**File C:** `09-implementation/CUTOVER-CHECKLIST.md`
**Line / Section:** §2 Pre-Cutover Gates，`L31`
**Current statement:**

> - [ ] Control Plane、ClickHouse、raw、Harbor restore drill 证据已归档。

**File D:** `09-implementation/RESTORE-DRILL-PLAN.md`
**Line / Section:** §6 Exit Criteria，`L41`
**Current statement:**

> 所有 Drill 完成、RTO/RPO 达标、证据提交 Git、Owner 签字；任一关键恢复失败则 **TASK-061～TASK-064** 不通过，**禁止生产切换**。

**Conflict:**
Baseline 的 S8（恢复演练）严格先于 S11（Cutover），并以独立规则 `L493` 明令"未完成 S7/S8 的恢复验证，不得进入 DA-SOC 生产切换"；CUTOVER-CHECKLIST 也把 restore drill 证据列为**切换前勾选项**。而 TODO §1 的总顺序与 §21 的依赖图把**恢复演练放在生产切换之后**，且依赖图未施加任何"drills 必须先于 cutover"的约束。此外 File D 的停单条件错误指向 `TASK-061～TASK-064`（实际其中 `TASK-061~063` 是 AI Ops MVP，正确的恢复演练任务是 `TASK-064/065/066`），使该依赖关系在跨文件引用时进一步失真。

**Impact:**
这是本次审计中**唯一带真实生产后果**的一致性缺陷：
1. 按 TODO 顺序执行，V0.1 完全可能在**没有任何一次真实恢复验证**的情况下完成 DA-SOC 生产切换，直接违反 Baseline `L493`；
2. 一旦切换后才发现备份不可恢复（`IMPLEMENTATION-RISKS RISK-011` 列为 Critical），回退将依赖尚未验证的 ECS 恢复路径；
3. 同时与 `ADR-006 L43`「没有恢复演练的备份不计入 Definition of Done」和 TODO 自身 `§24 DoD L1219`「真实 restore drill 通过」形成"结果要求 vs 过程顺序"自相矛盾。

**Recommended Resolution:**
调整 TODO 实施顺序为：…→ 业务验证 → **恢复演练 → 切换/回退** → AI Ops MVP → 最终验收；并在 §21 Dependency Graph 显式增加边：

```text
TASK-064/TASK-065/TASK-066 ─> TASK-057
```

同时把 `RESTORE-DRILL-PLAN.md §6` 的 `TASK-061～TASK-064` 更正为 `TASK-064～TASK-066`。
此为任务顺序修订，**不改变架构，不新增/删减组件**。
**Architecture Change Required:** **NO**

---

### 4.2 F-17（P2）— TODO 内部任务元数据自相矛盾（Preconditions / Dependencies / 依赖图）

```
Issue ID: F-17
Severity: P2
Category: C04 TODO 一致性
```

**File A:** `TODO.md`
**Line / Section:** `TASK-060` `L925`（Preconditions）、`L934`（Dependencies）
**Current statement:**

> **Preconditions:** TASK-004、TASK-036、TASK-042、**TASK-059** 完成；Agent image digest 已冻结。
> **Dependencies:** TASK-004、TASK-036、TASK-042。

**File B:** 同文件 §21 Dependency Graph，`L1160`
**Current statement:**

> `TASK-060 ─> TASK-061 ─> TASK-062 ─> TASK-063`

（`TASK-059 → TASK-060` 在图中不存在。）

**File C:** 同文件 `TASK-064` `L983`（Preconditions）/ `L992`（Dependencies）；`TASK-065` `L1006`（Dependencies）
**Current statement:**

> TASK-064 Preconditions: **TASK-012**、TASK-039、TASK-041 完成 → Dependencies: TASK-039、TASK-041
> TASK-065 Dependencies: TASK-040、TASK-047～049、**TASK-064**

而 §21 依赖图 `L1161` 把三者画成**并行**：`TASK-064/TASK-065/TASK-066 ─> TASK-067`。

**Conflict:**
同一份任务清单中，任务的 `Preconditions` 字段、`Dependencies` 字段与 §21 `Dependency Graph` 三种依赖表达方式互不覆盖且互不校验：`TASK-059`、`TASK-012`、`TASK-064` 三处依赖在图上缺失。

**Impact:**
根据 TODO `§3 Rule 4` 与依赖图进行的排程/并行判断会放行本应串行的任务；`TASK-060`（AI Ops MVP 框架）若早于 `TASK-059`（切换后观察窗口）执行，`L941` 描述的 L1/L2 场景会落在尚未稳定的生产链路上。

**Recommended Resolution:**
统一依赖来源：以 `Dependencies` 字段为唯一机器可读依据，由 One-way 校验脚本保证 `Preconditions ⊆ Dependencies` 且依赖图与 Dependencies 一致。修一处即可，无需重排整体顺序。
**Architecture Change Required:** **NO**

---

### 4.3 F-13（P2）— 实施附件的任务号与现行任务清单脱钩

```
Issue ID: F-13
Severity: P2
Category: C04 TODO 一致性
```

**File A:** `09-implementation/RESTORE-DRILL-PLAN.md`
**Line / Section:** §6 Exit Criteria，`L41`
**Current statement:**

> 任一关键恢复失败则 **TASK-061～TASK-064** 不通过，禁止生产切换。

**File B:** `TODO.md`
**Line / Section:** §18 Phase 14 `L921`；§19 Phase 15 `L979-1010`
**Current statement:**

> ## 18. Phase 14 — AI Ops MVP … `### TASK-061 — 实现 CrashLoopBackOff 真实闭环` `L937`
> ## 19. Phase 15 — Failure / Recovery Drills … `### TASK-064 — 执行 Control Plane / etcd 恢复演练` `L981`
> `### TASK-065 — 执行 ClickHouse、raw 和 n8n 恢复演练` `L995`
> `### TASK-066 — 执行 Harbor、节点/磁盘和 POP3 replay 演练` `L1009`

**Conflict:**
`RESTORE-DRILL-PLAN.md` 的 D1–D4 演练（`L9-37`）对应现行 `TASK-064/065/066`，但其门禁条款写的是 `TASK-061～TASK-064`，跨越了 Phase 14（AI Ops）与 Phase 15（Drills）两个不同领域，且漏掉了 `TASK-065/066` 所覆盖的 ClickHouse/raw/n8n/Harbor/POP3 重放。该文件写于 TODO 重编号之前，未同步。

**Impact:**
演练失败时"停止哪些任务、阻断什么"边界错误：按字面执行会错误阻断 AI Ops MVP 而放行两个未通过的恢复演练（ClickHouse 与 Harbor/POP3）。

**Recommended Resolution:**
更正为 `TASK-064～TASK-066`，并补一句"任一失败即阻止 `TASK-057` 切换"（与 F-01 一并修）。
**Architecture Change Required:** **NO**

---

### 4.4 F-14（P2）— 外部依赖的阻塞任务在两个文件中不一致

```
Issue ID: F-14
Severity: P2
Category: C04 TODO 一致性 / C05 Source of Truth
```

**File A:** `09-implementation/EXTERNAL-DEPENDENCIES.md`
**Line / Section:** `L23`
**Current statement:**

> | 出口防火墙白名单 | Security/Network Owner | Needed Before TASK-021 | **Blocking Task TASK-043** | TBD / BLOCKING | IMAPS/POP3S/DingTalk/API |

**File B:** `TODO.md` §22 External Dependencies，`L1182`
**Current statement:**

> | 外部出口白名单 | Network/Security Owner | **TASK-020/050/052** | 连通性、解析、时间报告 →（前一列即 Blocking Tasks）… 见 `L1182` 行结构：`| 外部出口白名单 | Network/Security Owner | TASK-020/050/052 | IMAPS/POP3S/DingTalk/API 规则 |`

**File C:** `09-implementation/EXTERNAL-DEPENDENCIES.md` `L37`
**Current statement:**

> | 离线/不可变备份副本 | Backup Owner | Needed Before TASK-040 | **Blocking Task TASK-061** | TBD / BLOCKING | 不能只有一份 |

**File D:** `TODO.md` §22，`L1189`
**Current statement:**

> | 不可变/离线备份介质 | Backup Owner | **TASK-038～041/064～066** | 副本、hash、恢复记录 |

**Conflict:**
两个文件自称描述同一依赖集合（TODO `§22 L1176` 明确要求"实施必须继续维护 `09-implementation/EXTERNAL-DEPENDENCIES.md`"），但对外网出口白名单的阻塞目标分别为 `TASK-043`（创建 Namespace）与 `TASK-020/050/052`（Service/Ingress/出口、测试邮箱、凭据隔离）；对离线不可变备份副本的阻塞目标分别为 `TASK-061`（CrashLoop 闭环）与 `TASK-038～041/064～066`（备份配置与恢复演练）。

**Impact:**
`EXTERNAL-DEPENDENCIES.md` 的作用是把 BLOCKING 依赖转化为停单信号（`L4`：任何 `BLOCKING` 项必须有 Owner、截止时间、证据和替代/降级路径）。阻塞目标错位会导致：
- 出口白名单未开通时错误停掉 `TASK-043`（创建 Namespace），而非真正受阻的 `TASK-020`（配置外部出口）；
- 离线副本未到位时错误停掉 AI Ops MVP，而真正应当被阻断的**恢复演练**与**生产切换**不受约束（与 F-01 叠加放大风险）。

**Recommended Resolution:**
以 `TODO.md §22` 为权威（它是最新重编号后的产物），同步修订 `EXTERNAL-DEPENDENCIES.md` 的两行 Blocking Task 列，并在 TODO §22 中显式声明本表为该依赖集合的唯一 Source of Truth。
**Architecture Change Required:** **NO**

---

### 4.5 F-12（P2）— 两清两固任务模板缺字段，违反 TODO 自身 Rule 2

```
Issue ID: F-12
Severity: P2
Category: C04 TODO 一致性 / C07 两清两固
```

**File A:** `TODO.md` §3 Implementation Rules，`L59`
**Current statement:**

> 2. 所有任务都必须保留 Validation、Evidence、**Rollback**；没有证据不算完成。

**File B:** 同文件 §20B Two-Clear-Two-Firm Security Operations，`L1231-1285`
**Current statement:**

- `TASK-SEC-001`（`L1231-1239`）字段：Objective / Inputs / Actions / Validation / Expected Output / Approval / Dependencies / Definition of Done —— **缺 Risk、缺 Evidence、缺 Rollback**
- `TASK-SEC-002`（`L1241-1248`）：有 Rollback，**缺 Risk、缺 Evidence**
- `TASK-SEC-003`（`L1250-1257`）：有 Rollback，**缺 Risk、缺 Evidence**
- `TASK-SEC-004`（`L1259-1266`）：有 Rollback，**缺 Risk、缺 Evidence**
- `TASK-SEC-005`（`L1268-1275`）：有 Rollback，**缺 Risk、缺 Evidence**
- `TASK-SEC-006`（`L1277-1284`）：有 Rollback，**缺 Risk、缺 Evidence**

对比：`TASK-001`～`TASK-068` 均含 `Risk` / `Rollback` / `Evidence` 三段（如 `TASK-006 L148,L150,L151`）；`TASK-CLM-001`～`006` 也完整（`L1062,L1064,L1065` 等）。

**Conflict:**
新增的 SEC 系列任务使用了简化模板，不满足同一文件 §3 Rule 2 的强制性要求。

**Impact:**
`TASK-SEC-005` 与 `TASK-SEC-006` 是 DoD 第 13 条（两清两固闭环）的验收对象；缺少 Evidence 字段会让"Ticket 完成"缺少证据锚点，与 `security-baseline.yaml L12-13`（`required_fields` 含 `evidence`、`verification`、`rollback`；`evidence_rules.required_stages = [discovery, assessment, task, execution, verification, closure]`）直接脱节。

**Recommended Resolution:**
为 6 个 SEC 任务补齐 `Risk` 与 `Evidence` 字段（内容可直接引用 `04-security/*.yaml` 的 required_stages），保持与其余 68+6 个任务同构。
**Architecture Change Required:** **NO**

---

### 4.6 F-19（P3）— 章节编号失序

```
Issue ID: F-19
Severity: P3
Category: C04 TODO 一致性（文档结构）
```

**File A:** `TODO.md`
**Line / Section:** 章节序列：`## 20. Phase 16 — Final Acceptance`（`L1023`）、`## 20A. Component Lifecycle Management`（`L1053`）、`## 21. Dependency Graph`（`L1139`）、`## 22`、`## 23`、`## 24. Definition of Done`（`L1208`）、`## 20B. Two-Clear-Two-Firm Security Operations`（`L1229`）

**Current statement:** 如上，字母子节 `20A` 插在 `20` 与 `21` 之间，而 `20B` 出现在 `24` 之后、`DoD` 的停止条件（`L1227`）之后。

**Conflict / Impact:** 编号违反文档自身顺序；`DoD L1227` 的停止条件把 `TASK-SEC-006` 列为终点项，但该任务定义在此之后才出现，阅读与自动化切片会遗漏。

**Recommended Resolution:** 将 `20A` → `§19A`、`20B` → `§19B`，或统一改为 `§25` / `§26` 顺序编号（不改动任务内容）。
**Architecture Change Required:** **NO**

---

### 4.7 F-15（P2）— README 状态漂移（同一文件内自称 Frozen 又自称 Planning）

```
Issue ID: F-15
Severity: P2
Category: C01 项目目标一致性
```

**File A:** `README.md`
**Line / Section:** §23 当前项目状态，`L1143`
**Current statement:**

> **当前版本：V0.1 --- Architecture Frozen / Implementation TODO Ready**

**File B:** 同一文件，页脚 `L1269`
**Current statement:**

> **Status:** Planning / Architecture

**File C:** 同一文件，`L1165-1174`
**Current statement:**

> 当前首要任务：
> 1. 完成玄武云盾项目顶层设计
> 2. **完成 V0.1 架构设计**
> 3. 完成平台治理和安全基线
> 4. 完成 AI Ops 最小模型
> 5. **规划测试环境基础设施**
> 6. 建设 KubeSphere Kubernetes 平台
> 7. 将 DA-SOC v0.1 部署到玄武云盾
> 8. 验证 AI 辅助运维闭环

**File D:** `TODO.md` `L3,L5` 与 `§0 L18`
**Current statement:**

> **Implementation Readiness: READY**
> 当前任务清单可以从 `TASK-001` 开始执行。
> | Architecture | **FROZEN** | | Implementation TODO | **READY；从 `TASK-001` 开始** |

**Conflict:**
README 同一文件内的两处状态互斥（`Frozen` vs `Planning / Architecture`）。同时 `L1165-1174` 的"当前首要任务"仍是裁决**之前**的 8 项（含"完成 V0.1 架构设计""规划测试环境基础设施"），而这些内容在 Baseline / Adjudication / TODO 中均已完成或已拆解为 TASK-001～068。

**Impact:**
README 是 README `L815` 自称的"最高层级总纲"，也是 Agent 执行前必须首先读取的文档（README §19）；状态漂移会让后续 Agent/维护者无法判断当前所处阶段，并可能重新触发已被裁决关闭的架构讨论。

**Recommended Resolution:**
将 `L1269` 的 Status 改为 `Architecture Frozen / Implementation TODO Ready`；并把 `L1165-1174` 的"当前首要任务"替换为"当前阶段：Phase 0 参数冻结，入口 `TASK-001`"。
**Architecture Change Required:** **NO**

---

### 4.8 F-16（P2）— README 版本路线未纳入已入 V0.1 的 CLM 与两清两固，且与 ADR-003 在漏洞扫描上冲突

```
Issue ID: F-16
Severity: P2
Category: C01 项目目标一致性 / C08 范围一致性
```

**File A:** `README.md` §15 版本路线 → V0.1，`L824-843`
**Current statement:**

> ### V0.1 --- DA-SOC 承载平台
> 主要能力：Kubernetes / KubeSphere / 基础网络 / CNI / 基础 NetworkPolicy / Harbor / RBAC / 基础监控 / 基础日志 / 基础审计 / 基础备份 / 基础 AI Ops
> V0.1 不追求一次性建设完整安全体系。

**File B:** `10-decisions/ARCHITECTURE-BASELINE-V0.1.md` §16.5 `L435` 与 §25 `L573-579`
**Current statement:**

> V0.1 纳入组件生命周期管理（CLM）最小闭环 …
> V0.1 纳入四项持续安全管理能力：清高危漏洞、清高危端口、固弱账号口令、固弱访问控制。

**File C:** `TODO.md` §24 Definition of Done，`L1223-1224`
**Current statement:**

> 12. CLM 已覆盖 V0.1 组件，版本/digest 可自动发现 … Git Task/审批/验证/审计/回退闭环通过；生产自动升级保持禁用。
> 13. 两清两固已覆盖高危漏洞、高危端口、账号口令和访问控制；… 覆盖率达到 100%，UNKNOWN 可见且不默认为 PASS。

**File D:** `README.md` §15 V0.3，`L865-875`
**Current statement:**

> ### V0.3 --- 安全平台
> 增加：EDR / **镜像漏洞扫描** / 容器安全 / Runtime Security / 更完整的审计 / 安全事件管理 / SIEM 能力

**File E:** `10-decisions/ADR/ADR-003-harbor.md` `L15`
**Current statement:**

> 开启基础漏洞扫描；**扫描是 V0.1 基础门禁**，不等于完整供应链安全。

**Conflict:**
（a）README §15 的 V0.1 能力清单缺少 CLM 与两清两固，而 Baseline §16.5/§25 与 TODO DoD 第 12/13 条已把它们列为 V0.1 必成项。
（b）README §15 把"镜像漏洞扫描"列为 **V0.3 增加项**，ADR-003 明确其为 **V0.1 基础门禁**；这是一条真实的前后冲突。

**Impact:**
README 仍被后续 Agent 当作总纲读取。若以其 V0.1 清单做范围核对，会把 CLM/两清两固误判为范围外；若以其 V0.3 清单做排期，会把 ADR-003 的 V0.1 Harbor 扫描门禁推迟两个版本，与 `TASK-027`「导入并锁定 DA-SOC 镜像」及 Baseline §21 验收相关的镜像来源控制脱节。

**Recommended Resolution:**
更新 README §15：V0.1 增加"组件生命周期管理（CLM）最小闭环"与"两清两固最小可管理闭环（含 Harbor 基础漏洞扫描门禁）"；V0.3 删除"镜像漏洞扫描"（改为"完整 SBOM/供应链签名/漏洞运营平台"）。
**Architecture Change Required:** **NO**

---

### 4.9 F-20（P3）— README §24 实施顺序与 §27 闭环链路未随补丁更新

```
Issue ID: F-20
Severity: P3
Category: C01 项目目标一致性
```

**File A:** `README.md` §24 第一阶段实施顺序，`L1184-1211`
**Current statement:**

```text
Project Charter → Overall Architecture → Governance → Security Baseline
→ AI Ops Model → V0.1 Implementation Plan → Infrastructure → Kubernetes
→ KubeSphere → Security Baseline → Observability → DA-SOC → AI Ops MVP → V0.1 Validation
```

**File B:** `TODO.md` §1 `L35` 与 Phase 0～16（`L67-1051`）
**Current statement:** 参数冻结 → 基础设施 → Kubernetes → KubeSphere → Calico/网络 → Local PV → Harbor → 观测 → 安全 → 备份 → DA-SOC 部署 → 数据迁移 → 业务验证 → 切换/回退 → AI Ops MVP → 恢复演练 → 最终验收

**File C:** `README.md` §27，`L1275`
**Current statement:**

> 四项能力都必须完成：发现 → 判断 → 任务 → 整改 → 验证 → 审计。

**File D:** `ARCHITECTURE-BASELINE-V0.1.md` §25，`L577`
**Current statement:**

> 四项能力均使用 `Desired State → Observed State → Deviation → Risk → Task → **Approval** → Remediation → Verification → Audit`

**Conflict:**
（a）README §24 仍是裁决前的活动序列，且缺 Local PV / Harbor / Calico / 备份 / 切换回退 / 恢复演练等环节；（b）README §27 的六步链省略了 `Approval`，而 Baseline §25 与 `security-baseline.yaml L21-23` 均把 Approval 作为必需的独立环节（并定义 L2 requires `[human_approval, change_diff, backup_or_checkpoint, rollback_plan, audit_record]`）。

**Impact:**
纯粹文档滞后；但 §27 省略 Approval 恰与本项目最重要的"高风险操作必须人工批准"原则相左，容易被引用为"两清两固不需要审批"。

**Recommended Resolution:**
§24 改为引用 TODO Phase 0～16；§27 链路补齐 `→ Approval →`，与 Baseline §25 对齐。
**Architecture Change Required:** **NO**

---

### 4.10 F-18（P3）— README §14 声明的项目文档仍为占位，且无任务负责填充

```
Issue ID: F-18
Severity: P3
Category: C01 项目目标一致性 / C04 TODO 一致性
```

**File A:** `README.md` §14 项目目录，`L779-784`
**Current statement:**

> ├── 00-project/
> │   ├── VISION.md
> │   ├── GOALS.md
> │   ├── SCOPE.md
> │   ├── PRINCIPLES.md
> │   └── VERSIONING.md

**File B:** `00-project/VISION.md`（实际内容，全文 7 行）
**Current statement:**

> # VISION
> > 状态：占位 · 待编写
> TODO: 描述玄武云盾的愿景与长期定位。

（`GOALS.md`、`SCOPE.md`、`PRINCIPLES.md`、`VERSIONING.md` 同形态，均为占位。）

**File C:** `10-decisions/ARCHITECTURE-ADJUDICATION-V0.1.md` §2.1，`L36`
**Current statement:**

> `01-architecture/` 与 `02-governance/` 当前没有可供读取的正式架构文档，相关文件仍由占位状态开始。本裁决因此将 README/TODO 与任务中明确的 DA-SOC 约束作为上位输入，并把缺失的治理内容转化为 Baseline 和 ADR 的实施前置条件。

**Conflict:**
Adjudication 已承认治理/架构文档缺失并将其列为"实施前置条件"，但 TODO（`TASK-004 L115`）的布局清单只规划了 `architecture/`、`governance/` 等目录，**没有任何一条任务以"填充 VISION/GOALS/SCOPE/PRINCIPLES/VERSIONING 或产出 `01-architecture/`、`02-governance/` 正式文档"为目标**；`01-architecture/` 与 `02-governance/` 至今仍为 `.gitkeep`。

**Impact:**
README §13「文档即平台知识」要求 Agent 执行前读取 Architecture/Policy/Runbook/ADR；若 `00-project/*` 与 `01-architecture/`、`02-governance/` 长期为空，AI-Native 运维的"上下文读取"环节缺少事实来源，削弱 §22 成功标准中"AI 能理解平台"一项。

**Recommended Resolution:**
在 TODO 增加一条 Phase 0 任务（建议并入 `TASK-004`）：产出 `00-project/*` 五篇与 `01-architecture/`、`02-governance/` 的最小正式版本，并作为 `TASK-067` 验收输入。
**Architecture Change Required:** **NO**

---

## 5. Source of Truth Consistency

### 5.1 已明确且健康的 SoT 关系

| 类别 | Source of Truth | 证据 |
|---|---|---|
| 架构 | `ARCHITECTURE-BASELINE-V0.1.md` | `L1-8`（唯一实施依据 / Approved Baseline）、`L567`（本文件是实施唯一架构依据）、TODO `L9`、`L65` |
| 架构裁决过程解释 | `ARCHITECTURE-ADJUDICATION-V0.1.md` | `L6`（权威性低于 Baseline） |
| 单组件决策 | `10-decisions/ADR/ADR-001～008` | TODO `L10` |
| CLM 组件身份/策略/升级规则 | `07-aiops/component-lifecycle/{components,policies,upgrade-rules}.yaml` | Baseline `L435`、CLM README `L71`、TODO `L114` |
| 端口/账号/访问控制 Desired State | `04-security/{port,account,access-control}-baseline.yaml` | Baseline `L575`、README `L1273` |
| 统一安全状态模型 | `04-security/security-baseline.yaml` | Baseline `L575`、CLM README `L139` |
| 实施任务 | 根 `TODO.md` | TODO `L1225`（唯一实施任务源）、`L1285` 不在范围内 duplicating |
| 外部依赖 | `TODO.md §22` + `09-implementation/EXTERNAL-DEPENDENCIES.md` | TODO `L1176`（但见 F-14） |
| 实施风险 | `09-implementation/IMPLEMENTATION-RISKS.md` | TODO `L1198`（权威风险登记） |
| 执行方法 | Runbook / Cutover / Rollback / Restore 计划 | Baseline `L268` 引用、TODO `L128` |
| 实际运行事实 | Discovery evidence（不由人工覆盖） | CLM README `L70-72`、TODO `L1074`、`L1087` |

### 5.2 F-03（P1）— Git 目录布局有两个互斥方案

```
Issue ID: F-03
Severity: P1
Category: C05 Source of Truth 一致性
```

**File A:** `README.md` §14 项目目录，`L774-813`
**Current statement:**

```text
xuanwu-secureops-stack/
├── README.md
├── 00-project/
├── 01-architecture/
├── 02-governance/
├── 03-platform/
├── 04-security/
├── 05-operations/
├── 06-runbooks/
├── 07-aiops/
├── 08-business/
├── 09-implementation/
├── 10-decisions/
├── 11-incidents/
├── 12-assets/
├── manifests/
├── configs/
└── scripts/
```

（该结构在仓库中是**实际存在的**：`00-project/`、`01-architecture/`…`07-aiops/component-lifecycle/`、`04-security/*.yaml` 均按此编号落盘。）

**File B:** `TODO.md` `TASK-004 — 冻结 Git Source of Truth 和仓库布局`，`L115`
**Current statement:**

> **Actions:** 规划 `architecture/`、`governance/`、`security/`、`platform/`、`manifests/`、`tasks/`、`runbooks/`、`da-soc/workflows/`、`backup/`、`incidents/` 和 `07-aiops/component-lifecycle/`；配置主分支保护、双人审批、签名/审计；定义 generated artifact 与源文件关系；Secrets 仅保存加密引用。

**Conflict:**
`TASK-004` 给出的是**扁平命名**（`architecture/`、`governance/`、`security/`、`platform/`、`runbooks/`、`incidents/`），与 README §14 及仓库现状的**编号命名**（`01-architecture/`、`02-governance/`、`04-security/`、`03-platform/`、`06-runbooks/`、`11-incidents/`）指向同一组概念但路径不同；且 `TASK-004` 同时保留了带编号的 `07-aiops/component-lifecycle/`，说明该清单内部本身混合了两种命名惯例。两个文件都声称自己是仓库布局的依据，`TASK-004` 未声明是否**替代** README §14。

**Impact:**
`TASK-004` 位于 Phase 0 且是 `TASK-049/TASK-060/TASK-CLM-001/TASK-SEC-001` 的前置。若按其字面执行，会在既有编号目录旁新建一套平行目录，造成同一类内容两处存放——这将直接破坏 Baseline `§11.1 L266`「Git 是所有声明式定义的 Source of Truth」的唯一性要求，并使 `L116` 的 Validation「任一部署配置都能追溯到 Git commit」与 drift 检查失效。

**Recommended Resolution:**
在 `TASK-004` 中明确："沿用 README §14 编号目录结构；`architecture/`、`governance/`、`security/`、`platform/`、`runbooks/`、`incidents/` 为逻辑名，分别对应 `01-architecture/`、`02-governance/`、`04-security/`、`03-platform/`、`06-runbooks/`、`11-incidents/`；新增（`tasks/`、`backup/`、`da-soc/workflows/`）需在同一 PR 中补充进 README §14。" 一次性对齐，不改动实际目录。
**Architecture Change Required:** **NO**

---

### 5.3 F-02（P2）— Baseline §24 Rule 2 与实际仓库状态矛盾

```
Issue ID: F-02
Severity: P2
Category: C05 Source of Truth 一致性
```

**File A:** `10-decisions/ARCHITECTURE-BASELINE-V0.1.md` §24 Baseline Rules，`L568`
**Current statement:**

> 2. **`TODO.md` 不由本阶段修改**；最终实施 TODO 必须以本文件重新生成。

**File B:** `TODO.md` 当前实际形态，`L1`、`L8`、`L14-26`
**Current statement:**

> # 玄武云盾 V0.1 最终实施任务清单
> - 生成日期：2026-10-08
> | Implementation TODO | READY；从 `TASK-001` 开始 |

**File C:** `ARCHITECTURE-ADJUDICATION-V0.1.md` §27，`L501`
**Current statement:**

> **不修改 `TODO.md`**：下一阶段根据 Baseline 重新生成最终实施 TODO。

**File D:** `TODO.md` §3 Rule 8，`L65`
**Current statement:**

> Baseline 和 ADR 在本阶段只读；本次版本基线补充允许修订根目录 `TODO.md` 及 `09-implementation/` 的实施附件，但不得创建第二份 TODO 或修改已冻结架构结论。

**Conflict:**
Baseline Rule 2 是一条**进行时约束**（"本阶段不修改 TODO"）。当前 TODO 已被重新生成（标题、任务编号、DoD 全部重写），说明该阶段已经推进到"下一阶段"；但 Rule 2 仍以现在时存在于 Baseline 中，会成为一条**已失效但仍然生效（未修订）的规则**。

**Impact:**
后续执行者（尤其是 Agent）读取 Baseline §24 时会得到「不要修改 TODO」的指令，而实际的一致性问题恰恰需要在 TODO 上修订；这会制造「改也不是、不改也不是」的执行阻塞，同时削弱 Rule 1/3/4/5 的可信度。

**Recommended Resolution:**
将 Baseline §24 Rule 2 改写为过去/已完成时态：*"最终实施 TODO 已依据本文件重新生成于根 `TODO.md`（2026-10-08），该文件是唯一实施任务源；后续对实施任务的修订不得修改已冻结的架构结论。"*
**Architecture Change Required:** **NO**

---

### 5.4 F-04（P2）— 组件期望版本有两个 Source of Truth 且成员清单不一致

```
Issue ID: F-04
Severity: P2
Category: C05 Source of Truth 一致性 / C06 CLM 一致性
```

**File A:** `09-implementation/VERSION-MATRIX.md` §2-§3，`L18-43`
**Current statement:**

> | Component | Selected Version | Status | Freeze Condition | Owner |
> 覆盖：`Kernel`、`local-path`、`Harbor`、`Ingress Controller`、`Prometheus`、`Grafana`、`Alertmanager`、`Fluent Bit`、`Loki`、`ClickHouse`、`n8n`、`render/archive`、`Agent image`、`SOPS/age`（+ OS/K8s/KS/containerd/CNI）

**File B:** `07-aiops/component-lifecycle/components.yaml` 全文件，`L3-163`
**Current statement:**

> 16 个组件：`os.uos-server`、`platform.kubernetes`、`platform.kubesphere`、`runtime.containerd`、`network.calico`、`registry.harbor`、`data.clickhouse`、`business.n8n`、`business.render-archive`、`observability.prometheus`、`observability.grafana`、`observability.alertmanager`、`observability.fluent-bit`、`observability.loki`、`image.da-soc`、`image.agent`

**File C:** 两者的 SoT 自我声明
**Current statement:**

> VERSION-MATRIX `L6`：本矩阵管理"组件期望状态和生命周期策略"的入口声明见 CLM 三件套，但 `§2 标题 = OS and Cloud-Native Baseline`、CLM README `L69`：`09-implementation/VERSION-MATRIX.md`：批准的期望版本、候选状态和冻结条件。
> CLM README `L71`：**Git CLM YAML：组件、策略、升级规则和审批边界的 Source of Truth。**

**Conflict:**
1. **双重 SoT**：CLM README `L69` 把 VERSION-MATRIX 定义为"批准的期望版本"来源，同时 `components.yaml` 自身每个条目都携带 `desired.version` 与 `desired.status`；二者均未声明优先级。当前取值一致（见 §3.1），属**潜伏漂移**，非现行冲突。
2. **成员清单不一致**（现行差异）：
   - 只在 VERSION-MATRIX：`Kernel`（`L30`）、`local-path`（`L31`）、`Ingress Controller`（`L33`）、`SOPS/age`（`L43`）
   - 只在 components.yaml：`image.da-soc`（`L144-153`，核心业务生产镜像）
3. `TODO.md TASK-CLM-001 L1067` 的 DoD 要求"Registry 覆盖 V0.1 **所有**平台、DA-SOC、镜像和观测组件"，但上述差异使"所有"的边界不可判定。

**Impact:**
`CLM README §9` 的核心指标 `Component Coverage = 已纳入 CLM 管理的组件 / 应管理组件 = 100%` 与 `Unknown Version Count = 0` 因分母未定义而不可验收（该指标被 TODO `L111`、`SEC-006 L1280` 引用）。特别地，**`local-path` 是 DA-SOC 全部持久数据的承载面，却不在 CLM 组件清单内**；而 `SOPS/age` 与 `Ingress Controller` 作为生产路径组件同样缺失。

**Recommended Resolution:**
在 CLM README §4 增加一句优先级声明：`desired.version/status 的取值以 VERSION-MATRIX 为准，components.yaml 内的 desired 仅为缓存镜像，须通过 TASK-001/TASK-002 同步校验`；并把两组清单取并集后统一到一个集合（建议向 VERSION-MATRIX §3 增补 `image.da-soc`，向 components.yaml 增补 `local-path`、`ingress-controller`、`kernel`、`secret-encryption`），或明确声明 CLM 的"应管理组件"= Baseline §9/§10/§15 中出现的可版本化组件。
**Architecture Change Required:** **NO**

---

### 5.5 F-11（P2）— Task schema 缺少两清两固/CLM 要求的 `category` 字段

```
Issue ID: F-11
Severity: P2
Category: C05 Source of Truth 一致性 / C07 两清两固
```

**File A:** `10-decisions/ARCHITECTURE-BASELINE-V0.1.md` §16.3 Task YAML，`L394-427`
**Current statement:**

```yaml
spec:
  target: namespace/resource
  severity: low|medium|high|critical
  risk: L0|L1|L2
  evidence: [...]
  impact / plan / rollback
approval: { required, approver, decision: pending|approved|rejected }
execution: { job, action, started_at, finished_at }
verification: { status: pending|passed|failed, evidence }
audit: { agent_digest, git_commit, result: open|closed|escalated }
```

（字段全集：`id, created_at, source, target, severity, risk, evidence, impact, plan, rollback, approval, execution, verification, audit`）

**File B:** `04-security/security-baseline.yaml` `L9-11`
**Current statement:**

> ```yaml
> task_model:
>   categories: [VULNERABILITY, PORT, ACCOUNT, ACCESS_CONTROL]
>   required_fields: [id, category, asset, finding, severity, evidence, expected_state, observed_state, remediation, approval_required, owner, status, verification, rollback]
> ```

**File C:** `07-aiops/component-lifecycle/README.md` `L144`
**Current statement:**

> A finding that deviates from the desired state creates a Git Task with category `VULNERABILITY`, `PORT`, `ACCOUNT` or `ACCESS_CONTROL`.

**File D:** `10-decisions/ADR/ADR-008-task-model.md` `L10`
**Current statement:**

> 每个 Task 至少包含：ID、来源、目标、严重性、风险等级、证据、影响、计划、回滚、审批、Agent Job、执行结果、验证结果、审计 commit 和最终状态。

**Conflict:**
Baseline §16.3 的权威 Task schema 与 ADR-008 的必含字段清单都**不含 `category`**；而 security-baseline.yaml 与 CLM README 把 `category`（四值枚举）规定为**安全类 Task 的必需字段**，且 `TASK-060 L927` 要求在 Task 中纳入 CLM 的 `component_id`/`CVE/KEV/EOL` 等字段，但未提及 `category`。

**Impact:**
`TASK-SEC-005`（`L1268-1275`）要求把四类 finding 接入现有 Git Task 流程；由于权威 schema 没有承载字段，实施者只能自行扩展，导致三类实体的 schema 分叉，`TASK-SEC-006 L1280`「每个样例有完整闭环证据」与 DoD 第 13 条的类别统计无法统一查询。

**Recommended Resolution:**
在 Baseline §16.3 的 `spec` 中增加一个可选但受控的 `category: VULNERABILITY|PORT|ACCOUNT|ACCESS_CONTROL|null`（对非安全 Task 留空），并同步更新 ADR-008 §10 的字段清单。属增量字段对齐，不改变 Task 模型决策。
**Architecture Change Required:** **NO**

---

## 6. CLM Consistency

### 6.1 概念模型完整性：通过

CLM 的**三态分离**（Component Registry / Vulnerability State / Upgrade State）在 CLM README `§2 L23-49`、`components.yaml`（每组件 `desired`/`security`/`lifecycle`/`upgrade`/`audit` 分列）、Baseline `§16.5 L435-437`、TODO `TASK-CLM-001 L1059` 四处一致；"期望态在 Git、运行态由 discovery Job 写入、不手工覆盖"（CLM README `L37,L70-72`；TODO `L1074`）未被违反。

`unknown_version ≠ 安全` 在 CLM README `L65,L83,L134`、components.yaml `L1060`（DoD 要求未知字段不被默认为安全）、`upgrade-rules.yaml L39-43`、TODO `L1074,L1081,L1087`、Baseline `L437`、security-baseline `L7`（`unknown_is_pass: false`）六处一致。

### 6.2 F-08（P2）— `upgrade.status: planned` 不在合法枚举内；`review` 同名异义

```
Issue ID: F-08
Severity: P2
Category: C06 CLM 一致性
```

**File A:** `07-aiops/component-lifecycle/components.yaml`
**Line / Section:** 全部 16 个组件，例如 `L13`、`L23`、`L33`…`L163`
**Current statement:**

> `upgrade: { approval_required: true, status: planned }`

**File B:** `07-aiops/component-lifecycle/upgrade-rules.yaml`
**Line / Section:** `lifecycle:` `L46`
**Current statement:**

> ```yaml
> lifecycle:
>   statuses: [discovered, assessed, upgrade-required, task-created, approved, executing, verifying, closed, rolled-back, review]
> ```

**File C:** 同文件 `L26-37`、`L41`
**Current statement:**

> `- id: kev / when: "kev == true" / set: { upgrade_required: review, minimum_priority: P0 }`
> `- id: not-affected / set: { upgrade_required: review }`
> `- id: unknown-version / set: { upgrade_required: review, risk: high }`

**File D:** CLM README `§2 L47`
**Current statement:**

> `upgrade: { required: ", reason: [], priority: ", target_version: ", approval_required: true, status: " }`

**Conflict:**
1. **枚举冲突**：`upgrade-rules.yaml L46` 定义了 lifecycle statuses 的闭合列表，其中没有 `planned`；但 components.yaml 中全部 16 个组件的 `upgrade.status` 均为 `planned`。二者必有一处未对齐。
2. **同名异义**：`review` 同时是 (a) `upgrade_required` 的取值之一（`false`/`true`/`review` 三态，见 File C），(b) `lifecycle.statuses` 中的状态（File B），(c) `policies.yaml L46 stale_state: REVIEW`（大写形式），(d) `security-baseline.yaml L6 statuses: [PASS, FAIL, REVIEW, UNKNOWN]` 的评估结论。四个语义轴共用同一词。
3. 字段名也存在 `upgrade_required`（CLM README/upgrade-rules）与 `upgrade.required`（File D 示例）两种写法。

**Impact:**
`TASK-CLM-004`（`L1097-1109`）要求输出 `upgrade_required`、`upgrade_reason`、`priority`、`target_version`、`approval_required`、`status`，并以 `policies.yaml`/`upgrade-rules.yaml` 为输入；若 `planned` 非法而 `review` 四义，则该 Task 的"`status` 字段"无法与 `TASK-CLM-005` 的 Task 状态、`TASK-SEC-005` 的 finding 状态对齐，`Upgrade Closure Rate` 指标失去可比口径。

**Recommended Resolution:**
(a) 把 components.yaml 的 `upgrade.status: planned` 改为 `discovered`（初始态），或把 `planned` 加入 `upgrade-rules.yaml L46` 枚举——二选一，建议前者；(b) 把 (a) 轴的 `review` 更名为 `manual-review`（保留 `review` 仅用于 lifecycle status）；(c) 统一字段名 `upgrade_required`。
**Architecture Change Required:** **NO**

---

### 6.3 F-22（P2）— CLM 策略未覆盖 `container-image` 组件类型

```
Issue ID: F-22
Severity: P2
Category: C06 CLM 一致性
```

**File A:** `07-aiops/component-lifecycle/policies.yaml`
**Line / Section:** `policies:` `L4-27`
**Current statement:**

> - `platform-core`：`component_types: [os, platform, runtime, cni, registry]`
> - `da-soc-core`：`component_types: [database, workflow, business-service]`
> - `observability`：`component_types: [observability, logging]`
> - `non-production`：`environment: [validation, test]`

**File B:** `07-aiops/component-lifecycle/components.yaml`
**Line / Section:** `L144-163`
**Current statement:**

> `- id: image.da-soc / type: container-image / owner: da-soc / environment: [da-soc] / criticality: core-business`
> `- id: image.agent / type: container-image / owner: aiops / environment: [platform] / criticality: platform-support`

**Conflict:**
16 个组件中使用了 11 种 `type`，其中 `container-image` **未被任何 policy 的 `component_types` 覆盖**；两个 image 组件的 `environment` 分别为 `[da-soc]`、`[platform]`，也不是 `[validation, test]`，因此 `non-production` policy 同样不适用。结果是：DA-SOC **生产容器镜像（core-business）没有 default_priority 与 validation_runbook**。

（缓解事实：`approval_rules L36-38` 的 `condition: "environment == production" → L2` 仍会对生产镜像升级施加人工审批，因此**不存在 L2 绕过**。）

**Impact:**
`TASK-CLM-004`（`L1101`：实现各类规则；`L1105` 需组件 Owner 批准）对 `image.da-soc` 无策略可依，只能依赖 approval_rules 的一般条款；`policies.yaml` 要求的 `validation_runbook: da-soc-upgrade-validation` 不会自动绑定给生产镜像，`TASK-CLM-006 L1129` 的"业务 golden test + 漏洞复评"缺少触发条件。

**Recommended Resolution:**
在 `policies.yaml` 增加 `container-image` 类型：建议新增 `- id: container-image`（`L2`、`validation_runbook: image-upgrade-validation`），或把它并入 `da-soc-core` 的 `component_types`。
**Architecture Change Required:** **NO**

---

### 6.4 F-07（P1）— Git Task 生命周期存在多套互斥状态枚举

```
Issue ID: F-07
Severity: P1
Category: C06 CLM 一致性 / C04 TODO 一致性 / C05 Source of Truth
```

**File A:** `10-decisions/ARCHITECTURE-BASELINE-V0.1.md` §16.3 Task YAML，`L394-427`
**Current statement:**

> `approval.decision: pending|approved|rejected`
> `verification.status: pending|passed|failed`
> `audit.result: open|closed|escalated`

**File B:** `10-decisions/ADR/ADR-008-task-model.md` `L15-16`
**Current statement:**

> ```text
> open → analyzed → planned → pending-approval → approved/rejected
> → executing → verified/failed → closed/escalated
> ```

**File C:** `TODO.md` `TASK-005 — 建立实施门禁、变更和证据目录`，`L129`
**Current statement:**

> **Actions:** 创建任务状态模型 **`PENDING/IN_PROGRESS/BLOCKED/PASSED/ROLLED_BACK`**；…

**File D:** `07-aiops/component-lifecycle/upgrade-rules.yaml` `L46`
**Current statement:**

> `statuses: [discovered, assessed, upgrade-required, task-created, approved, executing, verifying, closed, rolled-back, review]`

**File E:** `04-security/security-baseline.yaml` `L8`
**Current statement:**

> `lifecycle: [discovered, assessed, task-created, approval-pending, remediation, verifying, closed, escalated]`

**File F:** `TODO.md` `TASK-060 L928`（行为描述）
**Current statement:**

> **Validation:** L0 可自动创建只读 Job；L1 无白名单被拒；L2 无人工审批不创建/不执行；完成后 Job 清理且审计可查。

**Conflict:**
同一概念「Git Task 的生命周期状态」在五份正式文件中存在五套互不兼容的词汇：

| 语义 | Baseline §16.3 | ADR-008 | TODO TASK-005 | upgrade-rules | security-baseline |
|---|---|---|---|---|---|
| 未开始 | `open`(audit.result) | `open` | `PENDING` | `discovered` | `discovered` |
| 审批中 | `pending`(approval.decision) | `pending-approval` | — | `approved`(混用) | `approval-pending` |
| 执行中 | — | `executing` | `IN_PROGRESS` | `executing` | `remediation` |
| 成功 | `passed`(verification) | `verified` | `PASSED` | `closed` | `closed` |
| 失败 | `failed` | `failed` | — | `rolled-back` | `escalated` |
| 回退 | — | — | `ROLLED_BACK` | `rolled-back` | — |

同一 Token 亦含义不同：Baseline 的 `open` 属于 `audit.result`，而 ADR-008 的 `open` 是 Task 起始状态；`PASSED`（TODO）与 `passed`（Baseline）大小写不同且分别描述任务级与验证级结果；`approved` 在 Baseline 是审批决策，在 upgrade-rules 是 lifecycle 状态。

**Impact:**
`TASK-005` 的 DoD（`L137`）要求"后续平台任务和 CLM 任务均能引用统一 Change/Task/Evidence ID，且组件状态、升级判断和审批可审计"，`TASK-060` 的 Validation 要求"L2 无人工审批不创建/不执行"。没有统一枚举时：n8n 路由、DingTalk 通知模板、Prometheus/Graph 的 `Upgrade Closure Rate` / `remediation closure rate`、以及 `security-baseline L34` 的 `audit_completeness` 指标都无法实现一致的查询口径——这直接威胁 DoD 第 10/12/13 三条的可验收性。

**Recommended Resolution:**
在 Baseline §16.3（Task 模型权威处）增加一张**唯一权威状态表**，并令其余四文件引用它：
`open → analyzed → planned → pending-approval → approved|rejected → executing → verified|failed → closed|escalated`，另加正交字段 `rollback: none|requested|rolled-back`。
ADR-008 已给出最接近的形态，建议以其为基准。仅做词汇统一，**不改变 Task 决策（无 CRD、Git YAML/Markdown）**。
**Architecture Change Required:** **NO**

---

## 7. Two-Clear-Two-Firm Consistency

### 7.1 通过项

| 检查项 | 证据 | 结论 |
|---|---|---|
| 四项能力统一使用同一 9 步模型 | Baseline `L577`、README `L1275`、security-baseline `L139`、CLM README `L144`、port/account/access-control 三份 baseline 的 `remediation`/`verification` 结构 | ✅ 一致（README 省略 Approval，见 F-20 P3） |
| `PASS/FAIL/REVIEW/UNKNOWN` 闭合 | security-baseline `L6`、CLM README `L144`、README `L1275`、Baseline `L577` | ✅ 一致 |
| `UNKNOWN ≠ PASS/SAFE` | security-baseline `L7 unknown_is_pass: false`、CLM README `L144 "UNKNOWN is never treated as safe"`、README `L1275`、TODO `L1235,L1280`、Baseline `L577` | ✅ **五处一致，未发现任何把 UNKNOWN 处理为 PASS/SAFE 的位置** |
| 强制 FRESH → REVIEW | security-baseline `L33 stale_status: REVIEW`；policies.yaml `L46 stale_state: REVIEW`；CLM README `L130` | ✅ 一致 |
| 发现新鲜度口径 | CLM `discovery 24h / vulnerability 168h`（policies `L43-45`）vs 安全基线 `port 24h / account 24h / access-control 24h / vulnerability 168h`（security-baseline `L28-32`） | ✅ 一致 |
| L0/L1/L2 定义 | security-baseline `L18-23`、`remdiation` 三份 baseline、CLM README `L94-98`、Baseline `§12.2 L303-305`、`§19 L461-471`、ADR-007 `L10` | ✅ 一致（措辞粒度差异见 F-23 P2） |
| 不引入新安全基础设施 | Baseline `L579`、README `L1277`、CLM README `L113-118`、TODO `L1278` | ✅ 一致；全库无 SIEM/SOAR/CMDB/NDR/独立漏洞平台/新 Task Center/第二套监控的实施项 |
| 复用现有能力 | Baseline `L579` 列举"复用现有 Kubernetes、KubeSphere、Calico、Harbor、Prometheus/Grafana/Alertmanager、Fluent Bit/Loki、Git 和短生命周期 Agent Job"；`TASK-SEC-005 L1270` 明确"复用 TASK-060～063 的 Agent Job、Task YAML、L0/L1/L2、DingTalk 通知和审计；漏洞复用 TASK-CLM-003～006" | ✅ 一致；**未创建第二套 Task 模型执行体**（尽管存在词汇冲突，见 F-07） |
| 敏感信息禁止入证据 | security-baseline `L24-27`、CLM README `L145`、account-baseline `L11 secret_values: never_collect`、`L65 never_store_password/never_log_password/never_test_against_production_mailbox`、TODO `L61,L1235,L1257` | ✅ 一致 |
| 不创建新 SIEM/SOAR/CMDB/AMS/Vulnerability Platform | CLM README `L115-116`、Baseline `L579`、README `L1277`、TODO `L1278,L1282` | ✅ 一致 |

### 7.2 未发现的问题（明确记录）

未发现两清两固补丁创建第二套 Task 模型、第二套状态树的**执行实体**：四类 finding 全部收敛到同一个 Git Task（security-baseline `L11 required_fields` 与 Baseline §16.3 schema 字段高度重叠），差异仅集中在 vocabulary 层（F-07、F-11），可在一次性文本修订内解决。

### 7.3 本类别下的问题

- **F-11（P2）**：Task schema 缺 `category` 字段 → 见 §5.5
- **F-12（P2）**：SEC 任务模板缺 `Risk`/`Evidence` → 见 §4.5
- **F-07（P1）**：状态枚举互斥 → 见 §6.4
- **F-20（P3）**：README §27 链路省略 Approval → 见 §4.9

---

## 8. V0.1 Scope / HA Consistency

### 8.1 隐性范围膨胀检查：**全部为阴性**

逐项对 C08 清单做全库扫描（排除 `09-implementation/00-architecture-review/`，依据 Baseline `L567`「候选方案不具有实施权威性」）：

| 被禁/延后项 | 出现位置 | 判定 |
|---|---|---|
| 多 Control Plane | Baseline `L559`（V0.2）、Adjudication `L448`（V0.2）、`L463`（被否决清单 8） | ✅ 仅 V0.2/否决语境；TODO `TASK-011` 明确"单 Control Plane" |
| 多集群 | Adjudication `L442` Non-Goals、Baseline `L149` 禁用 Multi-cluster、TODO `L54` | ✅ 无实施项 |
| 跨地域 DR | Adjudication `L441` Non-Goals、Baseline `L562`（V1.0） | ✅ |
| Ceph / Longhorn | Baseline `L203`、ADR-004 `L12`、TODO `L54,L377`、Adjudication `L441` | ✅ 四处"不采用" |
| Service Mesh | Baseline `L149,L183`、ADR-002、Adjudication `L441`、TODO `L54,L275` | ✅ |
| SIEM / SOAR / CMDB / NDR / 完整 IAM | Baseline `L579`、README `L1277`、CLM README `L116`、ADR-005 `L27` | ✅ |
| GPU | Adjudication `L441`、TODO `L54` | ✅ |
| 复杂 Policy Engine | Adjudication `L442`、TODO `L54` | ✅ |
| Task CRD | ADR-008 `L8`、Adjudication `L118,L459`、TODO `L50,L54,L969` | ✅ 四处禁止；`L1043` 明确放入 V0.2 backlog 不实施 |
| 常驻高权限 Agent | ADR-007 `L8`、Baseline `L15,L386`、TODO `L49,L54,L930,L969` | ✅ |
| `xw-opsapi` | Baseline `L15,L390`、ADR-007 `L14`、Adjudication `L306-315,L460`、TODO `L54,L969,L1043` | ✅ |
| 自动 Kubernetes 升级 / 自动 OS 升级 / 自动生产 Patch | `upgrade-rules.yaml L47-53 forbidden_automatic_actions`、`L117-118`、Baseline `L437`、TODO `L1129,L1137`、README `L1163` | ✅ 六处一致禁止 |
| 自动 OS 升级 | `upgrade-rules.yaml L51 forbidden_automatic_actions: os_upgrade`、CLM README `L117` | ✅ |
| RBAC/NetworkPolicy 自动变更 | `upgrade-rules.yaml L53 forbidden_automatic_actions: rbac_or_network_policy_change`、Baseline `L305,L471`（L2）、security-baseline `L22` | ✅ |

**结论：未发现任何文档把上述能力偷偷变成 V0.1 必须实施内容。**

### 8.2 反向检查：是否因"非 HA"过度删减了可靠性能力

**未发现删减。** 相反，可靠性要求是充分的：

- 备份：Baseline `§11.2 L276-286` 9 类对象；ADR-006 `L16-26`；TODO Phase 9 `TASK-038～041`
- 恢复演练：Baseline `§11.3 L293` 6 项强制；ADR-006 `L32-43` 6 项；RESTORE-DRILL-PLAN `D1-D4`；TODO `TASK-064～066`
- 故障演练：TODO `TASK-021`（网络）、`TASK-024`（存储）、`TASK-061～062`（Agent）
- 离线/不可变副本：Baseline `L292`、ADR-006 `L14`、TODO `TASK-038`

**但存在一个削弱：** 这些能力虽然齐备，其在实施序列中的位置错误（见 **F-01**），使得可靠性在实际执行路径上被推迟到切换之后。这是本次唯一影响 C08 实质结论的问题。

### 8.3 本类别下的问题

- **F-01（P1）**：恢复演练排在切换之后 → 见 §4.1
- **F-16（P2）**：README §15 V0.1 清单缺失 CLM/两清两固，V0.3 错误包含 V0.1 已有的镜像漏洞扫描 → 见 §4.8
- **F-18（P3）**：治理/架构文档长期占位，AI-Native 上下文缺来源 → 见 §4.10

---

## 9. Agent / L0-L1-L2 Consistency

### 9.1 L0 / L1 / L2 定义一致性

| 源 | L0 | L1 | L2 |
|---|---|---|---|
| Baseline §12.2 `L303-305` | 只读检查、报告、查询、告警聚合 | 白名单 Runbook（`da-soc-render` 重启、非生产临时文件清理），须前置条件与自动验证 | RBAC、NetworkPolicy、CNI、节点、存储、生产 n8n、ClickHouse 数据删除/恢复、凭据和镜像策略变更；人工审批 |
| Baseline §19 `L463-471` | 只读状态、健康检查、日志/指标查询、备份年龄、证书检查、日报和 Task 汇总 | 非破坏动作（含"重启非关键平台 Pod"）→ **见 F-23** | 同上，须短时授权并 diff/影响/回滚 |
| ADR-007 `L10` | 默认只读 | 明确白名单动作 | 人工审批后短时授权 |
| CLM README `L96-98` | 自动发现、漏洞/EOL 判断、报告、Git Task 创建 | 验证环境 Patch、非生产组件或**可回滚白名单 Runbook** | K8s/KS/Calico/Harbor/OS/CH/生产镜像/节点/存储/网络/RBAC 必须人工 |
| security-baseline `L18-23` | discover/assess/report/create_task/notify | `non_production_remediation`, **`approved_low_risk_runbook`** | 生产账号/RBAC/cluster-admin/核心 firewall/Calico NP/K8s API/Harbor admin/DA-SOC 访问控制/核心业务端口/OS 或平台升级 |
| 三份 remediation baseline | audit/report/create_task | 非生产低风险 + 已批准 Runbook | 生产 RBAC/核心 NP/firewall/Secret 边界/账号策略/凭证轮换/cluster-admin 绑定 |
| TODO `TASK-036`、`TASK-060 L928` | L0 自动创建只读 Job | L1 无白名单被拒 | L2 无人工审批不创建/不执行 |

**判定：✅ 语义一致。** CLM 与安全基线的 `approved_low_risk_runbook` 与 Baseline 的白名单 Runbook 是同一事物，不构成冲突（见 §11 FP-3）。

### 9.2 Agent 越权路径检查：**阴性**

| 禁止项 | 证据 | 判定 |
|---|---|---|
| `cluster-admin` | Baseline `L317`（break-glass/人工/短时/双人/记录）、`L530`、security-baseline `L22`、access-control `L24-25`、TODO `L303`、ADR-007 `L22` | ✅ 无授权路径 |
| 主机 root / 任意 shell | ADR-007 `L22`、Baseline `L324`、access-control `L12-16`、TODO `L62` | ✅ |
| 长期 kubeconfig | ADR-007 `L22` | ✅（Job SA + TTL） |
| 生产 Secret 读取 | Baseline `L164`（`agent-l1-render` 不得读 Secret）、`L321-322`、security-baseline `L26`、TODO `L942` | ✅ |
| 生产邮箱凭据 | Baseline `L292`（不进 Loki/Agent 上下文）、ADR-007 `L22`、access-control `L55-58`、TODO `L1275`（"README 明示不团原则...不得进入 Agent 上下文"） | ✅ |
| DingTalk 凭据 | ADR-007 `L22`、security-baseline `L26`、TODO `L61` | ✅ |
| 修改 RBAC / NetworkPolicy / CNI / etcd / PVC / 业务数据 | Baseline `L322-324`、`L162`、TODO `L62` | ✅ |
| 修改 DA-SOC SQL / 出数 / 出图 | ADR-007 `L22`、Baseline `L373`、TODO `L62,L1216` | ✅ |
| CLM 自动生成升级绕过审批 | `upgrade-rules L47-53`、policies `L28 production_auto_upgrade:false`、TODO `L1115,L1116`、RISK-CLM-004 | ✅ 三重防护 |
| 常驻 Agent / `xw-opsapi` | ADR-007 `L8,L14`、TODO `L930,L969` | ✅ |
| Agent Owner 不得批准外部依赖 | EXTERNAL-DEPENDENCIES `L53`「Owner 不能填写 Agent；Agent 不能批准外部依赖、生产窗口或安全例外」 | ✅ |

**未发现任何绕过 L2 审批的路径。**

### 9.3 本类别下的问题

- **F-23（P2）**：Baseline 内部 L1 边界两种写法（`§5.4` 仅 `da-soc-render` vs `§19` 含"重启非关键平台 Pod"）→ 见 §2.2
- **F-07（P1）**：Task 状态枚举互斥影响 L2 审批在新工单上的落位（`L2 无人工审批不创建/不执行` 依赖可枚举状态）→ 见 §6.4

---

## 10. Detailed Findings

| ID | Severity | Category | File A | File B | Conflict | Recommended Resolution |
|---|---|---|---|---|---|---|
| **F-01** | **P1** | C02/C04/C08 | `ARCHITECTURE-BASELINE-V0.1.md` §20 `L475-493`（S8 先于 S11；`L493` 未完成 S7/S8 不得切换）；`CUTOVER-CHECKLIST.md L31` | `TODO.md` §1 `L35`、§21 `L1159-1161`；`RESTORE-DRILL-PLAN.md L41` | TODO 把恢复演练（064-066）排在生产切换（057-059）之后，且依赖图无约束边；与 Baseline 强制规则和切换前勾选项冲突 | 顺序改为「恢复演练 → 切换/回退」；依赖图加边 `064/065/066 ─> 057`；修正 Drill Plan 的 `TASK-061~064` → `064~066` |
| **F-03** | **P1** | C05 | `README.md` §14 `L774-813`（编号目录，且已落盘） | `TODO.md` `TASK-004 L115`（扁平目录 `architecture/`、`governance/`…） | 两份仓库布局依据互斥，未声明优先级；TASK-004 是 4 个后续任务的前置 | 在 TASK-004 声明沿用编号目录并给出逻辑名↔编号映射；新增目录同步回 README §14 |
| **F-06** | **P1** | C02/C04 | `ARCHITECTURE-BASELINE-V0.1.md` §5.2 `L133-140`（含 `xw-platform`/`xw-observability`/`xw-aiops`） | `TODO.md` 全文（`xw-platform`/`xw-observability`/`xw-aiops` 命中 0）；`TASK-016 L289` 仅建 `da-soc`/`da-soc-validate` | 三个正式平台 Namespace 无创建/加固任务，Phase 7/9/14 缺承载位置 | 在 Phase 3/4 增加创建并加固三 Namespace 的任务（含 Quota/LimitRange/PSA/default-deny/Owner） |
| **F-07** | **P1** | C05/C06/C04 | Baseline §16.3 `L394-427`（`pending/passed/open`） | ADR-008 `L15-16`；`TODO.md L129`（`PENDING/IN_PROGRESS/BLOCKED/PASSED/ROLLED_BACK`）；`upgrade-rules.yaml L46`；`security-baseline.yaml L8` | 同一 Git Task 生命周期存在 5 套互斥词汇；同形 Token 语义不同 | Baseline §16.3 增加唯一权威状态表（以 ADR-008 为基准），其余四文件引用 |
| **F-02** | P2 | C05 | Baseline §24 Rule 2 `L568`（"TODO.md 不由本阶段修改"） | `TODO.md L1,L8,L14-26`（已重新生成并自称 READY）；`TODO §3 Rule 8 L65` | 已失效的现在时规则仍以 Baseline 强制条款形式存在 | 改写 Rule 2 为"已重新生成 + 唯一实施源 + 不得改已冻结架构结论" |
| **F-04** | P2 | C05/C06 | `VERSION-MATRIX.md` §2-§3 `L18-43`（含 Kernel/local-path/Ingress/SOPS） | `components.yaml L3-163`（含 `image.da-soc`；缺 Kernel/local-path/Ingress/SOPS） | 组件期望版本双重 SoT 无优先级；成员清单不一致，`Component Coverage` 分母不可判定 | 声明 VERSION-MATRIX 为版本值权威、components.yaml 为组件身份/策略权威；两清单取并集统一 |
| **F-05** | P2 | C03 | `VERSION-MATRIX.md L10`（自声明 3 态） | 同文件 `L20-24,L30-43,L40-41`；`components.yaml`（`candidate`/`pending-freeze`/`pending-validation`/`planned`）；`TODO L18-26`（`FROZEN`） | 至少 8 种状态令牌描述 3 种状态，无映射表；`FROZEN`（架构）与 `Frozen`（版本）同名异指 | 新增状态字典并声明 VERSION-MATRIX / CLM 的职责边界（详见 §3.2 F-05 的建议） |
| **F-08** | P2 | C06 | `components.yaml`（16 处 `upgrade.status: planned`） | `upgrade-rules.yaml L46`（statuses 枚举无 `planned`）；`L26-41`（`review` 四义）；CLM README `L47`（`upgrade.required` vs `upgrade_required`） | `planned` 越出合法枚举；`review` 同时是 upgrade_required 取值、lifecycle 状态、stale 状态、安全评估结论 | `planned`→`discovered`；`review`→`manual-review`；统一字段名 |
| **F-09** | P2 | C03/C05 | Baseline §11.3 `L290-292`；ADR-006 `L30`（RPO/RTO 已定） | `VERSION-MATRIX.md` §6 `L74`（RPO/RTO 列为 Pending Freeze） | 同一参数在 Baseline/ADR 为已决策阈值，在版本矩阵为待决策参数 | VERSION-MATRIX §6 删除 RPO/RTO（保留容量与保留策略），引用 Baseline §11.3 |
| **F-10** | P2 | C02/C03 | Baseline §5.3 `L149`（按 KS3 可插拔语义罗列禁用组件） | `VERSION-MATRIX.md L22,L49-55`；`TODO TASK-015 L274-275`（冻结 KS 4.1.x） | 组件禁用清单与 4.1.x 的扩展模型不匹配，TASK-015 无法字面验收 | Baseline §5.3 改为机制无关表述："安装结果必须可枚举并与清单逐项比对" |
| **F-11** | P2 | C05/C07 | Baseline §16.3 `L394-427`；ADR-008 `L10`（无 `category` 字段） | `security-baseline.yaml L9-11`；CLM README `L144`（`category` 为必需字段） | 权威 Task schema 缺两清两固/CLM 要求的 `category` | Baseline §16.3 `spec` 增加受控 `category` 枚举；同步 ADR-008 字段清单 |
| **F-12** | P2 | C04/C07 | `TODO.md` §3 Rule 2 `L59`（所有任务须有 Validation/Evidence/Rollback） | `TODO.md` §20B `L1231-1285`（SEC-001~006 缺 Risk/Evidence，部分缺 Rollback） | 新增补丁任务模板不满足 TODO 自身强制规则 | 为 6 个 SEC 任务补齐 Risk 与 Evidence 字段 |
| **F-13** | P2 | C04 | `RESTORE-DRILL-PLAN.md L41`（`TASK-061～TASK-064`） | `TODO.md L937,L981,L995,L1009`（演练实为 064/065/066） | 演练门禁的任务号跨领域错位，漏掉 065/066 | 更正为 `TASK-064～TASK-066`，并加"任一失败阻止 TASK-057" |
| **F-14** | P2 | C04/C05 | `EXTERNAL-DEPENDENCIES.md L23,L37`（阻塞 TASK-043 / TASK-061） | `TODO.md` §22 `L1182,L1189`（阻塞 TASK-020/050/052 / 038-041,064-066） | 同一外部依赖的阻塞目标两处不一致 | 以 TODO §22 为权威同步两份附件；明示单一 SoT |
| **F-15** | P2 | C01 | `README.md L1143`（Architecture Frozen / TODO Ready） | 同文件 `L1269`（Status: Planning / Architecture）、`L1165-1174`（首要任务仍为裁决前 8 项） | 同一总纲内状态互斥，且首要任务已过期 | 统一 Status；替换"当前首要任务"为 Phase 0 入口 |
| **F-16** | P2 | C01/C08 | `README.md` §15 `L824-843`（V0.1 无 CLM/两清两固）、`L865-875`（V0.3 含镜像漏洞扫描） | Baseline §16.5 `L435`、§25 `L573-579`；`TODO §24 L1223-1224`；ADR-003 `L15`（扫描为 V0.1 门禁） | README 版本路线落后两个补丁，且把 V0.1 已有门禁推迟到 V0.3 | 更新 README §15：V0.1 增 CLM 与两清两固（含基础扫描门禁）；V0.3 删除"镜像漏洞扫描" |
| **F-17** | P2 | C04 | `TODO.md` `TASK-060 L925`（Preconditions 含 TASK-059）、`L934`（Dependencies 无）；`TASK-064 L983/L992` | 同文件 §21 依赖图 `L1160-1161`（三者并行、`TASK-065` 依赖 `TASK-064`） | Preconditions / Dependencies / 依赖图三种表达互不校验 | 以 Dependencies 为唯一依据，补齐三处缺失依赖（并做一致性校验） |
| **F-22** | P2 | C06 | `policies.yaml L4-27`（4 条 policy 的 component_types） | `components.yaml L144-163`（`type: container-image`） | `container-image` 无任何 policy 覆盖，DA-SOC 生产镜像缺 default_priority 与 validation_runbook（approval_rules 仍兜底 L2，无越权） | 新增或并入 `da-soc-core` 的 `container-image` policy |
| **F-23** | P2 | C09 | Baseline §5.4 `L161-164`（`agent-l1-render` 仅限 `da-soc-render`） | 同文件 §19 `L467`（L1 含"重启非关键平台 Pod"）；§12.2 `L304`（第三版措辞） | 同一 Baseline 内 L1 边界三处措辞不一致，"非关键平台 Pod"无定义 | §19 L1 增加"须登记为已批准白名单 Runbook + 自动验证 + 回滚；未登记默认 L2" |
| **F-18** | P3 | C01/C04 | `README.md` §14 `L779-784` | `00-project/*.md`（全为占位）；`01-architecture/`、`02-governance/`（仅 `.gitkeep`） | 总纲声明的治理文档为空，且无任务负责填充 | 在 TASK-004 增加"产出 00-project 五篇 + 01/02 目录最小正式版"目标 |
| **F-19** | P3 | C04 | `TODO.md` 章节序列 `L1023/1053/1139/1208/1229` | — | `20A` 插在 20 与 21 之间，`20B` 在 24 之后，DoD 停止条件早于其引用的任务 | 重编为 `19A/19B` 或顺序编号 25/26 |
| **F-20** | P3 | C01 | `README.md` §24 `L1184-1211`（裁决前序列） | `TODO.md` §1 `L35` + Phase 0-16 | README 实施顺序未更新；§27 `L1275` 六步链省略 Approval（Baseline `L577` 为九步） | §24 改为引用 TODO Phase；§27 补齐 Approval |
| **F-21** | P3 | C03 | `components.yaml L30,L40,L80`（`4.1.x`/`1.7.x`/"or approved existing version"） | `TODO.md` §3 Rule 1 `L58`（禁止 latest/未锁定版本） | 半开区间期望版本与"精确冻结"规则存在解释空间（当前状态为 candidate，不构成违规） | TASK-002 DoD 增加"冻结后不得残留 `x`/`*`/`or …` 形式" |

---

## 11. Potential False Positives

以下为审计过程中发现、但经核证**判定不是真正冲突**的问题：

| FP | 现象 | 为什么不是冲突 |
|---|---|---|
| **FP-1** | `09-implementation/00-architecture-review/` 下 5 份候选方案含 `Promtail`、`Velero`、`MinIO`、`Task CRD`、`xw-opsapi`、3/4 VM、ECS 平行承载、3 CP 等已否决内容 | Baseline `L567` Rule 1 明确「本文件是实施唯一架构依据；**候选方案不具有实施权威性**」；Adjudication `L42-50` 亦将其界定为"候选输入"。该目录不属于一致性审计范围 |
| **FP-2** | `ARCHITECTURE-ADJUDICATION-V0.1.md` §16 `L295` 首批告警 12 条，Baseline §10.4 `L258` 为 14 条（多 `MemoryHigh`、`NetworkPolicy 拒绝异常增加`） | Adjudication `L6` 自述「本文件解释裁决过程；`ARCHITECTURE-BASELINE-V0.1.md` 是后续实施唯一架构依据」。Baseline 为准，过程记录差异不构成冲突 |
| **FP-3** | CLM README `L97` 与安全基线 `L20` 把 L1 限定为"非生产/非生产组件"，而 Baseline §19 `L467` 的 L1 含"重启非关键平台 Pod"，看似允许 Agent 在生产执行 L1 | 两者可用同一事物统一：CLM README `L97` 为"验证环境 Patch、非生产组件**或可回滚白名单 Runbook**"，security-baseline `L20` 为"`non_production_remediation`, **`approved_low_risk_runbook`**"。Baseline §19 同句要求"必须自动验证，失败自动升级人工"，且 §5.4 `L161-164` 的 RBAC 承载体严格限定。**不存在绕过 L2 的通道**。（措辞歧义已另记为 F-23 P2） |
| **FP-4** | `components.yaml` 与 `VERSION-MATRIX.md` 中 Harbor、Prometheus、Loki、ClickHouse 等版本为 `unknown` / `Pending Version Freeze` | 任务明确要求"不要因为版本是 TBD 就直接判错"。且 `VERSION-MATRIX L14` 已建立保护：`TASK-001～TASK-005` 完成前不得安装；`TODO L29` 明确候选组合不代表官方认证。**（仅 RPO/RTO 一项为真问题，见 F-09）** |
| **FP-5** | `account-baseline.yaml L65` 允许 `controlled_authentication_test`，看似与"`L11 secret_values: never_collect`"、"禁止采集真实密码"矛盾 | 同行的 `controlled_authentication_test_rules: [never_store_password, never_log_password, never_test_against_production_mailbox]` 已完整约束该例外，**不构成冲突** |
| **FP-6** | `components.yaml L30` KubeSphere 期望版本 `4.1.x` 为半开区间，看似违反"禁止未锁定版本" | 该条目 `status: candidate`，受 `VERSION-MATRIX L14` 安装门禁保护；仅存在长期解释空间，已降级为 F-21 P3 |
| **FP-7** | `TODO.md §0 L18` 声明 `Architecture: FROZEN`，但 Baseline 文件本身从未出现 `FROZEN` 字样 | Baseline `L3` 的「状态：唯一实施依据 / Approved Baseline」与 Adjudication `L3`「状态：已裁决」构成冻结事实；`FROZEN` 是 TODO 对该状态的表述。虽希望有显式冻结记录（建议但非阻塞），**不构成冲突** |
| **FP-8** | `EXTERNAL-DEPENDENCIES.md` 全部 36 行状态均为 `TBD / BLOCKING` | 这是 Phase 0 之前的正确状态：`TODO §22 L1176` 已定义"未满足时只能停在对应任务，不得使用未批准替代方案"。**不是缺陷**（其阻塞目标错位才构成 F-14） |

---

## 12. No-Issue Areas

经逐项核验，以下区域**完全一致，无需修订**：

1. **V0.1 目标与平台/业务定位**：「玄武云盾是平台，DA-SOC 是第一个业务应用」在 README §17、Baseline §1、Adjudication §3.1、TODO §1 四处一致（C01 ✅）
2. **物理拓扑与 VM 规格**：5 台 VM 的 CPU/内存/磁盘在 Baseline §2.1 与 TODO `TASK-006 L145` **逐项对齐**，故障域隔离在 Baseline §2.2/§3.2、ADR-003 `L21`、TODO `L146` 一致（C02 ✅）
3. **Harbor 外置理由与恢复顺序**：ADR-003 `L19-25`（先恢复 Harbor 再恢复工作负载）、Baseline §9.3 `L221-230`、TODO `TASK-028` 一致（C02 ✅）
4. **network policy 职责划分**：Baseline §4.3 `L104`「NetworkPolicy 负责 Pod 东西向边界，出口防火墙/安全组负责南北向地址白名单」在 Adjudication `L254`、TODO `TASK-019/020/021` 一致；未出现"用 NetworkPolicy 做域名出网白名单"的错误假设（C02 ✅）
5. **观测唯一栈**：六份文件一致，且 Promtail/ELK/SIEM 的禁止在四份文件以"禁用/否决"语境出现（C02 ✅）
6. **DA-SOC 九条业务不变量**：n8n 编排权、SQL Path A、ClickHouse 出数、render 出图、archive 熔断、null 语义、不 Mark as Read、DingTalk 目标来自凭据、LLM 不参与——在 Baseline §15.2 `L361-373`、Adjudication §3.3 `L83-89`、`L342-350`、ADR-001、TODO §2 `L53`、§24 `L1216-1217`、CUTOVER-CHECKLIST §2 一致（C01/C02 ✅）
7. **禁止双生产消费者**：六处一致，且幂等/去重要求在 Adjudication `L171`、TODO `TASK-055`、ROLLBACK-PLAN §4 `L27` 闭环（C02 ✅）
8. **备份对象与文件备份策略**：Baseline §11.2、ADR-006 Coverage、TODO `TASK-038～041` 一致；明确不用 Velero/MinIO（Adjudication `L122,L461`）（C02 ✅）
9. **Agent 权限红线**：11 类越权路径全部阴性，详见 §9.2（C09 ✅）
10. **UNKNOWN 处理**：五份文件一致声明 `UNKNOWN ≠ PASS/SAFE`，未发现任何反向处理（C07 ✅）
11. **两清两固不引入新基础设施**：Baseline `L579`、README `L1277`、CLM README `L113-118/L151`、TODO `L1278` 四处一致，且 `TASK-SEC-005 L1270` 明确"复用现有 Agent Job / Task YAML / DingTalk / 审计"（C07 ✅）
12. **发现新鲜度口径**：CLM 24h/168h 与安全基线 24h/24h/24h/168h 一致；stale → REVIEW 两处一致（C07 ✅）
13. **CIDR 示例网段**：Baseline §4.1 `L75-81`、Adjudication §12 `L246-252`、TODO `TASK-008` 使用同一组示例，且 Baseline `L83` 声明"不能照抄示例地址"（C02 ✅）
14. **V0.2～V1.0 演进方向**：Baseline §23 与 Adjudication §23 方向一致（差异仅为 Baseline V0.2 增补了 "可选 xw-opsapi"，且与 TODO `L1043` backlog 呼应）（C08 ✅）

---

## 13. Final Verdict

### Q1：当前仓库是否存在阻断 V0.1 实施的 P0/P1 一致性问题？

**无 P0。存在 4 项 P1。**

```
P0: 0
P1: F-01  恢复演练排在 DA-SOC 生产切换之后（违反 Baseline L493 强制条款）
    F-03  Git 仓库布局两个互斥方案未定（README §14 vs TASK-004）
    F-06  Baseline 正式 Namespace（xw-platform/xw-observability/xw-aiops）未落任务
    F-07  Git Task 生命周期 5 套互斥状态枚举
```

其中 **F-01 具有真实生产后果**（可能在无任何恢复验证的情况下完成生产切换），建议在 `TASK-057` 之前修正；**F-03 阻塞 Phase 0 出口**（`TASK-004` 的下游 4 项任务）；**F-06/F-07** 阻塞 Phase 3 与 Phase 14 的质量验收。

四者**均不涉及架构决策变更**（`Architecture Change Required: NO`）。

### Q2：是否建议进行一次统一修订？

**建议，且只做一次、且只针对一致性。**

修订包建议包含：
1. `TODO.md`：调整实施顺序 + 依赖图补边（F-01）；补三个平台 Namespace 任务（F-06）；补 6 个 SEC 任务的 Risk/Evidence（F-12）；依赖字段三处校验（F-17）；章节编号（F-19）；TASK-004 布局映射声明（F-03）
2. `VERSION-MATRIX.md`：删除 §6 的 RPO/RTO（F-09）；状态字典（F-05）； `EXTERNAL-DEPENDENCIES.md` 两行阻塞目标校正（F-14）
3. `RESTORE-DRILL-PLAN.md`：`TASK-061~064` → `TASK-064~066`（F-13）
4. `components.yaml` / `upgrade-rules.yaml` / `policies.yaml`：`planned`→`discovered`、`review`→`manual-review`（F-08）；补 `container-image` policy（F-22）；成员清单取并集（F-04）
5. `README.md`：状态与首要任务（F-15）；§15 版本路线（F-16）；§24/§27（F-20）；§14 目录同步（F-03）
6. `ARCHITECTURE-BASELINE-V0.1.md`：**最小文字澄清**，含 §24 Rule 2 改过去时（F-02）、§16.3 增加权威状态表与 `category` 字段（F-07/F-11）、§5.3 改为机制无关表述（F-10）、§19 L1 限定语（F-23）
7. `00-project/*.md` 与 `01-architecture/`、`02-governance/` 最小填充（F-18）

### Q3：是否需要修改 Architecture Baseline？

**需要，但仅限 4 处文字澄清，不是架构变更：**

| 位置 | 修订 | 性质 |
|---|---|---|
| §24 Rule 2 `L568` | 由"TODO.md 不由本阶段修改"改为"已重新生成 / 唯一源 / 不得改已冻结结论" | 消除失效规则（F-02） |
| §16.3 `L394-427` | 增加唯一权威 Task 状态表 + 可选 `category` 枚举 | 词汇统一（F-07/F-11） |
| §5.3 `L149` | 改为机制无关的组件/扩展清单比对要求 | 兼容 KS 4.1.x（F-10） |
| §19 `L467` | L1 增加"须为已批准白名单 Runbook，未登记默认 L2" | 消除 L1 边界歧义（F-23） |

**Kubernetes 架构、KubeSphere 定位、Calico、Harbor 位置、Local PV、观测栈、DA-SOC 承载方式、Agent Runtime、Task 模型、Backup 模型、L0/L1/L2、V0.1 Scope——全部保持不变。**

> Baseline 不需要重新裁决、不需要重新评审；只需一次性 Patch + version bump（建议在文件头记录 `Consistency Patch 2026-10-08`，例如 `V0.1-r1`）。

### Q4：是否可以直接进入 TASK-001？

**可以启动 `TASK-001`，但建议把一致性修订与 Phase 0 并行进行。**

理由：
- `TASK-001`（建立实施参数冻结记录与责任矩阵）**本身不被任何 P1 阻塞**；
- 但 `TASK-004`（依赖 F-03）位于同一 Phase，`TASK-005`（依赖 F-07 的状态枚举）紧随其后；
- 建议：**🟢 进入 Phase 0 / TASK-001，同时执行 Q2 的修订包，并在 `TASK-004` 完成前闭合 F-03/F-06/F-07；`TASK-057` 前必须闭合 F-01。**

即：**不是"停止等待"，而是"边冻结边修"，把一致性修订作为 Phase 0 的一部分。**

### Q5：是否建议再次进行架构设计？

**明确不建议。**

依据本审计：
- 未发现 P0；
- 未发现架构级矛盾；
- 全部 23 项发现的 `Architecture Change Required` 均为 **NO**；
- 5 台 VM / 1 CP + 2 Worker / 单集群 / Harbor 外置 / local-path / 单一观测栈 / `da-soc` 实际承载 / Job Agent / Git Task / L0-L1-L2 九处核心结论在所有生效文件中自洽；
- CLM 与两清两固补丁遵守了"复用现有平台能力"的原则，未引入第二套栈、第二套 Task 模型、第二套漏洞机制或任何新安全基础设施。

**建议立即停止架构讨论，进入 Implementation Preflight：执行一致性补丁 → 冻结 Baseline → `TASK-001`。**

---

**审计结束 · READ ONLY · 未修改任何既有项目文件**
