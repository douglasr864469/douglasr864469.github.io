---
layout: page
title: "Android 蓝牙与 BLE 配对、绑定与权限模型手册"
description: "把「配对就安全了」拆成五段：发现／连接／配对／绑定／授权访问，权限侧讲清 Android 12 三件套为什么是三种能力（`BLUETOOTH_SCAN`／`ADVERTISE`／`CONNECT`）、`neverForLocation` 是应用承诺且会让系统过滤部分 beacon、定位耦合看 `targ..."
date: 2026-11-02
permalink: /android-bluetooth-ble-pairing-permissions/
---

> 蓝牙的安全问题经常被一句「配对就安全了」带过。但配对解决的是「两台设备之间有没有共享密钥」，绑定解决的是「这把密钥有没有被存下来」，权限解决的是「哪个应用有权发起这些操作」——三件事分别发生在协议层、系统服务层和应用沙箱层，任何一层被误解，都会得出错误的结论。这份手册把「发现 → 连接 → 配对 → 绑定 → 授权访问」拆成五段，逐段说明 Android 的控制点、公开缺陷的落点，以及防御方真正能检查的东西。只讨论公开且已修复的问题，不含可利用代码。

## 0. 使用方式与边界

这份手册面向三类读者：写蓝牙应用的应用开发者、做蓝牙外设的固件与系统工程师、以及需要评估移动设备蓝牙暴露面的安全人员。三者的「安全边界」并不相同：

- **应用开发者**能控制的只有权限声明、配对发起、GATT 读写与本地数据处理；协议层的加密强度、外设的特征权限位，都不在应用手里。
- **外设工程师**控制配对模型、GATT 属性位、密钥存储；手机端的权限弹窗与外设无关。
- **安全评估者**需要同时看两端，并区分「协议允许什么」「系统实现强制什么」「应用实际检查什么」。

本文的三条自我约束：

1. 只引用公开且已修复的问题，CVE 描述以 Android Security Bulletin 与 MITRE 记录为准。
2. 不写利用步骤、不写 PoC、不写可复现的攻击参数。
3. 查不到一手来源的说法，一律进「未核实与失效来源」，不写进正文。

与已发布的《无线与近场通信攻击面手册》的分工：那一篇把 Wi-Fi / 蓝牙 / NFC / UWB 放在同一张桌子上比较「射频入口」；这一篇只处理蓝牙，并且深入到配对模型、绑定状态机、权限执行点与 GATT 安全属性。两篇可以对照读，但不要指望这一篇再讲 Wi-Fi 与 NFC。

## 1. 五件事：发现、连接、配对、绑定、授权

先把术语钉死。日常语言里「连上蓝牙」同时指了至少五件不同的事，而它们的信任含义完全不同。

| 阶段 | 协议层发生了什么 | Android 侧的控制点 | 失败时的典型后果 |
|------|------------------|--------------------|------------------|
| 发现 | 经典蓝牙的 inquiry / 可发现窗口；BLE 的 advertising 与 scanning | `BLUETOOTH_SCAN`（扫描方）、`BLUETOOTH_ADVERTISE`（广播方）、可发现性设置 | 设备被持续识别、地址被长期跟踪 |
| 连接 | 建立 ACL 链路；BLE 建立 LE 链路 | `BLUETOOTH_CONNECT`（对已配对设备）、GATT 客户端 API | 未授权读写未受保护的特征值 |
| 配对 | 交换并确认密钥材料（SSP 或 SMP） | 系统配对界面、`createBond()`、配对广播 | 中间人、降级到弱密钥 |
| 绑定 | 把长期密钥存下来，供下次免配对重连 | 系统蓝牙栈的 bond 数据库 | 解绑残留、bond 被冒用 |
| 授权访问 | 在已建立的链路上读写 GATT、走 HID、传文件 | 特征权限位 + 应用自身业务校验 | 把「连上了」当成「已授权」 |

这张表最重要的一行是最后一行。**协议层的「连接成功」从来不是「应用层已授权」**：BLE 连接只证明链路存在，GATT 特征是否可读、可写，取决于外设声明的属性位；而属性位之外的业务授权（这个用户能不能改这个配置）只能由应用或固件自己判断。

另一个常被忽略的区分是「配对」与「绑定」。Bluetooth SIG 的定义是：配对是产生共享密钥的机制，绑定是在配对之后交换并保存长期密钥、供以后使用。换句话说：

- **配对可以是一次性的**：本次连接加密，结束后密钥不留。
- **绑定一定包含配对**：先有密钥，再谈保存。
- **绑定的产物是本地数据库里的一条记录**，不是「这台设备永远可信」的证明。

Android 把绑定状态暴露成三个常量：`BOND_NONE`（10）、`BONDING`（11）、`BONDED`（12）。参考文档对 `BOND_BONDED` 的说明是「link key is still stored locally」——注意主语是「本地存储的链路密钥」，不是「对端当前在线」，也不是「用户已认证」。

## 2. 权限模型：三个运行时权限与它们的边界

### 2.1 Android 12 起的三件套

Android 12（API 31）把蓝牙从「安装即授予」的普通权限，改成了三个运行时权限。三者的保护级别都是 `dangerous`，在系统弹窗里统一显示为「附近的设备（Nearby devices）」。

| 权限 | 官方描述覆盖的操作 | 典型 API | 被拒绝后的表现 |
|------|--------------------|----------|----------------|
| `BLUETOOTH_SCAN` | 发现并配对周边设备 | `BluetoothLeScanner.startScan()`、`BluetoothAdapter.startDiscovery()` | 扫描启动失败或返回空结果 |
| `BLUETOOTH_ADVERTISE` | 让本机可被周边设备发现 | `BluetoothLeAdvertiser.startAdvertising()` | 广播启动失败 |
| `BLUETOOTH_CONNECT` | 与已配对设备通信 | `getBondedDevices()`、`createBond()`、`getName()`、GATT 读写、`getProfileConnectionState()` | 调用抛 `SecurityException` 或返回空 |

这三条不是「蓝牙权限」的三个别名，而是三种**性质不同的能力**：

- 扫描是**读取周边**，隐私敏感点在「能推断你在哪」；
- 广播是**暴露自己**，隐私敏感点在「别人能长期识别你」；
- 连接是**操作已建立的信任关系**，敏感点在「能读写别人的设备数据」。

把它们合成一个开关，是 Android 12 之前的设计；分开之后，应用可以只申请「连接我的手环」，而不申请「扫描整栋楼的 beacon」。

### 2.2 旧权限：`maxSdkVersion` 是给旧系统留的门

`BLUETOOTH` 与 `BLUETOOTH_ADMIN` 仍然存在，保护级别是 `normal`（安装即授予）。官方对迁移的建议是给这两条旧声明加上 `android:maxSdkVersion="30"`：

```xml
<uses-permission android:name="android.permission.BLUETOOTH" android:maxSdkVersion="30" />
<uses-permission android:name="android.permission.BLUETOOTH_ADMIN" android:maxSdkVersion="30" />
```

语义是：在 Android 11 及以下使用旧权限，在 Android 12 及以上只申请新权限。AOSP 的 `@RequiresLegacyBluetoothPermission` 注解也明确写了适用对象是「targeting R or lower」的应用；Android 12 起新增的蓝牙 API 一律标注 `@RequiresPermission(BLUETOOTH_CONNECT)` 或对应的新权限。

