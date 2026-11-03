---
layout: page
title: "Android 窗口管理、悬浮窗与界面覆盖攻击面手册"
description: "把「用户看到的就是他点到的那一层」拆成四段：窗口分层、触摸命中、覆盖授权、遮挡判定。类型侧列清 `TYPE_SYSTEM_ALERT` 是 2003（2002 才是 `TYPE_PHONE`）、`TYPE_TOAST` 2005、`TYPE_APPLICATION_OVERLAY` 2038 必须叠在 ..."
date: 2026-11-03
permalink: /android-window-overlay-tapjacking/
---

> 一次「点按」在 Android 上要穿过好几层代码：应用把视图画到 Surface 上，WindowManagerService 决定它排在哪一层、占多大、能不能接收触摸，SurfaceFlinger 把各层合成到屏幕，InputDispatcher 再按 Z 序和可触摸区域把触摸送回某一个窗口。攻击者不需要攻破其中任何一层，只要让「用户看到的」和「实际接收触摸的」错位，就能借用户的手指去点安全确认按钮。这类问题叫 tapjacking（界面覆盖 / 点击劫持），它在 Android 上前后修了十多年：从 2016 年的权限对话框，到 2020 年的 `addWindow`，再到 2024 年的旋转绕过、2026 年的 `setHiddenWhileSuspended`，公告里一直有新的落点。这份手册沿「窗口怎么分层 → 触摸怎么命中 → 覆盖层怎么被授权 → 哪些状态会让判定失效 → 防御方怎么查」这条链，把机制、公开缺陷模式和加固手段拆开说明。只讨论公开且已修复的问题，不含可利用代码。

## 0. 使用方式与边界

这份手册面向三类读者：

- **应用开发者**：需要知道什么时候该用 `filterTouchesWhenObscured`，什么时候必须用 `setHideOverlayWindows`，以及为什么「弹一个自定义确认框」在安全上不可靠。
- **系统与平台工程师**：需要理解 `WindowManagerService`、`SurfaceFlinger`、`InputDispatcher` 三者在「遮挡」这件事上的分工，以及为什么一个状态没同步就会变成提权。
- **安全评估者**：需要在看到一条「tapjacking」公告时，判断它落在窗口服务、输入分发、合成器还是某个系统应用对话框，而不是把所有覆盖层问题混为一类。

本文的三条自我约束：

1. 只引用公开且已修复的问题；CVE 的编号、函数归属与公告月份以 NVD 条目及其引用的 Android Security Bulletin 为准，厂商公告单独标注。
2. 不写利用步骤、不写 PoC、不写可直接复现的攻击参数；成因只描述到「哪一类状态或校验缺失」。
3. 查不到一手来源的说法一律进「未核实与失效来源」，不写进正文。

与已发布手册的分工：

- 《图形与显示栈手册》讲的是 SurfaceFlinger、GPU、HWC 与合成管线；本文只讲窗口层级、输入命中与覆盖层权限，不展开渲染。
- 《框架 IPC 导出面手册》讲 Binder 接口与组件导出；本文只讲窗口服务和输入分发中与「遮挡」相关的入口。
- 《无障碍、输入法与自动填充通道手册》讲无障碍服务的能力边界；本文只讲无障碍覆盖层为什么被系统视为「受信窗口」。
- 《蓝牙与 BLE 配对、绑定与权限模型手册》《USB 主机、配件与有线外设攻击面手册》讲各自协议栈；本文把它们被覆盖层劫持的对话框作为案例，不重复协议细节。
- 《摄像头、麦克风与屏幕捕获面手册》讲采集与截屏权限；本文只把 `FLAG_SECURE` 作为易混点纠正，不展开录屏。

## 1. 一次触摸的完整旅程

把「用户点了一下屏幕」拆开，至少经过五个阶段，每个阶段都可能让「看到的位置」和「收到事件的窗口」不一致：

| 阶段 | 主要代码 | 做的事 | 与覆盖层的关系 |
|------|----------|--------|----------------|
| 1. 窗口布局 | `WindowManagerService`（system_server） | 决定窗口类型、层级、大小、可触摸区域 | 层级决定谁在上；类型决定要不要悬浮窗权限 |
| 2. 合成 | `SurfaceFlinger` | 把各层按 Z 序合成到显示 | 视觉上的「上面」由它决定 |
| 3. 输入命中 | `InputDispatcher`（native input 栈） | 按 Z 序找命中窗口，计算是否被遮挡 | 这里产生 `FLAG_WINDOW_IS_OBSCURED` |
| 4. 通道投递 | `InputChannel` | 把事件送进目标应用 | 目标窗口可能不是用户以为的那个 |
| 5. 视图分发 | `ViewRootImpl` → `View` | 派发到具体控件 | `onFilterTouchEventForSecurity()` 可在这里丢弃事件 |

关键点是：**「视觉上被盖住」和「输入上被判定为遮挡」不是同一件事**。合成器只负责画；输入分发要自己根据窗口信息重算遮挡关系。两者一旦不同步，就会出现「用户看到一个按钮，触摸却落到另一个窗口」的情况。公开 CVE 里有相当一部分就落在这种不同步上，而不是落在某个内存破坏缺陷上。

## 2. 对象模型与进程边界

### 2.1 关键对象

| 对象 | 所在位置 | 职责 |
|------|----------|------|
| `WindowManagerService`（WMS） | `system_server` | 窗口的添加、移除、重排、权限检查、输入窗口信息生成 |
| `WindowState` | WMS 内部 | 单个窗口的状态：类型、flag、可见性、挂起状态、输入通道 |
| `DisplayContent` | WMS 内部 | 一块显示上的窗口集合与层级计算 |
| `SurfaceControl` | 应用与系统均可持有 | 指向 SurfaceFlinger 中一个层的句柄 |
| `InputWindowHandle` / `WindowInfo` | 输入栈 | 交给 `InputDispatcher` 的窗口描述：层级、可触摸区、遮挡策略 |
| `InputChannel` | 每窗口一对 | 应用与输入分发之间的单向通道 |
| `Layer` | SurfaceFlinger | 合成用的层；没有输入信息的层只参与遮挡检测 |

