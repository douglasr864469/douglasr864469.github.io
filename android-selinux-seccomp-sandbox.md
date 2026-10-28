---
layout: page
title: "Android SELinux 策略与 seccomp 沙箱手册"
description: "SELinux 与 seccomp 两条强制轴的分工（对象访问矩阵 vs 系统调用集合）；Android 5.0/6.0/8.0/9.0 四步强化的官方逐字口径；`system/sepolicy` 的 public/private/vendor/reqd_mask/prebuilts 真实布局与「pub..."
date: 2026-10-28
permalink: /android-selinux-seccomp-sandbox/
---

> SELinux 与 seccomp 经常被并排写成"Android 的两道沙箱"，但它们管的根本不是同一个问题：**SELinux 决定"这个域能不能动那个对象"，seccomp 决定"这个进程能不能叫某个内核入口"**。一个是强制访问控制矩阵，一个是系统调用集合过滤器。把两者混为一谈，会导致读公告、读源码、做自查时用错坐标系。本手册把两条轴拆开，并用一份可逐行复核的 `system/sepolicy` 检出处处给出实名与行号：域名怎么分配、`neverallow` 到底禁止了什么、`plat_sepolicy.cil` 是怎么拼出来的、厂商为什么不能随便放宽策略。定位是防御与教育，只引用公开且已修复的信息，不含任何绕过代码。

---

## 0. 使用方式与边界

**本手册的证据基础。** 除官方文档外，本手册大量结论来自一份 AOSP `system/sepolicy` 的本地检出（HEAD 提交号 `4571ddd`，`Merge "Fix duplicated comments" into main`）。凡是写成 `private/app.te:562` 这类形式的引用，都是**该份检出里可以直接翻到的行**，读者克隆同一分支即可复核。需要特别说明：这份检出的目录 `mac_permissions/` 只剩 `Android.bp`，真正的 `mac_permissions.xml` 在 `private/` 与 `reqd_mask/` 下，两处内容并不相同（见 §4）。

**与相邻手册的分工。**

- [多用户、工作资料与进程隔离手册](/android-multiuser-sandbox-isolation/) 只给了 SELinux 与 seccomp 的**现行官方措辞**和 app 域归属结论，并把机制细节留给本手册。
- [系统底层漏洞类型图谱](/android-low-level-vulnerability-types/) 按组件定位缺陷坐标，其中 Binder 三域、ashmem/memfd、ION/dma-buf 都只在"缺陷落在哪"的层面出现；本手册负责**这些对象在策略里叫什么类型、谁被成文禁止访问**。
- [通用漏洞类型手册](/android-common-system-vulnerabilities/) 讲缺陷类型清单；[安全性评估与定级手册](/android-vulnerability-severity-assessment/) 讲定级口径。策略缺陷（错放的 allow、被删掉的 neverallow）在公告里如何定级，去看后者，本手册不重复。
- 内核侧的缓解机制（CFI、PAC/MTE、GKI 与供应商模块）不属于本手册，见 §13 的分工表。

**能回答的问题。**

| 你想知道 | 看哪节 |
| --- | --- |
| 一个第三方 APK 到底被标成哪个域 | §4、§5 |
| 为什么 `targetSdkVersion` 会改变 SELinux 域名 | §5 |
| 平台禁止厂商放宽哪些权限、写在哪个文件 | §7 |
| isolated / Instant / SDK sandbox 的应用有什么不成文差别 | §5、§7 |
| seccomp 过滤器在源码哪个位置、被禁的系统调用返回什么 | §8 |
| 为什么设备上有 `/system/etc/selinux/mapping` 这种目录 | §9 |
| 改策略会撞上哪些 CI/CTS/VTS 检查 | §10 |

---

## 1. 两条轴：SELinux 看对象，seccomp 看入口

Android 官方对应用沙箱的成文描述里没有把这两者混在一起的说法。可直接引用的原句（来自 AOSP Application Sandbox 页，逐字）：

- "Android uses the UID to set up a kernel-level Application Sandbox."
- "The kernel enforces security between apps and the system at the process level through standard Linux facilities such as user and group IDs that are assigned to apps."
- "Because the Application Sandbox is in the kernel, this security model extends to both native code and OS apps."
- "Generally, to break out of the Application Sandbox in a properly configured device, one must compromise the security of the Linux kernel."

注意第一条与第二条：最老的那道隔离根本不是 SELinux，而是**传统的 UID/GID 自主访问控制（DAC）**。SELinux 是在 DAC 之上叠了一层强制访问控制（MAC），seccomp 则是在"能不能发起某个系统调用"这个更靠前的位置做裁剪。三者的层次是：

| 层次 | 机制 | 判据 | 拒绝时的表现 |
| --- | --- | --- | --- |
| 系统调用入口 | seccomp-bpf（BPF 程序，运行在内核 `prctl`/`seccomp(2)` 装的过滤器上） | 系统调用号 + 架构 | 由过滤器返回值决定，AOSP 的 app 过滤器使用 `SECCOMP_RET_TRAP`（见 §8） |
| 对象访问 | SELinux MAC | 源域 × 目标类型 × 对象类 × 权限位 | 审计记录 + 系统调用返回错误 |
| 属主检查 | Linux DAC（UID/GID、文件模式位） | 进程 UID 与 inode 属主 | `EACCES` 一类错误 |

这三层是**乘法关系不是加法**：一个能被 DAC 允许的操作，仍要先过 seccomp 再过 SELinux；反过来，SELinux 再严格，也管不住一个被允许的 syscall 内部的行为。这就是为什么 §7 里那些 `neverallow` 与 §8 的 syscall 过滤要一起看。

关于 ptrace，策略文件里有一段把这套关系说得很清楚的成文注释（`private/app.te`，逐字）：

> "The Android security model guarantees the confidentiality and integrity of application data and execution state. Ptrace bypasses those confidentiality guarantees. Disallow ptrace access from system components to apps. Crash_dump is excluded, as it needs ptrace access to produce stack traces. llkd is excluded, as it needs ptrace access to inspect stack traces for live lock conditions."

这段值得停下来读三遍：官方明确承认 **ptrace 会绕过 Android 安全模型的机密性与完整性保证**，所以不但禁止 app 跟踪非 app 域，还**反过来禁止系统组件跟踪 app**，只给 `crash_dump`（崩溃栈采集）和 `llkd`（活锁检测，且仅在 userdebug/eng 构建里，见 §7 的 `userdebug_or_eng()` 包裹）开了明确的例外。

---

## 2. 四步演进：Android 5.0 / 6.0 / 8.0 / 9.0

AOSP Application Sandbox 页用四个版本节点描述了沙箱的强化过程。逐字引用（这一节全篇都可以逐字回查）：

- "**In Android 5.0, SELinux provided mandatory access control (MAC) separation between the system and apps.**"
- "**However, all third-party apps ran within the same SELinux context so inter-app isolation was primarily enforced by UID DAC.**"
- "**In Android 6.0, the SELinux sandbox was extended to isolate apps across the per-physical-user boundary.**"
- "**In Android 8.0, all apps were set to run with a `seccomp-bpf` filter that limited the syscalls that apps were allowed to use, thus strengthening the app/kernel boundary.**"
- "**In Android 9 all nonprivileged apps with `targetSdkVersion >= 28` must run in individual SELinux sandboxes, providing MAC on a per-app basis.**"

