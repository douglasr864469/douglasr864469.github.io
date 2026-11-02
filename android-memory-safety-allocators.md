---
layout: page
title: "Android 原生内存安全与分配器手册"
description: "四道性质不同的防线：Scudo 的 chunk header 校验与 quarantine（Android 11 起默认，但低内存设备仍用 jemalloc）、MTE 的 tag 传递与 SYNC/ASYNC 失败语义（App 默认关闭、需 bootloader 配合、内核 MTE-KASAN 是另一套..."
date: 2026-11-02
permalink: /android-memory-safety-allocators/
---

> 「内存安全」在大多数资料里被处理成一份工具清单：ASan、HWASan、MTE、Scudo、Rust。清单本身没错，但它丢掉了三件真正决定防护效果的事——**每件工具拦的是什么、不拦什么，开销落在谁身上，以及它到底能不能进量产**。本手册把 Android 原生内存安全拆成四道性质不同的防线：分配器硬化（Scudo 的 chunk header 与 quarantine）、硬件内存标记（MTE 的 tag 传递与同步/异步取舍）、运行时检测（HWASan / GWP-ASan / KASAN）、以及消除整类缺陷（零初始化与 Rust）。每道防线都用官方原文钉死其边界，并在最后给出一张"机制 × 拦什么 × 不拦什么 × 成本 × 可否量产"的对照表。定位是防御与教育：只引用公开且已修复的信息，不含任何可利用代码。

---

## 0. 使用方式与边界

**本手册的证据基础。** 官方口径来自 source.android.com 与 developer.android.com 的英文原文（`?hl=en` 抓取）、Android 官方博客（security.googleblog.com、blog.google）、LLVM Scudo 文档、Linux 内核文档，以及 Android Security Bulletin 与 Linux CNA 的 CVE 记录。引文逐字给出并附链接，中文结论紧随其后，**同时写明引文支撑的边界在哪**——例如官方说 "can augment"（能力）时，本手册不会写成"默认对所有进程开启"。

**本手册刻意不做的事。**

- 不给任何堆布局、绕过或利用步骤。分配器与 tag 机制的绕过材料具体化即为攻击工具，本手册只讨论机制、成因与防护边界。
- 不把"默认开启"当作无条件的结论。Scudo 有低内存设备例外、MTE 对 App 默认关闭、zero-init 不覆盖 zygote 与 App——这些条件在原文里都写着，本手册逐条保留。
- 不引用 AOSP 源码中的具体常量、配额与函数名。凡需查源码才能确认的（quarantine 默认配额、BoundsSan 覆盖组件清单），一律进文末「未核实」。
- 不重复内核侧的攻击面划分（GKI 镜像、模块、签名主体）与 IPC 组件导出面，那两部分分别归相邻手册。

**与相邻手册的分工。**

| 问题 | 归本手册 | 归哪本手册 |
| --- | --- | --- |
| 分配器、MTE、检测工具、零初始化、Rust 的定位与边界 | ✅ | — |
| 内核镜像、GKI 模块、KMI 与内核加固 CONFIG 的选择 | 仅内存相关项 | [内核与 GKI 驱动攻击面手册](/android-kernel-gki-drivers/) |
| 应用沙箱、SELinux 域、seccomp 白名单 | — | [SELinux 策略与 seccomp 沙箱手册](/android-selinux-seccomp-sandbox/) |
| 组件导出、PendingIntent、Binder 接口面 | 仅 binder 的内存生命周期 | [框架 IPC 与组件导出攻击面手册](/android-framework-ipc-export/) |
| 缺陷类型图谱（按组件定位） | 仅内存破坏类 | [底层漏洞类型图谱手册](/android-low-level-vulnerability-types/) |
| 公告字段读法、CVE 与 Android ID | 仅本条案例 | [漏洞编号与公告体系手册](/android-cve-and-security-bulletins/) |
| Critical/High 的定级与可利用性要素 | — | [安全性评估与定级手册](/android-vulnerability-severity-assessment/) |

---

## 1. 先把问题量化：内存安全曾是 Android 漏洞的绝对主因

两份官方数据钉住了这条曲线的两端。第一份来自 CFI 文档，是一个历史基线：

> "As of 2016, about 86% of all vulnerabilities on Android are memory safety related."
> — <https://source.android.com/docs/security/test/cfi>

第二份来自 2025 年的官方博客，是当前口径：

> "We adopted Rust for its security and are seeing a 1000x reduction in memory safety vulnerability density compared to Android’s C and C++ code."
> "The 2025 data continues to validate the approach, with memory safety vulnerabilities falling below 20% of total vulnerabilities for the first time."
> — <https://blog.google/security/rust-in-android-move-fast-fix-things/>

从 86% 到不足 20%，中间是十年。**这两条数据最重要的用法不是"证明某措施有效"，而是防止把功劳记在单一措施上。** 这十年里同时发生的事至少有四件：新代码大量转向 Rust、分配器从 jemalloc 换成 Scudo 并硬化、MTE 从硬件到内核到用户态逐步接通、以及平台代码的零初始化。任何"因为上了 X，所以内存漏洞降了"的单因果叙事，都无法与这四条并存的事实相符。

**读法结论。** 评估一台设备或一个版本的内存安全水位时，不要看"有没有 MTE"或"是不是用了 Scudo"，而是看**四道防线各覆盖到哪一层**：分配器硬化覆盖所有原生分配、硬件标记覆盖哪些进程、检测工具在哪些构建里生效、以及新代码的语言结构。这四件事的覆盖范围互不替代。

