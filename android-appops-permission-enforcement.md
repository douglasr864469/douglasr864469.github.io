---
layout: page
title: "Android AppOps 与运行时权限的底层执行路径手册"
description: "把「应用有没有权限」拆成权限状态机与 AppOps 状态机两套：`PermissionManagerServiceImpl` 记录授予标志与来源、`AppOpsService` 在 system_server 按 uid＋包名＋op 记录 `MODE_ALLOWED`／`IGNORED`／`ERRORE..."
date: 2026-11-05
permalink: /android-appops-permission-enforcement/
---

> 用户在设置里看到的「允许/拒绝」，在系统内部并不只落在一个布尔值上：运行时权限由 `PermissionManagerServiceImpl` 维护，而「这次访问要不要记录、要不要拦截、算不算活跃使用」则由 `AppOpsService` 决定。两个子系统一个管「有没有授权」，一个管「此刻能不能做、做了要不要留痕」，任何一处校验缺失，都会让权限对话框变成摆设。这份手册沿「权限状态 → AppOps 判定 → 调用链归因 → 特例与特殊访问 → 公开缺陷 → 怎么防」拆解执行路径。只讨论公开且已修复的问题，不含利用代码。

## 0. 使用方式与边界

面向三类读者：

- **应用开发者**：需要知道 `checkSelfPermission` 为真不等于「一定能拿到数据」，还要经过 AppOps 与数据提供方自己的校验。
- **系统与 ROM 工程师**：需要区分权限服务、AppOps 服务与具体数据提供方三层校验，理解 `noteOp` / `startOp` / `unsafeCheckOp` 的差异。
- **安全评估者**：需要在看到「权限绕过」类公告时，判断缺陷落在权限授予、AppOps 模式判定、调用者归因还是数据方过滤。

三条自我约束：

1. 只引用公开且已修复的问题；CVE 描述以 NVD 与 Android Security Bulletin 为准。
2. 不写利用步骤、不写 PoC、不写可直接复现的权限组合。
3. 查不到一手来源的说法放进「未核实与失效来源」。

与已发布手册的分工：

- 《Android SELinux 策略与 seccomp 沙箱手册》讲内核强制访问控制；本文讲应用层权限与 AppOps。
- 《Android 无障碍、输入法与自动填充手册》讲三类高敏感服务的授权；本文只把它们当作 AppOps 的一种使用场景。
- 《Android 企业设备管理、托管配置与工作资料攻击面手册》讲管理员如何下发策略；本文讲策略落地后的每次访问如何被判定。

## 1. 两套状态机

### 1.1 权限状态

运行时权限的授予状态由 `PermissionManagerServiceImpl`（旧文档里常写作 `PermissionManagerService`）维护，宿主是 `PackageManagerService`。它记录的不只是「授予/拒绝」，还包括：

- 授权标志（如 `FLAG_PERMISSION_ONE_TIME`、`FLAG_PERMISSION_USER_SET`）；
- 授权来源（用户、管理员、系统预置）；
- 每个包的权限状态与跨用户状态。

`FLAG_PERMISSION_ONE_TIME` 容易被误解为「一种权限类型」。它不是——**一次性授权是加在普通运行时权限上的标志位**，应用离开活跃状态后由系统撤销。

### 1.2 AppOps 状态

`AppOpsService`（`com.android.server.appop`）运行在 system_server，注册名为 `appops`；`AppOpsManager` 只是客户端代理。它按「uid + 包名 + op」记录模式：

| 模式 | 含义 |
|------|------|
| `MODE_ALLOWED` | 允许 |
| `MODE_IGNORED` | 静默拒绝，调用方通常拿到空结果或占位值 |
| `MODE_ERRORED` | 显式报错 |
| `MODE_DEFAULT` | 回落到权限或其他默认规则 |

AOSP 的 `AppOps.md` 明确写道：**每个运行时权限都有对应的 app-op**。这解释了一个常见现象：权限已授予，访问仍可能被 AppOps 拦截；权限被撤销，某些 op 仍可能因特殊规则放行。

## 2. 一次访问的判定链

以「应用读取位置」为例，典型路径是：

