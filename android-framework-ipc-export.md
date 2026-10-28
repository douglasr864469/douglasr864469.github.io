---
layout: page
title: "Android 框架 IPC 与组件导出攻击面手册"
description: "组件导出与权限属性、640 条 `<protected-broadcast>` 实名清单口径、PendingIntent 与后台活动启动（BAL）规则的现行形态、ContentProvider 与四类 URI 授权标志、Deep Link 与 App Links 验证、Task 劫持的官方术语与 opt..."
date: 2026-10-27
permalink: /android-framework-ipc-export/
---

> 本手册按「组件导出模型 → 广播 → PendingIntent 与后台启动 → ContentProvider 与 URI 授权 → Deep Link/App Links → Task 与 Intent 重定向」组织 Android 框架跨进程组件交互的官方成文规则与公开案例。视角是防御与教育：只讲属性语义、常量实名与自查方法，不含任何可用于攻击的代码、步骤或载荷样例。全部结论取自 AOSP 源码与 developer.android.com / source.android.com 公开一手来源并附链接，未能核实的表述统一置于 §11。

---

## 0. 使用方式与边界

- **读者**：应用与系统组件开发者、安全评审人员、平台维护者。
- **范围**：`android:exported` 导出模型、intent filter 匹配、运行时广播注册标志、protected broadcast、`PendingIntent` 可变性与后台活动启动（BAL）限制、`ContentProvider` 权限与 URI 临时授权、App Links 域验证、Task 劫持与 Intent 重定向的官方风险页与加固动作。
- **不含**：攻击链、绕过细节、PoC 或载荷；处置方向一律指向官方风险页的 Mitigations 章节。
- **口径**：常量与行号为核验当日（2026-10-27）AOSP `main` / `androidx-main` 分支快照。公告链接只用现行有效形式 `https://source.android.com/docs/security/bulletin/YYYY-MM-01`（带年份目录层与旧 `-05` 日期形式实测 404，见 §11）。案例严重级别做**公告 vs NVD 双轨标注**，分歧显式写出、不做调和。

---

## 1. 组件导出模型

### 1.1 `android:exported` 与 Android 12+ 强制声明

组件能否被其他应用启动或访问，由 manifest 的 `android:exported` 决定。对 provider，官方 `<provider>` 文档的语义更精确：`"true"` 时任何应用可依 provider 声明的权限使用其 content URI；`"false"` 时仅同 UID 应用，或经 `android:grantUriPermissions` 临时获授的应用可访问（§4.1）。provider 不参与 intent filter 匹配。

《Behavior changes: Apps targeting Android 12 or higher》（https://developer.android.com/about/versions/12/behavior-changes-12#exported ）逐字：

> "…contains activities, services, or broadcast receivers that use intent filters, you must explicitly declare the android:exported attribute for these app components. Warning: If an activity, service, or broadcast receiver uses intent filters and doesn't have an explicitly-declared value for android:exported, your app can't be installed on a device that runs Android 12 or higher. If the app component includes the LAUNCHER category, set android:exported to true. In most other cases, set android:exported to false."

即**带 `<intent-filter>` 的 activity/service/receiver 必须显式声明 `android:exported`，否则无法安装到 Android 12+ 设备**；含 LAUNCHER category 才置 `true`。同页给出 lint 实名 "When using intent filters, please specify android:exported as well" 与构建错误实名 "Manifest merger failed : Apps targeting Android 12 and higher are required to specify an explicit value for android:exported…"。成文边界：该强制句式未把 content provider 列入。

### 1.2 IntentFilter 匹配三分面

《Intents and intent filters》（https://developer.android.com/guide/components/intents-filters ）："Implicit intents do not name a specific component, but instead declare a general action to perform, which allows a component from another app to handle it."；系统按 Action test / Data test（URI 与数据类型）/ Category test 三面匹配。`Intent.java` javadoc 补充：`Context.startActivity` 解析未指定组件的 Intent 时必须带 `DEFAULT` category；`component` 字段为 "Specifies an explicit name of a component…"。防御含义：导出且带 intent filter 的组件即处于系统解析路径、接收外部可控数据，配置须最小化。

---

## 2. 广播

### 2.1 `RECEIVER_*` 真实常量

AOSP `core/java/android/content/Context.java`（main）逐字核实：

| 常量 | 值 | 行号 | 官方语义（节译） |
| --- | --- | --- | --- |
| `RECEIVER_VISIBLE_TO_INSTANT_APPS` | `0x1` | L768 | 是否对 Instant Apps 可见 |
| `RECEIVER_EXPORTED` | `0x2` | L770-774 | "The receiver can receive broadcasts from other Apps. Has the same behavior as marking a statically registered receiver with \"exported=true\"" |
| `RECEIVER_NOT_EXPORTED` | `0x4` | L784-788 | "The receiver cannot receive broadcasts from other Apps. Has the same behavior as marking a statically registered receiver with \"exported=false\"" |

另有 `RECEIVER_EXPORTED_UNAUDITED = RECEIVER_EXPORTED`（L782，已 `@Deprecated`、`@hide`/`@TestApi`），不属于面向开发者的现行 API。

### 2.2 Android 14（targetSdk 34 = `UPSIDE_DOWN_CAKE`）强制原文

`registerReceiver` javadoc（L3324-3331）逐字：

> "For apps targeting {@link android.os.Build.VERSION_CODES#UPSIDE_DOWN_CAKE}, either RECEIVER_EXPORTED or RECEIVER_NOT_EXPORTED must be specified if the receiver is not being registered for system broadcasts or a SecurityException will be thrown."

《Behavior changes: Apps targeting Android 14》（https://developer.android.com/about/versions/14/behavior-changes-14 ）"Runtime for broadcasts" 节同义成文并给出豁免："Exception for receivers that receive only system broadcasts: …then it shouldn't specify a flag when registering the receiver." 最佳实践页（旧 URL /guide/components/broadcasts 现 301 至 /develop/background-work/background-tasks/broadcasts）逐字："you export the broadcast receiver, other apps could send unprotected broadcasts to your app."；对来自系统或其他应用（含自家其他应用）的广播使用 `RECEIVER_EXPORTED`。

