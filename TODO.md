# 玄武云盾 V0.1 最终实施任务清单

> **Implementation Readiness: READY**
>
> 当前任务清单可以从 `TASK-001` 开始执行。`READY` 表示实施规划和架构输入已经具备；具体安装任务仍必须遵守 Phase 0 参数冻结、外部依赖和每项任务的前置条件。任何实现阶段发现的架构冲突必须停止并记录为 Architecture Blocker，不得在本清单中擅自改写 Baseline 或 ADR。

- 版本：V0.1
- 生成日期：2026-10-08
- 唯一架构依据：`10-decisions/ARCHITECTURE-BASELINE-V0.1.md`
- 决策依据：`10-decisions/ADR/ADR-001-da-soc-hosting.md` 至 `ADR-008-task-model.md`
- 实施范围：平台基础设施、Kubernetes/KubeSphere、DA-SOC 承载迁移、备份恢复、AI Ops MVP 和最终验收
- 明确不做：创建第二份 TODO、重新设计架构、直接开始 Kubernetes 实施、把 V0.2/V1.0 组件提前引入；本次仅同步根 `TODO.md` 中既有版本/实施任务。

## 0. Version Baseline Status

| 项目 | 当前状态 |
|---|---|
| Architecture | FROZEN |
| Implementation TODO | READY；从 `TASK-001` 开始 |
| OS | 统信服务器操作系统 V20 1060e AMD64；Candidate / Pending Compatibility Validation |
| Kubernetes | v1.30.6；Candidate / Pending Compatibility Validation |
| KubeSphere | 4.1.x，优先验证 4.1.2；Candidate / Pending Compatibility Validation |
| containerd | 1.7.x；Candidate / Pending Compatibility Validation |
| CNI | Calico；Candidate / Pending Compatibility Validation |
| VM Resource Request | READY |
| Version Matrix | Pending Freeze |
| Installation | NOT STARTED |

> 上述 OS + Kubernetes + KubeSphere + containerd + Calico 仅是 V0.1 候选组合，不代表官方认证的完整组合。只有完成官方兼容矩阵核对和项目实际安装验证后，才能改为 `Frozen`。

## 1. Implementation Overview

玄武云盾 V0.1 必须建设一个 5 台 VM、1 Control Plane + 2 Worker 的单集群 KubeSphere 私有云，并在 `da-soc` Namespace 实际承载 n8n、ClickHouse、render/archive 和 raw archive。Harbor 与备份仓库位于集群外。生产迁移采用“验证双跑、生产单活”：允许隔离回放和测试邮箱验证，禁止 ECS n8n 与 Kubernetes n8n 同时消费生产邮箱。

实施顺序固定为：参数冻结 → 基础设施 → Kubernetes → KubeSphere → Calico/网络 → Local PV → Harbor → 观测 → 安全 → 备份 → DA-SOC 部署 → 数据迁移 → 业务验证 → 切换/回退 → AI Ops MVP → 恢复演练 → 最终验收。

## 2. Frozen Architecture Reference

| 项目 | 冻结结论 |
|---|---|
| VM | `xw-cp-01`、`xw-wk-01`、`xw-wk-02`、`xw-harbor-01`、`xw-backup-01` |
| Kubernetes | 单集群，1 Control Plane + 2 Worker，containerd，etcd 位于 `xw-cp-01` |
| KubeSphere | 部署于现有集群管理面，不改变 Kubernetes 拓扑 |
| CNI | Calico；默认拒绝并显式放行 |
| Registry | Harbor 独立 VM，镜像以固定版本和 digest 使用，支持离线导入 |
| Storage | local-path/Local PV；ClickHouse、n8n、raw archive 使用节点绑定 PVC |
| Observability | Prometheus + Grafana + Alertmanager + Fluent Bit + Loki；关键告警进入运维 DingTalk |
| Backup | Git + etcd snapshot + ClickHouse BACKUP/RESTORE + 文件备份 + Harbor 备份 |
| Agent | 短生命周期 Kubernetes Job，默认只读；L0/L1/L2 分级；不使用常驻高权 Agent |
| Task | Git 中 YAML/Markdown Task，不使用 Task CRD |
| DA-SOC Namespace | 正式生产为 `da-soc`；临时验证为 `da-soc-validate`，验证后删除 |
| DA-SOC 单活 | K8s 切换后 K8s n8n 是唯一生产消费者，ECS 仅短期回退源 |
| 业务硬约束 | 数字只来自 ClickHouse SQL；图片只由 render；LLM 不参与计算/出图；archive 失败则不入库、不出图、不发送；生产邮箱不 Mark as Read/删除/修改；生产/测试 DingTalk 严格隔离 |
| V0.1 不引入 | `xw-opsapi`、Ceph、Longhorn、ELK、Istio/Linkerd、GPU、多集群、复杂 Policy Engine、Task CRD |

## 3. Implementation Rules

1. 所有版本、镜像、digest、证书和变更记录进入 Git；禁止 `latest`。
2. 所有任务都必须保留 Validation、Evidence、Rollback；没有证据不算完成。
3. 生产邮箱只允许经批准的单活消费者读取；测试验证使用测试邮箱或脱敏回放，不得通过双生产读取实现“双跑”。
4. 生产数据、Secret、token、邮箱正文和 DingTalk 凭据不得明文进入 Git、Loki 或审计日志。
5. 所有高风险动作走 Git Task、审批和审计；Agent 不得修改架构、RBAC、NetworkPolicy、StorageClass、节点、etcd 或生产数据。
6. 任何关键门禁失败即 NO-GO；不得用“先上线再修”替代 Cutover Gate。
7. 实施发现架构矛盾时，状态为 `Architecture Blocker`，暂停受影响任务并提交架构 Owner；不得私自扩展组件。
8. Baseline 和 ADR 在本阶段只读；本次版本基线补充允许修订根目录 `TODO.md` 及 `09-implementation/` 的实施附件，但不得创建第二份 TODO 或修改已冻结架构结论。

## 4. Phase 0 — Parameter Freeze

### TASK-001 — 建立实施参数冻结记录
- **Objective:** 建立唯一参数冻结工作项和责任人矩阵。
- **Preconditions:** 已阅读 README、Baseline、ADR-001～008。
- **Inputs:** 架构文件、`09-implementation/` 附件、Owner 名单。
- **Actions:** 创建变更记录；登记 Platform/Security/DA-SOC/Backup/Business/Network/Registry Owner；标记所有 TBD 参数；登记 UOS Server V20 1060e AMD64、Kubernetes v1.30.6、KubeSphere 4.1.x（优先验证 4.1.2）、containerd 1.7.x 和 Calico 的 Candidate 状态及其验证 Owner。
- **Validation:** Owner、审批人、回退负责人均已实名；没有未归属的关键参数。
- **Expected Output:** 参数冻结记录、责任矩阵、变更编号。
- **Risk:** 责任不清导致安装绕过门禁。
- **Approval:** Platform Owner、Security Owner、DA-SOC Owner。
- **Rollback:** 删除未批准的参数草案，不影响 Baseline。
- **Evidence:** Git commit、责任矩阵、会议/审批记录。
- **Dependencies:** 无。
- **Definition of Done:** 所有冻结项均有 Owner、状态和截止时间；OS/云原生候选基线已写入 `VERSION-MATRIX.md`，但未被误标为 Frozen。

### TASK-002 — 冻结版本、镜像和 digest
- **Objective:** 将候选版本、官方兼容矩阵、实际验证结果和镜像 digest 转为可执行的版本冻结记录。
- **Preconditions:** TASK-001 完成；可访问 KubeSphere 官方兼容矩阵和离线包。
- **Inputs:** `09-implementation/VERSION-MATRIX.md`、官方兼容性矩阵、镜像仓库清单。
- **Actions:** 记录 UOS Server V20 1060e AMD64、Kubernetes v1.30.6、KubeSphere 4.1.x（优先验证 4.1.2）、containerd 1.7.x 和 Calico 的候选状态；核对 KubeSphere 官方兼容矩阵；选择 kernel、local-path、Harbor、Ingress、Prometheus、Grafana、Alertmanager、Fluent Bit、Loki、ClickHouse、Agent、SOPS/age 的精确版本；记录 digest、校验和、来源和回滚版本。
- **Validation:** 兼容矩阵和实际安装验证均通过；明确区分官方资料、项目组合验证和历史安全评估；所有生产镜像均非 `latest`；离线包可校验。
- **Expected Output:** 冻结后的版本矩阵和镜像 manifest。
- **Risk:** 版本不兼容造成集群或 DA-SOC 不可用。
- **Approval:** Platform Owner、Security Owner、DA-SOC Owner。
- **Rollback:** 不安装未冻结版本，回到待冻结状态。
- **Evidence:** 兼容矩阵快照、digest、SHA256、审批 commit。
- **Dependencies:** TASK-001。
- **Definition of Done:** `VERSION-MATRIX.md` 中所有安装相关参数均为 Frozen，或有明确 Pending/Blocked 状态、Owner、替代路径和截止时间；未经组合验证不得宣称 UOS/Kubernetes/KubeSphere 为官方认证组合。

### TASK-003 — 冻结 Secret 管理和恢复方案
- **Objective:** 确定 Secret 的生成、注入、轮换、备份和灾备恢复路径。
- **Preconditions:** TASK-001 完成；已识别 n8n encryption key、邮箱、DingTalk、Harbor、备份凭据。
- **Inputs:** Baseline Security/Backup 章节、SOPS/age 参数、企业密钥保管要求。
- **Actions:** 采用加密文件管理 Secret；明文只在受控运行时注入；建立恢复密钥双人保管、轮换周期、紧急吊销和恢复演练流程；禁止 Secret 值进入 Git/Loki。
- **Validation:** 在隔离环境恢复一个测试 Secret；审计记录只含 metadata；丢失单个密钥时有明确处置。
- **Expected Output:** Secret 管理决策、密钥清单、恢复 Runbook。
- **Risk:** 密钥丢失导致无法启动业务或恢复。
- **Approval:** Security Owner、DA-SOC Owner、Backup Owner。
- **Rollback:** 不启用未验证的 Secret 注入方式，保留原受控凭据。
- **Evidence:** 加密文件、密钥托管记录、恢复测试日志。
- **Dependencies:** TASK-001、TASK-002。
- **Definition of Done:** 所有必须 Secret 均有来源、注入、轮换和恢复证据。

### TASK-004 — 冻结 Git Source of Truth 和仓库布局
- **Objective:** 确定 Git 中的架构、治理、安全、平台、任务和 DA-SOC workflow 事实来源。
- **Preconditions:** TASK-001～003 完成。
- **Inputs:** Baseline、ADR、现有仓库目录、DA-SOC workflow 源码。
- **Actions:** 规划 `architecture/`、`governance/`、`security/`、`platform/`、`manifests/`、`tasks/`、`runbooks/`、`da-soc/workflows/`、`backup/`、`incidents/` 和 `07-aiops/component-lifecycle/`；配置主分支保护、双人审批、签名/审计；定义 generated artifact 与源文件关系；Secrets 仅保存加密引用。
- **Validation:** 任一部署配置都能追溯到 Git commit；workflow 可由源文件生成 artifact；无明文 Secret。
- **Expected Output:** Git layout、分支规则、CODEOWNERS/审批规则、drift 检查规则。
- **Risk:** 手工配置漂移或未审查变更进入生产。
- **Approval:** Governance Owner、Security Owner、Platform Owner、DA-SOC Owner。
- **Rollback:** 禁止直接 apply 未审查分支，恢复到最近批准 commit。
- **Evidence:** 仓库树、保护规则截图/导出、示例 PR、drift 报告。
- **Dependencies:** TASK-001、TASK-003。
- **Definition of Done:** Git 被正式声明为 Architecture、Governance、Security Policy、NetworkPolicy、RBAC、Runbook、Task、ADR、Backup configuration、DA-SOC workflow 和 CLM component/policy/rule 的 Source of Truth；运行时数据和明文 Secret 不入 Git。

### TASK-005 — 建立实施门禁、变更和证据目录
- **Objective:** 让每项任务、审批、事故和证据可追踪。
- **Preconditions:** TASK-001～004 完成。
- **Inputs:** 任务清单、外部依赖、风险表、Cutover/Restore/Rollback 附件。
- **Actions:** 创建任务状态模型 `PENDING/IN_PROGRESS/BLOCKED/PASSED/ROLLED_BACK`；定义证据目录、Incident/Change ID、审批记录和保留周期；登记未决外部依赖；定义 CLM discovery、vulnerability、upgrade、approval、verification 和 rollback evidence 的关联字段。
- **Validation:** 用演练任务走通创建、审批、执行、证据归档和回退。
- **Expected Output:** 实施门禁模板、证据目录、变更模板。
- **Risk:** 完成状态无法审计或恢复。
- **Approval:** Governance Owner、Platform Owner。
- **Rollback:** 恢复到上一版流程模板，不修改架构文件。
- **Evidence:** 模板 commit、演练记录、依赖登记。
- **Dependencies:** TASK-001～004。
- **Definition of Done:** 后续平台任务和 CLM 任务均能引用统一 Change/Task/Evidence ID，且组件状态、升级判断和审批可审计。

## 5. Phase 1 — Infrastructure