一个容易踩的坑：**旧权限在 Android 12+ 上并没有「消失」，只是不再能放行新 API**。如果应用同时声明了新旧权限，在 Android 12+ 上系统会按新权限模型弹窗；如果只声明旧权限且 `targetSdk ≥ 31`，调用新 API 会失败。官方的表述是 `maxSdkVersion` 帮系统「只授予应用真正需要的蓝牙权限」，并没有逐字写「旧权限失效」，本文按行为描述而不做绝对断言。

### 2.3 `neverForLocation`：一个承诺，不是一个保证

`BLUETOOTH_SCAN` 上可以加一个清单属性：

```xml
<uses-permission
    android:name="android.permission.BLUETOOTH_SCAN"
    android:usesPermissionFlags="neverForLocation" />
```

官方对它的定义是：应用**强断言**自己不会用蓝牙权限推导物理位置。加上它之后，应用在 Android 12+ 不再需要 `ACCESS_FINE_LOCATION` 就能扫描。

官方同时给出一条非常容易忽略的副作用，原文是：「If you include `neverForLocation` in your `usesPermissionFlags`, the system filters out some BLE beacons from your scan results.」——**声明免定位会让系统从扫描结果里过滤掉一部分 BLE beacon**。所以当应用报告「扫不到某些 beacon」时，第一件事是检查这条声明，而不是先怀疑硬件或权限被拒。

安全含义要分两层看：

- 对系统而言，这是一个**应用自证**：系统据此不再要求定位权限，但无法验证应用拿到扫描结果后到底做了什么。
- 对用户而言，**带定位权限做扫描**仍然可以反推位置——这正是 Android 12 之前蓝牙权限长期与位置权限绑定的原因。

### 2.4 定位耦合的分界：`targetSdk` 决定规则

官方文档把分界写得很清楚：

- `targetSdk ≤ 30`：蓝牙扫描必须持有 `ACCESS_FINE_LOCATION`。原因是「on Android 11 and lower, a Bluetooth scan could potentially be used to gather information about the location of the user」。
- `targetSdk ≥ 31`：只有在**用扫描结果推导位置**时才需要 `ACCESS_FINE_LOCATION`；否则用 `neverForLocation` 声明即可。

这里有两个常见误读：

1. 「Android 12 之后扫描完全不需要定位」——不对。规则看的是 `targetSdk` 与实际用途，不是设备版本。一个 `targetSdk 30` 的应用跑在 Android 14 手机上，仍然要定位权限。
2. 「加了 `neverForLocation` 就绝对拿不到位置」——不对。它约束的是权限模型，不是数据本身；应用仍可能通过其他传感器或网络信息推断位置。

### 2.5 权限在哪里被执行

Android 的蓝牙调用不是「应用直接和芯片说话」，中间隔着至少三层：

```
应用进程
  └─ android.bluetooth.* API（框架，@RequiresPermission 注解）
       └─ Binder → 蓝牙系统进程（packages/modules/Bluetooth/android/app）
            └─ JNI → 原生蓝牙栈（packages/modules/Bluetooth/system）
                 └─ HAL（android.hardware.bluetooth，HIDL 1.0/1.1 与 AIDL 并存）
                      └─ 控制器与射频
```

权限检查发生在**框架层与系统服务层**：`@RequiresPermission` 是编译期提示，真正的强制在系统进程的 `enforceCallingOrSelfPermission` 一类调用里。AOSP 中以下方法都标注了 `@RequiresPermission(BLUETOOTH_CONNECT)`：

- `BluetoothAdapter.getBondedDevices()`
- `BluetoothDevice.createBond()`
- `BluetoothDevice.getName()` / `getType()` / `getBondState()`
- `BluetoothGatt.readCharacteristic()` / `writeCharacteristic()`

Android 14 起，`BluetoothAdapter.getProfileConnectionState()` 也开始强制 `BLUETOOTH_CONNECT`（在此之前部分调用不强制）。这类「同一 API 在不同版本要求不同」的变化，是跨版本自查时最容易漏掉的一类。

### 2.6 后台与前台服务：`connectedDevice`

Android 14（`targetSdk 34`）起，每个前台服务都必须声明类型。蓝牙相关的是 `connectedDevice`，对应权限 `FOREGROUND_SERVICE_CONNECTED_DEVICE`，并且要求应用已经持有 `BLUETOOTH_CONNECT`、`BLUETOOTH_SCAN`、`BLUETOOTH_ADVERTISE` 中的至少一个。

这条限制的实际意义：**「保持蓝牙连接」不再是一个可以无限期在后台跑的理由**。应用要么把连接工作放进合规的前台服务，要么接受系统在后台收紧调度。对安全评估者来说，一个长期驻留、持有 `BLUETOOTH_CONNECT` 的后台服务，本身就是需要重点看的对象。

### 2.7 Companion Device Manager：代扫不等于免权限

`CompanionDeviceManager`（CDM）提供了一条「陪绑设备」的路径：应用通过 `AssociationRequest` 声明自己关心的设备类型，由系统代扫并让用户选择。`AssociationRequest` 支持三种过滤器：

- `BluetoothDeviceFilter`（经典蓝牙）
- `BluetoothLeDeviceFilter`（BLE）
- `WifiDeviceFilter`（Wi-Fi）

官方说明里有两个关键点：

1. CDM 代扫**不需要 `ACCESS_FINE_LOCATION`**，但**要求定位服务处于开启状态**，并且需要用户同意。
2. 用户选中设备后，应用调用 `createBond()` 仍然受 `BLUETOOTH_CONNECT` 约束。

所以「用 CDM 就能绕过蓝牙权限」是错的。CDM 绕过的是**扫描侧的定位耦合**，不是连接侧的权限。

Android 16（`targetSdk 36`）给 CDM 补了一个公开的解绑入口：对已由 CDM 管理的关联设备，应用可以调用 `removeBond(int)` 触发解绑，并通过 `ACTION_BOND_STATE_CHANGED` 观察状态变化。官方原文是「All apps targeting Android 16 are now able to unpair bluetooth devices using a public API in CompanionDeviceManager」。这填补了此前「普通应用没有公开解绑 API」的空白。

### 2.8 权限组的真相

三条蓝牙权限在系统弹窗里同属「附近的设备」组。AOSP 的 `AndroidManifest.xml` 中，这三条 `<permission>` 的 `permissionGroup` 属性在多个分支上是 `UNDEFINED`，实际分组由 `PermissionManagerServiceImpl` 里的 `NEARBY_DEVICES_PERMISSIONS` 列表维护。

这意味着：

- **弹窗按组呈现**是确定的行为；
- **运行时是否「一次授权、三条全给」**没有官方明文。不同版本、不同厂商的权限控制器可能有差异。

本文只写「同组弹窗」，不写「一次授予三条」，后者列入未核实。

## 3. 配对与绑定：四种模型、两代协议、一套状态机

### 3.1 四种配对模型

经典蓝牙的 Secure Simple Pairing（SSP）有四种关联模型（association model）。它们的安全性差别不在「加密算法」，而在**密钥确认过程能不能被中间人利用**。

