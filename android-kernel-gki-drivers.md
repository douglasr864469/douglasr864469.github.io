---
layout: page
title: "Android 内核与 GKI 驱动攻击面手册"
description: "官方根本没有「GKI 2.0」这个说法，现行口径是一条双条件强制句（Android 12 起且内核 ≥5.10 必须用 GKI）；`androidNN-K.K` 分支命名与 Android 12–17 的多内核并存矩阵、KMI 世代号写在 `build.config.common`、冻结后影响 KMI ..."
date: 2026-10-28
permalink: /android-kernel-gki-drivers/
---

> 「Android 内核漏洞」这个说法在中文资料里经常被写成一句结论——内核就是最大攻击面、提权必上 Critical、GKI 之后厂商驱动与内核解耦了。三句话都对了一半，但都对在错误的坐标上。真实情况是：**GKI 改变的从来不是"内核有多脆弱"，而是"脆弱的那部分被放到了哪个可交付单元里、由谁签名、随哪条更新通道走"**。本手册沿这条线把实名钉死：Generic Kernel Image 的官方强制口径到底是什么句式、`android14-6.1` 这类分支名怎么读、KMI 的世代号写在哪个文件里、`boot.img` 为什么在 Android 13 起只剩内核、`system_dlkm` 里的模块由谁签名、vendor hook 的 `android_rvh_` 前缀会给卸载行为留下什么后果，以及月度公告里内核那张表的列头和 Subcomponent 到底长什么样。定位是防御与教育：只引用公开且已修复的信息，不含任何可利用代码，也不给出可被直接抄用的注入或绕过步骤。

---

## 0. 使用方式与边界

**本手册的证据基础。** 所有官方口径都来自 source.android.com 的英文原文（`?hl=en` 抓取，避免页面按 IP 本地化成日/韩文而误读术语），引文以英文逐字给出，中文结论紧随其后。公告形态一节来自 **2025-12-01 期 Android Security Bulletin 的实测页面**，其中 CVE 编号、References 的 `A-<数字>`、Type 缩写、Severity、Subcomponent 全部是表格里可直接读到的单元格值。凡是没能取得英文原文的说法，一律降级到文末「未核实与失效来源」，正文不写。

**本手册刻意不做的事。**

- 不逐文件浏览 AOSP 内核源码树取证。因此像符号表文件名、defconfig 里逐条 CONFIG 这类"能在源码里翻到但文档没写"的细节，本手册写成"官方文档未列举"，而不是拿源码搜索结果冒充官方口径。
- 不给 CVE 补充公告之外的成因描述。公告内核行只有 `CVE / References / Type / Severity / Subcomponent` 五列，没有函数名、没有补丁片段，本手册也不代劳。
- 不讨论基带侧内核接口（QMI/MHI/QRTR 那一层），那部分归 [基带与电话子系统手册](/android-baseband-telephony/)。

**与相邻手册的分工。**

| 问题 | 归本手册 | 归哪本手册 |
| --- | --- | --- |
| 谁能调用这个 syscall、哪个域能碰哪个对象 | — | [SELinux 策略与 seccomp 沙箱手册](/android-selinux-seccomp-sandbox/) |
| 内核/模块被装进哪个镜像、由谁签名、随哪条通道更新 | ✅ | [系统更新链路与回滚防护手册](/android-ota-update-rollback/)（载荷与回滚索引） |
| GKI 模块作为可更新交付单元的版本/回滚语义 | 部分 | [Mainline 与 APEX 模块更新手册](/android-mainline-apex-modules/) |
| 公告字段的通用读法（CVE/SPL/OSV 对应关系） | 仅内核表 | [漏洞编号与公告体系手册](/android-cve-and-security-bulletins/) |
| 为什么内核本地提权能被评到 Critical | 部分 | [安全性评估与定级手册](/android-vulnerability-severity-assessment/) |
| 驱动/ioctl 类缺陷的成因模式 | — | [底层漏洞类型图谱手册](/android-low-level-vulnerability-types/)、[远程代码执行与权限提升](/android-rce-and-privilege-escalation/) |

---

## 1. GKI 的官方口径是一条强制要求，而不是一个版本号

很多资料把 GKI 写成"Android 11 引入、Android 12 演化为 GKI 2.0"。翻遍现行 GKI 页面，**"GKI 2.0" 这个字符串一次都没有出现**，官方给的是一条带版本与内核版本双条件的强制句：

> "Beginning with Android 12, devices shipping with kernel version 5.10 or higher must ship with the GKI kernel."
> — <https://source.android.com/docs/core/architecture/kernel/generic-kernel-image>

这句话里有两个条件轴，读公告和做设备评估时经常被丢掉一个：**Android 版本 ≥ 12**，并且**内核版本 ≥ 5.10**。落在任一条件之外的设备（例如仍跑 5.4 的 Android 12 设备）不在这条强制范围内——这解释了为什么现实里会同时存在"GKI 设备"和"仍用传统单体内核的设备"，也解释了公告里为什么还留着 `Kernel LTS` 这种按内核版本而不是按 Android 版本组织的表。

同一页给出的 GKI 定义，值得整句读：

> "The GKI kernel is built from the ACK sources."
> "The GKI kernel is a single-kernel binary plus associated loadable modules per architecture, per LTS release."
> "unifying the core kernel and moving SoC and board support out of the core kernel into loadable vendor modules"
> "a stable Kernel Module Interface (KMI) for vendor modules, so modules and kernel can be updated independently"
> "you can install these boot images without making changes to vendor images"

把五条拼起来，GKI 的实质是三件事：**每架构、每 LTS 一份统一的内核二进制**；**SoC 与板级支持被搬出内核主体，变成可加载的厂商模块**；**内核与模块之间承诺一个稳定的 KMI，让两者可以各自更新**。安全侧真正要紧的是第二件事——它把"最容易出问题的那部分代码"（GPU、显示、相机、触控、各类 platform driver）从"跟着内核版本走"改成了"跟着厂商交付单元走"，于是问题从"内核脆不脆弱"变成"边界画在哪、谁签名、怎么更新、能不能单独更新"。

**读法结论。** 判断一台设备是否属于 GKI 时代，不要靠"是不是 Android 12 以上"这个粗糙直觉，而是看两件事：内核版本是否 ≥ 5.10，以及分区表里是否存在 `system_dlkm`/`vendor_dlkm`（见 §4、§6）。

---

## 2. ACK 分支与版本矩阵：一个 Android 版本可以选多个内核

GKI 内核的源码不是 AOSP 的 `kernel/` 目录，而是 **Android Common Kernel（ACK）**：

> "also known as the Android common kernels or ACKs"
> "downstream of kernel.org kernels"
> "patches of interest to the Android community that haven't been merged into mainline"
> "the branches are named `ANDROID_RELEASE`-`KERNEL_VERSION`"
> "the 6.1 GKI kernel for Android 14 is named `android14-6.1`"
> "the two v5.15 kernels are `android13-5.15` and `android14-5.15`"
> — <https://source.android.com/docs/core/architecture/kernel/android-common>