### TASK-006 — 交付五台 VM 和数据盘
- **Objective:** 按 Baseline 交付五个故障域清晰的 VM。
- **Preconditions:** TASK-002 完成；基础设施依赖已确认。
- **Inputs:** VM 拓扑、规格、磁盘规划、UOS Server V20 1060e AMD64 安装介质依赖、`EXTERNAL-DEPENDENCIES.md`。
- **Actions:** 创建 `xw-cp-01`（4–8 vCPU/16 GiB/100 GiB）、`xw-wk-01`（8/32/300 GiB）、`xw-wk-02`（8/32/500 GiB）、`xw-harbor-01`（4/8/500 GiB）、`xw-backup-01`（4/8/按数据量起步 1 TiB）；确认独立数据盘、快照策略、故障域、OS 安装入口和介质导入路径。
- **Validation:** 主机名、CPU、内存、数据盘和宿主故障域符合基线；控制面、Harbor、备份未合并。
- **Expected Output:** VM inventory、磁盘挂载表、资产标签。
- **Risk:** 资源不足或故障域合并。
- **Approval:** Infrastructure Owner、Platform Owner。
- **Rollback:** 销毁未初始化 VM 或退回资源申请，不触碰生产数据。
- **Evidence:** 云平台/虚拟化导出、`lsblk`/资源检查、资产记录。
- **Dependencies:** TASK-002、TASK-005。
- **Definition of Done:** 五台 VM 均可通过管理路径访问且数据盘已识别。

### TASK-007 — 固化 OS、SSH、NTP 和主机安全基线
- **Objective:** 在五台 VM 上安装候选 UOS Server V20 1060e AMD64，并固化主机安全基线。
- **Preconditions:** TASK-006 完成；OS ISO/介质 SHA256、来源和免费使用授权已确认；OS 仍标记为 Candidate。
- **Inputs:** `VERSION-MATRIX.md`、UOS 安装介质、授权确认、主机安全基线、企业 NTP/DNS、管理 VPN/堡垒机要求。
- **Actions:** 校验并安装 UOS Server V20 1060e AMD64；记录架构、内核和软件源；禁用 swap（例外需记录）；配置 containerd 前置依赖、chrony、Asia/Shanghai、auditd、journald 持久化、安全补丁、最小 sudo、非 root SSH；禁止共享管理员密码；不得将该组合描述为官方认证或 SLA。
- **Validation:** 五台主机 OS 版本、AMD64 架构、ISO/介质 hash、授权记录、时间偏差、swap、审计、SSH、软件源和补丁状态通过；与 UOS 安全基线冲突项形成验证记录。
- **Expected Output:** 主机基线报告。
- **Risk:** 时间漂移破坏证书、审计和 workflow 去重。
- **Approval:** Security Owner、Infrastructure Owner。
- **Rollback:** 只回退本次基线变更，不回退安全补丁；保留堡垒机恢复入口。
- **Evidence:** 命令输出、配置快照、补丁清单、时间同步报告。
- **Dependencies:** TASK-006。
- **Definition of Done:** 五台 VM 均符合 OS/SSH/NTP/审计基线。

### TASK-008 — 配置 DNS、CIDR、路由和防火墙
- **Objective:** 消除节点、Pod、Service、企业网络和外部出口冲突。
- **Preconditions:** TASK-006、TASK-007 完成；网络 Owner 提供 CIDR。
- **Inputs:** 节点网段、Pod CIDR、Service CIDR、DNS、NTP、外部出口清单。
- **Actions:** 配置正反向 DNS、静态主机名、节点路由、管理访问、集群内部端口、Harbor HTTPS、备份通道和最小外部出口；明确 IMAPS/POP3S、DingTalk HTTPS、镜像包和 Git 访问策略。
- **Validation:** CIDR 不重叠；节点互通；DNS/NTP 正常；仅允许审批端口；出口白名单可审计。
- **Expected Output:** 网络参数表、防火墙规则、连通性矩阵。
- **Risk:** 安装后无法初始化或 DA-SOC 无法访问外部依赖。
- **Approval:** Network Owner、Security Owner。
- **Rollback:** 恢复到变更前规则；保留紧急管理通道。
- **Evidence:** DNS 查询、路由表、防火墙导出、端口测试。
- **Dependencies:** TASK-006、TASK-007。
- **Definition of Done:** 节点、管理面、Harbor、备份和计划外部出口均有可验证路径。

### TASK-009 — 安装 containerd 和节点运行时前置项
- **Objective:** 让节点具备与冻结 Kubernetes 版本兼容的容器运行时。
- **Preconditions:** TASK-002、TASK-007、TASK-008 完成。
- **Inputs:** `VERSION-MATRIX.md` 中的 containerd 1.7.x 候选版本、UOS 内核模块、sysctl、镜像加速/Harbor CA。
- **Actions:** 安装 containerd；配置 systemd cgroup、内核模块、IP forwarding、br_netfilter、日志轮转、Harbor CA 和 registry mirror（如批准）；不安装 Docker daemon 作为 Kubernetes 运行时。
- **Validation:** `containerd` 健康；cgroup、sysctl、模块和证书检查通过；可在非生产测试镜像上拉取。
- **Expected Output:** 三台 K8s 节点运行时报告。
- **Risk:** CRI 不兼容导致集群安装失败。
- **Approval:** Platform Owner、Security Owner。
- **Rollback:** 卸载未通过验证的运行时配置并恢复批准包。
- **Evidence:** 版本输出、配置 hash、CRI 检查、拉取日志。
- **Dependencies:** TASK-002、TASK-007、TASK-008。
- **Definition of Done:** 三台节点均通过 CRI 前置检查。

### TASK-010 — 完成基础设施 Preflight
- **Objective:** 在安装 Kubernetes 前一次性验证所有主机前置条件。
- **Preconditions:** TASK-006～009 完成。
- **Inputs:** Preflight 脚本、版本矩阵、网络/磁盘/安全基线。
- **Actions:** 检查 OS release/架构/ISO hash、CPU/内存/磁盘 inode、挂载点、时间、DNS、端口、内核、containerd、SSH、cgroup、swap、sysctl、firewall、SELinux/安全模块、iptables/nftables、证书、备份目标和 Harbor 包；输出失败项并阻断后续安装。
- **Validation:** UOS OS 基线和所有必检项 PASS；任何 WARN 均有批准的处置记录；不得把未验证的 OS/Kubernetes/KubeSphere 组合标记为 Frozen。
- **Expected Output:** Preflight 报告和安装放行单。
- **Risk:** 隐藏前置故障在集群阶段暴露。
- **Approval:** Platform Owner、Infrastructure Owner。
- **Rollback:** 对失败项修复后重新执行，不进入 TASK-011。
- **Evidence:** 带时间戳的报告、命令日志、放行签字。
- **Dependencies:** TASK-006～009。
- **Definition of Done:** Preflight 全绿且依赖清单无未处理 BLOCKING 项。

## 6. Phase 2 — Kubernetes

### TASK-011 — 初始化单 Control Plane Kubernetes 集群
- **Objective:** 按 Baseline 初始化 `xw-cp-01` + `xw-wk-01` + `xw-wk-02` 集群。
- **Preconditions:** TASK-002、TASK-010 通过；Kubernetes 版本已冻结。
- **Inputs:** `VERSION-MATRIX.md`、Kubernetes v1.30.6 安装包/镜像、KubeSphere 兼容矩阵、Pod/Service CIDR、containerd 配置、节点 inventory。
- **Actions:** 仅使用批准的 Kubernetes v1.30.6 安装路径初始化 Control Plane 和 etcd；保存 join 信息到受控加密位置；加入两个 Worker；设置节点标签和 Control Plane `NoSchedule` 污点；不部署额外控制面节点；记录 UOS + Kubernetes 组合仍为 Candidate。
- **Validation:** `kubectl get nodes` 全部 Ready；CoreDNS、kube-proxy 和 etcd 健康；节点角色符合基线；安装、加入、重启和恢复报告通过后才允许推进 KubeSphere 验证。
- **Expected Output:** 可用单集群、集群凭据和初始化证据。
- **Risk:** 单控制面故障导致管理面中断。
- **Approval:** Platform Owner、Security Owner。
- **Rollback:** 在未承载 DA-SOC 前按安装 Runbook 清理并重建；保留 etcd snapshot。
- **Evidence:** 版本、节点、etcd health、初始化日志。
- **Dependencies:** TASK-010。
- **Definition of Done:** 三节点集群健康，且没有未批准的组件进入集群。

### TASK-012 — 配置 etcd、API Server 和控制面健康检查
- **Objective:** 固化控制面数据保护、健康检查和恢复入口。
- **Preconditions:** TASK-011 完成。
- **Inputs:** etcd snapshot 计划、证书、审计策略、健康检查阈值。
- **Actions:** 配置 etcd 数据盘/权限、snapshot 脚本和 retention；启用 API Server、scheduler、controller-manager 健康探针；配置资源/证书到期告警和管理面备份。
- **Validation:** 手工生成并校验 etcd snapshot；恢复到隔离环境可读取 API；健康检查和告警触发。
- **Expected Output:** etcd 备份 Job/脚本、控制面健康 Runbook。
- **Risk:** etcd 快照存在但不可恢复。
- **Approval:** Platform Owner、Backup Owner、Security Owner。
- **Rollback:** 禁用未验证的自动任务，保留手工 snapshot 和旧配置。
- **Evidence:** snapshot hash、恢复日志、健康检查输出。
- **Dependencies:** TASK-011、TASK-005。
- **Definition of Done:** etcd snapshot 与隔离恢复均通过。

### TASK-013 — 启用 Kubernetes Audit 和基础审计留存
- **Objective:** 记录人、Agent、ServiceAccount 对集群资源的关键操作。
- **Preconditions:** TASK-011、TASK-012 完成；审计保留策略已批准。
- **Inputs:** Audit policy、日志脱敏规则、Loki/备份目标。
- **Actions:** 启用审计策略，覆盖用户、Agent/SA、资源、动作、时间、Task ID 关联 metadata；排除 Secret value；配置轮转、访问控制和备份。
- **Validation:** 创建/修改/删除测试资源可查；Secret 值不出现在审计；审计日志可导出和恢复。
- **Expected Output:** Audit policy、留存配置、审计查询示例。
- **Risk:** 审计过少无法追责，过多造成敏感信息泄露或磁盘耗尽。
- **Approval:** Security Owner、Governance Owner。
- **Rollback:** 回到最小批准策略并保留事件记录，不关闭关键审计。
- **Evidence:** 测试事件、脱敏检查、日志轮转状态。
- **Dependencies:** TASK-012。
- **Definition of Done:** 审计覆盖和脱敏检查均通过。

### TASK-014 — 建立集群健康、容量和节点故障告警
- **Objective:** 在平台组件部署前发现节点、API、etcd 和容量问题。
- **Preconditions:** TASK-011～013 完成。
- **Inputs:** Prometheus 后续采集约定、基线阈值、节点角色。
- **Actions:** 先配置可用的基础健康检查；定义 NodeReady、DiskPressure、MemoryPressure、API、etcd、证书、PVC 和资源阈值；把告警路由到运维入口。
- **Validation:** 在非生产窗口模拟 Node NotReady/DiskPressure；告警、恢复和审计均可见。
- **Expected Output:** 健康检查 Runbook、告警阈值初版。
- **Risk:** 过早告警噪声或关键故障无告警。
- **Approval:** Platform Owner、Observability Owner。
- **Rollback:** 恢复到上一版阈值，不删除告警规则。
- **Evidence:** 告警样例、恢复样例、阈值评审记录。
- **Dependencies:** TASK-011～013。
- **Definition of Done:** 集群基础故障具备可观察和可处置路径。

## 7. Phase 3 — KubeSphere

### TASK-015 — 安装 KubeSphere 管理面
- **Objective:** 在已验证 Kubernetes 上安装批准版本 KubeSphere。
- **Preconditions:** TASK-011～014 通过；KubeSphere 兼容矩阵已冻结。
- **Inputs:** `VERSION-MATRIX.md`、KubeSphere 4.1.x 候选安装包（优先验证 4.1.2）、官方兼容矩阵、版本 digest、访问域名、TLS 证书。
- **Actions:** 先在独立验证环境按官方兼容路径验证 KubeSphere 4.1.x；仅安装 Baseline 所需管理组件；使用独立管理员入口和最小权限；不额外引入多集群、DevOps、Service Mesh 或扩展运行时能力；记录 UOS + Kubernetes + KubeSphere 组合状态。
- **Validation:** KubeSphere 控制台、API、项目管理、监控入口正常；组件版本符合矩阵；UOS 1060e、Kubernetes v1.30.6 和 KubeSphere 候选组合通过实际安装验证后，才可提交 Frozen 评审。
- **Expected Output:** KubeSphere 管理面及管理员访问 Runbook。
- **Risk:** 管理面资源消耗影响 DA-SOC。
- **Approval:** Platform Owner、Security Owner。
- **Rollback:** 按官方卸载/回退步骤在未承载业务前恢复集群。
- **Evidence:** 安装日志、版本、健康状态、TLS 检查。
- **Dependencies:** TASK-011、TASK-002。
- **Definition of Done:** KubeSphere 可用于后续 Workspace/Project/RBAC 配置。

### TASK-016 — 创建 Workspace、Project 和资源配额
- **Objective:** 建立平台与 DA-SOC 的边界和资源护栏。
- **Preconditions:** TASK-015 完成。
- **Inputs:** Namespace 规划、ResourceQuota、LimitRange、节点标签。
- **Actions:** 创建 KubeSphere Workspace；创建生产 `da-soc` Project 和临时 `da-soc-validate` Project；配置 CPU/内存/PVC/对象数量配额、LimitRange 和 DA-SOC 节点选择策略。
- **Validation:** 超配额 Pod/PVC 被拒绝；生产与验证资源互不可见；Control Plane 不被调度业务。
- **Expected Output:** Workspace/Project/Quota manifests。
- **Risk:** 配额过低导致业务异常，过高导致平台耗尽。
- **Approval:** Platform Owner、DA-SOC Owner。
- **Rollback:** 在无业务前调整配额；保留旧值和审批记录。
- **Evidence:** `kubectl`/KubeSphere 导出、拒绝测试、资源报告。
- **Dependencies:** TASK-015。
- **Definition of Done:** 两个 Project 边界、配额和节点调度规则生效。

