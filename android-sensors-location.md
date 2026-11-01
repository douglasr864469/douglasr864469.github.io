---
layout: page
title: "Android 传感器与定位攻击面手册"
description: "把「设备知道你在哪」拆成两条互不相同的通路：免权限 motion 传感器经 BitTube 队列与 `SensorDirectChannel` 旁路上报、定位经 LocationManagerService 与 GNSS HAL；另含权限/精度/频率三个独立维度、Android 12 的 200 Hz ..."
date: 2026-11-01
permalink: /android-sensors-location/
---

> 传感器是「推断」的原料，定位是「结论」。加速度计的一组读数可以推断出你在走路、在打字、甚至正在输入什么；GNSS 的一串伪距可以直接变成坐标。这条链路的特别之处在于：**大多数传感器不需要任何权限就能读**，而定位虽然是高强度权限，它的精度分档、后台限制、模拟位置又各有独立的判定逻辑。本手册只讨论公开且已修复问题的机制与自查方法，不含可利用代码。

---

## 0. 这篇手册解决什么问题

已发布的手册里，《摄像头、麦克风与屏幕捕获面》讲的是「看」与「听」，《多用户、工作资料与进程隔离》讲的是「谁能看到谁的数据」。本篇讲**「感知」**——设备怎么知道自己在哪里、在怎么动，以及这些结论能被谁读走、被谁改写。

三个必须先分清的概念：

1. **权限**：能不能访问（`BODY_SENSORS`、`ACCESS_FINE_LOCATION`）。
2. **精度**：给多准（精确 / 大概）。
3. **频率**：给多快（`HIGH_SAMPLING_RATE_SENSORS` 决定 200 Hz 这条线）。

三者互相独立。很多「传感器隐私」的讨论把三者混为一谈，于是得出「没有权限就不能推断」这种错误结论——**加速度计恰恰是不需要权限的**。

---

## 1. 架构：一个服务、一条队列、一条低延迟旁路

### 1.1 `sensorservice`：system_server 里的传感器中枢

`SensorService` 是运行在 **system_server** 进程内的系统服务，向上提供 `SensorManager`，向下通过 `SensorDevice` 与厂商的传感器 HAL 通信。

```
应用进程              system_server                    vendor 进程
────────              ─────────────                    ───────────
SensorManager
  └─ SensorEventQueue ──(BitTube/socket)──▶ SensorService      SensorDevice
  └─ SensorDirectChannel ─(共享内存)────────▶  └──────────────▶ ISensors (HAL)
                                                                    └─ 传感器驱动 / DSP
```

三个关键点：

- **事件不走 Binder**。`SensorEventQueue` 用 `BitTube`（本质是 socketpair）接收事件；Binder 只用于 `registerListener` / `unregisterListener` 这类控制调用。
- **HAL 在 vendor 进程**。HIDL 形态是 `android.hardware.sensors@2.0` / `@2.1`；较新版本提供 AIDL 形态（具体起始版本未核实）。这一层的代码由厂商维护。
- **`SensorDirectChannel` 是旁路**。API 26 起，应用可以申请一块共享内存（`HardwareBuffer` 或 ashmem），让 HAL 直接写入采样数据，绕过 `SensorService` 的常规事件队列。延迟更低，但**可观测点更少**。

### 1.2 传感器的分类维度

`sensorType` 只是第一层。真正影响风险的还有两个维度：

| 维度 | 取值 | 含义 |
|------|------|------|
| 是否唤醒 | wake-up / non-wake-up | wake-up 传感器在 AP 休眠时也能触发唤醒，功耗与隐私影响都更大 |
| 上报模式 | continuous / on-change / one-shot / special | one-shot 用于「一次性事件」，special 用于「步数」这类由 HAL 聚合的结果 |

把「步数」当原始数据用是个常见错误：`TYPE_STEP_COUNTER` 的数值是 HAL 或 DSP 聚合后的结果，不同厂商的判定规则不同，跨设备不可比。

### 1.3 采样率：200 Hz 这条线

Android 12 引入了 `HIGH_SAMPLING_RATE_SENSORS`（normal 级权限）。没有它时，应用的采样率被限制在 **200 Hz**。

这条限制的意义需要准确理解：它限制的是**应用请求的速率**，不是传感器硬件的速率。HAL 与 DSP 仍可能以更高频率内部采样并做聚合。所以「限到 200 Hz 就推断不出按键」这种说法是过度乐观的——200 Hz 对多数侧信道研究来说已经足够。

### 1.4 定位：一个服务、多个 provider

