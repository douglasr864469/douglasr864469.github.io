---
layout: page
title: "Android 账户、凭据与无密码认证手册"
description: "把「证明你是你」拆成三条绑定对象不同的链路：`AccountManager` 的账户只由 `name` + `type` 标识、数据在 system_server 而认证器在应用进程，`GET_ACCOUNTS` 自 targetSdk 26 起「不被使用」、真正的门槛是五档可见性（同 UID 恒可见，..."
date: 2026-11-02
permalink: /android-accounts-credentials-passkeys/
---

「证明你是你」在 Android 上不是一条链路，而是三条互不替代的链路：系统要回答「这台设备上有哪些账户」，应用要回答「这一次登录该用哪个已存凭据」，而 Passkey 又要回答「这把私钥属于哪个域名」。三者分别由 `AccountManager`、凭据管理器（Credential Manager）与 WebAuthn/CTAP 生态承担，授权点、数据落点、失败模式全不相同。

把它们混成一句「登录凭据都在系统里」，会同时错过三个真实的攻击面：账户 token 的签名门槛、凭据提供方的系统分发规则、以及 Passkey 的域名绑定校验。本手册沿这三条链路逐层拆开，并在最后给出公告台账的读法、六种成因模式与常见误传纠正。

---

## 0. 使用方式与边界

- **定位**：防御与教育视角。只讨论公开且已修复的问题，不提供任何可直接用于攻击的代码或步骤。
- **来源纪律**：机制表述、API 实名、公告列值均取自官方一手来源并附链接；查不到、只有二手转述或页面已失效的，一律单列在「未核实与失效来源」，**不写进正文**。
- **版本口径**：Android 版本与 API 级别并列写出（如「Android 14（API 34）」）；Jetpack 库单独标版本，因为它与平台版本不同步。
- **术语**：
  - **账户（account）**：由 `name` 与 `type` 两个字符串标识的系统级记录。
  - **认证器（authenticator）**：实现 `AbstractAccountAuthenticator` 的应用组件，负责真正去服务器换 token。
  - **凭据（credential）**：一次登录可用的材料（密码、Passkey、自定义类型）。
  - **凭据提供方（credential provider）**：向系统注册、能存取凭据的应用。
  - **RP / RP ID**：WebAuthn 里的依赖方与其域名标识，Passkey 的信任根。

---

## 1. 三条通道的一句话对照

| 通道 | 回答的问题 | 数据落在哪 | 绑定的东西 | 真正的授权点 | 最常见的误判 |
| --- | --- | --- | --- | --- | --- |
| `AccountManager` | 设备上有哪些账户、能不能取到 token | system_server 内的账户库 + 认证器应用 | `account type` 字符串 | 签名/UID 匹配 + 账户可见性 | 以为 `GET_ACCOUNTS` 是唯一门槛 |
| 凭据管理器 | 这次登录用哪个已存凭据 | 凭据提供方应用 | 用户选中的提供方 | 系统按「已启用 + 支持该类型」分发 | 以为它就是 `AutofillService` |
| Passkey | 这把私钥属于哪个域名 | 认证器（设备或提供方） | **RP ID = 域名** | `Digital Asset Links` 校验 | 以为绑定的是包名或签名 |

一句话概括差异：账户模型绑定的是**类型字符串**，凭据管理器绑定的是**用户选择**，Passkey 绑定的是**域名**。前两者出错通常是越权读取，第三者出错则是「把凭据交给错误的 RP」，性质更接近钓鱼。

---

## 2. 账户模型：`AccountManager`

### 2.1 `Account` 只有两个字段

`android.accounts.Account` 的全部标识就是 `name` 与 `type`。`type` 是认证器自己定义的字符串，官方对 manifest 中 `android:accountType` 的说明是它「唯一标识你的认证器」，并与 `Account#type` 对应。这意味着：**账户的身份不是邮箱、不是用户 ID，而是「认证器声明的类型 + 该类型下的名字」**。两个不同应用若声明同一个 type，语义上是在争抢同一批账户记录。

`AccountManager` 还维护「每个用户一份账户表」。官方在受管配置（managed profiles）文档里写得很直白：`AccountManagerService` maintains a separate list of accounts for each user。多用户设备上，账户不是全局共享的。

### 2.2 数据在 system_server，认证器在应用进程

这是账户模型最容易被误解的一层。`AccountManagerService` 由 `SystemServer` 启动（源码为 `mSystemServiceManager.startService(AccountManagerService.Lifecycle.class);`），因此账户记录与 token 缓存都在 **system_server 进程**里。

而认证器是认证器应用自己声明的 Service：官方要求「写一个在 `onBind()` 中返回 `getIBinder()` 结果的 service」。调用方 uid 会被 `AccountManagerService` 与 `AccountAuthenticators` 同时知道。所以一次取 token 的实际路径是：

1. 应用调用 `AccountManager.getAuthToken()`；
2. 请求进入 system_server 的 `AccountManagerService`；
3. 由它绑定到认证器应用里的 Service，把请求转过去；
4. 认证器访问网络、拿到 token，回传；
5. `AccountManagerService` 把 token 放进缓存（并落库）。

**推论**：账户数据的保密性并不只取决于调用方权限，还取决于「谁能被绑定为认证器」与「谁与认证器同签名」。这也是公告里那批 `AccountManagerService.java` 缺陷几乎都落在**跨进程边界上的检查**、而不是数据库本身的原因。

