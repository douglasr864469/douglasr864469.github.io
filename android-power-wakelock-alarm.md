---
layout: page
title: "Android 电源管理、唤醒锁与闹钟调度攻击面手册"
description: "把「熄屏后谁还能跑」拆成四套状态机：`PowerManagerService`（system_server）结算唤醒锁、`AlarmManagerService` 与 `DeviceIdleController` 位于 jobscheduler APEX、`AppStandbyController` 负..."
date: 2026-11-04
permalink: /android-power-wakelock-alarm/
---

> 手机熄屏之后并没有「停下来」：`PowerManagerService` 仍在结算唤醒锁，`AlarmManagerService` 仍在按批次投递闹钟，`DeviceIdleController` 正在判断是否该进入 Doze，`AppStandbyController` 正在给每个应用分桶。这四套机制共同决定「谁能醒、什么时候醒、醒多久」，也共同构成了一个常被低估的攻击面——一个持有唤醒锁不放的应用可以耗尽电池，一个伪造的精确闹钟可以成为侧信道，一个算错的空闲判定可以让整台设备反复重启。这份手册沿「电源状态机 → 唤醒锁 → 空闲与分桶 → 闹钟与 Job」这条链，说明各层的真实边界、公开缺陷模式与防御方的观测手段。只讨论公开且已修复的问题，不含可利用代码。

## 0. 使用方式与边界

这份手册面向三类读者：

- **应用开发者**：需要判断自己的后台任务在 Doze、App Standby 与精确闹钟权限收紧之后还能不能按时执行，以及为什么「申请了唤醒锁」并不等于「一定能跑」。
- **系统与平台工程师**：需要理解 `PowerManagerService`、`AlarmManagerService`、`DeviceIdleController` 与 `AppStandbyController` 的职责切分，以及它们分别位于 system_server 还是 jobscheduler APEX。
- **安全评估者**：需要在一条「电源相关」公告出现时，判断问题落在电源服务、闹钟服务、空闲控制、分桶逻辑还是 JNI 边界，而不是把所有带 `power` 字样的 CVE 混为一类。

本文的三条自我约束：

1. 只引用公开且已修复的问题；CVE 编号、落点与描述以 NVD 条目为准，公告归类与严重级别以 Android Security Bulletin 为准，两者不一致处单独标注。
2. 不写利用步骤、不写 PoC、不写可复现的攻击参数；成因只描述到「哪一类校验、状态或边界缺失」。
3. 查不到一手来源的说法一律进「未核实与失效来源」，不写进正文。

与已发布手册的分工：

- 《Android 通知、状态栏与通知监听攻击面手册》讲通知的呈现与监听；本文只讲通知背后的唤醒与调度成本，不重复通知渠道与锁屏可见性。
- 《Android 多用户、工作资料与沙箱隔离手册》讲用户边界；本文提到的「按用户」只作为电源策略的作用域，不展开多用户机制。
- 《Android 媒体解码与容器解析攻击面手册》讲解码器与容器；本文只把前台服务类型限制作为调度背景引用。

## 1. 电源状态机：一次「醒来」的完整旅程

Android 的电源管理不是一个开关，而是一组互相制衡的状态机。把「设备为什么醒着」拆开，至少经过六个判断：

| 阶段 | 主要位置 | 做的事 | 安全/可靠性判断点 |
|------|----------|--------|------------------|
| 1. 请求 | 应用进程 | `newWakeLock()` 取锁、`acquire()` 申请 | 锁类型、标签、超时；是否真的需要 |
| 2. 结算 | `PowerManagerService`（system_server） | 汇总所有唤醒锁，决定「最小睡眠状态」 | 调用者 uid、锁是否被正确释放 |
| 3. 空闲判定 | `DeviceIdleController`（jobscheduler APEX） | 依据充电、屏幕、运动状态进入/退出 Doze | 白名单、强制空闲命令、状态迁移 |
| 4. 分桶 | `AppStandbyController`（同 APEX 的 usage 子系统） | 按使用频率把应用分入五个桶 | 桶位计算、限制生效范围 |
| 5. 调度 | `AlarmManagerService`（jobscheduler APEX） | 按窗口、优先级、Doze 约束投递闹钟 | 精确闹钟权限、批次窗口、待处理数量 |
| 6. 执行 | 应用 / JobScheduler / 前台服务 | 真正运行任务 | 超时、前台服务类型、电量与温度约束 |