### 2.3 有序广播与 sender 权限

`Context.java` javadoc：有序广播 "delivering them one at a time to allow more preferred receivers to consume the broadcast before it is delivered to less preferred receivers"；`resultReceiver` 为最终接收者，`initialCode` 为初始结果码（常为 `Activity.RESULT_OK`）；`receiverPermission` 是接收方必须持有的权限，"If null, no permission is required"——发送侧不加限定即对所有匹配接收器开放。

### 2.4 protected broadcast：实名以 AOSP manifest 的 640 条为准

权威清单是 `core/res/AndroidManifest.xml`（https://android.googlesource.com/platform/frameworks/base/+/refs/heads/main/core/res/AndroidManifest.xml ），当前含 **640** 条 `<protected-broadcast android:name="…"/>`。逐字样例：`android.intent.action.SCREEN_OFF`、`android.intent.action.SCREEN_ON`、`android.intent.action.USER_PRESENT`、`android.intent.action.TIME_SET`、`android.intent.action.TIME_TICK`、`android.intent.action.TIMEZONE_CHANGED`、`android.intent.action.DATE_CHANGED`、`android.intent.action.PRE_BOOT_COMPLETED`。

语义：清单内 action 由系统签名进程独占发送，第三方应用无法伪造；应用自定义隐式 action 不在清单内，一律按外部可控对待。集中罗列这些实名的官方专页并不存在（相关 URL 实测 404，§11），实名以该 manifest 为准。

---

## 3. PendingIntent 与后台活动启动（BAL）

### 3.1 `FLAG_IMMUTABLE` / `FLAG_MUTABLE` 与 Android 12 强制

AOSP `core/java/android/app/PendingIntent.java`（L256-289）：`public static final int FLAG_IMMUTABLE = 1<<26;`——"the additional intent argument passed to the send methods to fill in unpopulated properties of this intent will be ignored"；`public static final int FLAG_MUTABLE = 1<<25;`——"Up until R, PendingIntents are assumed to be mutable by default, unless FLAG_IMMUTABLE is set. Starting with S, it will be required to explicitly specify the mutability of PendingIntents on creation with either FLAG_IMMUTABLE or FLAG_MUTABLE. It is strongly recommended to use FLAG_IMMUTABLE …"

同页限定语："FLAG_IMMUTABLE only limits the ability to alter the semantics of the intent that is sent by send by the invoker of send. The creator of the PendingIntent can always update the PendingIntent itself via FLAG_UPDATE_CURRENT." 另有 `FLAG_MUTABLE_UNAUDITED = FLAG_MUTABLE`（`@Deprecated`、`@hide`）。Android 12 targeting 页："you must specify the mutability of each PendingIntent object that your app creates. This additional requirement improves your app's security."，对应 lint 实名 "Missing PendingIntent mutability flag [UnspecifiedImmutableFlag]"。

### 3.2 官方风险页实名

- **《Intent redirection》**（https://developer.android.com/privacy-and-security/risks/intent-redirection ）："An intent redirection occurs when an attacker can partly or fully control the contents of an intent used to launch a new component in the context of a vulnerable app… it might access private components like unexported ContentProvider objects." Mitigations 逐字："It's important to remember to check or clear flags (FLAG_GRANT_READ_URI_PERMISSION, FLAG_GRANT_WRITE_URI_PERMISSION, FLAG_GRANT_PERSISTABLE_URI_PERMISSION, and FLAG_GRANT_PREFIX_URI_PERMISSION), and to check where the intent is being redirected. IntentSanitizer can help with this process."
- **《Pending Intents》**（/privacy-and-security/risks/pending-intent ）："A PendingIntent is a reference to a token maintained by the system. Application A can pass a PendingIntent to application B in order to allow application B to execute predefined actions on behalf of application A; regardless of whether application A is still alive."；"Risk: Mutable Pending Intents …the unfilled fields of a PendingIntent can be modified by a malicious app and allow access to otherwise non-exported components of the vulnerable application."
- **《Sender of Pending Intents》**（/risks/sender-of-pending-intents ）。同目录另有《Implicit Intent hijacking》《Improperly Exposed Directories to FileProvider》《Improperly trusting ContentProvider-provided file…》《Permission-based access control to exported components》《Path traversal》《Sticky broadcast》。

Google 针对 intent redirection 的"赏金说明"官方本体未找到，本手册不引用，记为未核实（§11）。

### 3.3 BAL 成文规则与例外条件

《Restrictions on starting activities from the background》（https://developer.android.com/guide/components/activities/background-starts ）："A Background Activity Launch (BAL) occurs when an app not in the foreground, with no visible activities, or a PendingIntent received from a different app attempts to start a new activity." 成文例外条件：应用有可见窗口；应用是当前输入法（IME）；活动由系统发送的 PendingIntent 启动（如点击通知）；已授予 `SYSTEM_ALERT_WINDOW`；已授予 `START_ACTIVITIES_FROM_BACKGROUND`；被已获准后台启动的服务绑定；由桌面启动器发起（点击图标或 widget 交互）；来自必须常驻的系统核心部分（如 Telephony 拉起来电界面）。

同页 "New hardening and required opt-ins"："Android has introduced stricter rules requiring explicit opt-ins for apps that use PendingIntent and IntentSender to launch activities. A launch is only permitted if either the app that created the PendingIntent or the app that sends it opts-in to grant its background launch privileges."；"For the purpose of these BAL restrictions, IntentSender and PendingIntent are treated similarly." 失败观测：发起方不收到返回值或异常，系统内部向 Logcat 记录 `"Background activity launch blocked!"`（§9.3）。

### 3.4 `ActivityOptions` 的 BAL 模式常量（纠正误记位置）

**`MODE_BACKGROUND_ACTIVITY_START_*` 定义在 `ActivityOptions`（`core/java/android/app/ActivityOptions.java` L119-174），不是 `ActivityManager`**——对 `ActivityManager.java` grep `BACKGROUND_ACTIVITY_START` 无任何符号：

