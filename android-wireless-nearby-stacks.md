---
layout: page
title: "Android 无线与近场通信攻击面手册（Wi-Fi / 蓝牙 / NFC / UWB）"
description: "把「射频邻近即可攻击」拆成四套独立边界：Wi-Fi 是网络栈（`com.android.wifi` 的模块化有版本差异，Android 11 时 Wi-Fi 服务仍在 system_server 内）、蓝牙是外设与音频总线（Android 12 三个运行时权限与 `neverForLocation` 会..."
date: 2026-10-29
permalink: /android-wireless-nearby-stacks/
---

> 无线不是「一个开关」。Wi-Fi、蓝牙、NFC、UWB 在 Android 里是四套独立的框架、四套独立的 HAL、四种不同的信任前提：Wi-Fi 是网络栈的一部分，蓝牙是外设与音频总线，NFC 要求物理贴近，UWB 直接由 FiRa 的 UCI 规范约束 HAL。把它们当成一类，就会出现「关掉 Wi-Fi 就安全了」这种错判。这份手册按「框架在哪一层、权限由谁拦、哪些输入来自射频」来拆，只讨论公开且已修复问题的机制与自查方法，不含可利用代码。

---

## 0. 这篇手册解决什么问题

已发布的二十篇里，`基带与电话` 讲的是蜂窝侧，`内核与 GKI 驱动` 讲的是驱动与 KMI，但两者之间有一块常被含糊处理：**不经过运营商网络、又不在内核主路径上的那些射频通道**。攻击者不需要公网可达，只要在物理半径内即可——咖啡店的 AP、路过的 BLE 广播、贴一下的 NFC 标签。

这类通道有三个共同点，值得单独成册：

1. **输入来自未经认证的第三方。** 蓝牙广播、Wi-Fi beacon、NFC NDEF 记录，全部是「对方说了算」的字节流，解析器就是你暴露在外的第一层代码。
2. **权限粒度与直觉不符。** 同一个「附近设备」权限组，覆盖蓝牙扫描、广播、连接三件性质不同的事；Wi-Fi 扫描看的是位置权限；而 NFC 根本没有「读取标签」的运行时权限。
3. **默认状态就是暴露状态。** 蓝牙可发现、Wi-Fi 自动连接、NFC 屏幕点亮即可读，这些都是出厂默认，而用户以为「我没用过」。

因此本文不按「漏洞编号」组织，而按**边界**组织：先确定框架在哪一层（§2）、HAL 在哪一面（§3）、权限拦在哪一步（§4），再逐条看三类通道各自的输入面（§5–§7），最后给出公告读法（§8）、外部报告口径（§9）与自查清单（§10）。

---

## 1. 三条通道的边界

| 维度 | Wi-Fi | 蓝牙（经典 + BLE） | NFC |
|---|---|---|---|
| 对端身份 | 网络侧设备（AP、客户端） | 外设（耳机、键鼠、车机）与广播者 | 标签 / 另一台设备 / 读卡器 |
| 连接前提 | 用户选网或自动连接策略 | 配对（经典）或直接连（BLE GATT） | 物理贴近（通常数厘米） |
| 主要信任物 | 网络配置（PSK / 证书） | 绑定记录（link key / LTK） | NDEF 记录、APDU 会话 |
| 框架侧入口 | `WifiManager` / `WifiAwareManager` | `BluetoothAdapter` / `BluetoothLeScanner` / `BluetoothGatt` | `NfcAdapter` / `HostApduService` |
| 是否有「运行时权限」 | 附近设备（Wi-Fi 相关，见 §4） | 有，Android 12 起三个 | **没有**用于读标签的运行时权限 |
| 典型错误的代价 | 流量被中间人、企业凭据泄露 | 输入被注入、音频被劫持、文件被投递 | 标签内容被当作可信指令执行 |

**边界一：蓝牙的「配对」不等于「认证用户」。** 经典蓝牙的绑定记录证明的是「这台设备和那把 link key 匹配」，不证明「坐在设备前的是机主」。HID 类设备（键盘、鼠标）又天然拥有输入注入能力，于是「配对成功」与「可以打字」之间只隔着一层协议角色。

