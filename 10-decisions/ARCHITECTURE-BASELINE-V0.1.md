# 玄武云盾 V0.1 Architecture Baseline

> **文档性质：** **V0.1 实施唯一架构依据（Single Source of Truth for Implementation）**
> **上位依据：** `README.md`、`10-decisions/ARCHITECTURE-ADJUDICATION-V0.1.md`
> **配套 ADR：** `10-decisions/ADR/ADR-001` ～ `ADR-008`、`ADR-011`（见 §24）
> **生效范围：** 玄武云盾 V0.1 全部基础设施建设、平台部署、DA-SOC 承载、AI Ops MVP 与验收
> **使用方式：** 一个工程师或一个 Agent **只看本文件**即可知道玄武云盾 V0.1 应该搭成什么样；遇到架构争议，以本文件为唯一裁决依据

---

## 1. Architecture Statement

### 1.1 一句话架构

> **玄武云盾 V0.1 是一个由 7 台 VM 承载的、以 kubeadm 安装的单集群 Kubernetes（v1.26.x，3 控制面 + 2 业务 Worker）私有云平台：KubeSphere 3.4.x 提供管理面，Calico 提供默认拒绝的网络边界，Harbor 与 MinIO 部署在集群之外以确保恢复路径独立，节点本地存储配合 Git + etcd 快照 + Velero + ClickHouse 原生备份 + 源邮件重放实现"可重建、可恢复、可重放"，kube-prometheus-stack 与 Fluent Bit/Loki 构成唯一一套指标与日志，Argo CD 让 Git 成为定义态的唯一事实源并提供一键回滚，平台 n8n 负责触发、短生命周期 Kubernetes Job 承载 Agent、Git 中的轻量 Task 记录承载状态与审计——并在此平台之上实际承载 DA-SOC v0.1 的完整生产链路（n8n + ClickHouse + render/archive）。**

### 1.2 架构分解

| 维度 | 定义 |
|---|---|
| 拓扑 | 1 个集群、7 台 VM、2 个集群外服务节点（Harbor / MinIO）、8 个 Namespace |
| 高可用 | **控制面 HA**（3 CP，stacked etcd）；**业务数据不 HA**（单副本 + RPO 24h，显式接受） |
| 业务承载 | 1 个业务（DA-SOC v0.1），`n8n` + `ClickHouse` + `da-soc-render` 全部在集群内 |
| 边界机制 | Namespace + RBAC + ResourceQuota + NetworkPolicy（default-deny）+ Pod Security Admission |
| 恢复机制 | Git + etcd 快照 + Velero + ClickHouse BACKUP + 文件级 + 邮件重放；**4 项强制演练** |
| 审计机制 | K8s Audit（90 天）+ Git（定义与 Task）+ Agent 三方交叉审计 |
| AI 机制 | 平台 n8n 触发 + Agent Job 执行 + Task 文件承载 + 3 个最小权限 SA + L0/L1/L2 分级 |
| 可信基线 | **全部定义在 Git；集群可从 Git 重建** |

### 1.3 V0.1 唯一核心目标

> **安全、稳定、可维护地承载 DA-SOC v0.1；并验证 AI-Native 运维闭环真实成立。**

**验收不以"组件安装完成"为标准**（README §22）；每一项验收必须给出可验证证据。

### 1.4 十条不可违反的架构约束

```text
1.  V0.1 必须实际承载 DA-SOC v0.1（n8n + ClickHouse + render/archive 均在集群内运行）。
2.  同一能力域只允许存在一套实现（一套指标、一套日志、一个 Registry、一套 Agent 运行时）。
3.  默认拒绝：网络、RBAC、镜像来源三者均默认拒绝、逐条允许。
4.  Git 是定义态的唯一事实源；运行态与 Git 不一致即为漂移，必须产生 Task。
5.  备份必须与备份对象分属不同故障域；未做过真实恢复演练的备份不算完成。
6.  LLM 不得参与业务出数、出图、SQL 修改或生产邮箱操作。
7.  Agent 只读是默认；写操作必须白名单、可验证、可回滚；无法写出验证方法的动作不得执行。
8.  不可逆操作一律 L2，且默认由人执行；Agent 不得自行判定风险等级。
9.  Secret 不得明文进入 Git。
10. 任何组件不得以"更企业级"为理由进入 V0.1；新增组件必须回答"不做它会阻断哪项验收"。
```

---

## 2. Physical Architecture

### 2.1 VM 清单（7 台 + 1 台既有 ECS）

| # | 主机名 | 角色 | vCPU | RAM | 系统盘 | 数据盘 | IP | 进 K8s |
|---|---|---|---|---|---:|---|---|---:|
| 1 | `xw-cp-01` | Control Plane + etcd | 8 | 32 G | 200 G | — | 10.20.0.11 | ✅ |
| 2 | `xw-cp-02` | Control Plane + etcd | 8 | 32 G | 200 G | — | 10.20.0.12 | ✅ |
| 3 | `xw-cp-03` | Control Plane + etcd + kube-vip | 8 | 32 G | 200 G | — | 10.20.0.13 | ✅ |
| 4 | `xw-wk-01` | Worker（**业务**：DA-SOC） | 16 | 64 G | 200 G | 1 T | 10.20.0.21 | ✅ |
| 5 | `xw-wk-02` | Worker（**平台 / 可观测**） | 16 | 64 G | 200 G | 1 T | 10.20.0.22 | ✅ |
| 6 | `xw-mgmt-01` | **Harbor** + 管理面入口 + 跳板 | 8 | 32 G | 200 G | 1 T | 10.20.0.10 | ❌ |
| 7 | `xw-bak-01` | **MinIO** + etcd 快照副本 + raw 全量归档 + 镜像 tar | 4 | 8 G | 100 G | **2 T** | 10.20.0.30 | ❌ |
| — | ECS（既有） | DA-SOC 迁移期数据源 + 回退保障 | — | — | — | — | 现网 | ❌ |

**合计：68 vCPU / 264 GB RAM / 约 7.2 TB 存储。**
**VIP：** `10.20.0.100`（kube-apiserver，kube-vip ARP 模式）。

### 2.2 每台 VM 的职责与不可合并性

| VM | 职责 | 为什么不能合并 |
|---|---|---|
| `xw-cp-01/02/03` | 控制面 + etcd 仲裁 | 3 台是 etcd 多数派最小值；少于 3 台即失去 HA |
| `xw-wk-01` | DA-SOC 业务（ClickHouse / n8n / render / raw 窗口） | 业务数据需独立数据盘，且不应与平台可重建组件争抢 I/O |
| `xw-wk-02` | 平台与可观测（Prometheus / Loki / 平台 n8n / Agent Job / Argo CD / Velero） | 与业务分离后，观测组件故障不影响日报链路 |
| `xw-mgmt-01` | **Harbor**（镜像供给，恢复路径）+ 管理入口 | **Harbor 必须在集群故障域之外**（集群重建时由它提供镜像） |
| `xw-bak-01` | **MinIO**（备份目标，恢复路径） | **备份必须与备份对象分处不同故障域** |
| ECS（既有） | 迁移期数据源 + 回退路径 | 既有资产，位于不同环境 |

**降级方案（仅在资源确实受限时，且必须登记 ADR）：** 将 `xw-mgmt-01` 与 `xw-bak-01` 合并为 1 台（Harbor + MinIO 同机，数据盘 ≥ 2.5 TB）。**代价：Harbor 与备份共故障域。控制面 3 台与两台 Worker 不建议合并。**

### 2.3 节点资源预留

```text
system-reserved: cpu=500m, memory=1Gi
kube-reserved:   cpu=500m, memory=1Gi
eviction-hard:   memory.available<500Mi, nodefs.available<10%, imagefs.available<15%
imageGCHighThresholdPercent: 80
imageGCLowThresholdPercent: 70
```

### 2.4 数据盘与目录规划

| 节点 | 挂载点 | 用途 |
|---|---|---|
| `xw-wk-01` | `/data` | `/data/clickhouse`（local PV）、`/data/da-soc-raw`、`/data/n8n` |
| `xw-wk-02` | `/data` | `/data/prometheus`、`/data/loki`、`/data/xw-pv`（local-path 默认父目录） |
| `xw-mgmt-01` | `/data` | `/data/harbor` |
| `xw-bak-01` | `/data` | `/data/minio` |

### 2.5 OS 与基础依赖（全部节点）

| 项 | 规格 |
|---|---|
| OS | Ubuntu 22.04 LTS（全节点统一） |
| 内核参数 | 关闭 swap；`ip_forward=1`；`bridge-nf-call-iptables=1`；`overlay`；`vm.max_map_count=262144` |
| 文件系统 | 系统盘 ext4；数据盘 xfs |
| 容器运行时 | containerd（不用 Docker daemon；Docker 仅用于构建机 `docker save`） |
| 时间同步 | chrony → 内网 NTP 源；时区 `Asia/Shanghai`；偏移纳入监控 |
| DNS | 内网 DNS 解析 `*.xw.internal`；集群内 CoreDNS；**不新建 DNS 服务器** |
| 基础包 | curl、jq、git、rsync、kubectl、helm、containerd、chrony、auditd |
| 主机名规范 | `xw-<role>-<nn>` |

### 2.6 主机安全基线（全部节点，必做）

```text
[ ] 禁止 root SSH 直登；仅允许密钥登录；sudo 需授权
[ ] 主机防火墙：仅放行管理网段 → 22；集群必要端口；出向按需
[ ] 最小化安装，关闭无用服务
[ ] auditd 开启（至少覆盖 /etc/kubernetes、kubelet 配置）
[ ] AppArmor 保持默认启用
[ ] 系统补丁：月度窗口 + 变更记录；紧急补丁走例外流程
[ ] 所有基线操作以脚本 + Git 记录，禁止纯手工
[ ] 时间同步与磁盘水位纳入监控
```

---

## 3. VM Topology

### 3.1 逻辑平面

```text
┌──────────────────────── 管理面（VLAN 10 · MGMT 10.20.10.0/24）────────────────────────┐
│  管理终端 ──► Ingress VIP（10.20.0.200） ──► KubeSphere / Grafana / Harbor UI / Alertmanager │
│  管理终端 ──► SSH(22) ──► 全部节点（仅管理网段）                                          │
└──────────────────────────────────────────────────────────────────────────────────────────┘

┌──────────────────────── 集群面（VLAN 20 · K8S 10.20.0.0/24）───────────────────────────┐
│  xw-cp-01 ─┐                                                                             │
│  xw-cp-02 ─┼─► etcd 集群（stacked） / kube-apiserver:6443 ◄── kube-vip VIP 10.20.0.100   │
│  xw-cp-03 ─┘                                                                             │
│  xw-wk-01（业务负载）   xw-wk-02（平台/可观测负载）                                        │
└──────────────────────────────────────────────────────────────────────────────────────────┘

┌──────────────────────── 服务面（集群外）────────────────────────────────────────────────┐
│  xw-mgmt-01：Harbor（harbor.xw.internal:443）                                             │
│  xw-bak-01：MinIO（minio.xw.internal:9000）— 仅集群 → 备份节点单向可达                     │
└──────────────────────────────────────────────────────────────────────────────────────────┘

┌──────────────────────── 回退面（迁移期）────────────────────────────────────────────────┐
│  既有 ECS：DA-SOC 现有生产链路（只读数据源 + 回退；切换后冷备，V0.1 内保留）                │
└──────────────────────────────────────────────────────────────────────────────────────────┘
```

### 3.2 标签体系（强制）

| 标签 | 用途 | 取值 |
|---|---|---|
| `xuanwu.io/tier` | 风险分级与备份策略锚点 | `A` / `B` / `C` |
| `xuanwu.io/owner` | 归属 | `platform` / `da-soc` / `aiops` |
| `xuanwu.io/component` | 组件标识 | 如 `clickhouse`、`n8n`、`prometheus` |
| `xuanwu.io/backup` | 备份策略 | `daily` / `weekly` / `none` |
| `node-role.xuanwu.io/business` | 业务节点标记 | `"true"`（`xw-wk-01`） |
| `node-role.xuanwu.io/platform` | 平台节点标记 | `"true"`（`xw-wk-02`） |

**要求：** 所有工作负载对象**必须**带 `xuanwu.io/tier` 与 `xuanwu.io/owner`（缺失由每日漂移检查产生 Task）。

---

## 4. Network Architecture

### 4.1 网段规划