| 常量 | 值 | 状态 |
| --- | --- | --- |
| `…SYSTEM_DEFINED` | `0` | 现行 |
| `…ALLOWED` | `1` | main 分支已 `@Deprecated`、带 `@FlaggedApi(Flags.FLAG_BAL_ADDITIONAL_START_MODES)`，被 3/4 号细分模式取代 |
| `…DENIED` | `2` | 现行 |
| `…ALLOW_ALWAYS` | `3` | `@FlaggedApi` 细分模式 |
| `…ALLOW_IF_VISIBLE` | `4` | `@FlaggedApi` 细分模式 |

（前缀同为 `MODE_BACKGROUND_ACTIVITY_START`。）`setPendingIntentBackgroundActivityStartMode(int)` javadoc："A privileged sender of a PendingIntent should only grant MODE_BACKGROUND_ACTIVITY_START_ALLOWED if the PendingIntent is from a trusted source and/or executed on behalf the user."；旧 API `setPendingIntentBackgroundActivityLaunchAllowed(boolean)` 已 `@Deprecated`。Android 14 targeting 页 opt-in："the app must opt in if it wants to grant its own background activity launch privileges to start the pending intent… pass an ActivityOptions bundle with setPendingIntentBackgroundActivityStartMode(MODE_BACKGROUND_ACTIVITY_START_ALLOWED)." "notification trampoline" 为 Android 12 targeting 页实名术语（"you should avoid this UX pattern and use a PendingIntent instead"）。

---

## 4. ContentProvider 与 URI 授权

### 4.1 manifest 属性语义

《Manifest `<provider>`》（https://developer.android.com/guide/topics/manifest/provider-element ）逐字要点：

- `android:permission`："The name of a permission that clients must have to read or write the content provider's data. This attribute is a convenient way of setting a single permission for both reading and writing."
- `android:readPermission`："A permission that clients must have to query the content provider. If the provider sets android:grantUriPermissions to \"true\", or if a given client satisfies the conditions of a <grant-uri-permission> subelement, the client can gain temporary read access…"（`android:writePermission` 句式对应）。
- `android:grantUriPermissions`："Whether those who ordinarily don't have permission to access the content provider's data can be granted permission to do so, temporarily overcoming the restriction imposed by the readPermission, writePermission, permission, and exported attributes. …If \"false\", permission can be granted only to the data subsets listed in <grant-uri-permission> subelements, if any. The default value is \"false\"."

四者构成访问控制矩阵：`exported` 决定谁可达，`permission`/`readPermission`/`writePermission` 决定常态读写门槛，`grantUriPermissions`（默认 `false`）决定是否可对具体 URI 突破上述限制做临时授权。同页另有受 targetSdk 影响的默认值说明（"Because this attribute was introduced in API level 17…"）。

### 4.2 四个 `FLAG_GRANT_*_URI_PERMISSION`：现行源码无 `@Deprecated`

`Intent.java`：`FLAG_GRANT_READ_URI_PERMISSION = 0x00000001`（L7153，"the recipient of this Intent will be granted permission to perform read operations on the URI in the Intent's data and any URIs specified in its ClipData"）、`FLAG_GRANT_WRITE_URI_PERMISSION = 0x00000002`（L7162，同句式 write 版）、`FLAG_GRANT_PERSISTABLE_URI_PERMISSION = 0x00000040`（L7208，关联 `ContentResolver#takePersistableUriPermission`）、`FLAG_GRANT_PREFIX_URI_PERMISSION = 0x00000080`（L7219，"…only when scheme, authority, and all path segments defined by the prefix are an exact match."）。

核验结论：main 分支（Android 16 时代源码）上这 4 个标志**均无 `@Deprecated`、未被移除**；`FLAG_ACTIVITY_LAUNCH_ADJACENT = 0x00001000` 是独立的分屏窗口标志（"This flag is only used for split-screen multi-window mode…"），与 URI 权限无关。"Android 16 用 LAUNCH_ADJACENT 替代/废弃 URI 权限标志"不成立。

### 4.3 `FileProvider` 与 `content://` 授权

androidx `core/core/src/main/java/androidx/core/content/FileProvider.java`：以 "creating a content:// Uri for a file" 的方式分享应用文件；接收方 "can open the file and access its contents by calling ContentResolver.openFileDescriptor(Uri, String) to get a ParcelFileDescriptor"；`getUriForFile(getContext(), "com.mydomain.fileprovider", newFile)` 返回 `content://com.mydomain.fileprovider/my_images/default_image.jpg`；临时授权可经 `Context.grantUriPermission(package, Uri, mode_flags)` 或 Intent 携带 grant 标志完成。

官方风险页《Improperly Exposed Directories to FileProvider》（https://developer.android.com/privacy-and-security/risks/file-providers ）逐字："An improperly configured FileProvider can unintentionally expose files and directories to an attacker…an application with <root-path> set in the configuration could enable an attacker to access sensitive information…"。Mitigations："Do not use the <root-path> path element in the configuration. <root-path> corresponds to the root directory of the device ( / )."、"Share narrow path ranges…avoid sharing a broad path range like . or /."

### 4.4 服务端逐次调用 enforcement

`ContentProvider.java` 构造参数含 `@param readPermission The read permission you want this instance should have in the…`；其 `Transport` 按 `mReadPermission`/`mWritePermission` 并结合 AppOps（`AppOpsManager.permissionToOpCode(mReadPermission)`，L830/L848）在**每次 `query`/`openFile` 跨进程调用时**于服务端检查，另有 `Manifest.permission.GET_ANY_PROVIDER_TYPE` 特殊豁免分支（L318-321）。即权限并非"注册即一次性生效"，读写权限与授权路径范围必须与业务一致。

---

## 5. Deep Link 与 App Links

### 5.1 匹配常量与 Digital Asset Links

