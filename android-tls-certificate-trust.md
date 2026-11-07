---
layout: page
title: "Android TLS 与证书信任链：从 TrustManager 到系统信任库的完整验证路径"
description: "把一次 HTTPS 连接拆成协议协商、链验证、信任锚判定、主机名匹配与额外策略五道关，逐层讲清 Conscrypt（BoringSSL、`com.android.conscrypt` APEX 自 Android 10、内部实现类非公开 API）、系统信任库（`<subject_hash_old>.<n..."
date: 2026-11-07
permalink: /android-tls-certificate-trust/
---

手机上的每一次 HTTPS 请求，最终都要回答两个问题：**这个服务器是不是它声称的那个**，以及**我凭什么相信**。Android 把这两个问题的答案拆成了好几层——BoringSSL 负责握手，Conscrypt 负责把 OpenSSL 语义包装成 Java API，`TrustManagerImpl` 负责链验证、主机名匹配、证书固定与透明度检查，系统信任库负责提供根锚点，网络安全配置（Network Security Config，下称 NSC）负责让每个应用改写这些默认值。

本文沿着这条路径逐层拆开，说明每一层的真实接口、默认值与可被改动的点，再归纳公开记录过的失效模式与防护做法。**范围限于已公开且已修复的问题**，不涉及任何针对在线服务的攻击方法；对无法从官方来源确认的说法，一律放入「未核实」一节。

---

## 1. 一条连接要过五道关

把 TLS 验证拆成层，比记住某个 API 更有用。下面是 Android 上一次 HTTPS 连接实际经过的判定顺序：

| 顺序 | 关卡 | 由谁执行 | 失败后果 |
|------|------|----------|----------|
| 1 | 协议与密码套件协商 | BoringSSL（经 Conscrypt） | 握手失败 |
| 2 | 证书链构建与签名验证 | `TrustManagerImpl` | `CertificateException` |
| 3 | 信任锚判定 | 系统库 / 用户库 / NSC 指定锚 | `CertificateException` |
| 4 | 主机名匹配 | `TrustManagerImpl` / `HostnameVerifier` | `SSLPeerUnverifiedException` 等 |
| 5 | 额外策略（证书固定、CT） | `TrustManagerImpl` 内部的 PinManager / CT 检查 | 握手失败 |

容易混淆的是第 2 步与第 3 步：**链能验通，不等于锚点可信**。攻击者完全可以自签一张与目标域名完全一致的证书，链本身没有任何问题，问题出在「根不在信任库里」。反过来，**根可信也不等于主机名匹配**——把 `example.com` 的证书拿去连 `example.org`，链验证会通过，主机名校验会拦下。

> 绝大多数「TLS 绕过」的真实形态，不是攻破密码学，而是让这三道关中的某一道**不执行**。这也是后文「漏洞成因」一节的中心论点。

---

## 2. Conscrypt：Android 的 TLS 实现

### 2.1 它是什么

Conscrypt 是 Android 的 TLS 与密码学实现。官方描述是：它「用 Java 代码加一个本地库，提供 Android 的 TLS 实现以及相当大一部分 Android 密码学功能」。底层本地库是 **BoringSSL**——Google 维护的 OpenSSL 分支。

它不是一个可选依赖，而是系统的一部分。开发者平时写的 `HttpsURLConnection`、`OkHttp`、`HttpClient`，最终都落到 `javax.net.ssl.SSLSocket` / `SSLEngine`，而 Android 上这两个抽象的实现就是 Conscrypt。

官方还有一句关键说明：

> Android 的默认 `SSLSocket` 实现基于 Conscrypt。自 Android 11 起，该实现在内部构建于 Conscrypt 的 `SSLEngine` 之上。

这意味着 Android 11 之后，阻塞式 `SSLSocket` 与非阻塞式 `SSLEngine` 共用同一套握手与验证逻辑，过去两条路径行为不一致的问题被收敛了。

### 2.2 包名与 API 边界

这里有一个非常常见的误解：**`com.android.org.conscrypt.*` 不是给应用用的 API**。

