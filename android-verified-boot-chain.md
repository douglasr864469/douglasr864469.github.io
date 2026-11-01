---
layout: page
title: "Android Verified Boot 与 dm-verity 验证链攻击面手册"
description: "把「验证通过」拆成四段与四个断点：硬件信任根→bootloader→`vbmeta`（`AVB0`）→分区的 hash/hashtree/chain 描述符；另含 AVB 自 Android 8.0（7.0 强化的是 dm-verity）、`HASHTREE_DISABLED`/`VERIFICATIO..."
date: 2026-11-01
permalink: /android-verified-boot-chain/
---

> 整条信任链上，唯一不可被软件修改的那一环是一小块一次性写入的熔丝。它里面只有一个哈希；bootloader 用这个哈希验证自己的下一段代码，下一段再验证下一段，一直到 `system`、`vendor`、`product`。从那一刻起，设备上每一个字节都有了一个「应该是什么」的答案。本手册讨论这条链怎么搭起来、哪里会断、以及「验证通过」到底证明了什么、没证明什么。只讨论公开且已修复问题的机制与自查方法，不含可利用代码。

---

## 0. 这篇手册解决什么问题

已发布的手册里，《TEE、密钥管理与设备认证》讲了 `verifiedBootHash` 怎么进入密钥证明，《系统更新链路与回滚防护》讲了 OTA 怎么落地。两篇都用到「Verified Boot」这个前提，但都没有展开这条链本身。

本篇补三个问题：

1. **链是怎么搭的？** 从熔丝到文件系统，每一段用什么签名、验证什么。
2. **断在哪？** 解锁、降级、逃生标志、解析器——四个断点性质完全不同。
3. **「验证通过」证明了什么？** 这是最重要的一问：它证明的是**静态分区的字节没被改**，不是「设备安全」，也不是「数据没被读」。

---

## 1. 架构：四段验证，三种状态

### 1.1 信任链的全貌

官方对这条链的表述是明确的：

> "establishes a full chain of trust, starting from a hardware-protected root of trust to the bootloader"
> "each stage verifies the integrity and authenticity of the next stage before handing over execution"

画成图：

```
硬件信任根（eFuse / OTP / Boot ROM 内的公钥哈希）
   │  验证
   ▼
Bootloader（含 ABL / XBL 等厂商组件）
   │  验证
   ▼
vbmeta（签名 + 描述符集合）
   │  描述
   ├──▶ boot.img（kernel + ramdisk）   ── hash 描述符
   ├──▶ system / vendor / product      ── hashtree 描述符 + dm-verity
   ├──▶ init_boot / vendor_boot        ── hash 描述符
   └──▶ 其他分区的 vbmeta（链式分区）   ── chain 描述符
```

版本脉络必须记住三个点，它们常被合并成一句错话：

- **Android 4.4** 引入基于 dm-verity 的 Verified Boot（官方称 Verified Boot 1）。
- **Android 7.0** 起**严格强制** Verified Boot，被篡改的设备不能正常启动。
- **Android 8.0** 起包含 **AVB**（Android Verified Boot），也就是带 `vbmeta` 的这一代。

所以「AVB 是 Android 7 引入的」是错的，7.0 强化的是 dm-verity 那条老路径。

### 1.2 vbmeta：一个签名过的「说明书」

`vbmeta` 分区里放的是 `AvbVBMetaImageHeader` 结构，魔数是 **`AVB0`**。它自己不含文件系统，只含三样东西：

1. **公钥与算法**：用来验证后续分区。
2. **签名**：由上一级（bootloader 里的信任根或链式分区的上级 vbmeta）验证。
3. **描述符列表**：告诉验证者「哪些分区、用什么方式验证」。

描述符有几种类型，区别不是细节，而是**验证方式完全不同**：