### 2.2 进程与 SELinux 域

AOSP 的 `system/sepolicy` 为这条链路划了不同的域（本地 `system/sepolicy` 检出已核实这些类型存在）：

- 普通应用运行在 `untrusted_app` 域，只能通过 Binder 与 `system_server`、`SurfaceFlinger` 等交互；
- `WindowManagerService` 在 `system_server` 域内；
- 合成器在 `surfaceflinger` 域；
- 输入栈有独立的 `inputflinger` 域（策略中 `binder_call(inputflinger, system_server)`、`wakelock_use(inputflinger)`）；
- 厂商显示与分配器 HAL 分别在 `hal_graphics_composer_default`、`hal_graphics_allocator_default` 域。

这意味着：**覆盖层攻击通常不需要跨越 SELinux 边界**。攻击者用的是「合法 API + 合法权限 + 状态错位」，而不是越权读写另一个域的文件。反过来说，SELinux 也挡不住这类问题；能挡的是窗口服务和输入分发自己的判定逻辑。

## 3. 窗口类型与层级

### 3.1 类型编号

`WindowManager.LayoutParams` 把系统窗口定义在 2000–2999。AOSP `WindowManager.java` 中的实际取值（已逐行核对）如下：

| 常量 | 数值 | 说明 |
|------|------|------|
| `FIRST_SYSTEM_WINDOW` | 2000 | 系统窗口起始 |
| `TYPE_STATUS_BAR` | 2000 | 状态栏 |
| `TYPE_SYSTEM_ALERT` | 2003 | 旧式系统警告窗；API 26 起对非系统应用弃用 |
| `TYPE_TOAST` | 2005 | Toast 窗口；API 26 起对非系统应用弃用 |
| `TYPE_SYSTEM_OVERLAY` | 2006 | 旧式系统覆盖层；API 26 起对非系统应用弃用 |
| `TYPE_INPUT_METHOD` | 2011 | 输入法窗口 |
| `TYPE_NAVIGATION_BAR` | 2019 | 导航栏 |
| `TYPE_ACCESSIBILITY_OVERLAY` | 2032 | 无障碍覆盖层，API 22 |
| `TYPE_APPLICATION_OVERLAY` | 2038 | 应用悬浮窗，API 26 |
| `LAST_SYSTEM_WINDOW` | 2999 | 系统窗口结束 |

几个容易记错的点：

- `TYPE_SYSTEM_ALERT` 是 **2003**，不是 2002；2002 是 `TYPE_PHONE`。
- `TYPE_APPLICATION_OVERLAY` 是 **2038**，不是 2002/2003。
- 类型编号只表示「属于系统窗口区间」，**不等于最终 Z 序**。真正的前后关系由 WMS 的层级计算与窗口 flag 决定。

### 3.2 官方层级描述

AOSP 对 `TYPE_APPLICATION_OVERLAY` 的注释写得很明确：它显示在**所有 Activity 窗口之上**（即 `FIRST_APPLICATION_WINDOW` 到 `LAST_APPLICATION_WINDOW` 之间），但**在状态栏、输入法等关键系统窗口之下**。注释同时说明：该类型需要 `SYSTEM_ALERT_WINDOW` 权限，系统可能随时改变这类窗口的位置、大小或可见性，并会调整持有该窗口的进程的重要性以降低被 LMK 杀掉的概率。

这意味着应用悬浮窗的「可覆盖范围」是有限的：它盖得住别的应用，盖不住系统关键 UI。攻击者若想盖住状态栏或输入法，需要另一条路径（例如无障碍覆盖层或系统 UI 自身的缺陷），这也是为什么很多历史 CVE 落在系统应用对话框，而不是普通应用窗口。

### 3.3 关键 flag

| flag | 值 | 语义（官方原文要点） |
|------|----|----------------------|
| `FLAG_NOT_FOCUSABLE` | 0x00000008 | 永不获得按键焦点；隐含 `FLAG_NOT_TOUCH_MODAL` |
| `FLAG_NOT_TOUCHABLE` | 0x00000010 | 永不接收触摸，触摸交给下层窗口 |
| `FLAG_NOT_TOUCH_MODAL` | 0x00000020 | 窗口外的事件可传给后面的窗口 |
| `FLAG_LAYOUT_NO_LIMITS` | 0x00000200 | 允许窗口延伸到屏幕外 |
| `FLAG_SECURE` | 0x00002000 | 内容视为安全，禁止出现在截图/非安全显示上 |
| `FLAG_WATCH_OUTSIDE_TOUCH` | 0x00040000 | 配合 `FLAG_NOT_TOUCH_MODAL` 接收 `ACTION_OUTSIDE` |

`FLAG_NOT_TOUCHABLE` 是覆盖层攻击的核心：它让窗口「看得见、摸不着」，触摸继续落到下面的窗口。Android 12 的不可信触摸拦截，针对的正是这一类「穿透式」窗口。

## 4. 悬浮窗权限与 AppOps

### 4.1 SYSTEM_ALERT_WINDOW

AOSP `core/res/AndroidManifest.xml` 中该权限的声明原文是：

```
android:protectionLevel="signature|setup|appop|installer|pre23|development"
```

它不是普通运行时权限，而是由 **AppOps** 管控的特殊权限：