---

## 2. 第一道防线：Scudo 是默认分配器，但"默认"有条件

Scudo 的默认地位有明文，且**同一句话里就带着例外**：

> "Since the Android 11 release, scudo is used for all native code (except on low-memory devices, where jemalloc is still used)."
> "Scudo is currently the default allocator in Fuchsia, and in Android since Android 11."
> — <https://source.android.com/docs/security/test/scudo>
> — 同句亦见 <https://llvm.org/docs/ScudoHardenedAllocator.html>

这条例外在评估里非常实用：**"Android 11 及以上因此都用 Scudo"是一个错误推断。** 低内存设备仍跑 jemalloc，而低内存设备恰恰是预算受限、加固选项更容易被裁掉的一类。写设备评估时应当把"是否低内存设备"作为一个独立条件列出，而不是用系统版本号替代。

第二件必须先说清的事是 Scudo 的定位——它自己就不自称检测器：

> "Scudo is more of a mitigation than a fully fledged memory error detector like AddressSanitizer (ASan)."
> "the process is aborted if a corruption or suspicious behavior is detected in the heap"
> "Scudo is open source and part of LLVM's compiler-rt project."
> — <https://source.android.com/docs/security/test/scudo>

三句话各对应一个结论：

1. **它是缓解，不是检测器。** 缓解的意思是"让某类缺陷更难被稳定利用"，而不是"报告出缺陷的位置与类型"。把 Scudo 当成"Android 自带 ASan"是最常见的误解之一。
2. **检出后的动作是终止进程。** "aborted" 是硬失败，不是记录后继续。这决定了它在量产上的可行性（无额外常驻开销的检测不可得，但代价可控），也决定了它的观测价值——一次 abort 就是一条可观测事件。
3. **它来自 LLVM compiler-rt 上游。** 这意味着它的行为与文档是公开可比对的，不是厂商私有实现；评估时可以拿上游文档核对产品行为。

---

## 3. chunk header 与 quarantine：Scudo 的两件基础硬化

Scudo 的基础硬化是把元数据放在用户数据旁边并加校验：

> "Every chunk of heap memory returned to an application by the allocator will be preceded by a header."
> "the header is checksummed and corruption of the header will be detected when said header is accessed"
> "corrupted chunk header: The checksum verification of the chunk header has failed."
> — <https://source.android.com/docs/security/test/scudo>；<https://llvm.org/docs/ScudoHardenedAllocator.html>

机制读法：header 与用户数据相邻，所以**一次向前的溢出会改写 header**；当分配器在 free/realloc 路径上访问该 header 时，校验和失败暴露出来。这条链路有三个必然的盲区，官方文档自己写了第一个：

- **未被访问的损坏 header 不会被发现。** LLVM 文档明确这一点。也就是说这是"访问触发的抽样检测"，不是全程监控——溢出发生后，如果那块内存再没被分配器碰过，什么都不会发生。
- **向后（低地址）方向的越界与 header 无关。** header 在数据前面，保护的是"向上溢出"这个方向。
- **已经释放但仍在被使用的内存（UAF）本身不产生 header 校验失败。** 这类缺陷要靠 quarantine 制造时间窗。

quarantine 就是为此存在的：

> "the Quarantine: offers a way to delay the deallocation operations, preventing blocks to be immediately available for reuse"
> "This feature is fairly costly in terms of performance and memory footprint, is mostly controlled by runtime options and is disabled by default."
> "The size (in KB) of quarantine used to delay the actual deallocation of chunks."
> — <https://llvm.org/docs/ScudoHardenedAllocator.html>；<https://source.android.com/docs/security/test/scudo>

三条边界必须分开记：

1. **quarantine 用内存与性能换"UAF 检出窗口"。** 释放的块不立即回到可用池，于是"释放后还在被写"的那次写会落在一块已被标记为隔离的内存上。
2. **上游默认是关闭的。** 这句是 LLVM 文档的原话（"disabled by default"）。
3. **Android 产品上的默认配额，AOSP 页面没有给。** 该页只列了 `QuarantineSizeKb` 这类可调项。**因此本手册不写"Android 默认开启 quarantine、配额为 N"**——这个结论需要读产品配置才能得到，属未核实 E。

**这对评估意味着什么。** 如果一份分析把"Scudo 有 quarantine"当成"UAF 会被检出"，那它至少跳过了两步：这一步是否在目标产品上开启，以及配额是否足够覆盖真实的释放-再用间隔。前者是配置问题，后者是容量问题，两者都不是"有这项功能"能回答的。

---

## 4. 第二道防线：MTE，把 tag 从分配器一路带到 CPU

MTE 是这套体系里唯一由**硬件**参与判断的一层。它的官方定义在两个来源里几乎同义，但**硬件代次的表述不一致**，这本身就是一条值得记录的细节：

> "Arm v9 introduces Arm Memory Tagging Extension (MTE), a hardware implementation of tagged memory."
> — <https://source.android.com/docs/security/test/memory-safety/arm-mte>
>
> "ARMv8.5 based processors introduce the Memory Tagging Extension (MTE)"
> — <https://www.kernel.org/doc/html/latest/arch/arm64/memory-tagging-extension.html>

**处理方式：** 本手册写"MTE 自 Armv8.5-A 起引入（Android 安全文档以 Arm v9 表述）"，并同时给两个链接。凡是只引一方、把另一方当错的资料，都会在跨文档核对时被发现自相矛盾。

