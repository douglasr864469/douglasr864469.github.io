---
layout: page
title: "Android 通知、状态栏与通知监听攻击面手册"
description: "把「一条通知」拆成提交／入队／排序／分发／呈现／交互六段：`NotificationManagerService` 在 system_server 而非 SystemUI、渠道重要性的公开档位与 `IMPORTANCE_MAX` 系统保留、`POST_NOTIFICATIONS` 的预授予与前台服务通知..."
date: 2026-11-03
permalink: /android-notification-statusbar-listener/
---

> 一条通知从应用调用 `notify()` 到出现在状态栏，中间要穿过 system_server 里的 `NotificationManagerService`、用户设置的渠道与免打扰策略、系统 UI 的渲染与锁屏可见性判定，以及一组被用户显式授权过的「通知监听器」。它看起来只是提示消息，实际上承载着验证码、银行动账、聊天内容、通话邀请和前台服务保活凭证——所以 Android 在这个子系统上修了十多年的权限缺陷：从 2019 年的 `NotificationManagerService` 缺权限检查，到 2021 年的 `snoozeNotification` 权限混淆，再到 2022 年的锁屏隐藏通知失效、2023 年的后台启动绕过，以及 Android 15 把「验证码对不受信监听器脱敏」做成平台行为。这份手册沿「通知怎么进系统 → 谁知情 → 谁有权改 → 怎么呈现 → 防御方怎么查」这条链，把机制、公开缺陷模式和加固手段拆开说明。只讨论公开且已修复的问题，不含可利用代码。

## 0. 使用方式与边界

这份手册面向三类读者：

- **应用开发者**：需要知道通知内容为什么会出现在锁屏、投屏和他人的监听器里，`PendingIntent` 该用哪种可变性，以及什么样的通知会被系统当作「高优先级」再被用户关掉。
- **系统与平台工程师**：需要理解 `NotificationManagerService`、`SystemUI`、通知监听器与设备策略之间的分工，以及为什么一个「可见性」判断写错就会变成提权。
- **安全评估者**：需要在看到一条「通知相关」公告时，判断它落在通知服务、监听器接口、锁屏呈现、`PendingIntent` 还是前台服务保活，而不是把所有带「notification」字样的 CVE 混为一类。

本文的三条自我约束：

1. 只引用公开且已修复的问题；CVE 编号、代码落点、CVSS 与描述以 NVD 条目为准，公告归类与严重级别以 Android Security Bulletin 为准，两者不一致处单独标注。
2. 不写利用步骤、不写 PoC、不写可直接复现的攻击参数；成因只描述到「哪一类校验或状态缺失」。
3. 查不到一手来源的说法一律进「未核实与失效来源」，不写进正文。

与已发布手册的分工：

- 《Android 窗口管理、悬浮窗与界面覆盖攻击面手册》讲的是窗口层级与触摸命中；本文只讲通知在状态栏和锁屏上的可见性，不重复覆盖层与 tapjacking。
- 《框架 IPC 导出面手册》讲 Binder 接口与组件导出的一般规则；本文只讲通知子系统里的 Binder 入口与 `PendingIntent` 授权语义。
- 《无障碍、输入法与自动填充通道手册》讲无障碍服务读取界面内容；通知监听器是另一条独立授权通道，本文不把它与无障碍混为一谈。
- 《账户、凭据与无密码认证手册》讲凭据存储与认证流程；本文只把验证码通知当作泄露面，不展开账户体系。
- 《摄像头、麦克风与屏幕捕获面手册》讲采集与投屏权限；本文只引用 Android 15 的投屏通知隐藏作为防御变化。
- 《多用户、工作资料与沙箱隔离手册》讲用户与资料边界；本文只讲通知跨用户/跨资料的可见性缺陷。

## 1. 一条通知的完整旅程

把「应用发了一条通知」拆开，至少经过八个阶段，每个阶段都有独立的信任判断：

| 阶段 | 主要位置 | 做的事 | 安全判断点 |
|------|----------|--------|------------|
| 1. 构造 | 应用进程 | `Notification.Builder` 填内容、渠道、`PendingIntent`、可见性 | 内容是否含敏感信息；`PendingIntent` 是否可变 |
| 2. 提交 | 应用 → `INotificationManager` | 跨 Binder 调 `notify()` / `notifyAsPackage()` | 调用者 uid、包名、用户是否匹配 |
| 3. 入队 | `NotificationManagerService`（system_server） | 校验渠道、去重、限额、前台服务绑定 | 权限、渠道归属、前台服务类型 |
| 4. 排序与策略 | `RankingHelper` 与免打扰策略 | 计算重要性、是否静音、是否折叠、是否打断 | 用户设置优先级高于应用声明 |
| 5. 持久化 | 通知服务内部记录 | 保存到当前用户的通知列表，支持 snooze | 用户边界；snooze 归属 |
| 6. 分发 | 通知服务 → 监听器 / 助手 / SystemUI | 通过 `INotificationListener` 回调推送 | 监听器授权范围；是否脱敏 |
| 7. 呈现 | `SystemUI` | 状态栏图标、通知栏、heads-up、锁屏、气泡、媒体控件 | 锁屏可见性、投屏状态、公共版本 |
| 8. 交互 | `PendingIntent.send()` | 以创建者身份执行点击、回复、全屏意图 | 可变性、目标组件是否显式 |

关键点是：**「应用声明的重要性」和「系统最终呈现的强度」不是一回事**。渠道重要性只是上限，用户设置、免打扰、锁屏策略、投屏状态都可以往下压；反过来说，一个被标成「高优先级」的通知如果走了全屏意图，还能绕过锁屏直接占据屏幕——这正是 Android 14 收紧全屏意图的原因。