| 描述符 | 适用 | 验证方式 |
|--------|------|----------|
| hash | 小分区（如 `boot`、`dtbo`） | 整段哈希，一次性比对 |
| hashtree | 大分区（`system`、`vendor`） | 构造哈希树，交给 dm-verity 逐块校验 |
| chain | 其他分区里的 vbmeta | 指定公钥与 rollback index 位置，递归验证 |
| kernel cmdline | 启动参数 | 把参数片段并入签名范围 |
| property | 系统属性 | 把属性值并入签名范围 |

`vbmeta` 头部还有两个标志位，它们的存在本身就是一种设计取舍：

- `AVB_VBMETA_IMAGE_FLAGS_HASHTREE_DISABLED`（值 `1`）：为该 vbmeta 覆盖的分区关闭 hashtree 验证。
- `AVB_VBMETA_IMAGE_FLAGS_VERIFICATION_DISABLED`（值 `2`）：整体关闭验证。

这两个标志是**逃生门**：它们让「验证不过就救不回来」的极端情况有解，但也意味着「vbmeta 存在」不等于「验证在跑」。

### 1.3 dm-verity：在块设备上加一棵树

dm-verity 是内核的 device-mapper 目标。它把数据分区切成固定大小的块（Android 上通常 4 KiB），对每块算哈希，再对哈希块算哈希，层层向上，直到一个根哈希。这个根哈希写在 vbmeta 的 hashtree 描述符里，受签名保护。

运行时的工作方式：

```
读 block N
  → 计算哈希
  → 与哈希树中对应节点比
     ├─ 一致：返回数据（并把已验证节点缓存）
     └─ 不一致：按 veritymode 处理
```

`veritymode` 有三个取值，通过 `ro.boot.veritymode` 暴露：

| 取值 | 行为 |
|------|------|
| `enforcing` | 校验失败即 I/O 错误，进程读到的是失败而非坏数据 |
| `logging` | 只记录，不阻断——**等于没有运行时保护** |
| `disabled` | 不启用 |

这条区分很实用：一台设备可以「AVB 通过、vbmeta 签名正确」，但 `veritymode=logging`，那么运行时的篡改不会被拦。

### 1.4 动态分区：两层面具

Android 10 引入动态分区后，`system`/`vendor`/`product` 不再是独立物理分区，而是 `super` 里的逻辑分区。于是验证链条变成两层：

```
super 分区
  └─ dm-linear（切出 system_a 等逻辑分区）
       └─ dm-verity（对切出来的设备做完整性校验）
```

顺序不能反：先切，再验。这也解释了为什么 `super` 本身的完整性通常由 vbmeta 的 hash 描述符保护，而内部逻辑分区由各自的 hashtree 描述符保护。

### 1.5 APEX 与 fs-verity

APEX（Android 10 起）是「可以独立更新的系统组件容器」。每个 APEX 的载荷是一个镜像文件，由 **dm-verity** 保护，其根摘要写在 APEX 自己的签名元数据里，公钥来自系统内置的 APEX 公钥集。所以「APEX 不受验证保护」是错的——它受，只是验证的锚点不是 `vbmeta`，而是 APEX 自身的签名链。

### 1.6 启动状态：四种颜色

设备把验证结论表达成四个状态，通过 `ro.boot.verifiedbootstate` 暴露：

| 状态 | 含义 | 典型触发 |
|------|------|----------|
| `green` | 锁定，全部用官方密钥验证通过 | 出厂状态 |
| `yellow` | 锁定，但使用自定义密钥 | 开发者/企业定制签名 |
| `orange` | **已解锁** bootloader | 用户执行解锁 |
| `red` | 验证失败 | 篡改或损坏 |

必须区分 `yellow` 与 `orange`：前者是「锁着但换了钥匙」，后者是「锁被打开了」。把 yellow 当作「被 root」是常见误读，它更接近「非官方签名但仍是受控状态」。

`red` 的实际后果取决于设备配置——可能拒绝启动、可能进入恢复模式、也可能仍启动但标记状态。**「验证失败一定不能启动」不是普遍成立的**。

---

## 2. 攻击面：四个断点

### 2.1 断点一：解锁 bootloader