`Intent.java`：L710 `public static final String ACTION_VIEW = "android.intent.action.VIEW";`；L5471 `public static final String CATEGORY_BROWSABLE = "android.intent.category.BROWSABLE";`。链接型 intent filter 依赖这两个常量，导出面由此形成。

《Verify Android App Links》（https://developer.android.com/training/app-links/verify-android-applinks ）逐字：站点侧需 "the Digital Asset Links file at https://<hostname>/.well-known/assetlinks.json."；开关语义 "When android:autoVerify=\"true\" is present in at least one of your app's intent filters, installing your app on a device that runs Android 6.0 (API level 23) or higher causes the system to automatically verify the hosts associated with the URLs in your app's intent filters. On Android 12 and higher, you can also invoke the verification process manually…"；完整性 "On Android 11 (API level 30) and lower, the system establishes your app as the default handler for the specified URL patterns only if it finds a matching Digital Asset Links file for all hosts in the manifest."；时延 "Android 15 …The system periodically re-verifies domains in the background. Changes can take up to seven days to propagate to all end-user devices… Android 14 (API level 34) and lower: The system does not perform periodic background re-verification."

### 5.2 Android 12+ 的 web intent 解析收紧

《Behavior changes: all apps》"Web intent resolution" 节逐字："Starting in Android 12 (API level 31), a generic web intent resolves to an activity in your app only if your app is approved for the specific domain contained in that web intent. If your app isn't approved for the domain, the web intent resolves to the user's default browser app instead… In your app's intent filters, check that you include the BROWSABLE category and support the https scheme."

### 5.3 默认应用与 `RoleManager` 观测点

官方 reference（https://developer.android.com/reference/android/app/role/RoleManager ）实名常量：`ROLE_BROWSER`（"Added in API level 29 … The name of the browser role. … Constant Value: \"android.app.role.BROWSER\""）、`ROLE_ASSISTANT`、`ROLE_DIALER`、`ROLE_SMS`、`ROLE_HOME`、`ROLE_NOTES`（API 34）、`ROLE_WALLET`（API 35）；方法 `createRequestRoleIntent(String)`、`isRoleAvailable(String)`、`isRoleHeld(String)`。`Settings.java`（main）另有 L716 `ACTION_APP_OPEN_BY_DEFAULT_SETTINGS` 与 L2315 `ACTION_MANAGE_DEFAULT_APPS_SETTINGS`。

---

## 6. Task 与 Intent 重定向

### 6.1 "task hijacking" 是官方用词

BAL 文档页逐字："Android protects users from malicious apps and provides a trustworthy UI experience. The Activity Security framework encompasses rules and platform restrictions. These rules and restrictions prevent unwanted UI interruptions, task hijacking, and other security threats." 即 "task hijacking" 为官方术语，成文规则位于 background-starts 页，而非某个独立专页。

### 6.2 Activity Sandwich 与 targetSdk 37 的 opt-in

同页 "Task Hijacking Prevention" 逐字："To prevent in-task hijacking attacks (like the \"Activity Sandwich\"), Android 15 introduces new rules for apps targeting API level 37 or higher. Rule 1: Within a single task, an activity can only be launched by another activity that belongs to the same application (that is, has the same UID) as the current top-most activity in the task. Rule 2: Only an activity within a foreground task that matches the UID of the top-most activity is allowed to create a new task or bring a different, existing task to the foreground." opt-in："This behavior can be enabled starting with target SDK 37, you must explicitly opt-in to enable. It is designed to prevent in-task hijacking (or Activity Sandwiching)…"；启用方式 "To opt-in and enable ASM for your application, set the android:allowCrossUidActivitySwitchFromBelow…"（ASM = 该页对 Activity Sandwich Mitigation 的缩写）。该页 Warning：启用可能阻断合法的 assistant/系统服务跨 UID 启动，故为显式 opt-in。

### 6.3 任务语义的实名控制项

`FLAG_ACTIVITY_NEW_TASK = 0x10000000`（`Intent.java` L7290），同节另核实 "This flag can not be used when the caller is requesting a result from the activity being launched."。《Tasks and back stack》对 `singleTask` 逐字："The system creates the activity at the root of a new task or locates the activity on an existing task with the same affinity. If an instance of the activity already exists, the system routes the intent to the existing instance through a call to its onNewIntent() method, rather than creating a new instance." 成文控制项实名：`launchMode`、`allowTaskReparenting`、`clearTaskOnLaunch`、`alwaysRetainTaskState`、`finishOnTaskLaunch`；flags：`FLAG_ACTIVITY_NEW_TASK`、`FLAG_ACTIVITY_CLEAR_TOP`、`FLAG_ACTIVITY_SINGLE_TOP`。

### 6.4 Intent 重定向的防御面

依《Intent redirection》风险页：应用若用外部可控的 Intent 启动组件，等效于以该应用身份触达其非导出组件与私有 `ContentProvider`。官方处置方向为校验或清除四个 `FLAG_GRANT_*_URI_PERMISSION`、确认重定向目标、使用 `IntentSanitizer`，并与 `FLAG_IMMUTABLE`（§3.1）、BAL opt-in（§3.4）、task 规则（§6.2）构成纵深。本手册不描述任何构造重定向请求的方法或场景细节。

---

## 7. 公开案例

约定：**EoP** = Escalation of privilege、**ID** = Information disclosure。「严重级别」列做公告 vs NVD 双轨标注，分歧以粗体标出且不调和；后三行为仅有 NVD 的旧公告期/应用侧案例，如实标注。

