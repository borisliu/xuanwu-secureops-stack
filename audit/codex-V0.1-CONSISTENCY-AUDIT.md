# Xuanwu SecureOps Stack V0.1
# Consistency Audit — Codex

审计日期：2026-10-08
审计范围：当前工作区 Git 仓库现状（架构、版本、TODO、CLM、两清两固、安全边界和历史候选方案）
审计模式：READ ONLY；本报告是本次唯一新增文件。

## 1. Executive Summary

总体判断：**BLOCKED**

当前没有发现 P0 级架构重设计问题，但存在 4 个 P1 一致性问题。它们不要求重新设计 V0.1，却必须在进入正式实施前完成一次小范围统一修订。主要问题集中在：L0/L1/L2 权限边界、TASK-SEC-006 与最终验收顺序、Baseline 对 TODO 修改的过时规则，以及正式项目文档/Runbook 的 Source of Truth 完整性。

- P0：0
- P1：4
- P2：5
- P3：2

结论不是“重新做架构”，而是“先做一次文档与任务依赖一致性修订，再进入 Implementation Preflight / TASK-001”。

## 2. Architecture Consistency

总体结果：**PASS WITH P1 CLARIFICATIONS**

已保持一致的最终架构事实：

- 5 台 VM：1 Control Plane + 2 Worker；Harbor 与 Backup 位于 Kubernetes 外。
- 单集群 KubeSphere，Calico，local-path/Local PV，ClickHouse 单副本并固定 DA-SOC 数据节点。
- 唯一观测栈为 Prometheus、Grafana、Alertmanager、Fluent Bit、Loki。
- DA-SOC 的 n8n、ClickHouse、render/archive、raw archive 在 `da-soc` Namespace 内实际运行。
- 验证双跑、生产单活；ECS 只作短期回退源，不得与 Kubernetes n8n 同时读取生产邮箱。
- Agent 使用短生命周期 Job；Task 使用 Git YAML/Markdown；不使用 Task CRD、常驻高权 Agent 或 `xw-opsapi`。

证据：`10-decisions/ARCHITECTURE-BASELINE-V0.1.md:11-15`、`10-decisions/ADR/ADR-001-da-soc-hosting.md:9-21`、`TODO.md:33-54`。

发现的架构边界冲突详见 `P1-001` 和 `P1-004`；它们是规范边界冲突，不是 VM、DA-SOC、存储或观测架构冲突。

## 3. Version Consistency

总体结果：**PASS**

版本状态没有发现同一组件被声明为互相矛盾的固定版本：

- UOS：V20 1060e AMD64。
- Kubernetes：v1.30.6。
- KubeSphere：4.1.x，优先验证 4.1.2。
- containerd：1.7.x。
- CNI：Calico。
- n8n：2.15.0 或批准的现有兼容版本。
- render/archive：`da-soc-render:0.1`。
- 其他观测、Harbor、ClickHouse、Agent 镜像按 Candidate/Pending Freeze/Approved digest 管理。

`09-implementation/VERSION-MATRIX.md:3-14` 明确矩阵仍是候选基线；`README.md:1143-1157`、`TODO.md:18-29`、`components.yaml` 的状态含义一致。TBD/unknown 不被直接判定为错误，因为当前规则要求先完成兼容性验证和运行态 Discovery。

历史 `09-implementation/00-architecture-review/*.md` 中出现 4 VM、Task CRD、ECS 平行最终承载等内容，但这些文件是候选方案输入；`ARCHITECTURE-ADJUDICATION` 已明确其不具实施权威性，不构成当前版本冲突。

## 4. TODO Consistency

总体结果：**P1 — FINAL ACCEPTANCE ORDER IS NOT CONSISTENT**

正向结果：Calico、Harbor、Local PV、Backup、Observability、DA-SOC、AI Ops、CLM 和两清两固都已经有明确任务或任务链。`TODO.md:1142-1169` 的依赖图也已经纳入 `TASK-SEC-001`～`TASK-SEC-006`。

但最终验收顺序存在问题：

