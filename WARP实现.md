# WARP 出口实现文档

> 本文档描述本项目中 Cloudflare WARP 出口的**完整实现**：它在架构中的位置、
> 端到端链路、每个参数的含义、mark 机制、与带宽限速的交互、三类静默故障的成因
> 与自愈机制、从零重建步骤、以及故障排查矩阵。
>
> 部署步骤见 [服务器配置.md](服务器配置.md) §6；本文档补充的是**为什么这么配**和**坏了怎么定位**。
>
> 所有参数、命令输出均来自真实运行环境实测，未经验证的内容会明确标注。

---

## 0. 读前须知：三个容易混淆的"节点"

这是本项目最容易误解的地方，先立规矩。

| 说法 | 实际指什么 | 数量 | 变不变 |
|---|---|---|---|
| **WARP 节点**（订阅里的 `WARP-01..05`） | 客户端到源站的 **Cloudflare 入口** | 5 个 | 写死在模板里，不变 |
| **隧道 Endpoint** | 源站到 Cloudflare **WARP 边缘的隧道对端** | 1 个 | 写死在 `warp0.conf`，不变 |
| **出口 IP** | 访问目标网站时对方看到的来源 IP | 1 个（随时变） | **会变** |

**关键结论**：那 5 个 `WARP-0x` 节点不是 5 个 WARP 出口，它们是**5 条通往同一台源站的 Cloudflare 路径**，背后共用**同一条** `warp0` 隧道。

所以在客户端把「WARP 自动选择」从 `WARP-01` 切到 `WARP-05`，**最终出口 IP 是同一个**。切换只改变入口路径（Cloudflare anycast 落点、链路延迟、可用性），不改变出口身份。

实测证据（同一台客户端，分别选不同 WARP 节点）：

```bash
curl -s https://www.cloudflare.com/cdn-cgi/trace | grep -E '^(ip|colo|warp)='
# 无论选 WARP-01 还是 WARP-05，ip 都落在同一个 Cloudflare 出口段
```

---

## 1. WARP 是什么，我们用它做什么

Cloudflare WARP 是 Cloudflare 的免费 WireGuard 出口服务。客户端的 WireGuard 流量进入
Cloudflare 边缘后，由 Cloudflare 以**自己的 IP** 转发到目标网站。

本项目的用法：**把源站 Xray 的一部分出站流量，从源站自己的 IP 换成 Cloudflare 的 IP**。

用途场景：

- 源站 IP 被目标站点风控/拉黑
- 需要访问对源站 IP 有限制的服务
- 想给不同订阅用户不同的出口身份

**没有**用它做的事：

- 不做全局代理（`Table = 100` 保证只有带标记的包进隧道，其余照常走源站 IP）
- 不做出口 IP 固定（WARP 本身不提供这个能力）
- 不做自动分流（订阅模板里 `♻️ WARP自动选择` 组存在，但没有规则指向它，由用户手动选）

---

## 2. 端到端链路：七个环节

```
① 客户端 Clash
      │  vless + ws + tls，port 2053 → Cloudflare anycast IP（WARP-01..05 之一）
      │  ws path = /WS_PATH_WARP，Host = VPN_HOST，servername = VPN_HOST
      ▼
② Cloudflare 边缘（橙色云回源）
      │  按 Host 回源到源站 ORIGIN_IP:2053
      ▼
③ 源站 Xray inbound `warp-in`（0.0.0.0:2053，vless + ws + tls）
      │  routing 规则按用户 email 决定出站：
      ├─ email 前缀 `lim{k}@warp`  → outbound `lim{k}w`（mark = 16k+1）→ 同时被限速
      └─ 其余（如 legacy 用户）    → outbound `warp`（mark = 1）
      ▼
④ outbound `warp` = freedom + sockopt.mark = 1
      │  Xray 在建连时给 socket 打上 SO_MARK = 1
      ▼
⑤ 内核路由
      │  ip rule: `from all fwmark 0x1/0x1 lookup 100`
      │  table 100: `default dev warp0 scope link`
      ▼
⑥ warp0（WireGuard 隧道）
      │  MTU 1280，PersistentKeepalive 25
      │  加密 UDP → 162.159.192.1:2408
      ▼
⑦ Cloudflare WARP 边缘 → 出口 IP（如 104.28.195.192）→ 目标网站
```

