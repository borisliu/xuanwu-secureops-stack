# 玄武云盾 V0.1 Restore Drill Plan

> 原则：没有真实恢复演练的备份不算完成。演练使用隔离资源，不直接破坏唯一生产实例。

## 1. Objectives

验证 etcd、Kubernetes 资源、KubeSphere 关键配置、ClickHouse、raw archive、n8n workflow/state、render/archive、Harbor 和加密 Secret 的可恢复性，并验证 DA-SOC 业务结果。

## 2. Drill D1 — Control Plane

- 前置：已验证 etcd snapshot、版本矩阵、空闲恢复窗口。
- 动作：在隔离 VM/受控环境重建 Control Plane，恢复 etcd 或 Git 定义态。
- 验证：API、节点、KubeSphere、RBAC、Audit、NetworkPolicy、监控正常。
- 证据：快照 hash、命令记录、时间线、RTO、验证结果。
- 回退：不触碰生产 Control Plane；恢复环境销毁并记录差异。

## 3. Drill D2 — DA-SOC Data

- 前置：ClickHouse BACKUP、raw archive、n8n 配置和 Secret 加密备份。
- 动作：在隔离 `da-soc-restore` 环境恢复 ClickHouse、raw、n8n/render 配置。
- 验证：当天、近 6 周、近 6 月 SQL；图片；`null` 语义；archive 失败门禁。
- 证据：表结构/行数/日期范围、文件校验和、图片比对、workflow digest。
- 回退：隔离环境删除，不影响 `da-soc`。

## 4. Drill D3 — Harbor

- 前置：Harbor 配置、数据、关键镜像 tar 和 CA 备份。
- 动作：在独立 VM 恢复 Harbor，使用 Worker 拉取 DA-SOC 镜像。
- 验证：HTTPS、权限、digest、Kubernetes pull、扫描结果可用。
- 证据：镜像 digest、pull 日志、恢复时间。

## 5. Drill D4 — POP3 Replay

- 前置：合法测试/历史回放数据，禁止直接读取或修改生产邮箱状态。
- 动作：按一次性回补路径回放到隔离 ClickHouse。
- 验证：数据结果和日报图可复现，不发送业务消息。
- 证据：输入 manifest、SQL 结果、图片 hash。

## 6. Exit Criteria

所有 Drill 完成、RTO/RPO 达标、证据提交 Git、Owner 签字；任一关键恢复失败则 TASK-061～TASK-064 不通过，禁止生产切换。