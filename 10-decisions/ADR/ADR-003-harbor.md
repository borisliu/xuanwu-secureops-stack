# ADR-003：Harbor 镜像仓库 —— 独立于 Kubernetes 集群部署

| 项 | 内容 |
|---|---|
| **状态** | Accepted（原则接受，本次裁决确认部署位置与实施细节） |
| **相关** | ADR-001、ADR-006（Backup）、ADR-009（镜像供应链，见 10-decisions/ARCHITECTURE-BASELINE-V0.1.md §9） |

---

## 1. 决策

> **Harbor 是玄武云盾 V0.1 唯一的镜像来源与离线镜像收敛点，部署在 Kubernetes 集群之外的独立节点（`xw-mgmt-01`，Docker Compose 形态），生命周期与业务 Kubernetes 集群解耦。集群可以从 Git 定义态重建，而 Harbor 不受该重建影响。**
>
> **离线镜像导入（`docker save` → 传输 → `docker load` → `docker push` Harbor）是 V0.1 的一等公民流程，不是应急手段。**

同时裁定：

1. **Harbor 必须能从零重建 K8s 集群提供镜像供给**（即：Harbor 不依赖集群存活）。
2. **所有生产镜像必须来自 Harbor，且以 digest 固定**；禁止 `latest`。
3. **Trivy 扫描在 V0.1 启用但不做阻断门禁**；高危漏洞必须产生 Task，修复门禁属 V0.2/V0.3。
4. **Harbor 自身必须有备份**（配置 + 数据卷/导出），且恢复路径经过演练。

---

## 2. 背景

### 2.1 为什么需要 Registry（5 份方案一致）

| 理由 | 说明 |
|---|---|
| 离线事实 | 现有 ECS 不能直连 Docker Registry，现状靠 `docker save`/`docker load` 手工搬运：无版本记录、无扫描、无审计 |
| 安全红线 | README §8："生产镜像必须来自企业认可的 Registry"、"镜像必须经过基本安全检查"——没有 Registry 就没有这条红线的落点 |
| 可重建性 | 集群损坏后需要重新拉取全部组件镜像；镜像来源必须独立于集群 |
| 可审计性 | 需要知道"集群里跑的到底是哪个镜像的哪个字节" |
| AI 可操作性 | Harbor 提供完整 REST API（repository / artifact / scan / robot），Agent 可用 JSON 判断镜像来源与漏洞状态 |

### 2.2 候选方案在"部署位置"上的分歧（本次裁决的焦点）

| 方案 | 位置 | 理由 |
|---|---|---|
| **kimi** | **K8s 之外（独立 `xw-svc-01`，docker-compose）** | "Registry 是集群的恢复路径——集群损坏时需从 Harbor 拉镜像重建；Harbor 若在集群内，则形成『救生机在沉船上』的循环依赖" |
| **codex** | **K8s 之外（独立 `xw-reg-01` VM，或复用企业 Registry）** | 与 ADR-003 的独立生命周期要求一致 |
| codebuddy | 集群内 `harbor` ns | "统一运维域（一套监控/日志/备份/审计）；独占节点避免争抢；镜像 tar 备份兜底" |
| cursor | 集群内 `xw-wk-01` | "需要平台托管" |
| dsh | 集群内 `platform-registry` ns | 简化运维 |

**裁决倾向：** 已原则接受的 ADR-003 明确"**Harbor 属于平台恢复路径的重要基础设施，不应与业务 Kubernetes 集群形成过强的生命周期耦合**"。这与 kimi/codex 的"K8s 外"一致，与 codebuddy/cursor/dsh 的"集群内"冲突。因此**采纳"K8s 之外"**。

### 2.3 "循环依赖"论证的检验

反方（集群内）的论证是"统一运维域"——即 Harbor 在集群内可以复用同一套监控/日志/备份/审计。

正方的论证是"恢复路径不能在故障域内"。逐一检验：

| 场景 | Harbor 在集群内 | Harbor 在集群外 |
|---|---|---|
| 单个业务 Pod 故障 | 无差异 | 无差异 |
| 单个 Worker 节点故障（Harbor 恰好在该节点） | Harbor 中断，需重建/等待 PV 恢复 | 无影响 |
| **整个集群重建（etcd 损坏 / 需要从 Git 重建）** | **Harbor 随集群一起消失；重建集群需要镜像，而镜像在已消失的 Harbor 里 → 死锁（只能退回原始 `docker load`，且失去全部 digest 记录）** | **Harbor 存活，重建集群可直接从 Harbor 拉取，digest 记录完整** |
| Harbor 故障本身 | 与集群故障叠加，难以区分根因 | 故障域隔离，诊断清晰 |