### 2.3 四个经典权限的现状

这一组权限的现状差异极大，是误传最密集的地方。

| 权限 | AOSP 清单中的状态 | 对应用的实际含义 |
| --- | --- | --- |
| `AUTHENTICATE_ACCOUNTS` | `protectionLevel="normal"`、`permissionFlags="removed"` | 仅对 targetSdk ≤ 22 有意义；API 23 起不再要求 |
| `MANAGE_ACCOUNTS` | 同上 | 同上 |
| `USE_CREDENTIALS` | 同上 | 同上 |
| `GET_ACCOUNTS` | 仍为 `dangerous`，权限组为 `UNDEFINED` | 官方称其为 deprecated；targetSdk ≥ 26 时「不被使用」 |
| `ACCOUNT_MANAGER` | `signature` | 注释明确「不供第三方应用使用」 |

两个关键结论：

- **三个旧权限不是被删掉，而是被标成 removed 且不再强制**。官方文档只说「若你的应用 target 到 API 22 及更早，才需要 `AUTHENTICATE_ACCOUNTS`」。把它们当成现行门槛去审代码，会得出错误结论。
- **`GET_ACCOUNTS` 既没被删除，也不再是可见性的决定因素**。它仍是 dangerous，但 Android 8.0（API 26）的行为变更写的是「`GET_ACCOUNTS` 权限不再足够」。真正的机制换成了下一节的可见性模型。

### 2.4 可见性五档

API 26 起，账户可见性由 `setAccountVisibility` 与一组 `VISIBILITY_*` 常量表达：

| 常量 | 值 | 含义 |
| --- | --- | --- |
| `VISIBILITY_UNDEFINED` | 0 | 未设置，走默认推断 |
| `VISIBILITY_VISIBLE` | 1 | 始终可见 |
| `VISIBILITY_USER_MANAGED_VISIBLE` | 2 | 默认可见，但用户可撤销 |
| `VISIBILITY_NOT_VISIBLE` | 3 | 不可见 |
| `VISIBILITY_USER_MANAGED_NOT_VISIBLE` | 4 | 默认不可见，但用户可授予 |

官方对 `getAccounts` 的解释是：账户之所以可见，是因为「用户授予了访问权」，或者「客户端与管理该账户的 `AbstractAccountAuthenticator` 共享签名」。源码里的判定更细：

- **调用方 uid 与认证器相同** → 直接返回 `VISIBILITY_VISIBLE`；
- **签名相同但 uid 不同** → 走 legacy 可见，实际落在 `USER_MANAGED_VISIBLE`，也就是说**用户可以撤销**。

这就是「同签名即可见」这句流行说法不准确的原因：同 UID 是恒可见，同签名是可被用户关掉的默认可见。审计时若把两者混为一谈，会高估共享签名的暴露面，也会低估同 UID 的暴露面。

### 2.5 token 缓存与 `peekAuthToken` 的签名门槛

Token 相关 API 的语义边界值得逐条记住：

- `peekAuthToken`：从 `AccountManager` 的**缓存**里取 token。官方注明它「面向认证器，而不是直接面向应用」。源码中若调用方与认证器签名不匹配，直接抛 `SecurityException`（信息为 cannot peek the authtokens）。
- `getAuthToken`：命中缓存则直接返回，否则用已存密码去换新 token。
- `invalidateAuthToken`：把 token 从缓存里移除，**不校验、不过期**。也就是说它只影响本机缓存，服务端是否仍接受该 token，取决于服务端自己的失效策略。
- `blockingGetAuthToken`：同步版本，供后台线程使用。

落盘位置方面，源码里的账户库文件名为 `accounts.db` 与 `accounts_ce.db`，token 表为 `authtokens`；`saveAuthTokenToDatabase` → `insertAuthToken` 把值作为普通 `ContentValues` 写入。另有 `TokenCache` 在内存中管理限时缓存，源码注释为「TokenCaches manage time limited authentication tokens in memory.」

**这里必须划一条线**：官方文档没有说 token 在数据库里做了列级加密；源码路径显示的是明文插入，实际保护来自文件级加密（FBE）与 system_server 的进程边界。因此正文**不写「token 加密存储」**，只写「受存储加密与进程边界保护，未核实列级加密」。

### 2.6 选账户界面与 user-managed 可见性

`newChooseAccountIntent` 返回一个「让用户从账户列表中选择」的 Intent；被选中的账户对调用方标记为 `VISIBILITY_USER_MANAGED_VISIBLE`。实现上它使用系统资源 `config_chooseTypeAndAccountActivity` 固定组件，也就是这个界面由系统提供，不是应用自绘。

安全含义有两面：

- 正面：账户选择发生在系统 UI 里，应用拿不到「用户看过哪些账户」的旁路信息；
- 需要谨慎的一面：官方文档没有就「应用能否伪造类似界面骗取授权」给出成文安全声明。所以本手册**不声称**该 Intent 具有防伪造保证，只陈述它的实际行为（系统组件 + user-managed 标记）。

---

## 3. 凭据管理器

### 3.1 平台 API 与 Jetpack 双轨