这四步的解释力比很多人以为的大，因为它能纠正两类常见误读：

1. **"SELinux 从 Android 很早就在保护 app 之间的隔离"是错的。** 官方自己说了：5.0 阶段所有第三方 app 共享同一个 SELinux 上下文，**app 之间的隔离主要靠 UID DAC**；SELinux 提供的是"系统 vs app"这一侧的 MAC 分离。跨用户（per-physical-user）的隔离要到 6.0，按 app 粒度的 MAC 要到 9.0，而且带 `targetSdkVersion >= 28` 这个条件。
2. **`targetSdkVersion` 不只是兼容性开关，它会改变强制访问控制的粒度。** 这一点在策略源码里是硬的：§5 会看到 `seapp_contexts` 用 `minTargetSdkVersion` 把应用分流到不同域名，`private/untrusted_app.te` 的注释逐档说明了每个类型对应哪个 targetSdk 区间。换句话说，一个把 `targetSdkVersion` 声明为 27 的老应用，和一个声明为 34 的新应用，**在 SELinux 里是两个不同的域**。

顺带一句 `private/app.te` 里那句被广泛引用的注释，与 §2 的第 2 条同源：官方在多处强调 `targetSdkVersion` 影响的是"你落在哪个强制域里"，这与"运行时权限授予了什么"（见 [通用漏洞类型手册](/android-common-system-vulnerabilities/) 的权限类缺陷）是两套完全独立的机制。

---

## 3. `system/sepolicy` 的真实布局，以及一条成文的目录纪律

先给顶层结构（本检出实测）：`apex/`、`build/`、`compat/`、`contexts/`、`flagging/`、`mac_permissions/`、`microdroid/`、`prebuilts/`、`private/`、`public/`、`reqd_mask/`、`tests/`、`tools/`、`treble_sepolicy_tests_for_release/`、`vendor/`，加上根目录的 `Android.bp`、`README.md`、`README.apps.md`、`TEST_MAPPING`、`OWNERS`、`CLEAN_SPEC`（`CleanSpec.mk`）与 `METADATA`。规模上，`public/` 有 148 个文件、`private/` 有 356 个文件。

**关键的一条目录纪律，就写在 `public/untrusted_app.te` 里（逐字）：**

> "# system/sepolicy/public is for vendor-facing type and attribute definitions.
> # DO NOT ADD allow, neverallow, or dontaudit statements here.
> # Instead, add such policy rules to system/sepolicy/private/*.te."

这段话信息量很大：`public/` 只负责**声明**类型和 attribute（因为 vendor 策略需要引用这些名字），而所有实际的授权规则——`allow`、`neverallow`、`dontaudit`——必须写在 `private/`。这就是 Treble 时代"平台策略 / 厂商策略"分离的落地形式之一：厂商能看到的是一份稳定的名字表，看不到也不需要平台的规则细节。

其余各目录的作用，只写本检出能直接看出来的：

| 目录 | 能确认的内容 |
| --- | --- |
| `private/` | 实际的授权规则与各 contexts 文件。`file_contexts`、`seapp_contexts`、`property_contexts`、`service_contexts`、`hwservice_contexts`、`genfs_contexts`、`port_contexts`、`initial_sid_contexts`、`tee_service_contexts`、`keystore2_key_contexts` 都在这里，另有 `file_contexts_asan` 与 `file_contexts_overlayfs` 两个变体 |
| `public/` | 类型与 attribute 声明、`te_macros` 宏定义 |
| `vendor/` | 厂商侧策略（本检出含在内） |
| `prebuilts/api/` | 按**发布格式版本**冻结的历史快照：本检出含 `29.0`–`34.0` 与 `202404`、`202504` 六个目录 |
| `reqd_mask/` | "必需策略掩码"：`access_vectors`、`security_classes`、`initial_sids`、`initial_sid_contexts`、`roles`、`mls`、`mls_decl`、`mls_macros`、`keys.conf`、`mac_permissions.xml`、`property_contexts`、`seapp_contexts`、`service_contexts`、`reqd_mask.te` |
| `apex/` | **每个 APEX 模块一份 file_contexts**，命名直接就是模块名，例如 `com.android.adbd-file_contexts`、`com.android.art-file_contexts`、`com.android.conscrypt-file_contexts`、`com.android.bt-file_contexts`、`com.android.btservices-file_contexts`、`com.android.crashrecovery-file_contexts`、`com.android.cellbroadcast-file_contexts`、`com.android.compos-file_contexts`、`com.android.devicelock-file_contexts`、`com.android.biometrics.virtual.fingerprint-file_contexts`、`apex.test-file_contexts` |
| `microdroid/` | AVF 的 Microdroid 客户机侧策略（含自己的一份 `private/mac_permissions.xml`） |
| `tests/` | 策略检查脚本与测试，见 §10 |
| `compat/` | 跨版本兼容策略（与 §9 的 `.compat.cil` 对应） |
| `flagging/` | 只含 `Android.bp` 与 `flagging_macros`（策略标记宏，用途未逐字核实） |

`apex/` 那张表值得单独说一句：它把 [Mainline 与 APEX 模块手册](/android-mainline-apex-modules/) 里的"模块"概念和一个具体的安全事实接上了——**可更新模块的标签上下文也是随模块分文件维护的**，也就是说一个 APEX 更新带来的行为面变化，在 SELinux 侧有对应的一份 contexts 文件。

---

## 4. 域名从哪来：签名 → seinfo → seapp_contexts 的三级映射

`public/untrusted_app.te` 开头那段注释是这套映射最简洁的官方说明（逐字，节选）：

> "# Apps are labeled based on mac_permissions.xml (maps signer and optionally package name to seinfo value) and seapp_contexts (maps UID and optionally seinfo value to domain for process and type for data directory). The untrusted_app domain is the default assignment in seapp_contexts for any app with UID between APP_AID (10000) and AID_ISOLATED_START (99000) if the app has no specific seinfo value as determined from mac_permissions.xml. In current AOSP, this domain is assigned to all non-system apps as well as to any system apps that are not signed by the platform key."

把这段拆成可操作的三步：

1. **签名 → seinfo**：`mac_permissions.xml` 把"用哪个密钥签的名"映射成一个 `seinfo` 标签。本检出的 `private/mac_permissions.xml` 头部把规则成文了（逐字，节选）："A signer tag must contain a seinfo tag XOR multiple package stanzas"、"The 'default' seinfo label is automatically applied"，并给出结构示例：`<signer signature="@PLATFORM" ><seinfo value="platform" /></signer>`，以及"多个证书必须全部匹配"的形式（`// multiple certs protecting seinfo (all contained certs must match)`）。可以看到成文的 seinfo 值包括 `platform` 与 `sdk_sandbox`（对应 `@PLATFORM`、`@SDK_SANDBOX` 两个密钥名）。
2. **(UID, seinfo, 包名, 若干布尔量) → 域与数据目录类型**：查 `private/seapp_contexts`。
3. **级别（MLS/MCS 的 sensitivity + categories）**：由同一个文件里的 `levelFrom` 决定。