| CVE | 组件或文件 | 类型 | 严重级别（公告 vs NVD 双轨） | AOSP 版本 | A-\<bug id\> | 公告日期与链接 |
| --- | --- | --- | --- | --- | --- | --- |
| CVE-2022-20116 | `OngoingCallController.kt`（`onEntryUpdated`） | Intent redirection / EoP | 公告 High vs NVD HIGH（一致） | 12, 12L | A-212467440 | 2022-05-01 · System · https://source.android.com/docs/security/bulletin/2022-05-01 |
| CVE-2021-1024 | `EventResultPersister.java`（`onEventReceived`，confused deputy） | Intent redirection / EoP | 公告 Moderate（Pixel 页）vs NVD MEDIUM 口径，两侧原文并存 | 11/12（Pixel 收录） | A-191283525 | 2021-12-01 · Pixel Update Bulletin · https://source.android.com/docs/security/bulletin/pixel/2021-12-01 |
| CVE-2020-0114 | `KeyguardSliceProvider.java`（`onCreateSliceProvider`） | PendingIntent 缺陷 → confused deputy / EoP（以 System UI 身份执行） | 公告 High vs NVD HIGH（一致） | 10 | A-147606347 | 2020-06-01 · Framework · https://source.android.com/docs/security/bulletin/2020-06-01 |
| CVE-2020-0188 | `SettingsSliceProvider.java`（`onCreatePermissionRequest`） | PendingIntent 权限绕过 / EoP | **公告 Moderate vs NVD HIGH/MEDIUM 两版评分并存（分歧）** | 10 | A-147355897 | 2020-06-01 · Pixel Update Bulletin · https://source.android.com/docs/security/bulletin/pixel/2020-06-01 |
| CVE-2020-0248 | `InstantAppNotifier.java`（`postInstantAppNotif`） | PendingIntent 错误 → 权限绕过 / ID | **公告 High vs NVD MEDIUM（分歧）** | 10 | A-154627439 | 2020-08-01 · Framework · https://source.android.com/docs/security/bulletin/2020-08-01 |
| CVE-2022-20470 | `AppWidgetServiceImpl.java`（`bindRemoteViewsService`） | BAL 绕过（improper input validation）/ EoP | 公告 High vs NVD HIGH 7.8（一致） | 10, 11, 12, 12L, 13 | A-234013191 | 2022-12-01 · Framework · https://source.android.com/docs/security/bulletin/2022-12-01 |
| CVE-2023-20950 | `AlarmManagerActivity.java` | 经 PendingIntent 绕过 BAL 限制 / EoP | **公告 Moderate vs NVD HIGH 7.8（分歧实例 A）** | 11, 12, 12L | A-195756028 | 2023-04-01 · Framework · https://source.android.com/docs/security/bulletin/2023-04-01 |
| CVE-2023-21081 | `PackageInstallerService.java` 及相关文件 | BAL 限制绕过（逻辑错误）/ EoP | 公告 High vs NVD HIGH（一致） | 11, 12, 12L, 13 | A-230492955 | 2023-04-01 · Framework · https://source.android.com/docs/security/bulletin/2023-04-01 |
| CVE-2023-21088 | `LocationProviderManager.java`（`deliverOnFlushComplete`） | BAL 限制绕过（逻辑错误）/ EoP | 公告 High vs NVD HIGH（一致） | 12, 12L, 13 | A-235823542 | 2023-04-01 · Framework · https://source.android.com/docs/security/bulletin/2023-04-01 |
| CVE-2023-40127 | MediaProvider（`packages/providers/MediaProvider`，confused deputy，可访问截图） | ContentProvider 授权 / ID | **公告 High vs NVD LOW 3.3（分歧实例 B）** | 11, 12, 12L, 13 | A-262244882 | 2023-10-01 · https://source.android.com/docs/security/bulletin/2023-10-01 |
| CVE-2014-8610 | `AndroidManifest.xml` 中 `SmsReceiver` 未要求 `SEND_SMS` 权限 | 接收器权限缺失 | **仅有 NVD**（Android 5.0 前旧公告期格式，现行公告表无该行） | Android 5.0.0 之前 | 无 A- bug id | 无现行公告行；仅 NVD：https://nvd.nist.gov/vuln/detail/CVE-2014-8610 |
| CVE-2021-33699 | 应用 `AndroidManifest.xml` 的 Task Control 特性错误配置 | Task Hijacking（**应用侧，仅有 NVD**；第三方学术编号，非 AOSP 公告条目） | 仅有 NVD，无公告评级可比 | 不套 AOSP 版本轨 | 无 | https://nvd.nist.gov/vuln/detail/CVE-2021-33699 |
| CVE-2024-34405 | McAfee Security: Antivirus VPN for Android（8.3.0 之前） | Deep Link 校验不当，可在应用内启动任意 URL（**应用侧，仅有 NVD**） | 仅有 NVD，无公告评级可比 | 第三方应用 | 无 | https://nvd.nist.gov/vuln/detail/CVE-2024-34405 |

**双轨评级的意义**：CVE-2023-20950（公告 Framework/EoP **Moderate** vs NVD **HIGH 7.8**）与 CVE-2023-40127（公告 **ID High** vs NVD **LOW 3.3**）说明两套体系不可互替：Google 公告评级含 Android 威胁模型与处置优先级口径，NVD 仅按 CVSS 基础向量计算，引用时保留各自原文。

**CISA KEV 未收录**：以上 13 条在 2026-10-27 KEV 快照（https://www.cisa.gov/sites/default/files/feeds/known_exploited_vulnerabilities.json ，快照 1,810,928 字节，以已知条目 CVE-2014-0160 做有效性自检命中正常）中全部 0 命中，故一律标注「CISA KEV 未收录」。推论边界：**未收录 ≠ 无真实利用**。KEV 只收录 CISA 认定"已知被利用"且通常面向企业/政府处置优先级的条目；Android 设备端在野利用更多由 Google Threat Analysis Group 经《Android Security Impressions Report》等渠道披露，不必然进入 KEV。因此只能表述为"截至快照日期未见于 KEV 清单"。

**分布解读**：10 条 AOSP 条目集中在 Framework 的 PendingIntent / BAL / Intent redirection 与 ContentProvider 授权语义，多呈 confused deputy（以系统或其他应用身份执行动作）形态；其余 3 条覆盖接收器权限缺失、Task 控制项错误配置与 Deep Link 校验。AOSP 公告中未检索到 autoVerify/域验证劫持类条目（§11）。

---

## 8. 常见误传纠正