**边界二：Wi-Fi 的信任落在「配置」而不是「信号」。** 同一个 SSID 可以是任何人的 AP；系统的自动连接策略以 SSID/安全类型为键，不以 BSSID 或证书为键（企业网络除外）。这就是「邪恶双胞胎」在协议层成立的原因。

**边界三：NFC 的物理贴近是唯一的访问控制。** 屏幕点亮、NFC 开启时，任何靠近的读卡器都可以发起选择（SELECT）并读取标签；HCE 场景下则是「你的应用冒充卡」，此时边界从「标签」变成「谁在调用你的 `HostApduService`」。

---

## 2. 框架在哪一层：Mainline 模块实名

无线相关的框架代码在不同版本被切进了不同的 Mainline 模块。这件事对安全评审很重要：**模块边界就是补丁边界**——一个缺陷落在 APEX 里可以由模块更新修掉，落在 system_server 里只能等系统 OTA。

已核实的模块实名（逐字引文见文末来源）：

- **`com.android.wifi`**：APEX 形式，官方口径「Android 11 及以上设备可用」，且「可更新（updatable），因此能脱离常规 Android 发布周期接收能力更新」。同一页面另有小标题 `Module boundary for Android 12 and Android 13`——也就是说 Android 11 时 Wi-Fi 服务仍运行在 system_server 进程内，从 Android 12 起才有独立进程边界。评审时若看到「Wi-Fi 服务崩溃拖垮整个系统」的报告，先确认版本。
- **`com.android.tethering`**：APEX，Android 11+；Android 12 起 `packages/modules/Connectivity` 下的全部文件与目录都归入该模块范围。热点、VCN、部分网络栈代码早期「寄居」在这里，这解释了为什么某些网络类补丁会出现在 tethering 模块的提交里。
- **蓝牙**：官方模块页写「Bluetooth module is an optional Mainline module introduced in Android 13」，给出的 APK 包名是 `com.google.android.bt`；并注明「In Android 16 and higher, the module is updatable」。**注意两件事**：一是「optional」意味着并非所有设备都带这个模块；二是 AOSP 构建产物名 `com.android.bt` 未能在官方页面核实（见未核实 B）。
- **NFC**：模块实名 `com.android.nfcservices`，官方页列出的成员包括 NFC API（Java）、NFC 系统 APK（Java，页面出现 `NfcNci.apk`）与内置 NCI 栈（`libnfc_nci.so` / `libnfc_nci_jni.so`），代码位置 `packages/modules/Nfc`。**引入版本存在口径差异**：官方页面现文写「released in Android 16」，而社区普遍记录该模块首现于 Android 13/14。正文按「模块实名确证、引入版本存疑」处理（见未核实 G）。
- **UWB**：模块实名 `com.android.uwb`（页面同时列出安装包 `com.google.android.uwb`），Android 13 引入，「contains a UWB stack above the HAL interface」，HAL 采用 FiRa 定义的 UCI 规范，接口代码在 `hardware/interfaces/uwb`。

**这条事实能直接指导自查**：在一台设备上跑 `pm list packages --apex-only` 或 `dumpsys package com.android.wifi`，就能判断该设备的 Wi-Fi 补丁是由模块更新下发，还是必须等系统版本。如果评估对象是 Android 11 的 Wi-Fi 问题，不能假设「模块化 = 可独立修」。

---

## 3. HAL 面：谁真正碰硬件

框架往下就是 HAL。对做设备侧评审的人来说，HAL 面比 Java API 更重要，因为**绕过权限检查最容易的位置就在 HAL 之上那一层**。

### 3.1 Wi-Fi 的三个面

官方口径是三个面（surface），分别对应三类进程：

| 面 | 对应守护进程 | 管什么 | 稳定 AIDL 时间 |
|---|---|---|---|
| Vendor HAL | 厂商 Wi-Fi 驱动/固件适配层 | 开关、扫描结果、连接 | Android 14 起三面全部稳定 AIDL |
| Supplicant HAL | `wpa_supplicant` | 关联、四次握手、EAP | Android 13 起改用 AIDL |
| Hostapd HAL | `hostapd` | SoftAP（热点） | Android 13 起改用 AIDL |

Android 13 之前这三个面用 HIDL 定义。AOSP 源码里 `android.hardware.wifi.hostapd@1.0`（`IHostapd.hal`，描述为「Top-level object for managing SoftAPs」）确实存在，这是 HIDL 时代的实证。

