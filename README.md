# 玄武云盾 Xuanwu SecureOps Stack

> **AI-Native Enterprise Private Cloud & Secure Operations Platform**
>
> 由 AI 协助建设、运维、审计和持续演进的企业级私有云平台。

------------------------------------------------------------------------

## 1. 项目定位

**玄武云盾（Xuanwu SecureOps Stack）** 是一个面向企业内部业务系统的 AI
原生私有云平台项目。

项目的目标不是简单搭建一套
Kubernetes，也不是简单部署一套安全产品，而是建设一个：

-   安全
-   可控
-   可审计
-   可恢复
-   可持续演进
-   可自动化运维
-   可由 AI 辅助长期维护

的企业级业务承载平台。

玄武云盾首先承载 **DA-SOC**，随后随着 DA-SOC
以及其他业务系统的发展持续演进，最终形成企业统一的私有云与安全运营底座。

------------------------------------------------------------------------

## 2. 项目背景

企业内部缺少成熟、专职的 Kubernetes / 云原生运维团队。

因此，本项目不以"培养少数几个专家依赖个人经验维护平台"为目标，而是采用
**AI-Native Platform Operations** 思路：

> **人定义目标、规则和高风险决策；AI
> 负责分析、规划、执行、验证、审计和持续维护。**

平台必须尽量做到：

> **即使没有某个专家长期驻场，其他人员也能够按照文档、策略、Runbook 和
> AI Agent 的指导完成标准化运维。**

------------------------------------------------------------------------

## 3. 最终目标

玄武云盾 V1.0 的最终目标：

> 建设一套能够长期承载企业内部核心业务的私有云平台，并通过 AI
> 完成绝大部分标准化平台运维、巡检、安全分析、故障处理和持续优化工作；所有高风险操作均具备明确的审批、审计和回滚机制。

最终形成：

``` text
                           人
                           │
                 自然语言 / 目标 / 决策
                           │
                           ▼
                ┌────────────────────┐
                │ Xuanwu AI Ops      │
                │      Manager       │
                └─────────┬──────────┘
                          │
             ┌────────────┼────────────┐
             │            │            │
          Planner       Executor     Auditor
             │            │            │
             └────────────┼────────────┘
                          │
                    Policy / Guard
                          │
                          ▼
                ┌────────────────────┐
                │ Xuanwu Platform    │
                └─────────┬──────────┘
                          │
          ┌───────────────┼────────────────┐
          │               │                │
      Infrastructure   Security          Data
          │               │                │
          └───────────────┼────────────────┘
                          │
                    Business Apps
                          │
                    ┌─────┴─────┐
                    │  DA-SOC   │
                    └───────────┘
```

------------------------------------------------------------------------

## 4. 核心理念

### 4.1 Goal-Driven

先定义最终目标，再由 Agent 将目标拆解成可执行、可验证的任务。

``` text
Goal
 ↓
Plan
 ↓
Execute
 ↓
Verify
 ↓
Operate
 ↓
Improve
```

------------------------------------------------------------------------

### 4.2 AI-Native Operations

AI 不是辅助写几条命令，而是逐步成为平台的日常运维力量。

AI 应能够：

-   发现问题
-   分析问题
-   查询上下文
-   读取平台知识
-   生成处理计划
-   执行标准操作
-   验证执行结果
-   创建待办
-   通知负责人
-   记录变更
-   更新知识
-   进行复盘

------------------------------------------------------------------------

### 4.3 Human-in-the-Loop

AI 不拥有无限权限。

根据风险等级，将操作划分为：

#### L0：自动执行

例如：

-   查询状态
-   健康检查
-   日志查询
-   资源统计
-   漏洞扫描
-   证书检查
-   生成报告

#### L1：受策略约束自动执行

例如：

-   重启异常 Pod
-   非生产环境扩容
-   清理明确判定为无用的资源
-   修复低风险配置问题

#### L2：必须人工批准

例如：