**七个环节中，任何一环断掉的表现都不同**，这是排障的抓手（见 §10）。

### 实证输出

```bash
# 环节 ⑤：路由规则与表
ip rule show | grep fwmark
# 32765:	from all fwmark 0x1/0x1 lookup 100

ip route show table 100
# default dev warp0 scope link

# 环节 ⑥：隧道状态
wg show warp0
# interface: warp0
#   public key: S3PdOygpvXcKXRcBQ3Qetzt1sby9L4JUBNSff5l/iVQ=
#   private key: (hidden)
#   listening port: 2121
# peer: bmXOC+F1FxEMF9dyiK2H5/1SUtzH0JuVo51h2wPfgyo=
#   endpoint: 162.159.192.1:2408
#   allowed ips: 0.0.0.0/0
#   latest handshake: 1 second ago
#   transfer: 5.25 MiB received, 1.03 MiB sent
#   persistent keepalive: every 25 seconds

# 环节 ⑦：两条出口对比（都要跑，否则看不出差异）
curl -s --interface warp0 https://www.cloudflare.com/cdn-cgi/trace | grep -E '^(ip|loc|colo|warp)='
# ip=104.28.195.192
# loc=US
# colo=LAX
# warp=on

curl -s https://www.cloudflare.com/cdn-cgi/trace | grep -E '^(ip|loc|colo|warp)='
# ip=ORIGIN_IP
# loc=US
# colo=LAX
# warp=off
```

`warp=on` 是 Cloudflare 自己标识"这个请求来自 WARP"，比看 IP 更权威。
注意 `colo=LAX` 两者相同——说明 WARP 边缘与源站同城，延迟代价极小
（实测 `ping 162.159.192.1` 平均 1.29 ms）。

---

## 3. 隧道配置逐字段解析

`/etc/wireguard/warp0.conf`：

```ini
[Interface]
PrivateKey = <WARP 注册分配的私钥>
Address = 172.16.0.2/32
MTU = 1280
Table = 100
PostUp = ip rule add fwmark 1/1 lookup 100
PreDown = ip rule del fwmark 1/1 lookup 100

[Peer]
PublicKey = bmXOC+F1FxEMF9dyiK2H5/1SUtzH0JuVo51h2wPfgyo=
AllowedIPs = 0.0.0.0/0
Endpoint = 162.159.192.1:2408
PersistentKeepalive = 25
```

| 字段 | 值 | 为什么是它 |
|---|---|---|
| `PrivateKey` | 注册时分配 | WARP 账户身份。**唯一敏感项**，权限必须 `600` |
| `Address` | `172.16.0.2/32` | WARP 分配的内网地址，**不是**公网 IP。`/32` 表示只此一个 |
| `MTU` | `1280` | WARP 官方推荐值。设大（如 1420）会因隧道额外封装而分片，表现为"能连但大文件传不动" |
| `Table` | `100` | **整份配置最关键的一行**。默认 `Table = auto` 会把隧道的默认路由塞进主路由表，服务器立刻失联 |
| `PostUp` | `ip rule add fwmark 1/1 lookup 100` | 只有带 mark 且 `mark & 1 == 1` 的包才查表 100。**必须带 `/1` 掩码**，见 §5 |
| `PreDown` | 对应的 `del` | 隧道停止时清理规则，否则残留规则会让流量进黑洞 |
| `PublicKey` | `bmXOC+F1FxEMF9dyiK2H5/1SUtzH0JuVo51h2wPfgyo=` | **Cloudflare WARP 的固定服务端公钥，全网所有 WARP 用户完全一致**。它公开、不敏感、也不随账户变化 |
| `AllowedIPs` | `0.0.0.0/0` | 表示隧道承载全部 IPv4 目标。真正的流量筛选发生在 §5 的 mark，不在这一行 |
| `Endpoint` | `162.159.192.1:2408` | WARP 边缘入口。`engage.cloudflareclient.com` 的 IPv4 解析结果，实测两者一致 |
| `PersistentKeepalive` | `25` | **不可省**。源站在 NAT 后面，不保活则 NAT 映射超时，隧道静默断开而接口仍显示 up |

