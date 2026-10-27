---
layout: page
title: "Android Project Mainline 与 APEX 模块化更新攻击面手册"
description: "把 Mainline 能力与 APEX 容器两个常被混用的概念分开：逐字模块清单与容器形态、APEX 双重签名与版本回滚语义、Mainline 与 Play 两条更新通道及扁平化 APEX 例外、公告中 Mainline 条目的实际形态、SPL 与模块版本不等价的双轴处置；含已核实条目与两处公告空白"
date: 2026-10-26
permalink: /android-mainline-apex-modules/
---

> Mainline 把一部分系统组件从「整机 OTA 才能修」变成「模块单独就能修」。这件事同时改变了两件事：**漏洞的修复可达性**，和**漏洞的归属与观测方式**。本手册沿这两条线给出组件实名、容器与签名机制、真实可观测项，以及逐条核实归属的公开案例。定位是防御与教育，不含可利用代码。

---

## 0. 使用方式与边界

**为什么单独写这一份。** 资料库里其它手册按「层」或「缺陷类型」组织，都默认一个前提：修一个漏洞要等厂商推送整机更新。Mainline 打破了这个前提。于是三个原本简单的问题变复杂了：这个漏洞属于哪个组件？这台设备到底修没修？SPL 能不能作为判据？本手册专门处理这三问。

**怎么用。**
- 判断某个包是不是可更新模块 → §2 的逐字清单
- 想知道 APEX 的完整性由什么保证 → §3
- 想把「模块可更新性」纳入定级 → §6 与 §7
- 核对某条公告是不是 Mainline 条目 → §5
- 某说法我没能核实 → §8 单列

**边界。** 只讨论公开、已修复内容与官方机制文档；不涉及任何构造恶意 APEX、伪造签名的做法。厂商对 Mainline 的裁剪程度属实现细节，本手册只描述 AOSP 与官方文档层面的契约。

---

## 1. 两个被混用的名词先分开

| 术语 | 官方定义 | 常见混用 |
|---|---|---|
| **Project Mainline** | 一种**能力**：让一部分组件在正常 Android 发布周期之外更新，官方描述为以"非侵入方式"广泛、快速、无缝地向最终用户设备分发更新 | 被当成一个"进程"或"服务" |
| **APEX** | **Android Pony EXpress** 容器格式，Android 10 引入。是**打包格式**，不是更新机制本身 | 常被写成 "Android Portable Extensions"——官方名称是 Pony EXpress |

关键区分（官方原话）："Some Mainline modules use the APEX container format (introduced in Android 10) and some use the APK format."
→ **Mainline 模块有两种容器：APEX 与 APK。** 我在别处见过的 "APK-in-APEX / HAL-in-APEX" 这对说法不是官方术语；官方对 APEX 内容的分类是 "native shared libs / executables / JAR / data / config"，而**以 APK 形态交付的模块本身就不是 APEX**（如 PermissionController、Network Stack）。服务类更新通过 `.rc` 文件里的 `updatable` 与 `override` 属性实现——这才是与"HAL 更新"最接近的官方机制。