**为什么这个分工是安全边界**：`wpa_supplicant` 与 `hostapd` 都是 C 代码、都直接解析对端发来的帧与 EAP 报文，历史上 Wi-Fi 侧的信息泄露与内存安全问题大多落在这两个进程的解析路径上。它们运行在独立的 SELinux 域里，权限被收得很窄，但**它们能看到的凭证（PSK、EAP 参数、证书）比任何应用都多**。

### 3.2 蓝牙的 HCI 与音频面

AOSP 构建文件里同时出现 `android.hardware.bluetooth@1.0`、`android.hardware.bluetooth@1.1`、`android.hardware.bluetooth.audio@2.0`、`android.hardware.bluetooth.audio@2.1`，以及 AIDL 命名风格的 `android.hardware.bluetooth.audio-V2-ndk`。含义很直接：**蓝牙核心（HCI）与蓝牙音频路由是两个不同的 HAL 面**，且音频面已经在新版本上走 AIDL。

对防御方来说，这两个面的差别在于：HCI 面能看到链路层事件（连接、配对、角色切换），音频面能看到「音频往哪条通路走」。**把通话音频劫持到攻击者可控的蓝牙设备**，落点是音频面而不是 HCI 面；反过来，**未配对设备发起的按键注入**落在 HCI 之上的 HID 主机角色里。

### 3.3 UWB：HAL 由外部规范定义

UWB 的 HAL 接口采用 FiRa 定义的 UCI 规范。这意味着接口形态不完全由 Google 决定，**评测时需要同时看 FiRa 规范的版本与厂商实现**。UCI 全称与具体报文格式在本轮核验中未展开（见未核实 H）。

---

## 4. 权限模型：三套互不相同的拦截点

### 4.1 蓝牙：Android 12 起三个运行时权限

官方逐字口径：面向 Android 12（API 31）及以上的应用，需在清单声明 `BLUETOOTH_SCAN`、`BLUETOOTH_ADVERTISE`、`BLUETOOTH_CONNECT`，并且「`BLUETOOTH_ADVERTISE`、`BLUETOOTH_CONNECT` 和 `BLUETOOTH_SCAN` 是运行时权限」。

三者对应三件不同的事：

- `BLUETOOTH_SCAN`：发现周边设备（含 BLE 扫描）。
- `BLUETOOTH_ADVERTISE`：让自己可被周边发现（广播/可发现）。
- `BLUETOOTH_CONNECT`：连接已配对设备、读写 GATT、访问设备名。

**「免定位」的清单属性**是 `android:usesPermissionFlags="neverForLocation"`，加在 `BLUETOOTH_SCAN` 上。官方同时给出一条容易忽略的副作用：「If you include `neverForLocation` in your `usesPermissionFlags`, the system filters out some BLE beacons from your scan results.」——也就是说，**声明免定位会让一部分 beacon 从扫描结果里被系统过滤**。如果你的应用声称「扫描不到某些 beacon」，先检查这条声明，而不是先怀疑硬件。

这条属性还有一个安全含义：`neverForLocation` 是**应用对系统的承诺**，不是系统对应用的保证。系统据此可以不再要求定位权限，但反过来说，**带定位权限做扫描的应用，其扫描结果可以反推物理位置**——这就是「BLE 扫描 = 室内定位」的原因，也是 Android 12 之前蓝牙权限长期与位置权限纠缠的根源。

### 4.2 Wi-Fi：扫描看的是位置，不是 Wi-Fi

`WifiManager.startScan()` 与扫描结果（`getScanResults()`）在 Android 9 之后与位置权限绑定，这是公开且长期存在的设计。Android 13 引入 `NEARBY_WIFI_DEVICES` 之后，部分 Wi-Fi 操作的权限口径发生变化，但**本轮未能取得该路径的官方逐字句**（见未核实 C）。

实践结论：**评估一个应用「是否能看到你周围的 AP 列表」，看的是它有没有拿到位置或附近设备权限，而不是它有没有 Wi-Fi 相关权限名**。

### 4.3 NFC：没有「读标签」运行时权限

NFC 的模型与其他两类完全不同：应用申请 `<uses-permission android:name="android.permission.NFC" />`（普通权限），然后通过 `NfcAdapter.enableReaderMode` / `NfcAdapter.setNdefPushMessage` 等注册前台回调。**读标签本身不弹权限框**，因为「贴近」被当成了授权动作。