**结论：** "整个集群重建"是 V0.1 必须能应对的场景（README §4.7、§22 的"可恢复"，且 TODO 明确要求恢复演练）。在这一场景下，集群内 Harbor 会形成死锁，而集群外 Harbor 恰好是解开死锁的钥匙。**恢复路径的独立性优先于运维域的便利性。**

**关于"统一运维域"的代价补偿：** Harbor 在集群外并不意味着脱离平台管理：

- 监控：Harbor 暴露 `/metrics`，由集群内 Prometheus 抓取（跨集群外目标抓取，已在访问矩阵中放行）；
- 日志：Harbor 日志经宿主机 Fluent Bit/journald 采集（见 ADR-005）；
- 备份：纳入 ADR-006 的备份对象与保留策略；
- 审计：Harbor 自带审计日志 + 由平台定期导出归档；
- Agent 可读：Harbor REST API 对 Agent 只读开放（robot account）。

即：**"独立部署"不等于"独立运维"，只是把故障域与生命周期解耦。**

---

## 3. 部署形态

| 项 | 裁定 |
|---|---|
| 部署位置 | `xw-mgmt-01`（管理/服务节点，K8s 之外） |
| 部署方式 | Docker Compose（官方 offline installer 形态）+ containerd/docker 运行时 |
| 组件范围 | 启用：Core、Portal、Registry、Jobservice、Trivy、内置 PostgreSQL、内置 Redis。**不启用**：Notary/签名、跨实例复制、HA 多副本 |
| TLS | 内部 CA 签发的证书（`harbor.xw.internal` 或等价内网域名）；集群节点 containerd 信任该 CA |
| 存储 | 本地数据盘（建议 ≥ 300 GB，独立于系统盘）；数据目录纳入备份 |
| 访问控制 | 项目级 RBAC；`xuanwu-platform/*` 与 `da-soc/*` 两个项目；节点/CI 使用项目级 robot account（只读 pull） |
| 准入域名 | 仅管理 VLAN + 节点 VLAN 可达；**不暴露到互联网** |
| 端口 | 443（HTTPS）+ 仅供监控抓取的管理端口；不开放 registry 明文端口到业务网 |

### 3.1 与 Kubernetes 集成

| 项 | 做法 |
|---|---|
| 节点拉取 | containerd `registries.yaml` 配置 Harbor 为默认 registry（或 mirror），信任内部 CA；**不使用 insecure-registry** |
| Pod 拉取凭据 | 每个命名空间配置 `imagePullSecret`（Harbor robot account，最小权限：仅可 pull 本项目） |
| 拉取策略 | `imagePullPolicy: IfNotPresent`（配合 digest 固定） |
| 来源强制 | 通过**资源配额 + 版本矩阵 + 每日漂移检查**保证镜像来自 Harbor。**V0.1 不引入策略引擎（见 ADR-010）**；若 V0.1 后期需要强制准入，则引入 Kyverno 单条策略作为独立 ADR |
| digest 固定 | 所有镜像在 Git `configs/images/image-manifest.yaml` 中记录 `<repo>:<tag>@sha256:<digest>`，部署清单引用 digest |

### 3.2 离线镜像导入流程（Standard Operating Procedure，必须脚本化 + 入 Git）

```text
[构建机 / 可出网跳板机]
  1. 按 configs/images/image-manifest.yaml 拉取镜像
  2. docker save → tar（按用途分组：base / harbor / platform / da-soc）
  3. 生成 sha256sum 校验文件 + 记录每个镜像的 RepoDigest
        │ 传输（内网通道 / 介质）
        ▼
[管理节点 xw-mgmt-01]
  4. 校验 sha256sum
  5. docker load（或 ctr -n k8s.io images import，仅用于 Harbor 自举阶段）
  6. docker tag → docker push harbor.xw.internal/<project>/<repo>:<tag>
  7. 在 Git 中更新 image-manifest.yaml（记录 digest、来源、审核人、日期）
        │
        ▼
[集群]
  8. 节点 containerd 从 Harbor 拉取；清单以 digest 引用
  9. 校验：kubectl get pod -o jsonpath 确认 imageID == manifest 中的 digest
```

**纪律：**

