# 玄武云盾 V0.1 Architecture Baseline

> 状态：**唯一实施依据 / Approved Baseline**  
> 版本：V0.1  
> 裁决日期：2026-10-08  
> 关联裁决：`10-decisions/ARCHITECTURE-ADJUDICATION-V0.1.md`  
> 适用范围：玄武云盾平台、DA-SOC v0.1、平台安全、备份、观测和 AI Ops MVP。

## 1. Architecture Statement

玄武云盾 V0.1 是一套**单站点、单 Kubernetes 集群、1 个 Control Plane + 2 个 Worker、5 台 VM 的 KubeSphere 私有云平台**。Harbor 和备份仓库位于 Kubernetes 集群外；Calico 提供默认拒绝的网络边界；local-path/Local PV 承载单实例业务数据；Prometheus + Grafana + Alertmanager + Fluent Bit + Loki 提供唯一观测栈；DA-SOC 的 n8n、ClickHouse、render/archive 和 raw archive 必须在 V0.1 内实际运行于 `da-soc` Namespace。

DA-SOC 采用**验证双跑、生产单活**：迁移期间可以用独立测试邮箱、回放数据和临时验证 Namespace 做并行验证，但不得让 ECS 和 Kubernetes 两个 n8n 同时读取生产邮箱。生产切换完成后，ECS 只作为短期回退源，不再是 V0.1 的最终生产承载架构。

AI Ops 使用短生命周期 Kubernetes Job。Agent 默认只读，L1 只能执行白名单 Runbook，L2 必须人工审批；Task 使用 Git YAML/Markdown，不使用 Task CRD，不建设常驻高权限 Agent 或 `xw-opsapi`。

## 2. Physical Architecture

### 2.1 VM 拓扑

| VM | 角色 | 建议规格 | 数据盘 | 是否进入 Kubernetes | 关键职责 |
|---|---|---:|---:|---|---|
| `xw-cp-01` | Control Plane + etcd | 4–8 vCPU / 16 GiB | 100 GiB etcd/日志 | 是 | API Server、scheduler、controller、etcd、KubeSphere 控制面 |
| `xw-wk-01` | Platform Worker | 8 vCPU / 32 GiB | 300 GiB | 是 | Ingress、平台 Job、观测 Agent、恢复辅助任务 |
| `xw-wk-02` | DA-SOC Worker | 8 vCPU / 32 GiB | 500 GiB | 是 | ClickHouse、n8n、render/archive、raw PVC |
| `xw-harbor-01` | 独立 Registry VM | 4 vCPU / 8 GiB | 500 GiB | 否 | Harbor、HTTPS、镜像权限、基础扫描、离线导入 |
| `xw-backup-01` | 独立 Backup VM | 4 vCPU / 8 GiB | 按数据量，建议 1 TiB 起 | 否 | etcd、ClickHouse、raw、n8n、render、Harbor 备份仓库 |

规格是起始值；实施前按真实数据量和监控结果调整，但不能取消 Harbor/备份与集群的故障域隔离。

### 2.2 故障域

- `xw-cp-01` 故障：Kubernetes 管理面和调度中断；已运行 Pod 可短时继续，但需要恢复 Control Plane。
- `xw-wk-01` 故障：平台辅助能力、观测采集或 Agent Job 受影响；不应导致 ClickHouse 数据丢失。
- `xw-wk-02` 故障：DA-SOC 业务面中断，需要恢复本地盘数据或从备份重建。
- `xw-harbor-01` 故障：新镜像拉取、重建和发布受阻；已有容器不因 Harbor 短时不可用而立即停止。
- `xw-backup-01` 故障：新恢复点暂不可用；必须有离线/不可变副本，不能只有一份备份。

### 2.3 OS 与基础依赖

- 所有节点使用统一的受支持 Linux LTS，具体版本按实施时 KubeSphere/Kubernetes 兼容矩阵冻结。
- Kubernetes 节点使用 containerd；构建机可使用 Docker，但 Docker daemon 不作为 Kubernetes 运行时。
- 所有节点配置企业 DNS、NTP/chrony、Asia/Shanghai 时区、静态主机名和时间同步监控。
- 禁止 root SSH；使用管理 VPN/堡垒机、个人密钥、最小 sudo 和主机防火墙。
- 关闭 swap 或将例外明确记录；启用桥接转发、IP forwarding、auditd、持久 journald 和安全补丁。
- 节点、Harbor、备份仓库使用不同的管理凭据，禁止共享管理员密码。

