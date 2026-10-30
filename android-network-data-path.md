---
layout: page
title: "Android 网络数据路径与内核过滤攻击面手册"
description: "把「应用能不能联网」拆成策略与执行两层：策略在 Java（`ConnectivityService`／`NetworkPolicyManager`／`Vpn`），执行在内核（eBPF cgroup 钩子、`xt_bpf` 挂于 `bw_raw_PREROUTING`／`bw_mangle_POSTROU..."
date: 2026-10-30
permalink: /android-network-data-path/
---

> 网络这一层有个悖论：它是唯一一个**默认对全世界开放**的子系统，却又几乎不归应用自己管。应用写一句 `socket()` 或 `OkHttp` 调用，背后要穿过 netd 的策略、eBPF 的 cgroup 挂载点、内核的 netfilter 钩子、按网络绑定的路由表，才最终落到某块网卡上。绝大多数安全讨论只停在「应用有没有 INTERNET 权限」，而真正的边界在这条路径的**每一跳**上：谁决定这个 socket 属于哪个 UID、哪个网络、走不走 VPN、能不能被过滤。本手册只讨论公开且已修复问题的机制与自查方法，不含可利用代码。

---

## 0. 这篇手册解决什么问题

已发布的手册里，《无线与近场通信攻击面》讲的是**射频侧**（Wi-Fi 帧、蓝牙配对、NFC 标签），《基带与电话子系统》讲的是**调制解调器侧**（RIL、QMI、固件）。两者都停在「数据进出设备的那一段」，中间那段——**数据在设备内部怎么走、谁在过滤、按什么维度记账**——一直缺一篇。

这一段之所以值得单独成册，因为它是 Android 安全模型里**最典型的「策略与执行分离」结构**：

- **策略**在 Java 侧（`ConnectivityService`、`NetworkPolicyManager`、`Vpn` 框架），可读、可审计、常有厂商定制。
- **执行**在内核侧（eBPF 程序、netfilter 规则、路由表），由 netd 与 bpfloader 装配。
- **记账**在两者之间（按 UID 的字节计数、按链的允许/拒绝）。

三者不同步的时候，就会出现「策略以为已经切断、执行其实没切」或者「执行切断了、记账仍算在别的 UID 上」这类问题。这类问题不会出现在任何单个文件的代码评审里。

---

## 1. 数据路径的真实形状

一个应用发出的报文，在 Android 上大致要经过这些环节：

```
应用（任意 UID）
  ↓ socket() + bind/connect
SELinux：域 → socket 类 → 目标端口/节点
  ↓
网络选择：默认网络 / bindProcessToNetwork / Network.bindSocket
  ↓
路由与规则：按 UID 的 ip rule、按网络的表
  ↓
eBPF cgroup 钩子（按 UID 记账、按链过滤）
  ↓
netfilter（xt_bpf 钩子、NAT、conntrack）
  ↓
VPN / TUN（如果该 UID 被路由进 VPN）
  ↓
网卡 / 无线驱动
```

每一跳都有独立的安全语义，而且**它们的失败模式不一样**：

| 环节 | 谁在管 | 失败时的表现 |
|---|---|---|
| SELinux socket 检查 | 内核 MAC | 连接被拒（`EACCES`），可预期 |
| 网络选择 | 框架 + netd | 走错网络（不报错，只是路由不同） |
| eBPF 过滤 | netd + bpfloader | 静默丢包（应用层看到超时） |
| netfilter | netd | 静默丢包或拒绝 |
| VPN 路由 | `VpnService` + 框架 | 流量绕过或全部中断 |
| 网卡 | 驱动/固件 | 各种异常 |

**这张表的实用价值**：排查「为什么连不上」时，按「有报错 → 静默丢包 → 走错网络」三类分诊，比从头 grep 日志快得多。安全评审反过来用：**静默丢包型的问题最容易被忽略，因为它没有任何错误码**。

---

## 2. netd：策略的执行者

