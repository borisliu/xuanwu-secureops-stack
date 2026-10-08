# Component Lifecycle Management

> 玄武云盾 V0.1 的组件生命周期管理（CLM）最小闭环规范。

## 1. Scope

CLM 管理平台运行所需的 OS、Kubernetes、KubeSphere、containerd、Calico、Harbor、ClickHouse、n8n、render/archive、Prometheus、Grafana、Alertmanager、Fluent Bit、Loki、Agent 镜像及其他生产容器镜像。

CLM 不是独立漏洞平台、完整 SBOM 平台或自动 Patch Management 产品。V0.1 只提供：

```text
Component Registry
  → Discovery
  → Vulnerability / EOL State
  → Upgrade Decision
  → Git Task
  → Human Approval
  → Execute
  → Verify
  → Audit
```

## 2. Three-State Model

### Component Registry

回答“平台管理什么”，由 `components.yaml` 定义组件身份、Owner、环境、关键性、期望版本和支持策略。

### Vulnerability State

回答“当前运行版本有什么风险”，由发现结果记录 CVE、CVSS、KEV、是否受影响、修复版本、来源和最后检查时间。

### Upgrade State

回答“现在是否需要做什么”，记录当前版本、目标版本、最新安全版本、升级原因、优先级、审批、执行、验证和回退状态。

三类状态不得合并为一个人工维护的版本字段。期望状态和策略在 Git；实际运行状态由发现 Job 写入审计结果或受控状态 artifact。

最小记录形状如下，组件清单可以按组件类型扩展，但不得删除三类状态的边界：

```yaml
component: { id: ", name: ", type: ", owner: ", environment: [], criticality: " }
desired: { version: ", digest: ", support_policy: " }
discovery: { method: ", endpoint: ", last_seen: " }
security: { highest_severity: ", cve_count: 0, kev: false, affected: null, fixed_version: " }
lifecycle: { support_status: ", eol: null, eos: null }
upgrade: { required: ", reason: [], priority: ", target_version: ", approval_required: true, status: " }
audit: { last_checked: ", source: " }
```

## 3. Discovery Methods

| Component Type | Discovery Method | Required Facts |
|---|---|---|
| UOS | `/etc/os-release`、`uname`、RPM/package metadata | OS、架构、内核、安装包版本 |
| Kubernetes | Kubernetes API | server、node、kubelet、组件镜像 |
| KubeSphere | Kubernetes resources / KubeSphere API | 控制面版本、组件状态、镜像 |
| containerd | 节点命令 / package manager | 运行版本、配置、CRI 状态 |
| Calico | Kubernetes API/CRD/Pod image | operator、node、CRD、镜像版本 |
| Harbor | Harbor API | 版本、项目、扫描状态、镜像 |
| Container image | Kubernetes workload / Harbor API | repository、tag、digest |
| ClickHouse | HTTP API / SQL / image | server version、schema、image digest |
| n8n/render/archive/observability | API + workload image/digest | 应用版本、运行状态、digest |

无法自动发现的版本必须标记为 `unknown_version`，进入风险和人工确认队列，不得当作“无漏洞”或“无需升级”。

## 4. Version Truth

- `09-implementation/VERSION-MATRIX.md`：批准的期望版本、候选状态和冻结条件。
- Kubernetes workload `image digest`：实际部署事实；tag 只用于人类可读显示。
- Git CLM YAML：组件、策略、升级规则和审批边界的 Source of Truth。
- Discovery evidence：实际运行状态、来源、采集时间和 hash；不手工覆盖。

## 5. Upgrade Policy

`upgrade_required` 不能由“存在 CVE”单独决定：

- Critical + 实际受影响 + 存在修复版本：`true`，至少 P0/P1。
- High + 实际受影响 + 存在修复版本：`true`，至少 P1/P2。
- KEV：提升优先级；是否受影响仍需记录。
- EOL/EOS：`true`，原因必须为生命周期风险。
- 存在 CVE 但 `affected: false`：不得机械升级，保留审计依据并进入 REVIEW。
- 版本未知：高风险状态，必须先完成发现或人工确认。

升级原因必须可解释，例如 `CVE`、`CVSS_CRITICAL`、`KEV`、`EOL`、`SECURITY_PATCH`、`POLICY`。

## 6. Priority and Risk Boundary

- `P0`：KEV、Critical、互联网暴露或核心生产组件。
- `P1`：Critical/High 的核心平台组件。
- `P2`：High/Medium 的非核心生产组件。
- `P3`：Low、非生产或生命周期优化。

风险等级遵守现有 `L0/L1/L2`：

- `L0`：自动发现、漏洞/EOL 判断、报告和 Git Task 创建。
- `L1`：验证环境 Patch、非生产组件或可回滚白名单 Runbook。
- `L2`：Kubernetes、KubeSphere、Calico、Harbor、OS、ClickHouse、生产镜像、节点、存储、网络和 RBAC 等必须人工审批。