## 3. VM Topology

### 3.1 节点标签与污点

```text
xw-cp-01:
  node-role.xuanwu.io/control-plane=true
  taint: NoSchedule

xw-wk-01:
  node-role.xuanwu.io/platform=true

xw-wk-02:
  node-role.xuanwu.io/da-soc=true
  xuanwu.io/data-node=true
```

Control Plane 不调度 DA-SOC。ClickHouse、raw archive 和 n8n 的持久卷通过节点亲和性固定到 `xw-wk-02`；平台 Job 和观测组件优先调度到 `xw-wk-01`。

### 3.2 外部 VM 要求

`xw-harbor-01` 使用 HTTPS 和内部 CA，数据盘独立挂载；Harbor 数据、配置、数据库和关键镜像 tar 进入 `xw-backup-01`。`xw-backup-01` 不运行 Kubernetes、KubeSphere 或业务服务，并且至少有一份离线/不可变复制。

## 4. Network Architecture

### 4.1 网络分区

| 网络 | 示例 CIDR | 主要用途 |
|---|---|---|
| 管理网 | `10.20.10.0/24` | VPN、堡垒机、K8s API、KubeSphere、SSH |
| 节点网 | `10.20.20.0/24` | Control Plane/Worker 节点流量 |
| 服务网 | `10.20.30.0/24` | Harbor、备份仓库、恢复流量 |
| Pod CIDR | `10.244.0.0/16` | Calico Pod 网络 |
| Service CIDR | `10.96.0.0/12` | Kubernetes ClusterIP |

实际网段由实施前网络变更单确定，不能照抄示例地址。

### 4.2 南北向规则

- K8s API 6443：仅管理 VPN/堡垒机和必要控制面路径。
- KubeSphere 控制台：仅管理 VPN/堡垒机 HTTPS。
- etcd 2379/2380：仅 Control Plane 本机/必要集群路径，禁止业务网和公网访问。
- Harbor 443：Worker 到 Harbor 允许；管理网可访问管理界面；公网禁止。
- 备份仓库：仅备份 Job、恢复主机和受控管理路径访问。
- DA-SOC：不开放公网入站；n8n 管理 UI 仅通过管理 VPN 的受控 Ingress。

### 4.3 DA-SOC 外部出口

`da-soc` 默认拒绝 egress，只放行：

- CoreDNS/企业 DNS。
- 指定 IMAPS/POP3S 地址。
- 指定 DingTalk HTTPS/API 地址。
- 业务确认的必要外部 API。
- 备份目标的必要上传路径。

NetworkPolicy 负责 Pod 东西向边界，出口防火墙/安全组负责南北向地址白名单。不能只依赖域名匹配来假设出口安全；IP/域名变更必须有责任人和变更记录。

### 4.4 业务访问矩阵

| 来源 | 目标 | 端口 | 结果 |
|---|---|---:|---|
| n8n | render/archive | 8091 | 允许，仅 `da-soc` |
| n8n/render/archive | ClickHouse | 8123 | 允许，仅必要服务 |
| Prometheus | DA-SOC metrics/probe | 业务约定端口 | 允许，仅抓取 |
| n8n | IMAPS/POP3S | 993/995 | 允许，指定地址 |
| n8n | DingTalk | 443 | 允许，指定 API |
| Agent Job | K8s API | 443 | 只读；L1 仅限定动作 |
| Agent Job | Prometheus/Loki | HTTPS | 只读 |
| 任意业务 Pod | etcd/Kubelet/管理 API | — | 拒绝 |
| Internet | API/KubeSphere/ClickHouse/DA-SOC | — | 拒绝 |

## 5. Kubernetes Architecture

### 5.1 安装与版本

- 采用 Kubeadm 或 KubeKey 的受支持安装路径，最终安装器只选一个并写入 Git。
- Kubernetes 与 KubeSphere 版本必须来自同一官方兼容矩阵，实施前冻结完整版本号、镜像 digest 和离线包校验和。
- 禁止 `latest`、未锁定 Helm Chart 和未经兼容性验证的自动升级。
- containerd 作为运行时；Calico 作为 CNI；local-path/Local PV 作为存储实现。

### 5.2 Namespace

正式 Namespace：