另一个关键点是：**通知内容并不只给用户看**。用户授权的通知监听器、系统通知助手、SystemUI 自身、以及投屏/录屏通道都会拿到同一份数据。Android 15 之前，这份数据对监听器基本是原样的；Android 15 起才对「不受信监听器」做验证码脱敏。

## 2. 对象模型与进程边界

### 2.1 关键对象

| 对象 | 所在层 | 作用 |
|------|--------|------|
| `Notification` | 公开 SDK | 应用构造的通知内容、渠道 ID、动作、样式、可见性 |
| `NotificationChannel` / `NotificationChannelGroup` | 公开 SDK（API 26+） | 通知分类与用户可调的重要性、声音、震动 |
| `StatusBarNotification`（SBN） | 公开 SDK | 系统侧包装：包名、uid、用户、post 时间、通知本体 |
| `NotificationRecord` | AOSP 内部 | 通知服务保存的单条记录，附带排序与策略状态 |
| `RankingMap` / `Ranking` | 公开 SDK | 监听器拿到的排序与「是否打断」等系统判定 |
| `NotificationListenerService` | 公开 SDK | 用户授权的第三方通知读取与操作接口 |
| `NotificationAssistantService` | `@SystemApi` | 系统级通知助手，可调整、延后、自动回复 |
| `NotificationDelegate` | AOSP 内部 | 通知服务把操作委托给 SystemUI 的接口 |

### 2.2 进程与权限边界

| 进程 | 身份 | 能看到什么 | 能改什么 |
|------|------|------------|----------|
| 应用进程 | 普通应用 uid | 自己发布的通知（`getActiveNotifications()`） | 自己的通知与渠道 |
| system_server | 系统 uid | 所有用户的所有通知 | 通知列表、渠道、监听器绑定、策略 |
| SystemUI | 系统应用 | 当前用户的通知流 | 呈现、清除、部分交互（经系统服务） |
| 通知监听器 | 普通应用 uid + 用户授权 | 授权用户范围内的通知内容与排序 | 取消、延后、标记已读、读取活动通知 |
| 通知助手 | 系统应用（`@SystemApi`） | 当前用户通知流 | 调整、延后、自动回复、建议 |
| 设备策略控制器 | 设备管理应用 | 跨资料策略 | 限制工作资料通知对监听器的暴露 |

`NotificationManagerService` 跑在 **system_server**，系统服务名是 `notification`（即 `Context.NOTIFICATION_SERVICE`）；它不是一个独立应用，也不是 SystemUI 的一部分。SystemUI 自己也是通知服务的一个内部监听者，负责把通知画到状态栏、通知栏和锁屏上。把「通知服务」和「通知界面」混为一谈，是读公告时最常见的定位错误。

通知监听器通过 `android.permission.BIND_NOTIFICATION_LISTENER_SERVICE` 绑定。这个权限的保护级别是 `signature`——意味着**只有系统能绑定它**；普通应用在清单里声明它，并不等于自己拿到了权限，真正的开关是用户在「通知使用权」里把某个监听器加入允许列表。系统把这个允许列表存在安全设置里，并在绑定前核对。监听器服务通常声明 `android:exported="false"`，因为绑定方是系统而不是其他应用。

## 3. 渠道、重要性与用户设置

### 3.1 渠道是用户控制的边界

Android 8.0（API 26）起，每条通知必须归属一个渠道。渠道一旦创建，应用对它的修改会受到限制：

- 应用可以在用户尚未手动调整前修改渠道的重要性、声音等属性；
- 用户手动改过某个渠道后，应用再改重要性通常不会生效，避免应用「偷偷把被关掉的渠道重新打开」；
- 用户可以在应用级、渠道级、免打扰三个层次上分别压制通知，应用无法绕过。

重要性的公开常量是 `IMPORTANCE_NONE(0)`、`IMPORTANCE_MIN(1)`、`IMPORTANCE_LOW(2)`、`IMPORTANCE_DEFAULT(3)`、`IMPORTANCE_HIGH(4)`、`IMPORTANCE_MAX(5)`。其中 `IMPORTANCE_MAX` 保留给系统，普通应用最高只能用到 `HIGH`；`HIGH` 才具备触发 heads-up 的资格，而是否真的弹出还取决于免打扰和屏幕状态。

### 3.2 免打扰是另一项独立授权

免打扰（DND）不是普通通知权限的一部分。应用若要读取或修改中断过滤规则，需要「通知策略访问」这一特殊授权（`NotificationManager.isNotificationPolicyAccessGranted()`，设置入口是 `Settings.ACTION_NOTIFICATION_POLICY_ACCESS_SETTINGS`）。这项授权比「发通知」敏感得多：拿到它的应用可以改变设备的中断策略，让本该静音的通知重新响铃，或反过来压掉闹钟与通话。审计时要把「通知使用权」和「通知策略访问」分开看。

### 3.3 Android 13 的运行时权限

Android 13（API 33）引入 `POST_NOTIFICATIONS` 运行时权限：

- 新安装在 Android 13+ 设备上的应用，通知默认关闭，必须请求并由用户授予；
- 设备从 12L 及以下升级时，符合条件的既有应用会被系统预先授予（条件是已有渠道且用户此前没有明确关闭通知）；
- 用户拒绝后，普通通知与渠道全部被阻断，但前台服务通知仍会出现在**任务管理器**里，只是不进通知栏；
- 媒体会话通知、以及通过 `MANAGE_OWN_CALLS` + `ConnectionService` + `registerPhoneAccount()` 自管通话的应用，属于豁免情形。

这带来一个常见误判：**「用户拒绝通知权限」不等于「前台服务可以不要通知」**。前台服务仍然必须带通知，只是这条通知在拒绝后对用户不可见于通知栏。

### 3.4 应用侧的自检

