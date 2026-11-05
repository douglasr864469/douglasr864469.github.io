---
layout: page
title: "Android MediaProjection、屏幕录制与投屏通路攻击面手册"
description: "把「录屏」拆成意图／系统同意／框架服务／显示缓冲四层：`createScreenCaptureIntent` 只生成请求不授权、同意界面在 SystemUI 的 `MediaProjectionPermissionActivity` 而非某个「MediaProjectionService」、会话与令牌由..."
date: 2026-11-05
permalink: /android-media-projection-screen-capture/
---

> 屏幕录制看起来只是「把画面复制一份」，但在 Android 上，它是少数需要**系统 UI 亲自签发令牌**的高敏感能力：应用拿到 `MediaProjection` 之后，可以把整个屏幕（或单个应用窗口）写进自己创建的 `VirtualDisplay`，再通过 `Surface` 取走像素。授权对话框、令牌生命周期、虚拟显示与安全图层之间任何一处判断失误，都会把「用户当场同意的一次录屏」变成「长期、静默、跨应用的画面采集」。这份手册沿「谁能录 → 令牌怎么签发 → 画面怎么流 → 哪些边界必须成立 → 出了哪些公开缺陷 → 怎么防」拆解这条通路。只讨论公开且已修复的问题，不含利用代码。

## 0. 使用方式与边界

这份手册面向三类读者：

- **应用开发者**：需要区分「申请权限」「取得用户同意」「创建虚拟显示」三件不同的事，并理解 Android 14 之后令牌为何是一次性的。
- **系统与 ROM 工程师**：需要知道同意界面由 SystemUI 的哪个组件承担、`MediaProjectionManagerService` 在其中做什么、设备策略在哪一层拦截。
- **安全评估者**：需要在看到「录屏/投屏」类公告时，判断问题落在同意界面、令牌生命周期、虚拟显示参数、设备策略还是多用户状态机。

三条自我约束：

1. 只引用公开且已修复的问题；CVE 描述以 NVD 与 Android Security Bulletin 为准。
2. 不写利用步骤、不写 PoC、不写可直接复现的参数组合。
3. 查不到一手来源的说法一律放进「未核实与失效来源」。

与已发布手册的分工：

- 《Android 媒体捕获、麦克风与相机隐私手册》讲相机与麦克风的采集边界；本文只讲**屏幕**这一路。
- 《Android 图形与显示栈攻击面手册》讲 SurfaceFlinger、HWC 与图层合成；本文只讲「把合成结果导向应用 Surface」的投影通路。
- 《Android 窗口管理、悬浮窗与界面覆盖手册》讲 overlay 与 tapjacking；本文只把「覆盖同意界面」作为历史缺陷的一种成因。

## 1. 对象模型：四层而不是一层

很多资料把「录屏」简化成一个 API，实际上它跨了四层：

| 层 | 代表对象 | 职责 |
|----|----------|------|
| 用户意图层 | `MediaProjectionManager.createScreenCaptureIntent()` | 生成一个「去要授权」的 Intent，本身不授权 |
| 系统同意层 | SystemUI `MediaProjectionPermissionActivity` | 展示系统对话框，用户点「开始」后签发令牌 |
| 框架服务层 | `MediaProjectionManagerService`（system_server） | 校验令牌、绑定进程、管理会话生命周期 |
| 显示与缓冲层 | `VirtualDisplay`、`Surface`、`ImageReader`、SurfaceFlinger | 创建虚拟屏幕，把合成结果写入应用提供的缓冲 |

调用链的骨架是：

1. 应用调用 `createScreenCaptureIntent()`；
2. 系统弹出同意界面（由 SystemUI 承担，不是应用自己画）；
3. 用户同意后，`onActivityResult` 返回一个 `Intent`，应用把它交给 `getMediaProjection(resultCode, data)`；
4. 得到 `MediaProjection` 后调用 `createVirtualDisplay(...)`，传入宽高、DPI、标志位和一个 `Surface`；
5. SurfaceFlinger 把目标图层合成到这块虚拟屏幕上，像素进入应用提供的缓冲（常见是 `ImageReader.getSurface()`）。