```
应用
 └─ LocationManager（Java）
      └─ LocationManagerService（system_server）
           ├─ gps provider      ──▶ GNSS HAL（vendor）──▶ 基带 / GNSS 芯片
           ├─ network provider  ──▶ 网络定位（可能需要第三方服务）
           ├─ fused provider    ──▶ 融合结果（系统或 Play 服务提供）
           └─ passive provider  ──▶ 只转发别人的定位结果，不主动发起
```

`passive` 是一个长期被低估的通道：它不主动定位，但**只要系统里有人拿到了位置，它就能收到同一份结果**。它仍然受位置权限约束，但不产生新的定位行为，因此在审计「谁在定位」时容易被漏掉。

GNSS HAL 的形态：HIDL `android.hardware.gnss@2.0` / `@2.1` 是长期形态，AIDL 形态在较新版本提供。往上是 `GnssStatus.Callback`、`GnssMeasurementsEvent.Callback`（原始测量）、`GnssNavigationMessage.Callback`（导航电文）、`GnssAntennaInfo`（API 30 起）。

### 1.5 精度分档的三种实现位置

「大概位置」不是一个 UI 概念，它必须在数据层实现。三种可能的实现位置，效果完全不同：

1. **降采样**：只给低频位置更新。
2. **模糊化（fuzzing）**：给一个被扰动的坐标，官方口径是约 2 公里量级。
3. **网格化**：吸附到固定网格。

现行实现以模糊化为主，但**具体算法与参数跨版本、跨厂商会变**，不能把「2 公里」当作可依赖的精确值。

### 1.6 模拟位置

`Location.isMock()` 是应用侧唯一可靠的判别手段。它的开关在开发者选项里（「选择模拟位置信息应用」），底层由 `AppOps` 的 `MOCK_LOCATION` 操作控制。

这意味着：**mock 不是靠运行时权限挡住的，而是靠 AppOps + 开发者选项**。在受管设备上，「开发者选项是否打开」直接决定了这条通路是否存在。

---

## 2. 攻击面：四类互不相同的风险

### 2.1 免权限传感器：侧信道的主战场

大多数 motion 传感器（加速度计、陀螺仪、磁力计）**不需要任何运行时权限**。这是设计选择：它们被视为「低敏感」。而公开研究反复证明，高频加速度计数据可以用于推断：

- 输入内容（通过打字时的微振动）；
- 设备握持与移动模式（用于用户画像）；
- 环境特征（交通方式、是否在电梯里）。

Android 12 的 200 Hz 限制是对这类研究的直接回应，但它降低了门槛，没有消除可能性。

### 2.2 传感器与位置的耦合

磁力计可以给出朝向，气压计可以给出高度变化，Wi-Fi 扫描结果可以给出位置（在 Android 上，Wi-Fi 扫描受位置权限约束正是为此）。所以「我没给定位权限」并不等于「应用推断不出我在哪」——它只意味着「拿不到系统给的标准位置对象」。

这条推论在合规评审中经常被忽略：**位置权限管的是 `Location` 对象，不是「关于位置的一切信息」**。

### 2.3 后台定位与前台服务类型

时间线上的收紧：

| 版本 | 变化 |
|------|------|
| Android 10（API 29） | 后台位置需要单独的 `ACCESS_BACKGROUND_LOCATION` |
| Android 11（API 30） | 位置权限支持「仅这一次」；后台位置改为「设置页里单独申请」 |
| Android 12（API 31） | 用户可选择「大概位置」，应用可能只拿到 `ACCESS_COARSE_LOCATION` |
| Android 14（API 34） | 前台服务必须声明类型，位置类需 `FOREGROUND_SERVICE_LOCATION` |

实际含义：一个声明了 `ACCESS_FINE_LOCATION` 的应用，**不保证**能拿到精确位置。授权结果可以是粗粒度，应用必须能正确处理这种情况。

### 2.4 位置欺骗

三条路径，权限要求差别很大：

| 路径 | 前提 | 效果 |
|------|------|------|
| 模拟位置应用 | 开发者选项 + AppOps `MOCK_LOCATION` | 所有位置消费者都会收到 mock 结果，`isMock()` 为 true |
| 直接改 GNSS HAL / 注入 NMEA | root 或系统身份 | 可让 `isMock()` 为 false |
| 篡改网络定位结果 | 中间人 / 劫持定位服务端点 | 影响 network provider |

第一条是「明示的」，第三条是「网络层的」，第二条才是真正的「不可检测」——它需要设备已被攻破。做风险评估时必须把这三者分开，否则会把「开发者选项开着」和「系统已被攻陷」混为同一风险等级。

---

## 3. 成因分析

