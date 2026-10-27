---
layout: page
title: "Android 基带与电话子系统漏洞手册"
description: "应用处理器与基带两侧的分工及两条修复通道、RIL AIDL 子域实名与内核侧宿主接口（QMI/MHI/QRTR/SMD/BAM/CCCI）、高通/联发科/三星公告的组件命名轴、SPL 与基带固件版本的真实关系、Project Zero 暴露面结论与逐条核实的 modem 类 CVE 归属"
date: 2026-10-26
permalink: /android-baseband-telephony/
---

> 基带是 Android 设备上唯一"常年联网、常年解析外来报文、且大部分代码不在 AOSP 里"的处理器。本手册把这条攻击面拆开：谁在跑、代码归谁、缺陷落在哪一层、公告怎么命名、以及你能在自己设备上看到什么。定位是防御与教育，只引用公开且已修复的信息，不含可利用代码，不描述任何构造报文或触发路径的细节。

---

## 0. 使用方式与边界

**为什么单独成篇。** [底层漏洞类型图谱](/android-low-level-vulnerability-types/) 把无线与近场协议列为一个组件段落；本手册把它展开成完整一条链：基带处理器与主机的边界、RIL 分层、3GPP 协议面、厂商公告命名差异、以及"基带修复到底算不算 SPL"这个高频误答。蓝牙与 Wi‑Fi 的协议缺陷（BlueFrag、FragAttacks）在 §5 只做定位与纠错，内容归属仍在图谱手册。

**怎么用。**
- 判断某个"基带漏洞"到底在哪一侧（AP / 基带 / SoC 固件） → §1、§2
- 核对某条 Modem CVE 的真实归属与公告命名 → §5 案例表 + §6 误传纠正
- 读懂高通/联发科/三星公告里 "Technology Area = Modem" 的含义 → §3
- 评估修复责任与 SPL 覆盖 → §4
- 在设备上做实况核对 → §7

**边界。** 不讨论任何协议报文构造、不讨论固件提取与逆向操作步骤、不讨论基带提权后的持久化。厂商基带固件（通常未公开源码）的内部实现只描述到接口边界。

---

## 1. 两个处理器，两套代码，两条修复通道

一台 Android 设备里至少有两个执行环境在"处理网络"：

| | 应用处理器（AP）侧 | 基带 / Modem 侧 |
|---|---|---|
| 跑什么 | Android 内核 + 用户态 framework、`rild`、电话服务 | 独立 RTOS 或裸机固件，实现 L1/L2/L3 协议栈 |
| 代码来源 | AOSP（可审计） | SoC 厂商私有固件（多数不公开源码） |
| 归属公告 | Android 安全公告（ASB）/ Pixel 公告 | 高通、联发科、三星半导体各自的产品安全公告 |
| 修复分发 | OTA，随 SPL 计 | SoC 厂商 → OEM → 随设备固件更新（常与 Android 版本解耦） |