官方文档对 netd 的定位是：

> "The `netd` eBPF C program is used by the networking daemon (netd) in Android for various purposes such as socket filtering and statistics gathering."

并且 DNS 解析器 "is delivered as an APEX file and is dynamically linked by `netd`"——说明 netd 仍是解析器的主要消费者。

一个必须纠正的说法：**「netd 已经用 Rust 重写」在官方渠道找不到任何依据**。AOSP 主干 `system/netd` 树是 `client/ include/ netutils_wrappers/ server/ tests/`，没有 `rust/` 目录也没有 `Cargo.toml`。同时「native daemon」这个措辞在官方文档里也未见原句，宜写作 "networking daemon"。

### 2.1 从 xt_qtaguid 到 eBPF

Android 的按 UID 流量记账换过一次底座，官方对此说得很直接：

> "The new eBPF tool is replacing the `xt_qtaguid` module and the `xt_owner` module it is based on."
> "The new `trafficController` design is based on per-`cgroup` eBPF filter as well as `xt_bpf` netfilter module inside the kernel."
> "the userspace process `trafficController` creates the eBPF maps used for data collection"
> "the privileged process `bpfloader` loads the precompiled eBPF program into the kernel and attaches it to the correct `cgroup`"
> "pins all maps as a virtual file at `sys/fs/bpf`"
> "The `xt_bpf` netfilter is hooked at the `bw_raw_PREROUTING` and `bw_mangle_POSTROUTING` chain"

以及移除时间点：

> "We will start to remove the `xt_qtaguid` module from the Android kernel and disable its unnecessary configs."
> "The `xt_qtaguid` module will be completely deprecated when the minimum kernel version requirement for an Android release is 4.9 or above."

这段描述里藏着三个对安全有实际意义的细节：

1. **记账主体是 cgroup，不是 socket。** 所以「这个连接算谁的流量」由进程所属 cgroup 决定，而不是连接自身的属性。当一个进程被移动 cgroup（例如被判定为后台受限），它的流量归属随之改变。
2. **加载者是 `bpfloader`，不是 netd。** 官方明确它是独立的 "privileged process"。把 bpfloader 说成 netd 的一部分，会导致对该权限边界的错误判断——`bpfloader` 的能力比 netd 更接近内核。
3. **挂载点是内核的两个位置的组合。** cgroup 侧管按 UID 的过滤与统计；`xt_bpf` 的 `bw_raw_PREROUTING` / `bw_mangle_POSTROUTING` 管带宽与 NAT 前后的钩子。两个位置之间就是「同一个包在不同阶段看到的东西不同」的成因。

eBPF 对象的命名与类型也是可核对的：

> 挂载路径命名规则是 `/sys/fs/bpf/prog_FILENAME_PROGTYPE_PROGNAME` 与 `/sys/fs/bpf/map_FILENAME_MAPNAME`
> 程序类型含 "kprobe," "tracepoint," "skfilter," "schedcls," and "cgroupskb, cgroupsock"

**注意一个高频混淆**：`cgroupsock` 是**程序类型**，不是某个 netd 程序的名字。写成「netd 的 cgroupsock 程序」是把类型名当实例名。同理，`bpf_progs` 这个目录名也不成立——官方给的是 `system/bpfprogs`。

### 2.2 按 UID 的防火墙链

netd 侧的链是具名枚举，AOSP `NetdConstants.h` 里可以逐字读到：

```cpp
DOZABLE   = INetd::FIREWALL_CHAIN_DOZABLE,
STANDBY   = INetd::FIREWALL_CHAIN_STANDBY,
POWERSAVE = INetd::FIREWALL_CHAIN_POWERSAVE,
RESTRICTED= INetd::FIREWALL_CHAIN_RESTRICTED,
NONE      = INetd::FIREWALL_CHAIN_NONE,
```

