---
layout: page
title: "Android 设备标识符、广告 ID 与硬件指纹手册"
description: "把「设备 ID」拆成硬件／系统／服务／推断四层：IMEI 与序列号自 Android 10 起收进 `READ_PRIVILEGED_PHONE_STATE`、`ANDROID_ID` 自 8.0 起按「签名＋用户＋设备」派生（同签名重装不变、换签名会变）、广告 ID 可重置且 Android 12 起..."
date: 2026-11-05
permalink: /android-device-identifiers-fingerprinting/
---

> 「设备标识符」在 Android 上不是一个字段，而是一组被平台分多年拆开的接口：`ANDROID_ID` 按签名隔离，广告 ID 可重置可删除，MAC 默认随机化，IMEI 与序列号被收进特权权限。平台每收窄一条路径，指纹产业就转向另一组更弱的信号——Build 字段、传感器特征、已安装应用列表、字体与屏幕参数。这份手册沿「标识符怎么来 → 权限怎么收窄 → 指纹怎么拼 → 公开泄露案例 → 怎么防」拆解这条隐私攻击面。只讨论公开且已修复的问题，不含采集代码。

## 0. 使用方式与边界

面向三类读者：

- **应用与 SDK 开发者**：需要知道哪些标识符可用、哪些已不可用、哪些「可用但不该用」。
- **平台与 ROM 工程师**：需要理解标识符生成、随机化与权限收紧的版本差异。
- **安全与隐私评估者**：需要在看到「设备识别」类问题时，区分「系统泄露」「权限缺失」「合法接口被滥用」三种性质。

三条自我约束：

1. 只引用公开且已修复的问题；CVE 描述以 NVD 与厂商公告为准。
2. 不写采集代码、不写可用于追踪的字段组合。
3. 查不到一手来源的说法放进「未核实与失效来源」。

与已发布手册的分工：

- 《Android 账户、凭据与无密码认证手册》讲用户身份；本文讲设备侧标识。
- 《Android 无线与近场通信栈手册》讲 Wi-Fi/蓝牙协议；本文只讲它们暴露的地址类标识。
- 《Android 媒体捕获、麦克风与相机隐私手册》讲传感器采集；本文只讲「传感器清单」作为指纹信号的用途。

## 1. 标识符的四个层次

把「设备 ID」当成一个东西，是绝大多数误判的起点。按稳定性与授权门槛，可以分成四层：

| 层次 | 例子 | 稳定性 | 访问门槛 |
|------|------|--------|----------|
| 硬件级 | IMEI/MEID、序列号、真实 MAC | 极高 | 特权权限，第三方不可得 |
| 系统级 | `ANDROID_ID`、Widevine 设备 ID | 高（恢复出厂变化） | 普通 API，但按签名/用户隔离 |
| 服务级 | 广告 ID | 中（可重置、可删除） | Google Play services |
| 推断级 | Build 字段、传感器清单、字体、屏幕参数 | 低到中 | 无权限，但组合后具辨识度 |

平台的设计方向是：**把前两层收进权限，把第三层做成可撤销，把第四层留作「无法完全消除、只能降低熵」的残余面**。

## 2. 系统级标识：`ANDROID_ID`

`Settings.Secure.ANDROID_ID` 是 64 位十六进制字符串。它的取值规则在 Android 8.0 发生了关键变化：

- 由「应用签名密钥 + 用户 + 设备」共同派生；
- 同一签名、同一用户、同一设备上，卸载重装**不变**；
- 换签名、恢复出厂、换用户会变；
- Instant App 使用独立作用域。

这带来两个常见误解：

1. 「卸载重装就会换 ID」——错，同签名重装不变；
2. 「`ANDROID_ID` 是硬件序列号」——错，它是按应用隔离的派生值。

需要强调的是：**按签名隔离意味着同一开发者的多个应用可能看到同一个值**。如果开发者把 `ANDROID_ID` 当作「跨应用不可关联」的保证，就会高估隔离强度。