这是唯一合法的「关闭验证」方式，也是所有后续攻击的前提。现代设备的解锁流程通常包含：

- 在开发者选项里打开「OEM 解锁」；
- 执行 `fastboot flashing unlock`；
- 设备**清空用户数据**（这是防「解锁后取证」的设计）；
- 启动状态变为 `orange`。

这条设计的含义是：**解锁不是漏洞，是一个被明确建模的状态转移**。安全评估要问的不是「能不能解锁」，而是「解锁后哪些楼层会打开」——例如某些支付、企业应用、DRM 等级会拒绝在 `orange` 状态下工作。

### 2.2 断点二：签名密钥

整条链的安全性最终落在一个密钥上。密钥泄露的后果是**设备族级别的**：攻击者可以签发任意 `vbmeta`，让任意代码通过验证。

三个现实约束：

- 密钥通常在 HSM 里，签发流程需要多人审批；
- 密钥轮换成本极高（需要新信任根，通常意味着新硬件）；
- 一旦泄露，唯一的缓解是「更换信任根」，这在已出货设备上几乎不可能。

### 2.3 断点三：降级

验证回答的是「这段代码是不是官方签的」，**不是「这段代码是不是最新的」**。防止降级靠的是 **rollback index**：

- `vbmeta` 里记录一个单调递增的索引；
- 设备把它存在防篡改存储里（通常是 RPMB，或由 TEE 保护）；
- 启动时如果 `vbmeta` 的索引**小于**已存的值，拒绝启动。

如果厂商没有正确实现 rollback index，设备就可以被刷回旧版本，从而利用已经修复的漏洞。这是 Android 生态里最持久的结构性问题之一：**「补丁级别很高」不等于「不能降级到补丁级别很低」**。

### 2.4 断点四：解析器本身

`vbmeta` 是二进制结构，由 **bootloader 解析**。而 bootloader 运行在验证链的**最前端**——它比被验证的代码更早执行，也更难更新。

这里的缺陷类型与媒体解析完全同构：

- 描述符数量、长度字段、偏移；
- 公钥/签名的长度；
- 链式分区的递归深度；
- 魔数与版本不匹配时的回退分支。

一旦在解析阶段发生内存破坏，攻击者就已经站在了「验证者」的位置上——后面所有验证都失去意义。这是整条链上**危害最高、也最不透明**的一层，因为 bootloader 通常闭源且由 SoC 厂商维护。

### 2.5 附带说明：验证不覆盖什么

| 不覆盖 | 说明 |
|--------|------|
| `/data` 用户数据 | 由 FBE 负责机密性，不由 AVB 负责完整性 |
| 运行时内存 | 进程被注入后，静态验证无能为力 |
| Bootloader 自身 | 由硬件信任根验证，但更新困难 |
| `logging` 模式下的篡改 | 只记录不阻断 |

这张表是回答「设备验证通过是否安全」的正确起点。

---

## 3. 成因分析

### 3.1 信任根在硬件，决定了「不可更新」是默认属性

信任根只能烧一次。这带来一个结构性后果：**任何上游的密钥问题都无法通过软件修复**。整个生态因此把大量精力放在「密钥不要泄露」上，而不是「泄露后怎么办」。

### 3.2 分段密钥导致治理复杂

每一段可能有不同的密钥与不同的签发方（SoC 厂商、OEM、运营商定制）。密钥越多，越容易出现「某一段用了弱密钥」或「某一段忘了更新」。

### 3.3 验证是静态的，攻击是动态的

AVB 保证的是「磁盘上的字节」，而真实攻击发生在内存里、在运行时。所以 AVB 是**基线**，不是**防线**。它把攻击门槛从「改文件」提高到「必须有运行时能力」，但没有消灭后者。

### 3.4 逃生门是必要的，也是危险的

`HASHTREE_DISABLED` 与 `VERIFICATION_DISABLED` 让产线、维修、开发成为可能。但它们也意味着设备可以处于「有 vbmeta、但没验证」的状态，而这种状态很难从外部一眼看出。

