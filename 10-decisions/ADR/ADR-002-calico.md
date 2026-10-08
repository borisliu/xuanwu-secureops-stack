# ADR-002：CNI 选型 —— Calico

| 项 | 内容 |
|---|---|
| **状态** | Accepted（原则接受，本次裁决确认并固化实施细节） |
| **相关** | ADR-001（DA-SOC 承载）、ADR-005（Observability） |

---

## 1. 决策

> **玄武云盾 V0.1 使用 Calico 作为唯一 CNI，采用 VXLAN 封装模式，标准 Kubernetes `NetworkPolicy`（networking.k8s.io/v1）作为东西向隔离的唯一机制。**
>
> **V0.1 不引入 Cilium、不引入 Hubble、不引入 Tetragon、不引入 eBPF 策略、不引入 Service Mesh。**

同时裁定：

1. **CNI 不再"评估"，直接采用 Calico。** 所有 5 份候选方案均选择 Calico，无分歧；V0.1 不得为此设立并行评估期。
2. **NetworkPolicy 是边界落地的强制机制**，每个业务与平台命名空间必须有 `default-deny`（Ingress + Egress）。
3. **集群内（东西向）用 NetworkPolicy；出集群（南北向出网）用边界防火墙/安全组**。两者职责不重叠、不互相替代。
4. **Cilium 的评估时点为 V0.3**，触发条件见 §5。

---

## 2. 背景

### 2.1 候选方案共识

| 方案 | CNI | 关键表述 |
|---|---|---|
| codebuddy | Calico | "NetworkPolicy 支持成熟、排障资料密度最高、无 eBPF 内核依赖" |
| codex | Calico | "优先保证 NetworkPolicy 的成熟度和可审计性"；"P1.5 不再长期评估" |
| cursor | Calico | "不在 V0.1 评估期内并行试 Cilium"；Cilium 列 V0.3/V0.4 评估 |
| dsh | Calico | "更保守、组件少、文档最全"；Cilium 列 V0.3 |
| kimi | Calico | "eBPF 运维面超出无专职团队承受力"；Cilium 明确拒绝 |

**5/5 一致选择 Calico，无架构分歧。** 唯一差异是细节表述（是否保留"评估"字样），本次裁决统一为"直接采用，不再评估"。

### 2.2 为什么这个共识值得确认而不是跳过

CNI 是**替换成本极高的决策**（涉及节点级 DaemonSet、IPAM、路由、iptables/eBPF 路径、故障排查方式）。一旦在实施期反复，代价远超收益。因此在 Baseline 中把它**钉死**是有价值的动作。

---

## 3. 选型理由（按裁决标准加权）

| 标准 | 权重 | Calico | Cilium |
|---|---:|---|---|
| 生产安全 | 25% | NetworkPolicy（L3/L4）完备，语义与上游一致，策略可审计 | 同样完备且支持 L7，但 L7 策略在 V0.1 无需求 |
| 可恢复性 | 20% | 故障模式少、状态可读（`calicoctl`、IPPool、Felix 配置） | 依赖 eBPF 与内核版本，故障诊断链路更长 |
| AI 可维护性 | 15% | 策略是标准 YAML，Agent 可直接 `kubectl get/get -o yaml`；无 eBPF 概念负担 | Hubble 对 Agent 更"可视化"，但要求 Agent 理解 eBPF |
| 运维复杂度 | 15% | 低：一个 DaemonSet + 少量 CRD；iptables 路径成熟 | 中高：内核依赖、eBPF map、kube-proxy 替换（V0.1 会带来额外风险） |
| V0.1 可实施性 | 10% | 高 | 中 |
| 后续演进能力 | 10% | 中：L7/可观测需换 CNI | 高：L7、Hubble、Tetragon |
| 成本/资源 | 5% | 低 | 中 |

**结论：** 在 V0.1 的权重结构下（生产安全 + 可恢复性 + 可维护性 = 60%），Calico 的"故障模式少、语义标准、排障资料密度高"直接对应权重最高的三项。Cilium 的优势（L7 策略、eBPF 性能、Hubble 可观测）集中在 V0.1 **没有需求**的方向上（权重合计 10% 的演进能力 + 5% 的成本）。

**明确的代价（必须记录）：** 放弃 L7 网络策略能力、放弃 Hubble 的流级可观测、放弃 Tetragon 的运行时安全。这三项在 V0.3 通过重新评估 CNI（ADR）或引入独立运行时安全组件解决。

---

## 4. 实施细节（Baseline 级要求）

### 4.1 安装与模式

| 项 | 裁定 |
|---|---|
| 安装方式 | 官方 manifest（`tigera-operator` 或 `calico.yaml`），**版本锁定并写入版本矩阵** |
| 数据面模式 | **VXLAN 封装**（不启用 BGP peering，避免与现网路由交互） |
| kube-proxy | **保留 kube-proxy**（K8s 1.26 + Calico 的默认稳定组合）；**不启用** `kube-proxy` 替换（eBPF 模式），理由：V0.1 需要最少的新故障域 |
| IPAM | Calico IPAM，Pod CIDR 由集群规划统一分配（与 Service CIDR 不重叠） |
| MTU | 按底层网络确定并**显式配置**（VXLAN 需扣除封装开销）；记录进版本矩阵，避免"半通"类疑难故障 |
| 双栈 | **不启用** IPv6（V0.1 无需求） |
| Typha | 节点数少，可直接用默认（不额外引入 Typha 调优） |