## 3. 广告 ID：可重置的服务级标识

广告 ID 由 Google Play services 提供（`AdvertisingIdClient`），按设备用户区分，用户可以重置；Android 12 起设置中提供「删除」入口，删除或关闭个性化广告后返回全零串。

它的定位是「用于广告的、可撤销的标识」，因此：

- 不能当作硬件标识用于风控；
- 不能与 `ANDROID_ID` 混用做「永久绑定」；
- 必须处理全零串与用户关闭个性化的情形；
- 跨应用共享仍受平台与政策约束。

把广告 ID 写进长期账户画像、且不提供删除路径，既不符合平台设计，也容易触发应用商店政策问题。

## 4. 硬件级标识：被收窄的历史

### 4.1 MAC 地址

- Android 6.0 起，普通应用读取 Wi-Fi MAC 恒为 `02:00:00:00:00:00`；
- 后台扫描使用随机 MAC；
- Android 10 起默认对每个网络使用持久随机 MAC（可按网络关闭）；
- Android 11 起 `WifiInfo.getHardwareAddress()` 对第三方返回 `null`；
- Android 12 起部分场景使用非持久随机 MAC。

「随机 MAC 就不可关联」是过度结论：持久随机是按网络派生的，同一网络下仍可跨会话关联；只有非持久随机才显著削弱这种关联。

### 4.2 IMEI / MEID / 序列号

Android 10 起，读取 IMEI/MEID 需要 `READ_PRIVILEGED_PHONE_STATE`，第三方应用不可声明；targetSdk 29+ 调用会抛 `SecurityException`。`Build.SERIAL` 对普通应用恒为 `"unknown"`，`Build.getSerial()` 仅特权应用、运营商权限或设备所有者可调用。

这意味着：**第三方应用声称「读取 IMEI 做风控」在 Android 10+ 上基本不可行**，除非它是预置特权应用或系统角色。

### 4.3 Widevine 设备 ID

`MediaDrm` 的 device unique ID 在设备配置时建立，用于 DRM 场景。官方指南同时说明存在按 APK 的 Widevine ID。它是否可重置、跨应用是否一致，文档没有给出统一结论，**未核实**。把 DRM ID 当通用设备标识使用，是不合适且可能违反许可约定的。

## 5. 推断级指纹：没有权限也能拼

即使所有强标识都被收窄，仍有一组「无需权限」的信号可以组合：

- `Build.MANUFACTURER`、`MODEL`、`DEVICE`、`FINGERPRINT`、`HARDWARE`；
- 屏幕分辨率、密度、刷新率；
- 系统语言、时区、字体缩放；
- 传感器清单与精度（`SensorManager.getSensorList`）；
- 支持的编解码器、GLES/Vulkan 能力；
- 已安装应用列表（受 `QUERY_ALL_PACKAGES` 与商店政策约束）；
- 网络接口列表与部分网络参数。

单个字段的熵不高，但组合后足以在人群中稳定识别一台设备。平台无法也不打算完全消除这类信号，因此监管与行业实践更强调「最小化收集、明确告知、可删除」。

### 5.1 已安装应用列表

`QUERY_ALL_PACKAGES` 是一个高敏感权限：它能看到用户装了哪些应用，进而推断兴趣、职业、健康状况。Google Play 对该权限有明确的使用场景限制，要求开发者说明必要性。把包列表与广告 ID 结合，是最常见的跨应用画像手段之一。

## 6. 公开公告台账