1. **`ctr images import` 是唯一允许绕过 Harbor 的通道，且仅限"Harbor 尚未就绪"的自举阶段**；Harbor 就绪后该通道关闭（Runbook 中写明关闭条件）。
2. 每个镜像必须记录 digest；tag 可变，digest 不可变。
3. 禁止 `latest`。
4. 镜像 tar 保留在 `xw-bak-01`（离线镜像中转/存档），作为 Harbor 全损时的兜底。

---

## 4. 扫描与门禁

| 项 | V0.1 裁定 | 理由 |
|---|---|---|
| Trivy 扫描 | **启用**（推送时扫描 + 每周全量重扫） | 满足"镜像必须经过基本安全检查" |
| 阻断门禁 | **不启用** | ① README 把漏洞运营体系放在 V0.3；② 现有 DA-SOC 镜像可能存在历史漏洞，若开阻断会直接阻断业务修复路径；③ V0.1 的镜像来源可控（Harbor 唯一 + digest 固定），风险可接受 |
| 高危漏洞处置 | **产生 Task**（进入 AI Ops 闭环），修复 SLA 属 V0.2/V0.3 | 让漏洞可见、可追踪、可审计 |
| 镜像签名 / SBOM / cosign | **V0.3** | 需要密钥治理与供应链体系 |
| 保留策略 | 每个项目保留最近 N 个 tag（N 建议 10） | 控制存储增长 |
| 垃圾回收 | 每周 1 次（Harbor GC） | 同上 |

---

## 5. Harbor 自身的备份与恢复

| 项 | 做法 |
|---|---|
| 备份对象 | Harbor 配置（`harbor.yml`）、数据库导出、registry 数据目录、Trivy DB 状态 |
| 备份方式 | ① 配置与导出物进 Git（不含凭据明文）/ 加密存储；② registry 数据目录文件级备份（restic/rsync）→ MinIO（`xw-bak-01`） |
| 频率 | **每周**全量 + 每日增量（数据目录） |
| 保留 | 4 周 |
| 兜底 | **镜像 tar 存档**（`xw-bak-01`）——即使 Harbor 完全损毁，也能通过 `docker load` 恢复关键镜像 |
| 恢复演练 | V0.1 至少完成一次 **Harbor 数据恢复演练**（恢复到可用状态并能被集群拉取） |

**关键设计：** Harbor 的恢复不依赖 Kubernetes 集群（这正是 ADR-003 的核心收益）。恢复顺序为：恢复 MinIO 中的备份 → 恢复 Harbor → 集群按需拉取。

---

## 6. 被否决的方案及原因

| 被否决 | 原因 |
|---|---|
| Harbor 部署在业务 K8s 集群内 | 与集群形成生命周期耦合；"整个集群重建"场景下形成死锁，直接破坏 V0.1 的可恢复性目标 |
| 直接使用 CNCF Distribution（`registry:2`）替代 Harbor | 无项目权限、无 UI、无扫描、无 API 审计能力；"对无专家团队来说『看得见』比『更轻』更重要" |
| 使用公共 Registry 作为生产来源 | 违反 README 安全红线；且环境本身不具备直连条件 |
| 在 V0.1 启用 Notary/镜像签名与 SBOM 门禁 | 需要密钥治理与供应链体系，属 V0.3 |
| 在 V0.1 启用漏洞阻断门禁 | 与"先让业务稳定承载"冲突；现有镜像可能带历史漏洞，阻断会直接阻断修复路径 |
| Harbor HA 多副本 | V0.1 无该需求；备份 + 镜像 tar 兜底的成本远低于 HA 副本 |

---

## 7. 后果与影响

**正面：** 恢复路径独立、集群可重建、镜像来源可控可审计、离线导入流程标准化、故障域清晰。

**负面 / 代价：**

1. 管理节点多承担一个有状态服务（`xw-mgmt-01` 需要足够磁盘与内存）；
2. Harbor 成为**单点**（V0.1 接受，由备份 + 镜像 tar 兜底；HA 属 V0.5 多租户阶段）；
3. 离线导入流程需要人工参与（可脚本化，但无法完全自动化——这是环境的客观约束）；
4. 跨集群外抓取监控指标需要额外的网络放行。

**对后续版本的影响：** V0.2/V0.3 可评估 Harbor HA 或迁移到共享存储；V0.3 评估镜像签名与扫描门禁。在任何版本中，**Harbor "独立于集群生命周期"这一属性都不得破坏**。