分支命名规范本身就是可读的元信息：`androidNN-K.K` 的前半段是平台代次，后半段是上游内核版本。同一个上游内核版本可以被两个平台代次各承载一条分支（`android13-5.15` 与 `android14-5.15`），所以"5.15 内核"这种说法在跨设备比较时**没有区分力**，必须写全分支名。

该页的支持矩阵（每行是该 Android 版本可选用的 GKI 内核分支，逐字取自表格单元格）：

| Android 版本（页标年份） | 可用的 GKI 内核分支 |
| --- | --- |
| Android 12 (2021) | `android12-5.10` |
| Android 13 (2022) | `android13-5.15`、`android13-5.10`、`android12-5.10` |
| Android 14 (2023) | `android14-6.1`、`android14-5.15`、`android13-5.15`、`android13-5.10`、`android12-5.10` |
| Android 15 (2024) | `android15-6.6`、`android14-6.1`、`android14-5.15`、`android13-5.15`、`android13-5.10`、`android12-5.10` |
| Android 16 (2025) | `android16-6.12`、`android15-6.6`、`android14-6.1`、`android14-5.15`、`android13-5.15`、`android13-5.10`、`android12-5.10` |
| Android 17 (2026) | `android17-6.18` 起，向下直至 `android12-5.10`；其中部分条目页上标注 "not supported in Android 17 QPR1 or higher" |

三条对安全评估有直接意义的读法：

1. **多版本并存是设计而非例外。** 同一 Android 代次最多可挂六个 GKI 内核分支（Android 15/16 行）。所以"SPL 到了 X，我的内核分支一定修好了"是不成立的推断——修复是否落地取决于设备实际启动的那条 `androidNN-K.K` 分支，而不是取决于 Android 版本号。这与 [漏洞编号与公告体系手册](/android-cve-and-security-bulletins/) 里"SPL 与内核版本不等价"的结论同轴。
2. **上游支持周期缩短了。** 官方逐字句是 "Beginning with kernel 6.6, the support lifetime for the stable kernels is 4 years."。也就是说，越新的内核分支，其上游稳定维护窗口反而越短（此前更长），依赖"上游 LTS 一直在修"的时间表做长期设备规划要重新核对。
3. **新分支的推入节奏被 KMI 冻结牵制。** 该页另给出：每个上游内核版本原本可承载两条 GKI 分支，但从 Android 15 起官方口径为 "there is only one new GKI kernel per kernel version"；ACK 的 KMI 分支在新平台版本推入 AOSP 前会被冻结（frozen）。冻结的后果见 §3。

公告里还有一张专门按内核版本组织的表，列头逐字是 `"References"、"Android Launch Version"、"Kernel Launch Version"、"Minimum Update Version"`，行例 `"A-404245371" / "12" / "5.4" / "5.4.292"`。**这张表是"内核修到哪个具体上游小版本"的唯一权威位置**，它给出的粒度是 `5.4.292` 这种上游 stable 版本号，与 SPL 的日期粒度完全不同轴。做内核补丁核对时应该读这张表，而不是读月度公告的 CVE 列表。

---

## 3. KMI：稳定契约、世代号与"冻结之后"

KMI 是 GKI 全部更新语义的地基，官方术语实名：

> "a stable Kernel Module Interface (KMI) for vendor modules, so modules and kernel can be updated independently"
> "Any `CONFIG` change that affects the Kernel Module Interface (KMI) after it's frozen is rejected."
> — 前者出自 GKI 页；后者出自 <https://source.android.com/docs/core/architecture/kernel/kernel-code>

同一页还把 **"Changes to `KMI symbol lists`"** 单列为一类"上游 Linux 中不存在、属 Android 侧独有"的改动。注意：符号表的**具体文件名与本手册未取一手确认**（见未核实 B），所以本手册只写官方用的集合名词 KMI symbol lists，不写 `kmi_symbol_list` / `abi_symbollist` 这类文件名。

世代号是这里最容易读错的一项，逐字引文：

> "the _KMI generation number_ recorded in `build.config.common` is incremented"
> "The number after the platform release is the KMI generation (`6` in this case)."
> "the kernel isn't compatible with vendor modules that conform to the previous KMI generation"
> "The KMI can be extended with new exported symbols"

读法：KMI 世代号写在 `build.config.common` 里；跨世代即不兼容——**内核升一个 KMI 世代，同设备的全部厂商模块必须一起换**，没有"部分模块沿用旧 ABI"的中间态。同时官方明确 KMI 可以**扩展**（新增导出符号），这解释了为什么"稳定 ABI"不等于"ABI 永不变"：它是"在同一世代内不破坏"，而不是"跨世代也不破坏"。

**三条安全侧推论。**

- **兼容性检查就是供应链检查。** KMI 不匹配的后果是模块加载失败，而不是静默降级。因此一台设备能否接受某个"更新过的内核"，实际上由 KMI 世代号与符号表 CRC 决定；任何手工替换 boot/init_boot 而不换模块的做法，首先撞上的是这个契约。
- **CONFIG 变更被拒 ⇒ 内核加固项不由厂商决定。** "冻结后影响 KMI 的 CONFIG 变更会被拒绝"意味着 GKI 设备上的内核编译选项走向统一，厂商不能为了省事关掉某项加固来"兼容旧模块"。这既降低了配置漂移风险，也意味着 §8 列出的加固项在 GKI 设备上具备可比性——这是"GKI 让加固可核对"这条常被略过的好处。
- **vendor hook 的头文件规则暴露了真实痛点。** 官方要求 "The header files in `include/trace/hooks` should not include the kernel header file with the type definitions to avoid CRC changes which break the KMI." —— 换个说法：一个纯类型定义的头文件改动就能打破 KMI。这类"看不见的 ABI"正是本手册建议把 KMI 当供应链控制点来读的原因。

---

## 4. 镜像与分区：boot.img 里"只剩内核"改变了什么

GKI 时代最实用的一张图是启动镜像的职责切分。官方逐字：

> "the generic ramdisk is removed from the `boot` image and placed in a separate `init_boot` image."
> "This change leaves the `boot` image with only the GKI kernel."
> "the generic `boot` image, referred to as Generic Kernel Image (GKI), contains the generic ramdisk and the GKI kernel."
> "`init_boot` image added for devices launching with Android 13"
> "For devices without the init_boot partition, you need a ramdisk binary"
> — <https://source.android.com/docs/core/architecture/partitions/generic-boot>

同页给出的镜像头版本：init_boot 是 "Header version V4"，Generic `boot` image 是 "Header version V3 or V4"，`recovery` image 是 "Header version V2"。vendor_boot 的内容被逐条列为：`"vendor_boot` ramdisk image"、`"lib/modules`"、`"dtb` image"、"Recovery resources (if no dedicated recovery)"。