MTE 进入 Android 的方式是**从分配器开始**的，而不是一个全局开关：

> "In Android 12 the kernel and userspace heap memory allocator can augment each allocation with metadata."
> "This helps detect use-after-free and buffer-overflow bugs, which are the most common source of memory safety bugs in our codebases."
> — <https://source.android.com/docs/security/test/memory-safety/arm-mte>

注意原文用 "can augment"——**能力**，不是"默认对每个进程开启"。这一点在后面还会反复出现：MTE 在 Android 上的每一个"开启"都是有条件的，而条件的不同组合决定了防护的实际覆盖。

### 4.1 同步与异步：两种模式，两套语义

MTE 最有价值的一处官方文本，是把两种模式的**失败语义**写全了：

> "On a tag mismatch, the processor aborts execution immediately and terminates the process with SIGSEGV (code SEGV_MTESERR) and full information about the memory access and the faulting address."
> "On a tag mismatch, the processor continues execution until the nearest kernel entry (for example, a syscall or timer interrupt), where it terminates the process with SIGSEGV (code SEGV_MTEAERR) without recording the faulting address or memory access."
> — <https://source.android.com/docs/security/test/memory-safety/arm-mte>

| 模式 | 报错时机 | 信号 | 是否给出故障地址 | 适用面 |
| --- | --- | --- | --- | --- |
| SYNC | 立即 | `SEGV_MTESERR` | 给出，且含访问信息 | 测试、高风险面、需要定位的调试构建 |
| ASYNC | 最近的 kernel entry（syscall、时钟中断） | `SEGV_MTEAERR` | **不给出** | 低开销量产部署 |

读法有三条：

1. **ASYNC 也在拦，但它拦得更晚、更"盲"。** 进程仍会被终止，但你不会拿到故障地址——这在调试时几乎是致命的，在量产时却是可以接受的代价。
2. **官方把 SYNC 定位为安全缓解，但没有把 ASYNC 与之等同表述。** 这句话很重要：不要读成"两种模式防护强度一样，只是开销不同"。评估"某设备开了 MTE"时，必须问清是哪种模式。
3. **两种模式的信号码本身就是可观测事实。** `SEGV_MTESERR` 与 `SEGV_MTEAERR` 出现在崩溃记录里，就能反推该进程跑的是哪种模式——这是一个不依赖任何私有接口的判读点。

### 4.2 App 侧：默认关闭，进程级生效

> "If not specified MTE is disabled by default but apps that want to use MTE can do so by setting android:memtagMode under the <application> or <process> tag in the AndroidManifest.xml."
> "MTE enablement is a process-wide property: enabling it applies to all native heap allocations in a process."
> — <https://source.android.com/docs/security/test/memory-safety/arm-mte>

两条结论：**App 的 MTE 是选择性加入**（默认关闭，需 manifest 声明），且**粒度是进程级**（开启即覆盖该进程的所有原生堆分配）。后一句解决了一个常见误会：不存在"只对某个分配器类开启 MTE"的应用级配置。

边界同样要写清楚：这两句只覆盖 App。平台进程的 MTE 由构建配置与系统属性控制，"App 默认关闭"不能外推成"设备整体默认关闭"。

### 4.3 bootloader 与内核侧：MTE 不是纯内核开关

MTE 的 tag 区需要在启动极早期被保留下来，因此它牵涉 bootloader：

> "Android 13 introduces an ABI for user-space to communicate the requested MTE mode to the bootloader."
> "the bootloader sets up MTE tag reservation, enables tag checks in the lower exception levels, and communicates the tag reserved region to the kernel with the devicetree (DT)."
> — <https://source.android.com/docs/security/test/memory-safety/bootloader-support>

这解释了三件事：**为什么"内核支持 MTE"不等于"设备能用 MTE"**（还差 bootloader 与固件）；**为什么 Android 13 是一个时间点**（统一的运行时模式 ABI 才出现）；以及**为什么 MTE 的启用状态属于启动链的一部分**——这段与 [系统更新链路与回滚防护手册](/android-ota-update-rollback/) 的启动链完整性是同一根链条的不同段。

内核侧另有一套独立的开关，且**默认值是开**：

> "To enable MTE-accelerated KASAN for the kernel, configure the kernel with CONFIG_KASAN=y, CONFIG_KASAN_HW_TAGS=y."
> "These configs are enabled by default on GKI kernels, starting with Android 12-5.10."
> — <https://source.android.com/docs/security/test/memory-safety/arm-mte>

**这是最容易混为一谈的一处：内核的 MTE-KASAN 默认开启，不等于用户态进程的 MTE 默认开启。** 两者是两套开关、两个作用域、两种失败代价。把前者当后者，会得出"Android 12 起应用都有 MTE 保护"这种明确错误的结论。

### 4.4 一个必须写清的现状：Pixel 8 的 MTE 在官方口径里是开发者开关

关于具体机型，官方 2023 年的表述非常克制：

> "Tensor G3 integrates support for MTE only within the developer mode toggle."
> "Some devices don't enable MTE by default, but allow developers to reboot with MTE enabled."
> "Starting from Android 13, select devices have support for MTE."
> — <https://security.googleblog.com/2023/11/mte-promising-path-forward-for-memory.html>；<https://developer.android.com/ndk/guides/arm-mte>

**因此本手册不写"Pixel 8 默认开启 MTE"。** 现有的一手材料说的是"硬件支持、但在开发者开关里"。第三方文章里"Pixel 8 默认开启"的说法，在本次核验范围内没有官方来源支持（未核实 B）。这条的意义超出机型本身：**它示范了如何用官方原文给宣传性说法设边界。**