发通知前应调用 `NotificationManager.areNotificationsEnabled()` 判断用户是否整体关闭了通知；更细的判断可以结合渠道重要性（`getNotificationChannel().getImportance() == IMPORTANCE_NONE` 表示该渠道被关）。不要假设「权限授予了」就等于「用户能看到」。

## 4. 通知监听的授权模型

### 4.1 清单与绑定

一个通知监听器的最小声明如下（取自官方文档示例）：

```xml
<service android:name=".NotificationListener"
         android:label="@string/service_name"
         android:exported="false"
         android:permission="android.permission.BIND_NOTIFICATION_LISTENER_SERVICE">
    <intent-filter>
        <action android:name="android.service.notification.NotificationListenerService" />
    </intent-filter>
</service>
```

系统只绑定用户在「通知使用权」里勾选过的组件。设置页的入口常量是 `Settings.ACTION_NOTIFICATION_LISTENER_SETTINGS`；Android 13+ 的英文界面把它归到「Device & app notifications」，中文名称随厂商翻译（常见「通知使用权」）。允许列表落在 `Settings.Secure.ENABLED_NOTIFICATION_LISTENERS`，是一个用冒号分隔的组件名列表。

### 4.2 监听器能做什么

| 能力 | 接口 | 边界 |
|------|------|------|
| 接收新通知与移除事件 | `onNotificationPosted` / `onNotificationRemoved` | 只覆盖监听器有访问权的用户 |
| 读取当前活动通知 | `getActiveNotifications()` | 「对当前用户可见」的活动通知 |
| 读取已延后通知 | `getSnoozedNotifications()` | 文档明确为「该监听器有访问权的所有用户」 |
| 取消通知 | `cancelNotification()` / `cancelNotifications()` | 作用于授权范围内的通知 |
| 延后通知 | `snoozeNotification(key, durationMs)` | 2021 年多条 CVE 的落点 |
| 标记已展示 | `setNotificationsShownFromListener()` | 曾被用于判断应用是否安装 |
| 读取排序与打断判定 | `RankingMap` / `Ranking` | 系统算出的结果，不是应用声明值 |

注意最后一条：监听器拿到的不是「应用说自己多重要」，而是系统根据渠道、用户设置、免打扰算出的 `Ranking`。这决定了通知是否静音、是否在锁屏显示、是否属于会话。

### 4.3 监听器的三条限制

1. **工作资料里的监听器会被系统忽略。** 官方文档明确写道，系统会忽略运行在工作资料中的通知监听器。
2. **跨资料暴露受设备策略约束。** 设备策略控制器可以阻止工作资料产生的通知流向个人侧的监听器；Android 也提供了 `DevicePolicyManager.setPermittedCrossProfileNotificationListeners` 这类允许名单接口（其空列表与 `null` 的精确语义见第 13 节）。
3. **低内存设备上有额外限制。** 官方文档说明，Android 10 及以下的低内存设备可能不绑定通知监听器。

### 4.4 通知助手

`NotificationAssistantService` 是 `@SystemApi`，权限为 `android.permission.BIND_NOTIFICATION_ASSISTANT_SERVICE`，同一时间只有一个助手生效。它能读取通知、调整排序、延后、自动回复，权限比普通监听器更高，且通常由系统应用或厂商应用承担。评估时不要把第三方监听器与通知助手当作同一类主体。

### 4.5 为什么这是高敏权限

一条验证码通知、一条银行动账、一条聊天消息，往往同时构成「身份凭证」和「隐私内容」。监听器不仅能读，还能取消（让用户看不到）、延后（让验证码晚到）和标记已读。历史上多条公告就落在这些操作缺少范围校验上。Android 15 起，系统会对「不受信监听器」隐藏已识别出的验证码内容；受信应用（例如通过伴生设备管理器关联的应用）不受这条限制。这是平台层第一次在**内容**上而不仅是**权限**上给监听器划界。

## 5. PendingIntent 与通知跳转

### 5.1 可变性规则

`PendingIntent` 以**创建者**的身份和权限执行，这是它全部风险与价值的来源。可变性规则如下：

| 版本/目标 | 规则 |
|-----------|------|
| API 23 起 | 提供 `FLAG_IMMUTABLE` |
| API 31 起 | 提供 `FLAG_MUTABLE` |
| targetSdk 31（Android 12）起 | 创建时必须显式选择 `FLAG_IMMUTABLE` 或 `FLAG_MUTABLE` |
| Android 12 之前 | 未指定时默认可变 |
| targetSdk 34（Android 14）起 | 可变且隐式（未指定 component/package）的 `PendingIntent` 直接抛异常 |
| targetSdk 34 起 | 隐式 Intent 只投递给 exported 组件 |

这里最容易记错的是第二条：Android 12 **不是**「默认不可变」，而是「必须显式声明」。默认不可变是很多文章的口径错误，会让读者以为不写 flag 就安全。

### 5.2 通知跳转的三条路径

1. **contentIntent**：点击通知主体。应当是不可变的显式 `PendingIntent`，目标 Activity 不需要导出。
2. **action**：通知上的按钮。内联回复等需要接收方填充内容的场景仍需可变 `PendingIntent`，但必须指定明确的 component，且配合 `RemoteInput` 使用。
3. **fullScreenIntent**：锁屏或息屏时直接拉起全屏界面。它是最敏感的一条，见下节。

### 5.3 通知蹦床（trampoline）

targetSdk 31 起，应用不能再用「通知点击 → 广播接收器/服务 → 再启动 Activity」这种蹦床方式绕过后台启动限制；`SYSTEM_ALERT_WINDOW` 权限是官方列出的例外之一。正确做法是让 `PendingIntent` 直接指向 Activity。Android 14 又进一步要求：应用通过 `PendingIntent.send()` 把自己的后台启动特权授予他人时，必须显式选择加入（`ActivityOptions.setPendingIntentBackgroundActivityStartMode(MODE_BACKGROUND_ACTIVITY_START_ALLOWED)`）。

