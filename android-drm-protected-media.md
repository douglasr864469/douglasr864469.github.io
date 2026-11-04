---
layout: page
title: "Android DRM 与受保护媒体通路攻击面手册"
description: "把「受保护播放」拆成选方案／供应／取密钥／装密钥／建会话／配置解码／解密／显示八段：应用只见不透明请求与会话 ID、现行安全级别是 0–5 而非 L1/L2/L3、`MediaFormat` 无 `KEY_CRYPTO`（用 `FEATURE_SecurePlayback` + `MediaCrypto..."
date: 2026-11-04
permalink: /android-drm-protected-media/
---

> 流媒体能播、录屏却黑屏，靠的不是一个开关，而是一条从应用、`MediaDrm`、vendor DRM HAL 一直延伸到 TEE 与安全解码器的信任链。应用只拿到不透明的密钥请求与会话 ID，真正的密钥与解密运算留在 HAL 或可信执行环境里；解码器通过 `MediaCrypto` 与安全缓冲区把明文限制在受保护通路内；显示层再用 `SurfaceView.setSecure()` 阻止截屏。这条链上任何一环把「不透明」当成「可信」，都会变成密钥泄露、内容解密或权限提升。这份手册沿「取密钥 → 建会话 → 解密 → 解码 → 显示」拆解机制、公开缺陷与防御。只讨论公开且已修复的问题，不含可利用代码。

## 0. 使用方式与边界

这份手册面向三类读者：

- **应用开发者**：需要理解 `MediaDrm` 能做什么、不能做什么，为什么 `MediaFormat` 里没有 `KEY_CRYPTO`，以及为什么「设置 secure 标志」不等于「内容绝不会被录」。
- **系统与平台工程师**：需要区分 `MediaDrm`（应用 API）、`libmediadrm`（JNI/HAL 客户端）、vendor DRM HAL（`IDrmPlugin`）与安全解码器四层。
- **安全评估者**：需要在看到「DRM」公告时，判断问题落在 DRM 插件、密钥库、证书校验、解码器还是显示通路，而不是把所有带 `drm` 字样的 CVE 混为一类。

本文的三条自我约束：

1. 只引用公开且已修复的问题；CVE 编号、落点与描述以 NVD 为准，公告分类与级别以 Android Security Bulletin 为准。
2. 不写利用步骤、不写 PoC、不写可复现的密钥提取或绕过参数。
3. 查不到一手来源的说法一律进「未核实与失效来源」。

与已发布手册的分工：

- 《Android 媒体解码与容器解析攻击面手册》讲解码器与容器解析；本文只讲受保护通路里的密钥、会话与安全缓冲区。
- 《Android 图形与显示栈攻击面手册》讲合成与 Surface；本文只讲安全图层对截屏的语义。
- 《Android 账户、凭据与无密码认证手册》讲用户凭据；DRM 密钥属于设备内容保护，不与用户账户混为一谈。

## 1. 一次受保护播放的完整旅程

把「点击播放一部加密影片」拆开，至少经过八个阶段：

| 阶段 | 主要位置 | 做的事 | 安全判断点 |
|------|----------|--------|------------|
| 1. 选方案 | 应用进程 | 用 UUID 选择 DRM 方案，创建 `MediaDrm` | UUID 与插件是否匹配 |
| 2. 供应 | `MediaDrm` → HAL | `provisionRequest` 获取设备证书 | 证书链、设备唯一 ID |
| 3. 取密钥 | 应用 → 许可证服务器 | `getKeyRequest()` 生成不透明请求 | 请求绑定会话与内容 |
| 4. 装密钥 | 应用 → `MediaDrm` | `provideKeyResponse()` 提交许可证响应 | 响应完整性、会话归属 |
| 5. 建会话 | `MediaDrm` → HAL | `openSession()` 得到会话 ID | 会话数量、安全级别 |
| 6. 配置解码 | 应用 → `MediaCodec` | `MediaCrypto(UUID, sessionId)` 绑定 | 是否使用安全解码器 |
| 7. 解密解码 | HAL / TEE / 安全解码器 | 按 `CryptoInfo` 解密并解码到安全缓冲区 | 密钥是否离开 TEE |
| 8. 显示 | SurfaceFlinger / HWC | 安全图层合成，禁止截屏/非安全显示 | secure 标志是否生效 |

