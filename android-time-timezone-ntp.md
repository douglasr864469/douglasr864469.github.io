---
layout: page
title: "Android 时间、时区与网络授时的安全边界手册"
description: "把「现在几点」拆成 RTC／墙钟／单调时钟／授时源四路：`/dev/alarm` 在 Android 8.0 被内核 alarmtimer 取代、AOSP 的 SNTP 客户端无认证且未实现 NTS、NITZ 的信任基础是运营商、Android 12 起默认 NTP 优先于 NITZ；权限侧讲清 `SE..."
date: 2026-11-05
permalink: /android-time-timezone-ntp/
---

> 时间在系统里是最不起眼、却最常被安全逻辑引用的输入：TLS 证书有效期、令牌过期、日志取证、许可证窗口、双因素验证码，都默认「设备时钟大致正确」。Android 把时间拆成 RTC、内核 alarmtimer、NTP、NITZ 与时区数据库几条路径，每条都有独立的信任假设。一旦某条路径被污染或回拨，受影响的不只是显示时间，而是整条依赖时间的判断链。这份手册沿「时间从哪来 → 谁能改 → 授时怎么被污染 → 公开缺陷 → 怎么防」拆解这条边界。只讨论公开且已修复的问题，不含攻击代码。

## 0. 使用方式与边界

面向三类读者：

- **应用开发者**：需要知道本地时间不可信，安全判断应放在服务端或用单调时钟。
- **系统与 ROM 工程师**：需要区分 RTC、alarmtimer、NTP、NITZ 与时区数据的更新路径。
- **安全评估者**：需要在看到「时间绕过」类问题时，判断缺陷落在权限、授时源、时区解析还是业务依赖。

三条自我约束：

1. 只引用公开且已修复的问题；CVE 描述以 NVD 与厂商公告为准。
2. 不写修改系统时间的步骤、不写授时欺骗配置。
3. 查不到一手来源的说法放进「未核实与失效来源」。

与已发布手册的分工：

- 《Android 电源管理、唤醒锁与闹钟调度手册》讲 `AlarmManager` 的调度与唤醒；本文只讲时间基准与授时。
- 《Android Verified Boot 与 dm-verity 验证链手册》讲启动完整性；本文只讲时间回拨对其**没有**影响的部分。
- 《Android TEE、密钥与证明体系手册》讲证明结构；本文只讲证明中的时间戳如何被使用。

## 1. 时间的四个来源

| 来源 | 载体 | 特点 |
|------|------|------|
| RTC | 硬件时钟 | 断电后仍走时，精度有限，可被漂移 |
| 系统时钟 | 内核 `CLOCK_REALTIME` | 可被设置，受 NTP/NITZ 调整 |
| 单调时钟 | `CLOCK_MONOTONIC` / `elapsedRealtime` | 不受墙钟调整影响，适合计时 |
| 授时源 | NTP、NITZ | 网络或运营商提供，需信任判断 |

启动时，系统用 RTC 初始化墙钟，随后由授时机制校正。应用侧最常犯的错误，是用墙钟测量「经过了多久」——墙钟会被校正甚至回拨，单调时钟不会。

## 2. 内核与框架：从 `/dev/alarm` 到 alarmtimer

早期 Android 通过 `/dev/alarm` 向内核设置闹钟；Android 8.0 移除了这套用户态支持，改用内核 `alarmtimer` 提供的 timerfd。`AlarmManager` 的定时器可以在系统休眠（suspend）期间唤醒设备，这也是它与普通 `Handler` 定时器的本质区别。

需要区分两类闹钟：

- **墙钟闹钟**（`setExact`、`setAlarmClock`）：按「几点几分」触发，受系统时间调整影响；
- **经过时间闹钟**（`setExactAndAllowWhileIdle` 的 elapsed 变体、`elapsedRealtime` 基准）：按「多久之后」触发，不受墙钟回拨影响。

安全相关的定时任务（令牌刷新、证书检查、重试退避）应优先使用经过时间基准，避免用户改时间导致任务提前或延后。

## 3. 网络授时：NTP 与 NITZ

### 3.1 NTP

AOSP 的 SNTP 客户端是 `frameworks/base/core/java/android/net/SntpClient.java`（`@hide`），实现 SNTPv3，走 UDP 123，**没有认证**。它由 `NtpTrustedTime` 与 `NetworkTimeUpdateService` 调用，默认配置指向 Google 的授时服务器，超时约 5 秒，轮询间隔约 18 小时。