### 5.4 全屏意图的收紧

Android 11 起，任何声明了 `USE_FULL_SCREEN_INTENT` 的应用都可以在锁屏上弹全屏通知。Android 14 对 targetSdk 34 的应用把范围收窄到**通话与闹钟**两类；Google Play 会对不符合定位的应用撤销该权限（政策截止日期为 2024-05-31）。升级前已安装的应用保留权限，用户可以自行开关。应用可以用 `NotificationManager.canUseFullScreenIntent()` 检查，并通过 `Settings.ACTION_MANAGE_APP_USE_FULL_SCREEN_INTENT` 引导用户。

另外，Android 13 起，设备处于使用中（已解锁）时，全屏意图会降级为 heads-up 通知，而不是直接占屏。

## 6. 状态栏、锁屏与内容最小化

### 6.1 三级可见性

`Notification.Builder.setVisibility()` 有三个公开值：

| 值 | 锁屏表现 |
|----|----------|
| `VISIBILITY_PUBLIC` | 完整显示内容 |
| `VISIBILITY_PRIVATE` | 显示图标与标题，隐藏正文；若设置了 `publicVersion`，锁屏显示公共版本 |
| `VISIBILITY_SECRET` | 锁屏完全不显示 |

用户还可以在设置里统一选择「隐藏锁屏敏感内容」，此时系统的判定优先于应用的 `PUBLIC` 声明。`setPublicVersion()` 提供一份脱敏后的通知副本，供锁屏、投屏等「不安全上下文」使用；Android 15 的投屏隐藏也会优先使用这份公共版本。

### 6.2 锁屏隐藏的判定链

锁屏是否显示某条通知，由 SystemUI 结合用户设置与通知服务提供的可见性共同判定。公开公告里出现过两类缺陷：

- **该隐藏的没隐藏**：`KeyguardNotificationVisibilityProvider.shouldHideNotification` 的逻辑错误（CVE-2022-20477），使本应隐藏的通知重新可见；
- **隐藏偏好被绕过**：`fixNotification` 里的逻辑错误（CVE-2023-21191），使通知绕过了用户的隐藏偏好。

这两条都不是内存破坏，而是**状态判定错误**——与窗口覆盖手册里「可见性与可触摸性脱节」属于同一类思维方式，只是落点不同。

### 6.3 投屏与录屏

Android 15 增加了投屏保护：投屏会话期间通知内容会被隐藏；若应用提供了 `publicVersion`，系统显示公共版本，否则直接脱敏。应用还可以用 `View.setContentSensitivity()` 手动标记敏感视图，使其对远端观看者隐藏。系统自带录屏不受此限制，因为录制内容留在本机。Android 15 QPR1 起，投屏/录制期间状态栏会出现醒目的提示芯片，锁屏时投屏自动停止。

### 6.4 媒体通知、气泡与会话

- **媒体会话通知**不受 `POST_NOTIFICATIONS` 拒绝的影响，因为媒体控件属于系统级会话。
- **气泡**在 Android 10（API 29）引入；Android 11 起进一步收紧——通知必须满足会话条件并关联会话快捷方式，否则不能以气泡呈现。
- **heads-up** 需要渠道达到 `IMPORTANCE_HIGH`，同时未被免打扰压制，且屏幕状态允许。

这些呈现方式都会扩大内容的暴露面：锁屏、投屏、通知栏预览、气泡悬浮窗。设计通知时应按「最坏上下文」来写内容，而不是按「应用在前台」来写。

## 7. 公开公告台账

以下条目的编号、代码落点、公开时间与 CVSS 以 NVD 条目为准；「落点」是 NVD 描述里给出的类/方法名。Android Security Bulletin 的归类与严重级别在第 7.2 节单独列出，两者口径不同处不要混用。

### 7.1 通知服务与监听接口

| CVE | 公开时间 | 落点 | CVSS v3.1 | 成因要点 |
|-----|----------|------|-----------|----------|
| CVE-2019-2098 | 2019-06 | `NotificationManagerService.areNotificationsEnabledForPackage` | 7.8 | 缺权限检查，可查询/影响其他包的通知开关 |
| CVE-2019-2219 | 2019-12 | `NotificationManagerService` 及相关文件 | 4.7 | 权限绕过，可在后台录音而不向用户显示通知 |
| CVE-2020-0084 | 2020-03 | `NotificationManagerService` 多个函数 | 7.8 | 多处缺权限检查 |
| CVE-2020-0109 | 2020-05 | `simulatePackageSuspendBroadcast` | 7.8 | 缺权限检查 |
| CVE-2020-0313 | 2020-09 | `NotificationManagerService` | 5.5 | 不安全的 `PendingIntent` 导致权限绕过与信息泄露 |
| CVE-2021-0513 | 2021-06 | `deleteNotificationChannel` 及相关函数 | 7.8 | 状态校验不当，可绕过权限删除他人渠道 |
| CVE-2021-0682 | 2021-10 | `sendAccessibilityEvent` | 5.5 | 缺权限检查，导致通知数据泄露 |
| CVE-2021-0705 | 2021-10 | `sanitizeSbn` | 7.8 | 绕过后台服务限制，让服务保持前台状态与已授权限 |
| CVE-2021-0981 | 2021-12 | `enqueueNotificationInternal` | 7.8 | 输入校验不当，可运行前台服务而不显示通知 |
| CVE-2021-1019 | 2021-12 | `NotificationListenerService.snoozeNotification` | 7.3 | 误导性授权对话框造成权限混淆，可本地提权 |
| CVE-2021-1020 | 2021-12 | `snoozeNotification` | 7.3 | 输入校验不当，可关闭任意用户的通知 |
| CVE-2021-1021 | 2021-12 | `NotificationManagerService.snoozeNotificationInt` | 7.3 | 同类输入校验问题，可关闭任意用户的通知 |
| CVE-2021-1030 | 2021-12 | `setNotificationsShownFromListener` | 5.5 | 侧信道，可在无查询权限时判断某应用是否安装 |
| CVE-2021-1031 | 2021-12 | `cancelNotificationsFromListener` | 3.3 | 侧信道，可判断某应用是否安装 |
| CVE-2021-39704 | 2022-03 | `deleteNotificationChannelGroup` | 7.8 | 权限绕过，可运行前台服务而不显示通知 |
| CVE-2022-20011 | 2022-05 | `getArray` | 5.5 | 缺检查，把一个用户的通知泄露给另一个用户 |
| CVE-2022-20206 | 2022-06 | `setPackageOrComponentEnabled` | 5.5 | 缺权限检查 |
| CVE-2022-20448 | 2022-11 | `buzzBeepBlinkLocked` | 5.5 | 权限绕过，跨用户共享数据 |
| CVE-2023-40106 | 2024-02 | `sanitizeSbn` | 7.8 | 后台启动（BAL）绕过，可从后台启动 Activity |
| CVE-2024-23708 | 2024-05 | `NotificationManagerService` 多个函数 | 7.8 | 剪贴板被访问时可不显示提示，削弱用户感知 |

