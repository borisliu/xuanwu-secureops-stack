# ADR-006：Backup & Recovery 架构

| 项 | 内容 |
|---|---|
| **状态** | Accepted（原则接受，本次裁决确认工具链、对象清单、频率保留与演练门槛） |
| **相关** | ADR-001、ADR-003、ADR-004、ADR-005 |

---

## 1. 决策

> **玄武云盾 V0.1 的备份由五条互补路径构成：`Git`（定义态）+ `etcd 快照`（集群状态）+ `Velero`（Kubernetes 资源与卷数据）+ `ClickHouse 原生 BACKUP`（业务数据）+ `文件级备份`（raw archive / n8n / Harbor / 平台卷），统一落向**集群外**的 MinIO（`xw-bak-01`）。**
>
> **纪律：未做过真实恢复演练的备份，不算完成。** V0.1 必须完成 **4 项强制恢复演练**，否则 V0.1 不予验收。

同时裁定：

1. **备份目标必须在集群故障域之外**（`xw-bak-01`）。备份与集群同故障域直接破坏"可恢复"目标。
2. **引入 Velero（含 node-agent 文件系统备份）**，用于 Kubernetes 资源与 PVC 数据恢复。**不依赖 CSI 快照**（local-path 无快照能力）。
3. **Git 是定义态的唯一事实源**；Velero 是运行态与数据态的恢复手段。两者职责不重叠。
4. **ClickHouse 数据必须有两条恢复路径**：① 原生 `BACKUP`/`RESTORE`；② **源邮件重放（POP3/IMAP 回补）**作为兜底。
5. **备份本身必须被监控**：失败或超龄 → 告警（A7）→ 产生 Task。
6. **Harbor 与 MinIO 的恢复不依赖 Kubernetes 集群。**

---

## 2. 背景

### 2.1 候选方案共识与分歧

**共识（5/5）：** 备份必须覆盖 etcd、Kubernetes 资源、ClickHouse、raw archive、n8n、Harbor、平台配置；必须有真实恢复演练；Git 承载定义态。

**分歧：**

| 分歧点 | 方案立场 | 裁决 |
|---|---|---|
| 是否用 Velero | codebuddy **明确不用**（避免引入 MinIO/S3 依赖与运维面；local-path 无快照）；cursor/kimi/codex 使用或可选使用 | **使用 Velero** |
| 是否用 MinIO/S3 | codebuddy 用 restic 到备份节点；kimi 用 MinIO；cursor 用文件路径 | **使用 MinIO（S3）** |
| 备份对象是否含 raw archive | 一致要求覆盖；归属有分歧（ECS 保留 vs 迁入集群） | 按 ADR-001：raw 窗口在集群，全量在 MinIO |
| RPO 声明 | 多数未给出显式 RPO（仅给备份频率） | **本 ADR 明确声明 RPO/RTO 数值** |
| 演练数量 | 3 项（codebuddy）/ 4 项（kimi/dsh）/ 未定（codex） | **4 项强制** |

### 2.2 为什么采用 Velero（对 codebuddy"不用 Velero"论证的检验）

codebuddy 的论证是：Velero 需要对象存储（引入 MinIO/S3 两个组件与新运维面）、V0.1 无多集群/跨集群迁移需求、local-path 不支持 CSI 快照因此 Velero 的核心价值用不上。

**逐条检验：**

| 论点 | 检验 |
|---|---|
| "需要对象存储" | 对象存储本来就需要：ClickHouse 原生 BACKUP 需要一个外部目标，ADR-004 已确定 MinIO。因此 Velero 并没有"新增"对象存储，只是**复用**它。 |
| "local-path 无 CSI 快照，Velero 价值用不上" | **不成立。** Velero 的核心价值不止快照：① **命名空间级 / 标签选择器级的资源恢复**（`velero restore create --from-backup`）；② **PVC 数据的文件系统备份与恢复**（Restic/Kopia + node-agent，`--default-volumes-to-fs-backup`），这正是 local-path 场景下唯一可行的卷数据恢复手段；③ 备份对象清单与状态的 **CR + CLI/JSON**，可被 Agent 直接读取。 |
| "无多集群需求" | 本裁决不依赖多集群能力。 |