| 模型 | 用户看到什么 | 是否提供 MITM 保护 | 适用场景 |
|------|--------------|--------------------|----------|
| Just Works | 无输入、无确认（或只有一个「配对」按钮） | **否** | 无输入输出能力的外设 |
| Numeric Comparison | 两端显示 6 位数字，用户确认是否一致 | **是** | 两端都有显示屏 |
| Passkey Entry | 一端显示 6 位数字，另一端输入 | **是** | 一端有屏、一端有键盘 |
| Out of Band（OOB） | 通过 NFC 等旁路交换配对信息 | **是**（前提是旁路本身可信） | 带 NFC 的设备 |

NIST 对 Just Works 的表述很直接：「Just Works … does not provide MITM protection … results in an unauthenticated link key」。Bluetooth Core 规范在 LE Secure Connections 一章也写明：认证式 MITM 保护来自 Passkey Entry、Numeric Comparison 或 OOB；「Unauthenticated no MITM Protection does not have protection against MITM attacks」。

所以评审时不要看「有没有配对弹窗」，要看**弹窗让用户确认了什么**：

- 只让点「配对」→ 大概率是 Just Works，不防中间人。
- 让用户比对两端数字 → Numeric Comparison，防中间人。
- 让用户输入数字 → Passkey Entry，防中间人。
- 什么都不显示、自动完成 → 需要确认是不是 OOB 或厂商自定义。

### 3.2 常量与模型不是一一对应

Android 把配对界面抽象成一组 `BluetoothDevice.PAIRING_VARIANT_*` 常量：

| 常量 | 值 | 含义 |
|------|----|------|
| `PAIRING_VARIANT_PIN` | 0 | 输入 PIN |
| `PAIRING_VARIANT_PASSKEY` | 1 | 输入 passkey |
| `PAIRING_VARIANT_PASSKEY_CONFIRMATION` | 2 | 确认 passkey |
| `PAIRING_VARIANT_CONSENT` | 3 | 仅确认 |
| `PAIRING_VARIANT_DISPLAY_PASSKEY` | 4 | 显示 passkey |
| `PAIRING_VARIANT_DISPLAY_PIN` | 5 | 显示 PIN |
| `PAIRING_VARIANT_OOB_CONSENT` | 6 | OOB 确认 |

这组常量描述的是**界面变体**，不是协议模型。例如 `DISPLAY_PASSKEY` 与 `PASSKEY` 可能对应同一次 Passkey Entry 的两端；`CONSENT` 可能对应 Just Works 或 OOB。把它们当成「协议模型枚举」会得出错误的 MITM 结论。

补充两点：`PASSKEY`、`CONSENT`、`DISPLAY_PASSKEY`、`DISPLAY_PIN`、`OOB_CONSENT` 在 AOSP 中标注为 `@SystemApi`；在 Android 13–16 的分支上，这些常量**没有**被标 `@Deprecated`。网络上「这些常量已废弃」的说法，至少在所查分支上不成立。

### 3.3 两代 LE 配对：Legacy 与 Secure Connections

BLE 的配对有两代：

| 维度 | LE Legacy Pairing | LE Secure Connections |
|------|-------------------|------------------------|
| 引入版本 | Core 4.0 / 4.1 | Core 4.2 |
| 密钥交换 | 基于 STK（短期密钥） | 基于 ECDH（P-256），生成 LTK |
| 关联模型 | Just Works、Passkey Entry、OOB | 增加 Numeric Comparison |
| 抗窃听 | 较弱（依赖临时密钥） | 更强（长期密钥不直接上线） |
| Android 侧 API | 无独立开关，由栈与对端协商 | 同上 |

对应用开发者来说，这两代的差别通常不可见——**协商由协议栈完成，应用只能看到「配对成功/失败」**。这也意味着：应用无法从 `BOND_BONDED` 推断出到底用了哪一代、哪个关联模型、有没有 MITM 保护。要回答这些问题，必须看协议层日志或外设侧配置。

### 3.4 三种长期密钥：LTK、IRK、CSRK

绑定之后保存下来的不只是一把密钥，而是三种用途不同的材料：

| 密钥 | 用途 | 谁持有 | Android 是否公开 |
|------|------|--------|------------------|
| LTK（Long Term Key） | 链路加密 | 两端 | 否 |
| IRK（Identity Resolving Key） | 解析对端的可解析私有地址 | 分发方持有自己的 IRK，对端持有用于解析的副本 | 否 |
| CSRK（Connection Signature Resolving Key） | 对无连接数据进行签名与验签 | 两端 | 否 |

三者的共同点：**都在系统蓝牙栈里，应用层拿不到**。Android 的公开 SDK 没有「读取本机 IRK」的 API；IRK 由蓝牙栈持有，并在原生层以 `BTIF_DM_LE_LOCAL_KEY_IRK` 一类标识落盘，控制器侧维护 resolving list（`btm_ble_privacy.cc`）。

这条事实有两个推论：

1. 应用无法自己实现「按 IRK 解析地址」——那是栈的职责。
2. 任何声称「通过 Android API 读取 IRK」的教程都值得怀疑；本文未找到这样的公开 API。

### 3.5 绑定状态的 API 与权限

| API | 公开性 | 所需权限 | 说明 |
|-----|--------|----------|------|
| `createBond()` | public | `BLUETOOTH_CONNECT` | 发起配对；是否弹窗取决于设备与配对模型 |
| `createBond(int transport)` | `@SystemApi` | `BLUETOOTH_CONNECT` | 指定传输（经典/LE） |
| `createBondOutOfBand()` | `@SystemApi` | `BLUETOOTH_CONNECT` | OOB 配对 |
| `setPairingConfirmation(boolean)` | public，但需 `BLUETOOTH_PRIVILEGED` | `BLUETOOTH_CONNECT` + `BLUETOOTH_PRIVILEGED` | 普通应用不可用；`BLUETOOTH_PRIVILEGED` 是 `signature|privileged` |
| `setPin(byte[])` | public | `BLUETOOTH_CONNECT` | 参考文档已标注弃用 |
| `cancelBondProcess()` | `@SystemApi` | `BLUETOOTH_CONNECT` + `BLUETOOTH_PRIVILEGED` | 取消进行中的配对 |
| `removeBond()` | `@SystemApi` | 见未核实 | 删除本地绑定 |
| `getBondState()` | public | `BLUETOOTH_CONNECT` | 返回 `BOND_*` |
| `getBondedDevices()` | public | `BLUETOOTH_CONNECT` | 返回已绑定设备集合 |

表中「public 但需 `BLUETOOTH_PRIVILEGED`」这一行最容易被误读：方法签名是公开的，但权限是签名级，**普通应用调用会直接失败**。判断一个 API 能不能用，不能只看它是不是 `public`。

### 3.6 `ACTION_PAIRING_REQUEST`：能收，不能伪造

配对过程中，系统蓝牙进程会发出 `ACTION_PAIRING_REQUEST` 广播，携带 `EXTRA_PAIRING_VARIANT`（配对方式）与 `EXTRA_PAIRING_KEY`（6 位 passkey，0–999999）。

两个关键事实：

1. 它是 `<protected-broadcast>`，**只有系统蓝牙进程能发**。普通应用无法伪造一条配对请求。
2. 它以 `BLUETOOTH_CONNECT` 作为接收权限发送有序广播。持有该权限并注册接收器的应用**可以收到**，但收到不等于能改变配对结果。

对防御方的意义：如果某个应用注册了这个广播，它能看到配对事件与 passkey——这本身是一个信息暴露面，但不构成「伪造配对」的能力。审计时应关注「谁在监听」，而不是假设「谁能发送」。