### 7.2 锁屏、通知内容与相邻组件

| CVE | 公开时间 | 落点 | CVSS v3.1 | 成因要点 |
|-----|----------|------|-----------|----------|
| CVE-2020-0248 | 2020-08 | `InstantAppNotifier.postInstantAppNotif` | 5.5 | `PendingIntent` 错误导致权限绕过 |
| CVE-2020-0389 | 2020-09 | `RecordingService.createSaveNotification` | 5.5 | 不安全的 `PendingIntent`，本地数据暴露 |
| CVE-2022-20477 | 2022-12 | `KeyguardNotificationVisibilityProvider.shouldHideNotification` | 7.8 | 逻辑错误，使本应隐藏的锁屏通知可见 |
| CVE-2023-21191 | 2023-06 | `fixNotification` | 7.8 | 逻辑错误，绕过用户的隐藏偏好 |

### 7.3 公告归类核对

NVD 的「落点 + CVSS」与 Android Security Bulletin 的「组件 + 类型 + 级别」是两套口径。以下几条已逐条核对公告页，引用时不要互换：

| CVE | 公告 | 组件 | 类型 | 公告级别 |
|-----|------|------|------|----------|
| CVE-2021-1019 | Pixel 2021-12 | Framework | EoP | Moderate |
| CVE-2021-1020 | Pixel 2021-12 | System | EoP | Moderate |
| CVE-2022-20477 | 2022-12 | Framework | EoP | High |
| CVE-2020-0248 | 2020-08 | Framework | ID | High |
| CVE-2020-0389 | 2020-09 | Framework | ID | High |
| CVE-2020-0313 | Android 11 公告 | System | ID | Medium |

两个值得注意的偏差：

1. **CVE-2021-1019 / 1020 出现在 Pixel 公告而不是主公告**，且公告级别是 Moderate，NVD 给的是 7.3（High）。写「高危漏洞」时要么注明 NVD，要么注明公告，不要混着写。
2. **CVE-2022-20477 的公告页没有给出方法名**，方法名来自 NVD 描述。只引公告的人会以为它是个笼统的 Framework 问题，实际落点非常具体。

### 7.4 从台账能读出的趋势

1. **2019–2020 年**以「缺权限检查」为主：`areNotificationsEnabledForPackage`、`simulatePackageSuspendBroadcast`、`setPackageOrComponentEnabled`。修复方式多是补一次调用者校验。
2. **2020–2021 年**出现 `PendingIntent` 类问题：通知里携带的 `PendingIntent` 可被他人利用，或系统服务创建时默认不安全。
3. **2021 年**集中在监听接口的「范围校验」：`snoozeNotification`、`setNotificationsShownFromListener`、`cancelNotificationsFromListener`。监听器本来是合法授权，缺陷在于它能越过授权范围影响他人。
4. **2022 年**转向跨用户/跨资料边界：`getArray`、`buzzBeepBlinkLocked`。这类问题在单用户测试里看不出来，必须开工作资料或多用户复现。
5. **2023–2024 年**转向状态与后台启动：锁屏隐藏偏好、`sanitizeSbn` 的 BAL 绕过、剪贴板提示抑制。修复点越来越靠近「用户感知」而不是「数据访问」。
6. **2025 年起**平台开始做内容级防护：验证码脱敏、投屏隐藏。这是从「修单个入口」转向「降低整类数据价值」的信号。

### 7.5 容易误归的条目

| CVE | 常见误归 | 实际 |
|-----|----------|------|
| CVE-2021-39698 | 通知 / `PendingIntent` | 内核 `aio_poll_complete_work` 的 UAF（CWE-416），与通知无关 |
| CVE-2019-2219 | 纯音频漏洞 | 描述同时涉及通知服务权限绕过，是「后台录音不提示」的组合问题 |
| CVE-2020-0389 | 通用通知缺陷 | 落点在录音服务的「保存录音」通知，属于相邻组件 |
| CVE-2024-23708 | 剪贴板漏洞 | 落点在通知服务，问题是剪贴板访问提示被抑制 |

## 8. 六种成因模式

### 8.1 缺权限检查