### TASK-017 — 配置 KubeSphere RBAC 和管理员分权
- **Objective:** 实现平台、DA-SOC、观测、安全和只读角色分离。
- **Preconditions:** TASK-016 完成；Git Source of Truth 已建立。
- **Inputs:** RBAC 设计、Owner 名单、Break-glass 规则。
- **Actions:** 创建用户组和 Role/RoleBinding；平台管理员不默认拥有业务 Secret 读取；DA-SOC 运维不能修改集群网络和 etcd；`cluster-admin` 仅短时双人批准并产生 Incident/Task 记录。
- **Validation:** 正常账号权限正反向测试；Break-glass 审计、时限和回收测试。
- **Expected Output:** RBAC manifests、权限矩阵、Break-glass Runbook。
- **Risk:** 权限过大导致 Agent 或人员越权。
- **Approval:** Security Owner、Governance Owner、Platform Owner。
- **Rollback:** 回到最近批准 RoleBinding；紧急撤销可疑账号。
- **Evidence:** `kubectl auth can-i` 报告、审计记录、审批记录。
- **Dependencies:** TASK-004、TASK-016。
- **Definition of Done:** RBAC 最小权限和 Break-glass 流程验证通过。

## 8. Phase 4 — Network

### TASK-018 — 安装并验证 Calico
- **Objective:** 按 ADR-002 安装 Calico 作为唯一 CNI。
- **Preconditions:** TASK-002、TASK-011、TASK-010 完成。
- **Inputs:** `VERSION-MATRIX.md` 中的 Calico 候选版本/digest、Pod CIDR、网络 MTU、兼容矩阵。
- **Actions:** 在 Kubernetes/KubeSphere 候选组合上安装 Calico；配置 IPPool、MTU、Felix 基线和可观测指标；不安装 Cilium/Hubble/Tetragon；记录 Calico 与 UOS firewall/iptables/nftables 的实际兼容结果。
- **Validation:** Calico 节点和 Pod 健康；跨节点 Pod 通信、Service 通信、NetworkPolicy 和重启恢复通过；版本/digest 匹配；组合状态仍保持 Candidate，直到完整验证门禁通过。
- **Expected Output:** Calico manifests、健康报告、网络参数。
- **Risk:** CNI 错误导致整个集群不可用。
- **Approval:** Network Owner、Platform Owner。
- **Rollback:** 只在无业务承载时按 CNI 回退 Runbook 执行；失败时停止后续部署。
- **Evidence:** Calico 状态、网络测试、安装日志。
- **Dependencies:** TASK-011、TASK-002。
- **Definition of Done:** Calico 是唯一生效 CNI 且跨节点网络稳定。

### TASK-019 — 应用全局和 DA-SOC default-deny NetworkPolicy
- **Objective:** 建立默认拒绝、显式放行的网络边界。
- **Preconditions:** TASK-018、TASK-016 完成。
- **Inputs:** Baseline 网络策略、Namespace/Service 清单、DNS 和监控来源。
- **Actions:** 在 `da-soc` 和验证 Namespace 应用 ingress/egress default-deny；只放行 DNS、n8n→render/archive、render/archive→ClickHouse、必要 metrics、IMAPS/POP3S、DingTalk HTTPS 和批准的管理路径。
- **Validation:** 未授权跨 Namespace、API Server、etcd、kubelet 和业务端口访问均失败；授权链路成功。
- **Expected Output:** NetworkPolicy manifests 和端口矩阵。
- **Risk:** 策略过宽造成横向移动，过窄破坏业务。
- **Approval:** Security Owner、Network Owner、DA-SOC Owner。
- **Rollback:** 先在 `da-soc-validate` 调整；生产策略回退必须有变更审批。
- **Evidence:** deny/allow 测试、策略导出、抓包或连接报告。
- **Dependencies:** TASK-016、TASK-018。
- **Definition of Done:** 验证 Namespace 和生产 Namespace 均遵守最小放行。

### TASK-020 — 配置 DA-SOC Service、Ingress 和外部出口
- **Objective:** 把原始 `n8n → render/archive → ClickHouse` 转换为确定的集群网络。
- **Preconditions:** TASK-019 完成；DA-SOC 端口和 API contract 已确认。
- **Inputs:** Service/Ingress 端口、内部 DNS 名称、外部邮箱/DingTalk 地址、TLS 证书。
- **Actions:** 为 n8n、render/archive、ClickHouse 创建 ClusterIP；只按需要配置 Ingress；固定 `/archive`、`/render`、HTTP SQL 路径；限制 egress 目标和端口；生产/测试 DingTalk endpoint 分离。
- **Validation:** n8n 到 render/archive、render/archive 到 ClickHouse、SQL、`/archive`、`/render`、邮箱和 DingTalk 均成功；未授权入口失败。
- **Expected Output:** Service/Ingress/egress manifests、网络验证矩阵。
- **Risk:** DNS/端口变化破坏 workflow 确定性。
- **Approval:** DA-SOC Owner、Network Owner、Security Owner。
- **Rollback:** 切回验证 Namespace 或旧 ECS endpoint，不改变业务规则。
- **Evidence:** curl/SQL/端口测试、TLS、出口日志。
- **Dependencies:** TASK-019、TASK-002。
- **Definition of Done:** 全链路只通过批准的 Service/出口通信。

### TASK-021 — 执行网络安全和故障隔离测试
- **Objective:** 证明 Calico 和 NetworkPolicy 不允许越权访问且不阻断必要业务。
- **Preconditions:** TASK-018～020 完成；测试数据和测试邮箱可用。
- **Inputs:** 安全测试用例、端口白名单、故障注入窗口。
- **Actions:** 测试跨 Namespace、ServiceAccount、API Server、etcd、kubelet、外部非白名单访问；测试节点/Pod 重启、网络短断、DNS 故障；记录生产链路不受影响的边界。
- **Validation:** 所有 deny/allow 结果符合矩阵；失败时有告警和恢复路径。
- **Expected Output:** 网络安全测试报告和问题清单。
- **Risk:** 未发现的网络越权或业务中断。
- **Approval:** Security Owner、Network Owner、DA-SOC Owner。
- **Rollback:** 禁止带缺陷策略进入生产；恢复到上一版验证策略。
- **Evidence:** 测试命令、日志、告警、修复 commit。
- **Dependencies:** TASK-020、TASK-014。
- **Definition of Done:** 高风险网络测试通过或每个例外有签字和期限。

## 9. Phase 5 — Storage

### TASK-022 — 安装 local-path/Local PV 并固化节点绑定
- **Objective:** 提供可恢复、可追踪的单节点本地存储。
- **Preconditions:** TASK-016、TASK-018、TASK-021 完成；磁盘已分区挂载。
- **Inputs:** local-path 版本、节点路径、StorageClass、备份策略。
- **Actions:** 在 `xw-wk-02` 固化 DA-SOC 数据路径，在 `xw-wk-01` 固化平台/观测路径；创建 StorageClass、目录权限、容量水位和节点亲和性；不部署 Ceph/Longhorn。
- **Validation:** PVC 绑定到预期节点和目录；非预期节点不能写入；删除/重建 Pod 后数据保留。
- **Expected Output:** StorageClass、PV 路径、容量和恢复说明。
- **Risk:** 节点故障导致本地数据暂不可用。
- **Approval:** Platform Owner、DA-SOC Owner、Backup Owner。
- **Rollback:** 在无业务 PVC 时调整 StorageClass；有数据时只通过迁移/恢复 Runbook。
- **Evidence:** PVC/PV 导出、节点亲和性、读写和重启测试。
- **Dependencies:** TASK-006、TASK-016、TASK-018。
- **Definition of Done:** Local PV 可用且每个数据集有备份/恢复路径。

### TASK-023 — 创建 ClickHouse、n8n 和 raw archive PVC
- **Objective:** 为 DA-SOC 三类持久数据建立独立容量和权限边界。
- **Preconditions:** TASK-022 完成；历史数据量和保留策略已确认。
- **Inputs:** ClickHouse 数据盘、n8n state、`/data/da-soc/raw`、ResourceQuota。
- **Actions:** 创建 ClickHouse StatefulSet PVC、n8n state PVC、raw archive PVC；设置 requests/limits、容量水位、备份标签、只读/读写挂载；禁止把业务 raw 和临时日志混在同一目录。
- **Validation:** PVC 绑定和写入测试；容量告警；跨组件不能任意挂载彼此 PVC。
- **Expected Output:** PVC manifests、数据目录清单、容量阈值。
- **Risk:** PVC 容量不足或错误挂载造成数据污染。
- **Approval:** DA-SOC Owner、Platform Owner、Security Owner。
- **Rollback:** 在数据导入前删除重建；导入后仅按恢复 Runbook 操作。
- **Evidence:** PVC/PV、挂载、权限、配额和水位报告。
- **Dependencies:** TASK-022、TASK-016。
- **Definition of Done:** 三类 PVC 独立、可写、可备份且有容量告警。

### TASK-024 — 验证存储故障、备份和恢复接口
- **Objective:** 在 DA-SOC 部署前证明 Local PV 不可用时能恢复。
- **Preconditions:** TASK-023 完成；备份目标和恢复工具可用。
- **Inputs:** 空测试数据、PV 恢复 Runbook、xw-backup-01。
- **Actions:** 模拟 Pod 重启、节点不可调度、PVC 重建和数据盘只读；验证从备份恢复测试 PVC；记录 ClickHouse/raw/n8n 各自恢复步骤。
- **Validation:** 数据 hash/行数/文件 manifest 一致；恢复后应用可读；无生产数据被破坏。
- **Expected Output:** 存储故障测试报告和恢复脚本。
- **Risk:** Local PV 失败后只能人工猜测恢复。
- **Approval:** Backup Owner、DA-SOC Owner、Platform Owner。
- **Rollback:** 清理隔离测试资源，不触碰生产 PVC。
- **Evidence:** 故障时间线、恢复命令、hash/SQL 对比、RTO。
- **Dependencies:** TASK-023、TASK-005。
- **Definition of Done:** Local PV 的节点/磁盘故障限制和可恢复路径均有实证。

## 10. Phase 6 — Harbor

### TASK-025 — 准备 Harbor 独立 VM 和证书
- **Objective:** 为集群外 Harbor 建立独立故障域和可信 HTTPS 入口。
- **Preconditions:** TASK-006～010 完成；Harbor 版本和离线包已冻结。
- **Inputs:** `xw-harbor-01`、Harbor CA/证书、DNS、存储和防火墙规则。
- **Actions:** 按独立 VM 基线初始化 Harbor；挂载 registry 数据盘；配置 DNS、TLS、管理账号、镜像保留策略和最小网络入口；禁止将 Harbor 作为 K8s 内业务 Pod。
- **Validation:** HTTPS 证书链、磁盘权限、管理入口和容量告警通过。
- **Expected Output:** Harbor 安装前检查报告。
- **Risk:** Registry 与集群生命周期耦合或证书不可信。
- **Approval:** Registry Owner、Security Owner。
- **Rollback:** 未导入业务镜像前清理并重装；保留离线包和证书备份。
- **Evidence:** VM/磁盘、TLS、端口和基线报告。
- **Dependencies:** TASK-006～010、TASK-002。
- **Definition of Done:** Harbor VM 可独立恢复且集群可通过 HTTPS 访问。

### TASK-026 — 安装 Harbor 并配置项目权限
- **Objective:** 部署固定版本 Harbor，提供镜像存储、扫描和离线导入能力。
- **Preconditions:** TASK-025 完成；安装包校验通过。
- **Inputs:** Harbor 离线包、digest、CA、管理员和 robot account 方案。
- **Actions:** 安装 Harbor；创建平台、DA-SOC、Agent 项目；创建最小权限 robot account；配置镜像扫描、保留策略、审计和离线导入路径；禁止使用共享管理员凭据。
- **Validation:** 登录、push/pull、robot 权限、HTTPS、审计和扫描结果通过。
- **Expected Output:** Harbor 实例、项目和权限清单。
- **Risk:** 镜像来源不可信或 robot 权限过大。
- **Approval:** Registry Owner、Security Owner、Platform Owner。
- **Rollback:** 删除未使用项目/凭据；重装只在无业务镜像前进行。
- **Evidence:** 安装日志、项目导出、权限测试、扫描报告。
- **Dependencies:** TASK-025、TASK-002。
- **Definition of Done:** Harbor 可用、最小权限生效且离线包可导入。

### TASK-027 — 导入并锁定 DA-SOC 镜像
- **Objective:** 将 n8n、ClickHouse、render/archive 和 Agent 镜像以 digest 固定进入 Harbor。
- **Preconditions:** TASK-026 完成；DA-SOC Owner 提供镜像来源和授权。
- **Inputs:** 镜像 tar/源仓库、SBOM/校验和、版本矩阵。
- **Actions:** 离线导入镜像；核验来源、签名/校验和和 digest；记录镜像 manifest；为生产 manifests 生成 digest 引用；禁止部署 tag 漂移镜像。
- **Validation:** Worker 从 Harbor 拉取固定 digest；镜像内容 hash 与输入一致；扫描无未处置 Critical。
- **Expected Output:** Harbor 镜像清单、digest 锁定文件、扫描结果。
- **Risk:** 错镜像或供应链漂移影响业务确定性。
- **Approval:** DA-SOC Owner、Security Owner、Registry Owner。
- **Rollback:** 使用上一个已批准 digest；禁止直接切换未知镜像。
- **Evidence:** tar hash、digest、pull log、扫描/SBOM。
- **Dependencies:** TASK-026、TASK-002。
- **Definition of Done:** 所有 V0.1 生产镜像均可离线重建并以 digest 部署。

