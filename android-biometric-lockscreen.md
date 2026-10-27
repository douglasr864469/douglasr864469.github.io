---
layout: page
title: "Android 生物识别子系统与锁屏凭据链手册"
description: "现行强度分档 Class 3/2/1 与按用途的组织方式、CDD 生物识别条款门槛、框架 API 与 HAL 版本实名、LSKF→scrypt→synthetic password→CE 密钥的实际链路与 Weaver 限频；含厂商指纹 TA/驱动侧已核实案例与误传纠正"
date: 2026-10-26
permalink: /android-biometric-lockscreen/
---

> 本手册处理两个常被混为一谈的东西：**生物识别子系统**（谁验证了你的手指/脸）与**锁屏凭据链**（LSKF 如何变成解开数据的钥匙）。二者在 KeyMint 处汇合，但强度语义、失效条件和兼容性要求完全不同。定位是防御与教育，只引用公开且已修复的信息，不含可利用代码，不描述任何绕过步骤。

---

## 0. 使用方式与边界

**与 TEE 手册的分工。** [TEE、密钥管理与设备认证手册](/android-tee-key-attestation/) 讲的是 KeyMint、`HardwareAuthToken` 的字段与共享 HMAC 密钥这条**硬件侧契约**；本手册讲的是它的**上游**：生物识别按强度怎么分档、框架与 HAL 怎么分工、以及 PIN/图案/密码是怎么被拉伸并限频的。两份在 `USER_SECURE_ID` / 认证令牌处对接，不重复推导。

**怎么用。**
- 判断某类生物识别能否用于某个安全目的 → §1 的三档语义与 §2 的 CDD 门槛
- 核对 API 用法与废弃情况 → §3（含两处常被写错的类名）
- 搞清楚"锁屏凭据"与"CE 数据密钥"的实际关系 → §4
- 核对某条指纹类 CVE 的真实归属 → §6，以及 §7 的误传纠正

**边界。** 只讨论公开机制与已修复缺陷；不涉及传感器欺骗、样本制作、固件提取等任何实操内容。厂商 TA/驱动的内部实现不属本手册范围。

---

## 1. 强度分档：现行术语是 Class，不是 Strong/Weak