| CVE | 落点 / 类型 | 级别 | 修复 | 说明 |
|-----|-------------|------|------|------|
| [CVE-2018-9489](https://nvd.nist.gov/vuln/detail/CVE-2018-9489) | `WifiStateMachine` 广播泄露 SSID/BSSID/IP/DNS/MAC | 信息泄露 | Android 9（2018 年披露） | 广播接收方无需特殊权限即可获取 |
| [CVE-2021-25344](https://nvd.nist.gov/vuln/detail/CVE-2021-25344) | 三星 `knox_custom` 服务缺权限校验，泄露序列号 | 信息泄露 | 2021-03 SMR | 厂商组件 |
| [CVE-2020-11287](https://nvd.nist.gov/vuln/detail/CVE-2020-11287) | 高通 WLAN RTT 序列号可与未随机化 MAC 关联 | 信息泄露 | 2021-02 公告 | 芯片侧 |

补充说明：

- 本次检索**未找到**可确认的「`ANDROID_ID` 可预测」类 CVE，因此不列。
- CVE-2018-9489 的 Android 公告月份未在 MITRE 记录中给出，本文只写披露年份与修复版本。
- 厂商组件的标识符泄露往往不在 AOSP 公告内，需要单独跟踪 OEM 公告。

读表要点：

1. **泄露渠道不一定是「读取 API」**：广播、日志、调试接口都可能带出标识。
2. **关联比单点更危险**：RTT 序列号本身不敏感，能与 MAC 关联后才成为追踪手段。
3. **随机化不是万能**：持久随机仍可跨会话关联。

## 7. 成因模式

### 7.1 广播与日志的隐式泄露

系统状态变化广播常带详细网络信息（CVE-2018-9489）。修复方式通常是收窄广播接收者、移除敏感字段，或改为显式组件。

### 7.2 特权接口缺校验

厂商扩展服务若未校验调用者权限，就会把序列号、芯片 ID 等本应受保护的字段暴露给普通应用（CVE-2021-25344）。

### 7.3 随机化实现不完整

MAC 随机化覆盖了主动读取，却漏掉了旁路关联（如 RTT 序列号），导致「看似随机、实际可关联」（CVE-2020-11287）。

### 7.4 接口设计与政策脱节

某些标识符在技术上可读，但商店政策限制其用途。开发者只考虑「能不能读」，不考虑「该不该读」，就会在合规审查中出问题。

### 7.5 指纹组合

单字段脱敏无法阻止组合指纹。平台收窄强标识后，弱信号组合成为主要追踪手段；这类问题没有单一 CVE，只能靠整体隐私设计缓解。

## 8. 版本与隐私时间线

| 版本 / API | 变化 |
|------------|------|
| 6.0 / 23 | 应用读 MAC 恒为零地址；后台扫描随机化 |
| 8.0 / 26 | `ANDROID_ID` 按签名+用户+设备派生；后台执行限制 |
| 9 / 28 | 后台传感器限制；非活动应用权限收紧 |
| 10 / 29 | IMEI/序列号限制；MAC 默认持久随机；分区存储 |
| 11 / 30 | `getHardwareAddress()` 对第三方返回 null；一次性权限 |
| 12 / 31 | 广告 ID 可删除；大致位置；隐私指示器 |
| 13 / 33 | `POST_NOTIFICATIONS`；`READ_MEDIA_*`；通知默认关闭 |
| 14 / 34 | 部分照片访问；Data safety 更醒目 |
| 15 / 35 | 通知 OTP 脱敏；Private Space |

趋势是**把「可关联的强标识」逐层替换为「可撤销、可隔离、可解释」的接口**。

## 9. 观测点与自查

### 9.1 应用与 SDK

- 是否收集了不必要的标识符？是否能用更弱的信号替代？
- 是否处理广告 ID 全零、用户删除、`LIMIT_AD_TRACKING` 等状态？
- 是否依赖 `Build.SERIAL`、`getHardwareAddress()` 等已收窄的接口？
- 是否声明 `QUERY_ALL_PACKAGES`？必要性是否能在政策审查中说明？
- 是否为用户提供删除数据与重置标识的入口？

### 9.2 平台与 ROM

- 系统广播、日志、调试接口是否带出 SSID/BSSID/MAC/序列号？
- 随机 MAC 是否覆盖主动读取、被动扫描与旁路关联三条路径？
- 厂商扩展服务是否校验调用者权限？
- 预置特权应用是否滥用标识符读取权限？
- 是否提供用户可见的标识符使用说明？

### 9.3 评估侧

- 先区分「系统泄露」「权限缺失」「合法接口被滥用」「组合指纹」；
- 不要把「能读到某字段」直接等同于「漏洞」——要看权限模型与政策；
- 关注关联能力，而不只是单字段敏感性。

## 10. 防御要点

1. **最小化收集**：能用会话级随机值就不申请设备级标识。
2. **优先可撤销标识**：广告场景用广告 ID，并处理删除与全零。
3. **不把弱信号当强认证**：指纹只用于风险评分，不作为唯一身份。
4. **收窄广播与日志**：系统状态广播不带敏感网络字段；日志默认脱敏。
5. **随机化要覆盖旁路**：MAC 随机化需同时考虑扫描、关联与芯片侧特性。
6. **特权接口严校验**：厂商扩展服务必须校验 uid 与权限，敏感字段按需返回。
7. **政策与代码一致**：`QUERY_ALL_PACKAGES` 等高敏权限只在确有必要时声明。

## 11. 常见误传纠正

### 误传一：卸载重装后 `ANDROID_ID` 一定变

错。同签名、同用户、同设备上不变；只有换签名、恢复出厂或换用户才变。

### 误传二：开飞行模式就能阻止设备指纹

错。Build 字段、传感器清单、字体、屏幕参数等仍可在本地读取，组合指纹不依赖网络。

### 误传三：广告 ID 是硬件标识

错。它由 Google Play services 提供，可重置、可删除；删除后返回全零串。

### 误传四：MAC 随机化后完全不可关联

错。Android 10 的持久随机按网络派生，同网络下仍可跨会话关联；只有非持久随机才显著削弱关联。

### 误传五：第三方应用仍能读 IMEI 做风控

错。Android 10 起需要 `READ_PRIVILEGED_PHONE_STATE`，第三方不可声明；targetSdk 29+ 调用抛异常。

## 12. 未核实与失效来源

- Privacy Sandbox on Android 的当前状态（官方文档标注 2025-10-17 起弃用）与替代方案未逐一核实，本文不展开。
- Widevine 设备 ID 是否可重置、跨应用是否一致，官方文档未给出统一结论，标为未核实。
- 各 OEM 对 MAC 随机化默认值的实现差异未逐机型核实。
- 「组合指纹的唯一性比例」缺少可复现的公开基准，本文不给具体数字。
- CVE-2018-9489 对应的 Android 公告月份未在 MITRE 记录中给出。

## 13. 参考资料

- 用户数据 ID 指南：
  https://developer.android.com/identity/user-data-ids
- `Settings.Secure`：
  https://developer.android.com/reference/android/provider/Settings.Secure
- Android 10 隐私变更：
  https://developer.android.com/about/versions/10/privacy/changes
- Android 6.0 变更：
  https://developer.android.com/about/versions/marshmallow/android-6.0-changes
- Wi-Fi MAC 随机化：
  https://source.android.com/docs/core/connect/wifi-mac-randomization-behavior
- `Build`：
  https://developer.android.com/reference/android/os/Build
- `MediaDrm`：
  https://developer.android.com/reference/android/media/MediaDrm
- 广告 ID 政策：
  https://support.google.com/googleplay/android-developer/answer/6048248
- Android 14 行为变更：
  https://developer.android.com/about/versions/14/behavior-changes-all
- Android 15 行为变更：
  https://developer.android.com/about/versions/15/behavior-changes-all
- NVD 条目（本文台账）：
  https://nvd.nist.gov/vuln/detail/CVE-2018-9489
  https://nvd.nist.gov/vuln/detail/CVE-2021-25344
  https://nvd.nist.gov/vuln/detail/CVE-2020-11287

---

## 联系方式

咨询 Android 系统详情请联系 https://t.me/one00190

---

*本手册仅供安全学习与防御研究使用；所有机制表述、公告列值与 CVE 描述均取自公开一手来源并附链接，未能核实的部分已单列于「未核实与失效来源」而非写入正文。*