---

## 5. 第三道防线：检测工具，各有各的适用范围

检测器与缓解器不是一类东西。检测器的价值在"发现缺陷"，它们的设计目标通常是**让问题更容易暴露**，代价是开销与采样。Android 上三件主力工具的分工如下。

### 5.1 HWASan：靠 top-byte-ignore 把 tag 放进指针高位

> "Hardware-assisted AddressSanitizer (HWASan) is a memory error detection tool similar to AddressSanitizer."
> "HWASan is only available on Android 10 and higher, and only on AArch64 hardware."
> "HWASan relies on ARMv8 feature top byte ignore (TBI), also called virtual address tagging, to store the pointer tag in the highest bits of the address."
> — <https://source.android.com/docs/security/test/hwasan>

机制读法：TBI 允许硬件忽略指针最高字节，于是**这个字节可以被软件用来存 tag**；指针的 tag 与所指内存的 tag 不一致即报错。这是"用软件模拟 MTE 的核心思想"，代价是需要重新编译整棵代码树。

覆盖与成本：

> "Userspace support for HWASan is available starting with Android 11."
> "Similar CPU overhead (~2x)"
> "HWASan uses a lot less RAM compared to ASan, which makes it suitable for whole system sanitization."
> — 同上

三条边界：**仅 AArch64**（"only on AArch64 hardware"）；**CPU 约 2 倍**；**RAM 远小于 ASan**——最后一句是官方给出的"为什么它适合整机检测"的理由。反过来说，2 倍 CPU 决定了它不会是无差别的量产配置，而是平台代码与调试构建的选择。

### 5.2 GWP-ASan：1% 采样 + guard page，开销可忽略

> "GWP-ASan was introduced in Android 11 for userspace heap allocations."
> "GWP-ASan is randomly enabled, at start-up, for 1% of system processes or apps and for applications that set the android:gwpAsanMode in their AndroidManifest.xml file."
> "This allows detecting buffer overflows and use-after-frees at a negligible performance overhead."
> — <https://source.android.com/docs/security/test/memory-safety/gwp-asan-kfence>

三项要读准：

1. **触达方式是采样。** 启动时对 1% 的系统进程或 App 随机开启，加上显式 opt-in 的应用。它不是"全量开启"。
2. **检出靠的是 guarded allocation（guard page 前后隔离）。** 溢出与 UAF 会踩到不可访问页，因此能精确捕获并给出完整报告——这一点比 Scudo 的"访问触发"更主动。
3. **开销可忽略**，这解释了为什么它可以长期留在量产构建里：**GWP-ASan 是少数"能进量产"的检测器**。

**最重要的边界：1% 是进程采样率。** 一次具体的崩溃只能证明"这台设备上的这个进程在那一刻被采样了并且出了错"，**不能用来估计缺陷的总量**。把 GWP-ASan 的报告数当漏洞计数，是统计口径错误。

### 5.3 KASAN 与内核侧

内核侧与 MTE 结合的那套配置已在 §4.3 给出（`CONFIG_KASAN=y` + `CONFIG_KASAN_HW_TAGS=y`，GKI 12-5.10 起默认）。这里补一条判读原则：**内核 KASAN 报告的可读性依赖调试信息与符号，量产设备上的可用性远低于调试构建。** 因此"看到过 KASAN 报告"与"设备具备内核内存错误检测能力"是两件事。

---

## 6. 第四道防线：消除整类缺陷——零初始化与 Rust

前三道防线都是"在缺陷已存在的前提下降低成功率或提高发现率"。后两道不拦截，它们**减少缺陷的产生**，因此性质完全不同。

### 6.1 零初始化：Android 12 起，覆盖平台代码与平台进程

> "Since Android 12, stack memory is zero initialized in all platform native code (including JNI) and heap memory is zero initialized in all platform native processes (such as netd) but not in the zygote or in apps."
> "For stack initialization, GKI uses the CONFIG_INIT_STACK_ALL_ZERO config"
> "For heap initialization, GKI uses the CONFIG_INIT_ON_ALLOC_DEFAULT_ON"
> — <https://source.android.com/docs/security/test/memory-safety/zero-initialized-memory>

这段话一次给出四个范围，缺一不可：

| 对象 | 是否默认清零 | 原文依据 |
| --- | --- | --- |
| 平台原生代码的栈（含 JNI） | 是 | "all platform native code (including JNI)" |
| 平台原生进程的堆（如 netd） | 是 | "all platform native processes (such as netd)" |
| zygote | **否** | "but not in the zygote" |
| App | **否**（需自行选择） | "or in apps" |

**这直接修正了一条流传很广的说法**："Android 11 起默认零初始化"。官方页面写的是 **Android 12**（未核实 C）。更实际的一点是：zygote 与 App 不在覆盖范围内，而 App 恰恰是绝大多数第三方代码所在的位置——所以"零初始化已经解决了未初始化内存问题"是一个范围错误。

它消除的缺陷类别很具体：**未初始化栈/堆内存被当作有效数据使用**，即信息泄露类与"未初始化指针/长度"类的根因。这类缺陷的共同点是"读到了上一次留下的内容"，清零后读到的是零，于是从"可利用的信息泄露"退化为"可控的崩溃或逻辑错误"。

### 6.2 Rust：语言层面的类别消除，但只覆盖新代码