### TASK-028 — 验证 Harbor 备份和集群重建供镜像能力
- **Objective:** 证明 Harbor 故障或集群重建后仍能供应镜像。
- **Preconditions:** TASK-027 完成；`xw-backup-01` 可用。
- **Inputs:** Harbor 配置/数据库/registry data、关键镜像 tar、CA 和恢复 Runbook。
- **Actions:** 备份 Harbor 配置、数据和关键镜像；在隔离 Harbor 或恢复环境重建；让测试 Worker 使用恢复 Harbor 拉取 DA-SOC 镜像。
- **Validation:** HTTPS、项目权限、digest、Kubernetes pull 和扫描结果一致；记录恢复 RTO。
- **Expected Output:** Harbor restore drill 报告。
- **Risk:** 备份只恢复配置，镜像内容缺失。
- **Approval:** Registry Owner、Backup Owner、Platform Owner。
- **Rollback:** 删除隔离 Harbor，不影响生产实例。
- **Evidence:** 备份 hash、恢复日志、pull/digest 对比、RTO。
- **Dependencies:** TASK-027、TASK-040。
- **Definition of Done:** Harbor 可从备份和离线 tar 两条路径恢复并供应固定镜像。

## 11. Phase 7 — Observability

### TASK-029 — 部署 Prometheus、Grafana 和 Alertmanager
- **Objective:** 建立 V0.1 唯一指标、看板和告警栈。
- **Preconditions:** TASK-015～021、版本矩阵冻结；节点资源已评估。
- **Inputs:** KubeSphere/Prometheus stack 兼容版本、保留周期、TLS/RBAC。
- **Actions:** 部署 Prometheus、Grafana、Alertmanager；配置持久化、保留周期、访问控制、录制规则和告警路由；不部署第二套指标系统；预留 CLM discovery freshness、vulnerability freshness、unknown version 和 upgrade closure 指标。
- **Validation:** Kubernetes、节点、PVC、DA-SOC service metrics 可采集；Grafana 登录/RBAC；Alertmanager 路由可见。
- **Expected Output:** 指标栈 manifests、初始 dashboards 和规则。
- **Risk:** 观测栈占满 Local PV 或告警不可用。
- **Approval:** Observability Owner、Platform Owner。
- **Rollback:** 降低保留周期/关闭非关键 dashboard；不删除关键告警。
- **Evidence:** targets、query、dashboard、告警发送记录。
- **Dependencies:** TASK-022、TASK-017、TASK-002。
- **Definition of Done:** 单一指标栈覆盖平台、节点和 DA-SOC 基础指标。

### TASK-030 — 部署 Fluent Bit 和 Loki 日志链路
- **Objective:** 建立 V0.1 唯一日志采集和查询路径。
- **Preconditions:** TASK-029 完成；日志脱敏和容量策略已批准。
- **Inputs:** Fluent Bit/Loki 版本、字段规范、retention、脱敏规则。
- **Actions:** 以 Fluent Bit 采集容器和关键主机日志，输出 Loki；配置 namespace/pod/container/task/change 字段；过滤邮箱正文、token、Secret 和敏感 payload；设置容量水位和轮转。
- **Validation:** n8n、render/archive、ClickHouse、Agent Job、Kubernetes Audit 日志可查询；敏感数据不出现；Loki 恢复路径存在。
- **Expected Output:** 日志配置、字段字典、查询示例。
- **Risk:** 敏感信息泄露或 Loki 磁盘耗尽。
- **Approval:** Security Owner、Observability Owner。
- **Rollback:** 先停止高噪声采集并保留安全审计；恢复上一版过滤规则。
- **Evidence:** 查询结果、脱敏扫描、容量和轮转报告。
- **Dependencies:** TASK-022、TASK-029。
- **Definition of Done:** 单一日志链路可观察且满足脱敏、留存和恢复要求。

### TASK-031 — 接入 DA-SOC 指标、日志和业务 SLI
- **Objective:** 让业务链路的成功、失败和数据语义可被观测。
- **Preconditions:** TASK-029、TASK-030 完成；DA-SOC service contract 已确认。
- **Inputs:** n8n/render/archive/ClickHouse metrics、workflow event、SLI 定义。
- **Actions:** 配置 n8n execution、archive success/failure、render success/failure、ClickHouse query、raw archive、mailbox read、DingTalk delivery、duplicate prevention 和 data freshness 指标；统一 Task/Change correlation。
- **Validation:** 成功、失败、重试、空数据和 archive failure 均能在指标/日志中区分；不暴露业务 Secret。
- **Expected Output:** DA-SOC dashboard、SLI/SLO 和告警规则。
- **Risk:** 只看 Pod 健康而看不到业务错误。
- **Approval:** DA-SOC Owner、Observability Owner、Business Owner。
- **Rollback:** 回退非关键指标，不关闭核心业务失败告警。
- **Evidence:** dashboard 截图/导出、query、告警样例。
- **Dependencies:** TASK-029、TASK-030。
- **Definition of Done:** DA-SOC 业务链路可从邮箱到发送结果端到端观测。

### TASK-032 — 配置关键告警和 DingTalk 路由隔离
- **Objective:** 将关键平台和 DA-SOC 告警送达运维 DingTalk，且不混入业务生产群。
- **Preconditions:** TASK-031 完成；运维 DingTalk 群凭据已由 Security Owner 交付。
- **Inputs:** Alertmanager routes、运维群 webhook、测试群 webhook、去重/静默策略。
- **Actions:** 配置 Control Plane、Node DiskPressure、PVC、ClickHouse、n8n、archive、render、mailbox、备份、恢复和 Agent 告警；运维群、DA-SOC 生产群、测试群分别配置；敏感 payload 脱敏。
- **Validation:** 测试告警只到测试/运维目标；生产业务消息不会被告警路由替代；重复告警按指纹聚合。
- **Expected Output:** 告警路由表、测试记录和回滚开关。
- **Risk:** 告警误发生产群或漏报关键故障。
- **Approval:** Observability Owner、Security Owner、DA-SOC Owner。
- **Rollback:** 关闭新增路由、保留本地 Alertmanager；恢复旧 webhook。
- **Evidence:** Alertmanager config、发送记录、目标群确认。
- **Dependencies:** TASK-031、TASK-005。
- **Definition of Done:** 关键告警可达、隔离、去重且有审计证据。

### TASK-033 — 完成观测栈容量、权限和恢复验证
- **Objective:** 证明监控、日志和告警不会成为平台单点盲区。
- **Preconditions:** TASK-029～032 完成；备份目标可用。
- **Inputs:** 观测 retention、Local PV 容量、Grafana/RBAC、Loki/Prometheus 备份策略。
- **Actions:** 验证 query 性能、容量水位、用户分权、配置备份和恢复；模拟 Loki/Prometheus Pod 重启、磁盘水位和 Alertmanager 重启。
- **Validation:** 关键 dashboard、告警和历史窗口恢复；无 Secret/正文泄露；超阈值有处理 Runbook。
- **Expected Output:** 观测栈验收报告。
- **Risk:** 观测系统故障时无法发现 DA-SOC 故障。
- **Approval:** Observability Owner、Backup Owner、Security Owner。
- **Rollback:** 降低 retention/采集范围并保留业务指标；不删除审计。
- **Evidence:** 恢复日志、容量图、权限和脱敏报告。
- **Dependencies:** TASK-030～032、TASK-038。
- **Definition of Done:** 观测栈可用、可恢复、可审计。

## 12. Phase 8 — Security

### TASK-034 — 应用 Pod Security 和容器安全基线
- **Objective:** 让 `da-soc`、验证、Agent 和平台工作负载默认受限。
- **Preconditions:** TASK-016、TASK-017、TASK-027 完成。
- **Inputs:** Pod Security Admission restricted、镜像 digest、容器运行参数。
- **Actions:** 应用 restricted PSA；禁止 privileged、hostNetwork、hostPID、hostIPC、任意 root/capabilities；设置 runAsNonRoot、seccomp、readOnlyRootFilesystem（兼容时）、resources 和 probes；例外必须有审批和期限。
- **Validation:** 不安全 Pod 被拒绝；批准 workload 正常启动；扫描无未处置 Critical。
- **Expected Output:** PSA labels、securityContext manifests、例外登记。
- **Risk:** 过宽权限导致容器逃逸或横向移动。
- **Approval:** Security Owner、Platform Owner、DA-SOC Owner。
- **Rollback:** 在验证 Namespace 调整安全上下文；生产例外不得无期限保留。
- **Evidence:** Admission 拒绝测试、manifest 审查、扫描结果。
- **Dependencies:** TASK-017、TASK-027。
- **Definition of Done:** 生产/验证 Namespace 均启用受限安全策略。

### TASK-035 — 固化 DA-SOC RBAC、Secret 和业务边界
- **Objective:** 保证 n8n、render/archive、ClickHouse 及 Agent 只能访问其职责所需资源。
- **Preconditions:** TASK-034 完成；Secret 方案已冻结。
- **Inputs:** ServiceAccount、Role、Secret 清单、NetworkPolicy、业务硬约束。
- **Actions:** 为组件创建独立 ServiceAccount；n8n 不拥有集群管理权限；Agent 默认只读；Secret 按组件拆分；禁止 Secret 值进入日志；明确 ClickHouse 数据、raw archive、邮箱和 DingTalk 的访问边界。
- **Validation:** 正常和越权访问测试；Secret metadata 可审计而 value 不可见；生产邮箱操作仅允许读取且不允许 Mark as Read/删除/修改。
- **Expected Output:** DA-SOC RBAC/Secret manifests、权限矩阵。
- **Risk:** 业务组件或 Agent 越权修改生产数据。
- **Approval:** Security Owner、DA-SOC Owner、Business Owner。
- **Rollback:** 撤销新增 RoleBinding/Secret；保留审计和恢复凭据。
- **Evidence:** `auth can-i`、NetworkPolicy、邮箱保护测试、审计记录。
- **Dependencies:** TASK-003、TASK-019、TASK-034。
- **Definition of Done:** 业务硬边界通过自动化和人工双重验证。

### TASK-036 — 生成 Agent 最小 RBAC 和 L0/L1/L2 策略
- **Objective:** 为短生命周期 Agent Job 建立最小权限和人工审批边界。
- **Preconditions:** TASK-035 完成；Runbook 和 Task 模型已确定。
- **Inputs:** Baseline AI Ops、可执行 Runbook 白名单、审批人名单。
- **Actions:** 创建只读观察 ServiceAccount；为 L1 仅绑定批准 Runbook 的窄 Role；L2 需人工审批、短时凭据和双人确认；禁止修改 RBAC/NetworkPolicy/StorageClass/节点/etcd/PVC/业务数据；设置 activeDeadline/TTL/resources；CLM Job 默认只发现和生成 Task，不得自动执行生产升级。
- **Validation:** Job 只能读取批准对象；未批准动作被拒绝；L2 无审批不能执行；Job 完成后无常驻控制通道。
- **Expected Output:** Agent RBAC、分级矩阵、审批 Runbook。
- **Risk:** Agent 越权或执行不可逆动作。
- **Approval:** Security Owner、AIOps Owner、Governance Owner。
- **Rollback:** 立即撤销 Agent ServiceAccount/RoleBinding，暂停 Agent Job。
- **Evidence:** 权限测试、审批记录、Job 生命周期和审计日志。
- **Dependencies:** TASK-017、TASK-034、TASK-035。
- **Definition of Done:** L0/L1/L2 权限和审批边界可执行、可审计、可回收。

### TASK-037 — 执行安全、审计和敏感信息泄露测试
- **Objective:** 在 DA-SOC 部署前验证安全控制满足 V0.1 门禁。
- **Preconditions:** TASK-034～036、TASK-013、TASK-030 完成。
- **Inputs:** 安全测试用例、审计查询、Secret/邮箱/DingTalk 测试凭据。
- **Actions:** 测试越权 API、网络横向、容器权限、Secret 日志泄露、Agent 越权、审计完整性、邮箱 Mark-as-Read/删除/修改保护、生产/测试 DingTalk 隔离。
- **Validation:** 所有拒绝和审计结果符合预期；关键失败有立即停止和回退动作。
- **Expected Output:** 安全验收报告、缺陷与例外清单。
- **Risk:** 安全缺陷进入生产切换。
- **Approval:** Security Owner、Business Owner、DA-SOC Owner。
- **Rollback:** 阻断后续 DA-SOC 生产导入，修复后重新测试。
- **Evidence:** 测试日志、审计记录、脱敏扫描、审批。
- **Dependencies:** TASK-035、TASK-036。
- **Definition of Done:** Critical/High 安全问题关闭或获正式例外且有期限。

## 13. Phase 9 — Backup

### TASK-038 — 初始化备份仓库和不可变副本
- **Objective:** 让 `xw-backup-01` 成为集群外独立恢复路径。
- **Preconditions:** TASK-006～010 完成；备份介质/NAS/S3 和不可变副本已确认。
- **Inputs:** 备份目标、加密策略、容量、RPO/RTO、离线介质。
- **Actions:** 初始化备份 VM 和数据盘；配置访问控制、加密、校验、分层 retention、7 日/4 周/离线副本；备份仓库不加入 Kubernetes。
- **Validation:** 写入、读取、校验、权限和不可变属性通过；生产凭据不能直接修改历史副本。
- **Expected Output:** Backup repository、retention 和访问 Runbook。
- **Risk:** 备份与集群同故障域或可被一并删除。
- **Approval:** Backup Owner、Security Owner、Infrastructure Owner。
- **Rollback:** 停止新策略写入并保留旧副本；不删除唯一副本。
- **Evidence:** 仓库初始化、权限、hash、不可变策略。
- **Dependencies:** TASK-006、TASK-003。
- **Definition of Done:** 存在独立且至少一份不可变/离线备份副本。

### TASK-039 — 实现 Git、etcd 和 Kubernetes 配置备份
- **Objective:** 覆盖平台配置、KubeSphere、RBAC、NetworkPolicy、Task 和审计策略。
- **Preconditions:** TASK-004、TASK-012、TASK-038 完成。
- **Inputs:** Git remote、etcd snapshot、cluster resource export、备份调度。
- **Actions:** 配置 Git mirror/归档；定期 etcd snapshot；导出批准的 Kubernetes/KubeSphere 配置、证书 metadata 和审计策略；敏感值使用加密引用；生成 manifest/hash。
- **Validation:** 从备份恢复一个隔离 namespace/配置；snapshot 可读取；Git commit 和导出 hash 一致。
- **Expected Output:** 配置备份 Job、manifest、恢复示例。
- **Risk:** 只备份运行时状态而丢失 Source of Truth 或反之。
- **Approval:** Backup Owner、Platform Owner、Governance Owner。
- **Rollback:** 暂停失败调度并修复目标，不删除历史备份。
- **Evidence:** Git mirror、snapshot、导出文件、恢复日志。
- **Dependencies:** TASK-004、TASK-012、TASK-038。
- **Definition of Done:** Git/etcd/Kubernetes/KubeSphere 关键配置均可恢复。