唯一需要用户显式同意的，是卡片模拟（HCE）：用户要在系统设置里选择「默认支付应用」或「默认卡片模拟服务」，系统才会把读卡器的 SELECT 路由到你的 `HostApduService`。**这条「默认服务」选择是 NFC 侧最重要的用户授权点**，也是最容易被社工引导改掉的一项设置。

---

## 5. 蓝牙：从「可发现」到「能打字」

### 5.1 可发现性是最外层的暴露

经典蓝牙的「可发现（discoverable）」是一个显式窗口，默认不是常开；BLE 的广播（advertising）则相反，**默认就可以被任何扫描者收到**，且广播包里可以带制造商自定义数据。

自查两条：

- `dumpsys bluetooth_manager` 看当前扫描/广播状态；
- 检查应用是否在后台持续广播（这既耗电，也让设备可被长期追踪）。

**BLE 广播的隐私问题不是「你说什么」，而是「你一直在说」**：固定的制造商数据 + 不变的地址，等于一个可被零售店、车站统计设备长期识别的标识。Android 侧的地址轮换策略与设备实现相关，本文不做版本断言。

### 5.2 配对：三种模型，三种信任

- **Just Works**：无 PIN、无确认，双方直接绑定。中间人可以全程在位而用户毫无感知。
- **Passkey / Numeric Comparison**：屏幕上比对 6 位数字，需要用户真的看一眼。
- **Out of Band（OOB）**：通过 NFC 等旁路交换配对信息。

评审要点：**「设备要求输入 PIN」不等于「设备防中间人」**——如果 PIN 是固定的（如车机的 0000/1234），它只是防误连，不防攻击。真正的防中间人能力取决于配对模型，而不是有没有输入框。

### 5.3 HID：配对成功即输入注入

蓝牙键盘/鼠标走 HID 主机角色。一旦主机接受了 HID 设备，**该设备发送的按键事件与内置键盘没有区别**——没有「这是外接设备」的二次确认。

这就是 BLE HID 注入类研究的机制基础：攻击者不需要配对，利用的是「主机在特定状态下接受未经认证的 HID 连接」这一类实现缺陷。外部研究报告（2023 年 12 月公开的一批 BLE 键盘注入问题）常被概括为「无需配对的按键注入」，**但具体 CVE 与公告落点本轮未核实**（见未核实 D）。防御侧的可操作结论与编号无关：

- 关闭「可被新设备发现」的时间窗；
- 对高价值场景（锁屏解锁、支付确认）不依赖「输入来自可信外设」这一假设；
- 企业侧可用 MDM 策略限制 HID 类设备的绑定。

### 5.4 OBEX / OPP：文件投递面

经典蓝牙的文件传输（OPP）在现代 Android 上默认不是「静默接收」——需要用户确认接收，且多数机型要求先打开「通过蓝牙接收文件」的时间窗。但**这个确认框本身是社工面**：文件名、发送者名称都由对端控制。

自查：`dumpsys bluetooth_manager` 里看 OPP 服务状态；企业设备建议直接关闭 OBEX 接收。

### 5.5 GATT：BLE 的服务暴露面

BLE 外设通过 GATT 暴露服务与特征值。两个常见错误：

1. **把敏感操作放在无认证的特征值上。** GATT 的「加密需读/需写」属性（encrypted read/write）要由外设显式声明，未声明即明文可读写。
2. **把「连接成功」当成「已授权」。** BLE 连接本身不绑定用户身份，任何能连上的中心设备都能读写未加保护的特征值。

对做智能硬件的团队，这一节是最容易踩的：**手机端的配对界面给人一种「已经安全了」的错觉**，而实际保护强度完全取决于外设固件里的属性位。

### 5.6 地址与标识：两个不同的「MAC」

一条被反复引用的官方事实：`WifiInfo.getMacAddress()` 与 `BluetoothAdapter.getAddress()` 自 Android 6.0 起对第三方应用返回常量 `02:00:00:00:00:00`。

这解决的是「应用直接读 MAC」，**不解决「应用通过扫描结果推断设备」**。而 Android 10 官方行为变更把 MAC 随机化列为隐私类重大变更之一（每网络随机化的英文细则本轮未逐字取得，见未核实 E）。

