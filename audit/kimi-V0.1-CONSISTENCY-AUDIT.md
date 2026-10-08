# Xuanwu SecureOps Stack V0.1
# Consistency Audit — kimi

> 审计范围：main 分支全部正式项目文件（README、00-project、10-decisions、09-implementation、07-aiops/component-lifecycle、04-security、TODO.md）。
> 排除：`09-implementation/00-architecture-review/`（历史候选方案，非权威文件）、`.git/`。
> 审计性质：READ ONLY。未修改任何现有文件。

---

## 1. Executive Summary

**总体判断：PASS WITH P1**

补丁后的仓库主干（Baseline / Adjudication / ADR-001~008 / 再生成 TODO / VERSION-MATRIX / CLM / 两清两固）在**架构、版本、范围、权限四个维度高度一致**：没有发现任何 P0，没有发现架构级矛盾，没有发现隐性范围膨胀（无 Ceph/Longhorn/Service Mesh/SIEM/Task CRD/xw-opsapi 进入 V0.1 必须实施内容），没有发现 Agent 越权通道或 L2 绕过路径，UNKNOWN 的处理在全库一致。

**发现 3 个 P1，全部是"再生成 TODO 后遗留的跨文档交叉引用漂移"**，集中在 `09-implementation/EXTERNAL-DEPENDENCIES.md`、`09-implementation/RESTORE-DRILL-PLAN.md` 和 TODO 内部 TASK-067 ↔ TASK-SEC-006 的循环依赖。它们不涉及架构结论本身，但会误导实施门禁的归属，必须在进入 Phase 10/15 之前修复。

- P0：0
- P1：3（AUD-001 ~ AUD-003）
- P2：11（AUD-004 ~ AUD-014）
- P3：4（AUD-015 ~ AUD-018）

结论：**不需要重新做架构评审；建议一次统一修订（纯文档级），然后进入 TASK-001。**

---

## 2. Architecture Consistency

逐项核对 C02 检查清单，**全部一致**：

| 检查项 | 结论 | 证据 |
|---|---|---|
| 单集群、1 CP + 2 Worker、5 VM | ✅ 一致 | Baseline §2.1（`10-decisions/ARCHITECTURE-BASELINE-V0.1.md:21-27`）、Adjudication §1/§10、TODO §1/§2 |
| Harbor 在 K8s 外 | ✅ 一致 | ADR-003、Baseline §9、TODO TASK-025、EXTERNAL-DEPENDENCIES |
| 备份在 K8s 外 | ✅ 一致 | ADR-006、Baseline §11、Adjudication §10（`xw-backup-01` 独立 VM + 离线/不可变副本） |
| local-path / Local PV，无 Ceph/Longhorn | ✅ 一致 | ADR-004、Baseline §8、TODO TASK-022 |
| ClickHouse 单副本固定 `xw-wk-02` | ✅ 一致 | ADR-004、Baseline §3.1/§8.2、TODO TASK-044 |
| 唯一观测栈 Prometheus/Grafana/Alertmanager + Fluent Bit/Loki，无 ELK/Promtail/第二套/SIEM | ✅ 一致 | ADR-005、Baseline §10、Adjudication §8/§24（全文 grep 无违规引用，ELK/Promtail 仅出现在"禁用/否决"语境） |
| DA-SOC 必须运行于 `da-soc` Namespace、验证双跑、生产单活、ECS 短期回退 | ✅ 一致 | ADR-001、Baseline §1/§15、TODO §1/§2、CUTOVER-CHECKLIST |
| 禁止 ECS + K8s 同时读取生产邮箱 | ✅ 一致 | ADR-001 Parallel Run Rule、Baseline §1、TODO §3 rule 3、CUTOVER §4、RISK-006 |

"玄武云盾 = 平台、DA-SOC = 第一个业务应用"在 README §17、Baseline、Adjudication §3.1 表述一致。

---

## 3. Version Consistency

候选版本在 README §23、VERSION-MATRIX §2、TODO §0、EXTERNAL-DEPENDENCIES、components.yaml 中**完全一致**：

| 组件 | 版本表述 | 状态语义 | 结论 |
|---|---|---|---|
| OS | 统信服务器操作系统 V20 1060e AMD64（README:1151 / VM:20 / TODO:20 / components.yaml:10） | Candidate / Pending Compatibility Validation | ✅ |
| Kubernetes | v1.30.6（README:1152 / VM:21 / TODO:21 / components.yaml:20） | Candidate | ✅ |
| KubeSphere | 4.1.x，优先验证 4.1.2（README:1153 / VM:22 / TODO:22 / components.yaml:30） | Candidate | ✅ |
| containerd | 1.7.x | Candidate | ✅ |
| Calico | 未指定版本（VERSION-MATRIX:24；components.yaml:50 `version: unknown, status: candidate`） | Candidate；未知版本由 CLM `unknown-version` 规则进入高风险 REVIEW | ✅（见 §11 假阳性 #2） |
| n8n | 2.15.0 或批准的现有兼容版本（VM:40 / components.yaml:80） | Candidate / Verify | ✅ |
| render/archive | `da-soc-render:0.1`（VM:41 / components.yaml:90） | Candidate | ✅ |