-   修改生产网络策略
-   修改 RBAC
-   删除节点
-   升级 Kubernetes
-   修改存储
-   修改核心网络
-   删除生产数据
-   关闭安全控制
-   影响业务连续性的操作

------------------------------------------------------------------------

### 4.4 Multi-Agent Audit

重要操作不能完全依赖单 Agent。

对于高风险或复杂任务，可以采用：

``` text
Planner
   ↓
Security Agent
   ↓
Reliability Agent
   ↓
Policy Auditor
   ↓
Executor
   ↓
Verifier
```

不同 Agent 从不同角度进行交叉审计，降低单一 Agent 的知识盲区和误判风险。

------------------------------------------------------------------------

### 4.5 Policy First

所有重要操作都必须遵循平台政策。

``` text
Request
 ↓
Architecture
 ↓
Policy
 ↓
Risk Evaluation
 ↓
Approval
 ↓
Execution
 ↓
Verification
 ↓
Audit
```

不能依赖个人经验决定生产平台行为。

------------------------------------------------------------------------

### 4.6 Everything Is Auditable

平台应尽可能记录：

-   谁提出请求
-   哪个 Agent 分析
-   使用了哪些证据
-   得出了什么结论
-   执行了什么操作
-   操作前状态
-   操作后状态
-   谁批准
-   谁审计
-   是否成功
-   是否回滚

------------------------------------------------------------------------

### 4.7 Everything Important Should Be Reversible

对于重要变更，应优先设计：

-   Backup
-   Snapshot
-   Rollback
-   Version Control
-   Git History
-   Change Record

不能把"恢复"作为出问题后的临时行为。

------------------------------------------------------------------------

## 5. 核心能力域

玄武云盾最终包含以下能力域：

``` text
Xuanwu SecureOps Stack
│
├── 01 Infrastructure
├── 02 Kubernetes Platform
├── 03 Network & Access
├── 04 Security
├── 05 Data & Storage
├── 06 Observability
├── 07 Operations
├── 08 Governance
├── 09 AI Operations
└── 10 Business Platform
```

### 5.1 Infrastructure

负责：

-   VM
-   裸金属
-   CPU
-   Memory
-   Disk
-   Network
-   GPU
-   基础 OS

### 5.2 Kubernetes Platform

负责：

-   Kubernetes
-   KubeSphere
-   CNI
-   CSI
-   Ingress
-   Registry
-   Namespace
-   RBAC
-   Admission / Policy

### 5.3 Network & Access

负责：

-   防火墙
-   管理网络
-   业务网络
-   DMZ
-   VPN
-   堡垒机
-   DNS
-   NTP
-   南北向访问
-   东西向访问
-   后续 Zero Trust

### 5.4 Security

负责：

-   身份认证
-   RBAC
-   Secret
-   镜像安全
-   漏洞管理
-   Endpoint Security / EDR
-   Runtime Security
-   网络安全
-   审计
-   安全基线

### 5.5 Data & Storage

负责：

-   Kubernetes / etcd
-   数据库
-   Redis
-   ClickHouse
-   对象存储
-   持久化存储
-   Backup
-   Restore
-   Disaster Recovery

### 5.6 Observability

负责：

-   Metrics
-   Logs
-   Events
-   Traces
-   Alerts
-   Health
-   Capacity

### 5.7 Operations

负责：

-   Provision
-   Patch
-   Upgrade
-   Backup
-   Restore
-   Certificate
-   Capacity
-   Incident
-   Change
-   Lifecycle

### 5.8 Governance

负责：

-   架构
-   安全基线
-   权限
-   业务边界
-   IT / 业务职责边界
-   变更管理
-   例外管理
-   资产管理
-   生命周期管理

### 5.9 AI Operations

负责：

-   AI Planner
-   AI Executor
-   AI Auditor
-   Security Agent
-   SRE / Operations Agent
-   Task Center
-   Incident Response
-   Knowledge Base
-   Multi-Agent Audit

### 5.10 Business Platform

负责承载：

-   DA-SOC
-   DA 体系相关服务
-   其他企业业务系统

------------------------------------------------------------------------

