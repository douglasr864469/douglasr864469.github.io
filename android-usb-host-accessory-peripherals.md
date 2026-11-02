---
layout: page
title: "Android USB 主机、配件与有线外设攻击面手册"
description: "把「插上 USB」拆成三条角色不同的路径：主机模式由手机供电并枚举、配件模式（AOA）由配件供电并枚举、设备侧 gadget 由对端主机枚举；权限侧讲清 `requestPermission` 的授权只到设备断开为止、`UsbManager.ACTION_USB_PERMISSION` 并不存在（官方示..."
date: 2026-11-03
permalink: /android-usb-host-accessory-peripherals/
---

> USB 是手机上唯一一个「插上就能同时传数据、供电、还可能被当成键盘和网卡」的接口。它的安全模型与蓝牙、Wi-Fi 都不同：没有配对、没有加密协商、没有链路层身份，插入动作本身就是一次物理信任决策。这份手册沿着「谁当主机 → 怎么枚举 → 谁有权限打开 → 系统把哪类设备交给了谁 → 拔出时会发生什么」这条链，逐段说明 Android 的控制点、公开缺陷的落点，以及防御方真正能检查的东西。只讨论公开且已修复的问题，不含可利用代码。

## 0. 使用方式与边界

这份手册面向三类读者：写 USB 外设或配件固件的工程师、写 USB 主机应用的应用开发者、以及需要评估手机「有线暴露面」的安全人员。三者的控制点完全不同：

- **外设/配件工程师**决定设备描述符、接口与端点、AOA 握手字符串、供电行为；手机端的权限弹窗与设备策略不在固件里。
- **应用开发者**能控制的只有清单声明、权限请求、打开设备后的读写逻辑；类驱动的行为由内核与系统服务决定。
- **安全评估者**需要区分「物理上插得进去」「系统枚举成功」「应用获得授权」「数据真的被读取」这四个不同层次——它们各自有独立的失败点。

本文的三条自我约束：

1. 只引用公开且已修复的问题，CVE 描述以 Android Security Bulletin 的组件列值为准。
2. 不写利用步骤、不写 PoC、不写可复现的攻击参数。
3. 查不到一手来源的说法一律进「未核实与失效来源」，不写进正文。

与已发布手册的分工：《ADB、备份与数据迁移手册》讲的是调试通道与备份协议本身；这一篇只讲 USB 总线上的角色、权限与外设类别。《HAL 与 Treble 边界手册》讲的是 HAL 接口的一般安全边界；这一篇只处理 USB gadget HAL 这一个具体实例。《无线与近场通信攻击面手册》覆盖射频入口，与本文没有重叠。

## 1. 三条路径：主机、配件、设备侧

同一个 USB-C 口在 Android 上可能扮演三种角色，理解它们的区别是理解后面所有权限模型的前提。

| 路径 | 谁供电 | 谁枚举 | 手机侧 API | 典型场景 |
|------|--------|--------|------------|----------|
| 主机模式（host） | 手机 | 手机 | `UsbManager` / `UsbDevice` / `UsbDeviceConnection` | 接 U 盘、键鼠、相机、串口 |
| 配件模式（accessory） | 外接配件 | 外接配件 | `UsbManager` / `UsbAccessory` | 车载底座、音频 Dock、开发板 |
| 设备侧（gadget） | 对端主机 | 对端主机 | 无公开应用 API，走系统服务与 HAL | 连电脑传文件、ADB、网络共享 |

三条路径的共同点是：**插入动作由用户完成，但「插入之后系统把设备交给谁」由系统策略、描述符内容和已安装应用共同决定**。区别在于信任方向：

- 主机模式是「手机信任外设」——手机要给外设供电、解析它的描述符、把它的输入当成用户输入。这是攻击面最大的一条。
- 配件模式是「外设信任手机」——外设提供电源并枚举手机，手机只需要判断「这个配件声称自己是谁」。
- 设备侧是「电脑信任手机」——手机把自己伪装成存储、网卡、串口或调试设备，风险落在对端电脑与手机暴露的数据上。

一个常见的误解是把三条路径混为一谈，例如认为「USB 调试开关能挡住恶意 U 盘」。它挡不住：ADB 属于设备侧 gadget 的一个功能位，而 U 盘走的是主机模式的类驱动，两者由不同的代码路径处理。

## 2. 主机模式：枚举、权限与打开

### 2.1 API 面

主机模式的公开 API 集中在 `android.hardware.usb` 包。官方文档的表述是：手机在主机模式下「acts as the USB host, powers the bus, and enumerates connected USB devices」，主机模式从 Android 3.1（API 12）开始支持。

| 类 | 职责 |
|----|------|
| `UsbManager` | 枚举设备、检查/请求权限、打开设备或配件 |
| `UsbDevice` | 一个已枚举设备的元数据：VID/PID、类、接口列表 |
| `UsbInterface` | 设备的一个功能接口，含端点列表 |
| `UsbEndpoint` | 接口上的一个通信端点（方向、类型、最大包长） |
| `UsbDeviceConnection` | 打开后的连接句柄，负责 claim 接口与传输 |
| `UsbRequest` | 异步传输请求 |
| `UsbConstants` | 与内核 `linux/usb/ch9.h` 对应的常量 |

清单侧需要声明 `android.hardware.usb.host` 特性。注意这个 `<uses-feature>` 的作用是**声明依赖**，让应用商店在不支持主机模式的设备上不展示该应用，它不是权限，也不会阻止应用运行。真正的访问控制发生在打开设备那一刻。

