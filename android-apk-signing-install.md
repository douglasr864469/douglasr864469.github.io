---
layout: page
title: "Android APK 签名与安装校验攻击面手册"
description: "v1–v4 方案官方口径与保护范围、v3 proof-of-rotation 属性与 `apksigner rotate` 参数、Android 13 起才认 lineage、`PackageManager` 安装标志与 46 个 `INSTALL_FAILED_*` 实名、`GET_SIGNATURE..."
date: 2026-10-28
permalink: /android-apk-signing-install/
---

> 按「签名方案 → 密钥轮换 → 安装与校验路径 → 来源约束 → 篡改检测信号」组织，全部取自公开一手来源（source.android.com、developer.android.com、AOSP 源码、NVD、CISA KEV），仅供防御与教育；「未核实／查无出处」的名称只出现在 §9。

---

## 0. 使用方式与边界

约定：**实名优先**，类名/常量按源码原文并标注分支（`main`／`android13-release`／`android14-release`）；**公告链接**统一 `https://source.android.com/docs/security/bulletin/YYYY-MM-01`（`/YYYY/` 与 `-05` 页实测 404，§9）；**级别双轨**，公告与 NVD 不一致时两个都写、以公告表为准，KEV 一律「未收录」并附边界；**不含攻击细节**。

---

## 1. 签名方案 v1–v4：官方口径与保护范围

### 1.1 清单与 v1 攻击面

概览页只列 v1（"based on JAR signing"）、v2（"introduced in Android 7.0"）、v3（"introduced in Android 9"）；v4 单列于 `.../apksigning/v4`，另有 `v3-1`、`v3-2` 专页；**v1 无专页**（`.../apksigning/v1` 404），内容在概览页 "JAR signing" 小节。v1 原文："v1 signatures don't protect some parts of the APK, such as ZIP metadata … The APK verifier needs to process lots of untrusted (not yet verified) data structures … This offers a sizeable attack surface." 即 v1 不覆盖 ZIP 元数据、验证器须先解析未受信数据结构；**「仅 v1 签名」本身就是应告警状态**。

### 1.2 v2 / v3 / v4 的保护范围

- **v2**：whole-file signature scheme（Android 7.0），签名存放在 "an APK signing block"，保护 ZIP 的第 1、3、4 段与该 block；v2+ 把 APK 当整块 blob 校验："Any modification to the APK, including ZIP metadata modifications, invalidates the APK signature."
- **v3**：块在 APK Signing Block 内、ID `0x1b93ad61`，格式 "follows that of v2"，增量是 signer 序列与 proof-of-rotation（§2）。
- **v4**：Android 11 流式安装，基于覆盖 APK 全部字节、与 fs-verity 哈希树一致的 Merkle 树；签名存于旁路文件 `<apk name>.apk.idsig`；"A v4 signature requires a complementary v2 or v3 signature." 故 v4 不可独立成立。

### 1.3 v1/v2+ 共存与成文的降级保护规则

共存口径："sign apps with all schemes, first with v1, then v2, and then v3."（旧平台忽略 v2+，装到 Android 6.0 及以下仍须并存 v1）。降级保护原句：**"APKs with stripped v2+ signature are rejected… because their v1 signature specifies that the APK was v2-signed"**（句尾 "…which makes Android 7.0 and newer refuse to verify APKs using their v1 signatures."）——v1 签名内记录了「本 APK 曾被 v2 签名」，故 v2/v3 块被剥离后 Android 7.0+ 直接拒绝而非回退「仅按 v1 验证」；「v1 声称已 v2 签名但 signing block 缺失」是明确告警组合。

### 1.4 `apksigner` 实名子命令

`sign`、`verify`、`rotate`；关键选项 `--lineage`、`--rotation-min-sdk-version`、`--v4-signing-enabled <true|false|only>`、`--old-signer`/`--new-signer`、`--print-certs`、`-v`、`-Werr`。底层库在 AOSP `platform/tools/apksig`。

---

## 2. 密钥轮换：proof-of-rotation 的真实结构与版本边界

### 2.1 记录结构实名：v3 块内 ID `0x3ba06f8c`