### 3.7 状态机与竞态

配对是一个异步状态机。应用能观察到的只有 `ACTION_BOND_STATE_CHANGED` 与 `getBondState()` 的返回值，而**状态变化与用户操作、链路事件之间没有原子性保证**。常见的实现问题包括：

- 在 `BOND_BONDING` 期间重复调用 `createBond()`；
- 把 `BOND_NONE` 当成「从未配对」而不是「当前无本地密钥」；
- 在解绑后继续使用缓存的 `BluetoothDevice` 对象发起 GATT 连接；
- 忽略对端主动删除绑定导致的加密失败。

这些都不是协议漏洞，而是**应用把异步状态当同步事实**造成的逻辑缺陷。它们通常不会被分配 CVE，但在真实产品里比协议漏洞更常见。

### 3.8 Android 16 的 bond loss 变更

Android 16 改写了「对端丢失绑定」的处理方式。官方原文：

> If a previously bonded Bluetooth device could not be authenticated upon reconnection, the system will disconnect the link, retain local bond information, and display a system dialog informing users of the bond loss and directing them to re-pair.

变更前：系统会自动删除绑定并重新发起配对，可能导致「非预期的重新配对」。变更后：**保留本地绑定记录、断开链路、弹窗提示用户重新配对**。

这条变更的安全含义常被读反：

- 它不是「自动修复信任」，而是「停止自动重建信任」；
- 它把决定权交回用户，同时保留了本地记录，便于审计「这台设备曾经绑定过」；
- 对应用而言，`ACTION_BOND_STATE_CHANGED` 仍会到达，但**不能假设系统会自动把状态推回 `BOND_NONE`**。应用需要自己处理「绑定仍在、但对端已不可认证」的中间态。

### 3.9 解绑的残留问题

`removeBond()` 删除的是**本地**的链路密钥，并断开需要认证或加密的连接。官方文档没有承诺「对端也会删除密钥」，SIG 规范中 unbond 的对端语义本文也未取到可引用原文（见未核实）。

现实中的残留形态：

- 手机删了绑定，耳机仍保存着旧密钥，下次连接时认证失败；
- 耳机删了绑定，手机仍显示「已配对」，发起连接后加密失败；
- 双方都删了，但某一方的 UI 缓存仍显示历史设备。

Android 用 `ACTION_KEY_MISSING` 广播表达「对端删除了绑定，本端加密失败」这一类事件。应用如果只监听 `ACTION_BOND_STATE_CHANGED`，可能完全看不到这个中间态。

## 4. 地址与隐私：两个「MAC」不是一回事

### 4.1 经典地址与 BLE 地址

经典蓝牙用 BD_ADDR（48 位），BLE 也用 48 位地址，但 BLE 的地址有类型之分：

| 类型 | 含义 | 是否可长期跟踪 |
|------|------|----------------|
| Public | 厂商分配的公开地址 | 是 |
| Random Static | 随机固定地址，设备生命周期内不变 | 是（设备生命周期内） |
| Random Private Resolvable（RPA） | 由 IRK 参与生成，可被持有 IRK 的对端解析 | 否（对无 IRK 者） |
| Random Private Non-Resolvable | 完全随机，不可解析 | 否 |

「BLE MAC 是固定硬件地址」是错的。RPA 会周期性变化，只有持有 IRK 的对端才能把不同 RPA 关联到同一设备。

### 4.2 RPA 的解析链

RPA 由两部分构成：24 位的 `prand` 与 24 位的哈希 `ah(IRK, prand)`。持有 IRK 的一方可以对收到的地址做同样的哈希，从而判断「这个随机地址是不是我绑定的那台设备」。

这条链的三个节点：

1. **IRK 在配对阶段分发**（SMP 的 Identity Information 交换）；
2. **对端保存 IRK 的副本**，用于解析；
3. **本机地址周期性轮换**，轮换策略由控制器与栈决定。

所以「绑定」不只是保存一把加密密钥，还保存了「如何认出对方」的能力。删除绑定，等于同时删除加密能力与识别能力。

### 4.3 Android 侧能看到什么

应用层能拿到的地址，取决于 API：

- `BluetoothDevice.getAddress()` 返回的是当前地址；
- BLE 扫描结果里的地址可能是 RPA，且会变化；
- 应用**没有**公开 API 读取本机 IRK，也没有 API 主动「解析」一个 RPA。

这意味着应用侧无法自己实现「稳定设备标识」。如果产品需要跨会话识别同一台 BLE 设备，正确做法是使用绑定关系或应用层自己的标识，而不是把 MAC 当主键。

## 5. GATT：属性位、安全位与「谁在强制」

### 5.1 `PROPERTY_*` 与 `PERMISSION_*` 是两个层面

`BluetoothGattCharacteristic` 有两组容易混淆的常量：

| 组 | 例子 | 层面 | 谁看 |
|----|------|------|------|
| `PROPERTY_*` | `PROPERTY_READ`（0x02）、`PROPERTY_WRITE`（0x08）、`PROPERTY_NOTIFY` | 特征值自身支持哪些操作 | GATT 客户端（对端） |
| `PERMISSION_*` | `PERMISSION_READ`、`PERMISSION_WRITE`、`PERMISSION_READ_ENCRYPTED`、`PERMISSION_READ_ENCRYPTED_MITM` | 本地服务端允许怎么访问 | GATT 服务端（本机栈） |

一句话：**`PROPERTY_*` 是「我能做什么」的广告，`PERMISSION_*` 是「你必须满足什么条件才能做」的门禁**。

### 5.2 encrypted 与 encrypted-MITM 的差别

| 常量 | 值 | 官方描述 | 实际含义 |
|------|----|----------|----------|
| `PERMISSION_READ` | 0x01 | 允许读 | 明文可读 |
| `PERMISSION_READ_ENCRYPTED` | 0x02 | Allow encrypted read operations | 需要加密链路，但**不要求** MITM 保护 |
| `PERMISSION_READ_ENCRYPTED_MITM` | 0x04 | Allow reading with person-in-the-middle protection | 需要**认证过的**加密链路 |
| `PERMISSION_WRITE_ENCRYPTED` | 0x10 | Allow encrypted write operations | 同读 |
| `PERMISSION_WRITE_ENCRYPTED_MITM` | 0x40 | Allow write with MITM protection | 同读 |

这张表解释了一个常见误判：**「encrypted」不等于「authenticated」**。Just Works 配对也能产生加密链路，但它没有 MITM 保护；只声明 `PERMISSION_READ_ENCRYPTED` 的特征值，对 Just Works 连接是开放的。要挡住中间人，必须用 `_MITM` 变体，并且配对模型本身要支持 MITM。

### 5.3 服务端到底强制不强制

Android 作为 GATT 服务端时，安全位**会被原生栈执行**。AOSP 的 `gatt_sr.cc` 在读、写路径上分别调用 `gatts_read_attr_perm_check` 与 `gatts_write_attr_perm_check`，并传入当前链路的安全标志与密钥长度，据此返回「加密不足」「MITM 不足」一类错误。

但要注意边界：

- **栈执行的是安全位**（加密、MITM、密钥长度）；
- **业务授权不在栈里**。例如「这个特征值只有管理员能写」，栈不知道谁是管理员，必须由应用自己判断。