### 3.5 rollback index 依赖厂商实现

AOSP 提供了机制，但是否启用、存在哪里、如何更新，由厂商决定。于是「同一年份、同一补丁级别」的两台设备，防降级能力可能完全不同。

### 3.6 bootloader 是闭源定制件

SoC 厂商提供的 bootloader 是二进制，OEM 在其上做定制，安全团队无法审计。这条链的最前端恰好是透明度最低的一段。

---

## 4. 公开状态台账

| 版本 | 变化 |
|------|------|
| Android 4.4 | 引入基于 dm-verity 的 Verified Boot |
| Android 7.0 | 严格强制 Verified Boot |
| Android 8.0 | 包含 AVB（`vbmeta` + 描述符体系） |
| Android 10 | 动态分区（`super` + dm-linear）、APEX 引入 |
| Android 12+ | 密钥证明中引入 `verifiedBootHash`，把启动状态纳入证明 |

与前面几篇一致，本手册**不对 bootloader / AVB 相关 CVE 做逐条组件归属**：这类缺陷多在 SoC 厂商闭源组件中，公开公告的组件命名与补丁之间缺少稳定映射。

---

## 5. 防护技术

### 5.1 厂商侧

- **密钥分层与 HSM**：签名密钥不进构建机，签发走审批；对每个设备族使用独立密钥。
- **强制 rollback index**：把索引存进 RPMB 或 TEE 保护存储，并在启动时严格比较。
- **bootloader 解析器最小化**：在验证之前只解析必要字段；所有长度/偏移先做范围检查；对 vbmeta 做结构化模糊测试。
- **生产镜像禁用调试开关**：`avbctl` 之类工具在 user 构建中应不可用。
- **状态可视化**：启动时明确展示 `orange` / `red`，不要静默。

### 5.2 企业侧

- 基线里检查四项：`ro.boot.verifiedbootstate` 必须为 `green` 或受控的 `yellow`；`ro.boot.veritymode` 必须为 `enforcing`；`ro.boot.flash.locked` 必须为 `1`；`ro.boot.vbmeta.device_state` 必须为 `locked`。
- 把「允许解锁 bootloader」作为独立策略项，而不是默认放开。
- 对高敏感应用，结合密钥证明中的 `verifiedBootHash` 做设备状态绑定。

### 5.3 应用侧

- 通过 `KeyStore` 的密钥证明读取 `verifiedBootHash` 与设备锁定状态，用于风控**加分项**，不要作为唯一判据。
- 理解边界：证明反映的是**启动时**的状态，不反映运行时是否被注入。
- 不要自己实现「检测 root」的启发式规则来代替验证链，两者目的不同。

---

## 6. 自查与观测清单

```bash
# 1. 启动状态（只读属性，不需要 root）
getprop ro.boot.verifiedbootstate      # green / yellow / orange / red
getprop ro.boot.veritymode             # enforcing / logging / disabled
getprop ro.boot.flash.locked           # 1 = 锁定
getprop ro.boot.vbmeta.device_state    # locked / unlocked
getprop ro.boot.verifiedboothash       # 部分设备提供

# 2. 分区与验证设备
ls -l /dev/block/by-name/ | grep -i "vbmeta\|super\|boot"
mount | grep -i "dm-verity\|/dev/block/dm-"
cat /proc/cmdline | tr ' ' '\n' | grep -i "verity\|androidboot"

# 3. 补丁级别与构建信息
getprop ro.build.version.security_patch
getprop ro.build.fingerprint

# 4. APEX 与验证
apexctl list 2>/dev/null || ls /apex
```

判读要点：

- `verifiedbootstate=orange` 且 `flash.locked=0`：bootloader 已解锁，验证链不再作为信任依据。
- `veritymode=logging`：即便 `verifiedbootstate=green`，运行时篡改也不会被阻断。
- `ro.boot.verifiedboothash` 缺失：并非所有设备都提供，缺失本身不说明不安全。
- `mount` 中 `dm-verity` 只覆盖部分分区：属正常，说明该设备未对所有分区启用 hashtree。

