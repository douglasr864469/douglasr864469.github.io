---
layout: page
title: "Android ADB、备份与数据迁移通道手册"
description: "adbd 授权判定与 RSA-2048/SHA-256 指纹真相、ro.adb.secure 的实际读取条件、无线调试配对引导与 7 天授权超时、fastboot 的 AOSP 与 OEM 命令边界、备份与 D2D 迁移的两条独立控制面（dataExtractionRules、bmgr、ro.backu..."
date: 2026-10-26
permalink: /android-adb-backup-transfer/
---

> USB 调试、备份框架、换机迁移这三件事常被当成"便利功能"，但它们共同构成设备上**唯一一条被产品化允许的数据外导通道**，而且授权状态、有效期与加密语义各有独立开关。本手册把这条通道拆开：adbd 怎么起、密钥怎么算、授权何时免除、备份由谁控制、哪些数据会被带到新设备上。定位是防御与教育，只引用公开且已修复的信息，不含可利用代码。

---

## 0. 使用方式与边界

**为什么单独成篇。** [通用漏洞类型手册](/android-common-system-vulnerabilities/) 把"调试与运维面"列为一类；本手册把它展开成可核对的机制细节，并与 [多用户与进程隔离手册](/android-multiuser-sandbox-isolation/) 对接——`DISALLOW_DEBUGGING_FEATURES` 是那条防线的企业侧，本手册是设备侧。

**怎么用。**
- 判断"这台设备的 adb 授权到底由什么决定" → §1（含 `ro.adb.secure` 的真实读取条件）
- 核对无线调试的配对与撤销语义 → §2
- 别把 fastboot 的 OEM 私有命令当标准 → §3
- 搞清 `adb backup` 现状、`bmgr`、`dataExtractionRules` 与 D2D 的差别 → §4
- 核对某条 ADB/备份 CVE 的归属 → §5，以及 §6 的误传纠正
- 在企业环境落地禁用与观测 → §7

**边界。** 不描述任何授权绕过、密钥提取或固件刷写的操作过程；不涉及第三方 root/恢复工具；不讨论主机侧工具链的攻击实现。

---

## 1. adbd：进程、密钥与授权的真实判定