- 没有发现同一组件在不同文件中出现互相矛盾的版本号。
- 没有发现把 Candidate 误标为 Frozen/Approved 的情况；README:1157、VERSION-MATRIX §1 rule 1/§4、TODO:29 均明确"候选 ≠ 官方认证组合、候选 ≠ 安装放行"。
- 状态词汇存在跨文件不统一（P2，见 AUD-014），但**语义不冲突**，不构成 C03 定义的一致性问题。

---

## 4. TODO Consistency

**架构 → Task 覆盖完整**：Calico（TASK-018/019）、Harbor（TASK-025~028）、Local PV（TASK-022~024）、Backup（TASK-038~041）、Observability（TASK-029~033）、DA-SOC（TASK-042~059）、AI Ops（TASK-060~063）、Security Baseline（TASK-034~037）、CLM（TASK-CLM-001~006）、两清两固（TASK-SEC-001~006）均有明确实施任务，且每项任务带 Validation/Evidence/Rollback/Approval/Owner，符合 Baseline §24 和 README §19。

**Task → 架构无违规**：未出现 Task CRD、第二套监控/日志、Ceph/Longhorn、xw-opsapi、常驻 Agent 的实施任务；TODO:50/54/969 明确将这些列为"不使用/不引入"。DoD（TODO:1208-1227）与 Baseline §21 Acceptance 对齐。

问题集中在**交叉引用与结构**（详见 §10）：

- **AUD-001（P1）**：TASK-067 ↔ TASK-SEC-006 循环依赖。
- **AUD-002（P1）**：`EXTERNAL-DEPENDENCIES.md` 至少 5 行任务编号沿用了再生成前的旧 TODO 编号。
- **AUD-003（P1）**：`RESTORE-DRILL-PLAN.md:41` 的门槛语句引用了错误的任务区间。
- AUD-015（P3）：20A/20B 章节位置错乱。

Baseline §24 Rule 2"TODO 必须按 Baseline 重新生成"已经兑现（当前 TODO 头部即"最终实施任务清单"，生成日期 2026-10-08，唯一架构依据 = Baseline），**但附件文件没有同步再生成**，这是本次 P1 的根因。

---

## 5. Source of Truth Consistency

**已正确建立 SoT 的文件**（与审计任务期望的映射完全吻合）：

```text
ARCHITECTURE-BASELINE-V0.1.md   → 架构（Baseline §24 Rule 1 自声明"唯一实施依据"）
09-implementation/VERSION-MATRIX.md → 版本（TODO TASK-001/002、CLM README §4 引用）
07-aiops/component-lifecycle/components.yaml → 组件（source_of_truth: git）
07-aiops/component-lifecycle/policies.yaml → 生命周期策略
07-aiops/component-lifecycle/upgrade-rules.yaml → upgrade_required 规则
04-security/security-baseline.yaml → 安全统一状态模型
04-security/port-baseline.yaml → 端口 Desired State
04-security/account-baseline.yaml → 账号 Desired State
04-security/access-control-baseline.yaml → 访问控制 Desired State
TODO.md → 实施任务（DoD item 14 自声明"唯一实施任务源"）
Git → 定义态（Baseline §11.1，附 SOPS/age 加密边界，明文清单明确）
Evidence → 运行事实（CLM README §4："Discovery evidence 不手工覆盖"）
```

**问题**：

- **AUD-008（P2）**：期望版本存在两个 SoT 声明——VERSION-MATRIX §前言/§2（"批准的期望版本"）与 components.yaml `desired:`（CLM README §2 亦称 components.yaml 定义"期望版本"）。当前值一致（1.30.6 / 4.1.x / V20 1060e），但**未声明谁优先**，漂移时无裁决规则。
- **AUD-009（P2）**：Task schema 在 4 处定义且字段名发散（README §10、Baseline §16.3、ADR-008、security-baseline.yaml `task_model`），未声明 canonical。
- **AUD-013（P2）**：Runbook 的 SoT 目录 `06-runbooks/` 仍为空（仅 .gitkeep），而 policies.yaml:4-27 已硬引用 4 个具名 `validation_runbook`（`platform-upgrade-validation` 等），多个 TASK 也以 Runbook 为前置条件。属"关键事实 SoT 暂缺"，需在 TASK-004/005 或对应实施任务中补齐。