关键点是：**「我申请了唤醒锁」只是第 1 步**。第 2 步之后，系统仍可能因为 Doze、分桶、低电或热限制把执行推后；反过来，一个没有唤醒锁的应用也可能因为精确闹钟或高优先级前台服务被唤醒。把「持有 `PARTIAL_WAKE_LOCK`」当成保活手段，是对这套状态机最常见的误解。

第二个关键点是：**四套机制的归属不同**。`PowerManagerService` 在 `frameworks/base/services/core/java/com/android/server/power/`，是 system_server 里的 `SystemService`；而 `AlarmManagerService`、`DeviceIdleController` 与分桶逻辑位于 `apex/jobscheduler/service/java/com/android/server/` 之下，随 jobscheduler APEX 更新。这意味着同一条「电源行为变更」，可能通过框架 OTA 到达，也可能通过 Mainline 模块到达——排查版本差异时必须先确认模块版本。

## 2. 对象模型与进程边界

### 2.1 关键对象

| 对象 | 所在层 | 作用 |
|------|--------|------|
| `PowerManager` | 公开 SDK | 取唤醒锁、查询交互状态、申请忽略电池优化 |
| `PowerManager.WakeLock` | 公开 SDK | 单个唤醒锁句柄，`acquire()` / `release()` |
| `PowerManagerService` | AOSP 内部（system_server） | 唤醒锁结算、屏幕状态、低电模式 |
| `AlarmManager` | 公开 SDK | 设置一次性/重复/精确/闹钟时钟闹钟 |
| `AlarmManagerService` | AOSP 内部（jobscheduler APEX） | 闹钟排队、批次、Doze 约束、投递 |
| `DeviceIdleController` | AOSP 内部（jobscheduler APEX） | Doze 状态机与白名单 |
| `AppStandbyController` | AOSP 内部（usage 子系统） | 应用分桶与限制 |
| `JobScheduler` / `JobInfo` | 公开 SDK | 约束式后台任务，受 Doze 与分桶共同影响 |
| `BatteryManager` | 公开 SDK | 电量、充电状态、热状态查询 |

### 2.2 权限与调用边界

| 能力 | 需要的权限/条件 | 说明 |
|------|----------------|------|
| 取普通唤醒锁 | `android.permission.WAKE_LOCK`（`normal` / `instant`） | 安装即授予；不等于保活成功 |
| 申请忽略电池优化 | 用户确认（`ACTION_REQUEST_IGNORE_BATTERY_OPTIMIZATIONS`） | 应用不能自批；只能查询或请求 |
| 精确闹钟（用户授权） | `SCHEDULE_EXACT_ALARM`（API 31 特殊访问） | 用户可撤销；target 33+ 新装不预授 |
| 精确闹钟（自动授予） | `USE_EXACT_ALARM`（API 33） | 自动授予、不可撤销，受 Play 政策限制 |
| 通过 `OnAlarmListener` 设定 | 无精确闹钟权限亦可 | 只在应用进程存活期间有效 |
| 强制空闲（测试） | shell / adb | `dumpsys deviceidle force-idle` 等 |

这里要特别区分「权限」和「效果」：`WAKE_LOCK` 是普通权限，几乎任何应用都能拿；但拿到之后能不能真正让 CPU 不睡，取决于 `PowerManagerService` 的结算结果与 Doze 状态。把权限声明当成能力证明，是评估电源类问题时第一个要纠正的直觉。

## 3. 唤醒锁：语义、失配与代价

### 3.1 现行锁类型

`PowerManager` 公开的唤醒锁类型经历过一轮大清理：