这里有两个容易记错的点：

- **同意界面属于 SystemUI**。AOSP 现行实现是 `com.android.systemui.mediaprojection.permission.MediaProjectionPermissionActivity`。并不存在一个负责弹窗的「MediaProjectionService」。
- **`MediaProjectionManager` 是客户端代理**。真正持有会话状态、校验调用者的是 system_server 里的 `MediaProjectionManagerService`。

## 2. 授权流程：令牌才是核心

### 2.1 返回的不是布尔值，是令牌

`onActivityResult` 里拿到的 `Intent` 携带的是一次性授权凭据。应用把它交给 `getMediaProjection()` 时，框架会核对调用者的 uid、会话状态与令牌有效性。**「用户点过同意」与「应用现在有权录屏」之间隔着这次核对**，这也是历史上多个缺陷的落点。

### 2.2 Android 14 的收紧

从 Android 14（API 34）开始，规则明显变严：

- targetSdk 34+ 的应用必须声明 `FOREGROUND_SERVICE` 与 `FOREGROUND_SERVICE_MEDIA_PROJECTION`，并以 `foregroundServiceType="mediaProjection"` 启动前台服务；
- **每次会话都要重新授权**：令牌只对一次 `createVirtualDisplay` 有效，复用同意 Intent 或重复创建会抛 `SecurityException`；
- 同意界面提供「单个应用」与「整个屏幕」两种范围；
- 应用崩溃、用户停止、会话结束都会使令牌失效。

这套设计针对的正是「一次同意、长期后台采集」的滥用模式。它不能阻止一个已经获得同意的应用在会话期间采集，但显著缩短了授权的时间窗。

### 2.3 Android 15 的可见性

Android 15 QPR1 及之后引入状态栏投屏提示、锁屏时自动停止、投屏期间隐藏通知内容与密码输入等行为。这些属于**用户可见性**加固：让「正在被录」从隐式状态变成显式状态。具体到某个 OEM 是否全部落地，需按机型核实。

## 3. 权限与策略：哪些是真权限

屏幕投影相关的权限经常被混写，下面几条按 AOSP 清单核对过：

| 名称 | 保护级别 | 作用 |
|------|----------|------|
| `FOREGROUND_SERVICE_MEDIA_PROJECTION` | normal / instant | 允许启动 mediaProjection 类型的前台服务 |
| `CAPTURE_VIDEO_OUTPUT` | signature | 直接捕获视频输出，普通应用不可持有 |
| `CAPTURE_AUDIO_OUTPUT` | signature / privileged / role | 捕获音频输出 |
| `CAPTURE_MEDIA_OUTPUT` | signature / privileged / role | 捕获媒体输出 |
| `MANAGE_MEDIA_PROJECTION` | signature | 管理他人投影会话 |
| `DETECT_SCREEN_CAPTURE` | normal（API 34） | 感知「本窗口正被截屏」 |
| `DETECT_SCREEN_RECORDING` | normal（API 34） | 感知「本窗口正被录制」 |

设备策略侧：

- `DevicePolicyManager.setScreenCaptureDisabled()` / `getScreenCaptureDisabled()`；
- 策略常量 `POLICY_DISABLE_SCREEN_CAPTURE`；
- AOSP 现行代码还要求调用方持有 `MANAGE_DEVICE_POLICY_SCREEN_CAPTURE`，因此真正能下发该策略的是设备所有者或资料所有者，而不是任意 DPC。

需要强调的是：**禁用截屏策略不是「让录屏应用报错」这么简单**，它要求系统在合成与投影两层都不把受保护内容送出去。策略是否真正生效，取决于 SystemUI、框架服务与显示管线的配合。

## 4. 画面数据路径：安全图层与黑屏

