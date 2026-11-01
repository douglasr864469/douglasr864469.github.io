---
layout: page
title: "Android 音频栈与媒体会话攻击面手册"
description: "把「麦克风开着」拆成四条链路：audioserver 里的 AudioFlinger/AudioPolicyService/AAudioService、共享内存而非 Binder 的 PCM 通路、HAL 从 HIDL 到 AIDL 的形态迁移、以及媒体会话这条独立控制面；另含 `RECORD_AUDI..."
date: 2026-11-01
permalink: /android-audio-media-session/
---

> 麦克风是手机上唯一一个「开着就一直在产生数据」的传感器：相机要你按下快门，位置要你移动，而麦克风每 10 毫秒就送来一帧。这条数据流在 Android 里穿过三层完全不同的代码——应用进程里的 `AudioRecord`、系统进程 `audioserver` 里的 `AudioFlinger`、以及厂商分区里的音频 HAL 与 DSP 固件。控制面还有第二条独立链路：媒体会话决定「谁在播、谁能控」。本手册只讨论公开且已修复问题的机制与自查方法，不含可利用代码。

---

## 0. 这篇手册解决什么问题

已发布的手册里，《摄像头、麦克风与屏幕捕获面》讲的是**授权时机**：谁能申请、什么时候要用户同意、状态栏怎么提示。本篇往下一层，讲**授权之后数据怎么流动**，以及一条几乎没人单独讲的链路：**媒体会话**。

三个问题：

1. **录到的声音经过谁？** 从 `AudioRecord` 到 HAL，中间有一个系统进程、一条共享内存、一个混音线程。
2. **别人能不能录到我放的声音？** 答案是「默认不能，但有一个应用可以自己开的门」——回放捕获。
3. **谁能控制我正在播的音乐？** 媒体会话是一个可被枚举、可被代理的系统对象，这条链路的权限模型比想象中松。

---

## 1. 架构：一个进程、两层策略、三条数据流

### 1.1 `audioserver`：音频的系统进程

Android 把音频核心放在一个独立的 native 进程 `audioserver` 里，它同时承载三个 Binder 服务：

| 服务名 | 实现 | 职责 |
|--------|------|------|
| `media.audio_flinger` | `AudioFlinger` | 混音、重采样、把数据写进 HAL；采集侧同理 |
| `media.audio_policy` | `AudioPolicyService` | 路由决策：走扬声器还是耳机、音量曲线、音频焦点 |
| `media.aaudio` | `AAudioService` | 低延迟直通路径（MMAP） |

它跑在 `audioserver` 这个 SELinux 域里，与 `system_server`、`media`、应用域都不同。这一点值得单独记住，因为「改音频策略」和「改应用权限」是两条独立的治理路径。

### 1.2 播放：共享内存 + 混音线程

```
应用进程                         audioserver                     vendor 进程
────────                        ───────────                     ───────────
AudioTrack
  └─ 共享内存环形缓冲 ─────────▶ PlaybackThread（每路一个）
  └─ write() 推进写指针            └─ 混音 / 重采样 / 音量
                                   └─ FastMixer（低延迟路径）
                                   └─ 写 HAL ────────────────▶ Audio HAL ──▶ DSP ──▶ 扬声器
```

关键性质：

- **音频数据不走 Binder**。Binder 只负责建立通道与传控制参数；PCM 数据在共享内存里，由指针同步。
- **混音必然发生**，除非走 Fast 路径。Fast 路径（`FastTrack`）绕过多路混音，延迟更低，但**可观测点更少**——这在下文的自查里是个实际障碍。
- **音量与路由在混音之后才应用**。所以「应用把音量设成最大」不等于「最终输出最大」，中间还有策略层。

### 1.3 采集：同一个进程的反方向

`AudioRecord` 走 `RecordThread`：HAL 把 PCM 写进共享缓冲，`RecordThread` 做重采样与格式转换，应用从缓冲里读。

三个容易被忽略的设计：