| Namespace | 用途 |
|---|---|
| `kube-system` | Kubernetes、Calico、CoreDNS、local-path |
| `kubesphere-system` 及其官方系统命名空间 | KubeSphere 控制面 |
| `xw-platform` | 平台 Runbook、备份 Job、平台配置 |
| `xw-observability` | Prometheus、Grafana、Alertmanager、Fluent Bit/Loki 配置 |
| `xw-aiops` | Agent Job 模板、只读 ServiceAccount、Task 触发器 |
| `da-soc` | DA-SOC 正式生产工作负载 |

迁移验证阶段可临时创建 `da-soc-validate`，只用于测试邮箱、历史回放和独立数据；切换后必须删除或标记为不可生产使用。正式生产命名空间只有 `da-soc`。

### 5.3 KubeSphere

- Workspace：`xuanwu`。
- Project：`da-soc`，由 KubeSphere 映射到正式 Namespace。
- 平台管理员、平台运维、DA-SOC 业务维护者、审计者分别建角色。
- 禁用 Jenkins/DevOps、Service Mesh、App Store、Multi-cluster、ES Logging 等 V0.1 非必要组件。
- 控制台访问只能走管理 VPN/堡垒机。

### 5.4 RBAC

| 身份 | 权限 |
|---|---|
| `platform-admin` | 集群治理和 break-glass，高风险人工使用 |
| `platform-operator` | 平台命名空间、观测、备份和受限运维 |
| `da-soc-maintainer` | `da-soc` 内 Deployment/ConfigMap/日志/业务验证 |
| `auditor` | 只读集群、日志、监控、Git Task 和 Audit |
| `agent-readonly` | K8s/Prometheus/Loki/Harbor 只读 |
| `agent-l1-render` | 只允许 `da-soc-render` 的限定 restart/rollout 操作 |
| `da-soc-runtime` | 业务所需最小权限，不得访问集群级对象 |

`agent-l1-render` 不得读取 Secret、修改 RBAC/NetworkPolicy、删除 PVC、修改镜像 digest、访问节点或执行任意命令。

### 5.5 配额与安全

`da-soc` 必须有 ResourceQuota、LimitRange、requests/limits、探针、PDB（仅适用于实际多副本组件）和 Pod Security restricted。任何 privileged/hostNetwork/hostPID/hostIPC 例外必须有独立 ADR、审批人、到期日和验证用例。

## 6. KubeSphere Architecture

KubeSphere 是管理和治理入口，不是 DA-SOC 业务编排器。平台运维通过 KubeSphere/API 查看资源、审计、指标和日志；业务维护者只能在 `da-soc` 范围内操作。

KubeSphere 的监控能力与 Prometheus/Grafana/Alertmanager 集成，但不启用需要 Elasticsearch 的 Logging 方案；日志固定由 Fluent Bit → Loki 提供。

## 7. CNI

- CNI：Calico。
- 必须启用 NetworkPolicy。
- `da-soc` 建立 default-deny ingress 和 egress。
- 显式放行：DNS、n8n→render、n8n/render→ClickHouse、监控抓取、必要外部出口。
- 必须执行允许/禁止流量的正反向测试。
- V0.1 不启用 Hubble、Tetragon、Service Mesh 或复杂 eBPF 运行时安全。

## 8. Storage

### 8.1 StorageClass

正式业务 StorageClass 为 local-path/Local PV。每个持久卷必须声明节点亲和性、数据用途、备份级别和容量阈值。

### 8.2 DA-SOC 数据放置

| 数据 | 资源 | 位置 |
|---|---|---|
| ClickHouse 数据 | StatefulSet 单副本 + PVC | `xw-wk-02` 独立数据盘 |
| raw archive | 独立 PVC | `xw-wk-02` 独立目录/盘 |
| n8n 状态 | PVC | `xw-wk-02`，单副本 |
| render 临时文件 | emptyDir/PVC，按应用要求 | 不保存不可恢复业务真数据 |
| SQL/workflow | Git | 不以 UI 为 Source of Truth |

### 8.3 明确不采用

V0.1 不部署 Ceph、Longhorn、GlusterFS、分布式 ClickHouse、跨节点同步数据库或 Service Mesh。Local PV 的单节点风险必须由 ClickHouse BACKUP、文件备份、外部仓库和 POP3 重放共同覆盖。

## 9. Harbor

### 9.1 位置

Harbor 独立部署在 `xw-harbor-01`，不进入 Kubernetes 集群，不由 Kubernetes 生命周期管理。这样在集群重建时仍能提供镜像供应能力。