> "Android 12 introduced Rust as a platform language."
> "We expect Rust to be the preferred choice for most new native projects."
> "Rust provides memory and thread safety at performance levels similar to C/C++."
> — <https://source.android.com/docs/security/test/memory-safety>

定位是"首选语言"，不是"禁止 C/C++"——同一页面也说明全量重写不可行。实际进展有两份时点数据：

> "In Android 13, about 21% of all new native code (C/C++/Rust) is in Rust."
> "To date, there have been zero memory safety vulnerabilities discovered in Android’s Rust code."
> — <https://security.googleblog.com/2022/12/memory-safe-languages-in-android-13.html>

> "We adopted Rust for its security and are seeing a 1000x reduction in memory safety vulnerability density compared to Android’s C and C++ code."
> "The 2025 data continues to validate the approach, with memory safety vulnerabilities falling below 20% of total vulnerabilities for the first time."
> — <https://blog.google/security/rust-in-android-move-fast-fix-things/>

**三条边界，写报告时必须带上：**

1. **"zero" 是截至 2022 年 12 月的统计。** 原文自己附了"不指望永远为零"的限定，不能外推到当前。
2. **1000x 是"漏洞密度"，不是漏洞总数。** 密度是单位代码量的比率，与"总量下降 1000 倍"是两回事。
3. **Rust 只覆盖新代码。** 存量 C/C++ 依然存在，因此 §2–§5 的四道防线仍然必要——这也是为什么"用了 Rust"不能替代分配器硬化与硬件标记。

---

## 7. 成因层：内存破坏在 Android 上的五种落点

前六节讲的是"防线"，这一节讲"缺陷本身"。五种成因形态，每种给出它在 Android 上的典型载体与一条可核对的证据。

### 7.1 UAF：生命周期跨越了信任边界

最典型的载体是 IPC 与内核对象：一个对象在 A 侧被释放，B 侧仍持有引用。Android 上被引用最多的实例是 binder：

> "On October 3, 2019, we disclosed issue 1942 (CVE-2019-2215), which is a use-after-free in Binder in the Android kernel."
> — <https://googleprojectzero.blogspot.com/2019/11/bad-binder-android-in-wild-exploit.html>

公告侧的表值见 §8。**成因模式**：对象所有权与释放时机分散在两侧，且没有强制的引用计数或代际校验。这也是 Scudo 的 quarantine（§3）与 MTE（§4）主要针对的形态。

### 7.2 越界写：解析器用错的长度算出缓冲区大小

> "This can lead to out of bounds writes since frames of this type were not taken into account when calculating the size of the frames buffer in uvc_parse_streaming."
> — <https://cveawg.mitre.org/api/cve/CVE-2024-53104>

**成因模式**：输入格式允许的变体多于解析器在计算尺寸时考虑的分支；尺寸算小了，写入按实际数据走，于是越界。这是"解析器 + 长度计算"这一类缺陷的教科书形态，也是 header 校验（§3）与 guard page（§5.2）的主要目标。

### 7.3 越界读：信息泄露，常被低估

Type 为 `ID`（Information Disclosure）的条目往往与越界读或未初始化读相关。它们的危害不体现在"控制流被劫持"，而体现在**地址、密钥材料、堆布局等元信息泄露**，为下一步利用提供条件。**这也是零初始化（§6.1）的靶心**：未初始化读在清零后不再泄露历史内容。

### 7.4 双重释放与类型混淆

两者都属"对象状态被错误理解"：前者是同一块内存被释放两次（可导致分配器元数据被操控），后者是把一种类型的内存当作另一种解释（可绕过类型约束）。**这两类在 MTE 下的表现值得单独注意**：它们不必然产生 tag 不匹配，因此**不能假设开了 MTE 就覆盖了全部堆缺陷类型**。

### 7.5 并发与尺寸整数溢出

并发释放（一侧 free、另一侧仍在读写）是 UAF 的并发版本；尺寸计算中的整数溢出（`len * count` 溢出为小值）则常是 §7.2 的根因。IntSan（§9）正是针对后者的插桩。

**一张对照表：**

| 成因 | 典型载体 | 主要防线 | 防线失效的典型原因 |
| --- | --- | --- | --- |
| UAF | IPC / 内核对象生命周期 | quarantine、MTE、HWASan | quarantine 未开启或配额不足；MTE 为 ASYNC 模式 |
| 越界写 | 解析器 + 长度计算 | header 校验、guard page、MTE | header 未被访问即不触发；采样未命中 |
| 越界读 / 未初始化读 | 缓冲区、栈变量 | 零初始化、MTE | zygote 与 App 不在 zero-init 范围 |
| 双重释放 | 分配器元数据 | Scudo 自身校验 | 元数据未被访问 |
| 类型混淆 | C++ 对象、union | CFI、Rust | CFI 只覆盖插桩组件；存量 C/C++ 仍在 |

---

## 8. 公告里的内存破坏条目：两个案例，一个表头陷阱

公告不写根因，只给字段值。下面两条是本手册唯一使用的案例，字段全部逐字取自公告页。

**CVE-2019-2215（2019-10-01 期）**

| 字段 | 值 |
| --- | --- |
| CVE | CVE-2019-2215 |
| References | A-141720095 |
| Type | EoP |
| Severity | High |
| 组件列 | **Binder**（该期表头用的是 `Component`，不是 `Subcomponent`） |

> — <https://source.android.com/docs/security/bulletin/2019-10-01>

**表头陷阱：** 早期公告的列名是 `Component`，现行的内核表列名是 `Subcomponent`。做跨年份的条目统计时，如果按列名硬匹配，会把一整套历史条目漏掉，或者把两个语义不同的字段强行对齐。