1. **多个客户端可以同时录音**。Android 不会因为 A 在录就拒绝 B。
2. **隐私敏感源会被「合并」**。当普通录音与语音通信同时存在时，系统会把它们路由到同一个输入源，避免出现「一方静默」这种暴露对方正在录制的副作用。这是有意的隐私设计，代价是**「谁在录」在音频数据层面不再可区分**。
3. **静音不等于断开**。系统的输入静音是通过写入静音数据实现的，HAL 与 DSP 仍然在工作。

### 1.4 HAL 的形态

音频 HAL 在厂商分区，形态随版本演进：

- HIDL：`android.hardware.audio@2.0` 到 `@7.x`，模块化后分 `primary`、`a2dp`、`usb`、`r_submix` 等 module。
- AIDL：Android 13 起提供 `android.hardware.audio.core` 形态（`IModule` 等接口）。`effect`（音效）在 AIDL 里是独立的包，与 `core` 分开。

这个演进的实际含义是：**越新的设备，音频处理越可能在 vendor 进程里以 AIDL 服务的形式存在**，而 AIDL 服务的注册与发现走 servicemanager，可用 `service list` 观察。

### 1.5 路由与焦点：策略层

`AudioPolicyService` 做两件事，必须分开理解：

- **路由**：根据 `AudioAttributes`（usage / content type）、已连接设备、通话状态，决定「从哪个设备出声、从哪个设备收音」。应用不能直接决定，只能「请求」。
- **焦点（audio focus）**：决定谁能「抢占」播放。请求焦点、获得、丢失、临时 ducking。焦点是**协作式**的——不申请焦点的应用照样能出声，系统不会阻止。

这就是「音频焦点不是权限」这句话的全部含义。

---

## 2. 攻击面：四条独立链路

### 2.1 采集授权：权限 + AppOps 双层

麦克风访问要同时满足两个条件：

1. `RECORD_AUDIO` 运行时权限；
2. `AppOpsManager` 的 `RECORD_AUDIO` 操作被允许。

两者分离的意义在于：撤销权限后，AppOps 记录仍然可以用于审计；反过来，某些系统路径（如助理）可以在没有运行时权限的情况下通过 AppOps 身份获得访问。**只看权限、不看 AppOps，会漏掉一整类访问。**

时间线上的关键收紧：

| 版本 | 收紧 |
|------|------|
| Android 9（API 28） | 后台应用（且无符合条件的前台服务）无法访问麦克风 |
| Android 11（API 30） | 麦克风权限支持「仅这一次」 |
| Android 12（API 31） | 状态栏麦克风指示器 + 设备级麦克风开关 |

### 2.2 回放捕获：默认关闭的一扇门

`AudioPlaybackCapture`（Android 10 起）允许一个应用录制**其他应用正在播放**的音频。它的模型是「被录方同意」：

- **采集方**需要 `MediaProjection` 授权，或者持有 `CAPTURE_AUDIO_OUTPUT`（签名/特权，普通应用拿不到）。
- **被录方**必须自己允许：`allowAudioPlaybackCapture`（清单/运行时），且音频的 `AudioAttributes` 必须是 `USAGE_MEDIA`、`USAGE_GAME` 或 `USAGE_UNKNOWN` 之一。
- **被录方还可以再收紧**：`AudioAttributes.Builder#setAllowedCapturePolicy()` 三档——允许所有、只允许系统、完全不允许。

三个常见误判：

- 「有 `RECORD_AUDIO` 就能录别人的播放」——错，`RECORD_AUDIO` 管的是**麦克风**，回放捕获走 `MediaProjection`。
- 「回放捕获默认可用」——错，被录方默认不允许。
- 「设成 `ALLOW_CAPTURE_BY_NONE` 就绝对录不到」——不完全对。系统级与特权路径仍在策略之外，只是普通应用被挡住。

### 2.3 通话音频：平台级限制而非权限问题

从 Android 10 起，平台限制了第三方应用获取电话通话音频的能力。这**不是**靠授予/不授予某个运行时权限实现的，而是采集路径本身对通话场景关闭。

