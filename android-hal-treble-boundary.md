---
layout: page
title: "Android HAL 与 Treble 边界攻击面手册"
description: "把「厂商代码怎么被系统调用」拆成四道互不替代的关卡：VINTF 声明只回答「提供什么／要求什么」而非访问控制、binderized 与 passthrough 的地址空间差别（SP-HAL 库被加载进框架进程）、servicemanager 与 hwservicemanager 两个不互通的注册中心、V..."
date: 2026-10-30
permalink: /android-hal-treble-boundary/
---

> Treble 常被讲成一句「把 vendor 和 system 拆开，方便升级」。这句话没错，但它只描述了**收益**，没有描述**代价**。代价是：Android 里凭空多出了一条横跨两个编译世界、两套 ABI、两个 service manager、两种 SELinux 域体系的接口边界，而绝大多数厂商代码、闭源驱动、DSP 固件都挂在这条边界上。攻击者关心的从来不是「拆开好不好升级」，而是「这条边上谁可以调谁、调的时候做不做检查」。这份手册按「边界怎么形成 → 谁在边界上说话 → 边界上出过什么错 → 怎么把它守住」展开，只讨论公开且已修复的问题的机制与自查方法，不含可利用代码。

---

## 0. 这篇手册解决什么问题

已发布的手册里，《内核与 GKI 驱动攻击面》讲的是**边界之下**（内核态、驱动、ioctl），《框架 IPC 与组件导出》讲的是**边界之上**（system_server 里的 Java 组件），唯独中间那一层——**HAL**——一直是被跳过的。跳过它的原因很实际：HAL 的官方文档以「怎么移植」为主，安全语义散落在 VINTF、SELinux、linker namespace 三个互不相邻的页面里，很少被拼到一起讲。

但这一层恰好是 Android 攻击面上**信噪比最高**的地方之一，理由有三个：

1. **它是唯一一处「厂商代码按设计必须被系统进程调用」的地方。** 其他位置厂商代码要么在自己进程里，要么根本跑不起来；HAL 的存在意义就是让 system 侧的客户端跨边界调用 vendor 侧的实现。
2. **它的接口是「带版本号的契约」。** 版本一旦冻结就长期兼容，这意味着一个接口设计缺陷的生命周期以「年」计，而不是以「一个安全补丁」计。
3. **它的校验责任被**分摊**了。** 类型检查在 AIDL/HIDL 生成代码里，权限检查在 SELinux 里，存在性检查在 VINTF 里，三处任何一处缺位都不会有明显报错——只会静默放行。

因此本文的顺序是刻意的：先把边界画清楚（§1–§3），再把「谁有权说话」讲透（§4–§7），最后才谈缺陷成因与防护（§8–§11）。

---

## 1. 边界从哪来：vendor 与 system 的分割

Treble 的核心不是某个具体接口，而是一条**编译期与运行期同时生效的分割线**：

- **编译期**：vendor 分区里的二进制只能链接 vendor 可用的库（VNDK 快照），system 分区的二进制只能链接 system 库。官方对此的表述是，这样做 "ensures that modules in the system partition won't depend on the shared libraries in the vendor partitions and vice versa"。
- **运行期**：两侧进程分属不同的 SELinux 域体系（`vendor` 属性域 vs 平台域），跨侧通信必须经过明确授权的 Binder 通道。

这条线带来的第一个安全后果很直接：**system 侧的一切加固（CFI、scudo、更强的 seccomp、更新的编译器）不会自动惠及 vendor 侧**，反之亦然。两侧的防御水平是**独立演进**的。评估一台设备时，「Android 版本号」只说明 system 侧，vendor 侧的补丁状态要另看（这也是为什么公告里 vendor 组件条目常常滞后于框架条目）。

第二个后果更微妙：**分割线本身成了攻击面**。任何「让 system 进程加载 vendor 代码」的机制，都会把 vendor 侧的缺陷直接搬进高权限进程的地址空间——这正是 §3 要讲的 passthrough 模式。

---

## 2. 三个「声明」：manifest、compatibility matrix 与 VINTF object

Android 不靠「跑起来试试」来判断 vendor 与 system 是否兼容，而是靠**静态声明 + 启动时校验**。官方对这套机制的定义是：

> "The vendor interface object (VINTF object) aggregates relevant information about a device and makes that information available through a queryable API."

而两个声明文件的分工，官方用一句话说完了：