`private/seapp_contexts` 的头部注释逐字列出了全部输入选择器，这份清单本身就是很好的自查表（逐字）：`isSystemServer`、`isEphemeralApp`、`user`、`seinfo`、`name`、`isPrivApp`、`minTargetSdkVersion`、`fromRunAs`、`isIsolatedComputeApp`、`isSdkSandboxNext`、`isSdkSandboxAudit`。并成文了四条匹配语义：

- "All specified input selectors in an entry must match (i.e. logical AND)."
- "An unspecified string or boolean selector with no default will match any value."
- "A user, or name string selector that ends in * will perform a prefix match." / "String matching is case-insensitive."
- "See external/selinux/libselinux/src/android/android_platform.c, seapp_context_lookup()."（**实现落在 libselinux 里，不在 AOSP framework 代码里**——这一点对读源码找 bug 很关键）

另外两条容易被忽略但很硬的成文约束：`isPrivApp=true` 只匹配预装在 `/{system,vendor,product}/priv-app` 的应用；`seinfo` 里的冒号字符被保留、不得使用（"The ':' character is reserved and may not be used in seinfo."）。

**levelFrom 的四种取值（逐字）：**

- `levelFrom (string; one of none, app, or user)` —— 注释里还列出了 `all`
- "levelFrom=none omits the level."
- "levelFrom=app determines the level from the process UID."
- "levelFrom=user determines the level from the user ID."
- "levelFrom=all determines the level from both UID and user ID."
- "levelFrom=user is only supported for _app or _isolated UIDs."
- "levelFrom=app or levelFrom=all is only supported for _app UIDs."

最后这两行是很多人写自定义策略时踩的坑：**`levelFrom` 的可用取值受 UID 类别约束**，不是想写哪个就写哪个。

> 一处诚实标注：这份检出的 `mac_permissions/` 目录只剩 `Android.bp`；`reqd_mask/mac_permissions.xml` 的内容是一个空壳（只有 `<?xml …?><policy></policy>`，61 字节）。所以本手册的 seinfo 取值只列 `private/mac_permissions.xml` 里能逐字看到的，其余一律不补。

---

## 5. appdomain 家族：九类应用域与它们的分流条件

`appdomain` 是所有 app 域共享的 attribute，声明位置是 `public/attributes:204`（`attribute appdomain;`）。AOSP 在 `README.apps.md`（这份仓库自带的 app 域总览文档）里给了逐字定义：

> "In general, all apps will have the `appdomain` attribute. You can think of `appdomain` as any app started by Zygote. The macro `app_domain()` should be used to define a type that is considered an app (see public/te_macros)."

`README.apps.md` 还逐个说明了每个域的归属条件，可以直接当分类表用（下面的表把它的描述与 `seapp_contexts` 的实际规则行对齐；行号是本检出实测）：

| 域 / 类型 | 谁会被标成它 | 依据 |
| --- | --- | --- |
| `untrusted_app` | 第三方 app，且 targetSdk 为当前档 | `seapp_contexts:205`：`user=_app minTargetSdkVersion=34 domain=untrusted_app type=app_data_file levelFrom=all` |
| `untrusted_app_32` / `_30` / `_29` / `_27` / `_25` | 按 targetSdk 降档的第三方 app | `seapp_contexts:206–211`；`public/untrusted_app.te` 注释逐档写明：`_32` 对应 `31 < targetSdkVersion <= 33`，`_30` 对应 `29 < targetSdkVersion <= 31`，`_29` 对应 `targetSdkVersion = 29`，`_27` 对应 `25 < targetSdkVersion <= 28`，`_25` 对应 `targetSdkVersion <= 25` |
| `untrusted_app_all`（attribute） | 上面所有非特例域的和集 | `private/untrusted_app_all.te` 头注释；注意它明确**不含** `ephemeral_app` 与 `isolated_app` |
| `priv_app` | 预装在 priv-app 目录的普通签名应用 | `seapp_contexts` 的 `user=_app isPrivApp=true domain=priv_app type=privapp_data_file levelFrom=user` |
| `platform_app` | 用平台密钥签名、装在 system/vendor 镜像里的 app（README 举例 `com.android.systemui`） | `user=_app seinfo=platform domain=platform_app type=app_data_file levelFrom=user` |
| `system_app` | 平台签名**且**跑在 system UID 上（README 举例 `com.android.settings`） | `user=system seinfo=platform domain=system_app type=system_app_data_file` |
| `ephemeral_app` | 免安装运行（Google Play Instant）的应用，"more constrained than `untrusted_app`" | `user=_app isEphemeralApp=true domain=ephemeral_app type=app_data_file levelFrom=all` |
| `isolated_app` / `isolated_compute_app`（合称 `isolated_app_all`） | `android:isolatedProcess` 的隔离服务进程 | `seapp_contexts:182–183`，两条都带 `levelFrom=user` |
| `sdk_sandbox_34` / `sdk_sandbox_next` / `sdk_sandbox_audit` | Privacy Sandbox 的 SDK runtime 应用 | `seapp_contexts:184–186`，三条都是 `user=_sdksandbox`、`type=sdk_sandbox_data_file levelFrom=all` |
| `app_zygote` / `webview_zygote` | 应用的 Zygote 派生进程 / WebView 的独立 zygote | `user=_app seinfo=app_zygote domain=app_zygote levelFrom=user`；`user=webview_zygote seinfo=webview_zygote domain=webview_zygote` |
| 按包名单独点名 | 少数包被单独分配域 | `com.android.traceur` → `traceur_app`（`:169`）、`com.android.DeviceAsWebcam` → `device_as_webcam`（`:171`）、permissioncontroller → `permissioncontroller_app`、`com.android.providers.media.module` → `mediaprovider_app` 等 |
| 按 UID 名的系统身份 | radio / bluetooth / nfc / network_stack / secure_element / shell | `user=bluetooth seinfo=bluetooth domain=bluetooth`、`user=nfc … domain=nfc`、`user=network_stack … type=radio_data_file`、`user=radio seinfo=platform domain=radio`、`user=shell seinfo=platform domain=shell name=com.android.shell` |

这一节里最值得记住的三件事：

1. **`untrusted_app` 这个名字本身已经不带"所有第三方应用"的含义。** 它只属于 targetSdk 达到当前档的应用；旧档应用落在 `untrusted_app_25/27/29/30/32`。`public/untrusted_app.te` 的注释还解释了为什么档数不是连续的——"Not all targetSdkVersion have a specific type, some version are skipped when no differences were introduced"（不是每个 targetSdk 都有独立类型，没有引入差异的版本被跳过）。
2. **`system_app` 与 `platform_app` 的区分依据不是"是不是系统应用"，而是 UID 与签名的组合**：`user=system seinfo=platform` 才是 `system_app`；平台签名但跑普通 app UID 的是 `platform_app`。这两个域在排查"某个系统组件为什么能/不能读某个文件"时经常被搞混。
3. **`mediaprovider`（`seinfo=media`）的数据类型是 `app_data_file`，而 `mediaprovider_app`（priv-app + `seinfo=media`）是 `privapp_data_file`**：同一个"媒体提供方"角色，因为安装位置不同而落在不同的数据类型上。