因此「通话录音应用需要什么权限」这个问题的正确答案是：**在现行 Android 上，光有权限不够**。厂商实现、地区法规、特定系统应用身份都会影响结果，跨机型不可移植。企业做合规录音时，应当把它当作「需要系统集成」的需求，而不是「装个应用」的需求。

### 2.4 媒体会话：可被枚举的控制面

媒体会话（`MediaSession`）是应用向系统登记「我在播什么、怎么控」的对象。围绕它有四个组件：

```
MediaSession（播放方）
    │  token
    ▼
MediaSessionManager（system_server）
    │  getActiveSessions()
    ▼
MediaController（控制方）  ──▶  transport controls / metadata / queue
```

权限模型的实际形状：

- `MediaSessionManager.getActiveSessions()` 需要 `MEDIA_CONTENT_CONTROL`（签名/特权），**或者** 调用方是一个已被用户启用的 `NotificationListenerService`。
- 拿到 `MediaController` 之后，控制方可以发送播放、暂停、上一首、下一首、seek、设置音量等指令。
- **媒体按键**（耳机线控、蓝牙按键、锁屏控件）由 `system_server` 中的媒体会话服务统一派发，优先给「最后活跃」的会话。

这条设计的代价是：**「有通知访问权限」等价于「能控制所有正在播放的媒体」**。这是通知监听器权限的真实半径，很多合规清单只把它写成「读取通知」。

### 2.5 硬件与固件：常驻的那一层

低功耗语音唤醒（hotword）在部分机型上由 DSP/ADSP 常驻处理，主 CPU 可以休眠。这条路径的特点：

- 它**不经过** `audioserver` 的常规采集链路，因此在 `dumpsys media.audio_flinger` 里看不到。
- 它的固件通常闭源、随整包更新，修复周期由厂商决定。
- 「设备级麦克风开关关闭后，DSP 是否仍能采到声音」取决于厂商实现——**这一点无法从 AOSP 层确认**，必须逐机型用硬件方法验证。

---

## 3. 成因分析

### 3.1 采集是「持续态」，授权却是「一次性」

相机授权可以理解为「这次会话」，位置可以理解为「这段时间」。麦克风一旦授权，应用在符合条件的前台服务下可以长期采集。授权模型与数据模型的粒度不匹配，是所有麦克风类问题的共同根源。

### 3.2 三层策略各管一段，责任边界模糊

- 权限层管「能不能申请」；
- AppOps 层管「这次操作允不允许」；
- 策略层（`AudioPolicyService`）管「数据从哪来、去哪」；
- HAL/DSP 管「实际有没有采」。

四层里任何一层被绕过，上层都看不见。于是「麦克风指示灯亮着但我没在录」这类现象，往往要跨层排查才能定位。

### 3.3 隐私合并机制降低了可辨识度

第 1.3 节说的「敏感源合并」是为了避免暴露，但它同时让音频数据本身失去了「来源标记」。这在防御上是双刃：攻击者更难区分，取证者也更难区分。

### 3.4 回放捕获的同意权交给了被录方

被录方的 `allowAudioPlaybackCapture` 是应用自己声明的。这意味着**一个不设防的应用，就是所有回放捕获者的素材来源**。平台把责任下放，默认值因此变得极其重要。

### 3.5 媒体会话是全局命名空间

会话在系统里是全局可见对象（受权限约束）。任何获得通知访问权的应用，都能枚举并控制全部会话。这是便利性与最小权限之间的一个长期取舍。

### 3.6 HAL 与 DSP 闭源，修复不能自证

音频 HAL 在 vendor 分区，DSP 固件通常以二进制 blob 形式随整包更新。安全团队无法通过 AOSP 代码审计确认修复是否落地，只能依赖厂商公告与版本号比对。

---

## 4. 公开行为变更台账

