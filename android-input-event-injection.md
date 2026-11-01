---
layout: page
title: "Android 输入子系统与事件注入攻击面手册"
description: "把「点一下」拆成三层三种身份：内核 evdev 的 24 字节 `input_event` 与 `SYN_REPORT` 提交屏障、system_server 内的 EventHub/InputReader/InputDispatcher、以及应用侧走 socketpair 而非 Binder 的 In..."
date: 2026-11-01
permalink: /android-input-event-injection/
---

> 手机做的每一件事，起点几乎都是一次触摸、一次按键、一次手势。这条通路在三个完全不同的世界里各有一套表示：内核里是 `/dev/input/event*` 上一个 24 字节的 `struct input_event`，框架里是 system_server 中两个线程之间的一条队列，应用里是一个 `MotionEvent` 对象。三次表示转换、两次跨进程搬运，每一次都要回答同一个问题——**这个事件是谁产生的**。答错一次，屏幕上的「点一下」就可以来自另一台设备。本手册只讨论公开且已修复问题的机制与自查方法，不含可利用代码。

---

## 0. 这篇手册解决什么问题

已发布的手册里，《框架 IPC 与组件导出攻击面》讲的是应用之间怎么互相调用，《摄像头、麦克风与屏幕捕获面》讲的是「看」与「听」的授权。本篇补的是第三种感官：**「碰」**。

输入子系统有三个常被混为一谈的边界：

1. **内核到框架**：evdev 字符设备 → `EventHub` → `InputReader`。这一层决定了「设备能报出什么」。
2. **框架到应用**：`InputDispatcher` → `InputChannel` → 应用主线程。这一层决定了「事件给谁」，而且它**不是 Binder**。
3. **应用或工具到应用**：`injectInputEvent()`、无障碍手势、`uinput`。这一层决定了「事件能不能凭空出现」。

三个边界的信任假设完全不同。把它们合成一句「输入可以被伪造」或「输入无法伪造」，都会得出错误结论。

---

## 1. 架构：三层、两种身份

### 1.1 内核侧：evdev 与 24 字节

Linux 输入子系统把每种设备抽象成 `input_dev`，evdev 再把每个设备暴露成一个字符设备节点：

```
/dev/input/event0   电源键
/dev/input/event1   触摸屏
/dev/input/event2   加速度计
/dev/input/uinput   空设备：往这里写，就会「长出」一个新输入设备
```

读出来的是固定结构的记录：

```
struct input_event {
    struct timeval time;   /* 16 字节（64 位）*/
    __u16 type;            /* EV_KEY / EV_ABS / EV_REL / EV_SYN / EV_MSC */
    __u16 code;            /* 具体键码或轴码，如 KEY_VOLUMEUP / ABS_MT_POSITION_X */
    __s32 value;           /* 按下 1、抬起 0、重复 2；轴值按设备量程 */
};
```

一条记录描述的是**一个码**，不是一次完整动作。一次点击由 `EV_ABS`（位置）、`EV_KEY`（`BTN_TOUCH`）、最后一次 `EV_SYN` + `SYN_REPORT` 组成。`SYN_REPORT` 是提交屏障：在它之前的所有记录属于同一帧，在它之后才是下一帧。**丢掉这个屏障，就会把两次滑动粘成一次**。

多点触控有两套协议：A 协议每帧重报所有触点，B 协议用 `ABS_MT_SLOT` 选槽位、`ABS_MT_TRACKING_ID` 标识触点生命周期。现代设备基本都是 B 协议。看到资料里说「触控只有一套协议」，那是把 A 协议当成了全部。

`EVIOCGRAB` 是个容易被忽略的开关：一个进程 grab 了某个设备后，其他读者就收不到事件。输入法、游戏手柄映射工具都可能用它。

`uinput` 的意义在于：**它让用户态进程可以创建一个内核认可的输入设备**。写进去的事件，从内核往下看和真实硬件没有任何区别。Android 的蓝牙 HID 主机就是靠它把远端鼠标键盘变成系统输入设备的——这条设计的副作用，是整个「蓝牙外设注入」问题的基础。

### 1.2 框架侧：EventHub 与 InputReader

官方对这段的描述是直接的：