| 常量 | 状态 | 语义 |
|------|------|------|
| `PARTIAL_WAKE_LOCK` | 现行 | 保持 CPU 运行，屏幕与键盘背光可关 |
| `PROXIMITY_SCREEN_OFF_WAKE_LOCK` | 现行 | 贴近传感器时熄屏，用于通话 |
| `FULL_WAKE_LOCK` | API 17 起废弃 | 旧式「屏幕+键盘全亮」 |
| `SCREEN_DIM_WAKE_LOCK` | API 17 起废弃 | 旧式「屏幕变暗」 |
| `SCREEN_BRIGHT_WAKE_LOCK` | API 15 起废弃 | 旧式「屏幕全亮」 |
| `ACQUIRE_CAUSES_WAKEUP` | API 33 起废弃 | 取锁瞬间点亮屏幕 |
| `ON_AFTER_RELEASE` | 未废弃 | 释放后短暂保持亮屏 |
| `DOZE_WAKE_LOCK` / `DRAW_WAKE_LOCK` | 不在公开 SDK | 系统内部使用，第三方不可依赖 |

这份表本身就是一条防御线索：**大量老教程仍在教 `FULL_WAKE_LOCK` 或 `SCREEN_DIM_WAKE_LOCK`**，而它们在现行平台上已被废弃。评估时看到这类代码，应先判断它是不是照抄了十年前的示例，而不是假定它真的能点亮屏幕。

### 3.2 引用计数与失配

`WakeLock` 是引用计数语义：同一标签多次 `acquire()` 需要同样次数的 `release()`。常见的工程缺陷有三类：

1. **异常路径漏释放**：在 `try` 中取锁，异常从 `finally` 之外抛出，锁永久悬挂。
2. **重复取锁未配对**：每次回调都 `acquire()`，但只在正常结束时 `release()` 一次。
3. **超时缺失**：`acquire()` 不带超时，进程被系统回收前锁一直有效。

从安全角度看，悬挂的唤醒锁首先是一个**可用性问题**（耗电、发热、触发热降频），其次才可能被用于「低成本拖住设备」的骚扰场景。它通常不构成提权，但在共享设备、Kiosk 与车载场景里会直接影响业务可用性。

### 3.3 唤醒锁与 Doze 的关系

Doze 的核心语义之一就是**忽略普通唤醒锁**：设备进入空闲后，应用持有的 `PARTIAL_WAKE_LOCK` 不再阻止系统睡眠。这一点在官方 Doze 文档里有明确表述，也是「唤醒锁保活」这一误传的直接反证。

因此，判断一个后台任务能否执行，正确顺序是：

1. 设备是否处于 Doze（充电、屏幕、运动状态）；
2. 应用是否在分桶的低桶位；
3. 任务用的是闹钟、Job 还是前台服务；
4. 是否持有对应的精确闹钟权限或前台服务类型；
5. 最后才看唤醒锁。

把顺序倒过来，就会得出「我明明持有唤醒锁，为什么任务没跑」的错误结论。

## 4. Doze、App Standby 与豁免

### 4.1 Doze 的触发与效果

Doze 在设备**拔电、静止、熄屏**一段时间后进入。进入后系统会：

- 断开网络访问（维护窗口除外）；
- **忽略普通唤醒锁**；
- 推迟普通闹钟、JobScheduler 任务与账户同步；
- 允许 `setExactAndAllowWhileIdle` 与 `setAlarmClock` 等少数高优先级路径穿透。

维护窗口的间隔会随空闲时长拉长。具体阈值属于实现细节，本文不给出精确分钟数（见「未核实与失效来源」）。

### 4.2 App Standby 与五个桶

Android 9（API 28）起，App Standby 使用五个桶：active、working set、frequent、rare、restricted。桶位由使用频率、通知、前台进程等因素共同决定；桶位越低，Job 与闹钟的配额越少。Android 12（API 31）引入 restricted 桶，作为比 rare 更严格的档位；Android 13（API 33）把应用进入 restricted 桶的判定窗口从 45 天缩短到 8 天，并让这类应用收不到 `BOOT_COMPLETED`。

从攻击面角度看，分桶是一个**降权机制**：它限制的是「后台执行机会」，不是数据访问权限。评估时不要把「被分到 restricted 桶」误读成「被沙箱隔离」——应用仍然可以启动前台界面，只是后台调度被大幅收紧。