| 版本 | 变更 | 影响 |
|------|------|------|
| Android 9（API 28） | 后台应用不能访问麦克风 | 采集必须绑定合规的前台服务 |
| Android 10（API 29） | `AudioPlaybackCapture` 引入；通话音频访问受限 | 回放捕获从「不可能」变成「需双方同意」 |
| Android 11（API 30） | 麦克风/相机/位置支持「仅这一次」 | 长期后台采集的默认成本上升 |
| Android 12（API 31） | 麦克风与相机指示器、设备级开关 | 采集对用户可见 |
| Android 13（API 33） | 音频 HAL 提供 AIDL 形态 | 观测点从 HIDL 服务转到 AIDL 服务 |

与输入子系统同理，本手册**不对音频相关 CVE 做逐条组件归属**：音频缺陷大量分布在厂商 HAL、DSP 固件与编解码库中，公告里的组件名（`Media Framework`、`Audio`）与具体补丁之间缺少稳定映射。

---

## 5. 防护技术

### 5.1 应用侧

- **自己播放的内容，主动声明不可捕获**：对敏感音频（会议、语音消息）设 `setAllowedCapturePolicy(ALLOW_CAPTURE_BY_NONE)`；对普通媒体保持默认。
- **自己采集的内容，最小化生命周期**：用完立刻 `release()`，不要靠进程存活维持会话。
- **录音前检查 AppOps 状态**，而不是只检查权限；被拒时给出明确提示而不是静默失败。
- **不要在锁屏或后台做「顺便录一下」**：既过不了平台限制，也会被用户判为恶意。

### 5.2 平台侧

- 把「麦克风 / 通知访问 / 悬浮窗」列为同一档高敏感授权，做统一的可视化与撤销入口。
- 对 `CAPTURE_AUDIO_OUTPUT` 的授予做白名单，不允许动态授予。
- 对 `MediaSessionManager` 的调用做审计日志：谁枚举了会话、枚举了几次。

### 5.3 厂商侧

- 音频 HAL 按「不可信参数」对待：采样率、通道数、缓冲区大小、位深都来自应用，全部要校验。
- 打开 HAL 的模糊测试，重点覆盖 `createStream`、`setParameters`、`openInputStream` 的异常组合。
- 明确并公开「设备级麦克风开关」对 DSP 常驻路径的实际效果。

### 5.4 企业侧

- MDM 基线里不要只写「禁用录音」，要写三条：`RECORD_AUDIO` 授权清单、通知访问授权清单、`CAPTURE_AUDIO_OUTPUT` 持有者清单。
- 会议类应用统一配置 `ALLOW_CAPTURE_BY_NONE`，并按季度复核。

---

## 6. 自查与观测清单

```bash
# 1. 音频服务与 HAL
dumpsys media.audio_flinger | head -60      # 打开的设备、线程、采样率
dumpsys media.audio_policy                  # 路由、策略、已连接设备
service list | grep -i audio                # HIDL/AIDL 服务注册情况

# 2. 谁在用麦克风
cmd appops get <package> RECORD_AUDIO
dumpsys audio | grep -i "record\|active"    # 当前活跃的录音客户端
dumpsys media.audio_flinger | grep -i "record thread" -A 5

# 3. 回放捕获相关
cmd appops get <package> PROJECT_MEDIA      # MediaProjection 授权
dumpsys media_projection

# 4. 媒体会话
dumpsys media_session                       # 活跃会话、控制者、播放状态
dumpsys media_session | grep -i "package\|state"

# 5. 通知访问（等价于媒体控制权）
settings get secure enabled_notification_listeners

# 6. 设备级开关
settings get global mic_toggle_enabled      # 部分版本可用，未核实
cmd sensor_privacy get MICROPHONE           # 部分版本可用，未核实
```

判读要点：

- `dumpsys media.audio_flinger` 里如果出现预期之外的 `Record Thread`，说明有客户端正在采集。
- `dumpsys media_session` 的「控制者」列表如果包含你不认识的应用，先查它的通知访问权。
- HIDL 音频服务在较新设备上可能完全不存在（被 AIDL 取代），「查不到」不等于「没有音频 HAL」。

---

## 7. 常见误传纠正