## 6. 人与 AI 的职责边界

玄武云盾遵循：

> **Human defines Goals and Policies. Agents plan, execute and verify.
> Humans approve high-risk decisions. Every action is auditable and
> reversible.**

### 人负责

-   定义最终目标
-   定义平台政策
-   确定安全红线
-   确定业务优先级
-   批准高风险操作
-   处理重大例外
-   处理重大业务决策

### AI 负责

-   信息收集
-   状态检查
-   分析
-   任务拆解
-   方案生成
-   标准化执行
-   结果验证
-   告警
-   待办创建
-   审计
-   复盘
-   知识沉淀

------------------------------------------------------------------------

## 7. IT 与业务职责边界

玄武云盾必须明确：

> **IT / 平台团队负责平台稳定性；业务团队负责应用正确性和业务结果。**

### IT / 平台负责

-   基础设施
-   Kubernetes
-   KubeSphere
-   网络
-   存储
-   Registry
-   平台安全
-   平台监控
-   平台备份
-   平台审计
-   平台生命周期

### 业务负责

-   应用代码
-   业务逻辑
-   业务数据
-   业务配置
-   业务指标
-   应用级日志
-   应用级 SLA
-   业务连续性要求

### 边界原则

业务团队：

-   不直接修改 Kubernetes 核心组件
-   不直接修改 Node
-   不直接修改 CNI
-   不直接修改集群级 RBAC
-   不绕过平台安全策略
-   不自行开放高风险网络访问
-   不绕过企业镜像仓库
-   不通过临时手工操作破坏平台状态

特殊需求必须通过平台变更 / 例外流程。

------------------------------------------------------------------------

## 8. 平台安全红线

以下原则属于玄武云盾核心安全红线。

### 管理面

-   Kubernetes API 不直接暴露互联网
-   etcd 不暴露业务网络或互联网
-   KubeSphere 管理面只允许授权管理路径访问
-   高权限账号不得作为普通业务账号使用

### 权限

-   遵循最小权限原则
-   生产环境禁止无必要的 cluster-admin
-   ServiceAccount 不得无理由使用高权限
-   权限申请必须可审计

### 容器

-   默认禁止 privileged
-   默认禁止 hostNetwork
-   默认禁止 hostPID / hostIPC
-   默认禁止不必要的 hostPath
-   默认使用非 root
-   必须设置资源 requests / limits
-   必须配置健康检查

### 网络

-   默认拒绝
-   明确允许
-   生产 Namespace 必须具备网络访问边界
-   禁止业务通过临时方式绕过网络策略

### 镜像

-   生产镜像必须来自企业认可的 Registry
-   镜像必须经过基本安全检查
-   禁止直接使用未经审核的未知镜像

### 变更

-   生产核心组件禁止随意手工修改
-   重要变更必须记录
-   重大变更必须具备回滚方案
-   禁止为了临时解决问题永久破坏架构原则

------------------------------------------------------------------------

## 9. 平台的基本运行原则

玄武云盾不依赖"记忆中的配置"。

重要状态必须尽量：

-   文档化
-   Git 化
-   自动化
-   可审计
-   可验证
-   可恢复

目标是逐渐形成：

``` text
Architecture
     ↓
Policy
     ↓
Configuration
     ↓
Automation
     ↓
Runtime
     ↓
Audit
     ↓
Knowledge
```

平台实际状态必须尽量与定义状态保持一致。

------------------------------------------------------------------------

## 10. AI 运维任务模型

所有标准化运维事件最终尽量统一为 Task。

``` text
Task
│
├── ID
├── Source
├── Asset
├── Severity
├── Description
├── Evidence
├── Impact
├── Proposed Action
├── Risk
├── Approval Required
├── Executor
├── Auditor
├── Execution Result
├── Verification Result
└── Audit Log
```

典型 Task：

-   服务器漏洞
-   EDR 告警
-   Kubernetes Node 异常
-   Pod CrashLoopBackOff
-   CPU / Memory / Disk 异常
-   证书即将过期
-   镜像漏洞
-   RBAC 异常
-   网络策略异常
-   Backup 失败
-   Configuration Drift
-   Kubernetes 升级
-   KubeSphere 异常

