# ADR-008: V0.1 Task 使用 Git YAML/Markdown

- 状态：Accepted
- 日期：2026-10-08

## Decision

V0.1 Task 使用 Git 中的 YAML/Markdown 文件，不创建 Kubernetes Task CRD，不建设独立 Web Task Center。

每个 Task 至少包含：ID、来源、目标、严重性、风险等级、证据、影响、计划、回滚、审批、Agent Job、执行结果、验证结果、审计 commit 和最终状态。

## Lifecycle

**本 ADR 是运维 Task 生命周期（Task Lifecycle）的唯一权威定义。**

```text
open → analyzed → planned → pending-approval → approved/rejected
→ executing → verified/failed → closed/escalated
```

**状态维度分工（不得混用，不得新建第五套）：**

| 维度 | 权威来源 | 状态词表 |
|---|---|---|
| 运维 Task 生命周期 | **本 ADR** | `open` / `analyzed` / `planned` / `pending-approval` / `approved` / `rejected` / `executing` / `verifying` / `verified` / `failed` / `closed` / `escalated` / `rolled-back` |
| Task 字段 Schema | `10-decisions/ARCHITECTURE-BASELINE-V0.1.md` §16.3 | `api_version` / `kind` / `spec` / `approval` / `execution` / `verification` / `audit` |
| 安全检查 / finding 状态 | `04-security/security-baseline.yaml` | `PASS` / `FAIL` / `REVIEW` / `UNKNOWN` + `finding_lifecycle` |
| 组件升级状态 | `07-aiops/component-lifecycle/upgrade-rules.yaml` | `lifecycle_statuses` |
| 版本冻结状态 | `09-implementation/VERSION-MATRIX.md` | `candidate` / `validating` / `frozen` / `superseded` |
| 实施任务状态（仅 TODO.md 内部） | `TODO.md` | `PENDING` / `IN_PROGRESS` / `BLOCKED` / `PASSED` / `ROLLED_BACK` |

安全类与组件类 Task 只允许在 §16.3 的 Schema 之上**扩展字段**（如 `04-security/security-baseline.yaml` 的 `extension_fields`），不得定义第二套 Task Schema。

n8n 负责定时、告警、DingTalk 和 Task 触发；Agent Job 负责分析、执行和验证；Git 保存不可变审计记录，Loki 保存运行日志。

## Security

Task 文件不得包含明文 Secret、邮箱密码、DingTalk token、私钥或业务数据。证据使用引用、摘要和时间范围；敏感值只在受控运行时读取。

## Why Not CRD

V0.1 Task 规模和查询需求不足以抵消 CRD 的 Schema、Controller、升级、备份和 RBAC 复杂度。V0.2 根据真实 Task 数量、跨 Task 查询和并发审批需求评估 CRD 或轻量数据库。