v3 专页原文 "ID - 0x3ba06f8c value - Proof-of-rotation struct"：轮换记录 stored inside the v3 block，由 additional attribute ID `0x3ba06f8c` 承载，是 "a singly-linked list, with each node containing a signing certificate used to sign previous versions of the app." 须纠正：`SigningDetails` **不是**记录结构，而是 framework 侧解析结果类（`core/java/android/content/pm/SigningDetails.java`）。校验规则可引用："verify that the struct is valid and this signer is the last certificate in the list"（当前签名者须为链表末端，可作自查判据）。

### 2.2 命令实名与版本边界

真实命令是 `apksigner rotate --in … --out … --old-signer … --new-signer …` 生成 lineage，再以 `apksigner sign --lineage …` 写入 APK；**`apksigner rotate-key` 不存在**（检索 0 命中，§9）。`--rotation-min-sdk-version` 指「轮换后的签名密钥产生该 APK 签名所用的最低 API level」，轮换密钥由 Android 13（API level 33）及以上支持、默认使用 **v3.1** 签名块。成文边界（v3 专页原句）："APK key rotation is not recommended for Android 12 (API level 31) and earlier. In Android 13 (API level 33) and higher, checkSignatures recognizes the proof-of-rotation and returns success across the lineage." 结论两层：**记录自 Android 9 起即可携带**，但 **`checkSignatures` 识别整条 lineage 的边界是 Android 13（API 33）及以后**。

### 2.3 轮换与身份绑定

安装时 Package Manager 校验 APK 内证书：公钥与设备上其他 APK 一致才可声明 shared UID，`signature` 级权限同样限定同钥，故轮换属高影响动作（对照 §5 的 CVE-2023-20927）。

---

## 3. 安装与校验路径：AOSP 源码实名

### 3.1 客户端与服务端的真实类

路径相对 `platform/frameworks/base/+/refs/heads/<branch>/`。客户端 `core/java/android/content/pm/PackageInstaller.java`：内部类 `public class Session`（构造 `Session(IPackageInstallerSession)`）、`commit(@NonNull IntentSender statusReceiver)`；Binder 侧 `IPackageInstaller`／`IPackageInstallerSession`。服务端 `main` 的 `services/core/java/com/android/server/pm/`：`PackageInstallerService.java`、`PackageInstallerSession.java`、`PackageSessionVerifier.java`、`VerificationUtils.java`；`android14-release` 上为 `VerifyingSession.java`、`VerificationInfo.java`（同一职责在不同分支类名不同，**必须按分支引用**）。

### 3.2 安装标志：`INSTALL_*` 共 25 个

`core/java/android/content/pm/PackageManager.java`（`main`）逐行核实共 **25 个**。带真实值的代表条目：`INSTALL_REPLACE_EXISTING = 0x00000002`、`INSTALL_ALLOW_TEST = 0x00000004`、`INSTALL_INTERNAL = 0x00000010`、`INSTALL_REQUEST_DOWNGRADE = 0x00000080`、`INSTALL_GRANT_ALL_REQUESTED_PERMISSIONS = 0x00000100`、`INSTALL_DISABLE_VERIFICATION = 0x00080000`、`INSTALL_BYPASS_LOW_TARGET_SDK_BLOCK = 0x01000000`。shell 映射（`PackageManagerShellCommand.java`）：`-t`→`INSTALL_ALLOW_TEST`、`-l`→`INSTALL_INTERNAL`、`-g`→`INSTALL_GRANT_ALL_REQUESTED_PERMISSIONS`、`-r`→`INSTALL_REPLACE_EXISTING`；安装入口实名 `pm install [-r] [-t] [-i PACKAGE] PATH`、`pm install-write [-S BYTES] SESSION_ID SPLIT_NAME [PATH|-]`、`adb install -t`。真实常量是 `INSTALL_INTERNAL`（`0x10`）与 `INSTALL_ALLOW_TEST`（`0x04`），**不存在 `INTERNAL_TEST` 安装标志**（检索 0 命中，§9）。

### 3.3 失败错误码：`INSTALL_FAILED_*` 共 46 个

同一文件核实为 **46 个**。与签名/完整性直接相关：`BAD_SIGNATURE`、`MISSING_SPLIT`、`MULTIPACKAGE_INCONSISTENCY`、`SHARED_LIBRARY_BAD_CERTIFICATE_DIGEST`、`SHARED_USER_INCOMPATIBLE`、`UID_CHANGED`、`VERIFICATION_FAILURE`（`-22`）、`VERIFICATION_TIMEOUT`（`-21`）、`SANDBOX_VERSION_DOWNGRADE`、`TEST_ONLY`。