- 上游开源库的包名是 `org.conscrypt`；
- AOSP 构建时经 `repackaged/**` 重打包为 `com.android.org.conscrypt`，以避免与第三方打包的 Conscrypt 冲突；
- `ConscryptEngine`、`ConscryptEngineSocket`、`ConscryptFileDescriptorSocket` 这些名字虽然出现在源码里，但都是**包内可见的实现类**，不是公开 API。

应用应当依赖的是标准接口：

- `javax.net.ssl.*`（`SSLSocket`、`SSLEngine`、`X509TrustManager`、`SSLContext`）；
- `android.net.ssl.SSLSockets`（在标准 socket 上开关 ALPN 等）；
- `android.net.http.X509TrustManagerExtensions`（暴露 Android 增加的「按主机名验证」与「是否用户安装证书」能力）。

> 只要代码里出现 `com.android.org.conscrypt` 或 `org.conscrypt` 的 import，这段代码就绑定了某个 Android 版本的内部实现，跨版本随时可能失效。这是兼容性问题，不是安全问题，但会诱使开发者为了「修编译错误」而绕开正确的验证路径。

### 2.3 APEX：可以单独更新的 TLS

Conscrypt 以 APEX 形式发布，模块名 **`com.android.conscrypt`**，自 **Android 10** 起进入 Mainline 更新体系。

这件事的安全意义常被低估：

- TLS 实现是高频攻击面，历史上多次出现内存安全与验证逻辑缺陷；
- 过去修这类问题只能等整机 OTA，碎片化设备可能数月甚至数年拿不到修复；
- 改为 APEX 后，Conscrypt 可以随 Google Play 系统更新独立下发，**不必等厂商整包**。

代价是：设备上的实际 TLS 行为取决于「系统版本 + Conscrypt 模块版本」两个变量。做兼容性排查时，只报 Android 版本号是不够的。

### 2.4 TLS 1.3 的默认值

官方口径很明确：

- **Android 10 起，TLS 1.3 对所有 TLS 连接默认启用**；
- TLS 1.3 的密码套件**不可自定义**；
- **不支持 0-RTT**（0-RTT 的重放风险是官方不做支持的直接原因）；
- Android 10 起，TLS 连接中**不再信任 SHA-1 签名的证书**；
- **Android 15 起限制 TLS 1.0 与 1.1**。

对开发者而言，能配置的空间比想象中小：协议版本可以设下限，套件基本不可选。想「兼容老旧服务器」而把下限调到 TLS 1.0，在 Android 15 上会直接失效——这不是 bug，是既定策略。

---

## 3. 信任库：根锚点从哪里来

### 3.1 系统 CA 的存放与命名

系统根证书以单文件形式存放，文件名格式是 **`<hash>.<n>`**，例如 `7651b327.0`。

这个 `hash` 不是证书的 SHA-1 指纹，也不是新版 OpenSSL 的 `-subject_hash`，而是 **旧版 subject name hash**。AOSP 的 `system/ca-certificates/README.cacerts` 写得很直白：文件名里的 hash 来自

```
openssl x509 -subject_hash_old -in filename
```

`.n` 是从 0 开始的序号，用来处理同一 subject 下多张证书的哈希碰撞。

> 这一点在排查「自建 CA 装进系统库不生效」时特别关键：用 `-subject_hash` 生成的文件名看起来对，系统却找不到。正确做法是 `-subject_hash_old`。

### 3.2 Android 14：根信任库搬进 APEX

Android 14 之前，系统 CA 在 `/system/etc/security/cacerts/`。AOSP 代码里，当系统判断应使用 APEX（SDK 34+）时，系统 CA 目录切换为：

```
/apex/com.android.conscrypt/cacerts
```

官方文档的表述是：

> Android 14 在 Conscrypt 内引入了可更新的根信任库。
> 从 Android 14 起，根信任证书存放在 Conscrypt 模块 APEX 与系统分区中。

这带来的直接后果是：**根信任库也能随 Mainline 更新**。过去某个根 CA 被吊销，要等整机 OTA；现在可以通过模块更新下发。对企业与 ROM 维护者来说，检查「设备到底信任哪些根」时，不能再只看 `/system`。

### 3.3 用户 CA：另一个库

用户手动安装或由 MDM/DPC 下发的 CA，不进系统库，而是放在：