投影的目标是 `Surface`，但并非所有图层都会进入这块 Surface。理解以下三个概念，才能判断「录到了什么」：

### 4.1 安全图层（secure layer）

带有 `FLAG_SECURE` 的窗口（`WindowManager.LayoutParams.FLAG_SECURE`）在投影中显示为黑屏。这是**平台级**保证，不依赖录屏应用自觉。支付、密码、部分 DRM 画面都使用这一标志。

### 4.2 受保护内容与 DRM

DRM 视频可能使用安全缓冲与安全显示通路，是否可在投影中出现由内容许可与设备能力共同决定。把「DRM 视频录不到」当成绝对结论是不准确的：不同 DRM 级别、不同输出保护配置下行为不同。

### 4.3 单应用投影

Android 14 起用户可以选择只共享某一个应用。框架会为该会话限定图层范围。实现上，这要求系统正确识别「目标应用的任务与窗口」，一旦任务栈、分屏或画中画状态判断错误，就可能出现「选了 A 却录到 B」或「该录的没录到」。这类问题通常表现为**范围判断逻辑缺陷**，而不是权限绕过。

## 5. 音频：另一条独立授权链

屏幕录制常伴随音频采集，但音频走的是另一套机制：

- 应用自身音频可用 `AudioPlaybackCaptureConfiguration` 采集，前提是目标应用允许被捕获（`allowAudioPlaybackCapture`）；
- 系统级音频捕获需要 `CAPTURE_AUDIO_OUTPUT`，普通应用不可持有；
- 麦克风采集属于 `RECORD_AUDIO`，与屏幕投影无关。

把「录屏授权」当成「同时获得麦克风与系统音频授权」是常见误解。三者是三条独立的授权链。

## 6. 公开公告台账

