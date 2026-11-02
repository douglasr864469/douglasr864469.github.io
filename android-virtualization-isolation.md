---
layout: page
title: "Android 虚拟化与隔离边界手册"
description: "五层隔离边界逐层取证：应用沙箱的 UID/DAC、Android 8.0 起「all apps」受 seccomp-bpf 约束、Android 9 起按应用的 SELinux 沙箱如何叠加；AVF 自 Android 13 引入且官方明确不取代 TrustZone；pKVM 不是新 hyperviso..."
date: 2026-11-02
permalink: /android-virtualization-isolation/
---

> 「Android 有沙箱」这句话在中文资料里通常是句废话——它既不说明隔离发生在哪一层，也不说明谁被信任、失效后炸多大。真实的 Android 隔离是一条**从应用 UID 一直排到独立安全芯片的五层边界**，每层的强制点、信任根、以及"保证机密性还是连可用性一起保证"都不一样。本手册沿官方原文把这五层逐个钉死：应用沙箱的 UID/DAC、SELinux、seccomp-bpf 三层是怎么叠上去的；为什么普通 KVM 在宿主态直接关掉 stage-2 翻译，因而给不了 pVM 级别的隔离；pKVM 把"约束宿主"的权力放进 EL2 之后，可信边界在启动的哪一刻切换；pVM 明确**只保证机密性与完整性、不保证可用性**，且 OEM 与 SoC 厂商仍在信任集里；Trusty 与 StrongBox 是另外两条边界而不是 hypervisor 的替代品；以及月度公告里 `pKVM` 与 `Kernel Virtual Machine` 是两个不同的 Subcomponent 标签。定位是防御与教育：只引用公开且已修复的信息，不含任何可利用代码，也不给出任何绕过步骤。

---

## 0. 使用方式与边界

**本手册的证据基础。** 官方口径全部取自 source.android.com 的英文原文（`?hl=en` 抓取，避免页面按 IP 本地化成日/韩/中文而误读术语），另有一条来自 kernel.org 的 pKVM 内核文档。引文以英文逐字给出，中文结论紧随其后，并**明确写出引文支撑到哪一步为止**——凡是官方页面没写的东西（机型矩阵、首发平台、服务名），一律进文末「未核实与失效来源」，正文不写。

**本手册刻意不做的事。**

- 不给任何绕过、逃逸或提权步骤。虚拟化与 TEE 的绕过材料一旦具体化即为攻击工具，本手册只讨论边界画在哪、为什么这么画、失效后的半径。
- 不宣称机型可用性。官方 overview 只说了 "AVF is supported only on ARM64 devices."，没有给受支持机型清单；任何"某某机型是首发"的说法都不在本手册里出现（见未核实 A、E）。
- 不把 pVM 说成绝对安全。官方自己写了可用性例外与供应链信任例外（§9），本手册照抄这两条例外，不美化。
- 不讨论 SELinux 策略写法与 seccomp 白名单构造，那部分归 [SELinux 策略与 seccomp 沙箱手册](/android-selinux-seccomp-sandbox/)；不讨论用户/工作资料的 UID 计算与 `isolatedProcess`，那部分归 [多用户、工作资料与进程隔离手册](/android-multiuser-sandbox-isolation/)。

**与相邻手册的分工。**

| 问题 | 归本手册 | 归哪本手册 |
| --- | --- | --- |
| 应用沙箱由哪几层构成、各自从哪个版本开始强制 | ✅（只讲官方口径） | [SELinux 策略与 seccomp 沙箱手册](/android-selinux-seccomp-sandbox/)（策略写法与 neverallow） |
| 用户、工作资料、`isolatedProcess` 的 UID 与域映射 | — | [多用户、工作资料与进程隔离手册](/android-multiuser-sandbox-isolation/) |
| hypervisor 层（pKVM/AVF/pVM）的架构与保证边界 | ✅ | [内核与 GKI 驱动攻击面手册](/android-kernel-gki-drivers/)（内核镜像、模块与签名主体） |
| TEE/StrongBox 的密钥用途与证明链 | 仅隔离层级 | [TEE、密钥管理与设备认证手册](/android-tee-key-attestation/) |
| 公告字段怎么读、CVE 与 Android ID 的对应 | 仅虚拟化条目 | [漏洞编号与公告体系手册](/android-cve-and-security-bulletins/) |
| 为什么 hypervisor 漏洞能评到 Critical | 部分 | [安全性评估与定级手册](/android-vulnerability-severity-assessment/) |

---

## 1. 五层边界与一条判断法则

先把全景摆出来，再逐层取证。下表每一行的「强制点」都指**这条边界的规则在哪一级被强制执行**，不是指它由谁设计。

| 边界 | 隔离对象 | 强制点 | 信任根 | 单点失守后的半径 |
| --- | --- | --- | --- | --- |
| 应用沙箱（UID/DAC） | 应用之间 | 内核（文件权限与进程身份） | 内核 + 包管理发放的 UID | 同 UID 数据互访；跨应用数据面 |
| 应用沙箱（SELinux） | 应用对象访问 | 内核 LSM | 策略镜像（boot/system） | 对象访问矩阵对该域失效 |
| 应用沙箱（seccomp-bpf） | 应用的系统调用集合 | 内核 seccomp | 进程自我保护（不可逆） | 系统调用面恢复完整 |
| pVM（pKVM） | 宿主 Android 与 pVM 之间 | EL2 hypervisor | bootloader 校验过的启动链 | 主机密性/完整性假设失效（可用性本不保证） |
| TEE（如 Trusty） | Android 与安全 OS 之间 | 硬件隔离 + 专用内核驱动 | 安全世界的启动与签名链 | 密钥与凭据保护失效 |
| StrongBox | 密钥运算与主 SoC | 独立 CPU/存储/封装 | 独立安全芯片 | 同上，但物理与侧信道门槛更高 |