「Credential Manager 从 Android 14 开始」这句话只对了一半。官方原文是：Credential Manager「指 Android 14 引入的一组 API」——这是**平台侧**（第三方凭据提供方、系统分发）。应用侧接入走的是 Jetpack 的 `androidx.credentials`，其能力下限按凭据类型分档：

| 能力 | 官方给出的最低版本 |
| --- | --- |
| 密码凭据 | Android 4.4（API 19）及以上 |
| Passkey | Android 9（API 28）及以上 |
| 库自身 minSdk | 23（1.6.0-alpha05 起由 21 提到 23） |
| 平台侧凭据提供方（framework 的 `CredentialProviderService`） | 标注 `@RequiresApi(34)` |

所以正确的表述是：**应用可以在很老的设备上用密码凭据；Passkey 需要 API 28；「第三方应用作为凭据提供方」才是 Android 14 起的事。** 三者写成一个版本号，是最常见的版本口径错误。

### 3.2 方法面

`CredentialManager` 接口除了三个主干方法，还有几个容易被漏掉的：

| 方法 | 作用 |
| --- | --- |
| `createCredential` | 创建并保存一条凭据（注册 Passkey、保存密码） |
| `getCredential` | 取一条凭据用于登录 |
| `clearCredentialState` | 清除当前用户的凭据状态（官方原文：「Clears the current user credential state from all credential providers」） |
| `prepareGetCredential` | 预取，用于提前知道有哪些候选，减少一次交互的等待 |
| `signalCredentialState` | 向提供方回传状态信号 |
| `createSettingsPendingIntent` | 跳到凭据设置界面 |

`prepareGetCredential` 的存在说明了一件事：**取凭据是一次可能弹 UI 的跨进程交互**，不是本地读表。任何把它当成「同步读缓存」的代码，在多提供方、需要用户确认的场景下都会表现异常。

### 3.3 请求与响应实名

这组类型名错一个字母就会编译失败，但文档与博客里错写极多。已核实的对应关系如下：

| 方向 | 类型 | 关键字段 |
| --- | --- | --- |
| 取凭据请求 | `GetCredentialRequest` | 构造参数为凭据选项列表与 origin；另有 `Builder.addCredentialOption` |
| 取凭据选项 | `GetPublicKeyCredentialOption` / `GetPasswordOption` / `GetCustomCredentialOption` | 各自承载对应类型的请求 JSON |
| 取凭据响应 | `PublicKeyCredential` / `PasswordCredential` / `CustomCredential` | `PublicKeyCredential.authenticationResponseJson` |
| 创建请求 | `CreateCredentialRequest`（abstract） | 无 Builder，须直接实例化子类 |
| 创建子类 | `CreatePublicKeyCredentialRequest` | 构造参数含 `requestJson` 与 `clientDataHash` |
| 创建响应 | `CreatePublicKeyCredentialResponse` | `registrationResponseJson` |

两个高频错误：

- **不存在 `creationResponseJson`**。创建侧的字段是 `registrationResponseJson`；`PublicKeyCredential` 上只有 `authenticationResponseJson`（用于认证）。
- **`CreateCredentialRequest` 没有 Builder**。它是抽象类，创建 Passkey 要直接构造 `CreatePublicKeyCredentialRequest`，而不是像取凭据那样链式 `Builder`。

注册与认证的 JSON 分别对应 WebAuthn 的 attestation 与 assertion：前者证明「新生成的公钥属于这个 RP」，后者证明「持有私钥的一方同意这次挑战」。两者混用会得到「注册成功但登录永远失败」这种看似玄学的结果。

### 3.4 凭据提供方的三要素

第三方应用要成为凭据提供方，manifest 里三样东西缺一不可（以下为官方给出的形态）：

```xml
<service
    android:name=".MyCredentialProviderService"
    android:exported="true"
    android:permission="android.permission.BIND_CREDENTIAL_PROVIDER_SERVICE">
    <intent-filter>
        <action android:name="android.service.credentials.CredentialProviderService" />
    </intent-filter>
    <meta-data
        android:name="android.credentials.provider"
        android:resource="@xml/provider" />
</service>
```

- **绑定权限** `android.permission.BIND_CREDENTIAL_PROVIDER_SERVICE`：官方原文是「只有 Android 系统能绑定这个 service」。这是 `signature` 级保护，第三方无法绑定。
- **action** `android.service.credentials.CredentialProviderService`：系统据此枚举候选提供方。
- **meta-data** `android.credentials.provider` 指向的 XML 里声明支持的能力类型，实名为 `android.credentials.TYPE_PASSWORD_CREDENTIAL` 与 `androidx.credentials.TYPE_PUBLIC_KEY_CREDENTIAL`。

请注意最后一条的类型名分属两个命名空间：密码类型在 `android.credentials` 下，公钥凭据类型在 `androidx.credentials` 下。写成同一个包名，会在部分设备上被静默忽略。

### 3.5 要覆写的是带 `Request` 的方法

`CredentialProviderService` 的抽象方法实名是：

- `onBeginGetCredentialRequest`
- `onBeginCreateCredentialRequest`
- `onClearCredentialStateRequest`

基类里另有一组**去掉 `Request`** 的同名方法，是 final 的桥接实现。文档的原话是「override the `onBeginCreateCredentialRequest()` method」。只覆写不带 `Request` 的版本会编译不过；反过来在文章里把 `onBeginGetCredential` 当成要覆写的方法，也是常见错误。