---

## 6. CLM Consistency

**总体一致**：README §23 CLM 段、Baseline §16.5、TODO 20A、CLM README、components.yaml、policies.yaml、upgrade-rules.yaml 五层文档相互吻合：

- 三态模型（Component Registry / Vulnerability State / Upgrade State）在 CLM README §2 与 Baseline §16.5、TODO TASK-CLM-001/003 一致，且明确"不得合并为单个人工版本字段"。
- 未知版本处理一致：README:1163"未知版本必须标记为高风险"、CLM README §3 `unknown_version`、upgrade-rules.yaml `unknown-version` 规则（`risk: high`）、TODO TASK-CLM-002 DoD。未发现 UNKNOWN 被当作 PASS/SAFE 的任何位置。
- `upgrade_required` 判定逻辑一致：CLM README §5 与 upgrade-rules.yaml 规则集（critical-fixed-version P1 / high-fixed-version P2 / kev / eol-eos / not-affected REVIEW / unknown-version）吻合；"存在 CVE ≠ 机械升级"（affected=false → REVIEW）在两处一致。
- L0/L1/L2 边界与 Baseline §12.2/§19 一致；生产自动升级禁用在 policies.yaml `production_auto_upgrade: false`、upgrade-rules.yaml `forbidden_automatic_actions`、Baseline §16.5、TODO TASK-036 四处一致。
- Freshness 一致：policies.yaml `discovery_max_age_hours: 24 / vulnerability_max_age_hours: 168` 与 CLM README §9 指标表、security-baseline.yaml `cadence` 一致。
- **未引入**独立漏洞平台/SBOM 平台/自动 Patch 产品/新数据库/新 Agent Runtime（CLM README §1/§7），符合"复用现有平台能力"的补丁原则。

**问题**：

- **AUD-012（P2）**：CLM Registry 覆盖范围与 VERSION-MATRIX §3 不一致。VERSION-MATRIX 登记了 Kernel、local-path、Ingress Controller、SOPS/age 四个待冻结组件，components.yaml 与 CLM README §1 的组件清单均未包含（只覆盖 16 项）；而 TASK-CLM-001 DoD 要求"Registry 覆盖 V0.1 所有平台、DA-SOC、镜像和观测组件"。
- **AUD-010（P2）**：upgrade-rules.yaml:46 的组件生命周期状态机 `[discovered, assessed, upgrade-required, task-created, approved, executing, verifying, closed, rolled-back, review]` 与 ADR-008 的 Task 生命周期 `[open → analyzed → planned → pending-approval → ...]`、security-baseline.yaml:8 的 finding 生命周期 `[discovered, assessed, task-created, approval-pending, remediation, verifying, closed, escalated]` 三套并存，未声明包含关系。
- **AUD-014（P2）**：状态词汇跨文件不统一（见 §3）。

---

## 7. Two-Clear-Two-Firm Consistency

**一致**：四项能力统一走 `Desired → Observed → Deviation → Risk → Task → Approval → Remediation → Verification → Audit`（security-baseline.yaml `state_model`、Baseline §25、README:1275、CLM README §10），状态模型统一为 `PASS/FAIL/REVIEW/UNKNOWN` 且 `unknown_is_pass: false`；四份基线的 L0/L1/L2 划分与 Baseline §19、README §4.3 一致；敏感信息禁入 Git/日志/DingTalk/Agent 上下文在 security-baseline.yaml `privacy`、account-baseline `secret_values: never_collect`、Baseline §10.3、ADR-005/008 一致；指标名与目标在 security-baseline.yaml `metrics`、CLM README §11、TODO DoD item 13 一致。

**复用性检查通过**：未引入 SIEM/SOAR/CMDB/NDR/完整 IAM/新安全平台/第二套监控/新数据库（README:1277、Baseline §25、CLM README §10），完全复用 K8s/KubeSphere/Calico/Harbor/Prometheus/Fluent Bit/Loki/Git/Agent Job。

**问题**：

- **AUD-011（P2）**：两清两固指标集在 CLM README §11 与 security-baseline.yaml:34 重复定义（8 项同名指标），未声明 canonical（应是 security-baseline.yaml）。
- **AUD-001（P1）**：TASK-SEC-006 与 TASK-067 的依赖闭环（见 §10）。
- AUD-016（P3）：port-baseline.yaml:70 `port_required: REVIEW` 越出该字段 YES/NO 值域（REVIEW 应放在 `status`）。

---

## 8. V0.1 Scope / HA Consistency