| 单元 | 装什么 | 谁负责 | 更新含义 |
| --- | --- | --- | --- |
| `boot.img` | Android 13 起只有 GKI 内核 | OEM 用设备专属 AVB key 签名（见 §5） | 可与 vendor 镜像无关地替换 |
| `init_boot.img` | generic ramdisk（Android 13 起从 boot 拆出） | 同上；建议 chained vbmeta | 拆分的目的是让 boot 保持"纯内核" |
| `vendor_boot.img` | vendor ramdisk、`lib/modules`、`dtb`、（无独立 recovery 时）recovery 资源 | 厂商 | **厂商内核模块的主要落点** |
| `system_dlkm` | Google 构建期签名的 GKI 模块 + `depmod` 产物（super 内动态分区） | Google 签名 | Android 13 起强制 |
| `vendor_dlkm` / ODM DLKM | 厂商/ODM 模块（也可能是 vendor_boot 承载） | 厂商 | first-stage init 加载 |

那句 "you can install these boot images without making changes to vendor images"（GKI 页）配上 §3 的 KMI 承诺，就是整张表的意义：**内核可以单独换，前提是模块与新内核的 KMI 对得上**。

**注意措辞。** generic-boot 页对 `boot.img` 明确写了签名主体，对 vendor_boot / dtbo 的签名密钥**原文未指明**；`dtbo.img` 的定义页本手册未抓到。因此不要把"所有启动镜像都由 OEM 的 vbmeta 链覆盖"当成官方逐字口径引用，它只能作为合理推断对待（见未核实 D）。AVB 与四色启动状态本身在 [TEE、密钥管理与设备认证手册](/android-tee-key-attestation/) 里有逐字口径，本手册不重复展开。

---

## 5. 两条签名主体：Google 签模块，OEM 重签 boot.img

这一节是 GKI 资料里流传最广、也最常被说反的一点。官方逐字：

> "The certified GKI `boot.img` isn't signed for verified boot."
> "OEMs must still sign the prebuilt `boot.img` with a device-specific AVB key."
> "Chained vbmeta is highly recommended for the `boot` and `init_boot` images to allow independent OTA."
> — <https://source.android.com/docs/core/architecture/partitions/generic-boot>

而 `system_dlkm` 里的模块走的是另一条：

> "Google build-time signed kernel modules"
> "signed by Google using the kernel build-time key pair"
> — <https://source.android.com/docs/core/architecture/partitions/gki-partitions>

两条合起来得到的真实模型是：**Google 提供未做 verified boot 签名的认证 GKI `boot.img`，OEM 必须用自己的设备专属 AVB key 重签；但 Google 构建期的 GKI 模块由 Google 的 kernel build-time key pair 签名。** 于是"内核镜像的签名主体"这个问题在 GKI 设备上答案不是一个而是两个，取决于被签的是 boot 里的内核还是 system_dlkm 里的模块。

安全上这条区分有直接后果：

1. **"Google 已经签好了，所以换内核就是换官方件"是错的。** OEM 重签意味着设备信任链里的 boot 哈希由厂商密钥决定；替换一个"未经本设备 AVB key 签名"的 boot 会直接把 `ro.boot.verifiedbootstate` 推到非 green（AVB 的判定与四色状态见 [TEE 手册](/android-tee-key-attestation/)）。
2. **chained vbmeta 是"独立 OTA"的前提。** 官方把 boot 与 init_boot 建议用链式 vbmeta，正是为了让内核侧可以随独立分区更新——这条与 [系统更新链路与回滚防护手册](/android-ota-update-rollback/) 里 A/B、Virtual A/B 的分区级更新语义直接对上；读那条链的完整性时应按分区维度而不是按"整个 boot"维度。
3. **模块签名是新的必查项。** 既然 SoC/板级代码以 `.ko` 形式交付，检查点就从"内核二进制的来源"扩展到"哪个密钥签了这个 .ko、它被允许从哪个分区加载"（§6）。

---

## 6. system_dlkm 与厂商模块的加载路径

`system_dlkm` 是 GKI 拆分后新增的那个交付单元，官方逐字：

> "reside on a separate dynamic partition in the super image called `system_dlkm`."
> "`depmod` artifacts"
> "There's no ABI stability between GKI and GKI modules."
> "Any device launching with Android 13 and higher must include a `system_dlkm` partition."
> 板级开关实名：`BOARD_USES_SYSTEM_DLKMIMAGE`、`TARGET_COPY_OUT_SYSTEM_DLKM`、`BOARD_SYSTEM_DLKM_SRC`（Android 13）、`BOARD_SYSTEM_KERNEL_MODULES`（Android 14）
> fstab 挂载选项示例中出现 `slotselect,avb`
> — <https://source.android.com/docs/core/architecture/partitions/gki-partitions>

注意那句看似矛盾、其实关键的话：**"There's no ABI stability between GKI and GKI modules."** —— 稳定的 KMI 是**内核 ↔ 厂商模块**之间的承诺；GKI 与**它自带的模块**之间反而明确不承诺 ABI 稳定，因此这两组必须成对更新。做完整性核对时，把 system_dlkm 与 boot 当作**一对**、把 vendor_dlkm 当作另一对，才符合官方语义。

厂商侧的加载路径：

> "The vendor boot partition stores kernel modules compatible with GKI, and is loaded by first stage init."
> — <https://source.android.com/docs/core/architecture/partitions/vendor-odm-dlkm-partition>
> "GKI modules required for the device should be loaded from /system/lib/modules which is a symbolic link to /system_dlkm/lib/modules ."
> — <https://source.android.com/docs/core/architecture/kernel/loadable-kernel-modules>

（后一条引文来自搜索摘要，URL 已确认但该页未整页抓取，引用时应标注为摘要级；见未核实附录。）

**为什么这是攻击面读法的核心。** 在 GKI 之前，SoC 驱动代码编译进内核镜像，改它就要改内核；在 GKI 之后，它是**独立交付、独立加载、独立被更新（或不更新）**的 `.ko`。攻击面没有增加，但**"修复要走哪条通道"变了**：同一个驱动缺陷，可能随 Google 的 GKI 模块更新，可能随厂商 vendor 镜像更新，也可能随厂商自己那份模块 blob 更新，三条通道的节奏和生效条件互不相同。这正是 [Mainline 与 APEX 模块更新手册](/android-mainline-apex-modules/) 反复强调的"版本轴与 SPL 轴不等价"在内核侧的镜像。

厂商模块的开发/构建框架（社区常称 DDK）与 `build_with_bazel.py`、`klm`/`kkm`/`TEST_KKM` 这类目标名，**本手册未能在官方页面取到任何字样**，因此不作为实名引用；可核实的替代事实是 §7 末的构建入口。

---

## 7. vendor hook：ACK 独有的机制，以及它留下的三个后果

内核侧的厂商扩展在 GKI 里被收进一个成文机制——vendor hook。官方逐字：