### Endpoint 的来源与稳定性

```bash
getent ahostsv4 engage.cloudflareclient.com
# 162.159.192.1   STREAM engage.cloudflareclient.com

getent hosts engage.cloudflareclient.com
# 2606:4700:d0::a29f:c001 engage.cloudflareclient.com   ← IPv6 是另一个地址
```

`162.159.192.0/24` 是 Cloudflare WARP 数据的 anycast 段，长期稳定。
但 **`wg-quick` 不会重新解析域名**——它用的是配置里写死的那个 IP。
所以：

- **正常情况下永远不用管它**
- 万一该地址失效，隧道会断，但 `systemctl is-active wg-quick@warp0` 依然返回 `active`（静默故障）
- 换地址的方法见 §9.2

---

## 4. mark 机制：整套设计的中枢

### 4.1 为什么需要 mark

一个服务器要同时做两件相反的事：

- 大部分流量走**源站自己的 IP**（直连出口）
- 一部分流量走**WARP 隧道**

两者都是"从本机发出的包"，内核无法凭五元组区分。所以用 `SO_MARK` 给 socket 打标记，
再让路由规则按标记分流。

### 4.2 完整数据流

```
Xray outbound 配置 sockopt.mark = M
        ↓  Xray 建连时 setsockopt(SO_MARK, M)
本机发出的包 skb->mark = M
        ↓  ip rule: fwmark X/Y lookup 100
命中则查 table 100 → default dev warp0（进隧道）
未命中则查 main       → default via 网关（走源站 IP）
```

### 4.3 为什么是 `1/1` 而不是 `1`

历史上有两种写法：

| 写法 | 匹配条件 | 后果 |
|---|---|---|
| `fwmark 1` | `mark == 1` 精确匹配 | **只有 mark 恰好为 1 的包进隧道**。限速槽位的 mark（16k / 16k+1）全部漏掉，WARP 限速节点会悄悄退回直连 |
| `fwmark 1/1` | `mark & 1 == 1` 按位匹配 | bit0 为 1 的**所有** mark 都进隧道（1、17、33、49…），限速槽位的 WARP 变体也能正确进隧道 |

本项目的 mark 编码：

| 用途 | mark 计算 | 举例 |
|---|---|---|
| 不限速直连 | 无 mark | — |
| 不限速 WARP | `1` | 1 |
| 限速槽位 k 直连 | `16 × k` | k=1 → 16，k=2 → 32 |
| 限速槽位 k **WARP** | `16 × k + 1` | k=1 → 17，k=2 → 33 |

**bit0 专门留作「WARP 路由标志」**：直连槽位的 mark 都是偶数（bit0=0），WARP 槽位都是奇数（bit0=1）。
这样一条 `fwmark 1/1` 规则就能覆盖全部 WARP 场景，同时不影响直连槽位。

> `patch-xray-config.py` 里有一步会检查并修复 `fwmark 1` → `fwmark 1/1`，
> 就是因为历史上出现过漏写掩码、导致 WARP 限速节点实际走直连的问题。

### 4.4 绝对禁止：把 ct mark 写回 meta mark

带宽限速的下载方向依赖 conntrack 标记（见 [服务器配置.md](服务器配置.md) §7）。
有一条 nft 规则用来把出口 socket 的 mark 记进 conntrack：

```nft
chain out {
    type filter hook output priority mangle; policy accept;
    meta mark != 0 ct mark set meta mark
}
```

**看起来"顺手"再反向写回 `meta mark = ct mark` 很自然，但绝对不行。**

原因：下载方向的包是**发给本机、最终本地投递**的。如果它在 prerouting 阶段被打上
WARP 槽位的 mark（bit0=1），就会命中 `ip rule fwmark 1/1 lookup 100`，
被错误地路由进 `warp0` 隧道——本该送到 Xray 的数据包进了隧道，连接直接断掉。

需要还原标记给 tc 用时，只能用 `act_connmark`（它只把 ct mark 复制到 skb->mark 供
qdisc 分类，**不参与路由决策**），并且只在入口 qdisc 上做。

---

## 5. WARP 与带宽限速的交互

