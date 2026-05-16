---
title: 重庆大学校园网多设备检测对抗实战（OpenWrt + ua3f）
date: '2026-05-16 20:00'
tags:
  - OpenWrt
  - 校园网
  - 多设备检测
  - nftables
  - ua3f
categories:
  - 网络
abbrlink: 1e43e556
---

> 声明：本文仅记录在**本人自有账号、自有路由器**上的网络配置实践，用于多设备共享自用，不涉及破解计费、不针对他人。文中所有账号 / 密码 / token / 公网 IP 均已打码。请遵守所在学校的网络管理规定，风险自负。

## 背景

重庆大学校园网认证端点迁移到了 `login.cqu.edu.cn`（旧的固定 IP 认证作废），同时上线了**多设备共享检测**：一个账号下若被识别出多台设备，会提示"检测到共享 / 请勿使用代理"并冻结一段时间。

我的方案是一台 OpenWrt 路由器做主路由：

```
客户端 → nftables(TPROXY拦截) → hev-socks5-tproxy → ua3f(UA改写) → 校园网
```

认证部分用了一个自己 fork 改造的 LuCI 插件（适配 `login.cqu.edu.cn` 的新 JSONP 协议，账号 `2025********`、密码均在路由器本地配置，不外泄）。本文不讲认证，只讲**怎么尽量不被多设备检测，同时还能用**。

## 一个核心认知

ua3f 是个 **SOCKS5 终结点**：hev-socks5-tproxy 把 TPROXY 拦下的流量交给它，它**关闭客户端 TCP、用路由器自己的协议栈重新发起一条新连接**。

所以——凡是**进了代理**的流量，对外的 L3/L4 指纹（TTL、IP-ID、TCP options/时间戳、源端口行为）统一成路由器这一台，再叠加 HTTP UA 改写，对外等效"一台主机"。

> 结论：检测能抓到的，只有**漏出代理**的流量。加固 = 让尽量多的流量进代理 + 抑制进不了代理的流量。

下面按"检测维度 → 处理"组织。环境示例：LAN `192.168.1.0/24`、WAN 口 `eth0.1`、WAN 地址 `10.244.x.x`。

## P0：关掉 IPv6（最关键）

TPROXY 规则只处理 IPv4（`tproxy ip to`）。客户端只要有全局 IPv6 地址，就绕过 NAT 和代理，**每台设备的 v6 地址直接暴露**，前面所有努力白费。

```bash
uci set dhcp.lan.ra_slaac='0'
uci set dhcp.lan.ra='disabled'
uci set dhcp.lan.dhcpv6='disabled'
uci -q delete network.wan6
uci commit
grep -q disable_ipv6 /etc/sysctl.conf || {
  echo 'net.ipv6.conf.all.disable_ipv6=1'     >> /etc/sysctl.conf
  echo 'net.ipv6.conf.default.disable_ipv6=1' >> /etc/sysctl.conf
}
sysctl -p
/etc/init.d/odhcpd restart; /etc/init.d/network reload
```

> `network reload` 会瞬断 WAN，校园门户掉线几十秒后认证守护会自动重连。

## P1：拦截范围从"端口白名单"改成"几乎全部 TCP"

很多教程的 TPROXY 只拦 `{80,8080,...}` 几个端口。实测发现：任意端口的明文 HTTP（例如某服务跑在 `:8000`）会**整条漏出去**，真实 UA 直达服务器。

把规则改成"除 443 和 LAN 本地外的全部 TCP 都进代理"（`/etc/nfts/100-tproxy.nft`）：

```nft
table inet tproxy_table {
    chain prerouting {
        type filter hook prerouting priority -100; policy accept;
        iifname "br-lan" ip daddr != 192.168.1.0/24 tcp dport != 443 \
            tproxy ip to 127.0.0.1:1088 mark set 1
    }
}
```

- 排除 `192.168.1.0/24`：不劫持本机 SSH/管理页和内网互访（**防把自己锁外面**，按自己 LAN 段改）。
- 排除 `dport 443`：HTTPS 直连，避免把 TLS 大流量全压到弱 CPU（性价比折中）。
- **不要**排除 `10.0.0.0/8`：校园认证 / 内网就是 `10.x`，得继续走代理改写。
- 代价：443 仍直连 → 它的 TLS 指纹（见后文残留）改不掉。

## P2：统一 TTL —— 以及那个最容易踩的坑

多种客户端 OS 的初始 TTL 不同（Windows 128 / Linux·Android·iOS 64），经路由器再 -1，服务器看到一组离散 TTL 就知道你后面挂了多台不同系统。

新增 `/etc/nfts/110-cqu-hardening.nft`：

```nft
table inet cqu_hardening {
    chain cqu_post {
        type filter hook postrouting priority mangle; policy accept;
        oifname "eth0.1" ip ttl set 64
    }
    chain cqu_forward {
        type filter hook forward priority -10; policy accept;
        iifname "br-lan" oifname "eth0.1" udp dport 443 reject   # P3: 封 QUIC
        iifname "br-lan" meta nfproto ipv6 counter drop          # P0 兜底
    }
    chain cqu_nat {
        type nat hook prerouting priority dstnat; policy accept;
        iifname "br-lan" udp dport 123 redirect to :123          # NTP 收敛
    }
}
```

> 小坑：nft 里 chain 不能命名为 `fwd`，那是保留字。

### ⚠️ 关键坑：不关 flow offload，TTL 改写是假的