> "By convention, normal vendor hook names begin with `android_vh`, so the name for `sched_exit()` hook would be `android_vh_sched_exit`."
> "Restricted vendor hook names begin with `android_rvh`."
> "Normal vendor hooks use `DECLARE_HOOK()` to create a tracepoint function with the name `trace_name`"
> "Support for vendor hooks is only in ACK, so don't send these patches to upstream Linux."
> "Restricted vendor hooks can't be detached, so modules that attach to a restricted hook can never unload."
> — <https://source.android.com/docs/core/architecture/kernel/kernel-code>

同页还给出这类改动的归类（"上游不存在"的 Android 侧改动包括 "Changes to `KMI symbol lists`"）与若干硬性禁令：

> "Upstream Linux actively discourages support for building out-of-tree modules."
> "Don't add new sysfs nodes to the GKI kernel (such additions are valid only in vendor modules)."
> "Don't request the non-GPL variant, `EXPORT_SYMBOL()`."
> "To add a vendor hook, file an issue in IT and submit patches"

三个安全侧后果：

1. **受限 hook 改变模块生命周期。** "挂载 `android_rvh_` 的模块永远无法卸载"是官方逐字陈述。对运行时完整性检查的含义是：这类模块一旦加载就常驻，任何"通过卸载可疑模块来止血"的操作对它们无效；响应处置时能做的通常是重启而不是卸载。
2. **内核里多了一层"厂商逻辑入口"。** hook 本身是把控制权交给可加载模块（注册回调），因此**攻击面从"模块自己的代码"扩展到"内核在哪些点回调模块"**。审计时值得知道一台设备挂了哪些 hook；但 `include/trace/hooks` 下的完整清单与某设备实际注册的 hook 集合，**没有面向用户的官方查询接口**，本手册因此不给"一条命令看全"的假承诺。
3. **它解释了一批只在 Android 内核出现的缺陷位置。** vendor hook 与其生成的 `trace_*` 函数是 ACK 独有、上游没有的代码路径；上游的模糊测试覆盖与漏洞通报对这部分天然不对称。这与 §11 的"上游 CVE 与 Android 公告关系"是同一条线。

顺带给出可核实的内核构建入口（同 `building-kernels` 页逐字）：

> "Android 13 introduced building kernels with Bazel."
> "tools/bazel run //common:kernel_aarch64_dist [-- --destdir=$DIST_DIR]"
> "The GKI boot image is located in $DIST_DIR." / "The boot image is located at out/<kernel branch>/dist/boot.img."
> "The initramfs image is not embedded in the boot image."
> "repo init -u https://android.googlesource.com/kernel/manifest -b BRANCH"
> — <https://source.android.com/docs/setup/build/building-kernels>

配置文件实名 `gki_defconfig`；`kernel-code` 页还给出真实构建目标 `//build/kernel/static_analysis:checkpatch_presubmit`。这些值的用处在于：**要核对某条内核分支到底带没带某个补丁，可复现的路径是检出该 ACK 分支比对，而不是读设备上的属性。**

---

## 8. 内核加固：能一手实名引用的只有两项，别的多写都是冒险

中文资料常列出一长串"Android 内核加固配置"（`CONFIG_STACKPROTECTOR_STRONG`、`CONFIG_FORTIFY_SOURCE`、`CONFIG_INIT_ON_ALLOC_DEFAULT_ON`、`CONFIG_STRICT_DEVMEM`、`CONFIG_PROC_MEM_ALWAYS_RO` 等）。本手册按"官方文档页是否逐字出现"筛选，结果是**只有两项可以实名断言**。

第一项，控制流完整性：

> "CFI is enabled by default in GKI."
> "CFI is controlled by the CONFIG_CFI_CLANG option, which enables either Clang CFI or Clang KCFI."
> "Clang CFI requires compiling with Link Time Optimization (LTO)."
> "for Linux 6.0 and lower, Clang CFI, which relies on Clang LTO" / "for Linux 6.1 and higher, Clang KCFI"
> "enable CONFIG_CFI_PERMISSIVE, which prints out a warning instead of causing a kernel panic."
> — <https://source.android.com/docs/security/test/kcfi>

第二项，影子调用栈：

> "CONFIG_SHADOW_CALL_STACK=y"（"To enable ShadowCallStack for the kernel, add the following line to the kernel config file:"）
> "ShadowCallStack is only implemented for aarch64."
> "On aarch64, the instrumentation makes use of the `x18` register to reference the ShadowCallStack"
> — <https://source.android.com/docs/security/test/shadow-call-stack>

这两段合起来给的读法很有价值：**CFI 的实现随内核版本切换（≤6.0 走依赖 LTO 的 Clang CFI，≥6.1 走 Clang KCFI）**，所以"这台设备的内核有没有 CFI、是哪种 CFI"是可以从内核版本直接推的；`CONFIG_CFI_PERMISSIVE` 则提示一项真实风险——**permissive 模式只告警不 panic**，一台开了它的设备在日志里会留下 CFI 违规记录却不拦截，这属于"加固看似存在实则不生效"的典型形态，值得进自查清单。ShadowCallStack 的 `x18` 占用同时解释了为什么它只存在于 aarch64。

至于"GKI 用 Clang/LTO 构建"这句常见断言：官方页面的支撑是间接的（"CFI is enabled by default in GKI" + "Clang CFI requires compiling with Link Time Optimization (LTO)"），并没有 "The GKI kernel is built with Clang" 这样的直接句子。本手册按间接推论对待，不写成官方逐字。

内核侧的强制访问控制与 root 边界，官方 `kernel-security` 页只给泛化表述、**不含任何 CONFIG 清单**：

> "Android uses Security-Enhanced Linux (SELinux) to apply access control policies and establish mandatory access control (mac) on processes."
> "By default, on Android only the kernel and a small subset of the core services run with root permissions."
> "SELinux still constrains user space processes running as root."
> — <https://source.android.com/docs/security/overview/kernel-security>

第三句是本手册想单独标出来的一句：**root 不是终点**。它同时也是"内核提权之所以被评得很高，但并不等于拿到全部数据"的理由——越权后的对象访问仍要过 SELinux 矩阵与 seccomp 过滤，这两条轴的具体规则见 [SELinux 策略与 seccomp 沙箱手册](/android-selinux-seccomp-sandbox/)。至于 `/dev/mem`、`/dev/kmem` 与 `STRICT_DEVMEM` 的联合官方表述，该页没有出现，故本手册**不断言**（未核实 E/F）。

---

## 9. 公告里内核条目的真实形态

以下内容全部来自 <https://source.android.com/docs/security/bulletin/2025-12-01> 的实测页面。先给官方对表格的自述：

> "Issues are described in the tables below and include CVE ID, associated references, type of vulnerability, severity, and updated AOSP versions (where applicable)."
> "The severity assessment is based on the effect that exploiting the vulnerability would possibly have on an affected device," … "assuming the platform and service mitigations are turned off for development purposes or if successfully bypassed."
> "that are not publicly available have an \* next to the corresponding reference ID."