> "In general, a manifest describes what is provided and a compatibility matrix describes what is required."

| 文件 | 语义 | 典型路径 |
|---|---|---|
| vendor manifest | 「我提供了什么」 | `/vendor/etc/vintf/manifest.xml`；旧式单文件为 `/vendor/manifest.xml`（legacy, no fragments） |
| device manifest | 「设备整体提供了什么」 | 由 vendor manifest 与 ODM manifest 合并而成 |
| compatibility matrix | 「系统要求什么」 | 构建期输出到 `/vendor/etc/vintf/compatibility_matrix.xml` 与 `/etc/vintf/compatibility_matrix.device.xml` |

三个容易踩的点：

**其一，「声明」是准入条件，不是能力清单。** AIDL HAL 有一条硬规则：标注 `@VintfStability` 的 AIDL 服务端**必须**在 VINTF manifest 里声明；AIDL 客户端**必须**在 compatibility matrix 里声明自己。未声明的后果不是「降级可用」，而是这条通道根本不存在。官方文档在讲 `getService` 时只说「按名字与版本请求」，且 "if the transport method isn't available, nullptr is returned"——注意是返回空指针，**不是抛异常**。这意味着调用方如果不判空，会得到一个更难排查的空引用崩溃，而不是一条清晰的「未声明」错误。

**其二，工具名经常被写错。** AOSP 里 `checkvintf` 是 `cc_binary_host`——**构建期在主机上跑的工具**，源码是 `check_vintf.cpp` + `HostFileSystem.cpp`。设备上同名的模块是 `cc_binary { name: "vintf" }`（源码 `main.cpp`）。所以「在设备上执行 `checkvintf --check-compat`」这种写法是错的；设备侧应当用 `vintf`（或直接读 `/vendor/etc/vintf/` 下的文件比对）。同理 `assemble_vintf` 也是 host 工具，官方描述为 "a host-side tool that: … Verifies a compatibility matrix or manifest file is valid."

**其三，列举 HAL 要用两个工具。** `lshal` 的官方说明是 "lshal: List and debug HIDL HALs."，并明确补了一句 "for AIDL HALs, see `dumpsys`"。源码位于 `frameworks/native/cmds/lshal`，**不在** `system/libvintf`。所以只跑 `lshal` 得到的是**半个视图**——AIDL HAL（也就是 Android 11 之后新增的绝大多数 HAL）它根本看不见。这是自查时最常见的漏项。

---

## 3. 两种传输：binderized 与 passthrough

同一个 HAL 接口可以有两种落地方式，官方定义分别是：

> **binderized**: "Indicates HIDL is being used for remote procedure calls between processes, implemented over a Binder-like mechanism."

> **passthrough**: "the server is a shared library, `dlopen`ed by the client. In passthrough mode, client and server are the same process but separate codebases."（并注明 "not supported in Java"）

这段引文值得逐字读两遍，因为它把安全模型说透了：

- binderized 模式下，vendor 代码在**自己的进程**里跑，受自己的 SELinux 域、自己的 seccomp 策略、自己的崩溃边界约束。它被打穿，最多是那个 HAL 进程沦陷。
- passthrough 模式下，vendor 的 `.so` 被 `dlopen` 进**调用方进程**。官方在讲 VNDK 与 linker namespace 时的原话是："SP-HAL shared libraries and their dependencies, including VNDK-SP libraries, are loaded into framework processes."——**被加载进框架进程**。

也就是说：passthrough 模式下一个 vendor 库里的内存破坏缺陷，其影响半径是**宿主进程**，而不是那个库自己。这也解释了为什么 `hal_client_domain` 宏里会有一句注释：客户端被授予运行 passthrough 模式 HAL 所需的权限（"always grant clients of a HAL sufficient access to run the HAL in passthrough mode (i.e., in-process)"）——这条授权在 SELinux 层面是必要且合规的，但它同时意味着**「客户端域」与「HAL 实现域」在 passthrough 情况下没有边界**。

写作时必须守住的一条纪律：**官方文档里没有「因为地址空间污染所以限制 passthrough」这样的整句表述**。可以引用的是上面这些「同进程 / dlopen / 加载进框架进程」的事实，以及 sepolicy 里那条 in-process 授权注释；把推论写成引用是这一章最容易犯的错。

---

## 4. 两个 service manager：最容易记反的一处