枚举本身的门槛很低：`UsbManager.getDeviceList()` 返回总线上所有已连接设备的哈希表，官方参考页对这一方法只标注了特性依赖，没有要求任何权限。也就是说，**一个普通应用可以看到「现在插着一个 VID/PID 为某某的设备」，但看不到也读不了它的数据**。这个区分很重要：设备的存在与类型本身就可能泄露信息（例如「用户正在使用某型号的硬件密钥」），而读取内容需要另外的授权。

### 2.2 权限：一次授权，直到拔出

USB 的设备权限不是 Android 运行时权限体系里的那种「危险权限」，它是一套按设备实例发放的临时授权。官方参考页对 `requestPermission(UsbDevice, PendingIntent)` 的说明写得很明确：

> 请求成功后，「grants the caller permission to access the device only until the device is disconnected」。

这条语义有几个直接推论：

1. **权限不跨设备**。换一个同型号设备重新插入，需要重新授权（实现上按设备实例与用户维度记录，不是按 VID/PID 永久记住）。
2. **权限不跨重启持久保存**。拔出即失效，重启后自然也没有。
3. **权限是按包发放的**。同一个应用内的所有组件共享；不同应用各自申请。
4. **没有「始终允许」选项**。系统对话框只回答本次连接。

请求的结果通过传入的 `PendingIntent` 返回，携带两个 extra：`UsbManager.EXTRA_DEVICE`（设备对象）与 `UsbManager.EXTRA_PERMISSION_GRANTED`（布尔结果）。`hasPermission()` 可以在打开前做一次检查，避免在用户拒绝后调用 `openDevice()` 得到 `null`。

这里有一个流传很广的错误：**`UsbManager.ACTION_USB_PERMISSION` 并不存在**。官方参考页的常量列表里只有四个 attach/detach 动作与三个 extra，权限结果是通过调用方自己定义的 action 回传的——官方示例里写的是 `"com.android.example.USB_PERMISSION"`，那是一个**应用自定义字符串**，不是系统常量。把示例里的自定义 action 当成系统广播去静态注册，是常见的集成错误。

另一个容易被忽略的细节是视频类设备：官方参考页在 `hasPermission(UsbDevice)` 与 `requestPermission(UsbDevice, PendingIntent)` 上都加了一句——对于 `USB_CLASS_VIDEO` 的设备，**目标 SDK 为 P 及以上的应用还必须额外持有 `CAMERA` 权限**，否则授权不会生效。这条把「USB 摄像头」接入了相机权限体系：能打开 UVC 设备不等于能拿到画面，还得先有相机运行时权限。这是一个很有代表性的设计——**外设类别的敏感度由系统按类映射到既有权限**，而不是另造一套。

### 2.3 intent-filter 与 device_filter：被误解的「默认应用」语义

主机模式还有一条「自动发现」路径：应用在清单里注册 `android.hardware.usb.action.USB_DEVICE_ATTACHED` 的 intent-filter，并用 `<meta-data>` 指向 `res/xml/device_filter.xml`。官方文档对这个机制的描述是：

> 当用户插入匹配的设备时，「the system presents them with a dialog that asks if they want to start your application. If users accept, your application automatically has permission to access the device until the device is disconnected.」

这句话里有两个关键点，经常被读反：

- **不是静默授权**。系统会弹一个「是否打开某应用」的对话框，用户接受才授权。把 device_filter 描述成「插入即自动获得权限」是错的。
- **授权仍然只到拔出为止**。它和 `requestPermission` 的时效相同，只是省掉了应用内再弹一次权限框。

`device_filter.xml` 的匹配维度是 `vendor-id`、`product-id`、`class`、`subclass`、`protocol`。官方文档明确指出：「Specifying no attributes matches every USB device」——**空的 `<usb-device />` 会匹配所有设备**，所以只应在确实需要「任何设备」的场景使用。这条规则是很多「为什么我的应用对所有 U 盘都弹窗」问题的根源，也是安全评估时要检查的清单项：一个声明了空过滤器的应用，等于在每次插入任意 USB 设备时都参与竞争。

还有一层语义值得说明：这套机制的实质是「**用户选择一个应用来处理这类设备**」，接近默认应用，而不是「授予某个应用访问所有 USB 的能力」。因此：

- 多个应用声明同一过滤器时，用户的选择决定本次由谁处理；
- 用户拒绝后，应用不会获得权限，但仍可通过 `requestPermission` 再次请求；
- 过滤器匹配的是**设备自报的描述符字段**，见下一节。

### 2.4 设备自报的身份不是身份认证

`UsbDevice` 上的 VID/PID、类、子类、协议、厂商名、产品名，全部来自设备在枚举时提供的描述符。设备想声称自己是任何厂商的任何产品都可以。这一点决定了三件事：

1. **device_filter 只能做「分流」，不能做「鉴别」**。按 VID/PID 过滤只意味着「这类设备交给我」，不意味着「这台设备可信」。
2. **应用内的白名单校验同样可以被伪造**。如果业务逻辑是「只接受 VID=0x1234 的读卡器」，那么一个伪装成该 VID 的任意设备就能进入后续代码路径。
3. **USB 没有平台级的配件认证**。Android 公开文档中没有定义「USB 设备身份认证」机制；厂商可能在自有配件上做带内挑战应答，但那是厂商私有协议，不是 Android 的保证。把「VID/PID 对得上」当成「这是原厂配件」是典型的概念替换。

这也是为什么主机模式的真正边界不在「识别设备」，而在「打开设备后如何处理不可信输入」：所有来自端点的字节都必须当成攻击者可控数据。

### 2.5 接口、端点与 claimInterface

打开设备用 `UsbManager.openDevice(UsbDevice)`，返回 `UsbDeviceConnection` 或 `null`。之后要 `claimInterface(UsbInterface, boolean force)` 才能在该接口的端点上传输。`force` 参数的含义是「即使内核已有驱动占用该接口也强行接管」。

这个参数是主机模式里最值得警惕的一个开关：