Google 对这一分工的表述是一手的：ASB 概览页写明 SoC 厂商的修复**不来自 AOSP 代码**，而是 "Fixes from SOC manufacturers are available directly from the manufacturers"（[ASB overview](https://source.android.com/docs/security/bulletin/asb-overview)）。这句话是本手册所有"归属判断"的地基：**一条 Modem 漏洞不在 ASB 表格里出现，不代表它不存在，而多半意味着它根本不属于 AOSP 侧。**

### AP 侧的真实形状

- **RIL HAL 目录**：AOSP `hardware/interfaces/radio/` 下是 HIDL 版本目录 `1.0`–`1.6`，另有 `aidl/`、`config/`、`deprecated/`（[radio 目录](https://android.googlesource.com/platform/hardware/interfaces/+/refs/heads/main/radio/)）。
- **AIDL 接口按子域拆分，不存在一个顶层 `IRadio.aidl`**。实名是 `IRadioModem`、`IRadioNetwork`、`IRadioSim`、`IRadioVoice`、`IRadioMessaging`、`IRadioData`、`IRadioIms`、`IRadioConfig`，再加服务接入点 `ISap`（[radio/modem 目录](https://android.googlesource.com/platform/hardware/interfaces/+/refs/heads/main/radio/aidl/android/hardware/radio/modem/)）。每个子域还配 Request/Response/Indication 三件套，例如 `IRadioModemIndication.aidl`。
- **`rild` 仍在树里**：`hardware/ril/` 保留 `rild/`、`libril/`、`reference-ril/`；`rild` 通过命令行 `-l` 参数或 `rild.libpath` 属性加载厂商 RIL 库（[rild.c](https://android.googlesource.com/platform/hardware/ril/+/refs/heads/main/rild/rild.c)：原话 "vendor ril lib path either passed in as -l parameter, or read from rild.libpath property"）。这一句同时解释了"为什么基带侧的私有代码会跑在 AP 的进程里"——厂商 RIL 库就是 AP 侧的私有代码。

## 2. AP ↔ Modem 的宿主接口：内核里有实名

基带固件不可读，但它和 AP 之间的通道在内核源码里是可读的。以下都是**一手代码**确认存在的：

| 名称 | 证据 |
|---|---|
| QMI = **Qualcomm MSM Interface** | `drivers/net/usb/qmi_wwan.c` 原话："specific management protocol called Qualcomm MSM Interface (QMI)"（[kernel 源码](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/plain/drivers/net/usb/qmi_wwan.c)），承载于 USB-CDC |
| MHI | `include/linux/mhi.h`："mhi_ctxt: MHI device context, shared memory between host and device"（[源码](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/plain/include/linux/mhi.h)）——注意它是 host/device **共享内存**语义 |
| QRTR | `net/qrtr/af_qrtr.c` 存在 |
| SMD | `smd-rpm.c` 存在 |
| BAM DMA | `drivers/dma/qcom/bam_dma.c`："QCOM BAM DMA engine driver" |
| MediaTek CCCI | 联发科公告里以 Subcomponent **"ccci"** 实名出现 |

**为什么防御者要在意这一层。** 这些接口是 AP 内核里真正接收基带数据的入口：一条"基带侧"的越界写，最终可能被 AP 侧的 QMI/CCCI 驱动观察到；反过来，AP 内核驱动对这些通道的解析缺陷，会被归到内核/驱动类而不是协议类。**同一个现象，公告可能写在完全不同的组件下**——这是基带类 CVE 归属最容易谈错的地方。

未核实项：MCM 的厂商展开与接口细节；SMD/BAM 缩写的逐字展开（仅确认源文件存在）；`CCCI` 的缩写展开（仅确认公告里的 Subcomponent 名）。

## 3. 协议面：缺陷名字来自规范，不是来自 Android

蜂窝侧的缺陷类型基本可以按 3GPP 规范条目来归类，命名要用规范实名，不要用博客里的口语：

- **SUCI 的加密依赖归属网络公钥。** TS 33.501 列出的参数是 "Home Network Public Key, Home Network Public Key Identifier, protection scheme"（[ETSI PDF](https://www.etsi.org/deliver/etsi_ts/133500_133599/133501/16.07.01_60/ts_133501v160701p.pdf)）。因此"标识加密"的安全前提里包含公钥真实性与标识符选择，不是纯本地算法问题。
- **鉴权锚点是 SEAF，不是 "SEAU"。** 后者不是 3GPP 术语（正确为 **Security Anchor Function, SEAF**）。EAP-AKA' 的规范实名见 RFC 5448（2009）与其现行版 RFC 9048（[rfc9048](https://www.rfc-editor.org/info/rfc9048/)，原话："While EAP-AKA' as defined in RFC 5448 is not obsolete, this document defines the most recent … specification of EAP-AKA'."）。**注意 RFC 7437 与 EAP-AKA' 无关，不要拿它当出处。**
- **短信侧**：TS 23.040 标题即 "Technical realization of the Short Message Service (SMS)"（含 SMS User Data Header，[ETSI PDF](https://www.etsi.org/deliver/etsi_ts/123000_123099/123040/17.02.00_60/ts_123040v170200p.pdf)）；IP 网上承载短信是 TS 24.341 "Support of SMS over IP networks"（[3GPP 页](https://www.3gpp.org/dynareport/24341.htm)）。历史上"SMS 解析"类高危缺陷正是落在这两条的实现层。

框架侧的权限与服务名同样是实名可查的（`core/res/AndroidManifest.xml`，[源码](https://android.googlesource.com/platform/frameworks/base/+/refs/heads/main/core/res/AndroidManifest.xml)）：

`READ_PHONE_STATE`、`READ_BASIC_PHONE_STATE`、`READ_PRECISE_PHONE_STATE`、`MODIFY_PHONE_STATE`、`READ_PHONE_NUMBERS`、`CALL_PHONE`、`ANSWER_PHONE_CALLS`、`SEND_SMS`/`RECEIVE_SMS`/`READ_SMS`、`RECEIVE_STK_COMMANDS`、`ACCESS_IMS_CALL_SERVICE`。另有 `CAPTURE_AUDIO_OUTPUT`、`CALL_AUDIO_INTERCEPTION`——**这两条签名/特权权限不属电话专用，讨论"通话音频被截取"时要用它们的真名**，而不是找一条"电话专用录音权限"。

服务命名：`Context.TELEPHONY_SERVICE = "phone"`、`TELEPHONY_REGISTRY_SERVICE = "telephony_registry"`；而 `dumpsys` 用的服务名是 `telephony.registry`（AOSP 里可见 `{"/system/bin/dumpsys", "telephony.registry", "EmergencyDiagnostics"}`）。三个名字不是一套，混用会导致排查失败。

## 4. 公告怎么命名基带条目：三家三种轴

这一节是实操价值最高的部分。**"Modem" 在不同厂商公告里不是同一个字段。**

| 厂商 | 字段 | 实名取值 | 出处 |
|---|---|---|---|
| 高通 | **Technology Area** | `Modem`、`LTE`（**没有 "MPM" 这个值**） | [Aug 2025 bulletin](https://docs.qualcomm.com/securitybulletin/august-2025-bulletin.html) |
| 联发科 | **Subcomponent** | `Modem`、`ccci` | [Oct 2026 PSB](https://www.mediatek.com/product-security-bulletin/october-2026?hsLang=en) |
| 三星半导体 | 描述文 + SMSB 命名 | "… in Modem"、"Improper input validation in IpcProtocol in Modem" | [SMSB 2024-12](https://security.samsungmobile.com/securityUpdate.smsb?year=2024&month=12) |
| Google Pixel | 组件表 | 子组件"调制解调器/Modem" | [Pixel 2026-09-01](https://source.android.com/docs/security/bulletin/pixel/2026/2026-09-01) |

所以做批量统计时，"搜 Technology Area == Modem" 只能覆盖高通；联发科要搜 Subcomponent，三星要读描述句。**跨厂商基带统计用单一字段过滤必然漏。**

分发与责任链在原文里也是明确的。高通："Patches are being actively shared with OEMs, who have been notified and strongly recommended to deploy those patches on released devices as soon as possible. Please contact the device manufacturer for information on the patching status"（同 Aug 2025 bulletin）。联发科："we are not aware of any active exploitation of these vulnerabilities in the wild"，并按 CVSS v3.1 评级（同 Oct 2026 PSB）。

**SPL 与基带的关系要谨慎表述。** Google 官方成文的只有 SPL 语义本身——Pixel 公告 FAQ："Security patch levels of 2026-09-05 or later address all issues associated with the 2026-09-05 security patch level"，而 AOSP 侧对厂商修复说 "available directly from the manufacturers"。**"SPL 是否单独标注基带固件版本"没有可引用的成文措辞**（未核实）。正确的防御表述是：基带固件版本可用 `gsm.version.baseband` 观测，且**必须走 OEM 自己的公告核对**，不能从 SPL 推断基带是否已修。

## 5. 已修复公开案例（逐条一手核实）

只列**公告/厂商页原文可核**的条目。严重级别、组件、年份均取自出处原文。

### 高通

| CVE | 出处原文要点 | 级别/字段 |
|---|---|---|
| CVE-2025-21477 | "Improper Input Validation in Modem"，Access Vector Remote | High / CVSS 7.5，Technology Area **Modem**，2025-08-04 公告 |
| CVE-2025-21452 | "Reachable Assertion in Modem" | Technology Area **LTE**，同上公告 |
| CVE-2025-47371 | "Reachable Assertion in Modem / Transient DOS when an LTE RLC packet with invalid TB is received by UE" | 2026-03-02 公告 |

第三条值得单独注意：厂商在公告里写了触发条件的**协议层描述**（LTE RLC 收到无效 TB 的 UE 侧处理），而结论是 **Transient DoS + 断言**。这是"可达断言"类基带缺陷的典型形态——影响是可用性，不是执行；把它写成"基带 RCE"属于误读。

### 联发科

| CVE | 出处原文要点 |
|---|---|
| CVE-2026-20519 / CVE-2026-20520 | Subcomponent **Modem**，Critical，CWE-787，"There is a possible out of bounds write due to a missing bounds check."（Oct 2026 PSB）；同公告另有 Subcomponent **ccci** 条目 |

### 三星（Exynos 基带）

| CVE | 出处原文要点 |
|---|---|
| CVE-2023-26073 | "A heap-based buffer overflow in the 5G MM message codec …"（Exynos 移动/基带芯片组）—[三星页](https://semiconductor.samsung.com/support/quality-support/product-security-updates/cve-2023-26073/)、[NVD](https://nvd.nist.gov/vuln/detail/cve-2023-26073) |
| CVE-2024-48883 | "The UE incorrectly handles the malformed uplink scheduling message, resulting in an information leak of the UE"，Medium，含 Modem 5123/5300 —[三星页](https://semiconductor.samsung.com/support/quality-support/product-security-updates/cve-2024-48883/) |
| CVE-2024-34673 | SMSB 命名 "Improper input validation in IpcProtocol in Modem" —[SMSB 2024-12](https://security.samsungmobile.com/securityUpdate.smsb?year=2024&month=12) |

CVE-2023-26073 是"5G 移动性管理消息编解码器堆溢出"，CVE-2024-48883 是"畸形上行调度消息导致 UE 信息泄露"——两条恰好展示了协议面缺陷的两个常见落点：**消息编解码器**与**状态机对异常消息的处理**。

### Project Zero 的暴露面结论（可引用的原话）

[Internet-to-baseband remote code execution](https://projectzero.google/2023/03/multiple-internet-to-baseband-remote-rce.html)：

> "allow an attacker to remotely compromise a phone at the baseband level with no user interaction"
> "The four most severe … allowed for Internet-to-baseband remote code execution."
> "In late 2022 and early 2023, Project Zero reported eighteen 0-day vulnerabilities in Exynos Modems produced by Samsung Semiconductor."

这三句是"基带是否需要用户交互"讨论的正规出处：**同一处理器、同一网络路径，无交互即可达**。引用时用"十八个 0-day（2022 末至 2023 初报告）"，不要写成"十余个"或别的数字。

### CISA KEV 核对（全量 JSON，逐条精确匹配）

KEV 里与"modem/baseband"字面相关的移动端条目只有 **3 条**（[KEV JSON](https://www.cisa.gov/sites/default/files/feeds/known_exploited_vulnerabilities.json)）：

| CVE | KEV 描述 | 加入日期 |
|---|---|---|
| CVE-2026-58704 | Google Pixel，"improper authorization vulnerability in the cellular modem"，EoP | 2026-09-16 |
| CVE-2021-25487 | Samsung，"within the modem interface driver" | 2023-06-29 |
| CVE-2021-25489 | Samsung，同上 | 2023-06-29 |

后两条是 **AP 侧内核驱动**（modem interface driver），不是基带固件本身——这个区分对定级和处置都重要。**高通/联发科的 modem 条目在 KEV 命中 0**，也没有任何蜂窝基带 0-click 远程条目进 KEV。跨厂商碰撞：CVE-2025-24990 是 "Windows Agere Modem Driver"，不可当基带引用。

### 近场协议（只做归属纠正，细节见图谱手册）

- **FragAttacks = 12 个 CVE**（不是 16），公开于 2021-05-11，全部属 **Wi‑Fi / 802.11**，与蜂窝基带无关：设计缺陷 CVE-2020-24586/-24587/-24588；实现缺陷 CVE-2020-26139…-26147。官网原话："every Wi-Fi product is affected by at least one vulnerability"（[fragattacks.com](https://www.fragattacks.com/)）。**CVE-2020-26137 不在该集合内**（其归属未核实）。
- **BlueFrag = CVE-2020-0022，2020 年 2 月，Google 归入 System**（蓝牙，不是内核也不是 Framework）：公告原文 "DoS Moderate 10; RCE Critical 8.0, 8.1, 9"（[2020-02-01](https://source.android.com/docs/security/bulletin/2020-02-01)），研究描述见 [insinuator](https://insinuator.net/2020/04/cve-2020-0022-an-android-8-0-9-0-bluetooth-zero-click-rce-bluefrag/)）。**不在 KEV**。所谓 `CVE-2025-32565` 与 BlueFrag 无关，是另一产品的 SQLi（[CVE 记录](https://www.cve.org/CVERecord?id=CVE-2025-32565)）。

## 6. 常见误传纠正

| 误传 | 实际 |
|---|---|
| "基带漏洞都在 ASB 里" | SoC 厂商修复走自有通道："available directly from the manufacturers"；ASB/Pixel 公告只覆盖 AOSP 与其设备侧 |
| AIDL RIL 是 `IRadio`/`ISetwork`/`IMods` | 无顶层 `IRadio.aidl`；实名 `IRadioModem`/`IRadioNetwork`/`IRadioSim`/`IRadioVoice`/`IRadioMessaging`/`IRadioData`/`IRadioIms`/`IRadioConfig`/`ISap` |
| 高通公告有 "MPM" 这个 Technology Area | 实名为 `Modem` 与 `LTE` |
| "SPL 更新了就代表基带已修" | 无成文依据；基带固件需用 `gsm.version.baseband` + OEM 公告单独核对 |
| FragAttacks 是 16 个 CVE、或涉及蜂窝 | 12 个，全属 Wi‑Fi/802.11 |
| BlueFrag 是内核/Framework 漏洞 | 归入 **System**，CVE-2020-0022 |
| QMI 的 Q 是 "Quick" | 内核注释实名 "Qualcomm MSM Interface" |
| SEAU 是 3GPP 术语 | 正确为 SEAF（Security Anchor Function） |
| EAP-AKA' 只有 RFC 5448 | RFC 9048 是现行规范；RFC 7437 与之无关 |
| Titan M 参与基带安全 | Titan M 是 Pixel 安全元件，不是基带；两者关联无可引用来源（未核实） |
| 基带断言类缺陷＝RCE | 公告可写为 Transient DoS（如 CVE-2025-47371）；影响要看原文 |
| 通话截音频有电话专用权限 | 用签名/特权权限 `CAPTURE_AUDIO_OUTPUT`、`CALL_AUDIO_INTERCEPTION` |

## 7. 观测点：一台设备上真正读得到的东西

**属性。** `PROPERTY_BASEBAND_VERSION = "gsm.version.baseband"`（[TelephonyProperties.java](https://android.googlesource.com/platform/frameworks/opt/telephony/+/refs/heads/main/telephony/java/com/android/internal/telephony/TelephonyProperties.java)），SELinux 类型为 `radio_prop`；另有 `gsm.version.ril-impl`。设置里对应项实名 BasebandVersionPreference。

```
adb shell getprop gsm.version.baseband
adb shell getprop gsm.version.ril-impl
```

未核实：具体由哪个进程写入 `gsm.version.baseband`（可确认它可读，不建议在无出处时断言写入方）。

**服务与调试组件。** 官方文档给出的一手命令是：

```
adb shell dumpsys activity service com.android.phone/com.android.phone.TelephonyDebugService
adb shell dumpsys telephony.registry
```

组件实名 `com.android.phone.TelephonyDebugService`，出处为 [Telephony time zone detection](https://source.android.com/docs/core/connect/time/telephony-tz-detection)（该页给出上面这条命令）。

**防御动作（只到可执行清单，不含实操）。**
1. 把设备 `gsm.version.baseband` 与 **SoC 厂商公告**（高通 docs.qualcomm.com / 联发科 PSB / 三星 SMSB 与半导体页）逐条对齐，而不是只对齐 ASB。
2. 建立"双轴"记录：Android SPL 一轴，基带固件版本一轴。二者不互相推断。
3. 组件归属统计跨厂商时分别用 Technology Area / Subcomponent / 描述句三种轴。
4. 用 KEV 做"是否已被在野利用"的判据时，注意 modem 类命中只有 3 条且两条属 AP 侧驱动——不要用它当基带风险的全集。
5. 远程可达性评估按 Project Zero 的结论口径：**基带路径可无用户交互到达**，因此优先级不应因"不是 Framework"而降档。
6. 对"信息泄露"类（如 CVE-2024-48883）注意其影响面在 UE 侧而非网络侧，处置上属固件更新，与 AP 侧权限模型无关。

## 8. 与其他手册的分工

| 需要 | 去看 |
|---|---|
| Binder、内核内存、驱动 ioctl 的通用缺陷形态 | [底层漏洞类型图谱](/android-low-level-vulnerability-types/) |
| 远程代码执行与提权的链条组装、上下文与影响半径 | [RCE 与提权手册](/android-rce-and-privilege-escalation/) |
| 公告字段读法、CVE 与 Android bug ID 对齐 | [编号与公告手册](/android-cve-and-security-bulletins/) |
| 严重级别与 CVSS 口径、处置优先级 | [评估与定级手册](/android-vulnerability-severity-assessment/) |
| Mainline 模块更新通道（与基带固件更新是两条不同通道） | [Mainline 与 APEX 手册](/android-mainline-apex-modules/) |
| 本文 | 基带/AP 边界、RIL 与宿主接口实名、三家公告命名轴、SPL 与基带关系、协议面规范实名 |

---

## 参考

- [ASB overview](https://source.android.com/docs/security/bulletin/asb-overview) · [Pixel 2026-09-01](https://source.android.com/docs/security/bulletin/pixel/2026/2026-09-01) · [2020-02-01](https://source.android.com/docs/security/bulletin/2020-02-01)
- [radio HAL 目录](https://android.googlesource.com/platform/hardware/interfaces/+/refs/heads/main/radio/) · [radio AIDL modem](https://android.googlesource.com/platform/hardware/interfaces/+/refs/heads/main/radio/aidl/android/hardware/radio/modem/) · [rild.c](https://android.googlesource.com/platform/hardware/ril/+/refs/heads/main/rild/rild.c)
- [qmi_wwan.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/plain/drivers/net/usb/qmi_wwan.c) · [mhi.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/plain/include/linux/mhi.h) · [bam_dma.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/plain/drivers/dma/qcom/bam_dma.c)
- [高通 2025-08 公告](https://docs.qualcomm.com/securitybulletin/august-2025-bulletin.html) · [高通 2026-03 公告](https://docs.qualcomm.com/securitybulletin/march-2026-bulletin.html) · [联发科 2026-10 PSB](https://www.mediatek.com/product-security-bulletin/october-2026?hsLang=en) · [三星 SMSB 2024-12](https://security.samsungmobile.com/securityUpdate.smsb?year=2024&month=12) · [CVE-2023-26073](https://semiconductor.samsung.com/support/quality-support/product-security-updates/cve-2023-26073/) · [CVE-2024-48883](https://semiconductor.samsung.com/support/quality-support/product-security-updates/cve-2024-48883/)
- [Project Zero: Internet-to-baseband RCE](https://projectzero.google/2023/03/multiple-internet-to-baseband-remote-rce.html) · [FragAttacks](https://www.fragattacks.com/) · [BlueFrag 分析](https://insinuator.net/2020/04/cve-2020-0022-an-android-8-0-9-0-bluetooth-zero-click-rce-bluefrag/)
- [TS 33.501](https://www.etsi.org/deliver/etsi_ts/133500_133599/133501/16.07.01_60/ts_133501v160701p.pdf) · [TS 23.040](https://www.etsi.org/deliver/etsi_ts/123000_123099/123040/17.02.00_60/ts_123040v170200p.pdf) · [TS 24.341](https://www.3gpp.org/dynareport/24341.htm) · [RFC 9048](https://www.rfc-editor.org/info/rfc9048/)
- [TelephonyProperties.java](https://android.googlesource.com/platform/frameworks/opt/telephony/+/refs/heads/main/telephony/java/com/android/internal/telephony/TelephonyProperties.java) · [AndroidManifest 权限](https://android.googlesource.com/platform/frameworks/base/+/refs/heads/main/core/res/AndroidManifest.xml) · [Telephony 时区检测（TelephonyDebugService）](https://source.android.com/docs/core/connect/time/telephony-tz-detection)
- [CISA KEV JSON](https://www.cisa.gov/sites/default/files/feeds/known_exploited_vulnerabilities.json) · [KEV 精确检索](https://www.cisa.gov/known-exploited-vulnerabilities-catalog?field_cve=CVE-2026-58704)

## 9. 未核实与失效来源

未核实（刻意不写入正文结论）：MCM 的厂商展开；SMD/BAM 缩写的逐字展开；`CCCI` 缩写展开；`CVE-2020-26137` 真实归属；`gsm.version.baseband` 的写入方；Titan M 与基带的任何关联；"SPL 是否单独覆盖基带固件"的 Google 显式措辞；TS 33.501 中 SUCI 公钥下发流程的逐字条款（仅核到术语行）。

来源失效（本手册核对时实测）：`source.android.com/docs/characters/input/ril-hal`（404）；`source.android.com/docs/security/bulletin/2025-01-05`（404）；`docs.kernel.org/driver-api/mhi.html`（404）；`drivers/net/wwan/qmi_wwan.c`（404，正确路径为 `drivers/net/usb/`）；`drivers/soc/qcom/smd.c`、`net/qrtr/qrtr.c`、`drivers/bus/mhi/core/init.c`（404）；`fragattacks.com/research/`、`kb.cert.org/vuls/id/813790`（404）；`mediatek.com/legal/security-advisory`（抓取失败）。

---

## 联系方式

咨询 Android 系统详情请联系 https://t.me/one00190

---

*本手册仅供安全学习与防御研究使用；所有接口名、公告字段取值与案例归属均取自公开一手来源并附链接，未能核实的表述已单列于 §9 而非写入正文。*