Android 有**两个**服务注册中心，它们不互通：

- `servicemanager`：平台侧，管 AIDL 服务（含 AIDL HAL），上下文文件是 `plat_service_contexts` / `service_contexts`。
- `hwservicemanager`：HIDL 侧，用于 vendor 与 system 之间。官方有一句全大写的强制句：

> "This is because all interaction between `vendor` and `system` processes MUST go through `hwservicemanager`/`hwbinder`."

- 此外还有 `vndservicemanager`，读取 `vndservice_contexts`（官方描述："loaded by `vndservicemanager`"）。

**记反的典型表现**是「AIDL HAL 注册到 hwservicemanager」。事实相反：AIDL HAL 走 `servicemanager`，其注册规则由 context manager 执行——官方原话 "The enforcement of these registration rules is done by the context manager (`servicemanager`)"，对应宏是 `hal_attribute_service`（"Ability for domain to get a service to service_manager and find it."）。HIDL 才走 `hwservicemanager`，对应的是 `plat_hwservice_contexts` / `vendor_hwservice_contexts`。

为什么这个区分对安全重要：**「谁能在哪个 manager 里注册什么名字」是权限模型的第一道闸。** 一个域如果能往 `servicemanager` 里注册一个系统期望的 AIDL HAL 名字，它就成了所有该 HAL 客户端的「上游」；如果只能往 `hwservicemanager` 注册，影响面被限制在 HIDL 客户端。两个 manager 的准入表是分开审计的，把它们的归属搞混，等于审计错了对象。

---

## 5. HIDL → AIDL：迁移的时间线与安全含义

时间线要以官方口径为准，因为这里流传着多个错误版本：

> "As of Android 10, HIDL is deprecated and has been replaced By Android Interface Definition Language (AIDL)."
> "Android 11 introduces the ability to use AIDL for HALs in Android, making it possible to implement parts of Android without HIDL."

即：**Android 10 弃用 HIDL，Android 11 引入 AIDL for HALs**。常见的「HIDL 在 Android 13 被弃用」是错的；「Android 13 起禁止新增 HIDL 接口」这句在官方文档里也**查不到**（文档只说 "Transition HALs to use AIDL exclusively where possible (when upstream HALs use HIDL, HIDL must be used)."），属于**未核实**，不应作为事实引用。

命名形态也随迁移改变，这是自查时常被忽略的细节：

| 形态 | 实例名写法 | 清单写法 |
|---|---|---|
| HIDL | `android.hardware.foo@1.0::IFoo/default` | `<name>` + `<version>` + `<interface>` |
| AIDL | `android.hardware.foo.IFoo/default` | `<name>android.hardware.foo</name>` + `<fqname>IFoo/default</fqname>`，`format="aidl"` |

官方对 AIDL 的命名约定原话是："By convention, AIDL HAL services have an instance name of the format `$package.$type/$instance`."，例子给的是 `android.hardware.vibrator.IVibrator/default`。**AIDL 实例名里没有 `@x.y` 版本段**——版本写在 manifest 的 `<version>` 元素里。凡是看到「AIDL HAL 叫 `android.hardware.foo@2.0.IFoo/default`」的写法，基本可以判定是照搬 HIDL 形态。

Native 侧的链接目标也常被搞错：AIDL 的 NDK 后端是 **`libbinder_ndk`**（官方："native vendor code must use the NDK backend of AIDL, link against `libbinder_ndk` (which is backed by system `libbinder.so`)"），而 `libhwbinder` 是 **HIDL 的 hwbinder 传输库**。把 `libhwbinder` 说成「AIDL 的 NDK 后端」是两个概念混用，在评审 vendor 代码时会导致完全错误的结论。

安全含义上有两点值得记住：

1. **AIDL 默认更强调「稳定性标注」。** `@VintfStability` 把「这个接口可以被跨分区调用」变成显式声明，而不是默认行为。这降低了「无意中把内部接口暴露成跨边界契约」的概率，但一旦标注，版本冻结的长期负担也随之产生。
2. **迁移期是双栈并存的。** 一台 Android 11+ 设备上通常同时有 HIDL 与 AIDL HAL。审计时只查一侧就等于只查了一半——这也是为什么 §2 强调 `lshal` 必须配 `dumpsys`。

---

## 6. VNDK 与 linker namespace：从「库共享」到「库隔离」

VNDK 的官方定义是：