**未发现隐性范围膨胀**：多 CP/多集群/跨地域 DR/Ceph/Longhorn/Service Mesh/SIEM/SOAR/CMDB/GPU/复杂 Policy Engine/Task CRD/常驻高权限 Agent/自动 K8s 升级/自动 OS 升级/自动生产 Patch 全部只出现在"不引入/Non-Goals/被否决/Deferred/V0.2+"语境（Baseline §23/§24、Adjudication §22/§24、TODO:54、CLM README §7、upgrade-rules.yaml `forbidden_automatic_actions`）。

**另一侧也检查通过**：备份、恢复演练、故障演练能力没有被"单 CP/非 HA"叙事削弱——TASK-038~041 + TASK-064~066 + RESTORE-DRILL-PLAN D1~D4 + DoD item 8 完整保留，且"未完成 S7/S8 恢复验证不得进入 DA-SOC 生产切换"（Baseline §20）与"没有恢复演练的备份不算完成"（ADR-006）一致。

**问题**：

- **AUD-007（P2）**：README §15 版本路线未与补丁同步——V0.1 能力清单（README:830-843）未包含 CLM 与两清两固（它们在 §23/§27 已被定义为 V0.1 内容），且 V0.2 清单中的"漏洞检查"（README:855）与 V0.1 CLM 的漏洞状态能力产生轻度叠义。属路线图漂移，非范围膨胀。

---

## 9. Agent / L0-L1-L2 Consistency

**一致，未发现越权或绕过**：

- L0/L1/L2 定义在 README §4.3、Baseline §12.2/§19、ADR-007、CLM README §6、security-baseline.yaml `approval` 五处语义一致；生产 RBAC/NetworkPolicy/CNI/存储/节点/生产 n8n/凭据/生产数据 = L2 的清单一致。
- Agent 权限边界一致且收窄：短生命周期 Job、固定 digest、独立 SA、TTL、默认只读；无 cluster-admin / root / 长期 kubeconfig / 任意 shell（Baseline §13、ADR-007 Prohibited、TODO TASK-036、account-baseline `cluster-admin-binding`/`high-privilege-serviceaccount` 均为 critical + L2）。
- `agent-l1-render` 白名单（Baseline §5.4：仅 da-soc-render 限定 restart/rollout，禁读 Secret/改 RBAC/NetworkPolicy/删 PVC/改 digest）与 Adjudication §15、TASK-036 一致。
- 未发现任何 L2 绕过路径：CLM 自动升级禁用、Git 分支保护、审批前置（TASK-CLM-005"审批前不能创建升级 Job"）、access-control-baseline `agent_escalation: BLOCK_AND_ESCALATE_TO_L2` 互相咬合。
- 邮箱红线一致：IMAP ALL/不标已读、不覆盖「监测bjfz邮箱广电报送信息」、双消费者禁止，在 ADR-001、Baseline §15.2/§21、TODO §3、CUTOVER §4、account-baseline `never_test_against_production_mailbox` 全链一致。

---

## 10. Detailed Findings