> "EventHub reads input events from the kernel by opening the evdev driver associated with each input device."
> "InputReader decodes the input events according to the device class and produces a stream of Android input events."
> "InputDispatcher forwards them to the appropriate window."

三句话对应三个职责，但官方页面**没有**说明它们跑在哪个进程。这一点必须从代码结构上确认，因为它是很多误判的源头：

- `EventHub`、`InputReader`、`InputDispatcher` 都在 **system_server** 进程内（`InputManagerService` 持有它们，各自跑在 `InputReaderThread` 与 `InputDispatcherThread` 上）。
- 它们**不在**应用进程，**也不在** SurfaceFlinger。
- 因此「应用卡住会不会拖慢输入」这个问题的正确答案是：不会拖慢采集与分发，拖慢的是**该应用窗口自己的事件队列**，最终表现为 ANR。

`InputReader` 做的事比名字暗示的多：

- **设备分类**：根据能力位判断是键盘、触摸屏、鼠标、触控板、摇杆、开关（`SW_LID`）还是传感器。
- **键位映射**：读取 `/vendor/usr/keylayout/*.kl` 与 `/system/usr/keylayout/*.kl`，把扫描码翻成 `AKEYCODE_*`。OEM 自定义按键的差异都在这一层。
- **触摸校准**：把设备量程映射到屏幕坐标，应用旋转与显示缩放。
- **手势解释**：电源键长按、音量键组合、三指截屏这类**系统级组合**在这里被识别，不再作为普通按键往下发。

这一步的关键性质是：**InputReader 输出的已经是「语义」，不是原始硬件数据**。一旦下到 `InputDispatcher`，事件的「设备身份」就只剩下一个 `deviceId` 和一个 `source`。

### 1.3 分发侧：InputChannel 不是 Binder

输入事件的分发用的是 **UNIX domain socket 的 socketpair**，不是 Binder。原因是延迟：Binder 一次事务的代价对 120 Hz 触控采样来说太高。

```
system_server                                 应用进程
────────────                                  ────────
InputDispatcher
  └─ Connection
       └─ InputPublisher ──(socketpair fd)──▶ InputConsumer
                                                 └─ 投递到 ViewRootImpl
                                                     └─ 主线程 Handler
                                                 ◀── finish() 回执
```

两个后果：

1. **应用必须在限定时间内 `finish`**。超时后系统打出 `Input dispatching timed out`，这就是 ANR 里最常见的「输入分发超时」。
2. **socket 的另一端是文件描述符**。谁拿到这个 fd，谁就能读走事件、注入回执。这也是「调试工具能模拟点击」的技术前提——它不需要绕过多强的权限，只需要拿到一个可写的通道或调用注入 API。

### 1.4 注入路径清单

把「事件能不能凭空产生」拆开看，入口有五类，权限要求差别很大：

| 入口 | 身份要求 | 能注入什么 | 备注 |
|------|----------|------------|------|
| `InputManager.injectInputEvent()` | 持有 `INJECT_EVENTS`（签名级） | 任意 `InputEvent` | 框架主入口，`mode` 决定是否等待完成 |
| `adb shell input` | shell UID（持 `INJECT_EVENTS`） | 文本、按键、点击、滑动 | 实现是 `frameworks/base/cmds/input`，不是 Binder 客户端工具 |
| `UiAutomation.injectInputEvent()` | instrumentation，运行身份为 shell | 任意 `InputEvent` | 测试框架与自动化脚本的入口 |
| `AccessibilityService.dispatchGesture()` | 已启用的无障碍服务 | 只能是 `GestureDescription`（路径手势） | **不能**合成任意按键，也没有 `injectInputEvent` 的通用性 |
| `/dev/input/uinput` 写入 | SELinux 允许该域访问 `uinput_device` | 可以「新建一个设备」 | 不受 `INJECT_EVENTS` 约束，受 SELinux 与节点权限约束 |

这张表解释了一个常见困惑：为什么「无障碍服务能力很大」和「`input` 命令能做的事」是两件事。前者被限制在「手势 + 节点动作」，后者是完整的注入接口，只是身份被锁死在 shell。

### 1.5 「来源」能不能被信任

注入事件的两个字段值得单独说：