本文没有找到官方文档逐字写出「业务授权由应用自己检查」，这条结论是从「栈只接收 sec_flag 与 key_size」这一实现事实推导的，属于**机制推断**，已在未核实中说明。

### 5.4 客户端侧：返回值不等于结果

`BluetoothGatt.writeCharacteristic()` 的返回值语义是「写操作是否**成功发起**」，不是「写成功」。真正的结果在 `onCharacteristicWrite()` 回调里，状态码可能是成功、失败、或者根本没回来（超时）。

同理：

- `readCharacteristic()` 返回 `true` 只表示请求已发出；
- `requestMtu()` 返回 `true` 只表示 MTU 协商已发起，结果在 `onMtuChanged()`；
- Android 14 起，首个客户端 `requestMtu()` 会直接请求 517，并「disregards all subsequent MTU requests on that ACL connection」。

把「返回 true」当成「操作成功」，是蓝牙应用最常见的逻辑缺陷之一。它不会产生 CVE，但会产生「界面显示已同步、设备实际没收到」这类生产事故。

### 5.5 两个已修复的解析器缺陷

GATT 与 SDP 的解析器长期是蓝牙栈的高危区。两个有公告支撑的例子：

- **CVE-2023-20951**：`gatt_cl.cc` 缺少边界检查，远程（邻近）攻击者可造成越界写，CVSS 9.8，公告定级 Critical，影响 Android 11–13。
- **CVE-2024-43096**：`gatt_sr.cc` 的 `build_read_multi_rsp` 存在缺陷，公告定级 RCE / Critical，影响 Android 12–15，补丁落在 `packages/modules/Bluetooth`。

这两个例子的共同点：**输入来自射频、解析在原生栈、后果是内存破坏**。它们与「配对是否加密」无关——攻击者不需要绑定，只需要能让目标收到构造过的 GATT/SDP 数据。这也说明为什么「配对」不能作为蓝牙攻击面的唯一边界。

## 6. HID：配对成功即输入注入

### 6.1 HID 主机的信任模型

蓝牙键盘、鼠标走 HID（Human Interface Device）协议。手机作为 HID Host，一旦接受了某个 HID 设备，**该设备发来的按键事件与内置键盘没有区别**：系统不会在每次按键时再问「这是外接设备，你确定吗」。

这是设计使然，不是缺陷。但它意味着：

- 「配对成功」在 HID 场景下等价于「获得了输入注入能力」；
- 输入注入可以解锁、可以点「允许」、可以在用户不知情时完成操作；
- 因此 HID 设备的绑定记录，安全等级应当按「可信输入设备」而不是「普通外设」来管理。

### 6.2 CVE-2023-45866：配对确认缺失

2023 年 12 月的 Android 安全公告在 `System` 节列出一条 EoP / Critical：

| 项目 | 内容 |
|------|------|
| CVE | CVE-2023-45866 |
| 公告 | 2023-12 |
| 类型 | EoP |
| 严重级 | Critical |
| 影响版本 | Android 11、12、12L、13、14 |
| 补丁位置 | `packages/modules/Bluetooth` |
| 利用前提 | 邻近（remote/proximal），无需用户交互 |

公告对该节的描述是「remote (proximal/adjacent) escalation of privilege with no additional execution privileges needed. User interaction is not needed for exploitation.」

这条漏洞的机制落在 **HID Host 的配对确认环节**：在特定条件下，未经用户确认即可完成与伪键盘的配对，随后注入按键。研究者公开的描述里，「无需配对」这个说法需要加限定：

- Android 侧需要蓝牙处于开启状态；
- 其他平台的条件不同（例如 Linux 需要可发现/可连接，iOS/macOS 需要已配对的特定键盘）；
- MITRE 对该编号的描述写的是 BlueZ HID Host 的缺陷，与 Android 公告的表述存在口径差异。

所以更准确的说法是：**「无需用户确认即可完成配对」**，而不是「完全不需要配对」。

### 6.3 防御侧的可操作结论

与 CVE 编号无关，HID 场景的加固点很明确：

1. **缩短可发现窗口**：只在配对新设备时打开可发现性，配对完成后立即关闭。
2. **审查 HID 绑定列表**：定期检查「哪些设备有键盘能力」，尤其是用户不认识的设备。
3. **企业侧限制**：通过设备管理策略限制外接 HID，或要求配对走受控流程。
4. **区分「已绑定」与「当前连接」**：绑定列表里的历史设备不等于当前在线，但每一个都曾经拥有输入能力。

## 7. 公告台账与读法

### 7.1 表形态：没有 Component 列

Android Security Bulletin 的月度漏洞表，表头是：

```
CVE | References | Type | Severity | Updated AOSP versions
```

**没有 Component 列**。组件信息以两种方式出现：

- 小节标题：`Framework`、`System`、`Media Framework`、`Kernel components`；
- `References` 里的补丁路径：`system/bt`（旧）或 `packages/modules/Bluetooth`（新）。

蓝牙条目一律落在 `System` 节。闭源厂商（Qualcomm、MediaTek、Unisoc 等）的表才有 `Component` 列，那是另一套表。

这条事实的实用价值：**不能靠「搜 Component 列」来筛蓝牙漏洞**。正确做法是看 `References` 的补丁路径，或者按 CVE 描述里的文件名（`gatt_cl.cc`、`sdp_db.cc`、`btif_rc.cc`）反查。

### 7.2 Remote 与 Proximal：蓝牙算远程

官方定义：

- Remote：「the bug can be exploited without installing an app or without physical access」；
- Proximal：「Proximal attack vectors are considered remote」，例子就是「malformed Wi-Fi or Bluetooth packets」。

所以蓝牙漏洞在公告里按**远程**计，不会因为「需要靠近」而降级。定级看的是后果：能远程执行代码的通常是 Critical，只能信息泄露或提权的通常是 High。

### 7.3 台账（逐条回到公告核对）

| CVE | 公告 | 类型 | 严重级 | 影响版本 | 机制落点 | 前提 |
|-----|------|------|--------|----------|----------|------|
| CVE-2017-0781 | 2017-09 | RCE | Critical | 4.4.4–8.0 | `system/bt` | 邻近 |
| CVE-2020-0022 | 2020-02 | RCE / DoS | Critical（8.0/8.1/9）、Moderate（10） | 8.0–10 | `system/bt`，`packet_fragmenter.cc` 越界写 | 邻近、无需配对、需知道地址 |
| CVE-2020-0023 | 2020-02 | ID | Critical | 10 | `packages/apps/Bluetooth`，`AdapterService.setPhonebookAccessPermission` 权限检查缺失 | 邻近/已连接 |
| CVE-2020-26555 | 2021-06 | EoP | High | 8.1–11 | 传统 BR/EDR PIN 配对，BD_ADDR 冒充 | 邻近、无需 PIN |
| CVE-2021-0507 | 2021-06 | RCE | Critical | 8.1–11 | `system/bt`，`btif_rc.cc` | 邻近 |
| CVE-2023-20951 | 2023-03 | RCE | Critical（CVSS 9.8） | 11–13 | `gatt_cl.cc` 缺边界检查 | 邻近、无 UI |
| CVE-2023-20954 | 2023-03 | RCE | Critical（CVSS 9.8） | 11–13 | `sdp_db.cc` 越界写 | 邻近、无 UI |
| CVE-2023-40078 | 2023-12 | EoP | High | 14 | `a2dp_vendor_opus_decoder.cc` 堆溢出 | **已配对设备** |
| CVE-2023-40088 | 2023-12 | RCE | Critical | 11–14 | 回调线程 UAF | 邻近 |
| CVE-2023-45866 | 2023-12 | EoP | Critical | 11–14 | HID Host 配对确认缺失 | 邻近、蓝牙开启 |
| CVE-2024-43096 | 2025-01 | RCE | Critical | 12–15 | `gatt_sr.cc`，`build_read_multi_rsp` | 邻近 |