### TASK-040 — 实现 DA-SOC、Harbor 和 Secret 备份
- **Objective:** 覆盖 ClickHouse、raw、n8n、render/archive、Harbor 和重要 Secret 恢复资料。
- **Preconditions:** TASK-003、TASK-028、TASK-038～039 完成；数据路径已确定。
- **Inputs:** ClickHouse BACKUP、raw file manifest、n8n state/workflow、render 配置、Harbor backup、加密 Secret。
- **Actions:** 配置 ClickHouse BACKUP 和 catalog；备份 `/data/da-soc/raw`；备份 n8n workflow/state 与 encryption key 恢复资料；备份 render/archive 配置；备份 Harbor 数据/配置/关键镜像 tar；加密 Secret 只备份密文和恢复 key metadata。
- **Validation:** 每种数据生成可读 manifest/hash；随机恢复 ClickHouse 表、raw 文件、workflow 和测试 Secret；凭据值不出现在普通备份日志。
- **Expected Output:** DA-SOC/Harbor/Secret backup jobs、manifest 和恢复 Runbook。
- **Risk:** 关键数据存在但恢复顺序或密钥不完整。
- **Approval:** Backup Owner、DA-SOC Owner、Security Owner、Registry Owner。
- **Rollback:** 失败任务只暂停新轮次，保留最近成功副本并告警。
- **Evidence:** 备份日志、hash、恢复样例、密钥托管记录。
- **Dependencies:** TASK-003、TASK-028、TASK-038、TASK-039。
- **Definition of Done:** 所有 Baseline 要求的数据集均有成功备份和至少一次恢复证据。

### TASK-041 — 配置备份告警、保留和恢复索引
- **Objective:** 让备份失败、过期、容量不足和不可恢复状态及时可见。
- **Preconditions:** TASK-039、TASK-040 完成；观测栈可用。
- **Inputs:** Backup job metrics/logs、retention、RPO/RTO、Alertmanager。
- **Actions:** 配置备份成功/失败/延迟/容量/校验/恢复测试告警；维护最近成功点、RPO、RTO、文件 manifest 和恢复依赖索引；每次切换前必须生成最新备份。
- **Validation:** 注入失败任务，告警进入运维 DingTalk；按索引找到每个恢复对象和依赖。
- **Expected Output:** Backup SLI、告警规则、恢复索引。
- **Risk:** 备份静默失败或误判为成功。
- **Approval:** Backup Owner、Observability Owner、Platform Owner。
- **Rollback:** 恢复上一版调度/路由；不关闭失败告警。
- **Evidence:** 失败注入、告警、索引和 RPO/RTO 报告。
- **Dependencies:** TASK-032、TASK-040。
- **Definition of Done:** 备份健康状态可观察，且任何关键备份失败都会阻断切换。

## 14. Phase 10 — DA-SOC Deployment

### TASK-042 — 固定 DA-SOC 镜像、配置和发布清单
- **Objective:** 生成可审查、可复现的 DA-SOC 发布 artifact。
- **Preconditions:** TASK-027、TASK-035、TASK-040 完成；镜像和 Secret 已批准。
- **Inputs:** n8n、ClickHouse、render/archive、raw archive 镜像 digest；配置模板；Service/Ingress/PVC/Policy。
- **Actions:** 生成 `da-soc` manifests；固定 image digest、资源、探针、节点亲和、PVC、Service、NetworkPolicy、Secret 引用和备份标签；生产与验证值分离；不把生产凭据写入 manifest。
- **Validation:** 静态 lint、schema、digest、Secret reference、Policy、PVC 和 quota 检查通过。
- **Expected Output:** Git 发布 commit、artifact manifest、发布清单。
- **Risk:** 手工 apply 导致环境漂移或错误凭据进入验证环境。
- **Approval:** DA-SOC Owner、Platform Owner、Security Owner。
- **Rollback:** 使用上一个批准 artifact；不直接修改运行中资源。
- **Evidence:** commit、rendered manifest、lint 和 diff。
- **Dependencies:** TASK-027、TASK-035、TASK-040。
- **Definition of Done:** 发布 artifact 可在验证和生产复用，仅通过参数/Secret 引用区分环境。

### TASK-043 — 创建 `da-soc` 和 `da-soc-validate` Namespace
- **Objective:** 建立正式生产与临时验证的隔离边界。
- **Preconditions:** TASK-016、TASK-019、TASK-034、TASK-042 完成。
- **Inputs:** Namespace labels、Quota、LimitRange、PSA、RBAC、NetworkPolicy。
- **Actions:** 创建正式 `da-soc` 和临时 `da-soc-validate`；应用 restricted PSA、quota、LimitRange、default-deny、审计标签和独立 ServiceAccount；验证 namespace 只能由批准角色访问。
- **Validation:** namespace、quota、PSA、policy、RBAC 全部生效；验证环境不能访问生产 Secret/邮箱/DingTalk。
- **Expected Output:** 两个 Namespace 配置和隔离测试报告。
- **Risk:** 测试污染生产或验证凭据误用于生产。
- **Approval:** Security Owner、DA-SOC Owner、Platform Owner。
- **Rollback:** 删除未承载数据的验证 Namespace；生产 Namespace 只按批准回退。
- **Evidence:** namespace/PSS/quota/policy/RBAC 导出、越权测试。
- **Dependencies:** TASK-016、TASK-019、TASK-034、TASK-042。
- **Definition of Done:** `da-soc` 作为唯一正式 Namespace，验证边界独立且可删除。

### TASK-044 — 部署 ClickHouse StatefulSet 和数据访问路径
- **Objective:** 在 Local PV 上部署单实例 ClickHouse 并保持 SQL 确定性。
- **Preconditions:** TASK-022～024、TASK-042～043 完成；ClickHouse 版本和 schema 已确认。
- **Inputs:** ClickHouse digest、PVC、用户/权限、schema、HTTP SQL endpoint。
- **Actions:** 在 `da-soc-validate` 先部署 ClickHouse；配置 StatefulSet、PVC、探针、资源、Service、最小用户权限、备份目录和 HTTP SQL；固定时区/字符集/连接参数；不部署分布式 ClickHouse。
- **Validation:** 启动、写入、查询、备份和恢复测试通过；SQL 结果可复现；只允许 render/archive/n8n 访问批准端口。
- **Expected Output:** ClickHouse manifests、schema 初始化 artifact、SQL 连接验证。
- **Risk:** 数据目录或 schema 错误导致业务数字错误。
- **Approval:** DA-SOC Owner、Security Owner。
- **Rollback:** 删除验证实例并从批准 backup 重建；生产实例不做未经批准的 schema 变更。
- **Evidence:** schema hash、SQL 结果、PVC、探针、权限测试。
- **Dependencies:** TASK-023、TASK-042、TASK-043。
- **Definition of Done:** ClickHouse 在验证 Namespace 可用且 SQL 只读/写入边界明确。

### TASK-045 — 部署 render/archive 服务
- **Objective:** 在 K8s 内承载原 render/archive，并保持 `/archive`、`/render` 合约。
- **Preconditions:** TASK-020、TASK-027、TASK-042～044 完成。
- **Inputs:** `da-soc-render:0.1` digest、API contract、配置、ClickHouse Service。
- **Actions:** 部署 render/archive Deployment/Service；配置 `/archive`、`/render` 探针和超时；archive 成功才允许后续 SQL/出图/发送，失败显式返回错误并终止 workflow；render 只从批准 SQL 结果生成图片。
- **Validation:** 正常 archive、空数据、异常数据、archive failure、render success/failure 均符合业务规则；不调用 LLM 生成数字或图片。
- **Expected Output:** render/archive manifests、API 合约测试、错误路径测试。
- **Risk:** archive 失败仍入库/出图/发送，或 render 结果不一致。
- **Approval:** DA-SOC Owner、Business Owner、Security Owner。
- **Rollback:** 切换到已验证的上一版 render image digest；生产入口保持关闭。
- **Evidence:** API response、ClickHouse 行数、图片 hash、workflow branch 日志。
- **Dependencies:** TASK-020、TASK-027、TASK-044。
- **Definition of Done:** `/archive` 和 `/render` 行为与基准一致，失败熔断规则通过。

### TASK-046 — 部署 raw archive 和文件恢复路径
- **Objective:** 在 K8s 内保存原始归档并保证其可审计、可恢复。
- **Preconditions:** TASK-023、TASK-042～045 完成；raw archive 目录/保留策略已确认。
- **Inputs:** raw PVC、文件命名/manifest 规则、权限、备份脚本。
- **Actions:** 挂载 raw archive PVC；配置只允许批准组件写入、运维最小读取；为每批文件生成 manifest、hash、来源 Message-ID/UID 关联；禁止将 raw 内容写入日志。
- **Validation:** 文件写入、重复、损坏检测、备份和隔离恢复通过；非授权 ServiceAccount 不能写入/删除。
- **Expected Output:** raw archive Deployment/Job 或挂载配置、manifest 示例。
- **Risk:** 原始数据缺失或被错误覆盖。
- **Approval:** DA-SOC Owner、Security Owner、Backup Owner。
- **Rollback:** 只停止新写入并恢复最近 manifest；不直接删除生产 raw。
- **Evidence:** 文件 hash/manifest、权限测试、恢复记录。
- **Dependencies:** TASK-023、TASK-040、TASK-045。
- **Definition of Done:** raw archive 进入 `da-soc`，具备唯一来源、hash 和恢复路径。

### TASK-047 — 恢复并验证 ClickHouse 历史数据
- **Objective:** 将批准的历史 ClickHouse 数据恢复到验证环境。
- **Preconditions:** TASK-040 完成；ClickHouse 验证实例健康。
- **Inputs:** ClickHouse BACKUP、schema、表结构、历史范围（当天/近 6 周/近 6 月）。
- **Actions:** 按 schema→表→数据→索引/TTL 顺序恢复；记录 backup manifest、行数、时间范围、hash/校验和；恢复后执行标准 SQL，不修改来源数据。
- **Validation:** 表结构、行数、时间范围、关键 SQL 结果和 null/暂无数据语义与基准一致。
- **Expected Output:** 历史数据恢复报告、SQL golden baseline。
- **Risk:** 数据缺行、重复、时区错位或将 null 填为 0。
- **Approval:** DA-SOC Owner、Business Owner、Backup Owner。
- **Rollback:** 删除隔离恢复实例并重新恢复，不触碰源 backup。
- **Evidence:** backup hash、表统计、SQL 结果、差异报告。
- **Dependencies:** TASK-044、TASK-040。
- **Definition of Done:** 当天、近 6 周、近 6 月三个窗口均可复现批准结果。

### TASK-048 — 恢复 raw archive 并校验关联关系
- **Objective:** 将历史 raw archive 恢复到验证 PVC，保持与数据和 workflow 的关联。
- **Preconditions:** TASK-046、TASK-040 完成；恢复索引可用。
- **Inputs:** raw backup、file manifest、Message-ID/UID 映射、时间范围。
- **Actions:** 按 manifest 恢复文件；校验文件 hash、命名、来源、归档时间和 ClickHouse 关联；检测缺失/重复/损坏；不访问或修改生产邮箱。
- **Validation:** 文件数量/hash/时间范围与基准一致；异常文件进入隔离清单，不被误作有效数据。
- **Expected Output:** raw restore report、异常清单、关联索引。
- **Risk:** raw 与 SQL 不一致导致错误图片或错误报告。
- **Approval:** DA-SOC Owner、Business Owner、Backup Owner。
- **Rollback:** 清空验证 PVC 后按 manifest 重放；源备份只读。
- **Evidence:** file manifest、hash、关联 SQL、恢复日志。
- **Dependencies:** TASK-046、TASK-040、TASK-047。
- **Definition of Done:** raw 可恢复且与 ClickHouse/业务时间窗口关联一致。

### TASK-049 — 导入 n8n workflow、配置和 drift 检查
- **Objective:** 通过 Git 源文件确定性生成并导入 n8n workflow。
- **Preconditions:** TASK-004、TASK-042～048 完成；n8n encryption key 恢复资料可用。
- **Inputs:** Git workflow source、`build_workflow.py`、生成 artifact、n8n 2.15.0 或批准版本、验证凭据。
- **Actions:** 从 Git 运行 `build_workflow.py`；生成并校验 workflow artifact/digest；bootstrap/import 到验证 n8n；配置 n8n state PVC、encryption key、内部 Service endpoint 和测试凭据；执行 drift check。
- **Validation:** workflow digest 与 Git artifact 一致；节点、SQL、`/archive`、`/render`、邮件和 DingTalk 分支无漂移；UI 手工修改可被发现。
- **Expected Output:** workflow artifact、import log、drift report、bootstrap Runbook。
- **Risk:** 工作流被手工改写、endpoint 错误或 encryption key 不匹配。
- **Approval:** DA-SOC Owner、Security Owner、Governance Owner。
- **Rollback:** 导入上一个批准 workflow artifact；暂停 trigger；恢复 n8n state/Key。
- **Evidence:** source commit、artifact hash、n8n export、drift diff。
- **Dependencies:** TASK-004、TASK-042～048。
- **Definition of Done:** n8n workflow 可从 Git 重建，且验证环境 drift 为零。

## 15. Phase 11 — DA-SOC Data Migration