**内核表的列头逐字是：`CVE` | `References` | `Type` | `Severity` | `Subcomponent`。** 与相邻的 Framework/System 表（列头 `CVE` | `References` | `Type` | `Severity` | **`Updated AOSP versions`**）不同，内核表用 `Subcomponent` 取代了 `Updated AOSP versions`。这一处差别非常要紧：

- **`Subcomponent` 才是"驱动/子系统实名"字段。** 该期实测出现的值：`pKVM`、`IOMMU`、`Net`、`EPoll`、`XFRM`、`Kernel Virtual Machine`；厂商组分表里还有 `Modem`、`Mali`、`PowerVR-GPU`、`Bootloader`、`ims service`、`preloader`。写分析报告时应当直接引用这些值，而不是自己翻译成"GPU 驱动""虚拟化模块"这种查不到第二处的说法。
- **内核行不给"影响哪些 Android 版本"这一列。** 想知道某内核 CVE 波及哪些设备，要去找 §2 的 `Kernel LTS` 表（`Android Launch Version` / `Kernel Launch Version` / `Minimum Update Version`）与厂商组分表，而不是盯着内核 CVE 表推断。

**`Type` 与 `Severity` 的实测取值。** 该期内核行的 Type 只出现 `EoP`（提权）与 `ID`（Information Disclosure）；Severity 出现 `Critical` / `High` / `Moderate`。也就是说**内核本地提权可以被官方评到 Critical**（该期 `pKVM`、`IOMMU` 两条就是）。这纠正了"本地提权最多 High"的常见误记，级别口径的完整判读见 [安全性评估与定级手册](/android-vulnerability-severity-assessment/)。

**`References` 是 Android ID，不是 CVE。** 该列放 `A-<数字>`（如 `A-436580278`），CVE-ID 独立成列；细节未公开的条目在 ID 后带 `*`。厂商组分表常常**没有 `Type` 列**（只有 CVE / References / Severity / Subcomponent）。

**该期存在的小节结构**（内核相关部分）：`Kernel`、`Kernel LTS`、`Arm components`、`Imagination Technologies components`、`MediaTek components`、`Unisoc components`、`Qualcomm components`、`Qualcomm closed-source components`。厂商内核类缺陷不在"Kernel"小节里，而在各自厂商小节，靠 `Subcomponent` 表达组件——这与 [基带与电话子系统手册](/android-baseband-telephony/) 里"高通/联发科公告有自己的命名轴"的结论一致。

**两项本手册实测否掉的常见说法。** 2025-12 期的内核表**没有 `Execution privileges` 列，也没有 `Reported by` 列**；整页也**没有**"本地 vs 远程 / Low/None 权限"这类分级正文，只有上引那句评估依据并链接到独立的 severity 页面。所以"公告表格里有 Execution Privileges 说明内核本地提权"这类引用是不成立的。另外 URL 形态实测为 `…/docs/security/bulletin/YYYY-MM-01`；`2022-09-05` 这种形式在核验时为 **404**（历史期数需 `-01`/`-05` 两种都试或走站内搜索）。

---

## 10. 案例台账：可直接逐字引用的内核条目

下表全部为 2025-12-01 期公告页的逐字单元格值。**每条只写公告给了什么，不写成因、不写函数名、不写利用条件**——公告内核行没有这些字段，任何补充都是推测。

| CVE | References | Type | Severity | Subcomponent | 本手册的读法 |
| --- | --- | --- | --- | --- | --- |
| CVE-2025-48623 | A-436580278 | EoP | Critical | pKVM | 内核侧虚拟化组件（protected KVM）被评到最高档的本地提权 |
| CVE-2025-48624 | A-443053939 | EoP | Critical | IOMMU | 设备 DMA 隔离机制自身的提权——典型"驱动边界"缺陷 |
| CVE-2025-48637 | A-443763663 | EoP | Critical | pKVM | 同一组件同期第二条，说明组件级而非偶发 |
| CVE-2025-48638 | A-442540376 | EoP | Critical | pKVM | 同上 |
| CVE-2024-35970 | A-432804305 | EoP | High | Net | 上游编号（2024 序列）被 Google 以 Android ID 收录 |
| CVE-2025-38236 | A-432753641 | EoP | High | Net | 同一子组件同期多条 |
| CVE-2025-38349 | A-432751421 | EoP | High | EPoll | `epoll` 子系统；上游内核 CVE 进入 Android 公告的形态 |
| CVE-2025-48610 | A-432439762 | ID | High | Kernel Virtual Machine | 该期内核行里唯一的 `ID`（信息泄露） |
| CVE-2025-38500 | A-436201996 | EoP | Moderate | XFRM | 同一张表内出现 Moderate，说明内核并非一律高档 |
| CVE-2025-47351 | A-430043154 | （厂商表无 Type 列） | High | Kernel | Qualcomm components 小节；Subcomponent 直接写 "Kernel" |
| CVE-2025-47354 | A-430042895 | （同上） | High | Kernel | 同上 |
| CVE-2025-47382 | A-442620646 | （同上） | High | Bootloader | 同一厂商小节内组件跨到 bootloader |
| CVE-2025-6349 | A-428702264* | （同上） | High | Mali | Arm components；带 `*` = 细节未公开 |
| CVE-2025-8045 | A-443063131* | （同上） | High | Mali | 同上 |
| CVE-2025-6573 / 2025-25177 / 2025-46711 / 2025-58410 | 见公告 | （同上） | High | PowerVR-GPU | Imagination Technologies components，同期四条 |
| CVE-2025-20725 | A-442288325 | （同上） | High | ims service | MediaTek components，17 行全 High，多数 Subcomponent 为 "Modem" |
| CVE-2025-20730 | A-442288321 | （同上） | High | preloader | 同上；preloader 属启动早期组件 |
| CVE-2025-31717 | A-435065128 | （同上） | High | Modem | Unisoc components，13 行全 High、全 "Modem" |
| CVE-2025-61619 | A-448276914 | （同上） | High | Modem | 同上 |

**从这张表能安全得出的四条结论：**

1. **内核表的严重性跨度是 Moderate → Critical，不是"一律 Critical"。** 定级判读以 [安全性评估与定级手册](/android-vulnerability-severity-assessment/) 为准。
2. **同一 Subcomponent 常在同期出现多条**（pKVM 三条、Net 两条、Mali 两条、PowerVR-GPU 四条、MediaTek/Unisoc 各十几条）。做补丁优先级时应按**组件聚合**，而不是按单条 CVE 排——同一组件的多条修复通常来自同一轮审计，一次更新会一起落地。
3. **GPU 驱动是内核公告里的稳定大户**（`Mali`、`PowerVR-GPU` 都是内核侧驱动组件），且它们在厂商小节而非 `Kernel` 小节。这意味着**修复节奏由 GPU IP 厂商 + SoC 厂商两级决定**，与 AOSP 月度节奏不同轴；核对方式见 [漏洞编号与公告体系手册](/android-cve-and-security-bulletins/) 里"Android 与厂商公告分层结构"。
4. **`pKVM` / `Kernel Virtual Machine` 已经进入内核表**，说明"虚拟化组件本身是 Android 内核攻击面"不再是假设。同时 `IOMMU` 一条提醒：负责隔离的机制自己也会带缺陷，不能把某机制存在当成组件免检。