`areNotificationsEnabledForPackage`、`simulatePackageSuspendBroadcast`、`setPackageOrComponentEnabled` 都属于这一类：接口本身需要调用者是系统或持有特定权限，但实现里少了一次 `checkCallingPermission` 或 uid 校验。结果是普通应用可以查询或修改其他包的通知状态。防御上，系统服务里每个接收包名/uid 参数的入口都要显式核对调用者与目标是否一致。

### 8.2 不安全的 PendingIntent

通知天然携带 `PendingIntent`：点击、按钮、全屏意图。如果创建时用了可变 `PendingIntent` 且目标不显式，接收方就能填充未指定的字段，把「点击打开详情页」变成「点击执行别的组件」。CVE-2020-0313、CVE-2020-0248、CVE-2020-0389 都是这一类。防御规则很直接：能用不可变就用不可变；必须可变时绑定显式 component，并只暴露必要的字段。

### 8.3 输入校验与状态校验缺失

`snoozeNotification` 的时长、`deleteNotificationChannel` 的渠道归属、`deleteNotificationChannelGroup` 的组归属，都需要校验「这个对象是不是调用者能动的」。缺了这层校验，就会出现「关闭任意用户的通知」「删除他人渠道后让前台服务不再提示」这类后果。CVE-2021-0513、CVE-2021-1021、CVE-2021-0981、CVE-2021-39704 属于这一类。

### 8.4 跨用户与跨资料边界

Android 的通知按用户隔离。`getArray` 的缺检查导致一个用户的通知泄露给另一个用户（CVE-2022-20011），`buzzBeepBlinkLocked` 的权限绕过允许跨用户共享数据（CVE-2022-20448）。这类缺陷在单用户、无工作资料的测试环境里不会暴露，评估时必须构造多用户场景。

### 8.5 可见性状态逻辑错误

锁屏隐藏、免打扰、投屏隐藏都依赖一串状态判断。`shouldHideNotification`（CVE-2022-20477）和 `fixNotification`（CVE-2023-21191）都是判断逻辑写错，导致「该隐藏的显示出来」或「用户的隐藏偏好被绕过」。这类问题没有内存破坏，也不会触发权限异常，只会让内容出现在不该出现的界面上，因此很难被自动化测试抓到。

### 8.6 侧信道与后台启动

`setNotificationsShownFromListener`、`cancelNotificationsFromListener` 曾被用来在无查询权限时判断某应用是否安装（CVE-2021-1030/1031）；`sanitizeSbn` 的缺陷则让应用从后台启动 Activity（CVE-2023-40106）。前者是「合法接口泄露了本不该泄露的一位信息」，后者是「通知服务成了绕过后台启动限制的跳板」。防御上，通知服务的每个回调都要考虑：调用者能否从「成功/失败/耗时」里推断出额外信息。

## 9. 版本加固时间线

| 版本 | 变化 | 与本文的关系 |
|------|------|--------------|
| Android 8.0（API 26） | 引入通知渠道与渠道组 | 用户首次获得细粒度控制权 |
| Android 10（API 29） | 引入气泡；通知助手接口 | 呈现方式增多，助手成为高权限主体 |
| Android 11（API 30） | 任何声明 `USE_FULL_SCREEN_INTENT` 的应用都可锁屏全屏；气泡要求收紧 | 全屏意图风险扩大，为 14 的收紧埋下伏笔 |
| Android 12（API 31） | `PendingIntent` 必须显式可变性；禁止通知蹦床 | 直接消灭一大类可变 `PendingIntent` 误用 |
| Android 13（API 33） | `POST_NOTIFICATIONS` 运行时权限；全屏意图在解锁状态降级为 heads-up | 用户首次能整体拒绝通知 |
| Android 14（API 34） | 全屏意图限通话/闹钟；可变隐式 `PendingIntent` 抛异常；隐式 Intent 只投递给导出组件；BAL 需显式选择加入 | 收窄三个历史高危面 |
| Android 15（API 35） | 对不受信监听器脱敏验证码；投屏时隐藏通知内容；强制停止时取消 `PendingIntent` | 平台开始在内容层保护用户 |
| Android 15 QPR1 | 投屏/录制状态栏提示芯片；锁屏自动停止投屏 | 提升用户对投屏的感知 |

这张表说明一件事：**加固不是一次性的**。每一轮「收紧」都会把攻击面从「读取内容」推向「推断状态」或「绕过呈现」，公告里的落点也随之从权限检查转向逻辑判断。

## 10. 观测点与自查

### 10.1 设备侧排查

以下命令用于防御性自查，需要在已授权调试的设备上执行：

```bash
# 查看通知服务状态、监听器、延后记录（--noredact 保留内容，仅限调试设备）
adb shell dumpsys notification --noredact

# 查看当前被授予「通知使用权」的组件
adb shell settings get secure enabled_notification_listeners

# 查看某个应用的通知权限/AppOp 状态
adb shell cmd appops get com.example.app POST_NOTIFICATION

# 发一条测试通知，验证渠道、可见性与锁屏表现
adb shell cmd notification post -S bigtext -t "测试" tag "正文内容"

# 过滤监听器与锁屏相关行
adb shell dumpsys notification --noredact | grep -iE "listener|snooze|keyguard"
```

若机型不支持某个 AppOp 名称，可先执行 `adb shell cmd appops get com.example.app` 全量列出再查找。不同厂商对 `dumpsys` 输出做了裁剪，字段名可能不同，不要把字段名当成稳定接口。

### 10.2 代码审计清单

