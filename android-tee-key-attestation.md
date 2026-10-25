---
layout: page
title: "Android TEE、密钥管理与设备认证攻击面手册"
description: "沿「信任根 → 密钥存储 → 凭据认证 → 状态证明 → 存储加密」一条链展开：AVB 2.0 与四色状态、Rollback Protection、KeyMint 三档安全等级与 StrongBox 要求、Gatekeeper 与 HardwareAuthToken 实名、密钥证明的 RootOfTru..."
date: 2026-10-25
permalink: /android-tee-key-attestation/
---

> 按「信任根 → 密钥存储 → 凭据认证 → 状态证明 → 存储加密」这条链条，逐层给出组件的真实名称、接口形态、缺陷模式与已核实的公开案例。定位是防御与教育：所有内容均来自公开一手来源（AOSP 源码、Android 官方文档、厂商安全公告、NVD），不含任何可直接用于攻击的代码或操作步骤。

---

## 0. 使用方式与边界

**这份手册解决什么问题。** Android 上很多资料把 TEE、Keystore、Verified Boot 当成三个孤立话题讲，但它们实际上是一条依赖链：Verified Boot 决定设备处于什么状态，这个状态被写进密钥证明，密钥证明与密钥使用授权又依赖 KeyMint，而 KeyMint 的使用授权由 Gatekeeper/生物识别发出的认证令牌解锁，最后 FBE 的密钥保护同样落回这条链。缺陷往往出现在**接缝处**（例如令牌校验、状态传递），而不是单个组件内部。

**怎么用。**
- 想判断一个漏洞该归到哪一层 → 看 §1 的坐标表与 §7 的缺陷形态汇总
- 想核对某台设备/某个机型的实现强度 → 看 §10 的观测点（全部是真实存在的属性名与 API 名）
- 看到别人引用一个"TEE 漏洞" → 先看 §9，那里列了几条被广泛误传的归属
- 想知道某条说法我没能核实 → §11 单列了未核实与失效来源，不假装确定

**边界。** 只讨论已公开且已修复的缺陷；不涉及物理攻击实施细节、不涉及侧信道攻击操作、不提供绕过锁屏或伪造证明的方法。TEE 内部实现多为厂商私有，本手册只描述**接口契约与公开披露的缺陷形态**。

---

## 1. 一条链的五个位置