历史期数的经典内核 CVE（例如常被引用的 binder 类缺陷）本手册**未取得一手引文**，因此不列（未核实 J）；需要台账时按上表的现行形态去当期公告页取。

---

## 11. 上游 Linux CVE 与 Android 公告的关系

§10 表里 `EPoll`、`Net`、`XFRM` 三条挂的是 **2024/2025 序列的上游 Linux CVE 编号**，同时 References 是 Android ID。这个形态本身就是一条可读事实：**Google 已经把上游内核 CVE 引入公告，并按 Android 侧的追踪 ID 归因**，而不是只发自己的 `CVE-20xx-00NN` 命名空间。

于是三条常见误读可以直接纠正：

- **"公告里没有这条上游 CVE ⇒ Android 不受影响。"** 不成立。至少要看 §2 的 `Kernel LTS` 表（它按 `Kernel Launch Version` / `Minimum Update Version` 给上游 stable 版本号），还要看厂商小节；上游编号、厂商编号与 Android ID 三套编号的对应关系见 [漏洞编号与公告体系手册](/android-cve-and-security-bulletins/)。
- **"内核 CVE 编号在公告里 ⇒ 全部内核分支都受影响。"** 也不成立。是否受影响取决于设备实际那条 `androidNN-K.K` 分支与上游 stable 小版本，公告内核行**不给** `Updated AOSP versions` 列。
- **"SPL 到了就修好了。"** 内核侧最弱的一条推断，因为内核修复的权威位置是 `Kernel LTS` 表与厂商更新；SPL 与内核/模块版本不等价的完整论述见 [Mainline 与 APEX 模块更新手册](/android-mainline-apex-modules/)。

**要谨慎的一点。** Android 关于"不影响 Android 的上游内核 CVE 如何处理"的官方政策页，本手册按 `…/docs/security/testing/basics/cve-announcements` 抓取时返回 **404**，且预算内未定位到现行路径。因此**本手册不引用任何"官方声明：不收录不影响 Android 的上游 CVE"这类句子**，只把"公告用上游编号 + Android ID 双列"这一实测形态写为事实（未核实 G）。同理，内核模糊测试（syzkaller）在 Android 官方文档里的表述、以及常被引用的模糊测试研究及其 CVE，本手册**未取一手引文**，不作为实名依据（未核实 I）。

---

## 12. 常见误传纠正

| 流传说法 | 实际情况 | 依据 |
| --- | --- | --- |
| "Android 11 引入 GKI，Android 12 升级为 GKI 2.0" | 官方口径是强制句：Android 12 起内核 ≥5.10 的设备必须用 GKI；**"GKI 2.0" 在现行 GKI 页面不存在** | §1；未核实 A |
| "GKI = Android 版本与内核版本一一对应" | 一个 Android 版本可挂最多六个 GKI 内核分支（Android 15/16 行）；同一上游版本还能被两个平台代次各承载一条 | §2 矩阵 |
| "KMI 稳定 = 内核 ABI 永不变" | KMI 可**扩展**新导出符号；但跨 **KMI 世代**不兼容，世代号写在 `build.config.common` | §3 |
| "GKI 和它的模块之间也 ABI 稳定" | 官方逐字相反："There's no ABI stability between GKI and GKI modules." | §6 |
| "Google 已经签好了 GKI boot.img，所以可以直接换" | "The certified GKI `boot.img` isn't signed for verified boot." OEM 必须用设备专属 AVB key 重签 | §5 |
| "内核镜像由 Google 签名 / 由 OEM 签名"（二选一） | 两个签名主体并存：boot 由 OEM 重签；system_dlkm 内的 GKI 模块 "signed by Google using the kernel build-time key pair" | §5 |
| "厂商不能改内核配置，所以配置都一样" | 方向对但理由错：官方句是"冻结后影响 KMI 的 CONFIG 变更会被拒绝"，这是 ABI 约束而非安全条款；两者后果恰好一致 | §3 |
| "Android 内核加固清单包括 FORTIFY_SOURCE / STACKPROTECTOR_STRONG / STRICT_DEVMEM…" | 官方文档页**只逐字给出** `CONFIG_CFI_CLANG`、`CONFIG_CFI_PERMISSIVE`、`CONFIG_SHADOW_CALL_STACK`；其余未在任何抓到的页面出现 | §8；未核实 E |
| "GKI 内核一定有 CFI 所以不怕控制流劫持" | `CONFIG_CFI_PERMISSIVE` 只告警不 panic；且 ≤6.0 走 Clang CFI、≥6.1 走 Clang KCFI，形态随内核版本变 | §8 |
| "ShadowCallStack 全平台都有" | "ShadowCallStack is only implemented for aarch64."（占用 `x18`） | §8 |
| "公告内核表有 Updated AOSP versions 列" | 内核表用 `Subcomponent`；`Updated AOSP versions` 属于 Framework/System 表 | §9 |
| "公告内核表有 Execution Privileges: Low/None 说明本地提权" | 2025-12 期内核表**没有** Execution privileges、也没有 Reported by 列；页面只链接独立 severity 页 | §9 |
| "内核本地提权最多 High" | 该期 `pKVM`、`IOMMU` 为 **Critical**；同时 `XFRM` 为 Moderate，跨度真实存在 | §9、§10 |
| "公告里有 GPU 驱动缺陷 ⇒ 属于 Kernel 小节" | GPU（`Mali`、`PowerVR-GPU`）在 **Arm / Imagination Technologies components** 小节，靠 Subcomponent 表达 | §9、§10 |
| "老公告 URL 形如 `bulletin/2022-09-05`" | 现行实测形态 `bulletin/YYYY-MM-01`；`-05` 形式在核验时 404，历史期需两种都试 | §9 |
| "vendor hook 可以随意增删来打补丁" | 加 hook 需 "file an issue in IT and submit patches"；且 `android_rvh_` 受限 hook 不可摘除，挂它的模块**永远无法卸载** | §7 |
| "DDK、`build_with_bazel.py` 是官方构建入口" | 官方页可核实的入口是 Bazel/Kleaf：`tools/bazel run //common:kernel_aarch64_dist`（Android 13 引入 Bazel）；DDK 类名称未在官方页面出现 | §7；未核实 C |
| "sysfs 里能看到内核模块状态，所以能据此判断是否 GKI" | 官方明令 "Don't add new sysfs nodes to the GKI kernel (such additions are valid only in vendor modules)." —— 新 sysfs 节点只应来自厂商模块 | §7 |

---

## 13. 观测点与自查

以下清单只列**可由设备端命令或文件系统直接读到**的项，并逐项标注性质。凡是"官方文档说 X，但设备上无法直接验证"的关系，本手册写在 §5、§7 而不是这里。