### 3.4 `PackageParser` 与解析层演进：不是一次「改名」

`main` 上 `core/java/android/content/pm/PackageParser.java` **文件仍在**但已 `@Deprecated @hide`，注释要求改用 `ParsingPackageUtils`。Android 13+ 服务端承担解析的真实类名即 `ParsingPackageUtils`，实测在 `android13-release/services/core/java/com/android/server/pm/pkg/parsing/ParsingPackageUtils.java`；**该路径在 `main` 上 NOT_FOUND**，`main` 的解析工具在 `core/java/android/content/pm/parsing/`。两条路径长期共存：§5 中 CVE-2023-21090 落在前者、CVE-2025-32312 落在后者。

### 3.5 split APK 的绑定规则与校验边界

`PackageParser.java` 类注释成文：所有 APK 必须具有完全相同的包名、versionCode 与签名证书，split 名必须唯一，一次安装必须只含一个 base APK；对应失败码 `INSTALL_FAILED_MISSING_SPLIT`、`INSTALL_FAILED_MULTIPACKAGE_INCONSISTENCY`。边界：manifest 属性 `android:requiredSplitTypes`／`android:splitTypes` **未核实到实名**（§9）；split 方向的公开 AOSP CVE **未检索到条目**（§9）。

---

## 4. 来源约束：权限、未知来源与验证器

### 4.1 权限与用户限制常量的真实归属

`core/res/AndroidManifest.xml`（`main`）实名声明 `android.permission.INSTALL_PACKAGES`（signature 级，供系统组件/特权安装器直接发起安装）与 `android.permission.REQUEST_INSTALL_PACKAGES`（Android 8.0／API 26 起替代旧全局「未知来源」设置，允许应用**发起**安装请求）；配套 API 为 `PackageManager.java` 的 `public abstract boolean canRequestPackageInstalls();`。权限声明在 `core/res/AndroidManifest.xml`，判定入口在 `PackageManager` 与服务端 `pm` 实现，**不属** `PackageInstaller` 类。

`core/java/android/os/UserManager.java`（`main`）原文：`DISALLOW_INSTALL_UNKNOWN_SOURCES = "no_install_unknown_sources"`、`DISALLOW_INSTALL_UNKNOWN_SOURCES_GLOBALLY = "no_install_unknown_sources_globally"`（注释称后者是前者的 "device-wide version"，由持有 `MANAGE_DEVICE_POLICY_INSTALL_UNKNOWN_SOURCES` 者经 DevicePolicyManager API 设置）。两常量定义在 **`android.os.UserManager`**，**不是** `UserRestrictionsManager`（后者只是查询/设置入口，MDM 经 `DevicePolicyManager#addUserRestriction` 落地）；设备级语义强于逐用户版本，混用会导致下发失效。

### 4.2 安装验证器（Verifier）类实名

`main` 的 `core/java/android/content/pm/` 实名为 `VerifierDeviceIdentity.java`、`VerifierInfo.java`；`Verifier.java`、`Verifier2.java`、`VerificationParams.java` 在该目录不存在（`VerificationParams` 在 API 24–33 时代为独立顶层类文件）。「legacy verifier vs V2 verifier」对应公开类 `Verifier`/`Verifier2`（`android.content.pm.Verifier2`，Android 14 起从 `main` 移除），但无成文对比页，只按目录事实陈述（§9）。`INSTALL_DISABLE_VERIFICATION` 是「验证器被跳过」在代码层唯一的显式开关。

---

## 5. 公开案例

### 5.1 案例总表（13 条）

级别列为「公告／NVD」；类型缩写：`EoP` 提权、`InfoLeak` 信息泄露、`DoS` 拒绝服务、`IntentRedir` 意图重定向、`SigBypass` 签名校验绕过、`MissingCheck` 缺失校验。