这张表的价值在于它暴露了一个常被混淆的事实：**这五层不是"层层加固同一样东西"，而是各自保护不同对象、由不同级别的代码强制、失效后果也不同。** 把"有沙箱"当结论的人，通常在评估时会把"应用沙箱"的假设套到 hypervisor 上，或者反过来把 TEE 的保证当成整机保证。

**判断法则（本手册后面每一节都在用）。** 读任何一句隔离声明，问三个问题：

1. **强制点在内核态、EL2，还是在硬件？** 强制点越靠下，被上方任何一层缺陷推翻的可能性越小。
2. **谁在 TCB 里？** 官方是否明确写了"某某即使在什么情况下仍被信任"。没写，就是没承诺。
3. **机密性（C）、完整性（I）、可用性（A）分别保证了哪几件？** 三者不是打包的。pKVM 的例子最典型：C/I 保证，A 明确例外（§9）。

---

## 2. 应用沙箱：三层叠加的官方口径，以及"root 不等于逃逸"

应用沙箱是被引用最多、也最容易被简化成一句"每个应用一个用户"的一层。官方的说法是三层、按版本逐步叠加：

> "To do this, Android assigns a unique user ID (UID) to each Android app and runs it in its own process."
> "Android uses the UID to set up a kernel-level Application Sandbox."
> — <https://source.android.com/docs/security/app-sandbox>

这是第一层：**UID + 独立进程**，强制点是内核的文件权限与进程身份。它的失效后果很具体——一旦某条路径让两个应用以同一 UID 运行，或让一个应用能读到另一个 UID 的文件，那么"沙箱"在这一对应用之间就不存在了。

第二层是系统调用集合：

> "In Android 8.0, all apps were set to run with a seccomp-bpf filter that limited the syscalls that apps were allowed to use, thus strengthening the app/kernel boundary."
> — 同上

注意这里的时间点是 **Android 8.0**，"all apps"。这一层保护的不是"应用之间"，而是**应用与内核之间**：它缩小的是攻击者能触达的内核入口集合。

第三层是按应用划分的强制访问控制：

> "In Android 9 all nonprivileged apps with targetSdkVersion >= 28 must run in individual SELinux sandboxes, providing MAC on a per-app basis."
> — 同上

条件读全：**Android 9 起**、**非特权应用**、**targetSdkVersion ≥ 28**。三个条件缺一不可，写在评估结论里时少了任何一个都会把范围说大。

三层之上还有一条最常被忽略的结论——**拿到 root 不等于穿过 SELinux**：

> "As part of the Android security model, Android uses Security-Enhanced Linux (SELinux) to enforce mandatory access control (MAC) over all processes, even processes running with root/superuser privileges (Linux capabilities)."
> "In Android 5.0 and later, SELinux is fully enforced, building on the permissive release of Android 4.3 and the partial enforcement of Android 4.4."
> — <https://source.android.com/docs/security/features/selinux>

这解释了为什么"提权到 root"与"突破隔离"在 Android 上不是同一件事：DAC 层的越权（UID=0）仍然要被 MAC 层的域规则约束。反过来说，**root 级漏洞的真实半径取决于它能落到哪个 SELinux 域**，而不是取决于 `id` 输出里那个 `uid=0`。

**一个官方页面没有覆盖的点。** `isolatedProcess` 组件（隔离进程）在很多资料里被写成"沙箱里的又一层边界"，但现行 app-sandbox 页面对 "isolated" 零命中（见未核实 B）。本手册因此**不引用该属性的语义**，相关讨论留给 [多用户、工作资料与进程隔离手册](/android-multiuser-sandbox-isolation/)。

---

## 3. 为什么需要 hypervisor 级隔离：官方给的成因是"TCB 太大"与"域太粗"

这是本手册里唯一一处官方直接把"漏洞为什么会出现"写在架构文档里的地方，值得整段读。AVF 的动机页先描述现状：

> "However, this requirement has become problematic in that today's TCB is too large: it includes most device and bus drivers, complex schedulers, file systems, network stack and protocols, caches, executable parsers and loaders, and sockets. It has become very difficult to ensure every corner of this complicated system is safe."
> — <https://source.android.com/docs/core/virtualization/whyavf>

然后指出既有隔离手段的粒度问题：

> "The main limitation of these kind of solutions is that the domains are too coarse-grained: only secure and nonsecure. As more use cases that require isolation from the operating system are introduced, the attack surface increases and vulnerabilities are likely to lead to compromise of the entire device."
> — 同上

把这两句连起来读，得到的是一条**结构性成因**，而不是某个 bug：

