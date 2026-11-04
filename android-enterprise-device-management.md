---
layout: page
title: "Android 企业设备管理、托管配置与工作资料攻击面手册"
description: "把「企业管控」拆成角色／权限／配置／资料／策略五层：legacy admin 只执行 XML 声明策略、Device Owner 仅能在初始化阶段设定、`PROVISION_DEVICE_OWNER` 权限不存在（用 `MANAGE_PROFILE_AND_DEVICE_OWNERS` 或 shell、..."
date: 2026-11-04
permalink: /android-enterprise-device-management/
---

> 企业移动管理（EMM/MDM）是 Android 上权限最集中的一块：一个被设为 Device Owner 的应用，可以在不弹用户确认的情况下配置 Wi-Fi、安装证书、禁用相机、强制密码策略、静默安装应用。它之所以能这样做，是因为平台把「设备所有者」定义为一个受特殊保护的角色；而围绕这个角色的配置流程、跨资料调用与策略下发，又长期是权限提升与隔离绕过的温床。这份手册沿「角色怎么获得 → 策略怎么下发 → 资料怎么隔离 → 配置怎么被滥用 → 怎么防」拆解机制、公开缺陷与防御。只讨论公开且已修复的问题，不含可利用代码。

## 0. 使用方式与边界

这份手册面向三类读者：

- **企业应用开发者 / DPC 开发者**：需要理解 Device Owner 与 Profile Owner 的能力差异、配置流程在 Android 12 之后的变化、以及哪些策略在 Android 9/10 已废弃。
- **系统与平台工程师**：需要区分 `DevicePolicyManager`（策略接口）、`DeviceAdminReceiver`（legacy 管理入口）、`UserManager`（用户限制）与 `CrossProfileApps`（跨资料启动）四组组件。
- **安全评估者**：需要在看到「企业/托管」公告时，判断问题落在权限检查、配置消息、跨资料调用还是策略状态机，而不是把所有带 `device owner` 的 CVE 混为一类。

本文的三条自我约束：

1. 只引用公开且已修复的问题；CVE 编号、落点与描述以 NVD 为准，公告分类与级别以 Android Security Bulletin 为准。
2. 不写利用步骤、不写 PoC、不写可复现的配置参数。
3. 查不到一手来源的说法一律进「未核实与失效来源」。

与已发布手册的分工：

- 《Android 多用户、工作资料与沙箱隔离手册》讲用户与资料的底层隔离；本文只讲企业角色如何在这个隔离之上施加策略。
- 《Android 账户、凭据与无密码认证手册》讲用户凭据；本文只讲设备策略对密码复杂度、认证方式的要求。
- 《Android 通知、状态栏与通知监听攻击面手册》讲通知可见性；本文只把「工作资料通知」作为跨资料策略的一个例子。

## 1. 企业管理的角色模型

Android 的企业管理有三层角色，能力逐级增强：

| 角色 | 获得方式 | 作用范围 | 典型能力 |
|------|----------|----------|----------|
| Legacy Device Admin | 用户通过 `ACTION_ADD_DEVICE_ADMIN` 授权 | 整机（受 XML 声明限制） | 擦除、锁屏、密码策略、禁用相机 |
| Profile Owner | 配置工作资料时设定 | 单个用户/资料 | 资料内策略、应用管理、证书 |
| Device Owner | 设备初始化时设定 | 整机与所有用户 | 全局策略、静默安装、用户管理 |

关键点是：**legacy admin 不是「弱一点的 Device Owner」，而是另一套模型**。它只能执行在 `res/xml` 中声明的策略（`USES_POLICY_*`），用户授权后生效；它不能静默安装任意应用，也不能管理其他用户。把三者混为一谈，会高估 legacy admin 的能力，也会低估 Device Owner 的风险。

第二个关键点是：**Device Owner 只能在「没有已配置账户」的设备初始化阶段设定**。它不能由普通应用在运行中自行升级获得；量产配置依赖 QR、NFC 或 zero-touch，测试才用 `dpm set-device-owner`。

## 2. 对象模型与权限边界

### 2.1 关键对象