- **`deviceId`**：真实设备有正数 id；通过 `injectInputEvent()` 注入的事件，`deviceId` 为 `-1`。
- **`source`**：由注入者填写（`SOURCES_TOUCHSCREEN`、`SOURCES_KEYBOARD` 等）。框架不会把它反推回真实设备。

所以正确的结论是：**`deviceId == -1` 可以用来识别「这不是来自某个物理设备」，但 `source` 与 `deviceId` 都不能证明「这来自用户」**。反过来的误判更常见——认为「有 deviceId 就一定是真人操作」，这只在未被注入的路径上成立。

---

## 2. 四个真实的风险面

### 2.1 无障碍服务：完整的手眼替代

一个启用了 `canRetrieveWindowContent` 的无障碍服务，同时具备两种能力：读取界面树（看到什么）与执行节点动作或手势（点到什么）。这两者相加，等价于一台可以远程操作手机的机械臂加摄像头。

需要注意的是权限模型的实际形状：

- 它是**用户显式开启**的（设置 → 无障碍），不是安装即得。
- 它有 `android:isAccessibilityTool` 标记（Android 13 起可用），用于把「辅助工具」与「自动化工具」区分开，Play 政策对后者有额外限制。
- Android 13 起，从非商店渠道安装的应用在开启无障碍前会进入「受限设置」，需要用户额外确认。这是对「诱导开启」的缓解，不是能力上的削弱。

结论不是「无障碍危险所以别用」，而是：**一旦开启，该服务在能力上等同于完全控制**。做威胁建模时不应把它当成「读屏权限」，而应把它划进「可代表用户操作」的最高档。

### 2.2 覆盖层与点击劫持

点击劫持（tapjacking）的本质不是「伪造事件」，而是**让用户在 A 上点击，却由 B 收到**。攻击者不需要注入权限，只需要一个覆盖窗口。

框架提供了三层检测与阻断，名字必须分清：

| 机制 | 类型 | 作用 |
|------|------|------|
| `MotionEvent.FLAG_WINDOW_IS_OBSCURED`（`0x00000001`） | 事件标志 | 该事件发生时有窗口部分遮挡 |
| `MotionEvent.FLAG_WINDOW_IS_PARTIALLY_OBSCURED`（`0x00000002`） | 事件标志 | 遮挡面积未达完全遮挡阈值 |
| `View.setFilterTouchesWhenObscured(true)` | 视图属性 | 被遮挡时直接丢弃触摸 |
| `Window.setHideOverlayWindows(true)` | 窗口能力（API 31） | 请求系统隐藏非系统覆盖层 |
| 不可信触摸阻断 | 系统行为（Android 12 起） | 见下 |

Android 12 的行为变更需要准确表述：对 targetSdk ≥ 31 的应用，系统会**阻断经由不可信窗口传递过来的触摸**。判定依据是覆盖窗口的「不可信程度」与不透明度组合，官方给出的关键阈值是最大遮挡不透明度 `0.8`。这不是「所有覆盖层都被禁」，而是「不可信覆盖层既收不到、也不再向后传递触摸」。

还要区分两个容易混淆的点：

- `SYSTEM_ALERT_WINDOW`（悬浮窗）是**权限**，`TYPE_APPLICATION_OVERLAY` 是**窗口类型**。Android 8（API 26）起，targetSdk ≥ 26 的应用不能再使用旧的 `TYPE_SYSTEM_ALERT`。
- 覆盖层的坐标系与目标窗口的坐标系可以不同（缩放、偏移、分屏）。所以「挡住按钮」不等于「事件坐标相同」。

### 2.3 输入法：能读上下文的那一端

输入法天生要拿到「当前正在编辑什么」。`InputConnection` 提供的 `getTextBeforeCursor`、`getSelectedText`、`commitText`、`sendKeyEvent` 构成了一条双向通道：既能读用户输入框里的内容，也能往里写。

风险不在「输入法能读输入框」——那是它的职责；风险在于三处边界：

- **密码框**：`InputType.TYPE_TEXT_VARIATION_PASSWORD` 等标志会改变 `getTextBeforeCursor` 的可见性，但不同控件、不同实现的行为并不一致，不能把「设了密码类型」当作绝对保证。
- **`sendKeyEvent`**：输入法可以向当前焦点控件发送按键事件，作用域限于该连接。
- **全屏输入法**：`IME_FLAG_NO_EXTRACT_UI` 之外的模式会接管整个屏幕，此时用户看到的界面不是应用画的。