1. 宿主 Android 的 TCB 里塞满了驱动、调度器、文件系统、网络栈、缓存、解析器与套接字——每一项都是历史漏洞的常客。
2. 传统 TEE 只提供"安全/非安全"两个域。任何需要"与 OS 隔离、但又不需要进 TEE"的新用例，只能挤进这两个域之一：进 TEE 会把富环境塞进小 TCB，留在 OS 侧就等于没有隔离。
3. 于是每增加一个这样的用例，攻击面就朝 OS 侧或 TEE 侧各推一步；任一侧被攻破，按官方的原话，后果是 "compromise of the entire device"。

**这就是"漏洞成因"章节应有的写法**：先讲清信任域的粒度不足如何必然产生新的攻击面，再谈具体缺陷。反过来，凡是把虚拟化漏洞讲成"hypervisor 有 bug"的资料，都跳过了这一层——hypervisor 有 bug 是现象，边界为什么必须画在 EL2 才是成因。

---

## 4. AVF：定义、版本、组件，以及"仅 ARM64"

AVF 的官方定位句把它的比较对象写得很清楚——比的是**应用沙箱**：

> "Android Virtualization Framework (AVF) provides secure and private execution environments for executing code. AVF is ideal for security-oriented use cases that require stronger, even formally verified, isolation assurances over those offered by Android's app sandbox."
> "Android provides a reference implementation of all the components needed to implement AVF. AVF is supported only on ARM64 devices."
> — <https://source.android.com/docs/core/virtualization>

版本与替代关系同样有明文：

> "To address these limitations and enable Android to provide a robust foundation for next generation use cases, Android 13 introduces secure virtualization as the Android Virtualization Framework (AVF)."
> "AVF doesn't fully replace TrustZone. Today, Android SoCs can't function without TrustZone as key devices assume secure access only."
> — <https://source.android.com/docs/core/virtualization/whyavf>

第二句是本节最该记住的一句：**AVF 与 TrustZone 是并存关系**。它直接否掉了"有了 hypervisor 就不需要 TEE"这个常见推断。

组件清单（逐字取自 AVF overview 的术语表）：

> "hypervisor — The virtualization technology used by AVF, also known as pKVM. The hypervisor maintains the integrity of the executed code and confidentiality of the pVM's assets, even if host Android or any of the other pVMs are compromised."
> "crosvm — A virtual machine monitor written in rust. crosvm allocates VM memory, creates virtual CPU threads, and implements the virtual device's back-ends."
> "pVM firmware (pvmfw) — The first code that runs on a pVM, pvmfw verifies the payload and derives the per-VM secret."
> "Microdroid Manager — Manages the pVM lifecycle, inside the pVM, and instance disk."
> "VirtualizationService — The Android service that manages the lifecycle of pVMs."
> "binder — Primary means of inter-VM communication."
> "authfs — A fuse file system for securely sharing multiple files between Android and pVM (host and guest)."
> — <https://source.android.com/docs/core/virtualization>

把这些组件按"运行在哪一级"排开，架构就出来了：

| 组件 | 运行位置 | 职责 | 安全含义 |
| --- | --- | --- | --- |
| hypervisor（pKVM） | EL2 | 维护被执行的代码完整性、pVM 资产机密性 | 即使宿主或其它 pVM 被攻陷也成立 |
| crosvm | 宿主用户态（VMM） | 分配 VM 内存、建 vCPU 线程、设备后端 | 属宿主侧，**不在** hypervisor 的保护范围里 |
| pvmfw | pVM 内的第一段代码 | 校验 payload、派生每 VM 密钥 | 把"这个 VM 跑的是哪份代码"钉在启动处 |
| Microdroid Manager | pVM 内 | 生命周期与实例磁盘 | guest 侧管理面 |
| VirtualizationService | Android 侧 | pVM 生命周期 | 宿主侧入口 |
| binder | 跨 VM 通信 | 主要通信手段 | 新增的解析面（§12） |
| authfs | 宿主机/guest 之间 | 安全共享文件 | 新增的共享面（§12） |

**读法结论。** 只有 hypervisor 一列在"宿主被攻陷仍成立"的承诺之内。VMM（crosvm）、VirtualizationService、binder、authfs 都在宿主侧，它们被攻陷时能做什么，取决于 pKVM 的 stage-2 约束（§6）——这正是"隔离边界"与"实现代码"必须分开看的原因。

---

## 5. pKVM：不是新 hypervisor，而是把"约束宿主"的权力放进 EL2

pKVM 最容易被误读成"Google 新写的 hypervisor"。官方措辞是"扩展"，而且说清楚了扩的是什么：

> "The protected kernel-based virtual machine (pKVM) is built upon the Linux KVM hypervisor, which has been extended with the ability to restrict access to the payloads running in guest virtual machines marked ‘protected’ at the time of creation."
> "pKVM leverages this design by extending the hypervisor code with new features allowing it to put restrictions on the Android host kernel and user space, and limiting host access to guest memory and the hypervisor."
> "The EL2 layer allows the introduction of a hypervisor that enables isolating memory and devices into individual pVMs at EL1/EL0, with strong confidentiality and integrity guarantees."
> — <https://source.android.com/docs/core/virtualization/architecture>

三句话各有一个关键词，对应三条读法：

- **"built upon the Linux KVM hypervisor"** —— 复用 KVM，不是另起炉灶。这决定了漏洞面继承了 KVM 的历史与代码路径，而不是"全新、无历史包袱"。
- **"put restrictions on the Android host kernel and user space"** —— 被约束的对象是**宿主**。这是与传统虚拟化最不直觉的一点：传统虚拟化的 guest 是"不可信的租客"，pKVM 的宿主才是被降权的一方。
- **"at EL1/EL0"** —— pVM 自己的内核跑在 EL1，hypervisor 独占 EL2。边界不在内核里，而在内核**之下**。