数据流方面：`BeginGetCredentialRequest` 承载一组 `BeginGetCredentialOption`；创建侧由 `BeginCreateCredentialRequest` 提出，提供方用 `BeginCreateCredentialResponse` 加 `CreateEntry` 应答。这些 `Begin*` 类型是**系统与提供方之间**的协议对象，应用开发者通常不直接接触，但它们决定了「提供方能否看到足够信息去匹配凭据」。

### 3.6 `allowedProviders` 的权限其实是 `normal`

`CREDENTIAL_MANAGER_SET_ALLOWED_PROVIDERS` 常被写成 signature 级权限，实际在 AOSP 清单里是 `android:protectionLevel="normal"`。两个配套事实：

- 未持有该权限时设置 `allowedProviders` 会抛 `SecurityException`；
- API < 34 上不生效。

它控制的是「本次取凭据允许考虑哪些提供方」，与「提供方本身是否可信」无关。把它当成强隔离手段，是权限模型上的误读。

另外，用户必须在设置里启用某个提供方，系统才会向它分发请求。官方原文是「The user has enabled this service as a credential provider from the settings」。这与账户模型里的 user-managed 可见性是同一种设计取向：**用户是最终授权者，应用不能自行扩大候选范围**。

---

## 4. Passkey 的绑定链

### 4.1 三个名字：RP ID、包名、证书指纹

Passkey 的安全前提是「私钥只交给它所属域名的挑战」。链路上有三个名字，容易被混用：

| 名字 | 出现在哪 | 作用 |
| --- | --- | --- |
| **RP ID** | WebAuthn 请求 | 决定凭据属于哪个域名，抗钓鱼的根 |
| **包名** | `assetlinks.json` 的 `target.package_name` | 声明哪个 Android 应用代表该域名 |
| **证书指纹** | `assetlinks.json` 的 `sha256_cert_fingerprints` | 声明哪个签名证书才算「那个应用」 |

官方给出的指纹格式是 32 组、大写、冒号分隔（示例形如 `14:6D:E9:83:…:44:E5`）。证书轮换或改签名后指纹变化，`assetlinks.json` 必须同步更新，否则应用无法代表该域名使用 Passkey——这是「换了签名就登录不了」的根本原因，而不是凭据损坏。

### 4.2 `assetlinks.json` 的两个 relation

文件必须放在 `https://<domain>/.well-known/assetlinks.json`。里面最常被混淆的是两个 relation：

| relation | 官方语义 | 用于 |
| --- | --- | --- |
| `delegate_permission/common.get_login_creds` | 「允许共享登录凭据」 | **Passkey / 密码共享** |
| `delegate_permission/common.handle_all_urls` | 「允许处理该域名的所有 URL」 | **App Links** |

```json
[{
  "relation": ["delegate_permission/common.get_login_creds"],
  "target": {
    "namespace": "android_app",
    "package_name": "com.example.app",
    "sha256_cert_fingerprints": ["14:6D:E9:83:…:44:E5"]
  }
}]
```

把 `handle_all_urls` 当成 Passkey 绑定关系是明确的错误：前者让应用接管链接跳转，后者才让应用代表域名存取凭据。两者都写、或只写错一个，后果不同——只写 `handle_all_urls` 时链接能打开，但 Passkey 依然不可用。

### 4.3 WebAuthn 参数的真实取值

`authenticatorSelection` 与 `userVerification` 的取值在不同版本里有过调整，现行（WebAuthn Level 3）口径：

| 参数 | 取值 | 说明 |
| --- | --- | --- |
| `residentKey` | `discouraged` / `preferred` / `required` | 是否要求「可发现凭据」（常说的 resident key） |
| `requireResidentKey` | 布尔，默认 `false` | 为向后兼容保留 |
| `userVerification` | `required` / `preferred` / `discouraged`，默认 `preferred` | 是否要求用户验证（生物识别/锁屏凭据） |
| `signCount` | 整数 | 即 signature counter |

`signCount` 的官方语义常被写错：规范说它的用途是「帮助 RP 检测被克隆的认证器」，而不是「防重放」。防重放靠的是每次挑战的新鲜性；计数器只是**事后**发现克隆的辅助信号，且许多平台认证器并不递增它。

### 4.4 抗钓鱼的准确含义

官方措辞是「phishing-resistant sign in using multi-device credentials」，Android 侧写作「not subject to phishing attacks」。它的机制不是「界面不可伪造」，而是：

1. 浏览器/系统把 **origin** 传给认证器；
2. 认证器只在 origin 与 RP ID 匹配时才签名；
3. 所以即使攻击者做出完全相同的登录页，只要域名不同，签名就不会产生。

因此「抗钓鱼」保护的是**凭据不被交出去**，不保护「用户不被骗着在真站点上完成一次他自己不理解的登录」。后者属于社会工程，不在 WebAuthn 的保证范围内。这一点在评估 Passkey 部署效果时必须写清楚。

关于私钥存放，官方页面只说到「the private key is securely stored on the user's device」。**本手册不写「私钥在 TEE/安全芯片内」**，因为这属于未核实的实现细节，见「未核实与失效来源」B。

### 4.5 restore credential 与卸载语义

