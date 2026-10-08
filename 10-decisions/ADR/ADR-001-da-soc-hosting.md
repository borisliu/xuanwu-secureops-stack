# ADR-001: DA-SOC v0.1 Hosting Strategy

- 状态：Accepted
- 日期：2026-10-08
- 影响范围：DA-SOC、Kubernetes、邮箱、DingTalk、备份、迁移和回退

## Decision

玄武云盾 V0.1 必须实际承载 DA-SOC v0.1。正式生产承载位置为 Kubernetes 的 `da-soc` Namespace；ECS 只保留为迁移期回退源，不作为 V0.1 最终生产架构。

组件裁决：

| 组件 | 决策 |
|---|---|
| n8n | 进入 `da-soc`，单副本、单活生产工作流 |
| ClickHouse | 进入 `da-soc`，单副本 StatefulSet + Local PV |
| render/archive | 进入 `da-soc`，Deployment + ClusterIP |
| raw archive | 进入 `da-soc` 独立 PVC |
| SQL/workflow | Git 为 Source of Truth，n8n 只运行构建结果 |
| Secrets | K8s Secret + 加密备份，明文不得进 Git |
| ECS | 回退源和短期保留实例，不与新 n8n 同时消费生产邮箱 |

## Migration

1. 建立 Kubernetes、Harbor、备份和 `da-soc` 安全边界。
2. 镜像离线导入 Harbor，固定 digest。
3. 在临时 `da-soc-validate` 使用测试邮箱/回放数据部署完整栈。
4. 使用 ClickHouse BACKUP/RESTORE 和文件备份导入历史数据。
5. 从 Git 构建并导入 n8n workflow，禁止 UI 改 SQL。
6. 验证当天、近 6 周、近 6 月、图片、archive 失败门禁和 DingTalk。
7. 停止 ECS n8n，确认无活动执行，启动 K8s n8n。
8. 观察完整日报周期，ECS 保持可启动但禁止消费。

## Parallel Run Rule

允许验证双跑，不允许生产双消费者。两个 n8n 不得同时读取生产邮箱、写入同一生产数据、发送同一日报或修改同一状态。验证使用独立测试邮箱和数据回放，不使用生产发送凭据。

## Cutover Gate

数据一致、图片一致、业务流程一致、邮箱不标已读、DingTalk 行为一致、备份成功、恢复测试成功、监控/日志/告警正常、回退路径成功、单活消费者验证成功，全部满足后才允许切换。

## Rollback

首次生产输出前发现启动/网络/配置问题，停止 K8s n8n，恢复 ECS。生产输出后发现数据差异、archive/render 异常、邮件行为异常、DingTalk 异常、备份失败或不可解释数据缺失时，先冻结消费和发送，保存证据并由业务负责人批准回退；禁止盲目同时启动两个 n8n。

## Rationale

项目负责人已明确 ADR-001。只纳管 ECS 或把迁移推迟到 V0.2 不符合项目要求；通过验证双跑和生产单活可以同时满足“实际承载”和“确定性业务不被破坏”。