- Android 6（API 23）起，用户必须在设置里显式授予，应用才能创建覆盖层；
- 应用用 `Settings.canDrawOverlays(Context)`（API 23）检查是否已获授权；
- 跳转入口是 `Settings.ACTION_MANAGE_OVERLAY_PERMISSION`（API 23），可带 `package:` URI 直达本应用；
- AOSP 在添加窗口时由窗口管理策略调用 `AppOpsManager.noteOpNoThrow(OP_SYSTEM_ALERT_WINDOW, …)` 判定，`MODE_ALLOWED` 或 `MODE_IGNORED` 才放行（`MODE_IGNORED` 时窗口实际被隐藏）。

注意 `OP_SYSTEM_ALERT_WINDOW` 在新版 AOSP 中已改为通过 `AppOpEnums` 间接定义，**不要再把「24」当作稳定编号写进检测脚本**；用字符串名 `SYSTEM_ALERT_WINDOW` 查询 AppOps 更稳妥。

### 4.2 HIDE_OVERLAY_WINDOWS

AOSP 同时声明了 `android.permission.HIDE_OVERLAY_WINDOWS`，保护级别是 **normal**：

```xml
android:protectionLevel="normal"
```

它配合 `Window#setHideOverlayWindows(boolean)`（API 31）使用。该方法在 `Window.java` 中是 `final`，源码注释是「Prevent non-system overlay windows from being drawn on top of this window.」；实现里检查 `HIDE_OVERLAY_WINDOWS` 或 `HIDE_NON_SYSTEM_OVERLAY_WINDOWS`，并设置内部 flag `SYSTEM_FLAG_HIDE_NON_SYSTEM_OVERLAY_WINDOWS = 1 << 19`。

两个细节必须说清：

1. 它隐藏的是**非系统**覆盖层，系统关键窗口不受影响；
2. 它是「本窗口可见期间」的抑制，不是全局关闭悬浮窗。窗口不可见后，其他覆盖层照常显示。

### 4.3 AppOps 与用户可见性

悬浮窗权限的用户可见入口是「显示在其他应用上层」。企业或评估场景可以用 AppOps 做盘点与收紧，但要注意：

- `appops set ... deny` 只改变 AppOps 状态，不卸载应用；
- 部分厂商 ROM 会在标准 AppOps 之外再加一层二次确认（见「未核实」F）；
- 修改 AppOps 属于设备管理动作，需在授权范围内进行。

## 5. 输入分发如何判定「被遮挡」

### 5.1 Android 12 之前：两个 flag

输入系统在投递触摸时，会根据目标窗口上方是否有其他可见窗口，设置 `MotionEvent` 的 flag：

| flag | 值 | 语义 |
|------|----|------|
| `FLAG_WINDOW_IS_OBSCURED` | 0x1 | 窗口被另一个可见窗口部分或全部遮挡，且事件**直接穿过了**遮挡区域 |
| `FLAG_WINDOW_IS_PARTIALLY_OBSCURED` | 0x2 | 窗口被遮挡，但事件**没有**直接穿过被遮挡区域；Android 10（API 29）起公开 |

应用侧的经典防护是 `View#setFilterTouchesWhenObscured(true)`（API 9）或布局属性 `android:filterTouchesWhenObscured="true"`。AOSP `View.onFilterTouchEventForSecurity()` 的实现只检查 `FLAG_WINDOW_IS_OBSCURED`：

```java
if ((mViewFlags & FILTER_TOUCHES_WHEN_OBSCURED) != 0
        && (event.getFlags() & MotionEvent.FLAG_WINDOW_IS_OBSCURED) != 0) {
    return false;
}
return true;
```

也就是说：**它挡的是「全遮挡且事件穿透」，不挡部分遮挡**。部分遮挡需要应用自己检查 `FLAG_WINDOW_IS_PARTIALLY_OBSCURED`，官方明确说这一场景「没有默认防护」。

### 5.2 Android 12 的不可信触摸拦截

Android 12（API 31）起，系统默认阻止应用消费「被覆盖层不安全遮挡」的触摸。官方行为变更页的规则可以归纳为：

**受影响的对象**：

- 使用 `SYSTEM_ALERT_WINDOW` 的覆盖层（例如 `TYPE_APPLICATION_OVERLAY`）且带 `FLAG_NOT_TOUCHABLE`；
- 带 `FLAG_NOT_TOUCHABLE` 的 Activity 窗口。

**放行的例外**：

1. **同一应用内的交互**：覆盖层只在本应用交互时出现；
2. **受信窗口**：无障碍窗口（`TYPE_ACCESSIBILITY_OVERLAY`）、输入法窗口（`TYPE_INPUT_METHOD`）、Assistant 窗口等；官方特别注明 **`TYPE_APPLICATION_OVERLAY` 不属于受信窗口**；
3. **不可见窗口**：根视图为 `GONE` 或 `INVISIBLE`；
4. **完全透明窗口**：`alpha == 0.0`；
5. **足够透明的系统警告窗**：单个 `TYPE_APPLICATION_OVERLAY` 的 alpha，或多个此类窗口的**组合不透明度**，不超过系统允许的最大遮挡不透明度。Android 12 默认阈值是 **0.8**。

组合不透明度的公式在 `FLAG_NOT_TOUCHABLE` 的文档里给出：

```
opacity({W1, .., Wn}) = 1 - (1 - opacity(W1)) * ... * (1 - opacity(Wn))
```

系统还提供 `InputManager#getMaximumObscuringOpacityForTouch()` 读取当前阈值。触摸被拦截时，Logcat 会打印：

```
Untrusted touch due to occlusion by PACKAGE_NAME
```

调试用的兼容开关（官方文档给出）：

```bash
adb shell am compat disable BLOCK_UNTRUSTED_TOUCHES com.example.app
adb shell settings put global block_untrusted_touches 0
# 恢复默认
adb shell am compat reset BLOCK_UNTRUSTED_TOUCHES com.example.app
adb shell settings put global block_untrusted_touches 2
```