| CVE | 组件／文件（公告口径） | 类型 | 级别 | 版本 | Bug ID | 公告 | 链接 |
| --- | --- | --- | --- | --- | --- | --- | --- |
| CVE-2017-13156 | **System** 小节；bug 链向 `platform/system/core`；NVD 作 "(art)" | `EoP` | High／HIGH | 5.1.1–8.0 | A-64211847 | 2017-12-01 | https://source.android.com/docs/security/bulletin/2017-12-01 |
| CVE-2018-9374 | `PackageManagerService.java`：`installPackageLI` 权限绕过 | `EoP` | 无公告行／HIGH | 未列明 | 未载 | 2018-06 | https://source.android.com/docs/security/bulletin/2018-06-01 |
| CVE-2018-9586 | `InstallPackageTask.java`：`run` 竞态致验证保持关闭 | `EoP` | High／MEDIUM | 7.0–9 | A-116754444 | 2019-01-01 | https://source.android.com/docs/security/bulletin/2019-01-01 |
| CVE-2020-0366 | `PackageInstaller`：会话 UI tapjacking 权限绕过 | `EoP` | MEDIUM | Android-11 | A-138443815 | android-11 | https://source.android.com/docs/security/bulletin/android-11 |
| CVE-2020-0401 | `PackageManagerService.java`：`setInstallerPackageName` 缺权限检查 | `EoP` | HIGH | 8.0–11 | A-150857253 | 2020-09-01 | https://source.android.com/docs/security/bulletin/2020-09-01 |
| CVE-2020-0439 | `PackageManagerService.java`：`generatePackageInfo` 检查错误（instant apps） | `EoP` | High／MEDIUM | 8.0–11 | A-140256621 | 2020-11-01 | https://source.android.com/docs/security/bulletin/2020-11-01 |
| CVE-2021-1010 | `PackageManagerService.java`：`getSigningKeySet` 缺权限检查 | `InfoLeak` | MEDIUM | Android-12 | A-189857801 | 2021-12 | https://source.android.com/docs/security/bulletin/2021-12-01 |
| CVE-2023-20927 | `AndroidManifest.xml`：可违规授予 signature 权限 | `EoP` | High | 13 | A-244216503 | aaos-2023-02 | https://source.android.com/docs/security/bulletin/aaos/2023-02-01 |
| CVE-2023-21090 | `ParsingPackageUtils.java`：`parseUsesPermission` 资源耗尽致 boot loop | `DoS` | MEDIUM | Android-13 | A-259942609 | 2023-04-01 | https://source.android.com/docs/security/bulletin/2023-04-01 |
| CVE-2025-32312 | `PackageParser.java`：`createIntentsList` 不安全反序列化绕过 lazy bundle hardening | `EoP` | High | 13–15 | A-373467684 | 2025-06-01 | https://source.android.com/docs/security/bulletin/2025-06-01 |
| CVE-2020-0204 | `package.cpp`：`InstallPackage` TOCTOU 绕过 zip 签名检查（**OS 更新包**） | `SigBypass` | —／MEDIUM | — | A-136498130 | 2020-06 | https://source.android.com/docs/security/bulletin/2020-06-01 |
| CVE-2023-30728 | `PackageInstallerCHN` < 13.1.03.00（厂商变体）：意图重定向任意文件 | `IntentRedir` | —／MEDIUM | OEM | 无 A- 号 | 无 AOSP 条目 | https://services.nvd.nist.gov/rest/json/cves/2.0?cveId=CVE-2023-30728 |
| CVE-2023-46139 | Huawei HiWallet < 5.0.3.100：对 APK 不做签名校验 | `MissingCheck` | —／未标注 | 应用层 | 无 A- 号 | 无 AOSP 条目 | https://services.nvd.nist.gov/rest/json/cves/2.0?cveId=CVE-2023-46139 |

边界与归属：`CVE-2020-0204` 对象是 OS 更新包（update_engine 侧）而非 APK，仅作 TOCTOU 对照；后两条属厂商/应用层、无 AOSP 公告条目，不代表 AOSP。**Janus（CVE-2017-13156）**名称出自研究者、公告与 NVD 均未使用，组件归属为 **System** 小节，与流传的「Framework／`PackageParser.java`」不符（§9）。**"APK Import Confusion"** 只有 Google 威胁博客口径，**无 CVE、无公告条目**（§9）。方向覆盖：v1/v2 校验→第 1、11 行；解析层→第 9、10 行；安装会话与安装器→第 2–7 行；`signature` 权限→第 8 行；密钥轮换与 split 专项 CVE→**未收录／未检索到条目**。

### 5.2 CISA KEV 核对

