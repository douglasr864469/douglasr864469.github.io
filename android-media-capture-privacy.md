---
layout: page
title: "Android 摄像头、麦克风与屏幕捕获面手册"
description: "把「能录你」拆成三条对象不同的链路：相机与麦克风受权限 + app-op 双层控制（`AppOpsManager.startOp/stopOp/noteOp`、Android 9 起后台受限、Android 12 起有设备级开关与状态栏指示器），屏幕捕获则是**每次会话一次同意**（Android 14..."
date: 2026-10-29
permalink: /android-media-capture-privacy/
---

> 「捕获」是三类性质完全不同的东西：相机给你一帧帧图像，麦克风给你一条音频流，屏幕捕获给你**别人家应用的像素**。它们在 Android 里被三套 API、三种授权、三个不同的隐私开关管理，却经常被同一句话概括——「它有个权限就能录你」。这份手册把三条链路的对象、授权与失败语义分开讲，只讨论公开且已修复问题的机制与自查方法，不含可利用代码。

---

## 0. 这篇手册解决什么问题

已发布的二十篇里，`生物识别与锁屏` 讲的是「怎么证明是你」，`崩溃诊断与日志暴露` 讲的是「崩溃会留下什么」，但两者都绕开了一个更日常的问题：**设备上的传感器与屏幕，什么时候正在被别人看着**。

把「捕获」单独成册，有三个理由：

1. **三条链路的对象不同。** 相机与麦克风是**本机硬件采集**；屏幕捕获是**把别的应用的界面合成成像素交给你**。前者侵犯的是「环境」，后者侵犯的是「别人的数据」。
2. **授权模型不同。** 相机/麦克风是运行时权限 + 全局开关；屏幕捕获是**每次会话的用户同意**，而且这条规则在 Android 14 被显著收紧。
3. **失败语义不同。** 权限被拒是 `SecurityException`，全局开关关闭是「设备无相机可用」，屏幕捕获令牌复用也是 `SecurityException`——三种错误，处理方式完全不同，代码里混为一谈就是线上崩溃的来源。

因此本文按「对象 → 授权 → 链路 → 可观测性」展开，并在 §10 给出把捕获类 CVE 读对的三个要点。

---

## 1. 三条通道的对象

| 维度 | 相机 | 麦克风 | 屏幕捕获 |
|---|---|---|---|
| 采集对象 | 本机摄像头帧（含外接） | 本机/外接输入音频 | 显示合成的像素 |
| 主要 API | `CameraManager` / Camera2 / CameraX | `AudioRecord` / `MediaRecorder` | `MediaProjection` + `VirtualDisplay` |
| 授权形态 | 运行时权限 | 运行时权限 | **每次会话的用户同意** |
| 全局开关 | 有（Android 12 起） | 有（Android 12 起） | 无「全局开关」，有每次同意 |
| 后台可用性 | Android 9 起受限 | Android 9 起受限 | 需前台服务（类型 `mediaProjection`） |
| 用户可见性 | 状态栏指示器 | 状态栏指示器 | 状态栏/胶囊提示（版本相关） |
| 典型误用后果 | 偷拍、静默开启 | 窃听、环境录音 | 窃取他人会话、录屏含敏感信息 |

**边界一：屏幕捕获拿到的不是「你的界面」，而是「当前显示」。** 一旦合成进来，通知、弹窗、验证码、密码管理器界面都在画面里。这也解释了为什么系统要在捕获侧而不是展示侧加限制。

**边界二：相机与麦克风的权限是「应用级」，全局开关是「设备级」。** 应用拿到权限不等于能取到数据；用户把设备级开关关掉时，所有应用一起失效。代码必须处理这条路径，否则会把它误判成「硬件坏了」。

**边界三：音频输出的捕获是另一条完全不同的权限。** 采集「设备正在播放的声音」（内录）不是 `RECORD_AUDIO` 能覆盖的，它需要系统级签名权限 `CAPTURE_AUDIO_OUTPUT`，普通应用无法申请。真正给普通应用的合法内录通道是 `AudioPlaybackCapture`（Android 10 起），它只允许捕获**明确允许被捕获的应用**的音频。