### 3.1 「低敏感」这个判断本身就是根源

传感器被划为低敏感，所以不需要权限；因为不需要权限，所以任何应用都能高频读取；因为能高频读取，所以侧信道成立。**这是一个从分类假设出发、最终落到实际能力上的链条**，而不是某个实现缺陷。

### 3.2 采样率限制落在应用层，硬件层不受约束

200 Hz 是应用请求的上限，HAL 与 DSP 的内部控制回路不受影响。这意味着限制的效果取决于「HAL 是否真的按请求速率上报」，而这一层在厂商手里。

### 3.3 融合把多源合并成一个结论

`fused` provider 把 GNSS、Wi-Fi、基站、传感器融合成一个 `Location`。好处是省电与平滑，代价是**出问题时无法从结果反推来源**。取证时经常需要回到原始 GNSS 测量，而那条通路对普通应用并不开放。

### 3.4 精度分档的实现不透明

「大概位置」的模糊化算法不在应用可见的契约里。应用只能观察到「精度半径很大」，无法验证这个值是否被正确实现。这给了实现方很大的自由度，也让「是否真的模糊了」难以自证。

### 3.5 mock 是开发功能，却长期留在生产镜像

模拟位置的本意是给开发者调试导航类应用。它依赖「开发者选项」这个总开关，而开发者选项在企业设备上经常被整体打开（用于调试、日志采集）。于是调试便利与管理边界直接冲突。

### 3.6 HAL、校准数据与固件

传感器的**校准数据**（出厂标定、温度补偿）通常存在 vendor 分区。校准被篡改会导致数据系统性偏移，而且很难被发现——因为它看起来仍然「合理」。这类问题不在常规权限模型内。

---

## 4. 公开行为变更台账

| 版本 | 变更 | 影响 |
|------|------|------|
| Android 10（API 29） | 后台位置独立授权 | 后台定位从「顺便」变成「显式」 |
| Android 11（API 30） | 「仅这一次」；后台位置单独设置页申请 | 长期静默定位的默认成本上升 |
| Android 12（API 31） | 大概位置；`HIGH_SAMPLING_RATE_SENSORS`；200 Hz 上限 | 位置精度与传感器频率同时被收紧 |
| Android 14（API 34） | 前台服务类型强制声明 | 位置前台服务必须显式建模 |

与输入、音频同理，本手册**不对传感器与定位相关 CVE 做逐条组件归属**：缺陷大量分布在厂商 HAL、GNSS 固件与校准数据中，公告组件名与补丁之间缺少稳定映射。

---

## 5. 防护技术

### 5.1 应用侧

- **按需申请**：能要「大概」就不要「精确」；能前台就不要后台。
- **正确处理粗粒度授权**：不要假设 `ACCESS_FINE_LOCATION` 一定被授予，代码里要能降级运行。
- **优先使用 `fused` provider**，而不是自己直接开 GNSS：省电，且融合层已经做了合理性校验。
- **检查 `Location.isMock()`**，对导航、签到、配送这类场景，mock 结果应当被拒绝或标记。
- **不要用传感器做身份判断**：步态、握持习惯这类特征不稳定，误判成本高。

### 5.2 平台侧

- 把「传感器采样率」「直接通道」「后台位置」放在同一个可视化面板里管理，而不是分散在三处。
- 对 `SensorDirectChannel` 的创建做审计：它绕过了常规事件队列，是观测盲区。
- 位置权限的「大概/精确」选择要可撤销、可追溯。

### 5.3 厂商侧

- 传感器 HAL 的参数（采样率、批处理时长、唤醒标志）来自应用，必须校验上下界。
- 校准数据要有完整性保护（签名或受保护分区），并纳入产线校验。
- GNSS 侧要能区分「真实定位」与「注入定位」，至少在日志层留下痕迹。

### 5.4 企业侧

- 基线里写清三件事：位置权限授予清单、开发者选项开关状态、模拟位置应用是否为 `none`。
- 对高安全场景，把「传感器采样率是否被提升」也纳入检测——这通常意味着有应用在做高频采集。

---

## 6. 自查与观测清单

```bash
# 1. 传感器服务
dumpsys sensorservice                      # 已注册监听者、采样率、传感器列表
dumpsys sensorservice | grep -i "active\|rate"

# 2. 定位服务
dumpsys location                           # provider 状态、最近请求者
dumpsys location | grep -i "mock\|provider"

# 3. 权限与 AppOps
cmd appops get <package> FINE_LOCATION
cmd appops get <package> COARSE_LOCATION
cmd appops get <package> MONITOR_LOCATION
cmd appops get <package> MOCK_LOCATION

# 4. 开发者选项与模拟位置
settings get global development_settings_enabled
settings get secure mock_location

# 5. 传感器 HAL 形态
service list | grep -i sensor

# 6. 前台服务类型（Android 14+）
dumpsys activity services | grep -i "foregroundServiceType\|location"
```