这里有两个常见误解：

- 它**不是**「所有覆盖层的所有触摸都拦」；同一 UID、受信窗口、不可见、全透明、低不透明度的系统警告窗都在例外里。
- 它**不是**「Android 12 之后 tapjacking 就没了」；部分遮挡仍然没有默认防护，系统应用对话框也仍然出过 CVE（见第 7 节）。

### 5.3 受信覆盖层与内部表达

AOSP 在输入窗口信息里用 `touchOcclusionMode` 表达窗口的遮挡策略（默认 `BLOCK_UNTRUSTED`），`InputDispatcher` 侧对应 `findTouchedWindowTargetsLocked`、`isWindowObscuredAtPointLocked`、`isTouchTrustedLocked`、`filterUntrustedTargets` 等函数，判据涉及 `WindowInfo::InputConfig::TRUSTED_OVERLAY`。

平台侧还有一条隐藏通路：`SurfaceControl.Transaction#setTrustedOverlay(SurfaceControl, boolean)`。AOSP 注释写的是「Trusted overlay state prevents SurfaceControl from being considered as obscuring for input occlusion detection purposes.」调用者必须持有 `ACCESS_SURFACE_FLINGER` 权限。它是**平台内部 API**，不是第三方应用可用的公开能力；它的存在说明「受信覆盖层」是合成器与输入分发共享的一种标记，而不是应用自己能声明的属性。

### 5.4 触摸区域与 SurfaceControl 边界

`InputDispatcher` 命中窗口时依赖 WMS 提供的**可触摸区域**（touchable region）。这个区域应当与窗口自己的 `SurfaceControl` 边界一致。历史上有多条 CVE 落在这里：

- `updateInputChannel` 允许把触摸区设到自身 `SurfaceControl` 之外（CVE-2023-21026，DoS）；
- `finishDrawingWindow` 对输入参数的校验不足（CVE-2022-20226）；
- `SurfaceFlinger.setTransactionState` 的逻辑错误导致合成与输入状态不一致（CVE-2024-34743）。

这类问题的共同点是：**视觉边界、触摸边界、输入窗口信息三者由不同代码路径维护**，任何一处没对齐，就会产生「点在 A 上、事件进 B」的效果。

## 6. 覆盖层的合法用途与被滥用方式

### 6.1 应用悬浮窗

`TYPE_APPLICATION_OVERLAY` 的合法用途包括悬浮球、录屏控制条、翻译浮层等。风险点：

- 带 `FLAG_NOT_TOUCHABLE` 的透明/半透明覆盖层是 tapjacking 的典型载体；
- 覆盖层可以跟随用户操作移动，形成动态遮挡；
- 即使不接收触摸，也能通过视觉诱导改变用户点击位置。

### 6.2 无障碍覆盖层

`TYPE_ACCESSIBILITY_OVERLAY`（API 22）的官方定义是：仅由已连接的无障碍服务覆盖，用于拦截用户交互而不改变无障碍服务可内省的窗口。它被 Android 12 的规则列为**受信窗口**，因此不受不可信触摸拦截。

这带来一个结构性事实：**无障碍服务本身是高权限组件**，一旦用户开启，它既能读屏、也能注入手势（`dispatchGesture`），还能绘制覆盖层。防御重点不是「把无障碍窗口当普通覆盖层拦掉」，而是：

- 只在确有必要时开启无障碍服务；
- 应用用 Android 14 的 `accessibilityDataSensitive` 保护敏感视图（只允许声明 `android:isAccessibilityTool="true"` 的服务读取或操作）；
- Android 16（API 36）起，`accessibilityDataSensitive` 还会阻止未声明该属性的服务通过 `dispatchGesture` 注入模拟触摸（官方 tapjacking 页明确写出）。

### 6.3 自定义 Toast 与 toast burst

`Toast.setView()` 允许自定义外观。官方 tapjacking 页记录了两条演进：

- Android 11（API 30）起，targetSdk ≥ 30 的应用在后台无法弹自定义 Toast；
- Android 12（API 31）起，toast burst（前台排队、退到后台继续弹）也被完全缓解。

但这不等于 Toast 不再是视觉诱导载体；它只是不能从后台持续弹出。评估时仍应把「前台期间的自定义 Toast」计入遮挡面。

### 6.4 Bubbles 与画中画

- **Bubbles**：`Notification.BubbleMetadata`，由系统 UI 管理，官方文档未要求 `SYSTEM_ALERT_WINDOW`；
- **画中画（PiP）**：`Activity#enterPictureInPictureMode`，由系统窗口管理，官方明确建议**不要**用 `SYSTEM_ALERT_WINDOW` 去实现 PiP。

把 Bubbles 或 PiP 说成「需要悬浮窗权限」是不准确的；它们走的是系统托管的窗口通道，权限与生命周期都由系统控制。

### 6.5 Activity sandwich（三明治）

官方 tapjacking 页描述了一种部分遮挡手法：恶意应用先让用户打开自己的界面，再启动受害应用的 Activity，并在其上覆盖自己的 Activity，形成「受害界面夹在中间」的结构。由于触摸区域本身没有被盖住，这类攻击不触发全遮挡防护。

对应的防御有两条：

1. 按第 5.1 节处理 `FLAG_WINDOW_IS_PARTIALLY_OBSCURED`；
2. **不要导出不需要导出的 Activity**，减少被「夹」的入口。

### 6.6 系统对话框

历史上最多的案例不是普通应用窗口，而是**系统应用自己弹出的安全确认对话框**：权限授予、蓝牙配对、USB 授权、证书安装、卸载确认、账户授权、工作资料设置等。攻击者用一个悬浮窗盖住对话框的一部分，诱导用户点到「允许」。