关键点是：**应用从头到尾看不到密钥**。`getKeyRequest()` 返回的是加密后的请求，`provideKeyResponse()` 接收的是许可证响应；密钥由 DRM 插件在 HAL 或 TEE 内解封并保存。应用能做的只是「转交不透明数据」和「引用会话 ID」。任何声称「从 `MediaDrm` 里读出密钥」的方案，要么依赖插件缺陷，要么是在描述别的平台。

第二个关键点是：**DRM 的安全强度由「安全级别」描述，而不是由「是否用了 DRM」描述**。Android 定义了 0–5 的 `SECURITY_LEVEL_*` 常量；软件级实现（SW_*）把密钥放在普通内存里，硬件级实现（HW_*）把密钥与运算放进 TEE。把「用了 Widevine」当成「内容一定安全」，忽略了级别差异。

## 2. 对象模型与进程边界

### 2.1 关键对象

| 对象 | 所在层 | 作用 |
|------|--------|------|
| `MediaDrm` | 公开 SDK（API 18） | 方案选择、会话、密钥请求/响应、属性查询 |
| `MediaDrm.ProvisionRequest` | 公开 SDK | 设备证书请求（opaque 数据 + 默认 URL） |
| `MediaCrypto` | 公开 SDK | 把 DRM 会话绑定到解码器 |
| `MediaCodec.CryptoInfo` | 公开 SDK | 每帧的加密信息（IV、子样本、模式） |
| `MediaFormat` | 公开 SDK | 解码器配置；安全播放通过 `setFeatureEnabled` 开启 |
| `SurfaceView.setSecure()` | 公开 SDK | 把图层标记为安全，阻止截屏 |
| `DrmUtils` / `DrmHal*` | AOSP native | JNI 侧选择并调用 HAL |
| `IDrmFactory` / `IDrmPlugin` | vendor HAL | 厂商实现的 DRM 插件 |
| `MediaCodecInfo.CodecCapabilities` | 公开 SDK | 查询 `FEATURE_SecurePlayback` 等能力 |

### 2.2 安全级别

Android 公开的级别常量是 0–5，而不是行业里常说的 L1/L2/L3：

| 常量 | 值 | 含义 |
|------|----|------|
| `SECURITY_LEVEL_UNKNOWN` | 0 | 未知 |
| `SECURITY_LEVEL_SW_SECURE_CRYPTO` | 1 | 密钥与加解密在软件白盒中 |
| `SECURITY_LEVEL_SW_SECURE_DECODE` | 2 | 软件解密 + 软件解码 |
| `SECURITY_LEVEL_HW_SECURE_CRYPTO` | 3 | 密钥与运算在 TEE |
| `SECURITY_LEVEL_HW_SECURE_DECODE` | 4 | TEE 密钥 + 硬件解码 |
| `SECURITY_LEVEL_HW_SECURE_ALL` | 5 | 含压缩/非压缩数据的全硬件通路 |

`getSecurityLevel()` 与 `openSession(int)` 在 API 28 引入。行业中的 L1/L2/L3 是 Widevine 的等级命名，与 Android 的 0–5 常量不是同一套编号；把 L1 直接等同于某个常量，是没有官方依据的映射（见「未核实与失效来源」）。

### 2.3 进程与信任边界

| 层 | 身份 | 能看到密钥吗 | 能做什么 |
|----|------|--------------|----------|
| 应用进程 | 应用 uid | 不能 | 发起请求、绑定会话、提交许可证 |
| `libmediadrm` | 应用进程内 native | 不能（仅转发） | 枚举 HAL、调用插件 |
| vendor DRM HAL | vendor 进程 | 取决于级别 | 解封密钥、执行解密 |
| TEE | 安全世界 | 能（硬件级） | 密钥保管与运算 |
| 安全解码器 | vendor 组件 | 不直接持有密钥 | 在安全通路内解码 |
| SurfaceFlinger | 系统进程 | 不能 | 合成安全图层 |

这张表解释了一个常见困惑：**为什么有些设备能播 1080p、有些只能 480p？** 因为内容授权策略会检查安全级别与 HDCP 状态；软件级实现无法满足高等级要求，许可证服务器会降级或拒绝。这不是应用能绕过的策略，而是信任链的能力差异。