| 对象 | 所在层 | 作用 |
|------|--------|------|
| `DevicePolicyManager`（DPM） | 公开 SDK | 策略设置、查询、委托 |
| `DeviceAdminReceiver` | 公开 SDK | legacy admin 的广播接收器 |
| `DeviceAdminInfo` | 公开 SDK | 解析 admin 的 XML 声明与 `USES_POLICY_*` |
| `DevicePolicyManager#setDeviceOwner` | `@SystemApi` | 设定 Device Owner |
| `CrossProfileApps` | 公开 SDK（API 28） | 跨资料启动调用者的 Activity |
| `RestrictionsManager` | 公开 SDK | 读取托管配置（application restrictions） |
| `UserManager` | 公开 SDK | 用户限制（`DISALLOW_*`）与用户管理 |
| `ProvisioningMode` / 配置 Intent | 公开 SDK | QR/NFC/zero-touch 的配置消息 |

### 2.2 权限真相

企业框架的权限经常被记错，下面几条是核对过的：

- `BIND_DEVICE_ADMIN` 的保护级别是 `signature|role`：它限制谁可以被系统绑定为 admin，不是「只有系统应用能当 admin」。
- `MANAGE_DEVICE_ADMINS` 是 `@SystemApi`，第三方应用不可用。
- `setDeviceOwner` / `setProfileOwner` 需要 `MANAGE_PROFILE_AND_DEVICE_OWNERS`（系统权限）或 shell 身份。
- **不存在 `PROVISION_DEVICE_OWNER` 权限**：在 Android 5.0–16 的清单与公开权限页中都查不到；配置消息使用的是 `DISPATCH_PROVISIONING_MESSAGE`。
- `DISALLOW_*` 常量定义在 `android.os.UserManager`，不是 `DevicePolicyManager`；DPM 的 `addUserRestriction()` 只是把 key 传给用户限制系统。

把这些记错，会导致两个方向的误判：要么以为普通应用「申请个权限就能当 Device Owner」，要么以为「Device Owner 什么都能做」——实际上它仍受系统权限、SELinux 与用户边界约束。

### 2.3 委托（delegation）

DPM 支持把部分能力委托给其他应用，例如证书安装（`DELEGATION_CERT_INSTALL`）、应用管理、网络日志等。委托是「缩小权限面」的机制，但也是常见的配置错误来源：委托范围过大，等于把 Device Owner 的能力分发给了不受同等保护的应用。

## 3. 配置流程：从 QR 到 zero-touch

### 3.1 测试用命令

`dpm set-device-owner` 是**测试命令**，不是量产方案。它要求设备已恢复出厂、没有已配置账户，并通过 adb shell 执行。它的存在常被误读为「应用可以自己调用」——实际上普通应用没有 shell 身份，也无法满足前置条件。

### 3.2 量产配置

- **QR 配置**：设备初始化向导扫描二维码，读取配置载荷；
- **NFC 配置**：通过 `MIME_TYPE_PROVISIONING_NFC` 传递配置；
- **zero-touch**：由授权经销商与 GMS 兼容设备支持，开机自动下发配置。

这些流程的共同点是：**配置发生在「设备还没有用户数据」的阶段**，因此可以把 Device Owner 设定为受信 DPC。配置消息本身是外部输入，必须校验来源与完整性；否则攻击者可以在配置阶段注入自己的 DPC。

### 3.3 Android 12 的变化

Android 12 起，旧的 `ACTION_PROVISION_MANAGED_DEVICE` 流程失效，改为：

1. `ACTION_GET_PROVISIONING_MODE`：让配置方选择「整机托管」还是「工作资料」；
2. `ACTION_ADMIN_POLICY_COMPLIANCE`：在选定模式后执行管理策略。

这个变化的意义是**把「选择模式」与「执行策略」拆成两步**，让配置方能根据设备状态与用户意图做决定，而不是由一个 Intent 直接决定整机归属。

## 4. 工作资料与跨资料边界

### 4.1 工作资料是什么

工作资料（managed profile）在 Android 5.0（API 21）引入，官方明确它**以 secondary user 的形式实现**。它拥有独立的存储、独立的账户、独立的加密上下文；Profile Owner 只能管理这个资料，不能读取个人资料的数据。

这一点是很多误传的根源：**Profile Owner 不是「能看两个资料的超级管理员」**，它只是「工作资料的管理员」。跨资料访问必须经过显式通道。

### 4.2 `CrossProfileApps`

API 28 引入的 `CrossProfileApps` 允许应用在另一个资料中启动**自己的** Activity。它需要：

- `INTERACT_ACROSS_PROFILES` 权限（或由 DPC 通过 `setCrossProfilePackages` 配置）；
- 用户同意；
- 目标 Activity 属于调用者自身。