| 用途 | 网段 | 说明 |
|---|---|---|
| 管理网 MGMT | `10.20.10.0/24` | 人 → Ingress VIP / SSH；与互联网不通 |
| 节点网 K8S | `10.20.0.0/24` | 节点间通信、VIP `10.20.0.100` |
| 备份网 BACKUP | `10.20.30.0/24` | 仅集群 → `xw-bak-01` 单向可达 |
| Ingress VIP 池 | `10.20.0.200/29` | MetalLB L2，仅管理面使用 |
| Pod CIDR（Calico） | `10.233.64.0/18` | 与节点网段不重叠 |
| Service CIDR | `10.233.0.0/18` | 与 Pod CIDR 细分不重叠（实施时按 Calico 文档确认） |
| DMZ | **V0.1 空置** | 无对外服务 |

### 4.2 访问控制矩阵（默认拒绝）

| 源 → 目的 | MGMT | 节点 | Pod | apiserver 6443 | etcd 2379 | Harbor 443 | MinIO 9000 | 邮件 993 | DingTalk 443 |
|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|
| 管理终端 | — | ✅(22) | ❌ | ✅(VIP) | ❌ | ✅ | ❌ | ❌ | ❌ |
| 节点 | ❌ | ✅ | ✅ | ✅ | ✅(CP 间) | ✅ | ✅(仅备份 Job) | ❌ | ❌ |
| `da-soc` Pod | ❌ | ❌ | ✅(策略内) | ❌ | ❌ | ✅(拉镜像) | ✅(仅 CH 备份) | ✅ | ✅ |
| `xw-ops` Pod | ❌ | ❌ | ✅(策略内) | ✅ | ❌ | ✅ | ✅ | ❌ | ✅(仅 n8n) |
| `xw-obs` Pod | ❌ | ❌ | ✅(抓取) | ✅ | ❌ | ✅ | ❌ | ❌ | ❌ |
| 业务人员终端 | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | — |
| 互联网 | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | — |

**三条红线：** ① API 不暴露互联网；② etcd 不跨控制面节点暴露；③ `da-soc` 出向仅"集群内 + IMAP 993 + DingTalk 443 + MinIO 备份"。

### 4.3 南北向 / 东西向 / 出网职责划分

| 方向 | 承担者 | 说明 |
|---|---|---|
| 东西向（Pod 间） | **NetworkPolicy** | 每个命名空间 default-deny + 显式放行 |
| 南北向入向 | **ingress-nginx（管理面专用，来源白名单限管理 VLAN）** | 无业务入向入口 |
| 南北向出网 | **边界防火墙 / 安全组 IP+端口白名单** | NetworkPolicy 不承担域名白名单 |
| DNS | CoreDNS → 内网 DNS（不直连公网 DNS） | — |

### 4.4 NetworkPolicy 强制要求

```yaml
# 每个业务/平台命名空间必须有（含 da-soc / xw-ops / xw-obs / xw-system）
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: { name: default-deny-all, namespace: <ns> }
spec:
  podSelector: {}
  policyTypes: [Ingress, Egress]
```

**`da-soc` 放行清单（8 条，逐条必须有用途/责任/验证方法）：**

| # | 方向 | 源 → 目的 | 端口 | 用途 |
|---|---|---|---|---|
| 1 | Egress | `da-soc` → `kube-system` CoreDNS | 53 | 域名解析 |
| 2 | Ingress | `da-soc/n8n` → `da-soc/da-soc-render` | 8091 | `/archive`、`/render` |
| 3 | Ingress | `da-soc/n8n` → `da-soc/clickhouse` | 8123, 9000 | HTTP SQL 与写入 |
| 4 | Ingress | `xw-obs/prometheus` → `da-soc` | metrics | 指标抓取 |
| 5 | Egress | `da-soc/n8n` → 邮件服务器 | 993 | IMAP（**只读，不设 `\Seen`**） |
| 6 | Egress | `da-soc/n8n` → DingTalk API | 443 | POC-06A/06C 发送 |
| 7 | Egress | `da-soc/clickhouse` → MinIO | 9000 | `BACKUP` |
| 8 | Egress | **仅迁移期** `da-soc/migration-job` → ECS ClickHouse | 8123 | 历史回补；**完成后立即删除** |

**禁止：** `0.0.0.0/0` 出向；跨命名空间通配（`namespaceSelector: {}`）；对 `kube-system` 的非必要访问；NodePort；hostNetwork。

---

## 5. Kubernetes Architecture

### 5.1 发行与版本

| 项 | 规定 |
|---|---|
| 安装器 | **kubeadm** |
| 版本 | **v1.26.x**；建集群时取该 minor 最新安全补丁（如 v1.26.15），**精确值在 Phase 0 核实后写入 `configs/version-matrix.yaml`** |
| 控制面 | 3 节点 stacked etcd |
| Worker | 2 节点 |
| 容器运行时 | containerd |
| VIP | kube-vip（ARP 模式，静态 Pod），`10.20.0.100:6443` |
| 双栈 | 不启用 IPv6 |
| 配置来源 | `InitConfiguration` / `ClusterConfiguration` / `JoinConfiguration` / `KubeletConfiguration` 全部文件化进 Git |

### 5.2 强制配置（kube-apiserver）

```text
--authorization-mode=Node,RBAC          # 禁止 AlwaysAllow
--anonymous-auth=false
--audit-policy-file=/etc/kubernetes/audit/policy.yaml
--audit-log-path=/var/log/kubernetes/audit/audit.log
--audit-log-maxsize=100 --audit-log-maxbackup=10 --audit-log-maxage=30
--encryption-provider-config=/etc/kubernetes/enc/encryption-config.yaml
--enable-admission-plugins=NodeRestriction,PodSecurity
```
**禁止：** `--insecure-port`；任何形式的匿名认证；将 apiserver 绑定到非管理网段。

### 5.3 控制面与 Worker 约束

| 项 | 规定 |
|---|---|
| 控制面污点 | `NoSchedule`（业务与平台负载均不调度到控制面） |
| 业务负载 | `nodeSelector` 固定 `xw-wk-01` |
| 平台负载 | `nodeSelector` 固定 `xw-wk-02` |
| etcd 快照 | CronJob 每 **30 分钟**；本地 2 天 + 远端 30 天 |
| 证书 | `kubeadm certs check-expiration` 纳入日常巡检；续期走 Runbook（**L2**） |
| 升级 | 每次最多 1 个 minor，**L2 审批** + etcd 快照 + 回滚方案 |
| 生命周期 | 安装/升级/扩缩容**只通过 Git 中的 kubeadm 配置 + 幂等脚本** |

### 5.4 DaemonSet 与关键系统组件（`kube-system`）

Calico（VXLAN，保留 kube-proxy）、kube-vip、kube-proxy、CoreDNS、MetalLB（L2，VIP 池 `10.20.0.200/29`）、ingress-nginx（2 副本，分布在两个 Worker）、local-path-provisioner、node-exporter、Fluent Bit。

**YAML 静止加密策略（不可违反）：** 对 `Secret`、`RBAC`、`NetworkPolicy`、`ResourceQuota`、`pods/exec`、`pods/portforward` 记 `RequestResponse`；**对 Secret 只记元数据，禁止记录 Secret 明文内容**。

---

## 6. KubeSphere Architecture

| 项 | 规定 |
|---|---|
| 版本 | **3.4.x**（**不采用 4.x**：LuBan 架构变化大，对无专家团队风险过高） |
| 部署方式 | 在已就绪的 kubeadm 集群上以 Helm / 官方 manifest 安装；**不采用 kubeadm 一体化安装器接管集群** |
| 定位 | **管理面 UI + 用户管理 + RBAC 视图 + 审计入口**；**不是业务运行时** |
| 自带组件开关 | Monitoring **关** / Logging **关** / Auditing **关** / DevOps **关** / Service Mesh **关** / App Store **关** / Edge & Multicluster **关** / Alerting & Notification **关**（告警统一走 Alertmanager → 平台 n8n） |
| Workspace | 建 1 个 Workspace（`xuanwu`），下设 Project 与 Namespace 一一对应 |
| 用户与角色 | 与 §13.3 的 4 类人类角色绑定；**禁止共享账号**；审计账号独立 |
| Console 暴露 | 仅经 Ingress，**来源白名单限管理 VLAN**，不暴露互联网、不使用 NodePort |
| 关键纪律 | RBAC 与审计的**事实源是 Kubernetes**，KubeSphere 只提供视图；**关键能力不得依赖 KubeSphere 存续** |

> **重要说明（须写入文档与培训）：** KubeSphere 自带监控/日志已关闭，因此**其控制台的原生监控与日志面板不可用**；统一使用 Grafana。

---

## 7. CNI

| 项 | 规定 |
|---|---|
| CNI | **Calico** |
| 数据面模式 | **VXLAN**（不启用 BGP peering） |
| kube-proxy | **保留**（不启用 eBPF 替换） |
| IPAM | Calico IPAM；Pod CIDR `10.233.64.0/18` |
| MTU | 按底层网络显式配置（VXLAN 需扣除封装开销；精确值 Phase 0 确认） |
| 安装 | 官方 manifest，版本锁定写入版本矩阵 |
| 策略 | 标准 `networking.k8s.io/v1` NetworkPolicy；**不使用 Calico 专有扩展**（保持可移植与可读） |
| 验收 | 双向连通性测试（允许的通 / 禁止的不通）→ `V0.1 NetworkPolicy Validation Report` |
| 变更纪律 | NetworkPolicy 变更一律 **L2**；变更后**必须**跑连通性双向测试 |
| Cilium 评估 | **V0.3**（触发条件见 ADR-002 §5），V0.1 不得并行评估 |

---

## 8. Storage

### 8.1 StorageClass（仅 2 个）

| 名称 | Provisioner | 回收策略 | 用途 |
|---|---|---|---|
| `local-path` | local-path-provisioner | Delete | 默认；n8n、Prometheus、Loki |
| `local-static` | `kubernetes.io/no-provisioner` | **Retain** | **ClickHouse 专用** |

### 8.2 持久化对象分配

| 对象 | StorageClass | 容量 | 落点 | 备份策略 |
|---|---|---|---|---|
| ClickHouse 数据 | `local-static`（手工 PV + 节点亲和） | 300 GiB | `xw-wk-01` | daily → MinIO，30 天 |
| raw archive（窗口） | `local-path` | 100 GiB | `xw-wk-01` | daily → MinIO，30 天；**全量另存 MinIO** |
| n8n 数据 | `local-path` | 20 GiB | `xw-wk-01` | daily（Velero），14 天 |
| Prometheus | `local-path` | 100 GiB | `xw-wk-02` | **不备份**（可重建） |
| Loki | `local-path` | 50 GiB | `xw-wk-02` | **不备份**（可重建） |

### 8.3 ClickHouse 静态 PV 模板

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-clickhouse-01
  labels: { xuanwu.io/tier: A, xuanwu.io/backup: daily }
spec:
  capacity: { storage: 300Gi }
  accessModes: [ReadWriteOnce]
  persistentVolumeReclaimPolicy: Retain
  storageClassName: local-static
  local: { path: /data/clickhouse }
  nodeAffinity:
    required:
      nodeSelectorTerms:
        - matchExpressions:
            - { key: kubernetes.io/hostname, operator: In, values: [xw-wk-01] }