> "a set of libraries used by other libraries or binaries, in the vendor or product partition, on runtime for dlopen."

其中 VNDK-SP 是 "a predefined subset of eligible VNDK libraries"，**SP = same-process**，也就是允许被**框架进程**加载的那一小撮库（与 §3 的 passthrough 直接相关）。常被写反的说法是「VNDK-SP 是给 vendor 进程用的」——正好相反，它是为「被 system 侧同进程加载」而设的白名单。

**VNDK 已被弃用**，这一点很多资料还没更新：

> "Based on these issues, we decided to deprecate VNDK starting with Android 15."

替代方式是 "former VNDK libraries are installed in the vendor (or product) image, same as other vendor-available libraries."；同时 Android 15 起移除了 VNDK APEX 与 `ro.vndk.version` / `ro.product.vndk.version` 属性（旧口径曾要求 "VTS mandates a non-empty `ro.vndk.version` property."）。所以「`ro.vndk.version` 是必备属性」这句话，在 Android 15 之后对以 15 构建的 vendor/product 分区已不成立——自查脚本里如果还把它的缺失当作异常，会误报。

与 VNDK 配套的是 **linker namespace**。厂商侧有一套独立的命名空间，官方对应用侧的说法是：

> "The namespace for the app native libraries has a link that allows only the library names specified in `public.libraries.txt`."

配套的几处事实：

- "In Android 7.0 and higher, system libraries are separated from app libraries."
- `/vendor/etc/public.libraries.txt` 用于 "libraries from silicon vendors"，`/system/etc/public.libraries-COMPANYNAME.txt` 用于厂商自有库。
- SDK 31+ 起，应用还需在 manifest 里声明 `<uses-native-library>`，否则 "apps can't access native shared libraries that don't appear in the app manifest."

**命名精度提醒**：AOSP `libnativeloader/library_namespaces.cpp` 里命名空间名是常量 `kClassloaderNamespaceName`（拼接 `-%d` 计数）；字面串 `classloader-namespace` 在已取到的一手文件中**未出现**，属**未核实**，正文不宜当作既定名称引用。

另一个常被误判的点是 `android_dlopen_ext`。它是**公开 NDK API**，头文件 `<android/dlext.h>` 的原文是："Opens the given library. The `__filename` and `__flags` arguments are the same as for `dlopen(3)`, with the Android-specific flags supplied via the `flags` member of `__info`."。真正内部专用的是**命名空间相关的那一个 flag**：`ANDROID_DLEXT_USE_NAMESPACE`，其注释写明 "This flag is for internal use only (since there is no NDK API for namespaces)."。所以「`android_dlopen_ext` 是内部 API」这个说法不成立——内部的是那个 flag。

---

## 7. SELinux 如何把 HAL 变成可授权的对象

HAL 在 SELinux 里不是「一个文件」，而是**一对属性（attribute）+ 三个宏**。AOSP `public/te_macros` 的原文：

> `hal_attribute`: "Add an attribute for hal implementations along with necessary restrictions."
> `hal_server_domain`: "Allow a base set of permissions required for a domain to offer a HAL implementation of the specified type over HwBinder."
> `hal_client_domain`: "Allow a base set of permissions required for a domain to be a client of a HAL of the specified type."

`hal_attribute` 的实际效果是展开出 `hal_$1`、`hal_$1_client`、`hal_$1_server` 三个属性，并写 `expandattribute hal_$1_server false;`——最后这一句很关键：它让 `hal_$1_server` **不被展开**，从而可以被 `neverallow` 精确约束。

这套机制的安全价值在于：**「谁能实现这个 HAL」和「谁能调用这个 HAL」是两个独立可审计的集合**。一个第三方域想要冒充某 HAL，不仅要拿到 binder 调用权，还要能进入 `hal_x_server`，而后者在策略里通常是极小的集合。

由此得到三条审计口径：

1. **查实现方**：`grep -r "hal_server_domain("` 找谁在提供；实现方集合应当是「一个（或少数几个）具体域」，出现 `domain`、`appdomain` 这类宽泛主体即为异常。
2. **查调用方**：`hal_client_domain(` 的主体集合通常是系统域；如果出现 `untrusted_app`，说明该 HAL 有应用直通路径，需要单独评估其参数校验。
3. **查 passthrough 授权**：`hal_client_domain` 里含 in-process 运行所需的权限。这条授权是设计的一部分，但它意味着该客户端域可能加载 vendor 库（见 §3）。