- `TASK-067` 是最终综合验收，前置条件在 `TODO.md:1025-1037`，没有要求 `TASK-SEC-006`。
- 依赖图在 `TODO.md:1167-1169` 将 `TASK-SEC-006` 放在 `TASK-067` 之后。
- `TASK-SEC-006` 又在 `TODO.md:1277-1284` 声明依赖 `TASK-067`，并要求把两清两固纳入 `TASK-067/TASK-068` 交接。
- `TODO.md:1224` 同时要求最终完成前两清两固验收覆盖率达到 100%。

这会导致“最终综合验收先通过，专项安全综合验收后完成”的不确定执行顺序。详见 `P1-002`。

## 5. Source of Truth Consistency

总体结果：**P1/P2 — PARTIALLY CONSISTENT**

当前能够形成的 Source of Truth 链路为：

```text
Architecture Baseline → 架构与边界
VERSION-MATRIX → 候选版本、冻结条件和兼容性状态
components.yaml → 组件注册表
policies.yaml → 生命周期策略和审批策略
upgrade-rules.yaml → upgrade_required 与优先级规则
security-baseline.yaml → 两清两固统一状态/证据/隐私模型
port-baseline.yaml → 端口 Desired State
account-baseline.yaml → 账号 Desired State
access-control-baseline.yaml → 访问控制 Desired State
TODO.md → 实施任务和依赖
Discovery Evidence → 运行态事实
```

`CLM README:67-72` 与 `ARCHITECTURE-BASELINE:264-272` 对 Git、运行态 Evidence 和业务数据边界的定义一致。

但存在两个完整性问题：
- `10-decisions/ARCHITECTURE-BASELINE-V0.1.md:568` 仍写着 TODO 不由本阶段修改，而当前 TODO 已包含 CLM 和两清两固任务。详见 `P1-003`。
- `00-project/GOALS.md:3-6`、`SCOPE.md:3-6`、`VISION.md:3-6`、`VERSIONING.md:3-6` 仍是占位文件；`06-runbooks/` 仅有 `.gitkeep`。这使正式目标、范围、版本和执行方法的 Source of Truth 尚未完整落地，详见 `P2-003`、`P2-004`。

## 6. CLM Consistency

总体结果：**PASS WITH P2 STATUS CLARIFICATION**

CLM 没有引入第二套漏洞平台、第二套 Task Runtime 或新的基础设施。它与既有架构保持以下一致：

- CLM 管理 Component Registry、Vulnerability State、Lifecycle State 和 Upgrade State。
- `VERSION-MATRIX.md` 仍负责候选版本与冻结条件；CLM 负责组件身份、发现、漏洞/生命周期判断和升级任务。
- L0 可发现、评估、报告和创建 Git Task；生产升级、核心平台变更仍需 L2。
- 未知版本进入 REVIEW/高风险，不能当成安全。

证据：`07-aiops/component-lifecycle/README.md:23-37`、`:67-100`，`components.yaml`，`policies.yaml`，`upgrade-rules.yaml`。

需要澄清但不构成 P1 的问题：同名 `status` 在 Desired State、Upgrade State、Lifecycle State、Finding State 和 TODO Task 状态中含义不同。当前文档大多按上下文区分，但没有统一字段命名空间，详见 `P2-002`。

## 7. Two-Clear-Two-Firm Consistency

总体结果：**PASS WITH P1 L1/L2 CLARIFICATION**

四项能力已经统一接入：

- 清高危漏洞：复用 CLM。
- 清高危端口：`04-security/port-baseline.yaml`。
- 固弱账号口令：`04-security/account-baseline.yaml`。
- 固弱访问控制：`04-security/access-control-baseline.yaml`。

统一状态 `PASS/FAIL/REVIEW/UNKNOWN`、统一闭环和敏感信息禁止规则均在 `04-security/security-baseline.yaml:5-34` 定义；四项能力未引入 SIEM、SOAR、CMDB、NDR、独立数据库、第二套监控或新的 Agent Runtime。