读这张表要注意三件事：

1. **不是所有蓝牙漏洞都要求配对**。表中多数条目的前提是「邻近」，`CVE-2023-40078` 明确写的是「paired device」。
2. **不是所有条目都在 `system/bt`**。`CVE-2020-0023` 在 `packages/apps/Bluetooth`，属于框架/应用层；`CVE-2024-43096` 在 `packages/modules/Bluetooth`。
3. **类型列决定防御重点**。RCE 要防内存破坏与解析器边界；ID 要防权限检查缺失；EoP 要防信任边界被绕过。

### 7.4 台账不能说明什么

- 它**不是**「Android 蓝牙漏洞全清单」。公告只覆盖 AOSP 与部分厂商组件，闭源固件、外设侧、协议本身的缺陷不在其中。
- 它**不能**用来判断某台设备是否安全。设备是否包含补丁，取决于厂商的 SPL（Security Patch Level）与是否回移。
- 它**不能**证明「配对是安全的」或「配对是不安全的」。CVE 描述的是具体缺陷，不是协议模型的整体结论。

## 8. 成因模式：六种

把台账与协议事实放在一起，可以归纳出六种反复出现的成因模式。

### 8.1 解析器边界检查缺失

**表现**：越界读/写、整数溢出、堆溢出。

**落点**：`gatt_cl.cc`、`gatt_sr.cc`、`sdp_db.cc`、`packet_fragmenter.cc`、A2DP 解码器。

**为什么反复出现**：蓝牙协议字段长度由对端控制，解析器要在「不可信长度」与「固定缓冲区」之间做检查；这类代码路径多、历史长、测试覆盖不均。

**防御**：及时更新 SPL；对自研固件做结构化模糊测试；不要把「协议格式固定」当成「输入可信」。

### 8.2 权限检查缺失

**表现**：某个系统 API 没有 `enforceCallingPermission`，导致低权限应用能读不该读的数据。

**落点**：`AdapterService.setPhonebookAccessPermission`（CVE-2020-0023）。

**为什么反复出现**：系统服务的方法多，权限注解是「声明式」的，漏一个就少一道门；而且这类缺陷往往不涉及内存，静态扫描不容易发现。

**防御**：对自研系统服务做「方法 → 权限」映射审计；对第三方 ROM 尤其要看厂商新增的蓝牙扩展接口。

### 8.3 状态机竞态

**表现**：配对/解绑过程中重复调用、状态缓存过期、回调顺序假设错误。

**落点**：应用侧的 bond 状态处理；系统侧的配对状态机。

**为什么反复出现**：蓝牙是异步的，状态变化来自多个线程（用户操作、链路事件、超时），而 API 只暴露「当前状态」，不暴露「状态迁移的原因」。

**防御**：只把 `getBondState()` 当快照，不当事实；所有连接动作以回调结果为准；解绑后丢弃缓存的 `BluetoothDevice`。

### 8.4 信任语义混淆

**表现**：把 bonded 当 authorized，把 encrypted 当 authenticated，把 connected 当 trusted。

**落点**：应用业务逻辑、GATT 特征权限配置、企业策略。

**为什么反复出现**：这三组词在自然语言里都接近「安全」，在协议里却是三个不同的位。

**防御**：在代码评审清单里明确写出三者的区别；对敏感特征值使用 `_MITM` 变体；业务授权单独实现。

### 8.5 密钥派生与覆盖

**表现**：一种传输的配对派生出另一种传输的密钥，攻击者覆盖或降级对端密钥。

**落点**：CTKD（Cross-Transport Key Derivation）、BLURtooth（CVE-2020-15802）。

**为什么反复出现**：双模设备（经典 + BLE）为了体验，允许「配一次、两种传输都能用」；派生链越长，可攻击面越大。

**防御**：关注 SIG 的密钥覆盖限制；对高安全场景要求两种传输分别认证；不要把「配了经典蓝牙」当成「BLE 也安全」。

### 8.6 用户交互被绕过

**表现**：本该由用户确认的配对/授权，被实现缺陷跳过。

**落点**：HID Host 配对确认（CVE-2023-45866）。

**为什么反复出现**：配对流程有多种「无输入」路径（Just Works、OOB、自动确认），实现时容易把「不需要输入」扩展成「不需要确认」。

**防御**：明确区分「无输入能力」与「无确认」；对 HID 这类高权限设备强制确认；缩短可发现窗口。

## 9. 观测点与自查

### 9.1 应用侧

| 检查项 | 怎么看 | 风险信号 |
|--------|--------|----------|
| 权限声明 | `AndroidManifest.xml` | 同时声明新旧权限且未加 `maxSdkVersion`；申请了不需要的 `BLUETOOTH_ADVERTISE` |
| `neverForLocation` | 清单属性 | 声称免定位却用扫描结果做定位；或反过来，明明只用连接却申请定位 |
| 权限申请时机 | 运行时 | 启动即申请「附近的设备」，与功能无关 |
| GATT 特征权限 | 服务端代码 | 敏感特征值只有 `PERMISSION_READ`，没有 `_ENCRYPTED_MITM` |
| 回调处理 | `onCharacteristicWrite` / `onMtuChanged` | 只看 `writeCharacteristic()` 返回值，不处理失败回调 |
| bond 状态 | `ACTION_BOND_STATE_CHANGED` | 不处理 `ACTION_KEY_MISSING`；解绑后继续用缓存对象 |
| 后台服务 | 前台服务类型 | 长期持有 `BLUETOOTH_CONNECT` 的后台服务没有明确业务理由 |
| 设备标识 | 地址使用 | 把 BLE MAC 当稳定主键，忽略 RPA 轮换 |

### 9.2 外设侧

| 检查项 | 怎么看 | 风险信号 |
|--------|--------|----------|
| 配对模型 | 配对流程 | 敏感设备用 Just Works；固定 PIN（0000/1234） |
| 特征权限 | GATT 表 | 写操作没有加密要求；敏感读没有 MITM 要求 |
| 密钥存储 | 固件 | 密钥明文存储、可被读取、无防回滚 |
| 地址策略 | 广播参数 | 固定 public 地址长期广播；无 RPA |
| 解绑处理 | 配对管理 | 删除绑定时不清理对端密钥；不验证密钥新鲜度 |
| 固件更新 | OTA | 更新包不签名；降级不阻止 |

### 9.3 设备管理

| 检查项 | 怎么看 | 风险信号 |
|--------|--------|----------|
| 可发现窗口 | 系统设置 | 长期处于可发现状态 |
| HID 绑定列表 | 蓝牙设置 | 存在用户不认识的键盘/鼠标 |
| 已绑定设备 | `getBondedDevices()` | 历史设备从不清理 |
| SPL | 系统信息 | 补丁级别落后于公告 |
| 企业策略 | DPC | 没有限制外接 HID、没有限制配对 |