1. 应用调用 API，进入框架或系统服务；
2. `ContextImpl.checkPermission` 发起权限检查；
3. 请求经 Binder 进入 `ActivityManagerService`，后者校验 pid/uid、组件是否导出，并调用 `checkComponentPermission`；
4. 权限判定落到 `PermissionManagerServiceImpl`；
5. 真正访问数据前，系统服务或数据提供方调用 `AppOpsManager.noteOp` / `startOp`；
6. 数据方根据返回的 mode 决定放行、返回空数据还是报错；
7. 长时访问结束时调用 `finishOp`，AppOps 据此更新活跃状态。

关键点是第 5–6 步：**权限检查通过只是获得了「候选资格」，AppOps 与数据方才是实际闸门**。只做第 2–4 步、跳过第 5 步的代码，就是典型的「检查了但没执行」。

### 2.1 noteOp / startOp / finishOp

- `noteOp`：记录一次瞬时访问，返回当前 mode；
- `startOp`：开始一段长时访问，把该 op 置为 active；
- `finishOp`：结束长时访问；
- `unsafeCheckOp`：只查模式、不记录。它适合内部快速判断，**不适合作为授权决策的唯一依据**。

「active」状态会驱动隐私指示器、后台限制与使用统计。CVE-2021-39693 的根因正是 `onUidStateChanged` 的逻辑错误，使位置访问没有触发可见指示器——访问发生了，用户却看不到。

### 2.2 归因（attribution）

Android 12 引入 `<attribution android:tag>`，应用可以在调用时携带标签，系统据此区分「同一个应用内的不同功能」。`AttributionSource` 还能表达「A 应用通过 B 应用访问数据」的链式关系。

归因不是装饰：它决定这条访问记在谁头上、隐私指示器显示谁、企业审计看到谁。归因链一旦被伪造或截断，责任主体就会错位。

## 3. 特殊访问与特殊权限

有一类能力不走普通运行时权限，而走「特殊应用访问」（special app access）：

| 能力 | 判定接口 | 底层 |
|------|----------|------|
| 显示在其他应用上层 | `Settings.canDrawOverlays()` | `OP_SYSTEM_ALERT_WINDOW` |
| 所有文件访问 | `Environment.isExternalStorageManager()` | 存储访问框架 + AppOps |
| 后台位置 | 运行时权限 + 设置页 | `ACCESS_BACKGROUND_LOCATION` |
| 通知 | `POST_NOTIFICATIONS`（Android 13+） | 运行时权限 + AppOps |
| 使用情况访问 | `AppOpsManager` 查询 | `OP_GET_USAGE_STATS` |

这些能力的共同点是：**授权入口在设置页，而不是弹窗**；判定在 AppOps，而不是单纯的权限位。因此「应用有权限」与「应用能用」之间可能隔着一次设置页操作。

### 3.1 全文件访问的真实边界

`MANAGE_EXTERNAL_STORAGE` 常被理解为「能读整个存储」。准确的表述是：

- 它让应用通过 MediaStore / SAF 之外的传统路径访问共享存储；
- 它仍受 SELinux、AppOps 与具体目录的过滤规则约束；
- Android 11 起，`Android/data` 与 `Android/obb` 的访问被进一步限制；
- 平台对应用商店上架有额外政策要求。

CVE-2024-43093 就是这一层的典型：`ExternalStorageProvider.shouldHideDocument` 对 Unicode 规范化的处理不正确，导致本应隐藏的目录被绕过。它提醒我们，**路径过滤必须与规范化后的字符串比较**，否则「看起来不同、实际相同」的路径就能穿过检查。

## 4. 公开公告台账