两套机制共用 mark 空间，靠 bit0 分工：

```
mark = 16k      → 直连出口 + 限速槽位 k
mark = 16k + 1  → WARP 出口 + 限速槽位 k
```

Xray 侧的对应关系（由 `patch-xray-config.py` 生成，共 16 槽位 × 2 = 32 个 outbound）：

```json
{"protocol": "freedom", "tag": "lim1",  "streamSettings": {"sockopt": {"mark": 16}}}
{"protocol": "freedom", "tag": "lim1w", "streamSettings": {"sockopt": {"mark": 17}}}
```

routing 规则把它们接起来（`lim{k}@warp` 用户 → `lim{k}w` outbound）：

```json
{"type": "field", "ruleTag": "rlim1w", "user": ["lim1@warp"], "outboundTag": "lim1w"}
```

**顺序很重要**：这些 `rlim{k}w` 规则必须排在兜底规则
`{"inboundTag": ["warp-in", "warp-local"], "outboundTag": "warp"}` **之前**，
否则所有 WARP 入站流量都会被兜底规则吃掉，限速槽位失效。

**结果**：走 WARP 的限速订阅，在 `eth0` 的 HTB 上按 `16k+1` 命中同一个类 `1:(10+k)`，
上传/下载限速与直连槽位完全等效。

---

## 6. 响应式握手：最容易踩的坑

**WireGuard 的握手是被动的**：没有出站流量就不会主动完成握手。

重启服务器后典型症状：

```bash
systemctl is-active wg-quick@warp0
# active                         ← 看起来完全正常

wg show warp0
# transfer: 0 B received, 2.75 KiB sent
# latest handshake: (空)         ← 从未握手

tcpdump -ni any udp port 2408
# 只有出向 UDP，没有任何回包
```

此时 **WARP 节点全部静默不可用**：带 mark 1 的包进了 `warp0` 却没人应答，
而且不像直连那样能回落，客户端表现为直接断流。

### 证据链

1. `systemctl restart wg-quick@warp0` 后**干等 3 分钟**，`latest handshake` 仍是 0
2. 同一个实例随便发一个包出去，**立刻**开始握手并成功

### 结论：重启隧道后必须主动踢一个包

```bash
systemctl restart wg-quick@warp0
sleep 2
ping -c 1 -W 3 -I warp0 1.1.1.1 >/dev/null 2>&1 || true   # ← 关键动作
sleep 8
wg show warp0 latest-handshakes
```

### 正常握手节奏

- WireGuard 的 `REKEY_AFTER_TIME` 是 120 秒，正常运行时会自动 rekey
- `PersistentKeepalive = 25` 每 25 秒发一个保活包，保证 NAT 映射不失效
- 所以健康状态下 `latest handshake` 应该始终在 **2 分钟以内**

---

## 7. 看门狗：三类静默故障的自愈

`wems-shape.sh check` 由 `wems-shape.service` 定时调用，与 WARP 相关的检查有三项：

| 检查 | 故障表现 | 自愈动作 |
|---|---|---|
| `ip rule show \| grep 'lookup 100'` | 规则丢失 → WARP 全部退回直连 | `ip rule add fwmark 1/1 lookup 100` |
| 握手时间戳 > `WARP_MAX_AGE`（300 秒） | 隧道死但接口 active | 重启 `wg-quick@warp0` + 主动 ping 踢握手 |
| `ip route show table 100` | 缺默认路由 → 包无处可去 | 重新 apply 整形（含路由） |

### 为什么要看握手而不是看 `is-active`

`wg-quick@warp0` 显示 `active` 只代表**配置加载成功**，不代表隧道可用。
实测（2026-10-16 重启后）：接口 up、路由正常、`ip rule` 正常，但隧道从未完成握手，
WARP 节点全部静默不可用。**只有握手时间戳能反映真实可用性。**

### 冷却机制

同一故障在 `WARP_COOLDOWN`（300 秒）内最多重启一次。
避免 Cloudflare 侧真故障时，看门狗每轮都重启隧道、把日志刷爆。

---

## 8. 从零重建 / 换 Endpoint / 换账号

### 8.1 完整重建（换一台机器）

需要四个值：`PrivateKey`、`Address`、`Peer PublicKey`、`Endpoint`。