| CVE | 落点 / 类型 | 级别 | 修复 | 说明 |
|-----|-------------|------|------|------|
| [CVE-2015-3878](https://nvd.nist.gov/vuln/detail/CVE-2015-3878) | Media Projection 信息泄露 | 公告 Moderate | 2015-10 | 早期录屏授权提示可被遮挡或干扰 |
| [CVE-2018-9524](https://nvd.nist.gov/vuln/detail/CVE-2018-9524) | System UI 覆盖窗口保护不足（提权） | 公告 High | 2018-11 | 公告归类 System UI；与录屏同意界面的关联见第三方分析 |
| [CVE-2023-21350](https://nvd.nist.gov/vuln/detail/CVE-2023-21350) | System 信息泄露（投影侧信道可推断应用是否安装） | 公告 Moderate | 2023-10 | Android 14 公告批次 |
| [CVE-2024-0029](https://nvd.nist.gov/vuln/detail/CVE-2024-0029) | Framework 提权（多用户状态下屏录禁用策略被绕过） | 公告 High | 2024-02 | 设备策略绕过 |
| [CVE-2025-32322](https://nvd.nist.gov/vuln/detail/CVE-2025-32322) | System（`MediaProjectionPermissionActivity` 输入校验不当，未授权签发令牌） | NVD 7.8 / 公告 High | 2025-09（Wear 公告） | 影响 Android 13/14 |

读表要点：

1. **落点分散**：同意界面、令牌签发、策略状态、侧信道各有一例，说明这条通路的安全不取决于单点。
2. **「信息泄露」不一定是画面泄露**：CVE-2023-21350 泄露的是「某应用是否安装」这类推断信息。
3. **设备策略也会被绕过**：CVE-2024-0029 提醒我们，`setScreenCaptureDisabled` 的生效依赖多用户状态判断正确。

## 7. 成因模式

### 7.1 同意界面被遮挡或误触

早期同意流程依赖一个普通对话框，如果攻击者能用悬浮窗覆盖或诱导点击，就可能让用户在不知情下授权（CVE-2015-3878、CVE-2018-9524）。修复方向包括：提高同意界面的图层等级、限制覆盖、Android 12 起对不受信任的遮挡降低触摸透传。

### 7.2 令牌生命周期过长或可复用

「一次同意、永久有效」会放大滥用后果。Android 14 改成单次令牌，正是把生命周期问题从「应用自觉」提升为「框架强制」。

### 7.3 令牌签发前校验不足

当同意 Activity 解析外部传入的 Intent 时，如果未校验调用者、附加字段或结果码，就可能在没有真实用户操作的情况下签发令牌（CVE-2025-32322）。这类缺陷的危险在于它**绕过了唯一的用户交互点**。

### 7.4 多用户与资料状态判断

设备策略、用户切换、工作资料会让「当前是否允许截屏」变成一个状态机问题。只在一个用户路径上检查、切换后不重算，就会出现策略绕过（CVE-2024-0029）。

### 7.5 范围判断与任务栈

单应用投影依赖对任务、窗口、分屏状态的正确解析。范围过宽是隐私问题，范围过窄是可用性问题，两者都源于「我以为这个 Surface 属于谁」。

### 7.6 侧信道

即使没有画面泄露，录屏相关 API 的时序、回调、错误码也可能泄露信息（CVE-2023-21350）。防御上要避免让「是否安装某应用」这类状态反映到可观测的差异上。

## 8. 版本加固时间线

| 版本 / API | 变化 |
|------------|------|
| 5.0 / 21 | MediaProjection 引入，需用户同意 |
| 10 / 29 | `READ_FRAME_BUFFER`、`CAPTURE_VIDEO_OUTPUT` 收紧为 signature |
| 12 / 31 | 不可信 overlay 的触摸透传受限，缓解覆盖点击 |
| 14 / 34 | 单次令牌；每次会话重新授权；前台服务类型；单应用/全屏范围；`DETECT_SCREEN_*` |
| 15 / 35 | 状态栏投屏提示、锁屏自动停止等可见性加固（QPR1 起逐步落地） |

趋势是**把「一次宽授权」拆成「每次窄授权」**，并让用户随时看得见、停得掉。

## 9. 观测点与自查

### 9.1 应用侧

- 是否注册 `MediaProjection.Callback` 并处理 `onStop`？停止后是否释放 `VirtualDisplay`、`Surface`、`ImageReader`？
- 是否缓存同意 Intent 或跨会话复用令牌？targetSdk 34+ 下这会直接失败。
- 前台服务是否声明了正确的 `foregroundServiceType`？是否在会话结束时停止服务？
- 敏感界面是否使用 `FLAG_SECURE` 或 `setContentSensitivity`？
- 是否优先使用单应用共享，而不是默认全屏？

### 9.2 平台与 ROM 侧

- 同意界面是否由 SystemUI 强制展示，应用能否以任何方式跳过？
- 令牌是否与 uid、进程、会话一一绑定？进程死亡后是否失效？
- 设备策略禁用截屏后，虚拟显示创建、画面合成、投屏输出三条路径是否都拦截？
- 用户切换、资料切换、锁屏后，会话是否重新评估？
- 是否存在可被应用观测的「是否安装某应用」差异？

### 9.3 评估侧

- 先定位落点：同意界面 / 令牌签发 / 会话管理 / 显示管线 / 设备策略 / 侧信道；
- 区分「画面泄露」「状态推断」「权限提升」「拒绝服务」；
- 不要把所有录屏问题都归为「悬浮窗攻击」——Android 12 之后这条路径已大幅收窄。

## 10. 防御要点

1. **同意必须来自真实用户操作**：令牌签发前校验调用者、结果码与 Intent 来源，禁止从后台或服务直接触发。
2. **令牌单次、短命**：一次会话一授权，会话结束立即失效；不要把令牌写进持久存储。
3. **最小范围**：默认单应用，非必要不请求全屏；向用户明确说明采集范围。
4. **安全窗口默认开启**：密码、支付、私密内容一律 `FLAG_SECURE`；不要依赖「用户不会录」。
5. **策略与状态联动**：设备策略、用户切换、锁屏状态变化时重新计算是否允许投影。
6. **可见性**：提供持续的系统级提示，让用户随时知道「正在被录」，并能一键停止。
7. **及时打补丁**：关注 2024-02、2025-09 等批次的屏录相关条目，尤其是 OEM 定制同意界面的机型。

## 11. 常见误传纠正

### 误传一：有 `SYSTEM_ALERT_WINDOW` 就能静默录屏

错。悬浮窗只能画 UI，不能签发令牌；令牌只能由系统同意流程产生。Android 12 起对不可信遮挡还限制了触摸透传。

### 误传二：用户同意一次就永久有效

错。Android 14 起令牌按会话单次有效，复用同意 Intent 会抛 `SecurityException`。

### 误传三：声明了前台服务权限就能录屏

错。`FOREGROUND_SERVICE_MEDIA_PROJECTION` 只是安装期 normal 权限，仍需要用户同意与正确的服务类型。

### 误传四：`FLAG_SECURE` 只防截图，不防录屏

错。安全窗口在投影中同样显示为黑屏，这是平台级行为。

### 误传五：录屏授权等于麦克风授权

错。屏幕、系统音频、麦克风是三条独立授权链，权限与同意流程都不同。

## 12. 未核实与失效来源

- 「Android 12 引入 MediaProjection 令牌失效机制」的说法未在官方行为变更页找到对应条目，本文不采纳。
- Android 15 状态栏投屏提示与锁屏停止的具体 QPR 版本，各 OEM 落地时间不一致，未逐一核实。
- DRM 内容在不同安全级别下的投影行为差异较大，本文只给方向性描述。
- 各 OEM 定制同意界面的实现与补丁节奏未核实，CVE-2025-32322 的覆盖范围以公告为准。

## 13. 参考资料

- MediaProjection 官方指南：
  https://developer.android.com/media/grow/media-projection
- Android 14 行为变更：
  https://developer.android.com/about/versions/14/behavior-changes-14
- Android 15 行为变更：
  https://developer.android.com/about/versions/15/behavior-changes-all
- Android 10 隐私变更：
  https://developer.android.com/about/versions/10/privacy/changes
- SystemUI 同意界面（AOSP）：
  https://android.googlesource.com/platform/frameworks/base/+/refs/heads/main/packages/SystemUI/src/com/android/systemui/mediaprojection/permission/MediaProjectionPermissionActivity.java
- 权限清单（AOSP）：
  https://android.googlesource.com/platform/frameworks/base/+/refs/heads/main/core/res/AndroidManifest.xml
- DevicePolicyManager（AOSP）：
  https://android.googlesource.com/platform/frameworks/base/+/refs/heads/main/core/java/android/app/admin/DevicePolicyManager.java
- 2015-10 公告：
  https://source.android.com/docs/security/bulletin/2015-10-01
- 2018-11 公告：
  https://source.android.com/docs/security/bulletin/2018-11-01
- Android 14 公告：
  https://source.android.com/docs/security/bulletin/android-14
- 2024-02 公告：
  https://source.android.com/docs/security/bulletin/2024-02-01
- Wear 2025-09 公告：
  https://source.android.com/docs/security/bulletin/wear/2025/2025-09-01
- NVD 条目（本文台账）：
  https://nvd.nist.gov/vuln/detail/CVE-2015-3878
  https://nvd.nist.gov/vuln/detail/CVE-2018-9524
  https://nvd.nist.gov/vuln/detail/CVE-2023-21350
  https://nvd.nist.gov/vuln/detail/CVE-2024-0029
  https://nvd.nist.gov/vuln/detail/CVE-2025-32322

---

## 联系方式

咨询 Android 系统详情请联系 https://t.me/one00190

---

*本手册仅供安全学习与防御研究使用；所有机制表述、公告列值与 CVE 描述均取自公开一手来源并附链接，未能核实的部分已单列于「未核实与失效来源」而非写入正文。*
