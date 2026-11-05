---
layout: page
title: "Android UWB 超宽带测距与安全测距攻击面手册"
description: "把「测到距离」拆成框架／模块／HAL／协议四层：`android.uwb` 没有公开的 `UwbAdapter` 类（只有 `IUwbAdapter.aidl`）、第三方走 `androidx.core.uwb`、`UWB_RANGING` 危险权限＋前台＋`android.hardware.uwb` ..."
date: 2026-11-05
permalink: /android-uwb-ranging-security/
---

> UWB（超宽带）在 Android 上承担的是「厘米级距离 + 角度」这类高价值判断：数字车钥匙靠它判断手机是否真的在车旁，附近查找靠它给出方向。它和 Wi-Fi/蓝牙最大的不同是——**距离本身会被当作安全决策依据**。于是攻击目标不再只是「连上设备」，而是「让测距结果说谎」：中继、重放、首径伪造，都能让「人在远处」变成「人在门前」。这份手册沿「框架与 HAL → 会话与密钥 → 安全测距原理 → 公开缺陷 → 隐私与滥用 → 怎么防」拆解这条通路。只讨论公开且已修复的问题，不含测距代码。

## 0. 使用方式与边界

面向三类读者：

- **应用开发者**：需要区分「拿到距离」与「确认距离可信」，理解 static STS 与 provisioned STS 的差别。
- **系统与 HAL 工程师**：需要知道 `android.uwb` 的框架层、AIDL HAL 与 FiRa UCI 之间的分工。
- **安全评估者**：需要在看到 UWB 相关公告时，判断问题落在协议栈解析、加密使用、权限校验还是测距参数配置。

三条自我约束：

1. 只引用公开且已修复的问题；CVE 描述以 NVD 与厂商公告为准。
2. 不写测距会话参数、不写中继步骤、不写可复现的配置。
3. 查不到一手来源的说法放进「未核实与失效来源」。

与已发布手册的分工：

- 《Android 无线与近场通信栈手册》讲 Wi-Fi、蓝牙与 NFC；本文只讲 UWB 的测距会话与安全测距。
- 《Android 传感器与定位攻击面手册》讲定位数据与传感器权限；本文讲「距离测量」这一独立信任链。
- 《Android TEE、密钥与证明体系手册》讲密钥与证明；本文只讲会话密钥如何影响测距可信度。

## 1. 框架、模块与 HAL

### 1.1 版本与形态

- Android 12 的 AOSP 分支已包含 `android.uwb` 包；
- Android 13 起提供默认框架实现，并把 UWB 作为可选的 Mainline 模块；
- 第三方应用通常通过 Jetpack `androidx.core.uwb` 使用，要求 Android 12+ 且设备声明 `android.hardware.uwb` 特性。

### 1.2 分层

| 层 | 组件 | 职责 |
|----|------|------|
| 应用 | `androidx.core.uwb` | 封装会话、参数与回调 |
| 框架 | `UwbManager`、`RangingSession`、`UwbAddress` | 会话生命周期、地址分配 |
| HAL | AIDL `android.hardware.uwb`（`IUwbChip`、`IUwbClientCallback`） | 芯片控制与事件上报 |
| 固件/协议 | FiRa UCI | 测距参数、STS 配置、报告 |

注意：AOSP 的 `android.uwb` 中**没有 `UwbAdapter` 这个类**，只有内部的 `IUwbAdapter.aidl`。把 `UwbAdapter` 写成公开 API，是常见记错。

### 1.3 权限与前台限制

- 第三方应用需要危险权限 `android.permission.UWB_RANGING`；
- 平台系统 API 还需要 `UWB_PRIVILEGED`（signature|privileged）；
- 发起会话要求应用处于前台；后台启动仅部分设备支持，且后台通常收不到测距报告；
- 设备必须声明 UWB 特性，否则框架层直接不可用。

这三条共同构成「谁能测距」的第一道边界：**权限 + 前台 + 硬件特性**。

## 2. 会话模型

一次测距会话大致包含：

1. 发现对端（通常经带外通道，如 BLE 或二维码，交换地址与参数）；
2. 协商测距参数（信道、前导码、会话 ID、STS 模式）；
3. 建立 `RangingSession`，开始测距；
4. 周期性收到距离/角度报告；
5. 结束会话并释放资源。