P1 风险在于权限等级定义不完全一致：
- `10-decisions/ARCHITECTURE-BASELINE-V0.1.md:461-471` 将 L0 定义为状态/查询类，将 RBAC/NetworkPolicy 修改归入 L2。
- `04-security/security-baseline.yaml:18-23` 允许 L1 `non_production_remediation`。
- `04-security/access-control-baseline.yaml:70-73` 明确允许 L1 `non_production_low_risk_policy_change_with_approved_runbook`。
- `TODO.md:1246` 主要把生产防火墙、Ingress、Calico 和核心业务端口升级为 L2，未明确非生产 RBAC/NetworkPolicy 的边界。

详见 `P1-001`、`P1-004`。

## 8. V0.1 Scope / HA Consistency

总体结果：**PASS**

未发现 V0.1 被当前正式 Baseline 偷换成 HA、多集群或完整安全平台。

- 单 Control Plane 是明确接受的非 HA 边界，恢复、备份和演练没有被删除。
- Ceph、Longhorn、分布式 ClickHouse、Service Mesh、ELK/SIEM、CMDB、多集群、跨地域 DR、GPU、完整 Runtime/Supply Chain Security、Task CRD、常驻高权 Agent 均被明确排除或推迟。
- `10-decisions/ARCHITECTURE-BASELINE-V0.1.md:557-563` 与 `TODO.md:54` 的 V0.2/V0.3 演进方向一致。

需要注意的范围文字漂移：`README.md:921-934` 仍把 V0.1 “唯一核心目标”写成承载 DA-SOC 和验证 AI Ops，而 `README.md:1271-1277`、`10-decisions/ARCHITECTURE-BASELINE-V0.1.md:573-579` 已正式加入两清两固。详见 `P2-001`。

## 9. Agent / L0-L1-L2 Consistency

总体结果：**P1**

Agent 的 Job、最小 ServiceAccount、TTL、固定 digest、Git Task、L2 审批、不得使用 cluster-admin/root/任意 shell/长期 kubeconfig等边界基本一致。证据：`10-decisions/ARCHITECTURE-BASELINE-V0.1.md:15`、`:152-164`、`:375-390`、`10-decisions/ADR/ADR-007-agent-runtime.md:8-22`、`TODO.md:923-963`。

但 L0 定义存在明显文本冲突：
- Baseline `:461-463` 只列只读状态、健康检查、查询和汇总。
- CLM `README.md:94-100` 与 Security Baseline `security-baseline.yaml:18-23` 把创建 Git Task/通知列为 L0。
- `TODO.md:927-928` 又要求 L0 可自动创建只读 Job。

创建非执行性 Task 本身可以是安全的 L0 动作，但当前 Baseline 没有把它写出来。应在实施前统一为“L0 可创建任务和通知，但不得执行运行态变更”。详见 `P1-001`。

## 10. Detailed Findings