### 9.2 镜像治理

- 项目：`xuanwu/platform`、`xuanwu/da-soc`。
- HTTPS、企业 CA、机器人账号最小权限、审计和基础漏洞扫描。
- 所有工作负载使用不可变 tag 或 digest。
- 未授权 Registry 镜像必须被策略/验收拒绝。
- 镜像记录来源、版本、digest、扫描结果、导入人和导入时间。

### 9.3 离线 Bootstrap

```text
构建/获取镜像
  → 生成 digest 与扫描记录
  → docker save / 介质传输
  → xw-harbor-01 导入并推送到项目
  → Worker 通过 HTTPS 拉取
  → 记录镜像验证证据
```

Harbor 自身配置、数据库、registry data 和关键镜像 tar 必须备份；不把 Harbor 视为“装坏后再从公网拉回”的临时工具。

## 10. Observability

### 10.1 唯一栈

```text
Prometheus → Grafana → Alertmanager → n8n/DingTalk
Fluent Bit → Loki → Grafana/Agent Job
```

不使用 Elasticsearch/ELK、Promtail、第二套日志系统或独立 SIEM。

### 10.2 监控对象

- 节点：Ready、CPU、内存、磁盘、时间同步、容器运行时。
- Kubernetes：API、etcd、scheduler、controller、DNS、调度失败、PVC Pending、证书。
- KubeSphere：核心组件、控制台健康和审计。
- Harbor：可用性、存储容量、证书、扫描失败。
- DA-SOC：n8n 执行成功率、render/archive 健康、archive 失败、ClickHouse HTTP、最近日报时间、数据新鲜度、DingTalk 发送结果。
- 备份：成功/失败、最近成功时间、备份大小、恢复演练日期和目标可达性。

### 10.3 日志

Fluent Bit 收集 journald、容器 stdout/stderr、Kubernetes Audit、KubeSphere 关键审计、DA-SOC 结构化运行日志和 Agent Job 日志；写入 Loki。原始生产邮箱正文、Secret、Token 和不必要业务数据不进入 Loki/Agent 上下文。

### 10.4 告警

首批告警：NodeDown、DiskFull、MemoryHigh、PodCrashLoop、PodRestart、API/etcd异常、证书到期、Harbor不可用、备份失败、ClickHouse不可用、archive失败、日报未按时完成、DingTalk发送失败、NetworkPolicy拒绝异常增加。

平台告警发送到独立“玄武云盾运维群”；DA-SOC 业务日报仍使用其既定测试/生产群凭据。

## 11. Backup

### 11.1 Source of Truth 与备份对象

Git 是所有声明式定义的 Source of Truth，但不是业务数据仓库：

进入 Git：架构、治理、安全策略、NetworkPolicy、RBAC、Runbook、Task YAML、ADR、Incident、备份配置、DA-SOC SQL、workflow JSON、镜像 digest、Kubernetes manifests 和 KubeSphere values。

不进入明文 Git：Secret 值、邮箱密码、DingTalk token、LLM key、私钥和业务数据。Secret 使用 SOPS/age 等加密形式或外部受控文件；解密密钥只由授权人和离线备份保存。

不进入 Git：ClickHouse 数据、raw archive、Loki 数据、Prometheus TSDB、PVC 二进制内容、Harbor registry data。

### 11.2 备份策略

| 对象 | 方式 | 触发 |
|---|---|---|
| Git | 远端 + 定期离线镜像 | 每次变更 |
| etcd | `etcdctl snapshot save` | 每 6 小时、集群变更前 |
| K8s 资源 | Git manifests + 定期导出 | 每次变更、每日 |
| ClickHouse | 原生 `BACKUP`/`RESTORE` | 每日、切换前 |
| raw archive | 文件备份、校验和 | 每日 |
| n8n workflow/state | Git JSON + PVC/配置备份 | 每次变更、每日 |
| render/archive | ConfigMap、镜像 digest、PVC | 每次变更 |
| Harbor | 配置、数据库、registry data、关键镜像 tar | 每日/每周 |
| Secret | 加密导出、离线密钥 | 每次轮换 |

### 11.3 目标与演练