- 对 HID、存储、音频这类**已有内核类驱动**的接口，强行 claim 会把设备从系统手里抢过来；
- 对厂商自定义接口（class 255），通常没有内核驱动，claim 是正常路径；
- claim 成功后，原本依赖该类驱动的系统功能（输入、媒体、存储挂载）可能同时失效或行为异常。

传输本身分同步与异步：`bulkTransfer` / `controlTransfer` 是同步的，`UsbRequest` + `requestWait` 是异步的。官方文档反复强调要在工作线程上做传输，这既是性能要求也是稳定性要求——主线程阻塞会触发 ANR，而 USB 传输的超时与拔出事件都不可预测。

拔出时，`ACTION_USB_DEVICE_DETACHED` 会送达，应用需要 `releaseInterface()` 与 `close()`。**没有正确处理拔出，是 USB 应用崩溃与句柄泄漏的主要来源**；从安全角度看，拔出后继续使用旧连接对象，属于典型的生命周期缺陷模式。

## 3. 配件模式与 AOA

### 3.1 谁供电、谁是主机

配件模式（Android Open Accessory，AOA）把角色反过来：

> 「the attached Android USB accessory acts as the host, provides power to the USB bus, and enumerates connected devices.」

也就是说，外接配件负责供电与枚举，手机成为被枚举的一方。这条路径的存在理由很实际：很多设备（尤其是早期的车机、音箱、底座）不具备 USB 主机能力，但它们可以提供电源和一颗简单的 USB 控制器。

官方参考页还给出两条硬约束：

- **同一时刻只支持一个配件**（`getAccessoryList()` 的说明是「in the current implementation there can be at most one」）；
- 配件通过 `UsbManager.openAccessory(UsbAccessory)` 打开，返回 `ParcelFileDescriptor`，读写就是对这个 fd 做流式 I/O；API 36 起另有 `openAccessoryInputStream()` / `openAccessoryOutputStream()` 两个更直接的流接口。

清单侧声明 `android.hardware.usb.accessory`。权限模型与主机模式同构：`requestPermission(UsbAccessory, PendingIntent)` 弹框，结果同样走 `EXTRA_PERMISSION_GRANTED`，时效同样到断开为止；用 intent-filter + `accessory_filter.xml` 时，匹配维度是 `manufacturer`、`model`、`version`。

### 3.2 AOA 握手做了什么

AOA 的握手由配件侧发起，使用三个厂商控制请求：

| 请求值 | 名称 | 作用 |
|--------|------|------|
| 51 | Get Protocol | 询问手机支持的 AOA 协议版本 |
| 52 | Send String | 发送标识字符串（索引 0–5） |
| 53 | Start | 请求手机切换到配件模式 |

字符串索引固定为：0 厂商名、1 型号、2 描述、3 版本、4 URI、5 序列号，每条最长 256 字节（零结尾）。握手完成后，手机以 Google 的 VID（`0x18D1`）重新枚举，配件模式下 PID 为 `0x2D00`，配件 + ADB 为 `0x2D01`。

### 3.3 字符串不是认证

这是本节最重要的一句话：**AOA 的厂商名、型号、序列号全部由配件自报，没有任何签名或挑战应答**。官方 AOA 页面描述的四步是「检测 → 查询版本 → 发送标识 → 启动」，其中没有认证环节。因此：

- 用 `manufacturer` / `model` 过滤配件，和用 VID/PID 过滤设备一样，只是**分流**；
- 序列号可以被任意伪造，不能作为「这台底座是原厂」的证据；
- 应用如果基于这些字符串做特权判断（例如「型号匹配就开放调试功能」），等于把信任建立在攻击者可控字段上。

同一条逻辑也适用于主机模式的 `getManufacturerName()` / `getProductName()`：它们返回的是描述符里的字符串，用于展示与过滤可以，用于授权不行。

还有一个容易忽略的细节：官方 AOAv2 文档提到，配件**可以省略厂商名与型号字符串**，以避开应用关联对话框。这意味着「缺少字符串」不是异常，而是被协议允许的行为——任何「没有厂商名就拒绝」的校验都可能误伤合法配件，反过来，「有厂商名就信任」则完全不可靠。

### 3.4 AOAv2 的音频与 HID

AOAv2 在 v1 的基础上增加了两类能力：

- **音频**：通过标准 USB Audio Class 接口向配件输出 2 声道、16 位 PCM、44.1 kHz 音频，需要在 Start 之前发送 `SET_AUDIO_MODE`。官方页面明确写着「AOAv2 audio support has been deprecated in Android 8.0」——**这条能力已经退场**，新设计不应依赖它。
- **HID**：配件可以注册为一个或多个 HID 设备，通过控制请求 54–57 完成注册、注销、设置报告描述符与发送 HID 事件。官方特别说明实现「makes no assumptions about the content or type of events」，即系统不解释事件语义，只做转发。

HID 这条路径的安全含义需要单独说明：配件注册的 HID 事件会进入系统的输入管线，最终表现为**用户输入**。从系统视角看，它和真实键盘没有区别。因此：

- 「配件能发 HID 事件」等价于「配件能模拟用户操作」，这是 AOA 设计上就接受的能力，不是漏洞；
- 抵御它的手段不在输入层做区分，而在**不把敏感操作建立在单次输入之上**（例如关键设置要求二次确认、企业设备禁用 USB 数据信令、自助终端使用锁定任务模式）；
- 把「外接键盘输入」当成「用户本人在场」的强证据，在任何平台上都不成立。

官方 AOAv2 页面还给出了带 ADB 的 PID 变体（`0x2D01`、`0x2D03`、`0x2D05`），并提示除非配件确实实现了 ADB 透传，否则不要使用。这条提示本身就说明：**ADB 能力是配件协议里的一个可选功能位**，不是独立的安全边界。