## 3. 密钥请求与供应：不透明数据的边界

### 3.1 供应（provisioning）

首次使用某个 DRM 方案时，设备可能需要向供应服务注册：

- `MediaDrm.getProvisionRequest()` 返回 opaque 数据与默认 URL；
- 应用把请求发给供应服务，拿到响应；
- `provideProvisionResponse()` 把响应交回插件；
- 插件据此建立设备身份（`PROPERTY_DEVICE_UNIQUE_ID`）。

设备唯一 ID 是内容保护的重要锚点，也是隐私敏感信息。应用不应把它当作通用设备标识使用；平台也把它限制在 DRM 上下文中。

### 3.2 密钥请求（key request）

`getKeyRequest()` 的参数包括初始化数据、MIME 类型、密钥类型（流式/临时/离线）与可选参数。返回的请求是加密的，只有许可证服务器能解。请求与「会话」绑定：一个会话对应一组密钥与一个内容上下文。

`provideKeyResponse()` 提交许可证响应。响应可能包含多个密钥、有效期、策略（如 HDCP 要求、录制限制）。插件负责校验响应的完整性与归属；如果实现中缺少对「响应是否属于当前会话」的检查，就可能出现密钥串用。

### 3.3 会话生命周期

- `openSession()` 创建会话，返回会话 ID；
- 会话可查询属性、恢复（`restoreKeys`）、关闭（`closeSession`）；
- 会话数量有上限，插件应拒绝超额请求；
- 关闭会话后，绑定的解码器不能再使用该会话。

常见的工程缺陷是**会话泄漏**：异常路径未关闭会话，长期运行后耗尽配额。这在安全上通常表现为拒绝服务，而不是密钥泄露。

## 4. 解密与解码：安全通路的真实含义

### 4.1 没有 `KEY_CRYPTO`

`MediaFormat` 中**没有** `KEY_CRYPTO` 常量。开启安全播放的正确方式是：

1. 查询解码器能力：`MediaCodecInfo.CodecCapabilities` 是否支持 `FEATURE_SecurePlayback`（字符串 `"secure-playback"`）；
2. 选择名称以 `.secure` 结尾的安全解码器；
3. 在 `MediaFormat` 上调用 `setFeatureEnabled(FEATURE_SecurePlayback, true)`；
4. 用 `MediaCrypto` 绑定 DRM 会话；
5. 每帧通过 `getSampleCryptoInfo()` 取 `CryptoInfo`，再用 `queueSecureInputBuffer()` 提交。

安全解码器的输出进入安全缓冲区，普通应用无法直接映射读取。这是「安全通路」的核心：不是把明文加密一次，而是让明文只存在于受保护的内存与显示路径中。

### 4.2 ClearKey：不保密但真实存在

AOSP 包含 ClearKey 插件，UUID 为 `e2719d58-a985-b3c9-781a-b030af78d30e`。它的许可证就是 JWK Set，密钥以明文下发（W3C EME 规范中的 Clear Key 机制）。它的用途是测试与互操作基线，**不提供内容保密性**。把它当成「AOSP 自带的 DRM 所以不安全」是误解：它本来就是公开密钥的基线实现。

Widevine 插件不在 AOSP 中，由 Google 与厂商提供；因此「AOSP 里没有 Widevine」是正常现象，不代表设备没有 Widevine。

### 4.3 HDCP：只能查询，不能强制

`MediaDrm.getConnectedHdcpLevel()` 与 `getMaxHdcpLevel()`（API 28）返回 HDCP 状态与能力。官方对前者的说明明确：它**仅供参考，不用于强制执行合规**，真正的强制由 DRM 系统完成。

`Display`、`MediaRouter`、`HdmiControlManager` 中没有「应用强制 HDCP」的接口。应用能做的，是读取状态并把「不满足要求」反馈给许可证服务器或用户，而不是自行开启 HDCP。

## 5. 显示面：secure 标志的边界

`SurfaceView.setSecure(true)` 必须在 Surface 附加之前调用，它会给 `SurfaceControl` 打上 `SECURE` 标志。效果是：