---

## 2. 权限与 app-op：两层开关

Android 的传感器访问在实现上是两层的：

- **权限层**（permission）：`android.permission.CAMERA`、`android.permission.RECORD_AUDIO`，运行时授予，用户可拒绝、可选择「仅本次」。
- **操作层**（app-op）：真正决定「这一次能不能开」的是 `AppOpsManager` 里的操作记录。AOSP 的隐私指示器文档把这条写得很直接：**应用通过调用 `AppOpsManager.startOp`、`stopOp` 或 `noteOp` 来使用麦克风、相机或位置**。

这意味着三件事：

1. **权限允许 ≠ 操作放行。** 设备级开关、后台限制、厂商策略都可能让 app-op 结果为「忽略」，此时 API 调用会抛异常或返回空数据。
2. **指示器看的是 app-op，不是权限。** 一个持有相机权限但从未调用 `startOp` 的应用不会点亮指示器；反之，只要有一次相机打开，就会留下记录。
3. **审计点应落在 app-op 上。** 想回答「上周三谁开了麦克风」，看权限授予历史是错的，应看 app-op 的历史记录与隐私面板。

---

## 3. 后台限制：从「能不能用」到「什么时候能用」

Android 9（API 28）开始限制后台应用访问用户输入与传感器。官方行为变更文档的表述是：「Android 9 limits the ability for background apps to access user input and sensor data.」——**限制的是「后台」这个状态，而不是权限本身**。

对开发者的直接后果：

- 后台线程里直接开相机或录音，在 Android 9+ 上会失败或拿到静默数据；
- 需要长时间采集时，只能用**前台服务**，并声明对应的 `foregroundServiceType`（相机、麦克风各有类型）；
- 前台服务类型不是装饰：声明了类型却无对应权限，或权限被撤销，系统会拒绝启动该服务。

对评审者的直接后果：**「应用在我睡觉时录了音」这类报告，第一件事是核对它当时是否处于前台服务状态、类型是否声明正确、以及 app-op 记录里 microphone 操作的时间线**。三条对不上，结论就站不住。

---

## 4. 隐私指示器与全局开关

### 4.1 AOSP 机制

AOSP 的隐私指示器文档（`source.android.com/docs/core/permissions/privacy-indicators`）给出三条可引用的机制：

- 「指示器在应用通过麦克风与相机的 **app-op 权限**使用私有数据源时出现。」
- 「在 Android 12 或更高版本，麦克风与相机的使用被认为是**活跃的**，如果系统将其标记为正在运行，或该使用发生在**五秒**之内。」
- 系统界面响应设备配置键 **`privacy/mic_camera_indicators_enabled`**。

注意两点：其一，该文档**没有**规定指示器的颜色（外界的「绿色圆点」说法来自用户界面层与厂商实现，不是 AOSP 文档口径）；其二，指示器依赖系统界面读取 `PermissionManager` 的数据，**系统界面自身不能因为读不到就崩溃**——这是文档里明确写的失败处理要求。

### 4.2 全局开关的语义

Android 12 引入相机与麦克风的**设备级开关**（部分文献称 sensor privacy）。它的语义是：开关关闭时，**所有**应用都取不到该传感器数据，无论权限是否已授予。

应用侧必须处理这条路径，推荐做法是：

1. 判断失败原因时区分「无权限」「硬件不可用」「被设备开关屏蔽」三类；
2. 向用户给出可操作的说明（「系统设置里关闭了麦克风访问」而不是「录音失败」）；
3. 不要在启动时静默重试——重试不会让开关自己打开。

### 4.3 可观测性不等于安全性

指示器解决的是**告知**，不是**阻止**。一个恶意应用完全可以在指示器亮起的几秒内取走一帧画面。**因此把「有指示器」当成「不会被采集」是错误的**；它的价值在于事后审计与用户干预的入口。