关键结论：**NTP 响应本身没有密码学认证**。NTPv4 只有可选的对称密钥 MAC，真正提供认证加密的是 NTS（RFC 8915），而 AOSP 客户端并未实现 NTS。因此「NTP 是可信授时」只在网络路径可信的前提下成立；在可被中间人的网络里，时间可被引导。

### 3.2 NITZ

NITZ（Network Identity and Time Zone）由运营商在信令中下发，电话框架解析为 `NitzData`，再形成 `TelephonyTimeZoneSuggestion` 送入时间检测器。它的信任基础是「运营商网络可信」，在伪基站或漫游场景下并不总是成立。

Android 12 起，时间源优先级可配置，默认顺序是 network（NTP）先于 telephony（NITZ）。这意味着 NITZ 更多是兜底，而不是主授时源。

### 3.3 设置键与 API

- `Settings.Global.AUTO_TIME`（键名 `auto_time`）；
- `Settings.Global.AUTO_TIME_ZONE`（键名 `auto_time_zone`）；
- Android 12 起建议改用 `TimeManager` 查询与监听。

## 4. 谁能改时间

修改系统时间不是普通应用能做的事：

| 能力 | 权限 | 保护级别 |
|------|------|----------|
| 设置系统时间 | `android.permission.SET_TIME` | signature / privileged / role |
| 设置时区 | `android.permission.SET_TIME_ZONE` | signature / privileged / role |
| 强制自动时间 | `DevicePolicyManager.setAutoTimeEnabled` | 设备所有者/资料所有者（API 30） |
| 强制自动时区 | `DevicePolicyManager.setAutoTimeZoneEnabled` | 同上 |
| 禁止用户改时间 | `DISALLOW_CONFIG_DATE_TIME` | 设备策略 |

`AlarmManager.setTime()` 与 `setTimeZone()` 分别要求前两项权限。传统 device admin 不足以调用 DPM 的自动时间接口，必须是 device owner 或 profile owner。

这组权限设计说明：**平台把「改时间」视为高敏感操作**，因为时间会影响证书、令牌与审计。

## 5. 时区数据：Mainline 的独立更新通道

从 Android 10 起，时区数据（tzdata）以 APEX 模块形式随 Mainline 更新，不再依赖整机 OTA。这带来两个安全含义：

- 时区规则变更（如某国取消夏令时）可以较快下发；
- 但更新渠道本身成为供应链的一部分，模块签名与版本回滚保护必须成立。

时区错误不会直接导致提权，却会让「按本地时间」的业务规则错位：例如交易截止时间、许可证窗口、考勤与日志时间线。评估时应把 tzdata 版本纳入基线。

## 6. 时间在安全判断中的位置

### 6.1 TLS 证书

RFC 5280 用 notBefore/notAfter 判定有效期。回拨本地时钟确实可能让**本地**校验接受已过期证书，但：

- 服务端仍会校验自己的时间；
- 证书固定（pinning）与撤销检查不受本地时间影响；
- 现代客户端还会校验证书链与签名算法。

因此「改时间就能绕过 HTTPS」是过度结论，但在**只做本地校验**的场景里确实成立。

### 6.2 Play Integrity 与密钥证明

Play Integrity 的判决由 Google 签名，包含 `timestampMillis`。正确做法是**服务端校验判决的新鲜度**；如果客户端用本地时间判断「这份判决是否过期」，回拨就能延长旧判决的有效期。

密钥证明的 `RootOfTrust` 包含 `verifiedBootState`、`deviceLocked`、`verifiedBootHash`，由 TEE 签名；`ro.boot.verifiedbootstate` 是 bootloader 在启动时设置的只读属性。**时间回拨不会改变这些字段**；AVB 的回滚保护使用 rollback index，也不依赖实际时间。

### 6.3 日志与取证

设备日志的时间戳来自墙钟，可被修改。取证时应结合：

- 单调时钟与启动次数；
- 服务端日志的交叉时间线；
- 安全启动状态与证明记录。

单靠设备本地时间戳，不足以证明事件顺序。

### 6.4 TOTP 与许可证

TOTP（RFC 6238）以 30 秒为步长。回拨本机时间只会让本机生成的验证码失配，真正的防护在服务端：限制验证窗口、记录失败次数、检测时间漂移。DRM 许可证的时间窗由 CDM 与服务端共同判定，本地时钟的作用因实现而异，**未逐一核实**。