### 9.4 一张最小 checklist

如果只能做五件事：

1. 确认敏感 GATT 特征值使用 `_ENCRYPTED_MITM`，而不是裸 `PERMISSION_READ/WRITE`。
2. 确认配对模型不是 Just Works（或明确接受其风险）。
3. 确认应用处理了 `ACTION_KEY_MISSING` 与 `onCharacteristicWrite` 失败。
4. 确认没有把 BLE MAC 当稳定设备标识。
5. 确认设备的 SPL 覆盖了本文台账中的相关条目。

## 10. 常见误传纠正

| # | 说法 | 判定 | 依据 |
|---|------|------|------|
| 1 | BLE 配对一定加密 | 部分正确 | 配对产生密钥，但加密是否启用取决于特征/连接的安全要求；明文特征值本身是合法配置 |
| 2 | Just Works 配对能防中间人 | 错误 | NIST：Just Works 不提供 MITM 保护，产生的是 unauthenticated link key |
| 3 | 有 `BLUETOOTH_SCAN` 就能读取任意 BLE 设备的数据 | 错误 | 扫描只负责发现；读写 GATT 需要 `BLUETOOTH_CONNECT`，还受特征权限约束 |
| 4 | GATT 的 encrypted 属性等同于 authenticated | 错误 | Android 明确区分 `PERMISSION_READ_ENCRYPTED` 与 `_ENCRYPTED_MITM` |
| 5 | Android 12 之后蓝牙扫描完全不需要定位权限 | 部分正确 | 仅 `targetSdk ≥ 31` 且声明 `neverForLocation` 时；用扫描结果定位仍需 `ACCESS_FINE_LOCATION` |
| 6 | 配对成功等于设备被信任、用户已认证 | 错误 | `BOND_BONDED` 只表示本地存有 link key；CVE-2023-40078 的前提就是「已配对设备」 |
| 7 | 关闭蓝牙可发现就完全不可被连接 | 错误 | `SCAN_MODE_CONNECTABLE` 明确表示「不可被发现但仍可接收连接」；知道地址即可直连 |
| 8 | BLE MAC 地址是固定的硬件地址 | 错误 | BLE 支持 Random Private Resolvable 地址，会周期性变化 |
| 9 | 三条蓝牙权限一次授权、全部生效 | 未核实 | 弹窗按「附近的设备」组呈现是确定的；是否批量授权无官方明文 |
| 10 | `createBond()` 一定会弹配对框 | 错误 | 是否弹窗取决于设备能力与配对模型；OOB、自动确认等路径可能不弹 |
| 11 | 普通应用可以调用 `setPairingConfirmation()` | 错误 | 需要 `BLUETOOTH_PRIVILEGED`（`signature|privileged`） |
| 12 | `ACTION_PAIRING_REQUEST` 可以被应用伪造 | 错误 | 它是 `<protected-broadcast>`，只有系统蓝牙进程能发 |
| 13 | 解绑后对端也会删除密钥 | 错误 | `removeBond()` 只删本地；对端语义官方未承诺 |
| 14 | 普通应用可以直接调用 `removeBond()` | 错误 | 当前为 `@SystemApi`；Android 16 起 CDM 提供 `removeBond(int)` |
| 15 | GATT 服务端的所有属性都由系统强制执行 | 部分正确 | 栈执行安全位（加密/MITM/密钥长度）；业务授权需应用自己实现 |
| 16 | 应用可以通过公开 API 读取本机 IRK | 错误 | 公开 SDK 无 IRK API；IRK 由蓝牙栈持有 |
| 17 | Companion Device Manager 可以绕过所有蓝牙权限 | 部分正确 | 它绕过扫描侧的定位耦合，不绕过 `BLUETOOTH_CONNECT`；且要求定位服务开启 |
| 18 | Android 16 会在 bond loss 时自动重新配对 | 错误 | 官方改为「断开链路、保留本地绑定、弹窗提示用户重新配对」 |
| 19 | 关闭蓝牙后 BLE 广播仍能被扫描到 | 错误 | 蓝牙关闭时射频与广播均停止 |
| 20 | BLE 连接必须先配对 | 错误 | BLE 可以直接连接并访问未加密的特征值；配对只在需要加密/MITM 时才必要 |
| 21 | `PAIRING_VARIANT_*` 常量已经废弃 | 未核实 | 所查 Android 13–16 分支未标 `@Deprecated`；不排除未来版本变更 |
| 22 | 蓝牙漏洞都要求先配对 | 错误 | 台账中多数条目只要求邻近；只有 CVE-2023-40078 明确写「paired device」 |

## 11. 与其他手册的分工

蓝牙不是孤岛。以下手册处理的是相邻但不同的问题：

- **《无线与近场通信攻击面手册》**：[android-wireless-nearby-stacks.md](/android-wireless-nearby-stacks/) —— Wi-Fi / 蓝牙 / NFC / UWB 的横向对比、Mainline 模块、HAL 面。
- **《内核与 GKI 驱动攻击面手册》**：[android-kernel-gki-drivers.md](/android-kernel-gki-drivers/) —— 蓝牙驱动与内核侧的攻击面、KMI 稳定性。
- **《框架 IPC 与组件导出攻击面手册》**：[android-framework-ipc-export.md](/android-framework-ipc-export/) —— 系统服务权限检查缺失的通用模式。
- **《多用户、工作资料与进程隔离手册》**：[android-multiuser-sandbox-isolation.md](/android-multiuser-sandbox-isolation/) —— 蓝牙绑定记录在多用户下的隔离问题。
- **《账户、凭据与无密码认证手册》**：[android-accounts-credentials-passkeys.md](/android-accounts-credentials-passkeys/) —— 「设备认证」与「用户认证」的区别，可与配对信任对照读。
- **《传感器与定位攻击面手册》**：[android-sensors-location.md](/android-sensors-location/) —— 蓝牙扫描与位置推断的关系。
- **《音频栈与媒体会话攻击面手册》**：[android-audio-media-session.md](/android-audio-media-session/) —— A2DP/LE Audio 的路由与音频面。
- **《无障碍、输入法与自动填充通道手册》**：[android-accessibility-ime-autofill.md](/android-accessibility-ime-autofill/) —— 另一条「输入注入」路径，可与 HID 对照。

## 参考

### Android 官方文档

- 蓝牙权限：<https://developer.android.com/develop/connectivity/bluetooth/bt-permissions>
- 查找蓝牙设备：<https://developer.android.com/develop/connectivity/bluetooth/find-bluetooth-devices>
- 配对与连接：<https://developer.android.com/develop/connectivity/bluetooth/connect-bluetooth-devices>
- Companion Device Manager：<https://developer.android.com/develop/connectivity/bluetooth/companion-device-pairing>
- `BluetoothDevice` 参考：<https://developer.android.com/reference/android/bluetooth/BluetoothDevice>
- `BluetoothGattCharacteristic` 参考：<https://developer.android.com/reference/android/bluetooth/BluetoothGattCharacteristic>
- `BluetoothGatt` 参考：<https://developer.android.com/reference/android/bluetooth/BluetoothGatt>
- 前台服务类型：<https://developer.android.com/develop/background-work/services/fgs/service-types>
- Android 12 行为变更：<https://developer.android.com/about/versions/12/behavior-changes-12>
- Android 13 行为变更：<https://developer.android.com/about/versions/13/behavior-changes-13>
- Android 14 行为变更（所有应用）：<https://developer.android.com/about/versions/14/behavior-changes-all>
- Android 16 行为变更（所有应用）：<https://developer.android.com/about/versions/16/behavior-changes-all>
- Android 16 行为变更（targetSdk 36）：<https://developer.android.com/about/versions/16/behavior-changes-16>