所以 `FIREWALL_CHAIN_DOZABLE`（低电耗/待机）、`FIREWALL_CHAIN_STANDBY`、`FIREWALL_CHAIN_POWERSAVE`、`FIREWALL_CHAIN_RESTRICTED` 都是真实常量名。

需要标**未核实**的两条：

- `NetworkManagementService` 中 `setFirewallUidRule(...)` 的完整签名与 `FIREWALL_CHAIN_*` 的数值（如 1、2、3）——对应文件页持续 404，无法取原文。
- `fw_dozable` 这类 iptables 链名字符串——当前 `FirewallController.cpp` 中只出现 `"fw_INPUT"`、`"fw_OUTPUT"`、`"fw_FORWARD"`，其余链名未核实。

**为什么这四条链值得记住**：它们把「省电状态」和「网络可达性」直接绑在了一起。也就是说，一个应用被判定进入受限状态后，网络会被**在执行层切断**，而应用看到的只是连接超时。这是「静默丢包」最典型的生产事故来源。

---

## 3. VPN：应用层能看到什么、看不到什么

`VpnService` 的官方说明把边界讲得很清楚：

> "If the list includes one or more apps, then only the apps in the list use the VPN. All other apps (that aren't in the list) use the system networks as if the VPN isn't running."
> "You can create either an allowed list, or, a disallowed list, but not both. If you don't create allowed or disallowed lists, the system sends all network traffic through the VPN."
> "Apps that bind to a specific network don't have a connection when somebody blocks traffic that doesn't go through the VPN."

三句连起来，得到三条可操作结论：

1. **VPN 的覆盖面由「名单」而非「开关」决定。** 一个 VPN 应用没有设置名单时才是「全量」；一旦设置，未列入的应用**完全不受影响**——它们「use the system networks as if the VPN isn't running」。
2. **允许与禁止名单互斥。** 这避免了「既白名单又黑名单」的语义歧义，也意味着审计时必须先确认是哪一种模式，再判断覆盖面。
3. **`bindProcessToNetwork` / `Network.bindSocket` 会绕过 VPN 的可达性。** 官方在前一段说明应用会调用这两个 API 绑定特定网络；配合第三句，结果是：**绑定了具体网络的应用，在「阻止非 VPN 流量」生效时会直接失去连接**，而不是悄悄走 VPN。这是设计上的取舍，但常被误读为「VPN 有漏洞」。

因此「VPN 能看到的只是被路由进 TUN 的流量」这句结论是成立的，且它有官方依据，不是推断。评估一个「安全 VPN」时，第一个要问的问题是：**它的名单模式是什么，有没有应用可以通过绑定网络绕过。**

---

## 4. DNS：一个独立于「网络」的信任面

DNS 是唯一一个**用户不可见、但决定一切**的协议：域名解析结果决定了连接去哪台服务器，而用户看到的只有最终页面。

### 4.1 解析服务是 AIDL 接口

AIDL 实名与关键方法可以直接读到：

```aidl
package android.net;
interface IDnsResolver {
    void setResolverConfiguration(in ResolverParamsParcel resolverParams);
    void createNetworkCache(int netId);
    void setAllowBypassPrivateDnsOnNetwork(int netId, int uid, boolean allowed);
    ...
}
```

从中可以读出三条设计事实：

- **配置是按网络（`netId`）下发的**，不是全局单一配置。这解释了为什么切换 Wi-Fi/蜂窝时解析行为会变。
- **`createNetworkCache(netId)`** 说明每个网络有自己的缓存——缓存投毒的影响半径是一个网络，不是整机。
- **`setAllowBypassPrivateDnsOnNetwork(netId, uid, allowed)`** 说明「允许某个 UID 绕过 Private DNS」是**逐 UID 的显式决定**。这是一条真实存在的降级通道，评估时必须把它当作「谁被允许不加密解析」的清单来读。

### 4.2 加密 DNS 的版本边界

官方对加密解析的表述是：

> "Since SDK level 28, Android has added security to DNS transport through DNS over TLS, and then DNS over HTTP/3 in SDK level 30."