| 层 | 主要组件 | 保护的是什么 | 官方定义位置 |
|---|---|---|---|
| 启动完整性 | AVB 2.0（`vbmeta`、dm-verity）、bootloader | 运行的软件是哪一个版本 | [Verified Boot](https://source.android.com/docs/security/features/verifiedboot)、[external/avb README](https://android.googlesource.com/platform/external/avb/+/refs/heads/main/README.md) |
| 密钥存储 | KeyMint HAL（取代 Keymaster） | 密钥材料不被提取、使用受策略约束 | [Hardware-backed Keystore](https://source.android.com/docs/security/features/keystore)、[SecurityLevel.aidl](https://android.googlesource.com/platform/hardware/interfaces/+/refs/heads/main/security/keymint/aidl/android/hardware/security/keymint/SecurityLevel.aidl) |
| 凭据认证 | Gatekeeper、Fingerprint/Face 子系统 | 用户是谁、什么时候算"刚刚验证过" | [Gatekeeper](https://source.android.com/docs/security/features/authentication/gatekeeper)、[Authentication](https://source.android.com/docs/security/features/authentication) |
| 状态证明 | Key/ID attestation（`RootOfTrust`） | 让**远端**依赖方能验证设备状态 | [Key and ID attestation](https://source.android.com/docs/security/features/keystore/attestation) |
| 存储加密 | FBE（DE/CE）+ metadata encryption（`dm-default-key`） | 静态数据，以及关机后密钥是否还在 | [File-Based Encryption](https://source.android.com/docs/security/features/encryption/file-based)、[Metadata Encryption](https://source.android.com/docs/security/features/encryption/metadata) |

兼容性要求侧的锚点（Android 16 CDD）：**9.10 Device Integrity** 管启动验证、**9.11 Keys and Credentials** 管密钥存储。注意 10.2 是 CTS Verifier，与密钥存储无关，网上偶有引用错章节的情况。

---

## 2. Verified Boot：从 vbmeta 到四色状态

### 2.1 AVB 2.0 的实际结构

AVB 2.0 的核心是一个签名的 **VBMeta struct**，存放在 `vbmeta` 分区里，内部通过描述符（descriptor）描述被保护的分区：

- **hash descriptor** — 单个分区的摘要
- **hashtree descriptor** — 供 dm-verity 使用的树状结构。官方描述："the `vbmeta` partition holds the root hash, salt, and offset of the hashtree in hashtree descriptors"
- **chained partition descriptor** — 允许子分区携带自己的签名与公钥哈希，用于厂商分区独立签名的场景

AVB 1.0（Android 4.4 时代）是"启动验证 + dm-verity"的组合，官方中文文档称之为"启动时验证 1"。AVB 后续小版本陆续增加了 chain descriptor flags、`rollback_index_location`、`check_at_most_once` 等字段。

**读法。** 一个链式分区被独立签名，意味着它的完整性依赖那颗链上的公钥是否仍可信；`vbmeta` 根签名只保证"链的描述正确"，不保证"链尾的实现强"。评估厂商定制 ROM 时，链式分区数量与各自的哈希强度是看得见的差异点。

### 2.2 四色状态与锁语义

官方定义（[boot-flow](https://source.android.com/docs/security/features/verifiedboot/boot-flow)、AVB README 的 Android Specific Integration 一节）：

| 状态 | 锁状态 | 含义 | 对应 `verifiedBootState` 枚举 |
|---|---|---|---|
| green | LOCKED | 验证密钥并非终端用户设置 | `Verified(0)` |
| yellow | LOCKED | 用户自行配置了信任根 | `SelfSigned(1)` |
| orange | UNLOCKED | bootloader 已解锁 | `Unverified(2)` |
| red | — | 验证失败 | `Failed(3)` |

red 的具体表现是警告屏本身，官方文档举了两个实例文案："红色 (eio)：针对 dm-verity 损坏的警告屏幕"、"红色 (no os found)：未找到有效的操作系统"。

**关键细节。** 在 `Unverified(2)`（orange）时，`verifiedBootKey` 是 **32 字节全 0**；`Verified(0)` 时它是出厂 ROM 内的根信任哈希。这就是远端仅凭密钥材料也能区分状态的原因。

### 2.3 解锁与重锁的真实代价

AVB 官方 README 对状态转换的要求相当硬：LOCKED 与 UNLOCKED 之间的切换必须有**物理确认**，并且"所有用户数据必须清除……包括 `userdata` 分区和任何 NVRAM 空间……所有 `stored_rollback_index[n]` 位置必须清零"。

CDD 侧对应条款把这条要求延伸到了密钥：**[C-1-12]** 要求相关密钥 "MUST be securely erased during bootloader unlock and lock"。也就是说，解锁 bootloader 不只是清数据，还必须把绑定到锁状态的密钥抹掉——这是评估"解锁后旧密钥是否仍可用"的判据。

orange 状态下系统仍会启动，但会显示"引导加载程序已解锁，无法保证软件完整性……"这类提示（[locking_unlocking](https://source.android.com/docs/core/architecture/bootloader/locking_unlocking)）。

### 2.4 Rollback Protection（注意术语）

**术语纠正：没有"anti-rollback index"这种官方叫法。** 官方机制名是 **Rollback Protection**，字段是 `rollback_index[n]` 与 `stored_rollback_index[n]`。定义："having the device reject an image unless `rollback_index[n]` >= `stored_rollback_index[n]`"，且 `stored_rollback_index[n]` 存放在防篡改存储（tamper-evident storage）里。

**为什么重要。** 防回滚的强度取决于 `stored_rollback_index` 存在哪种存储里——那是厂商实现差异最大的地方，也是"降级到旧版本以绕过补丁"是否可行的分水岭。§8 会给出一个真实后果案例。

---

## 3. KeyMint：密钥存储与强制

### 3.1 版本与 HAL 形态演进

| 阶段 | Android 版本 | 形态 |
|---|---|---|
| Keymaster 2.0 | 更早 | 旧式 C 结构 HAL |
| Keymaster 3 | Android 8.0 | 转向 HIDL 生成的 C++ HAL 接口 |
| Keymaster 4.0 / 4.1 | Android 9 / 10 | HIDL |
| **KeyMint 1.0** | **Android 12** | 取代 Keymaster；AIDL server 实现 `IKeyMintDevice` |
| KeyMint 2.0+ | Android 13 起 | Android 13 加入 v2（Curve25519 支持） |

官方原话："Android 12 introduced the new KeyMint HAL, which replaces the Keymaster HAL"、"Android 13 added v2 of the KeyMint HAL"、"KeyMint HAL service — An **AIDL** server that implements the IKeyMintDevice HAL"。

密钥证明里的 `attestationVersion` 与上表对应：1=Keymaster 2.0、2=Keymaster 3.0、3=Keymaster 4.0、4=Keymaster 4.1、100=KeyMint 1.0、200=KeyMint 2.0、300/400/500=v3.0/v4.0/v5.0。这是**无需读厂商文档就能判断实现代际**的一个字段。

（Keymaster 3.0/3.1/3.2 与 HIDL 的细粒度对应关系官方文档未记载，见 §11。）

**实现归属。** KeyMint HAL 本身是 "an **OEM-provided** service"。AOSP 提供的是 Rust 参考实现（[system/keymint](https://android.googlesource.com/platform/system/keymint/+/refs/heads/main/README.md)，以 Cuttlefish 为示例），它不是 StrongBox。所以"有没有 KeyMint"和"KeyMint 跑在哪一档安全等级"是两件事。

### 3.2 三档安全等级与 CDD 依据

`SecurityLevel.aidl` 的枚举与定义：

- `SOFTWARE = 0` — "runs in an Android process"
- `TRUSTED_ENVIRONMENT = 1` — 满足 **CDD 9.11.1 [C-1-2]**
- `STRONGBOX = 2` — "runs in security hardware that satisfies … **CDD 9.11.2** … discrete, security-focus computing environments hardened against physical and side channel attack"
- `KEYSTORE = 100` — 内部值，永不用于 `getHardwareInfo`

CDD 9.11 侧的要求是 **[9.11/H-0-2]** "MUST back up the keystore implementation with an isolated execution environment"。

**读法。** `SOFTWARE` 档意味着密钥材料与 Android 进程同权限域——上层一旦有代码执行缺陷，密钥就不再受硬件保护。这正是"为什么框架级 RCE 足以造成密钥泄露"的结构性原因（与 [RCE 与提权手册](/android-rce-and-privilege-escalation/) §影响上下文那一节相接）。

### 3.3 StrongBox 的硬性要求

`IKeyMintDevice.aidl` 的措辞："Completely separate, purpose-built and certified secure CPUs, called 'StrongBox' devices. Examples … embedded Secure Elements (eSE) or on-SoC secure processing units (iSE)"。

[Android Keystore system](https://developer.android.com/privacy-and-security/keystore) 列出的 StrongBox 要求组件：自己的 CPU、安全存储、真随机数发生器、安全定时器、**重启通知引脚（a reboot notification pin）**。

那个"重启通知引脚"值得单独说：它不是装饰。密钥使用授权依赖时间戳型令牌（§4），而重启会让 per-boot HMAC 密钥变化；没有可信的重启信号，"跨重启不复活已失效授权"这件事就无法在硬件层强制。评估"是不是真 StrongBox"时，这一条是可见判据之一。

### 3.4 授权标签：真实名称与生命周期

密钥的强制策略由一组 **Tag** 表达（[Tag.aidl](https://android.googlesource.com/platform/hardware/interfaces/+/refs/heads/main/security/keymint/aidl/android/hardware/security/keymint/Tag.aidl)），与认证/锁屏/启动阶段相关的实名：

| Tag | 语义（官方） |
|---|---|
| `USER_SECURE_ID` | 绑定到用户安全标识；与 `NO_AUTH_REQUIRED` 互斥 |
| `AUTH_TIMEOUT` | "timeout-based key"，超时型授权 |
| `TRUSTED_CONFIRMATION_REQUIRED` | 需要可信确认 |
| `UNLOCKED_DEVICE_REQUIRED` | 需要设备处于解锁状态 |
| `EARLY_BOOT_ONLY` | "may only be used during early boot, until `IKeyMintDevice::earlyBootEnded()` is called" |
| `MAX_USES_PER_BOOT` | 每次启动限次使用 |
| `ALLOW_WHILE_ON_BODY` | 随身佩戴期可用 |
| `OS_VERSION` / `BOOT_PATCHLEVEL` | 绑定到系统版本 / 启动时补丁级别 |

两点提醒：
- `UNLOCKED_DEVICE_REQUIRED` 的硬件强制路径在官方注释里标注为已废弃/"never enabled by Keystore"，引用时不要当成现行强制保证。
- `BOOT_PATCHLEVEL` 把密钥与**安全补丁级别**绑死——补丁回滚或被伪造时，这类密钥会直接不可用，这既是保护也是可用性风险。

**不可导出性**的官方表述："Once keys are in the keystore, you can use them for cryptographic operations, with the key material remaining **non-exportable**"。注意这句话的隐含前提：不可导出说的是**密钥材料**，不代表"使用该密钥的授权"不会被绕过——§8 的案例里多数正是后者。

### 3.5 缺陷形态（KeyMint/Keymaster 域）

- **令牌解析缺陷**：授权令牌由上层传入硬件，解析/转换代码里的越界写能直接破坏"是否需要认证"的判断。实例见 §8 的 CVE-2022-20549。
- **IPC 边界上的共享秘密误用**：本该每启动变化的 HMAC 密钥被固定/可预测，导致授权可被伪造。实例见 CVE-2022-20377。
- **锁状态判定分支缺陷**：`is_device_locked` / `set_device_locked` 一类的状态写入有误，使绑定 keyguard 的密钥绕过锁屏要求。实例见 CVE-2021-0320。
- **资源耗尽与生命周期**：密钥导入路径的资源管理缺陷（DoS、UAF）。实例见 CVE-2019-1998、CVE-2019-9350。
- **厂商 trustlet/TA 侧输入校验**：QTEE keymaster app、KeyMaster Trusted Application 一类的输入校验与内存破坏。实例见 CVE-2018-5869、CVE-2019-10615、CVE-2023-24850。

---

## 4. 认证链：Gatekeeper 与 HardwareAuthToken

### 4.1 Gatekeeper 的职责边界

[官方页](https://source.android.com/docs/security/features/authentication/gatekeeper)的描述集中而明确：

- "performs device pattern/password authentication in a Trusted Execution Environment (TEE)"
- "Gatekeeper **enrolls and verifies passwords using a hardware-backed secret key**"
- 验证成功后 "emits an authentication token that is signed with a **per-boot HMAC key** … sent to the hardware-backed Keystore"
- 涉及 "64-bit user secure identifier (**SID**)"
- 实现须通过 "vendor test suite (VTS)" 测试

与生物识别的分工（[Authentication](https://source.android.com/docs/security/features/authentication)）："Gatekeeper for PIN/pattern/password … Fingerprint for fingerprint authentication. Devices that ship with Android 9 and higher can use **BiometricPrompt** as a single integration point"，类型上是 "for fingerprints or **faced** for face"。

**共享的 HMAC 密钥。** 关键的一句："The HMAC key is kept solely in KeyMint; Fingerprint and Gatekeeper request the key from KeyMint for each use"。也就是说 Gatekeeper 与 Fingerprint **共用**同一把每启动密钥向 KeyMint 换取签名授权。这把钥匙一旦出问题，两个认证通道同时失效——这正是 §8 里那个"迫使 gatekeeper、fingerprint、faceauth 使用已知 HMAC key"案例的破坏半径来源。

**术语纠正两处：**
- KeyMint 没有 `setAuthorization` 这种方法。`HardwareAuthToken` 是 "passed to **begin(), update(), and finish()**"。
- 测试实名是 `VtsHalGatekeeperV1_0TargetTest.cpp`，不是"VtsGateKeeper"。

### 4.2 HardwareAuthToken 的真实字段

[HardwareAuthToken.aidl](https://android.googlesource.com/platform/hardware/interfaces/+/refs/heads/main/security/keymint/aidl/android/hardware/security/keymint/HardwareAuthToken.aidl)：`challenge` / `userId` / `authenticatorId` / `authenticatorType` / `timestamp` / `mac`。

这六个字段构成"谁、以什么方式、在什么时候、通过了认证"的可验证声明，`mac` 由那把 per-boot HMAC 密钥生成。防御上值得关注的三个语义：

- `timestamp` 与 `AUTH_TIMEOUT` 配对 — 超时型密钥的安全上限就是时间戳的可信度
- `challenge` — 把令牌绑定到一次具体的密钥操作，防止拿旧令牌复用
- `userId` / `authenticatorId` — 多用户与"重新录入凭据后旧授权作废"的落点（应用侧对应 `setInvalidatedByBiometricEnrollment(boolean)`）

### 4.3 缺陷形态（认证域）

- **令牌铸造侧**：签名授权时的内存管理缺陷。CVE-2019-2115（`GateKeeper::MintAuthToken` double free）
- **令牌校验侧**：`mac` 或时间戳校验不严格 → 认证要求形同虚设。CVE-2021-0320、CVE-2022-20549
- **共享秘密侧**：per-boot HMAC 密钥可预测或被强制替换。CVE-2022-20377
- **指纹通道自身的判定缺陷**：账号维度处理不当。CVE-2016-3917（Android 6.0.1 指纹登录 "does not track the user account"）
- **限频缺失**：认证子系统未做速率限制，使猜测凭据成为可能。这一类的公开条目归属我未能逐条核实，见 §11

---

## 5. 密钥证明：让远端也能验证设备状态

### 5.1 Root of Trust 的实际结构

密钥证明在 X.509 扩展里携带一个 `RootOfTrust`，官方 ASN.1 定义就是这四个字段（[attestation](https://source.android.com/docs/security/features/keystore/attestation)）：

```
RootOfTrust ::= SEQUENCE {
    verifiedBootKey    OCTET_STRING,
    deviceLocked       BOOLEAN,
    verifiedBootState  VerifiedBootState,
    verifiedBootHash   OCTET_STRING }

VerifiedBootState ::= ENUMERATED { Verified(0), SelfSigned(1), Unverified(2), Failed(3) }
```

对应的 tag 名是 `Tag::ROOT_OF_TRUST`，且文档明确它 "never provided to or returned from the IKeyMintDevice"——它只出现在证明证书里，不是可设置的密钥参数。

**常见误传：** 网上流传的 `VERIFIED_BOOT_KEY` / `DEBUG_PERMITTED` / `DEVICE_LOCKED` 这类大写 ASN.1 **字段名**并不存在；大写形式是 `Tag::` 常量名，与 ASN.1 里的驼峰字段名是两套命名，混用会导致解析代码对不上。

**版本注意：** 早期 Keymaster 版本的 ASN.1 里没有 `verifiedBootHash`，该字段自 Keymaster 4 / KeyMint 起才存在。写验证逻辑时用 `attestationVersion`（§3.1）分派，不要假设字段恒在。

### 5.2 verifiedBootHash 与依赖方核对

官方定义："A digest of all data protected by Verified Boot. For devices that use the Android Verified Boot reference implementation, this field contains the **VBMeta digest**."

这句话是整个远程验证的支点，AVB README 把闭环写得很直白："if the VBMeta digest is included in hardware-backed attestation data a relying party can extract the digest and compare it with list of digests for known good operating systems"。

**读法。** 四色状态只告诉依赖方"链是否完整验证过"，`verifiedBootHash` 才告诉依赖方"具体是哪一个软件版本"。因此一个只检查 `deviceLocked==true` 且 `state==Verified` 的服务端策略是**不够**的：它无法区分"已解锁后重新锁定并运行未授权镜像"之外的场景，也无法把设备精确对齐到某个已知安全版本。把 `verifiedBootHash` 与自家维护的已知良好摘要清单比对，才是版本级的判定。

Google 侧的记录/编解码实现可作参考：[security/keymint/support](https://android.googlesource.com/platform/hardware/interfaces/+/refs/heads/main/security/keymint/support/)（`attestation_record.cpp`）。

### 5.3 ID attestation 与能力探测

设备标识证明字段实名：`attestationIdBrand[710]`、`attestationIdDevice[711]`、`attestationIdProduct[712]`、`attestationIdSerial[713]`、`attestationIdImei[714]`、`attestationIdMeid[715]`、`attestationIdManufacturer[716]`、`attestationIdModel[717]`；另有 `attestationApplicationId[709]`。

文档明确 ID attestation 自 Keymaster 3 起是**可选**特性，探测方式是 `PackageManager.FEATURE_DEVICE_ID_ATTESTATION`（`"android.software.device_id_attestation"`）。

**工程后果。** 依赖"证明里一定带序列号"的设计在大量机型上不成立。先探测再启用，且不要把它当作必要失败条件。

### 5.4 应用侧 API 实名

[KeyGenParameterSpec.Builder](https://developer.android.com/reference/android/security/keystore/KeyGenParameterSpec.Builder)：

- `setIsStrongBoxBacked(boolean)` — "Sets whether this key should be protected by a StrongBox security chip"
- `setAttestationChallenge(byte[])` — 防重放的挑战值，属 `KeyDescription`
- `setUserAuthenticationRequired(boolean)`、`setUserAuthenticationParameters(int timeout, int type)`
- `setUnlockedDeviceRequired(boolean)`
- `setInvalidatedByBiometricEnrollment(boolean)`

**两个高频错误：**
1. `setStrongBoxRequired` **不存在**，实名是 `setIsStrongBoxBacked`。
2. `setUserAuthenticationValidityDurationSeconds(int)` 已在 API level 30 废弃，改用 `setUserAuthenticationParameters(int, int)`。
3. 另有一处检索陷阱：`AndroidKeyStoreProvider` 没有公开的 developer.android.com 参考页，只有 AOSP 内部源文件，不应作为应用开发接口引用。Provider 名 `AndroidKeyStore` 自 Android 4.3（API 18）引入；StrongBox KeyMint 是 Android 9（API 28）起的可选组件。

---

## 6. 存储加密：FBE、metadata encryption 与被混淆的解封路径

### 6.1 版本要求（注意年份）

**纠正一处常被混用的时间线：** FBE 的强制要求是 **Android 10**，不是 11——"All devices launching with Android 10 and higher are required to use file-based encryption"（FBE 本身引入于 Android 7.0）。Android 11 起强制的是元数据加密："Devices launched with Android 11 or higher must have metadata encryption on internal storage enabled."

只读分区（system、vendor 等）由 Verified Boot / dm-verity **保护而非加密**——这是常见误解：它们的威胁模型是篡改，不是泄露。

### 6.2 DE 与 CE 的准确名称

**DE = Device-Encrypted**（不是 "Default-Encrypted"），**CE = Credential-Encrypted**。官方以 "Per-user credential-encrypted data on internal storage" / "Per-user device-encrypted data…" 表述，并列出 `/data/system_ce`、`/data/user_de` 等目录；System DE key 位于 `/data/unencrypted`。

**Direct Boot** 的官方定义："Direct Boot allows encrypted devices to boot straight to the lock screen…before users have provided their credentials"。能在这个阶段访问的只有 DE 数据。

KeyMint 侧的对应 tag 是 `EARLY_BOOT_ONLY`——它把"早启动可用"的窗口显式收束到 `earlyBootEnded()` 被调用之前，与 Direct Boot 的边界互为印证。

### 6.3 CE 密钥到底怎么解封（易混淆点）

一个常见的简化说法是"CE 密钥由 Gatekeeper 认证令牌解封"。**AOSP 的实际机制不是这样**：

- FBE 页原文："vold encrypts each CE key …using an AES-256-GCM key derived from the user's **synthetic password**"（用户设置的锁屏凭据 LSKF 经 scrypt 拉伸参与派生）
- Gatekeeper 令牌的作用是让 Keystore 知道 "authentication-bound keys … can be used"——它**授权密钥使用**，不是解开磁盘密钥的钥匙
- metadata key 才走硬件那条路："protected by KeyMint…which in turn is protected by Verified Boot"

**为什么这个区分重要。** 它决定了威胁面在哪：CE 磁盘密钥的强度取决于 synthetic password 的派生链与 LSKF 本身；而"某个应用密钥能不能用"取决于认证令牌与 Tag 策略。把两者混为一谈，会让人误判"攻破 Gatekeeper 就等于解密磁盘"——实际影响半径完全不同（这一点与 [RCE 与提权手册](/android-rce-and-privilege-escalation/) 的"影响半径"框架一致）。

### 6.4 metadata encryption 的实现现状

Android 11+ 使用 `dm-default-key`，由 Android common kernel 4.14+ 支持，走 `blk-crypto` 内联加密框架（`CONFIG_DM_DEFAULT_KEY=y`、`CONFIG_FS_ENCRYPTION_INLINE_CRYPT=y`，即 **inlinecrypt**）。Android 10 及以下的 common kernel 不支持 `dm-default-key`，旧式实现可用 dm-crypt。两者都由 `vold` 管理。

---

## 7. 本域缺陷形态汇总

| 缺陷形态 | 出现的接缝 | 已核实实例 |
|---|---|---|
| 授权令牌解析越界 | 上层 → KeyMint 的令牌转换 | CVE-2022-20549 |
| 共享 HMAC 秘密可预测/被替换 | KeyMint ↔ Gatekeeper ↔ 生物识别 | CVE-2022-20377 |
| 锁状态判定分支错误 | AVB 状态 → 密钥强制 | CVE-2021-0320 |
| 令牌铸造的内存破坏 | Gatekeeper 内部 | CVE-2019-2115 |
| 信任根/元数据重签 | `vbmeta`、测试密钥残留 | CVE-2025-48613 |
| 下载/刷写路径输入校验 | fastboot、闪存刷写 | CVE-2017-8273、CVE-2021-1928 |
| BootROM / 安全启动组件 | 最早期执行 | CVE-2018-6240、CVE-2025-27374 |
| 内存保护器绕过 | 协处理器启动驱动 | CVE-2021-25481 |
| TA/trustlet 输入校验与整数溢出 | 厂商 TEE 内 app | CVE-2018-5869、CVE-2019-10615、CVE-2023-24850 |
| 密钥生命周期资源管理 | keymaster app | CVE-2019-9350、CVE-2019-1998 |
| 磁盘加密信息泄露 | FBE 早期实现 | CVE-2017-0493 |
| 认证判定未覆盖账号维度 | 指纹登录 | CVE-2016-3917 |

**共同特征。** 这一域的缺陷绝大多数不是"密码学算法错了"，而是**状态传递与边界校验错了**：锁状态、认证令牌、启动摘要这些事实穿越了特权边界或子系统边界，在转换环节被处理错。防御投入应优先放在转换代码的输入校验与"默认拒绝"上，而不是算法替换。

---

## 8. 公开案例复盘（逐条已核实归属）

所有条目均经 NVD 记录与对应官方公告交叉核对。"组件"列使用公告原文措辞。

| CVE | 组件/位置（公告原述） | 影响 | 版本/Android ID | 一手来源 |
|---|---|---|---|---|
| CVE-2019-9350 | Keymaster，use-after-free | EoP | Android 10 / A-129562815 | [NVD](https://nvd.nist.gov/vuln/detail/CVE-2019-9350) |
| CVE-2019-1998 | `keymaster_app.c` 资源耗尽 | DoS | Android 9 | [NVD](https://nvd.nist.gov/vuln/detail/CVE-2019-1998) |
| CVE-2018-5869 | Qualcomm **QTEE keymaster app** 输入校验不当 | EoP | 高通平台 | [NVD](https://nvd.nist.gov/vuln/detail/CVE-2018-5869) / [Qualcomm 公告](https://www.qualcomm.com/company/product-security/bulletins) |
| CVE-2019-10615 | **keymaster 4** 整数溢出 | 见公告 | 高通平台 | [NVD](https://nvd.nist.gov/vuln/detail/CVE-2019-10615) |
| CVE-2023-24850 | 向 **KeyMaster Trusted Application** 导入密钥时 HLOS 内存破坏 | 见公告 | 高通平台 | [NVD](https://nvd.nist.gov/vuln/detail/CVE-2023-24850) |
| CVE-2021-39741 | Keymaster 越界写 | EoP | Android 12L / A-173567719 | [Android 12L 公告](https://source.android.com/security/bulletin/android-12l) |
| CVE-2022-20377 | `keymaster_ipc.cpp` — 可迫使 **gatekeeper、fingerprint、faceauth 使用已知 HMAC key** | 认证授权绕过 | A-222339795 | [Pixel 2022-08-01](https://source.android.com/security/bulletin/pixel/2022-08-01) |
| CVE-2022-20549 | `KeyMintUtils.cpp` 的 `authToken2AidlVec` 越界写 | 见公告 | Pixel | [Pixel 2022-12-01](https://source.android.com/docs/security/bulletin/pixel/2022-12-01) |
| CVE-2021-0320 | `keystore_keymaster_enforcement.h` 的 `is_device_locked`/`set_device_locked` — **绕过 keyguard 绑定密钥的锁屏要求** | 策略绕过 | Android 10/11 / A-169933423 | [NVD](https://nvd.nist.gov/vuln/detail/CVE-2021-0320) |
| CVE-2019-2115 | `GateKeeper::MintAuthToken` double free → 内存破坏 | 见公告 | 7.1.1/7.1.2/8.0/8.1/9 | [2019-09-01 公告](https://source.android.com/security/bulletin/2019-09-01) |
| CVE-2016-3917 | Android 6.0.1 指纹登录 **未跟踪用户账号** | 认证判定缺陷 | internal bug 30744668 | [2016-10-01 公告](http://source.android.com/security/bulletin/2016-10-01.html) |
| CVE-2025-48613 | "In VBMeta, there is a possible way to modify and resign VBMeta using a test key" | High（信任根重签） | A-416491056 | [2026-03-01 公告](https://source.android.com/docs/security/bulletin/2026/2026-03-01) |
| CVE-2017-8273 | Qualcomm CAF：处理 fastboot boot 命令时，**验证启动被禁用**情形下的缓冲区溢出 | 见公告 | 高通平台 | [NVD](https://nvd.nist.gov/vuln/detail/CVE-2017-8273) / [2017-07-01 公告](https://source.android.com/security/bulletin/2017-07-01) |
| CVE-2021-1928 | Qualcomm：**刷写 eMMC 设备时**缓冲区越读 | 见公告 | 2021-08 公告 | [Qualcomm Aug-2021](https://www.qualcomm.com/company/product-security/bulletins/august-2021-bulletin) |
| CVE-2018-6240 | NVIDIA **BootROM** EoP | 最早期执行 | 见公告（Android 公告含 NVIDIA 组件） | [2019-09-01 公告](https://source.android.com/security/bulletin/2019-09-01) |
| CVE-2025-27374 | **三星移动处理器（Exynos）Secure Boot 组件** | 见公告 | Exynos | [Samsung](https://semiconductor.samsung.com/support/quality-support/product-security-updates/cve-2025-27374/) |
| CVE-2021-25481 | Exynos **CP booting driver** 绕过 Secure Memory Protector | 内存保护绕过 | 2021-10 | [Samsung SMSB](https://security.samsungmobile.com/securityUpdate.smsb?year=2021&month=10) |
| CVE-2017-0493 | **File-Based Encryption** 信息泄露，用于绕过锁屏 | 磁盘加密侧 | 7.0/7.1.1 / A-32793550 | [2017-05-01 公告](https://source.android.com/security/bulletin/2017-05-01) |
| CVE-2026-0145 | "In keymint … Permission Bypass … local information disclosure" | 权限绕过 | Pixel 6 月公告 | [Pixel 2026-06-01](https://source.android.com/docs/security/bulletin/pixel/2026/2026-06-01) |

### 8.1 KEV 核对（是否被积极利用）

逐条对照 CISA [KEV 全量 JSON](https://www.cisa.gov/sites/default/files/feeds/known_exploited_vulnerabilities.json)（该目录只有 `field_cve` 精确检索可靠，全文搜索参数不过滤）：

- **在 KEV**：CVE-2024-29745（Pixel **fastboot 固件**信息泄露，2024-04-04 加入）、CVE-2024-29748、CVE-2024-32896
- **不在 KEV**：CVE-2025-48613、CVE-2017-0493、CVE-2022-20377、CVE-2022-20549、CVE-2017-8273、CVE-2020-0096、CVE-2019-2122

**读法。** 这一域的 KEV 条目集中在**刷写/固件路径**而非 TEE 内部。对防守方的含义：外部可交互的刷写与启动接口比 TEE 内部更早进入实战视野；而 TEE 侧缺陷即便未被列名"积极利用"，一旦上层同时存在代码执行缺陷，其修复优先级仍应上调（参考 [评估与定级手册](/android-vulnerability-severity-assessment/) 里"上下文阶梯"的用法）。

### 8.2 三条读法

1. **接缝比模块更危险。** 表中破坏半径最大的 CVE-2022-20377 位于 `keymaster_ipc.cpp`——一次跨进程调用，同时命中三个认证子系统。审计精力放在边界，而不是逐个信任 TEE 声明。
2. **"禁用验证启动"本身构成缺陷面。** CVE-2017-8273 的前提是 "when verified boot feature is disabled"。安全特性被关闭的分支同样要有输入校验。
3. **测试密钥是真实的生产风险。** CVE-2025-48613 是"用测试密钥重签 VBMeta"。量产件是否残留测试信任根，属于可核查项。

---

## 9. 常见误传纠正

以下几条在二手资料里流传很广，但**一手核对不成立**。写在这里是为了避免继续扩散。

| 流传的说法 | 实际核实结果 |
|---|---|
| CVE-2020-0041 是 TrustZone/QSEE 漏洞 | **不是**。是 `binder.c` 中 `binder_transaction` 的越界写、Android 内核 EoP，2020-03-01 公告，A-145988638（[NVD](https://nvd.nist.gov/vuln/detail/CVE-2020-0041)） |
| CVE-2021-0703 / CVE-2021-0704 是高通 QSEE 漏洞 | **都不是**。前者是 `init.cpp` 的 `SecondStageMain` use-after-free（A-184569329），后者是 `AccountManagerService.java` 的 `createNoCredentialsPermissionNotification`（A-179338675） |
| CVE-2019-2188 是 Fingerprint 组件漏洞 | **不是**。是 **Easel driver** 竞态导致的内存破坏、Android 内核，Pixel 专属公告 2019-09-01，A-112309571 |
| CVE-2020-0096 出在 fastboot | **不是**。是 `ActivityStartController.java` 的 confused deputy EoP，Android 8.0–9，A-145669109 |
| CVE-2020-0034 是 vold/加密漏洞 | **不是**。是 `libvpx` 的 `vp8_decode_frame` 越界读，Media framework，A-62458770 |
| CVE-2019-2122 是加密实现漏洞 | **不是**。是 `LockTaskController.java` 的 parcel 差异 EoP，A-127605586 |
| CVE-2020-11174 可作 TEE/安全启动案例 | **不可**。归属是 **adsp 驱动** |
| CVE-2025-21479 是 QSEE/TEE 漏洞 | **不是**。KEV 记录为 GPU micronode 未授权命令执行导致的内存破坏（2025-06-03 加入） |
| 存在 `setStrongBoxRequired()` API | **不存在**。实名 `setIsStrongBoxBacked(boolean)` |
| DE 表示 "Default-Encrypted" | **不是**。DE = **Device-Encrypted** |
| Android 11 起强制 FBE | **不准确**。FBE 自 **Android 10** 起强制；Android 11 起强制的是 metadata encryption |
| 官方术语叫 "anti-rollback index" | 官方名是 **Rollback Protection**，字段 `rollback_index[n]` / `stored_rollback_index[n]` |
| ASN.1 里有 `VERIFIED_BOOT_KEY` / `DEVICE_LOCKED` 字段 | 实际字段是 `verifiedBootKey` / `deviceLocked`；大写形式是 `Tag::` 常量 |
| KeyMint 用 `setAuthorization` 绑定授权 | **没有这方法**。`HardwareAuthToken` 传给 `begin()`/`update()`/`finish()` |
| DevicePolicyManager 有 `VERITY_MODE_*` / `BOOTLOADER_*` 常量 | **不存在**（AOSP 与 developer.android.com 均无）。该页真实的只有 `ENCRYPTION_STATUS_*` 系列 |
| `VtsGateKeeper` 是测试名 | 实际文件是 `VtsHalGatekeeperV1_0TargetTest.cpp` |
| 存在名为 `development/samples/KeyAttestation`、`external/keystore-tools` 的官方仓库 | 均 **404**，不可引用 |

**误传的共性。** 上表右列几乎每一条都是"组件名相似但归属完全不同"。跨平台引用 CVE 时，唯一可靠的做法是回到 NVD 记录（其 description 里的组件与 Android ID 与官方公告逐字一致）再对照公告页——见 [编号与公告手册](/android-cve-and-security-bulletins/) 的核对流程。

---

## 10. 自查与观测点

### 10.1 本机可读取的状态（真实存在的属性名）

| 观测点 | 依据 |
|---|---|
| `ro.boot.verifiedbootstate` | 官方文档直接使用，AVB 状态的可读投影 |
| `ro.boot.flash.locked` | [locking_unlocking](https://source.android.com/docs/core/architecture/bootloader/locking_unlocking)："the bootloader should indicate the lock status by setting…`androidboot.flash.locked`…use `ro.boot.verifiedbootstate` to set the value of `ro.boot.flash.locked` to 0; this unlocks the bootloader if the verified boot state is orange" |
| `ro.boot.vbmeta.digest` | 见 cs.android.com 检索（frameworks 侧与 fastboot 工具侧引用） |
| `ro.boot.vbmeta.device_state` | `external/avb/tools/avbctl/avbctl.cc` |
| `ro.boot.vbmeta.avb_version` | `system/libvintf/RuntimeInfo-target.cpp` |

AOSP 自己的 VTS 就是用这个属性做判定的——`BootloaderStateTest.cpp` 里断言 `GetProperty("ro.boot.verifiedbootstate") == "orange"`（[链接](https://android.googlesource.com/platform/hardware/interfaces/+/refs/heads/main/security/keymint/aidl/vts/functional/BootloaderStateTest.cpp)）。

**判读。** 一台"应当锁定"的设备如果 `ro.boot.flash.locked` 为 0、或 `ro.boot.verifiedbootstate` 不是 `green`，就应视为信任根已被动过；这两项与 §8 的密钥清除要求（CDD [C-1-12]）互相印证：解锁过，密钥就应当已经不可用。

### 10.2 实现强度的核查项

- **KeyMint 的档位**：`getHardwareInfo` 返回的 security level 是 `TRUSTED_ENVIRONMENT` 还是 `SOFTWARE`。落在 SOFTWARE 时，密钥保护与 Android 进程同域。
- **证明版本**：`attestationVersion` 落在哪一档（100/200/300…），据此决定字段解析路径；`verifiedBootHash` 只在 Keymaster 4+ 存在。
- **ID attestation 能力**：用 `PackageManager.FEATURE_DEVICE_ID_ATTESTATION` 探测，不要假设一定携带设备标识。
- **StrongBox 是否真的存在**：`setIsStrongBoxBacked(true)` 是否成功、以及是否有对应硬件（多数设备并无 eSE/iSE）。

### 10.3 服务侧验证要点

1. 只验证书链合法性**不够**——把 `verifiedBootHash` 与已知良好镜像摘要清单比对，才能到版本级。
2. 不要写死字段存在性：按 `attestationVersion` 分派解析。
3. 拒绝策略默认走"未知即拒绝"：`SelfSigned(1)` 与 `Unverified(2)` 对高价值凭据都应视为不满足条件。
4. 需要"bootloader 锁定"的硬件级证据时，Android 13+ 的 Play Integrity 提供该保证（`MEETS_DEVICE_INTEGRITY`），但这是**服务侧判定**，与密钥证明是两套机制，不要混用字段名。

### 10.4 处置优先级（本域内）

| 优先级 | 条件 |
|---|---|
| 最高 | 影响授权令牌校验或共享 HMAC 秘密（同时波及多认证通道），且机型上仍可被上层代码执行缺陷串联 |
| 高 | 影响 `vbmeta`/信任根重签、BootROM、防回滚存储 |
| 中 | 单认证通道判定缺陷、TA 侧 DoS |
| 视场景 | `UNLOCKED_DEVICE_REQUIRED` 一类官方已标注废弃的强制路径，不要作为合规依据引用 |

---

## 11. 未核实与失效来源（不作断言）

以下条目在本次核验中**没有**取得可引用的一手来源，因此本手册不对其作任何结论：

1. **"OS target vs VM target"** — 在 implementer-ref、keystore overview、`SecurityLevel.aidl`、`IKeyMintDevice.aidl`、`system/keymint` README 均未出现该术语。
2. **Keymaster 3.0/3.1/3.2 与 HIDL 的细粒度对应** — Google 文档只写 "Keymaster 3"。
3. **证明证书链（Root/中间/Leaf）层级的逐字一手描述** — 未取到（[developer.android.com/training/articles/security-key-attestation](https://developer.android.com/training/articles/security-key-attestation) 页面为 JS 渲染，抓取仅得空正文，URL 本身有效且被 AVB README 引用）。
4. **`fastboot getvar vbstate` / `getvar unlocking` 的具体取值** — AOSP 文档与代码中未找到。RKP device-info 里的字段是 `vb_state` / `bootloader_state`，与 fastboot 变量不是同一套，不要混用。
5. **MediaTek 安全启动/TEE 专属 CVE** — NVD 关键词组合（`mediatek secure boot` / `mediatek TEE` / `mediatek bootloader`）无可直引条目；厂商公告页 `corp.mediatek.com/product-security-bulletin/...` 现 302 跳转至主站。
6. **Qualcomm TEE（QSEE/TrustZone）严格归属条目** — 除 §8 中 keymaster/TA 相关条目外，未取得可直接引用的"TEE 内部"条目（`CVE-2020-11174` 实测为 adsp 驱动）。
7. **dm-verity 专属 CVE** — 未找到。
8. **Project Zero《Trust Issues》** 文中各厂商逐条 CVE 对应 — 页面可访问（[链接](https://projectzero.google/2017/07/trust-issues-exploiting-trustzone-tees.html)），但本次未能列出完整对应，仅见 Qualcomm Widevine CVE-2015-6639，故不引用其他编号。
9. **`CVE-2020-10849`（"The Gatekeeper trustlet allows a brute-force attack on the screen lock password"）厂商归属** — NVD 关键词 `gatekeeper` 的结果里混有 Apple/macOS "Gatekeeper" 条目，厂商归属未核实，本手册不引用。
10. **失效 URL（勿引用）**：`source.android.com/security/keystore/keymint`、`source.android.com/docs/security/features/keystore/keymint`、`docs/security/features/authentication/biometrics`（404）；`platform/hardware/interfaces/+/main/security/keymint/README.md`、`platform/system/security/keymint/...`（正确项目是 `platform/system/keymint`）；`platform/external/keystore-tools`、`development/samples/KeyAttestation`（404）。
11. **CISA KEV 检索方式**：`search_api_fulltext=TrustZone` 不过滤（返回全量），只有 `field_cve` 精确检索有效；判断"是否在 KEV"必须逐条精确查。另注意 `CVE-2026-47222`/`CVE-2026-47223` 虽命中 "AVB vbmeta" 关键词，实为 **NanaZip（Windows）** 解析器缺陷，与 Android 无关。
12. **Play Integrity** verdict 中的 `bootloaderLocked` 字段名 — 未在该页取到，页面可见的是 `deviceRecognitionVerdict`；不要把传闻字段名当官方字段引用。

---

## 12. 与文档库其他部分的分工

| 需要 | 去看 |
|---|---|
| 整体攻击面与启动链鸟瞰 | [总览报告](/android-system-vulnerability-report/) |
| 底层架构与漏洞成因的技术叙述 | [底层漏洞研究文章](/android-low-level-vulnerability-research/) |
| 按层分类的通用缺陷清单 | [通用漏洞类型手册](/android-common-system-vulnerabilities/) |
| 组件 × 缺陷类别的坐标定位（内核/驱动/Binder/HAL/协议） | [底层漏洞类型图谱](/android-low-level-vulnerability-types/) |
| CVE / bug ID / SPL / 公告怎么读、怎么核对归属 | [编号与公告手册](/android-cve-and-security-bulletins/) |
| 执行上下文与提权三段式 | [RCE 与提权手册](/android-rce-and-privilege-escalation/) |
| 给具体缺陷定级、比较修复优先级 | [评估与定级手册](/android-vulnerability-severity-assessment/) |
| 本文 | TEE / 密钥 / 认证 / 启动完整性 / 存储加密这一条链的接口实名、接缝缺陷与已核实案例 |

---

## 参考

**一手来源（组件与接口）**
- [Hardware-backed Keystore](https://source.android.com/docs/security/features/keystore)
- [Key and ID attestation](https://source.android.com/docs/security/features/keystore/attestation)
- [KeyMint functions（implementer 参考）](https://source.android.com/docs/security/features/keystore/implementer-ref)
- [SecurityLevel.aidl](https://android.googlesource.com/platform/hardware/interfaces/+/refs/heads/main/security/keymint/aidl/android/hardware/security/keymint/SecurityLevel.aidl) · [Tag.aidl](https://android.googlesource.com/platform/hardware/interfaces/+/refs/heads/main/security/keymint/aidl/android/hardware/security/keymint/Tag.aidl) · [HardwareAuthToken.aidl](https://android.googlesource.com/platform/hardware/interfaces/+/refs/heads/main/security/keymint/aidl/android/hardware/security/keymint/HardwareAuthToken.aidl) · [IKeyMintDevice.aidl](https://android.googlesource.com/platform/hardware/interfaces/+/refs/heads/main/security/keymint/aidl/android/hardware/security/keymint/IKeyMintDevice.aidl)
- [keymaster 4.1 HIDL types.hal](https://android.googlesource.com/platform/hardware/interfaces/+/refs/heads/main/keymaster/4.1/types.hal) · [KeyMint Rust Reference Implementation](https://android.googlesource.com/platform/system/keymint/+/refs/heads/main/README.md) · [attestation_record.cpp](https://android.googlesource.com/platform/hardware/interfaces/+/refs/heads/main/security/keymint/support/)
- [Verified Boot](https://source.android.com/docs/security/features/verifiedboot) · [Boot flow](https://source.android.com/docs/security/features/verifiedboot/boot-flow) · [external/avb README](https://android.googlesource.com/platform/external/avb/+/refs/heads/main/README.md)
- [Bootloader locking & unlocking](https://source.android.com/docs/core/architecture/bootloader/locking_unlocking)
- [Gatekeeper](https://source.android.com/docs/security/features/authentication/gatekeeper) · [Authentication](https://source.android.com/docs/security/features/authentication)
- [File-Based Encryption](https://source.android.com/docs/security/features/encryption/file-based) · [Metadata Encryption](https://source.android.com/docs/security/features/encryption/metadata)
- [Android 16 CDD（9.10 Device Integrity / 9.11 Keys and Credentials）](https://source.android.com/docs/compatibility/16/android-16-cdd)
- [Android Keystore system](https://developer.android.com/privacy-and-security/keystore) · [KeyGenParameterSpec.Builder](https://developer.android.com/reference/android/security/keystore/KeyGenParameterSpec.Builder) · [API diff 30](https://developer.android.com/sdk/api_diff/30/changes/android.security.keystore.KeyGenParameterSpec.Builder) · [Play Integrity verdicts](https://developer.android.com/google/play/integrity/verdicts)

**案例与公告**
- Android 逐月与机型专属公告：[2016-10-01](http://source.android.com/security/bulletin/2016-10-01.html) · [2017-05-01](https://source.android.com/security/bulletin/2017-05-01) · [2017-07-01](https://source.android.com/security/bulletin/2017-07-01) · [2019-09-01](https://source.android.com/security/bulletin/2019-09-01) · [2020-03-01](https://source.android.com/security/bulletin/2020-03-01) · [2020-05-01](https://source.android.com/security/bulletin/2020-05-01) · [Pixel 2022-08-01](https://source.android.com/security/bulletin/pixel/2022-08-01) · [Pixel 2022-12-01](https://source.android.com/docs/security/bulletin/pixel/2022-12-01) · [Android 12L](https://source.android.com/security/bulletin/android-12l) · [2026-03-01](https://source.android.com/docs/security/bulletin/2026/2026-03-01) · [Pixel 2026-06-01](https://source.android.com/docs/security/bulletin/pixel/2026/2026-06-01)
- [Qualcomm Product Security bulletins](https://www.qualcomm.com/company/product-security/bulletins)（含 [2021-08](https://www.qualcomm.com/company/product-security/bulletins/august-2021-bulletin)） · [Samsung 半导体安全更新 CVE-2025-27374](https://semiconductor.samsung.com/support/quality-support/product-security-updates/cve-2025-27374/) · [Samsung Mobile SMSB 2021-10](https://security.samsungmobile.com/securityUpdate.smsb?year=2021&month=10)
- [NVD REST 检索入口](https://services.nvd.nist.gov/rest/json/cves/2.0?keywordSearch=keymaster) · [CISA KEV JSON](https://www.cisa.gov/sites/default/files/feeds/known_exploited_vulnerabilities.json)
- [Project Zero: Trust Issues — Exploiting TrustZone TEEs](https://projectzero.google/2017/07/trust-issues-exploiting-trustzone-tees.html)

---

## 联系方式

咨询 Android 系统详情请联系 https://t.me/one00190

---

*本手册仅供安全学习与防御研究使用；所有组件事实、接口名称与案例归属均取自公开一手来源并附链接，未能核实的表述已单列于 §11 而非写入正文。*
