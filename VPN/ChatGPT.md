最适合实现的是：**NetBird，其次 Headscale，传统 VPN 里选 SoftEther。**

## 推荐排序

| 方案                | 钉钉扫码/国家网络ID扫码 | 改造难度 |     推荐度 |
| ------------------- | ----------------------: | -------: | ---------: |
| **NetBird**   |                  最适合 |       中 | ⭐⭐⭐⭐⭐ |
| **Headscale** |                    可以 |     中高 |   ⭐⭐⭐⭐ |
| **SoftEther** |                间接实现 |     中高 |     ⭐⭐⭐ |
| pfSense / OPNsense  |        不太适合深度定制 |       高 |       ⭐⭐ |
| OpenZiti            |            能力强但复杂 |       高 |     ⭐⭐⭐ |

## 为什么首选 NetBird

NetBird 自托管版支持接入外部身份提供方，核心协议是  **OIDC / OpenID Connect** ，可以对接 Keycloak、Authentik、Okta、Auth0 等身份系统。([NetBird 文档](https://docs.netbird.io/selfhosted/identity-providers?utm_source=chatgpt.com "Authentication and Identity Providers (IdPs)"))

你的目标可以做成：

> 钉钉扫码 / 国家网络ID扫码
> → 自建统一身份网关
> → 转成 OIDC
> → NetBird 登录
> → 分配 VPN / 零信任访问权限

这条路最工程化，也最适合后续扩展审计、审批、分组、ACL。

## Headscale 也可以

Headscale 也支持 OIDC 登录，而且是通用 OIDC 方式。([Headscale](https://headscale.net/stable/ref/oidc/?utm_source=chatgpt.com "OpenID Connect"))
但它更像“自建 Tailscale 控制面”，管理 UI、企业权限、用户生命周期能力不如 NetBird 完整。

适合你自己实验，不如 NetBird 适合企业化推广。

## SoftEther 的方式

SoftEther 支持 RADIUS、AD、证书等认证。([SoftEther](https://www.softether.org/4-docs/1-manual/2._SoftEther_VPN_Essential_Architecture/2.2_User_Authentication?utm_source=chatgpt.com "2.2 User Authentication - SoftEther VPN Project"))
你可以做：

> 钉钉/国家网络ID扫码
> → 自建认证服务
> → RADIUS Server
> → SoftEther VPN

但这属于“把扫码结果转换成传统 VPN 认证”，能做，但体验和权限模型不如 NetBird/Headscale 现代。

## 我的建议

你要做的是“基于身份的安全接入”，不是单纯 VPN。

所以建议选：

**P0：NetBird + Keycloak/Authentik + 钉钉扫码 + 国家网络ID认证适配器**

架构是：

```text
用户扫码
  ↓
钉钉 / 国家网络身份认证
  ↓
统一身份适配层
  ↓
Keycloak / Authentik
  ↓ OIDC
NetBird
  ↓
内网系统 / RDP / OpenClaw / 运维入口
```

结论：**选 NetBird 最合适。Headscale 适合轻量实验，SoftEther 适合传统 VPN 兼容。**