## 4. 外设类别与系统消费点

主机模式枚举到设备后，系统会按接口类把设备分发给不同的消费者。理解这张分发表，才能判断「一个恶意设备能触达哪一层」。

| 类别（USB class） | 系统消费点 | 应用是否参与 | 安全含义 |
|-------------------|------------|--------------|----------|
| HID（键盘/鼠标/手柄） | 内核 HID → evdev → `InputReader` | 否（事件进入输入系统） | 设备可模拟用户输入 |
| 音频（UAC） | 音频 HAL / tinyalsa | 否 | 设备成为音频输入输出端点 |
| 视频（UVC） | `ExternalCameraProvider` → Camera2 | 是（需 `CAMERA` 权限） | 设备成为外接相机 |
| MTP / PTP | `MtpService`（`android.process.media`） | 否 | 把共享存储暴露给对端主机 |
| RNDIS / NCM | 网络共享（Tethering） | 否 | 手机侧出现一个网络接口 |
| MIDI | `MidiService` | 是（MIDI API） | 设备成为 MIDI 端点 |
| 大容量存储（UMS） | 无官方设备侧实现 | — | Android 不作为 UMS 从设备 |
| CDC 串口 / 打印机 | 无系统级封装 | 是（`UsbManager` 自行实现） | 应用直接面对原始字节 |

几条值得展开的：

**HID 不需要应用授权**。键盘、鼠标走内核 HID 栈，事件直接进入 `InputReader`，不经过 `UsbManager` 的权限对话框。这意味着「插入一个 HID 设备」在系统看来就是「多了一个输入源」，不存在「某个应用被授权使用这个键盘」的概念。防御方要关注的不是权限，而是**输入能否触达敏感界面**。

**UVC 需要相机权限**。如 2.2 节所述，视频类设备除了 USB 设备授权，还要求目标 SDK P+ 的应用持有 `CAMERA`。这让外接摄像头与内置摄像头共享同一套隐私开关。

**RNDIS/NCM 进入网络栈**。USB 网络设备会被网络共享组件接管，表现为一个网络接口；它不是「某个应用获得了网卡」，而是系统级网络路径的变化。评估时应当把它与 Wi-Fi、蜂窝并列看待，而不是当成外设。

**MIDI 有独立服务**。`MidiService` 管理 MIDI 设备的枚举与端口，应用通过 MIDI API 打开端口。它与音频路径不同，值得在「USB 音频」之外单独检查。

**没有官方 UMS**。Android 设备侧不提供 USB Mass Storage 模式；对端主机看到的「手机存储」是 MTP/PTP，不是块设备。这一点在下一节展开。

## 5. MTP/PTP：共享存储的边界

### 5.1 解锁门槛

MTP 的实现是 `MtpService`，运行在 `android.process.media` 进程。它的一个关键行为是：**只有在收到 `UsbManager.USB_DATA_UNLOCKED` 为 true 的意图时，才会把存储卷加入 MTP 会话**。这个 extra 由 `UsbDeviceManager` 在用户解锁后发出（源码里对应 `intent.putExtra(UsbManager.USB_DATA_UNLOCKED, ...)`），它属于系统内部接口，不在公开 API 列表中。

这条设计的含义很直接：**锁屏状态下插上电脑，电脑看不到文件列表**。这是防止「捡到手机直接拷数据」的第一道门槛。它不能替代全盘加密，但把「物理接触」到「数据可读」之间插入了一个必须由用户完成的解锁动作。

### 5.2 PTP 的目录白名单

PTP（照片传输协议）不是 MTP 的简化版，它在实现上有明确的范围限制：`MtpService` 中的 `PTP_DIRECTORIES` 只包含 `DCIM` 与 `Pictures` 两个目录。也就是说，选择 PTP 模式时，对端主机能看到的只有相机与图片目录，而不是全部共享存储。

这是一条经常被忽略的最小权限设计：**同一个服务，用不同的 USB 功能位暴露不同范围的数据**。评估时如果只看到「MTP 已连接」，还要确认当前功能位是 MTP 还是 PTP。

### 5.3 不暴露 /data

MTP 暴露的是**已挂载的共享存储卷**（主存储是必须项，其他卷按挂载情况加入），不是块设备，也不是应用私有目录。`/data` 下的应用沙箱、数据库、`shared_prefs` 不在 MTP 的可见范围内。这与「root 后用 adb pull」是两件完全不同的事：前者是协议层就没有对应对象，后者是绕过沙箱的调试路径。

需要强调的是，「MTP 不暴露 `/data`」不等于「MTP 没有风险」：

- 共享存储里可能有下载的文档、聊天附件、备份文件；
- 应用如果把敏感数据写在外部存储，MTP 就能读到；
- PTP 的白名单之外，MTP 的范围仍然覆盖整个共享存储。

### 5.4 功能位与系统属性

设备侧的 USB 配置通过一组系统属性表达，`UsbDeviceManager` 源码中可以直接看到：

| 属性 | 含义 |
|------|------|
| `sys.usb.config` | 当前请求的 USB 功能组合 |
| `sys.usb.state` | 已生效的功能组合（状态机切换完成后写入） |
| `sys.usb.controller` | 使用的 USB 控制器名 |
| `persist.sys.usb.config` | 跨重启保留的默认功能组合 |

`sys.usb.config` 与 `sys.usb.state` 的区别值得记住：前者是「想要什么」，后者是「实际成了什么」。两者不一致通常意味着切换失败或正在进行中。评估设备异常时，先看这两个值比看 UI 更可靠。

另外，网络上常见的 `ro.usb.vendor` 在 AOSP 中并不存在（本次未在源码中检索到），属于厂商私有属性，不要把它当成平台标准。