| ID | Severity | Category | File A | File B | Conflict | Recommended Resolution |
|---|---|---|---|---|---|---|
| AUD-001 | **P1** | C04 TODO / C07 | TODO.md:1283（TASK-SEC-006 Dependencies: `TASK-SEC-002～005、TASK-067`） | TODO.md:1224（DoD item 13 要求两清两固覆盖率 100% 方可完成 V0.1）+ TODO.md:1027（TASK-067 前置未含 TASK-SEC-*） | 循环依赖：TASK-067 的完成要求两清两固达标，而两清两固验收 TASK-SEC-006 又依赖 TASK-067 完成 | 将 TASK-SEC-006 的 Dependencies 改为 `TASK-SEC-002～005`（其结果**汇入** TASK-067）；或在 TASK-067 Preconditions 中显式加入 TASK-SEC-006，二选一，消除环 |
| AUD-002 | **P1** | C04 TODO | 09-implementation/EXTERNAL-DEPENDENCIES.md:23,28,29,33,37 | TODO.md 新编号（TASK-020/044/045/049/052/061/064~066） | 附件沿用旧 TODO 编号：`workflow source` 指向 TASK-044（新编号下是 ClickHouse 部署，应为 TASK-049）；`encryption key recovery` 指向 TASK-045（应为 TASK-049）；`生产 DingTalk 凭据` Needed Before TASK-054（应为 TASK-052）；`离线/不可变备份副本` Blocking TASK-061（应为 TASK-064~066）；`出口防火墙白名单` Needed Before TASK-021 / Blocking TASK-043 前后列自相矛盾（应为 TASK-020） | 按当前 TODO 重排这 5+ 行的 Needed Before / Blocking Task；同步复核该行表全部 40 行编号 |
| AUD-003 | **P1** | C04 TODO | 09-implementation/RESTORE-DRILL-PLAN.md:41（"任一关键恢复失败则 TASK-061～TASK-064 不通过"） | TODO.md（TASK-061~063 = AI Ops MVP；TASK-064~066 = 恢复演练） | 门槛指向错误任务区间：会把恢复失败错误地归咎于 AI Ops 任务，并漏掉真正的恢复演练 TASK-065/066 | 改为"任一关键恢复失败则 TASK-064～TASK-066 不通过，且 TASK-067 阻断" |
| AUD-004 | P2 | C01 | README.md:1265-1269（页脚 `Status: Planning / Architecture`） | README.md:1143（§23 `V0.1 --- Architecture Frozen / Implementation TODO Ready`） | 同一文件两个矛盾的项目状态 | 页脚状态更新为 `Architecture Frozen / Implementation TODO Ready` |
| AUD-005 | P2 | C01 | README.md:1165-1174（§23 "当前首要任务"仍列"完成 V0.1 架构设计/治理/AI Ops 最小模型"） | README.md:1143 + TODO.md:18（Architecture FROZEN / READY） | 状态冻结后仍把已完成事项列为当前首要任务，后续 Agent 会误判阶段 | 替换为 TASK-001 起的实施导向清单 |
| AUD-006 | P2 | C01 | 00-project/VISION.md、GOALS.md、SCOPE.md、PRINCIPLES.md、VERSIONING.md（均"占位 · 待编写"；VERSIONING 还写 `V0.1 Planning / Architecture`） | README.md:1143 / Baseline / TODO | 项目自称 Frozen 但 00-project 五件正式资料仍为空，VISION/GOALS/SCOPE/PRINCIPLES 的事实内容目前只存在于 README 与 Baseline | 一次性补齐五件（可从 README/Baseline 收敛生成），或显式声明"以 Baseline 为临时权威、00-project 于 TASK-004 补齐" |
| AUD-007 | P2 | C01/C08 | README.md:824-862（§15 版本路线：V0.1 能力清单无 CLM/两清两固；V0.2 含"漏洞检查"） | README.md:1159-1163（CLM = V0.1）、README.md:1271-1277（两清两固 = V0.1）、Baseline §16.5/§25 | 版本路线与补丁后的事实漂移：V0.1/V0.2 边界描述未同步 | 在 §15 V0.1 清单中加入"CLM 最小闭环、两清两固安全运营"，并将 V0.2"漏洞检查"改述为"漏洞运营深化"避免叠义 |
| AUD-008 | P2 | C05 | 09-implementation/VERSION-MATRIX.md（§前言"批准的期望版本"） | 07-aiops/component-lifecycle/components.yaml `desired:` + CLM README §2（components.yaml 亦定义"期望版本"） | 期望版本存在两个 SoT 声明，当前值一致但无优先级规则，漂移时无法裁决 | 声明 VERSION-MATRIX 为期望版本的唯一 SoT，components.yaml `desired` 仅镜像引用（或反之），写进 CLM README §4 |
| AUD-009 | P2 | C05 | README.md:594-612（§10 Task 模型：Asset/Proposed Action）+ Baseline §16.3（target/plan）+ ADR-008（目标/计划/回滚） | 04-security/security-baseline.yaml:9-16（task_model.required_fields：category/finding/expected_state/observed_state/remediation…） | Task schema 四处定义、字段名发散，未声明 canonical schema；实施 Agent/人可能按不同 schema 写 Task | 以 Baseline §16.3 为 canonical；security-baseline 声明为"扩展字段"，README §10 改为引用 |
| AUD-010 | P2 | C05/C06 | ADR-008 Lifecycle（open→analyzed→planned→…） | upgrade-rules.yaml:46（discovered→…→review）+ security-baseline.yaml:8（discovered→…→escalated） | 三套任务/发现生命周期状态机并存，未说明是同一 Task 的不同视图还是各自独立 | 声明 ADR-008 为 Git Task 的 canonical 生命周期，另两套为专项覆盖层并给出字段映射 |
| AUD-011 | P2 | C05/C07 | 07-aiops/component-lifecycle/README.md:149-162（§11 指标表） | 04-security/security-baseline.yaml:34（metrics） | 两清两固 8 项指标在两处重复定义，无 canonical 声明 | 声明 security-baseline.yaml 为 canonical，CLM README §11 改为引用 |
| AUD-012 | P2 | C06 | 09-implementation/VERSION-MATRIX.md §3（Kernel、local-path、Ingress Controller、SOPS/age 均为 Pending Version Freeze） | 07-aiops/component-lifecycle/components.yaml（16 项，不含上述 4 个）+ TODO.md:1067（TASK-CLM-001 DoD：覆盖所有平台/DA-SOC/镜像/观测组件） | CLM Registry 覆盖面 < VERSION-MATRIX 覆盖面，与"Component Coverage 100%"目标存在缺口 | 在 components.yaml 增补 kernel/local-path/ingress/sops 四组件（或在 TASK-CLM-001 中显式登记豁免理由） |
| AUD-013 | P2 | C05 | 06-runbooks/（仅 .gitkeep，为空） | 07-aiops/component-lifecycle/policies.yaml:4-27（validation_runbook: platform-upgrade-validation 等 4 个具名 Runbook）+ 多个 TASK 以 Runbook 为前置 | Runbook 的 SoT 目录为空，但策略文件已硬引用不存在的具名 Runbook | 在 TASK-004/005 或 TASK-036/060/063 中建立 Runbook 骨架，或先在 policies.yaml 标注"runbook 名称待 TASK-036 创建" |
| AUD-014 | P2 | C03/C06 | README:1149-1155、VERSION-MATRIX:18-43（Candidate / Pending Compatibility Validation / Pending Version Freeze / Verify）、components.yaml（candidate / pending-freeze）、TODO.md:20-24 | — | 状态词汇跨文件不统一；语义当前不冲突（均未达 Frozen），但无统一词表，后续易出现同一状态多种写法 | 在 VERSION-MATRIX §1 增加状态词表（Candidate/Pending-Freeze/Frozen + 判断规则），各 YAML 对齐 |
| AUD-015 | P3 | C04 | TODO.md 结构（§20 在 1023 行；20A 在 1053 行；20B 在 1229 行，位于 §24 之后） | — | 章节编号顺序错乱：20A 插在 21~24 之前、20B 位于 24 之后；Phase 编号与章节编号不一致（Phase 16 = §20） | 重排为 §20 Final Acceptance、§21 CLM、§22 两清两固、§23 Dependency Graph…顺序顺延 |
| AUD-016 | P3 | C07 | 04-security/port-baseline.yaml:70（`port_required: REVIEW`） | 同文件 :22/:34/:46/:58（YES）、:70（REVIEW） | `port_required` 值域在其他条目为 YES/NO，此处为 REVIEW（状态值混入布尔语义字段） | 改为 `port_required: NO` + `status: REVIEW`（status 已在 record_fields:10 定义） |
| AUD-017 | P3 | C06 | 07-aiops/component-lifecycle/policies.yaml:33（`component in [kubernetes, kubesphere, calico, harbor, os, clickhouse]`） | components.yaml（组件 id 为 `platform.kubernetes`、`network.calico` 等） | approval_rules 的组件名与 Registry 的 component_id 命名不一致，机器匹配会失配 | 统一为 component_id 或 type 枚举 |
| AUD-018 | P3 | 格式 | 全库 | — | 行尾/BOM 不一致：README.md 为 CRLF；TODO.md 带 BOM 且 CRLF/LF 混排（209/413/659/875 行孤 `\r`）；components.yaml:143/163、policies.yaml:55、IMPLEMENTATION-RISKS.md:26 有孤 `\r`；EXTERNAL-DEPENDENCIES.md 出现第二个无编号表 + 表后"## 1. Ownership Rule"标题编号重置 | 统一为 LF（或全库 CRLF）+ 去 BOM；修正 EXTERNAL-DEPENDENCIES 表格与标题结构 |