- RPO：DA-SOC 数据不超过 24 小时；配置变更即时进入 Git。
- RTO：普通 Pod/节点级故障分钟级；Control Plane 恢复不超过 4 小时；整套 DA-SOC 恢复不超过 8 小时。
- 保留至少 7 个日备、4 个周备和 1 个离线/不可变副本。
- V0.1 必须完成：etcd/控制面恢复、`da-soc` 资源重建、ClickHouse 恢复、raw 恢复、Harbor 恢复拉镜像、POP3 重放至少一次。

## 12. Security

### 12.1 Identity

人类使用个人账号和管理路径；平台管理员与业务维护者分离；Agent 使用独立 ServiceAccount；n8n 使用业务专用 Secret；备份恢复使用独立恢复凭据。

### 12.2 Authorization

- L0：只读检查、报告、查询、告警聚合。
- L1：白名单 Runbook，例如 `da-soc-render` 重启、非生产临时文件清理；必须有前置条件和自动验证。
- L2：RBAC、NetworkPolicy、CNI、节点、存储、生产 n8n、ClickHouse 数据删除/恢复、凭据和镜像策略变更；必须人工审批。

### 12.3 Container Security

restricted PSA、非 root、最小 capabilities、资源限制、镜像 digest、Harbor 来源、禁止 privileged/hostNetwork/hostPID/hostIPC。例外必须有 ADR 和到期时间。

### 12.4 Audit

Audit 必须记录：用户/Agent/ServiceAccount、资源、动作、时间、结果、Task ID、审批、diff、验证和回滚。Secret 审计只记录 metadata，避免把值写进日志。

## 13. RBAC

RBAC 清单、Role、RoleBinding 和 ServiceAccount 定义进入 Git。任何 `cluster-admin` 使用都必须是 break-glass、人工、短时、双人确认并产生 Incident/Task 记录。

Agent Job 不得：

- 访问 Secret 值，除非某个已批准 Runbook 明确需要且凭据临时注入。
- 修改 RBAC、NetworkPolicy、CNI、StorageClass、节点或 etcd。
- 删除业务数据/PVC。
- 执行任意 shell 或宿主机命令。

## 14. NetworkPolicy

`da-soc` 至少包含：

1. default-deny ingress。
2. default-deny egress。
3. DNS egress。
4. n8n → render/archive ingress。
5. n8n/render/archive → ClickHouse ingress。
6. Prometheus → metrics/probe ingress。
7. 必要 IMAPS/POP3S/DingTalk egress。
8. 明确禁止到 API Server、etcd、Kubelet 和其他业务 Namespace。

策略变更必须先在 `da-soc-validate` 验证，再进入正式 `da-soc`。

## 15. DA-SOC Hosting

### 15.1 正式部署

```text
Namespace: da-soc

n8n Deployment (replicas=1, production active)
  ├─ ClusterIP → da-soc-render:8091
  ├─ ClusterIP → clickhouse:8123
  ├─ Egress → IMAPS/POP3S
  └─ Egress → DingTalk HTTPS

da-soc-render Deployment (replicas=1)
ClickHouse StatefulSet (replicas=1 + Local PV)
raw archive PVC
n8n state PVC
ConfigMap / Secret / NetworkPolicy / Quota / probes
```

### 15.2 业务不变量

平台迁移不得改变：

- n8n 的日常编排权。
- SQL Path A 和 workflow 构建方式。
- ClickHouse 确定性出数。
- render 出图。
- archive 失败熔断。
- 无数据的真实语义。
- 生产邮箱不 Mark as Read。
- DingTalk 目标来自凭据。
- LLM 不进入业务计算链路。

## 16. AI Ops

### 16.1 Agent Job

每个 Agent 任务创建一个短生命周期 Job：

- 镜像固定 digest。
- ServiceAccount 独立。
- `activeDeadlineSeconds`、资源 requests/limits、`ttlSecondsAfterFinished`。
- 默认只读；L1 仅绑定限定 Role。
- 日志写入 Loki；Task YAML 写入 Git。
- Job 终止后不保留常驻控制通道。

### 16.2 直接数据源

V0.1 不建设 `xw-opsapi`。Agent Job 直接通过最小权限访问 Kubernetes API、Prometheus、Loki、Git 和 Harbor API。所有访问凭据只读、短时或受限；V0.2 再按真实重复适配和授权复杂度评估聚合层。

### 16.3 Task YAML