结论要分两层说：**读 API 拿到的是掩码值（6.0 起）；连接侧用的是每网络随机 MAC（10 起）**。把两者混为一谈，就会得出「Android 10 之后 MAC 不可用于追踪」的过强结论。

---

## 6. Wi-Fi：配置即信任

### 6.1 开放网络与邪恶双胞胎

系统按 SSID + 安全类型匹配已保存网络。攻击者只要广播一个同名同类型的 AP，设备在自动连接策略允许时就会尝试关联——**这是协议层的设计，不是实现 bug**。

防御侧能做的：

- 关掉「自动连接开放网络」；
- 高价值网络用企业级 EAP 并固定 CA 证书；
- 不在保存列表里保留「同名但不认识」的配置。

### 6.2 企业 EAP：证书校验是最容易被跳过的环节

企业 Wi-Fi 的信任根是服务端证书。**若客户端不校验 CA，任何人都能搭一个同名 SSID 骗取域凭据**。这不是 Android 独有的问题，但 Android 的配置项粒度容易让人误以为「选了 EAP 就安全」：`Phase 2 authentication`、`CA certificate`、`Domain` 三项里，只有把 CA 指定为具体证书、并把 Domain 填成真实域名，才构成有效校验。

### 6.3 `wpa_supplicant`：C 代码、外部输入、高价值凭证

`wpa_supplicant` 解析对端发来的管理帧与 EAP 报文。官方公告中确有落在这里的已修复条目，例如 **CVE-2021-0466**，其参考路径逐字为 `frameworks/opt/net/wifi`，类型为信息泄露，严重级 High。注意：这条的代码路径是框架侧，不是 `external/wpa_supplicant_8`——**读公告时要看参考路径，不能只看标题里的 "Wi-Fi"**。

### 6.4 SoftAP：把手机变成攻击面

开热点意味着本机运行 `hostapd` 并提供 DHCP/DNS 服务，同时承担「谁可以连」的策略。三个常见配置错误：

- 使用弱口令或默认口令；
- 不限制客户端隔离，导致热点下的设备互相可达；
- 热点开启时同时保持「自动连接已知网络」，形成双面桥接。

### 6.5 随机 MAC：隐私与策略的冲突点

每网络随机 MAC 降低了跨场所追踪，但也让「基于 MAC 的白名单/家长控制」失效。企业网络若用 MAC 过滤，需要显式针对该网络关闭随机化。**这类配置冲突常被误读成「随机化有漏洞」**。

---

## 7. NFC：贴近即授权

### 7.1 前台调度与 Reader Mode

`NfcAdapter.enableReaderMode` 让前台应用优先接收标签事件，并可关闭平台默认的「听到提示音、弹出应用选择」流程。**打开 Reader Mode 时，你的应用成为标签数据的第一个解析者**——这正是 NDEF 解析类缺陷的落点。

### 7.2 NDEF：外部可控的结构化数据

NDEF 记录里有类型、长度、载荷，还有「分块（chunked）」标志。解析器最容易出错的三处：

1. **相信长度字段**：外部 NDEF 的长度与实际载荷不一致时，是否越界读；
2. **递归/嵌套记录**：某些记录类型允许承载另一段 NDEF，深度不设限；
3. **URI 前缀缩写**：NDEF 的 URI 记录用单字节前缀码替代常见前缀，展开后得到的 URL 是否被再次直接打开。

防御原则只有一条：**标签内容是「陌生人给的字符串」，不是「配置」**。任何「读到标签就执行某个动作」的设计都要先经过用户确认。

### 7.3 HCE 与 `HostApduService`：你成为被读的一方

卡片模拟把角色反过来：读卡器发 APDU，你的服务回 APDU。边界问题变成三个：

- **谁能选择到这个服务**：由系统里的「默认支付/卡片模拟服务」决定；
- **你的服务是否校验了读卡器**：APDU 层面没有「认证读卡器」的通用机制，敏感操作必须自己加（如令牌化、限额、二次确认）；
- **超时与取消**：`HostApduService` 的会话生命周期短，状态机写错会导致「上一笔的状态被下一笔复用」。

### 7.4 安全元件（SE）与 OMAPI