### TASK-050 — 建立测试邮箱和回放数据通道
- **Objective:** 提供不触碰生产邮箱状态的迁移验证输入。
- **Preconditions:** TASK-049 完成；DA-SOC Owner 提供测试邮箱/合法历史回放数据。
- **Inputs:** 测试邮箱、脱敏/回放数据、Message-ID/UID、测试 DingTalk 群。
- **Actions:** 配置测试 IMAP/POP3/POP3S 凭据和 egress；建立 replay manifest；设置只读或模拟发送模式；明确测试群和生产群凭据隔离。
- **Validation:** 测试 n8n 可读取测试输入；不会 Mark as Read、删除或修改生产邮件；输出只到测试目标。
- **Expected Output:** replay dataset、测试凭据、隔离验证报告。
- **Risk:** 测试数据污染生产或误发生产群。
- **Approval:** DA-SOC Owner、Security Owner、Business Owner。
- **Rollback:** 吊销测试凭据、删除验证数据和测试 webhook。
- **Evidence:** replay manifest、邮箱审计、DingTalk 测试消息、凭据隔离记录。
- **Dependencies:** TASK-049、TASK-003。
- **Definition of Done:** 可重复执行测试回放且不会改变生产邮箱或生产群。

### TASK-051 — 执行数据、raw、workflow 的迁移校验
- **Objective:** 证明 K8s DA-SOC 数据和工作流与基准一致。
- **Preconditions:** TASK-047～050 完成。
- **Inputs:** SQL golden baseline、raw manifest、workflow digest、测试输入。
- **Actions:** 对当天、近 6 周、近 6 月执行标准 workflow；比较 ClickHouse 行数/结果、raw 文件 hash、workflow output、null 语义和 archive/render 结果；记录 Message-ID/UID 和发送记录。
- **Validation:** 数据一致、无重复、无缺失；archive failure 不产生下游副作用；LLM 未参与数字/图片生成。
- **Expected Output:** migration comparison report、差异清单。
- **Risk:** 静默数据差异或重复处理。
- **Approval:** DA-SOC Owner、Business Owner、Security Owner。
- **Rollback:** 清理验证输出并从基准重放；不进入生产切换。
- **Evidence:** SQL diff、file hash diff、workflow log、发送记录。
- **Dependencies:** TASK-047～050。
- **Definition of Done:** 三个历史窗口和失败路径均达到业务 Owner 批准的一致性标准。

### TASK-052 — 验证邮箱和 DingTalk 生产/测试边界
- **Objective:** 证明切换所需的外部交互不会破坏业务约束。
- **Preconditions:** TASK-050～051 完成；生产邮箱只读授权和生产 DingTalk 凭据已审批。
- **Inputs:** 邮箱读取策略、Message-ID/UID、发送记录、测试/生产 DingTalk endpoint。
- **Actions:** 在隔离/人工窗口验证生产邮箱访问策略（不 Mark as Read、删除、修改）；验证测试和生产 DingTalk 目标、凭据、内容和去重 key；验证重复运行不会重复发送。
- **Validation:** 邮箱行为和目标完全符合规则；无两个生产消费者；发送记录可对账。
- **Expected Output:** 外部依赖验收报告、单活控制 Runbook。
- **Risk:** 邮件状态改变、重复 DingTalk 或目标串群。
- **Approval:** DA-SOC Owner、Business Owner、Security Owner。
- **Rollback:** 立即禁用 K8s trigger、撤销外部凭据或回到已知安全的 ECS 单活。
- **Evidence:** 邮箱服务端审计、UID/Message-ID 对账、DingTalk delivery log。
- **Dependencies:** TASK-050、TASK-051、TASK-035。
- **Definition of Done:** 邮箱、发送、去重和生产/测试隔离均由 Owner 签字确认。

## 16. Phase 12 — DA-SOC Validation

### TASK-053 — 执行端到端业务 golden test
- **Objective:** 验证 K8s DA-SOC 完成邮箱→archive→ClickHouse→SQL→render→DingTalk 闭环。
- **Preconditions:** TASK-049～052 完成；验证 Namespace 健康。
- **Inputs:** 当天、近 6 周、近 6 月 golden dataset 和 expected output。
- **Actions:** 逐窗口运行 workflow；捕获输入 Message-ID/UID、archive 结果、入库行数、SQL 结果、图片 hash、发送记录和日志/指标关联。
- **Validation:** 数据只来自 ClickHouse SQL；图片只由 render；null/暂无数据语义保留；输出与基准一致。
- **Expected Output:** golden test report、差异和批准记录。
- **Risk:** 迁移后业务链路行为变化。
- **Approval:** DA-SOC Owner、Business Owner。
- **Rollback:** 保持验证环境，修复 workflow/image/config 后重跑。
- **Evidence:** input/output manifest、SQL、图片 hash、delivery log。
- **Dependencies:** TASK-051、TASK-052。
- **Definition of Done:** 三个时间窗口端到端通过。

### TASK-054 — 执行异常、空数据和 archive 熔断测试
- **Objective:** 证明错误不会产生错误数据、图片或消息。
- **Preconditions:** TASK-053 完成；验证环境可注入失败。
- **Inputs:** archive 失败、render 失败、空数据/null、ClickHouse 不可用、DingTalk 失败、邮箱超时场景。
- **Actions:** 逐一注入异常；验证 workflow 分支、重试、幂等、告警和恢复；特别验证 `/archive` 失败时不入库、不出图、不发送。
- **Validation:** 所有失败行为符合业务规则；没有把 null 填成 0；失败任务有 Task/Incident 和可回退路径。
- **Expected Output:** negative test report、故障注入记录。
- **Risk:** 错误被吞掉造成错误日报或重复发送。
- **Approval:** DA-SOC Owner、Business Owner、Security Owner。
- **Rollback:** 关闭 trigger、恢复批准 artifact、清理验证副作用。
- **Evidence:** failure logs、ClickHouse 对账、图片/发送为空的证据、告警。
- **Dependencies:** TASK-045、TASK-049、TASK-053。
- **Definition of Done:** 所有关键异常均满足硬业务规则。

### TASK-055 — 完成生产切换前单活和回退演练
- **Objective:** 在真实切换前证明不会出现双生产消费者、重复输出或不可回退。
- **Preconditions:** TASK-052～054 完成；ECS workflow 可控且回退窗口可用。
- **Inputs:** `CUTOVER-CHECKLIST.md`、`ROLLBACK-PLAN.md`、ECS/K8s trigger 状态、Change ID。
- **Actions:** 在非业务高峰执行无副作用演练：记录 ECS/K8s trigger；模拟 ECS 禁用→K8s 启用、K8s 停止→ECS 恢复；检查 active execution=0、Message-ID/UID、发送记录和 checkpoint。
- **Validation:** 任一时刻只有一个生产消费边界；演练可在窗口内完成；无生产邮箱状态变化或重复发送。
- **Expected Output:** 单活/回退演练报告和签字。
- **Risk:** 切换期间两个 n8n 同时消费或两次发送。
- **Approval:** Platform Owner、DA-SOC Owner、Business Owner、Rollback Owner。
- **Rollback:** 演练立即回到 ECS 单活；撤销 K8s 生产凭据。
- **Evidence:** trigger 状态、时间线、执行数、发送对账、审批。
- **Dependencies:** TASK-052～054、TASK-057。
- **Definition of Done:** 单活和回退路径经 Owner 实演验证。

### TASK-056 — 生成 Cutover Gate 评审包
- **Objective:** 汇总正式切换所需全部门禁证据。
- **Preconditions:** TASK-053～055 完成；所有关键备份和恢复演练通过。
- **Inputs:** Cutover checklist、migration/golden/negative test、backup/restore、observability/security 报告。
- **Actions:** 汇总数据/图片/流程/邮箱/DingTalk/备份/恢复/监控/日志/告警/回退/单活证据；列出未关闭风险、例外和有效期；提交 GO/NO-GO 评审。
- **Validation:** 每个 Gate 有证据、Owner 和签字；Critical/High 未关闭项不得隐藏。
- **Expected Output:** Cutover Gate package、GO/NO-GO 结论。
- **Risk:** 证据不完整却进入生产。
- **Approval:** Platform Owner、Security Owner、DA-SOC Owner、Business Owner。
- **Rollback:** 若 NO-GO，停留验证环境并回到对应 TASK 修复。
- **Evidence:** 完整评审包、签字、Change ID。
- **Dependencies:** TASK-041、TASK-053～055、TASK-061～066。
- **Definition of Done:** 评审包完整且正式批准 GO 后才能执行 TASK-057。

## 17. Phase 13 — Cutover

### TASK-057 — 执行生产切换前冻结和最终备份
- **Objective:** 在生产窗口建立可回退的最后一致性点。
- **Preconditions:** TASK-056 GO；业务窗口、Owner、审批人和回退负责人在场。
- **Inputs:** Cutover Gate、Change ID、最新 Git/workflow digest、ECS/K8s 状态、backup jobs。
- **Actions:** 冻结无关变更；执行 Git、etcd、ClickHouse、raw、n8n、render 和 Harbor 备份；记录两侧 workflow digest、最后处理 Message-ID/UID、ClickHouse checkpoint 和发送记录；确认 ECS trigger 可立即停止。
- **Validation:** 所有关键备份成功并可读；active execution=0 或已批准处理完；checkpoint 可对账。
- **Expected Output:** Cutover snapshot、冻结记录、最终备份 manifest。
- **Risk:** 切换后无法准确回退或重复处理。
- **Approval:** Platform Owner、DA-SOC Owner、Backup Owner、Business Owner。
- **Rollback:** 不满足任一条件则 NO-GO，保持 ECS 单活。
- **Evidence:** backup hash、状态快照、审批和时间线。
- **Dependencies:** TASK-056、TASK-040～041。
- **Definition of Done:** 最终备份/冻结/状态对账全通过。

### TASK-058 — 禁用 ECS 生产消费者并启用 K8s 单活
- **Objective:** 将生产消费边界从 ECS 原子切换到 K8s `da-soc`。
- **Preconditions:** TASK-057 通过；切换窗口和回退 Owner 在场。
- **Inputs:** Cutover checklist、ECS/K8s trigger 控制、生产凭据、change approval。
- **Actions:** 停止 ECS n8n 生产 trigger；等待并确认执行归零；记录 checkpoint；启用 K8s `da-soc` n8n 生产 trigger；保持 ECS 进程可恢复但不得消费生产邮箱。
- **Validation:** 任何时刻只有 K8s 生产消费者；首轮流程完成且链路、数据、图片、发送和告警正常。
- **Expected Output:** K8s production active 记录和切换时间线。
- **Risk:** 双消费、漏消费、重复 DingTalk 或邮箱状态异常。
- **Approval:** DA-SOC Owner、Business Owner、Platform Owner。
- **Rollback:** 立即禁用 K8s trigger，确认归零后恢复 ECS 单活；按 checkpoint 对账。
- **Evidence:** trigger 状态、执行数、Message-ID/UID、delivery log、监控截图。
- **Dependencies:** TASK-055～057。
- **Definition of Done:** K8s 成为唯一生产消费者，首轮业务输出通过。

### TASK-059 — 观察窗口、切换签收和 ECS 回退保留
- **Objective:** 在观察窗口内确认稳定后正式签收，同时保留短期 ECS 回退能力。
- **Preconditions:** TASK-058 完成；观察窗口和成功阈值已批准。
- **Inputs:** SLI/SLO、告警、日志、业务对账、`ROLLBACK-PLAN.md`。
- **Actions:** 持续观察 n8n、archive、ClickHouse、render、邮箱、DingTalk、备份、资源和告警；完成至少一个完整业务周期；确认 ECS 不消费生产邮箱；记录异常处理和回退截止时间。
- **Validation:** 无不可解释数据差异、重复发送、错误邮箱操作、archive 熔断失败、关键备份/告警失败；Business Owner 签收。
- **Expected Output:** Cutover sign-off、观察报告、ECS 回退保留/下线计划。
- **Risk:** 过早宣布成功导致隐性问题进入常态。
- **Approval:** Business Owner、DA-SOC Owner、Platform Owner、Security Owner。
- **Rollback:** 观察窗口内触发条件满足即按 TASK-058 回退；窗口外按 Incident/Change 执行。
- **Evidence:** SLI 报告、业务签字、异常/回退记录。
- **Dependencies:** TASK-058、TASK-032、TASK-041。
- **Definition of Done:** 生产单活稳定，且回退能力已验证并按批准窗口保留。

## 18. Phase 14 — AI Ops MVP

### TASK-060 — 实现 Git Task、Job 模板和审批入口
- **Objective:** 建立短生命周期 Agent Job 的最小执行框架。
- **Preconditions:** TASK-004、TASK-036、TASK-042、TASK-059 完成；Agent image digest 已冻结。
- **Inputs:** Task YAML/Markdown schema、Job template、L0/L1/L2 policy、审批渠道。
- **Actions:** 定义 Task ID、触发事件、观察证据、分析、计划、风险等级、审批、执行步骤、验证、回滚和审计字段；Job 使用固定 digest、独立 SA、deadline、TTL、资源限制；Task 进入 Git，审批后才创建 Job；把 CLM 的 `component_id`、current/target version、digest、CVE/KEV/EOL、upgrade_reason 和 compatibility evidence 纳入 Task。
- **Validation:** L0 可自动创建只读 Job；L1 无白名单被拒；L2 无人工审批不创建/不执行；完成后 Job 清理且审计可查。
- **Expected Output:** Task schema、Job template、审批 Runbook、审计格式。
- **Risk:** Agent 变成常驻高权控制器或执行未批准动作。
- **Approval:** AIOps Owner、Security Owner、Governance Owner。
- **Rollback:** 禁止创建新 Job，撤销 Agent RBAC；保留已完成审计。
- **Evidence:** Task 示例、Job YAML、权限/审批测试、审计记录。
- **Dependencies:** TASK-004、TASK-036、TASK-042。
- **Definition of Done:** Observe→Task→Approval→Job 的最小框架可运行，且能承载 CLM 升级 Task 而不绕过 L0/L1/L2。

