---
layout: page
title: "Android 多用户、工作资料与进程隔离手册"
description: "用户类型与 UID 计算公式、DISALLOW_* 策略常量的真实宿主类、SELinux 域与 seccomp 的现行官方措辞、isolated process / USAP / AVF 三层进程隔离边界；含 10 条多用户与 Zygote 侧已核实案例、15 条常见误传纠正与可执行的实况观测清单"
date: 2026-10-26
permalink: /android-multiuser-sandbox-isolation/
---

> "隔离"在 Android 里是两个不同层次的东西：**用户维度的隔离**（谁的数据、谁的策略、谁能看见谁）与**进程维度的隔离**（哪个 SELinux 域、哪个 UID 区间、哪个 seccomp 过滤器）。它们交叉出现在同一个缺陷里，但修复责任完全不同。本手册把这两条轴拉直，并把大量教程里写错的命令、常量名与 API 归属逐条纠正。定位是防御与教育，只引用公开且已修复的信息，不含可利用代码。

---

## 0. 使用方式与边界

**与相邻手册的分工。** [生物识别与锁屏手册](/android-biometric-lockscreen/) 处理"凭据→密钥"这条链；[TEE 与密钥认证手册](/android-tee-key-attestation/) 处理密钥硬件侧契约；本手册处理**数据与策略的归属边界**：用户类型、工作资料的 UID 计算、`DISALLOW_*` 到底在哪个类、isolated process 与 SELinux 域、Zygote 侧的清理，以及 AVF 这条新一代隔离边界。

**怎么用。**
- 判断某条策略/常量/API 的真实宿主类 → §1、§2、§6
- 搞清楚工作资料到底是"用户"还是"进程" → §1（UID 公式）
- 核对沙箱与 seccomp 的现行官方措辞（而不是旧文档的） → §3
- 排查一个 isolated 进程该有什么不该有什么 → §4
- 核对某条多用户/工作资料 CVE 的真实归属 → §5

**边界。** 不涉及任何跨用户数据读取、策略绕过或容器逃逸的操作细节；不涉及 MDM 厂商实现（只到 AOSP 契约层）。

---

## 1. 用户维度：定义、UID 公式与真实命令

### 官方定义（逐字）