------------------------------------------------------------------------

## 11. 自动化运维闭环

玄武云盾最终采用：

``` text
发现
 ↓
分析
 ↓
生成 Task
 ↓
制定方案
 ↓
风险评估
 ↓
自动执行 / 请求批准
 ↓
执行
 ↓
验证
 ↓
记录
 ↓
通知
 ↓
知识沉淀
```

例如：

``` text
EDR
 ↓
告警
 ↓
n8n / Event Trigger
 ↓
Security Agent
 ↓
调查主机 / Pod / 网络 / 日志
 ↓
生成安全事件
 ↓
创建待办
 ↓
钉钉通知
 ↓
高风险操作请求人工批准
 ↓
Executor 执行
 ↓
Verifier 验证
 ↓
Audit Agent 审计
```

------------------------------------------------------------------------

## 12. n8n 与 Agent 的职责

n8n 或同类工作流工具主要负责：

-   Scheduler
-   Webhook
-   Email
-   EDR Event
-   DingTalk
-   API Integration
-   Notification
-   Workflow Trigger

它不应该成为整个系统的"大脑"。

Agent 负责：

-   理解
-   判断
-   规划
-   执行
-   验证
-   审计

推荐关系：

``` text
n8n
 │
 ├── 定时任务
 ├── Webhook
 ├── EDR
 ├── Email
 └── DingTalk
        │
        ▼
   AI Agent System
        │
        ▼
   Xuanwu Platform
```

------------------------------------------------------------------------

## 13. 文档即平台知识

玄武云盾必须把平台知识纳入 Git。

重要知识包括：

-   Architecture
-   Governance
-   Security Policy
-   Runbook
-   ADR
-   Incident
-   Change
-   Asset
-   Implementation Plan

Agent 在执行任务前，应优先读取：

``` text
README.md
 ↓
相关 Architecture
 ↓
相关 Policy
 ↓
相关 Runbook
 ↓
相关 ADR
 ↓
执行
```

Agent 不应在没有读取相关平台上下文的情况下随意修改核心平台。

------------------------------------------------------------------------

## 14. 项目目录

项目采用以下逻辑结构：

``` text
xuanwu-secureops-stack/
│
├── README.md
│
├── 00-project/
│   ├── VISION.md
│   ├── GOALS.md
│   ├── SCOPE.md
│   ├── PRINCIPLES.md
│   └── VERSIONING.md
│
├── 01-architecture/
│
├── 02-governance/
│
├── 03-platform/
│
├── 04-security/
│
├── 05-operations/
│
├── 06-runbooks/
│
├── 07-aiops/
│
├── 08-business/
│
├── 09-implementation/
│
├── 10-decisions/
│
├── 11-incidents/
│
├── 12-assets/
│
├── manifests/
├── configs/
└── scripts/
```

README 是项目最高层级的总纲；专业细节必须进入对应目录，不应无限堆积到
README。

------------------------------------------------------------------------

## 15. 版本路线

玄武云盾采用持续演进方式。

### V0.1 --- DA-SOC 承载平台

目标：

> 安全、稳定、可维护地承载 DA-SOC v0.1。

主要能力：

-   Kubernetes
-   KubeSphere
-   基础网络
-   CNI
-   基础 NetworkPolicy
-   Harbor
-   RBAC
-   基础监控
-   基础日志
-   基础审计
-   基础备份
-   基础 AI Ops
-   组件生命周期管理（CLM）最小闭环
-   两清两固安全运营最小闭环（清高危漏洞、清高危端口、固弱账号口令、固弱访问控制）

V0.1 不追求一次性建设完整安全体系。

------------------------------------------------------------------------

### V0.2 --- 可运维平台

增加：

-   自动巡检
-   漏洞检查
-   证书检查
-   Backup 检查
-   资源检查
-   AI Task Center
-   钉钉告警
-   Runbook
-   基础自动修复