```bash
# 方式一：wgcf 注册（需要能访问 GitHub）
curl -L -o /usr/local/bin/wgcf <wgcf release 地址>
chmod +x /usr/local/bin/wgcf
cd /root && wgcf register --accept-tos && wgcf generate
cat /root/wgcf-profile.conf
```

`wgcf-profile.conf` 与 `warp0.conf` 的字段对应：

| profile 字段 | 填到 warp0.conf |
|---|---|
| `[Interface] PrivateKey` | `[Interface] PrivateKey` |
| `[Interface] Address` | `[Interface] Address` |
| `[Peer] PublicKey` | `[Peer] PublicKey` |
| `[Peer] Endpoint` | `[Peer] Endpoint` |

然后在 `[Interface]` 里**额外加** `Table = 100`、`PostUp`、`PreDown`、`MTU = 1280`，
在 `[Peer]` 里**额外加** `PersistentKeepalive = 25`——这五项 `wgcf` 不会生成。

> 如果源站访问不了 GitHub（常见），直接复用一台已在用 WARP 的机器上的这四个值，
> **不需要再注册**。`Peer PublicKey` 全网相同，`Endpoint` 也基本相同，
> 真正需要自己生成的是 `PrivateKey` 和 `Address`。

### 8.2 更换 Endpoint

```bash
# 看当前解析结果
getent ahostsv4 engage.cloudflareclient.com

# 改配置
sed -i 's|^Endpoint = .*|Endpoint = 162.159.192.1:2408|' /etc/wireguard/warp0.conf
systemctl restart wg-quick@warp0
sleep 2
ping -c 1 -W 3 -I warp0 1.1.1.1 >/dev/null 2>&1 || true
sleep 8
wg show warp0 latest-handshakes
```

### 8.3 完全移除 WARP

```bash
systemctl disable --now wg-quick@warp0
rm -f /etc/wireguard/warp0.conf
ip rule del fwmark 1/1 lookup 100 2>/dev/null

# Xray 侧：仍需保留 warp 相关 outbound 与 routing（否则配置校验失败），
# 但订阅里应删除 WARP-xx 节点，或让它们不再被选用。
```

> 更干净的做法是保留隧道结构不动，只在客户端不选 WARP 节点。
> 因为 `♻️ WARP自动选择` 组没有规则指向它，移除节点的收益有限。

---

## 9. 故障排查矩阵

按链路顺序定位，**从外往内**。

| # | 症状 | 断点 | 判据 | 处理 |
|---|---|---|---|---|
| 1 | WARP 节点全不通，直连节点正常 | ① / ② | 外部 `curl -k https://VPN_HOST:2053/WS_PATH_WARP` 无响应 | 查 Cloudflare 橙云、源站 2053 监听 |
| 2 | 能连上但完全没网 | ③ | `xray api inbounduser -s 127.0.0.1:10085 -tag=warp-in` 里没有该用户 | `/api/sync` 重挂 |
| 3 | 有网但出口是源站 IP | ④ / ⑤ | `curl --interface warp0 .../trace` 返回 `warp=on` 但客户端不是 | routing 规则顺序错、或 outbound 没带 mark |
| 4 | 出口是源站 IP，`--interface warp0` 也是源站 IP | ⑤ | `ip rule show` 无 `fwmark 1/1 lookup 100` | 补规则；检查是否写成了无掩码的 `fwmark 1` |
| 5 | `ip rule` 正常但仍是源站 IP | ⑤ | `ip route show table 100` 无 `default dev warp0` | 重新 deploy 整形脚本 / 重启隧道 |
| 6 | 隧道 active 但握手为 0 | ⑥ | `wg show warp0 latest-handshakes` 空或过旧 | 重启 + 主动 ping 踢握手（见 §6） |
| 7 | 握手正常但没网 | ⑦ | `curl --interface warp0` 超时 | Cloudflare 侧问题，换 Endpoint |
| 8 | 小请求正常，大文件卡死 | ⑥ | — | MTU 太大，确认是 `1280` |
| 9 | WARP 限速节点不限速 | ④ | `tc class show dev eth0` 对应类计数为 0 | `fwmark` 缺掩码，或 `rlim{k}w` 规则被兜底规则截胡 |
| 10 | 隧道每隔一阵断一次 | ⑥ | 握手时间戳周期性归零 | `PersistentKeepalive` 缺失或被 NAT 超时 |