关键点是第 1、2 步：**UWB 本身不负责「对端是谁」**。身份与参数通常由带外通道提供，UWB 只负责「这段距离测得准不准」。因此 UWB 的安全性高度依赖带外通道的认证与密钥交换。

把 UWB 当作「自带身份认证的测距」是高估；把带外通道当作「只是配对用的辅助」是低估。

## 3. 安全测距：STS 与距离界定

### 3.1 STS 是什么

IEEE 802.15.4z 引入的 STS（Scrambled Timestamp Sequence）是一段由密钥生成的伪随机脉冲序列，接收端用它验证首径到达时间。常见两种模式：

| 模式 | 密钥 | 抗重放 | 说明 |
|------|------|--------|------|
| Static STS | 固定 8 字节 | 弱 | 同一密钥重复使用，易被重放/中继 |
| Provisioned STS | 每会话 16 字节（Android 14+） | 较强 | 需要安全的带外密钥分发 |

Jetpack 通过 `RangingParameters` 的 `uwbConfigType` 与 `sessionKeyInfo` 选择模式；Android 16 的 `UwbRangingParams` 才以 `CONFIG_*` 常量显式区分 STATIC 与 P-STS。官方文档明确：**static STS 对 replay/relay 攻击敏感，provisioned STS 可缓解**。

### 3.2 STS 不等于完整 distance bounding

STS 提升了时间戳的完整性，但它不是完整的距离界定协议：

- 接收端仍需正确识别「首径」，而首径检测阈值可能被更强的伪造信号影响；
- 密钥分发若走不安全的带外通道，攻击者可参与会话；
- 中继攻击可以在物理层放大信号，STS 只能提高门槛，不能单独消灭风险。

因此，安全测距的正确表述是「STS + 安全密钥分发 + 可靠首径检测 + 业务侧多因素」，而不是「开了 STS 就安全」。

### 3.3 数字车钥匙

CCC Digital Key Release 3.0（2021-07）的组合是：BLE 做认证与唤醒，UWB 做安全测距定位，NFC 做低电量后备。UWB 在这里的价值是**防中继**：车端不仅要知道「钥匙在附近」，还要知道「距离在阈值内且测量未被重放」。这也解释了为什么车钥匙场景对 provisioned STS 与安全元件背书的依赖远高于普通应用。

## 4. 公开公告台账