### 4.3 豁免只能「查询」和「请求」

电池优化豁免是用户级决定：

- 查询：`PowerManager.isIgnoringBatteryOptimizations()`；
- 请求：`ACTION_REQUEST_IGNORE_BATTERY_OPTIMIZATIONS`，由用户在系统界面上确认。

应用**不能自行把自身加入白名单**，也没有可静默调用的公开接口。这条边界是 Doze 设计的基础；任何声称「一行代码加入白名单」的方案，要么依赖系统签名/特权，要么是在误导。

### 4.4 测试命令

平台侧提供了可复现的空闲测试入口：

- `dumpsys deviceidle force-idle` / `unforce`：强制进入/退出空闲；
- `dumpsys deviceidle step`：单步推进状态机；
- `dumpsys deviceidle whitelist` / `tempwhitelist`：查看与临时调整白名单；
- `am set-inactive`：设置应用的闲置状态，用于验证分桶行为。

这些命令需要 shell 权限，普通应用无法调用；它们属于评估者的观测工具，不是应用可利用的接口。

## 5. 精确闹钟：权限收紧与副作用

### 5.1 两套权限

| 权限 | 引入 | 授予方式 | 特点 |
|------|------|----------|------|
| `SCHEDULE_EXACT_ALARM` | API 31 | 特殊访问，用户在设置中授予 | 可撤销；target 33+ 新装默认不预授 |
| `USE_EXACT_ALARM` | API 33 | 自动授予 | 不可撤销；受 Play 政策与用例限制 |

应用在使用前必须查询 `AlarmManager.canScheduleExactAlarms()`；未获授权时调用精确闹钟接口会抛异常。一个容易忽略的例外是 `OnAlarmListener`：通过监听器设定的闹钟不受这条权限约束，但它只在应用进程存活时有效，进程被杀后就失效。

这意味着「精确闹钟权限」并不是一个可以长期依赖的保活手段：用户随时可以在设置里收回，系统升级也可能改变默认授予策略。设计后台任务时应准备降级路径（`setAndAllowWhileIdle`、JobScheduler 或前台服务），而不是把精确闹钟当成唯一调度器。

### 5.2 三类闹钟的实际行为

| 接口 | 精确性 | Doze 中能否触发 | 备注 |
|------|--------|----------------|------|
| `setExactAndAllowWhileIdle` | 精确 | 能，但约每 9 分钟一次 | 适合必须准点的低频任务 |
| `setAndAllowWhileIdle` | 非精确 | 能 | 系统可批处理，功耗更友好 |
| `setAlarmClock` | 精确 | 能，必要时退出低电 | 面向用户可见的闹钟，系统会显示图标 |

「约 9 分钟一次」是官方功耗文档给出的量级，不是可以随意提高的频率上限。把 `setExactAndAllowWhileIdle` 用在秒级轮询上，既违反平台设计，也会被系统限流。

### 5.3 闹钟与安全

闹钟本身不处理敏感数据，但它是一个**时间侧信道**和**唤醒放大器**：

- 时间侧信道：应用可以通过闹钟的实际触发时间推断设备是否处于 Doze、是否被其他高优先级闹钟唤醒；
- 唤醒放大器：大量高频闹钟会持续把设备拉出低功耗状态，形成拒绝休眠（battery DoS）的效果；
- 权限绕过：闹钟携带的 `PendingIntent` 若指向未导出组件或可变意图，可能成为跨应用调用的跳板——这类问题属于 `PendingIntent` 语义，本文只在公告台账中标注，不展开。

## 6. 公开公告台账

下表只收录与本主题直接相关的公开条目。分类与级别照抄公告；NVD 描述用于确认落点。