来源：[modular-system](https://source.android.com/docs/core/ota/modular-system) · [apex](https://source.android.com/docs/core/ota/apex)

---

## 2. 可更新模块清单（逐字抄自官方模块页）

包名、容器形态、进入清单的版本，均按 [Project Mainline modules](https://source.android.com/docs/core/ota/modular-system) 英文页逐字记录。**这是判断"这个 CVE 能不能走模块通道修"的第一手依据**——修复可达性取决于漏洞所在包在不在表里、以什么容器交付。

| 模块 | 包名 | 容器 | 起始版本 |
|---|---|---|---|
| adbd | `com.android.adbd` | APEX | 11 |
| AdServices | `com.google.android.adservices` | APEX | 13 |
| Android Health | `com.google.android.healthfitness` | APEX | 14 |
| AppSearch | `com.android.appsearch` | APEX | 13 |
| ART | `com.android.art` | APEX | 12 |
| Bluetooth | `com.google.android.bt` | APEX | 13 |
| Cell Broadcast | `com.android.cellbroadcast` | APEX | 11 |
| Config Infrastructure | `com.android.configinfrastructure` | APEX | 14 |
| Conscrypt | `com.android.conscrypt` | APEX | 10 |
| Device Scheduling | `com.android.scheduling` | APEX | 12 |
| DNS Resolver | `com.android.resolv` | APEX | 10 |
| Documents UI | `com.android.documentsui` | **APK** | 10 |
| Ext Services | `com.android.ext.services` | APK(10) / APEX(11) | 10 |
| IPsec / IKEv2 | `com.android.ipsec` | APEX | 11 |
| Media | `com.android.media` | APEX | 10 |
| Media Codecs | `com.android.media.swcodec` | APEX | 10 |
| Media Provider | `com.android.mediaprovider` | APEX | 11 |
| Module Metadata | `com.android.modulemetadata` | APK | 10 |
| Network Stack | `com.android.networkstack`（+ `.permissionconfig`、`com.android.captiveportallogin`） | **APK** | 10 |
| NNAPI Runtime | `com.android.neuralnetworks` | **APK** | 11 |
| On-device Personalization | `com.google.android.ondevicepersonalization` | APEX + APK | 13 |
| Federated Compute | `com.google.android.federatedcompute` | APK | 13 |
| Permission Controller | `com.android.permissioncontroller` | APK | 10 |
| Remote Key Provisioning | `com.android.rkpd` | APEX | 14 |
| SDK Extensions | `com.android.sdkext` | APEX | 11 |
| Statsd | `com.android.os.statsd` | APEX | 11 |
| Tethering | `com.android.tethering` | APEX | 11 |
| Time Zone Data | `com.android.tzdata` | APEX | 10 |
| UWB | `com.android.uwb` | APEX | 13 |
| Wi-Fi | `com.android.wifi` | APEX | 11 |

三条读法：

1. **APK 形态的模块同样可更新**，别把"Mainline = APEX"当成等式。DocumentsUI、Network Stack、PermissionController、NNAPI 这些高攻击面组件都是 APK 形态。
2. **`com.android.i18n` 不在现行清单**（旧资料常列它）。同样，官方页面**没有** "required/eligible vs optional" 这一分类字样，网传的分类法我没有找到一手支撑（见 §8）。
3. 中文页把蓝牙写作 `com.google.android.btservices`，与英文页 `com.google.android.bt` 不一致——**以英文页为准**，写检测规则时不要依赖译文。

---

## 3. APEX 容器的完整性契约

### 3.1 结构与运行位置

`apexd` 是独立原生进程，官方职责表述："The APEX manager (or apexd) is a standalone native process responsible for verifying, installing, and uninstalling APEX files."

真实常量（[apex_constants.h](https://android.googlesource.com/platform/system/apex/+/refs/heads/main/apexd/apex_constants.h)）：`kApexPackageSuffix = ".apex"`、`kApexRoot = "/apex"`；运行时装载路径 `/apex/<apex_name>@<version>`；另有 `.capex`（压缩）、`.decompressed.apex`、`.ota.apex` 几种扩展形态。设备侧模块清单的已核实载体是 `/apex/apex-info-list.xml`（`kApexInfoList`）。

一个 APEX（ZIP 层）含四个文件：`apex_manifest.json`、`AndroidManifest.xml`、`apex_payload.img`、`apex_pubkey`。

### 3.2 双重签名（最容易被误述的一点）

官方描述了两把**不同用途**的密钥：

> "First, the apex_payload.img (specifically, the vbmeta descriptor appended to apex_payload.img) file is signed with a key. Then, the entire APEX is signed using the APK signature scheme v3."

拆开看：

| 层 | 保护什么 | 用什么 |
|---|---|---|
| payload 镜像 | `apex_payload.img` 内容 | 附加的 **vbmeta 描述符**，由 `apex_key` 签名，公钥以 `avb_pubkey` 内置；payload 由 libavb 生成哈希树并挂 **dm-verity** |
| APEX 整包 | 容器与清单本身 | **APK 签名方案 v3**（`certificate`） |
| `apex_pubkey` | 下载来的 APEX 与内置 APEX **是否同一签名者** | 用于比对签名者，不是加密 |

两把密钥**可以不同**。防御含义：只看 APEX 的 APK v3 签名并不能证明 payload 的 dm-verity 根来自同一信任根；反过来只验 payload 哈希树也不覆盖清单与元数据。评估实现时要问清厂商这两处分别接的是谁的密钥体系。

### 3.3 与 Verified Boot 的关系（含一处纠正）

AVB 侧的 rollback protection 是有官方明文的机制："Rollback protection helps to prevent a possible exploit from becoming persistent by ensuring devices only update to newer versions of Android."（[Verified Boot](https://source.android.com/docs/security/features/verifiedboot)）

**纠正一处常见说法：** 我没有在 Verified Boot 文档里找到 "chained partition descriptor 用于 APEX" 这类表述——该页既不出现 "chained partition" 也不出现 "APEX"。APEX 与 AVB 的实际联系就是 §3.2 里 payload 上那段 vbmeta 描述符与 libavb/dm-verity，而不是分区链式描述符。把二者混为"链式分区"是**未核实**的推导（§8）。

### 3.4 版本约束与"能不能回滚"

官方对更新门槛的措辞很具体："If an APEX is present in a built-in partition, the APEX can be updated by providing an APEX package with the same package name and a **greater than or equal to** version code."

**注意是"大于或等于"**，不是严格大于——这意味着同名同版本的重复安装是被允许的形态，而不是异常。

源码里确实存在 `apexrollback` 快照目录与 `blocklist.json`（"brand new apex" 校验文件），但官方文档**没有**把它们表述成防降级机制，所以"模块级防降级"这一说法我只能标为未核实。与之相反，"模块可回滚"有官方产品化能力：Google Play System Update Rollbacks（GPSUR，[文档](https://developer.android.com/work/dpc/system-updates)）——即官方预期"某个模块更新出问题要能撤回"。

**防御读法。** 这两条合起来是本手册最重要的结构性结论：**模块更新通道同时支持"更快修复"与"官方支持的撤回"**。因此"某个模块版本出现在设备上"不等于"该模块处于最新安全状态"，也不能只靠 SPL 判断（§7）。

---

## 4. 分发通道与真实可观测项

### 4.1 官方只写了两条通道

[modular-system](https://source.android.com/docs/core/ota/modular-system) 列出的是：**Google Play system updates** 与 **partner OTA**。网上常见的"第三条厂商私有通道"我没有找到官方表述，属未核实。

用户侧入口（官方措辞）："Google Play System updates can also be manually installed by navigating to **Settings > About > Android Version > Google Play system update**."

**一个重要例外：扁平化 APEX 不走这条快车道。** 官方："Flattened APEXes can be updated only via a regular OTA."（Flattened APEX 自 Android 15 起已废弃。）也就是说，同样标称"Mainline 模块"，在旧机型上可能实际只能等整机更新——这是跨机型比较修复速度时必须问的一句。

### 4.2 命令实名（含两处纠正）

服务名是 **`apexservice`**（`constexpr const char* kApexServiceName = "apexservice";`，[apexservice.cpp](https://android.googlesource.com/platform/system/apex/+/refs/heads/main/apexd/apexservice.cpp)）。

| 观测 | 真实形态 |
|---|---|
| shell 子命令 | `cmd apexservice` 仅 `help` / `getAllPackages` / `getActivePackages` / `getStagedSessionInfo` |
| dump | `dumpsys apexservice` 输出含 `ACTIVE PACKAGES:`、`SESSIONS:` 段 |
| 设备侧清单文件 | `/apex/apex-info-list.xml` |
| 挂载视图 | `/apex/<name>@<version>` |

**两处纠正：**
- **`adb shell cmd apex` 不成立**——源码里没有 `apex` 这个服务名，实名是 `cmd apexservice`。
- **`adb shell pm list modules` 未核实/疑不存在**——`PackageManagerShellCommand` 里没有 `list modules`，能找到的是 `get-moduleinfo [--all|--installed]`。引用这条命令的资料需要复核。

### 4.3 版本与状态怎么读

`getActivePackages` / `dumpsys apexservice` 的 `ACTIVE PACKAGES` 给出包名与活动版本；`getStagedSessionInfo` 给出暂存会话状态。把它们与公告里的**受影响版本区间**对照，是目前唯一可在设备侧独立验证"模块版本是否落在受影响区间"的办法——`ro.build.version.security_patch` 不覆盖这一层（§7）。

---

## 5. Mainline 在安全公告里的真实形态

### 5.1 现行措辞

**纠正一处常见引用：** 现行公告里我没能找到 "Google Play components update only" 这一措辞。现行表格的原文是：

> "The following issues are included in Project Mainline components."

即公告用**单独的 Mainline 表格**来标注，而不是在每行加"仅 Play 更新"标签。另外，Pixel 机型专属公告的表**没有 "Android version" 列**（AOSP 公告才有），跨表比对时不要按 AOSP 表的列结构去找。

### 5.2 已核实的公告条目（两期为例）

**2026-03-01（Published March 2, 2026）Mainline 表：** Documents UI `CVE-2026-0013`；MediaProvider `CVE-2025-48544`/`48567`/`48578`/`48579`/`48582`、`CVE-2026-0024`/`0035`；Media Codecs `CVE-2026-0006`；Profiling `CVE-2025-48585`/`48587`。明细：`CVE-2026-0006` A-423894847 RCE / Critical（System）；`CVE-2026-0035` A-418773439 EoP / High；`CVE-2026-0024` A-326211886 ID / High；`CVE-2025-48585` A-425360742 DoS / High；`CVE-2025-48544` A-415783046 EoP / High。来源 [2026-03-01](https://source.android.com/docs/security/bulletin/2026/2026-03-01)

**2025-09-01（Published September 2, 2025）Mainline 表：** ART `CVE-2025-48543`；AppSearch `CVE-2025-26464`；Cell Broadcast `CVE-2025-48534`；Documents UI `CVE-2025-32323`；MediaProvider `CVE-2025-32327`/`48532`/`48544`；Permission Controller `CVE-2025-48547`；Wi-Fi `CVE-2025-48524`。明细：`CVE-2025-48543` A-421834866 EoP / High，Android Runtime，**13–16**。来源 [2025-09-01](https://source.android.com/docs/security/bulletin/2025-09-01)

**读法。** 一条 EoP/High 的 ART 缺陷出现在 Mainline 表里，意味着**理论上可经 Play 模块通道触达**；这也解释了为什么它的受影响区间横跨四个大版本（13–16）——版本跨度反映的是模块分发范围，不是整机 OTA 范围。

### 5.3 apexd 自身的公开条目

容器管理器本体并非空白面：

| CVE | 位置与描述（NVD 原述要点） | 影响 |
|---|---|---|
| CVE-2020-0322 | "In apexd, there is a possible out of bounds read due to a missing bounds check" | 本地信息泄露 |
| CVE-2021-0702 | `apexd.cpp`，不当文件共享 | 本地信息泄露 |
| CVE-2025-48581 | "In `VerifyNoOverlapInSessions` of apexd.cpp, there is a possible way to **block security updates**… local escalation of privilege" | 阻断更新 + 本地提权 |

CVE-2025-48581 值得单独注意：它的语义不是"读多了点内存"，而是**让安全更新装不上**。这类缺陷破坏的是更新链路本身，处置优先级不能按信息泄露常规排。来源 [NVD 关键词 apexd](https://services.nvd.nist.gov/rest/json/cves/2.0?keywordSearch=apexd)

### 5.4 KEV 核对

对照 CISA [KEV 全量 JSON](https://www.cisa.gov/sites/default/files/feeds/known_exploited_vulnerabilities.json)（vendorProject = Android 共 17 条）逐条精确核查：

- **在 KEV**：`CVE-2025-48543`（"Android Runtime Use-After-Free Vulnerability"，product "Runtime"，dateAdded 2025-09-04，dueDate 2025-09-25）
- **不在 KEV**：`CVE-2026-0006`、`CVE-2026-0024`、`CVE-2026-0035`、`CVE-2025-48581`、`CVE-2020-0322`、`CVE-2021-0702`

**含义。** 已列名的那条恰好落在**可经模块通道分发的 ART** 上，而它的 KEV 记录里 product 写的是 "Runtime"、厂商属性是 Android——这类条目在实战意义上直接考验"Play 通道是否真的到得了这台设备"。KEV 的 `dueDate`（此处 2025-09-25）比 `dateAdded` 更适合作为企业整改的时限锚点。

### 5.5 两处「找不到」比找到更有信息量

- **Conscrypt**：NVD 关键词 `conscrypt` 仅命中 Android 10 之前时代的条目（CVE-2016-0818/-2461/-2462/-3840/-6709、CVE-2017-13309，后者描述 "In `readEncryptedData` of ConscryptEngine.java, there is a possible plaintext leak"）。**Mainline 时代的 Conscrypt 公告条目：未核实。**
- **NNAPI**：NVD 关键词 `NNAPI`、`neuralnetworks` 命中 0 条；2025-09 与 2026-03 两期 Mainline 表均无 NNAPI 行。**未核实。**

这两处空白不该读成"这两个模块没有漏洞"，而应读成：Mainline 表里出现的**组件名与实际代码库**之间存在映射缝隙（例如加密实现缺陷被记在别的组件名下）。做组件级资产盘点时，用 §2 的包名而不是模块显示名去对齐。

---

## 6. 缺陷形态汇总（模块化引入的额外面）

| 缺陷形态 | 说明 | 已核实支撑 |
|---|---|---|
| 模块本体内存缺陷 | ART / MediaProvider / Media Codecs 里的 UAF、越界 | §5.2 各条 |
| 容器管理器缺陷 | apexd 边界校验、会话重叠检查，后果含**阻断更新** | CVE-2025-48581、2020-0322、2021-0702 |
| 信任根二元性 | payload（vbmeta/avb）与整包（APK v3）两把密钥可能不一致 | §3.2 官方原话 |
| 分发通道差异 | 扁平化 APEX 只能走整机 OTA；Android 10 才有 Play 通道 | §4.1 官方原话 |
| 版本门槛语义 | "大于或等于" 允许同名同版本重复安装 | §3.4 官方原话 |
| 撤回能力 | GPSUR 支持官方撤回，故"版本存在 ≠ 最新安全状态" | §3.4 |
| 观测口径缺失 | `cmd apex` / `pm list modules` 常被误引为可用命令 | §4.2 |

---

## 7. 把「可更新性」做成第二维判据

### 7.1 SPL 与模块更新不等价（官方自己加了限定）

公告 FAQ 的现行措辞是**有条件的**：

> "For some devices on Android 10 or later, the Google Play system update will have a date string that matches the 2026-03-01 security patch level."

以及："Devices with Android 10 and later may receive security updates as well as Google Play system updates."

同时，SPL 的声明范围仍只绑定公告条目："Security vulnerabilities that are documented in this security bulletin are required to declare the latest security patch level…"，而 SPL 本身是厂商属性 `ro.build.version.security_patch`。

**结论。** 官方用 "For some devices" 限定，说明**不能把 SPL 当作模块实际状态的充分证明**。（"Google 明文说明 SPL 不反映 Mainline 更新"这一更强的命题我没有找到一手表述，标未核实。）

### 7.2 建议的双轴处置

把每个条目按两轴落位，而不是只看级别：

| | 模块可更新（APEX/APK 在 §2 清单内） | 仅整机 OTA |
|---|---|---|
| **KEV / 已列名活跃** | 最高优先：确认 Play 通道可达 + 用 `dumpsys apexservice` 核版本 | 最高优先：等厂商 OTA，评估临时缓解 |
| **未列名但高危** | 高优先：核对模块版本即可确认修复，通常数日内可达 | 常规：按厂商节奏 |

这里"可达"是设备属性，不是机型属性——扁平化 APEX、Android 10 以下、以及厂商未接入 Play 通道的设备，都落在右列。

### 7.3 自查清单

1. 目标 CVE 的组件名能否映射到 §2 的某个包？映射不到（如 Conscrypt/NNAPI 的 Mainline 条目）→ 按整机 OTA 处理。
2. 该模块是 APEX 还是 APK 形态？两者都可更新，但观测手段不同（APEX 看 `/apex` 与 `apexservice`，APK 形态看包版本）。
3. 设备是否具备 Play 通道（Android 10+ 且厂商接入）？设置入口见 §4.1。
4. 是否扁平化 APEX？是 → 只能整机 OTA。
5. 修复确认用什么口径？**模块版本**（`cmd apexservice getActivePackages` / `dumpsys apexservice`），而非仅 SPL。

---

## 8. 未核实与来源失效（不作断言）

1. **required / eligible vs optional 模块分类** — [modular-system](https://source.android.com/docs/core/ota/modular-system) 页面未出现 required/mandatory/optional 字样（另试 `/modular-system/conscrypt`）。
2. **`pm list modules`、`cmd apex`** — 分别疑不存在／实名应为 `cmd apexservice`（查 [PackageManagerShellCommand.java](https://android.googlesource.com/platform/frameworks/base/+/refs/heads/main/services/core/java/com/android/server/pm/PackageManagerShellCommand.java)；`apexd/apex_service.cc` 为 404，正确文件 `apexd/apexservice.cpp`）。
3. **"Google Play components update only"** — 现行公告未找到该措辞。
4. **APEX 使用 AVB chained partition descriptor** — Verified Boot 页无 "chained partition"、无 "APEX"。
5. **`apexrollback` / `blocklist.json` 的防降级语义** — 文件存在于源码，官方文档未如此表述。
6. **Mainline 时代的 Conscrypt / NNAPI 公告条目** — NVD 关键词与两期公告均无命中。
7. **"SPL 不反映 Mainline 更新"的强命题** — 官方仅有 "For some devices…" 的条件句。
8. **Google Play 覆盖的设备比例等数字** — 官方文档未给出。
9. **失效 URL（勿引用）**：`/docs/security/bulletin/2022/2022-06-05`、`/2025/2025-09-05`、`/2025/2025-09-01`（2025-09 正确为 `/bulletin/2025-09-01`，2026 起为 `/bulletin/2026/2026-03-01`）。
10. **中文页与英文页模块包名不一致**（蓝牙 `com.google.android.btservices` vs `com.google.android.bt`）— 以英文页为准。

---

## 9. 与文档库其他部分的分工

| 需要 | 去看 |
|---|---|
| 整体攻击面与启动链 | [总览报告](/android-system-vulnerability-report/) |
| 组件 × 缺陷类别坐标（内核/驱动/Binder/HAL/协议） | [底层漏洞类型图谱](/android-low-level-vulnerability-types/) |
| CVE / bug ID / SPL / 公告体系怎么读 | [编号与公告手册](/android-cve-and-security-bulletins/) |
| 定级口径与处置矩阵 | [评估与定级手册](/android-vulnerability-severity-assessment/) |
| TEE / 密钥 / 启动完整性 / 存储加密 | [TEE 与密钥认证手册](/android-tee-key-attestation/) |
| 本文 | 可更新组件的容器契约、分发通道、公告归属与版本核验 |

---

## 参考

- [Project Mainline (modular-system)](https://source.android.com/docs/core/ota/modular-system)
- [Android Pony EXpress (APEX) format](https://source.android.com/docs/core/ota/apex)
- [apex_constants.h](https://android.googlesource.com/platform/system/apex/+/refs/heads/main/apexd/apex_constants.h) · [apexservice.cpp](https://android.googlesource.com/platform/system/apex/+/refs/heads/main/apexd/apexservice.cpp)
- [Google Play System updates（企业侧）](https://developer.android.com/work/dpc/system-updates)
- [Verified Boot（rollback protection）](https://source.android.com/docs/security/features/verifiedboot)
- [Android Security Bulletin 2026-03-01](https://source.android.com/docs/security/bulletin/2026/2026-03-01) · [2025-09-01](https://source.android.com/docs/security/bulletin/2025-09-01) · [ASB overview](https://source.android.com/docs/security/bulletin/asb-overview)
- [NVD REST 检索（apexd）](https://services.nvd.nist.gov/rest/json/cves/2.0?keywordSearch=apexd) · [NVD CVE-2020-0322](https://nvd.nist.gov/vuln/detail/CVE-2020-0322) · [NVD CVE-2025-48581](https://nvd.nist.gov/vuln/detail/CVE-2025-48581)
- [CISA KEV JSON](https://www.cisa.gov/sites/default/files/feeds/known_exploited_vulnerabilities.json)

---

## 联系方式

咨询 Android 系统详情请联系 https://t.me/one00190

---

*本手册仅供安全学习与防御研究使用；所有模块名、接口名与案例归属均取自公开一手来源并附链接，未能核实的表述已单列于 §8 而非写入正文。*