## 6. ADB：调试通道的边界

ADB 是设备侧 gadget 的一个功能位，但它值得单独一节，因为它是「物理接触 → 逻辑控制」最短的一条路径。

### 6.1 三道门

要建立 ADB 连接，至少需要同时满足：

1. **开发者选项与 USB 调试被打开**。对应 `Settings.Global.ADB_ENABLED`（键名 `adb_enabled`），默认关闭。
2. **主机密钥被授权**。主机在 `~/.android/adbkey` 持有私钥，设备在 `/data/misc/adb/adb_keys` 保存已授权公钥；首次连接时设备弹「允许 USB 调试」对话框，用户勾选后写入。
3. **adbd 侧允许认证**。`ro.adb.secure` 决定是否强制主机密钥认证；正式（user）构建上即使该属性为 0 也不允许免认证连接，免认证只可能出现在 debuggable 构建上。

「撤销 USB 调试授权」会清空设备侧的 `adb_keys`，所有主机需要重新授权。这个操作是事件响应里的标准动作之一。

### 6.2 常见误判

- **「USB 调试默认开启」**：不成立，默认关闭，且需要用户手动进入开发者选项。
- **「无线调试绕过了授权」**：不成立。无线调试（`adb pair`）需要设备显示的配对码完成配对，配对成功后才建立信任；它是把 USB 传输换成了 TCP，不是把认证去掉。
- **`ADB_VENDOR_KEYS` 是后门**：不是。该环境变量只在 userdebug/eng 构建上有效，用于加载厂商密钥；正式构建不使用它。
- **「关掉 USB 调试就安全了」**：不成立。ADB 只是 gadget 的一个功能位，MTP、PTP、RNDIS 仍然可能暴露数据或改变网络路径。

### 6.3 企业侧开关

设备管理侧有两个与 USB 相关的限制：

- `UserManager.DISALLOW_USB_FILE_TRANSFER`（对应 XML 里的 `no_usb_file_transfer`）禁用 USB 文件传输；
- `UserManager.DISALLOW_DEBUGGING_FEATURES`（`no_debugging_features`）禁用调试相关功能，包括 ADB。

Android 12 起又增加了 USB 数据信令开关。`DevicePolicyManager` 源码中的说明是：由设备所有者（device owner）或组织拥有工作资料的组织所有者（profile owner of an organization-owned managed profile）调用，**关闭后「USB data connections (except from charging functions) are prohibited」**，即只保留充电。配套方法有三个：

| 方法 | 作用 | 异常 |
|------|------|------|
| `setUsbDataSignalingEnabled(boolean)` | 开/关 USB 数据信令 | 无权限抛 `SecurityException`；设备不支持或设置失败抛 `IllegalStateException` |
| `isUsbDataSignalingEnabled()` | 查询当前状态 | — |
| `canUsbDataSignalingBeDisabled()` | 设备是否支持该能力 | — |

注意这三个方法在 AOSP main 上都不带 `ComponentName` 参数（早期文档与二手资料里常见的带 `ComponentName` 版本不是当前签名）。`canUsbDataSignalingBeDisabled()` 的存在本身就说明：**该能力依赖硬件与 HAL 支持，不是所有设备都能用**。部署前必须先探测，而不是假定调用一定成功。

## 7. 设备侧 gadget：HAL、功能位与 configfs

手机作为 USB 设备时，功能组合由 gadget HAL 管理。当前的 AIDL 接口是 `android.hardware.usb.gadget.IUsbGadget`，核心方法包括 `setCurrentUsbFunctions()`、`getCurrentUsbFunctions()` 与查询速度的方法；功能位定义在 `GadgetFunction.aidl`：

| 常量 | 值 | 含义 |
|------|----|------|
| `NONE` | 0 | 无功能 |
| `ADB` | 1 | 调试 |
| `ACCESSORY` | 1<<1 | 配件模式 |
| `MTP` | 1<<2 | 媒体传输 |
| `MIDI` | 1<<3 | MIDI |
| `PTP` | 1<<4 | 照片传输 |
| `RNDIS` | 1<<5 | USB 网络（RNDIS） |
| `AUDIO_SOURCE` | 1<<6 | 音频源 |
| `UVC` | 1<<7 | USB 摄像头 |
| `NCM` | 1<<10 | USB 网络（NCM） |

这些位可以组合。系统把组合写给 HAL，HAL 再通过 configfs 配置内核 gadget：`/sys/kernel/config/usb_gadget/<gadget>/` 下挂 `functions/`、`configs/`，最后把 UDC 绑定到控制器。**普通应用不能直接写 configfs**——该路径由 init 与厂商 HAL 进程按 SELinux 策略控制，应用只能通过系统服务间接触发功能切换。

这条链路的安全含义有两层：

1. **功能位是攻击面开关**。同时打开 MTP + ADB + RNDIS 的设备，暴露面是三者之和；减少功能位就是减少暴露面。某些定制系统提供「仅充电」模式，本质就是把功能位设为 `NONE`。
2. **HAL 是特权边界**。gadget HAL 运行在厂商域，能改内核配置；它的实现缺陷影响的是整机，而不是单个应用。评估厂商实现时，功能位切换的状态机（切换失败、并发切换、超时）比单个功能本身更值得看。

## 8. USB-C、PD 与 Alt Mode

USB-C 把「插头方向」和「角色协商」都交给了硬件。Android 框架对这部分的支持有一个明确的边界：**公开 API 只暴露端口状态，不解析 PD 报文**。

`UsbPort` 与 `UsbPortStatus` 都是 `@SystemApi`，需要 `MANAGE_USB` 级别的权限才能通过 `UsbManager.getPorts()` 读取。它们能告诉你：