**A. 内核与分支身份（判定 GKI 时代坐标）**

1. `adb shell uname -r` — 取内核版本串。GKI 设备的版本串常带 ACK 分支痕迹，但**格式不是官方成文契约**，只作线索，不作判据。
2. 设置里的内核版本行（关于手机 → 版本信息）与 `uname -r` 应一致；不一致说明读到的是不同来源的信息，需复核。
3. 内核版本 ≥5.10 且 Android ≥12 ⇒ 落在 §1 的强制范围内；两者任一不满足就不是"GKI 强制设备"。

**B. 分区与交付单元（§4 的直接验证）**

4. `adb shell ls -l /dev/block/by-name/`（或 OEM 等价路径）— 看是否存在 `boot`、`init_boot`、`vendor_boot`、`system_dlkm`、`vendor_dlkm`。有 `init_boot` 通常对应 Android 13 起的拆分布局；缺 `init_boot` 则按 generic-boot 页那句 "For devices without the init_boot partition, you need a ramdisk binary" 对待。
5. `adb shell mount | grep dlkm` — 确认 `system_dlkm`/`vendor_dlkm` 是否作为独立分区挂载，以及挂载选项里是否出现 `avb`（§6 的 fstab 示例值是 `slotselect,avb`）。
6. `adb shell ls -l /system/lib/modules` — 按官方口径它应是指向 `/system_dlkm/lib/modules` 的符号链接（摘要级引文，需现场确认）。

**C. 模块与加载面**

7. `adb shell cat /proc/modules` — 已加载模块列表。重点看数量级与厂商模块占比：GKI 之后 SoC/板级代码就是在这里，而不是在内核二进制里。
8. `adb shell ls /vendor/lib/modules /odm/lib/modules 2>/dev/null` — 厂商模块文件的落点；`vendor_boot` 内的 `lib/modules` 由 first-stage init 加载（§6 引文），因此这里的文件与 `/proc/modules` 之间应能对应。
9. 内核符号导出面：`adb shell cat /proc/kallsyms` 在多数生产内核上对非 root 不可读，**因此不要把它当成可行自查项**；能读与不能读本身就是配置差异，记录结论即可，不要试图提升权限。

**D. 加固是否真生效**

10. `CONFIG_CFI_PERMISSIVE` 的后果可以在 dmesg 里被动观测：permissive 模式的 CFI 违规是**告警**而非 panic。若日志里出现此类告警而系统未终止，说明处于 permissive 形态——这是"加固看似存在实则不拦截"的实例。读取内核日志需权限，未授权设备上应记录为"无法验证"。
11. 内核版本与 CFI 形态可按 §8 的官方切分推：`Linux ≤6.0` → Clang CFI（依赖 LTO）；`≥6.1` → Clang KCFI。这是**唯一不需要设备权限就能得到**的加固信息。
12. ShadowCallStack 只对 aarch64 成立；x86 设备上不要按"有 SCS"评估。

**E. 更新与修复核对**

13. `adb shell getprop ro.build.version.security_patch` — SPL；按 §11，它**不能**用来确认内核修复是否到位。
14. `adb shell getprop ro.boot.verifiedbootstate` — 启动状态（AVB 侧口径见 [TEE 手册](/android-tee-key-attestation/)）。替换未由本设备 AVB key 重签的 boot（§5）会在这里体现。
15. 内核修复的权威核对位置是当期公告的 **`Kernel LTS` 表**：读 `Kernel Launch Version` + `Minimum Update Version`（形如 `5.4` → `5.4.292`），与设备 `uname -r` 的上游小版本比对。**这是本手册认为唯一可靠的"内核修没修"判据。**
16. GPU / 厂商内核缺陷的核对要落到对应厂商小节（`Mali`、`PowerVR-GPU`、`Kernel` + Qualcomm、`Modem` + MediaTek/Unisoc）与厂商自己的公告，按 [漏洞编号与公告体系手册](/android-cve-and-security-bulletins/) 的分层结构走。

**F. 处置侧的两条硬约束（来自 §7、§5）**

17. 挂了 `android_rvh_` 受限 hook 的模块**无法卸载**——"卸载可疑模块"不是可用处置手段，止血路径是更新或重启。
18. boot/init_boot 的独立 OTA 依赖 chained vbmeta（§5 官方建议）。若一台设备把 boot 与 vbmeta 顶层绑死，内核侧单独更新的可行性就低；这属于评估"能不能只修内核"时的实际约束。

---

## 14. 与其他手册的分工

- 想弄清楚**进程侧能不能调用某个 syscall、某个域能不能碰某个对象**：[SELinux 策略与 seccomp 沙箱手册](/android-selinux-seccomp-sandbox/)。本手册只负责内核与模块这一层——越权之后仍要过那两条轴，这也是"内核提权 ≠ 拿走全部数据"的原因。
- 想把**内核/模块更新**放进完整更新链（A/B、Virtual A/B、载荷验签、回滚索引）里看：[系统更新链路与回滚防护手册](/android-ota-update-rollback/)。
- 想比较**多种交付单元的更新通道**（APEX、Mainline、扁平化 APEX、厂商 blob）：[Mainline 与 APEX 模块更新手册](/android-mainline-apex-modules/)；§11 的"SPL 与内核版本不等价"是那条结论在内核侧的投影。
- 想把**AVB、链式 vbmeta、启动状态**作为信任根理解（而不是只当自查项）：[TEE、密钥管理与设备认证手册](/android-tee-key-attestation/)。
- 想读**缺陷成因模式**（驱动 ioctl、use-after-free、缺少边界检查如何变成提权）：[底层漏洞类型图谱手册](/android-low-level-vulnerability-types/) 与 [远程代码执行与权限提升](/android-rce-and-privilege-escalation/)。本手册刻意不给成因，以免超出公告字段。
- 想搞清**编号体系**（Android ID / CVE / 厂商编号）与公告分层：[漏洞编号与公告体系手册](/android-cve-and-security-bulletins/)。
- 想判**级别为什么是 Critical / Moderate**：[安全性评估与定级手册](/android-vulnerability-severity-assessment/)。
- 基带处理器那一侧的内核接口（QMI/MHI/QRTR/SMD/BAM/CCCI）不在本手册范围：[基带与电话子系统手册](/android-baseband-telephony/)。
- 面向开发者的落地清单：[通用漏洞类型手册](/android-common-system-vulnerabilities/)。

---

## 参考

**官方一手来源（本手册逐字引用过）**

