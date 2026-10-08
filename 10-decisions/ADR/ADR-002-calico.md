# ADR-002: CNI 采用 Calico

- 状态：Accepted
- 日期：2026-10-08

## Decision

V0.1 使用 Calico 作为 Kubernetes CNI，并启用 NetworkPolicy。只使用基础网络和策略能力，不引入 Hubble、Tetragon、Service Mesh 或复杂 eBPF 运行时安全。

## Rationale

Calico 满足 `da-soc` default-deny、跨 Namespace 隔离和外部出口约束；组件和排障面适合当前单集群规模，也更容易被 Runbook 和 AI Agent 解释、审计和回滚。

## Required Controls

- `da-soc` 默认拒绝 ingress/egress。
- 显式放行 DNS、n8n→render、业务→ClickHouse、监控抓取和必要外部出口。
- API Server、etcd、Kubelet 和管理面不可从业务 Pod 访问。
- 所有策略进入 Git，变更先在验证 Namespace 测试。
- 通过正向和反向连通性测试验收。

## Deferred

Cilium/Hubble/Tetragon、Service Mesh 和细粒度 egress gateway 延后到 V0.2/V0.3，以真实可观测和安全需求触发。