[Support multiple users](https://source.android.com/docs/devices/admin/multi-user)：

> "System user: First user added to a device… cannot be removed except by factory reset"
> "Secondary user: Any user added to the device other than the system user"

profile 的分类含 Managed / Restricted / Clone / Private profile；**user type 实名**是 `android.os.usertype.full.SECONDARY`、`android.os.usertype.profile.MANAGED`、`android.os.usertype.profile.CLONE`、`android.os.usertype.profile.PRIVATE`。"full secondary user" 这一说法见于 Android 15 Automotive 段落。

工作资料的定义同样是一手的（[Employ work profiles](https://source.android.com/docs/devices/admin/managed-profiles)）：

> "A work profile is a managed profile that has separate app data from the primary user profile… create a segregated and secure container to hold managed data"
> "Work profiles are implemented as secondary users, such that the apps running in the work profile have a UID of **uid = 100000 \* userid + appid**. These profiles have separate app data (**/data/user/userid**)."

最后这句是本手册最重要的结构性事实：**工作资料在用户维度是一个 secondary user，在存储维度是一个独立目录，在权限维度由 UID 前缀区分。** 讨论"工作资料能不能访问个人数据"时，答案是沿这三条轴分别成立的机制决定的，不是某一条"隔离开关"。

### 常量实名

[UserHandle.java](https://android.googlesource.com/platform/frameworks/base/+/refs/heads/main/core/java/android/os/UserHandle.java)：`PER_USER_RANGE = 100000`，`getUid(userId, appId) = userId * PER_USER_RANGE + (appId % PER_USER_RANGE)`。

[Process.java](https://android.googlesource.com/platform/frameworks/base/+/refs/heads/main/core/java/android/os/Process.java)：`FIRST_ISOLATED_UID = 99000`、`LAST_ISOLATED_UID = 99999`；另有 `FIRST_APP_ZYGOTE_ISOLATED_UID = 90000`、`LAST_APP_ZYGOTE_ISOLATED_UID = 98999`。

**纠错：`UserHandle.AID_ISOLATED_OFFSET` 不存在**（源码 grep 无匹配）。sepolicy 侧的注释使用的是 `AID_ISOLATED_START`/`AID_ISOLATED_END`——这是两套命名，写讨论时不要混。

### 命令实名（这一段能纠正很多流传的教程）

现行创建用户命令（[PackageManagerShellCommand.java](https://android.googlesource.com/platform/frameworks/base/+/refs/heads/main/services/core/java/com/android/server/pm/PackageManagerShellCommand.java)）：

```
pm create-user [--profileOf USER_ID] [--managed] [--restricted] [--guest] [--user-type …] USER_NAME
```

**`pm create-managed-user` 不存在**——在 main、android-11.0.0_r48、android-9.0.0_r35 三个分支的源码中都检索不到。`am` 侧实名是 `get-current-user`、`start-user`、`stop-user`；**`am multi-users` 同样不存在**。

## 2. 策略轴：`DISALLOW_*` 的宿主类是 UserManager

这是被写错频率最高的一处：**`DISALLOW_*` 常量定义在 `android.os.UserManager`，不是 `DevicePolicyManager`**（[UserManager.java](https://android.googlesource.com/platform/frameworks/base/+/refs/heads/main/core/java/android/os/UserManager.java)）。

已核实存在：

| 常量 | 值/含义 |
|---|---|
| `DISALLOW_DEBUGGING_FEATURES` | `"no_debugging_features"`，官方成文："it disables debugging features altogether, including USB debugging" |
| `DISALLOW_CROSS_PROFILE_COPY_PASTE` | 跨 profile 复制粘贴 |
| `DISALLOW_ADD_USER` / `DISALLOW_REMOVE_USER` | 用户增删 |
| `DISALLOW_ADD_MANAGED_PROFILE` / `DISALLOW_REMOVE_MANAGED_PROFILE` | 托管 profile 增删 |
| `DISALLOW_CONFIG_CREDENTIALS` | 凭据配置 |
| `DISALLOW_INSTALL_UNKNOWN_SOURCES_GLOBALLY` | 全局未知来源安装 |
| `DISALLOW_CAMERA` / `DISALLOW_NETWORK_RESET` | 相机 / 网络重置 |

**未检索到：`DISALLOW_NETWORK_ACCESS`。** 讨论"MDM 能不能断网"时不要引用这个不存在的常量名。

托管模式命名同样要用实名：官方为 **"Fully managed device (also referred to as device owner mode)"**、**"profile owner mode"**、**"work profiles on company-owned devices"**。**"organization owner"、`setOrganizationOwner`、"globally managed" 在 AOSP 与 source.android.com 均检索不到**。可用的公开 API 是 `isDeviceOwnerApp`、`isProfileOwnerApp`、`isDeviceManaged`、`getDeviceOwner`、`getProfileOwner`、`getOrganizationNameForUser`；`set/getDeviceOwnerType`（DEFAULT/FINANCED）是 `@hide`。另：**没有 `getOwnerType` 这个方法**。

「managed configuration」的官方文档实名是 "Set up managed configurations"（[developer.android.com](https://developer.android.com/work/managed-configurations)）。

删除语义是明确的一句（[Google 工作区帮助](https://support.google.com/work/android/answer/6191949?hl=en)）：

> "After the Work Profile is deleted, all local data on the device within that profile is deleted"

注意范围限定在 **within that profile**——source.android.com 上没有更细的"清除范围"专段（未核实）。

## 3. 沙箱轴：SELinux 与 seccomp 的现行措辞

### SELinux

现行官方表述**不是**旧页面上的 "enabled in enforcing mode by default"，而是（[SELinux in Android](https://source.android.com/docs/security/features/selinux)）：

> "Android includes SELinux in enforcing mode and a corresponding security policy that works by default across AOSP"

域与类型实名（sepolicy 源码）：`type untrusted_app`、`type priv_app, domain`、`type platform_app`、`type system_app, domain`、`type isolated_app, domain`、`type ephemeral_app, domain`，以及属性 `appdomain`（[isolated_app.te](https://android.googlesource.com/platform/system/sepolicy/+/refs/heads/main/public/isolated_app.te)、[untrusted_app_all.te](https://android.googlesource.com/platform/system/sepolicy/+/refs/heads/main/private/untrusted_app_all.te)）。

数据标签实名：`app_data_file`、`privapp_data_file`、`system_app_data_file`，属性 `app_data_file_type`。**没有 `app_domain`、也没有 `app-data-file-label`**。

目录侧：`/data/user`、`/data/user_de`，且存在 `symlink("/data/data", "/data/user/0")`——也就是说 `/data/data` 是 user 0 的符号链接，不是独立目录（[Zygote JNI](https://android.googlesource.com/platform/frameworks/base/+/refs/heads/main/core/jni/com_android_internal_os_Zygote.cpp)）。

存储视图：官方页实名 **"SDCardFS deprecation"**，措辞是 "SDCardFS is deprecated on devices that launch with Android 11 or higher and run kernel version 5.4 or higher"——**它写的是 deprecated，不是"已移除"**，引用时要保持这个精度。存储侧隔离的文档实名是 **"Scoped storage"**（[链接](https://source.android.com/docs/core/storage/scoped)）。

失效提示：`source.android.com/docs/security/features/selinux/domains`（及 `/app`）现均 404，旧域名页不可再作为出处。

### seccomp

专门文档页 404，现行可引的是应用沙箱页里的一句（[Application Sandbox](https://source.android.com/docs/security/app-sandbox)）：

> "In Android 8.0, all apps were set to run with a seccomp-bpf filter that limited the syscalls that apps were allowed to use"

源码侧：Zygote 的 `SetUpSeccompFilter()` 分发到 `set_app_seccomp_filter()` / `set_app_zygote_seccomp_filter()` / `set_system_seccomp_filter()`，且存在这段判定：

```
if (!gIsSecurityEnforced) { ALOGI("seccomp disabled by setenforce 0"); }
```

（[com_android_internal_os_Zygote.cpp](https://android.googlesource.com/platform/frameworks/base/+/refs/heads/main/core/jni/com_android_internal_os_Zygote.cpp)）。这行日志值得所有做加固核查的人注意：**SELinux 被设为 permissive 会连带关闭 seccomp 过滤器**——这两条常被当成独立开关。

bionic 侧注释用词是 **"allowlist"**，不是 "whitelist"（[seccomp_policy.cpp](https://android.googlesource.com/platform/bionic/+/refs/heads/main/libc/seccomp/seccomp_policy.cpp)）。未核实：media_codecs 是否有独立 seccomp 策略——bionic 该文件只有 app / app_zygote / system 三类，frameworks/av 的候选路径均 404。

## 4. 进程轴：isolated process、USAP 与 AVF

### isolated process

Manifest 属性实名是 **`android:isolatedProcess`**（**没有 `android:isolatedService`**），官方描述：

> "If set to \"true\", this service runs under a special process that is isolated from the rest of the system and has no permissions of its own."（[\<service> 文档](https://developer.android.com/guide/topics/manifest/service-element)）

配套事实：`ServiceInfo.FLAG_ISOLATED_PROCESS = 0x0002`（[ServiceInfo.java](https://android.googlesource.com/platform/frameworks/base/+/refs/heads/main/core/java/android/content/pm/ServiceInfo.java)）；`seapp_contexts` 中 `user=_isolated` **必须**映射到 `isolated_app` 域（由 neverallow 强制），另有 `isolated_compute_app`（[seapp_contexts](https://android.googlesource.com/platform/system/sepolicy/+/refs/heads/main/private/seapp_contexts)）。UID 落在 §1 的 99000–99999 区间。

WebView 是这套机制的教科书用例：官方 "Renderer process… It is isolated from the rest of the system"，Google 博客进一步写 "the renderer process is run within an isolated process sandbox"（[WebView 内存指南](https://developer.android.com/topic/performance/memory/guide/webview-memory)）。未核实：WebView 沙箱逃逸对应的 Android 侧公告条目（NVD 关键词检索无果）。

### USAP 与 App Zygote

USAP 真实存在，不是概念杜撰：`Zygote.java` 注释 "Specialize an unspecialized app process"、常量 `USAP_POOL_PRIMARY_SOCKET_NAME = "usap_pool_primary"`、方法 `getUsapPoolCount()`；JNI 侧 "forks an unspecialized app process from the Zygote"；入口二进制是 `app_process`（[Zygote.java](https://android.googlesource.com/platform/frameworks/base/+/refs/heads/main/core/java/com/android/internal/os/Zygote.java)、[app_main.cpp](https://android.googlesource.com/platform/frameworks/base/+/refs/heads/main/cmds/app_process/app_main.cpp)）。

防御意义：预特化的进程池意味着"fork 时未清干净"这类缺陷有真实载体——§5 的 CVE-2020-0257 正落在这里。未核实：USAP/App Zygote 池的专属 CVE（NVD "app zygote Android" 仅 3 条，其中无 USAP 专条）。

### AVF：设备内新增的虚拟化边界

**没有 `VirtualizationManager` 这个类**。SystemApi 实名是 `android.system.virtualmachine.VirtualMachineManager`，服务名 `Context.VIRTUALIZATION_SERVICE = "virtualization"`。

官方表述（[AVF](https://source.android.com/docs/core/virtualization)）：

> "Android Virtualization Framework (AVF) provides secure and private execution…"
> "…even if host Android or any of the other pVMs are compromised"

pKVM = protected kernel-based virtual machine；[Microdroid](https://source.android.com/docs/core/virtualization/microdroid) 的实名定义是 "Microdroid is a mini-Android OS that runs in a pVM"。

这是本手册里唯一一条"隔离层次高于 SELinux 域"的机制：它假设的是宿主可能已被 compromises。讨论"下一代沙箱"时引用这段，而不是引用不存在的 `VirtualizationManager`。

## 5. 已修复公开案例（组件、严重级、版本、Android ID 均取自公告表格）

### 多用户与工作资料

| CVE | 落点 | 组件/类型/级别/版本/ID/公告 |
|---|---|---|
| CVE-2022-20138 | DevicePolicyManagerService（`ACTION_MANAGED_PROFILE_PROVISIONED`） | Framework，EoP，High，10/11/12/12L，A-210469972，[2022-06-01](https://source.android.com/docs/security/bulletin/2022-06-01)（[NVD](https://nvd.nist.gov/vuln/detail/CVE-2022-20138)） |
| CVE-2021-0982 | `getOrganizationNameForUser` 组织名泄露 | Pixel 2021-12-01，Framework，ID，Moderate，12，A-192368508（[公告](https://source.android.com/docs/security/bulletin/pixel/2021-12-01)） |
| CVE-2021-0986 | `hasGrantedPolicy` device/profile owner 信息泄露 | 同上，A-192247339 |
| CVE-2021-0568 | `onReceive`：被禁用的 profile 被重新启用 | Pixel 2021-06-01，Framework，EoP，Moderate，11，A-170121238（[公告](https://source.android.com/docs/security/bulletin/pixel/2021-06-01)） |
| CVE-2023-20914 | `AdminRestrictedPermissionsUtils`：工作资料读取短信 | Framework，ID，High，11，A-189942529，[2023-05-01](https://source.android.com/docs/security/bulletin/2023-05-01) |
| CVE-2022-20124 | `DeletePackageHelper`：Guest 用户重置其他用户预装应用 | Framework，EoP，High，10–13，A-170646036，[2022-12-01](https://source.android.com/docs/security/bulletin/2022-12-01) |
| CVE-2021-0602 | `WifiNetworkDetailsFragment`：guest 查看/修改全部 Wi‑Fi | System，EoP，High，10/11，A-177573895，[2021-07-01](https://source.android.com/docs/security/bulletin/2021-07-01) |

读法：这一组的分布很有指导性——**Guest 用户是"跨用户影响"的高发入口**（两条 High 都涉及 guest 操作影响他人），而**工作资料的高发影响是信息维度（ID）而非执行维度**。MDM 侧最该盯的是 profile 生命周期事件（provisioned/enabled/disabled）。

### 进程隔离与 Zygote

| CVE | 落点 | 组件/类型/级别/版本/ID/公告 |
|---|---|---|
| CVE-2023-21117 | `ActivityManagerService.registerReceiverWithFeature`：isolated 进程注册广播接收器 | Framework，EoP，High，13，A-263358101，[2023-05-01](https://source.android.com/docs/security/bulletin/2023-05-01) |
| CVE-2020-0257 | `com_android_internal_os_Zygote.cpp` `SpecializeCommon` 清理不完整（"incomplete cleanup"） | Framework，EoP，High，10，A-156741968，[2020-08-01](https://source.android.com/docs/security/bulletin/2020-08-01) |
| CVE-2020-0390 | "In the app zygote SE Policy" | Framework，ID，High，10，A-157598026，[2020-09-01](https://source.android.com/docs/security/bulletin/2020-09-01) |

CVE-2023-21117 值得单独强调：它证明 **isolated process 的约束是靠框架侧 API 校验兜底的**，SELinux 域本身不足以阻止一个 isolated 进程拿到广播接收器。写自查清单时，"声明了 isolatedProcess 就安全"是不成立的假设。

### KEV 核对

CISA KEV（catalogVersion 2026.10.23，1756 条）对上述 **10 个 CVE 逐条精确匹配：全部不在 KEV**；且 KEV 产品列中无 Android 条目（连跨厂商碰撞也不存在）。出处：[KEV JSON](https://www.cisa.gov/sites/default/files/feeds/known_exploited_vulnerabilities.json)。

## 6. 常见误传纠正

| 误传 | 实际 |
|---|---|
| `DISALLOW_*` 在 `DevicePolicyManager` | 定义在 `android.os.UserManager` |
| `DISALLOW_NETWORK_ACCESS` 存在 | 源码未检索到 |
| `pm create-managed-user`、`am multi-users` 可用 | main/11/9 均不存在；实名 `pm create-user --managed`、`am get-current-user/start-user/stop-user` |
| `UserHandle.AID_ISOLATED_OFFSET` | 不存在；sepolicy 注释用 `AID_ISOLATED_START/END` |
| `android:isolatedService` | 实名 `android:isolatedProcess`；`ServiceInfo.FLAG_ISOLATED_PROCESS = 0x0002` |
| `DevicePolicyManager.getOwnerType` / `setOrganizationOwner` / "globally managed" | 均不存在；用 `isDeviceManaged`/`isDeviceOwnerApp`/`isProfileOwnerApp`/`getDeviceOwner`/`getProfileOwner`/`getOrganizationNameForUser` |
| SELinux 官方说法是 "enabled in enforcing mode by default" | 现行措辞："Android includes SELinux in enforcing mode and a corresponding security policy that works by default across AOSP" |
| seccomp 官方页可读 | `features/seccomp` 已 404；改用 Application Sandbox 页的 Android 8.0 那句 |
| 有 `app_domain`、`app-data-file-label` | 实名 `app_data_file` 等 + 属性 `app_data_file_type` |
| SDCardFS 已移除 | 官方写 deprecated（Android 11+ 且内核 5.4+ 的设备） |
| 有 `VirtualizationManager` | 实名 `android.system.virtualmachine.VirtualMachineManager`，服务名 `virtualization` |
| `/data/data` 是独立于 `/data/user/0` 的目录 | 它是 `/data/user/0` 的符号链接 |
| 工作资料是"同一用户下的容器" | 官方明确 "implemented as secondary users"，`uid = 100000 * userid + appid` |
| 声明 `isolatedProcess` 即自带完整隔离 | 框架侧 API 校验同样关键（见 CVE-2023-21117） |
| setenforce 0 只影响 SELinux | Zygote 代码在 `!gIsSecurityEnforced` 时跳过 seccomp 安装 |

## 7. 观测点与自查

**可读实况。**

```
adb shell pm list users
adb shell dumpsys user
adb shell dumpsys device_policy
```

`pm list users` 实名存在（"list users — Lists the current users."）；`dumpsys user` 由 `UserManagerService.dump` 输出 `"Users:"` 段；官方 Automotive 文档甚至给出 `adb shell dumpsys user | grep PACKAGE_SUBSTRING` 与 `adb shell cmd user report-system-user…` 形式（[disable packages](https://source.android.com/docs/automotive/users_accounts/disable_packages)）。`dumpsys device_policy` 对应 `Context.DEVICE_POLICY_SERVICE = "device_policy"`，DPMS.dump 输出 `"Current Device Policy Manager state:"`。未核实：`pm list users` 的逐行输出格式（不凭记忆补字段）。

**自查清单（防御视角）。**
1. 枚举设备上的全部用户与 profile，确认每个 user type 是否预期（`android.os.usertype.*` 四类实名）。
2. 对每个托管 profile，记录 device owner / profile owner 是谁、组织名（`getOrganizationNameForUser`）是否可见——后者曾有信息泄露条目。
3. 检查是否设置 `DISALLOW_DEBUGGING_FEATURES`；它与 [ADB 与备份手册](/android-adb-backup-transfer/) 的 `adb_enabled` 是同一防线的两端。
4. 核查 app 的 SELinux 域预期与实际一致（`untrusted_app`/`priv_app`/`isolated_app`/`ephemeral_app`），别把"用了 isolatedProcess"当成结论。
5. 内核/策略侧确认未运行在 permissive（否则 seccomp 同时失效）。
6. 数据目录核查用 `/data/user/<id>` 视角，不要基于 `/data/data` 做跨用户推断。
7. Clone profile（双开）在 AOSP 只有平台层支持："Clone profile… enable running two instances of a single app… AOSP doesn't provide end-to-end support… OEMs need to add customizations"——**没有官方机型/版本支持清单**，机型差异必须单独核。
8. Guest 相关的历史 High 集中于"影响他人"，把 guest 策略作为独立审计项。
9. 评估 AVF 用例时按"宿主可能已失陷"的假设建模，而不是把它当更强的 SELinux。

## 8. 与其他手册的分工

| 需要 | 去看 |
|---|---|
| `HardwareAuthToken`、CE/DE 密钥与凭据链 | [TEE 与密钥认证手册](/android-tee-key-attestation/) · [生物识别与锁屏手册](/android-biometric-lockscreen/) |
| 调试通道、备份与迁移的数据外泄面 | [ADB 与备份手册](/android-adb-backup-transfer/) |
| 提权链条与执行上下文 | [RCE 与提权手册](/android-rce-and-privilege-escalation/) |
| 定级与处置优先级 | [评估与定级手册](/android-vulnerability-severity-assessment/) |
| 本文 | 用户维度与进程维度两条轴、UID 公式、策略常量宿主类、沙箱/seccomp 现行措辞、isolated 与 AVF、多用户侧案例归属 |

---

## 参考

- [Support multiple users](https://source.android.com/docs/devices/admin/multi-user) · [Employ work profiles](https://source.android.com/docs/devices/admin/managed-profiles) · [Set up managed configurations](https://developer.android.com/work/managed-configurations) · [Delete work profile data](https://support.google.com/work/android/answer/6191949?hl=en)
- [UserManager.java](https://android.googlesource.com/platform/frameworks/base/+/refs/heads/main/core/java/android/os/UserManager.java) · [UserHandle.java](https://android.googlesource.com/platform/frameworks/base/+/refs/heads/main/core/java/android/os/UserHandle.java) · [Process.java](https://android.googlesource.com/platform/frameworks/base/+/refs/heads/main/core/java/android/os/Process.java) · [DevicePolicyManager.java](https://android.googlesource.com/platform/frameworks/base/+/refs/heads/main/core/java/android/app/admin/DevicePolicyManager.java) · [PackageManagerShellCommand.java](https://android.googlesource.com/platform/frameworks/base/+/refs/heads/main/services/core/java/com/android/server/pm/PackageManagerShellCommand.java)
- [Application Sandbox](https://source.android.com/docs/security/app-sandbox) · [SELinux in Android](https://source.android.com/docs/security/features/selinux) · [SDCardFS deprecation](https://source.android.com/docs/core/storage/sdcardfs-deprecate) · [Scoped storage](https://source.android.com/docs/core/storage/scoped)
- [sepolicy：isolated_app.te](https://android.googlesource.com/platform/system/sepolicy/+/refs/heads/main/public/isolated_app.te) · [untrusted_app_all.te](https://android.googlesource.com/platform/system/sepolicy/+/refs/heads/main/private/untrusted_app_all.te) · [seapp_contexts](https://android.googlesource.com/platform/system/sepolicy/+/refs/heads/main/private/seapp_contexts) · [file.te](https://android.googlesource.com/platform/system/sepolicy/+/refs/heads/main/public/file.te)
- [Zygote JNI](https://android.googlesource.com/platform/frameworks/base/+/refs/heads/main/core/jni/com_android_internal_os_Zygote.cpp) · [Zygote.java](https://android.googlesource.com/platform/frameworks/base/+/refs/heads/main/core/java/com/android/internal/os/Zygote.java) · [seccomp_policy.cpp](https://android.googlesource.com/platform/bionic/+/refs/heads/main/libc/seccomp/seccomp_policy.cpp) · [\<service> 文档](https://developer.android.com/guide/topics/manifest/service-element) · [WebView 内存指南](https://developer.android.com/topic/performance/memory/guide/webview-memory)
- [AVF](https://source.android.com/docs/core/virtualization) · [Microdroid](https://source.android.com/docs/core/virtualization/microdroid) · [Automotive：disable packages](https://source.android.com/docs/automotive/users_accounts/disable_packages)
- [公告 2023-05-01](https://source.android.com/docs/security/bulletin/2023-05-01) · [2022-12-01](https://source.android.com/docs/security/bulletin/2022-12-01) · [2022-06-01](https://source.android.com/docs/security/bulletin/2022-06-01) · [2021-07-01](https://source.android.com/docs/security/bulletin/2021-07-01) · [2020-08-01](https://source.android.com/docs/security/bulletin/2020-08-01) · [2020-09-01](https://source.android.com/docs/security/bulletin/2020-09-01) · [Pixel 2021-12-01](https://source.android.com/docs/security/bulletin/pixel/2021-12-01) · [Pixel 2021-06-01](https://source.android.com/docs/security/bulletin/pixel/2021-06-01)
- [CISA KEV JSON](https://www.cisa.gov/sites/default/files/feeds/known_exploited_vulnerabilities.json)

## 9. 未核实与失效来源

未核实：`DISALLOW_NETWORK_ACCESS`、`UserHandle.AID_ISOLATED_OFFSET`、`android:isolatedService`、`getOwnerType`、`setOrganizationOwner`、"organization owner"、"globally managed"、`app_domain`/`app-data-file-label`、`VirtualizationManager`、`pm create-managed-user`、`am multi-users`（以上均为源码/站点检索无匹配，已在正文列为纠正项）；media_codecs 的独立 seccomp 策略；USAP/App Zygote 池专属 CVE；WebView 沙箱逃逸的 Android 侧公告条目；`pm list users` 逐行输出格式；工作资料删除的更细清除范围；AOSP 对 clone profile 的机型/版本支持清单。

来源失效（实测）：`source.android.com/docs/security/features/seccomp`（404）；`source.android.com/docs/security/features/selinux/domains` 与 `/domains/app`（404）。

---

## 联系方式

咨询 Android 系统详情请联系 https://t.me/one00190

---

*本手册仅供安全学习与防御研究使用；所有常量名、类归属、公告字段与案例归属均取自公开一手来源并附链接，未能核实的表述已单列于 §9 而非写入正文。*