按 KEV 目录（`catalogVersion` **2026.10.26**，`count` 1758）的 `"cveID"` 匹配 §5.1，13 条**全部 0 命中 → CISA KEV 未收录**；同一文件内 `"vendorProject": "Android"` 条目共 17 条，0 命中不是抓取失败。边界：**未收录 ≠ 无真实利用**——KEV 只收有主动利用证据的条目且建立于 2021 年、不追溯历史；此处只能表述为「截至 2026-10-26 目录未列入」。

---

## 6. 常见误传纠正

| # | 流传说法 | 纠正后的事实（一手依据） |
| --- | --- | --- |
| 1 | v1 有官方专页 | `.../apksigning/v1` 实测 404；概览页只列 v1/v2/v3 |
| 2 | 用 `apksigner rotate-key` 轮换密钥，或存在 `PackageManager.INTERNAL_TEST` 标志 | 两者检索均 0 命中→不存在；实名 `apksigner rotate`＋`sign --lineage`、`INSTALL_INTERNAL`（`0x10`）与 `INSTALL_ALLOW_TEST`（`0x04`） |
| 3 | 轮换记录叫 `SigningDetails` | 实名是 v3 块内 additional attribute ID `0x3ba06f8c` 的 proof-of-rotation struct；`SigningDetails` 只是 framework 解析类 |
| 4 | Android 9 起即可安全轮换 | 记录自 Android 9 可携带，但 `checkSignatures` 识别整条 lineage 的边界是 **Android 13（API 33）**；官方不建议用于 API 31 及更早 |
| 5 | v4 可独立作为签名方案 | 须有配套 v2 或 v3 签名，签名体另存于 `.apk.idsig` |
| 6 | 剥离 v2 块后 Android 7.0+ 会退回 v1 验证 | 恰好相反：官方成文为直接拒绝（§1.3 原句） |
| 7 | Janus 属 Framework／`PackageParser.java` | 公告在 **System** 小节、bug 链向 `platform/system/core`、NVD 写 "(art)"；"Janus" 非官方名称 |
| 8 | "APK Import Confusion" 有 CVE／公告条目 | 仅 Google 威胁博客口径，**无 CVE、无公告条目** |
| 9 | CVE-2020-15186、CVE-2018-9567 属 APK 签名面 | 前者属 **Helm**，后者属 **Bootloader／verified boot**，均为同名噪声 |
| 10 | `PackageParser` 在 Android 13+ 被「改名」 | `main` 上文件仍在但 `@Deprecated @hide`；`ParsingPackageUtils` 实测在 `android13-release` 服务端，`main` 上该路径 NOT_FOUND 且迁入 `android.content.pm.parsing`；引用必须带分支 |
| 11 | `DISALLOW_INSTALL_UNKNOWN_SOURCES(_GLOBALLY)` 在 `UserRestrictionsManager` | 定义在 **`android.os.UserManager`**；`UserRestrictionsManager`、`DevicePolicyManager#addUserRestriction` 只是入口 |
| 12 | 用 `GET_SIGNATURES` 取 `signatures[0]` 即可 | `GET_SIGNATURES` 已 `@Deprecated`，应改 `GET_SIGNING_CERTIFICATES`（`0x08000000`）与 `signingInfo`；`PackageInfo.signatures` 注释警告 "Signature ordering is not guaranteed to be stable"，且轮换场景返回最旧证书 |
| 13 | split 有已核实属性与直接 CVE；设备一条命令可查 v2/v3；公告可用 `/YYYY/` 或 `-05` 页 | `android:requiredSplitTypes`／`android:splitTypes` grep 0 命中、split 方向**未检索到 AOSP CVE**（§9）；官方未承诺 `dumpsys` 打印 scheme 版本（§7.1）；公告链接统一 `bulletin/YYYY-MM-01` |

---

## 7. 可观测面与自查

### 7.1 实名观测点与 v2/v3 确认边界

PC 侧：`apksigner verify --print-certs app.apk` 读签名者证书（另有 `--print-certs-pem`）；`apksigner verify --verbose` 逐 scheme 报告可用性，用于发现「仅 v1」与「v1 声称 v2 已签但 signing block 缺失」两类信号；`-Werr` 可做 CI 门禁。设备侧：包数据库实名（`services/core/java/com/android/server/pm/Settings.java`，`main`）`mSettingsFilename = new File(mSystemDir, "packages.xml")`、`mPackageListFilename = new File(mSystemDir, "packages.list")`，`packages.list` 权限 `0640, SYSTEM_UID, PACKAGE_INFO_GID`（普通 shell 不可直读，需 root）、写入方法实名 `writePackageListLPr()`；安装来源入口 `pm install -i installer_package_name`；`dumpsys package` 支撑类实名 `pm/DumpHelper.java`、`pm/DumpState.java`，签名逐字段名未核实（`PackageInfoPrinter.java` 候选路径在 `main` 均 404，§9），故不承诺具体 dump 字段。