由此可以确定：**平台确实实现了 DoT（SDK 28）与 DoH3（SDK 30）**。所以「Android 不支持 DoH」是错的。

但要注意精度：**经典型 DoH（HTTP/2 上的 DoH）是否有平台内建实现，未核实**。正文只写「DoT 与 DoH3」，不写「DoH」这个宽泛说法。

---

## 5. IPv6：被忽略的默认路径

IPv6 在 Android 上是**默认启用**的，而它带来两个与 IPv4 本质不同的性质：

1. **地址自动配置**：主机可以从路由器通告（RA）里拿到前缀并自行构造地址，不需要 DHCP 服务器。
2. **邻接发现（ND）**：地址解析与路由发现都依赖 ICMPv6 报文，而这类报文在链路上是**可被邻近设备发送**的。

内核侧的相关 sysctl 语义有权威定义：

> `accept_ra`: "Accept Router Advertisements; autoconfigure using them."；取值 "0 Do not accept Router Advertisements." / "1 Accept Router Advertisements if forwarding is disabled." / "2 Overrule forwarding behaviour."
> functional default: "enabled if local forwarding is disabled." / "disabled if local forwarding is enabled."
> `accept_ra_defrtr` 的 functional default: "enabled if accept_ra is enabled."

这组语义说明：**「是否接受 RA」与「是否转发」是耦合的**。这个耦合在路由器上是对的（转发设备不该按 RA 自动配置），但在终端上它意味着「只要转发被打开，RA 就会被忽略」——这类隐含耦合是评测配置时容易漏掉的点。

**必须标注未核实**：Android 专有的 `accept_ra` 加固取值与写入点（netd/init 具体写了什么值）未取得一手依据。正文只使用内核文档的通用语义，不对 Android 的具体取值下断言。

---

## 6. 应用能碰到的内核网络面

这一节是全篇最需要精确的地方，因为流传的错误说法最多。AOSP `private/app_neverallows.te` 里的原文是：

```
neverallow all_untrusted_apps *:{ socket netlink_socket packet_socket key_socket appletalk_socket … } *;
neverallow all_untrusted_apps domain:netlink_socket *;
neverallowxperm all_untrusted_apps domain:{ icmp_socket rawip_socket tcp_socket udp_socket } ioctl priv_sock_ioctls;
neverallow all_untrusted_apps domain:netlink_route_socket { bind nlmsg_readpriv };
neverallow { all_untrusted_apps -untrusted_app_25 -untrusted_app_27 -untrusted_app_29 -untrusted_app_30 } domain:netlink_route_socket nlmsg_getneigh;
```

**第一个陷阱**：这些规则的主体是 `all_untrusted_apps`，而这个宏的定义是

```
define(`all_untrusted_apps',`{ ephemeral_app isolated_app isolated_app_all isolated_compute_app
   mediaprovider mediaprovider_app untrusted_app untrusted_app_25 untrusted_app_27
   untrusted_app_29 untrusted_app_30 untrusted_app_all }')