### TASK-061 — 实现 CrashLoopBackOff 真实闭环
- **Objective:** 完成至少一个真实 Observe→Analyze→Plan→Task→Approval→Execute→Verify→Audit→Rollback 闭环。
- **Preconditions:** TASK-060 完成；可在非生产窗口对验证 workload 注入 CrashLoopBackOff。
- **Inputs:** Prometheus/Loki、Kubernetes API 只读权限、批准 Runbook、验证 Deployment。
- **Actions:** Agent Job 发现 CrashLoopBackOff；读取 Pod events/logs 和指标；生成 Git Task；按 L0/L1/L2 判断（只读分析为 L0，重启验证 workload 为受限 L1，生产 rollout 为 L2）；经审批执行批准重启/回退；再次观察并写审计。
- **Validation:** 原因、计划、执行对象和变更前后 diff 可追溯；错误执行能回滚；Job 结束后消失；不访问 Secret 或 DA-SOC 业务数据。
- **Expected Output:** 一次完整事件的 Task、审批、Job、验证、回滚和审计证据。
- **Risk:** 错误诊断导致扩大故障或越权修复。
- **Approval:** AIOps Owner、Security Owner、Platform Owner；L2 需人工批准。
- **Rollback:** 恢复 Deployment 上一版本/副本状态，撤销 Agent SA，创建 Incident。
- **Evidence:** 指标/日志时间线、Task commit、审批、Job log、diff、恢复结果。
- **Dependencies:** TASK-060、TASK-029～031、TASK-036。
- **Definition of Done:** 一个真实 CrashLoopBackOff 闭环完整通过并可复盘。

### TASK-062 — 验证 Agent 失败、超时和回滚
- **Objective:** 证明 Agent 失败不会留下隐性权限或半完成变更。
- **Preconditions:** TASK-061 完成；验证 workload 可重复部署。
- **Inputs:** Job timeout、API timeout、验证失败、回滚 Runbook。
- **Actions:** 注入分析超时、执行失败、验证失败和 Job 被删除；检查 TTL、审计、Task 状态、资源回收、回滚动作和告警。
- **Validation:** 失败均标记为失败/需人工处理；不会自动升级权限；回滚或暂停路径可执行。
- **Expected Output:** Agent failure drill 报告。
- **Risk:** Agent 失败后资源、权限或状态残留。
- **Approval:** AIOps Owner、Security Owner。
- **Rollback:** 撤销 Job/SA/临时 RoleBinding，恢复 workload。
- **Evidence:** Job events、审计、资源清单、回滚结果。
- **Dependencies:** TASK-061。
- **Definition of Done:** Agent 失败和回滚行为满足安全和审计要求。

### TASK-063 — 完成 AI Ops 维护交接和边界确认
- **Objective:** 让后续 Agent/工程师能按 Task 模型维护，而不扩大 V0.1 范围。
- **Preconditions:** TASK-060～062 完成。
- **Inputs:** Task schema、Runbook、Agent image/digest、RBAC、审计和演练证据。
- **Actions:** 编写运行/升级/回滚/暂停 Agent Runbook；登记已支持和不支持动作；明确不引入 `xw-opsapi`、常驻 Agent、Multi-Agent、Task CRD；建立版本和 drift 检查。
- **Validation:** 新维护者可从 Git Task 复现一次只读观察和一次受限执行；边界问题会阻断而非绕过。
- **Expected Output:** AI Ops handover package、支持矩阵、维护责任。
- **Risk:** 后续维护逐步扩张为未批准平台。
- **Approval:** AIOps Owner、Governance Owner、Security Owner。
- **Rollback:** 暂停 Agent 自动化，保留人工 Runbook。
- **Evidence:** 交接文档、复现记录、版本/digest、权限复核。
- **Dependencies:** TASK-060～062。
- **Definition of Done:** AI Ops MVP 可维护、可暂停、可审计且范围冻结。

## 19. Phase 15 — Failure / Recovery Drills

### TASK-064 — 执行 Control Plane / etcd 恢复演练
- **Objective:** 证明控制面和 KubeSphere 关键配置可恢复。
- **Preconditions:** TASK-012、TASK-039、TASK-041 完成；隔离 VM/环境可用。
- **Inputs:** etcd snapshot、Git config、恢复 Runbook、RTO/RPO。
- **Actions:** 在隔离环境重建 Control Plane；恢复 etcd 或 Git 声明式配置；验证 API、节点、KubeSphere、RBAC、Audit、NetworkPolicy、监控和 Namespace。
- **Validation:** 资源和权限可用；快照 hash/时间正确；达到批准 RTO/RPO；不触碰生产 Control Plane。
- **Expected Output:** D1 restore report、时间线、差异和改进项。
- **Risk:** 恢复操作误触生产或快照不可用。
- **Approval:** Platform Owner、Backup Owner、Security Owner。
- **Rollback:** 销毁隔离环境，不执行生产写入。
- **Evidence:** snapshot hash、命令日志、资源 diff、RTO/RPO、Owner 签字。
- **Dependencies:** TASK-039、TASK-041。
- **Definition of Done:** Control Plane 恢复演练通过。

### TASK-065 — 执行 ClickHouse、raw 和 n8n 恢复演练
- **Objective:** 证明 DA-SOC 数据、raw、workflow/state 和 Secret 可完整恢复。
- **Preconditions:** TASK-040、TASK-047～049、TASK-064 完成；隔离 `da-soc-restore` 环境可用。
- **Inputs:** ClickHouse BACKUP、raw manifest、n8n artifact/state/key、render config、测试 Secret。
- **Actions:** 恢复 schema/数据、raw、n8n workflow/state/encryption key、render/archive 配置；运行当天/6 周/6 月 SQL 和图片验证；执行 archive failure 和 null 语义测试。
- **Validation:** 行数、SQL、文件 hash、workflow digest、图片和失败门禁与基准一致；恢复后无生产发送。
- **Expected Output:** D2 DA-SOC restore report、差异和 RTO/RPO。
- **Risk:** 恢复顺序错误导致业务数据和图片不一致。
- **Approval:** DA-SOC Owner、Backup Owner、Business Owner、Security Owner。
- **Rollback:** 删除隔离恢复环境并重新从只读备份恢复。
- **Evidence:** backup hash、SQL/image/file 对账、workflow import、失败测试。
- **Dependencies:** TASK-040、TASK-047～049、TASK-064。
- **Definition of Done:** DA-SOC 全量恢复演练通过，且不触碰生产外部系统。

### TASK-066 — 执行 Harbor、节点/磁盘和 POP3 replay 演练
- **Objective:** 覆盖镜像供应、Local PV 故障和输入回放恢复路径。
- **Preconditions:** TASK-028、TASK-050、TASK-064～065 完成；隔离节点/Harbor 可用。
- **Inputs:** Harbor backup/tar、Local PV restore Runbook、POP3 replay manifest、测试 DingTalk。
- **Actions:** 恢复 Harbor 并让 Worker 拉取固定 digest；模拟 `xw-wk-02`/数据盘故障并恢复 ClickHouse/raw；执行 POP3/历史回放到隔离路径；验证输出但禁止业务发送。
- **Validation:** Harbor HTTPS/权限/digest、PVC 恢复、历史 SQL/图片、replay manifest 和测试目标均通过。
- **Expected Output:** D3 Harbor、Local PV、D4 POP3 replay 报告。
- **Risk:** 节点故障或回放造成重复业务消息。
- **Approval:** Registry Owner、Backup Owner、DA-SOC Owner、Business Owner。
- **Rollback:** 清理隔离环境，撤销测试凭据和 webhook；不改变生产状态。
- **Evidence:** pull digest、PVC/hash、replay SQL/image、测试发送记录为空。
- **Dependencies:** TASK-028、TASK-050、TASK-065。
- **Definition of Done:** Harbor、Local PV 和 POP3 replay 三类恢复路径均有实证。

## 20. Phase 16 — Final Acceptance

### TASK-067 — 执行架构、业务、安全、恢复综合验收
- **Objective:** 对照 Baseline、ADR 和 V0.1 Scope 做最终验收。
- **Preconditions:** TASK-059、TASK-063、TASK-064～066、TASK-CLM-006 完成；所有 Critical/High 风险已关闭或有批准例外。
- **Inputs:** Baseline、ADR、全部任务证据、Cutover/Restore/Rollback、风险和依赖清单。
- **Actions:** 检查 VM 拓扑、K8s/KubeSphere/Calico、Harbor、Local PV、观测、备份、安全、DA-SOC、单活、AI Ops、IT/Business Boundary 和 V0.1 Non-Goals；逐项标记 PASS/FAIL/EXCEPTION。
- **Validation:** 任何未满足的硬约束均阻断验收；`TODO.md` 未被修改；没有未批准新增组件。
- **Expected Output:** Final Acceptance report、残余风险和例外清单。
- **Risk:** 只验收组件健康，遗漏业务和恢复要求。
- **Approval:** Architecture/Platform Owner、Security Owner、DA-SOC Owner、Business Owner。
- **Rollback:** FAIL 时保持现状或回退生产，禁止宣布 V0.1 完成。
- **Evidence:** 签署报告、任务索引、审计、restore/cutover 证据。
- **Dependencies:** TASK-056、TASK-059、TASK-063～066、TASK-CLM-006。
- **Definition of Done:** 所有硬门禁 PASS，例外有 Owner、期限和补救任务。

### TASK-068 — 交付实施基线、运维交接和关闭 V0.1
- **Objective:** 将可运行平台和证据正式交接，并冻结后续演进入口。
- **Preconditions:** TASK-067 PASS；TASK-CLM-006 已完成；Business Owner 接受生产结果。
- **Inputs:** Final Acceptance、Git commit、运行/恢复/回退/安全/AI Ops Runbook、版本矩阵。
- **Actions:** 交付最终 manifests、版本/digest、备份索引、监控 dashboard、告警路由、RBAC/Policy、DA-SOC workflow artifact、Task/Incident 记录和已知风险；登记 V0.2 backlog（xw-opsapi 评估、HA、多副本存储、Task CRD 等）但不实施；关闭本阶段变更。
- **Validation:** 新工程师/Agent 只使用 README + Baseline + ADR + `09-implementation/` 可定位运行、验证、回退和恢复入口；所有交付文件可从 Git checkout 重现。
- **Expected Output:** V0.1 handover package、签收记录、V0.2 演进清单。
- **Risk:** 交接后知识只存在个人环境，无法长期维护。
- **Approval:** Platform Owner、Governance Owner、DA-SOC Owner、Business Owner。
- **Rollback:** 交接发现缺失时回到 TASK-067，不关闭变更；生产回退仍按正式 Runbook。
- **Evidence:** Git tag/commit、交接清单、培训/演练记录、Owner 签收。
- **Dependencies:** TASK-067、TASK-CLM-006。
- **Definition of Done:** V0.1 运行、恢复、审计和交接资料完整，后续工作不改变本 Baseline。

## 20A. Component Lifecycle Management

### TASK-CLM-001 — 建立 Component Registry Schema
- **Objective:** 在 Git 中建立 CLM 的组件身份、期望版本、Owner、关键性、发现方式和生命周期字段。
- **Preconditions:** TASK-004、TASK-005 完成；版本矩阵和 Baseline 可读取。
- **Inputs:** `07-aiops/component-lifecycle/components.yaml`、`VERSION-MATRIX.md`、Baseline、组件 Owner 清单。
- **Actions:** 维护 UOS、Kubernetes、KubeSphere、containerd、Calico、Harbor、ClickHouse、n8n、render/archive、Prometheus、Grafana、Alertmanager、Fluent Bit、Loki 和生产镜像条目；分离 desired、discovery、security、lifecycle、upgrade、audit 状态。
- **Validation:** 每个受管组件都有唯一 `component_id`、Owner、environment、criticality、discovery method 和 approval boundary；未知字段不被默认为安全。
- **Expected Output:** Git Registry schema、组件清单、字段校验规则。
- **Risk:** 组件漏登记或三类状态混成单一手工版本字段。
- **Approval:** Platform Owner、Security Owner、DA-SOC Owner、Observability Owner。
- **Rollback:** 恢复到上一版 Registry；不删除历史 discovery evidence。
- **Evidence:** YAML 校验、组件覆盖清单、Owner 签字。
- **Dependencies:** TASK-004、TASK-005。
- **Definition of Done:** Registry 覆盖 V0.1 所有平台、DA-SOC、镜像和观测组件。

### TASK-CLM-002 — 实现组件版本和 Digest Discovery
- **Objective:** 自动发现实际运行版本、镜像 repository/tag/digest 和采集时间。
- **Preconditions:** TASK-CLM-001 完成；只读 Kubernetes API、Harbor API 和节点观察权限可用。
- **Inputs:** `components.yaml` discovery method、Kubernetes API、Harbor API、节点命令、ClickHouse/n8n/观测 API。
- **Actions:** 使用组件专用 discovery method；Kubernetes/KubeSphere 读取 API 和 workload；UOS 读取 `/etc/os-release`、`uname`、RPM metadata；containerd 读取 CRI/package；Calico 读取 API/CRD/Pod image；镜像同时记录 tag 和 digest；结果写入受控 evidence artifact。
- **Validation:** 运行态版本不由人工填写；unknown version 明确标记并产生风险；digest 与实际 workload 一致；Discovery freshness 可查询。
- **Expected Output:** 短生命周期 discovery Job、结果 artifact、采集日志和失败告警。
- **Risk:** 错误版本或 tag 漂移导致错误升级判断。
- **Approval:** Platform Owner、Security Owner、Registry Owner。
- **Rollback:** 停止 discovery Job，不修改生产 workload；保留最近一次有效 evidence。
- **Evidence:** API 输出、节点命令、镜像 digest、采集时间和 hash。
- **Dependencies:** TASK-CLM-001、TASK-011、TASK-015、TASK-018、TASK-027。
- **Definition of Done:** V0.1 组件可自动发现；无法发现的组件进入 `unknown_version` 高风险状态。