**关键论证：** 没有 Velero 时，"误删命名空间 → 恢复" 与 "PVC 数据恢复" 只能靠**手工**：`kubectl apply` 回 Git（能恢复资源定义）+ 从文件备份手工还原目录（能恢复数据），但**两者之间的对应关系、恢复顺序、部分失败的处置都需要人工编排**。这正是"缺少专职运维团队"最脆弱的环节。Velero 把这一过程变成**一条命令 + 一个可查询的 CR 状态**，对 AI-Native 目标是净收益。

**代价（显式接受）：** 引入 1 个控制器（Deployment）+ 1 个 node-agent（DaemonSet）+ 备份状态 CRD，以及 MinIO 这一对象存储。因 MinIO 已被 ADR-004 需要，净增组件仅为 Velero 自身。

### 2.3 为什么必须用 MinIO（对象存储）

| 需求 | 需要对象存储的理由 |
|---|---|
| ClickHouse `BACKUP ... TO Disk('backups', 's3://…')` | ClickHouse 原生备份**原生支持 S3 目标**，无需自研导出逻辑 |
| Velero | 标准备份存储后端为 S3/对象存储 |
| Harbor 备份 | 大体积镜像层适合对象存储 |
| 归档不可变副本 | 对象存储便于做"保留 + 只读"归档 |
| 未来扩展 | 不需要在 V0.1 就换存储后端 |

MinIO 部署在 `xw-bak-01`（**集群外**），单实例（V0.1 接受非 HA，由文件级备份保护 MinIO 自身数据目录）。

---

## 3. 备份对象清单（必须全覆盖）

| # | 对象 | 路径 | 工具 | 频率 | 保留 |
|---|---|---|---|---|---|
| B1 | **etcd 快照** | etcd | `etcdctl snapshot save`（CronJob 或 systemd timer） | 每 30 分钟 | 本地 2 天 + 远端 30 天 |
| B2 | **平台定义态** | Git | Git（含远端镜像仓库） | 每次变更 | 永久 |
| B3 | **Kubernetes 资源 + 卷数据** | 全集群 / 关键命名空间 | **Velero + node-agent（文件系统备份）** | 每日（`da-soc`、`xw-ops`、`xw-obs`）+ 每周全量 | 30 日 / 12 周 |
| B4 | **ClickHouse 数据** | `da-soc` | ClickHouse 原生 `BACKUP TABLE ... TO Disk('backups','s3://…')` | 每日（日报成功后） | 30 天 |
| B5 | **ClickHouse 表结构** | `da-soc` | `SHOW CREATE TABLE` 导出 → Git + MinIO | 每日 | 永久（Git） |
| B6 | **raw archive（窗口）** | `da-soc` PVC | 文件级备份（restic/rsync） | 每日 | 30 天 |
| B7 | **raw archive（全量历史）** | MinIO（不可变对象） | 一次性归档 + 增量追加 | 每日 | 长期/按业务要求 |
| B8 | **n8n 数据（工作流 + 执行历史）** | `da-soc` | Velero（PVC）+ 工作流 JSON 在 Git | 每日 | 14 天 |
| B9 | **n8n 工作流 JSON** | Git | `build_workflow.py` 产物入 Git | 每次变更 | 永久 |
| B10 | **SQL 源文件** | Git | Git | 每次变更 | 永久 |
| B11 | **Harbor 配置 + 数据** | `xw-mgmt-01` | 配置导出 + 数据目录文件级备份；**镜像 tar 存档**兜底 | 每周全量 + 每日增量 | 4 周 |
| B12 | **平台配置** | `xw-obs` 等 | Velero + Git（Helm values / 清单） | 每日 | 30 天 |
| B13 | **KubeSphere 关键配置** | `kubesphere-*` | 资源导出 + Git | 变更时 | 永久（Git） |
| B14 | **MinIO 自身数据** | `xw-bak-01` | 数据目录文件级备份（异盘/异地） | 每日 | 30 天 |
| B15 | **凭据恢复机制** | Secret / Sealed 文件 | 见 §4 | 变更时 | 永久 |
| B16 | **审计日志** | Loki | 由 Loki 保留策略承担（90 天）；不单独备份 | — | 90 天 |
| B17 | **监控数据** | Prometheus | **明确不备份**（可丢失，非 V0.1 恢复目标） | — | — |