| 维度 | 常量 | 含义 |
|------|------|------|
| 数据角色 | `DATA_ROLE_HOST` / `DATA_ROLE_DEVICE` | 当前谁在数据上做主 |
| 电源角色 | `POWER_ROLE_SOURCE` / `POWER_ROLE_SINK` | 当前谁供电 |
| 端口模式 | `MODE_DFP` / `MODE_UFP` / `MODE_DUAL` | 下行/上行/双角色 |
| Alt Mode | `FLAG_ALT_MODE_TYPE_DISPLAYPORT`、`isAltModeSupported()` | 是否支持 DisplayPort 等替代模式 |

几个容易踩的点：

- **没有 `MODE_DRP` 常量**。双角色端口对应的是 `MODE_DUAL`，不是「DRP」。
- **数据角色与电源角色相互独立**。USB-C 允许「一边供电一边做设备」，所以「正在充电」不能推出「没有数据连接」。
- **Alt Mode 是能力声明，不是当前状态**。`FLAG_ALT_MODE_TYPE_DISPLAYPORT` 表示端口支持该模式，具体是否进入要结合端口状态判断。
- **PD 协商发生在端口控制器侧**。框架看到的是协商结果（角色、模式），没有 PD 报文接口；因此「应用能读取 PD 电压/电流」这类说法在标准 API 上不成立。

从安全角度看，USB-C 带来的新问题是**物理层的模糊性**：同一个口既可能接充电器，也可能接数据主机，还可能接一个伪装成充电器的数据设备。系统能做的只是把「数据信令」这个开关暴露给企业策略（见 6.3），或者在「仅充电」模式下把功能位清零。用户侧能做的判断非常有限——**外观与「是否在充电」都不能证明「没有数据连接」**。

### 8.5 五类接触场景的判断顺序

把端口状态、功能位与设备类别放在一起看，日常遇到的五类场景可以这样快速定位：

| 场景 | 系统里能看到什么 | 关键判断 | 主要防线 |
|------|------------------|----------|----------|
| 陌生充电器/公共充电口 | 可能同时存在电源角色与数据角色 | 「在充电」不能证明「无数据」 | 仅充电话 / 关闭 USB 数据信令 |
| 未知 HID 设备 | 输入系统多出一个键盘或指针设备 | 输入事件会被当成用户操作 | 锁定任务模式、敏感操作二次确认 |
| 伪装成某型号的存储设备 | 枚举成功，VID/PID 与预期一致 | 描述符可伪造，白名单不是认证 | 不在外部存储放敏感数据、按内容而非身份校验 |
| 声称是某品牌的 AOA 配件 | 配件模式 + 自报字符串 | 字符串无签名，可任意填写 | 不基于字符串授予特权 |
| 受控主机（取证、企业） | 对端发起 MTP 或 ADB | 解锁门槛与主机密钥授权是两道独立门 | 撤销调试授权、关闭调试、企业策略 |

这张表的用法是「先看角色，再看功能位，最后看类别」：角色决定谁在控制总线，功能位决定手机暴露了什么，类别决定哪段系统代码在处理对端数据。三步都确认过，才谈得上「这条连接是否安全」。

## 9. 公告台账：已核实的公开缺陷

Android 安全公告没有独立的「USB」章节，USB 相关条目以**内核组件/子组件**的形式出现在公告表格里，常见取值有 `USB`、`USB driver`、`USB gadget`、`USB HID`、`USB audio`、`MTP USB driver`、`UVC`。下面这些条目均取自公告表格原文，只保留组件列值与类型/级别。

### 9.1 已核实条目

| CVE | 公告月份 | 组件/子组件 | 类型 / 级别 | 备注 |
|-----|----------|-------------|-------------|------|
| CVE-2017-13181 | 2018-01 | Media framework（实际为 MtpServer） | EoP / High | 公告措辞与真实组件不一致，见 9.2 |
| CVE-2018-9417 | 2018-07 | USB driver | EoP / High | 主机侧驱动 |
| CVE-2018-19824 | 2019-10 | USB audio | EoP / High | USB 音频类 |
| CVE-2020-0179 | Pixel 2020-06 | Media framework（实际为 MtpServer） | EoP / Moderate | 见 9.2 |
| CVE-2020-13143 | Pixel 2020-12 | USB gadget | EoP / Moderate | 设备侧 gadget |
| CVE-2021-39640 | Pixel 2021-12 | USB | EoP / High | — |
| CVE-2022-24958 | 2022-06 | USB | EoP / High | — |
| CVE-2022-25258 | 2022-06 | USB | EoP / High | 与上一条同月 |
| CVE-2022-20132 | 2022-06 | USB HID | ID / High | 信息泄露 |
| CVE-2022-25375 | 2022-06 | RNDIS driver | ID / High | USB 网络类 |
| CVE-2023-20941 | 2023-04 | USB | EoP / High | 公告编号 A-264029575 |
| CVE-2024-53104 | 2025-02 | UVC | EoP / High | 公告含「有限定向利用」提示 |
| CVE-2024-53150 | 2025-04 | USB | ID / High | 公告含「有限定向利用」提示 |
| CVE-2024-53197 | 2025-04 | USB | EoP / High | 与上一条同月修复 |

这张表能看出两个分布特征：

- **EoP 占多数**。USB 是内核代码，缺陷的典型后果是越界写、UAF 一类的内存破坏，最终表现为提权；信息泄露（ID）集中在 HID 与 RNDIS 这类「解析对端数据」的子组件上。
- **2022-06 是一个密集月**。同一个公告里出现四条 USB 相关条目，覆盖 USB、USB HID、RNDIS 三个子组件，说明这条链路上的缺陷往往是成批发现的。

### 9.2 公告措辞与真实组件不一致

有两个条目需要特别说明，因为它们的公告组件列值和实际修复的代码位置不同：