------------------------------------------------------------------------

### V0.3 --- 安全平台

增加：

-   EDR
-   镜像漏洞扫描
-   容器安全
-   Runtime Security
-   更完整的审计
-   安全事件管理
-   SIEM 能力

------------------------------------------------------------------------

### V0.4 --- AI 运维平台

增加：

-   Planner
-   Executor
-   Auditor
-   Multi-Agent
-   Policy Engine
-   自动修复
-   变更审批
-   自动验证
-   自动回滚
-   Configuration Drift Detection

------------------------------------------------------------------------

### V0.5 --- 企业私有云平台

增加：

-   多业务承载
-   多租户
-   资源配额
-   服务目录
-   标准化业务接入
-   SLA
-   生命周期管理

------------------------------------------------------------------------

### V1.0 --- 企业级 AI-Native SecureOps Platform

达到：

> 玄武云盾能够长期承担企业内部核心业务，并能够由 AI
> 协助完成绝大部分标准化平台运维、安全运营和故障处理，同时确保重大操作可审批、可审计、可恢复。

------------------------------------------------------------------------

## 16. V0.1 的明确边界

V0.1 的唯一核心目标：

> **让 DA-SOC v0.1 在玄武云盾上稳定运行，并验证 AI-Native 运维模式。**

V0.1 暂不强制完成：

-   完整 SIEM
-   完整 Zero Trust
-   完整 Runtime Security
-   完整漏洞运营体系
-   完整软件供应链安全
-   完整多集群
-   完整灾备体系
-   完整自动修复体系

这些能力进入后续版本。

------------------------------------------------------------------------

## 17. DA-SOC 与玄武云盾的关系

玄武云盾不是 DA-SOC 的组成部分。

关系为：

``` text
Xuanwu SecureOps Stack
│
├── Infrastructure
├── Kubernetes
├── Security
├── Operations
├── AI Ops
│
└── Business Platform
     │
     ├── DA-SOC
     ├── 业务系统 A
     ├── 业务系统 B
     └── 未来业务系统
```

DA-SOC
是玄武云盾的第一个核心业务承载对象，同时也是验证玄武云盾架构的第一个实际业务。

------------------------------------------------------------------------

## 18. Agent 开发与实施原则

所有重大建设任务建议采用 Multi-Agent 模式。

推荐：

``` text
Task
 ↓
Architect Agent
 ↓
Security Agent
 ↓
Operations Agent
 ↓
Domain Agent
 ↓
Reviewer Agent
 ↓
Final Agent
```

最终输出必须：

-   有明确目标
-   有明确范围
-   有前置条件
-   有执行步骤
-   有验证步骤
-   有失败处理
-   有回滚方案
-   有安全检查
-   有完成标准

------------------------------------------------------------------------

## 19. Agent 执行任务的基本要求

任何 Agent 在修改玄武云盾之前：

1.  必须读取 README.md。
2.  必须确认当前版本。
3.  必须读取相关架构文档。
4.  必须读取相关治理规则。
5.  必须读取相关 Runbook / ADR。
6.  必须判断任务风险等级。
7.  必须避免违反平台红线。
8.  涉及高风险操作时必须请求人工批准。
9.  执行后必须验证。
10. 必须记录变更。

------------------------------------------------------------------------

## 20. 变更原则

玄武云盾遵循：

> **Change the definition, then change the platform.**

即：

``` text
需求
 ↓
设计
 ↓
文档 / Policy
 ↓
Review
 ↓
实施
 ↓
验证
 ↓
记录
```

不鼓励：

``` text
先直接改生产
 ↓
成功了再说
```

特别是平台核心组件、网络、安全策略、存储和权限。

------------------------------------------------------------------------

## 21. 故障原则

故障发生时：

> **先保护业务，再恢复平台，最后分析根因。**

推荐：

``` text
Detect
 ↓
Assess
 ↓
Contain
 ↓
Recover
 ↓
Verify
 ↓
Root Cause Analysis
 ↓
Corrective Action
 ↓
Knowledge Update
```