## 6. file_contexts：对象标签的落点，以及三个 binder 域

`private/file_contexts` 把路径映射成类型。本节只列本检出逐字可见、且对读安全公告最有用的几行（行号实测）：

```
101: /dev/ashmem           u:object_r:ashmem_device:s0
102: /dev/ashmem(.*)?      u:object_r:ashmem_libcutils_device:s0
104: /dev/binder           u:object_r:binder_device:s0
134: /dev/hwbinder         u:object_r:hwbinder_device:s0
222: /dev/vndbinder        u:object_r:vndbinder_device:s0
651: /data/user            u:object_r:system_userdir_file:s0
652: /data/user_de         u:object_r:system_userdir_file:s0
```

三个 binder 节点对应三个类型，这就是 [底层漏洞类型图谱](/android-low-level-vulnerability-types/) 里"Binder 三域"在 SELinux 侧的实名：`binder_device`（framework 侧）、`hwbinder_device`（HAL 侧）、`vndbinder_device`（vendor 内部）。**跨这三个节点的访问在策略里是三种不同的对象**，所以一条"某服务可以访问 binder"的授权并不自动意味着它能访问 hwbinder。这是读 sepolicy 类补丁时最容易漏掉的一点。

`/dev/ashmem` 那两行也值得注意：**同一个前缀被分成了两个类型**——精确匹配 `/dev/ashmem` 得 `ashmem_device`，带后缀（正则 `/dev/ashmem(.*)?`）得 `ashmem_libcutils_device`。这与 ashmem → memfd 的迁移史有关，读旧设备策略时不要把两者当成同一个东西（迁移本身的状态见底层类型图谱手册）。

关于"标签从哪来"，还需要补一个容易踩的认知坑：**app 进程的上下文不是 `file_contexts` 给的**。`file_contexts` 管的是**对象**（文件、节点、socket、属性等）的标签，而 app 进程的域来自 §4 的 `seapp_contexts` 三级映射。两者都是"上下文映射文件"，但输入完全不同——一个是路径，一个是 (UID, seinfo, 包名, targetSdk)。把它们并列写在同一节里讲是常见的科普错误。

> 一处诚实标注：`/data/data` 这一条我在本检出的 `private/file_contexts` 里**没有**以独立行找到（grep 只命中 `/data/user`、`/data/user_de` 与若干 `app_data_file` 相关条目）。app 私有目录的类型来自 `seapp_contexts` 的 `type=app_data_file` 输出。具体某个版本的 `file_contexts` 是否含 `/data/data` 行，请以你手上的检出为准，本手册不替它背书。

---

## 7. neverallow：平台用哪些成文规则把"放宽"变成构建失败

`neverallow` 是 SELinux 策略里的"负向断言"：它不授权任何东西，而是声明**任何允许这类访问的策略都非法**。本检出里 `public/`、`private/`、`vendor/` 三处以 `neverallow` 开头的行合计 **988 条**——这就是"厂商随便加一条 allow 就能过编译"这种说法的现实对照。

下面这些是直接与 app 沙箱相关的、可逐行复核的条目（文件:行号 + 逐字规则 + 它到底挡住了什么）：

| 出处 | 规则（逐字） | 挡住的事 |
| --- | --- | --- |
| `private/app.te:562` | `neverallow appdomain { domain -appdomain }:process ptrace;` | 任何 app 跟踪非 app 进程 |
| `private/app.te:571–578` | `neverallow { domain -appdomain -crash_dump userdebug_or_eng(`-llkd') } appdomain:process ptrace;` | 反过来：系统组件跟踪 app（仅 `crash_dump`、以及 userdebug/eng 下的 `llkd` 例外） |
| `private/app.te:580–581` | `neverallow appdomain { domain -appdomain }:file no_w_file_perms;` / `neverallow { appdomain -shell } { domain -appdomain }:file no_rw_file_perms;` | app 读别人的 `/proc/pid/*`；注释直接写了这是"另一种形式的 hidepid=2 保护" |
| `private/app.te:589–590` | `neverallow appdomain { domain -appdomain -perfetto }:process { sigkill sigstop signal };` | app 给非 app 进程发信号（`perfetto` 有明确例外，因为 shell 本身也是 appdomain） |
| `private/app.te:524` | `neverallow { appdomain -bluetooth -network_stack } self:capability_class_set *;` | app 拿到 Linux capability（只有 bluetooth、network_stack 两个域被成文排除） |
| `private/app.te:527` | `neverallow appdomain dev_type:blk_file { read write };` | app 直接读块设备（典型场景：读别的 app 的底层分区数据） |
| `private/app.te:530` | `neverallow { untrusted_app isolated_app shell } graphics_device:chr_file { read write };` | 普通 app 直接操作图形设备节点 |
| `private/app.te:532–536` | `neverallow { appdomain -nfc } nfc_device:chr_file`、`neverallow { appdomain -bluetooth } hci_attach_dev:chr_file`、`neverallow appdomain tee_device:chr_file { read write };` | app 摸 NFC/HCI/TEE 设备节点——**TEE 那一行值得单独记住**：appdomain 直接读写 `tee_device` 被成文禁止，这是 [TEE 与密钥认证手册](/android-tee-key-attestation/) 里"应用只能通过 KeyMint/Gatekeeper 这类受控服务访问安全元件"的强制面依据 |
| `private/app.te:555–559` | `neverallow appdomain socket_device:sock_file write;`、`neverallow appdomain adbd_socket:sock_file write;`、`neverallow { appdomain -radio } rild_socket:sock_file write;` | app 写这些设备侧 Unix socket |
| `private/app.te:593–612` | 四条同构规则：`neverallow appdomain rootfs:dir_file_class_set { create write setattr relabelfrom relabelto append unlink link rename };` 以及针对 `system_file_type`、`exec_type`、`system_data_file` 的同款 | app 写 rootfs、写 `/system`、写自己的入口可执行文件、写 system 持有的 `/data` 区域。`system_data_file` 那条的注释还成文了正确做法："Define a different type for portions that should be writable by apps." |

**角色专属的条目**（这几条最能说明"同为 app，差别很大"）：

| 出处 | 规则（逐字） | 含义 |
| --- | --- | --- |
| `private/isolated_app_all.te:43` | `neverallow isolated_app_all app_data_file_type:file open;` | 隔离进程连"打开应用数据文件"这个动作都被成文禁止——不是权限少一点，是一条根本做不到的事 |
| `private/ephemeral_app.te:59` | `neverallow ephemeral_app app_data_file_type:file execute_no_trans;` | 免安装应用不能就地执行自己数据目录里的文件 |
| `private/priv_app.te:275` / `:278` | `neverallow priv_app { app_data_file_type -privapp_data_file }:file no_x_file_perms;` / `…:lnk_file { open read getattr };` | priv-app 不得执行别人的应用数据文件，也不得跟随这类符号链接（防止借链接越界） |
| `private/app_zygote.te:123` | `neverallow app_zygote app_data_file_type:file { rwx_file_perms };` | app zygote 对应用数据文件不能有读写执行——fork 出的子进程才是干活的那个 |
| `private/dex2oat.te:115` / `private/profman.te:48` | `neverallow dex2oat app_data_file_type:notdevfile_class_set open;` / profman 同款 | 编译器和 profile 管理器不许"打开"应用数据文件（`open` 这个被单列的动作针对 app 数据，值得注意：文件操作权限被拆开约束） |
| `private/netd.te:164` / `private/logpersist.te:44` | `neverallow netd { app_data_file_type system_data_file }:dir_file_class_set write;` / logpersist 同款 | 网络守护进程和日志持久化组件都不能写应用数据 |
| `private/init.te:822` | `neverallow init app_data_file_type:lnk_file read;` | 连 init 也被挡住读 app 数据的符号链接 |