| ID | Severity | Category | File A | File B | Conflict | Recommended Resolution |
|---|---|---|---|---|---|---|
| P1-001 | P1 | Agent / L0-L1-L2 | `10-decisions/ARCHITECTURE-BASELINE-V0.1.md:461-471` | `07-aiops/component-lifecycle/README.md:94-100`; `04-security/security-baseline.yaml:18-23`; `TODO.md:927-928` | Baseline 的 L0 仅描述只读查询，而 CLM/安全基线/TODO 允许 L0 创建 Git Task、通知或只读 Job；同时 L1 非生产策略整改与 Baseline 的 L2 修改边界未完全对齐。 | 以 Baseline 为权威做一次边界澄清：L0 可发现/评估/报告/创建非执行 Task/通知；L1 仅执行明确白名单且不修改 RBAC/NetworkPolicy 等 L2 控制；L2 统一要求人工审批。 |
| P1-002 | P1 | TODO / Final Acceptance | `TODO.md:1025-1037` | `TODO.md:1167-1169`; `TODO.md:1277-1284` | TASK-067 最终验收没有 TASK-SEC-006 前置条件，但依赖图把 SEC-006 放在 TASK-067 之后；DoD 又要求两清两固在最终完成前通过。 | 在不改变架构的前提下统一任务顺序：推荐 TASK-SEC-006 先于 TASK-067，或明确它是 TASK-067 之后的补充验收并把 TASK-068 前置条件同步修改。 |
| P1-003 | P1 | Source of Truth / Governance | `10-decisions/ARCHITECTURE-BASELINE-V0.1.md:565-570` | `TODO.md:3-12`, `TODO.md:1053-1137`, `TODO.md:1224-1284` | Baseline 仍写“TODO.md 不由本阶段修改”，但当前仓库已经把 CLM 和两清两固任务写入 TODO；同时 TODO 又声明自己是当前唯一实施任务源。 | 把 Baseline 规则限定为“架构裁决阶段不生成 TODO”；当前实施 TODO 的增量必须通过变更记录/审计进入唯一任务源。 |
| P1-004 | P1 | Security Boundary | `10-decisions/ARCHITECTURE-BASELINE-V0.1.md:467-471` | `04-security/security-baseline.yaml:18-23`; `04-security/access-control-baseline.yaml:70-73` | Baseline 将 RBAC/NetworkPolicy 修改列入 L2；安全基线却允许 L1 非生产整改/低风险策略变更，未说明哪些策略变更仍属于 L2。 | 明确所有 RBAC/NetworkPolicy 核心边界变更均为 L2；若确需非生产例外，必须在 Baseline、Security Baseline、TODO 和 Runbook 同步定义范围、审批、回退和验证条件。 |
| P2-001 | P2 | Project Status | `README.md:1141-1143` | `README.md:1265-1269`; `TODO.md:3-5` | README 同时出现“Architecture Frozen / Implementation TODO Ready”和“Planning / Architecture”。 | 将 README 当前状态统一为 Architecture Frozen / Implementation TODO Ready，并保留版本候选状态。 |
| P2-002 | P2 | CLM State Model | `07-aiops/component-lifecycle/README.md:39-48` | `07-aiops/component-lifecycle/upgrade-rules.yaml:45-53`; `TODO.md:129` | `status` 同时承载 desired/upgrade/lifecycle/finding/task 等不同语义，虽可按上下文理解，但机器实现容易发生状态漂移。 | 明确字段命名空间，例如 `desired.status`、`upgrade.status`、`lifecycle.status`、`finding.status`、`task.status`，不改变现有三态分离原则。 |
| P2-003 | P2 | Source of Truth | `00-project/GOALS.md:3-6`; `00-project/SCOPE.md:3-6`; `00-project/VERSIONING.md:3-6` | `10-decisions/ARCHITECTURE-ADJUDICATION-V0.1.md:36`; `README.md:770-815` | 正式项目目录中的目标、范围、愿景、原则和版本文档仍是占位文件，不能独立表达当前批准基线。 | 在不重开架构的前提下补齐正式文档，或明确它们在 V0.1 前由 README/Baseline 代管。 |
| P2-004 | P2 | Runbook / Source of Truth | `06-runbooks/.gitkeep` | `README.md:44-45`, `:750-766`; `TODO.md:123`, `:927-929`; `ADR-007:14` | 文档把 Runbook 作为 Agent 执行依据和 Git Source of Truth，但仓库当前没有实际 Runbook 文件。 | 在 TASK-001/TASK-005 阶段登记 Runbook 目录和最小 Runbook 清单；实施前至少补齐平台升级、DA-SOC、端口、账号、访问控制和回退 Runbook。 |
| P2-005 | P2 | YAML / State Semantics | `04-security/security-baseline.yaml:5-17` | `04-security/port-baseline.yaml:1-10,22-70`; `07-aiops/component-lifecycle/components.yaml` | 安全基线声明字符串枚举和 schema，但 YAML 中 `schema_version: 0.1`、`port_required: YES/NO` 在 YAML 1.1 解析器中可能成为数字/布尔值，机器读取结果不稳定。 | 统一用明确字符串表示 schema 和枚举，并加入 YAML schema/CI 校验；不改变安全架构。 |
| P3-001 | P3 | Document Order | `TODO.md:1208-1227` | `TODO.md:1229-1284` | `## 20B` 两清两固任务被追加在 `## 24 Definition of Done` 之后，阅读顺序与依赖图不一致。 | 将专项任务放回 CLM/Phase 章节之前或明确为附录，不改变任务内容。 |
| P3-002 | P3 | Wording | `README.md:921-934` | `README.md:1271-1277` | V0.1 “唯一核心目标”与新增两清两固能力的表述层级不同，读者可能误判安全运营不属于 V0.1 必选范围。 | 统一 V0.1 核心目标措辞，明确 DA-SOC 承载、AI Ops MVP 和两清两固是同一验收基线的组成部分。 |