```
/data/misc/keychain/cacerts-added
```

被用户禁用的证书记录在 `cacerts-removed`。这两个目录由 `TrustedCertificateStore` 管理。

用户库的存在本身是合理的：企业内网、抓包调试、自签服务都需要它。风险在于**默认信任策略**——见下一节。

### 3.4 默认信任策略：按 targetSdk 分档

这是全篇最容易被记错的一张表：

| 应用 targetSdk | 明文流量 | 系统 CA | 用户 CA |
|----------------|----------|---------|---------|
| ≤ 23（Android 6.0 及以下） | 允许 | 信任 | **默认信任** |
| 24–27（Android 7.0–8.1） | 允许 | 信任 | 不信任 |
| ≥ 28（Android 9+） | **默认禁止** | 信任 | 不信任 |

补充两条官方口径：

- **特权应用（privileged apps）在 Android P（9）及以上不信任用户证书库**。也就是说，即便某个系统应用把 targetSdk 留在 23，也不能靠旧默认值吃到用户 CA。
- Android 14 起 targetSdk < 23 的应用无法安装；Android 15 起 targetSdk < 24 的应用无法安装。旧默认值正在被安装门槛逐步关死。

> 「targetSdk 24 起不信任用户 CA」是**按应用**生效的，不是设备全局开关。设备上装了用户 CA，只影响那些 targetSdk ≤ 23 的应用（以及显式在 NSC 里声明信任 `user` 的应用）。把「装了抓包证书就能抓所有 App」当成普遍事实，是错的。

### 3.5 信任库的更新链路

把上面几节串起来，Android 的信任锚有四个来源，更新节奏各不相同：

| 来源 | 位置 | 更新方式 | 典型延迟 |
|------|------|----------|----------|
| 系统根库（旧） | `/system/etc/security/cacerts` | 整机 OTA | 取决于厂商 |
| 系统根库（Android 14+） | `/apex/com.android.conscrypt/cacerts` | Mainline 模块更新 | 数周级 |
| 用户库 | `/data/misc/keychain/cacerts-added` | 用户/DPC 安装 | 即时 |
| 应用内嵌锚 | APK 资源（`@raw/...`） | 应用更新 | 随发版 |

排查「为什么这台设备信任某张证书」时，要按这张表逐个排除，而不是只看系统版本。

---

## 4. 网络安全配置：应用级策略层

### 4.1 入口

在 `AndroidManifest.xml` 的 `<application>` 上声明：

```xml
<application
    android:networkSecurityConfig="@xml/network_security_config"
    ... >
```

对应 `res/xml/network_security_config.xml`。没有声明时，应用使用平台默认值（即 3.4 节的表）。

### 4.2 基本结构

一个覆盖常见需求的配置如下（示例，非推荐模板）：

```xml
<network-security-config>
    <base-config cleartextTrafficPermitted="false">
        <trust-anchors>
            <certificates src="system" />
        </trust-anchors>
    </base-config>

    <domain-config cleartextTrafficPermitted="false">
        <domain includeSubdomains="true">api.example.com</domain>
        <trust-anchors>
            <certificates src="system" />
        </trust-anchors>
        <pin-set expiration="2027-06-30">
            <pin digest="SHA-256">AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA=</pin>
            <pin digest="SHA-256">BBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBB=</pin>
        </pin-set>
    </domain-config>

    <debug-overrides>
        <trust-anchors>
            <certificates src="user" />
        </trust-anchors>
    </debug-overrides>
</network-security-config>
```

几个标签的语义：

| 标签/属性 | 作用 |
|-----------|------|
| `<base-config>` | 所有域名的默认策略 |
| `<domain-config>` | 针对特定域名的覆盖，可嵌套 |
| `<domain includeSubdomains="true">` | 是否包含子域 |
| `<trust-anchors>` | 信任锚集合 |
| `<certificates src="system">` | 系统库 |
| `<certificates src="user">` | 用户库 |
| `<certificates src="@raw/name">` | 应用内嵌的证书资源 |
| `<pin-set expiration="yyyy-MM-dd">` | 证书固定集合 |
| `<pin digest="SHA-256">` | 单个 pin，当前**只支持 SHA-256** |
| `<debug-overrides>` | 仅在 `android:debuggable="true"` 时生效的覆盖 |