1. **「`audioserver` 是厂商进程」**——错。它是 AOSP 的系统进程，SELinux 域为 `audioserver`。
2. **「`AudioFlinger` 跑在应用进程里」**——错。它在 `audioserver`。
3. **「音频数据通过 Binder 传输」**——错。Binder 只传控制面，PCM 走共享内存。
4. **「有 `RECORD_AUDIO` 就能后台一直录」**——Android 9 起后台受限，还需要合规的前台服务。
5. **「有 `RECORD_AUDIO` 就能录别的应用播放的声音」**——错。回放捕获走 `MediaProjection`，与麦克风权限无关。
6. **「回放捕获默认开启」**——错。被录方默认不允许。
7. **「`setAllowedCapturePolicy(NONE)` 能挡住系统」**——不能。它约束的是普通应用路径。
8. **「静音模式等于不采集」**——错。静音影响输出，与输入采集无关。
9. **「关掉麦克风开关，DSP 一定采不到」**——未核实。取决于厂商实现。
10. **「媒体会话只有拥有者能控制」**——错。持有 `MEDIA_CONTENT_CONTROL` 或已启用的通知监听器都能拿到 `MediaController`。
11. **「音频焦点是权限」**——错。它是协作机制，不申请焦点的应用照样能出声。
12. **「音量控制受权限保护」**——`MODIFY_AUDIO_SETTINGS` 是 normal 级权限，装机即得，不能当作安全边界。
13. **「多个应用不能同时录音」**——错。可以，且隐私敏感源会被合并。

---

## 未核实与失效来源

- 设备级麦克风开关对 DSP 常驻唤醒路径的实际效果：未核实，需逐机型硬件验证。
- AIDL 音频 HAL 与 HIDL `android.hardware.audio@7.x` 的完整版本对应关系：只核实了「Android 13 起提供 AIDL 形态」，逐版本映射未核实。
- `mic_toggle_enabled`、`cmd sensor_privacy` 等命令在不同 Android 版本上的可用性：未核实。
- 音频相关 CVE 的组件归属：本手册有意不做逐条归属，理由见第 4 节。

---

## 参考

- 音频框架概览：[source.android.com/docs/core/audio](https://source.android.com/docs/core/audio)
- 音频 HAL：[source.android.com/docs/core/audio/implement](https://source.android.com/docs/core/audio/implement)
- `AudioPlaybackCapture`：[developer.android.com/media/platform/av-capture](https://developer.android.com/media/platform/av-capture)
- `AudioAttributes#setAllowedCapturePolicy`：[developer.android.com/reference/android/media/AudioAttributes.Builder](https://developer.android.com/reference/android/media/AudioAttributes.Builder)
- `MediaSessionManager`：[developer.android.com/reference/android/media/session/MediaSessionManager](https://developer.android.com/reference/android/media/session/MediaSessionManager)
- 后台麦克风限制：[developer.android.com/about/versions/pie/android-9.0-changes](https://developer.android.com/about/versions/pie/android-9.0-changes)
- Android 12 行为变更（麦克风指示器）：[developer.android.com/about/versions/12/behavior-changes-12](https://developer.android.com/about/versions/12/behavior-changes-12)
- `AudioFlinger`：[android.googlesource.com/platform/frameworks/av/+/refs/heads/main/services/audioflinger](https://android.googlesource.com/platform/frameworks/av/+/refs/heads/main/services/audioflinger)
- `AudioPolicyService`：[android.googlesource.com/platform/frameworks/av/+/refs/heads/main/services/audiopolicy](https://android.googlesource.com/platform/frameworks/av/+/refs/heads/main/services/audiopolicy)

---

## 联系方式

如果你在企业里做终端隐私合规，需要就「麦克风访问审计」「回放捕获是否被误开」「通知访问与媒体控制权半径」做一次现网核查，可以通过 Telegram 联系：<https://t.me/one00190>

---

*本手册仅用于防御与教育目的，讨论的均为公开且已被修复问题的机制与设计缺陷，不包含任何可直接利用的代码。文中标注「未核实」的内容不应作为事实引用；发现错误欢迎指出。*