**P1 逐项证据块**：

**AUD-001**
- File A：TODO.md:1283，TASK-SEC-006 `Dependencies: TASK-SEC-002～005、TASK-067`
- File B：TODO.md:1224，DoD item 13"两清两固已覆盖…达到 100%…UNKNOWN 可见且不默认为 PASS"；TODO.md:1027，TASK-067 Preconditions 只列 `TASK-059、TASK-063、TASK-064～066、TASK-CLM-006`，未列 TASK-SEC-006
- Conflict：V0.1 完成（TASK-067/068）要求两清两固达标（DoD 13 + 执行停止条件 TODO:1227 要求 TASK-SEC-006 完成），而 TASK-SEC-006 又声明依赖 TASK-067，形成环；按任一方向执行都会在门禁处死锁或被迫跳过门禁
- Impact：最终验收阶段无法确定合法执行顺序；实施方可能擅自打破其中一个门禁
- Recommended Resolution：删除 TASK-SEC-006 对 TASK-067 的依赖（SEC-006 汇入 TASK-067），并把 TASK-SEC-006 加入 TASK-067 的 Preconditions
- Architecture Change Required：NO

**AUD-002**
- File A：EXTERNAL-DEPENDENCIES.md:28-29（`n8n workflow source/SQL … Blocking Task TASK-044`、`n8n encryption key recovery … TASK-045`）、:33（`生产 DingTalk 凭据 … Needed Before TASK-054`）、:37（`离线/不可变备份副本 … Blocking TASK-061`）、:23（`出口防火墙白名单 … Needed Before TASK-021 | Blocking TASK-043`）
- File B：TODO.md 新编号——TASK-044=ClickHouse 部署、TASK-045=render 部署、TASK-049=n8n workflow 导入、TASK-052=邮箱/DingTalk 边界验证、TASK-061=CrashLoop 闭环、TASK-064~066=恢复演练、TASK-020=Service/Ingress/出口
- Conflict：附件中的任务编号是再生成前旧 TODO 的编号体系，与现行 TODO 不一致；同一份文件内 :23 行 Needed Before 与 Blocking Task 两列也互相矛盾
- Impact：外部依赖的 BLOCKING 门禁挂错任务——例如"生产 DingTalk 凭据"实际阻塞的是 TASK-052 而非 TASK-054，实施时可能在错误任务上等待/放行；TASK-001 的参数冻结又以本附件为输入，会把错误编号固化进参数记录
- Recommended Resolution：以当前 TODO 重排全表 40 行的 Needed Before / Blocking Task 两列（至少修正上述 5 行），并在表头注明"编号以根 TODO.md 为准"
- Architecture Change Required：NO

