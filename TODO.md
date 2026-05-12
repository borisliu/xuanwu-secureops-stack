# 玄武云盾 · P0 执行清单（企业 VPN 开源替代测试）

按顺序勾选。详细用例、通过标准与评分模板见 [`VPN/README.md`](VPN/README.md)。

---

## 使用说明

- 每完成一步，将 `[ ]` 改为 `[x]`（或在你使用的任务工具中打勾）。
- 每个产品在 `VPN/<产品>/` 下维护四份文档：`install.md`、`config.md`、`test-cases.md`、`test-result.md`，证据进 `screenshots/` 或附录日志路径。
- 遇到阻塞：在 `test-result.md` 记录现象、版本号、规避办法，仍算有效进展。

---

## 阶段 A：启动与脚手架

- [ ] **A1** 已通读 [`VPN/README.md`](VPN/README.md) 全文（重点：§1 目标、§2 评价维度、§3 环境与 **§3.3 部署形态**、§5 用例、§6 各产品重点、§8 Cursor 工作要求）。
- [ ] **A2** 已确认测试资源：虚拟化平台（Proxmox / ESXi / Hyper-V / KVM 等）、至少 3 台 VM 配额（SoftEther、OPNsense、NetBird 管理面可分时复用或按需分配）、Windows/macOS/Android 等客户端。
- [ ] **A3** 已画出实际拓扑（外网 / DMZ / 内网测试区 / 客户端），与 [`VPN/README.md`](VPN/README.md) §3.2 对照，标注 IP 与网段。
- [ ] **A4** 已创建目录与空文档（若尚不存在）：
  - [ ] `VPN/softether/`：`install.md`、`config.md`、`test-cases.md`、`test-result.md`、`screenshots/`
  - [ ] `VPN/opnsense/`：同上
  - [ ] `VPN/netbird/`：同上
  - [ ] `VPN/docs/`（可选）：`00-requirements.md`、`01-test-environment.md`、`02-product-comparison.md`
  - [ ] `VPN/report/`：`scoring-table.xlsx`（或等价表格）、`final-report.md`
- [ ] **A5**（可选）已准备日志或身份联调环境：ELK / Wazuh / Loki 其一；Keycloak 或 Authentik（用于 OIDC 类用例）。

---

## 阶段 B：第一轮 — SoftEther VPN

部署基线：x86 **Linux VM + 原生安装**（见 `VPN/README.md` §3.3）。

- [ ] **B1** 在 `install.md` 记录：发行版与版本、内核、安装包来源、安装命令与版本号。
- [ ] **B2** 完成单节点部署（对应 TC-001），管理界面或监听端口可访问；在 `test-result.md` 记证据。
- [ ] **B3** 在 `config.md` 记录：监听协议、Hub、用户/组、与内网测试区的路由或桥接方式。
- [ ] **B4** 按 `VPN/README.md` §5.2 至少完成 Windows + 一种移动或桌面客户端接入（TC-101～TC-105 中与你环境相关的子集）。
- [ ] **B5** 按 §5.4 完成至少：内网 Web 或 SSH（TC-301 / TC-302）、固定 VPN IP 或等价能力验证（TC-304，若不支持则记为结论与风险）。
- [ ] **B6** 按 §5.5 抽样：登录/连接日志是否可审计、是否可 Syslog 外送（TC-401～TC-405 子集）。
- [ ] **B7** 按 §5.3 评估 LDAP/RADIUS/OIDC/钉钉链路（不必全部打通，无则写清缺口与改造路径）。
- [ ] **B8** 按 §5.7 做 iperf3 或等效吞吐与延迟记录（TC-601～TC-603）；资源占用简记（TC-604）。
- [ ] **B9** 在 `test-result.md` 汇总：已通过用例表、未测项、阻塞项；更新 `VPN/README.md` §7 评分表中 **SoftEther** 列草稿分。

---

## 阶段 C：第二轮 — OPNsense

部署基线：**独立 VM**（双网卡或多网段更佳），勿按 Linux 容器「整机」方式部署。

- [ ] **C1** 在 `install.md` 记录：镜像版本、虚拟硬件（CPU/内存/磁盘/网卡）、安装方式。
- [ ] **C2** 完成 WAN/LAN（或测试等价区）对接，Web 管理可登录；基础防火墙策略说明写入 `config.md`。
- [ ] **C3** 启用至少一种远程接入：WireGuard 或 OpenVPN 或 IPsec（与现有能力匹配），客户端接入步骤写入 `install.md` / `config.md`。
- [ ] **C4** 执行与 B4～B8 对等的接入、内网访问、日志外送、身份（若有）与性能抽样，结果写入 `test-result.md`。
- [ ] **C5** 记录 IDS/IPS 或插件是否纳入本轮（纳入则写配置与负载；不纳入则标为后续）。
- [ ] **C6** 更新评分表中 **OPNsense** 列草稿分。

---

## 阶段 D：第三轮 — NetBird

部署基线：**Docker Compose**（自托管管理面 + 中继）；需要时可再验证二进制部署作为附录。

- [ ] **D1** 在 `install.md` 记录：Compose 文件版本、镜像 tag、主机 OS、放行端口列表。
- [ ] **D2** 管理面与客户端完成首次组网；在 `config.md` 记录 OIDC（若启用）、Network、Group、路由资源。
- [ ] **D3** 完成至少一种资源访问（SSH / RDP / Web 子集），与 §5.4 对齐；策略（用户/设备/资源）写入 `config.md`。
- [ ] **D4** 审计与日志：连接事件是否可满足 TC-401～TC-405 中的抽样要求。
- [ ] **D5** 性能与稳定性抽样（iperf3、短时并发或长连二选一），写入 `test-result.md`。
- [ ] **D6** 更新评分表中 **NetBird** 列草稿分。

---

## 阶段 E：横向对比与收尾

- [ ] **E1** 填写 `VPN/README.md` §7 评分矩阵三列，按 §7 规则换算加权分。
- [ ] **E2** 完成 `docs/02-product-comparison.md`（或并入 `report/final-report.md` 一章）：对照表含 **产品类型**（VPN 软件 / 网络 OS / 零信任平台）、部署形态、运维成本、审计、HA、与「替代硬件 VPN」目标的匹配度。
- [ ] **E3** 撰写 `report/final-report.md`，结构不少于 `VPN/README.md` §9 所列章节（背景、环境、执行情况、评分、风险、推荐结论、分阶段落地建议）。
- [ ] **E4** 与干系人评审结论：主方案 / 备选方案 / 暂不采纳项；若分场景多选（例如边界用 OPNsense、远程接入用 NetBird），写入「组合架构」建议。
- [ ] **E5** 将 P0 结论回写到根目录 [`README.md`](README.md) 的「分阶段推进」表（例如把 P0 标为已完成，并增加一行「选型结论摘要」链接到 `VPN/report/final-report.md`）。

---

## 阶段 F：移交 P1（可选前置）

仅在 P0 报告已定稿后执行：

- [ ] **F1** 根据结论列出 P1 依赖：例如选定边界平台（OPNsense 是否独立采购/部署）、JumpServer 对接网段、证书与域名。
- [ ] **F2** 在仓库或 `VPN/docs/` 中创建 P1 草案里程碑（无需本清单展开）。

---

**完成标准**：`report/final-report.md` 已定稿，且三款产品各自 `test-result.md` 可追溯；评分表与风险、推荐结论一致。
