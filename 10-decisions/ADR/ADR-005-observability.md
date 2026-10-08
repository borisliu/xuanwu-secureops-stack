# ADR-005: 单一 Observability 栈

- 状态：Accepted
- 日期：2026-10-08

## Decision

V0.1 统一使用：

```text
Prometheus → Grafana → Alertmanager → n8n/DingTalk
Fluent Bit → Loki → Grafana/Agent Job
```

## Scope

监控节点、Kubernetes/API/etcd、KubeSphere、Pod/PVC、Harbor、备份、证书、DA-SOC 的 n8n/render/archive/ClickHouse、archive 失败、日报时间、数据新鲜度和 DingTalk 发送结果。

日志采集节点 journald、容器 stdout/stderr、Kubernetes Audit、KubeSphere 关键审计、DA-SOC 结构化日志和 Agent Job 日志。敏感邮箱正文、Secret、Token 和不必要业务数据必须脱敏或不采集。

## Notification

平台告警进入独立玄武云盾运维 DingTalk 群；DA-SOC 业务日报使用独立业务凭据和目标群。n8n 负责传输和通知，不替代 Agent 做根因判断。

## Rejected

不引入 Elasticsearch/ELK、Promtail、第二套日志系统、独立 SIEM 或多套监控，避免 V0.1 的资源和恢复面膨胀。