---

## 5. `MediaProjection`：一次会话一次同意

屏幕捕获的授权模型在 Android 14（API 34）发生了实质性变化。官方行为变更页的标题是 **“User consent required for each MediaProjection capture session”**，三段关键原文：

- 「Your app must ask the user to give consent before each capture session. A single capture session is a single invocation on `MediaProjection#createVirtualDisplay`, and each `MediaProjection` instance must be used only once.」
- 「For apps targeting Android 14 (API level 34) or higher, a `SecurityException` is thrown by `MediaProjection#createVirtualDisplay` in either of the following scenarios:」——场景一是缓存 `createScreenCaptureIntent()` 返回的 `Intent` 并多次传给 `getMediaProjection`；场景二是在同一个 `MediaProjection` 实例上多次调用 `createVirtualDisplay`。
- 「If your app doesn't register this callback, `MediaProjection#createVirtualDisplay` throws an `IllegalStateException` when your app invokes it.」——回调指 `MediaProjection.Callback#onStop`。

**这三条合起来改变了什么**：Android 14 之前，「拿到一次同意」等于「拿到一枚长期令牌」；之后，**令牌与一次 `createVirtualDisplay` 绑定**，且必须注册停止回调。旧代码里常见的「在 `onCreate` 里取一次 token，之后反复建虚拟显示」在 Android 14 上直接抛异常。

**配置变化怎么办？** 官方给了明确路径，不要重新申请同意：

1. 调用 `VirtualDisplay#resize` 传入新的宽高；
2. 用新尺寸创建 `Surface` 并通过 `VirtualDisplay#setSurface` 替换。

也就是说，**旋转屏幕不该触发第二次同意弹窗**——如果你看到应用每次旋转都要重新授权，那是实现错误，不是系统行为。

---

## 6. 屏幕捕获的隐私边界

屏幕捕获把「显示」变成「数据」，因此系统在三个维度上加限制：

1. **共享范围**：Android 14 起支持「应用级共享」——状态栏、导航栏、通知与其他系统界面元素被排除在共享内容之外。对做会议的团队，这意味着「共享整个屏幕」与「共享某个应用」在泄露面上差一个数量级。
2. **状态提示**：Android 15 引入媒体投影状态栏胶囊，提示正在共享/投屏/录制。需要精确引用的是**官方变更页的限定语**：「Media projection status bar chip alerts users to screen sharing, casting, and recording.」紧随其后的说明把生效范围限定在 **Android 15 QPR1 及之后的设备**——不是所有 Android 15 设备一升级就有。
3. **中断条件**：锁屏、用户主动停止、被捕获应用退出等都可能终止会话。应用必须实现 `onStop` 并释放 `VirtualDisplay` 与 `Surface`，否则会出现「黑屏但仍在采集」的假象。

**评审要点**：不要只问「有没有同意弹窗」，要问「同意的作用域是什么」。一次同意覆盖一次 `createVirtualDisplay`，**不覆盖后续的每一次显示**；而应用级共享与全屏共享是两种不同的泄露面。

---

## 6.5 被捕获方：不是所有窗口都愿意被录

从「被录」一侧看，边界落在三处：

- **安全窗口**：`FLAG_SECURE` 会让该窗口在屏幕捕获中显示为空白/黑块。它由**窗口所属应用**设置，不是系统对「敏感应用」的自动策略。
- **系统层保护**：部分系统界面（如锁屏密码输入、权限弹窗）会被系统排除在捕获之外，但这条依版本与厂商实现，不能当作通用保证。
- **应用级共享**：当共享方选择「仅共享某应用」时，其他应用的窗口根本不会进入合成。

因此「我开了 `FLAG_SECURE` 所以不会被录」只在**该窗口**成立；用户用相机对着屏幕拍，这条防线完全不存在。

---

## 7. 相机链路：从会话到 HAL