- 内容不会出现在截屏与录屏中；
- 内容不会出现在非安全显示（如部分虚拟显示、投屏）上。

但它**不是万能的**：

- 文档只承诺「截屏与非安全显示」这一层，不承诺对物理拍摄、外部采集设备或已攻陷的显示通路免疫；
- 软件级 DRM 与普通（非 secure）解码路径不受此保护；
- 某些系统级采集路径（如受权的无障碍/投屏）是否被阻止，取决于平台实现与权限。

把 `setSecure(true)` 当成「内容绝不可能泄露」是对文档的过度解读。

## 6. 公开公告台账

| CVE | NVD 落点 / 类型 | 级别 | 修复 | 分类 |
|-----|----------------|------|------|------|
| [CVE-2020-0125](https://nvd.nist.gov/vuln/detail/CVE-2020-0125) | `mediadrm` 越界读（信息泄露，CWE-125） | Moderate（NVD 5.5） | 2020-09 | Media framework |
| [CVE-2021-0592](https://nvd.nist.gov/vuln/detail/CVE-2021-0592) | Widevine 越界写（RCE，CWE-787） | Critical（NVD 8.8） | 2021-07 | Widevine |
| [CVE-2021-0639](https://nvd.nist.gov/vuln/detail/CVE-2021-0639) | `libl3oemcrypto.cpp` 混淆机制弱点（信息泄露，CWE-922） | High（NVD 5.5） | 2021-08 | Widevine |
| [CVE-2023-21120](https://nvd.nist.gov/vuln/detail/CVE-2023-21120) | `cdm_engine.cpp` 锁不当导致 UAF（提权，CWE-416） | High（NVD 7.8） | 2023-06 | Hardware DRM |
| [CVE-2024-0042](https://nvd.nist.gov/vuln/detail/CVE-2024-0042) | OEM 与 DRM 证书混淆（内容保护绕过，CWE-295） | High（NVD 7.8） | 2024-04 | Widevine |

公告链接：

- Android 11：https://source.android.com/security/bulletin/android-11
- 2021-07：https://source.android.com/security/bulletin/2021-07-01
- 2021-08：https://source.android.com/security/bulletin/2021-08-01
- 2023-06：https://source.android.com/security/bulletin/2023-06-01
- 2024-04：https://source.android.com/security/bulletin/2024-04-01

读表要点：

1. **组件要分清**：`mediadrm`、Widevine 插件、`libl3oemcrypto`、`cdm_engine` 是不同代码，修复责任方也不同（AOSP、Google、OEM）。
2. **影响类型多样**：越界读、越界写、混淆弱点、UAF、证书混淆。不能把「DRM 漏洞」统一描述成「内容被解密」。
3. **级别以公告为准**：NVD 与公告可能不同，表中同时给出以便交叉核对。

## 7. 成因模式

### 7.1 不透明数据的解析边界

DRM 请求/响应、许可证、证书都是「不透明」的二进制结构。插件在解析时必须校验长度、版本、嵌套结构；CVE-2020-0125 的越界读与 CVE-2021-0592 的越界写都属于这一类。防御上，解析器要把每个外部字段当成不可信输入。

### 7.2 混淆不等于加密

`libl3oemcrypto` 的「混淆机制」被 NVD 描述为弱点（CVE-2021-0639）。这提醒我们：**在软件级实现中，密钥保护依赖混淆而非硬件隔离**；一旦混淆逻辑被理解，保护就失效。这不是「实现偷懒」，而是软件级 DRM 的固有限制。

### 7.3 并发与生命周期

`cdm_engine.cpp` 的 UAF（CVE-2023-21120）源于锁使用不当。DRM 会话是多线程访问的：应用线程、解码线程、HAL 回调都可能操作同一份状态。缺少同步或释放顺序错误会导致对象在仍被引用时被释放。

### 7.4 证书链与身份混淆

CVE-2024-0042 被描述为「OEM 与 DRM 证书混淆」，导致内容保护被绕过（CWE-295）。证书校验必须把「用途」与「签发者」绑定，不能只看签名是否有效；否则一张合法但用途不同的证书就能冒充 DRM 证书。

### 7.5 安全级别声明与实际能力不符

插件声明的 `SECURITY_LEVEL_*` 是内容授权决策的输入。如果声明高于实际能力（例如软件实现声称硬件级），许可证服务器会做出错误的内容授权判断。防御上，级别应由可验证的能力推导，而不是由配置字符串决定。

### 7.6 显示标志失效

`setSecure(true)` 的调用时机（必须在 Surface 附加前）与图层属性传递，是显示面最常见的缺陷来源。标志未生效时，内容会被普通截屏捕获，而应用侧往往没有任何报错。

## 8. 版本加固时间线

| 版本 / API | 变化 |
|------------|------|
| 4.3 / 18 | `MediaDrm` 引入；供应与密钥请求 API |
| 5.0 / 21 | 安全解码器普及；通过 `FEATURE_SecurePlayback` 开启 |
| 8.0 / 26 | DRM HAL binderized（Treble） |
| 9 / 28 | DRM HAL 1.1；`getSecurityLevel()`、HDCP 查询 |
| 10 / 29 | DRM HAL 1.2；`isCryptoSchemeSupported(..., level)`、HDCP 2.3 |
| 11 / 30 | DRM HAL 1.3；`getSupportedCryptoSchemes()` |
| 12 / 31 | DRM HAL 1.4 |
| 13 / 33 | DRM HAL 强制 AIDL（`drm/aidl`） |

趋势是**DRM 从进程内插件走向 binderized HAL，再走向稳定 AIDL**：接口越标准化，厂商实现越可替换，但「HAL 之下」的安全责任也越清晰地落到 vendor 与 TEE。评估时看到 DRM 问题，要先确认它发生在 AOSP 客户端、HAL 接口还是 vendor 插件。

## 9. 观测点与自查

### 9.1 应用侧

- 是否查询 `FEATURE_SecurePlayback` 并选择 `.secure` 解码器？是否处理了不支持时的降级？
- 是否在异常路径关闭 `MediaDrm` 会话？是否限制了并发会话数？
- 是否把 `MediaDrm` 的异常（如 `ProvisionException`、`DeniedByServerException`）当成正常分支？
- `SurfaceView.setSecure(true)` 是否在 Surface 附加前调用？
- 是否把 HDCP 状态当作「可强制」的开关？是否只是读取并反馈？

### 9.2 平台侧

- `dumpsys media.drm` 查看已注册的 DRM 插件、会话与安全级别；
- 核对 vendor 插件的 `SECURITY_LEVEL_*` 声明与实测能力；
- 关注 DRM HAL 版本与 AIDL 迁移状态；
- 审计 `mediadrmserver` 相关 SELinux 策略与实际进程模型是否一致。

### 9.3 评估侧

- 先定位落点：AOSP 客户端、HAL 接口、vendor 插件、TEE、解码器、显示；
- 区分「密钥泄露」「内容解密」「证书绕过」「拒绝服务」「显示标志失效」；
- 不要把所有 Widevine CVE 都归到 AOSP——Widevine 插件由 Google/厂商维护。

## 10. 防御要点

1. **按能力选解码器**：先查 `FEATURE_SecurePlayback`，再选 `.secure`，最后才设置 feature；顺序错误会导致配置被忽略。
2. **会话必须关闭**：用 `try/finally` 管理 `openSession` / `closeSession`，避免配额耗尽。
3. **不缓存密钥材料**：应用不应保存任何许可证响应中的密钥字段；只保存会话 ID 与必要的策略信息。
4. **校验证书用途**：如果实现自定义证书校验，必须同时校验签发链与用途，不能只看签名有效。
5. **安全级别可验证**：不要仅凭配置字符串信任插件声明；用平台提供的查询接口与实测结果交叉验证。
6. **正确使用 secure 图层**：在 Surface 附加前设置，并验证截屏确实被阻止。
7. **HDCP 只读**：把 HDCP 状态作为策略输入反馈给许可证服务器，不要试图由应用强制。

## 11. 常见误传纠正

### 误传一：L1 用软件实现即可

错。行业语境中的 L1 通常对应硬件级安全（密钥与运算在 TEE），而 Android 的软件级常量明确写着 whitebox 软件实现。混淆这两套命名会得出完全错误的结论。

### 误传二：用 `KEY_CRYPTO` 开启安全解码

错。`MediaFormat` 中没有 `KEY_CRYPTO` 常量；正确路径是 `setFeatureEnabled(FEATURE_SecurePlayback, true)` + `MediaCrypto` + `.secure` 解码器。

### 误传三：secure 缓冲区对所有录屏免疫

过度解读。文档只承诺不出现在截屏与非安全显示；软件级实现、非 secure 路径以及物理采集不在承诺范围内。

### 误传四：应用可以强制 HDCP

错。`getConnectedHdcpLevel()` 明确标注「仅供参考，不用于强制执行合规」；强制由 DRM 系统完成，应用没有对应接口。

### 误传五：ClearKey 不安全，所以可以忽略

错。ClearKey 是 W3C EME 的必选基线，也是 AOSP 中真实存在的 HAL 插件；它确实不保密内容，但这不等于它没有安全影响——错误的实现同样可能引入解析或会话缺陷。

### 误传六：L1/L2/L3 是 Android 官方安全等级

错。Android 使用 0–5 的 `SECURITY_LEVEL_*` 常量；L1/L2/L3 是 Widevine 的行业命名，官方文档没有给出与 0–5 的逐项映射。

### 误传七：DRM 漏洞等于编解码漏洞

错。NVD 以 `mediadrm`、WVDrmPlugin、`libl3oemcrypto`、`cdm_engine` 等组件区分落点；编解码器漏洞是另一条通路。

## 12. 未核实与失效来源

- Widevine 的 L1/L2/L3 与 TEE、解码、缓冲区之间的对应关系没有官方逐项说明；`/guides/security_levels` 页面 404，本文不给出确定映射。
- `MediaDrmPlugin` 在 main、9.0、7.1.2、5.1.1 的源码路径均 404，现行接口是 `IDrm` / `DrmHal*`；`MediaDrmPlugin` 的历史归属未核实。
- 官方文档称 `mediadrmserver` 创建 CryptoHal/DrmHal，但 main 分支没有对应目录，而 SELinux 仍保留 `mediadrmserver.te`；两者矛盾，本文按源码描述进程内 JNI + HAL 的路径。
- Widevine 证书链、插件 `.so` 实名与 provisioning 协议细节未在公开文档中给出。
- Android 14/15 的 DRM 行为变更没有找到公开条目，时间线止于 13 的 AIDL 强制。

## 13. 参考资料

- MediaDrm（官方参考）：
  https://developer.android.com/reference/android/media/MediaDrm
- MediaCodecInfo.CodecCapabilities（安全播放能力）：
  https://developer.android.com/reference/android/media/MediaCodecInfo.CodecCapabilities#FEATURE_SecurePlayback
- SurfaceView.setSecure（官方参考）：
  https://developer.android.com/reference/android/view/SurfaceView#setSecure(boolean)
- DRM 架构（官方文档）：
  https://source.android.com/docs/core/media/drm
- DrmUtils（AOSP）：
  https://android.googlesource.com/platform/frameworks/av/+/refs/heads/main/drm/libmediadrm/DrmUtils.cpp
- ClearKey UUID（AOSP）：
  https://android.googlesource.com/platform/frameworks/av/+/refs/heads/main/drm/mediadrm/plugins/clearkey/common/ClearKeyUUID.cpp
- DRM HAL 接口（AOSP）：
  https://android.googlesource.com/platform/hardware/interfaces/+/refs/heads/main/drm/
- W3C Encrypted Media Extensions（Clear Key）：
  https://www.w3.org/TR/encrypted-media/#clear-key
- NVD 条目（本文台账）：
  https://nvd.nist.gov/vuln/detail/CVE-2020-0125
  https://nvd.nist.gov/vuln/detail/CVE-2021-0592
  https://nvd.nist.gov/vuln/detail/CVE-2021-0639
  https://nvd.nist.gov/vuln/detail/CVE-2023-21120
  https://nvd.nist.gov/vuln/detail/CVE-2024-0042

---

## 联系方式

咨询 Android 系统详情请联系 https://t.me/one00190

---

*本手册仅供安全学习与防御研究使用；所有机制表述、公告列值与 CVE 描述均取自公开一手来源并附链接，未能核实的部分已单列于「未核实与失效来源」而非写入正文。*