`RestoreCredential` 属于 Jetpack（1.5.0 起），官方给出的下限是 Android 9（API 28）且 GMS core 版本 24220000 以上。它解决的是「换机后 Passkey 还在不在」的问题：把可恢复的凭据密钥与账户绑定，换机时通过备份通道恢复。

卸载语义有一条成文规则：「Uninstalling an app is interpreted as an intent to delete the corresponding restore key」，并且 restore key 只在「用户卸载应用或清除其数据」时移除。至于**普通 Passkey 与密码在卸载重装、用户切换后的行为**，官方没有成文说明，本手册按未核实处理。

---

## 5. 公告台账与读法

### 5.1 先认清表格形状

月度 Android Security Bulletin 的 Framework 表列头是：

```
CVE | References | Type | Severity | Updated AOSP versions
```

**没有 `Subcomponent` 列**。带 `Subcomponent` 的是 Pixel Update Bulletin，两张表的列不能混用。想从月度表直接读出「这条属于 AccountManager」是读不到的——`Component` 只写到 `Framework` 这一级。

### 5.2 三步读法

1. **按 Component 筛**：账户类缺陷全部落在 `Framework`，与 Media Framework、System、Kernel 等并列；
2. **按 References 回查 CVE 描述**：描述里会写明具体文件与函数，例如 `validateAccountsInternal of AccountManagerService.java`；
3. **按 Updated AOSP versions 判断影响面**：这一列是修复覆盖的版本，不是「漏洞引入版本」。

第 2 步是唯一能把「Framework」细化到组件的官方途径。本手册的台账就是这样做出来的：每一条都回到 CVE 描述核对函数名，而不是凭公告行猜测。

### 5.3 已核对的账户类条目

下表 12 条均可在 CVE 描述中看到 `AccountManagerService.java`，Type/Severity 取自对应月度公告：

| 公告月份 | CVE | 描述中的落点 | Type | Severity | 修复版本（Updated AOSP versions） |
| --- | --- | --- | --- | --- | --- |
| 2021-12 | CVE-2021-0704 | `createNoCredentialsPermissionNotification` 及相关函数 | ID | High | 9, 10, 11 |
| 2022-01 | CVE-2020-0338 | `checkKeyIntent` | ID | High | 9, 10 |
| 2023-04 | CVE-2023-21098 | 多个函数（向 System Settings 加载任意代码，confused deputy） | EoP | High | 11, 12, 12L, 13 |
| 2023-09 | CVE-2023-35669 | `checkKeyIntentParceledCorrectly`（不安全反序列化，可控制其他运行中的 Activity） | EoP | High | 11, 12, 12L, 13 |
| 2023-12 | CVE-2023-45777 | `checkKeyIntentParceledCorrectly`（Parcel mismatch 导致以系统权限启动任意 Activity） | EoP | High | 13, 14 |
| 2024-03 | CVE-2024-0048 | `Session`（null 响应处理不当，保留前台 Service 特权） | EoP | High | 12, 12L, 13, 14 |
| 2024-06 | CVE-2024-31316 | `onResult`（parcel mismatch 导致任意后台 Activity 启动） | EoP | High | 12, 12L, 13, 14 |
| 2024-10 | CVE-2024-40676 | `checkKeyIntent`（绕过 intent 检查安装未知应用） | EoP | High | 12, 12L, 13, 14, 15 |
| 2024-11 | CVE-2024-43086 | `validateAccountsInternal`（confused deputy 泄漏凭据给第三方） | ID | High | 12, 12L, 13, 14, 15 |
| 2025-01 | CVE-2024-49724 | 多个函数（竞态，越权启动受保护 Activity） | EoP | High | 12, 12L, 13, 14, 15 |
| 2025-01 | CVE-2024-49744 | 反序列化绕过（Parcel mismatch） | EoP | High | 12, 12L, 13, 14, 15 |
| 2025-03 | CVE-2025-0086 | `onResult`（缺权限检查，可覆盖 auth token） | ID | High | 12, 12L, 13, 14, 15 |

另有两条同样落在 `AccountManagerService.java`、但根因更偏「资源与身份判断」：CVE-2025-48542（多个函数，资源耗尽导致永久性 DoS，Type 为 DoS，修复 13–16）与 CVE-2025-48545（`isSystemUid` 的 confused deputy，Type 为 ID，修复 13–16）。

### 5.4 这张表能说明什么、不能说明什么

- **能说明**：账户服务的缺陷几乎全部是**跨进程边界上的判断错误**（confused deputy、parcel 一致性、缺权限检查、竞态），而不是密码学问题。修复手段也集中在「补检查、钉住 Parcel、加锁」。
- **不能说明**：不能据此推断「账户系统不安全」。这些是历年累积、已修复的个例；`AccountManagerService` 同时承载着签名校验、可见性、token 缓存等强约束，缺陷恰恰说明这些约束是承重墙。
- **不能说明**：Type 列不是根因分类。同一条可能既被标 `EoP` 又是「信息泄露」，取决于公告作者选择的主后果。

---

## 6. 成因模式：六种

把上面 12 条按根因归并，只有六类，且每一类都能在别的系统服务里复现：