还有一条"允许"性质的规则，恰好说明为什么 app 能在自己沙箱里执行 so 文件（`private/untrusted_app_all.te`，逐字）：

```
allow untrusted_app_all privapp_data_file:file     { r_file_perms execute };
allow untrusted_app_all app_data_file:file         { r_file_perms execute };
auditallow untrusted_app_all app_data_file:file execute;
```

注意第三条是 `auditallow` 而不是 `allow`：**"允许但仍记录审计"**。这类写法在排障时很关键——看到 denial 日志并不必然意味着操作被拒绝了。

同一文件里还有一条把安全与崩溃上报组件的耦合写进了策略，附带真实 bug 单号（逐字）：

```
# Chrome Crashpad uses the the dynamic linker to load native executables
# from an APK (b/112050209, crbug.com/928422)
allow untrusted_app_all system_linker_exec:file execute_no_trans;
```

**`dontaudit` 的规模**：本检出 `private/` 与 `public/` 里以 `dontaudit` 开头的行有 **236 条**。这是"SELinux 拒绝一定会留下 `avc: denied` 日志"这一说法的直接反例（见 §11）。

> 一处诚实标注：我在本检出**没有**找到针对 `permissive` 声明的 neverallow 约束，`private/*.te` 里也未匹配到以 `permissive` 开头的行。因此本手册不断言"平台成文禁止厂商给敏感域加 permissive"——这条说法是否正确，请以你手上的完整策略树与 CI 配置为准。

---

## 8. seccomp-bpf：过滤器在源码里的真实位置

前面 §2 提到成文说法是"Android 8.0 起所有 app 都跑在 `seccomp-bpf` 过滤器下"。实现位置在 **bionic 的 libc 里**，不在内核、也不在 sepolicy 仓库。`bionic/libc/seccomp/` 目录本检出没有，我按目录页核对到的文件名是：`include/`、`gen_syscall_nrs.cpp`、`gen_syscall_nrs_x86.cpp`、`gen_syscall_nrs_x86_64.cpp`、`seccomp_bpfs.h`（注意：这个文件名的拼写就是这样，`bpfs`）、`seccomp_policy.cpp`。

从 `libc/seccomp/seccomp_policy.cpp` 核对到的事实（只列逐字可见的标识符，不做外推）：

- 安装过滤器的函数名包括：`install_filter`、`_set_seccomp_filter`、`set_app_seccomp_filter`、`set_app_zygote_seccomp_filter`、`set_system_seccomp_filter`、`install_setuidgid_seccomp_filter`、`_install_setuidgid_filter`。
- app 的过滤器通过 `_set_seccomp_filter(FilterType::APP)` 选择，内部是 `switch (type)` + `case APP:` 的结构，使用的 BPF 程序变量名是 `primary_app_filter` / `primary_app_filter_size` 与 `secondary_app_filter` / `secondary_app_filter_size`。
- **被禁的系统调用返回 `SECCOMP_RET_TRAP`**：`Disallow` 分支落到 `BPF_STMT(BPF_RET|BPF_K, SECCOMP_RET_TRAP)`。`SECCOMP_RET_TRAP` 的语义是"像系统调用不存在一样，向进程投递 SIGSYS"，因此 app 调用未过滤放行的 syscall 拿到的是信号，不是某个可读的 errno——这也解释了为什么应用层看到的是"莫名崩溃"而不是"错误码"。（策略文件本身没有在此处写明 errno；这一句是 `SECCOMP_RET_TRAP` 的既有语义。）
- 过滤器按**架构成对**配置：ARM 侧是 `AUDIT_ARCH_AARCH64` 配 `AUDIT_ARCH_ARM` 作为 secondary，x86 侧是 `AUDIT_ARCH_X86_64` 配 `AUDIT_ARCH_I386`，**RISC-V 只有 `AUDIT_ARCH_RISCV64`，没有 secondary**。所以"64 位与 32 位 ABI 的策略集不同"在源码层面是显式存在的两套程序（primary/secondary），不是运行时临时推导。
- 除 app 之外还有独立的一套：zygote 变体、system 变体，以及针对 `setresuid`/`setresgid` 的 `_install_setuidgid_filter`（含 `primary_setresuid`/`secondary_setresuid`、`primary_setresgid`/`secondary_setresgid`）。
- 系统调用号来自 `func_to_syscall_nrs.h`（被 include），同目录下的 `gen_syscall_nrs*.cpp` 是生成器。

**对安全研究的实际含义（三条）：**

1. **seccomp 是 libc 侧的安装动作**。`set_app_seccomp_filter()` 这类函数在进程启动路径上被调用；因此"一个 app 通过 `dlopen` 之外的奇怪路径起了个子进程"这类问题，要看它的执行上下文（是否 exec 出 zygote 体系）才知道过滤器是哪一套。这属于 [RCE 与权限提升手册](/android-rce-and-privilege-escalation/) 的"执行上下文"轴。
2. **架构差异会直接改变可利用面**：同一份策略对 arm/arm64、x86/x86_64 是两套 BPF 程序；某些 syscall 在一个 ABI 上被拦、在另一个 ABI 上未必同构。审计时必须逐 ABI 看。
3. **别把"系统调用被 seccomp 拦下"当成"SELinux 也拒绝了它"**。两套机制的日志、返回值和修复责任方完全不同（前者归 libc/zygote 启动路径，后者归 sepolicy）。

> 两处诚实标注：① app 策略里**具体哪些 syscall 编号被 Disallow** 的清单，我没有逐字读 `func_to_syscall_nrs.h` 与生成的表，本手册不列；② `set_app_seccomp_filter()` 在 zygote fork 路径上的确切调用点、以及是否由某个系统属性或环境变量开关，我在策略文件内容里**没有看到**环境属性相关的分支（核对结果明确记为"absent"），因此不写。

---

## 9. 两条版本轴：`SEPOLICY_VERSION` 与发布格式版本

这是本手册最想纠正的一组概念混淆。Android 里"SELinux 策略的版本"至少有**两个互不等价的数字**：

**(a) 内核侧策略格式版本。** 本检出 `Android.bp:44–46` 逐字：

```
cc_defaults {
    name: "selinux_policy_version",
    cflags: ["-DSEPOLICY_VERSION=30"],
}
```

`SEPOLICY_VERSION` 是编进 libselinux 等组件里的宏，对应内核 SELinux 的策略版本概念。

**(b) Android 发布格式版本。** 体现在目录与产物名上：`prebuilts/api/` 下同时存在 `29.0 … 34.0` 与 `202404`、`202504` 两种命名；`Android.bp` 的产物清单里出现 `plat_sepolicy_genfs_202504.cil`、`plat_sepolicy_genfs_202604.cil` 这样的按发布日期命名的文件。这说明**从某个节点起，策略冻结的锚点从"API level"换成了"发布日期（YYYYMM）"**，两套命名并存，不是同一根轴。