### 7.1 会话模型

Camera2 的核心是 `CameraDevice` + `CaptureSession` + `Surface`。安全上最重要的三点：

1. **打开相机就必然有 app-op 记录**，这是审计锚点。
2. **`openCamera` 在无权限时抛 `SecurityException`**，与被设备开关屏蔽的错误不同，必须分开处理。
3. **会话配置由应用决定**：分辨率、格式、是否带深度/元数据。**元数据（EXIF、时间戳、传感器参数）常被忽略**——一张「打了码的照片」仍可能带着拍摄时间、设备型号与方向信息。

### 7.2 HAL 与隐私灯

相机 HAL 在厂商侧实现，AOSP 只定义接口。硬件隐私灯（部分机型）通常在**供电或驱动层**控制，不由 HAL 的软件开关决定——这也是为什么「软件关了相机但灯亮」会成为硬件侧的问题报告点。**是否有隐私灯、是否由硬件控制，取决于机型**，不能作为通用结论。

### 7.3 外接与虚拟相机

外接 USB 相机、以及部分系统提供的虚拟相机（用于测试或视频通话替换），走的是不同的 provider。评审时要问一句：**这个 `CameraId` 背后是物理传感器还是软件源**？两者的取证价值完全不同。

---

## 8. 麦克风链路

### 8.1 采集

`AudioRecord` 与 `MediaRecorder` 都受 `RECORD_AUDIO` 与 app-op 双层控制。几个容易忽略的点：

- **采样率与声道**：不是所有设备支持任意组合，失败时应回退而不是假定「无权限」。
- **静音数据**：被全局开关屏蔽时，某些实现返回**全零数据而不是抛异常**——这会让应用误以为在正常录音。
- **音频焦点与通话**：通话期间采集行为受音频策略影响，不能假定「权限在手就能录」。

### 8.2 内录：`AudioPlaybackCapture`

Android 10 起提供的合法内录通道。它的关键约束在**被捕获方**：应用可以在自己的播放上声明「允许被捕获」，也可以禁止。因此：

- 内录**不是**「录下设备所有声音」的通用手段；
- 想挡住内录的应用，应在播放侧设置对应的使用标记；
- 评审「某应用能否录走别的应用的声音」时，答案是：**只有当被录方允许时才行**。

### 8.3 蓝牙音频：捕获面的延伸

通话与媒体音频可以被路由到蓝牙设备。**一旦音频路由到外部设备，本机的麦克风权限模型就不再覆盖这段链路**——它的安全性取决于配对与音频面 HAL（见 `android-wireless-nearby-stacks.md` 的 §3.2）。

---

## 9. 七种成因模式

1. **把权限当成开关**（有 `CAMERA` 就假定能开），忽略 app-op 与设备级开关。
2. **令牌寿命假设错误**（屏幕捕获 token 长期复用），在 Android 14 变成异常。
3. **缺少停止回调**，会话被系统终止后资源未释放。
4. **共享范围过大**（默认全屏共享，包含系统 UI 与通知）。
5. **元数据泄露**（图像 EXIF、文件名、时间戳）。
6. **静默失败**（全零音频、黑帧），被误判为「正常工作」。
7. **审计点选错**（只看权限授予历史，不看 app-op 时间线）。

---

## 10. 把捕获类 CVE 读对

### 10.1 公告里的落点

捕获相关缺陷在公告里通常不叫「Camera」或「Microphone」，而落在更宽的组件名下：

- **`Media framework`**：编解码与容器解析，历史上条目最多。2017 年形态的公告里，这一节实读到 `CVE-2017-0832` 至 `CVE-2017-0836` 这一组条目（当年表格只有 CVE / 类型 / 严重级，**没有 Component 列**）。
- **`System`**：系统服务与 HAL 交界处的问题。
- **厂商节**（Qualcomm / MediaTek / Unisoc / 闭源组件）：相机与音频的厂商实现绝大多数落在这里，**AOSP 公告里看不到细节**。