---

## 7. 常见误传纠正

1. **「AVB 是 Android 7 引入的」**——错。7.0 强化的是 dm-verity；AVB 自 Android 8.0 起包含。
2. **「Verified Boot 保护用户数据」**——错。`/data` 由 FBE 负责，AVB 不覆盖。
3. **「dm-verity 会加密分区」**——错。它只做完整性校验，不提供机密性。
4. **「yellow 表示设备被 root」**——错。它表示「锁定 + 自定义密钥」。
5. **「orange 等于已经 root」**——错。它表示 bootloader 已解锁，是 root 的常见前提而非结果。
6. **「red 一定会阻止启动」**——不一定，取决于设备配置。
7. **「解锁后仍能通过所有完整性检查」**——主流风控与 DRM 会拒绝 `orange` 状态。
8. **「dm-verity 每次读都校验整棵树」**——错。逐块校验并缓存已验证节点。
9. **「veritymode=logging 也算有保护」**——错。只记录不阻断，运行时等同无保护。
10. **「vbmeta 是所有分区的签名集合」**——它是描述符集合，可含 hash / hashtree / chain 三类。
11. **「APEX 不受验证保护」**——受 dm-verity 保护，锚点为 APEX 自身签名。
12. **「verifiedBootHash 能证明设备未被入侵」**——它只反映启动时的验证状态。
13. **「有 vbmeta 分区就说明验证在跑」**——`VERIFICATION_DISABLED` 标志可以让它不跑。

---

## 未核实与失效来源

- 各 SoC 平台 bootloader 的具体解析实现与模糊测试覆盖情况：未核实，不做厂商级断言。
- rollback index 在各类设备上的存储位置（RPMB / TEE / 其他）：官方口径是防篡改存储，逐机型实现未核实。
- `ro.boot.verifiedboothash` 的提供范围：未核实，部分设备不提供。
- 本文未对 AVB / bootloader 相关 CVE 做编号归属，理由见第 4 节。

---

## 参考

- Verified Boot 概览：[source.android.com/docs/security/features/verifiedboot](https://source.android.com/docs/security/features/verifiedboot)
- AVB（`avbtool`、vbmeta 结构）：[source.android.com/docs/core/architecture/partitions/avb](https://source.android.com/docs/core/architecture/partitions/avb)
- dm-verity：[source.android.com/docs/security/features/verifiedboot/dm-verity](https://source.android.com/docs/security/features/verifiedboot/dm-verity)
- 设备状态与解锁：[source.android.com/docs/security/features/verifiedboot/device-state](https://source.android.com/docs/security/features/verifiedboot/device-state)
- 动态分区：[source.android.com/docs/core/architecture/partitions/dynamic-partitions](https://source.android.com/docs/core/architecture/partitions/dynamic-partitions)
- APEX：[source.android.com/docs/core/architecture/apex](https://source.android.com/docs/core/architecture/apex)
- 内核 dm-verity 文档：[docs.kernel.org/admin-guide/device-mapper/verity.html](https://docs.kernel.org/admin-guide/device-mapper/verity.html)
- `avbtool` 源码：[android.googlesource.com/platform/external/avb](https://android.googlesource.com/platform/external/avb)
- 密钥证明与 `verifiedBootHash`：[source.android.com/docs/security/features/keystore/attestation](https://source.android.com/docs/security/features/keystore/attestation)

---

## 联系方式

如果你在企业里需要对现网机型做「启动状态基线核查」「防降级能力评估」「vbmeta 与 verity 配置确认」，可以通过 Telegram 联系：<https://t.me/one00190>

---

*本手册仅用于防御与教育目的，讨论的均为公开且已被修复问题的机制与设计缺陷，不包含任何可直接利用的代码。文中标注「未核实」的内容不应作为事实引用；发现错误欢迎指出。*