---

## 6. 为什么普通 KVM 给不了 pVM 的机密性：宿主态关掉 stage-2

这是整份手册里机制性最强的一段官方文字，值得逐字读两遍：

> "Historically, KVM runs with stage 2 translation enabled while running guests and with stage 2 disabled while running the host Linux kernel. This architecture allows memory accesses from the host stage 1 MMU to pass through the stage 2 MMU, hence allowing unrestricted access from the host to guest memory pages. On the other hand, pKVM enables stage 2 protection even in host context, and puts the hypervisor in charge of protecting guest memory pages instead of the host."
> — <https://source.android.com/docs/core/virtualization/architecture>

这段的解释力在于：**普通 KVM 给不了机密性，不是因为 KVM 有缺陷，而是因为它按设计在宿主态关掉了第二级翻译。** 宿主的内核页表访问直接穿过 stage-2，于是宿主对 guest 内存页没有任何限制。任何"在 KVM 上跑一个受保护 VM"的想法，只要宿主态仍是这个模式，保护承诺就落不了地。

pKVM 的改动是把 stage-2 在宿主态也保留，并把"保护 guest 内存"的责任**从宿主移交给 hypervisor**。内核文档给出的实现细节是页表级的：

> "Booting a host kernel with ‘kvm-arm.mode=protected’ enables “Protected KVM” (pKVM). During boot, pKVM installs a stage-2 identity map page-table for the host and uses it to isolate the hypervisor running at EL2 from the rest of the host running at EL1/0."
> "The hypervisor isolates pVMs from the host by unmapping pages from the stage-2 identity map as they are accessed by a pVM. Hypercalls are provided for a pVM to share specific regions of its IPA space back with the host, allowing for communication with the VMM."
> — <https://docs.kernel.org/virt/kvm/arm/pkvm.html>

两条机制事实值得单独记住：

1. **隔离的方式是"取消映射"，不是"加密"。** pVM 触及的页会从宿主的 stage-2 恒等映射里被摘掉。信息不在宿主地址空间里被"藏起来"，而是**根本不可达**。把 pVM 说成"内存加密的 VM"是常见误传（§14 第 5 条）——加密可以是额外手段，但官方在这里描述的隔离机制是页表。
2. **共享是显式、按页、由 guest 主动发起的。** 通信需要 pVM 通过 hypercall 把特定 IPA 区域交回给宿主。也就是说，"默认不共享、共享要举手"这条纪律是设计的一部分，而不是实现的偶然结果——它让"哪块内存被共享了"成为一个可审计的事实。

**边界提醒。** 上述内核文档自述为开发中的特性，**不能直接等同于 Android 商用实现的当前状态**；本手册只把它当作"pKVM 机制如何工作"的说明，不据此对具体设备下结论。

---

## 7. 信任边界在哪一刻切换：启动期仍信 bootloader

pKVM 的"自我保护"不是从加电第一纳秒就成立的。官方把启动流程拆成两步，并明确写下切换点：

> "Deprivilege Kernel: The generic kernel detects that it's running at EL2 and deprivileges itself to EL1. pKVM and its modules continue to run at EL2."
> "After the kernel is deprivileged, it's no longer considered trusted by the hypervisor, which is then responsible for protecting itself even if the kernel is compromised."
> "The boot procedure trusts the bootloader to verify and maintain the integrity of the kernel image for the initialization phase."
> — <https://source.android.com/docs/core/virtualization/architecture>

读法有三层：

- **切换之前**，内核仍以 EL2 特权运行，hypervisor 还没"独立"；此时对内核镜像完整性的信任由 **bootloader** 提供。
- **切换之后**，内核降到 EL1，hypervisor 不再信任它，并自行负责保护自己。这才是"即使宿主内核被攻陷，pVM 的机密性仍成立"这句话的成立区间。
- 因此，**启动链的完整性是 pVM 保护承诺的前置条件**。bootloader 校验失效时，pKVM 保护的是一个"已经被换过的宿主"，而不是官方描述的那个威胁模型。这条与 [系统更新链路与回滚防护手册](/android-ota-update-rollback/) 中的 verified boot 与回滚保护是同一根链条的不同段，也解释了为什么 [内核与 GKI 驱动攻击面手册](/android-kernel-gki-drivers/) 要把 `boot.img` 的签名主体单独列一节。

---

## 8. pVM 与 Microdroid：保护对象、最小化的真实含义，以及换来的新接口面

pVM 的官方定义把"保护什么"写得非常具体：

> "protected virtual machine (pVM) — A mutually distrusted isolated execution environment (guest) that runs alongside the main Android operating system (host). One important aspect of pVM security is even if the host is compromised, the host doesn't have access to a pVM's memory. pKVM is the standard hypervisor for running pVMs."
> "Compared to existing trusted execution environments (TEEs), pVMs provide a richer environment, including the ability to run a mini-Android distribution called Microdroid (though Microdroid can also run on an unprotected VM)."
> — <https://source.android.com/docs/core/virtualization>

注意 "mutually distrusted"（互相不信任）这个词：宿主不信 pVM，pVM 也不信宿主。这是对等隔离，不是"主人管租客"。