### AOSP 源码与架构

- 蓝牙架构：<https://source.android.com/docs/core/connect/bluetooth>
- Mainline 模块：<https://source.android.com/docs/core/architecture/modular-system>
- `AndroidManifest.xml`（权限声明）：<https://android.googlesource.com/platform/frameworks/base/+/refs/heads/main/core/res/AndroidManifest.xml>
- `PermissionManagerServiceImpl`：<https://android.googlesource.com/platform/frameworks/base/+/refs/heads/main/services/core/java/com/android/server/pm/permission/PermissionManagerServiceImpl.java>
- `packages/modules/Bluetooth`：<https://android.googlesource.com/platform/packages/modules/Bluetooth/+/refs/heads/main>
- `gatt_sr.cc`：<https://android.googlesource.com/platform/packages/modules/Bluetooth/+/refs/heads/main/system/stack/gatt/gatt_sr.cc>
- `btm_ble_privacy.cc`：<https://android.googlesource.com/platform/packages/modules/Bluetooth/+/refs/heads/main/system/stack/btm/btm_ble_privacy.cc>
- 蓝牙 HAL 接口：<https://android.googlesource.com/platform/hardware/interfaces/+/refs/heads/main/bluetooth/>

### 公告与漏洞记录

- 2017-09 公告：<https://source.android.com/docs/security/bulletin/2017-09-01>
- 2020-02 公告：<https://source.android.com/docs/security/bulletin/2020-02-01>
- 2021-06 公告：<https://source.android.com/docs/security/bulletin/2021-06-01>
- 2023-03 公告：<https://source.android.com/docs/security/bulletin/2023-03-01>
- 2023-12 公告：<https://source.android.com/docs/security/bulletin/2023-12-01>
- 2025-01 公告：<https://source.android.com/docs/security/bulletin/2025-01-01>
- 严重级与远程定义：<https://source.android.com/docs/security/overview/updates-resources>
- CVE-2020-0022：<https://www.cve.org/CVERecord?id=CVE-2020-0022>
- CVE-2020-0023：<https://www.cve.org/CVERecord?id=CVE-2020-0023>
- CVE-2020-26555：<https://www.cve.org/CVERecord?id=CVE-2020-26555>
- CVE-2023-40078：<https://www.cve.org/CVERecord?id=CVE-2023-40078>

### 协议与研究

- Bluetooth Core 规范（Security Manager）：<https://www.bluetooth.com/wp-content/uploads/Files/Specification/HTML/Core-60/out/en/host/security-manager-specification.html>
- Bluetooth Core 规范（GAP）：<https://www.bluetooth.com/wp-content/uploads/Files/Specification/HTML/Core-60/out/en/host/generic-access-profile.html>
- SIG 配对介绍：<https://www.bluetooth.com/blog/bluetooth-pairing-part-1-pairing-feature-exchange/>
- NIST SP 800-121 Rev.2：<https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-121r2.pdf>
- KNOB（CERT VU#918987）：<https://www.kb.cert.org/vuls/id/918987>
- BIAS：<https://francozappa.github.io/about-bias/>
- BLESA：<https://www.usenix.org/conference/woot20/presentation/wu>
- BLURtooth：<https://hexhive.epfl.ch/BLURtooth/>
- Method Confusion（CERT VU#534195）：<https://www.kb.cert.org/vuls/id/534195>

## 未核实与失效来源

以下各项在正文中没有被写成确定结论，集中列出原因。

**A. 蓝牙 Mainline 模块的三个包名**
官方 Mainline 表写 `com.google.android.bt`；AOSP 的 APEX 名是 `com.android.bt`，APK 模块名 `Bluetooth`，应用包名沿用 `com.android.bluetooth`。三者是「Google 签名发布名 / AOSP 名 / 应用包名」的区别，但官方没有逐字解释。**正文只说「模块名与包名口径不同」，不指定唯一包名。**

**B. `cancelBondProcess()` 的权限**
参考文档称它在 API 37 转为 public，且 `BLUETOOTH_PRIVILEGED` 只在「调用方不是配对发起者」时强制；AOSP main 仍是 `@SystemApi @hide` 且无条件要求 `BLUETOOTH_CONNECT + BLUETOOTH_PRIVILEGED`。**正文按当前 AOSP 行为描述，并标注冲突。**

**C. 权限组是否批量授权**
`permissionGroup` 在清单中为 `UNDEFINED`，实际分组由系统服务维护；「同组弹窗」确定，「一次授权三条全给」无官方明文。**正文只写前者。**

**D. GATT 业务授权**
没有找到官方文档逐字写出「业务授权由应用自己检查」。正文的结论是从 `gatt_sr.cc` 只接收安全标志与密钥长度这一实现事实推导的，属于机制推断。

**E. BR/EDR 逐模型 MITM 表**
引用的 Core 6.0 章节是 LE Secure Connections；BR/EDR 的 SSP 四种模型逐条 MITM 属性未逐字取到。**正文的模型表按通行定义写，并与 NIST 的 Just Works 表述交叉验证。**

**F. CSRK 的定义**
「CSRK 用于签名无连接数据」是通行说法，但本文未取到可引用的 SIG 规范原文。**正文把它列为三类密钥之一，不展开细节。**

**G. Bluetooth 模块是否 optional**
官方模块页未给出「optional」字样（第 15 轮手册曾引用过这一说法）。**本手册不重复该断言。**

**H. Android 15 的蓝牙变更**
所查两页行为变更文档均未列出蓝牙条目。这是「未发现」，不是官方「无变更」声明。**正文的版本表只列 12/13/14/16。**

**I. KNOB 的 Android 官方受影响表述**
CERT 把 AOSP 列为 Unknown，Google 未发布对应公告。**正文只把 KNOB 作为协议层背景，不声称 Android 受影响或不受影响。**

**J. CVE-2023-45866 的口径冲突**
Android 公告写的是「System 节、EoP、Critical、Android 11–14」；MITRE 描述写的是 BlueZ HID Host 缺陷。两者描述的是同一编号的不同侧面，本文以 Android 公告的版本与定级为准，并在正文注明冲突。

**K. SIG 规范中 unbond 的对端语义**
未取到可引用的原文，无法确认「解绑是否要求对端也删除密钥」。**正文只说「官方未承诺」，不推断协议要求。**

**L. `PAIRING_VARIANT_*` 是否废弃**
所查 Android 13–16 分支未标 `@Deprecated`，但这不排除未来版本变更；网络流传的「已废弃」说法未找到依据。

**M. Windows 键盘注入的 CVE-2024-21306**
仅见研究页面，未在 Microsoft 官方公告核实，故未写入正文。

---

## 联系方式

咨询 Android 系统详情请联系 https://t.me/one00190

---

*本手册仅供安全学习与防御研究使用；所有机制表述、公告列值与 CVE 描述均取自公开一手来源并附链接，未能核实的部分已单列于「未核实与失效来源」而非写入正文。*