生产环境禁止自动升级；Agent 可以分析、生成 Task、执行批准的 Runbook、验证和审计，但不得绕过审批。

## 7. V0.1 Boundary

### Implemented / Planned in V0.1

- Git YAML Component Registry、策略和升级规则。
- 组件版本和镜像 digest 发现。
- 基础 CVE/KEV/EOL 状态记录，支持离线导入。
- 可解释的 `upgrade_required`、原因、优先级和截止时间。
- Git Upgrade Task、L0/L1/L2 审批、短生命周期 Job、验证、审计和失败回退。
- Component Coverage、Discovery Freshness、Vulnerability Freshness、Critical Upgrade SLA、Upgrade Closure Rate、EOL Count、Unknown Version Count 指标。

### Deferred to V0.2/V0.3

- 完整 SBOM 平台。
- 独立漏洞管理平台、SIEM、CMDB、SOAR。
- 自动 Kubernetes minor/major upgrade、自动 OS 升级、跨版本升级和生产自动 Patch。
- 自动兼容性证明、复杂依赖图、全自动补丁编排。

## 8. Audit and Evidence

每次发现、判断、Task、审批、执行和验证必须带上 `component_id`、source、采集时间、当前 digest/版本、Git commit、Task ID、审批人、结果和回退记录。Secret、token、邮箱正文和业务数据不得进入 CLM 状态或日志。

## 9. Operating Metrics

| Metric | Definition | V0.1 Target |
|---|---|---|
| Component Coverage | 已纳入 CLM 管理的组件 / 应管理组件 | 100% |
| Version Discovery Freshness | 距最近一次成功 discovery 的时间 | ≤ 24h |
| Vulnerability Assessment Freshness | 距最近一次漏洞/生命周期评估的时间 | ≤ 7d；Critical/KEV 事件立即复核 |
| Critical Upgrade SLA | Critical/KEV 发现到 Git Upgrade Task 创建的时间 | 按 P0 窗口，必须可测量 |
| Upgrade Closure Rate | 已验证关闭的升级 Task / 应升级 Task | 持续上升；未关闭项必须有 Owner 和期限 |
| EOL Component Count | 当前 EOL/EOS 组件数量 | 0；例外必须有批准期限 |
| Unknown Version Count | 无法自动发现当前版本的组件数量 | 0 |
## 10. Two-Clear-Two-Firm Security Operations

CLM supplies the vulnerability and lifecycle part of the V0.1 security loop. The other three controls use the declarative baselines in `04-security/`:

- `security-baseline.yaml` defines the shared Desired State to Observed State to Deviation to Risk to Task to Approval to Remediation to Verification to Audit model.
- `port-baseline.yaml` defines host listeners, Kubernetes Service/NodePort/LoadBalancer/Ingress exposure, Harbor and Kubernetes API boundaries.
- `account-baseline.yaml` defines UOS, Kubernetes, KubeSphere and Harbor account inventory, privileged-account review and credential handling.
- `access-control-baseline.yaml` defines host, Kubernetes, network and application/platform access boundaries.

All four capabilities use `PASS`, `FAIL`, `REVIEW` and `UNKNOWN`; `UNKNOWN` is never treated as safe. A finding that deviates from the desired state creates a Git Task with category `VULNERABILITY`, `PORT`, `ACCOUNT` or `ACCESS_CONTROL`.
Evidence records metadata, hashes, redacted output, source and timestamp only; passwords, tokens, Secret values and mailbox bodies never enter Git, logs, DingTalk or Agent context. The V0.1 closure target is continuous manageability, not universal automatic remediation: discover, assess, create a task, remediate through an approved L0/L1/L2 path, rescan or re-audit, verify and close with evidence.

Production RBAC, core firewall rules, Calico NetworkPolicy, Kubernetes API exposure, Harbor administration, production credentials and DA-SOC access-control changes remain L2 human-approved actions.

## 11. Two-Clear-Two-Firm Metrics

The existing Prometheus/Grafana/Alertmanager stack exposes the minimum measures below without introducing a separate security platform.

| Metric | V0.1 target |
|---|---|
| Security component coverage | 100% of managed components and images |
| Vulnerability assessment coverage | Current assessment or visible `UNKNOWN` for every managed component |
| Port baseline coverage | 100% of managed hosts and critical services |
| Privileged-account visibility | 100% of UOS, Kubernetes, KubeSphere and Harbor privileged identities |
| RBAC/access-control audit coverage | 100% of declared host, Kubernetes, network and platform controls |
| Unknown finding count | Visible and owned; never silently passed |
| Remediation closure rate | Closed findings have verification evidence and an owner |
| Audit completeness | Discovery, evidence, task, execution, verification and closure present |