`prebuilts/api/` 的作用是**兼容快照**：`Android.bp` 里能看到 `plat_29.0.cil`、`29.0.compat.cil` 直到 `plat_31.0.cil`、`31.0.compat.cil` 这类成对模块。也就是每个历史版本都留了"基础策略 + 兼容补丁"两份产物，用来让老设备/老厂商策略在新平台上仍然可用。

**产物拼接链条**（本节只列 `Android.bp` 里逐字可见的模块名，不推断顺序）：

- `reqd_policy_mask.conf` / `reqd_policy_mask.cil` —— 必需掩码策略，来自 `reqd_mask/`
- `pub_policy.conf` / `pub_policy.cil`、`plat_pub_policy.conf` / `plat_pub_policy.cil` —— 平台公开名字表
- `plat_sepolicy.conf` / `plat_sepolicy.cil` —— 平台完整策略
- `userdebug_plat_sepolicy.cil` / `system_ext_userdebug_plat_sepolicy.cil` —— **userdebug 专用的一份策略**（§7 里 `userdebug_or_eng()` 包裹的例外就是在这里生效的）
- `system_ext_sepolicy.conf` / `system_ext_sepolicy.cil`、`product_sepolicy.conf` / `product_sepolicy.cil` —— system_ext / product 分区各自一份
- `gsi_se_policy_cil` —— GSI（通用系统镜像）用
- `technical_debt.cil`（模块名 `sepolicy_technical_debt`）—— 官方成文的"技术债"策略，专门用来标记暂时必须保留的授权
- 编译工具模块 `secilc`
- 映射文件模块：`plat_mapping_file`（`Android.bp:369–374`，逐字）：

```
se_versioned_policy {
    name: "plat_mapping_file",
    base: ":plat_pub_policy.cil",
    mapping: true,
    version: "current",
    relative_install_path: "mapping", // install to /system/etc/selinux/mapping
}
```

同结构还有 `system_ext_mapping_file` 等。**这段是"设备上为什么有 `/system/etc/selinux/mapping` 目录"的直接答案**：mapping 文件是把厂商策略接到平台公开名字表上的那层桥，`base` 指向 `plat_pub_policy.cil`，按版本产出（`version: "current"`，配合 `prebuilts/api/` 里的历史版本）。

`userdebug_plat_sepolicy.cil` 的存在还有一层安全含义：userdebug/eng 构建与 user 构建的强制策略**不是同一份**。做攻击面自评时拿 userdebug 版本的 `plat_sepolicy` 结论去推断 user 版的行为，会得出偏乐观的结果。

---

## 10. 改策略会撞上什么：仓库自带的检查与测试

`tests/` 目录里的脚本名就是这些检查的实名（逐个可见）：`sepolicy_freeze_test.py`、`treble_sepolicy_tests.py`、`apex_sepolicy_tests.py`、`apex_sepolicy_tests_test.py`、`policy.py`、`policy_test.py`、`mini_parser.py`、`searchpolicy.py`、`check_prop_prefix.py`、`fc_sort.py`、`fc_sort_test.py`、`combine_maps.py`、`sepol_wrap.cpp`、`fix_policies.sh`、`build/`、`include/`。

- `sepolicy_freeze_test.py` 对应"策略冻结"检查——即 `prebuilts/api/` 里的历史快照与当前源码之间不允许出现破坏性改动（具体判定规则请读脚本本身，本手册只给存在性）。
- `treble_sepolicy_tests.py` + 顶层 `treble_sepolicy_tests_for_release/` 目录：Treble 相关的策略检查，按发布版本有一份。
- `apex_sepolicy_tests.py` 与 §3 的 `apex/*.-file_contexts` 是一对：APEX 模块的 contexts 也要过检查。
- `check_prop_prefix.py`：属性名前缀检查（与 `private/property_contexts` 呼应）。
- `searchpolicy.py`：策略查询工具——**排查"某条访问到底被谁授权"时可以先想到它有现成工具**，不用手工 grep。
- `fc_sort.py`：file_contexts 排序检查（排序影响可读性/二分查找效率）。

CI 侧，仓库自带 `TEST_MAPPING`，其中 presubmit 明确挂到了 CTS 的类与方法名（逐字）：测试目标是 `CtsSecurityHostTestCases`，`include-filter` 四条为：

```
android.security.cts.SELinuxHostTest#testPermissionControllerDomain
android.security.cts.SELinuxHostTest#testVzwOmaTriggerDomain
android.security.cts.SELinuxHostTest#testGMSCoreDomain
android.security.cts.SELinuxHostTest#testNoBugreportDenials
```

这四条很值得注意：其中三条是**针对具体包被分配到期望域**的检查（PermissionController、Verizon 的 OMA 触发组件、GMS Core），一条是**要求不产生 bugreport 相关的 denial**。也就是说，"某个 app 域名被改了"是会直接把 CTS 打挂的——域不只是内部实现细节。此外 `avf-presubmit` 挂了 `ComposHostTestCases` 与 `MicrodroidHostTestCases`，与 §3 的 `microdroid/` 目录和 AVF 组件式更新相呼应。

VTS 侧能看到一条线索（`Android.bp:50–52` 的注释与 filegroup）：

```
// For vts_treble_sys_prop_test
filegroup {
    name: "private_property_contexts",
    srcs: ["private/property_contexts"],
    visibility: ["//test/vts-testcase/security/system_property"],
}
```

**`property_contexts` 是被 VTS 的系统属性测试消费的**，并且通过 `visibility` 限定了只有那个测试目标能用——这是"属性标签也是被强制测试的接口"的成文依据。

---

## 11. 常见误传纠正

以下每条都对应本手册正文里可复核的出处；未标注出处的属于"缺乏依据、因此不采信"类。