### 4.3 pin-set 的真实语义

关于证书固定，官方文档里有两句话必须一起读：

> `expiration`：pin 的过期日期，格式 `yyyy-MM-dd`，**到期后即停用固定**（thus disabling pinning）。若未设置该属性，pin 不会过期。

> 给 pin 设置过期时间**可能让攻击者绕过**你固定的证书。

也就是说：

1. **过期不是「连接失败」，而是「不再固定」**。很多人以为 pin 过期会让 App 直接断网，从而把它当作强制轮换的保险丝——事实相反，过期那一刻起，固定保护静默消失。
2. **`expiration` 不是必填项**，即便 targetSdk ≥ 24 也没有强制。
3. `digest` 只支持 `SHA-256`，写 `SHA-1` 或 `sha256` 都不会按预期工作。

第 1 点是最值得写进检查清单的：**证书固定过期是静默降级，不会在日志里报错**。

### 4.4 debug-overrides 的边界

官方措辞：

> 当 `android:debuggable` 为 `"true"` 时应用的覆盖项……`debug-overrides` 中指定的信任锚会**被加到所有其他配置之上**，并且当服务器证书链使用这些仅调试用信任锚时，**不执行证书固定**。若 `android:debuggable` 为 `"false"`，该段**完全忽略**。

这带来两个推论：

- 调试期用用户 CA 抓包是官方支持的路径，不必去改 `TrustManager`；
- 风险点是**发布构建的 `debuggable` 标志**。一旦某个渠道包误开 `debuggable="true"`，`debug-overrides` 就会随之上线，用户 CA 被信任、固定被跳过，而这一切在代码 review 里看不出来——问题出在构建配置。

### 4.5 官方明确不推荐的写法

开发者在「连不上自签服务器」时最常搜到的三种改法，官方都有专页点名：

1. **空实现的 `X509TrustManager`**：`checkServerTrusted` 什么都不做。官方原文是「很多网站介绍一种糟糕的替代方案，就是装一个什么都不做的 TrustManager……**不要这么做，哪怕只是临时**」。
2. **恒返回 true 的 `HostnameVerifier`**：官方有专页，lint 也有 `BadHostnameVerifier` 检查。`HostnameVerifier` 本身已被 `NetworkSecurityConfig` 取代。
3. **WebView 里 `SslErrorHandler.proceed()`**：官方在 `SslErrorHandler` 的文档里加了 Warning——「宿主应用应始终调用 `cancel()` 而不是 `proceed()`，因为无效的 SSL 证书意味着连接不安全」；`onReceivedSslError` 的文档也写明「应始终调用 `SslErrorHandler#cancel()`，绝不要越过错误继续」。

这三条的共同点是：**它们不是「放宽验证」，而是「取消验证」**。一旦上线，中间人可以出示任意自签证书。

---

## 5. 证书固定与 Certificate Transparency

### 5.1 两代「额外验证」

链验证回答「证书是否由受信根签发」，固定（pinning）回答「是否**只**接受这几张证书/这几个公钥」，CT（Certificate Transparency）回答「这张证书是否被公开记录过」。三者是叠加关系，不是替代关系。

### 5.2 固定：官方立场与历史教训

浏览器侧的 HPKP（HTTP Public Key Pinning，RFC 7469）因为「配错就把站点锁死」而被厂商陆续移除。需要澄清的是：**RFC 7469 在 RFC 层面至今仍是 Proposed Standard，没有被标为 Historic，也没有 obsoleting RFC**；浏览器移除属厂商决策，不等于标准被撤销。

Android 侧的态度更直接。官方在 TLS 安全页写明：

> 证书固定……**不推荐用于 Android 应用**。未来服务器配置变更……会让固定了证书的应用无法连接……

这不是说固定没有价值，而是说它的**运维成本与故障半径**对普通应用不划算。真要固定，官方给出的路径是 NSC 的 `<pin-set>`，而不是自己写 `TrustManager`。

### 5.3 CT：从「浏览器的事」变成平台能力

一个流传很广的说法是「Android 平台不做 CT，只有 Chrome 做」。这个说法在 Android 16 之后已经过时：