### 一条命令看全量状态

```bash
echo "--- rule ---";   ip rule show | grep fwmark
echo "--- table ---";  ip route show table 100
echo "--- tunnel ---"; wg show warp0
echo "--- egress ---"; curl -s --max-time 10 --interface warp0 \
    https://www.cloudflare.com/cdn-cgi/trace | grep -E '^(ip|colo|warp)='
```

四项全对才算健康：`fwmark 1/1` 存在、table 100 有默认路由、握手在 5 分钟内、
`warp=on`。

---

## 10. 已知限制与权衡

| 限制 | 说明 | 缓解 |
|---|---|---|
| **出口 IP 不固定** | WARP 是无状态 anycast，出口按 CF 负载动态分配，每次重连都可能不同 | 无法缓解，这是 WARP 的本质。不要指望拿它做目标侧 IP 白名单 |
| **5 个节点共用出口** | 切换 `WARP-0x` 只改入口，不改出口身份 | 若需要不同出口，只能申请多个 WARP 账户或换别的出口方案 |
| **MTU 1280** | 比常规 1500 少 220 字节，大包传输效率略低 | 用 WARP 的场景通常不需要极致吞吐 |
| **静默故障** | 隧道断了接口仍 active，普通 `is-active` 看不出 | 靠看门狗看握手时间戳（§7） |
| **Endpoint 静态** | `wg-quick` 不重新解析域名，CF 换地址需手工改 | 概率很低，`162.159.192.0/24` 稳定 |
| **mark 空间共享** | bit0 被 WARP 占用，块大小 16，所以限速槽位上限 16 | 足够用；扩槽位需改 `MARKBASE` 并同步改 `shape.sh` 与 Xray outbound |
| **不能反向写 ct mark** | 会把本地投递包误导进隧道 | 下载方向只用 `act_connmark`，见 §4.4 |

---

## 11. 一页速查

```bash
# 状态
wg show warp0                                  # 握手/流量/endpoint
ip rule show | grep fwmark                     # 路由规则
ip route show table 100                        # 隧道路由表

# 出口验证（两条都要跑）
curl -s --interface warp0 https://www.cloudflare.com/cdn-cgi/trace | grep -E '^(ip|warp)='
curl -s                   https://www.cloudflare.com/cdn-cgi/trace | grep -E '^(ip|warp)='

# 重启后必须踢一脚
systemctl restart wg-quick@warp0 && sleep 2 \
  && ping -c 1 -W 3 -I warp0 1.1.1.1 >/dev/null 2>&1; sleep 8; wg show warp0 latest-handshakes

# 看门狗
/usr/local/bin/wems-shape.sh check
tail -20 /var/log/wems-shape.log

# Xray 侧用户
/usr/local/bin/xray api inbounduser -s 127.0.0.1:10085 -tag=warp-in
```

---

## 附：本文档的事实来源

以下数值均取自真实运行环境（源站为 ColoCrossing 洛杉矶，Cloudflare 橙色云回源）：

| 项 | 实测值 |
|---|---|
| `engage.cloudflareclient.com` IPv4 | `162.159.192.1` |
| `engage.cloudflareclient.com` IPv6 | `2606:4700:d0::a29f:c001` |
| WARP 服务端 PublicKey | `bmXOC+F1FxEMF9dyiK2H5/1SUtzH0JuVo51h2wPfgyo=` |
| 隧道接口地址 | `172.16.0.2/32` |
| 到 Endpoint 的 RTT | 1.199 / 1.288 / 1.377 ms（min/avg/max） |
| WARP 出口示例 | `104.28.195.192`，`colo=LAX`，`warp=on` |
| 直连出口 | `ORIGIN_IP`，`colo=LAX`，`warp=off` |
| `ip rule` 优先级 | `32765` |
| outbound `warp` | `freedom` + `sockopt.mark: 1` |
| 兜底 routing 规则 | `inboundTag: [warp-in, warp-local] → outboundTag: warp` |
