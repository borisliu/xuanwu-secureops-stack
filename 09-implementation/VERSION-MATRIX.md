# 玄武云盾 V0.1 Implementation Version Matrix

> 状态：Implementation Parameter Freeze 模板  
> 依据：`10-decisions/ARCHITECTURE-BASELINE-V0.1.md` 及 ADR-002～008  
> 规则：所有 `TBD` 必须在 TASK-001～TASK-005 完成前冻结；未冻结不得安装对应组件。

## 1. Version Freeze Rules

1. Kubernetes 版本必须来自实施时 KubeSphere 官方兼容矩阵，禁止猜测。
2. 所有组件使用固定版本和镜像 digest，禁止 `latest`。
3. 版本变更必须记录 Implementation Decision、影响、验证和回滚。
4. 版本矩阵提交 Git 后，TASK-011 及后续安装任务才可执行。

## 2. Required Matrix

| 组件 | 选定版本 | 来源/兼容依据 | 镜像或包校验和 | Owner | 状态 |
|---|---|---|---|---|---|
| OS | TBD | 企业 LTS 与 KubeSphere 兼容要求 | TBD | Platform Owner | BLOCKED until freeze |
| Kernel | TBD | Calico/containerd/Kubernetes requirements | TBD | Platform Owner | BLOCKED until freeze |
| containerd | TBD | Kubernetes compatibility | TBD | Platform Owner | BLOCKED until freeze |
| Kubernetes | TBD | KubeSphere official compatibility matrix | TBD | Platform Owner | BLOCKED until freeze |
| KubeSphere | TBD | Baseline + official compatibility matrix | TBD | Platform Owner | BLOCKED until freeze |
| Calico | TBD | K8s/CNI compatibility | TBD | Network Owner | BLOCKED until freeze |
| local-path | TBD | Kubernetes minor compatibility | TBD | Platform Owner | BLOCKED until freeze |
| Harbor | TBD | Independent VM and offline installation package | TBD | Registry Owner | BLOCKED until freeze |
| Ingress Controller | TBD | Baseline admin UI requirement | TBD | Platform Owner | BLOCKED until freeze |
| Prometheus | TBD | KubeSphere stack compatibility | TBD | Observability Owner | BLOCKED until freeze |
| Grafana | TBD | KubeSphere stack compatibility | TBD | Observability Owner | BLOCKED until freeze |
| Alertmanager | TBD | Prometheus stack compatibility | TBD | Observability Owner | BLOCKED until freeze |
| Fluent Bit | TBD | Kubernetes log format/runtime | TBD | Observability Owner | BLOCKED until freeze |
| Loki | TBD | Fluent Bit output compatibility | TBD | Observability Owner | BLOCKED until freeze |
| ClickHouse | TBD | Existing DA-SOC compatibility | TBD | DA-SOC Owner | BLOCKED until freeze |
| n8n | 2.15.0 or approved current existing version | Existing DA-SOC workflow compatibility | TBD | DA-SOC Owner | VERIFY |
| render/archive | `da-soc-render:0.1` | Existing image and API contract | TBD | DA-SOC Owner | VERIFY |
| Agent image | TBD | Job runtime and tool set | TBD | AIOps Owner | BLOCKED until freeze |
| SOPS/age | TBD | Secret encryption/recovery method | TBD | Security Owner | BLOCKED until freeze |

## 3. Freeze Evidence

- Compatibility matrix URL/document snapshot.
- Version and digest list.
- Offline package list.
- Signature/checksum verification output.
- Approval by Platform Owner, Security Owner and DA-SOC Owner.
- Git commit containing this file and associated manifests.