**进程形状。** adbd 是 init 管理的独立守护进程，SELinux 域 `u:r:adbd:s0`，socket 定义 `adbd seqpacket 660 system system`（[init.usb.rc](https://android.googlesource.com/platform/system/core/+/refs/heads/main/rootdir/init.usb.rc)：原话 "service adbd /system/bin/adbd --root_seclabel=u:r:su:s0"）。

**密钥是 RSA 2048，指纹哈希在两侧不同——这是本手册最容易谈错的一处。**

| 侧 | 实名 | 出处 |
|---|---|---|
| adbd 密钥生成 | `#include <adb/crypto/rsa_2048_key.h>` / `CreateRSA2048Key()` | [daemon/auth.cpp](https://android.googlesource.com/platform/packages/modules/adb/+/refs/heads/main/daemon/auth.cpp) |
| adbd 指纹 | `SHA256(dkey, len, digest)` → **SHA256** | 同上 |
| 框架侧指纹 | `getFingerprints()` 使用 **MD5**、冒号分隔 | frameworks/base 侧 |
| 弹窗文案 | "Allow USB debugging?" / "The computer's RSA key fingerprint is:" / "Always allow from this computer" / "Allow" | [SystemUI strings.xml](https://android.googlesource.com/platform/frameworks/base/+/refs/heads/main/packages/SystemUI/res/values/strings.xml) |

实务结论：**你在弹窗里看到的指纹与用框架 API 取到的指纹不是同一种哈希，不能直接比对。** 讨论"如何核对主机身份"时必须先说明用的是哪一侧。

**授权文件。** 用户键 `/data/misc/adb/adb_keys`，出厂键 `/adb_keys`；目录 `mkdir /data/misc/adb 02750 system shell`（init.usb.rc）；文件权限 rw-r-----（`S_IRUSR|S_IWUSR|S_IRGRP`）；SELinux 标签 `adb_keys_file`（[private/file_contexts](https://android.googlesource.com/platform/system/sepolicy/+/refs/heads/main/private/file_contexts)：`/data/misc/adb(/.*)?  u:object_r:adb_keys_file:s0`）。

**`ro.adb.secure` 的真实语义**（这条被误解得最广）。源码是：

```
auth_required = GetBoolProperty("ro.adb.secure", false)
```

（[daemon/main.cpp](https://android.googlesource.com/platform/packages/modules/adb/+/refs/heads/main/daemon/main.cpp)）。要点三条：
1. **默认值是 `false`**；
2. 该属性**仅在"已解锁（verifiedbootstate=orange）或 debuggable"时才被读取**；
3. `auth_required` 初值为 `true`——**锁定的 user 版恒需授权**，改这个属性对它无效。

配套文档也印证了 developer 侧的自动化行为："In the case of `userdebug` and `eng`, properties set at build-time, such as `ro.adb.secure` … will automate adbd starting up and disable authentication."（[how adbd starts](https://android.googlesource.com/platform/packages/modules/adb/+/refs/heads/main/docs/dev/how_adbd_starts.md)）

**一个反向事实值得写进威胁模型**：解锁（orange）后 adbd **免除授权**（daemon/main.cpp）。也就是说"解锁引导加载"这一动作的安全代价不只在于可刷镜像，还在于调试通道语义改变。这条与 [TEE 手册](/android-tee-key-attestation/) 的四色状态一节互相引用。

**root 与 shell。** 官方手册只有 root/unroot 两个词："root: Restart adbd with root permissions"、"unroot: Restart adbd without root permissions"（[adb.1.md](https://android.googlesource.com/platform/packages/modules/adb/+/refs/heads/main/docs/user/adb.1.md)）；判定使用 `ro.secure` / `ro.debuggable` / `service.adb.root`（daemon/main.cpp 注释）。SELinux 域实名 `adbd`、`shell` 存在，`su` 域仅 `userdebug_or_eng`：[private/su.te](https://android.googlesource.com/platform/system/sepolicy/+/refs/heads/main/private/su.te) — "domain_auto_trans(shell, su_exec, su)"。**"unrooted shell" 不是官方术语。**

## 2. 授权有效期与无线调试

**可查设置键实名**（[Settings.java](https://android.googlesource.com/platform/frameworks/base/+/refs/heads/main/core/java/android/provider/Settings.java)）：

| 键 | 位置 | 说明 |
|---|---|---|
| `ADB_ENABLED = "adb_enabled"` | Global | 开发者选项里的 USB 调试开关 |
| `ADB_WIFI_ENABLED = "adb_wifi_enabled"` | Global | 无线调试 |
| `ADB_ALLOWED_CONNECTION_TIME = "adb_allowed_connection_time"` | Global | **默认 7 天自动撤销授权** |
| `USB_MASS_STORAGE_ENABLED` | Global | 大容量存储 |

`Settings.Secure` 侧只有 `usb_audio_automatic_routing_disabled`。**不存在 `usb_mode`，也不存在 `settings secure usb_*` 那一组键**（AOSP `Settings.java` 全量 grep 无对应实现）——网上流传的"用 settings 把 USB 切成仅充电"这类命令在这台系统上没有对应键。UI 侧实名是 **Default USB configuration** 与 "No data transfer"，官方提示语："When another device is connected and your phone is unlocked, these settings will be applied. Only connect to trusted devices."（[Settings strings.xml](https://android.googlesource.com/platform/packages/apps/Settings/+/refs/heads/main/res/values/strings.xml)）

 gadget 组合实际由 `sys.usb.config` / `persist.sys.usb.config` 决定，属于系统属性而非 Settings 键——这两层别混。

**无线调试的信任引导**（[adb_wifi.md](https://android.googlesource.com/platform/packages/modules/adb/+/refs/heads/main/docs/dev/adb_wifi.md)）：

> "Trust is bootstrapped using a shared secret, seeded either by a six-digit number (pairing code) or a 10-digit number (QR code pairing)"
> "`_adb-tls-pairing._tcp`"

命令实名 `adb pair HOST[:PORT] [PAIRING_CODE]`（adb.1.md）；用户可见开关是 "Wireless debugging"；撤销入口 "Revoke adb debugging authorizations"。依赖关系也有一句成文："adb relies on mDNS to automatically discover and connect to paired devices"（[developer.android.com/tools/adb](https://developer.android.com/tools/adb)）。

安全语义上的关键点：**配对码是短熵共享秘密**（六位/十位数字），因此"配对窗口开放"本身就是一个需要控制的暴露面，而不是"配过就长期安全"。超时与撤销见 `adb_allowed_connection_time` 与开关 "Disable adb authorization timeout"（Settings strings）。

## 3. fastboot 边界：哪些是 AOSP 定义，哪些是 OEM 私有

**`fastboot oem device-info` 不是 AOSP 定义命令。** AOSP 只有 `oem [COMMAND...] Execute OEM-specific command`（[fastboot.cpp](https://android.googlesource.com/platform/system/core/+/refs/heads/main/fastboot/fastboot.cpp)），设备信息是否返回、返回什么完全取决于 OEM 实现。`fastboot getvar all` 能拿到锁状态同样属 OEM 依赖（未核实通用性）。

AOSP 定义的解锁接口与官方要求（[locking_unlocking](https://source.android.com/docs/core/architecture/bootloader/locking_unlocking)）：

> "Devices should deny the `fastboot flashing unlock` command unless the `get_unlock_ability` is set to 1 … enable the OEM unlocking option"
> "perform a factory data reset to prevent unauthorized data access"

即解锁判定用 `flashing unlock` / `flashing get_unlock_ability`，且官方要求解锁伴随**工厂数据重置**。这与 [TEE 手册](/android-tee-key-attestation/) 里 `deviceLocked` / `verifiedBootState` 的证明语义直接对接：解锁改变的不只是可刷性，还有证明链里的两个字段。

## 4. 备份、恢复与换机迁移

**现状表述要精确。** `adb backup` **未被官方宣布移除**：Android 12 起默认排除应用数据（[behavior-changes-12](https://developer.android.com/about/versions/12/behavior-changes-12)）：

> "For apps that target Android 12 … app data is excluded from any other system data that is exported from the device"
> "you can now opt in … by setting `android:debuggable` to `true`"

同时 AOSP main 仍实现 `adbBackup()/adbRestore()`（[BackupManagerService.java](https://android.googlesource.com/platform/frameworks/base/+/refs/heads/main/services/backup/java/com/android/server/backup/BackupManagerService.java)），但现行 adb 手册页里已经没有 backup 条目。**所以准确说法是"行为被限制、文档中不再出现"，不是"官方移除"（后者无成文出处）。**

**类与属性实名**：`BackupAgent`、`BackupAgentHelper`、`onFullBackup`、`android:fullBackupOnly`；`android:allowBackup` 的官方说明 "set the boolean value `android:allowBackup` … The default value is `true`"（[autobackup](https://developer.android.com/guide/topics/data/autobackup)）。默认 true 这一点对数据外泄面评估很关键。

**控制面**。Android 12（API 31）起用 **`android:dataExtractionRules`**，内部分段为 `<cloud-backup>`、`<device-transfer>`、`<cross-platform-transfer>`；`fullBackupContent` 在新设备上被忽略。设备级开关实名 **`ro.backup.disable`**（BackupManagerService.java 第 122 行）。**没有 `ro.backup.allow_adb_backup`**。

**命令与传输实名**。shell 命令实名是 **`bmgr`（不是 `bm`）**：

```
usage: bmgr [--user <userId>] [backup|restore|list|transport|run]
```

另有 `enable` / `init` / `backupnow`（[Bmgr.java](https://android.googlesource.com/platform/frameworks/base/+/refs/heads/main/cmds/bmgr/src/com/android/commands/bmgr/Bmgr.java)）。服务名 `Context.BACKUP_SERVICE = "backup"`；传输实名 `com.android.localtransport/.LocalTransport`、`com.google.android.gms/.backup.BackupTransportService`，D2D 用 `com.google.android.gms/.backup.migrate.service.D2dTransport`（[testingbackup](https://developer.android.com/guide/topics/data/testingbackup)）。

**D2D 与云备份不是一条开关。** 官方明确差异（behavior-changes-12）："Device-to-device (D2D) transfers: User data is sent directly to the user's new device …"，以及 **"specifying `android:allowBackup="false"` does disable backups to Google Drive, but doesn't disable D2D transfers"**（部分厂商行为）。这条是数据外泄面梳理里最常见的漏项：只关了云备份不等于关了迁移。未核实：D2D 会带走哪些类别数据的官方清单（帮助页正文为折叠 DOM，抓取未渲染出条目）。

**配对类旁路**。Fast Pair 的认证语义是一手的："The Seeker verifies that the Provider is in possession of the anti-spoofing private key."（[Fast Pair characteristics](https://developers.google.com/nearby/fast-pair/specifications/characteristics)）。权限实名现行要求见 [蓝牙权限](https://developer.android.com/develop/connectivity/bluetooth/permissions)：`BLUETOOTH_SCAN`、`BLUETOOTH_CONNECT`、`ACCESS_FINE_LOCATION`，及 `usesPermissionFlags="neverForLocation"`。Companion Device Manager 的免定位扫描成文为："performs a Bluetooth or Wi-Fi scan … without requiring the `ACCESS_FINE_LOCATION` permission"（[companion-device-pairing](https://developer.android.com/guide/topics/connectivity/companion-device-pairing)）。

注意：**`com.android.fastpair` 不是 AOSP 包名**（Fast Pair 属 Google 服务文档）；`developers.google.com/nearby/fast-pair/specifications` 根路径 404，仅 `/characteristics`、`/configuration` 可访问。

**加密现状的两种说法要分清**。开发者文档："The backup is end-to-end encrypted on devices running Android 9 or higher using the device's PIN, pattern, or password."；用户支持文档："Your backups are uploaded to Google and encrypted with your Google Account password."（[Google One 备份](https://support.google.com/googleone/answer/9149304)）——前者讲设备侧端到端，后者讲上传侧账户加密，不是同一件事。`adb backup` 口令在 AOSP 中仅以 PBKDF2 哈希保存："Does not persist the password itself, but persists a PBKDF2 hash"（BackupPasswordManager.java）；**`adb backup` 输出本身是否加密，官方无成文表述**（未核实）。

## 5. 已修复公开案例（描述以 NVD/公告为准）

| CVE | 落点（实名文件/方法） | 归属与级别 |
|---|---|---|
| CVE-2026-0073 | `adbd/auth.cpp` — "possible bypass of wireless ADB mutual authentication"，邻近远程 RCE（shell 用户） | Android 14/15/16/16-qpr2，严重(RCE)，A-469080888，[2026-05-01](https://source.android.com/docs/security/bulletin/2026/2026-05-01)（[NVD](https://nvd.nist.gov/vuln/detail/CVE-2026-0073)） |
| CVE-2021-39794 | `AdbService.java` / `broadcastPortInfo` — 开启无线调试时应用可以 shell 用户执行代码 | A-205836329，Android 11/12/12L，HIGH（[NVD](https://nvd.nist.gov/vuln/detail/CVE-2021-39794)） |
| CVE-2021-39700 | `adbd.te` 策略逻辑错误 → 本地信息泄露 | A-201645790，Android 10/11/12，中等，[2022-05-01](https://source.android.com/docs/security/bulletin/2022-05-01)（[NVD](https://nvd.nist.gov/vuln/detail/CVE-2021-39700)） |
| CVE-2023-35689 | `DeviceVersionFragment` / `checkDebuggingDisallowed` — SUW 完成前可访问 adb（不安全默认值） | （[NVD](https://nvd.nist.gov/vuln/detail/CVE-2023-35689)） |
| CVE-2021-0554 | `isBackupServiceActive` of BackupManagerService.java 缺权限检查 → 本地信息泄露 | A-158482162，Android 11（[NVD](https://nvd.nist.gov/vuln/detail/CVE-2021-0554)） |
| CVE-2023-21387 | User Backup Manager 令牌泄露，可**绕过备份用户确认** | A-280296227，Android 14，中等（[Android 14 安全更新](https://source.android.com/docs/security/bulletin/android-14)） |
| CVE-2023-21035 | `BackupHelper.java` 权限绕过（同名包） | A-184847040，Android 13（[NVD](https://nvd.nist.gov/vuln/detail/CVE-2023-21035)） |
| CVE-2017-13163 | "kernel mtp usb driver" 提权 | A-37429972，Android kernel（[NVD](https://nvd.nist.gov/vuln/detail/CVE-2017-13163)） |
| CVE-2024-57913 | Linux kernel `usb: gadget: f_fs`（functionfs_bind） | 上游内核 CVE，**非 Android 公告条目**（[NVD](https://nvd.nist.gov/vuln/detail/CVE-2024-57913)） |
| CVE-2021-0933 | `CompanionDeviceActivity` / `DeviceChooserActivity`，HTML 标签干扰同意弹窗 — "confusing the user into accepting pairing of a malicious Bluetooth device" | A-172251622，Android 9–12，HIGH（[NVD](https://nvd.nist.gov/vuln/detail/CVE-2021-0933)） |

**这一组的形态分布值得单独读。** ADB/备份通道的缺陷并不集中在"解析远端数据"，而是三类：授权判定与策略（CVE-2026-0073、CVE-2021-39700）、**同意/确认语义被削弱**（CVE-2021-0933、CVE-2023-21387）、不安全默认值与设置向导期（CVE-2023-35689）。第二类和第三类在纯代码审计里容易漏，因为它们的表现是"逻辑正确但语义不对"。

**KEV 逐条核对**（目录版本 2026.10.23）：上述全部条目 `?field_cve=` 检索**均不在 KEV**。KEV 中 Android 条目共 16 条，Product 仅为 Framework/Runtime/Kernel/Pixel，**无一条属 ADB/备份/USB/Fast Pair 通道**。跨厂商碰撞提醒：**CVE-2025-34251 是 Tesla TCU 的 adbd，不是 Google Android**，不可引为 Android 基带/ADB 案例。出处：[KEV 目录](https://www.cisa.gov/known-exploited-vulnerabilities-catalog)。

## 6. 常见误传纠正

| 误传 | 实际 |
|---|---|
| adbd 弹窗显示 MD5 指纹 | adbd 侧用 **SHA256**；MD5 是框架侧 `getFingerprints()`，两者不可比对 |
| adb 授权用 4096 位或其他密钥 | `CreateRSA2048Key()`，RSA 2048 |
| `ro.adb.secure=true` 就能锁定任何设备 | 默认 false，且仅在 orange/debuggable 时被读取；锁定 user 版恒需授权 |
| 解锁只影响能不能刷镜像 | 解锁（orange）后 adbd **免除授权** |
| `settings secure usb_mode=…` 可切仅充电 | 无 `usb_mode`、无 `settings secure usb_*`；用 `sys.usb.config` / UI 的 Default USB configuration |
| `fastboot oem device-info` 是标准命令 | AOSP 只定义 `oem [COMMAND...]`；解锁判定用 `flashing unlock` / `flashing get_unlock_ability` |
| `adb backup` 已被官方移除 | 无移除公告；Android 12 起默认排除应用数据，手册页不再列出 |
| 备份命令是 `bm` | 实名 **`bmgr`** |
| 设备级开关是 `ro.backup.allow_adb_backup` | 实名 `ro.backup.disable` |
| `allowBackup=false` 就切断全部外导 | 官方明确它不切断 **D2D** 迁移；要用 `dataExtractionRules` 分段控制 |
| 云备份端到端加密与上传加密是一回事 | 两句官方表述分别是"设备 PIN/图案/密码端到端"与"Google 账户密码加密上传" |
| adb root 有个官方叫法 "unrooted shell" | 手册只有 root/unroot；SELinux 域是 `adbd`、`shell`，`su` 仅 userdebug_or_eng |
| CVE-2024-57913 是 Android 公告条目 | 上游 Linux 内核 gadget 驱动 CVE |
| CVE-2025-34251 说明 Android adbd 被在野利用 | Tesla TCU 的 adbd，跨厂商碰撞 |
| Fast Pair 属 AOSP，包名 `com.android.fastpair` | 属 Google 服务文档；AOSP 无该包名 |

## 7. 观测点与防御动作

**可读实况。**

```
adb get-state                       # "Print offline | bootloader | device"（adb.1.md）
adb shell settings get global adb_enabled
adb shell dumpsys adb               # "ADB MANAGER STATE (dumpsys adb)"
```

授权撤销窗口相关：`adb_allowed_connection_time`（默认 7 天）与开关 "Disable adb authorization timeout"。文件侧：`/data/misc/adb/adb_keys`（02750 目录、rw-r----- 文件、`adb_keys_file` 标签）。

**企业侧成文禁用。** `UserManager.DISALLOW_DEBUGGING_FEATURES = "no_debugging_features"`，官方成文："it disables debugging features altogether, including USB debugging"（[UserManager.java](https://android.googlesource.com/platform/frameworks/base/+/refs/heads/main/core/java/android/os/UserManager.java)）；`AdbService` 侧会捕获 SecurityException 并给出 "ADB_ENABLED is restricted."（[AdbService.java](https://android.googlesource.com/platform/frameworks/base/+/refs/heads/main/services/core/java/com/android/server/adb/AdbService.java)）。

**自查清单。**
1. 枚举 `adb_enabled` 与 `adb_wifi_enabled` 的当前值，以及 `adb_keys` 条目数——**长期存在的授权主机是关键风险项**。
2. 确认 `adb_allowed_connection_time` 未被改成无限期（"Disable adb authorization timeout" 是否开启）。
3. 无线调试默认关闭；配对窗口只在维护期开放（短熵引导码是信任起点）。
4. 核对 verifiedbootstate：orange 设备上 adb 免授权，应按"调试通道常开"建模。
5. 应用侧：`allowBackup` 默认 true，逐个确认；用 `dataExtractionRules` 的三段分别收敛云备份/设备迁移/跨平台迁移。
6. 明确 `allowBackup=false` 不等于 D2D 关闭，迁移面需单独评估。
7. SUW（开机向导）期间的调试入口状态是历史缺陷点（CVE-2023-35689），纳入新设备验收。
8. Companion Device Manager 的同意弹窗语义要审计（CVE-2021-0933 型），别只看权限声明。
9. 企业策略：`DISALLOW_DEBUGGING_FEATURES` 是成文手段；同时检查 userdebug/eng 构建是否混入生产（`ro.debuggable`）。
10. 内核 gadget/USB 驱动与上游内核 CVE 分开跟踪（CVE-2024-57913 型条目不在 Android 公告里）。

## 8. 与其他手册的分工

| 需要 | 去看 |
|---|---|
| `DISALLOW_DEBUGGING_FEATURES` 所在的用户/策略体系 | [多用户与进程隔离手册](/android-multiuser-sandbox-isolation/) |
| 解锁状态、`deviceLocked`、Verified Boot 四色与证明字段 | [TEE 与密钥认证手册](/android-tee-key-attestation/) |
| 凭据与 CE 数据密钥的关系（备份端到端加密的上游） | [生物识别与锁屏手册](/android-biometric-lockscreen/) |
| 蓝牙/Wi‑Fi 协议缺陷归属 | [底层漏洞类型图谱](/android-low-level-vulnerability-types/) |
| 定级与处置优先级 | [评估与定级手册](/android-vulnerability-severity-assessment/) |
| 本文 | adbd 与授权判定、无线调试信任引导、fastboot 边界、备份与 D2D 控制面、ADB/备份/USB 通道案例归属 |

---

## 参考

- [adb.1.md](https://android.googlesource.com/platform/packages/modules/adb/+/refs/heads/main/docs/user/adb.1.md) · [adb_wifi.md](https://android.googlesource.com/platform/packages/modules/adb/+/refs/heads/main/docs/dev/adb_wifi.md) · [how_adbd_starts.md](https://android.googlesource.com/platform/packages/modules/adb/+/refs/heads/main/docs/dev/how_adbd_starts.md) · [daemon/auth.cpp](https://android.googlesource.com/platform/packages/modules/adb/+/refs/heads/main/daemon/auth.cpp) · [daemon/main.cpp](https://android.googlesource.com/platform/packages/modules/adb/+/refs/heads/main/daemon/main.cpp)
- [init.usb.rc](https://android.googlesource.com/platform/system/core/+/refs/heads/main/rootdir/init.usb.rc) · [private/file_contexts](https://android.googlesource.com/platform/system/sepolicy/+/refs/heads/main/private/file_contexts) · [private/su.te](https://android.googlesource.com/platform/system/sepolicy/+/refs/heads/main/private/su.te) · [Settings.java](https://android.googlesource.com/platform/frameworks/base/+/refs/heads/main/core/java/android/provider/Settings.java) · [SystemUI strings.xml](https://android.googlesource.com/platform/frameworks/base/+/refs/heads/main/packages/SystemUI/res/values/strings.xml) · [Settings strings.xml](https://android.googlesource.com/platform/packages/apps/Settings/+/refs/heads/main/res/values/strings.xml) · [AdbService.java](https://android.googlesource.com/platform/frameworks/base/+/refs/heads/main/services/core/java/com/android/server/adb/AdbService.java)
- [fastboot.cpp](https://android.googlesource.com/platform/system/core/+/refs/heads/main/fastboot/fastboot.cpp) · [locking_unlocking](https://source.android.com/docs/core/architecture/bootloader/locking_unlocking)
- [BackupManagerService.java](https://android.googlesource.com/platform/frameworks/base/+/refs/heads/main/services/backup/java/com/android/server/backup/BackupManagerService.java) · [Bmgr.java](https://android.googlesource.com/platform/frameworks/base/+/refs/heads/main/cmds/bmgr/src/com/android/commands/bmgr/Bmgr.java) · [autobackup](https://developer.android.com/guide/topics/data/autobackup) · [testingbackup](https://developer.android.com/guide/topics/data/testingbackup) · [Android 12 behavior changes](https://developer.android.com/about/versions/12/behavior-changes-12) · [Google One 备份加密](https://support.google.com/googleone/answer/9149304)
- [Fast Pair characteristics](https://developers.google.com/nearby/fast-pair/specifications/characteristics) · [蓝牙权限](https://developer.android.com/develop/connectivity/bluetooth/permissions) · [Companion Device Pairing](https://developer.android.com/guide/topics/connectivity/companion-device-pairing) · [developer.android.com/tools/adb](https://developer.android.com/tools/adb)
- [公告 2026-05-01](https://source.android.com/docs/security/bulletin/2026/2026-05-01) · [Android 14 安全更新](https://source.android.com/docs/security/bulletin/android-14) · [CISA KEV 目录](https://www.cisa.gov/known-exploited-vulnerabilities-catalog)

## 9. 未核实与失效来源

未核实："unrooted shell" 术语与 `adb_root`/`adb_unauthorized` 域名；`settings secure usb_*`、`usb_mode`；`ro.backup.allow_adb_backup`；`adb backup` 的官方废弃声明与日期；`fastboot oem device-info` 的通用性与 `getvar all` 是否返回锁状态；AOSP 内 Fast Pair 服务包名；`com.android.backup` 作为服务/进程实名、旧 Google BackupTransport 的源码实名；D2D 迁移数据类别官方清单；`adb backup` 输出是否加密。

来源失效（实测）：`source.android.com/docs/security/bulletin/2022/2022-05-01`（404，正确形式 `/bulletin/2022-05-01`）；`frameworks/base/services/core/java/com/android/server/backup/transport/LocalTransport.java`（各分支 404，已迁至 `packages/LocalTransport`）；`developers.google.com/nearby/fast-pair/specifications`（根路径 404）。

---

## 联系方式

咨询 Android 系统详情请联系 https://t.me/one00190

---

*本手册仅供安全学习与防御研究使用；所有属性名、文件路径、权限位与案例归属均取自公开一手来源并附链接，未能核实的表述已单列于 §9 而非写入正文。*