**AUD-003**
- File A：RESTORE-DRILL-PLAN.md:41"任一关键恢复失败则 TASK-061～TASK-064 不通过，禁止生产切换"
- File B：TODO.md:937-1021——TASK-061=CrashLoop 闭环、062=Agent 失败回滚、063=AI Ops 交接、064=CP/etcd 恢复、065=DA-SOC 恢复、066=Harbor/POP3 演练
- Conflict：恢复演练在新 TODO 中是 TASK-064~066；原文区间 TASK-061~064 混入三个 AI Ops 任务、漏掉 TASK-065/066
- Impact：恢复失败时错误地判定 AI Ops 任务不通过，且对真正的恢复演练 TASK-065/066 无约束；与 TASK-067 Preconditions（TASK-064~066）不一致
- Recommended Resolution：改为"TASK-064～TASK-066 不通过，TASK-067 阻断"
- Architecture Change Required：NO

---

## 11. Potential False Positives

以下为检查过但**判定不是真正冲突**的项：

1. **KubeSphere 4.1.x vs 候选方案中的 3.4.x**：`09-implementation/00-architecture-review/` 下的候选方案（含 kimi.md 自身）主张 3.4.x，但那是已终结的候选输入；权威文件中 README/VERSION-MATRIX/TODO/EXTERNAL-DEPENDENCIES 一致为 4.1.x Candidate。不构成冲突。
2. **Calico 版本为 unknown**：components.yaml:50 `version: unknown, status: candidate` 看似违反"版本必须冻结"，但 VERSION-MATRIX:24 同样未定版本，且 CLM `unknown-version` 规则（upgrade-rules.yaml:39-43）将未知版本显式置为高风险 REVIEW，与 README:1163"未知版本必须标记为高风险，不能被当作安全"一致。这是设计行为。
3. **Candidate / Pending-Freeze 等状态词差异**：跨文件状态词不统一（AUD-014），但同一组件在任何文件中都未出现相互矛盾的状态（没有任何文件把该组合标为 Frozen/Approved）。按 C03 规则，状态词不统一记 P2，不构成状态含义冲突。
4. **Baseline §24 Rule 2"TODO 不由本阶段修改" vs TODO 已被再生成**：规则约束的是裁决阶段；当前 TODO 头部明确"唯一架构依据 = Baseline、生成日期 2026-10-08"，正是规则要求的结果，不冲突。
5. **CLM 与两清两固的"漏洞"边界**：README:1277/V0.1 边界声明 CLM"不是完整漏洞运营体系"，README §16 把"完整漏洞运营体系"留在 V0.1 之外——两者一致（V0.1 做最小闭环，完整平台延后），无范围膨胀。
6. **n8n"2.15.0 或批准的现有兼容版本"的弹性表述**：VERSION-MATRIX、components.yaml、Adjudication §3.2 三处口径一致（当前事实为 2.15.0），不构成版本漂移。
7. **Baseline §13 允许"已批准 Runbook 临时注入 Secret"与 ADR-007 的 Agent 禁令**：Baseline 是 ADR-007 的上位细化且有"已批准、临时"限定，与 security-baseline `secret_read_is_namespace_and_workload_scoped` 一致，不是越权通道。
8. **Velero/MinIO/Promtail/ELK 在仓库中的出现位置**：grep 确认仅出现在 Adjudication §5"备份工具分歧"记录与 §24 被否决清单（"不以 Velero/MinIO 作为 V0.1 必需组件"），以及历史候选方案目录；权威实施文件中零引用。不构成第二套栈。

---

## 12. No-Issue Areas

以下关键区域经逐项核对**确认一致**：