---

## 8. 成因分析：这一层为什么反复出问题

把公开案例与接口形态放在一起看，HAL/Treble 边界的缺陷可以归为五类**结构性**成因，而不是零散的人为疏忽。

### 8.1 类型契约与语义契约的落差

AIDL/HIDL 生成代码保证的是**类型安全**（字段类型、数组边界之外的部分），不保证**语义安全**。典型落差：

- 传进来一个 `int` 表示「长度」，生成代码只保证它是 int；是否为负、是否超过实现侧缓冲区，由实现自己决定。
- 传进来一个 `handle`/`fd`，生成代码保证 fd 有效，不保证它指向实现期望的资源类型，也不保证调用方有权使用它。
- 传进来一个 `union`/`parcelable`，生成代码保证标签合法，不保证「标签与载荷组合」的语义合法。

**这一类是所有 HAL 缺陷里占比最高的。** 防御上的对应做法也是结构性的：**实现侧必须在入口做语义校验，不能默认「客户端已经检查过」**。因为客户端与被调用方通常不是同一个安全域（甚至跨分区），任何「我们对端会检查」的假设都等于把校验责任丢给了不可信方。

### 8.2 passthrough 的地址空间合并

已在 §3 说明。要补的一点是**影响评估的口径**：passthrough 模式下 vendor 库的缺陷，其 CVSS 的「攻击向量」与「权限要求」应当按**宿主进程**算，而不是按库自身算。很多分析把 passthrough 缺陷按「需要 vendor 权限」定级，这是低估。

### 8.3 版本冻结带来的「兼容性包袱」

HAL 接口一旦被某个 compatibility matrix 要求，就不能随意增删方法。于是修复常常只能这样做：**不改接口，只在实现里加校验**。这本身是正确做法，但它有一个副作用——**同一接口的多代实现长期并存**，老实现留在老设备上，新实现只在新设备上；「已修复」在公告里指的往往只是最新分支。

自查时的对应动作：不要只问「这个 CVE 修了吗」，要问「这台设备的 vendor 分区里的那个 HAL 实现，属于哪个版本、由谁编译、何时编译」。

### 8.4 校验责任的三方分摊

前面提过：类型在生成代码，权限在 SELinux，存在性在 VINTF。三方分摊的后果是**任何一方都容易假设另外两方已经做了**。真实案例里常见的组合是：

- SELinux 允许了某域调用某 HAL（因为功能需要），但实现侧没有对调用方做 UID/包名级校验，于是**同一 SELinux 域内的任意进程**都能调用。
- VINTF 声明了某 HAL 可用，但实现侧对「未声明的客户端」没有额外检查，于是 VINTF 从「兼容性声明」被误当成「访问控制」。

这条是本文最想强调的一条：**VINTF 不是权限系统**。它回答的是「设备应当提供什么」，不是「谁可以调用」。把前者当后者，是这一层最典型的认知错误。

### 8.5 闭源与二进制交付的可审计性缺口

vendor 侧大量组件以二进制交付，源码不可见。这不是「漏洞」，但它是**漏洞长期存在**的结构原因：静态审计做不了，模糊测试需要真实设备，补丁验证依赖厂商公告粒度。对防御方的现实含义是：**把 vendor 侧的暴露面在架构上收窄**（尽量 binderized、尽量少 passthrough、尽量把校验放在有源码的一侧），比指望逐个修 vendor 缺陷更有效。

---

## 9. 公开案例台账（逐条核实归属）

这一节只列**已核实**组件归属的公开案例。之所以要单独做这张表，是因为 HAL/Treble 主题下 CVE 张冠李戴极其严重——很多被引用为「HAL 漏洞」的编号，实际组件完全无关。