## 11. Potential False Positives

以下内容经过核对，不判定为当前一致性问题：

- `dsh.md` 等历史候选方案中的 4 VM、Task CRD、ECS 平行最终承载、Promtail 或其他组件，是候选输入，不是最终规范；`ARCHITECTURE-ADJUDICATION` 已明确最终采用 Baseline。
- `VERSION-MATRIX.md` 的 Candidate/Pending Freeze 与 TODO 的 READY 不冲突：READY 表示实施规划可开始，不表示版本组合已 Frozen。
- `unknown`/`pending-freeze`/`planned` 不等于错误状态；CLM 明确要求运行态未知进入 REVIEW，版本矩阵明确要求安装前冻结。
- README 中的 GPU、完整 SIEM、完整灾备、多集群等能力域描述属于长期能力或 Non-Goals，不是 V0.1 必须实施项。
- “验证双跑”与“生产单活”不冲突；正式文档明确双跑只使用测试邮箱/回放数据，生产只保留一个消费者。

## 12. No-Issue Areas

以下关键区域未发现 P0/P1 架构冲突：

- VM 拓扑、Harbor 外置、Backup 外置、local-path/Local PV、ClickHouse 单副本。
- Calico、单一 Prometheus/Grafana/Alertmanager/Fluent Bit/Loki 观测栈。
- DA-SOC Namespace、n8n→render/archive→ClickHouse 业务链路、邮箱不标已读和 DingTalk 隔离。
- Git Task/Job/TTL/固定 digest/最小 RBAC/禁止 cluster-admin/root/任意 shell。
- CLM 与 VERSION-MATRIX 的职责分工，以及漏洞发现、评估、升级 Task、审批、验证和回退闭环。
- 两清两固四项能力均已声明统一状态、证据脱敏、任务类别和 L0/L1/L2 处理路径。

## 13. Final Verdict

### Q1
当前仓库是否存在阻断 V0.1 实施的 P0/P1 一致性问题？

**是。** 没有 P0，但存在 P1-001～P1-004。它们不要求重新设计架构，却会让实施团队对权限边界、最终验收顺序和当前 TODO 权威性产生不同解释。

### Q2
是否建议进行一次统一修订？

**是。** 建议只做一次小范围一致性修订，先处理四个 P1，再顺手处理 P2/P3 文档歧义。

### Q3
是否需要修改 Architecture Baseline？

**需要最小文字修订，不需要重新裁决架构。** 至少应澄清 L0 可创建非执行 Task、L1/L2 策略变更边界，以及 Baseline 与当前实施 TODO 的关系。VM、DA-SOC、存储、观测、Harbor、CNI 和 Agent Runtime 不需要重设计。

### Q4
是否可以直接进入 TASK-001？

**不建议直接进入。** 先完成 P1-001～P1-004 的统一修订和签字，再进入 TASK-001；之后仍按现有 Phase 0 参数冻结流程执行。

### Q5
是否建议再次进行架构设计？

**不建议。** 当前问题是规范、任务依赖和状态语义的一致性问题，不是新的架构选型问题。

最终建议路径：

```text
一次性修订 P1/P2/P3
        ↓
重新审计一致性
        ↓
Architecture Baseline 继续冻结
        ↓
TASK-001 / Implementation Preflight
```