Microdroid 是把"富环境"落地的那个组件：

> "Microdroid is a mini-Android OS that runs in a pVM. You don't have to use Microdroid, you can start a VM with any OS."
> "Microdroid is a stripped down version of Android with a few additional components specific to pVMs."
> "Native code is built against Bionic, communication happens over Binder, and it allows importing APEXes from the host Android and exposes a subset of the Android API, such as keystore for cryptographic operations with hardware-backed keys."
> "vendor_boot.img - Contains VM-specific kernel modules, such as virtio."
> — <https://source.android.com/docs/core/virtualization/microdroid>

三句读法：

- **"stripped down"是相对 Android，不是相对 TEE。** 它比宿主 Android 小得多，但它带着 Bionic、Binder、APEX 导入与一部分 Android API。与 TEE 的极小 TCB 相比，这是一次**用 TCB 规模换取可用性**的交换。
- **"You don't have to use Microdroid"** —— pVM 不是 Microdroid 的同义词。把"pVM"与"Microdroid"混用，会导致在评估时把 Microdroid 的镜像内容当成隔离边界本身。
- **Binder 是跨 VM 的主要通信手段**（§4 术语表原话）。这把一个成熟的 IPC 面搬进了隔离边界两侧——成熟意味着有历史、有复杂度、有解析器，也意味着它继承了自己的漏洞类别。

**这就是"新增隔离＝新增接口面"这条通用规律的实例。** 隔离边界每增加一条通道（binder、authfs、virtio 设备模型），就增加一处必须被正确解析、正确鉴权、正确生命周期管理的代码；而这处代码往往在宿主侧或跨域，恰好不在 hypervisor 的保护承诺内（§4 末的读法结论）。评估 pVM 方案时，"隔离做得多好"与"通道做得多窄"必须一起看。

---

## 9. 隔离的边界：pKVM 明确保证 C 与 I，明确不保证 A

这一节是整个主题里最容易被宣传语言覆盖掉的部分，而官方写得毫不含糊：

> "pKVM is designed to maintain the confidentiality and integrity of VMs. To ensure a smoother user experience, by design, the host VM can influence the availability of guest VMs."
> "Actors who have access to the device's supply chain, such as OEMs and SoC vendors, are still necessarily trusted."
> — <https://source.android.com/docs/core/virtualization/security>

两条例外的性质完全不同，必须分开记：

| 维度 | 是否保证 | 官方措辞要点 | 实践含义 |
| --- | --- | --- | --- |
| 机密性（C） | 保证 | "maintain the confidentiality ... of VMs" | 宿主读不到 pVM 内存 |
| 完整性（I） | 保证 | "maintain the ... integrity of VMs" | 宿主改不了 pVM 的执行 |
| 可用性（A） | **不保证（设计如此）** | "the host VM can influence the availability of guest VMs" | 拒绝服务、资源耗尽型威胁不在防护范围内 |
| 供应链 | **不在模型内** | "OEMs and SoC vendors, are still necessarily trusted" | 供应链级攻击者可直接控制信任根 |

**读法结论（写评估报告时的口径）。** 遇到"pVM 可以防住某某威胁"的说法，先按这张表分流：

- 如果威胁属于**读取或篡改 pVM 内容**，且在宿主被攻陷的前提下——属于 pKVM 的承诺范围。
- 如果威胁属于**让 pVM 不可用、变慢、被抢资源**——按官方是设计例外，**不要**写进防护承诺。把它写成"pVM 能防"是过度承诺，代价在真实评估里会变成误判。
- 如果威胁来自 OEM/SoC 供应链——按官方是"necessarily trusted"，不在隔离模型内。这与 [TEE、密钥管理与设备认证手册](/android-tee-key-attestation/) 里"信任根由谁掌握"是同一条轴。

---

## 10. 另外两条边界：Trusty TEE 与 StrongBox

pVM 不是 Android 上唯一的强隔离。TEE 侧与独立芯片侧各有一条边界，且与 hypervisor **并存**（§4 已引 "AVF doesn't fully replace TrustZone."）。

Trusty 的官方描述把三个事实钉在一起：

> "Trusty is a secure operating system (OS) that provides a Trusted Execution Environment (TEE) for Android. The Trusty OS runs on the same processor as the Android OS, but Trusty is isolated from the rest of the system by both hardware and software."
> "Trusty's isolation protects it from malicious apps installed by the user and potential vulnerabilities in Android."
> "A Linux kernel driver to transfer data between the secure environment and Android"
> — <https://source.android.com/docs/security/features/trusty>

三个事实的读法：

1. **同一个处理器**——所以它的隔离不可能靠"另一颗芯片"，只能靠硬件隔离机制加软件边界。这也解释了为什么它的通信必须经过一个**专用的 Linux 内核驱动**。
2. **保护目标是"用户装的恶意应用"与"Android 本身的缺陷"**——注意后半句：TEE 的威胁模型明确包含"Android 被攻陷"，这与 pVM 的威胁模型同向。
3. **那一个内核驱动就是边界上的接口**——它既是唯一通道，也是 TEE 侧在 Android 侧的攻击面入口。评估 TEE 时，"TA 写得多安全"和"这条驱动的解析面有多窄"是两件必须分开看的事。

StrongBox 是再往下一层，但官方的语气是**资格要求**，不是绝对承诺：