```

——**它不包含 `appdomain`**。所以写「neverallow appdomain … packet_socket」是错的；正确写法是「`neverallow all_untrusted_apps … packet_socket`」。这个区别不是抠字眼：`appdomain` 是一个更大的集合，包含系统应用域；把主体写错，会得出「连系统应用都不能用 packet socket」的错误结论。

**第二个陷阱**：`rawip_socket` 只出现在 `neverallowxperm`（针对 `ioctl` 的扩展权限）中，**不能据此断言「应用不能创建 raw socket」**，更不能写「应用可以申请 `CAP_NET_RAW` 来开 raw socket」——后者与 neverallow 模型不符，属未核实。

**第三个陷阱**：`nlmsg_getneigh` 的 neverallow 带了一串减法（`-untrusted_app_25` 等），意思是**只有较新 targetSdk 的应用**被禁止读取邻居表。这正是 `targetSdk` 影响 SELinux 策略的具体例子（与《SELinux 策略与 seccomp 沙箱》里讲的按 targetSdk 降档是同一机制）。

---

## 7. 成因分析：网络栈的缺陷从哪来

### 7.1 策略与执行不同步

这是本节最核心的一类。前面几节已经埋了线索：

- 策略用「网络 ID」表达，执行用「路由表 + cgroup + netfilter 链」表达，两套标识之间的映射由 netd 维护。
- 网络切换（Wi-Fi ↔ 蜂窝）、网络消失（掉线）、网络替换（VPN 建立/断开）都会触发重新映射。
- 映射过程中的**短暂窗口**里，某个 UID 的包可能落到旧网络上，或绕过新加的过滤。

这类窗口没有 CVE 编号，但它是最典型的「策略正确、执行有洞」问题。自查方法不是读代码，而是**做状态迁移测试**：在切换网络的瞬间跑持续连接，看是否有包从非预期路径出去。

### 7.2 UID 归属在多路径上漂移

按 UID 的过滤与记账都依赖「这个包属于哪个 UID」。这个归属在以下情况下会变：

- **共享 socket**：一个进程把 socket fd 传给另一个进程（`SCM_RIGHTS`），包的归属仍是原 UID。
- **代理/转发**：本地代理进程代发时，包属于代理进程的 UID。
- **tethering**：被共享的流量属于下游设备，不是本机应用。
- **VPN**：进入 TUN 后重新封装，内外层 UID 语义不同。

当策略假设「UID 归属稳定」而实际会漂移时，就会出现「按 UID 禁止了某应用联网，但它通过代理仍然出去了」这类结果。防御做法是**在策略层同时约束 UID 与网络路径**，而不是只信 UID。

### 7.3 钩子位置决定「看到什么」

eBPF 与 netfilter 在数据路径上有多个可挂载点（`PREROUTING`、`INPUT`、`FORWARD`、`OUTPUT`、`POSTROUTING`，以及 cgroup 的 ingress/egress）。同一个包在不同钩子上看到的信息不同：

- 在 `PREROUTING` 之前，NAT 还没发生，看到的是原始目的地址。
- 在 `POSTROUTING` 之后，源地址可能已被 SNAT。
- 在 cgroup 钩子上，看到的是「哪个 cgroup 发的」，但**不保证**看到最终的出口网络。

所以「在 A 点过滤了某个目的地址」不等于「这个包到不了」。这是 §2.1 里 `bw_raw_PREROUTING` 与 `bw_mangle_POSTROUTING` 之所以要成对出现的原因——带宽策略必须同时管两端，否则可以被单侧绕过。

### 7.4 解析先于连接

DNS 的问题在于它是**信任链的最上游**：应用连接的是「名字解析出来的地址」，而不是「用户以为的地址」。三个结构性弱点：

1. **缓存是分网络的**（`createNetworkCache`），所以投毒影响的是「当前网络上的所有客户端」。
2. **Private DNS 有逐 UID 的绕过通道**（`setAllowBypassPrivateDnsOnNetwork`），绕过者看到的是明文解析。
3. **DoT/DoH3 保护的是传输**，不保护「解析结果是否被上游故意给错」——加密只防窃听与篡改，不防恶意权威服务器。

### 7.5 邻近协议的可达性

IPv6 的 RA/ND 让「同一链路上的设备」天然具备一定影响力。这不是漏洞，是协议设计。防御侧的对应项是：**显式配置 RA 接受策略、对关键接口禁用不必要的自动配置、以及在网络边界上做 RA Guard**（后者属网络设备侧，不在 Android 内）。

### 7.6 内核网络代码的内存安全

最后是大类：内核网络栈自身的解析缺陷。历史上 Android 公告里的网络类内核条目多数落在这里（`Net`、`XFRM`、`EPoll` 等 Subcomponent）。特点是：

- 触发面可能是**远端**（收到一个构造过的包），不需要本地权限；
- 修复在 kernel LTS，**内核版本比 Android 版本更能反映风险**；
- 与本文关系是「数据路径的最后一跳」，但它属于《内核与 GKI 驱动攻击面》的范畴，此处只做衔接说明，不重复展开。

---

## 8. 公开案例台账（逐条核实归属）

网络主题是 CVE 张冠李戴的重灾区，因为「网络相关」这个词太宽。下表只列已核实组件：

| 编号 | 实际组件（记录原文要点） | 与网络栈的关系 |
|---|---|---|
| CVE-2020-0423 | "In binder_release_work of binder.c … use-after-free due to improper locking"（内核 Binder） | **无关**：Binder 驱动 |
| CVE-2021-39624 | "In PackageManager … permanent denial of service due to resource exhaustion" | **无关**：包管理 |
| CVE-2022-20007 | "In startActivityForAttachedApplicationIfNeeded of RootWindowContainer.java … overlay an app that believes it's still in the foreground … due to a race condition" | **无关**：窗口与活动管理 |
| CVE-2023-21282 | "In TRANSPOSER_SETTINGS of lpp_tran.h … out of bounds write … remote code execution"（AAC 解码器） | **无关**：媒体解码，但**攻击面是远端的**，属「远端可达但非网络栈」 |
| CVE-2024-43093 | "In shouldHideDocument of ExternalStorageProvider.java … incorrect unicode normalization"（并列入 CISA KEV） | **无关**：存储路径过滤 |

**结论必须写清楚**：这五个编号在二手资料里常被归入「Android 网络漏洞」，实际全部不是。真实的 netd / eBPF / VPN 层缺陷**极少进入公开 CVE 通道**——它们大多以「安全加固」「行为变更」的形式出现在版本说明里，或以厂商内部编号修复。

这与 §7.1 的结论互相印证：**这一层的多数问题是设计语义问题，而不是可编号的内存破坏**。因此本章同样不提供「网络栈专属 CVE 清单」，而是给出成因分类与可执行的观测方法。

---

## 9. 防护技术

### 9.1 设计层

| 措施 | 做法 | 挡住什么 |
|---|---|---|
| 策略与路径双约束 | 禁止某 UID 联网时，同时约束其可用的网络与是否允许代理 | UID 归属漂移导致的绕过 |
| 过滤点成对布置 | 入口与出口各设一处，避免单侧可绕过 | 钩子位置差异 |
| 默认拒绝 VPN 名单 | 用 allowlist 而非「全量 + 例外」 | 新装应用自动获得通道 |
| Private DNS 收紧 | 检查 `setAllowBypassPrivateDnsOnNetwork` 的 UID 清单，非必要不放开 | 明文解析降级 |
| 解析结果校验 | 对高价值域名固定使用证书校验 + pinning，不依赖解析正确性 | DNS 层投毒 |

### 9.2 实现层（应用与服务侧）

1. **不把「有 INTERNET 权限」当作可达性保证。** 必须处理防火墙链导致的静默丢包（超时、重试、退避），并在 UI 上区分「无网络」与「被策略阻断」。
2. **绑定网络要显式。** 需要特定网络时用 `bindProcessToNetwork` / `Network.bindSocket`；不要依赖「当前默认网络」在切换瞬间的含义。
3. **代理与转发不要隐式。** 任何代发流量的组件都应显式记录「代表哪个 UID」，否则审计时无法还原归属。
4. **DNS 结果不当凭据。** 不要把「解析到了内网地址」当作身份验证。
5. **IPv6 同等对待。** 网络策略、防火墙规则、日志采集都要覆盖 IPv6；只按 IPv4 写的规则在双栈环境里等于没写。

### 9.3 验证层

- **规则快照**：把 `ip rule`、`ip route`、`iptables-save`、`/sys/fs/bpf` 下的对象清单做成基线，网络切换前后各抓一次做 diff。
- **归属验证**：用已知 UID 的应用发起连接，在 netd/内核侧核对记账归属是否与预期一致。
- **旁路验证**：对「应当被阻断」的 UID，测试三种路径——直连、经代理、经 VPN——确认三条都被阻断。
- **解析验证**：在同一网络下分别用系统解析与直连 DoT 服务器解析同一域名，比对结果差异。

---

## 10. 自查与观测清单

```bash
# 1. 路由规则与表（按 UID 的规则在这里）
adb shell ip rule show
adb shell ip route show table all | head -40