确认某包是否 v2/v3 签名，成文可行的只有两条：**离线** `apksigner verify --verbose`；**代码侧** `PackageManager.GET_SIGNING_CERTIFICATES` 读 `PackageInfo.signingInfo`（解析结果类 `android.content.pm.SigningDetails`）。边界：官方**未**提供「设备上一条命令查出 v2/v3」的方式，也**未**承诺 `dumpsys` 打印 scheme 版本；生效关系按逐条引文表述（v2→Android 7.0、v3→Android 9、v4→Android 11、lineage 识别与 v3.1→Android 13）。

### 7.2 企业/开发者自查动作

1. 流水线对每个产物跑 `apksigner verify --verbose --print-certs`，把「仅 v1」「v1 声称 v2 已签但无 signing block」设为阻断项；以 v2/v3 为签名基线、v1 仅为兼容旧设备保留（官方共存口径 "first with v1, then v2, and then v3"）。
2. 调整 `minSdk`/`targetSdk` 时评估 `--rotation-min-sdk-version` 与 v3.1 组合，把轮换生效范围限定在 Android 13（API 33）及以上；轮换前核对 lineage（当前签名者须为链表末端）并归档 `apksigner rotate` 产物。
3. 代码侧弃用 `GET_SIGNATURES` 与 `PackageInfo.signatures`，改用 `GET_SIGNING_CERTIFICATES`/`signingInfo`，禁止按下标取证书做判定。
4. 把 `INSTALL_FAILED_BAD_SIGNATURE`、`MISSING_SPLIT`、`MULTIPACKAGE_INCONSISTENCY`、`SHARED_USER_INCOMPATIBLE`、`UID_CHANGED`、`VERIFICATION_FAILURE`/`VERIFICATION_TIMEOUT` 列为安装失败必采错误码；下载后安装或更新的 APK 一律先验签（对照 §5 第 12、13 行），分发 split 时校验 base 与各 config split 签名一致。
5. MDM 侧区分 `UserManager.DISALLOW_INSTALL_UNKNOWN_SOURCES`（逐用户）与 `…_GLOBALLY`（设备级），确认无非特权组件持有 `INSTALL_PACKAGES`，并监控 `INSTALL_DISABLE_VERIFICATION`、`INSTALL_BYPASS_LOW_TARGET_SDK_BLOCK`、`INSTALL_GRANT_ALL_REQUESTED_PERMISSIONS` 的异常使用。
6. 定期用 `dumpsys package` 与包数据库对已发设备做包清单与签名者一致性抽样，出现 `INSTALL_FAILED_UID_CHANGED`、`SHARED_USER_INCOMPATIBLE` 即时上报。

---

## 8. 与其他手册的分工

- 与《框架 IPC 与组件导出》：本篇只管「签名→安装→包校验」链上的实名接口与规则，`Intent` 语义、组件 `exported`/`permission` 治理、Binder 鉴权归该篇。与《Mainline 与 APEX 模块》：`INSTALL_APEX`、`INSTALL_STAGED` 在此只作安装标志列出，APEX 签名体系与 staged session 生命周期归该篇。
- 公告链接形式、`EoP` 归类与双轨定级须与已发布篇目一致：[CVE 与安全公告](/android-cve-and-security-bulletins/)、[安卓系统漏洞报告](/android-system-vulnerability-report/)、[常见系统漏洞](/android-common-system-vulnerabilities/)、[提权与 RCE](/android-rce-and-privilege-escalation/)、[漏洞定级方法](/android-vulnerability-severity-assessment/)、[多用户沙箱隔离](/android-multiuser-sandbox-isolation/)、[TEE 密钥证明](/android-tee-key-attestation/)。APK signing key 与密钥证明、verified boot、基带签名不属同一体系，勿把 §2 的 lineage 与 attestation 混用（§6 第 9 行即一例同名误引）。

---

## 9. 未核实与来源失效

以下条目**不得**作为正文结论引用，只记录尝试过的 URL/关键词与失效事实。