> "It's further recommended to also support a StrongBox KeyMint, which is implemented in tamper-resistant hardware."
> "Ensure that the StrongBox Keymaster is running in an environment that has a discrete CPU, secure storage, a high quality true random number generator, tamper resistant packaging, and side channel resistance to meet the requirements to qualify as a StrongBox Keymaster."
> — <https://source.android.com/docs/security/best-practices/hardware>

逐项读这五个条件：**独立 CPU、安全存储、高质量真随机、防篡改封装、侧信道抵抗力**。这是一条"要满足这些条件才能叫 StrongBox"的准入清单。它支撑的结论是"StrongBox 的隔离层级高于普通 TEE"，**不**支撑"StrongBox 不可攻破"——官方没有给出任何绝对化措辞（§14 第 15 条）。

---

## 11. 公告里的虚拟化条目：形态、三个 Critical，以及一个容易混淆的标签

虚拟化条目在月度公告里没有什么特殊的表结构，仍是 Kernel 章节下的 `CVE / References / Type / Severity / Subcomponent` 五列。2025-12-01 期的实测值：

| CVE | References | Type | Severity | Subcomponent |
| --- | --- | --- | --- | --- |
| CVE-2025-48623 | A-436580278 / Upstream kernel [2] | EoP | Critical | pKVM |
| CVE-2025-48637 | A-443763663 / Upstream kernel [2] | EoP | Critical | pKVM |
| CVE-2025-48638 | A-442540376 / Upstream kernel [2] | EoP | Critical | pKVM |
| CVE-2025-48610 | A-432439762 / Upstream kernel [2] | ID | High | Kernel Virtual Machine |

> 表格说明句逐字："The most severe vulnerability in this section could lead to local escalation of privilege with no additional execution privileges needed."
> — <https://source.android.com/docs/security/bulletin/2025-12-01>

四条结论：

1. **hypervisor 层不是"没有漏洞"的代名词。** 同一期内三个 pKVM 条目都是 Critical 级 EoP。任何"隔离层因为层级高所以不会出问题"的假设都被这张表直接推翻。
2. **虚拟化子组件的缺陷类型不止提权。** 同页有一条 High 级 `ID`（信息泄露），Subcomponent 是 **"Kernel Virtual Machine"**。
3. **"pKVM" 与 "Kernel Virtual Machine" 是两个不同的 Subcomponent 标签**，不能合并统计，也不能互相代指。做虚拟化条目的趋势统计时，把两者混为一谈会得出错误的时间序列。
4. **公告只给五列，不给根因。** 表格没有函数名、没有补丁片段，References 里的 `[2]` 指向内核提交链接。本手册不据 `[2]` 对具体成因下结论（见未核实节末的说明）。

**读法提醒。** 这四条只说明"这一期公告里有哪些虚拟化条目"，不说明"这些条目覆盖了虚拟化代码的全部问题"。公告是修复披露，不是攻击面清单——这个区别在 [漏洞编号与公告体系手册](/android-cve-and-security-bulletins/) 里有更完整的展开。

---

## 12. 隔离失效的五类成因模式

把前面各节的机制事实收敛成可复用的成因分类。每类都只描述"缺陷落在哪一层"，不提供任何构造方法。

**第一类：边界两侧共享同一份内存或对象。** 这是 pKVM 用 stage-2 取消映射、并要求 guest 显式 hypercall 才交回页面的直接原因（§6）。共享是有代价的：一旦共享区里的对象生命周期没有被两侧一致地管理，释放后使用、双重释放这类缺陷就会跨越边界生效。**观测信号**：某条通道是否被文档明确写为"显式、按页、可审计"；没有这条纪律的共享通道，默认按高风险对待。

**第二类：接口面膨胀。** 每新增一条跨域通道（binder、authfs、virtio 设备模型），就新增一个解析器与一套鉴权逻辑（§4、§8）。**观测信号**：通道数量与复杂度是否随用例增长而失控；"富环境"（Microdroid 带 Bionic/Binder/APEX）换来的可用性，是否被算进了 TCB 规模。

**第三类：策略与配置漂移。** 前两层（UID/DAC、SELinux、seccomp-bpf）都是"策略驱动"的边界（§2），它们的强度取决于策略镜像与设备配置，而不是取决于代码。**观测信号**：策略是否随版本单调收紧；是否存在为兼容而放开的域或系统调用。

**第四类：信任根错位。** pKVM 在启动早期信任 bootloader（§7），pKVM 的威胁模型把 OEM/SoC 供应链排除在外（§9），TEE 隔离依赖独立启动链（§10）。**观测信号**：任何"隔离保证"的成立条件里，是否隐含了"启动链与供应链可信"这个前提；评估时不写这个前提，结论就会比实际强。

**第五类：把可用性当隔离的一部分。** 官方明文写"宿主可以影响 pVM 可用性"（§9）。把拒绝服务类威胁算进隔离防护承诺，会导致优先级排序失真：真正需要防的是 C/I 失守，而 A 的问题应当按"资源与调度"而不是"隔离"来治理。

---

## 13. 观测与自查

本节的每一条都对应一个**官方可核对的口径**，不给设备侧私有路径或未验证的属性名（那些进 [内核与 GKI 驱动攻击面手册](/android-kernel-gki-drivers/) §13 的同类标注规则）。

