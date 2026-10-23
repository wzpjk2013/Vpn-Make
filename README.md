# Vpn-Make

一套**可复现**的自建代理服务部署方案：Xray + WARP + 带宽限速 + 订阅服务 + Web 管理面板，
以及一份写给 **AI Agent 也能照着执行**的完整部署手册。

## 核心交付物

**[服务器配置.md](服务器配置.md)** —— 单文件、自包含的部署手册。约 8300 行，包含：

| 内容 | 位置 | 说明 |
|---|---|---|
| 部署步骤 | §3 – §16 | 从空白 Ubuntu 24.04 到可用服务的每一步、命令、预期输出、验证方法 |
| AI Agent 部署规程 | §0.1 | 12 项必问清单、9 道验证闸门、7 条失败红线、可跳过组件 |
| 全部配套源码 | 附录 D | Web 面板、带宽整形、安装/回滚脚本、84 项本地自测（逐行收录） |
| 完整订阅模板 | 附录 E | 15 个节点 + 14 个策略组 + 38 个 mrs 规则集 + 1129 条分流规则（1958 行） |

**文档是自包含的**——不需要另外去拿任何文件，也不需要访问原始服务器。

另有 **[WARP实现.md](WARP实现.md)** —— WARP 出口的深入实现文档：七个环节的端到端链路、
mark 机制（为什么 bit0 留给 WARP、为什么必须写 `1/1` 掩码）、与带宽限速的交互、
响应式握手的证据链、三类静默故障与看门狗、十条排查矩阵、从零重建与换 Endpoint。
属于**补充阅读**，部署不需要它，但排障时很有用。

## 架构

```
客户端 ──► Cloudflare（橙云） ──► 源站 Xray :443 / :2053 ──┬─► 直连出口
                                                          └─► WARP 出口
管理面 ──► SSH（可选经跳板机）
订阅面 ──► Cloudflare ──► 面板 :2083（订阅亦由此分发）
```

三个平面必须分清，这是整套架构的核心约束：

| 平面 | 走什么 | 用途 | 关键约束 |
|---|---|---|---|
| 管理面 | SSH | 运维、改配置 | 跳板机 IP 必须进 fail2ban 白名单 |
| 数据面 | 代理流量（经 Cloudflare） | 实际翻墙 | **绝不经过跳板机** |
| 订阅面 | HTTPS（经 Cloudflare） | 客户端取节点 | 同上 |

## 组件

| 组件 | 作用 |
|---|---|
| **Xray** | VLESS + WebSocket + TLS，双入站（直连 443 / WARP 2053） |
| **WireGuard** | WARP 出站，用于换一个干净的出口 IP |
| **Web 面板** | 多 UUID / 多订阅链接 / 每条独立限速 / 配额 / 到期 / 流量统计 / 服务器实时带宽 |
| **带宽限速** | 每条订阅独立 Mbps；上传走 HTB，下载走 ifb + conntrack 标记，双向都限 |
| **订阅分发** | 由面板在 :2083 提供，gzip 压缩，标准 `Subscription-Userinfo` 头 |
| **证书续签** | Let's Encrypt DNS-01（Cloudflare API），certbot 自动续期，签发后自动部署并重载 Xray 与面板 |
| **fail2ban** | SSH 爆破防护，ipset 模式（封几百个 IP 规则数恒定） |
| **分流规则** | mihomo `rule-providers` + mrs 规则集，38 个分类；境外域名由 `geolocation-!cn` 兜底 |

## 分流与 DNS

订阅模板的分流不是手写列表，而是 **mihomo 原生规则集**：38 个 `format: mrs` 的远端
规则集（[MetaCubeX/meta-rules-dat](https://github.com/MetaCubeX/meta-rules-dat)），
全部经 `proxy:` 字段走代理下载，绕开 `cdn.jsdelivr.net` 在国内的抖动。

- 分类交给上游持续维护，规则条数 9582 → 1129，订阅体积 471 KB → 79.5 KB（gzip 后 10.4 KB）
- `apple@cn` / `microsoft@cn` / `steam@cn` 等**中国区子集必须先匹配直连**，
  顺序颠倒就会把国内服务推上代理
- `google@cn` 已于 2026-10-23 移除：该集合把 `google.cn` / `googleapis.cn` / `gstatic.cn`
  整体送去直连，与「Google 全部过代理」的预期冲突。现在 Google 域名统一落到
  `RULE-SET,google → 🚀 节点选择`
- 兜底链：精细分类 → `cn` 直连 → `geolocation-!cn` 代理 → `GEOIP,CN` 直连 → `MATCH`

**DNS 做两层防泄漏**（细节见 §E.2）：

- `fake-ip-filter-mode: rule` —— 国内域名下发 `real-ip`（直连正确、`GEOIP,CN` 可用），
  其余下发 `fake-ip`，客户端拿不到境外域名的真实 IP。没有这个开关就只能二选一
- `nameserver-policy` 用 `rule-set:` 前缀**直接引用本文件的 38 个 provider** 做分域，
  境外查询写成 `tls://8.8.8.8:853#🚀 节点选择`，DNS 请求本身从代理出去

这套配置是渐进增强的：不开 TUN 时第一层不介入，但第二层照常工作，污染已修好；
开了 TUN 才彻底关掉客户端侧的泄漏面。

细节见 §E.1 / §E.2。

## 快速开始

1. **准备**：一台 Ubuntu 24.04（KVM/独立机）、一个托管在 Cloudflare 的域名、一个 CF API Token
2. **先读 §0.1**（尤其 A 节的 12 项必问清单）
3. **填 §2.3 的变量表**，然后从 §3 开始照做

如果你是人，直接按章节顺序走；如果你是 AI Agent，§0.1 规定了分阶段执行与验证闸门。

## 安全与隐私

本仓库**只包含脱敏后的文档**。所有 IP、域名、密码、UUID、密钥、Token 均已替换为占位符：

| 占位符 | 含义 |
|---|---|
| `ORIGIN_IP` / `ORIGIN_GW` | 源站公网 IP / 网关 |
| `VPN_HOST` / `DOMAIN` | 代理子域名 / 主域名 |
| `UUID_CF` / `UUID_WARP` | 两个入口的用户 UUID |
| `WS_PATH_CF` / `WS_PATH_WARP` | 两个入口的 WebSocket 路径 |
| `SUB_TOKEN` | 订阅服务的访问令牌 |
| `CF_API_TOKEN` | Cloudflare API Token |

**请勿**把填好真实值的副本提交回本仓库。

## 免责声明

本项目仅供**学习与技术研究**使用。请遵守你所在地区的法律法规以及服务商的条款。
使用后果由使用者自行承担。

## 许可

[MIT License](LICENSE) © 2026 wzpjk2013

可自由使用、修改、分发、商用，保留版权声明与许可原文即可。