1. `.../features/apksigning/v1`→404；v1 内容仅在概览页 "JAR signing" 小节。
2. 公告 `-05` 日期页（`2018-01-05`/`2019-09-05`/`2020-11-05`/`2021-09-05`/`2022-09-05`/`2023-10-05`/`2024-01-05`/`2024-06-05`）与 `/YYYY/` 目录形式均 404；可用形式仅 `bulletin/YYYY-MM-01`。
3. **未找到官方成文的 Janus 技术分析页**；「Janus 归属 Framework／`PackageParser.java`」与官方记录不符。
4. `PackageManager.INTERNAL_TEST`、`apksigner rotate-key`：检索 0 命中→不存在。
5. `android:requiredSplitTypes`／`android:splitTypes`：`main` 的 `core/res/AndroidManifest.xml` grep 0 命中；`SessionParams#setRequiredSplitTypes(...)`、`IPackageInstaller#createSession` 线索未逐行核实。
6. "APK Import Confusion"：两轮检索未命中 NVD CVE 或公告条目，也未定位 Google 官方博客原文→仅见于媒体/威胁博客口径。
7. split APK 方向的公开 AOSP CVE：`keywordSearch=split apk` 仅命中无关项、`keywordSearch=PackageInstaller session` 0 命中→**未检索到条目**。
8. `developer.android.com/google/play/app-signing`→404；Play App Signing 只能引 support.google.com 的 Play Console Help 页（发布管道约束，不属设备端校验）。
9. 同名噪声（已剔除留档）：CVE-2018-9567 属 **Bootloader/verified boot**；CVE-2020-15186 实为 **Helm**。
10. `PackageInfoPrinter`：`.../pm/printer/` 与 `.../pm/pkg/print/` 在 `main` 均 404；`dumpsys` 签名逐字段名与 `installerPackageName`/`installSource` 的 dump 字段名未核实（仅 `DumpHelper`/`DumpState` 实名）。
11. 「legacy verifier vs V2 verifier」无成文对比页，只有 `Verifier`/`Verifier2`/`VerificationParams` 的目录存在性证据；设备单条命令确认 v2/v3 无成文方式（§7.1）。
12. 密钥轮换专项 CVE、未知来源安装的 CVE 级条目：素材无对应条目；`CVE-2018-9374` 的 `A-` 号与受影响版本在 NVD 未出现，按「未载」标注。

---

## 10. 参考

- 签名方案：`source.android.com/docs/security/features/apksigning`（Overview、`/v2`、`/v3`、`/v3-1`、`/v3-2`、`/v4`）；公告：`source.android.com/docs/security/bulletin/`＋`2017-12-01`、`2018-06-01`、`2018-12-01`、`2019-01-01`、`android-11`、`2020-06-01`、`2020-09-01`、`2020-11-01`、`2021-12-01`、`2023-04-01`、`2025-06-01`、`aaos/2023-02-01`。
- 工具：`developer.android.com/tools/apksigner`、`android.googlesource.com/platform/tools/apksig/`、`developer.android.com/tools/adb`。
- AOSP `platform/frameworks/base`（`+/refs/heads/<branch>/`）：`main/core/java/android/content/pm/`（`PackageManager`、`PackageInfo`、`PackageInstaller`、`PackageParser`、`SigningDetails`、`VerifierDeviceIdentity`、`VerifierInfo`）、`main/core/java/android/os/UserManager.java`、`main/core/res/AndroidManifest.xml`、`main/services/core/java/com/android/server/pm/`（`PackageInstallerService`、`PackageSessionVerifier`、`VerificationUtils`、`DumpHelper`、`DumpState`、`Settings`、`PackageManagerShellCommand`）、`android13-release/.../pm/pkg/parsing/ParsingPackageUtils.java`、`android14-release` 的 `VerifyingSession`。
- 漏洞库：`services.nvd.nist.gov/rest/json/cves/2.0?cveId=CVE-…`（核对 §5.1 全部 13 条）、`cisa.gov/.../known_exploited_vulnerabilities.json`（`catalogVersion` 2026.10.26）。

---

## 11. 联系方式

咨询 Android 系统详情请联系 https://t.me/one00190

*本手册仅供安全学习与防御研究使用；所有组件事实、接口名称与案例归属均取自公开一手来源并附链接，未能核实的表述已单列于 §9 而非写入正文。*