| CVE | 落点 / 类型 | 级别 | 修复 | 说明 |
|-----|-------------|------|------|------|
| [CVE-2021-39693](https://nvd.nist.gov/vuln/detail/CVE-2021-39693) | `AppOpsService.onUidStateChanged` 逻辑错误（无指示器访问位置） | 公告 High | 2022-03 | 活跃状态判定缺陷 |
| [CVE-2024-0044](https://nvd.nist.gov/vuln/detail/CVE-2024-0044) | `PackageInstallerService.createSessionInternal` 输入校验缺失（run-as 任意应用） | 公告 High | 2024-10 | 安装会话与包身份 |
| [CVE-2024-43093](https://nvd.nist.gov/vuln/detail/CVE-2024-43093) | `ExternalStorageProvider.shouldHideDocument` Unicode 规范化绕过路径过滤 | 公告 High | 2025-03 | 存储访问过滤 |
| [CVE-2026-28586](https://nvd.nist.gov/vuln/detail/CVE-2026-28586) | `AppOpsService` 多函数缺失权限检查（信息泄露） | 公告 Low（NVD 3.3） | 2026-11 | AppOps 接口本身被越权查询 |

读表要点：

1. **不是所有权限问题都在权限服务里**：CVE-2024-0044 落在包安装会话，CVE-2024-43093 落在存储提供方。
2. **AppOps 自身也是攻击面**：CVE-2026-28586 说明查询接口的权限校验同样重要。
3. **「有权限」不等于「有权访问」**：这几例都发生在权限位之外。

## 5. 成因模式

### 5.1 只查权限、不查 AppOps

代码里写了 `checkSelfPermission` 就直接读数据，跳过了 `noteOp` / `startOp`。结果是：用户关闭了某项访问（AppOps 为 `MODE_IGNORED`），数据仍然被读取。

### 5.2 用 `unsafeCheckOp` 做授权

`unsafeCheckOp` 不记录访问，也不更新活跃状态。用它做最终判定，会让隐私指示器与使用统计失真；在需要审计的场景里等于「没有留痕」。

### 5.3 归因缺失或可伪造

调用方不传 attribution，或数据方只拿 uid 不校验 package/tag，导致访问被记到错误的主体上。企业审计与用户可见提示都会因此失真。

### 5.4 状态变化未重算

用户切换、应用进入后台、UID 状态变化时，活跃 op 需要重新评估（CVE-2021-39693）。只在请求发起时算一次，就会留下「后台仍在访问但无提示」的窗口。

### 5.5 路径过滤未规范化

`shouldHideDocument` 这类过滤函数如果先比较原始字符串、后做 Unicode 规范化，就会出现等价路径绕过（CVE-2024-43093）。正确顺序是**先规范化、再比较**，并对大小写、编码、分隔符做统一处理。

### 5.6 包身份与会话绑定不严

安装会话、`run-as`、共享 uid 等场景里，「这个会话属于哪个包」必须由系统确认。输入校验缺失会让调用者获得不属于自己的包身份（CVE-2024-0044）。

## 6. 版本与行为变更时间线

| 版本 / API | 变化 |
|------------|------|
| 6.0 / 23 | 运行时权限模型引入 |
| 8.0 / 26 | 权限组行为调整；后台限制加强 |
| 10 / 29 | 后台位置权限；分区存储；设备标识符限制 |
| 11 / 30 | 一次性权限、自动重置、全文件访问设置页 |
| 12 / 31 | 大致位置；attribution 标签；隐私指示器 |
| 13 / 33 | `POST_NOTIFICATIONS`、`READ_MEDIA_*` |
| 14 / 34 | 部分照片访问（`READ_MEDIA_VISUAL_USER_SELECTED`） |
| 15 / 35 | 通知 OTP 脱敏、Private Space 等 |

趋势是**从「一次性授予」转向「按次、按范围、按用途」**：一次性权限、部分照片、单次位置、attribution 都是同一方向的产物。

## 7. 观测点与自查

### 7.1 应用侧

- 每次访问敏感数据前，是否同时检查权限与 AppOps？
- 是否使用 `unsafeCheckOp` 作为最终授权判断？
- 长时访问是否配对 `startOp` / `finishOp`？
- 是否传递 attribution，便于系统与用户识别用途？
- 是否处理「权限被自动重置」「用户改为大致位置」「只授权部分照片」等情况？

### 7.2 系统与 ROM 侧

- 数据提供方（MediaStore、联系人、位置服务等）是否在每次访问时校验 uid、包名与 attribution？
- 后台/前台状态变化时，活跃 op 是否重算？
- 路径过滤函数是否先规范化再比较？
- 安装会话、`run-as`、共享 uid 的身份绑定是否严格？
- AppOps 查询接口是否限制调用者？

### 7.3 评估侧

- 先定位落点：权限服务 / AppOps 判定 / 数据方过滤 / 包身份 / 归因；
- 区分「权限绕过」「信息泄露」「指示器缺失」「拒绝服务」；
- 不要把所有「权限」相关 CVE 都归到 `PermissionManagerService`——很多在具体数据方。

## 8. 防御要点

1. **双闸门**：权限检查 + AppOps 判定，两者都过才放行。
2. **拒绝 `unsafeCheckOp` 做授权**：需要留痕的路径必须用 `noteOp` / `startOp`。
3. **归因完整**：传递 attribution，数据方校验 uid + 包名 + tag。
4. **状态联动**：前后台切换、用户切换、权限变更时重算活跃状态。
5. **路径先规范化**：Unicode、大小写、分隔符统一后再做包含/前缀判断。
6. **身份不靠自报**：安装会话、`run-as`、共享 uid 由系统确认包身份。
7. **最小权限与最小范围**：优先 Photo Picker、SAF、单次位置，减少长期授权。

## 9. 常见误传纠正

### 误传一：AppOps 只是给开发者看的统计

错。`MODE_IGNORED` 会让调用方拿到空结果或占位值；悬浮窗、全文件访问本身就是以 AppOps 为核心的授权。

### 误传二：有 `READ_EXTERNAL_STORAGE` 就能读所有媒体

错。Android 10 起分区存储；13 起改用 `READ_MEDIA_*`；14 起用户可只授权部分照片。

### 误传三：一次性权限是独立的权限类型

错。它是加在运行时权限上的 `FLAG_PERMISSION_ONE_TIME` 标志，应用离开活跃状态后撤销。

### 误传四：`checkSelfPermission` 返回 granted 就一定能访问

错。还要经过 AppOps 与数据方校验；用户可能已通过特殊访问设置关闭了该能力。

### 误传五：`unsafeCheckOp` 与 `noteOp` 等价

错。前者不记录、不更新活跃状态，不能作为需要审计的授权依据。

## 10. 未核实与失效来源

- CVE-2026-28586 的影响版本范围未在公开描述中逐一列出，本文只写公告月份与缺陷类型。
- 各 OEM 对 AppOps 的定制（额外 op、额外拦截）未逐一核实。
- 「权限自动重置」在不同厂商的默认周期可能有差异，未逐机型核实。
- 部分第三方汇总把 CVE-2024-0044 的公告月份写错；本文以 MITRE 引用中的 2024-10 公告为准。

## 11. 参考资料

- AppOps 设计说明（AOSP）：
  https://github.com/aosp-mirror/platform_frameworks_base/blob/main/core/java/android/app/AppOps.md
- AppOpsService（AOSP）：
  https://android.googlesource.com/platform/frameworks/base/+/refs/heads/main/services/core/java/com/android/server/appop/AppOpsService.java
- AppOpsManager（AOSP）：
  https://android.googlesource.com/platform/frameworks/base/+/refs/heads/main/core/java/android/app/AppOpsManager.java
- ContextImpl（AOSP）：
  https://android.googlesource.com/platform/frameworks/base/+/refs/heads/main/core/java/android/app/ContextImpl.java
- ActivityManagerService（AOSP）：
  https://android.googlesource.com/platform/frameworks/base/+/refs/heads/main/services/core/java/com/android/server/am/ActivityManagerService.java
- PermissionManagerServiceImpl（AOSP）：
  https://android.googlesource.com/platform/frameworks/base/+/refs/heads/main/services/core/java/com/android/server/pm/permission/PermissionManagerServiceImpl.java
- attribution 元素：
  https://developer.android.com/guide/topics/manifest/attribution-element
- Android 11 权限变更：
  https://developer.android.com/about/versions/11/privacy/permissions
- Android 12 大致位置：
  https://developer.android.com/about/versions/12/behavior-changes-12
- Android 13 通知权限：
  https://developer.android.com/develop/ui/views/notifications/notification-permission
- Android 14 部分照片访问：
  https://developer.android.com/about/versions/14/changes/partial-photo-video-access
- 全文件访问：
  https://developer.android.com/training/data-storage/manage-all-files
- 公告：
  https://source.android.com/docs/security/bulletin/2022-03-01
  https://source.android.com/docs/security/bulletin/2024-10-01
  https://source.android.com/docs/security/bulletin/2025-03-01
  https://source.android.com/docs/security/bulletin/2026/2026-11-01
- NVD 条目（本文台账）：
  https://nvd.nist.gov/vuln/detail/CVE-2021-39693
  https://nvd.nist.gov/vuln/detail/CVE-2024-0044
  https://nvd.nist.gov/vuln/detail/CVE-2024-43093
  https://nvd.nist.gov/vuln/detail/CVE-2026-28586

---

## 联系方式

咨询 Android 系统详情请联系 https://t.me/one00190

---

*本手册仅供安全学习与防御研究使用；所有机制表述、公告列值与 CVE 描述均取自公开一手来源并附链接，未能核实的部分已单列于「未核实与失效来源」而非写入正文。*