### 4.2 NetworkPolicy 基线（强制）

```text
1. 每个业务命名空间（da-soc）与平台命名空间（xw-* / harbor）必须有：
   - NetworkPolicy: default-deny-all  (podSelector: {}, policyTypes: [Ingress, Egress])
2. 所有放行必须逐条显式列出，且每条必须注明：
   - 用途、源、目的、端口、责任、验证方法
3. 禁止以下放行：
   - 0.0.0.0/0 的出向放行（出网必须走边界防火墙 + IP/端口白名单）
   - 跨命名空间的通配放行（如 from: namespaceSelector: {}）
   - 对 kube-system 的非必要访问
4. 变更纪律：NetworkPolicy 变更一律 L2（人工审批），且变更后必须执行连通性双向测试
```

### 4.3 与 DA-SOC 相关的放行清单（来自 ADR-001 §5.3）

| # | 方向 | 源 → 目的 | 端口 | 用途 |
|---|---|---|---|---|
| 1 | Egress | `da-soc` → `kube-system` CoreDNS | 53 | 域名解析 |
| 2 | Ingress | `da-soc/n8n` → `da-soc/da-soc-render` | 8091 | `/archive`、`/render` |
| 3 | Ingress | `da-soc/n8n` → `da-soc/clickhouse` | 8123, 9000 | 查询与写入 |
| 4 | Ingress | `xw-obs` Prometheus → `da-soc` | metrics | 抓取 |
| 5 | Egress | `da-soc/n8n` → 邮件服务器 | 993 | IMAP（只读，不标已读） |
| 6 | Egress | `da-soc/n8n` → DingTalk API | 443 | 发送 |
| 7 | Egress | `da-soc/clickhouse` → MinIO（集群外） | 9000 | `BACKUP` |
| 8 | Egress | **仅迁移期** `da-soc/migration-job` → ECS ClickHouse | 8123 | 历史数据回补；**完成后删除** |

### 4.4 验收（可证伪，必须留证）

| 测试 | 期望 | 证据 |
|---|---|---|
| n8n Pod → `da-soc-render:8091/healthz` | 成功 | `V0.1 NetworkPolicy Validation Report` |
| render Pod → 邮件服务器 993 | 超时/拒绝 | 同上 |
| `default` 命名空间测试 Pod → `clickhouse:8123` | 失败 | 同上 |
| 业务人员终端 → apiserver 6443 | 失败 | 同上 |
| 节点 → Pod IP 直连 8123 | 失败（无 hostNetwork、无 NodePort） | 同上 |
| `da-soc` Pod → 互联网任意地址 | 失败（除白名单） | 同上 |

> **纪律：允许的流量必须通，禁止的流量必须真的不通。** 不接受"策略已 apply"作为完成标准。

---

## 5. Cilium 重新评估的触发条件（V0.3）

满足**任一**条件时，V0.3 启动 Cilium 评估（而非直接替换）：

1. 出现明确的 **L7 策略需求**（如按 HTTP 路径/方法限制东西向流量）；
2. 出现 **流级可观测**（Hubble）的真实排障需求，且 Loki/Prometheus 无法满足；
3. 出现 **NetworkPolicy 规模或性能瓶颈**（策略条目数、iptables 规则数成为可测量瓶颈）；
4. 出现 **运行时安全**（Tetragon/Falco）需求，且评估表明与 CNI 合并更经济；
5. 节点规模显著增长（> 10 worker）导致 Calico 运维成本上升。

**评估不等于替换**：替换 CNI 是一次高风险变更（L2、需要维护窗口、需要回退方案），必须在 ADR 中重新裁决。

---

## 6. 被否决的方案及原因

| 被否决 | 原因 |
|---|---|
| 在 V0.1 并行评估 Cilium 与 Calico | 二者都会落地节点级组件，并行评估会把不确定性带入实施期；且 5 份方案已一致给出结论 |
| 直接采用 Cilium（eBPF + kube-proxy 替换） | 引入内核版本依赖、eBPF map 排障负担、与 kube-proxy 替换相关的额外风险；V0.1 无 L7/性能需求 |
| 启用 Cilium 的 Hubble/Tetragon | V0.1 的可观测由 Prometheus + Loki 覆盖；运行时安全属 V0.3 |
| 仅靠边界防火墙不做 NetworkPolicy | 无法实现东西向默认拒绝；README 安全红线要求"生产 Namespace 必须具备网络访问边界" |
| 用 Service Mesh 实现东西向控制 | 明确禁止过早引入 Service Mesh（组件堆砌） |

---

## 7. 后果与影响

**正面：** 网络策略语义与上游一致、可被 Agent 直接读取与验证、故障模式少、排障资料密度最高、无内核版本强依赖。

**负面 / 代价：** 无 L7 策略；无流级可观测；未来若需 eBPF 能力需换 CNI（高成本变更）。

**对后续版本的影响：** V0.3 评估 Cilium（触发条件如上）；在此之前，任何"为了未来能力"而提前引入 Cilium 的提案应被否决。