OMAPI 提供了访问嵌入式安全元件（eSE）的接口，权限是系统级的（`android.permission.SECURE_ELEMENT_PRIVILEGED_OPERATION` 一类），普通应用无法直接使用。**因此「NFC 支付被攻破」在绝大多数公开案例里不是 SE 被读穿，而是上层应用或令牌化服务的设计问题**。OMAPI 的具体类名与 SELinux 域本轮未逐字核验（见未核实 F），正文不给出断言。

### 7.5 Beam 已死，但配置还在

Android Beam 自 Android 10 起被弃用。自查时若在老设备或定制 ROM 上看到 `setNdefPushMessage` 仍有效，按「对端可推送 NDEF 到你」的旧模型评估。

---

## 8. 公告怎么读：三种表形态

无线类条目在 Android 安全公告里的呈现方式随年份变化，这直接影响「我到底该不该把它算成无线问题」：

- **2017 年形态**：按受影响子系统分节（`Media framework`、`System`、`Kernel` 等），表格只有 CVE / 类型 / 严重级，**没有 Component 列**。蓝牙修复以补丁路径呈现（如 `system/bt`），Wi-Fi 常见 `external/wpa_supplicant_8`，内核节有 `WLAN`。实读条目：`CVE-2017-0842 | EoP | High | system/bt`。
- **2021 年形态**：分 Framework / System / Kernel + 厂商节，表格含参考编号、类型、严重级、受影响版本。实读条目：`CVE-2021-0466 | frameworks/opt/net/wifi | 信息泄露 | High | Android 10`。
- **2023 年形态**：分 Framework / System / Kernel / 厂商（MediaTek、Unisoc、Qualcomm）/ 闭源组件，并出现两句固定句式：`Security patch levels of 2023-03-05 or later address all of these issues.` 与在野利用提示 `There are indications that CVE-2023-20963 may be under limited, targeted exploitation.` 蓝牙相关实读条目：`CVE-2023-20951`、`CVE-2023-20954`。

**读表三原则**：

1. **以参考路径为准判定组件**，不要以标题关键词为准；
2. **区分 AOSP 公告与 Pixel 更新公告**——有些蓝牙修复只出现在后者；
3. **区分「组件」与「子组件」**：同一份公告里 `Bluetooth` 与 `System` 可能同时出现，靠路径与子组件列区分。

---

## 9. 外部研究报告：怎么用、不能怎么用

无线领域最有影响力的几组公开研究，名称广为人知，但**在 Google 自有公告里的落点往往难以逐字对应**。本文的处理是：机制照讲，编号与公告位置一律标注未核实。

| 外部名称 | 机制（可讲） | 本文核验状态 |
|---|---|---|
| BlueBorne（2017） | 经典蓝牙在「可发现 + 未配对」状态下即可能被攻击，涉及多个实现层缺陷 | 公告落点未核实（未核实 D） |
| BlueFrag 类 | 蓝牙LE 相关解析/状态机缺陷导致邻近攻击 | 具体 CVE 与公告页未核实（未核实 D） |
| BLE HID 注入（2023） | 主机在特定状态下接受未认证 HID 连接，可注入按键 | 编号与公告落点未核实（未核实 D） |
| FragAttacks（2021） | Wi-Fi 分片与聚合的帧解析缺陷，影响多厂商 | 在 Google 公告中的落点未核实（未核实 D） |
| BIAS / KNOB / BrakTooth 等 | 蓝牙配对协议与厂商固件缺陷 | 无 Google/AOSP 自有来源命中（未核实 D） |

**为什么要把它们单列**：这些研究改变了行业的威胁模型（「物理邻近即可攻击」），值得在架构评审里体现；但把它们当成「Android 已确认漏洞编号」引用，会在合规文档里制造错误。手册的立场是——**机制可以讲，编号必须可核对**。

---

## 10. 自查清单

### 10.1 应用侧

1. 清单里是否声明了三个蓝牙运行时权限，且 `neverForLocation` 只加在真正不需要定位的扫描上。
2. 是否在后台持续扫描/广播；是否有停止条件。
3. 收到的 BLE 广播与 GATT 特征值是否当作不可信输入做了长度与范围校验。
4. NFC 读到的 NDEF 是否先展示、后执行；有没有「读到就跳转」的路径。
5. `HostApduService` 是否对敏感命令做了令牌化或二次确认。
6. Wi-Fi 相关功能是否依赖「扫描结果」推断位置，是否有对应权限与告知。