### 2.4 内核与固件：真正的第一现场

框架层再怎么校验，都建立在「内核报上来的东西是真的」这个前提上。因此：

- **触摸屏驱动**解析 OEM 私有协议（压力、手套模式、防误触），任何长度与索引缺失都是内核内存问题。
- **蓝牙 HID**：远端设备通过 `uhid` 变成系统输入设备，这是设计使然。所以「蓝牙配对被滥用」不是漏洞，而是**配对即信任**这一模型的必然后果。配对时看到的设备类型与之后上报的事件类型不绑定，是这条路径最值得检查的地方。
- **USB HID**：外接键盘鼠标在锁屏状态下通常可用（用于输入密码），这条通路在 kiosk 场景下需要单独评估。

---

## 3. 成因分析：为什么这个子系统总是出问题

### 3.1 三层表示，没有一层做端到端认证

`input_event` 里有时间戳，没有来源签名；`MotionEvent` 里有 `deviceId` 与 `source`，没有可信标记；`InputChannel` 是裸 socket。整条链没有任何一处能证明「这是物理接触」。**可辨识性**（能看出是注入）和**可认证性**（能证明是本人）是两件事，这个子系统只提供了前者的一部分。

### 3.2 权限被切成「签名级」与「用户授权」两套，中间没有灰区

`INJECT_EVENTS` 是签名级，普通应用拿不到；无障碍是用户授权，拿到即全权。结果是：一旦攻击者跨过「用户点同意」这道门，就直接站在了最高档，没有中间梯度。这与存储、相机那类「分级权限」的设计形成对比。

### 3.3 覆盖层是窗口系统的通用能力，输入系统只能被动防御

窗口管理允许任意有权限的应用叠加窗口，输入系统在分发时才知道「上面还有一层」。于是一切防御都只能是**事后判定**：先算出遮挡，再决定丢不丢。这类判定天然存在竞态——判定与投递之间，覆盖层可能已经变化。

### 3.4 兼容层让「旧类型」长期存在

`TYPE_SYSTEM_ALERT`、旧版 `AccessibilityService` 回调、A 协议多点触控，这些兼容路径在真实设备上仍然存在。安全评审如果只看新 API，就会漏掉旧路径。

### 3.5 测试与运维工具被当成正常入口

`adb shell input`、`UiAutomation`、`uiautomator` 在生产镜像里通常仍然可用（受开发者选项与 USB 调试控制）。它们不是漏洞，但它们是**已经写好的注入器**。评估「设备被物理接触后的风险」时，必须先算清这条通路是否开着。

### 3.6 事件洪泛是可用性攻击面

`InputDispatcher` 对每个连接有队列上限，超时即 ANR。持续注入、或者在窗口切换时卡住回执，都能造成「应用无响应」的观感。这类问题很少被计入安全评分，但在真实对抗中很有效。

---

## 4. 公开行为变更台账

输入子系统的很多「问题」在官方口径里属于**行为变更**而非 CVE。区分这两者很重要：行为变更说明设计在演进，CVE 说明实现有缺陷。本手册只登记前者，且不臆造编号。

| 版本 | 变更 | 影响 |
|------|------|------|
| Android 8（API 26） | `TYPE_SYSTEM_ALERT` 对 targetSdk ≥ 26 失效，改用 `TYPE_APPLICATION_OVERLAY` | 旧式悬浮窗路径被切断，但 `SYSTEM_ALERT_WINDOW` 权限仍在 |
| Android 12（API 31） | 不可信触摸阻断；新增 `Window.setHideOverlayWindows()`；新增 `FLAG_WINDOW_IS_PARTIALLY_OBSCURED` | 点击劫持的默认成本显著提高 |
| Android 13（API 33） | 无障碍服务可用 `isAccessibilityTool` 声明自身性质；侧载应用的「受限设置」 | 把「辅助」与「自动化」分开治理 |