这类问题的修复方式通常不是「加一个权限」，而是：

- 给对话框窗口设置「隐藏非系统覆盖层」；
- 对关键按钮启用遮挡触摸过滤；
- 避免使用默认不设防的窗口 flag；
- 在对话框重建、旋转、配置变化时重新计算遮挡状态。

## 7. 公开公告台账

以下条目的编号、函数名与公告月份，均以 NVD 条目及其引用的 Android Security Bulletin 为准；「类型」取自 NVD 描述（EoP = 本地提权，DoS = 拒绝服务）。修复版本以各公告页为准，本表不替代公告。

### 7.1 窗口服务、输入分发与合成器

| CVE | 落点 | 类型 | 公告月份（NVD 引用） | 成因要点 |
|-----|------|------|----------------------|----------|
| CVE-2016-2496 | Framework UI 权限对话框 | EoP | 2016-06 | 部分重叠窗口可劫持权限对话框，进而访问私有存储 |
| CVE-2018-9524 | System UI 覆盖窗口 | EoP | 2018-11 | 覆盖窗口周围的保护不足 |
| CVE-2020-0099 | `WindowManagerService.addWindow` | EoP | 2020-12 | 不安全的默认值导致窗口覆盖攻击 |
| CVE-2020-0475 | `WindowManagerService.createInputConsumer` | EoP | Pixel 2020-12 | 缺权限检查，可创建输入消费者拦截事件 |
| CVE-2021-0438 | `InputDispatcher.cpp`、`WindowManagerService.java` | EoP | 2021-04 | `FLAG_OBSCURED` 取值错误导致 tapjacking |
| CVE-2021-39691 | WindowManager 输入处理 | EoP | 2022-06 | 处理用户输入时窗口 flag 错误 |
| CVE-2022-20226 | `WindowManagerService.finishDrawingWindow` | EoP | 2022-07 | 输入校验不当，窗口可冒充可触摸层 |
| CVE-2022-20443 | `Layer.cpp` 的 `hasInputInfo` | EoP | Android 13 公告 | 输入信息不实，绕过用户交互要求 |
| CVE-2023-21026 | `WindowManagerService.updateInputChannel` | DoS | Pixel 2023-03 | 触摸区可超出自身 `SurfaceControl` |
| CVE-2024-31324 | `WindowState.hide` | EoP | 2024-06 | 先竖屏再旋转到横屏，状态不同步绕过防护 |
| CVE-2024-34743 | `SurfaceFlinger.cpp` 的 `setTransactionState` | EoP | 2024-08 | 逻辑错误导致 tapjacking |
| CVE-2025-48634 | `WindowManagerService.relayoutWindow` | EoP | 2026-03 | 缺权限检查，可重排他人窗口 |
| CVE-2026-28577 | `WindowManagerService.addWindow` | EoP | 2026-06 | 覆盖窗口校验缺失 |
| CVE-2026-28650 | `WindowState.setHiddenWhileSuspended` | EoP | 2026-09 | 隐藏状态逻辑错误导致覆盖绕过 |

### 7.2 AOSP 系统应用对话框

| CVE | 落点 | 类型 | 公告月份（NVD 引用） | 成因要点 |
|-----|------|------|----------------------|----------|
| CVE-2020-0051 | `SettingsHomepageActivity` | EoP | Pixel 2020-03 | 设置首页可被覆盖点击 |
| CVE-2020-0386 | `RequestPermissionActivity` | EoP | 2020-09 | 不安全默认值，可诱导授予蓝牙权限 |
| CVE-2020-0394 | `BluetoothPairingDialog` | EoP | 2020-09 | 配对对话框可被覆盖，诱骗配对不可信设备 |
| CVE-2020-0416 | 多个 Settings 页面 | EoP | 2020-10 | 设置页默认可被覆盖，骗取权限 |
| CVE-2020-27059 | `AuthenticationClient` | EoP | Pixel 2021-01 | 指纹确认对话框被覆盖 |
| CVE-2021-0315 | `GrantCredentialsPermissionActivity` | EoP | 2021-01 | 诱骗用户授予账户访问 |
| CVE-2021-0314 | `UninstallerActivity` | EoP | 2021-02 | 无知情同意下卸载应用 |
| CVE-2021-0333 | `BluetoothPermissionActivity` | EoP | 2021-02 | 蓝牙连接时电话簿权限对话框被遮挡 |
| CVE-2021-0433 | `DeviceChooserActivity` | EoP | 2021-04 | 绕过蓝牙配对用户同意 |
| CVE-2021-0506 | `ActivityPicker` | EoP | 2021-06 | Intent 解析的用户交互被绕过 |
| CVE-2021-0954 | `ResolverActivity` | EoP | 2021-12 | 选择器对话框可被覆盖 |
| CVE-2021-0963 | `KeyChainActivity` | EoP | 2021-12 | 可诱导使用 KeyChain 中的应用证书 |
| CVE-2021-1016 | `UsbPermissionActivity` | EoP | Pixel 2021-12 | 无知情同意下授予 USB 访问 |
| CVE-2022-20442 | `ReviewPermissionsActivity` | EoP | 2022-12 | 对 API < 23 的应用替用户授权 |
| CVE-2022-20501 | `EnableAccountPreferenceActivity` | EoP | 2022-12 | 诱骗启用恶意电话账户 |
| CVE-2024-31323 | 健康权限相关多个文件 | EoP | 2024-06 | 诱骗授予健康权限 |

### 7.3 两个「类型容易写错」的条目

| CVE | 实际类型 | 说明 |
|-----|----------|------|
| CVE-2021-0569 | 信息泄露 | `ContactsDumpActivity` 被覆盖后可访问联系人；不是提权 |
| CVE-2023-21026 | DoS | 触摸区越界导致拒绝服务；不是提权 |