### 10.2 三条读表原则

1. **区分「解析」与「采集」**：Media framework 的绝大多数 CVE 是**解析不可信媒体文件**导致的，与「偷偷开相机」是两回事，处置优先级也不同。
2. **看受影响版本与补丁级别**：同一 CVE 在不同 Android 版本上的可达性可能不同，尤其涉及 HAL 的条目。
3. **区分 AOSP 公告与 Pixel 更新公告**：部分相机/音频修复只在后者出现。

---

## 11. 自查清单

### 11.1 应用侧

1. 是否区分处理 `SecurityException`（权限）、设备开关屏蔽（无数据/全零）与硬件不可用（打开失败）三类失败。
2. 屏幕捕获是否每次会话重新申请同意；是否注册了 `MediaProjection.Callback`。
3. 旋转/尺寸变化时是否走 `resize` + `setSurface`，而不是重建会话。
4. 是否默认全屏共享；能否改为应用级共享。
5. 采集到的图像是否剥离了不必要的 EXIF。
6. 前台服务类型是否与采集行为匹配（相机/麦克风/媒体投影各有类型）。
7. 是否在 `onStop` 后停止写盘并释放 `VirtualDisplay`、`Surface`、`AudioRecord`。

### 11.2 系统与设备侧

```bash
# 相机/麦克风的 app-op 历史（时间线审计的锚点）
adb shell dumpsys appops | grep -iE "CAMERA|RECORD_AUDIO" | head -40

# 当前前台服务及其类型
adb shell dumpsys activity services | grep -iE "foregroundServiceType|mediaProjection|camera|microphone" | head -30

# 设备级相机/麦克风开关状态（键名依版本，见未核实 D）
adb shell settings get secure mic_mute
adb shell settings get global camera_toggle_enabled 2>/dev/null

# 正在使用传感器的进程
adb shell dumpsys media.camera | head -30
adb shell dumpsys audio | grep -iA3 "recording" | head -30

# 屏幕捕获相关服务
adb shell dumpsys media_projection | head -30
```

### 11.3 企业侧

1. 用设备管理策略强制「禁止后台采集」与「禁止屏幕捕获」（依 EMM 能力）。
2. 对高敏应用启用 `FLAG_SECURE`，并明确它的边界（防截图/录屏，不防拍屏）。
3. 审计「相机/麦克风 app-op 历史」而不是只看权限授予。
4. 关注 Android 15 QPR1 之后的状态栏胶囊，作为「共享正在进行」的旁证。

---

## 12. 误传纠正

- **「有相机权限就能随时开相机」**：Android 9 起后台受限，Android 12 起还有设备级开关。
- **「关了麦克风开关，应用会收到异常」**：部分实现返回全零数据，应用可能完全无感。
- **「屏幕捕获只要同意一次就长期有效」**：Android 14 起一次同意对应一次 `createVirtualDisplay`，复用抛 `SecurityException`。
- **「没注册回调只是少了个通知」**：Android 14 起未注册 `MediaProjection.Callback` 会让 `createVirtualDisplay` 抛 `IllegalStateException`。
- **「旋转屏幕必须重新同意」**：官方给的是 `resize` + `setSurface`，不需要重新授权。
- **「绿色圆点是 AOSP 规定」**：AOSP 文档只描述指示器的触发条件与配置键，没有规定颜色。
- **「有了隐私指示器就不会被偷录」**：指示器是告知机制，不是阻断机制。
- **「`FLAG_SECURE` 能防拍照」**：它只影响该窗口在屏幕捕获中的呈现，防不住外部相机。
- **「`RECORD_AUDIO` 可以录设备播放的声音」**：内录需要系统级 `CAPTURE_AUDIO_OUTPUT` 或 `AudioPlaybackCapture`，且后者受被捕获方允许。
- **「应用级共享和全屏共享安全性一样」**：应用级共享排除系统 UI，泄露面小得多。
- **「Android 15 一升级就有共享胶囊」**：官方把该变更限定在 QPR1 及之后。
- **「Media framework 的 CVE 都是相机漏洞」**：绝大多数是解析不可信媒体文件的问题。