OpenWrt 默认开 **软件/硬件流量卸载（flow offloading）**。被卸载的连接走 fastpath 快速转发，**直接跳过 netfilter 的 postrouting hook**——也就是说上面的 `ip ttl set 64` 对这些连接**根本没执行**。

实测：开着 offload 时 `conntrack` 里大量连接带 `[OFFLOAD]` 标记，那条 `ttl set` 规则的计数远小于真实流量；关掉后单核负载从 `0.07` 直接飙到 `1.16`（流量全部回到协议栈）。

所以必须关：

```bash
uci set firewall.@defaults[0].flow_offloading='0'
uci set firewall.@defaults[0].flow_offloading_hw='0'
uci commit firewall
/etc/init.d/firewall restart
```

> 同理 `fullcone`（全锥 NAT）在部分实现里也会绕 hook，介意可一并关，代价是 P2P/游戏 NAT 类型变差。

这一步是好多人"配了 TTL 还是被检测"的根因。

## P3：封 QUIC

浏览器默认机会性走 HTTP/3（QUIC over UDP 443），它带类 TLS 指纹、流量大，且**完全绕过 TCP 代理链路**。上面 `cqu_forward` 里 `udp dport 443 reject` 一句把它打掉，浏览器会自动回落到 TCP（再被 ua3f 处理）。`reject` 比 `drop` 回落更快。

## NTP：被忽略的强特征

不同设备默认 NTP 服务器、对时频率都不一样，`udp/123` 直连出去就是一组多设备特征。处理方式：路由器自己当 NTP server，把客户端的 123 全部重定向到本地（上面 `cqu_nat` 已含 redirect）：

```bash
uci set system.ntp.enable_server='1'
uci commit system
/etc/init.d/sysntpd restart
```

效果：客户端以为在和公网 NTP 对话，其实被路由器本地应答；对外只剩路由器一个 NTP client 同步上游，多设备 NTP 特征收敛成一个。

## UA：伪装值本身别穿帮

ua3f 的 `-f` 一定要是**真实存在、当前主流**的 UA。我接手时它被设成了带 `CoolMarket/14.2.3-...`（某 App 尾巴）的串——桌面 Chrome 不可能带这种尾巴，所有设备统一成这个**不存在的客户端组合**，本身就是个显眼特征。换成干净的：

```bash
uci set ua3f.main.ua='Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/124.0.0.0 Safari/537.36'
uci commit ua3f
/etc/init.d/ua3f restart
```

## 自检脚本

在路由器上跑，逐项确认而不是凭感觉（节选）：

```sh
NF=/proc/net/nf_conntrack
# P0: 应为 0
grep -c '^ipv6' $NF
# P2: 应为 0（还有 OFFLOAD 说明 TTL 没真生效）
grep -c -i offload $NF
# P3: 应为 0
grep -cE 'udp .*(dport=443|sport=443)' $NF
# NTP: 客户端 123 应被本地(网关 IP)应答；src=WAN 的是路由器自身同步，正常
grep 'dport=123' $NF
# UA: 取最新一行启动日志确认伪装值
grep 'User-Agent:' /var/log/ua3f/ua3f.log | tail -1
```

客户端侧再配一个多 UA / 多端口探测脚本（思路同经典的 `ua_test.sh`：用不同 UA、不同端口请求一个回显服务，看服务器实际收到什么），就能定位还有哪一维在漏。**测的时候记得关掉客户端自己的代理软件**，否则流量绕过路由器，结果不可信。

## 诚实的残留（路由器侧改不掉）

| 维度 | 状态 |
| --- | --- |
| HTTPS/443 的 UA（在 TLS 里）、JA3/JA4 | 不做中间人解密就改不了；现代浏览器 JA3 已随机化弱化，JA4 仍可区分 |
| IP-ID（Windows 全局递增） | 经代理的已统一；直连转发的要彻底解决得编 `kmod-rkp-ipid`（重编固件） |
| TCP 时间戳 / options | 经代理的统一；直连泄露 |
| 并发连接数 / 账号行为 | 任何方案都藏不住"一个号大量并发" |

思路是：把能收敛的都收敛掉，让单一残留维度难以单独定位到"多设备"。

## 性能取舍

关 flow offload + 几乎全部 TCP 进 ua3f，对**弱硬件**（入门级单核 + 64MB 内存那种）是实打实的负担：实测关 offload 瞬间单核负载就上 1.1（≈满载），高带宽 / 多设备 / 测速会明显降速、延迟抖动、偶发卡顿，内存也吃紧。

可选：

- **接受**：日常网页 / IM / 文档够用。
- **折中**：P1 退回只代理少数明文 HTTP 端口，网速回升，但重新暴露非标端口明文（取舍）。
- **换硬件**（双核及以上 / x86 软路由）：根本解，强烈建议——弱核跑这套本就勉强。

## 致谢

- 认证插件 fork 自 [lurenjiamax/luci-app-cquauth](https://github.com/lurenjiamax/luci-app-cquauth)，协议参考 [haowang02/cqu-net-auth](https://github.com/haowang02/cqu-net-auth)
- UA 改写 [SunBK201/UA3F](https://github.com/SunBK201/UA3F)，TPROXY [heiher/hev-socks5-tproxy](https://github.com/heiher/hev-socks5-tproxy)
- 检测原理参考：褐瞳《校园网防止多设备检测指北》、SunBK201《某大学校园网共享上网检测机制研究》及若干博客园文章