---

## 4. Secret 与凭据的恢复机制（关键设计）

**问题：** Secret 不能明文进 Git，但"恢复后系统必须能用"意味着凭据必须可恢复。

**裁决（三段式）：**

| 层 | 机制 | 说明 |
|---|---|---|
| ① 加密入 Git | Secret 以**加密形态**（Sealed Secret 或 SOPS 密文）进 Git | 满足"Git 即事实源"；Git 中不含明文 |
| ② 静态加密 | kube-apiserver `EncryptionConfiguration`（aescbc/secretbox） | 保护 etcd 中的 Secret |
| ③ 加密密钥托管 | 加密密钥**离线保管于受控位置**（离线介质/密钥保险柜），**不进 Git**，并登记在 `04-security/secret-inventory.md` | 恢复集群时用于解密 Sealed Secret 或恢复加密能力 |

**必须登记的信息（进 Git）：** 凭据清单（名称、用途、位置、负责人、轮换周期、恢复方式）。**不得进 Git：** 凭据明文值、加密密钥本体。

**恢复演练要求：** 密钥可读性验证纳入年度检查；恢复路径（从加密文件恢复 Secret）必须在 R1/R2 演练中被顺带验证。

---

## 5. RPO / RTO 声明

| 场景 | RPO | RTO | 主要路径 |
|---|---|---|---|
| 单 Pod 被误删 | 0 | ≤ 15 分钟 | Git re-apply / 控制器自愈 |
| 配置被误改（声明式） | 0 | ≤ 15 分钟 | Argo CD 回退 / Git revert |
| `da-soc` 命名空间整体误删 | ≤ 24h（数据）/ 0（定义） | ≤ 2 小时 | Velero restore + Git apply |
| 单个 Worker 节点故障 | 0（优先恢复原节点） | ≤ 4 小时 | 节点恢复；必要时 Velero + ClickHouse RESTORE |
| ClickHouse 数据盘损坏 | ≤ 24 小时 | ≤ 4 小时 | ClickHouse RESTORE；兜底邮件重放 |
| 控制面全失（etcd 损坏） | ≤ 30 分钟 | ≤ 4 小时 | etcd 快照恢复 |
| 整集群重建 | ≤ 24 小时 | ≤ 8 小时 | Git + Harbor（集群外）+ Velero + MinIO |
| Harbor 故障 | 0 | ≤ 4 小时 | 数据目录备份 / 镜像 tar |
| MinIO 故障 | ≤ 24 小时 | ≤ 4 小时 | MinIO 数据目录备份 |

**诚实声明：** 表中 RPO/RTO 是**目标值**，必须在演练中验证；若演练结果不达标，需修订本 ADR（而不是默认维持）。

---

## 6. 强制恢复演练（V0.1 必须完成 4 项）

| # | 演练 | 方法 | 成功判据 | 优先级 |
|---|---|---|---|---|
| **R1** | **Velero 命名空间恢复** | 删除/污染 `da-soc`（或先建沙箱命名空间）→ 从备份恢复 | 资源与 PVC 数据完整；Pod Running；健康检查通过 | P0 |
| **R2** | **etcd 快照恢复** | 在**隔离沙箱单节点**上恢复（优先）；如需在生产控制面执行，必须走维护窗口 + L2 审批 | apiserver 起来；核心资源与备份时刻一致 | P0 |
| **R3** | **ClickHouse 数据恢复** | `RESTORE` 到 `da_soc_restore` 库 → 与源比对 | `count(*)` 与三窗口关键聚合**完全一致**；空值语义正确（无数据不为 0） | **P0（业务关键）** |
| **R4** | **DA-SOC 端到端恢复** | 恢复 `da-soc` 全栈 + 数据 → 触发一次完整日常流程 → 发到测试群 | 测试群收到图，且**数字为真实值**（非 0、非空、与恢复前一致） | **P0（业务关键）** |

**纪律：**