```yaml
api_version: xuanwu/v0.1
kind: Task
metadata:
  id: TSK-YYYYMMDD-0001
  created_at: 2026-10-08T00:00:00+08:00
  source: alert|human|schedule
spec:
  target: namespace/resource
  severity: low|medium|high|critical
  risk: L0|L1|L2
  evidence:
    - source: prometheus|loki|kubernetes|git
      reference: "..."
  impact: "..."
  plan: "..."
  rollback: "..."
approval:
  required: true|false
  approver: "..."
  decision: pending|approved|rejected
execution:
  job: "..."
  action: "..."
  started_at: "..."
  finished_at: "..."
verification:
  status: pending|passed|failed
  evidence: "..."
audit:
  agent_digest: "..."
  git_commit: "..."
  result: open|closed|escalated
```

### 16.4 真实验证场景

`da-soc-render` CrashLoopBackOff：告警 → n8n 创建 Task → Agent Job 收集 K8s/Prometheus/Loki 证据 → 判断 L1/L2 → 受限重启或请求审批 → 健康探针和 n8n→render 连通性验证 → 失败则 rollout undo/升级人工 → Task 和审计完成。

### 16.5 Component Lifecycle Management

V0.1 纳入组件生命周期管理（CLM）最小闭环，但不建设独立漏洞平台或自动 Patch Management 平台。CLM 的 Git Source of Truth 位于 `07-aiops/component-lifecycle/`，由 `components.yaml`、`policies.yaml` 和 `upgrade-rules.yaml` 分别维护 Component Registry、升级/审批策略和 `upgrade_required` 规则。

CLM Agent Job 按组件类型发现当前版本、运行状态、镜像 repository/tag/digest、CVE/CVSS/KEV/EOL 和来源时间；未知版本必须进入高风险 REVIEW。CLM 可以生成 Git Upgrade Task、通知、审计、验证和回退，但生产升级、Kubernetes/KubeSphere/Calico/Harbor/OS/ClickHouse/节点/存储/网络/RBAC 变更必须遵守 L2 人工审批；V0.1 禁止生产自动升级、自动 Kubernetes minor/major upgrade 和自动 OS upgrade。

## 17. Human / AI Boundary

| 能力 | 人 | AI |
|---|---|---|
| 策略、安全红线、业务口径 | 决定 | 提取/检查 |
| L0 查询与报告 | 监督 | 执行 |
| L1 白名单动作 | 定义策略 | 执行和验证 |
| L2 平台/业务高风险变更 | 审批/拒绝 | 生成计划和证据 |
| DA-SOC 出数/出图 | 负责业务定义 | 不参与 |
| 生产邮箱和凭据 | 负责授权 | 不直接操作 |
| 恢复与重大事故 | 决策和升级 | 提供 Runbook/证据 |

## 18. IT / Business Boundary

IT/平台：VM、OS、Kubernetes、KubeSphere、Calico、Storage、Harbor、Ingress、RBAC、NetworkPolicy、监控、日志、备份、审计、安全基线、Agent Runtime。

业务/DA-SOC：镜像内容、n8n 工作流、SQL、解析、render 逻辑、邮箱 Filter、DingTalk 业务目标、业务数据、业务指标、SLA、统计口径。

任何跨界请求必须通过 Namespace、RBAC、Quota、NetworkPolicy、Secret 和审批流程，不能以共享管理员账号或临时 hostNetwork 解决。

## 19. Risk Levels

### L0

只读状态、健康检查、日志/指标查询、备份年龄、证书检查、日报和 Task 汇总。

### L1

受策略范围限制的非破坏动作，例如重启 `da-soc-render`、重启非关键平台 Pod、清理已确认的临时文件。必须自动验证，失败自动升级人工。

### L2

修改 RBAC/NetworkPolicy/CNI/StorageClass、删除节点/PVC/业务数据、修改生产 n8n、恢复生产 ClickHouse、修改邮箱/DingTalk 凭据、镜像来源和生产切换。必须人工批准，使用短时授权，并记录变更 diff、影响和回滚。

## 20. Deployment Sequence