它不能用来启动其他应用的任意组件，也不能读取对方资料的数据。把 `CrossProfileApps` 当成「跨资料任意跳转」是高估了它的能力。

### 4.3 通知与剪贴板

工作资料与个人资料的通知、剪贴板默认是隔离的。DPC 可以通过策略调整部分行为，但每次放开都是一次显式的信任决定。评估时要检查：哪些跨资料通道被打开、由谁打开、是否有用户可见的提示。

## 5. 托管配置（managed configurations）

应用可以通过 `RestrictionsManager.getApplicationRestrictions()` 读取管理员下发的配置。关键边界：

- 应用只能读**自己**的限制，不能读其他应用的；
- 变更广播 `Intent.ACTION_APPLICATION_RESTRICTIONS_CHANGED` 只由系统发送，应用不能伪造；
- API 34 增加 `getApplicationRestrictionsPerAdmin()`，用于区分不同 admin 下发的配置。

托管配置常被用来下发服务器地址、功能开关、证书别名等。它是「配置通道」，不是「权限通道」；应用不应把敏感授权决策建立在可由管理员任意修改的配置上，除非这正是企业策略的本意。

## 6. 策略的废弃与兼容

Android 9（API 28）起，四项 legacy 策略被废弃：禁用相机、锁屏、密码质量、密码长度相关的一组接口。Android 10（API 29）起，对 target 10 的应用调用这些接口会抛 `SecurityException`；部分接口在 Android 11 起被静默忽略。

这带来一个典型的兼容陷阱：**同一段代码在不同 targetSdk 上行为不同**。评估时看到「策略没生效」，先确认 targetSdk 与接口废弃状态，而不是直接怀疑厂商实现。

另外，`setCameraDisabled` 对 legacy admin 需要 XML 中声明 `USES_POLICY_DISABLE_CAMERA`；`USES_POLICY_*` 是 `DeviceAdminInfo` 的常量，不是权限。把常量当权限申请，是清单配置里常见的错误。

## 7. 公开公告台账