```

### 8.4 明确不做

**不部署** Ceph / Rook / Longhorn / GlusterFS / 分布式 MinIO / 分布式 ClickHouse / 多副本事务数据库。**不使用 CSI 快照**（local-path 无此能力，卷恢复走 Velero 文件系统备份）。

### 8.5 显式接受的存储代价

| 代价 | 数值 | 说明 |
|---|---|---|
| ClickHouse RPO | **24 小时** | 节点/数据盘故障时最多丢失 24h 数据；由备份 + 邮件重放对冲 |
| Pod 跨节点漂移 | **不支持** | 节点故障需"恢复"而非"迁移"；**Runbook 必须明确"优先恢复原节点"** |
| 自动扩容 | **不支持** | 容量需人工规划；磁盘水位纳入告警（A2/A3） |

---

## 9. Harbor

### 9.1 部署规定

| 项 | 规定 |
|---|---|
| 位置 | **集群外，`xw-mgmt-01`**（Docker Compose） |
| 域名 | `harbor.xw.internal`（仅管理 VLAN + 节点网段可达，**不暴露互联网**） |
| TLS | 内部 CA 签发；节点 containerd 信任该 CA；**不使用 insecure-registry** |
| 组件范围 | 启用 Core / Portal / Registry / Jobservice / Trivy / 内置 PostgreSQL / 内置 Redis；**不启用** Notary（签名）、跨实例复制、HA 多副本 |
| 项目 | `xuanwu-platform/*`（平台组件）、`da-soc/*`（业务镜像） |
| 访问控制 | 项目级 RBAC；节点/CI 使用项目级 robot account（**仅 pull 本项目**） |
| 存储 | 数据盘 ≥ 1 TB（`/data/harbor`） |
| 保留策略 | 每项目保留最近 10 个 tag；每周执行 GC |

### 9.2 镜像纪律（强制）

```text
[ ] 所有生产镜像必须来自 Harbor（唯一来源）
[ ] 所有镜像以 repo:tag@sha256:<digest> 形式记录在 configs/images/image-manifest.yaml（Git）
[ ] 禁止 latest
[ ] 禁止未经审核的未知镜像
[ ] 每个命名空间配置 imagePullSecret（Harbor robot account，最小权限）
[ ] imagePullPolicy: IfNotPresent
[ ] 每周执行一次"运行中镜像 digest vs Git 清单"漂移检查；不一致 → 产生 Task
```

### 9.3 Trivy 扫描

| 项 | 规定 |
|---|---|
| 扫描时机 | 推送时 + 每周全量重扫 |
| 门禁 | **不阻断** |
| 高危漏洞 | **必须产生 Task**（进入 AI Ops 闭环） |
| 修复 SLA | V0.2 建立 |
| 镜像签名 / SBOM | **V0.3** |

### 9.4 离线镜像导入 SOP（必须脚本化 + 入 Git）

```text
[构建机 / 可出网跳板机]
 1. 按 image-manifest.yaml 拉取镜像
 2. docker save → tar（分组：base / harbor / platform / da-soc）
 3. 生成 sha256sum + 记录 RepoDigest
        │ 内网传输
        ▼
[xw-mgmt-01]
 4. 校验 sha256sum
 5. docker load（Harbor 自举阶段允许 ctr -n k8s.io images import）
 6. docker tag → docker push harbor.xw.internal/<project>/<repo>:<tag>
 7. 更新 configs/images/image-manifest.yaml（digest / 来源 / 审核人 / 日期）
        ▼
[集群]
 8. 节点 containerd 从 Harbor 拉取；清单以 digest 引用
 9. 校验：kubectl get pod -o jsonpath='{.status.containerStatuses[*].imageID}' == manifest digest
```

**纪律：** ① **`ctr images import` 是唯一允许绕过 Harbor 的通道，仅限 Harbor 尚未就绪的自举阶段；Harbor 就绪后必须关闭该通道**（Runbook 写明关闭条件）；② 镜像 tar 存档保留在 `xw-bak-01`，作为 Harbor 全损时的兜底；③ 禁止 `latest`。

### 9.5 Harbor 自身备份

| 项 | 规定 |
|---|---|
| 备份对象 | 配置、数据库导出、registry 数据目录 |
| 频率 | 每周全量 + 每日增量 |
| 保留 | 4 周 |
| 目标 | MinIO（`xw-bak-01`）+ 镜像 tar 存档 |
| 恢复演练 | V0.1 至少一次（可选，优先级低于 4 项强制演练） |
| 关键属性 | **Harbor 的恢复不依赖 Kubernetes 集群** |

---

## 10. Observability

### 10.1 组件（单栈，不得新增第二套）

| 能力 | 组件 | 部署 |
|---|---|---|
| 指标 | **kube-prometheus-stack**（Prometheus Operator + Prometheus + Alertmanager + Grafana + node-exporter + kube-state-metrics） | `xw-obs` |
| 探测 | blackbox-exporter（HTTP/TCP 探测） | `xw-obs` |
| 批处理指标入口 | Pushgateway | `xw-obs` |
| 日志 | **Fluent Bit（DaemonSet）** | `xw-obs` / 全节点 |
| 日志存储 | **Loki**（单副本，文件系统存储） | `xw-obs` |
| 可视化 | **Grafana**（唯一入口） | `xw-obs` |

**保留策略：** Prometheus **15 天**；容器与节点日志 **30 天**；**Kubernetes Audit 90 天**。

**明确禁止：** Elasticsearch / OpenSearch / Kibana / Logstash / Fluentd / 第二套 Prometheus / KubeSphere 自带监控与日志 / Traces / SIEM。

### 10.2 监控内容（六层，全部必须）

| 层次 | 监控项 |
|---|---|
| Infrastructure | 节点 Up/Down、CPU 使用与饱和、内存可用、**磁盘使用率与 24h 预测耗尽**、inode、磁盘 I/O 延迟、网络吞吐与丢包、**NTP 偏移**、系统负载 |
| Kubernetes | 节点 Ready/NotReady、控制面组件健康、etcd leader/DB 大小/fsync/快照成功、apiserver 延迟与 5xx、**证书到期**、Pod Pending、调度失败、PVC/PV 状态、副本可用性 |
| Pod / Application | CrashLoopBackOff、重启次数、OOMKilled、ImagePullBackOff、readiness 失败、资源使用 vs limits、Quota 使用率、驱逐事件 |
| Platform | Harbor 可用与磁盘、Loki 写入、Prometheus 自身、**备份成功与时效**、Velero/etcd 快照状态、**Argo CD 同步状态与漂移** |
| **Business（DA-SOC）** | **数据新鲜度**、日报按日产出、`/archive` 成功率、`/render` 成功率、DingTalk 发送结果、ClickHouse 可用与行数、**IMAP 未读计数** |
| Security / Audit | 审计管道存活、异常 privileged/hostNetwork 检测、非 Harbor 来源镜像检测、Secret 读取异常、认证失败率、Harbor 高危漏洞计数 |

### 10.3 业务指标采集（关键设计）

```text
在 da-soc 命名空间部署「只读 metrics adapter」（CronJob，每 5 分钟）：
  1. 以 da_soc_ro（只读账号）查询 ClickHouse：
       SELECT max(data_date), count(*) ...
  2. 读取 n8n API：最近执行状态与最近成功时间
  3. 推送到 Pushgateway（xw-obs）
  4. Prometheus 抓取 → 告警 A10/A11
```

**四条边界纪律（必须写入 Runbook 与验收）：**

```text
[ ] adapter 只读（使用 da_soc_ro 账号，无写权限）
[ ] adapter 不参与业务链路，失败不影响日报
[ ] adapter 不产生任何业务数字，只上报观测元数据
[ ] adapter 输出不得用于生成日报图或替代 SQL 出数
```

### 10.4 告警规则（V0.1 共 12 条，不得随意增加）

| # | 告警 | 条件 | 等级 | Runbook |
|---|---|---|---|---|
| A1 | NodeNotReady | Ready=False > 5 分钟 | critical | `node-not-ready.md` |
| A2 | DiskWillFillIn24h | `predict_linear(...[6h], 86400) < 0` | critical | `disk-full.md` |
| A3 | DiskSpaceLow | 可用 < 15% | warning | `disk-full.md` |
| A4 | EtcdUnhealthy / SnapshotStale | 无 leader > 1 分钟；或快照 > 2 小时 | critical | `etcd-recovery.md` |
| A5 | ApiServerUnavailable | 不可用 > 1 分钟；或 5xx > 1% | critical | `k8s-api-failure.md` |
| A6 | PodCrashLoop | 重启 > 3 次/15 分钟 | warning | `pod-crashloop.md` |
| A7 | BackupFailed / BackupStale | 备份失败；或最新 > 26 小时 | critical | `backup-failure.md` |
| A8 | CertExpiringIn30d | 剩余 < 30 天 | warning | `certificate-expiry.md` |
| A9 | NtpOffsetHigh | 偏移 > 1 秒 | warning | `time-sync.md` |
| **A10** | **DASOCNoFreshData** | 业务时限前无当日数据；或最近 archive 成功 > 26 小时 | critical | `da-soc-failure.md` |
| **A11** | **DASOCChainDegraded** | `/archive` 或 `/render` 连续失败；或 n8n 执行失败 | critical | `da-soc-failure.md` |
| A12 | PlatformComponentDown | Harbor / Loki / Prometheus / Argo CD 不可用 | warning | `platform-component-down.md` |

**纪律（强制）：**

```text
[ ] 每条告警必须能回答：谁处理 / 怎么处理（Runbook 链接）/ 能否自动化
[ ] 没有 Runbook 的告警不允许上线
[ ] 告警必须携带：对象、影响、证据链接（Grafana/Loki）、Runbook、风险等级、是否需要审批
[ ] 业务日报群与运维告警群严格分离；平台告警绝不发往 DA-SOC 业务日报群
[ ] 同类告警 15 分钟内聚合；维护窗口使用 silence 并记录原因
```

### 10.5 日志采集范围

| 日志源 | 采集 | 标签 | 保留 |
|---|---|---|---|
| 节点 journald（ssh/kernel/systemd/chrony/auditd） | Fluent Bit systemd 输入 | `{job="journal", host=…}` | 30 天 |
| 容器 stdout/stderr | Fluent Bit tail + K8s 元数据 | `{namespace, pod, container}` | 30 天 |
| **Kubernetes Audit** | Fluent Bit tail 读控制面文件 | `{job="k8s-audit"}` | **90 天** |
| KubeSphere 操作审计 | 定期导出 + tail | `{job="ks-audit"}` | 90 天 |
| Harbor / MinIO（集群外） | 宿主机 Fluent Bit | `{job="harbor"}` / `{job="minio"}` | 30 天 |

### 10.6 脱敏纪律（强制）

**日志中不得出现：** 生产邮箱原文、DingTalk token 与群 ID、Secret 值、ClickHouse 密码、LLM API Key、业务数据明细。

**实现：** ① 应用侧不打印凭据；② Fluent Bit 过滤规则替换疑似 token/邮箱/密码模式；③ 定期抽样检查；④ **禁止把生产邮箱原文送入 AI 上下文**。

### 10.7 Agent 读取观测数据

| 信息 | 通道 | 权限 |
|---|---|---|
| 集群对象 | Kubernetes API（`-o json`） | `sa-agent-readonly`（**无 Secret 读权限**） |
| 指标 | Prometheus HTTP API | 只读 |
| 日志与审计 | Loki HTTP API（LogQL） | 只读 |
| 当前告警 | Alertmanager API | 只读 |
| 备份状态 | Velero CR + MinIO 对象列表 | 只读 |
| 镜像与漏洞 | Harbor REST API | robot account 只读 |
| 定义与知识 | Git（Runbook / ADR / 架构 / 资产） | 只读 |

**纪律：** Agent **不得**写观测数据；**不得**直接连生产邮箱/改 ClickHouse/发日报；**每次观测必须留证**（证据引用写入 Task）。

---

## 11. Backup

### 11.1 五条路径（全部必须实现）

| 路径 | 对象 | 工具 |
|---|---|---|
| 定义态 | 架构、策略、清单、RBAC、NetPol、Quota、工作流 JSON、SQL、资产、ADR、Task | **Git** |
| 集群状态 | etcd | `etcdctl snapshot save` |
| 资源 + 卷数据 | Kubernetes 对象与 PVC 数据 | **Velero + node-agent（文件系统备份）** |
| 业务数据 | ClickHouse | **ClickHouse 原生 `BACKUP` → S3** |
| 文件 | raw archive、n8n、Harbor、平台卷 | 文件级（restic/rsync）+ 镜像 tar |

### 11.2 备份对象与频率（完整清单）

| # | 对象 | 工具 | 频率 | 保留 | 目标 |
|---|---|---|---|---|---|
| B1 | etcd | snapshot | **每 30 分钟** | 本地 2 天 + 远端 30 天 | 本地 + MinIO |
| B2 | 定义态 | Git | 每次变更 | 永久 | Git |
| B3 | K8s 资源 + 卷数据 | Velero + node-agent | 每日（`da-soc`/`xw-ops`/`xw-obs`）+ 每周全量 | 30 日 / 12 周 | MinIO |
| B4 | ClickHouse 数据 | 原生 BACKUP | 每日（日报成功后） | 30 天 | MinIO |
| B5 | CH 表结构 | `SHOW CREATE TABLE` | 每日 | 永久 | Git + MinIO |
| B6 | raw archive（窗口） | 文件级 | 每日 | 30 天 | MinIO |
| B7 | raw archive（全量） | 归档（不可变对象） | 一次性 + 每日增量 | 长期 | MinIO |
| B8 | n8n 数据 | Velero | 每日 | 14 天 | MinIO |
| B9 | 工作流 JSON | Git | 每次变更 | 永久 | Git |
| B10 | SQL 源 | Git | 每次变更 | 永久 | Git |
| B11 | Harbor 配置 + 数据 | 导出 + 文件级 + 镜像 tar | 每周全量 + 每日增量 | 4 周 | MinIO + `xw-bak-01` |
| B12 | 平台配置 | Velero + Git | 每日 | 30 天 | MinIO |
| B13 | KubeSphere 配置 | 导出 + Git | 变更时 | 永久 | Git |
| B14 | MinIO 自身数据 | 文件级（异盘） | 每日 | 30 天 | 独立盘 |
| B15 | 凭据恢复机制 | 加密文件 + 静态加密 + 离线密钥 | 变更时 | 永久 | Git + 离线介质 |
| B16 | 审计日志 | Loki 保留策略 | — | 90 天 | Loki |
| B17 | 监控数据 | **不备份**（非恢复目标） | — | — | — |

### 11.3 Secret 恢复机制（三段式，强制）

| 层 | 机制 |
|---|---|
| ① 加密入 Git | Secret 以加密形态（Sealed Secret / SOPS 密文）进 Git |
| ② 静态加密 | kube-apiserver `EncryptionConfiguration`（aescbc / secretbox） |
| ③ 离线密钥托管 | 加密密钥**离线保管于受控位置，不进 Git**，登记在 `04-security/secret-inventory.md` |

**必须登记（进 Git）：** 凭据清单（名称、用途、位置、负责人、轮换周期、恢复方式）。
**不得进 Git：** 凭据明文值、加密密钥本体。

### 11.4 备份有效性保障（强制）

```text
[ ] 每次备份必须有明确的成功/失败状态
[ ] 最新备份年龄 > 26 小时 → 告警 A7
[ ] 备份目标使用率 > 80% → 告警
[ ] 每月抽查一次备份内容可读性
[ ] 每类备份必须有明确责任人（登记在 12-assets/ 与 Runbook）
[ ] 备份状态进入 Prometheus，纳入 A7
[ ] 失败自动重试最多 2 次；仍失败 → 告警 + 产生 Task（不静默重试）
```

### 11.5 RPO / RTO（必须在演练中验证）

| 场景 | RPO | RTO |
|---|---|---|
| 单 Pod 误删 | 0 | ≤ 15 分钟 |
| 配置误改（声明式） | 0 | ≤ 15 分钟 |
| `da-soc` 命名空间误删 | ≤ 24h（数据）/ 0（定义） | ≤ 2 小时 |
| 单个 Worker 节点故障 | 0（优先恢复原节点） | ≤ 4 小时 |
| ClickHouse 数据盘损坏 | ≤ 24 小时 | ≤ 4 小时 |
| 控制面全失（etcd 损坏） | ≤ 30 分钟 | ≤ 4 小时 |
| 整集群重建 | ≤ 24 小时 | ≤ 8 小时 |
| Harbor 故障 | 0 | ≤ 4 小时 |
| MinIO 故障 | ≤ 24 小时 | ≤ 4 小时 |

### 11.6 强制恢复演练（4 项，全部必须真实完成）

| # | 演练 | 成功判据 |
|---|---|---|
| **R1** | Velero 命名空间恢复 | 资源与 PVC 数据完整；Pod Running；健康检查通过 |
| **R2** | etcd 快照恢复（**优先隔离沙箱**；生产需维护窗口 + L2） | apiserver 起来；核心资源与备份时刻一致 |
| **R3** | ClickHouse 数据恢复（**业务关键**） | 三窗口聚合**完全一致**；空值语义正确（无数据不为 0） |
| **R4** | DA-SOC 端到端恢复（**业务关键**） | 测试群收到图，且**数字为真实值**（非 0、非空、与恢复前一致） |

**纪律：** ① R3/R4 必须与业务方共同完成并签字；② 每次演练必须产出记录（范围、步骤、**实测 RTO**、问题、改进项、证据链接）；③ 演练失败 → 产生 Task → 修复 → 重演（计入验收）；④ **演练不得影响生产日报**（优先沙箱；生产演练需维护窗口且以"日报已成功产出"为前提）。

### 11.7 恢复顺序（标准骨架）

```text
0. 冻结变更（声明维护窗口；停止自动化变更与 Agent 执行）
1. 恢复/重建控制面（etcd 快照 或 Git 重建）
2. 恢复 CNI / 核心插件就绪
3. 恢复命名空间与资源定义（Git apply / Velero restore）
4. 恢复 Secret / 凭据（加密文件 + 离线密钥）
5. 确认镜像供给（Harbor 存活或恢复 Harbor）
6. 恢复 ClickHouse 表结构与数据（B5 → B4/B7）
7. 恢复 raw archive 与 n8n 数据（B6/B7/B8）
8. 恢复 render/archive 与 n8n 工作负载 → 验证健康
9. 业务验证：SQL 结果 → archive 失败门禁 → 日报图 → 测试钉钉群
   （★ 数字必须与基线一致；不为 0、不为空）
10. 记录证据 → 解除冻结 → 形成 Incident/Drill Record → 更新 Runbook
```

---

## 12. Security

### 12.1 身份与认证

| 类别 | 实现 |
|---|---|
| 人类管理员 | KubeSphere 用户 + 强密码；`platform-admin` **仅 break-glass（1–2 人，使用需双人 + 记录）** |
| 日常运维 | `platform-operator`（平台命名空间 admin + 集群只读；**不可读 `da-soc` Secret**） |
| 业务用户 | `da-soc-owner`（仅 `da-soc`；不可改 NetPol/Quota/RBAC） |
| 审计用户 | `auditor`（集群只读 + 审计日志读取；独立账号，不用于日常操作） |
| Agent | **3 个独立 ServiceAccount**（见 §16.2） |
| 自动化 | `sa-n8n-platform`、`sa-da-soc`、`sa-backup`（各司其职） |
| **禁止** | 共享账号；任何业务/Agent 主体持有 `cluster-admin`；`ClusterRoleBinding` 到 `system:authenticated` |

**认证强化：** `--anonymous-auth=false`；`default` SA `automountServiceAccountToken: false`；KubeSphere 登录失败锁定；**（建议）** 接入企业 LDAP/AD。

### 12.2 授权

```text
[ ] 以 Role/RoleBinding（命名空间级）为主；ClusterRole 仅在必要时使用且必须在 02-governance/ 登记
[ ] 高权限账号只经管理 VLAN + Ingress 登录（有审计）
[ ] 节点上的 admin kubeconfig 存于管理节点受控路径（权限 600），使用即记录
[ ] 季度权限评审（V0.1）：导出 RBAC 清单 + Agent 辅助生成报告
[ ] 越权测试（10 项）：RBAC 越权 / NetworkPolicy / privileged / hostNetwork /
    NodePort / 非授权 Registry / Secret 权限 / API 暴露 / etcd 暴露 / Audit 有效性
```

### 12.3 网络安全

- Calico NetworkPolicy（§4.4）；管理面 Ingress 来源白名单限管理 VLAN。
- 主机防火墙仅放行管理网段 SSH 与必要端口。
- 出向控制：边界防火墙 IP + 端口白名单。
- 禁止 NodePort；禁止 hostNetwork（白名单例外见 §21）。

### 12.4 容器与 Pod 安全

| 控制 | 实现 |
|---|---|
| Pod Security Admission | `da-soc`/`xw-ops` = `restricted`；`xw-obs`/`xw-system`/`kubesphere-*`/`kube-system` = `baseline` + 例外登记；`default` = `restricted` |
| 强制禁止 | privileged、hostNetwork、hostPID、hostIPC、不必要 hostPath、非 root、缺失 requests/limits、缺失探针 |
| 例外登记 | 每条必须有：对象、原因、风险、补偿、**消除期限**、ADR 编号 |
| **纪律** | **绝不因为一个例外而把整个 Namespace 降级** |
| V0.1 不引入 | **Kyverno / OPA / Gatekeeper**（由 PSS + ResourceQuota + digest 固定覆盖） |

### 12.5 Secret 与凭据

| 项 | 规定 |
|---|---|
| 静态加密 | kube-apiserver `EncryptionConfiguration`（**必须**） |
| 入 Git | **仅加密形态**（Sealed Secret / SOPS） |
| 离线密钥 | 受控离线保管，不进 Git；年度可读性验证 |
| 凭据清单 | `04-security/secret-inventory.md` 登记（名称/用途/位置/负责人/轮换周期/恢复方式） |
| 群 ID | **只能来自 Secret**；不得出现在工作流 JSON、ConfigMap、日志、Git 明文 |
| 轮换 | 邮箱/钉钉每季度；LLM Key 90 天；Harbor robot 180 天；轮换走 Runbook（L1 执行 + 验证） |
| **禁止** | 用 ConfigMap 存凭据；环境变量明文注入到可打印位置；凭据进入 Agent Prompt |

### 12.6 审计

| 源 | 内容 | 落地 | 保留 |
|---|---|---|---|
| **Kubernetes Audit** | 所有 apiserver 请求（who/what/result） | 控制面文件 → Fluent Bit → Loki（`job="k8s-audit"`） | **90 天** |
| KubeSphere 操作审计 | 控制台登录、用户/角色/Workspace 变更 | 定期导出 + tail → Loki | 90 天 |
| **Agent 操作审计** | Job 镜像 digest、SA、证据引用、计划、审批人、执行、验证 | Task（Git）+ 容器日志（Loki）+ K8s Audit | Git 永久 |
| 变更审计 | 所有集群变更 = Git 提交（Argo CD）或 kubectl 审计事件 | Git history（含 PR）+ K8s Audit | Git 永久 |

**审计告警（必做 4 条）：** 高危 verb 出现在 `kube-system`/`xw-system`；Secret 读取异常频次；非白名单主体尝试提权；Agent SA `Forbidden` 突增。

### 12.7 安全能力分期

| 能力 | V0.1 必须 | V0.1 建议 | V0.2 | V0.3 |
|---|---|---|---|---|
| API/etcd 不暴露 | ✅ | | | |
| RBAC 最小权限 + 越权测试 | ✅ | | | |
| NetworkPolicy 默认拒绝 | ✅ | | | |
| PSS + 例外登记 | ✅ | | | |
| Secret 静态加密 + 加密入 Git | ✅ | | | |
| K8s Audit 集中化 90 天 | ✅ | | | |
| Harbor + Trivy + digest 固定 | ✅ | | | |
| 主机安全基线 | ✅ | | | |
| 凭据轮换 | | ✅ | | |
| 备份加密 | | ✅ | ✅ | |
| 漏洞修复 SLA / 门禁 | | | ✅ | ✅ |
| Runtime Security / EDR | | | | ✅ |
| SIEM | | | | ✅ |
| 镜像签名 / SBOM | | | | ✅ |

---

## 13. RBAC

### 13.1 权限矩阵

| 主体 | 类型 | `kube-system` | `xw-system` | `xw-obs` | `xw-ops` | `da-soc` | 集群级 |
|---|---|---|---|---|---|---|---|
| `platform-admin` | 人 | — | — | — | — | — | `cluster-admin`（仅 break-glass） |
| `platform-operator` | 人 | RO | Admin | Admin | Admin | **RO（不可读 Secret）** | RO |
| `da-soc-owner` | 人 | — | — | — | — | **Admin**（不可改 NetPol/Quota/RBAC） | — |
| `auditor` | 人 | RO | RO | RO | RO | **RO** | RO + 审计日志读 |
| `sa-agent-readonly` | SA | RO | RO | RO | RO | RO（**无 Secret**） | RO（view 等价） |
| `sa-agent-executor` | SA | ❌ | ❌ | **L1 白名单** | **L1 白名单** | **L1 白名单** | ❌ |
| `sa-agent-auditor` | SA | RO | RO | RO | RO + Task 写 | RO | RO |
| `sa-n8n-platform` | SA | ❌ | ❌ | ❌ | Job 创建/查询 + Task 写 | ❌ | ❌ |
| `sa-da-soc` | SA | ❌ | ❌ | ❌ | ❌ | 自身资源 | ❌ |
| `sa-backup` | SA | 备份相关 | 备份相关 | ❌ | 备份 Job | PVC/资源读 | ❌ |

### 13.2 强制约束（全部必须验证）

```text
[ ] default SA 禁止自动挂载 token
[ ] 禁止业务/Agent 主体持有 cluster-admin
[ ] 禁止 ClusterRoleBinding 到 system:authenticated / system:unauthenticated
[ ] 禁止同一非管理员主体同时具备"读 Secret + 创建 Pod"（提权组合）
[ ] 所有集群级绑定必须在 02-governance/ 登记
[ ] 权限变更走 Git（ADR + Change 记录）
[ ] 越权测试全部通过
```

---

## 14. NetworkPolicy

**权威定义见 §4.4。** 本节的强制要求：

```text
[ ] 每个业务/平台命名空间都有 default-deny-all（Ingress + Egress）
[ ] 所有放行逐条显式；每条必须注明：用途 / 源 / 目的 / 端口 / 责任 / 验证方法
[ ] 禁止 0.0.0.0/0 出向放行
[ ] 禁止 namespaceSelector: {} 通配
[ ] 变更一律 L2；变更后必须执行连通性双向测试
[ ] 迁移期第 8 条放行在使用后立即删除（并验证已删除）
[ ] 验收产出：V0.1 NetworkPolicy Validation Report（含允许通/禁止不通双向证据）
```

---

## 15. DA-SOC Hosting

### 15.1 部署清单

| 组件 | 形态 | Service | 镜像 | 存储 |
|---|---|---|---|---|
| n8n | Deployment 1 副本 | `n8n:5678` | `ghcr.io/deluxebear/n8n:chs@<digest>`（**不升级版本**） | PVC 20 GiB |
| da-soc-render | Deployment **2 副本** | `da-soc-render:8091` | `da-soc-render:0.1@<digest>` | 无（无状态） |
| ClickHouse | Deployment 1 副本 | `clickhouse:8123,9000` | `clickhouse/clickhouse-server@<digest>` | `local-static` 300 GiB |
| raw archive | PVC | — | — | `local-path` 100 GiB + MinIO 全量 |
| metrics adapter | CronJob（5 分钟） | — | 平台侧轻量镜像 | — |
| clickhouse-backup | CronJob（每日） | — | 平台侧脚本镜像 | — |

**Namespace：`da-soc`**（PSS `restricted`；ResourceQuota 见 §13/§21；NetworkPolicy 见 §4.4）。

### 15.2 与现状的 5 个必要改造（必须记录为架构变更）

| # | 现状 | 集群形态 | 验证 |
|---|---|---|---|
| C1 | ClickHouse **host network**，`127.0.0.1:8123` | Pod 网络 + ClusterIP | n8n/render 通过 Service 名访问正常；节点网段无法直连 8123；无 NodePort |
| C2 | n8n **host network** | Pod + 无入向入口（仅出向） | DingTalk 交互采用**出向轮询**（集群无公网入向回调） |
| C3 | render/archive 同镜像、本机 8091 | 同镜像 2 副本 + Service | **保持同版本约束**（同一 Deployment，禁止分别升级） |
| C4 | 数据在本机磁盘 | `local-static` PV + Retain | 一次性回补 + 数字一致性报告 |
| C5 | 时区/时间来自宿主 | 容器 `TZ` + CH `timezone` + 节点 chrony | **三窗口边界日与 ECS 一致** |

> **迁移前唯一必须修改的业务代码：** 把硬编码的 `127.0.0.1:8123` / `127.0.0.1:8091` 改为**服务名或环境变量**。该改造在 P3 阶段（集群内、不接生产数据）完成验证，从而把唯一的代码改动风险隔离在非生产环境。**实施期先核查现有实现是否已使用配置项（见未决问题 Q7）。**

### 15.3 ClickHouse 双账号（强制）

| 账号 | 权限 | 持有者 |
|---|---|---|
| `da_soc_writer` | 业务库表读写 | 仅 `da-soc` 内的 render / n8n 工作负载 |
| `da_soc_ro` | **只读** | Agent、metrics adapter、人工排障 |

**验证：** 用 `da_soc_ro` 尝试写入**必须失败**（纳入安全验证）。

### 15.4 八条硬业务规则的强制实现（逐条可验证）

| # | 规则 | 强制手段 | 验证方式 |
|---|---|---|---|
| 1 | 数字只能来自 ClickHouse SQL | Agent/平台无补数改数路径；SQL 仅来自 Git | Agent SA 对 CH 写请求计数 = 0 |
| 2 | 图片只能由 render 生成 | 出图链路唯一 | 工作流定义审查 + 产物可追溯 |
| 3 | **LLM 不参与出数出图** | ① RBAC 无写权限；② CH 双账号只读；③ Task 证据强制 | DB 权限导出 + Agent 权限清单 + 审计 |
| 4 | `null` / 暂无数据不得填 0 | 平台不引入任何自动补值 | 空窗口样例专项验证 |
| 5 | `/archive` 失败 → 不入库/不出图/不发送 | 平台不提供绕过通路；监控显式检测中止语义 | 人为制造失败，确认全链路中止 |
| 6 | 不得 Mark as Read / 删改生产邮件 | n8n 只读配置；无写协议出向；未读计数监控 | 拉取前后未读计数一致 |
| 7 | 不得覆盖「监测bjfz邮箱广电报送信息」 | 集群内无该邮箱凭据、无写通路 | 审计：对该邮箱的任何写尝试 |
| 8 | 生产群与测试群严格隔离 | 群 ID 只来自 Secret；迁移期只配测试群 | 生产群在迁移期收不到平台消息 |

### 15.5 SQL 与工作流纪律（路径 A）

```text
[ ] SQL 源在 v0.1/sql/（或等价位置）并在 Git 中版本化
[ ] 工作流 JSON 由 tools/build_workflow.py 在构建机生成后进 Git
[ ] n8n 以只读方式加载工作流（禁止 UI 修改）
[ ] 禁止使用社区 ClickHouse 节点
[ ] 每日比对「运行中工作流 vs Git 产物」；不一致 → 产生 Task
[ ] tools/build_workflow.py 与 6 个 Python 脚本不进集群（归档在 Git 供对照）
```

### 15.6 迁移 10 阶段

| Phase | 内容 | 门禁 |
|---|---|---|
| **P1** | 建立 K8s 环境（集群、网络、存储、Harbor、可观测、备份、RBAC/Quota/NetPol） | 平台验收通过；**不触碰 ECS** |
| **P2** | 镜像进入 Harbor（3 个镜像，记录 digest） | **digest 与 ECS 运行镜像逐一比对一致** |
| **P3** | 部署三组件（ClickHouse 空库 + 一致 schema；工作流只读注入） | Pod Ready；**不接生产数据**；服务名改造验证通过 |
| **P4** | 导入历史数据（只读导出 → 导入；raw 归档） | **Data Parity Report：三窗口数字完全一致** |
| **P5** | 导入 workflow（Git 产物 → ConfigMap → 只读加载） | 运行中工作流 = Git 产物 |
| **P6** | 数据验证（三窗口独立出数比对） | 全部一致；空值语义正确 |
| **P7** | 图片验证 | 出图一致 |
| **P8** | DingTalk 验证（**只发测试群**） | 测试群收到；生产群零污染 |
| **P9** | 切换生产入口（Cutover Gate → 单写者切换） | 首个日报由集群产出且数字正确 |
| **P10** | 保留 ECS 回退能力（冷备 + Runbook + 演练） | **回退演练 ≤30 分钟完成** |

### 15.7 双跑规则（双读单写）

| 项 | 规则 |
|---|---|
| 双跑定义 | **双读单写**（不是双写） |
| 邮箱并发读 | ✅ 允许（只读幂等）；两侧均**不设 `\Seen`**、不写/删/移动 |
| 数据双写 | ❌ **禁止**；ClickHouse 单侧权威 |
| 防重复发送 | **物理隔离**：迁移期平台侧只配**测试群**；切换后 ECS 调度停用 |
| 防重复标记已读 | 两侧只读 + **未读计数**作为一等监控指标（下降即告警并中止双跑） |
| 防数据污染 | 两侧独立实例；禁止跨实例写入 |
| 比对方式 | **逐项数字比对报告**（不接受"看起来一样"） |
| 时长 | **连续 3 个自然日**三窗口全绿（任一不一致则计时归零） |

### 15.8 Cutover Gate（12 项，全部满足才允许切换）

| # | Gate | 判定标准 |
|---|---|---|
| G1 | 数据一致 | 三窗口与 ECS **逐项完全一致** |
| G2 | 图片一致 | 内容/字段/时间标注一致 |
| G3 | 业务流程一致 | 成功率、顺序、中止语义一致（连续 3 日） |
| G4 | 邮件读取行为一致 | 未读计数不变；无 `\Seen`；无写/删除 |
| G5 | DingTalk 行为一致 | 测试群收到；生产群零污染；群 ID 来自 Secret |
| G6 | 备份成功 | etcd + Velero + CH + 配置 + raw + n8n 全部成功 |
| G7 | 恢复测试成功 | ClickHouse 恢复演练成功且数字一致 |
| G8 | 监控正常 | 含数据新鲜度在内的业务指标可见 |
| G9 | 日志正常 | Pod 日志与审计日志可检索 |
| G10 | 告警正常 | 人为触发 ≥3 类告警均送达 |
| G11 | **回退路径验证成功** | 回退演练**已真实执行并成功** |
| G12 | Runbook 就绪 | 步骤/责任人/时限/判据明确 |

### 15.9 切换执行（单写者切换）

```text
T-1d  冻结变更
T0    确认最后一次双跑三窗口全绿 + 12 项 Gate 通过
T1    停用 ECS 侧 DA-SOC 调度                 ← 保证单写者
T2    集群 n8n 发送目标：测试群 → 生产群        ← 仅修改 Secret 引用，不改工作流
T3    启用集群侧调度（对齐原生产时间）
T4    观察首个完整日报：收到图 + 数字与基线一致
T5    记录切换完成；ECS 转冷备
```

> **关键设计：** 切换**不修改任何业务逻辑**，只做两件事（转移调度权 + 切换群 ID），因此是**可逆的最小操作**。

### 15.10 回退（≤30 分钟）

**9 项触发条件（任一出现即回退）：**

```text
R1 三窗口任一数字不一致且原因不可解释
R2 /archive 行为异常（成功未入库 / 失败仍继续 / 部分写入）
R3 /render 行为异常（出图失败 / 内容错误 / 字段缺失）
R4 ClickHouse 数据异常（重复 / 缺失 / 乱码 / 时区偏移）
R5 邮件读取异常（未读计数下降 / 出现写删迹象）→ 立即回退并人工核查邮箱
R6 DingTalk 异常（发错群 / 重复发送 / 静默失败）
R7 备份异常（连续 2 次失败或 >26 小时）
R8 出现不可解释的数据缺失（当日无数据但源邮件存在）
R9 当日日报在业务时限内未产出
```

**执行：**

```text
1. 停用集群侧调度（恢复单写者）
2. 集群 n8n 发送目标改回测试群（防止误发生产群）
3. 启用 ECS 侧原调度（镜像/数据/配置/凭据均保持可用）
4. 触发或等待 ECS 产出当日日报，确认数字正确
5. 记录 Incident（触发条件、时间线、影响、根因、改进项）
6. 集群侧修复 → 重走 P6–P8 → 重新过 Gate
```

**可行性依据：** ECS 在切换期间**只被停用、未被修改**。

### 15.11 ECS 退役判据（V0.1 结束前评审）

```text
[ ] 切换后稳定运行 ≥ 30 天，无 R1–R9 类事件
[ ] 至少一次真实 ClickHouse 恢复演练 + 一次集群级恢复演练
[ ] 集群侧 raw archive 与 ClickHouse 数据经比对确认完整
[ ] 业务方书面确认日报连续正确
[ ] 退役方案经批准（含历史数据最终归置）
```

---

## 16. AI Ops

### 16.1 架构

```text
触发层：平台 n8n（xw-ops）—— Schedule / Alertmanager webhook / DingTalk
   ↓ 生成 Task（Git）+ 创建 Agent Job
状态层：Git 中的 Task（tasks/YYYY/MM/TASK-<id>.yaml + .md）
   ↓
执行层：Agent Job（一次性，SA 绑定，无 SSH）
   Observe → Analyze → Plan →（Approval）→ Execute（白名单）→ Verify → 写回 Task
   ↓
证据层：Git（Task/定义）+ Loki（审计/日志）+ K8s Audit
```

### 16.2 三个 ServiceAccount

| SA | 范围 | 允许 | 明确禁止 |
|---|---|---|---|
| `sa-agent-readonly` | 集群级 view + Pod 日志 | get/list/watch 全资源；pods/log | **Secret 读**；任何写；`pods/exec`；`pods/portforward` |
| `sa-agent-executor` | 仅 `xw-obs` / `xw-ops` / `da-soc` | 9 条白名单动词（§16.4） | ClusterRole 绑定；`kube-system`/`xw-system` 写；Secret 读写；RBAC/NetPol/Quota/Node/PV 操作；`exec` |
| `sa-agent-auditor` | 集群级 RO + Task 写 | 读 Task/审计/集群对象；写 Task | 其他任何写 |

### 16.3 Agent Job 强制规格

```yaml
spec:
  backoffLimit: 0                 # 不自动重试
  activeDeadlineSeconds: 900      # 强制超时
  ttlSecondsAfterFinished: 86400  # 保留 24h 供审计
  template:
    spec:
      serviceAccountName: sa-agent-readonly    # 或 executor / auditor
      restartPolicy: Never
      securityContext:
        runAsNonRoot: true
        runAsUser: 10001
        seccompProfile: { type: RuntimeDefault }
      containers:
        - name: agent
          image: harbor.xw.internal/xuanwu-platform/agent-runtime@sha256:<digest>
          resources:
            requests: { cpu: 200m, memory: 512Mi }
            limits:   { cpu: "1",  memory: 2Gi }
          securityContext:
            readOnlyRootFilesystem: true
            allowPrivilegeEscalation: false
            capabilities: { drop: ["ALL"] }
```

**纪律：** ① `backoffLimit: 0`；② 必须设置超时；③ 只读根文件系统；④ **不挂载任何业务 Secret**；⑤ 不使用 hostPath / hostNetwork / privileged；⑥ 镜像以 digest 固定。

### 16.4 执行白名单（V0.1 完整清单，不得私自扩展）

| # | 动作 | 目标 | 等级 |
|---|---|---|---|
| E1 | 删除 CrashLoop/Evicted 的 Pod | `xw-obs` / `xw-ops` | L1 |
| E2 | 重启 Deployment（`rollout restart`） | `xw-obs` / `xw-ops` | L1 |
| E3 | 扩缩容 Deployment（Quota 内） | `xw-obs` / `xw-ops` | L1 |
| E4 | 删除已完成的 Job（按标签） | `xw-ops` / `xw-obs` | L1 |
| E5 | 触发 Harbor GC / Trivy 重扫 | Harbor API | L1 |
| E6 | 重新触发失败的备份 Job | `xw-ops` | L1 |
| E7 | 重启失败的前置检查 Job | `xw-ops` | L1 |
| **E8** | **重启 `da-soc/da-soc-render`（无状态）** | `da-soc` | **L1，仅在非日报发送窗口内；窗口内强制 L2** |
| E9 | 更新 Task 记录 | Git | L0/L1 |

**绝对不在白名单（任何情况不允许自动执行）：** 修改 RBAC / NetworkPolicy / Quota / CNI / StorageClass / PV / Node；删除 PVC 或命名空间；重启 ClickHouse；删除或修改 DA-SOC 工作流与 SQL；修改 DingTalk 群配置；读写生产邮箱；`kubectl exec`；关闭任何安全控制。

**每个动作必须：** ① 先 dry-run；② 幂等或显式标记不可重复；③ 有前置条件校验；④ 有验证方法；⑤ 有回滚路径。

### 16.5 执行通道（仅 3 条）

| 通道 | 用途 | 审计 |
|---|---|---|
| Kubernetes API | 白名单变更 | K8s Audit + 容器日志 |
| HTTP API | 只读观测（Prom/Loki/Alertmanager/Harbor/Velero/Argo CD）+ 触发 n8n webhook | 容器日志 + 服务日志 |
| Git 提交 | Task 记录、审计记录、ADR 提案 | Git history |

**明确不用：** SSH 到节点、Docker socket、任意 shell、MCP（V0.2 评估）。

### 16.6 验证方法（强制要求）

| 动作 | 验证方法 | 判据 |
|---|---|---|
| E1 | 查新 Pod 状态 + 重启计数 | Ready 持续 5 分钟；重启不再增长 |
| E2 | `rollout status` + 副本就绪 + 告警清除 | 全部 Ready；告警消除 |
| E3 | 实际副本数 + 无 Pending + 无资源紧张 | 双条件 |
| E4 | 按标签列出结果 | 目标 Job 不存在；非目标未被误删 |
| E5 | Harbor API 任务状态 | 任务成功 |
| E6 | 备份状态 + 时效指标 | 成功且年龄 < 26h |
| E8 | Pod Ready + 探针 + 业务链路未被中断 | 日报链路正常 |
| 通用【业务】 | **数字比对**（不是"发送成功"） | 与基线一致 |

> **硬规则：无法写出验证方法的动作，一律不得执行。** 验证失败 → 立即回滚。

### 16.7 回滚

| 变更类型 | 回滚机制 |
|---|---|
| 声明式配置 | Argo CD 回退到上一 revision / `git revert` + re-sync |
| Pod 类（E1/E2/E8） | 无需回滚（控制器自愈 / 无状态重建）；必要时恢复上一镜像 digest |
| 扩缩容（E3） | 恢复变更前副本数（记录在 Task） |
| Job 清理（E4） | 从 Git 定义重建 |
| 备份重触发（E6） | 幂等，无需回滚 |
| 数据类 | **不在白名单**（L2，默认由人执行） |
| 不可回滚 | **禁止自动执行**；必须 L2 且在计划中显式声明 |

**回滚本身也必须创建/更新 Task 并记录**（理由、命令、结果）。

### 16.8 审计（每次 Job 必须产出）

| 产物 | 内容 |
|---|---|
| Task（Git） | 触发源、任务 ID、风险等级、**证据引用（查询 + 时间范围）**、结论、计划、前置条件、dry-run 结果、审批记录、执行动作、结果、验证、回滚方式、Job 名与镜像 digest |
| 容器日志 | 完整执行日志（脱敏） |
| K8s Audit | 该 SA 的全部集群请求 |
| **交叉要求** | **仅凭 Git + Loki，事后任何人能完整重建"Agent 为什么这么做"** |

### 16.9 必须验证的 AI Ops 场景（2 个真实场景）

**场景 1（L1 自动闭环）：`Pod CrashLoopBackOff`**

```text
触发：A6（PodCrashLoop）→ 平台 n8n → Task（L1）
1 Observe：读 Pod 状态 / 事件 / 容器日志 / 最近变更
2 Analyze：定位根因
3 Plan：计划 + 前置条件 + dry-run + 回滚说明
4 Risk：按规则推导 = L1
5 Execute：E1 或 E2
6 Verify：Ready 持续 5 分钟 + 重启计数不再增长 + 告警消除
7 Audit：Task 写回 + Loki + K8s Audit
8 Close：通知 DingTalk 运维群
```

**场景 2（L2 人工审批闭环 + 失败回滚）：`Node DiskPressure` 或 `BackupStale`**

```text
触发：A2 或 A7 → Task（L2）
1–3 同场景 1
4 Risk：L2（涉及节点磁盘处置或数据类操作）
5 Approval：DingTalk 审批卡片（影响/证据/计划/回滚/预计时长）
            → 人工批准（签名回调或轮询）→ 写入 approvals[]
6 Execute：批准后执行
7 Verify：磁盘可用回升 ≥ 目标 / 备份 phase=Completed 且年龄 < 26h
8 失败路径：验证不通过 → 执行回滚 → 置 RolledBack → 生成 Incident
9 Audit：完整链路（含审批人、消息 ID、回滚记录）
```

### 16.10 AI Ops 验收项（V1–V8，全部必须通过）

```text
V1 L0 巡检可用          定时 Task 自动完成（Node/Pod/Disk/Cert/Backup/资源/日报摘要）
V2 L1 自动执行可用      场景 1 由真实告警触发并自动完成
V3 L2 人工审批可用      场景 2 完成一次真实审批并执行
V4 验证独立可用         验证方法与执行方法的信号源不同
V5 回滚可用             至少一次真实的验证失败 → 回滚 → 记录
V6 审计可复盘           随机抽取一次操作，仅用 Git + Loki 重建决策链
V7 配置漂移可发现       人为修改运行态 → 24 小时内产生 Task
V8 数据新鲜度可发现     人为阻断数据 → A10 触发并产生 Task
```

### 16.11 n8n 与 Agent 职责边界

| n8n（手） | Agent（脑） |
|---|---|
| 定时、Webhook、邮箱、钉钉、API 集成；触发 Task 与 Job；通知与审批收发 | 理解问题、分析、判断、规划、工具调用、执行、验证、复盘 |
| **不做判断、不做根因分析** | **不改触发规则、不自我扩权、不改定义态** |

**边界铁律：Agent 可以改变平台的运行态，但不能单方面改变平台的定义态。**

---

## 17. Task Model

### 17.1 载体

```text
tasks/YYYY/MM/TASK-<YYYYMMDD>-<NNNN>.yaml   # 机器可读：状态机与字段
tasks/YYYY/MM/TASK-<YYYYMMDD>-<NNNN>.md     # 人可读：分析与证据记录
tasks/TEMPLATE.yaml / TEMPLATE.md
tasks/README.md                              # Schema 与状态机说明
```

**Git 是 Task 的唯一事实源。** V0.1 **不引入** Task CRD、独立 Task Center 服务、Task 数据库、Task Web UI、Jira/ITSM。

### 17.2 状态机

```text
New → Analyzing → Planned → (AwaitingApproval) → Executing → Verifying → Closed
异常终态：Failed / RolledBack / Expired / TimedOut
```

| 状态 | 含义 | 下一状态 |
|---|---|---|
| New | 已创建 | Analyzing / Failed |
| Analyzing | 读取证据与上下文 | Planned / Failed |
| Planned | 计划与风险等级已确定 | AwaitingApproval（L2）/ Executing（L0/L1） |
| AwaitingApproval | 等待人工审批 | Executing / Expired / Failed |
| Executing | 正在执行 | Verifying / Failed / RolledBack |
| Verifying | 正在验证 | Closed / RolledBack / Failed |
| Closed / Failed / RolledBack / Expired / TimedOut | 终态 | — |

### 17.3 强制字段

```yaml
task_id / created_at / created_by / schema_version
source / trigger_ref
asset: { kind, name, namespace, node }
severity / impact / affected_business
risk_level / risk_rule / requires_approval      # ★ 由规则推导，不由 Agent 判定
evidence[]: { source, query, time_range, result_summary }   # ★ 必须可复现
analysis / plan[] / preconditions / dry_run_result / rollback_plan
approvals[]: { by, role, at, channel, message_id, decision, comment }
expires_at
execution: { agent_job, agent_image_digest, service_account, started_at, finished_at, result, commands, artifacts }
verification: { method, performed_at, result, detail }      # ★ 必须独立方法
status / status_history[] / lessons / related[]
```

### 17.4 写入纪律

```text
[ ] 单一写入者：仅平台 n8n 与 Agent Job 写入
[ ] 按 Task ID 串行化；写入前 git pull --rebase
[ ] rebase 失败重试 ≤3 次；仍失败 → 告警并转人工
[ ] 人的意见通过 Issue/PR 评论，不直接改文件
[ ] 每日归档 tag
[ ] 风险等级由规则推导（§19）；Agent 只能提出规则修改提案（ADR），不得自行降级
[ ] 证据必须可复现；无证据的 Task 视为不合格
```

---

## 18. Human / AI Boundary

### 18.1 风险等级定义

| 等级 | 定义 | 执行者 | 通知 |
|---|---|---|---|
| **L0** | 只读 / 检查 / 报告；不改变任何状态 | Agent 自动 | 无（记入 Task） |
| **L1** | 白名单内的可逆操作；有明确验证与回滚 | Agent 自动（dry-run + 验证 + 留痕） | 事后通知 |
| **L2** | 高风险 / 不可逆 / 涉及 A 级边界 | **人审批**；**V0.1 默认由人执行**，Agent 负责准备、建议、验证、记录 | 审批请求 |
| **L3** | 人类专属（不建议交给 Agent）：架构决策、业务优先级、安全红线例外、重大 Incident 定级与对外沟通 | 人 | — |

### 18.2 RACI

| 活动 | 人 | Agent | n8n |
|---|---|---|---|
| 定义目标与策略 | **R/A** | C | — |
| 平台观测 | I | **R** | C |
| 异常分析与计划生成 | C | **R** | — |
| L0 执行 | I | **R** | C |
| L1 执行 | I（事后知悉） | **R** | C |
| **L2 审批** | **A/R** | C | C |
| L2 执行（V0.1） | **R** | C（准备与验证） | — |
| 验证 | I | **R** | — |
| 回滚 | A | R | C |
| 审计 | **A** | R（自动化审计） | — |
| **业务数字生成** | A（业务负责） | **禁止** | — |

### 18.3 审批通道

| 项 | 规则 |
|---|---|
| 通道 | DingTalk 卡片（主）；KubeSphere/CLI 手动写 Task（备） |
| 实现 | 集群无公网入向 → **采用出向轮询**（n8n 定时查询审批状态）；若必须回调，需 Ingress + IP 白名单 + 签名校验并登记 ADR |
| 审批人 | 白名单；涉及 DA-SOC 时须业务负责人共同批准 |
| 超时 | 默认 4 小时 → `Expired`（**不自动执行、不自动降级**）；紧急任务可设 30 分钟 |
| 拒绝 | 必须填理由并写入 Task；同一计划被拒 2 次则该 Task 关闭并转人工 |
| 双人原则 | 涉及不可逆且影响业务的，需 2 人批准（V0.1 建议，V0.2 强制） |

---

## 19. Risk Levels

### 19.1 资源分级（tier）

| tier | 定义 | 示例 |
|---|---|---|
| **A** | 影响业务连续性或平台控制面 | `kube-system`、`xw-system`、集群级 RBAC/NetPol/CNI/StorageClass/PV/Node、etcd、apiserver、**`da-soc` 的 ClickHouse 数据与工作流定义** |
| **B** | 影响运维效率但可重建 | `xw-obs`、`xw-ops`、`da-soc` 的 Deployment（render） |
| **C** | 可随时重建、无业务影响 | 临时 Job、测试命名空间 |

### 19.2 判定规则（确定性，**Agent 不得覆盖**）

```text
R1  只读                                     → L0
R2  可逆变更 ∧ tier=C                        → L1
R3  可逆变更 ∧ tier=B ∧ 非业务命名空间        → L1
R4  可逆变更 ∧ tier=A                        → L2
R5  可逆变更 ∧ 位于 da-soc ∧ 影响出数链路     → L2
R6  不可逆变更（任意 tier）                   → L2
R7  业务数字生成                             → 仅允许 DA-SOC 自身工作流执行；Agent 禁止
R8  无法写出验证方法的动作                    → 禁止执行
R9  规则未覆盖的动作                          → 默认 L2（fail-safe）
```

### 19.3 权威等级清单（V0.1）

| 操作 | 等级 | 依据 |
|---|---|---|
| 查询 Pod/Node 状态、查看指标、查日志 | L0 | R1 |
| 生成平台日报 / 巡检报告 | L0 | R1 |
| 读取 Task / 告警 / 备份状态 | L0 | R1 |
| 只读探测（连通性/越权验证测试） | L0 | R1（探测不得改变状态） |
| 重启 `xw-obs`/`xw-ops` 中 CrashLoop 的组件 | L1 | R3 |
| 扩容/缩容 `xw-obs`/`xw-ops` 的 Deployment（Quota 内） | L1 | R3 |
| 清理已完成的 Job | L1 | R2/R3 |
| 触发 Harbor GC / Trivy 重扫 | L1 | R3 |
| 重新触发失败的备份 Job | L1 | R3 |
| **重启 `da-soc/da-soc-render`** | **L1（非发送窗口）/ L2（窗口内）** | R3/R5 |
| 触发 DA-SOC 日常流程重跑 | **L2** | R5 |
| 重启 `da-soc` 的 ClickHouse | **L2** | R4/R5 |
| 删除 `da-soc` 的 PVC / 命名空间 | **L2** | R6 |
| 修改 RBAC / ClusterRoleBinding | **L2** | R4 |
| 修改 NetworkPolicy | **L2** | R4 |
| 修改 CNI / StorageClass / PV | **L2** | R4 |
| 删除 / drain 节点，修改 kubelet 参数 | **L2** | R4/R6 |
| Kubernetes / KubeSphere 升级 | **L2** | R4/R6 |
| 修改 Harbor 项目策略 / 保留策略 | **L2** | R4 |
| 修改 Secret / 加密密钥 | **L2** | R4 |
| 修改 DA-SOC 工作流 JSON / SQL | **L2**（且必须由 `build_workflow.py` 生成） | R5 |
| 任何数据删除 | **L2** | R6 |

---

## 20. IT / Business Boundary

### 20.1 职责划分

| 维度 | IT / 平台 | 业务（DA-SOC） |
|---|---|---|
| 负责对象 | VM、OS、K8s、KubeSphere、CNI、存储、Harbor、Ingress、RBAC、NetPol、监控、日志、备份、审计、安全基线、AI Ops 平台、Argo CD | 应用代码、业务逻辑、业务数据、SQL、工作流 JSON、业务配置、业务指标、业务日志、业务 SLA |
| 变更权 | 集群级与平台命名空间 | 仅 `da-soc` 内自身资源 |
| 不越界 | 不修改业务数据、不生成业务数字、不改业务代码 | 不改节点/CNI/集群级 RBAC/平台策略；不绕过 Harbor；不用 NodePort；不手工破坏平台状态 |

### 20.2 边界落地的 5 个机制（缺一不可）

| 机制 | 实现 | 违反证据 |
|---|---|---|
| **Namespace** | `da-soc` 承载全部业务对象 | 业务对象出现在其他命名空间 |
| **RBAC** | 业务主体仅对 `da-soc` admin；平台主体对其只读 | 审计中出现业务主体对非 `da-soc` 的写请求 |
| **ResourceQuota** | 含 `nodeports: 0`、`loadbalancers: 0`、CPU/内存/存储上限 | NodePort/LB 创建被拒 |
| **NetworkPolicy** | default-deny + 8 条放行 | 周期连通性测试发现未放行连接 |
| **Pod Security** | `restricted` + 例外登记 | 违规 Pod 创建被拒事件 |

### 20.3 业务自服务范围

**可自主：** 在 `da-soc` 内创建/更新 Deployment、ClusterIP Service、ConfigMap、Secret（加密）、Job、CronJob；Quota 内调整副本与资源；查看自己的日志/指标/事件；提交工作流/SQL 变更（Git PR）。

**必须申请（L2）：** Ingress / 对外暴露；NodePort / LoadBalancer；NetworkPolicy 变更；超 Quota；特权容器 / hostPath / hostNetwork；非 Harbor 镜像来源；访问平台命名空间或集群 API。

**绝对禁止：** 修改节点 / CNI / kubelet / 集群级 RBAC / 平台策略 / StorageClass；使用 `cluster-admin`；绕过 n8n 手工重跑业务链路并对外发送；直接改 ClickHouse 数据以"修"日报。

---

## 21. Deployment Sequence

### Phase 0：前置条件闭环（**在所有实施之前**）

| # | 动作 | 出口条件 |
|---|---|---|
| 0.1 | PoC 验证 **KubeSphere 3.4.x × K8s v1.26.x 补丁版 × Calico × MetalLB × PSS** 兼容性 | 单节点验证通过；确定精确版本 |
| 0.2 | PoC 验证 **ClickHouse 在 PSS `restricted` 下可运行** | 通过，或登记 1 条例外（含消除期限） |
| 0.3 | 记录 3 个现有镜像的 **digest**（在 ECS 上 `docker inspect`） | digest 写入版本矩阵 |
| 0.4 | 提交**出网白名单**与防火墙变更申请（邮件 993/995、DingTalk 443、NTP、内网源） | 获批 |
| 0.5 | 确认 **VM 资源获批**（或确认降级方案并登记 ADR） | 获批或降级方案确认 |
| 0.6 | 产出 `configs/version-matrix.yaml` + `configs/images/image-manifest.yaml` | 版本与 digest 全部固定 |
| 0.7 | 产出 `02-governance/`（治理、IT/业务边界、使用规范、安全基线） | 文档就绪 |
| 0.8 | 产出 `06-runbooks/` 首批 Runbook（A1–A12 对应） | **无 Runbook 的告警不允许上线** |
| 0.9 | 依据本 Baseline **重新生成 `TODO.md`** | 实施清单与 Baseline 一致 |

**门禁：Phase 0 未完成，不进入 Phase 1。**

### Phase 1：基础设施

| 步 | 动作 | 验证 |
|---|---|---|
| 1.1 | 7 台 VM 就位 + 数据盘 + 静态 IP | 网络连通、磁盘可用 |
| 1.2 | OS 基线（内核参数、containerd、chrony、auditd、SSH、主机防火墙） | Node Security Baseline Report；NTP 偏移 < 1s |
| 1.3 | 内网 DNS 登记（`*.xw.internal`） | 域名可解析 |
| 1.4 | `xw-mgmt-01` / `xw-bak-01` 服务基础（Docker/containerd、数据目录） | 就绪 |

### Phase 2：Kubernetes 集群

| 步 | 动作 | 验证 |
|---|---|---|
| 2.1 | kubeadm 首个控制面初始化（含 apiserver 安全参数、audit、加密配置） | 控制面健康；audit 日志产生；Secret 加密生效 |
| 2.2 | 加入 `xw-cp-02` / `xw-cp-03` | 3 节点 etcd 健康；VIP 可切换 |
| 2.3 | 加入 `xw-wk-01` / `xw-wk-02` | 节点 Ready；污点与标签正确 |
| 2.4 | 安装 Calico | Pod 跨节点通信正常 |
| 2.5 | 安装 MetalLB + ingress-nginx | 管理 VIP 可达；测试页可访问 |
| 2.6 | 安装 local-path-provisioner + ClickHouse 静态 PV | PVC 供给成功；PV Retain 生效 |
| 2.7 | Namespace / RBAC / ResourceQuota / LimitRange / PSS | 越权测试通过 |
| 2.8 | etcd 快照 CronJob | 快照产生并同步到 MinIO |

### Phase 3：平台服务

| 步 | 动作 | 验证 |
|---|---|---|
| 3.1 | MinIO（`xw-bak-01`）+ bucket | S3 可写；集群可访问 |
| 3.2 | Harbor（`xw-mgmt-01`）+ TLS + 项目 + robot account | `docker push/pull` 成功 |
| 3.3 | **离线镜像导入通路端到端验证（至少 1 个镜像）** | 导入 → push → 集群拉取 → digest 校验一致 |
| 3.4 | 把所有 V0.1 组件镜像导入 Harbor 并登记 digest | 清单完整；digest 与清单一致 |
| 3.5 | KubeSphere（最小组件集）+ 用户 + 角色 + Workspace | 控制台仅管理 VLAN 可达；自带组件已关闭（核对组件清单） |
| 3.6 | Argo CD + Git 仓库接入 + 首批 Application | 集群状态与 Git 一致；回滚演练成功 |
| 3.7 | kube-prometheus-stack + 12 条告警 | 告警规则加载；≥3 类人为触发均送达 DingTalk |
| 3.8 | Fluent Bit + Loki（含 K8s Audit 采集） | 可按 SA/verb 检索一条记录 |
| 3.9 | Velero + node-agent | 首次备份成功 |
| 3.10 | 平台 n8n（`xw-ops`）+ DingTalk 通知通道 | 平台告警可送达运维群；业务群零污染 |

### Phase 4：备份与恢复验证（**业务接入前必须完成**）

| 步 | 动作 | 验证 |
|---|---|---|
| 4.1 | etcd 快照恢复演练（沙箱） | R2 通过 |
| 4.2 | Velero 命名空间恢复演练 | R1 通过 |
| 4.3 | Harbor 数据恢复（可选） | 通过或记录为 V0.2 |
| 4.4 | 备份监控与告警（A7） | 人为使备份失败 → A7 触发 |

### Phase 5：DA-SOC 承载（迁移 10 阶段）

见 §15.6（P1–P10）。**出口条件：** 12 项 Cutover Gate 全部通过 + 切换完成 + 回退演练通过。

### Phase 6：AI Ops MVP

| 步 | 动作 | 验证 |
|---|---|---|
| 6.1 | Agent 运行时镜像构建 → Harbor（digest 固定） | 可运行 |
| 6.2 | 3 个 SA + Role/RoleBinding | 权限矩阵测试通过 |
| 6.3 | Task 模板与目录结构 | 可创建与更新 |
| 6.4 | 触发链路（Schedule / Alertmanager webhook / DingTalk） | 均可触发 Task 与 Job |
| 6.5 | L0 巡检任务族（Node/Pod/Disk/Cert/Backup/资源/日报摘要） | V1 通过 |
| 6.6 | 场景 1（L1 自动闭环） | V2 通过 |
| 6.7 | 场景 2（L2 审批闭环 + 失败回滚） | V3/V5 通过 |
| 6.8 | 审计可复盘验证 | V6 通过 |
| 6.9 | 配置漂移检测 + 数据新鲜度检测 | V7/V8 通过 |

### Phase 7：演练与验收

| 步 | 动作 | 验证 |
|---|---|---|
| 7.1 | 恢复演练 R1–R4（全部真实完成） | 通过，含实测 RTO |
| 7.2 | 故障演练 8 项 | 通过，含告警触发确认 |
| 7.3 | 安全验证（10 项越权/暴露测试 + 3 项补充） | `V0.1 Security Validation Report` |
| 7.4 | NetworkPolicy 双向连通性验证 | `V0.1 NetworkPolicy Validation Report` |
| 7.5 | Runbook 完整性（≥11 篇，8 段结构） | ≥3 篇被真实执行验证 |
| 7.6 | 架构与实际一致性核对 + 例外登记册 | 一致 |
| 7.7 | V0.1 验收报告 | 按 §22 逐项给出证据 |

---

## 22. Acceptance Criteria

### 22.1 平台类验收

```text
[ ] 7 台 VM 建成；OS 基线通过（Node Security Baseline Report）
[ ] 3 控制面集群健康；单控制面重启不影响集群
[ ] Calico 生效；NetworkPolicy default-deny + 8 条放行双向测试通过
[ ] KubeSphere 控制台仅管理 VLAN 可达；自带 monitoring/logging/auditing 已关闭（核对）
[ ] Namespace/RBAC/Quota/PSS 落地；越权测试 10 项全通过
[ ] local-path + local-static PV；ClickHouse PV Retain 生效
[ ] Harbor 提供全部镜像；digest 与 Git 清单一致；非授权来源镜像可被检出
[ ] ingress-nginx + MetalLB；管理域名可达；证书 > 30 天
[ ] 12 条告警就绪；≥3 类人为触发送达 DingTalk 运维群
[ ] Loki 可按 SA/verb 检索审计记录；K8s Audit 保留 90 天
[ ] Argo CD 同步正常；一次回滚演练成功
[ ] 架构与实际状态一致；所有例外汇总登记
```

### 22.2 恢复类验收（**未通过不得验收**）

```text
[ ] R1 Velero 命名空间恢复通过
[ ] R2 etcd 快照恢复通过
[ ] R3 ClickHouse 数据恢复通过（三窗口数字完全一致）
[ ] R4 DA-SOC 端到端恢复通过（测试群收到且数字为真实值）
[ ] 实测 RTO 记录在案；不达标项已修订 ADR
[ ] 备份监控告警有效（A7 可被触发）
```

### 22.3 DA-SOC 类验收

```text
[ ] n8n / ClickHouse / da-soc-render 全部在集群 da-soc 命名空间稳定运行
[ ] 三窗口（当天 / 近 6 周 / 近 6 月）数字与 ECS 基线完全一致
[ ] 出图与基线一致
[ ] 无数据场景输出 null/暂无数据（不为 0）
[ ] /archive 失败时全链路中止（不入库/不出图/不发送）
[ ] 邮件未读计数不变；无 Mark as Read 迹象
[ ] 未覆盖「监测bjfz邮箱广电报送信息」
[ ] 钉钉发送到正确群；生产群/测试群隔离有效
[ ] 工作流与 SQL 与 Git 产物一致（路径 A）
[ ] ClickHouse 双账号生效（da_soc_ro 写入被拒绝）
[ ] ECS 冷备可运行；回退演练 ≤30 分钟
```

### 22.4 AI Ops 类验收

```text
[ ] V1 L0 巡检可用
[ ] V2 L1 自动执行可用（由真实告警触发）
[ ] V3 L2 人工审批可用（一次真实审批并执行）
[ ] V4 验证独立可用
[ ] V5 回滚可用（一次真实失败 → 回滚 → 记录）
[ ] V6 审计可复盘（仅用 Git + Loki 重建决策链）
[ ] V7 配置漂移可发现（24 小时内产生 Task）
[ ] V8 数据新鲜度可发现（A10 触发并产生 Task）
```

### 22.5 运维类验收

```text
[ ] 监控 / 日志 / 告警 / 备份 / 恢复 / Runbook / Incident 记录全部就绪
[ ] 8 项故障演练完成并记录（含失败与改进项）
[ ] Runbook ≥11 篇，≥3 篇被真实执行验证
[ ] 例外登记册完整（每条含消除期限）
```

### 22.6 验收纪律

```text
[ ] 不以"组件安装完成"作为完成标准
[ ] 每项验收必须给出可验证证据（命令输出 / 截图 / 报告 / 日志检索结果）
[ ] 恢复演练未通过的项目不得勾选完成
[ ] AI 闭环必须由真实告警触发（不接受演示式闭环）
[ ] 数字一致性必须逐项比对（不接受"看起来一样"）
```

---

## 23. Rollback Strategy

### 23.1 分层回滚策略

| 层 | 回滚机制 | 适用 |
|---|---|---|
| **定义态** | Git revert / Argo CD 回退到上一 revision | 声明式清单、策略、配置变更 |
| **资源态** | Velero restore / Git re-apply | 资源被误删或损坏 |
| **数据态** | ClickHouse `RESTORE` / 文件级恢复 | 业务数据损坏 |
| **业务态** | **单写者回切 ECS（≤30 分钟）** | DA-SOC 迁移后出现 R1–R9 条件 |
| **集群态** | etcd 快照恢复 / Git 重建集群 | 控制面或整集群故障 |
| **不可回滚** | **禁止自动执行**；只能 L2 且必须预先声明 | 数据删除、节点删除、CNI 变更 |

### 23.2 回滚的前置条件（必须常备）

```text
[ ] Git 中有完整的定义态（含 kubeadm 配置、清单、Helm values、策略）
[ ] Harbor 独立于集群（集群重建时提供镜像）
[ ] 备份在集群外（MinIO）且时效 < 26 小时
[ ] Secret 可通过加密文件 + 离线密钥恢复
[ ] ECS 在 V0.1 期间保持可运行（镜像/数据/配置/凭据未被修改）
[ ] 回滚 Runbook 已编写且经过演练
```

### 23.3 回滚也是变更（纪律）

```text
[ ] 每次回滚必须创建/更新 Task，记录：理由、命令、时间、结果、后续动作
[ ] 回滚后必须重新走验证流程（对应 Gate 或验收项）
[ ] 回滚原因必须进入 Incident 记录并形成改进项
[ ] 若同一问题回滚 ≥2 次，必须停止推进并重新评审设计（产生 ADR）
```

---

## 24. Future Evolution

| 版本 | 新增 | 为什么不是 V0.1 |
|---|---|---|
| **V0.2** | 自动巡检任务族常态化；cert-manager；备份加密强制；恢复证据自动汇总；凭据轮换常态化；n8n 工作流双向漂移检测；Agent 工具集扩展（评估 MCP）；**评估 Task CRD（ADR-008 §7）**；**评估 `xw-opsapi`（ADR-007 §7.1）**；漏洞修复 SLA 与看板；Prometheus 长期存储评估 | V0.1 先证明能力存在；V0.2 把日常化做出来 |
| **V0.3** | EDR 接入；Runtime Security（Falco）；镜像签名 / SBOM / cosign；漏洞阻断门禁；SIEM-lite（审计关联）；CIS 基线常态化（kube-bench）；**Cilium 重新评估（ADR-002 §5）**；Vault / External Secrets；**评估 Kyverno/OPA（ADR-011）** | 需要前置的资产可见性与日志管道；且需外部系统就绪 |
| **V0.4** | Policy Engine（风险分级规则代码化）；Planner/Executor/Auditor 多 Agent 流水线；**L2 执行权有条件下放**；自动修复（限白名单）；自动验证框架；自动回滚；漂移自动修复（白名单）；Agent 长期记忆与知识库；混沌/故障注入常态化 | 需先积累足够的 Task 样本与验证经验；过早自动化会产生"错误的确定性" |
| **V0.5** | 第二/第三个业务接入；多租户与配额治理；服务目录/自助交付；SLA 与容量/成本管理；**分布式或共享存储（按真实需求决策）**；生命周期管理；Harbor HA；异地/不可变备份副本 | 需要至少 2 个真实业务才能提炼正确抽象 |
| **V1.0** | 跨站点/灾备；完整 Zero Trust；平台自愈与自优化；全量 CIS/合规持续符合；AI 承担绝大部分标准化运维；平台知识资产完备且与实际一致 | 需要前序版本沉淀的全部能力与运行数据 |

### 24.1 演进原则

```text
[ ] 每个版本只解决"上一版本证明存在的真实问题"，不解决"想象中未来会有的问题"
[ ] 任何新增组件必须回答：不做它会阻断哪项验收？
[ ] 任何版本的 Agent 权力扩张，必须伴随审计能力同步扩张
[ ] "未做过真实恢复演练的备份不算完成"在所有版本中不得放宽
[ ] "同一能力域只允许一套实现"在所有版本中不得放宽
```

### 24.2 V0.1 决策在后续版本的可复用性

| V0.1 决策 | 演进方式 | 是否需推翻 |
|---|---|---|
| kubeadm 3 CP | 升级 minor / 增 Worker / 节点池化 | ❌ 纯加法 |
| KubeSphere（最小） | 升级或替换管理面（关键能力不依赖它） | ❌ 可替换 |
| Calico | 评估 Cilium（NetPol 语义可平移） | ❌ 可替换 |
| local PV + MinIO | 数据可重建 + 备份在手 → 可在线迁移共享存储 | ❌ 可迁移 |
| Task（轻量文件） | 演进为 CRD（Git 保留归档）或维持 | ❌ 纯加法 |
| Agent Job + 3 SA | 细化角色、扩展工具集、V0.4 下放 L2 | ❌ 权限细化 |
| Harbor / MinIO 集群外 | 增加 HA、异地副本 | ❌ 纯加法 |
| Argo CD | App-of-Apps、多集群 | ❌ 纯加法 |
| PSS + Quota（无 Kyverno） | V0.3 增量引入 Kyverno/OPA | ❌ 纯加法 |
| ADR-001 承载策略 | ECS 退役 → 平台成为唯一承载 | ❌ 自然延续 |

### 24.3 参考 ADR 索引

| ADR | 主题 | 文件 |
|---|---|---|
| ADR-001 | DA-SOC v0.1 Hosting（承载 / 迁移 / 切换 / 回退） | `10-decisions/ADR/ADR-001-da-soc-hosting.md` |
| ADR-002 | CNI = Calico | `10-decisions/ADR/ADR-002-calico.md` |
| ADR-003 | Harbor 独立于 Kubernetes 集群 | `10-decisions/ADR/ADR-003-harbor.md` |
| ADR-004 | Storage = local PV / local-path | `10-decisions/ADR/ADR-004-storage.md` |
| ADR-005 | Observability 单栈 | `10-decisions/ADR/ADR-005-observability.md` |
| ADR-006 | Backup & Recovery | `10-decisions/ADR/ADR-006-backup.md` |
| ADR-007 | Agent Runtime = 短生命周期 Job | `10-decisions/ADR/ADR-007-agent-runtime.md` |
| ADR-008 | Task Model = 轻量文件 + Git | `10-decisions/ADR/ADR-008-task-model.md` |
| ADR-011 | Kubernetes 发行方式与版本基线（kubeadm v1.26.x / 3CP） | 见 §5（**建议正式补写为独立 ADR 文件**：`ADR-011-kubernetes-distribution.md`） |

---

**Baseline 结束。**

| 项 | 内容 |
|---|---|
| **效力** | 本文件是玄武云盾 V0.1 实施的**唯一架构依据**；与任何候选方案冲突时以本文件为准 |
| **变更方式** | 任何偏离本文件的做法必须走 ADR + 例外登记（含消除期限） |
| **未修改** | `TODO.md` 未在本阶段修改；将依据本文件在下一阶段重新生成 |
| **未实施** | 本阶段仅完成架构裁决与 Baseline，未创建任何 VM / 集群 / 组件 |