**CVE-2024-53104（2025-02-01 期）**

| 字段 | 值 |
| --- | --- |
| CVE | CVE-2024-53104 |
| References | A-378455392 / Upstream kernel |
| Type | EoP |
| Severity | High |
| Subcomponent | UVC |

> "Note: There are indications that CVE-2024-53104 may be under limited, targeted exploitation."
> — <https://source.android.com/docs/security/bulletin/2025-02-01>

**三条读法：**

1. **`Type = EoP` 不等于"成因是内存破坏"。** Type 描述的是**影响**（提权），不是机制。CVE-2019-2215 的 "use-after-free" 来自 Project Zero 的披露文字，不是公告字段。
2. **`Severity` 与成因类型无关。** 两条都是 High，但一条是 IPC 生命周期缺陷、一条是驱动解析缺陷；等级相同不代表同类。
3. **"limited, targeted exploitation" 是风险提示，不是"已确认大规模在野利用"。** 这句官方措辞的强度常被引用放大，写结论时应按原文保留限定词。

---

## 9. 防护矩阵：机制、拦什么、不拦什么、成本、能否量产

把全书收敛成一张可查表。**"能否量产"一列是本手册最实用的部分**——它把"有这项技术"与"这项技术在真实设备上生效"分开。

| 机制 | 引入/默认口径 | 拦什么 | 不拦什么 | 成本 | 能否进量产 |
| --- | --- | --- | --- | --- | --- |
| Scudo chunk header 校验 | Android 11 起默认（低内存设备除外） | 向上溢出改写 header、元数据损坏 | 未被访问的损坏；向后越界；纯 UAF | 低 | 是（默认即量产） |
| Scudo quarantine | 上游默认关闭，Android 配额未公开 | 释放后立即复用的 UAF | 长时间窗后的 UAF；且需开启 | 内存 + 性能 | 取决于产品配置 |
| MTE SYNC | 需硬件 + bootloader + 进程开启 | tag 不匹配的 UAF/越界，**带地址** | 双重释放、类型混淆；未打 tag 的分配 | 高 | 调试 / 高风险面 |
| MTE ASYNC | 同上 | tag 不匹配（延迟到 kernel entry） | 同上，且**无故障地址** | 较低 | 是（视设备） |
| HWASan | Android 11 起用户态；仅 AArch64 | UAF、越界（整机级覆盖） | 非 AArch64；非插桩代码 | CPU ~2x，RAM 远低于 ASan | 调试构建 |
| GWP-ASan | Android 11 起；1% 采样或 opt-in | 溢出、UAF（guard page，报告完整） | 未采样进程；无法作总量统计 | 可忽略 | 是 |
| KASAN（MTE 加速） | GKI 12-5.10 起默认编译 | 内核 UAF/越界 | 用户态；量产可读性受限 | 内核侧 | 编译默认，量产可用性受限 |
| CFI | Android 9 起系统默认；内核需另开 | 控制流劫持（间接调用） | 数据面错误、堆破坏 | 中 | 是（限 `cfi-common.mk` 组件） |
| IntSan / BoundsSan | UBSan 子集，官方称可作生产缓解 | 整数溢出、静态数组越界 | 动态数组越界；未插桩代码 | 中 | 是（按组件） |
| 零初始化 | Android 12 起，平台代码栈 + 平台进程堆 | 未初始化读、信息泄露 | zygote 与 App；已初始化但错误的值 | 极低 | 是 |
| Rust | Android 12 引入；新代码首选 | 整类内存安全缺陷 | 存量 C/C++；逻辑错误 | 学习/互操作 | 是（限新代码） |

CFI 与 IntSan/BoundsSan 的官方口径一并给出，避免只写工具名不给范围：

> "Control flow integrity (CFI) is a security mechanism that disallows changes to the original control flow graph of a compiled binary, making it significantly harder to perform such attacks."
> "In Android 8.1, we enabled LLVM's implementation of CFI in the media stack. In Android 9, we enabled CFI in more components and also the kernel. System CFI is on by default but you need to enable kernel CFI."
> "CFI is enabled by default for Arm64 devices for the set of components in /platform/build/target/product/cfi-common.mk."
> — <https://source.android.com/docs/security/test/cfi>

> "signed and unsigned integer overflow checking (IntSan), bounds checks for statically-sized arrays (BoundsSan)"
> "IntSan and BoundsSan are also suitable for enabling in production as a security mitigation."
> — <https://source.android.com/docs/security/test/sanitizers>

**读法结论（三条）。**

1. **"默认开启"永远带范围。** CFI 默认只覆盖 `cfi-common.mk` 列出的组件；Scudo 排除低内存设备；MTE 排除未声明 memtagMode 的 App；零初始化排除 zygote 与 App。这张表里最该被记住的不是"开了什么"，而是"排除了什么"。
2. **检测器与缓解器不能互相替代。** GWP-ASan 检出一次 UAF 不代表该进程之后安全；Scudo 拦下一次 header 损坏也不代表同类缺陷不再出现。前者是抽样，后者是概率性缓解。
3. **成本决定部署形态。** 2 倍 CPU 的 HWASan 与"可忽略"的 GWP-ASan，注定一个进调试构建、一个进量产。评估时若不问部署形态，"设备有 HWASan"这种说法毫无信息量。

---

## 10. 观测点与自查

