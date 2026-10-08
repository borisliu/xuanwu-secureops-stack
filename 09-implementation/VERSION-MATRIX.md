# 玄武云盾 V0.1 Implementation Version Matrix

> 状态：候选版本基线，尚未完成组合兼容性冻结。
> 依据：`10-decisions/ARCHITECTURE-BASELINE-V0.1.md`、ADR-002～ADR-008，以及 V0.1 OS/云原生版本补充结论。

组件期望状态和生命周期策略由 `07-aiops/component-lifecycle/components.yaml`、`policies.yaml` 和 `upgrade-rules.yaml` 管理；本矩阵不替代运行态 Discovery evidence。

## 1. Version Freeze Rules

1. 本文件区分 `Candidate`、`Pending Compatibility Validation` 和 `Frozen`；候选版本不得描述为官方认证组合。
2. Kubernetes、KubeSphere、containerd、Calico 和 UOS 的组合必须经过实际安装、节点加入、网络、存储、Harbor、观测和 DA-SOC 验证后才能冻结。
3. 所有最终版本、镜像 digest、ISO/RPM/离线包 SHA256 和回滚版本必须进入 Git；禁止使用 `latest`。
4. 版本变更必须记录 Implementation Decision、影响、验证和回滚路径。
5. `TASK-001`～`TASK-005` 完成前不得安装对应组件；候选状态不等于安装放行。

## 2. OS and Cloud-Native Baseline

| Component | Candidate | Status | Freeze Condition | Owner |
|---|---|---|---|---|
| OS | 统信服务器操作系统 V20 1060e AMD64（免费使用授权） | Candidate / Pending Compatibility Validation | ISO/介质完整性、授权确认、OS 基线与 Kubernetes/KubeSphere 实际验证 | Platform Owner |
| Kubernetes | v1.30.6 | Candidate / Pending Compatibility Validation | KubeSphere 官方兼容矩阵、安装/升级/恢复验证和 UOS 节点验证 | Platform Owner |
| KubeSphere | 4.1.x，优先验证 4.1.2 | Candidate / Pending Compatibility Validation | 官方矩阵、Kubernetes v1.30.6 验证、UOS 实际安装验证 | Platform Owner |
| containerd | 1.7.x | Candidate / Pending Compatibility Validation | Kubernetes/KubeSphere/OS 的 CRI、cgroup、镜像拉取和恢复验证 | Platform Owner |
| CNI | Calico | Candidate / Pending Compatibility Validation | 集群网络、NetworkPolicy、外部出口和故障恢复验证 | Network Owner |

## 3. Existing Component Matrix

| Component | Selected Version | Status | Freeze Condition | Owner |
|---|---|---|---|---|
| Kernel | UOS 1060e 配套内核，具体 patch 待冻结 | Pending Version Freeze | containerd、Calico、Kubernetes 内核参数和安全基线验证 | Platform Owner |
| local-path | 与 Kubernetes v1.30.6 兼容的批准版本 | Pending Version Freeze | Local PV、节点绑定、数据盘和恢复验证 | Platform Owner |
| Harbor | Approved independent-VM release | Pending Version Freeze | 离线安装包、CA、镜像导入、备份恢复验证 | Registry Owner |
| Ingress Controller | Baseline-compatible approved release | Pending Version Freeze | KubeSphere/管理入口/TLS 验证 | Platform Owner |
| Prometheus | KubeSphere-compatible approved release | Pending Version Freeze | 指标采集、保留、恢复和告警验证 | Observability Owner |
| Grafana | KubeSphere-compatible approved release | Pending Version Freeze | dashboard、RBAC 和恢复验证 | Observability Owner |
| Alertmanager | Prometheus-compatible approved release | Pending Version Freeze | 路由、去重、DingTalk 和恢复验证 | Observability Owner |
| Fluent Bit | Runtime-compatible approved release | Pending Version Freeze | journald/container stdout/Audit 到 Loki 验证 | Observability Owner |
| Loki | Fluent Bit-compatible approved release | Pending Version Freeze | 日志脱敏、留存、查询和恢复验证 | Observability Owner |
| ClickHouse | Existing DA-SOC-compatible release | Pending Version Freeze | schema、历史数据、BACKUP/RESTORE 和业务 SQL 验证 | DA-SOC Owner |
| n8n | 2.15.0 或批准的现有兼容版本 | Candidate / Verify | workflow、encryption key、邮箱和 drift 验证 | DA-SOC Owner |
| render/archive | `da-soc-render:0.1` | Candidate / Verify | `/archive`、`/render`、熔断和图片一致性验证 | DA-SOC Owner |
| Agent image | Approved fixed digest | Pending Version Freeze | Job、RBAC、L0/L1/L2、TTL 和审计验证 | AIOps Owner |
| SOPS/age | Approved secret-encryption implementation | Pending Version Freeze | Secret 注入、轮换、离线恢复和双人保管验证 | Security Owner |

## 4. Combination Status

以下组合是 V0.1 的**候选基线**，不是官方认证或官方保证的完整组合：

```text
统信服务器操作系统 V20 1060e AMD64
  + Kubernetes v1.30.6
  + KubeSphere 4.1.x（优先验证 4.1.2）
  + containerd 1.7.x
  + Calico
```

必须完成实际安装、Control Plane/Worker 加入、Calico 网络、Local PV、Harbor、Observability、DA-SOC 和恢复验证后，才能把该组合状态改为 `Frozen`。

## 5. Required Freeze Evidence

- KubeSphere 官方兼容矩阵或版本说明快照。
- UOS Server V20 1060e AMD64 ISO/安装介质 SHA256 和免费使用授权确认。
- Kubernetes、KubeSphere、containerd、Calico、镜像和离线包的精确版本、digest 或 SHA256。
- 三节点安装、加入、网络、cgroup、存储、Harbor、观测和恢复测试报告。
- Platform Owner、Security Owner、Network Owner 和 DA-SOC Owner 审批。
- Git commit 及关联 Implementation Decision。

## 6. Parameters Still Pending Freeze

- KubeSphere 最终 patch version。
- Kubernetes、containerd、Calico 和配套内核的最终 patch version。
- 所有镜像 digest、OS ISO SHA256、离线 RPM/包清单和签名校验。
- 节点/Pod/Service CIDR、DNS、NTP、出口白名单。
- Secret management implementation、backup target、RPO/RTO。