1. **Confused deputy（代理越权）**：服务持有特权，却没校验「请求方是否被允许让服务替它做这件事」。CVE-2024-43086、CVE-2024-40676、CVE-2025-48545 都属此类。
2. **缺权限检查**：某条分支忘记 `checkCallingPermission` 或 uid 校验。CVE-2025-0086（覆盖 auth token）。
3. **Parcel / 反序列化一致性**：写入与读取的字段顺序或类型不一致，导致「检查时看到 A，使用时拿到 B」。CVE-2024-31316、CVE-2024-49744。
4. **检查与使用之间的竞态（TOCTOU）**：权限在检查后被改变，或对象在检查后被替换。CVE-2024-49724。
5. **隐式代码/组件加载**：把 Intent 或类名交给特权进程去实例化，等于让调用方间接执行代码。CVE-2023-21098。
6. **资源耗尽**：可控输入导致对象或通知无限增长。CVE-2025-48542。

这六类的共同点：**它们都不需要破解加密，只需要找到「服务替谁办事」这一层缺失的判断**。这也是为什么账户类修复往往只有几十行 diff。

---

## 7. 观测点与自查

### 7.1 应用侧（静态）

| 检查项 | 观察什么 | 判读 |
| --- | --- | --- |
| 认证器声明 | manifest 中的 `android:accountType` 与 `AbstractAccountAuthenticator` 子类 | type 是否与其他应用冲突；是否暴露了不必要的 `onBind` |
| 凭据提供方声明 | 是否同时具备 `BIND_CREDENTIAL_PROVIDER_SERVICE` 权限、系统 action、`android.credentials.provider` meta-data | 缺任意一项都不会被系统枚举 |
| 能力类型 | XML 中的 `TYPE_PASSWORD_CREDENTIAL` / `TYPE_PUBLIC_KEY_CREDENTIAL` | 包名是否写混（`android.` vs `androidx.`） |
| Passkey 接入 | 是否使用 `GetPublicKeyCredentialOption` 且读取 `authenticationResponseJson` | 读了 `creationResponseJson` 说明抄了错误示例 |
| 创建路径 | 是否直接构造 `CreatePublicKeyCredentialRequest` | 试图 `new CreateCredentialRequest.Builder()` 是错的 |

### 7.2 域名侧（部署）

| 检查项 | 做法 | 判读 |
| --- | --- | --- |
| `assetlinks.json` 可达 | `curl https://<domain>/.well-known/assetlinks.json` | 必须 200 且 `Content-Type` 为 JSON，不能是重定向到登录页 |
| relation 正确 | 逐条看 `relation` 数组 | 凭据共享必须是 `get_login_creds` |
| 指纹匹配 | 与当前签名证书的 SHA-256 比对 | 指纹是 32 组大写冒号分隔；换签名后必须同步 |
| 多环境 | 测试/生产域名各自的文件 | 只在生产放文件，测试环境会静默失败 |

### 7.3 设备侧（运行时）

- `dumpsys -l` 若列出 `account` 服务，可用 `dumpsys account` 观察账户表与可见性相关字段（不同厂商可能裁剪输出，**以实际输出为准**）。
- 凭据提供方的启用状态属于用户设置，应用侧无法枚举「用户装了哪些提供方」——这正是设计意图。
- `adb shell settings` 下的账户相关键名随版本变化，本手册不给出具体键名（未核实跨版本稳定性）。

### 7.4 企业侧

- MDM 若依赖账户可见性做「工作资料隔离」，应验证 `USER_MANAGED_VISIBLE` 是否被用户撤销，而不是假设同签名即共享。
- 禁止在受管设备上把未知应用登记为凭据提供方；该动作需要用户在设置中主动启用，可作为审计事件。

---

## 8. 常见误传纠正

| # | 误传 | 实际 |
| --- | --- | --- |
| 1 | 「`GET_ACCOUNTS` 已被删除」 | 仍存在且仍为 `dangerous`；targetSdk ≥ 26 时官方称其「不被使用」，改由可见性模型决定 |
| 2 | 「`AUTHENTICATE_ACCOUNTS` 是现行必需权限」 | 已标 `removed`，API 23 起不再要求，只对 targetSdk ≤ 22 有意义 |
| 3 | 「同签名就能看到对方全部账户」 | 同 UID 恒为 `VISIBLE`；同签名只是 `USER_MANAGED_VISIBLE`，用户可撤销 |
| 4 | 「`peekAuthToken` 任何应用都能调」 | 需与认证器签名匹配，否则抛 `SecurityException` |
| 5 | 「`invalidateAuthToken` 会让 token 立刻失效」 | 只从本机缓存移除，不校验不过期；服务端是否接受另说 |
| 6 | 「账户 token 是加密存的」 | 无官方列级加密说明；落盘为 `accounts_ce.db` 的 `authtokens` 表，保护来自 FBE 与进程边界 |
| 7 | 「Credential Manager 要 Android 14 才能用」 | 平台侧提供方 API 是 14；密码凭据 API 19+、Passkey API 28+、库 minSdk 23 |
| 8 | 「创建响应字段是 `creationResponseJson`」 | 创建侧是 `CreatePublicKeyCredentialResponse.registrationResponseJson`；`PublicKeyCredential` 只有 `authenticationResponseJson` |
| 9 | 「凭据提供方要覆写 `onBeginGetCredential`」 | 抽象方法是带 `Request` 的 `onBeginGetCredentialRequest` 等三个 |
| 10 | 「`CREDENTIAL_MANAGER_SET_ALLOWED_PROVIDERS` 是 signature 权限」 | AOSP 清单为 `normal`；未持有时设置会抛 `SecurityException` |
| 11 | 「Passkey 绑定用 `handle_all_urls`」 | 凭据共享用 `common.get_login_creds`；`handle_all_urls` 是 App Links |
| 12 | 「Passkey 私钥在 TEE/安全芯片内」 | 官方只说 securely stored on the user's device，未写实现位置 |
| 13 | 「`signCount` 用来防重放」 | 规范说用于帮助 RP 检测被克隆的认证器 |
| 14 | 「CVE-2021-0704 是 TEE/QSEE 问题」 | 归属 `AccountManagerService.java` 的 `createNoCredentialsPermissionNotification` |
| 15 | 「CVE-2024-0039 是账户漏洞」 | 它是蓝牙 `att_protocol.cc` 的 RCE；同月账户条目是 CVE-2024-0048 |
| 16 | 「月度公告有 `Subcomponent` 列」 | 月度表没有；带该列的是 Pixel Update Bulletin |
| 17 | 「CVE-2022-20444 是账户案例」 | CVE 记录为 `REJECTED`，CNA 撤回，不能当案例 |