- **Android 16（API 36）**：CT 支持首次进入平台，通过 NSC 的 `<certificateTransparency enabled="true"/>` **选择启用**；
- **Android 17（API 37）**：官方文档写明 CT **默认启用**，应用可退出；
- Android 15 及以下：平台不提供该能力。

配套的查询接口是 `NetworkSecurityPolicy.isCertificateTransparencyVerificationRequired(String hostname)`，**API 36** 引入。

需要特别注意：**不存在 `android.security.CertificateTransparency` 这个类**（SDK reference 查不到）。CT 的接入点是 NSC 元素，加上 `X509TrustManagerExtensions` 在 API 36 新增的、带 `ocspData` 与 `tlsSctData` 参数的 `checkServerTrusted` 重载。

另外，CT 检查对**用户库或应用内嵌的信任锚会被跳过**，除非显式强制——这是设计上的取舍：自签/内网证书本来就不会进公共日志。

### 5.4 这些检查在哪里发生

`TrustManagerImpl` 不是只做链验证。源码层面它至少调用两处：

- `pinManager.checkChainPinning(host, wholeChain)` —— 固定检查；
- `ct.checkCT(wholeChain, ocspData, tlsSctData, host)` —— 透明度检查。

这意味着固定与 CT **不是应用层附加的钩子，而是验证路径的一部分**。反过来说，任何「替换 TrustManager」的做法，都会把这些内建检查一起绕过——包括你以为还在生效的固定。

---

## 6. 漏洞成因：五类真实失效模式

### 6.1 验证被短路（最常见，也最不需要「漏洞」）

这一类严格说不是平台漏洞，而是应用把自己变成了不设防状态：

- 空 `checkServerTrusted`；
- 恒真 `HostnameVerifier`；
- WebView `proceed()`；
- 自定义 `SSLContext` 时误用 `TrustAllManager`。

它们的共同特征是**代码看起来在「处理 SSL」**，实际把第 2–5 关全部跳过。检测手段也成熟：静态扫描（lint 的 `BadHostnameVerifier`、各类 TrustManager 规则）、动态抓包看客户端是否接受自签证书。

### 6.2 验证逻辑本身的缺陷：CVE-2016-0818

这是 Android TLS 历史上最值得研究的一条：**`TrustManagerImpl` 的中间 CA 缓存逻辑存在缺陷，可能导致信任判定错误，进而被中间人利用**。公告定为 Critical，修复覆盖 4.4.4 / 5.0.2 / 5.1.1 / 6.0 / 6.0.1。

它说明一件事：**「链验证正确」本身是需要被验证的**。缓存、复用、连接池这些性能优化，一旦把「上一次验证通过的结论」错误地套用到不同主机或不同链上，就会产生信任错配。这类缺陷不会出现在应用代码里，只能靠平台更新修——这也是 Conscrypt 被做成 APEX 的现实理由之一。

> 这条 CVE 常被误写成「Master Key 漏洞」。Master Key 是 CVE-2013-4787（APK 签名校验），与 TLS 无关。

### 6.3 内存安全：Conscrypt 的解析面

Conscrypt 解析的是攻击者可控的字节流：证书链、握手消息、ASN.1 结构。历史公告里有若干条落在这一层：

- CVE-2015-3837：`OpenSSLX509Certificate`，影响 5.1 及以下；
- CVE-2016-2461 / CVE-2016-2462：提权类；
- CVE-2016-3840：RCE；
- CVE-2016-6709：信息泄露；
- CVE-2017-13309：`ConscryptEngine.readEncryptedData` 明文泄露（AOSP 8.1）。

这些问题的共同背景是：**TLS 栈是「未认证的对端也能触达」的代码路径**。任何在证书解析、扩展解析、记录层分片上的越界或状态错误，都可能在握手阶段被触发。用 BoringSSL 替换 OpenSSL、把模块 APEX 化、持续模糊测试，都是针对这一类成因的工程回应。

### 6.4 配置漂移

平台默认值在变，配置却常常停在旧版本：