### 7.4 厂商公告

| CVE | 落点 | 说明 |
|-----|------|------|
| CVE-2022-33727 | Samsung `SecDevicePickerDialog` | 仅三星 SMR（2022-08），描述为覆盖层诱骗选择蓝牙设备；**不代表 AOSP 有同类缺陷** |

另有两条常被误归入覆盖层的厂商条目：CVE-2023-21440 描述的是 `WindowManagerService` 相关的**截屏/信息泄露**，CVE-2024-34637 描述的是**绕过后台启动限制**；两者都不是 tapjacking，引用时不要按组件名归类（见第 10 节）。

### 7.5 从台账能读出的趋势

1. **2016–2020 年**集中在系统应用对话框：修复方式多为给窗口加遮挡过滤或隐藏覆盖层。
2. **2020–2022 年**开始深入窗口服务与输入分发：`addWindow`、`createInputConsumer`、`FLAG_OBSCURED`、`hasInputInfo`。
3. **2023–2026 年**转向**状态同步**与**边界一致性**：旋转、隐藏/挂起、重排、触摸区越界、合成事务。
4. 公告里依然持续出现新条目，说明这不是「一次性修完」的类别，而是窗口/输入架构本身的持续属性。

## 8. 六种成因模式

### 8.1 对话框默认不设防

系统应用弹窗时使用了默认窗口 flag，既没有 `filterTouchesWhenObscured`，也没有「隐藏非系统覆盖层」。攻击者只要盖住按钮区域即可。代表：CVE-2016-2496、CVE-2020-0394、CVE-2020-0416、CVE-2021-0333、CVE-2024-31323。

### 8.2 「可见」与「可触摸」脱节

输入分发依赖 WMS 提供的窗口信息；合成器依赖自己的 Layer 状态。两者对「谁在上面」的判断不一致时，遮挡 flag 就会算错。代表：CVE-2021-0438、CVE-2022-20443、CVE-2024-34743。

### 8.3 状态切换不同步

旋转、进入画中画、应用被挂起、窗口被隐藏/重排后，遮挡状态没有重新计算。代表：CVE-2024-31324（旋转）、CVE-2026-28650（挂起隐藏）、CVE-2025-48634（重排）。

### 8.4 系统服务入口缺权限或默认值不安全

`addWindow`、`createInputConsumer`、`relayoutWindow` 这类入口如果没有严格校验调用者，第三方应用就能替别人建窗口、建输入通道或重排窗口。代表：CVE-2020-0099、CVE-2020-0475、CVE-2025-48634、CVE-2026-28577。

### 8.5 触摸区域越界

可触摸区域被设置到窗口自己的 `SurfaceControl` 之外，导致「点到窗口外也命中该窗口」。代表：CVE-2023-21026、CVE-2022-20226。

### 8.6 部分遮挡与手势注入

部分遮挡没有默认防护；再叠加无障碍服务的手势注入能力，就形成官方所说的「eavesdropping and click injection」场景。代表：官方 tapjacking 页的 partial occlusion 章节；Android 16 对 `dispatchGesture` 的限制就是针对这一类。

## 9. 加固时间线

| 版本 | API | 变更 | 对覆盖层的影响 |
|------|-----|------|----------------|
| Android 6 | 23 | `SYSTEM_ALERT_WINDOW` 改为用户显式授予 | 悬浮窗从「安装即得」变成「用户开关」 |
| Android 8.0 | 26 | 引入 `TYPE_APPLICATION_OVERLAY`，弃用非系统应用的 `TYPE_SYSTEM_ALERT`/`TYPE_TOAST`/`TYPE_SYSTEM_OVERLAY` | 应用覆盖层统一到 2038 |
| Android 10 | 29 | `FLAG_WINDOW_IS_PARTIALLY_OBSCURED` 公开 | 应用可自行处理部分遮挡 |
| Android 11 | 30 | 后台自定义 Toast 被阻止（targetSdk ≥ 30） | 减少后台视觉诱导 |
| Android 12 | 31 | 不可信触摸默认拦截；`Window#setHideOverlayWindows`；`ACTION_CLOSE_SYSTEM_DIALOGS` 弃用；toast burst 完全缓解 | 全遮挡有系统级兜底；系统对话框可主动隐藏覆盖层 |
| Android 13 | 33 | `Activity#setRecentsScreenshotEnabled` | 控制最近任务截图（注意：与 tapjacking 不是同一问题） |
| Android 14 | 34 | `accessibilityDataSensitive` | 限制非无障碍工具服务读取/操作敏感视图 |
| Android 16 | 36 | `accessibilityDataSensitive` 进一步阻止非无障碍工具服务注入手势 | 直接针对部分遮挡中的手势注入 |

这张表说明一个方向：**平台在把「全遮挡」变成系统默认防护，把「部分遮挡」和「手势注入」交给应用声明 + 无障碍模型处理**。应用如果什么都不做，并不等于安全。

## 10. 观测点与自查

### 10.1 设备侧排查

以下命令用于授权范围内的自查与取证；不同版本 `dumpsys` 字段会变化（见「未核实」G），不要把字段名写死进自动化。

```bash
# 查看哪些应用持有悬浮窗 AppOp
adb shell appops get <package> SYSTEM_ALERT_WINDOW

# 收紧某个应用的悬浮窗（设备管理动作）
adb shell cmd appops set <package> SYSTEM_ALERT_WINDOW deny

# 窗口层级、类型、可见性
adb shell dumpsys window windows

# 输入窗口信息与遮挡相关状态
adb shell dumpsys input

# 合成层
adb shell dumpsys SurfaceFlinger

# 无障碍服务与覆盖层
adb shell dumpsys accessibility

# 不可信触摸拦截日志
adb logcat | grep "Untrusted touch"
```