| 编号 | 实际组件（公告/记录原文要点） | 与 HAL 的关系 |
|---|---|---|
| CVE-2020-0041 | "In binder_transaction of binder.c … possible out of bounds write due to an incorrect bounds check"（Android 内核） | **无关**：内核 Binder 驱动，不是 HAL |
| CVE-2021-0703 | "In SecondStageMain of init.cpp … use after free due to incorrect shared_ptr usage"（init） | **无关**：init 第二阶段 |
| CVE-2021-0704 | "In createNoCredentialsPermissionNotification and related functions of AccountManagerService.java …"（框架） | **无关**：账户服务 |
| CVE-2019-2188 | "In the Easel driver … memory corruption due to race conditions"（Pixel 内核驱动） | **无关**：Pixel 专用协处理器驱动 |
| CVE-2020-11174 | "Array index underflow issue in adsp driver due to improper check of channel id before used as array index"（Qualcomm DSP 驱动） | **相邻**：厂商驱动，非 HAL 接口 |
| CVE-2024-43093 | "In shouldHideDocument of ExternalStorageProvider.java … bypass of a file path filter … due to incorrect unicode normalization"（Documents UI，且有在野利用迹象） | **无关**：存储/文档 UI |
| CVE-2024-0044 | "In createSessionInternal of PackageInstallerService.java … run-as any app due to improper input validation" | **无关**：包安装 |
| CVE-2021-0511 | "In Dex2oat of dex2oat.cc … inject bytecode into an app due to improper input validation"（Android runtime） | **无关**：ART 编译器 |

**这张表的教学价值在于它的「无关」列。** 八个编号里七个与 HAL 无关，而它们在二手资料里被反复当作「Treble 边界漏洞」的例证。反过来，这也提示了一条方法论：**在 HAL 这一层，公共 CVE 名录的覆盖度远低于其他层**——厂商驱动缺陷多由高通/联发科/三星自己的公告发布，框架层缺陷进 Android Security Bulletin，而「HAL 接口本身的语义校验缺失」往往不产生独立 CVE，只在版本更新说明里以一句话带过。

因此本章**不给出「HAL 专属 CVE 清单」**，因为经核验不存在这样一份可靠的清单。取而代之的是上面这张「归属纠正表」，以及 §8 的五类成因——成因比编号更有预测力。

---

## 10. 防护技术：把边界做成「默认拒绝」

按「设计 → 实现 → 验证」三层给可操作项。

### 10.1 设计层

| 措施 | 具体做法 | 挡住什么 |
|---|---|---|
| 默认 binderized | 新 HAL 一律独立进程，passthrough 仅用于性能确实需要的 SP-HAL | 地址空间合并导致的半径扩大 |
| 接口窄化 | 一个接口只做一件事；避免「万能 setXxx(int, byte[])」式接口 | 语义校验漏项的表面 |
| 显式稳定性标注 | 只有确实需要跨分区的接口才加 `@VintfStability` | 无意暴露内部接口 |
| 客户端身份入参 | 需要区分的场景，显式传 `uid`/包名并在实现侧校验，而不是依赖「能调到就说明有权」 | SELinux 域内越权 |
| 不用 VINTF 做鉴权 | 把 VINTF 严格限定在兼容性判断 | 把声明当访问控制的系统性误解 |

### 10.2 实现层

1. **入口全量校验**：长度/索引/枚举/句柄/嵌套深度五类，逐个显式检查，不依赖生成代码。
2. **整数运算前移**：所有 `a * b`、`a + b` 在用于分配或索引前，先做溢出判定；`size_t` 与 `int` 混用处显式转换。
3. **fd 与 handle 的类型校验**：拿到一个 fd，先确认它确实是期望的类型（如用 `fstat` 判断是否为期望的文件类型/大小），再做 `mmap`/`read`。
4. **失败即关闭**：校验失败时释放已获取的全部资源并返回明确错误码，不留半初始化状态——半初始化状态是 UAF 的温床。
5. **不信任客户端缓存**：客户端传来的「上次的句柄/序号/偏移」一律当作不可信输入重新校验。

### 10.3 验证层

- **策略侧**：`hal_server_domain` / `hal_client_domain` 的主体集合做定期 diff，任何集合扩张都要有变更理由。
- **接口侧**：用 `vintf`（设备侧）与 `lshal` + `dumpsys`（HIDL/AIDL 两侧）建立基线，把「设备上实际存在的 HAL 版本」与「manifest 声明」对齐；差异是移植残留或攻击痕迹的常见来源。
- **构建侧**：`assemble_vintf` 在构建期校验 manifest/matrix 合法性——把它放进 CI，避免「声明错误到运行时才发现」。
- **代码侧**：对 vendor 交付的 HAL 实现，至少要求提供「入口校验说明 + 模糊测试报告」；无法提供的，按 §8.5 的结论处理——在架构上收窄而非逐点修补。

---

## 11. 自查与观测清单

下面每一条都是**在设备上可直接执行**的，输出可作为基线快照比对。

