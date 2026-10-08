# DA-SOC V0.1 Rollback Plan

## 1. Rollback Principles

- 单活消费者优先；禁止盲目同时启动 ECS 和 K8s n8n。
- 先冻结消费和发送，再保存证据，最后恢复。
- 技术回退和业务回退分开处理。
- 回退必须由指定 Rollback Owner 执行，L2 变更需要人工批准。

## 2. Trigger

立即触发评估：数据不一致、图片异常、`/archive` 异常、ClickHouse 异常、workflow 异常、邮箱读取异常、DingTalk 异常、数据丢失、备份异常、严重网络异常、不可解释行为、两个消费者同时运行。

## 3. Technical Rollback Before First Production Output

1. 停止 K8s n8n production trigger。
2. 确认无活动执行，保留 Pod、Job、n8n execution、Loki 和 Audit 证据。
3. 验证 ECS workflow、凭据、ClickHouse/原始数据状态。
4. 启动 ECS n8n，保持 K8s trigger disabled。
5. 执行完整健康检查和测试群验证。
6. 建立 Incident/Task，记录原因和重新切换前置条件。

## 4. Business Rollback After Output

1. 立即冻结两个环境的消费和发送。
2. 记录已处理 Message-ID/UID、业务日期、ClickHouse 写入、图片和 DingTalk 发送状态。
3. 校验幂等/去重状态，防止 ECS 重复入库或发送。
4. 由 DA-SOC Owner 与 Business Owner 决定：K8s 内修复、从备份恢复，或切回 ECS。
5. 只启动一个消费者。
6. 重新执行业务验证并取得人工签字。

## 5. Verification

- 邮箱未读/原始状态不变。
- ClickHouse 数据和 SQL 结果可解释。
- archive 失败门禁仍成立。
- render 图片和 DingTalk 目标正确。
- 监控/日志/备份/审计正常。
- 回退 Task/Incident 有完整证据。