排查时按「谁在上、谁可触摸、谁被标记为遮挡」三列对照，比单看某一层更容易定位错位。

### 10.2 应用代码检查清单

1. 安全确认按钮、支付/授权/删除等敏感控件，是否设置了 `android:filterTouchesWhenObscured="true"` 或调用 `setFilterTouchesWhenObscured(true)`？
2. 是否处理了 `FLAG_WINDOW_IS_PARTIALLY_OBSCURED`？官方明确说这一场景没有默认防护。
3. 敏感界面是否调用了 `Window#setHideOverlayWindows(true)`（API 31+）？是否声明了 `HIDE_OVERLAY_WINDOWS`（normal）？
4. 是否还在使用 `TYPE_SYSTEM_ALERT`/`TYPE_TOAST`/`TYPE_SYSTEM_OVERLAY` 创建面向用户的窗口？targetSdk ≥ 26 应改用 `TYPE_APPLICATION_OVERLAY`。
5. 悬浮窗是否真的必要？是否避免在悬浮窗上放可被诱导点击的安全操作？
6. 安全决策是否只依赖「用户点了一个按钮」？关键操作应叠加生物识别（`BiometricPrompt` + `CryptoObject`）或服务端确认。
7. 敏感内容是否加了 `FLAG_SECURE`？注意它防的是截图/录屏，不防覆盖层点击。
8. 是否导出了不需要导出的 Activity？这会扩大 activity sandwich 的入口。
9. Android 14+ 的敏感视图是否使用 `accessibilityDataSensitive`？
10. 自定义 Toast 是否仍在前台展示敏感信息或诱导性文案？

### 10.3 系统与平台侧检查清单

1. 所有安全确认对话框是否统一走「隐藏非系统覆盖层 + 遮挡触摸过滤」的基类？
2. WMS 的 `addWindow`、`updateInputChannel`、`relayoutWindow`、`createInputConsumer` 是否有调用者校验与边界钳制？
3. 窗口隐藏、挂起、旋转、重排、进入/退出多窗口时，是否重新计算遮挡状态？
4. `SurfaceControl.Transaction#setTrustedOverlay` 是否只用于系统 UI，且调用方确实持有 `ACCESS_SURFACE_FLINGER`？
5. 是否监控 `Untrusted touch due to occlusion by ...` 日志并纳入异常检测？
6. 企业设备是否定期盘点持有 `SYSTEM_ALERT_WINDOW` 的应用？

## 11. 误传纠正

| 常见说法 | 实际情况 |
|----------|----------|
| 「`TYPE_SYSTEM_ALERT` 是 2002」 | 是 **2003**；2002 是 `TYPE_PHONE`。 |
| 「`TYPE_APPLICATION_OVERLAY` 是 2002 或 2003」 | 是 **2038**，API 26 引入。 |
| 「`FLAG_SECURE` 可以防 tapjacking」 | 它防的是截图/录屏与非安全显示；覆盖层点击是另一条路径。 |
| 「Android 12 之后所有覆盖层触摸都被拦」 | 只拦「穿透式」不可信触摸，且有同 UID、受信窗口、不可见、全透明、≤0.8 不透明度等例外。 |
| 「`setFilterTouchesWhenObscured` 能挡部分遮挡」 | AOSP 实现只检查 `FLAG_WINDOW_IS_OBSCURED`；部分遮挡要自己处理 `FLAG_WINDOW_IS_PARTIALLY_OBSCURED`。 |
| 「`HIDE_OVERLAY_WINDOWS` 是 signature 权限」 | 声明为 **normal**。 |
| 「`setHideOverlayWindows(true)` 会关闭所有悬浮窗」 | 只隐藏**非系统**覆盖层，且只在本窗口可见期间生效。 |
| 「Bubbles 需要悬浮窗权限」 | 官方文档未要求 `SYSTEM_ALERT_WINDOW`；它由系统 UI 托管。 |
| 「画中画就是悬浮窗」 | PiP 由系统窗口管理；官方明确不建议用 `SYSTEM_ALERT_WINDOW` 实现。 |
| 「无障碍窗口都是可信的，所以无障碍覆盖层安全」 | 系统把无障碍窗口列为受信窗口，但无障碍服务本身是高权限组件，开启需用户授权，滥用风险在服务侧。 |
| 「自定义 Toast 从后台弹是常态」 | Android 11 起 targetSdk ≥ 30 的后台自定义 Toast 被阻止；Android 12 起 toast burst 也被缓解。 |
| 「CVE-2023-21440 是覆盖层漏洞」 | 三星 SMR 条目，描述为 WMS 相关截屏/信息泄露，不是 tapjacking。 |
| 「CVE-2024-34637 是 tapjacking」 | 描述为绕过后台启动限制，与覆盖层无关。 |
| 「CVE-2021-0569 是提权」 | 是信息泄露（联系人导出界面被覆盖）。 |
| 「CVE-2022-39899 是覆盖层漏洞」 | 描述为 S Pen 手势输入注入，属输入鉴权问题，不是覆盖层。 |
| 「`TYPE_TOAST` 可以随便用」 | API 26 起对非系统应用弃用，targetSdk ≥ 26 后不能靠它自由建窗。 |

## 12. 与其他手册的分工

- 渲染、GPU、HWC、合成器内部：见《图形与显示栈手册》；本文只用到「合成层与输入信息可能不一致」这一点。
- Binder 接口、组件导出、PendingIntent：见《框架 IPC 导出面手册》；本文只讲 WMS 的窗口入口。
- 无障碍服务能力、输入法、自动填充：见《无障碍、输入法与自动填充通道手册》；本文只讲无障碍覆盖层为何受信。
- 蓝牙配对、USB 授权：见对应手册；本文把它们作为「系统对话框被覆盖」的案例。
- 截屏、录屏、媒体投影：见《摄像头、麦克风与屏幕捕获面手册》；本文只纠正 `FLAG_SECURE` 的用途。