官方现行命名（[Biometrics](https://source.android.com/docs/security/features/biometric)）：

> "The Android CDD defines three classes of biometric strength: **Class 3 (formerly Strong), Class 2 (formerly Weak), and Class 1 (formerly Convenience)**."

这一点必须写在最前面，因为它同时出现在公告、CDD、代码注释和旧教程里，术语换代是很多讨论对错分的根源：**"Strong biometric" 现在等价于 Class 3，"Weak" 是 Class 2，"Convenience"（含 face 一类）是 Class 1。**

官方同一页还按**用途**来组织要求，而不是按模态：Lockscreen / BiometricPrompt Integration / Keystore (time-based key) / Keystore (operation-based key)。也就是说"能不能解锁设备""能不能签密钥""能不能出现在 BiometricPrompt 里"是三个独立判定，不要用一个"支不支持指纹"概括。

应用侧另有认证目的常量 `AUTHENTICATION_PURPOSE_LOGIN` / `AUTHENTICATION_PURPOSE_PAYMENT` / `AUTHENTICATION_PURPOSE_UNKNOWN`（较新版本 API 引入，见 [source.android.com/docs/security/features/biometric](https://source.android.com/docs/security/features/biometric)）。这是"支付级"与"登录级"应当区别对待的官方锚点。

**官方对生物识别的定位有明确成文表述：**

> "Biometrics offer a more convenient, but potentially less secure way of confirming your identity"

以及 CDD 9.11.1 的分层："a knowledge factor-based primary authentication can be backed either by a secondary strong biometric, or by weaker tertiary modalities"（[CDD](https://source.android.com/docs/compatibility/17/android-17-cdd?hl=en)）。→ **"凭据为主、生物识别为辅"是成文要求，不是我的推断。**

---

## 2. CDD 门槛：哪档能做什么

生物识别条款在 **CDD 7.3.10 Biometric Sensors**，锁屏与密钥在 **9.11.1 / 9.11.2**。

**先纠一个常见错引：7.3.1 是 Accelerometer（加速度计），不是生物识别。**

已核实的条款要点（Android 17 CDD，[原文](https://source.android.com/docs/compatibility/17/android-17-cdd?hl=en)）：

| 条款 | 要求（要点） |
|---|---|
| [C-1-1] / [C-1-11] | 定义 Class 1 的 FAR / SAAR 上限 |
| [C-3-2] | Class 3 **MUST** 有 hardware-backed keystore 支撑 |
| [C-3-4] | Class 3 需周期性回落到主认证（**72 小时**量级） |
| [C-3-5] | 重新录入生物特征时 **MUST** 重新生成 Authenticator ID |
| [C-5-3] | 非 Class 3 生物识别 **MUST NOT** be treated as a secure lock screen |
| [C-6-2] | 当调用方要求 `BIOMETRIC_STRONG` 或传入 `CryptoObject` 时，**MUST** 只呈现 Class 3 生物识别 |
| [C-SR-17] | 建议实现新的 AIDL 接口（`IFace.aidl` / `IFingerprint.aidl`） |

**读法。** [C-3-5] 才是"重录指纹会不会让旧授权失效"的**规范层答案**：Authenticator ID 重生成，使得绑定到该 ID 的密钥授权随之失效，这是兼容性要求而非厂商可选行为。工程上常见的疑问"我新加了手指，旧密钥还能不能用"因此有了可验证的判据。

[C-6-2] 则是应用侧最容易踩的坑：要求 STRONG 或传 `CryptoObject` 时，系统**必须**只允许 Class 3 参与。若某设备在这里放行了 Class 2，那是合规缺陷，不是"配置宽松"。

---

## 3. 框架 API 与 HAL 现状

### 3.1 能力查询与常量

`canAuthenticate()`（无参版）在 API level 30 废弃，现行是 `canAuthenticate(int)`。返回常量：`BIOMETRIC_SUCCESS`、`BIOMETRIC_ERROR_HW_UNAVAILABLE`、`BIOMETRIC_ERROR_NO_HARDWARE`、`BIOMETRIC_ERROR_NONE_ENROLLED`、`BIOMETRIC_ERROR_IDENTITY_CHECK_NOT_ACTIVE`、`BIOMETRIC_ERROR_NOT_ENABLED_FOR_APPS`、`BIOMETRIC_ERROR_SECURITY_UPDATE_REQUIRED`。

来源：[BiometricManager](https://developer.android.com/reference/android/hardware/biometrics/BiometricManager)

最后一个 `BIOMETRIC_ERROR_SECURITY_UPDATE_REQUIRED` 值得注意：安全更新缺失会让生物识别**整体不可用**，这是官方在框架层给出的"补丁状态影响认证能力"的直接信号，可作观测点（§5）。

### 3.2 认证器常量：宿主类是 BiometricManager（常见写错）

**纠正一处流传很广的错误：常量类是 `BiometricManager.Authenticators`，不是 `BiometricPrompt.Authenticators`。**

| 常量 | 值 |
|---|---|
| `BIOMETRIC_STRONG` | 15 |
| `BIOMETRIC_WEAK` | 255 |
| `DEVICE_CREDENTIAL` | 32768 |
| `IDENTITY_CHECK` / `LESS_THAN_STRONG` | 后续 API 版本新增 |

来源：[BiometricManager.Authenticators](https://developer.android.com/reference/android/hardware/biometrics/BiometricManager.Authenticators)

组合语义官方写得很直白："Types may combined via bitwise OR into a single integer"，且 "this is a superset of BIOMETRIC_STRONG and is defined such that `BIOMETRIC_STRONG | BIOMETRIC_WEAK == BIOMETRIC_WEAK`"。

→ **注意这个反直觉点：同时给 STRONG 和 WEAK，结果是 WEAK 的语义（更宽松），不是"取强"。** 把 `BIOMETRIC_STRONG | BIOMETRIC_WEAK` 当成"优先强认证"是常见误用，实际放低了要求。

至于网上偶见的 "concurrency / stacking" 术语，官方页面没有这个说法，只有按位 OR 组合的表述 → 标未核实。

### 3.3 device credential 的正确接法

`setDeviceCredentialAllowed(boolean)` 已不在现行 Builder 方法列表里（仅历史注记），替代是 `setAllowedAuthenticators(DEVICE_CREDENTIAL)`；`KeyguardManager.createConfirmDeviceCredentialIntent` "deprecated in API level 29. see BiometricPrompt.Builder.setAllowedAuthenticators(int)"。

来源：[BiometricPrompt](https://developer.android.com/reference/android/hardware/biometrics/BiometricPrompt) · [KeyguardManager](https://developer.android.com/reference/android/app/KeyguardManager)

### 3.4 HAL 形态（含一处纠正）

`hardware/interfaces/biometrics/` 现含 `common/`、`face/`、`fingerprint/` 三个目录：

- fingerprint：HIDL **2.1 / 2.2 / 2.3** + `aidl/`
- face：HIDL **1.0** + `aidl/`
- **main 分支下没有 `iris/` 目录**

AIDL 接口实名 `IFace.aidl` / `IFingerprint.aidl` / `ISession.aidl`；common AIDL 有 `SensorStrength.aidl`、`OperationState.aidl` 等。**"BiometricSensor / BiometricFace / BiometricIris" 这类命名并不存在**——如果你的资料或脚本里用了这套名字，需要改。

来源：[biometrics/](https://android.googlesource.com/platform/hardware/interfaces/+/main/biometrics/) · [fingerprint/](https://android.googlesource.com/platform/hardware/interfaces/+/main/biometrics/fingerprint/) · [common/](https://android.googlesource.com/platform/hardware/interfaces/+/main/biometrics/common/aidl/android/hardware/biometrics/common/)

---

## 4. 锁屏凭据链：LSKF → synthetic password → 数据密钥

这是本手册最容易被讲错的环节。术语先纠正：**LSKF = Lock Screen Knowledge Factor**（PIN / 图案 / 密码），不是"Lock Screen Specific Form"之类；管理它的服务实名 **`LockSettingsService`**（system_server 内），"LSS" 不是一手缩写。

### 4.1 官方描述的链路

1. 用户设置 LSKF。
2. `LockSettingsService` 用 scrypt 拉伸：官方给出目标参数——"stretches the LSKF by passing it through scrypt, targeting a time of about **25 ms** and a memory usage of about **2 MiB**"。
3. 拉伸后的值构成 **synthetic password** 的一部分；synthetic password 再经 **Weaver**（SE/TEE 内的高熵 secret）或 **Gatekeeper** 做双重加密。
4. 数据侧：`vold encrypts each CE key for internal storage using an AES-256-GCM key derived from the user's synthetic password`。

来源：[File-Based Encryption](https://source.android.com/docs/security/features/encryption/file-based?hl=en) · [Weaver](https://source.android.com/docs/security/features/authentication/weaver)

### 4.2 关键澄清：认证令牌不是磁盘钥匙

官方对 CE 磁盘密钥的表述是 **synthetic password 派生**，而 Gatekeeper 认证令牌的作用是告知 Keystore「authentication-bound keys 现在可用」。

> "CE keys … cannot be unlocked without knowing either the user's Lock Screen Knowledge Factor (LSKF)"

**为什么这个区分重要：** 把"攻破 Gatekeeper/生物识别"直接等价于"解密 /data"是范围误判。CE 磁盘密钥的强度取决于 synthetic password 派生链与 LSKF 自身强度；而某个认证绑定密钥能否被使用，取决于令牌与 Tag 策略。二者影响半径不同（这与 [RCE 与提权手册](/android-rce-and-privilege-escalation/) 的影响上下文阶梯一致）。

### 4.3 Weaver 与限频

Weaver 的定位（官方）："provides SE or TEE-enforced rate-limiting of LSKF guesses"；`LockSettingsService` 使用 weaver slot（key/value）做限频，**没有 Weaver 时回退到 Gatekeeper**。

同一文档还承认弱凭据的天花板：对弱 LSKF 的拉伸 "usually does not provide much security"。→ 这是官方对"25 ms / 2 MiB 的 scrypt 参数在弱 PIN 面前意义有限"的明文表态，比任何第三方推断都更适合引用。

**未核实项：** `spfile`、`secstorage`、"spinning release" 这三个词在 FBE 页均未出现，我不对其作断言（§8）。

### 4.4 Smart Lock / Trust Agent 现状（谨慎表述）

`developer.android.com/reference/android/service/TrustAgentService` 与 AOSP `core/java/android/service/boservice/` **均返回 404**，即现行公开 API 参考中已无此类。但 Google 并未给出"已移除"的正式声明页面，因此**"Smart Lock 已废弃/已移除"这一结论我标为未核实**，只能陈述"公开参考中不可达"这一可验证事实。

另外 `KeyGenParameterSpec.setUserAuthenticationValidWhileOnBody` 仍在 API 且未见废弃注记——所以"on-body 一律已废弃"同样不成立。

---

## 5. 观测点与自查

| 观测 | 真实形态与来源 |
|---|---|
| 认证能力是否可用 | `BiometricManager.canAuthenticate(int)` 的返回值；`BIOMETRIC_ERROR_SECURITY_UPDATE_REQUIRED` 直接指示安全更新缺失 |
| 密钥授权类型 | `KeyProperties.AUTH_BIOMETRIC_STRONG` 等（Authenticators 页注明 "This corresponds to `KeyProperties.AUTH_BIOMETRIC_STRONG` during key generation"） |
| 授权时长语义 | `setUserAuthenticationParameters(int timeout, int type)`："Sets the duration of time (seconds) and authorization type for which this key is authorized to be used after the user is successfully authenticated"；旧 `setUserAuthenticationValidityDurationSeconds(int)` 在 API 30 废弃 |
| 所需权限 | `Manifest.permission.USE_BIOMETRIC`（"Allows an app to use device supported biometric modalities"）；`USE_FINGERPRINT` 在 API 28 废弃并指向 `USE_BIOMETRIC`；设备管理侧 `BIND_DEVICE_ADMIN` |
| HAL 代际 | 检查 `face` / `fingerprint` 目录下 HIDL 版本与是否存在 AIDL 实现（§3.4） |

来源：[KeyGenParameterSpec.Builder](https://developer.android.com/reference/android/security/keystore/KeyGenParameterSpec.Builder) · [Manifest.permission](https://developer.android.com/reference/android/Manifest.permission)

### 自查清单（面向 defenders）

1. 应用是否误用 `BIOMETRIC_STRONG | BIOMETRIC_WEAK`（= 放低要求）？
2. 是否把 Class 1/2 生物识别当作"安全锁屏"（违反 [C-5-3]）？
3. 绑定密钥的 operation-based 场景是否只用 Class 3（[C-6-2]）？
4. 是否假设"设备没有 StrongBox"就放弃硬件绑定，而没有回落到 TEE 档？（`SecurityLevel` 三档见 TEE 手册）
5. 审计机型时，是否记录了 HAL 是 HIDL 2.x 还是 AIDL（影响认证器 ID 与限频实现位置）？

---

## 6. 公开案例复盘（逐条已核实归属）

| CVE | 组件/位置（公告原述） | 影响 / 级别 | Android ID | 来源 |
|---|---|---|---|---|
| CVE-2026-55323 | **Goodix Fingerprint TA**（TEE 内 TA） | EoP / Critical | A-512579485 | [Pixel 2026-09](https://source.android.com/docs/security/bulletin/pixel/2026/2026-09-01?hl=en) |
| CVE-2026-0199 | **Goodix Fingerprint TA** | EoP / Critical | A-512579927 | 同上（SPL 2026-09-05） |
| CVE-2024-32926 | **Fingerprint Sensor** | ID / Moderate | A-315167037 | [Pixel 2024-06](https://source.android.com/docs/security/bulletin/pixel/2024-06-01?hl=en) |
| CVE-2026-56958 | 组件 **"Biometrics"**（框架侧） | ID / High | A-512580365 | [Pixel 2026-09](https://source.android.com/docs/security/bulletin/pixel/2026/2026-09-01?hl=en) |
| CVE-2017-0518 | "An elevation of privilege vulnerability in the **Qualcomm fingerprint sensor driver**" | EoP | — | [NVD](https://nvd.nist.gov/vuln/detail/CVE-2017-0518) · [2017-03-01 公告](https://source.android.com/security/bulletin/2017-03-01) |
| CVE-2017-0519 | 同上（Qualcomm 指纹传感器驱动） | EoP | — | 同上 |
| CVE-2016-3917 | Android 6.0.1 **指纹登录未跟踪用户账号** | 认证判定缺陷 | internal bug 30744668 | [2016-10-01 公告](http://source.android.com/security/bulletin/2016-10-01.html) |

三条读法：

1. **归属要落到"哪一侧"。** Goodix 两条位于 **TA（TEE 内应用）**，CVE-2017-0518/0519 位于**内核态驱动**，CVE-2016-3917 位于**框架判定逻辑**。同一句"指纹漏洞"在这三处意味着完全不同的修复通道与影响半径——这正是 [编号与公告手册](/android-cve-and-security-bulletins/) 强调按 description 逐字对齐的原因。
2. **公告表结构会坑人。** Pixel 机型专属公告**没有 "Android version" 列**（AOSP 公告才有），跨表统计覆盖版本时不要假定同构。
3. **"没有条目"也是信息。** NVD 关键词 `BiometricPrompt` 命中 **0 条**；抽查 AOSP 公告 2025-12-01、2024-06-01、2025-09-01、2026-03-01 四期内**没有** Fingerprint/Biometric/Face/Iris 组件行——本域的公开条目集中在**厂商侧**（Pixel 公告、高通公告），而不是 AOSP 公告。做覆盖率统计时把两类公告分开算，否则会系统性低估。

**KEV 核对**（逐条 `field_cve` 精确查）：CVE-2026-55323 → 无；CVE-2017-0518 → 无。
**检索陷阱实证：** `?field_cve=CVE-2025-22224` 命中的是 **VMware ESXi/Workstation 的 TOCTOU 条目**（Added 2025-03-04），与 Android 无关。→ **CVE 号跨厂商碰撞真实存在，KEV 结论必须核对 Resource/Product 列。**

---

## 7. 常见误传纠正

| 流传说法 | 核实结果 |
|---|---|
| "Class 3 就是旧的 Strong，可以随便写" | 术语对应没错，但**公告与 CDD 现行文本用 Class**；写规则时用 Strong 会漏掉 CDD 条款匹配 |
| `BiometricPrompt.Authenticators` | 宿主类是 **`BiometricManager.Authenticators`** |
| CDD 生物识别在 **7.3.1** | 7.3.1 是 **Accelerometer**；生物识别是 **7.3.10 Biometric Sensors** |
| 存在 `BiometricSensor` / `BiometricIris` HAL | 实名 `IFace.aidl` / `IFingerprint.aidl` / `ISession.aidl`；main 分支**无 iris 目录** |
| `Strong | Weak` 取强 | 官方原话结果为 `BIOMETRIC_WEAK`，是**放低要求** |
| `setInvalidatedByBiometricEnrollment` 在 `KeyguardManager` 上 | 它是 **`KeyGenParameterSpec.Builder`** 的方法；KeyguardManager 上没有，查错类会得出"该 API 不存在"的错误结论 |
| "Smart Lock 已官方废弃" | 公开参考与 AOSP 包目录 **404**（不可达），但无正式废弃声明 → 未核实 |
| "指纹漏洞多在 AOSP 公告里" | 抽查四期 AOSP 公告**无**相关行；条目集中在 Pixel/厂商公告 |
| LSKF = "Lock Screen Specific Form" | **Lock Screen Knowledge Factor** |

---

## 8. 未核实与来源失效（不作断言）

1. **`spfile` / `secstorage` / "spinning release"** — FBE 页无这些词，未找到一手页。
2. **"concurrency / stacking"** 官方术语 — 未见，仅有按位 OR 组合表述。
3. **face / iris 解锁的公开 CVE** — 四期公告与 NVD 关键词均未命中可引用条目。
4. **三星 / 联发科指纹侧公开条目** — NVD 检索 "Samsung fingerprint Android" 仅命中 Kryptowire 的 build-fingerprint 相关描述，与指纹子系统无关 → 未核实。
5. **`IDENTITY_CHECK` / `LESS_THAN_STRONG` 的确切引入 API level** — 只确认它们是较新版本常量，未逐字取到版本号。
6. **Smart Lock / on-body 的正式废弃声明** — 无一手页面（尝试 `developer.android.com/reference/android/service/TrustAgentService`、AOSP `service/boservice/`，均 404）。
7. **失效 URL**：`source.android.com/docs/security/features/seccomp`、`.../selinux/domains`（404）；旧公告路径 `/bulletin/2025-01-05`（404）。
8. **`BIOMETRIC_STRONG` 数值 15 / `BIOMETRIC_WEAK` 255 / `DEVICE_CREDENTIAL` 32768** 取自 developer.android.com 常量表，若后续版本调整需以页面为准。

---

## 9. 与文档库其他部分的分工

| 需要 | 去看 |
|---|---|
| KeyMint 档位、`HardwareAuthToken` 字段、共享 HMAC 密钥、CE/DE 密钥保护 | [TEE 与密钥认证手册](/android-tee-key-attestation/) |
| 公告字段读法、bug ID 与 CVE 对齐 | [编号与公告手册](/android-cve-and-security-bulletins/) |
| 缺陷定级与处置优先级 | [评估与定级手册](/android-vulnerability-severity-assessment/) |
| 分层缺陷清单 | [通用漏洞类型手册](/android-common-system-vulnerabilities/) |
| 本文 | 生物识别强度分档、HAL 与框架契约、LSKF→synthetic password 链、厂商侧案例归属 |

---

## 参考

- [Biometrics（强度分档与用途）](https://source.android.com/docs/security/features/biometric)
- [Weaver](https://source.android.com/docs/security/features/authentication/weaver) · [File-Based Encryption](https://source.android.com/docs/security/features/encryption/file-based)
- [BiometricManager](https://developer.android.com/reference/android/hardware/biometrics/BiometricManager) · [BiometricManager.Authenticators](https://developer.android.com/reference/android/hardware/biometrics/BiometricManager.Authenticators) · [BiometricPrompt](https://developer.android.com/reference/android/hardware/biometrics/BiometricPrompt) · [KeyguardManager](https://developer.android.com/reference/android/app/KeyguardManager)
- [biometrics HAL 目录](https://android.googlesource.com/platform/hardware/interfaces/+/main/biometrics/) · [fingerprint](https://android.googlesource.com/platform/hardware/interfaces/+/main/biometrics/fingerprint/) · [common AIDL](https://android.googlesource.com/platform/hardware/interfaces/+/main/biometrics/common/aidl/android/hardware/biometrics/common/)
- [Android 17 CDD](https://source.android.com/docs/compatibility/17/android-17-cdd?hl=en)
- [Pixel 2026-09 公告](https://source.android.com/docs/security/bulletin/pixel/2026/2026-09-01?hl=en) · [Pixel 2024-06 公告](https://source.android.com/docs/security/bulletin/pixel/2024-06-01?hl=en) · [2017-03-01 公告](https://source.android.com/security/bulletin/2017-03-01) · [2016-10-01 公告](http://source.android.com/security/bulletin/2016-10-01.html)
- [NVD CVE-2017-0518](https://nvd.nist.gov/vuln/detail/CVE-2017-0518) · [CISA KEV 精确检索](https://www.cisa.gov/known-exploited-vulnerabilities-catalog?field_cve=CVE-2026-55323)

---

## 联系方式

咨询 Android 系统详情请联系 https://t.me/one00190

---

*本手册仅供安全学习与防御研究使用；所有条款号、API 名与案例归属均取自公开一手来源并附链接，未能核实的表述已单列于 §8 而非写入正文。*