| 漂移形态 | 后果 |
|----------|------|
| `debug-overrides` 随 `debuggable=true` 进入发布包 | 信任用户 CA、跳过固定 |
| `pin-set` 过期未续 | 固定静默失效 |
| 为兼容旧服务器保留 `cleartextTrafficPermitted="true"` | 明文流量被允许 |
| 显式 `<certificates src="user"/>` 留在生产配置 | 用户 CA 可参与验证 |
| targetSdk 长期不升 | 吃到旧默认值（直到安装门槛关死） |

这类问题不需要任何漏洞，靠**时间**就能发生。它们也是「同一个 APK 在不同年份风险不同」的原因。

### 6.5 生态层信任错位

最后一类不在代码里，在设备管理面：

- 用户被诱导安装恶意根 CA（「装个证书才能上网」）；
- 企业 MDM/DPC 下发 CA 用于流量审计，员工不知情；
- 攻击者取得设备管理权限后安装 CA，配合代理做中间人。

平台的对策是分层的：targetSdk ≥ 24 默认不信任用户 CA；特权应用在 Android 9+ 不信任用户库；Android 14 起根库可更新以快速吊销；企业侧则依赖工作资料与 DPC 的可见性。**但这些都不保护那些显式信任 `user` 的应用**——是否信任用户库，最终是应用自己的配置决定。

---

## 7. 公开公告台账（已核实）

下表只收录能从 Android 安全公告或 NVD 核对的条目。等级与公告月份以公告页为准。

| CVE | 公告月份 | 等级 | 落点 | 说明 |
|-----|----------|------|------|------|
| CVE-2015-3837 | 2015-08 | High | `OpenSSLX509Certificate` | 影响 5.1 及以下 |
| CVE-2016-0818 | 2016-03 | Critical | **`TrustManagerImpl`** | 中间 CA 缓存导致信任判定错误，可被中间人利用；修复于 4.4.4 / 5.0.2 / 5.1.1 / 6.0 / 6.0.1 |
| CVE-2016-2461 | 2016-05 | Moderate | Conscrypt | 提权 |
| CVE-2016-2462 | 2016-05 | Moderate | Conscrypt | 提权 |
| CVE-2016-3840 | 2016-08 | Critical | Conscrypt | 远程代码执行 |
| CVE-2016-6709 | 2016-11 | High | Conscrypt | 信息泄露 |
| CVE-2017-13309 | 2018-05 | High | `ConscryptEngine.readEncryptedData` | 明文数据泄露（AOSP 8.1） |
| CVE-2024-49723 | 2025-02（Mainline 表） | — | Conscrypt | 公告表列出，但 **NVD 与 cve.org 均无记录**，仅登记不展开 |

读这张表要注意两点：

1. **数量少不代表风险低**。TLS 栈的高危问题往往在「被利用之前就被内部发现并静默修复」，进不了公告表；公告只覆盖有 CVE 编号的部分。
2. **等级与影响面不是一回事**。Critical 表示潜在后果严重，不表示利用门槛低；反过来，Moderate 的提权在特定链条里可能很关键。

---

## 8. 防护清单

### 8.1 应用开发者

1. **不要自己实现信任判断**。用平台默认值或 NSC，不要写 `TrustAllManager`，不要恒真 `HostnameVerifier`，不要 `proceed()`。
2. **targetSdk 保持在受支持区间**。API 28+ 的明文默认禁止、API 24+ 的用户 CA 默认不信任，都是免费的安全默认值。
3. **用 NSC 而不是代码做例外**。需要信任内网 CA 时，用 `<domain-config>` 精确到域名，避免 `<base-config>` 全局放开。
4. **`debug-overrides` 只服务于调试**。确认发布构建 `debuggable=false`；在 CI 里对发布包做一次 manifest 检查。
5. **固定要么不做，要么做对**。若确实需要 `<pin-set>`：至少两个 pin（一张备份）、设置合理的 `expiration` 并**把续期纳入运维日历**，同时接受「过期即停止固定」的语义。
6. **WebView 单独审**。`onReceivedSslError` 必须 `cancel()`；不要为了「让页面能打开」而 `proceed()`。
7. **证书透明度按需启用**。targetSdk 36+ 可按业务评估 `<certificateTransparency enabled="true"/>`；targetSdk 37+ 需了解默认开启并评估退出条件。

### 8.2 企业与 MDM