| # | 观测点 | 怎么判读 | 依据 |
| --- | --- | --- | --- |
| 1 | 应用是否同 UID 共享进程 | 同 UID 即第一层边界在该对应用间不存在 | §2 UID 口径 |
| 2 | 应用是否受 seccomp-bpf 约束 | Android 8.0 起应为 "all apps" | §2 引文 |
| 3 | 应用是否单独 SELinux 沙箱 | Android 9 起、非特权、targetSdk ≥ 28 三条件 | §2 引文 |
| 4 | 提权后是否仍受 MAC 约束 | 是——root 不豁免 SELinux | §2 引文 |
| 5 | 设备是否为 ARM64 | AVF 参考实现仅 ARM64 | §4 引文 |
| 6 | 系统版本是否 ≥ 13 | AVF 自 Android 13 引入 | §4 引文 |
| 7 | 是否存在 pVM 生命周期服务 | `VirtualizationService` 是官方组件名 | §4 术语表 |
| 8 | 内核是否以 protected 模式启动 | 内核文档给出 `kvm-arm.mode=protected`（开发中特性表述） | §6 引文 |
| 9 | pVM 共享面是否显式 | 官方机制为 hypercall 按页交回 | §6 引文 |
| 10 | 信任根由谁掌握 | bootloader（启动期）、OEM/SoC（供应链） | §7、§9 引文 |
| 11 | TEE 是否等于 AVF | 否，两者并存 | §4 引文 |
| 12 | StrongBox 是否满足五项准入 | 独立 CPU/安全存储/TRNG/防篡改封装/侧信道抵抗 | §10 引文 |

**自查的三条方法论结论。**

1. **先定边界，再谈强度。** 同一句"隔离"在不同层上是不同的承诺；把层定错，后面的强度评估全部作废。
2. **把"不保证"写进结论。** 可用性例外与供应链信任例外是官方明文，写进报告不是减分项，是准确性。
3. **公告计数不等于风险计数。** §11 的四条只反映披露，不反映攻击面；用披露数量做"哪个子系统更危险"的排序，会系统性低估少披露但接口面宽的子系统。

---

## 14. 常见误传纠正

| # | 常见说法 | 官方口径 | 出处 |
| --- | --- | --- | --- |
| 1 | AVF 会取代 TrustZone/TEE | "AVF doesn't fully replace TrustZone." | §4 |
| 2 | pVM 被攻陷宿主也没事，绝对安全 | 只保证 C 与 I，A 明确可由宿主影响 | §9 |
| 3 | pKVM 是 Google 全新写的 hypervisor | "built upon the Linux KVM hypervisor"，属扩展 | §5 |
| 4 | 普通 KVM 也能提供 pVM 级机密性 | 宿主态 stage-2 被关闭，宿主可无限制访问 guest 内存 | §6 |
| 5 | pVM 的内存是"加密"的所以宿主读不到 | 机制是 stage-2 取消映射；加密不是本节描述的隔离手段 | §6 |
| 6 | 跑在 pVM 里的只能是 Microdroid | "You don't have to use Microdroid, you can start a VM with any OS." | §8 |
| 7 | AVF 只在 Pixel 上可用 | 官方只说 "supported only on ARM64 devices"，无机型清单 | §4、未核实 E |
| 8 | Android 12 就有 AVF | "Android 13 introduces secure virtualization as ... AVF." | §4 |
| 9 | hypervisor 和 TEE 是同一件事 | 两条不同边界，安全世界与 EL2 分属不同信任根 | §4、§10 |
| 10 | 拿到 root 就等于突破沙箱 | SELinux 对包括 root 在内的进程强制 MAC | §2 |
| 11 | seccomp-bpf 是 Android 9 才引入的 | Android 8.0 起 "all apps" 即受约束 | §2 |
| 12 | 所有应用从 Android 9 起都有独立 SELinux 沙箱 | 仅非特权应用且 targetSdk ≥ 28 | §2 |
| 13 | 虚拟化漏洞必然是 Critical 提权 | 同页有 High 级信息泄露条目 | §11 |
| 14 | 公告里 pKVM 与 Kernel Virtual Machine 是同一子组件 | 两个不同 Subcomponent 标签 | §11 |
| 15 | StrongBox 不可攻破 | 官方是"满足五项条件才够格"的资格表述，无绝对化承诺 | §10 |
| 16 | pVM 的隔离能防住资源耗尽 | 可用性明确例外 | §9 |
| 17 | Microdroid 的"最小化"等于 TEE 级小 TCB | 它带 Bionic/Binder/APEX 与部分 Android API | §8 |

---

## 15. 与其他手册的分工

| 问题 | 归属 |
| --- | --- |
| SELinux 策略的写法、neverallow、seccomp 白名单的构造 | [SELinux 策略与 seccomp 沙箱手册](/android-selinux-seccomp-sandbox/) |
| 用户/工作资料/隔离进程的 UID 与域映射 | [多用户、工作资料与进程隔离手册](/android-multiuser-sandbox-isolation/) |
| 内核镜像、模块、签名主体、GKI 更新语义 | [内核与 GKI 驱动攻击面手册](/android-kernel-gki-drivers/) |
| 启动链完整性、verified boot、回滚保护 | [系统更新链路与回滚防护手册](/android-ota-update-rollback/) |
| 密钥存储、证明链、StrongBox 的密钥用途 | [TEE、密钥管理与设备认证手册](/android-tee-key-attestation/) |
| 公告字段的通用读法与 SPL | [漏洞编号与公告体系手册](/android-cve-and-security-bulletins/) |
| Critical/High 的定级与可利用性要素 | [安全性评估与定级手册](/android-vulnerability-severity-assessment/) |
| Hypervisor 条目之外的驱动与 ioctl 成因模式 | [底层漏洞类型图谱手册](/android-low-level-vulnerability-types/) |