- Generic Kernel Image — <https://source.android.com/docs/core/architecture/kernel/generic-kernel-image>
- Android Common Kernel — <https://source.android.com/docs/core/architecture/kernel/android-common>
- Develop kernel code for GKI — <https://source.android.com/docs/core/architecture/kernel/kernel-code>
- Generic boot image（含 `init_boot`）— <https://source.android.com/docs/core/architecture/partitions/generic-boot>
- Implement a GKI module partition — <https://source.android.com/docs/core/architecture/partitions/gki-partitions>
- Vendor/ODM DLKM 分区 — <https://source.android.com/docs/core/architecture/partitions/vendor-odm-dlkm-partition>
- Loadable kernel modules — <https://source.android.com/docs/core/architecture/kernel/loadable-kernel-modules>
- Building kernels — <https://source.android.com/docs/setup/build/building-kernels>
- Control flow integrity in the kernel (KCFI) — <https://source.android.com/docs/security/test/kcfi>
- Shadow call stack — <https://source.android.com/docs/security/test/shadow-call-stack>
- Kernel security（概述）— <https://source.android.com/docs/security/overview/kernel-security>
- Android Security Bulletin 2025-12-01 — <https://source.android.com/docs/security/bulletin/2025-12-01>

**相邻手册**：见 §14 列出的相对链接。

---

## 未核实与失效来源

以下条目**没有写进正文的事实断言**（正文相应处均已标注"未核实"或改用可核实替代）。列出来是为了防止后续版本把它们又当成已知事实补回去。

**A. "GKI 2.0" 的官方定义与所属 Android 版本**
推测：社区普遍把 Android 12 + 稳定 KMI + 模块化厂商驱动这套架构称作 "GKI 2.0"（相对更早基于 5.4 的探索）。未核实原因：`generic-kernel-image?hl=en` 整页抓取后确认**不含 "GKI 2.0" 字符串**；`.../kernel/gki-versions` 返回 **404**。正文一律改用"Android 12 起 5.10+ 必须使用 GKI"这条强制句。

**B. KMI 符号表的文件名与 AOSP 路径**
推测：`kmi_symbol_list` / `abi_symbollist` / `gki_kmi_symbol_list` 一类名称常见于构建产物与讨论，但**均未取一手**。未核实原因：只在 android-common 页侧栏看到链接文字 "Work with symbol lists"，未抓取该页；带 `allowed_domains` 的站内检索返回 0 结果。正文因此只使用官方集合名词 **KMI symbol lists**。

**C. DDK 与其构建入口、目标名**
推测：DDK（Driver Development Kit）是厂商内核模块的开发/构建框架，入口曾为 `build_with_bazel.py`，目标名包含 `klm`/`kkm`/`TEST_KKM`，路径 `kernel_platform`。未核实原因：`.../kernel/ddk-intro` 返回 **404**；`building-kernels` 与 `kernel-code` 两页**均未出现 DDK、build_with_bazel.py、klm/kkm/TEST_KKM、kernel_platform 任何字样**。正文改用可核实的 `tools/bazel run //common:kernel_aarch64_dist` 与 "Android 13 introduced building kernels with Bazel."

**D. verity 模式与 vendor_boot / dtbo 的签名主体**
推测：AVB 2.0 下 boot/vendor_boot/init_boot/dtbo 都由 OEM 的 vbmeta 链覆盖；`dtbo.img` 是独立的设备树 overlay 分区；verity mode 0/1/2 的说法来自 verified boot 文档。未核实原因：抓取预算用于分区与公告页，未取 AVB/dtbo 定义页；`generic-boot` 页只对 `boot.img` 明确签名主体，未对 vendor_boot/dtbo 指明密钥来源。

**E. 其余内核加固 CONFIG 名**
未取得一手：`CONFIG_ARM64_PTR_AUTH_KERNEL`、`CONFIG_STACKPROTECTOR_STRONG`、`CONFIG_FORTIFY_SOURCE`、`CONFIG_INIT_ON_ALLOC_DEFAULT_ON`、`CONFIG_RODATA_FULL_DEFAULT_ENABLED`、`CONFIG_STRICT_DEVMEM`、`CONFIG_PROC_MEM_ALWAYS_RO`。正文只断言 `CONFIG_CFI_CLANG`、`CONFIG_CFI_PERMISSIVE`、`CONFIG_SHADOW_CALL_STACK` 三项，其余写作"官方文档页未逐项列举 CONFIG"。

**F. `/dev/kmem`、`/dev/mem` 与 `STRICT_DEVMEM` 的联合官方表述**
`kernel-security` 整页无 `/dev/mem`、`/dev/kmem`、`STRICT_DEVMEM` 字样，只有 SELinux/root 权限的泛化描述（正文 §8 引用的三句）。因此正文不判断这些设备节点是否被禁。

**G. 「不影响 Android 的上游内核 CVE 如何处理」的政策页**
`.../docs/security/testing/basics/cve-announcements` 实测 **404**（路径已废弃或迁移），未能在预算内定位现行页面。正文因此只陈述实测形态（上游编号与 Android ID 双列），不引用任何"官方声明不收录"的句子。

**H. Android 严重性分级页关于本地/远程的措辞**
2025-12-01 期公告**不含** `Execution privileges` 列，也没有本地/远程分级正文，只链接独立 severity 页；该页正文未抓取。故 §9 只写"内核本地提权可被评到 Critical"这一实测列值事实，不引用分级页措辞。

**I. Syzkaller / 模糊测试研究与 CVE-2022-20421**
推测：Google 有 Android 专用 syzkaller 分支（AOSP `external/syzkaller`），并有以覆盖率引导模糊测试发现厂商驱动漏洞的公开研究（常称 "Fuzzler"）。未核实原因：站内检索 `syzkaller kernel fuzzing Android security testing documentation` 返回 **0 结果**；对应历史公告期 URL `2022-09-05` 形式 404，未取得一手引文。正文不点名 syzkaller 与 Fuzzler。

**J. 历史内核 CVE 的公告措辞**
CVE-2019-2215（binder 类）、CVE-2020-0030、CVE-2021-0439、CVE-2023-20941，以及 Component 字符串 "Kernel components (specific to Huawei devices)"：**均未取得一手引文**（历史期公告页未在预算内抓取，且 `-05` 形式 URL 404）。正文的案例只使用 2025-12-01 期逐字值。

**K. 版本串格式、`/proc` 可读性与设备侧观测**
`uname -r` 的 ACK 分支痕迹格式、`/proc/kallsyms` 在各生产内核上的可读性、`/dev/block/by-name/` 的存在与否，均属**设备实况**而非官方成文契约；§13 已逐项标注为线索而非判据。厂商定制可能使其中某些路径不存在。

**L. 抓取经验（写进参考以便复现）**
source.android.com 会按访问 IP 本地化正文语言，**必须在 URL 后加 `?hl=en`** 才能拿到英文原文；否则容易把翻译文本当成官方术语。公告 URL 现行形态实测为 `bulletin/YYYY-MM-01`。

---

## 联系方式

咨询 Android 系统详情请联系 https://t.me/one00190

---

*本手册仅供安全学习与防御研究使用；所有分支名、分区与镜像名、构建目标、CONFIG 名、公告列名与案例归属均取自公开一手来源并附链接，未能核实的表述已单列于「未核实与失效来源」而非写入正文。*