1. **R3/R4 必须与业务方共同完成并签字确认。**
2. 每次演练必须产出记录：范围、步骤、耗时（实测 RTO）、发现的问题、改进项、证据链接。
3. 演练失败 → 生成 Task → 修复 → 重新演练（计入 V0.1 验收）。
4. **演练不得影响生产日报**：优先在沙箱命名空间/隔离环境执行；生产侧演练必须在维护窗口内，且以"日报已成功产出"为前提。

**可选补充演练（时间允许）：** Harbor 数据恢复（kimi 列为第 4 项，本 ADR 将其降级为"可选"，理由是 Harbor 故障不影响业务日报链路，且镜像可重建 + tar 兜底）。

---

## 7. 备份有效性保障

| 机制 | 做法 |
|---|---|
| 成功校验 | 每次备份任务必须有明确的成功/失败状态（Velero CR phase、exit code、`system.backups` 表） |
| 年龄检查 | 最新备份年龄 > 26 小时 → 告警 A7 |
| 容量检查 | 备份目标使用率 > 80% → 告警 |
| 完整性抽查 | 每月抽查一次备份内容可读性（文件可列出、对象可下载、CH 备份文件存在） |
| 责任人 | 每类备份必须有明确责任人（登记在 `12-assets/` 与 Runbook） |
| 监控接入 | 备份状态经 Pushgateway/exporter 进入 Prometheus，纳入 A7 |
| 失败处置 | 自动重试最多 2 次；仍失败 → 告警 + 产生 Task（不静默重试） |
| 加密 | 备份传输与静态加密列为 **V0.1 建议项**，V0.2 强制 |

---

## 8. 恢复顺序（标准 Runbook 骨架）

```text
0. 冻结变更（声明维护窗口；停止自动化变更与 Agent 执行）
1. 恢复/重建控制面（etcd 快照 或 Git 重建集群）
2. 恢复 CNI / 核心插件 就绪
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

## 9. 被否决的方案及原因

| 被否决 | 原因 |
|---|---|
| 备份目标放在集群内（PVC/MinIO-in-cluster） | 集群故障时备份与业务同时丢失，直接破坏"可恢复"目标 |
| 不使用 Velero，仅靠"Git + 手工文件复制" | 命名空间级恢复、PVC 数据恢复、部分失败处置都需要人工编排；对无专职团队风险过高（见 §2.2 检验） |
| 使用 restic 替代 Velero 作为唯一 K8s 备份手段 | restic 只做文件级备份，**不理解 Kubernetes 对象语义**，无法做"命名空间恢复"；可保留为文件级备份工具（raw/Harbor/平台卷），但不作为资源恢复主工具 |
| 依赖 CSI 快照 | local-path 无快照能力（ADR-004）；引入快照能力需换存储，属本末倒置 |
| 只做"备份文件存在"形式验收 | 违反 TODO 原有纪律"没有做过恢复演练的备份不算完成"；无法证明可恢复 |
| 备份 Prometheus 监控数据 | 非 V0.1 恢复目标；数据可重建（且价值随时间衰减） |
| 把 Secret 明文放进 Git 以便恢复 | 违反安全红线。改用"加密入 Git + 静态加密 + 离线密钥"三段式（§4） |
| 在生产集群上直接演练 etcd 恢复 | 风险过高；优先沙箱/隔离环境，生产演练必须维护窗口 + L2 |
| 双写 / 异地实时复制作为 V0.1 方案 | 复杂度远超收益；异地副本属 V0.5/V1.0 |

---

## 10. 后果与影响

**正面：** 五条互补路径覆盖定义态/集群状态/资源/业务数据/文件；恢复顺序明确；Harbor 与备份都在集群外，"整集群重建"可行；备份状态可被 Agent 读取并产生 Task。

**负面 / 代价：**

1. 引入 Velero + node-agent + MinIO（净增 1 个控制器 + 1 个对象存储）；
2. 每日备份产生存储与网络开销（MinIO 需 ≥ 1.5 TiB）；
3. 4 项强制演练需要真实的人力与时间投入（预计 2–3 人日）；
4. 单实例 MinIO 是单点（由数据目录备份 + 独立故障域对冲）。

**对后续版本的影响：** V0.2 增加备份加密强制、恢复证据自动汇总、异地/不可变副本评估；V0.5 评估异地点与共享存储；V1.0 完整灾备。**"未经演练的备份不算完成"这一纪律在所有版本中不得放宽。**