1. **区分「审计」与「信任」**。下发 CA 用于流量审计时，明确告知范围与留存策略；不要把审计 CA 装成全局根。
2. **利用工作资料边界**。工作资料内的策略与个人侧分离，降低个人应用被审计面波及的概率。
3. **盘点信任锚**。定期核对设备上的用户 CA 列表与 DPC 下发记录；`DevicePolicyManager` 提供 `installCaCert` / `getInstalledCaCerts` 等能力（后者返回的列表**不含系统 CA**）。
4. **关注安装门槛**。targetSdk 过低的应用会在新系统上装不上，这既是兼容问题，也是清理旧默认值的机会。

### 8.3 普通用户

1. **不装来路不明的根证书**。「装证书才能上网/才能用某功能」是典型话术。
2. **定期检查用户证书**。系统设置里的「信任的凭据 → 用户」列表应当只有你认识的条目。
3. **公共 Wi-Fi 下的证书告警不要忽略**。浏览器或应用报证书错误时，正确动作是停止访问，而不是寻找「继续」按钮。

### 8.4 厂商与 ROM 维护者

1. **保持 Conscrypt 模块可更新**。APEX 化的价值只有在模块真的能更新时才成立。
2. **不要把调试根证书预置进系统库**。工程样机的便利配置一旦流入量产，就是全局信任后门。
3. **核对 cacerts 文件名**。自建 CA 入库必须用 `-subject_hash_old`；升级 Android 14+ 时核对 `/apex/com.android.conscrypt/cacerts` 与 `/system/etc/security/cacerts` 的一致性。
4. **跟踪 CT 时间线**。Android 16 选择启用、Android 17 默认启用，预装应用与系统组件需要提前评估。

---

## 9. 误传纠正

1. **「CVE-2016-0818 是 Master Key 漏洞」**——错。Master Key 是 CVE-2013-4787（APK 签名校验）；CVE-2016-0818 是 Conscrypt `TrustManagerImpl` 的中间 CA 缓存信任缺陷，属 TLS 验证问题。
2. **「pin-set 过期会让连接失败」**——错。官方语义是**到期停止固定**，连接继续，保护静默消失。
3. **「Google 推荐 Android 应用做证书固定」**——错。官方原文是 "not recommended for Android apps"，理由是运维变更会导致应用无法连接。
4. **「RFC 7469 已被标为 Historic / 已废弃」**——RFC 层面不成立。rfc-editor 信息页仍是 Proposed Standard（2015-04），没有 obsoleting RFC；浏览器移除是厂商行为。
5. **「CVE-2017-13310 是 Conscrypt 漏洞」**——错，它是 `ViewPager.java createFromParcel` 的权限绕过。
6. **「CVE-2018-9520 是 Android TLS 漏洞」**——该编号不存在（NVD 查询 0 条、cve.org 404），不要引用。
7. **「CVE-2020-0404 与 TLS 有关」**——错，它是内核 `uvc_driver.c` 的链表破坏。
8. **「CVE-2021-39624 是 Conscrypt 问题」**——错，它是 `PackageManager` 的永久 DoS。
9. **「CVE-2014-0224（CCS Injection）影响所有 Android 版本」**——无法从官方来源证实。NVD 记录没有 Android CPE，也没有 Android 公告引用；不要给出受影响版本。
10. **「`android.security.net.config.DomainConfig` 是配置类」**——AOSP 里是 `Domain.java`，没有 `DomainConfig`。
11. **「`NetworkSecurityConfig` 是稳定公开 API」**——AOSP 源码中该类为 `@hide`；SDK reference 直到 37.2 文档面才出现少量方法。应按「隐藏 API、不可依赖」处理。
12. **「`ConscryptEngine` 是给应用调用的 API」**——它是包内可见的实现类，公开面是 `javax.net.ssl.*` 与 `android.net.ssl.SSLSockets`。
13. **「cacerts 文件名里的 hash 是证书 SHA-1」**——不是。是 OpenSSL 旧版 subject name hash（`openssl x509 -subject_hash_old`）。
14. **「平台完全不做 Certificate Transparency」**——Android 16 起已提供平台级支持（选择启用），Android 17 计划默认启用。