# 2. netfilter 现状
adb shell iptables-save 2>/dev/null | head -60
adb shell ip6tables-save 2>/dev/null | head -40

# 3. eBPF 对象（需 root 或可读权限）
adb shell ls -l /sys/fs/bpf 2>/dev/null

# 4. 网络与默认路由
adb shell dumpsys connectivity | head -80
adb shell dumpsys netpolicy | head -60

# 5. DNS 解析器状态
adb shell dumpsys dnsresolver 2>/dev/null | head -60
adb shell getprop | grep -i private_dns

# 6. IPv6 与 RA 相关
adb shell ip -6 addr show
adb shell cat /proc/sys/net/ipv6/conf/all/accept_ra 2>/dev/null
adb shell cat /proc/sys/net/ipv6/conf/wlan0/accept_ra 2>/dev/null

# 7. 按 UID 的流量统计来源
adb shell dumpsys netstats | head -40
```

判读要点：

- **第 3 条在非 root 设备上通常读不到**，这是正常的；此时改用 `dumpsys netd` 观察装配状态。
- **第 6 条要注意「全局值 vs 接口值」**：`conf/all/accept_ra` 与具体接口的值可能不同，接口值优先。
- **第 2 条里 `iptables-save` 在现代设备上可能为空**——不代表没有过滤，过滤可能全在 eBPF 侧。这正是「只看 iptables 会得出错误结论」的原因。

---

## 11. 常见误传纠正

1. **「netd 已经用 Rust 重写」**——无任何官方出处；AOSP 主干 `system/netd` 无 Rust 代码。
2. **「`neverallow appdomain … packet_socket / netlink_socket`」**——原文主体是 `all_untrusted_apps`，且该宏不含 `appdomain`。
3. **「iptables 链叫 `fw_dozable`」**——未核实；可核实的只有 `FIREWALL_CHAIN_DOZABLE` 等常量与 `fw_INPUT` / `fw_OUTPUT` / `fw_FORWARD`。
4. **「Android 不支持 DoH」**——错。官方写明 SDK 30 起支持 DNS over HTTP/3，SDK 28 起支持 DoT。但「经典型 DoH（HTTP/2）」是否有平台内建实现未核实。
5. **「netd 用 network namespace 隔离每个网络」**——未核实；官方只描述按网络绑定（`bindProcessToNetwork` / `Network.bindSocket`）。
6. **「`bpfloader` 属于 netd 进程」**——官方明确它是独立的 privileged process。
7. **「`cgroupsock` 是 netd 的程序名」**——它是 eBPF **程序类型**名；BPF 源码目录是 `system/bpfprogs`，不是 `bpf_progs`。
8. **「应用申请权限即可拿 `CAP_NET_RAW` 开 raw socket」**——未核实，且与 neverallow 模型不符。
9. **「VPN 开着的应用一定全部走 VPN」**——错。设置了名单后，未列入的应用 "use the system networks as if the VPN isn't running"；绑定具体网络的应用会直接失去连接。
10. **「iptables 里没看到规则就是没有过滤」**——过滤可能全在 eBPF 侧（cgroup + `xt_bpf`）。
11. **CVE 张冠李戴**：CVE-2024-43093 是 ExternalStorageProvider 路径过滤、CVE-2023-21282 是 AAC 解码器、CVE-2020-0423 是内核 Binder、CVE-2021-39624 是 PackageManager、CVE-2022-20007 是窗口容器竞态——**均非 netd/无线/网络栈**。

---

## 未核实与失效来源

- **A. netd 的 Rust 重写。** 检索无官方表述；AOSP 主干无 Rust 源码，按未核实处理。
- **B. 「native daemon」措辞。** 官方用 "networking daemon"，未见 "native daemon" 原句。
- **C. `NetworkManagementService.setFirewallUidRule` 签名与链数值。** 对应源文件页持续 404，未取原文。
- **D. `fw_dozable` 等 iptables 链名。** 未核实；正文只用 `FIREWALL_CHAIN_*` 常量与 `fw_INPUT/fw_OUTPUT/fw_FORWARD`。
- **E. netd 内部 eBPF 程序实例名。** 官方页面只出现 `TrafficController*` 路径，未给出实例名清单。
- **F. Android 侧 `accept_ra` 的具体取值与写入点。** 未取得一手依据，正文只用内核文档的通用语义。
- **G. 经典型 DoH（HTTP/2）的平台内建实现。** 未核实；已核实的是 DoT 与 DoH3。
- **H. `TrafficStats` 文档中关于 qtaguid 的原句。** 参考页被导航截断，正文引用的是 eBPF 交通监控页的原文。
- **I. netd / eBPF / VPN 专属 CVE 清单。** 现网公告中未核到可靠条目，本节以归属纠正表替代。

---

## 参考

- eBPF 交通监控（qtaguid → eBPF、bpfloader、xt_bpf 钩子）：[source.android.com/docs/core/data/ebpf-traffic-monitor](https://source.android.com/docs/core/data/ebpf-traffic-monitor)
- Android eBPF 概览（挂载命名、程序类型）：[source.android.com/docs/core/architecture/kernel/bpf](https://source.android.com/docs/core/architecture/kernel/bpf)
- DNS Resolver 模块：[source.android.com/docs/core/architecture/modular-system/dns-resolver](https://source.android.com/docs/core/architecture/modular-system/dns-resolver)
- `NetdConstants.h`：[android.googlesource.com/platform/system/netd/+/refs/heads/main/server/NetdConstants.h](https://android.googlesource.com/platform/system/netd/+/refs/heads/main/server/NetdConstants.h)
- `IDnsResolver.aidl`：[android.googlesource.com/platform/packages/modules/DnsResolver/+/refs/heads/main/binder/android/net/IDnsResolver.aidl](https://android.googlesource.com/platform/packages/modules/DnsResolver/+/refs/heads/main/binder/android/net/IDnsResolver.aidl)
- VPN 开发指南：[developer.android.com/develop/connectivity/vpn](https://developer.android.com/develop/connectivity/vpn)
- 不安全 DNS 风险页（DoT/DoH3 版本）：[developer.android.com/privacy-and-security/risks/bad-dns](https://developer.android.com/privacy-and-security/risks/bad-dns)
- `app_neverallows.te`：[android.googlesource.com/platform/system/sepolicy/+/refs/heads/main/private/app_neverallows.te](https://android.googlesource.com/platform/system/sepolicy/+/refs/heads/main/private/app_neverallows.te)
- IPv6 sysctl 语义：[kernel.org/doc/html/latest/networking/ip-sysctl.html](https://www.kernel.org/doc/html/latest/networking/ip-sysctl.html)
- 公告：[source.android.com/docs/security/bulletin](https://source.android.com/docs/security/bulletin)

---

## 联系方式

如果你在企业里做终端网络策略审计，需要就「按 UID 的连通性基线」「VPN 名单与旁路核对」「DNS 解析链路核查」做一次针对现网设备或自有应用的验证，可以通过 Telegram 联系：<https://t.me/one00190>

---

*本手册仅用于防御与教育目的，讨论的均为公开且已被修复问题的机制与设计缺陷，不包含任何可直接利用的代码。文中标注「未核实」的内容不应作为事实引用；发现错误欢迎指出。*