关于具体 CVE 编号与本子系统的对应关系，本手册**不做归属断言**：输入相关缺陷大量出现在厂商触摸驱动与 HID 解析代码中，公开公告的组件命名（如 `HID`、`Input`）与具体补丁之间缺少稳定的一一映射，凭编号反推组件容易出错。相关编号核对方法见《漏洞编号与公告体系手册》。

---

## 5. 防护技术

### 5.1 应用侧：把「不可信输入」当成默认假设

- 对敏感操作（支付、改密、授权）使用 `BiometricPrompt` 并**绑定 `CryptoObject`**，让认证结果不可被重放或伪造。
- 全屏或关键界面调用 `setHideOverlayWindows(true)`，并检查触摸事件的 `FLAG_WINDOW_IS_OBSCURED`。
- 对 `View` 设 `android:filterTouchesWhenObscured="true"`，尤其是确认按钮。
- 不要用「点击位置」做安全判断；用「哪个控件收到了点击」。
- 密码输入框显式设置 `TYPE_TEXT_VARIATION_PASSWORD`，并且不要自行实现「屏蔽输入法」的假安全逻辑。

### 5.2 平台侧：把能力收紧在正确的档位

- 对 `INJECT_EVENTS` 的授予采取白名单，不做「按需临时授予」。
- 无障碍服务开启前的二次确认要覆盖「更新后仍生效」这一场景。
- `SYSTEM_ALERT_WINDOW` 的授予要可见、可撤销、可审计。
- 对 `uinput` 的 SELinux 访问做最小化：只有确实需要创建输入设备的域才允许。

### 5.3 厂商侧：把解析当攻击面

- 触摸屏与传感器 hub 的固件协议解析要按「不可信输入」对待，长度、索引、溢出全部校验。
- 打开内核输入栈的模糊测试（`syzkaller` 对 `uinput` 有专门支持），重点覆盖 `EVIOCG*`/`EVIOCS*` 与多点触控槽位状态机。
- 蓝牙 HID 主机侧要把「设备描述符声明的能力」与「实际上报的事件」做一致性校验。

### 5.4 企业侧：把「谁能代点」写进基线

MDM 与合规基线里通常只有「允许/禁止无障碍」。更实用的口径是三件事：哪些包被授予了无障碍、哪些包持有 `SYSTEM_ALERT_WINDOW`、设备上 `adb` 与开发者选项是否打开。这三项都能用只读命令采集。

---

## 6. 自查与观测清单

以下命令都是只读观测，不需要 root：

```bash
# 1. 输入设备与能力
getevent -pl                      # 列出设备、能力位、轴范围
cat /proc/bus/input/devices       # 设备名、handler、能力位

# 2. 事件流（观察来源与时间戳）
getevent -lt                      # 带时间戳与标签的实时事件

# 3. 输入服务状态
dumpsys input                     # 设备、Dispatcher 状态、各连接的等待情况
dumpsys input | grep -i "anr\|timeout\|pending"

# 4. 无障碍服务
settings get secure enabled_accessibility_services
settings get secure accessibility_enabled

# 5. 覆盖层
cmd appops get <package> SYSTEM_ALERT_WINDOW
dumpsys window windows | grep -i "type=\|isOverlay"

# 6. 设备节点与 SELinux 标签
ls -lZ /dev/input/
ls -lZ /dev/uinput

# 7. 调试通路
settings get global adb_enabled
settings get global development_settings_enabled
```

判读要点：

- `getevent -lt` 里同一帧的事件时间戳应当单调；出现负间隔或跨帧混序，说明驱动或 hub 有问题。
- `dumpsys input` 的 Dispatcher 段如果能持续看到某个窗口的 pending 事件，说明该应用主线程长时间未 `finish`。
- `/dev/uinput` 的 SELinux 标签若不是 `uinput_device`，或该节点对 `untrusted_app` 可写，属高危配置。

---

## 7. 常见误传纠正