- **CVE-2017-13181**：公告列在 Media framework，但公开的修复提交与漏洞描述指向 MTP 服务端（MtpServer），公告编号 A-67864232。
- **CVE-2020-0179**：Pixel 公告同样列在 Media framework，实际也是 MTP 服务端。

这两条说明「按公告章节归类」和「按代码模块归类」会得出不同结果。写台账时，**以公告的组件列值为准记录，同时注明真实组件**，不要把两者混成一个字段。反过来，如果看到「Media framework 漏洞」就断定与 USB 无关，同样会漏掉 MTP 这条路径。

还有一个边界案例：**CVE-2018-9445**（2018-08，Framework，A-80436257）的修复涉及 `e2fsprogs` 与 `system/vold`，NVD 描述提到 USB 设备挂载。它更接近「USB 存储介质上的文件系统解析」，而不是 USB 协议栈本身。本文把它列为边界案例，不并入上面的主表。

### 9.3 常见误挂载

以下编号在二手资料里经常被说成「USB 漏洞」，实际组件是别的，本文明确排除：

- CVE-2014-7939（V8）、CVE-2015-3837（conscrypt）、CVE-2016-2508（mediaserver）、CVE-2016-3822（jhead）；
- CVE-2017-13162、CVE-2019-2025、CVE-2019-2181、CVE-2020-0030、CVE-2020-0041（均为 binder）；
- CVE-2020-0096（ActivityStartController）、CVE-2021-0519（ih264e）、CVE-2021-39624（PackageManager）；
- CVE-2022-20186（Mali）、CVE-2023-20951（Bluetooth GATT）、CVE-2023-35669 / CVE-2024-0048（AccountManager）、CVE-2024-43093（ExternalStorageProvider）。

判断方法很简单：**打开对应月份的公告表格，看那一行的 Component/Subcomponent 列**。如果写的是 Framework、System、Media framework 或某个具体服务，就不应当因为漏洞描述里出现「USB」一词而把它归到 USB 栈。

## 10. 六种成因模式

把上面的机制抽象一下，USB 相关缺陷集中在六种模式上。它们与具体 CVE 无关，但对代码审计与设计评审更有用。

**模式一：描述符解析的长度假设**。设备上报的配置、接口、端点、类特定描述符都是变长结构，且数量由 `bNumInterfaces` / `bNumEndpoints` 等字段自报。解析方如果先按「典型设备」分配缓冲、再按自报数量填充，就会出现越界或截断。UVC 与 HID 的报告描述符是这类问题的典型位置。

**模式二：热插拔下的生命周期**。USB 的最大特点是「随时可能消失」。传输进行中拔出、claim 后设备被重置、异步请求回调晚于 close，都会产生 use-after-free 或空指针。正确做法是把「拔出」当成正常路径而不是异常路径。

**模式三：类驱动的对端数据信任**。RNDIS 的报文长度、HID 的报告长度、音频的采样描述，都来自对端。把「对端说有多长」直接当成「实际有多长」，是信息泄露与越界读的常见起点。

**模式四：权限与文件描述符的错配**。USB 设备权限是按包、按设备实例、到拔出为止的授权；一旦 fd 被跨进程传递，接收方拿到的能力就不再受原授权约束。设计上要明确「谁打开、谁关闭、fd 是否外传」。

**模式五：把用户交互当成认证**。device_filter 的「是否打开此应用」对话框、AOA 的厂商字符串、HID 的按键事件，都是**交互**而不是**认证**。把它们当成信任凭证，等于把安全决策交给可伪造的输入。

**模式六：策略覆盖不全**。关掉 ADB 不等于关掉 MTP；关掉 MTP 不等于关掉 RNDIS；关掉数据信令才覆盖全部数据功能。策略的有效性取决于它覆盖的功能位集合，而不是它名字里的关键词。

## 11. 观测点与自查

### 11.1 设备侧

| 目的 | 命令 / 位置 | 说明 |
|------|-------------|------|
| 看已枚举设备与端口 | `adb shell dumpsys usb` | 系统 USB 服务的状态转储 |
| 看当前功能位 | `adb shell getprop sys.usb.state` | 实际生效值；与 `sys.usb.config` 对比 |
| 看持久默认功能 | `adb shell getprop persist.sys.usb.config` | 跨重启保留 |
| 看调试开关 | `adb shell settings get global adb_enabled` | 1 表示已打开 |
| 看已授权主机 | `adb shell ls /data/misc/adb/` | 需要 root 或调试权限 |
| 看网络接口 | `adb shell ip link` | RNDIS/NCM 会新增接口 |

`dumpsys usb` 的输出通常包含设备列表、当前功能、端口状态与调试状态，是排查「插上了但没反应」的第一站。注意不同厂商会裁剪字段，字段缺失不代表功能不存在。

### 11.2 应用侧清单检查

- 是否声明了 `android.hardware.usb.host` / `android.hardware.usb.accessory`，以及是否设置了 `required`；
- `device_filter.xml` 是否存在空 `<usb-device />`（匹配所有设备）；
- 是否在 `onDestroy` / 拔出广播里释放接口与连接；
- 是否只按 VID/PID 做安全判断；
- 打开 UVC 设备前是否检查了 `CAMERA` 权限；
- 是否把 `ACTION_USB_PERMISSION` 误当成系统常量。

### 11.3 企业/终端侧

- `DISALLOW_USB_FILE_TRANSFER` 与 `DISALLOW_DEBUGGING_FEATURES` 是否按需启用；
- 是否探测过 `canUsbDataSignalingBeDisabled()` 再调用 `setUsbDataSignalingEnabled(false)`；
- 自助终端是否使用锁定任务模式，避免外接 HID 触达系统设置；
- 是否把「仅充电」模式作为默认，按需临时开放数据。

