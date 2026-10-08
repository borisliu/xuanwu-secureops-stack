# DA-SOC V0.1 Cutover Checklist

> 适用：ECS → Kubernetes `da-soc` 生产切换。  
> 原则：验证双跑、生产单活；任何未勾选项都禁止切换。

## 1. Change Record

- Change ID:
- Planned window:
- Business Owner:
- Platform Owner:
- DA-SOC Owner:
- Approver:
- Rollback Owner:
- K8s workflow commit/digest:
- ECS workflow commit/digest:

## 2. Pre-Cutover Gates

- [ ] Kubernetes/KubeSphere/Calico 健康。
- [ ] `da-soc` Namespace、Quota、Secret、NetworkPolicy、PVC 健康。
- [ ] ClickHouse 健康，历史数据已恢复并校验。
- [ ] render/archive 健康，`/archive` 与 `/render` 测试通过。
- [ ] n8n workflow 来自 Git artifact，UI drift 检查通过。
- [ ] 当天、近 6 周、近 6 月结果与基准一致。
- [ ] 图片与基准一致，`null`/“暂无数据”语义一致。
- [ ] archive 失败时不入库、不出图、不发送测试通过。
- [ ] 生产邮箱读取范围、不标已读、删除/修改保护测试通过。
- [ ] 测试 DingTalk 群发送成功，生产/测试凭据隔离。
- [ ] ClickHouse、raw、n8n、render、Harbor 备份成功。
- [ ] Control Plane、ClickHouse、raw、Harbor restore drill 证据已归档。
- [ ] 监控、日志、告警和运维 DingTalk 正常。
- [ ] 单活消费者证明完成。
- [ ] ECS 停止/启动和状态恢复 Runbook 已演练。
- [ ] 回退 Owner、审批人和生产窗口已确认。

## 3. Cutover Steps

1. 创建 Change/Task 并冻结无关变更。
2. 确认 ECS n8n trigger disabled，等待活动执行数为零。
3. 记录最后处理边界、workflow digest、数据快照和发送记录。
4. 确认 K8s n8n trigger disabled 状态可控。
5. 启用 K8s `da-soc` n8n 生产 trigger。
6. 观察第一轮完整流程：邮箱 → archive → ClickHouse → SQL → render → DingTalk。
7. 核对日志、指标、Task、数据和发送结果。
8. 在观察窗口内保持 ECS 可启动但不消费邮箱。

## 4. Immediate Stop Triggers

- 数据、图片、SQL 结果或空数据语义不一致。
- `/archive` 或 `/render` 行为异常。
- 邮箱 Mark as Read、删除、修改或读取范围异常。
- DingTalk 目标、凭据或发送状态异常。
- Backup/Restore/Monitoring/Logging/Audit 任何关键项失败。
- ECS 与 K8s 同时消费或存在无法解释的重复输出。

## 5. Sign-Off

- Platform Owner:
- Security Owner:
- DA-SOC Owner:
- Business Owner:
- Final decision: GO / NO-GO
- Evidence directory/commit: