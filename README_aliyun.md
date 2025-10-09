# 🐢 玄武云盾 · 阿里云安全原生版  
**Xuanwu SecureOps Stack · Apsara Edition**

> 基于阿里云原生安全服务打造的低成本、安全可视化、智能化私有云安全运营平台（SOC + 数据分析一体化）。

---

## 🚀 项目简介

**玄武云盾（Xuanwu SecureOps Stack）** 是一个开源安全运营与数据分析一体化平台。  
此版本基于 **阿里云安全生态** 实现全托管部署，无需自建 Wazuh、雷石、Loki、Doris 等复杂组件。

系统通过阿里云 **WAF + VPN + 堡垒机 + 安全中心 + 态势感知 + 日志服务 + ADB + Dify + 钉钉/宜搭**  
实现了从数据采集 → 日志分析 → 智能报告 → 可视化协作的完整闭环。

---

## 🧱 系统架构

```mermaid
flowchart TD

U[安全分析员/运维人员] -->|钉钉SSO| DING[钉钉/宜搭 安全运营看板]

subgraph 访问安全层
    WAF[阿里云WAF] --> VPN[VPN 网关]
    VPN --> BH[阿里云堡垒机 Bastionhost]
end

BH --> ACK[阿里云ACK(托管K8s集群)]

subgraph ACK 集群
    QL[青龙 定时任务/采集脚本]
    DF[Dify AI分析工作流]
    OUT[出站代理 (调用宜搭API)]
end

ACK --> NAS[(NAS/云盘)]
ACK --> OSS[(OSS归档)]

subgraph 托管安全服务
    EDR[阿里云安全中心(EDR)]
    SAS[态势感知(Security Center)]
    SLS[日志服务SLS]
    ADB[(AnalyticDB for MySQL)]
end

%% 数据流
QL -->|采集数据写入| ADB
SLS -->|日志加工/投递| ADB
EDR --> SLS
SAS --> SLS
ADB --> DF --> OUT --> DING
```

---

## 🧩 组件清单

| 层级 | 服务 | 功能定位 |
|------|------|-----------|
| **访问安全层** | WAF + VPN 网关 + 堡垒机 | 网络防护、远程接入、运维审计 |
| **容器运行层** | 阿里云 ACK | 承载自定义服务（青龙、Dify、代理） |
| **采集层** | 青龙 | 定时采集数据（API/数据库/系统） |
| **数据分析层** | AnalyticDB for MySQL | 高性能 OLAP 分析引擎 |
| **日志与安全层** | 日志服务 SLS | 统一日志收集、分析、投递 |
| **威胁检测层** | 安全中心 (EDR) + 态势感知 (SAS) | 主机防护、漏洞检测、威胁情报 |
| **AI分析层** | Dify | 智能分析、自然语言报告 |
| **展示协作层** | 宜搭 / 钉钉 | 告警可视化、工单流转 |

---

## 🧠 数据流闭环

1️⃣ **采集层**  
- 青龙采集业务与安全数据写入 ADB。  
- EDR 与 SAS 采集主机与云资源安全事件。  

2️⃣ **日志汇聚层**  
- 所有日志进入 SLS → 做加工与清洗 → 投递到 ADB。  

3️⃣ **智能分析层**  
- Dify 通过 SQL 查询 ADB 数据 → 调用大模型生成报告。  

4️⃣ **展示协作层**  
- Dify 输出结果推送到 宜搭 安全看板。  
- 钉钉机器人推送告警与报告。  

---

## ⚙️ 部署步骤

### Step 1: 云资源准备
- 开通阿里云服务：ACK、NAS、OSS、SLS、ADB、WAF、VPN、堡垒机、安全中心（含态势感知）；
- 配置统一身份体系（RAM + 钉钉SSO）。

### Step 2: 部署容器
```bash
# 在阿里云 ACK 中部署核心服务
helm install qinglong ./charts/qinglong
helm install dify ./charts/dify
helm install outbound ./charts/outbound
```

### Step 3: 日志接入
- 在 SLS 中创建 Project / Logstore；  
- 启用 ACK 日志采集（Logtail DaemonSet）；  
- 建立 SLS 投递任务 → ADB 表。

### Step 4: 配置 AnalyticDB
```sql
CREATE TABLE sec_events (
  id BIGINT AUTO_INCREMENT PRIMARY KEY,
  ts DATETIME,
  src VARCHAR(64),
  type VARCHAR(64),
  severity VARCHAR(16),
  msg TEXT
);
```

### Step 5: Dify 连接 ADB
- 配置 MySQL 数据源（ADB 连接串）；
- 编排 SQL → LLM → 输出卡片 的工作流；
- 输出数据写入宜搭表或钉钉机器人。

---

## 🔒 安全与运维规范

| 领域 | 策略 |
|------|-------|
| **访问控制** | WAF、VPN、堡垒机三级防护 |
| **主机防护** | EDR 统一威胁检测与修复 |
| **日志合规** | SLS 集中存储，OSS 归档 |
| **数据安全** | ADB 持久加密，NAS/OSS 快照备份 |
| **AI安全** | Dify 仅通过内网访问 ADB，不暴露公网 |
| **身份认证** | 全系统钉钉扫码SSO |
| **自动升级** | ACK + Helm + ADB 托管更新 |
| **告警通知** | SLS/SAS → 钉钉机器人/宜搭看板 |

---

## 🧩 优势总结

| 维度 | 优势 |
|------|------|
| 🧱 架构轻量 | 无需自建 Wazuh/雷石/Doris |
| 🔒 安全可靠 | 全托管安全中心 + WAF + VPN |
| 🧠 智能分析 | ADB + Dify 自动生成安全报告 |
| 📊 可视化 | 宜搭/钉钉 工单+看板一体化 |
| ⚙️ 易维护 | 云上服务运维自动化 |
| 🔁 可迁移 | 保留 K8s 架构，可平滑回私有云 |

---

## 📈 云上部署最小配置

| 模块 | 规格建议 |
|------|-----------|
| ACK 集群 | 3节点 8C32G |
| ADB MySQL | 按量型 4C16G 起 |
| SLS | 1 Project + 多 Logstore |
| NAS 存储 | 500GB 起步 |
| EDR + SAS | 默认启用全局检测 |
| Dify | 2C4G 容器即可 |
| 宜搭/钉钉 | SaaS 集成 |

---

## 🧩 一句话总结

> **玄武云盾 · 阿里云安全原生版（Xuanwu SecureOps Stack · Apsara Edition）**  
> 以阿里云 **WAF + VPN + 堡垒机 + EDR + 态势感知 + SLS + ADB + Dify + 钉钉/宜搭**  
> 打造一个 **零自建、零暴露、智能化、安全可视化** 的云上安全运营与分析平台。

---

## 📜 许可证
- License: **Mulan PSL v2**
- 可自由商用、修改、再发布，需保留原始声明。

---

## 🤝 参与共建
- Fork & PR 贡献模板与工作流  
- 欢迎在 GitHub Discussions 分享安全场景与最佳实践  

---

**玄武云盾 Xuanwu SecureOps Stack**  
> 🐢 安全为基 · 智能为核 · 以低成本构建可持续演进的云上安全平台。