## 13. 参考资料

- Android Developers：`WindowManager.LayoutParams` 常量与 flag 语义  
  https://developer.android.com/reference/android/view/WindowManager.LayoutParams
- Android Developers：`Window#setHideOverlayWindows`  
  https://developer.android.com/reference/android/view/Window
- Android Developers：`SYSTEM_ALERT_WINDOW` 权限  
  https://developer.android.com/reference/android/Manifest.permission#SYSTEM_ALERT_WINDOW
- Android Developers：Tapjacking 风险与缓解  
  https://developer.android.com/privacy-and-security/risks/tapjacking
- Android Developers：Android 12 行为变更（Untrusted touch events）  
  https://developer.android.com/about/versions/12/behavior-changes-all#untrusted-touch-events
- AOSP：`WindowManager.java`（窗口类型常量）  
  https://android.googlesource.com/platform/frameworks/base/+/refs/heads/main/core/java/android/view/WindowManager.java
- AOSP：`Window.java`（`setHideOverlayWindows`）  
  https://android.googlesource.com/platform/frameworks/base/+/refs/heads/main/core/java/android/view/Window.java
- AOSP：`MotionEvent.java`（遮挡 flag）  
  https://android.googlesource.com/platform/frameworks/base/+/refs/heads/main/core/java/android/view/MotionEvent.java
- AOSP：`View.java`（`onFilterTouchEventForSecurity`）  
  https://android.googlesource.com/platform/frameworks/base/+/refs/heads/main/core/java/android/view/View.java
- AOSP：`SurfaceControl.java`（`setTrustedOverlay`）  
  https://android.googlesource.com/platform/frameworks/base/+/refs/heads/main/core/java/android/view/SurfaceControl.java
- AOSP：`core/res/AndroidManifest.xml`（权限声明）  
  https://android.googlesource.com/platform/frameworks/base/+/refs/heads/main/core/res/AndroidManifest.xml
- NVD：CVE 详情与公告引用（按编号检索）  
  https://nvd.nist.gov/vuln/detail/CVE-2020-0099

## 14. 未核实与失效来源

**A. 系统窗口的精确 Z 序**
类型编号只给出区间；状态栏、导航栏、输入法、无障碍覆盖层之间的最终前后关系由 WMS 层级计算决定，官方没有一份与版本无关的完整排序表。正文只引用「应用覆盖层在 Activity 之上、状态栏/IME 之下」这一条官方描述。

**B. `getMaximumObscuringOpacityForTouch()` 的取值来源**
官方说 Android 12 默认是 0.8，但该方法是否允许厂商配置、以及各版本默认值是否变化，未逐一核实。正文按「默认 0.8」表述。

**C. `touchOcclusionMode` 的完整取值**
AOSP 当前源码中可确认 `InputWindowHandle` 使用该字段，并有 `BLOCK_UNTRUSTED` 语义；所有取值与默认值在 Android 16 上的组合未逐一核对。

**D. `WindowState.setHiddenWhileSuspended` 的完整生效条件**
可从源码读到「仅对无内部系统窗口权限、且为 system-alert 类型或 `TYPE_TOAST` 的窗口生效」这一方向，但不同版本细节可能变化；正文不把它写成跨版本承诺。

**E. 各 OEM 对悬浮窗的额外管控**
部分厂商 ROM 在标准 AppOps 之外增加二次确认、白名单或「后台弹出界面」开关，具体行为未逐机型核实；正文只描述 AOSP 语义。

**F. `dumpsys` 字段稳定性**
`dumpsys window`、`dumpsys input`、`dumpsys SurfaceFlinger` 的输出字段随版本和厂商变化；正文只把它们列为排查入口，不承诺字段名。

**G. AAOS 公告的适用性**
CVE-2021-0583、CVE-2021-1039 引用的是 Android Automotive OS 公告；它们是否影响手机版本未逐条核实，正文未将其列入主表。

**H. Pixel 专属公告**
CVE-2020-0475、CVE-2020-27059、CVE-2021-1016、CVE-2023-21026 引用的是 Pixel 公告；AOSP 主线是否同样受影响、影响范围多大，未逐条核对。

**I. 2026 年新公告的补丁细节**
CVE-2025-48634、CVE-2026-28577、CVE-2026-28650 的 NVD 条目与公告引用已核实；对应 AOSP 补丁提交的逐行差异未核对。

**J. Bubbles 与 PiP 在多窗口/折叠屏上的遮挡行为**
官方文档未给出与覆盖层判定相关的完整说明；正文只说明它们不由 `SYSTEM_ALERT_WINDOW` 控制。

**K. `TYPE_TOAST` 的历史行为**
不同 targetSdk 与版本下的建窗限制细节未逐版本核对；正文只写 API 26 起对非系统应用弃用。

**L. 无障碍覆盖层与输入法/状态栏的精确层级**
官方没有明文排序；正文避免给出具体前后关系。

**M. 部分遮挡的跨进程时序**
遮挡状态由输入分发计算、由合成器提供层信息，二者跨进程异步；具体时序窗口未在一手文档中找到可引用的精确描述。

**N. `accessibilityDataSensitive` 的厂商覆盖**
API 34/36 的语义已核实，但各厂商是否完整实现、对第三方无障碍服务的影响范围未逐机型核对。

---

## 联系方式

咨询 Android 系统详情请联系 https://t.me/one00190

---

*本手册仅供安全学习与防御研究使用；所有机制表述、公告列值与 CVE 描述均取自公开一手来源并附链接，未能核实的部分已单列于「未核实与失效来源」而非写入正文。*