---

## 9. 与其他手册的分工

| 想弄清的问题 | 去哪一篇 |
| --- | --- |
| 自动填充服务怎么读界面、和凭据提供方什么关系 | [无障碍、输入法与自动填充通道手册](/android-accessibility-ime-autofill/) |
| 组件导出、权限属性、PendingIntent 的通用规则 | [框架 IPC 与组件导出攻击面手册](/android-framework-ipc-export/) |
| 公告的编号体系、SPL、逐字段读法 | [漏洞编号与公告体系手册](/android-cve-and-security-bulletins/) |
| 应用签名、v3 轮换、签名证书与指纹 | [APK 签名与安装校验手册](/android-apk-signing-install/) |
| 密钥存储、KeyMint 安全等级与密钥证明 | [TEE、密钥管理与设备认证手册](/android-tee-key-attestation/) |
| 检查与使用之间的窗口、竞态的一般形态 | [竞态条件与 TOCTOU 手册](/android-race-conditions-toctou/) |
| SELinux 与 seccomp 如何限制 system_server 之外的服务 | [SELinux 策略与 seccomp 沙箱手册](/android-selinux-seccomp-sandbox/) |

---

## 参考

**官方文档（本手册逐字引用过）**

- `Account` 参考 — <https://developer.android.com/reference/android/accounts/Account>
- `AccountManager` 参考（权限、可见性、token API）— <https://developer.android.com/reference/android/accounts/AccountManager>
- `AbstractAccountAuthenticator` 参考（认证器 Service 形态）— <https://developer.android.com/reference/android/accounts/AbstractAccountAuthenticator>
- `AccountManagerFuture` — <https://developer.android.com/reference/android/accounts/AccountManagerFuture>
- Android 8.0 行为变更（`GET_ACCOUNTS` 不再足够）— <https://developer.android.com/about/versions/oreo/android-8.0-changes>
- 受管配置与「每用户一份账户表」— <https://source.android.com/docs/devices/admin/managed-profiles>
- Credential Manager 提供方接入 — <https://developer.android.com/identity/sign-in/credential-provider>
- 密码凭据接入（API 19 起）— <https://developer.android.com/identity/passwords>
- Passkey 接入（API 28 起）— <https://developer.android.com/identity/passkeys>
- `CredentialManager` 参考 — <https://developer.android.com/reference/kotlin/androidx/credentials/CredentialManager>
- `PublicKeyCredential` 参考 — <https://developer.android.com/reference/kotlin/androidx/credentials/PublicKeyCredential>
- `GetCredentialRequest` 参考 — <https://developer.android.com/reference/kotlin/androidx/credentials/GetCredentialRequest>
- `CreatePublicKeyCredentialRequest` 参考 — <https://developer.android.com/reference/kotlin/androidx/credentials/CreatePublicKeyCredentialRequest>
- `CredentialProviderService`（androidx，`@RequiresApi(34)`）— <https://developer.android.com/reference/androidx/credentials/provider/CredentialProviderService>
- `CredentialOption`（`allowedProviders` 与权限异常）— <https://developer.android.com/reference/kotlin/androidx/credentials/CredentialOption>
- androidx.credentials 版本说明（minSdk 23）— <https://developer.android.com/jetpack/androidx/releases/credentials>
- Smart Lock 迁移说明 — <https://developer.android.com/identity/sign-in/smart-lock-migration>
- restore credential 实现说明 — <https://developer.android.com/identity/sign-in/restore-credentials-implementation>
- Passkey 前置条件与 `assetlinks.json` — <https://developer.android.com/identity/credential-manager/prerequisites>
- App Links 与 `assetlinks.json` 配置 — <https://developer.android.com/training/app-links/configure-assetlinks>
- 凭据管理器与自动填充的衔接 — <https://developer.android.com/identity/autofill/credential-manager-autofill>

**源码（AOSP，用于核对权限级别与落盘行为）**