| CVE | 落点 / 类型 | 级别 | 修复 | 说明 |
|-----|-------------|------|------|------|
| [CVE-2022-25818](https://nvd.nist.gov/vuln/detail/CVE-2022-25818) | 三星 UWB 协议栈边界检查不当（可致代码执行） | 厂商公告 High | 2022-03 SMR | 芯片/协议栈侧 |
| [CVE-2022-20507](https://nvd.nist.gov/vuln/detail/CVE-2022-20507) | `UwbEventManager.java` 缺边界检查（本地提权） | 公告 High | 2022-12（Pixel） | 框架事件解析 |
| [CVE-2023-21358](https://nvd.nist.gov/vuln/detail/CVE-2023-21358) | UWB 加密使用不当，可冒充 `com.android.uwb.resources`（提权） | 公告 High | 2023-10（Android 14） | 加密误用 |
| [CVE-2023-30660](https://nvd.nist.gov/vuln/detail/CVE-2023-30660) | 三星 `UwbAospAdapterService.getDefaultChipId` 泄露芯片 ID | 厂商公告 | 2023-07 SMR | 信息泄露 |

需要如实说明：**本次检索未发现以「越权测距」或「测距拒绝服务」为主因的公开 CVE**，也未发现直接针对 Android UWB HAL/驱动的公开 CVE。上表四例分别是内存安全、解析边界、加密误用与信息泄露——这提示我们，当前公开缺陷更多在「实现质量」，而不是「测距协议被攻破」。

读表要点：

1. **协议栈解析是入口**：事件、消息、长度字段来自芯片或对端，解析不当即越界（CVE-2022-20507）。
2. **加密误用比算法强弱更常见**：CVE-2023-21358 不是算法被破，而是使用方式错误导致身份可冒充。
3. **标识泄露也有价值**：芯片 ID 可辅助指纹与关联（CVE-2023-30660）。

## 5. 隐私与滥用面

### 5.1 存在感知与雷达研究

学术研究已展示 UWB 信号可用于存在检测、呼吸/生命体征估计、设备无关定位。需要区分：

- **研究环境**：使用原始信道冲激响应（CIR）与专用设备；
- **Android 公开 API**：只返回距离/角度/位置报告，不向第三方应用暴露原始 CIR。

因此「装了 UWB 的手机就能被任意应用当雷达用」是夸大，但「UWB 信号完全不含隐私信息」同样不准确。

### 5.2 标签与跟踪

UWB 标签（如防丢器）提供精确方向查找。它的滥用形态是**更精准的跟踪**：攻击者若能把标签放入目标物品，就能获得比蓝牙 RSSI 更可靠的距离与方向。平台层面的缓解包括：未知标签提醒、跨品牌反跟踪联盟、用户可查看附近标签。

### 5.3 用户开关

Android 提供 UWB 用户开关（设置路径因 OEM 而异），关闭后调用 `setUwbEnabled(false)`；飞行模式也会关闭。企业可通过设备策略限制（如 Android 14 引入的 `DISALLOW_ULTRA_WIDEBAND_RADIO`）。

## 6. 成因模式

### 6.1 协议消息边界

UCI 消息与 HAL 回调携带长度、计数与地址字段。未校验长度就按固定偏移读取，会造成越界（CVE-2022-20507）。修复要点是「先验长度，再解析」。

### 6.2 加密原语误用

用错模式、复用 nonce、把「加密」当「认证」，都会让身份可冒充（CVE-2023-21358）。安全测距的密钥管理必须区分机密性、完整性与新鲜性三种需求。

### 6.3 会话密钥分发

static STS 的密钥固定，重放成本低；provisioned STS 依赖带外分发。若应用在明文 BLE 通道上交换 16 字节密钥，等于把安全测距退化成 static STS。

### 6.4 权限与前台校验缺失

UWB 会话涉及硬件与位置推断，若后台应用可静默发起、或系统 API 未校验 `UWB_PRIVILEGED`，就会扩大滥用面。公开 CVE 中未见此类，但评估时应作为重点检查项。

### 6.5 标识与日志泄露

芯片 ID、会话 ID、地址若进入日志或可被普通应用查询，会成为指纹与关联信号（CVE-2023-30660）。

## 7. 版本与能力时间线

| 版本 / API | 变化 |
|------------|------|
| 12 / 31 | AOSP 引入 `android.uwb`；Jetpack 支持 |
| 13 / 33 | 默认框架实现；UWB 成为可选 Mainline 模块 |
| 14 / 34 | provisioned STS 支持；企业策略可禁用 UWB 无线电 |
| 16 | `UwbRangingParams` 以 `CONFIG_*` 显式区分 STS 模式 |

趋势是**把 UWB 从「厂商私有实现」纳入 Mainline 与统一 HAL**，同时把安全测距参数显式化。

## 8. 观测点与自查

### 8.1 应用侧

- 是否优先使用 provisioned STS？密钥是否经认证的带外通道交换？
- 是否把距离作为唯一访问决策？是否结合密码学认证与业务二次确认？
- 是否在前台发起会话、结束后及时关闭？
- 是否导出了可触发测距的组件？是否校验调用者？
- 是否把芯片 ID、会话 ID 写入日志？

### 8.2 平台与 HAL 侧

- UCI 消息解析是否先验长度、再按字段读取？
- `IUwbChip` 回调是否校验调用者与状态机？
- 系统 API 是否强制 `UWB_PRIVILEGED` 与前台状态？
- 是否提供用户开关与企业策略？
- 固件与协议栈补丁是否跟进厂商公告？

### 8.3 评估侧

- 先定位落点：应用参数 / 框架会话 / HAL 解析 / 固件协议 / 密钥分发；
- 区分「内存安全」「加密误用」「信息泄露」「测距可信度」；
- 不要把「测距不准」直接等同于漏洞——要看是否影响安全决策。

## 9. 防御要点

1. **默认 provisioned STS**：能动态密钥就不要用 static STS。
2. **带外通道要认证**：密钥交换必须有机密性与完整性保护。
3. **距离不是唯一因素**：访问决策结合密码学身份、业务规则与用户确认。
4. **解析先验长度**：所有 UCI/HAL 消息按「长度 → 类型 → 负载」顺序校验。
5. **权限与前台双检**：系统 API 同时校验特权权限与前台状态。
6. **最小化标识输出**：芯片 ID、会话 ID 不进入日志与普通应用可见接口。
7. **给用户开关**：提供关闭 UWB 的系统设置，并支持企业策略禁用。

## 10. 常见误传纠正

### 误传一：UWB 精度高所以天然安全

错。精度与抗攻击是两件事。static STS 可被重放/中继；安全取决于密钥、STS 模式与首径检测。

### 误传二：有 UWB 硬件就能被任意应用测距

错。需要 `UWB_RANGING` 权限、前台会话、对端带外参数交换，系统 API 还需 `UWB_PRIVILEGED`。

### 误传三：UWB 可被动跟踪任何人

夸大。测距是成对会话，被动观察受 STS 缓解；Android 公开 API 不暴露原始 CIR。

### 误传四：Android 13 才有 UWB

不准确。Android 12 的 AOSP 已包含平台 API；Android 13 起才有默认框架实现与可选 Mainline 模块。

### 误传五：STS 就是完整的 distance bounding

错。STS 提升时间戳完整性，但仍需安全密钥分发与可靠首径检测；完整距离界定是协议与实现的组合。

## 11. 未核实与失效来源

- 「越权测距」「UWB 测距 DoS」类公开 CVE 未检索到，不能据此断言不存在。
- Android 16 `UwbRangingParams` 的具体常量命名与可用范围未在真机核实。
- 各 OEM 的 UWB 开关路径、默认 STS 策略与固件补丁节奏未逐一核实。
- UWB 雷达研究的可复现条件与手机端可行性差异较大，本文只作方向性说明。
- CCC Digital Key 3.0 的具体认证要求以其规范为准，本文未逐条核对。

## 12. 参考资料

- Android UWB 开发者指南：
  https://developer.android.com/develop/connectivity/uwb
- AOSP UWB 架构：
  https://source.android.com/docs/core/connect/uwb
- UWB Mainline 模块：
  https://source.android.com/docs/core/ota/modular-system/uwb
- UWB HAL 接口：
  https://source.android.com/docs/core/connect/uwb-hal-interface
- `UwbManager`（AOSP）：
  https://android.googlesource.com/platform/packages/modules/Uwb/+/refs/heads/main/framework/java/android/uwb/UwbManager.java
- `RangingSession`（AOSP）：
  https://android.googlesource.com/platform/packages/modules/Uwb/+/refs/heads/main/framework/java/android/uwb/RangingSession.java
- `UwbRangingParams`：
  https://developer.android.com/reference/android/ranging/uwb/UwbRangingParams
- IEEE 802.15.4z：
  https://standards.ieee.org/ieee/802.15.4z/10230/
- FiRa 技术 FAQ：
  https://www.firaconsortium.org/resource-hub/technical-faq
- CCC Digital Key 3.0 公告：
  https://carconnectivity.org/car-connectivity-consortium-publishes-digital-key-release-3-0-businesswire/
- 公告：
  https://source.android.com/docs/security/bulletin/pixel/2022-12-01
  https://source.android.com/docs/security/bulletin/android-14
  https://security.samsungmobile.com/securityUpdate.smsb?year=2022&month=3
  https://security.samsungmobile.com/securityUpdate.smsb?year=2023&month=07
- NVD 条目（本文台账）：
  https://nvd.nist.gov/vuln/detail/CVE-2022-25818
  https://nvd.nist.gov/vuln/detail/CVE-2022-20507
  https://nvd.nist.gov/vuln/detail/CVE-2023-21358
  https://nvd.nist.gov/vuln/detail/CVE-2023-30660

---

## 联系方式

咨询 Android 系统详情请联系 https://t.me/one00190

---

*本手册仅供安全学习与防御研究使用；所有机制表述、公告列值与 CVE 描述均取自公开一手来源并附链接，未能核实的部分已单列于「未核实与失效来源」而非写入正文。*