```bash
# 1. 这台设备声明了哪些 HAL（vendor 侧）
adb shell cat /vendor/etc/vintf/manifest.xml

# 2. 系统要求哪些 HAL
adb shell cat /vendor/etc/vintf/compatibility_matrix.xml
adb shell cat /etc/vintf/compatibility_matrix.device.xml

# 3. HIDL HAL 视图（注意：看不到 AIDL）
adb shell lshal

# 4. AIDL 服务视图（补上 lshal 的盲区）
adb shell dumpsys -l | grep -i hal
adb shell service list | grep -i hardware

# 5. 服务注册中心里的条目
adb shell service list

# 6. VNDK 相关属性（Android 15 起可能不存在，缺失不算异常）
adb shell getprop ro.vndk.version
adb shell getprop ro.product.vndk.version

# 7. 应用可见的 public native 库白名单
adb shell cat /vendor/etc/public.libraries.txt

# 8. 检查是否存在 passthrough 形态的 HAL（启发式：manifest 中 format 字段）
adb shell grep -o 'format="[a-z]*"' /vendor/etc/vintf/manifest.xml | sort | uniq -c
```

判读要点：

- **第 3 与第 4 条必须一起看**。只跑 `lshal` 会漏掉全部 AIDL HAL，这是最常见的误判来源。
- **第 6 条在 Android 15+ 上为空是正常的**（VNDK 已弃用、属性已移除），不要当作设备异常。
- **第 8 条是启发式**：`format="aidl"` / `format="hidl"` 是文档化的取值；**manifest 里没有 `passthrough` 属性**（见 §12 第 4 条），所以不要用「grep passthrough」来判断传输形态——那永远查不到。
- 快照的价值在于**比对**：把同一机型跨系统更新的两次输出 diff，能直接看出厂商新增/删除了哪些 HAL 声明。

---

## 12. 常见误传纠正

1. **「`/system/bin/checkvintf` 可以在设备上跑」**——错。`checkvintf` 是 `cc_binary_host`（构建期主机工具）；设备侧二进制叫 `vintf`。
2. **「`lshal` 能列出全部 HAL」**——错。官方明说它列 HIDL HAL，AIDL 要看 `dumpsys`。
3. **「`lshal` 属于 libvintf」**——错，源码在 `frameworks/native/cmds/lshal`。
4. **「VINTF manifest 里有 `passthrough` 属性（`<hal ... passthrough="true">`）」**——未见该 schema 字段。文档化的 `<hal>` 属性是 `format`（`hidl` 为默认、`aidl`、`native`）与 `override`。
5. **「HIDL 在 Android 13 被弃用」**——错。官方口径是 Android 10 弃用、Android 11 引入 AIDL for HALs。「Android 13 起禁止新增 HIDL」未核实。
6. **「AIDL HAL 注册到 hwservicemanager」**——反了。AIDL HAL 走 `servicemanager`（宏 `hal_attribute_service`），HIDL 走 `hwservicemanager`（`*_hwservice_contexts`）。
7. **「AIDL HAL 实例名带 `@1.0`」**——错。HIDL 形如 `android.hardware.foo@1.0::IFoo/default`；AIDL 形如 `android.hardware.foo.IFoo/default`，版本写在 manifest 的 `<version>`。
8. **「`libhwbinder` 是 AIDL 的 NDK 后端」**——错。AIDL NDK 后端是 `libbinder_ndk`；`libhwbinder` 是 HIDL 的 hwbinder 传输库。
9. **「`ro.vndk.version` 仍是必备属性」**——不成立。VNDK 自 Android 15 起弃用，该属性在对应分区上已移除。
10. **「VNDK-SP 是给 vendor 进程用的」**——反了。SP = same-process，是允许被**框架进程**加载的子集。
11. **「`android_dlopen_ext` 是内部 API」**——错。它是公开 NDK API；内部专用的是 `ANDROID_DLEXT_USE_NAMESPACE` 这个 flag。
12. **「VINTF 声明能阻止未授权客户端调用 HAL」**——错。VINTF 是兼容性声明，不是访问控制；拦截靠 SELinux 与实现侧的 UID/包名校验。
13. **CVE 张冠李戴**：CVE-2021-0703 是 init、CVE-2021-0704 是 AccountManagerService、CVE-2021-0511 是 dex2oat、CVE-2024-0044 是 PackageInstallerService、CVE-2024-43093 是 Documents UI 路径过滤、CVE-2020-0041 是内核 `binder.c`、CVE-2019-2188 是 Pixel Easel 驱动——**没有一个是 HAL/vendor 接口漏洞**。