- `AndroidManifest.xml`（权限定义）— <https://android.googlesource.com/platform/frameworks/base/+/refs/heads/main/core/res/AndroidManifest.xml>
- `SystemServer.java`（`AccountManagerService` 启动）— <https://android.googlesource.com/platform/frameworks/base/+/refs/heads/main/services/java/com/android/server/SystemServer.java>
- `AccountManagerService.java` — <https://android.googlesource.com/platform/frameworks/base/+/refs/heads/main/services/core/java/com/android/server/accounts/AccountManagerService.java>
- `AccountsDb.java` — <https://android.googlesource.com/platform/frameworks/base/+/refs/heads/main/services/core/java/com/android/server/accounts/AccountsDb.java>
- `TokenCache.java` — <https://android.googlesource.com/platform/frameworks/base/+/refs/heads/main/services/core/java/com/android/server/accounts/TokenCache.java>

**规范**

- WebAuthn Level 3：`authenticatorSelection` — <https://www.w3.org/TR/webauthn-3/#dictionary-authenticatorSelection>
- WebAuthn Level 3：`userVerification` 取值 — <https://www.w3.org/TR/webauthn-3/#enum-userVerificationRequirement>
- WebAuthn Level 3：签名计数器 — <https://www.w3.org/TR/webauthn-3/#signature-counter>
- WebAuthn Level 3：多设备凭据用例 — <https://www.w3.org/TR/webauthn-3/#sctn-usecase-consumer-mdc>

**公告与 CVE**

- Android Security Bulletin 索引 — <https://source.android.com/docs/security/bulletin>
- Linux/Google CNA 记录：CVE-2021-0704 — <https://cveawg.mitre.org/api/cve/CVE-2021-0704>
- CVE-2024-43086 — <https://cveawg.mitre.org/api/cve/CVE-2024-43086>
- CVE-2025-0086 — <https://cveawg.mitre.org/api/cve/CVE-2025-0086>
- CVE-2024-40676 — <https://cveawg.mitre.org/api/cve/CVE-2024-40676>
- CVE-2024-49724 — <https://cveawg.mitre.org/api/cve/CVE-2024-49724>
- CVE-2025-48545 — <https://cveawg.mitre.org/api/cve/CVE-2025-48545>

**相邻手册**：见 §9。

---

## 未核实与失效来源

**A. 「Credential Manager 必须依赖 Google Play services」**
推测：所有设备都要装 GMS 才能用。未核实原因：官方页面没有这句；只见可选依赖说明——`credentials-play-services-auth` 用于「运行 API ≤ 33 的设备」。**正文因此只写「按能力分档的最低版本」，不写「必须 GMS」。**

**B. Passkey 私钥位于 TEE / 安全芯片**
推测：既然叫「安全凭据」，私钥应在硬件里。未核实原因：Android 与 FIDO 页面均未出现 TEE、secure element 措辞；最接近的只是「the private key is securely stored on the user's device」。**正文只引用这半句，不推断实现位置。**

**C. 「凭据提供方不是 AutofillService」**
推测：两者是互斥的。未核实原因：没有这句明文；官方只说「The credential service is combined with any autofill service that's defined in the same package」，另有独立的自动填充二级 UI 通道（Android 15 Beta 2 + 1.5.0-alpha01）。**正文只写「同包合并、另有通道」，不写「互斥」。**

**D. Passkey / 密码在卸载重装、用户切换后的行为**
推测：卸载即删除、切换用户即不可见。未核实原因：仅 restore key 有成文规则（卸载视为删除意图）；普通凭据无说明。**正文只写 restore key 那一条。**

**E. `GoogleIdOption` 与 `androidx.credentials` 的隶属关系**
推测：属于 Credential Manager 的「Sign in with Google」选项。未核实原因：未取到含该实名的官方页面，故**正文不出现该类型名**。

**F. 账户库是否存在列级加密**
推测：token 存库所以应加密。未核实原因：官方文档无此表述；源码路径显示 `insertAuthToken` 以普通 `ContentValues` 写入 `accounts_ce.db`。存储层 FBE 是另一层保护，不能等同于列级加密。**正文只写「未核实列级加密」。**

**G. `newChooseAccountIntent` 的防伪造保证**
推测：系统组件即不可伪造。未核实原因：官方文档只描述行为与组件资源名，没有任何安全声明。**正文只陈述行为，不作保证。**

**H. 月度公告与 Pixel 公告的列差异是否始终成立**
已核实：月度 Framework 表列头为 `CVE / References / Type / Severity / Updated AOSP versions`；带 `Subcomponent` 的是 Pixel Update Bulletin。未核实的部分是「是否存在个别月份改版」——本手册只对台账中已逐条打开的 12 期负责。

**I. `dumpsys account` 的输出字段**
推测：可稳定读出账户表与可见性。未核实原因：未在实际设备上验证，且厂商可能裁剪。**正文已加「以实际输出为准」。**

**J. 账户类 CVE 的完整清单**
本表是「能在一手 CVE 描述中看到 `AccountManagerService.java`」的条目，不保证穷尽；按 Component 搜 `Framework` 无法直接筛出账户条目，可能存在描述未写文件名的遗漏。

---

## 联系方式

咨询 Android 系统详情请联系 https://t.me/one00190

---

*本手册仅供安全学习与防御研究使用；所有机制表述、公告列值与 CVE 描述均取自公开一手来源并附链接，未能核实的部分已单列于「未核实与失效来源」而非写入正文。*