| # | 观测点 | 判读方法 | 依据 |
| --- | --- | --- | --- |
| 1 | 是否使用 Scudo | Android 11 起默认，但**低内存设备仍用 jemalloc** | §2 |
| 2 | 分配器是否自述为"缓解" | Scudo 官方定性为 mitigation，不是检测器 | §2 |
| 3 | 堆损坏是否导致进程终止 | "the process is aborted" | §2 |
| 4 | 崩溃里是否有 `SEGV_MTESERR` | 有 → SYNC 模式，且带故障地址 | §4.1 |
| 5 | 崩溃里是否有 `SEGV_MTEAERR` | 有 → ASYNC 模式，无故障地址 | §4.1 |
| 6 | App 是否声明 `android:memtagMode` | 未声明即默认关闭；声明即进程级全量生效 | §4.2 |
| 7 | 设备是否具备 MTE 启动支持 | 需 bootloader 保留 tag 区（Android 13 起有统一 ABI） | §4.3 |
| 8 | 内核是否 MTE-KASAN | GKI 12-5.10 起默认编译，**与用户态 MTE 是两套开关** | §4.3 |
| 9 | 是否为"开发者开关"型 MTE | 官方对 Tensor G3 的表述即如此 | §4.4 |
| 10 | 是否 AArch64 | HWASan 仅在 AArch64；AVF 亦仅 ARM64 | §5.1 |
| 11 | 是否有 GWP-ASan 报告 | 需注意 1% 采样，报告数 ≠ 缺陷总数 | §5.2 |
| 12 | 是否零初始化覆盖到目标代码 | 平台代码栈/平台进程堆覆盖；zygote 与 App 不覆盖 | §6.1 |
| 13 | 新代码语言结构 | Rust 占比是新代码口径，不是存量占比 | §6.2 |
| 14 | 公告列名是 `Component` 还是 `Subcomponent` | 跨年份统计必须区分 | §8 |
| 15 | `Type=EoP` 是否等于内存破坏 | 否，Type 描述影响而非机制 | §8 |

**三条方法论结论。**

1. **先问覆盖范围，再问强度。** 每一个"已启用"都要追问排除了谁——低内存设备、未声明 manifest 的 App、未插桩的组件、zygote。
2. **把抽样与总量分开。** GWP-ASan 的 1% 与公告的披露数都不是"缺陷总量"，只是不同口径的样本。
3. **分清"发现"与"防护"。** 检测器的产出是报告，缓解器的产出是"更难利用"。把报告数当防护强度的度量，会系统性高估。

---

## 11. 常见误传纠正

| # | 常见说法 | 官方口径 | 出处 |
| --- | --- | --- | --- |
| 1 | Android 11 起所有设备都用 Scudo | 低内存设备仍用 jemalloc | §2 |
| 2 | Scudo 相当于内置 ASan | "more of a mitigation than a ... detector like ASan" | §2 |
| 3 | Scudo 能发现所有堆损坏 | header 只在被访问时校验，未被访问的损坏不会被发现 | §3 |
| 4 | Scudo 的 quarantine 默认开启 | LLVM 文档：disabled by default；Android 配额未公开 | §3、未核实 E |
| 5 | MTE 从 Android 12 起默认对所有进程开启 | "can augment"是能力表述；App 默认关闭 | §4、§4.2 |
| 6 | 内核 MTE-KASAN 默认就等于用户态 MTE 默认 | 两套开关、两个作用域 | §4.3 |
| 7 | Pixel 8 默认开启 MTE | 官方 2023 口径为"仅开发者开关内" | §4.4、未核实 B |
| 8 | MTE 只有"开/关"两态 | 有 SYNC/ASYNC 两种模式，失败语义不同 | §4.1 |
| 9 | 开了 MTE 就覆盖全部堆缺陷 | 双重释放、类型混淆不必然触发 tag 不匹配 | §7.4 |
| 10 | HWASan 可以无差别用在量产 | CPU 约 2x；仅 AArch64 | §5.1 |
| 11 | GWP-ASan 报告数可以当漏洞总量 | 启动时 1% 进程采样 | §5.2 |
| 12 | 零初始化自 Android 11 起 | 官方原文为 Android 12 | §6.1、未核实 C |
| 13 | 零初始化覆盖所有代码 | 不含 zygote 与 App | §6.1 |
| 14 | Rust 已让内存漏洞消失 | 只覆盖新代码，存量 C/C++ 仍在 | §6.2 |
| 15 | Rust 的"零漏洞"是当前状态 | 该表述截至 2022 年 12 月 | §6.2 |
| 16 | 1000x 指漏洞总数下降 | 指"漏洞密度" | §6.2 |
| 17 | CFI 覆盖全部二进制 | 默认覆盖 `cfi-common.mk` 所列组件；内核 CFI 需另开 | §9 |
| 18 | 公告 `Type=EoP` 说明该条是内存破坏 | Type 描述影响，不描述机制 | §8 |

---

## 12. 与其他手册的分工

| 问题 | 归属 |
| --- | --- |
| 内核镜像、GKI 模块、KMI、内核加固 CONFIG 全貌 | [内核与 GKI 驱动攻击面手册](/android-kernel-gki-drivers/) |
| SELinux 域与 seccomp 白名单的构造 | [SELinux 策略与 seccomp 沙箱手册](/android-selinux-seccomp-sandbox/) |
| Binder 接口面与组件导出 | [框架 IPC 与组件导出攻击面手册](/android-framework-ipc-export/) |
| 按组件定位的缺陷类型图谱 | [底层漏洞类型图谱手册](/android-low-level-vulnerability-types/) |
| 公告字段读法、SPL、CVE 对应关系 | [漏洞编号与公告体系手册](/android-cve-and-security-bulletins/) |
| 定级与可利用性要素 | [安全性评估与定级手册](/android-vulnerability-severity-assessment/) |
| 隔离边界（沙箱、pVM、TEE 的层级分工） | [虚拟化与隔离边界手册](/android-virtualization-isolation/) |