| # | 误传（不成立） | 纠正与一手依据 |
| --- | --- | --- |
| 1 | "Android 12+ 对所有组件都必须写 `android:exported`" | 强制句式只针对使用 intent filter 的 activity/service/receiver；provider 不经 intent filter 匹配，其 `exported` 语义由 `<provider>` 文档描述 |
| 2 | "不声明 `android:exported` 只是 lint 警告，不影响安装" | 原文："your app can't be installed on a device that runs Android 12 or higher"；另有构建期实名 "Manifest merger failed…" |
| 3 | "`RECEIVER_EXPORTED` 与 `RECEIVER_NOT_EXPORTED` 数值相同或可互换" | 实名值 `0x1`（`RECEIVER_VISIBLE_TO_INSTANT_APPS`）/`0x2`/`0x4`，语义分别等同静态 `exported=true`/`exported=false`（`Context.java` L768/L770-774/L784-788） |
| 4 | "Android 14 注册任何接收器都必须带导出标志" | 仅为系统广播注册时不应指定该标志；非系统广播场景缺失会抛 `SecurityException`（javadoc L3324-3331） |
| 5 | "`RECEIVER_EXPORTED_UNAUDITED` 是面向开发者的现行 API" | 它是 `RECEIVER_EXPORTED` 别名，已 `@Deprecated` 且 `@hide`/`@TestApi`（L782） |
| 6 | "存在一个官方专页集中罗列 protected broadcast 实名" | 该类专页 URL 实测 404；实名以 `core/res/AndroidManifest.xml` 的 **640** 条 `<protected-broadcast>` 为准（专页名与失效 URL 见 §11） |
| 7 | "第三方应用也能发送 `android.intent.action.SCREEN_ON` 这类系统 action" | 清单内 action 由系统签名进程独占发送；应用自定义隐式 action 应按外部可控对待 |
| 8 | "`FLAG_MUTABLE` 自 Android 10 起就是强制要求" | javadoc："Up until R, PendingIntents are assumed to be mutable by default… Starting with S, it will be required to explicitly specify the mutability…"，强制始于 Android 12（S） |
| 9 | "设了 `FLAG_IMMUTABLE` 后连创建方也无法改变 PendingIntent" | 原文限定 "…by the invoker of send. The creator of the PendingIntent can always update the PendingIntent itself via FLAG_UPDATE_CURRENT." |
| 10 | "`MODE_BACKGROUND_ACTIVITY_START_ALLOWED` 定义在 `ActivityManager`，且仍是现行推荐模式" | 定义在 **`ActivityOptions`**（L119-174），`ActivityManager.java` grep 无该符号；main 分支上值 `1` 已 `@Deprecated`，由 `…ALLOW_ALWAYS(3)`/`…ALLOW_IF_VISIBLE(4)` 细分；`SYSTEM_DEFINED=0`、`DENIED=2` 仍在 |
| 11 | "IntentSender 不受 BAL opt-in 约束，只有 PendingIntent 受影响" | 原文："For the purpose of these BAL restrictions, IntentSender and PendingIntent are treated similarly." |
| 12 | "Android 16 用 `FLAG_ACTIVITY_LAUNCH_ADJACENT` 替代或废弃了 URI 权限标志" | 不成立：四个 grant 标志（`0x1`/`0x2`/`0x40`/`0x80`）现行源码**无 `@Deprecated`、未移除**；`FLAG_ACTIVITY_LAUNCH_ADJACENT = 0x00001000` 是独立分屏窗口标志（"only used for split-screen multi-window mode"），与 URI 权限无关 |
| 13 | "`grantUriPermissions` 默认 true，任意 URI 都可临时授权" | 原文："The default value is \"false\"."；为 `false` 时仅可对 `<grant-uri-permission>` 列出的数据子集授权 |
| 14 | "`android:permission` 与 `readPermission`/`writePermission` 等价，写哪个都一样" | `permission` 是读写共用单一权限的便捷写法；另两者分轨约束 query 与写路径；enforcement 位于服务端每次跨进程调用（`ContentProvider.Transport` + AppOps） |
| 15 | "'task hijacking' 是社区臆造词，官方没有成文规则" | "task hijacking" 为官方用词，成文于 background-starts 页；"Activity Sandwich"、`android:allowCrossUidActivitySwitchFromBelow`、targetSdk 37 显式 opt-in 均为该页原文 |
| 16 | "BAL 被拦截时发起方会拿到异常，可在代码里判定" | 原文："The sender app does not receive a direct return value or exception… Instead the Android System internally logs a \"Background activity launch blocked!\" message to Logcat…" |
| 17 | "CISA KEV 未收录即代表没有真实利用" | 未收录 ≠ 无真实利用（§7 推论边界） |
| 18 | "App Links 域验证一次生效，改 `assetlinks.json` 会自动跟进" | 仅 Android 15+ 做周期性重验证且 "can take up to seven days to propagate…"；Android 14 及以下不做周期性重验证；Android 11 及以下要求 manifest 中所有 host 都匹配 |

---

## 9. 可观测面与自查

### 9.1 `dumpsys package` 的观测面与字段边界

`dumpsys package` 是查看已安装应用组件解析与权限状态的官方调试入口。收敛顺序：定位目标包段落中 Activity / Service / Provider / Receiver 的解析列表，确认哪些组件因带 intent filter 而对系统解析器可见、其导出状态与所需权限为何；再看权限授予段的实际授予结果；最后与 §9.2 交叉验证"编译期声明"与"设备实际状态"是否一致。**本手册不给字段级样例**：现行 `PackageManagerShellCommand.java`（原 gitiles 路径已迁移或改名）与 `PrintUtils.java`（NOT_FOUND）未取回，`cmd package query-activities` 等子命令字符串亦未取证，故只述观测思路、不虚构字段名（§11）。已取证的等价观测命令为官方 App Links 文档给出的 `adb shell pm get-app-links PACKAGE_NAME`（输出含成文状态词 `verified`、`legacy_failure`，原文样例 "Domain verification state: example.com: verified … sub.example.com: legacy_failure …"）、`adb shell pm verify-app-links --re-verify PACKAGE_NAME`、`adb shell pm set-app-links --package PACKAGE_NAME 0 all`（"This command puts the device in the same state that it's in before the user chooses default apps for any domains."），并需 "Wait at least 20 seconds for the asynchronous verification process to complete."