1. **"SELinux 和 seccomp 都是权限管理，配一个就行。"** 两者判据不同（对象访问矩阵 vs 系统调用号），层次不同（§1）。SELinux 拦不住一个被放行的 syscall 的行为；seccomp 也不管你这个被允许的调用能动谁的文件。
2. **"SELinux 从一开始就在保护 app 之间的隔离。"** 官方逐字说法相反：5.0 阶段"all third-party apps ran within the same SELinux context"，app 间隔离"primarily enforced by UID DAC"；按 app 粒度的 MAC 是 Android 9 起、且条件是 `targetSdkVersion >= 28`（§2）。
3. **"`untrusted_app` 就是第三方应用的域。"** 现行 `seapp_contexts:205` 把它限定为 `minTargetSdkVersion=34`；`_32/_30/_29/_27/_25` 各有区间，且注释明说不必要的档位被跳过（§5）。
4. **"`platform_app` 和 `system_app` 都算系统应用，区别不大。"** 分界是 `user=system` 与 `seinfo=platform` 的组合，数据目录类型也不同（`app_data_file` vs `system_app_data_file`）（§5）。
5. **"isolated process 只是少给一些权限。"** `private/isolated_app_all.te:43` 是 `neverallow … app_data_file_type:file open;`——被禁的动作是"打开"，不是"读"（§7）。
6. **"app 可以 ptrace 自己/调试工具总能 ptrace app。"** 双向都被成文禁止；唯二例外是 `crash_dump`，以及 userdebug/eng 下的 `llkd`（`private/app.te:562`、`:571–578`）（§1、§7）。
7. **"给 app 加个 capability（比如 `CAP_NET_RAW`）不影响 SELinux。"** `private/app.te:524` 成文禁止 appdomain 拿 capability，只排除 bluetooth 与 network_stack 两个域（§7）。
8. **"看到 `avc: denied` 就说明操作被拒绝了；没看到就说明没发生。"** 两条都不成立：`auditallow` 会"允许且记录"（§7 的 `untrusted_app_all.te` 实例），`dontaudit` 会"拒绝且不记录"，而本检出的 `dontaudit` 规则有 236 条（§7）。
9. **"厂商想加 allow 就加，策略是本地的事。"** 光 neverallow 就有 988 条（§7），且 `public/` 里成文写着 DO NOT ADD allow/neverallow/dontaudit，规则要落在 `private/`（§3）；Treble/冻结/CTS/VTS 检查在 §10。
10. **"`/dev/binder` 能用，hwbinder/vndbinder 也就差不多能用。"** 三个节点是三个类型（`binder_device` / `hwbinder_device` / `vndbinder_device`），授权彼此独立（§6）。
11. **"改 file_contexts 就能改一个 app 的域。"** app 进程上下文来自 `seapp_contexts` 的三级映射（签名→seinfo→域/类型/级别），`file_contexts` 管对象标签（§4、§6）。
12. **"SELinux 策略版本就是一個数字。"** 至少有两条轴：`SEPOLICY_VERSION=30`（编进代码的宏）与 `prebuilts/api/` 的发布格式版本（`29.0…34.0` 与 `202404/202504` 并存）（§9）。
13. **"userdebug 上验证过的策略行为等于 user 版。"** 构建产物里明确有 `userdebug_plat_sepolicy.cil`、`system_ext_userdebug_plat_sepolicy.cil`；§7 的 `userdebug_or_eng()` 例外只在 userdebug/eng 生效（§9）。
14. **"seccomp 被拦下时进程会收到一个可读的 errno。"** app 过滤器的 Disallow 分支返回 `SECCOMP_RET_TRAP`（§8），表现为 SIGSYS。
15. **"app 能通过设备节点直接和 TEE 说话。"** `neverallow appdomain tee_device:chr_file { read write };`（§7）——这也是"必须经 KeyMint/Gatekeeper 这类服务"结论在强制层的落点，但不要把它当成"TEE 侧无缺陷"的证据，见 [TEE 与密钥认证手册](/android-tee-key-attestation/)。
16. **"sepolicy 里没有 permissive 的问题需要关心。"** 我在本检出没有找到成文的 permissive 约束，也**不能**据此断言平台禁止——这条属于"未核实"，写作和评审时不要把它当结论引用（§7 末尾）。

---

## 12. 观测点与自查

**先说边界**：本节只列我能从 AOSP 成文内容或通用 Linux 语义说明的观测方向。**具体命令在 user 版构建上是否可用、输出格式如何，我没有设备可实测，一律标注为未核实**；不要因为本手册列了某个方向就认为该命令在你的目标机上存在。

从**设备侧**可考虑的方向（需自行实测）：

- 当前进程的 SELinux 上下文（`ps -Z` 一类的域标签视图）：能直接看到 §5 的域名分配结果，特别是 `targetSdk` 不同的两个应用会落在不同 `untrusted_app*` 上。
- 全局强制状态（`getenforce`/`setenforce`）与文件标签（`ls -Z`）：**user 版可用性、以及 `setenforce` 是否被策略允许，本手册未核实**。
- denial 审计：内核日志中的 avc 记录；注意 §7 的 `dontaudit`（236 条）会抑制记录，因此"没有 denial"不等于"没有越界尝试"。
- 策略产物与 mapping 文件的存在性：§9 给出了它们在源码里的模块名与安装目录（`/system/etc/selinux/mapping`），可据此在设备镜像里找对应文件；**具体设备上路径与文件名的实测结果未核实**。
- 属性标签：`private/property_contexts` 是 §10 里被 VTS `vts_treble_sys_prop_test` 消费的清单，因此它是**接口**的一部分，改动要连带跑该测试。

从**策略/源码侧**（这部分是本手册的主场，都可复核）：

1. 判断"某应用应该落在哪个域" → 读 `private/seapp_contexts`：先按签名查 `mac_permissions.xml` 的 `seinfo`，再查 `isPrivApp`/`isEphemeralApp`/`minTargetSdkVersion` 三条分流（§4、§5）。
2. 判断"这条访问是被允许还是被成文禁止" → 在 `private/*.te` 里找同名 `neverallow`（§7 的表给了 20 余处起点），而不是只看 `allow`。
3. 判断"这是不是厂商可以自行放宽的点" → 有 neverallow 覆盖的就不行；§3 的目录纪律与 §10 的测试是流程保障。
4. 判断"跨 binder 域的调用是否被误授权" → 对着 §6 的三个类型分别检查。
5. 判断"某 syscall 在 app 里能不能叫" → 定位到 bionic `libc/seccomp/seccomp_policy.cpp` 的对应架构过滤器（§8），并逐 ABI 检查 primary/secondary。
6. 判断"userdebug 上验证的结论能不能推到 user 版" → 看 §9 的 `userdebug_plat_sepolicy.cil` 与 `userdebug_or_eng()` 包裹。

**修复与处置优先级建议（针对策略类缺陷）**：

| 情形 | 优先级依据 |
| --- | --- |
| 某 app 域被 allow 到了 `tee_device` / 块设备 / 他人 `/proc` —— 即补丁**违反** neverallow | 属于策略正确性问题：这类规则的存在本身就是平台的成文意图，出现与之矛盾的授权应按高危处理 |
| 某厂商域过宽（例如把敏感服务放进 `appdomain` 家族） | 看该域实际能触达的对象类；`appdomain` 意味着进入 §7 全套约束，把非应用服务标成 app 域是常见错法 |
| `seapp_contexts` 的包名点名被误配（应用拿到比预期更高的域） | 直接改变 MAC 粒度，且会触发 §10 的 CTS 域名检查（`SELinuxHostTest#test*Domain` 三条） |
| 仅 `dontaudit` 导致排障看不见 | 观测缺口不等于风险缺口——把「审计里看不到」读成「没有风险」是本末倒置；反之，补一条 `auditallow` 也不等于收紧了边界 |

---

## 13. 与其他手册的分工