1. **清单**：监听器服务是否声明了 `BIND_NOTIFICATION_LISTENER_SERVICE`？是否 `exported="false"`？
2. **能力**：是否真的需要 `getActiveNotifications()`、`cancelNotification()`、`snoozeNotification()`？每多一个能力，滥用后的影响就大一分。
3. **存储**：监听到的通知内容是否落盘、上报或写入日志？验证码、聊天内容是否被持久化？
4. **`PendingIntent`**：通知里的每个 `PendingIntent` 是否不可变？可变的是否绑定了显式 component？
5. **全屏**：是否使用了 `setFullScreenIntent`？应用是否真的属于通话/闹钟类？
6. **可见性**：是否设置了 `setVisibility`？敏感通知是否提供了 `publicVersion`？
7. **渠道**：渠道重要性是否被用来「绕过用户关闭」？是否在用户调整后反复重建渠道？
8. **权限自检**：发通知前是否调用 `areNotificationsEnabled()`？拒绝后是否有降级路径？

### 10.3 评估时的三个问题

- 这条缺陷落在**谁能读**、**谁能改**，还是**谁能看见**？三者的影响完全不同。
- 复现是否需要**工作资料/多用户**？单用户环境下的「无影响」不代表安全。
- 修复是补了权限检查，还是改了默认值？后者往往意味着同类入口还有遗漏。

## 11. 防御要点

### 11.1 应用开发者

- 通知内容按「锁屏可见」来写：验证码只显示必要位数，聊天内容用 `VISIBILITY_PRIVATE` 并提供 `publicVersion`。
- 通知里的 `PendingIntent` 一律 `FLAG_IMMUTABLE`，并显式指定 component；只有内联回复等确需接收方填充的场景才用 `FLAG_MUTABLE`。
- 不要用通知蹦床启动 Activity；`contentIntent` 直接指向目标 Activity。
- 全屏意图只用于通话与闹钟，并在使用前用 `canUseFullScreenIntent()` 检查。
- 发通知前检查 `areNotificationsEnabled()` 与渠道重要性，拒绝后提供应用内提示，而不是反复弹权限框。
- 不要把验证码、会话令牌、完整卡号写进通知；如果业务必须，至少不要让它进入 `publicVersion`。

### 11.2 通知监听类应用

- 只申请业务真正需要的监听能力，取消/延后等写操作要单独评估。
- 通知内容尽量在本地处理，不落盘、不上传；确需缓存时加密并设置短过期。
- 在 Android 15+ 上正确处理「验证码被脱敏」的情况，不要假设一定能读到完整内容。
- 向用户明示读取范围与保留策略；被撤销授权后立即停止处理并清理缓存。

### 11.3 平台与厂商

- 每个接收包名、uid、key 的通知服务入口都要核对调用者身份与目标归属。
- 跨用户/跨资料路径单独测试，不能只跑单用户用例。
- 锁屏、免打扰、投屏三种「不安全上下文」的可见性判定要有独立测试用例，覆盖用户设置的组合。
- 对监听器做内容级防护（如验证码脱敏），并对受信应用建立可审计的豁免名单。
- 前台服务通知与普通通知的权限路径分开测试，特别是用户拒绝 `POST_NOTIFICATIONS` 之后的降级行为。

### 11.4 企业环境

- 用设备策略限制工作资料通知向个人侧监听器的暴露（允许名单接口见第 4.3 节）。
- 把「通知使用权」纳入终端合规检查：非业务必需的监听器应被移除。
- 对自带设备（BYOD）明确通知数据的处理边界，避免企业数据通过个人监听器外流。

### 11.5 普通用户

- 定期检查「通知使用权」列表，移除不认识或不再使用的应用。
- 锁屏设置为「隐藏敏感内容」，验证码类通知不要常驻锁屏。
- Android 15+ 保持「增强通知」类保护开启；投屏时注意状态栏提示。
- 不要给来源不明的应用授予通知监听或通知策略访问权限。

## 12. 常见误传纠正

| 说法 | 纠正 |
|------|------|
| 通知服务跑在 SystemUI 进程里 | 错。`NotificationManagerService` 在 system_server；SystemUI 只是内部监听者与呈现方 |
| 通知监听授权后能读取所有用户的通知 | 错。只覆盖监听器有访问权的用户；工作资料里的监听器会被系统忽略 |
| Android 12 起 `PendingIntent` 默认不可变 | 错。Android 12 要求**显式指定**可变性，默认不可变是错误口径 |
| 用户拒绝 `POST_NOTIFICATIONS` 后应用完全发不出通知 | 错。前台服务通知仍会在任务管理器可见；媒体会话与自管通话通知豁免 |
| 全屏意图可以随意在锁屏弹出 | 错。Android 13 起解锁状态降级为 heads-up；Android 14 起 targetSdk 34 仅限通话/闹钟 |
| 渠道重要性创建后不能修改 | 不准确。用户未调整前应用可以改；用户调整后应用的修改通常不生效；用户可随时再改 |
| Android 15 禁止了第三方通知监听 | 错。只是对不受信监听器脱敏验证码；伴生设备管理器关联的受信应用豁免 |
| 气泡是 Android 11 才有的 | 错。气泡 API 29（Android 10）引入，Android 11 起对 targetSdk 30 收紧条件 |
| 通知里可以随便带 `file://` URI | 错。targetSdk 24+ 暴露 `file://` 可能抛 `FileUriExposedException`，应使用 `content://` 与临时授权 |
| 通知监听器能读短信数据库、安装应用 | 错。能力限于通知流与排序信息，不包含短信库或安装权限 |
| CVE-2021-39698 是 `PendingIntent` 漏洞 | 错。它是内核 `aio_poll_complete_work` 的 UAF，与通知无关 |
| `setPublicVersion()` 只影响锁屏 | 不准确。Android 15 投屏隐藏也优先使用公共版本 |
| 通知监听是普通运行时权限，弹窗即可授予 | 错。它是特殊访问权限，需要跳到系统设置页手动勾选 |
| `snoozeNotification` 只能延后自己应用的通知 | 错。历史上正是范围校验缺失导致可影响任意用户（CVE-2021-1020/1021，已修复） |