### 10.2 系统与设备侧

```bash
# 当前蓝牙/扫描状态
adb shell dumpsys bluetooth_manager | head -60

# 已配对设备（含设备类别，用于识别 HID）
adb shell dumpsys bluetooth_manager | grep -iA3 "Bonded devices"

# Wi-Fi 模块与版本（判断补丁走模块还是 OTA）
adb shell dumpsys package com.android.wifi | grep -E "versionName|codePath"

# Wi-Fi 三面的 HAL 版本
adb shell dumpsys android.hardware.wifi.IWifi/default 2>/dev/null | head -20

# 当前连接与随机 MAC 状态
adb shell dumpsys wifi | grep -iE "mMacAddress|randomized" | head -20

# NFC 服务与默认卡片模拟应用
adb shell dumpsys nfc | head -40
adb shell settings get secure nfc_payment_default_component
```

### 10.3 企业侧

1. 强制关闭 HID 类未授权设备绑定（策略而非用户自觉）。
2. 企业 Wi-Fi 固定 CA 与域名，禁止「不校验证书」的配置模板。
3. 关闭自动连接开放网络；对遗留开放网络做单独例外清单。
4. 记录并审计「默认卡片模拟应用」的变更。

---

## 11. 七类成因模式

1. **解析器把外部字节当作良构输入**（NDEF 长度、BLE 广播字段、EAP 报文）。
2. **协议状态机缺少「未认证态」分支**（HID 未配对注入、BLE 未加密特征值）。
3. **权限粒度与语义不匹配**（一个「附近设备」覆盖扫描/广播/连接；扫描与位置耦合）。
4. **信任继承**（配对成功即认为用户在场；连接成功即认为已授权）。
5. **配置继承**（同名 SSID 的旧配置覆盖新配置；EAP 模板漏填 CA）。
6. **固件与闭源实现不在 AOSP 视野内**（基带、Wi-Fi 固件、蓝牙控制器）。
7. **默认开启的暴露面**（BLE 广播、自动连接、NFC 屏幕点亮即可读）。

---

## 12. 误传纠正

- **「关掉 Wi-Fi 就断网了」**：蜂窝数据仍然可达；且部分设备会走 Wi-Fi 辅助扫描（即使 Wi-Fi 开关关闭，某些版本仍允许扫描，具体行为依版本与厂商实现）。
- **「蓝牙关了就不会被扫描到」**：取决于关闭的是「蓝牙」还是「可发现性」；BLE 广播与经典可发现是两套状态。
- **「Android 10 之后 MAC 不可追踪」**：读 API 的掩码值始于 Android 6.0；每网络随机化始于 Android 10；两者都不是「无法推断设备」。
- **「配对需要 PIN 所以是安全的」**：固定 PIN 只防误连；防中间人取决于配对模型（Just Works / Passkey / OOB）。
- **「NFC 需要用户点确认才能读」**：读标签没有运行时权限弹窗，贴近即读；需要确认的是卡片模拟的默认服务选择。
- **「HCE 可以替代安全元件」**：HCE 的密钥在应用/系统侧，SE 的密钥在独立芯片内；两者的威胁模型不同，不能互相替代。
- **「Wi-Fi 模块化意味着所有 Wi-Fi 漏洞都能热修」**：模块边界有版本差异（Android 11 时 Wi-Fi 服务仍在 system_server），且内核与固件侧的问题不在 APEX 内。
- **「BLE 扫描只要蓝牙权限」**：Android 12 起 `BLUETOOTH_SCAN` 是运行时权限，且与定位权限的关系由 `neverForLocation` 声明决定。
- **「声明 neverForLocation 没有副作用」**：官方明确写会过滤掉部分 BLE beacon。
- **「企业 Wi-Fi 选了 EAP 就安全」**：不固定 CA、不填域名，等于不校验服务端。
- **「NFC 攻击需要物理接触」**：读卡器与手机不接触也可在数厘米内通信，「贴近」是距离而不是接触。
- **「UWB 只是更精确的蓝牙」**：UWB 有独立模块、独立 HAL（FiRa UCI），安全属性不同。

---

## 13. 与相邻手册的边界