---

## 13. 与相邻手册的边界

- 生物识别与锁屏（`android-biometric-lockscreen.md`）：认证与密钥绑定；本文只讲采集侧。
- 崩溃诊断与日志暴露（`android-diagnostics-log-exposure.md`）：日志与 tombstone 的留存；本文的审计点在 app-op。
- 无线与近场（`android-wireless-nearby-stacks.md`）：蓝牙音频路由、HID 输入；本文在 §8.3 只做衔接。
- 变体设备（`android-variants-automotive-tv.md`）：车机与电视的相机/麦克风形态。
- 虚拟化与隔离（`android-virtualization-isolation.md`）：pVM 内的传感器访问属于那篇的范畴。

---

## 参考

- `https://developer.android.com/about/versions/14/behavior-changes-14`（“User consent required for each MediaProjection capture session”）
- `https://source.android.com/docs/core/permissions/privacy-indicators`
- `https://developer.android.com/about/versions/15/behavior-changes-all`（媒体投影状态栏胶囊，限定 Android 15 QPR1）
- `https://developer.android.com/about/versions/pie/android-9.0-changes-all`（后台访问用户输入与传感器的限制）
- `https://developer.android.com/reference/android/Manifest.permission`（`CAPTURE_AUDIO_OUTPUT`）
- `https://android.googlesource.com/platform/frameworks/base/+/master/core/java/android/permission/Permissions.md`
- `https://source.android.com/docs/security/bulletin/2017-11-01`（Media framework 节，`CVE-2017-0832`–`0836`）

---

## 未核实与失效来源

- **A. 指示器颜色。** 正文不采用「绿色圆点」作为事实：AOSP 文档未规定颜色，外部资料中的颜色描述属于用户界面实现。写作时只描述「状态栏指示器」。
- **B. 设备级开关的官方英文页。** 本轮抓取到的 AOSP 隐私指示器页为葡萄牙语渲染（段落完整），英文原句未逐字取得；`privacy/mic_camera_indicators_enabled` 键名按该页原文保留。
- **C. `CAPTURE_AUDIO_OUTPUT` 的确切 protectionLevel。** 官方参考页只给出用途描述；「签名/特权级」依据 AOSP `Permissions.md` 的口径转述，未取到该权限的逐字声明行。
- **D. 设备级开关的 Settings 键名。** 正文示例中的 `mic_mute`、`camera_toggle_enabled` 为常见键名，**依版本与厂商实现可能不同**，仅作探测示例，不作为断言。
- **E. `AudioPlaybackCapture` 的允许/禁止标记实名。** 正文只讲模型（被捕获方可允许或禁止），未逐字核验 `AudioAttributes` 侧的具体常量名。
- **F. 相机隐私灯的硬件控制范围。** 「多数机型由硬件/驱动层控制」为工程共识，未取得官方的逐机型说明。
- **G. Android 15 胶囊的完整限定语。** 官方变更页该句在检索摘要中被截断（`… included on devices running Android 15 QPR1 or …`），正文按「QPR1 及之后」表述，未取得完整句尾。
- **H. 前台服务类型与采集行为的强制映射。** 正文只说明「类型需与行为匹配」，未逐版本核验每种类型对应的权限检查细节。

---

## 联系方式

如果你在企业里做终端隐私或应用合规评审，需要就「相机/麦克风后台限制」「屏幕捕获授权」「捕获行为的可审计性」做一次针对现网设备的核查，可以通过 Telegram 联系：<https://t.me/one00190>

---

*本手册仅用于防御与教育目的，讨论的均为公开且已被修复问题的机制与设计缺陷，不包含任何可直接利用的代码。文中标注「未核实」的内容不应作为事实引用；发现错误欢迎指出。*