## 13. 未核实与失效来源

**A. `setPermittedCrossProfileNotificationListeners` 的空列表语义**
`DevicePolicyManager` 参考页内容过长，抓取时未包含该方法原文；正文只说明它是跨资料允许名单接口，未写「空列表/`null` 分别代表什么」。

**B. 通知排序的完整算法**
`RankingHelper` 的存在与职责已核实，但完整的排序权重、heads-up 的全部触发条件没有公开的逐条说明；正文避免给出具体优先级公式。

**C. 各 CVE 的在野利用状态**
Android 公告与 NVD 多数条目未逐条说明是否在野利用；正文只用「公开且已修复」，不推断利用情况。

**D. 中文「通知使用权」的 AOSP 字符串**
Android 13+ 英文界面名称为「Device & app notifications」，中文名称随厂商翻译；「通知使用权」是常见译名，未核实为 AOSP 固定字符串。

**E. Android 16 的通知行为变化**
本文只写到 Android 15/15 QPR1；Android 16 的相关变更（如新的实时更新样式）未纳入核验范围，正文不提。

**F. `cmd notification` 子命令全集**
正文只使用官方文档中出现过的 `post` 语法；`allow_listener` 等子命令是否在所有版本可用未逐版核实。

**G. 厂商定制差异**
三星、小米等厂商可能对通知监听白名单、锁屏隐藏、后台限制做额外定制；正文按 AOSP 口径描述，不覆盖厂商差异。

**H. 验证码识别算法**
Android 15 的「OTP 检测」具体启发式规则未公开；正文只描述「已识别出的验证码会被脱敏」，不猜测识别条件。

**I. 低内存设备的判定阈值**
官方文档只说明 Android 10 及以下的低内存设备可能不绑定监听器，具体内存阈值未核实。

**J. 通知速率限制**
通知服务存在频率限制与折叠策略，但具体阈值与不同版本的差异未在一手文档中找到可引用的精确值。

**K. 可变 `PendingIntent` 与内联回复的兼容矩阵**
正文给出「确需接收方填充时用可变」的原则，但不同 Android 版本、不同通知样式下的精确兼容行为未逐项核验。

**L. 投屏隐藏的回退顺序**
Android 15 文档说明有 `publicVersion` 时显示公共版本、否则脱敏；但不同投屏方式（系统投屏、第三方录屏、无线显示）是否完全一致未核实。

## 14. 参考资料

以下链接为本文核验时使用的一手来源（按首次引用顺序）：

- NotificationManagerService（AOSP）：
  https://android.googlesource.com/platform/frameworks/base/+/refs/heads/main/services/core/java/com/android/server/notification/NotificationManagerService.java
- NotificationListenerService（官方参考）：
  https://developer.android.com/reference/android/service/notification/NotificationListenerService
- NotificationAssistantService（AOSP）：
  https://android.googlesource.com/platform/frameworks/base/+/refs/heads/main/core/java/android/service/notification/NotificationAssistantService.java
- NotificationChannel（官方参考）：
  https://developer.android.com/reference/android/app/NotificationChannel
- 通知渠道指南：
  https://developer.android.com/develop/ui/views/notifications/channels
- POST_NOTIFICATIONS 权限指南：
  https://developer.android.com/develop/ui/views/notifications/notification-permission
- PendingIntent（官方参考）：
  https://developer.android.com/reference/android/app/PendingIntent
- Android 12 行为变化（通知蹦床）：
  https://developer.android.com/about/versions/12/behavior-changes-12
- Android 14 行为变化（全屏意图、隐式 Intent、BAL）：
  https://developer.android.com/about/versions/14/behavior-changes-14
- Android 15 行为变化（验证码脱敏、投屏隐藏）：
  https://developer.android.com/about/versions/15/behavior-changes-all
- Notification.Builder#setFullScreenIntent（官方参考）：
  https://developer.android.com/reference/android/app/Notification.Builder#setFullScreenIntent(android.app.PendingIntent,%20boolean)
- Notification.BubbleMetadata（官方参考）：
  https://developer.android.com/reference/android/app/Notification.BubbleMetadata
- Notification（可见性与 publicVersion）：
  https://developer.android.com/reference/android/app/Notification
- Settings（通知监听设置入口）：
  https://developer.android.com/reference/android/provider/Settings#ACTION_NOTIFICATION_LISTENER_SETTINGS
- FileUriExposedException（官方参考）：
  https://developer.android.com/reference/android/os/FileUriExposedException
- Android Security Bulletin 2021-12（Pixel）：
  https://source.android.com/docs/security/bulletin/pixel/2021-12-01
- Android Security Bulletin 2022-12：
  https://source.android.com/docs/security/bulletin/2022-12-01
- Android Security Bulletin 2020-08：
  https://source.android.com/docs/security/bulletin/2020-08-01
- Android Security Bulletin 2020-09：
  https://source.android.com/docs/security/bulletin/2020-09-01
- NVD CVE 查询（本文台账的编号、CVSS 与描述来源）：
  https://nvd.nist.gov/vuln/detail/CVE-2022-20477
  https://nvd.nist.gov/vuln/detail/CVE-2021-1019
  https://nvd.nist.gov/vuln/detail/CVE-2021-1020
  https://nvd.nist.gov/vuln/detail/CVE-2021-39698

---

## 联系方式

咨询 Android 系统详情请联系 https://t.me/one00190

---

*本手册仅供安全学习与防御研究使用；所有机制表述、公告列值与 CVE 描述均取自公开一手来源并附链接，未能核实的部分已单列于「未核实与失效来源」而非写入正文。*