```text
S0  Baseline/ADR/参数确认
S1  VM、OS、DNS、NTP、SSH、防火墙、磁盘基线
S2  Harbor 与备份仓库、离线镜像 Bootstrap
S3  Kubernetes + containerd + Calico + Audit
S4  KubeSphere 精简安装、Workspace、Project、RBAC
S5  local-path、节点标签、Quota、PSA、NetworkPolicy
S6  Prometheus/Grafana/Alertmanager + Fluent Bit/Loki
S7  etcd/Git/ClickHouse/raw/n8n/Harbor/Secret 备份
S8  恢复演练：控制面、资源、ClickHouse、Harbor、raw
S9  da-soc-validate 回放验证
S10 da-soc 正式资源创建、数据导入、workflow 导入
S11 Cutover Gate 评审，停止 ECS n8n，启动 K8s n8n
S12 观察一个完整日报周期，确认回退路径
S13 Agent Job + Git Task + CrashLoop/Disk Pressure 闭环
S14 安全测试、故障演练、Baseline/Runtime 对照验收
```

未完成 S7/S8 的恢复验证，不得进入 DA-SOC 生产切换。

## 21. Acceptance Criteria

### 平台

- Kubernetes、KubeSphere、Calico、DNS、NTP、节点和平台配额正常。
- API/etcd 不暴露公网；管理员只能经授权路径访问。
- Harbor HTTPS、digest、离线导入、未授权 Registry 拒绝测试通过。

### 网络与安全

- `da-soc` default-deny 生效。
- 允许的 DNS、业务、监控和外部出口可通。
- 禁止的跨 Namespace、etcd、Kubelet、管理 API 流量不通。
- RBAC 越权、privileged、hostNetwork、Secret、Audit 测试通过。

### DA-SOC

- n8n、ClickHouse、render/archive、raw archive 均运行在 `da-soc`。
- 当天、近 6 周、近 6 月数据验证通过。
- SQL 结果、图片关键内容、空数据语义和 archive 失败门禁与基准一致。
- 邮箱 ALL/不标已读行为一致。
- DingTalk 测试群发送成功，生产/测试凭据隔离。
- 生产切换无两个 n8n 同时消费邮箱，幂等和回退验证成功。

### 备份与恢复

- etcd、K8s 资源、ClickHouse、raw、n8n、render、Harbor、Secret 均有备份记录。
- 完成至少一次真实控制面恢复、ClickHouse 恢复、Harbor 恢复和 POP3 重放。
- 恢复后的数据、SQL、图片和日报业务验证通过。

### AI Ops

- Agent Job 能读取平台上下文。
- Agent 能创建 Git Task、判断风险、请求 L2 审批。
- 至少一个 CrashLoopBackOff 或 Disk Pressure 场景完成 Observe→Analyze→Plan→Task→Approval→Execute→Verify→Audit→Rollback/升级闭环。
- Agent 没有 cluster-admin、root 或任意 shell 权限。

## 22. Rollback Strategy

### 技术回退

适用于首次生产输出前发现镜像、网络、探针、配置或启动异常：停止 K8s n8n、恢复 ECS 工作流、恢复原凭据/配置、验证旧链路。

### 业务回退

适用于已产生生产输出后发现数据差异、重复发送、邮件行为异常、archive/render 异常或不可解释缺失：先冻结消费和发送，保存 K8s 证据、ClickHouse 快照和 Task；业务负责人确认后选择 K8s 内恢复、从备份恢复或按幂等状态回退 ECS。任何回退都必须保证单活消费者。

### 立即停止条件

```text
数据差异
图片差异
/archive 行为异常
/render 行为异常
ClickHouse 写入/查询异常
邮件 Mark as Read 或读取范围异常
DingTalk 目标/凭据异常
备份或恢复校验失败
NetworkPolicy/RBAC 安全边界失效
无法解释的数据缺失
```

## 23. Future Evolution

- V0.2：三控制面 HA、集中身份、密钥服务、跨节点存储 PoC、可选 `xw-opsapi`。
- V0.3：运行时安全、EDR/SIEM、供应链签名、策略即代码。
- V0.4：多 Agent、Task 查询优化、按真实需求评估 Task CRD、更多 L1 自动化。
- V0.5：多业务、多租户、容量/成本和分布式存储。
- V1.0：企业级私有云、跨站点恢复、统一安全运营和成熟 AI Ops。

## 24. Baseline Rules

1. 本文件是实施唯一架构依据；候选方案不具有实施权威性。
2. `TODO.md` 不由本阶段修改；最终实施 TODO 必须以本文件重新生成。
3. 任何偏离本文件的实现必须先创建 ADR、说明影响、验证和回滚，再经人工批准。
4. 不得在 Baseline 未冻结前开始 Kubernetes 实施。
5. 不得以“测试环境”为理由取消安全、审计、备份、恢复和切换门槛。