---

## 未核实与失效来源

- **A. `classloader-namespace` 字面串。** AOSP `library_namespaces.cpp` 中命名空间名为常量 `kClassloaderNamespaceName`，字面串未在已取文件中出现，正文未采用该名称。
- **B. 「Android 13 起禁止新增 HIDL 接口」。** 官方文档未见此句，仅见「尽可能迁移到 AIDL」的表述，按未核实处理。
- **C. `getStub()` 的确切语义与「未在 VINTF 声明」时的运行时错误文本。** 官方文档只说明 `getService` 在传输不可用时返回 `nullptr`，未给出 `getStub` 的逐字定义与错误字符串，正文不引用任何「标准报错原文」。
- **D. `libhwbinder` 的官方定义句。** 仅核实其为 AOSP 独立模块（`vendor_available: true`），未见官方文档对其职责的成文描述。
- **E. 「因地址空间污染而限制 passthrough」的官方表述。** 未见整句；正文只引用「SP-HAL 库被加载进框架进程」与 sepolicy 的 in-process 授权注释这两处一手事实，推论部分已明确标注为推论。
- **F. HAL 专属 CVE 清单。** 经核验不存在可靠的「HAL 接口漏洞」公共清单；本文以归属纠正表替代，并说明原因。
- **G. `/sdcard` 符号链接、`ro.vndk.version` 的完整移除版本范围。** 未逐版本核验，正文只写「Android 15 起对以 15 构建的分区」。

---

## 参考

- VINTF：[source.android.com/docs/core/architecture/vintf](https://source.android.com/docs/core/architecture/vintf)
- VINTF objects：[source.android.com/docs/core/architecture/vintf/objects](https://source.android.com/docs/core/architecture/vintf/objects)
- HIDL（弃用声明）：[source.android.com/docs/core/architecture/hidl](https://source.android.com/docs/core/architecture/hidl)
- AIDL for HALs：[source.android.com/docs/core/architecture/aidl/aidl-hals](https://source.android.com/docs/core/architecture/aidl/aidl-hals)
- VNDK（弃用）：[source.android.com/docs/core/architecture/vndk](https://source.android.com/docs/core/architecture/vndk)
- Linker namespace：[source.android.com/docs/core/architecture/vndk/linker-namespace](https://source.android.com/docs/core/architecture/vndk/linker-namespace)
- SELinux 兼容性（hwservicemanager 强制句）：[source.android.com/docs/security/features/selinux/compatibility](https://source.android.com/docs/security/features/selinux/compatibility)
- `te_macros`：[android.googlesource.com/platform/system/sepolicy/+/refs/heads/main/public/te_macros](https://android.googlesource.com/platform/system/sepolicy/+/refs/heads/main/public/te_macros)
- `libvintf` Android.bp（checkvintf 为 host 工具）：[android.googlesource.com/platform/system/libvintf/+/refs/heads/main/Android.bp](https://android.googlesource.com/platform/system/libvintf/+/refs/heads/main/Android.bp)
- `lshal` 源码：[android.googlesource.com/platform/frameworks/native/+/refs/heads/main/cmds/lshal/Lshal.cpp](https://android.googlesource.com/platform/frameworks/native/+/refs/heads/main/cmds/lshal/Lshal.cpp)
- `dlext.h`：[android.googlesource.com/platform/bionic/+/refs/heads/main/libc/include/android/dlext.h](https://android.googlesource.com/platform/bionic/+/refs/heads/main/libc/include/android/dlext.h)
- 公告：[source.android.com/docs/security/bulletin](https://source.android.com/docs/security/bulletin)

---

## 联系方式

如果你在企业里做终端安全基线或 ROM 评审，需要就「HAL 暴露面清单」「vendor 侧补丁状态核对」「passthrough 模式的风险收窄」做一次针对现网机型的核查，可以通过 Telegram 联系：<https://t.me/one00190>

---

*本手册仅用于防御与教育目的，讨论的均为公开且已被修复问题的机制与设计缺陷，不包含任何可直接利用的代码。文中标注「未核实」的内容不应作为事实引用；发现错误欢迎指出。*