---

## 参考

**官方一手来源（本手册逐字引用过）**

- Android Virtualization Framework（overview）— <https://source.android.com/docs/core/virtualization>
- Why AVF — <https://source.android.com/docs/core/virtualization/whyavf>
- AVF architecture（含 Deprivilege Kernel、stage-2）— <https://source.android.com/docs/core/virtualization/architecture>
- AVF security（C/I/A 取舍、供应链信任）— <https://source.android.com/docs/core/virtualization/security>
- Microdroid — <https://source.android.com/docs/core/virtualization/microdroid>
- pKVM（Linux 内核文档，开发中特性）— <https://docs.kernel.org/virt/kvm/arm/pkvm.html>
- Trusty TEE — <https://source.android.com/docs/security/features/trusty>
- Application sandbox — <https://source.android.com/docs/security/app-sandbox>
- SELinux 概览 — <https://source.android.com/docs/security/features/selinux>
- 硬件安全最佳实践（StrongBox 准入条件）— <https://source.android.com/docs/security/best-practices/hardware>
- Android Security Bulletin 2025-12-01 — <https://source.android.com/docs/security/bulletin/2025-12-01>

**相邻手册**：见 §15。

---

## 未核实与失效来源

**A. pKVM 随 Pixel 6 / Tensor 首批商用**
推测：pKVM 在 Android 13 随 Pixel 6 系列率先默认启用。未核实原因：独立页 `https://source.android.com/docs/core/virtualization/pkvm?hl=en` 返回 **404**（内容已并入 architecture/security 两页）；architecture、security、whyavf 三页全文无 "Pixel" 相关机型表述；站外命中均为第三方博客，按规则不作一手引文。**正文只写"Android 13 引入 AVF"，不写首发机型。**

**B. `isolatedProcess` 组件的语义**
推测：`android:isolatedProcess="true"` 提供无权限、独立 UID 的隔离进程。未核实原因：现行 app-sandbox 页对 "isolated" 零命中（该页只讲 UID/DAC、SELinux、seccomp-bpf 与按用户隔离）；旧路径 `/docs/core/security/app-sandbox`、`/docs/core/security/selinux` 均 404。属性定义应在 developer.android.com 的 manifest 参考中，本次未抓取。**正文不引用该属性语义。**

**C. Microdroid 的 `vm_payload` 服务名**
推测：guest 内以 `vm_payload` binder 服务向 payload 暴露接口。未核实原因：microdroid 页与 writeavfapp 页全文均无 `vm_payload` 字符串；该名字可能只存在于 AOSP 源码与 Javadoc。**正文不写服务名。**

**D. 公告中的 Trusty 子组件历史条目**
推测：往期公告存在 Subcomponent 为 "Trusty" / "Trusty TLK" 的条目。未核实原因：对 2025-12-01、2025-09-01、2025-06-01、2025-03-01、2024-12-01 五期公告全文检索 "Trusty"，**命中 0 条**。因此正文**不用公告举证 TEE 侧的隔离失效历史**；§10 只写架构与隔离层级，不写案例。

**E. AVF/pVM 的机型与版本可用性矩阵**
推测：AVF 自 Android 13 起在部分 Pixel 与少数 OEM 旗舰上可用，需设备声明某特性常量。未核实原因：official overview 仅给 "AVF is supported only on ARM64 devices."，无版本/机型矩阵；tryavf 页亦无设备清单；特性常量名未取得官方逐字表述。**正文不写机型与特性常量。**

**F. virtio 在 AVF 中的正式角色**
推测：pVM 设备模型以 virtio 为主（crosvm 为后端），用于块设备、网络与 vsock。未核实原因：AVF overview 术语表未收录 virtio；只有 Microdroid 页出现一句 "vendor_boot.img - Contains VM-specific kernel modules, such as virtio."，不足以支撑"virtio 是 AVF 设备模型标准"。**正文只在 §8 按原意引用该句，不单列结论。**

**G. 公告 References 中 `[2]` 的内核提交内容**
本手册引用 2025-12-01 期四条虚拟化条目时，`[2]` 指向 android.googlesource.com 的内核提交链接。本次**未抓取该提交内容**，因此正文不对四条 CVE 的根因作任何断言，只使用表格单元格的逐字值。

**H. 抓取经验（便于复现）**
source.android.com 会按访问 IP 本地化正文语言，**必须在 URL 后加 `?hl=en`** 才拿到英文原文；加完仍非英文的页面以「未核实」处理，不把译文当原文。公告 URL 现行形态为 `bulletin/YYYY-MM-01`。

---

## 联系方式

咨询 Android 系统详情请联系 https://t.me/one00190

---

*本手册仅供安全学习与防御研究使用；所有组件名、保证范围、公告列值与版本口径均取自公开一手来源并附链接，未能核实的表述已单列于「未核实与失效来源」而非写入正文。*