## 7. 公开公告台账

| CVE | 落点 / 类型 | 影响 | 修复 | 说明 |
|-----|-------------|------|------|------|
| [CVE-2016-3831](https://nvd.nist.gov/vuln/detail/CVE-2016-3831) | telephony 处理 NITZ 时间值不当（远程 DoS） | Android 4.x–6.0 | 2016-08 | 传入 2038-01-19 或更晚的时间可致崩溃 |
| [CVE-2014-10044](https://nvd.nist.gov/vuln/detail/CVE-2014-10044) | 高通 time daemon 未授权修改系统时间（数组越界） | 多款骁龙 | 2018-04 | 编号年份早于公告 |
| [CVE-2023-42559](https://nvd.nist.gov/vuln/detail/CVE-2023-42559) | 三星 Knox Guard 异常处理不当，改时间可绕过锁定 | Android 11–14 | 2023-12 SMR | 厂商组件 |
| [CVE-2019-15615](https://nvd.nist.gov/vuln/detail/CVE-2019-15615) | Nextcloud Android 应用时间检查错误，回拨时间绕过锁 | 应用 3.9.0 | 2020-02 | 应用层示例，非系统组件 |

读表要点：

1. **授时输入是外部输入**：NITZ 来自网络，必须做范围与边界校验（CVE-2016-3831）。
2. **时间接口的权限校验不能少**：厂商守护进程缺校验会导致越权改时间（CVE-2014-10044）。
3. **业务把时间当安全条件时要谨慎**：Knox Guard 与应用锁的绕过都源于「时间可信」这一假设（CVE-2023-42559、CVE-2019-15615）。
4. **本次检索未确认** AOSP `SntpClient` 或 tzdata 解析类的公开 CVE，不排除存在但未被公开归类的可能。

## 8. 成因模式

### 8.1 外部时间值未做范围校验

NITZ 的时间字段来自网络，若直接转换为时间戳并参与定时器计算，超出范围的值会引发溢出或异常（CVE-2016-3831）。修复要点是限制合理区间并处理边界。

### 8.2 修改时间的接口缺权限

厂商守护进程若暴露了「设置时间」的 IPC 而未校验调用者，普通进程就能改时间（CVE-2014-10044）。这类问题常出现在定制服务里，不在 AOSP 公告范围。

### 8.3 业务逻辑依赖墙钟

锁、许可证、试用期若只比较本地墙钟，回拨即可延长（CVE-2023-42559、CVE-2019-15615）。正确做法是服务端时间 + 单调计时 + 防回滚计数。

### 8.4 授时路径可被中间人

未认证的 NTP 与不可信的 NITZ 都可能被引导。修复不是「换一个服务器」，而是端到端信任设计：NTS、VPN、受信网络或服务端时间。

### 8.5 时区数据滞后

tzdata 过期会导致规则错误。它不会造成提权，但会让依赖本地时间的合规判断出错。

## 9. 版本与加固时间线

| 版本 / API | 变化 |
|------------|------|
| 8.0 / 26 | 移除 `/dev/alarm` 用户态支持，改用 alarmtimer |
| 10 / 29 | tzdata 改为 APEX，随 Mainline 更新 |
| 11 / 30 | DPM 自动时间/时区接口（API 30） |
| 12 / 31 | 授权来源优先级可配置；推荐使用 `TimeManager` |
| 14 / 34 | 企业策略可禁用 UWB 等无线电（与时间无直接关系，列作同期变化） |

趋势是**把时间从「设备本地状态」变成「可配置信任源 + 可独立更新的数据模块」**。

## 10. 观测点与自查

### 10.1 应用侧

- 安全判断是否依赖本地墙钟？是否改为服务端时间或单调时钟？
- 计时是否使用 `elapsedRealtime`，而不是 `System.currentTimeMillis()` 差值？
- 证书、令牌、判决的新鲜度是否在服务端校验？
- 是否处理时间跳变（前进/回拨）导致的重试风暴或任务错乱？

### 10.2 平台与 ROM 侧

- `SET_TIME` / `SET_TIME_ZONE` 是否只授予系统与特权应用？
- 厂商时间服务是否校验调用者？
- NTP 服务器配置是否可控、是否走可信网络？
- NITZ 解析是否限制时间范围与字段长度？
- tzdata 模块版本是否及时更新？

### 10.3 评估侧

- 先定位落点：权限 / 授时源 / 解析 / 业务依赖 / 时区数据；
- 区分「时间被改」「时间被引导」「业务假设错误」；
- 不要把「能改时间」直接等同于「能绕过所有时间校验」。

## 11. 防御要点

1. **服务端时间为准**：所有安全有效期、令牌、证书与判决新鲜度都在服务端判定。
2. **单调时钟计时**：间隔与超时用 `elapsedRealtime`，不用墙钟差值。
3. **限制改时间权限**：`SET_TIME` 仅授予系统组件；厂商服务必须校验调用者。
4. **校验外部时间值**：NITZ/NTP 结果做范围与合理性检查。
5. **授时走可信路径**：高保证场景使用受信网络、VPN 或支持 NTS 的方案。
6. **企业强制自动时间**：用 DPM 的自动时间/时区接口与 `DISALLOW_CONFIG_DATE_TIME`。
7. **保持 tzdata 更新**：把时区模块版本纳入设备基线。

## 12. 常见误传纠正

### 误传一：普通应用可以随意修改系统时间

错。需要 `SET_TIME` 或 `SET_TIME_ZONE`，保护级别为 signature/privileged/role。

### 误传二：关闭自动时间就能绕过证书过期

错。关闭自动时间不会改变时钟；手动回拨只可能骗过**本地**校验，绕不过服务端校验、证书固定与撤销检查。

### 误传三：NTP 响应经过认证

错。AOSP 的 SNTP 客户端没有认证；NTS 才提供认证加密，AOSP 未实现。

### 误传四：改时间可以伪造密钥证明或启动状态

错。`RootOfTrust` 与 `ro.boot.verifiedbootstate` 由 TEE/bootloader 产生，AVB 回滚保护用 rollback index，不依赖墙钟。

### 误传五：时间只影响显示

错。证书、令牌、日志、TOTP、许可证、定时任务都依赖时间；错误的时间会产生连锁影响。

## 13. 未核实与失效来源

- AOSP 默认 NTP 服务器地址与轮询间隔可能随版本与配置变化，本文只给量级描述。
- tzdata 在主line 中的具体版本节奏未逐一核实。
- DRM 许可证时间窗对本地时钟的依赖程度因 CDM 实现而异，未逐一核实。
- 各 OEM 的时间服务实现与补丁节奏未核实。
- CVE-2019-15615 为应用层案例，列入仅为说明「业务依赖墙钟」这一模式，不代表 Android 系统缺陷。

## 14. 参考资料

- `SntpClient`（AOSP）：
  https://android.googlesource.com/platform/frameworks/base/+/master/core/java/android/net/SntpClient.java
- 时间源优先级（AOSP）：
  https://source.android.com/docs/core/connect/time-source
- 电话时区检测（AOSP）：
  https://source.android.com/docs/core/connect/time/telephony-tz-detection
- 时区数据模块（AOSP）：
  https://source.android.com/docs/core/ota/modular-system/timezone
- `Settings.Global`：
  https://developer.android.com/reference/android/provider/Settings.Global
- `AlarmManager.setTime`：
  https://developer.android.com/reference/android/app/AlarmManager#setTime(long)
- 权限清单（AOSP）：
  https://android.googlesource.com/platform/frameworks/base/+/refs/heads/main/core/res/AndroidManifest.xml
- 2016-08 公告：
  https://source.android.com/docs/security/bulletin/2016-08-01
- 2018-04 公告：
  https://source.android.com/docs/security/bulletin/2018-04-01
- Play Integrity 判决：
  https://developer.android.com/google/play/integrity/verdicts
- 密钥证明：
  https://source.android.com/docs/security/features/keystore/attestation
- RFC 5280 / RFC 8915 / RFC 6238：
  https://www.rfc-editor.org/rfc/rfc5280
  https://www.rfc-editor.org/rfc/rfc8915
  https://www.rfc-editor.org/rfc/rfc6238
- NVD 条目（本文台账）：
  https://nvd.nist.gov/vuln/detail/CVE-2016-3831
  https://nvd.nist.gov/vuln/detail/CVE-2014-10044
  https://nvd.nist.gov/vuln/detail/CVE-2023-42559
  https://nvd.nist.gov/vuln/detail/CVE-2019-15615

---

## 联系方式

咨询 Android 系统详情请联系 https://t.me/one00190

---

*本手册仅供安全学习与防御研究使用；所有机制表述、公告列值与 CVE 描述均取自公开一手来源并附链接，未能核实的部分已单列于「未核实与失效来源」而非写入正文。*