| 手册 | 它管什么 | 本手册把什么留给它 |
| --- | --- | --- |
| [多用户、工作资料与进程隔离](/android-multiuser-sandbox-isolation/) | 用户维度与进程维度的边界、`DISALLOW_*`、isolated/USAP/AVF | 域名的完整分流表与 neverallow 清单（§5、§7） |
| [系统底层漏洞类型图谱](/android-low-level-vulnerability-types/) | 缺陷按组件定位（Binder 三域、ashmem/memfd、ION/dma-buf、HAL） | 这些对象在 SELinux 里的类型实名（§6） |
| [TEE、密钥管理与设备认证](/android-tee-key-attestation/) | KeyMint/Gatekeeper/证明链与 FBE | `tee_device` 的强制面依据（§7），硬件侧契约不在此 |
| [Mainline 与 APEX 模块](/android-mainline-apex-modules/) | 模块与容器、更新通道、SPL | `apex/*-file_contexts` 与其检查（§3、§10） |
| [RCE 与权限提升](/android-rce-and-privilege-escalation/) | 执行上下文与提权三段式 | seccomp 过滤器随上下文变化这一点（§8） |
| [ADB、备份与数据迁移通道](/android-adb-backup-transfer/) | adbd 授权与数据通道 | `adbd_socket` 的 neverallow 依据（§7） |
| [基带与电话子系统](/android-baseband-telephony/) | RIL/调制解调器两侧 | `radio` 域与 `rild_socket` 的强制面（§7） |
| [生物识别与锁屏凭据](/android-biometric-lockscreen/) | 凭据链与 Class 分档 | 本手册不涉及 |

内核自身（GKI/KMI、CFI/PAC/MTE、供应商驱动）与 SELinux/seccomp 是**不同的修复责任面**：前者的补丁落在内核与模块，后者的补丁落在策略与 libc。本手册不含内核部分。

---

## 参考

本地一手（可逐行复核）
- AOSP `system/sepolicy`（本地检出 HEAD `4571ddd`）：`README.apps.md` · `public/attributes` · `public/untrusted_app.te` · `public/te_macros` · `private/untrusted_app_all.te` · `private/seapp_contexts` · `private/mac_permissions.xml` · `private/file_contexts` · `private/app.te` · `private/isolated_app_all.te` · `private/ephemeral_app.te` · `private/priv_app.te` · `private/app_zygote.te` · `private/dex2oat.te` · `private/profman.te` · `private/netd.te` · `private/logpersist.te` · `private/init.te` · `private/domain.te` · `reqd_mask/` · `apex/` · `prebuilts/api/` · `tests/` · `TEST_MAPPING` · `Android.bp`
- 在线同一仓库：[system/sepolicy](https://android.googlesource.com/platform/system/sepolicy/+/refs/heads/main) · [README.apps.md](https://android.googlesource.com/platform/system/sepolicy/+/refs/heads/main/README.apps.md) · [private/app.te](https://android.googlesource.com/platform/system/sepolicy/+/refs/heads/main/private/app.te) · [private/seapp_contexts](https://android.googlesource.com/platform/system/sepolicy/+/refs/heads/main/private/seapp_contexts) · [Android.bp](https://android.googlesource.com/platform/system/sepolicy/+/refs/heads/main/Android.bp)
- bionic seccomp：[libc/seccomp/ 目录](https://android.googlesource.com/platform/bionic/+/refs/heads/main/libc/seccomp/) · [seccomp_policy.cpp](https://android.googlesource.com/platform/bionic/+/refs/heads/main/libc/seccomp/seccomp_policy.cpp)

官方文档
- [Application Sandbox](https://source.android.com/docs/security/app-sandbox)（§1、§2 的逐字引用来源）
- [SELinux in Android](https://source.android.com/docs/security/features/selinux) · [Permissions allowlist（priv-app）](https://source.android.com/docs/core/permissions/perms-allowlist)
- [AVF / Microdroid](https://source.android.com/docs/core/virtualization) · [Compos](https://source.android.com/docs/core/virtualization/compos)
- [Treble](https://source.android.com/docs/core/architecture/treble)

源码交叉
- libselinux 的 app 上下文查找实现：`external/selinux/libselinux/src/android/android_platform.c`（`seapp_context_lookup()`，由 `seapp_contexts` 头部注释点名）

---

## 未核实与失效来源

**本手册未做的事（明确隔离，不写进正文结论）**

- **没有 CVE 案例表。** SELinux / `seapp_contexts` / seccomp 侧的公开条目需要逐条打开对应月度公告核对 Component、Android ID、严重级与修复版本；本次未做这份核对，因此本手册不给任何 CVE 编号，避免把不相关的条目（如显示驱动、指纹组件）误当作策略缺陷。需要案例的读者请先查 [漏洞编号与公告体系手册](/android-cve-and-security-bulletins/) 的查询方法，再逐条核对。
- 本地 sepolicy 检出的 git 历史里 **grep `CVE` 与 `neverallow` 关键字均无命中**（`git log --grep`，命中数 0）；这份检出的历史深度不足以从提交信息里提取安全修复台账，故未采用。

**未核实（正文里已就地标注）**

1. `seapp_contexts` 各条目之间的**匹配顺序/优先级**语义（first-match-wins 还是最优匹配）：文件注释只写了"所有指定选择器必须匹配"，未见顺序规则。
2. app seccomp 策略中**被 Disallow 的具体 syscall 编号清单**（需逐字读 `func_to_syscall_nrs.h` 与生成表）。
3. `set_app_seccomp_filter()` 在 zygote 启动路径上的**确切调用点**，以及是否存在属性/环境变量开关（策略文件内容里未见相关分支）。
4. `plat_sepolicy.cil` / mapping / vendor CIL 交给 `secilc` 的**运行时顺序与失败行为**（只见模块名与安装路径，未见运行时文档逐字确认）。
5. **permissive** 声明是否被平台成文限制；本检出未见相关 neverallow，也未见 `^permissive` 声明。
6. `flagging/flagging_macros` 的用途与语义。
7. `sepolicy_freeze_test.py` 的**具体判定规则**（只确认脚本存在）。
8. 设备侧命令（`getenforce`、`ps -Z`、`ls -Z`、`setenforce`）在 user 版构建上的可用性与输出格式；`/sys/fs/selinux` 下各节点在设备镜像里的实际存在情况。
9. `private/mac_permissions.xml` 里除 `platform`、`sdk_sandbox` 之外是否还有其他 seinfo 取值（本检出的这份文件只逐字给出这两例的 signer 结构）。
10. app 域对 `selinuxfs` 的**写**权限：`private/domain.te:253–256` 只逐字允许 `dir search`、`file getattr`、`filesystem getattr`；我没有找允许写的规则，但没有据此断言"绝对不可写"。
11. `/data/data` 是否在某个版本的 `file_contexts` 里以独立行存在。

**来源状态（实测）**

- `source.android.com/docs/security/features/seccomp` 与 `.../features/selinux/domains`、`/domains/app`：在早前轮次实测为 **404**，本手册因此改用 `app-sandbox` 页与 AOSP 源码作为引用来源。
- `developer.android.com/ndk/guides`：实测该页**不讨论 seccomp**（无任何 syscall 过滤相关表述），故未把它作为 seccomp 的依据。
- 本检出 `mac_permissions/` 目录只剩 `Android.bp`；`reqd_mask/mac_permissions.xml` 为 61 字节的空壳（`<policy></policy>`）。

---

## 联系方式

咨询 Android 系统详情请联系 https://t.me/one00190

---

*本手册仅供安全学习与防御研究使用；所有域名、策略文件路径、行号、宏名、测试名称与案例归属均取自 AOSP 源码与公开一手来源并附链接，未能核实的表述已单列于「未核实与失效来源」而非写入正文。*