| CVE | NVD 落点 / 类型 | 级别 | 修复 | 分类 |
|-----|----------------|------|------|------|
| [CVE-2021-0568](https://nvd.nist.gov/vuln/detail/CVE-2021-0568) | `onReceive` 缺权限检查（EoP） | NVD 7.8 / 公告 Moderate | 2021-06 | Pixel Framework |
| [CVE-2022-20138](https://nvd.nist.gov/vuln/detail/CVE-2022-20138) | `ACTION_MANAGED_PROFILE_PROVISIONED` 缺权限检查（EoP） | High | 2022-06 | ASB Framework |
| [CVE-2025-48633](https://nvd.nist.gov/vuln/detail/CVE-2025-48633) | `hasAccountsOnAnyUser` 逻辑错误（ID） | NVD 5.5 / 公告 High | 2025-12 | ASB Framework |
| [CVE-2026-58834](https://nvd.nist.gov/vuln/detail/CVE-2026-58834) | `setPermissionGrantState` 校验不当（DoS） | NVD 5.5 / 公告 High | 2026-10 | ASB Framework |

公告链接：

- Pixel 2021-06：https://source.android.com/security/bulletin/pixel/2021-06-01
- 2022-06：https://source.android.com/security/bulletin/2022-06-01
- 2025-12：https://source.android.com/security/bulletin/2025-12-01
- 2026-10：https://source.android.com/docs/security/bulletin/2026/2026-10-01

读表要点：

1. **都是 DPM 企业框架问题**，不是多用户隔离缺陷；但它们的共同背景是「企业角色拥有高权限，任何检查遗漏都会被放大」。
2. **级别存在 NVD 与公告不一致**：CVE-2021-0568、CVE-2025-48633、CVE-2026-58834 都有差异，表中同时列出。
3. **影响类型不只是提权**：CVE-2025-48633 是信息泄露，CVE-2026-58834 是拒绝服务。

## 8. 成因模式

### 8.1 广播接收器缺权限检查

企业框架大量使用广播传递状态（配置完成、资料创建、策略变更）。如果 `onReceive` 未校验发送方权限，任何应用都能伪造状态，触发本不该执行的逻辑（CVE-2021-0568、CVE-2022-20138）。修法通常是：发送方加签名权限，接收方校验 `getCallingUid()` 或使用显式组件。

### 8.2 高权限接口的调用者校验

`setPermissionGrantState` 一类接口只应由特定角色调用。若校验逻辑不完整（例如只检查包名、不检查 uid/用户），就会出现越权或异常输入导致的崩溃（CVE-2026-58834）。

### 8.3 状态查询的逻辑错误

`hasAccountsOnAnyUser` 这类「跨用户查询」接口，返回值会影响配置决策。逻辑错误会让调用方得到错误的账户状态，进而做出错误授权（CVE-2025-48633）。这类问题的修复往往只是改一个条件，但影响面取决于谁依赖这个返回值。

### 8.4 配置消息的外部输入

QR/NFC/zero-touch 的配置载荷来自设备外部。如果解析时未校验签名、版本、字段范围，就可能被注入恶意配置。防御上，配置消息必须验签、限定来源，并在应用前做完整性检查。

### 8.5 资料边界假设错误

Profile Owner 只能管理自己的工作资料；如果代码假定「我是 admin 所以能读个人资料」，就会越界。跨资料访问必须走 `CrossProfileApps` 或其他显式授权通道，不能靠文件路径或内容提供者猜测。

### 8.6 废弃接口的兼容分支

同一接口在不同 targetSdk 上行为不同，容易让开发者写出「在老设备上有效、在新设备上静默失败」的代码。安全影响是策略看似生效、实际未生效，形成防护缺口。

## 9. 版本加固时间线

| 版本 / API | 变化 |
|------------|------|
| 5.0 / 21 | 工作资料、Device Owner 引入 |
| 7.0 / 24 | QR 配置；工作资料安全挑战 |
| 9 / 28 | 四项 legacy 策略废弃；`CrossProfileApps`；fully-managed 设备 |
| 10 / 29 | 四项策略对 target 10 抛 `SecurityException`；`PROVISIONING_MODE_MANAGED_PROFILE` |
| 11 / 30 | 统一 COPE；`setCrossProfilePackages`；`resetPassword` 废弃 |
| 12 / 31 | 旧 provisioning Intent 失效；传感器权限接口废弃 |
| 13 / 33 | 默认需联网配置；离线需 `EXTRA_PROVISIONING_ALLOW_OFFLINE` |
| 14 / 34 | 跨资料联系人策略改名；新增 `DISALLOW_ULTRA_WIDEBAND_RADIO` |
| 15 / 35 | private space 管控、托管 eSIM、工作资料阻止 Circle to Search |

趋势是**企业能力从「整机一把抓」转向「按资料、按用途、按策略细分」**：COPE、private space、托管 eSIM 都是这个方向的产物。能力越细，配置错误的组合也越多。

## 10. 观测点与自查

### 10.1 DPC / 企业应用侧

- 是否在 `onReceive` 中校验发送方？是否使用显式组件与签名权限？
- 配置载荷是否验签、校验版本与字段范围？
- 是否使用了已废弃的策略接口？targetSdk 升级后是否重新验证？
- 委托范围是否最小？是否把 `DELEGATION_CERT_INSTALL` 等能力给了不必要的应用？
- 跨资料启动是否只启动自己的 Activity？是否处理了用户拒绝？

### 10.2 平台侧

- `dpm list-owners` / `dumpsys device_policy` 查看 owner、admin 与委托关系；
- 检查工作资料的创建来源与 Profile Owner；
- 审计配置广播的发送方权限；
- 关注 `DevicePolicyManagerService` 的补丁级别与公告。

### 10.3 评估侧

- 先定位落点：DPM 接口、广播接收、配置流程、跨资料通道、托管配置；
- 区分「提权」「信息泄露」「拒绝服务」「策略绕过」；
- 不要把所有企业相关 CVE 都归为「多用户漏洞」——它们多在 DPM 框架内。

## 11. 防御要点

1. **配置消息验签**：QR/NFC/zero-touch 载荷必须验证来源与完整性，拒绝未知字段与越界值。
2. **广播最小暴露**：企业状态广播使用签名权限或显式组件；接收方校验发送方身份。
3. **委托最小化**：只委托必要能力，定期审计委托列表。
4. **资料边界不猜**：跨资料访问走 `CrossProfileApps` 等显式接口，不靠路径或隐式 Intent。
5. **废弃接口迁移**：按 targetSdk 逐一核对策略接口状态，避免「静默失效」的防护缺口。
6. **权限常量别记混**：`DISALLOW_*` 在 `UserManager`，`USES_POLICY_*` 在 `DeviceAdminInfo`，`PROVISION_DEVICE_OWNER` 不存在。
7. **测试用命令不上量产**：`dpm set-device-owner` 只用于实验环境；量产走官方配置流程。

## 12. 常见误传纠正

### 误传一：`DISALLOW_*` 常量属于 `DevicePolicyManager`

错。它们定义在 `android.os.UserManager`（如 `DISALLOW_ADD_USER`）；DPM 的 `addUserRestriction()` 只是传递 key。

### 误传二：legacy admin 可以静默安装任意应用

错。legacy admin 只能执行 XML 中声明的擦除、锁屏、密码、相机等策略；静默安装是 Device Owner 的能力。

### 误传三：Profile Owner 能读个人资料

错。工作资料是独立的 secondary user，拥有独立存储；Profile Owner 只能管理该资料，跨资料只能通过 `CrossProfileApps` 启动自己的 Activity。

### 误传四：zero-touch 所有设备都支持

错。它限授权经销商、GMS 兼容设备，并要求 Android 9+ 或兼容 8.0/Pixel 7.0 的特定条件。

### 误传五：存在 `PROVISION_DEVICE_OWNER` 权限

错。Android 5.0–16 的清单与公开权限页都没有这个权限；设定 owner 使用 `MANAGE_PROFILE_AND_DEVICE_OWNERS` 或 shell，配置消息使用 `DISPATCH_PROVISIONING_MESSAGE`。

## 13. 未核实与失效来源

- `source.android.com/tools/dpm` 已重定向到 Android Studio；`/work/dpc/deprecated-apis` 与 `/work/device-owner` 均 404。
- `PROVISION_DEVICE_OWNER` 只能确认在 5.0–16 未列出，不能证明历史上从未出现过。
- CVE-2025-22442 的公告晚于本文基准月，未纳入台账。
- `dpm` 命令仅核对文档，未在真机上复跑；各 OEM 的额外限制未逐一核实。

## 14. 参考资料

- DevicePolicyManager（AOSP）：
  https://android.googlesource.com/platform/frameworks/base/+/main/core/java/android/app/admin/DevicePolicyManager.java
- AndroidManifest（权限声明，AOSP）：
  https://android.googlesource.com/platform/frameworks/base/+/main/core/res/AndroidManifest.xml
- DeviceAdminInfo（AOSP）：
  https://android.googlesource.com/platform/frameworks/base/+/main/core/java/android/app/admin/DeviceAdminInfo.java
- CrossProfileApps（AOSP）：
  https://android.googlesource.com/platform/frameworks/base/+/main/core/java/android/content/pm/CrossProfileApps.java
- UserManager（AOSP）：
  https://android.googlesource.com/platform/frameworks/base/+/main/core/java/android/os/UserManager.java
- 工作资料（官方文档）：
  https://developer.android.com/work/managed-profiles
- 测试环境搭建（官方文档）：
  https://source.android.com/docs/devices/admin/testing-setup
- zero-touch 支持条件（Google 支持）：
  https://support.google.com/work/android/answer/7514005?hl=en
- Android 5.0 版本说明：
  https://developer.android.com/about/versions/lollipop
- Android 7.0 企业变更：
  https://developer.android.com/work/versions/android-7.0
- Android 9 企业变更：
  https://developer.android.com/work/versions/android-9.0
- Android 10 企业变更：
  https://developer.android.com/work/versions/android-10
- Android 11 企业变更：
  https://developer.android.com/work/versions/android-11
- Android 12 企业变更：
  https://developer.android.com/work/versions/android-12
- Android 13 企业变更：
  https://developer.android.com/work/versions/android-13
- Android 14 企业变更：
  https://developer.android.com/work/versions/android-14
- Android 15 企业变更：
  https://developer.android.com/work/versions/android-15
- NVD 条目（本文台账）：
  https://nvd.nist.gov/vuln/detail/CVE-2021-0568
  https://nvd.nist.gov/vuln/detail/CVE-2022-20138
  https://nvd.nist.gov/vuln/detail/CVE-2025-48633
  https://nvd.nist.gov/vuln/detail/CVE-2026-58834

---

## 联系方式

咨询 Android 系统详情请联系 https://t.me/one00190

---

*本手册仅供安全学习与防御研究使用；所有机制表述、公告列值与 CVE 描述均取自公开一手来源并附链接，未能核实的部分已单列于「未核实与失效来源」而非写入正文。*