每次重大故障都应形成 Incident Record。

------------------------------------------------------------------------

## 22. 成功标准

玄武云盾不是以"组件安装完成"为成功标准。

真正的成功标准是：

### 平台

-   能稳定运行
-   能监控
-   能备份
-   能恢复
-   能审计
-   能升级
-   能发现异常

### 安全

-   有明确安全边界
-   有最小权限
-   有网络隔离
-   有镜像安全控制
-   有审计
-   有漏洞管理路径

### 运维

-   有标准 Runbook
-   有自动巡检
-   有任务中心
-   有通知机制
-   有故障处理流程

### AI

-   AI 能理解平台
-   AI 能执行标准任务
-   AI 能验证结果
-   AI 能发现问题
-   AI 能创建待办
-   AI 能通知人
-   多 Agent 能互相审计
-   高风险操作由人决策

### 治理

-   IT / 业务边界清晰
-   平台红线明确
-   变更可追踪
-   例外可审计
-   架构持续与实际状态保持一致

------------------------------------------------------------------------

## 23. 当前项目状态

**当前版本：V0.1 --- Architecture Frozen / Consistency Closed / Implementation Ready**

| 维度 | 当前状态 |
|---|---|
| Architecture | **FROZEN** —— 唯一实施依据为 `10-decisions/ARCHITECTURE-BASELINE-V0.1.md`（Approved Baseline） |
| Architecture Decision | `10-decisions/ADR/ADR-001` ~ `ADR-008`（Accepted） |
| Consistency Audit | 已完成 6 份独立横向一致性审计，见 `audit/` |
| Consistency Closure | 正在完成最终 P0/P1 收口；P2/P3 记录不实施 |
| Implementation TODO | `TODO.md` 为唯一实施任务源；从 `TASK-001` 开始 |
| Installation | **NOT STARTED** —— 尚未安装任何组件 |

**下一阶段：** 一致性收口完成并经人工确认后，从 `TASK-001`（Phase 0 参数冻结）开始实施。
本阶段**不是** Implementation Completed，也**不是** Production Ready；当前仅为 **Implementation Ready**。

### V0.1 OS 与云原生版本基线

当前记录的是候选基线，不是官方认证的完整组合。**版本唯一事实来源为 `09-implementation/VERSION-MATRIX.md`**；下表为摘要，不得作为第二份版本表维护：

| Component | Candidate | Status |
|---|---|---|
| OS | 统信服务器操作系统 V20 1060e AMD64（免费使用授权） | Candidate / Pending Compatibility Validation |
| Kubernetes | v1.30.6 | Candidate / Pending Compatibility Validation |
| KubeSphere | 4.1.x，优先验证 4.1.2 | Candidate / Pending Compatibility Validation |
| containerd | 1.7.x | Candidate / Pending Compatibility Validation |
| CNI | Calico | Candidate / Pending Compatibility Validation |

上述组合必须经过官方兼容矩阵核对、OS 介质校验、实际安装、节点加入、网络、存储、Harbor、观测、DA-SOC 和恢复验证后，才能在 `09-implementation/VERSION-MATRIX.md` 中冻结。当前安装尚未开始。

### V0.1 Component Lifecycle Management

玄武云盾 V0.1 同时具备组件生命周期管理（CLM）最小闭环。CLM 使用 Git 中的 `07-aiops/component-lifecycle/` 作为期望状态、策略和审计规则的 Source of Truth，由短生命周期 Job 按组件类型自动发现 OS、Kubernetes、KubeSphere、containerd、Calico、Harbor、DA-SOC 和 Observability 的当前版本与镜像 digest。

CLM 将 **Component Registry**、**Vulnerability State** 和 **Upgrade State** 分开维护；它可以发现版本、记录 CVE/CVSS/KEV/EOL、解释 `upgrade_required`、生成 Git Upgrade Task、执行 L0/L1/L2 审批、验证和回退，但不是独立漏洞平台，也不会自动升级生产环境。未知版本必须标记为高风险，不能被当作安全。