## 12. 常见误传纠正

| 说法 | 实际情况 |
|------|----------|
| USB 设备授权是永久的 | 只到设备断开为止，官方参考页原文如此 |
| `UsbManager.ACTION_USB_PERMISSION` 是系统常量 | 不存在；官方示例用的是应用自定义 action |
| device_filter 会让应用静默获得权限 | 系统会弹「是否打开此应用」对话框，用户接受才授权 |
| 空 device_filter 只匹配自家设备 | 不写任何属性会匹配所有 USB 设备 |
| MTP 能访问 `/data` | 只暴露已挂载的共享存储卷；应用私有目录不在协议对象里 |
| Android 支持设备侧 UMS | 没有官方实现；对端看到的是 MTP/PTP |
| 无线调试绕过了 ADB 授权 | 需要配对码完成配对，认证没有被去掉 |
| USB 调试默认开启 | 默认关闭，需用户手动打开开发者选项 |
| 框架会解析 PD 报文 | 公开 API 只暴露端口角色与模式，无 PD 报文接口 |
| USB 配件有平台级认证 | 没有；AOA 字符串与 VID/PID 都由设备自报 |
| 插着充电就说明没有数据连接 | 数据角色与电源角色独立，可同时存在 |
| 关掉 USB 调试就关闭了 USB 攻击面 | MTP/PTP/RNDIS 等 gadget 功能与主机模式仍然存在 |

## 13. 与其他手册的分工

| 主题 | 在哪一篇 |
|------|----------|
| ADB 协议、备份、数据迁移 | 《ADB、备份与数据迁移手册》 |
| HAL 接口的一般安全边界 | 《HAL 与 Treble 边界手册》 |
| 存储挂载与文件系统解析 | 《存储与文件系统栈手册》 |
| 输入事件管线（HID 的上游） | 《输入子系统与事件注入手册》 |
| 摄像头与麦克风隐私开关 | 《摄像头、麦克风与屏幕捕获面手册》 |
| 网络接口与内核过滤 | 《网络数据路径与内核过滤手册》 |
| USB 角色、权限、外设类别、gadget | 本篇 |

## 14. 参考

- USB 主机指南：<https://developer.android.com/develop/connectivity/usb/host>
- USB 配件指南：<https://developer.android.com/develop/connectivity/usb/accessory>
- `UsbManager` 参考：<https://developer.android.com/reference/android/hardware/usb/UsbManager>
- AOA 协议：<https://source.android.com/docs/core/interaction/accessories/aoa>
- AOA 2.0：<https://source.android.com/docs/core/interaction/accessories/aoa2>
- Android Security Bulletin 2018-01 / 2018-07 / 2018-08 / 2019-10 / 2022-06 / 2023-04 / 2025-02 / 2025-04
- Pixel Security Bulletin 2020-06 / 2020-12 / 2021-12
- AOSP 源码：`MtpService.java`、`GadgetFunction.aidl`、`IUsbGadget.aidl`、`UsbPort.java`、`UsbPortStatus.java`、`UsbDeviceManager.java`、`DevicePolicyManager.java`

## 未核实与失效来源

**A. AOA 配件的供电下限**
「配件需提供 5 V / 500 mA」是 ADK 时代的常见说法，但本次在 AOA 与 AOAv2 的开发者页面上都没有取到该句。**正文只写「配件负责供电」，不写具体数值。**

**B. `ro.usb.vendor`**
在 AOSP 源码中未检索到该属性，属厂商私有。**正文不把它列为平台属性。**

**C. USB gadget HAL 的版本对应**
正文只描述 AIDL 接口与功能位；「某版本起从 HIDL 迁到 AIDL」的具体版本号未逐条核对，**故不写版本号**。

**D. `setUsbDataSignalingEnabled` 的 HAL 版本要求**
早期笔记提到「需要 USB HAL 3.1」，但本次抓取的 `DevicePolicyManager` 源码 javadoc 中没有该表述（只有「不是所有设备都支持，先调用 `canUsbDataSignalingBeDisabled()`」）。**正文按源码表述写，不写 HAL 版本。**

**E. `UsbDevice.getSerialNumber()` 的权限语义**
不同版本对该方法在无授权时返回 `null` 还是抛异常的行为不完全一致，本文未逐版本核对，**正文不使用该方法做任何判断。**

**F. AOA 产品 ID 的完整映射**
开发者页面确认了 `0x2D00`、`0x2D01`，并提到 `0x2D03`、`0x2D05` 带 ADB，但 `0x2D02`–`0x2D05` 的完整逐项含义未逐字取到。**正文只列已确认的三个。**

**G. `dumpsys usb` 的字段稳定性**
该命令的字段随版本与厂商变化，正文只把它作为观测入口，不承诺字段名。

**H. CVE-2018-9445 的归类**
公告组件为 Framework，修复涉及 `e2fsprogs` 与 `system/vold`，NVD 描述提到 USB 设备挂载。它更像「USB 介质上的文件系统解析」而非 USB 协议栈，**正文列为边界案例，不并入主表。**

**I. MTP 的进程与 uid 细节**
正文写「运行在 `android.process.media`」，未进一步断言 sharedUserId 的具体取值。

**J. USB 音频在 Android 8.0 之后的状态**
AOAv2 页面明确「音频支持在 Android 8.0 弃用」，但「弃用」是协议侧能力的退场，不代表 USB Audio Class 主机模式被移除；正文只引用前者。

---

## 联系方式

咨询 Android 系统详情请联系 https://t.me/one00190

---

*本手册仅供安全学习与防御研究使用；所有机制表述、公告列值与 CVE 描述均取自公开一手来源并附链接，未能核实的部分已单列于「未核实与失效来源」而非写入正文。*