### TASK-CLM-003 — 建立 Vulnerability 和 Lifecycle State
- **Objective:** 记录 CVE/CVSS/KEV、实际受影响状态、修复版本、EOL/EOS 和来源新鲜度。
- **Preconditions:** TASK-CLM-002 完成；厂商公告、CVE/NVD/CNVD/CNNVD、CISA KEV 或离线数据入口已确认。
- **Inputs:** Discovery evidence、漏洞/厂商数据、组件支持策略、离线导入清单。
- **Actions:** 建立 Vulnerability State；记录 source、last_updated、CVE、CVSS、KEV、affected、fixed_version、EOL/EOS 和证据；数据过期标记 REVIEW；禁止因数据缺失生成“无漏洞”结论。
- **Validation:** Critical/High/KEV/EOL/非受影响样例均能被正确记录；离线数据保留来源和导入时间；敏感信息不入状态或日志。
- **Expected Output:** vulnerability/lifecycle state artifact、数据源清单、过期告警。
- **Risk:** 漏洞数据陈旧、误报或把未知当安全。
- **Approval:** Security Owner、Platform Owner。
- **Rollback:** 回退到上一份带来源的状态 artifact；标记数据过期而非删除证据。
- **Evidence:** source、last_updated、评估结果、CVE/KEV/EOL 对照表。
- **Dependencies:** TASK-CLM-002、TASK-013、TASK-030。
- **Definition of Done:** Vulnerability State 可解释、可追溯，且 Unknown Version Count 可观测。

### TASK-CLM-004 — 实现 Upgrade Policy 和优先级计算
- **Objective:** 将安全、生命周期和实际受影响状态转换为可解释的升级判断。
- **Preconditions:** TASK-CLM-003 完成；`policies.yaml`、`upgrade-rules.yaml` 已评审。
- **Inputs:** Vulnerability State、Component Registry、升级规则、P0/P1/P2/P3 和 L0/L1/L2 策略。
- **Actions:** 实现 Critical/High/KEV/EOL/EOS、affected=false、unknown_version、修复版本和核心组件规则；输出 `upgrade_required`、`upgrade_reason`、priority、target_version、approval_required、status 和 deadline；不得把“存在 CVE”机械等同于升级。
- **Validation:** 每个判断可回放；Critical/KEV/EOL/受影响 High 命中对应优先级；affected=false 为 REVIEW；生产升级始终需要 L2 审批。
- **Expected Output:** policy evaluation artifact、规则测试报告、DingTalk/告警输入。
- **Risk:** 误升级或漏升级造成生产中断或安全暴露。
- **Approval:** Security Owner、Platform Owner、受影响组件 Owner。
- **Rollback:** 恢复上一版规则并重新计算；不撤销已批准但未执行的 Task。
- **Evidence:** 输入状态、规则版本、计算结果、审批边界。
- **Dependencies:** TASK-CLM-003、TASK-036。
- **Definition of Done:** 升级判断带有可读原因和优先级，且规则不执行任何升级动作。

### TASK-CLM-005 — 生成 Git Upgrade Task 和审批链
- **Objective:** 将需要升级的组件生成可审查、可回退、可验证的 Git Task。
- **Preconditions:** TASK-CLM-004 完成；Git 分支保护、Task schema、Runbook 和 Owner 已就绪。
- **Inputs:** Upgrade State、current/target version、digest、CVE/KEV/EOL、兼容矩阵、升级 Runbook。
- **Actions:** 生成包含原因、影响、目标版本、兼容性检查、备份、审批、执行、验证、回滚和截止时间的 Task；根据 L0/L1/L2 路由；L0 自动创建只读/报告 Task，L1 仅白名单和非生产，L2 需人工审批；不自动升级生产。
- **Validation:** Task 可从 Git commit 追溯到 discovery evidence；审批前不能创建升级 Job；高风险组件包含回滚和恢复证据要求。
- **Expected Output:** Git Upgrade Task、PR/审批记录、DingTalk 通知、审计关联。
- **Risk:** 自动生成不完整 Task 或绕过人工审批。
- **Approval:** Governance Owner、Security Owner、Platform Owner、受影响组件 Owner。
- **Rollback:** 关闭/回退未执行 Task；保留原因和审批审计。
- **Evidence:** Git commit、Task diff、审批、通知和依赖检查。
- **Dependencies:** TASK-CLM-004、TASK-060。
- **Definition of Done:** 每个 `upgrade_required=true` 状态都有可执行或明确阻塞的 Git Task。

### TASK-CLM-006 — 执行受控升级验证、审计和回退
- **Objective:** 在验证环境或批准的低风险范围完成升级闭环。
- **Preconditions:** TASK-CLM-005 已批准；备份、兼容性、验证和回退 Runbook 已通过评审。
- **Inputs:** Approved Upgrade Task、固定镜像 digest/包、备份点、验证脚本、L1/L2 审批。
- **Actions:** L1 仅在验证环境或白名单 Runbook 执行；L2 在人工批准窗口执行；升级前生成 backup/checkpoint，执行后自动重新 discovery、健康检查、业务 golden test、漏洞复评和日志/指标验证；失败时进入 Rollback。
- **Validation:** 当前版本变为目标版本；digest、节点、NetworkPolicy、Harbor、DA-SOC、观测和恢复状态通过；失败不留下半完成变更；Task 关闭或标记 rolled-back。
- **Expected Output:** upgrade/verify/rollback evidence、关闭 Task、审计记录、KPI 更新。
- **Risk:** 兼容性问题、数据损坏、业务中断或回退失败。
- **Approval:** L1 组件 Owner；L2 Platform/Security/Business/组件 Owner 按策略共同批准。
- **Rollback:** 按组件 Runbook 恢复旧版本、旧 digest 或备份；冻结后续升级并创建 Incident。
- **Evidence:** pre/post discovery、备份 hash、验证结果、审批、Task 状态和回退时间线。
- **Dependencies:** TASK-CLM-005、TASK-040、TASK-055、TASK-061。
- **Definition of Done:** 至少一个非生产/验证组件完成一次受控升级验证；生产自动升级保持禁用。

## 21. Dependency Graph

```text
TASK-001
  ├─> TASK-002 ─> TASK-006 ─> TASK-007 ─> TASK-008 ─> TASK-009 ─> TASK-010
  ├─> TASK-003 ─> TASK-035/TASK-040
  ├─> TASK-004 ─> TASK-049/TASK-060
  └─> TASK-005 ─> all evidence gates

TASK-010 ─> TASK-011 ─> TASK-012 ─> TASK-013 ─> TASK-014
TASK-011/TASK-015 ─> TASK-016 ─> TASK-017
TASK-011/TASK-016 ─> TASK-018 ─> TASK-019 ─> TASK-020 ─> TASK-021
TASK-021 ─> TASK-022 ─> TASK-023 ─> TASK-024
TASK-025 ─> TASK-026 ─> TASK-027 ─> TASK-028
TASK-029 ─> TASK-030 ─> TASK-031 ─> TASK-032 ─> TASK-033
TASK-034 ─> TASK-035 ─> TASK-036 ─> TASK-037
TASK-038 ─> TASK-039/TASK-040 ─> TASK-041
TASK-042 ─> TASK-043 ─> TASK-044 ─> TASK-045 ─> TASK-046
TASK-047/TASK-048/TASK-049 ─> TASK-050 ─> TASK-051 ─> TASK-052
TASK-053 ─> TASK-054 ─> TASK-055 ─> TASK-056
TASK-057 ─> TASK-058 ─> TASK-059
TASK-060 ─> TASK-061 ─> TASK-062 ─> TASK-063
TASK-064/TASK-065/TASK-066 ─> TASK-067 ─> TASK-068

TASK-004/TASK-005 ─> TASK-CLM-001 ─> TASK-CLM-002 ─> TASK-CLM-003
TASK-CLM-003 ─> TASK-CLM-004 ─> TASK-CLM-005 ─> TASK-CLM-006
TASK-CLM-006 ─> TASK-067 ─> TASK-068
```

并行原则：同一阶段中无数据依赖的任务可以并行，但不得绕过安全、备份、证据和审批门禁。所有生产切换任务严格串行。

## 22. External Dependencies

实施必须继续维护 `09-implementation/EXTERNAL-DEPENDENCIES.md`。以下依赖为阻塞型，未满足时只能停在对应任务，不得使用未批准替代方案：

| Dependency | Owner | Blocking Tasks | Required Evidence |
|---|---|---|---|
| 五台 VM、数据盘、故障域 | Infrastructure Owner | TASK-006/011 | 资产和磁盘清单 |
| VPN/堡垒机、DNS、NTP、CIDR、路由 | Network/Infrastructure Owner | TASK-007～011 | 连通性、解析、时间报告 |
| 外部出口白名单 | Network/Security Owner | TASK-020/050/052 | IMAPS/POP3S/DingTalk/API 规则 |
| Harbor 离线包、CA、镜像授权 | Registry/Security Owner | TASK-025～028 | checksum、证书、digest |
| DA-SOC 镜像、workflow、SQL、render contract | DA-SOC Owner | TASK-027/042/049 | Git artifact、版本和授权 |
| ClickHouse/raw/n8n 历史数据及密钥恢复资料 | DA-SOC/Backup/Security Owner | TASK-047～049/065 | backup manifest 和 restore evidence |
| 测试邮箱、合法回放数据、测试 DingTalk 群 | DA-SOC/Business Owner | TASK-050～055/066 | replay manifest 和隔离证明 |
| 生产邮箱只读访问和生产 DingTalk 凭据 | DA-SOC/Security/Business Owner | TASK-052/055/058 | 服务端审计、凭据边界、签字 |
| 运维 DingTalk 告警入口 | Platform/Observability Owner | TASK-032 | 告警送达记录 |
| 不可变/离线备份介质 | Backup Owner | TASK-038～041/064～066 | 副本、hash、恢复记录 |
| L2 审批人和生产切换窗口 | Platform/Business Owner | TASK-036/055～059/061 | 审批表和窗口确认 |
| CLM 只读发现权限和 Job 执行边界 | Platform/Security Owner | TASK-CLM-001 | API/RBAC/审计验证 |
| 漏洞、KEV、EOL 数据源或离线导入 | Security Owner | TASK-CLM-003 | Source、Last Updated、导入 hash |
| CLM Git 审批和升级 Runbook | Governance/Platform Owner | TASK-CLM-004 | 分支保护、审批人、回退路径 |
| 升级验证环境、备份点和维护窗口 | Platform/Backup/Component Owner | TASK-CLM-005 | validation evidence、backup hash、窗口 |

## 23. Risks

以 `09-implementation/IMPLEMENTATION-RISKS.md` 为权威风险登记。实施期间至少持续跟踪：

- 版本兼容、VM/CIDR/出口不足、Harbor 不可用、Local PV 节点/磁盘故障。
- 生产邮箱误 Mark as Read/删除/修改、ECS/K8s 双消费、重复 DingTalk、archive 熔断失效。
- Secret 丢失或进入日志、Agent 越权、审计缺失、资源/磁盘耗尽。
- 备份文件存在但无法恢复、切换后无法回退、恢复顺序错误和数据/图片不一致。
- CLM 版本发现过期、镜像 tag/digest 不一致、漏洞误判、未知版本被当作安全或升级 Task 绕过审批。

风险处置规则：Critical 风险必须立即停止受影响任务；High 风险必须有批准的缓解、监控和回退；任何 Architecture Blocker 必须提交架构 Owner，不在实施阶段自行裁决。

## 24. Definition of Done

V0.1 只有在以下条件全部满足后才算完成：

1. 五台 VM、单集群、KubeSphere、Calico、Harbor 独立 VM 和 `xw-backup-01` 与 Baseline 一致。
2. 所有生产镜像、软件包、配置和 workflow 使用固定版本/digest，并可从 Git/离线包重建。
3. `da-soc` 内实际运行 n8n、ClickHouse、render/archive 和 raw archive；`da-soc-validate` 仅用于验证且可删除。
4. n8n→render/archive→ClickHouse 的 ClusterIP 链路、邮箱出口、SQL、`/archive`、`/render` 和 DingTalk 均通过验证。
5. 数字只来自 ClickHouse SQL，图片只来自 render，LLM 不参与业务计算/出图；null/暂无数据语义真实保留。
6. `/archive` 失败时不入库、不出图、不发送；生产邮箱不 Mark as Read、删除或修改；生产/测试 DingTalk 严格隔离。
7. Prometheus、Grafana、Alertmanager、Fluent Bit、Loki 统一部署，关键告警送达运维 DingTalk，日志和审计完成脱敏。
8. Git、etcd、ClickHouse、raw、n8n、render/archive、Harbor 和重要 Secret 均有成功备份；真实 restore drill 通过。
9. Cutover Gate 全部通过；生产切换后只有 K8s n8n 单活消费，ECS 只保留短期回退能力；回退路径已实演。
10. 至少一个 CrashLoopBackOff AI Ops 闭环完成 Observe→Analyze→Plan→Task→Approval→Execute→Verify→Audit→Rollback；Agent 使用 Job 且最小 RBAC。
11. 所有任务有 Validation、Evidence、Approval、Rollback 和 Owner；所有关键风险、外部依赖和例外均登记。
12. CLM 已覆盖 V0.1 组件，版本/digest 可自动发现，CVE/KEV/EOL 状态可追溯，升级判断可解释，Git Task/审批/验证/审计/回退闭环通过；生产自动升级保持禁用。
13. 最终验收和交接完成，V0.2/V1.0 事项只进入演进清单，不提前进入 V0.1；根 `TODO.md` 是唯一实施任务源，未创建第二份 TODO。

> **执行停止条件：** 当 `TASK-CLM-006`、`TASK-067` 和 `TASK-068` 完成后停止本阶段工作，不直接开始新的 Kubernetes 实施或生成新的架构；后续实施只能依据本清单、Baseline 和 ADR 通过变更流程推进。