| CVE | NVD 落点 / 类型 | 级别 | 分类 / 月份 |
|-----|----------------|------|-------------|
| [CVE-2021-0439](https://nvd.nist.gov/vuln/detail/CVE-2021-0439) | `setPowerModeWithHandle`，`com_android_server_power_PowerManagerService.cpp`，越界写（EoP / CWE-787） | High | Framework / 2021-04 |
| [CVE-2020-0312](https://nvd.nist.gov/vuln/detail/CVE-2020-0312) | Battery Saver 的 `PendingIntent` 权限绕过（ID） | Moderate | Framework / Android 11 首发 |
| [CVE-2022-20307](https://nvd.nist.gov/vuln/detail/CVE-2022-20307) | `AlarmManagerService` 侧信道探测已安装应用（ID） | Moderate | Framework / Android 13 |
| [CVE-2022-20414](https://nvd.nist.gov/vuln/detail/CVE-2022-20414) | `AlarmManagerService.java` 的 `setImpl`，异常导致启动循环（DoS） | High | System / 2022-11 |
| [CVE-2023-20950](https://nvd.nist.gov/vuln/detail/CVE-2023-20950) | `AlarmManagerActivity.java` 借 `PendingIntent` 绕过后台启动限制（EoP） | Moderate | Framework / 2023-04 |
| [CVE-2023-21128](https://nvd.nist.gov/vuln/detail/CVE-2023-21128) | `AppStandbyController.java` 逻辑错误导致提权（EoP） | High | Framework / 2023-06 |

公告链接：

- 2021-04：https://source.android.com/security/bulletin/2021-04-01
- Android 11 首发公告：https://source.android.com/security/bulletin/android-11
- Android 13 首发公告：https://source.android.com/security/bulletin/android-13
- 2022-11：https://source.android.com/security/bulletin/2022-11-01
- 2023-04：https://source.android.com/security/bulletin/2023-04-01
- 2023-06：https://source.android.com/security/bulletin/2023-06-01

读这张表要注意三点：

1. **落点分布很散**：一条在 JNI（`com_android_server_power_PowerManagerService.cpp`），三条在 Java 服务/活动，一条在分桶控制器。把它们统称为「电源漏洞」会掩盖真实边界。
2. **级别与类型不总是一致**：NVD 与公告对同一条的评级可能不同，表中以公告为准并在需要时注明 NVD。
3. **不是所有条目都关于「省电」**：CVE-2022-20307 是信息泄露（侧信道），CVE-2022-20414 是可用性（启动循环），CVE-2023-20950 是后台启动绕过。判断影响时必须逐条读描述。

## 7. 成因模式

把上述条目抽象，可以看到六种反复出现的成因：

### 7.1 边界值未校验

JNI 层把 Java 传入的模式值直接用于数组索引或位运算，缺少范围检查。CVE-2021-0439 的越界写属于这一类。防御上，JNI 入口必须把「Java 侧枚举」当成不可信输入重新校验，而不是假定上层已经检查过。

### 7.2 `PendingIntent` 身份混淆

电源/省电相关界面通过 `PendingIntent` 触发特权操作，但创建时未限定包名或未使用可变性正确的标志，导致其他应用可以借用创建者的身份。CVE-2020-0312 与 CVE-2023-20950 都落在这个模式上。核心修法是：显式指定目标组件、`FLAG_IMMUTABLE` 优先、不把敏感动作放进可变意图。

### 7.3 状态迁移缺锁

闹钟与空闲状态机是多线程共享的：Binder 线程、Handler 线程、定时器线程都可能修改同一份状态。缺少同步或错误加锁会导致竞态，轻则状态错乱，重则异常把系统服务打挂（CVE-2022-20414 的启动循环）。这类问题的防御不是「多加一把锁」，而是明确状态所有权与迁移合法性。

### 7.4 可观测差异未消除

系统在响应「查询某应用是否存在」「某闹钟是否已设定」时返回的耗时或错误码不同，形成侧信道（CVE-2022-20307）。修法通常是统一错误路径与响应时间，而不是简单地加权限——因为调用方往往本来就持有查询权限。

### 7.5 策略判断顺序错误

分桶、白名单、前台状态之间是有优先级的。判断顺序写错会让本应受限的应用获得额外配额，或让本应豁免的应用被误伤（CVE-2023-21128）。防御要点是把策略写成显式的决策表，并为每个分支写测试。

### 7.6 默认值随版本漂移

精确闹钟权限的默认授予、restricted 桶的判定窗口、前台服务类型的时限，都在版本间变化过。把某一版的默认值写死进业务逻辑，升级后就会出现「昨天能跑、今天不跑」的故障。正确做法是运行时查询能力，而不是编译期假定。

## 8. 版本加固时间线

| 版本 / API | 变化 |
|------------|------|
| 6.0 / 23 | Doze 与 App Standby 引入 |
| 9 / 28 | App Standby 五桶模型 |
| 10 / 29、11 / 30 | 行为变更页无电源/闹钟专项条目 |
| 12 / 31 | `SCHEDULE_EXACT_ALARM` 特殊访问；restricted 桶 |
| 13 / 33 | `USE_EXACT_ALARM`；restricted 判定 45 天 → 8 天；受限应用收不到 `BOOT_COMPLETED` |
| 14 / 34 | 恢复到 14 时拒绝精确闹钟权限；JobScheduler 超时触发 ANR |
| 15 / 35 | `dataSync` / `mediaProcessing` 前台服务 24 小时内合计 6 小时限额 |

这张表说明一个趋势：**平台在持续把「后台自由执行」收窄为「声明用途 + 用户授权 + 配额」**。对开发者而言，越晚适配成本越高；对评估者而言，看到「后台不执行」的报障，首先要问的是目标版本与 targetSdk，而不是直接怀疑厂商定制。

## 9. 观测点与自查

### 9.1 应用侧

- 是否在 `finally` 中释放唤醒锁？是否给 `acquire()` 设置了超时？
- 是否在运行时查询 `isIgnoringBatteryOptimizations()` 与 `canScheduleExactAlarms()`，而不是假定结果？
- 精确闹钟失败时是否有降级路径？
- 前台服务是否声明了正确的类型，并遵守 24 小时配额？
- Job 是否设置了合理的约束与退避，避免在低桶位被长期推迟后集中爆发？

### 9.2 平台侧

- `dumpsys power` 查看唤醒锁列表与持有时长，重点找长期未释放的条目；
- `dumpsys alarm` 查看闹钟队列、批次窗口与偏差；
- `dumpsys deviceidle` 查看当前空闲状态、白名单与最近迁移；
- `dumpsys usagestats` / `am get-inactive` 查看分桶结果；
- 关注 jobscheduler APEX 版本，因为闹钟与空闲逻辑可能随模块更新而变化。

### 9.3 评估侧

- 看到「电源」公告，先定位落点：JNI、Java 服务、`PendingIntent`、状态机还是分桶；
- 区分信息泄露、提权、拒绝服务与策略绕过四类影响；
- 不要把「耗电」直接等同于「安全漏洞」——只有当它能被用于稳定的拒绝服务或侧信道时，才进入安全评级。

## 10. 防御要点

1. **把唤醒锁当成短事务**：取锁即用、用完即放，设置超时；不要用唤醒锁表达「我要长期运行」。
2. **用能力查询代替版本判断**：`canScheduleExactAlarms()`、`isIgnoringBatteryOptimizations()`、`getSupportedForegroundServiceTypes()` 之类的运行时接口优先。
3. **精确闹钟只用于用户可感知的准点任务**：周期同步、日志上报、埋点应使用 JobScheduler 或 `setAndAllowWhileIdle`。
4. **`PendingIntent` 默认不可变**：必须可变时，限定包名与组件，避免把特权动作暴露给任意调用者。
5. **前台服务声明真实类型**：类型不是标签，而是配额与可见性的依据；错报类型会同时触发政策与平台限制。
6. **为低电量与 Doze 设计降级**：任务失败要能重试，重试要带退避，不要把失败当成异常。
7. **监控异常耗电**：把「单位时间唤醒次数」「持锁总时长」纳入线上指标，异常升高往往是代码缺陷的第一信号。

## 11. 常见误传纠正

### 误传一：加入 Doze 白名单可以自己完成

错。应用只能查询 `isIgnoringBatteryOptimizations()`，或通过 `ACTION_REQUEST_IGNORE_BATTERY_OPTIMIZATIONS` 请用户确认；没有公开接口能静默把自身加入白名单。所谓「一行代码白名单」要么依赖系统签名，要么是过时/虚假信息。

### 误传二：持有 `PARTIAL_WAKE_LOCK` 就能保活

错。Doze 明确忽略普通唤醒锁；分桶与前台服务策略也会限制执行。唤醒锁只影响「CPU 是否可睡」，不影响「系统是否愿意让你跑」。

### 误传三：`SCHEDULE_EXACT_ALARM` 安装即授予，`USE_EXACT_ALARM` 人人可用

错。前者是 API 31 的特殊访问，用户可授予也可撤销，target 33+ 的新装默认不预授；后者虽然自动授予且不可撤销，但受 Play 政策与有限用例约束，不是通用替代品。

### 误传四：`setExactAndAllowWhileIdle` 可以高频调用

错。Doze 中对这类闹钟有频率约束，官方给出的量级约为每 9 分钟一次。把它用于秒级轮询既无效也不合规。

### 误传五：CVE-2021-0439 是 Java 服务漏洞

不准确。它的落点是 JNI 文件 `com_android_server_power_PowerManagerService.cpp` 中的 `setPowerModeWithHandle`，属于 native 边界的越界写。把它归为「Java 层权限问题」会误导修复方向。

## 12. 未核实与失效来源

- `source.android.com/docs/core/power/device-idle` 已 404，Doze 的 light/deep 具体阈值未能从一手来源核实，文中只用「一段时间」描述。
- Play 关于 `USE_EXACT_ALARM` 的政策页在核验时内容截断，具体允许用例清单未完整核实。
- `DOZE_WAKE_LOCK` / `DRAW_WAKE_LOCK` 的内部语义未在公开 SDK 文档中给出，本文只写「不在公开 SDK」。
- 各厂商对分桶与自启动的额外限制不在 AOSP 范围内，本文未逐一核实。
- 本文的版本时间线以官方行为变更页为准；个别版本（10、11）没有电源/闹钟专项条目，不代表没有实现层调整。

## 13. 参考资料

- PowerManager（官方参考）：
  https://developer.android.com/reference/android/os/PowerManager
- PowerManager.WakeLock（官方参考）：
  https://developer.android.com/reference/android/os/PowerManager.WakeLock
- Doze 与 App Standby（官方指南）：
  https://developer.android.com/training/monitoring-device-state/doze-standby
- App Standby 分桶（官方指南）：
  https://developer.android.com/topic/performance/appstandby
- 精确闹钟权限（官方指南）：
  https://developer.android.com/training/scheduling/alarms#exact-permission-declare
- 功耗与唤醒（官方文档）：
  https://developer.android.com/topic/performance/power/power-details
- Android 9 功耗变更：
  https://developer.android.com/about/versions/pie/power#buckets
- Android 12 行为变更：
  https://developer.android.com/about/versions/12/behavior-changes-12
- Android 13 行为变更：
  https://developer.android.com/about/versions/13/behavior-changes-13
- Android 14 行为变更：
  https://developer.android.com/about/versions/14/behavior-changes-14
- Android 15 行为变更：
  https://developer.android.com/about/versions/15/behavior-changes-15
- PowerManagerService（AOSP）：
  https://android.googlesource.com/platform/frameworks/base/+/main/services/core/java/com/android/server/power/PowerManagerService.java
- jobscheduler APEX 源码目录：
  https://android.googlesource.com/platform/frameworks/base/+/main/apex/jobscheduler/service/java/com/android/server/
- NVD 条目（本文台账）：
  https://nvd.nist.gov/vuln/detail/CVE-2021-0439
  https://nvd.nist.gov/vuln/detail/CVE-2020-0312
  https://nvd.nist.gov/vuln/detail/CVE-2022-20307
  https://nvd.nist.gov/vuln/detail/CVE-2022-20414
  https://nvd.nist.gov/vuln/detail/CVE-2023-20950
  https://nvd.nist.gov/vuln/detail/CVE-2023-21128

---

## 联系方式

咨询 Android 系统详情请联系 https://t.me/one00190

---

*本手册仅供安全学习与防御研究使用；所有机制表述、公告列值与 CVE 描述均取自公开一手来源并附链接，未能核实的部分已单列于「未核实与失效来源」而非写入正文。*