1. **「输入事件走 Binder 分发」**——错。分发走 socketpair；Binder 只用于 `InputManagerService` 的控制面（注册、注入请求）。
2. **「InputReader 在应用进程里」**——错。它在 system_server。
3. **「注入的事件没有 deviceId，所以能被可靠识别」**——只能识别「非物理设备」。注入者仍可指定 `source`，且部分路径会分配 id。
4. **「无障碍只能读屏，不能操作」**——错。它能执行节点动作与路径手势。
5. **「无障碍必须用户开启，所以不算攻击面」**——开启后能力等价于完全控制，属于最高档授权。
6. **「`FLAG_SECURE` 能防输入」**——它只影响截屏与录屏，与输入无关。
7. **「覆盖层必须在最顶层才能劫持」**——部分遮挡同样会触发 `FLAG_WINDOW_IS_PARTIALLY_OBSCURED`。
8. **「Android 12 之后覆盖层完全失效」**——错。被阻断的是「不可信覆盖层传递的触摸」，覆盖层本身仍可显示。
9. **「`adb shell input` 走的是 uinput」**——错。它走 `InputManager.injectInputEvent()`。
10. **「`uinput` 只有 root 能用」**——错。决定因素是 SELinux 域与节点权限，不是 uid 是否为 0。
11. **「`SYSTEM_ALERT_WINDOW` 就是悬浮窗类型」**——错。它是权限；`TYPE_APPLICATION_OVERLAY` 才是类型。
12. **「多点触控只有 A 协议」**——错。B 协议才是现代设备的常态。
13. **「输入事件的时间戳来自可信时钟，不能伪造」**——注入路径可以指定时间戳。

---

## 未核实与失效来源

- 各厂商触摸控制器固件协议的细节（如高通、联发科、汇顶的具体报文格式）：未核实，本手册不做具体断言。
- 蓝牙 HID 主机侧对描述符与实际事件的一致性校验是否在所有 Android 版本上实施：未核实。
- Android 12 不可信触摸阻断中「可信窗口」的完整判定清单（除系统窗口外还有哪些例外）：官方页面给的是主规则与 `0.8` 阈值，完整实现细节未逐条核实。
- 输入子系统相关 CVE 的组件归属：本手册有意不做逐条归属，理由见第 4 节。

---

## 参考

- 输入子系统概览：[source.android.com/docs/core/interaction/input](https://source.android.com/docs/core/interaction/input)
- `InputManagerService` / `InputDispatcher`：[android.googlesource.com/platform/frameworks/base/+/refs/heads/main/services/core/java/com/android/server/input](https://android.googlesource.com/platform/frameworks/base/+/refs/heads/main/services/core/java/com/android/server/input)
- `InputChannel` / `InputPublisher` / `InputConsumer`：[android.googlesource.com/platform/frameworks/native/+/refs/heads/main/libs/input](https://android.googlesource.com/platform/frameworks/native/+/refs/heads/main/libs/input)
- `MotionEvent`（遮挡标志）：[developer.android.com/reference/android/view/MotionEvent](https://developer.android.com/reference/android/view/MotionEvent)
- `Window#setHideOverlayWindows`：[developer.android.com/reference/android/view/Window](https://developer.android.com/reference/android/view/Window)
- `InputManager#injectInputEvent`：[developer.android.com/reference/android/hardware/input/InputManager](https://developer.android.com/reference/android/hardware/input/InputManager)
- `AccessibilityService#dispatchGesture`：[developer.android.com/reference/android/accessibilityservice/AccessibilityService](https://developer.android.com/reference/android/accessibilityservice/AccessibilityService)
- Linux 输入子系统文档：[docs.kernel.org/input/input.html](https://docs.kernel.org/input/input.html)
- `uinput` uapi：[docs.kernel.org/input/uinput.html](https://docs.kernel.org/input/uinput.html)
- Android 12 行为变更（不可信触摸）：[developer.android.com/about/versions/12/behavior-changes-12](https://developer.android.com/about/versions/12/behavior-changes-12)
- 无障碍服务政策：[support.google.com/googleplay/android-developer/answer/10964491](https://support.google.com/googleplay/android-developer/answer/10964491)

---

## 联系方式

如果你在企业里做终端输入面安全评审，需要就「无障碍服务授权基线」「覆盖层与点击劫持核查」「注入通路是否关闭」做一次针对现网机型的验证，可以通过 Telegram 联系：<https://t.me/one00190>

---

*本手册仅用于防御与教育目的，讨论的均为公开且已被修复问题的机制与设计缺陷，不包含任何可直接利用的代码。文中标注「未核实」的内容不应作为事实引用；发现错误欢迎指出。*