### 9.2 Manifest 层自查项

| 自查项 | 属性 / 元素实名 | 判定依据 |
| --- | --- | --- |
| 导出声明完整性 | 带 `<intent-filter>` 组件的 `android:exported`；含 LAUNCHER 才置 `true` | Android 12+ 硬性安装要求（§1.1） |
| 接收器导出最小化 | `android:exported`、`android:permission`（限定发送方）、`receiverPermission` 是否为 `null` | "other apps could send unprotected broadcasts to your app" |
| 隐式广播面 | 自定义 action 是否落在 **640** 条 `<protected-broadcast>` 内 | 不在清单即按外部可发送对待（§2.4） |
| Provider 权限矩阵 | `android:exported`、`android:permission`、`android:readPermission`、`android:writePermission` | `<provider>` 四属性语义（§4.1） |
| URI 临时授权范围 | `android:grantUriPermissions`（默认 `false`）与 `<grant-uri-permission>` 子集 | "temporarily overcoming the restriction imposed by the readPermission, writePermission, permission, and exported attributes" |
| FileProvider 宽度 | `<paths>` 是否含 `<root-path>`，或使用 `.`、`/` 等过宽范围 | Mitigations："Do not use the <root-path> path element…"、"Share narrow path ranges…" |
| 链接解析授权 | `android:autoVerify="true"` 的 host 清单与站点 `/.well-known/assetlinks.json` 是否一一对应；是否含 `BROWSABLE` + `https` | §5.1、§5.2 |
| PendingIntent 与 Task 声明 | 创建处是否显式 `FLAG_IMMUTABLE`、发送处 BAL 模式取值；`launchMode`/`allowTaskReparenting`/`FLAG_ACTIVITY_NEW_TASK` 组合、`android:allowCrossUidActivitySwitchFromBelow` 是否评估 | §3.1、§3.4、§6.2、§6.3 |

### 9.3 日志、命令与默认应用观测点

- **BAL 拦截断言**：唯一官方成文途径是 Logcat 的 `"Background activity launch blocked!"`；因发起方无返回值/异常，回归测试应以日志断言而非异常捕获发现可疑后台启动尝试。
- **notification trampoline**：Android 12 targeting 页给出排查命令前缀 "you can identify which service or broadcast receiver acted as the notification trampoline… look at output of the following terminal command: adb shell dumpsys activity service …"（子命令 `dumpsys activity service` 已核实，后续参数以官方页面为准），并建议 "avoid this UX pattern and use a PendingIntent instead"。
- **默认应用与角色**：以 `RoleManager.isRoleHeld(String)` / `isRoleAvailable(String)` 核对 `ROLE_BROWSER`（`"android.app.role.BROWSER"`）、`ROLE_DIALER`、`ROLE_SMS`、`ROLE_ASSISTANT`、`ROLE_HOME` 等角色归属；系统设置入口常量 `Settings.ACTION_APP_OPEN_BY_DEFAULT_SETTINGS`、`Settings.ACTION_MANAGE_DEFAULT_APPS_SETTINGS`；再与 `pm get-app-links` 的域验证结果合并判断 https 链接落点。

### 9.4 开发者加固动作（9 条）

1. 所有带 intent filter 的组件显式声明 `android:exported`，默认 `false`，仅含 LAUNCHER category 时置 `true`；把 lint/构建报错当发布阻断项处理。
2. 动态注册接收器按 targetSdk 34 显式选 `RECEIVER_EXPORTED` / `RECEIVER_NOT_EXPORTED`；只收自家或系统广播优先 `RECEIVER_NOT_EXPORTED`，并清理已废弃别名常量。
3. 发送广播/绑定服务时指定 sender 权限，避免 `receiverPermission` 为 `null` 的默认开放姿态；自定义 action 不假设"没人会发"。
4. 创建 `PendingIntent` 一律显式 `FLAG_IMMUTABLE`；仅在确需填充时用 `FLAG_MUTABLE` 并记录评审理由。
5. 涉及 `PendingIntent#send()` / `IntentSender` 的后台启动按 Android 14 规则用 `ActivityOptions.setPendingIntentBackgroundActivityStartMode(int)` 显式授权，优先 `DENIED` 或更细的 `ALLOW_IF_VISIBLE`，不依赖已 `@Deprecated` 的 `…ALLOWED` 语义。
6. 处理外部传入 Intent 时检查并清除四个 `FLAG_GRANT_*_URI_PERMISSION`，用官方推荐的 `IntentSanitizer` 收敛重定向目标；需启动自家非导出组件时由应用自身构造显式 Intent，不转发外部数据。
7. 收紧 `ContentProvider`：优先 `android:exported="false"`；需共享时以 `readPermission`/`writePermission` 分轨而非笼统 `permission`，`grantUriPermissions` 保持默认 `false` 且只列必要 `<grant-uri-permission>` 子树。
8. `FileProvider` 禁用 `<root-path>`、只声明窄路径；分享用 `getUriForFile` 产出的 `content://` URI，接收方经 `ContentResolver.openFileDescriptor(Uri, String)` 读取，不暴露原始文件路径。
9. App Links 与 Task：确保每个 `autoVerify` host 都有对应 `assetlinks.json` 且 filter 含 `BROWSABLE` + `https`，发布前用 `pm verify-app-links --re-verify` / `pm get-app-links` 回归，并把 Android 15 的七天级重验证时延纳入变更流程；评估 targetSdk 37 的 `android:allowCrossUidActivitySwitchFromBelow` 对自家 assistant/系统服务链路的影响后再 opt-in。

---

## 10. 与其他手册的分工