判读要点：

- `dumpsys sensorservice` 里若出现非预期的 `active` 监听且采样率很高，先看是哪个包。
- `dumpsys location` 的 mock 段如果能列出应用，说明模拟位置通路是打开的。
- `settings get secure mock_location` 返回 `null` 不等于「没有模拟位置应用」，不同版本取值不同，需结合 `cmd appops` 判断。

---

## 7. 常见误传纠正

1. **「读加速度计需要权限」**——错。motion 传感器通常不需要运行时权限。
2. **「没有定位权限就推断不出位置」**——错。磁力计、气压计、Wi-Fi/BLE 扫描结果都携带位置信息。
3. **「200 Hz 限制让侧信道失效」**——错。它降低门槛，不消除可能性。
4. **「`HIGH_SAMPLING_RATE_SENSORS` 是危险权限」**——错。它是 normal 级。
5. **「声明了 `ACCESS_FINE_LOCATION` 就一定拿到精确位置」**——错。用户可以选择「大概」。
6. **「后台位置和前台位置是同一个权限」**——Android 10 起是两个。
7. **「`isMock()` 为 false 就一定是真实位置」**——不一定。HAL 层注入可以绕过。
8. **「passive provider 不定位，所以不用管」**——错。它会转发别人的定位结果。
9. **「步数是原始传感器数据」**——错。它是 HAL/DSP 聚合结果。
10. **「`SensorDirectChannel` 和普通监听只是性能差别」**——错。它改变了可观测性。
11. **「定位精度分档只是 UI 提示」**——错。它在数据层实现。
12. **「校准数据不在安全范围内」**——错。篡改校准会造成系统性偏差且难以发现。
13. **「开发者选项开着没风险」**——它会打开模拟位置等调试通路。

---

## 未核实与失效来源

- 传感器 HAL 的 AIDL 形态具体起始版本：未核实。
- `HIGH_SAMPLING_RATE_SENSORS` 在各厂商实现上的实际生效方式（是否真的限制到 200 Hz）：未核实。
- 坐标模糊化的现行算法与参数：官方只给出量级，具体实现未核实。
- `GnssMeasurementsEvent` 注册所需的完整权限集合：未核实。
- `settings get secure mock_location` 在不同 Android 版本的取值语义：未核实。

---

## 参考

- 传感器概览：[source.android.com/docs/core/interaction/sensors](https://source.android.com/docs/core/interaction/sensors)
- 传感器 HAL：[source.android.com/docs/core/interaction/sensors/sensor-stack](https://source.android.com/docs/core/interaction/sensors/sensor-stack)
- 定位概览：[source.android.com/docs/core/location](https://source.android.com/docs/core/location)
- GNSS HAL：[source.android.com/docs/core/connect/gnss](https://source.android.com/docs/core/connect/gnss)
- 位置权限与「大概位置」：[developer.android.com/develop/sensors-and-location/location/permissions](https://developer.android.com/develop/sensors-and-location/location/permissions)
- `SensorDirectChannel`：[developer.android.com/reference/android/hardware/SensorDirectChannel](https://developer.android.com/reference/android/hardware/SensorDirectChannel)
- `SensorManager`：[developer.android.com/reference/android/hardware/SensorManager](https://developer.android.com/reference/android/hardware/SensorManager)
- `Location#isMock`：[developer.android.com/reference/android/location/Location](https://developer.android.com/reference/android/location/Location)
- Android 12 行为变更（传感器采样率）：[developer.android.com/about/versions/12/behavior-changes-all](https://developer.android.com/about/versions/12/behavior-changes-all)
- `SensorService`：[android.googlesource.com/platform/frameworks/native/+/refs/heads/main/services/sensorservice](https://android.googlesource.com/platform/frameworks/native/+/refs/heads/main/services/sensorservice)

---

## 联系方式

如果你在企业里需要就「免权限传感器的采集基线」「定位精度分档是否按预期生效」「模拟位置通路是否关闭」做一次现网核查，可以通过 Telegram 联系：<https://t.me/one00190>

---

*本手册仅用于防御与教育目的，讨论的均为公开且已被修复问题的机制与设计缺陷，不包含任何可直接利用的代码。文中标注「未核实」的内容不应作为事实引用；发现错误欢迎指出。*