---

## 参考

**官方一手来源（本手册逐字引用过）**

- Scudo — <https://source.android.com/docs/security/test/scudo>
- Scudo（LLVM 上游文档）— <https://llvm.org/docs/ScudoHardenedAllocator.html>
- Memory Tagging Extension — <https://source.android.com/docs/security/test/memory-safety/arm-mte>
- MTE bootloader support — <https://source.android.com/docs/security/test/memory-safety/bootloader-support>
- MTE（Linux 内核 arm64 文档）— <https://www.kernel.org/doc/html/latest/arch/arm64/memory-tagging-extension.html>
- NDK：Arm MTE — <https://developer.android.com/ndk/guides/arm-mte>
- HWASan — <https://source.android.com/docs/security/test/hwasan>
- GWP-ASan 与 KFENCE — <https://source.android.com/docs/security/test/memory-safety/gwp-asan-kfence>
- Zero-initialized memory — <https://source.android.com/docs/security/test/memory-safety/zero-initialized-memory>
- Memory safety（总览，含 Rust 定位）— <https://source.android.com/docs/security/test/memory-safety>
- CFI — <https://source.android.com/docs/security/test/cfi>
- Sanitizers（IntSan / BoundsSan）— <https://source.android.com/docs/security/test/sanitizers>
- Rust in Android（2023 数据）— <https://security.googleblog.com/2022/12/memory-safe-languages-in-android-13.html>
- Rust in Android（2025 数据）— <https://blog.google/security/rust-in-android-move-fast-fix-things/>
- MTE 现状（2023）— <https://security.googleblog.com/2023/11/mte-promising-path-forward-for-memory.html>
- Project Zero：Bad Binder — <https://googleprojectzero.blogspot.com/2019/11/bad-binder-android-in-wild-exploit.html>
- Android Security Bulletin 2019-10-01 — <https://source.android.com/docs/security/bulletin/2019-10-01>
- Android Security Bulletin 2025-02-01 — <https://source.android.com/docs/security/bulletin/2025-02-01>
- Linux CNA：CVE-2024-53104 — <https://cveawg.mitre.org/api/cve/CVE-2024-53104>

**相邻手册**：见 §12。

---

## 未核实与失效来源

**A. jemalloc / dlmalloc 的完整沿革**
推测：Android 早期用 dlmalloc，4.x 起换 jemalloc，Android 11 再换 Scudo。未核实原因：source.android.com 现站无沿革专页，站内检索无结果；只能从 Scudo 页确认"Android 11 时低内存设备仍用 jemalloc"。**正文只写这一点，不写 4.x–10 的默认分配器。**

**B. "Pixel 8 上 MTE 默认开启"**
推测：后续版本对部分系统进程默认开启 MTE。未核实原因：现有官方材料相反——2023 年官方博客称 Tensor G3 的 MTE "only within the developer mode toggle"，NDK 文档称部分设备默认不开、需开发者选项。未找到任何官方页面支持"默认开启"，该说法只见于第三方文章。**正文按官方口径写"开发者开关"。**

**C. "Android 11 起默认零初始化"**
推测：zero-init 计划自 Android 11 启动、Android 12 全面落地。未核实原因：官方 zero-init 页明确写 "Since Android 12"，无任何 Android 11 默认零初始化的表述。**正文写 Android 12，并把"11 起"列为误传。**

**D. BoundsSanitizer / IntSanitizer 的独立机制页与默认状态**
推测：两者在部分组件中默认启用。未核实原因：`/docs/security/test/intsan?hl=en`、`/docs/security/test/memory-safety/bounds-sanitizer?hl=en` 均 404；`/docs/security/test/sanitizers` 只有一段 UBSan 说明，未给默认状态与组件覆盖清单。**正文只写"官方称可作生产缓解"，不写默认状态。**

**E. Scudo 在 Android 产品上的 quarantine 默认配额**
推测：Android 默认开启一定大小的 quarantine。未核实原因：AOSP Scudo 页只列 `QuarantineSizeKb` 等可调项，不给默认值；LLVM 文档明确上游默认关闭。产品默认配置需查 AOSP 源码，超出本次网页核验范围。**正文不写配额数值。**

**F. MTE 的漏检概率**
本手册在第 4 节只描述 SYNC/ASYNC 的失败语义，**不给出"tag 碰撞导致漏检的概率"**——该数值来自 tag 位宽与分配粒度的组合，本次未取得官方逐字表述，不写入正文。

**G. 抓取经验（便于复现）**
source.android.com 会按访问 IP 本地化正文语言，**必须在 URL 后加 `?hl=en`** 才拿到英文原文；developer.android.com 同理。引文跨行时按空格拼接，不改动词序。新旧公告的列名不同（`Component` vs `Subcomponent`），跨期比较前先核对表头。

---

## 联系方式

咨询 Android 系统详情请联系 https://t.me/one00190

---

*本手册仅供安全学习与防御研究使用；所有默认状态、版本口径、配置项名与案例字段均取自公开一手来源并附链接，未能核实的表述已单列于「未核实与失效来源」而非写入正文。*