- 蜂窝与基带（`android-baseband-telephony.md`）：运营商网络侧、SIM/eSIM、基带固件；本文不涉及蜂窝空口。
- 内核与 GKI 驱动（`android-kernel-gki-drivers.md`）：驱动与 KMI 稳定性；本文只在 HAL 面提及固件边界。
- 供应链与构建完整性（`android-supply-chain-build-integrity.md`）：模块与产物签名；本文只讲模块边界对补丁路径的影响。
- 变体设备（`android-variants-automotive-tv.md`）：车机的蓝牙/Wi-Fi 使用形态；本文给出通用机制，不重复变体细节。
- 路径与 URI（`android-path-uri-handling.md`）：NDEF 展开出的 URI 若被直接当作文件路径或 Intent，属于那篇的范畴。

---

## 参考

- `https://source.android.com/docs/core/architecture/modular-system/wifi`
- `https://source.android.com/docs/core/architecture/modular-system/bluetooth`
- `https://source.android.com/docs/core/architecture/modular-system/tethering`
- `https://source.android.com/docs/core/architecture/modular-system/uwb`
- `https://source.android.com/docs/core/ota/modular-system/nfc-services`
- `https://source.android.com/docs/core/connect/wifi-hal`
- `https://source.android.com/docs/core/connect/wifi-overview`
- `https://android.googlesource.com/platform/hardware/interfaces/+/master/wifi/hostapd/1.0/IHostapd.hal`
- `https://android.googlesource.com/platform/packages/modules/Bluetooth/+/87b15809d6/system/stack/Android.bp`
- `https://developer.android.com/guide/topics/connectivity/bluetooth/permissions`
- `https://developer.android.com/about/versions/marshmallow/android-6.0-changes`
- `https://developer.android.com/about/versions/10/behavior-changes-all`
- `https://source.android.com/docs/security/bulletin/2017-11-01`
- `https://source.android.com/docs/security/bulletin/2021-05-01`
- `https://source.android.com/docs/security/bulletin/2023-03-01`

---

## 未核实与失效来源

- **A. 蓝牙栈的代号（Fluoride / Gabeldorsche）。** WebSearch 未命中任何 Google 官方文档或博客使用这两个词；本文因此不采用代号作为事实，只写「AOSP 蓝牙栈」。外部媒体的普遍说法不能替代官方口径。
- **B. `com.android.bt` 作为 AOSP 构建名。** 官方模块页只出现 `com.google.android.bt`；`com.android.bt` 推测为构建产物名，未逐字核实。
- **C. Android 13 `NEARBY_WIFI_DEVICES` 对扫描权限的确切口径。** 本轮未取到官方逐字句，正文只说「口径发生变化」，不断言具体映射。
- **D. 外部研究（BlueBorne / BlueFrag / BLE HID 注入 / FragAttacks / BIAS / KNOB / BrakTooth）在 Google 公告中的编号与页面。** 两次抓取 2017-11-01 与 2021-05-01 均疑因截断未命中对应 CVE；2023-03-05 为独立页时 404（2023 年起并入同月页）。正文只讲机制，编号一律不作事实引用。
- **E. 「每网络随机 MAC」的英文原句。** Android 10 行为变更页被返回为本地化版本，正文按「官方列表确认存在 MAC 随机化」转述，未逐字引用英文。
- **F. OMAPI 类名与 SELinux 域。** `android.se.omapi.SEService` 等实名未抓取，正文只讲模型。
- **G. `com.android.nfcservices` 的引入版本。** 官方页现文写「released in Android 16」，与社区记录的 Android 13/14 不一致，可能是页面更新或表格截断；正文按「实名确证、版本存疑」处理。
- **H. UCI 全称与报文格式。** 官方页只写「UCI specification defined by FiRa」，未展开全称；本文不给出展开式。

---

## 联系方式

如果你在企业里做设备或应用的无线侧安全评审，需要就「蓝牙配对与 HID 策略」「企业 Wi-Fi 证书校验」「NFC/NDEF 解析边界」做一次针对现网的核查，可以通过 Telegram 联系：<https://t.me/one00190>

---

*本手册仅用于防御与教育目的，讨论的均为公开且已被修复问题的机制与设计缺陷，不包含任何可直接利用的代码。文中标注「未核实」的内容不应作为事实引用；发现错误欢迎指出。*