---

## 10. 未核实与需以官方文本为准的内容

1. **Conscrypt 成为 Android 默认 TLS 提供者的确切版本**：官方文档未给出日期；只能确认「Android 的默认 `SSLSocket` 实现基于 Conscrypt」以及「Android 11 起其内部构建于 `SSLEngine` 之上」。NVD 在 5.1 出现 `org/conscrypt/*` 只是间接线索，本文不据此断言具体版本。
2. **「工作资料或私有空间不信任用户 CA」**：Android 14/15 的行为变更页没有此表述。本文只采用「特权应用在 Android 9+ 不信任用户库」这一有出处的说法。
3. **`DevicePolicyManager.installCaCert` 装入用户库的逐字措辞**：官方页面抓取被截断，本文只作定性引用。
4. **CVE-2024-49723 的细节**：2025-02 Mainline 表列出 Conscrypt 条目，但 NVD 与 cve.org 均无记录，本文仅登记编号，不描述影响。
5. **Android 17 的 CT 默认开启**：属官方文档的前向声明，本文按「计划」表述，不声称已在现网设备生效。
6. **`NetworkSecurityConfig` 的公开状态**：AOSP 源码（`@hide`）与 SDK 37.2 文档面存在矛盾，本文按「隐藏 API」处理并保留此注记。

---

## 11. 参考资料

- Android TLS 与网络安全总览（Conscrypt、TLS 1.3、安全实践）：
  https://developer.android.com/privacy-and-security/security-ssl
- 网络安全配置（NSC 标签、pin-set、debug-overrides、CT 元素）：
  https://developer.android.com/privacy-and-security/security-config
- 证书透明度策略：
  https://developer.android.com/privacy-and-security/certificate-transparency-policy
- 不安全 TrustManager：
  https://developer.android.com/privacy-and-security/risks/unsafe-trustmanager
- 不安全 HostnameVerifier：
  https://developer.android.com/privacy-and-security/risks/unsafe-hostname
- `NetworkSecurityPolicy` 参考：
  https://developer.android.com/reference/android/security/NetworkSecurityPolicy
- `X509TrustManagerExtensions` 参考：
  https://developer.android.com/reference/android/net/http/X509TrustManagerExtensions
- `SslErrorHandler` 参考（proceed 警告）：
  https://developer.android.com/reference/android/webkit/SslErrorHandler
- Conscrypt 模块（APEX、Android 14 根信任库）：
  https://source.android.com/docs/core/ota/modular-system/conscrypt
  https://source.android.com/docs/core/architecture/modular-system/conscrypt
- Mainline 模块表（`com.android.conscrypt`）：
  https://source.android.com/docs/core/ota/modular-system
- 系统 CA 目录、用户 CA 目录与文件名规则（AOSP）：
  https://android.googlesource.com/platform/external/conscrypt/+/refs/heads/main/platform/src/main/java/org/conscrypt/TrustedCertificateStore.java
  https://android.googlesource.com/platform/system/ca-certificates/+/refs/heads/main/README.cacerts
- `TrustManagerImpl`（固定与 CT 调用点）：
  https://android.googlesource.com/platform/external/conscrypt/+/refs/heads/main/common/src/main/java/org/conscrypt/TrustManagerImpl.java
- 网络安全配置源码目录（`@hide` 与类清单）：
  https://android.googlesource.com/platform/frameworks/base/+/refs/heads/main/core/java/android/security/net/config/
- 2016-03 Android 安全公告（CVE-2016-0818）：
  https://source.android.com/docs/security/bulletin/2016-03-01
- 2018-05 Android 安全公告（CVE-2017-13309）：
  https://source.android.com/docs/security/bulletin/2018-05-01
- NVD（逐条 CVE 核对入口）：
  https://nvd.nist.gov/vuln/search
- RFC 7469（HPKP，Proposed Standard）：
  https://www.rfc-editor.org/info/rfc7469/

---

## 联系方式

咨询 Android 系统详情请联系 https://t.me/one00190

---

*本文仅供安全学习与防御研究使用，不含可利用代码；所述行为请遵守当地法律与目标系统授权范围。文中事实以官方文档与公告为准，无法核实之处已单列说明。*