- 与《APK 签名与安装校验》：签名、证书链与安装期校验归该手册；本手册只谈组件装配完成后的运行时 IPC 与导出语义。
- 与《多用户与工作资料隔离》：跨用户与 Profiles 的可见性、沙箱边界归该手册；本手册中 provider 的"同 UID 可访问"仅指 §4.1 的成文机制，不延伸到工作资料策略。
- 与《通用漏洞类型手册》：跨组件的通用缺陷分类（路径遍历、越权读写等）以该手册为主干；本手册只承担 IPC/导出这一子类的属性语义、常量实名与公告案例。
- 与《TEE 与密钥认证》《基带与通话链路安全》《主线模块与 APEX》：本手册全部条目位于 framework/java 层，不涉及可信执行、基带与模块分发。
- 已发布文档：公告体系与评级口径见 [Android CVE 与安全公告](/android-cve-and-security-bulletins/)、[漏洞严重性评估方法](/android-vulnerability-severity-assessment/)；提权与代码执行链路见 [RCE 与权限提升](/android-rce-and-privilege-escalation/)；底层组件缺陷见 [底层漏洞类型图谱](/android-low-level-vulnerability-types/)、[Android 通用系统漏洞](/android-common-system-vulnerabilities/)；调试与数据传输面见 [ADB、备份与数据传输](/android-adb-backup-transfer/)；总索引见 [README](/)。

---

## 11. 未核实与来源失效

1. **Google 关于 intent redirection 的赏金计划 / 安全博客专门说明**：两轮检索（`privacy-and-security/risks` 目录页 + risks-index 导航枚举）未找到赏金公告本体；可核实的官方成文位置只有风险页《Intent redirection》。赏金条目：**未核实**，本手册不引用任何具体 blog URL。
2. **以 "Broadcasts Overview" 为标题、集中罗列 protected broadcast 的文档专页**：预期 URL `https://developer.android.com/develop/background-work/system-broadcasts/broadcasts-implementers` 与 `…/system-broadcasts/overview` 均返回 404（响应体为 404 模板页）；现行可用入口是 `https://developer.android.com/guide/components/broadcasts` → 301 至 `/develop/background-work/background-tasks/broadcasts`。实名以 `core/res/AndroidManifest.xml` 的 640 条为准。
3. **`dumpsys package` 组件导出的字段名 / `cmd package query-*` 子命令字符串**：`services/package/java/com/android/server/pm/PackageManagerShellCommand.java` 路径已迁移或改名（返回非 base64 文本）、`services/core/java/com/android/server/print/PrintUtils.java` 返回 NOT_FOUND，故 §9.1 不给字段级样例。
4. **`RoleManager.java` 的 AOSP 源文件**：`…/main/core/java/android/app/role/RoleManager.java` 返回 NOT_FOUND（模块归属疑已变化），角色常量改以官方 reference 页取证。
5. **AOSP 公告中的 autoVerify / 域验证劫持类条目**：NVD 以 `autoVerify`、"app links android"、"link handler android default" 两轮检索均无 Google/AOSP 条目；该方向仅有应用侧案例（CVE-2021-33699、CVE-2024-34405）。
6. **公告 URL 形态（勘误）**：带年份目录层（`/bulletin/2024/2024-01-01`）与旧 `-05` 日期形式（`2022-08-05`、`2023-06-05`、`2023-10-05`、`2024-09-05`、`2025-03-05`、`2025-06-05` 实测）当前均 404；有效形式为 `…/bulletin/YYYY-MM-01` 与 Pixel 专页 `…/bulletin/pixel/YYYY-MM-01`（两者都取到了本手册案例的真实公告行）。
7. **`adb shell dumpsys activity service …` 的完整参数串**：官方页面仅给出命令前缀，本次抓取未完整保留后续参数，使用时以该页为准。

---

## 12. 参考

- **AOSP 源码**（android.googlesource.com，`main` / `androidx-main`）：`frameworks/base` 的 `core/java/android/content/Context.java`、`Intent.java`、`ContentProvider.java`、`core/java/android/app/PendingIntent.java`、`ActivityOptions.java`、`ActivityManager.java`（仅用于证伪）、`core/java/android/provider/Settings.java`、`core/res/AndroidManifest.xml`（https://android.googlesource.com/platform/frameworks/base/+/refs/heads/main/core/res/AndroidManifest.xml ）；`frameworks/support` 的 `core/core/src/main/java/androidx/core/content/FileProvider.java`
- **developer.android.com**：https://developer.android.com/about/versions/12/behavior-changes-12 、/about/versions/12/behavior-changes-all 、/about/versions/14/behavior-changes-14 ；https://developer.android.com/guide/components/activities/background-starts 、/guide/components/activities/tasks-and-back-stack 、/guide/components/intents-filters 、/guide/components/broadcasts 、/guide/topics/manifest/provider-element ；https://developer.android.com/training/app-links/verify-android-applinks ；https://developer.android.com/reference/android/app/role/RoleManager ；https://developer.android.com/privacy-and-security/risks/intent-redirection 、/risks/pending-intent 、/risks/sender-of-pending-intents 、/risks/implicit-intent-hijacking 、https://developer.android.com/privacy-and-security/risks/file-providers
- **source.android.com 安全公告**：`https://source.android.com/docs/security/bulletin/2020-06-01`、`2020-08-01`、`2022-05-01`、`2022-12-01`、`2023-04-01`、`2023-10-01`；Pixel 专页 `…/bulletin/pixel/2020-06-01`、`…/bulletin/pixel/2021-12-01`
- **NVD**：`https://services.nvd.nist.gov/rest/json/cves/2.0?cveId=<CVE>`（13 条逐一取回）；https://nvd.nist.gov/vuln/detail/CVE-2014-8610 、/CVE-2021-33699 、/CVE-2024-34405
- **CISA KEV**：https://www.cisa.gov/sites/default/files/feeds/known_exploited_vulnerabilities.json （2026-10-27 快照）

---

## 13. 联系方式

咨询 Android 系统详情请联系 https://t.me/one00190

*本手册仅供安全学习与防御研究使用；所有组件事实、接口名称与案例归属均取自公开一手来源并附链接，未能核实的表述已单列于 §11 而非写入正文。*