当前首要任务：

1.  完成玄武云盾项目顶层设计
2.  完成 V0.1 架构设计
3.  完成平台治理和安全基线
4.  完成 AI Ops 最小模型
5.  规划测试环境基础设施
6.  建设 KubeSphere Kubernetes 平台
7.  将 DA-SOC v0.1 部署到玄武云盾
8.  验证 AI 辅助运维闭环

------------------------------------------------------------------------

## 24. 第一阶段实施顺序

第一阶段不直接追求 V1.0。

建议：

``` text
Project Charter
        ↓
Overall Architecture
        ↓
Governance
        ↓
Security Baseline
        ↓
AI Ops Model
        ↓
V0.1 Implementation Plan
        ↓
Infrastructure
        ↓
Kubernetes
        ↓
KubeSphere
        ↓
Security Baseline
        ↓
Observability
        ↓
DA-SOC
        ↓
AI Ops MVP
        ↓
V0.1 Validation
```

------------------------------------------------------------------------

## 25. 长期原则

玄武云盾始终遵循以下原则：

1.  **安全优先，但不过度复杂化。**
2.  **自动化优先，但高风险操作必须有人负责。**
3.  **策略优先于个人经验。**
4.  **文档优先于记忆。**
5.  **可审计优先于"方便"。**
6.  **可恢复优先于"相信不会出问题"。**
7.  **最小权限优先。**
8.  **默认拒绝，明确允许。**
9.  **平台与业务解耦。**
10. **AI 可以执行，但不能拥有无限权力。**
11. **多个 Agent 可以互相审计。**
12. **所有重要变化都必须留下证据。**
13. **平台必须能够持续演进。**
14. **V0.1 解决实际问题，不追求一次性完美。**
15. **最终目标是让平台不依赖某一个人的个人知识。**

------------------------------------------------------------------------

## 26. 项目最终愿景

玄武云盾最终希望实现：

> **一个人 + 一组 AI Agent +
> 一套清晰的架构、策略和自动化体系，可以长期管理一个企业级私有云平台。**

人不再需要记住：

-   哪台服务器出了问题
-   哪个证书快过期
-   哪个 Pod 异常
-   哪个节点有漏洞
-   哪条网络策略被修改
-   哪个备份失败
-   哪个业务需要升级

这些事情由平台自动发现、AI 自动分析、Agent 自动处理或生成待办。

人只需要：

> **定义目标、制定规则、处理例外、做最终决策。**

这就是玄武云盾的最终目标。

------------------------------------------------------------------------

**Project:** Xuanwu SecureOps Stack / 玄武云盾\
**Current Version:** V0.1\
**Current Mission:** Securely host DA-SOC v0.1 and validate AI-Native
Platform Operations\
**Status:** Architecture Frozen / Consistency Closed / Implementation Ready
（Installation: NOT STARTED）

## 27. V0.1 两清两固安全运营能力

玄武云盾 V0.1 在既有架构上增加“两清两固”最小可管理闭环：清高危漏洞、清高危端口、固弱账号口令、固弱访问控制。高危漏洞复用 CLM；端口、账号和访问控制的期望状态分别由 `04-security/port-baseline.yaml`、`04-security/account-baseline.yaml`、`04-security/access-control-baseline.yaml` 管理，统一状态模型由 `04-security/security-baseline.yaml` 定义。

四项能力都必须完成：发现 → 判断 → 任务 → 整改 → 验证 → 审计。`UNKNOWN` 不得默认为 PASS；真实密码、Token、Secret 值、邮箱正文和业务载荷不得进入 Git、日志、DingTalk、报告或 Agent 上下文。生产 RBAC、核心防火墙/NetworkPolicy、生产账号、Harbor 管理权限和 DA-SOC 访问控制继续使用 L2 人工审批。

V0.1 不新增 SIEM、SOAR、CMDB、NDR、完整 IAM、完整漏洞管理平台或独立安全基础设施，也不承诺自动修复所有风险。