- **架构主干**：5 VM 拓扑与故障域、单 CP + 2 Worker、Harbor/备份独立于集群、Calico default-deny、local-path/Local PV、ClickHouse 单副本节点绑定——在 Baseline、Adjudication、ADR-002~004、TODO §2 完全一致。
- **DA-SOC 承载与迁移**：ADR-001、Baseline §15/§21/§22、TODO Phase 10~13、CUTOVER-CHECKLIST、ROLLBACK-PLAN、RESTORE-DRILL-PLAN D2/D4 的切换/回退/单活/幂等/邮箱纪律口径一致。
- **版本矩阵**：OS/K8s/KubeSphere/containerd/Calico/n8n/render 全部候选版本跨 5 份文件一致，无矛盾版本号，无过早 Frozen。
- **观测唯一栈**：ADR-005、Baseline §10、TODO TASK-029~033，无 ELK/Promtail/第二套监控/独立 SIEM 的实施要求。
- **备份模型**：ADR-006、Baseline §11、Adjudication §14、RESTORE-DRILL-PLAN 的备份对象/频率（etcd 6h）/RPO 24h/RTO（CP 4h、DA-SOC 8h）/演练清单一致。
- **Agent 与 L0/L1/L2**：README §4.3、Baseline §12/§19、ADR-007/008、CLM README §6、四份安全基线、TODO TASK-036 五处一致；无 cluster-admin/root/任意 shell/长期 kubeconfig/生产邮箱凭据/生产 DingTalk 凭据授权；无 L2 绕过路径。
- **UNKNOWN 处理**：security-baseline `unknown_is_pass: false`、README:1275、Baseline §25、CLM `unknown-version` 规则、account/port/access-control 基线的 UNKNOWN→TASK 规则全链一致，未发现 UNKNOWN 被当作 PASS/SAFE 的任何位置。
- **补丁纪律**：CLM 与两清两固均复用现有 K8s/KubeSphere/Calico/Harbor/Prometheus/Fluent Bit/Loki/Git/Agent Job，未新增平台级组件、数据库、Task Center 或 Agent Runtime。

---

## 13. Final Verdict

### Q1 当前仓库是否存在阻断 V0.1 实施的 P0/P1 一致性问题？

存在 **3 个 P1，全部是文档交叉引用级**（循环依赖 1 个 + 旧任务编号残留 2 个文件），**没有 P0，没有架构级矛盾**。P1 不会阻止 TASK-001~005（参数冻结阶段）启动，但会误导 Phase 10/15 的门禁归属，必须在到达对应 Phase 前修复，且因 TASK-001 以 EXTERNAL-DEPENDENCIES 为输入，建议**在 Phase 0 内一并修订**。

### Q2 是否建议进行一次统一修订？

**是。** 一次纯文档级统一修订即可覆盖全部 3 个 P1 + 11 个 P2 + 4 个 P3：修订对象限于 README（页脚/§23/§15）、00-project 五件、EXTERNAL-DEPENDENCIES、RESTORE-DRILL-PLAN、TODO（AUD-001 依赖行 + 章节重排）、CLM 三 YAML + README（覆盖范围/词表/canonical 声明）、04-security 四 YAML 的字段值域。**不触碰 Baseline/Adjudication/ADR-001~008 的任何架构结论。**

### Q3 是否需要修改 Architecture Baseline？

**不需要。** 所有 P1/P2 均为实施附件与派生文档的漂移；Baseline 自身内部一致，且其 §24 Rule 3 已规定了偏离流程。唯一需要的人工决策是 AUD-008/AUD-009/AUD-010/AUD-011 的"canonical 声明"，建议在修订时以 ADR 或 Baseline 附录形式一句话固定，仍属文档操作。

### Q4 是否可以直接进入 TASK-001？

**可以，建议带一个前置条件。** TASK-001（参数冻结）可以立即开始；把 AUD-002（EXTERNAL-DEPENDENCIES 编号修正）作为 TASK-001 的输入修订项同步完成即可；AUD-001/AUD-003 可在 Phase 10 之前任意时间点修复，不阻塞当前。若团队希望"一次性修订后再启动"，也是合理选择——修订工作量估计为小（约半日文档工作）。

### Q5 是否建议再次进行架构设计？

**不建议。** 未发现需要重新裁决的架构问题；再次打开架构讨论将直接违反 Baseline §24 Rule 4 的精神和"停止架构讨论、进入 Implementation Preflight / TASK-001"的原则。

---

**审计员结论**：仓库主干在补丁后仍然"说的是同一件事"。三个 P1 是同一根因——"TODO 再生成后附件未同步"——的一次性修订即可清零。建议：**执行统一修订 → 交叉复核 → Baseline 保持冻结 → 直接进入 TASK-001。**
