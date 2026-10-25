---
layout: page
title: "Android 远程代码执行与权限提升专题手册"
description: "执行上下文与影响半径、提权三段式、RCE 与 EoP 根因模式、链条组装与缓解矩阵、17 条公开案例复盘、自查检测与处置优先级"
date: 2026-10-24
permalink: /android-rce-and-privilege-escalation/
---

> 版本：v1.0 · 2026-10 · 视角：防御与教育 · 案例均来自公开且已修复的漏洞记录
>
> 配套文档：[Android 系统漏洞安全研究报告](/android-system-vulnerability-report/) · [Android 通用系统漏洞类型手册](/android-common-system-vulnerabilities/) · [Android 漏洞编号与公告体系手册](/android-cve-and-security-bulletins/)

## 0. 使用方式与边界

Android 月度安全公告里的类型缩写只有四个，但真正决定风险量级的是其中两个：**RCE（远程代码执行）与 EoP（权限提升）**。这份手册把这两类单独拆开讲清楚：它们各自在哪些执行上下文里发生、根因模式有哪些、防御措施分别阻断哪一环、以及如何在设备与企业侧发现异常。

- 只讨论**原理、识别与缓解**，不含利用代码、偏移、绕过步骤等可操作细节
- 第 7 节的案例全部来自 CISA KEV（已知被利用漏洞目录）与 CVE 公开记录，可直接回溯核对
- 结论偏保守：单点漏洞很少独立构成危害，**判风险要看它落在链条的哪一环**

## 1. 先把两个词说准确

| | RCE 远程代码执行 | EoP 权限提升 |
| --- | --- | --- |
| 攻击者得到什么 | 在目标进程内执行任意代码 | 在同一进程内获得更高权限/更多能力 |
| 关键区别 | **入口**问题：代码执行能力从哪来 | **高度**问题：执行能力被抬到哪个层级 |
| 触发面 | 通常是远程可达的解析器、协议栈、渲染栈；也可能由本地输入触发 | 几乎总是本地：已安装应用、辅助功能、Near-field UI、调试接口 |
| 公告里的表现 | `RCE`，多数为「无需用户交互」或「需用户交互」的远程路径 | `EoP`，描述常见「local escalation of privilege with no additional execution privileges needed」 |
| 单独危害 | 若落在受限进程（如带严格 seccomp 的沙箱内），落地后仍受限 | 单点提权通常不足以拿到数据，需要与入口漏洞组合 |
| 组合威力 | RCE（入口）+ EoP（抬升）= 完整链条 | 提权链的最上段一般是内核或系统服务 |

一句判断口诀：**RCE 决定「能不能进来」，EoP 决定「进来后能站多高」**。两者都命中且能连起来，才是真正意义上的设备沦陷。

一个常见误读：公告把某条漏洞标为 `EoP`，不代表它只能被本地恶意应用触发——`CVE-2024-36971`（Linux 内核网络路径 UAF）在 KEV 里被标为 **Android Kernel Remote Code Execution**，同一段内存破坏在不同可达性下会同时具备 RCE 与 EoP 属性。所以**类型列只是分类标签，可达性要回到描述文本读**。

## 2. 执行上下文决定影响半径

同样一次「代码执行」，落在不同进程里意味着完全不同的后果。Android 通过 UID + SELinux 域 + seccomp 策略把系统服务切分到不同上下文：

| 执行上下文 | 典型身份 | 被攻陷后能获得 | 现有约束 |
| --- | --- | --- | --- |
| 应用进程 | 独立 UID，`untrusted_app` 域，严格 seccomp | 仅本应用数据；需继续提权才有价值 | 沙箱、Mount namespace、Scoped Storage、随机化 |
| 媒体/相机等原生服务 | `media.extractor`、`media.codec`、`cameraserver` 等独立进程 | 解析到的媒体管线权限；历史上 `mediaserver` 权限过大 | 拆分进程 + 各自 SELinux 域 + seccomp；这是 Stagefright 之后的主要整改方向 |
| 系统服务 `system_server` | `system` UID，Java/ART 侧 | 几乎所有平台能力：权限授予、界面、账号、设置项 | SELinux 限制其可访问的文件/服务；Binder 调用需权限校验 |
| SurfaceFlinger / 图形栈 | `system` 相关域 | 截屏、输入提示（tapjacking）、界面伪造能力 | 独立域、GPU 驱动侧沙箱 |
| 无线协议栈 | 蓝牙 / NFC / 电话（`bluetooth`、`nfc`、`radio` 域） | 近场远程入口；跨设备传播能力 | 协议栈沙箱化、运行时权限、最小化自动响应 |
| 密钥与认证 | `keystore2`、`gatekeeperd`、`credstore`，后端在 TEE | 密钥使用能力（不一定能导出）、认证绕过 | KeyMint/StrongBox 硬件隔离、Key Attestation |
| 内核与驱动 | kernel 态 | 关闭 SELinux/校验、任意进程内存读写、持久化 | KASLR、CFI/SCS、PAN、KPTI、模块限制、IOMMU |
| TEE / 基带 / SE | 独立执行环境 | 绕过 HLOS 全部缓解措施 | 与主系统隔离；由独立固件公告覆盖 |

**要点**：
1. 一个只落在 `media.codec` 里的 RCE，与一个落在 `system_server` 里的 RCE，修复优先级不应相同——后者离「平台能力」更近。
2. 内核与驱动类漏洞几乎都是 **EoP**：拿到任意读写后，最直接的用途是关掉 SELinux  enforcement 或篡改凭据，而不是「远程执行」。
3. TEE/基带被攻破属于另一套风险模型，通常由厂商固件公告单独覆盖，不要拿 AOSP 的 SPL 去推断它是否已修。

## 3. 提权的三段式：边界在哪里

```
第 1 段  应用侧立足        边界：UID 沙箱 + 导出组件权限校验 + AppOps
        ↓ 需要：一个可被外部调用的入口（导出组件、Deep Link、通知/锁屏 UI、辅助功能）
第 2 段  平台侧抬升        边界：SELinux 域 + Binder 权限检查 + 签名/特权权限
        ↓ 需要：系统服务逻辑缺陷 或 合法高危授权被滥用
第 3 段  内核/持久化       边界：内核内存安全 + AVB/dm-verity + 回滚保护
        结果：关闭校验、修改系统分区、驻留
```

每一段都对应一类高频根因：

- **第 1 段常见**：导出组件缺少权限校验、 PendingIntent 可变性、任务栈/Activity 劫持（StrandHogg 2.0 一类）、Deep Link 未校验来源、`addJavascriptInterface` 与 WebView 文件域访问。
- **第 2 段常见**：系统服务权限判定遗漏或判定对象错误、后台拉起 Activity/界面伪造、设备管理员与 Profile Owner 的授予逻辑缺陷、文件路径过滤器可被绕过、通知/锁屏界面在解锁前暴露能力。
- **第 3 段常见**：Binder 与驱动 UAF、越界写、竞态、引用计数溢出；页缓存/管道写入类内核逻辑缺陷；DMA 接口缺少 IOMMU 约束。

**为什么链条常从第 2 段而不是第 3 段完成**：现代内核缓解措施让第 3 段成本高，而平台层的**逻辑缺陷**不需要内存破坏就能拿到 system 能力，且很多 OEM 定制服务在这一层暴露更多接口。公开案例里，Framework 组件的 EoP 数量长期高于内核。

## 4. RCE：根因模式与观察点

每一类给出：触发前提 → 公开案例（均已修复）→ 缓解 → 可观测信号。

### 4.1 媒体与容器解析

- **前提**：目标进程解析攻击者可控的文件/流；历史上可由 MMS 自动拉取触发，无需用户交互。
- **案例**：Stagefright 系列（2015，`libStagefright` 解析特制 MP4）。后续整改方向是把单体 `mediaserver` 拆成 `media.extractor`、`media.codec` 等独立沙箱进程并收紧 SELinux 域。
- **缓解**：进程拆分 + seccomp 白名单、解析器内存安全重写（Rust/C++ 边界加固）、禁用自动获取与自动解析、模糊测试持续覆盖。
- **信号**：媒体相关进程异常重启与 tombstone；`logcat` 中出现 codec/extractor 崩溃；下载目录出现来源不明的媒体文件。

### 4.2 图像、字体、文档与压缩格式

- **前提**：应用或系统组件渲染不可信内容（浏览器、邮件、消息类应用、图库）。
- **典型缺陷**：越界读写、整数溢出导致的尺寸计算错误、解压路径拼接（目录穿越）与符号链接处理。
- **缓解**：渲染沙箱（进程隔离 + 低权限域）、限制 `file://` 与混合内容、对 WebView 桥接做白名单、解压时校验目标路径。
- **信号**：渲染进程反复崩溃；WebView 控制台异常脚本；渲染后出现的越权文件访问记录。

### 4.3 无线与近场协议栈

- **前提**：协议栈在用户无感知的情况下处理对端数据（广播、配对前的连接请求、握手）。
- **案例**：BlueBorne 系列（2017，Linux 蓝牙栈），无需配对即可攻击；此类「近场可达」把攻击窗口压缩到几十秒。
- **缓解**：不使用时关闭蓝牙/Wi-Fi 扫描与自动连接、协议栈沙箱化、厂商按月修补链路层实现、避免自动响应未知设备。
- **信号**：不明来源的连接尝试日志、协议栈进程异常重启、电量与射频活动异常。

### 4.4 WebView、JS 桥与动态代码

- **前提**：应用把 Java 对象暴露给可被不可信页面调用的接口，或从远端下载 dex/so 后加载。
- **典型缺陷**：`addJavascriptInterface` 未加 `@JavascriptInterface` 边界控制、加载 `file://` 造成跨域读取、动态加载目录可写、依赖第三方 SDK 引入的执行面。
- **缓解**：只暴露最小接口且校验来源、`setAllowFileAccess(false)`、启用网络安全配置、动态代码走 Play Feature Delivery 之类受控通道、对 SDK 做持续审计。
- **信号**：应用私有目录中出现可执行文件、`/data/local/tmp` 异常写入、应用请求与其功能不符的权限组合。

### 4.5 IPC 反序列化与类型混淆

- **前提**：跨进程数据（Parcel、共享内存、序列化对象）由不可信来源构造，但被当成可信结构消费。
- **典型缺陷**：读写顺序不匹配、长度字段未校验、句柄与对象索引混用；一旦落在特权服务里就同时具备 RCE 与 EoP 属性。
- **缓解**：接口收敛（`@UnsupportedAppUsage`、只读校验）、在 Binder 线程做调用方身份校验、限制导出组件、FIDL/Rust 化关键接口。
- **信号**：系统服务崩溃并重启（`system_server` 重启是强信号）、`ServiceManager` 相关错误日志激增。

### 4.6 渲染与图形栈、内核任意写转化

- **前提**：GPU 驱动、shader 编译或内核对象生命周期缺陷，被外部输入触发。
- **落点**：多数公告标为 EoP，但具备**任意写**能力的内核缺陷实际可转化为代码执行（例如 `CVE-2024-36971` 这类内核 UAF，KEV 直接标为 Remote Code Execution）。
- **缓解**：图形栈沙箱化、内核缓解措施（CFI/SCS/PAN/KASLR/KPTI、MTE/PAC）、及时跟进内核 LTS、限制用户态可访问的驱动接口。
- **信号**：内核日志（`dmesg`）中的引用计数/UAF 类告警、图形服务异常重启、随机重启现象。

## 5. EoP：根因模式与观察点

### 5.1 Binder 与驱动内存缺陷

**案例**：`CVE-2019-2215`（`binder.c` 中 `binder_transaction` 相关 UAF，应用可提权至内核，无需用户交互但需恶意本地应用或另一处漏洞配合）、`CVE-2020-0041`（同一位置因边界检查错误导致越界写）。两者都进入 KEV，属于真实被使用过的内核提权缺陷。
**缓解**：驱动接口最小化与参数校验、内核缓解措施使「越界写→稳定提权」成本大幅上升、限制可打开的设备节点（SELinux + seccomp）。
**信号**：内核态崩溃日志、非常规系统调用模式、异常常驻高权限进程。

### 5.2 系统服务权限判定遗漏

**特征**：能力本身合理，但对「谁可以调用」判定不足或判定对象错误。公开描述里最常见的一句是「`possible way to launch activities from the background due to a permissions bypass`」（`CVE-2025-48572`，Framework），以及 `CVE-2023-35674` 的「`WindowState.java` 中 `onCreate` 允许后台拉起 Activity」。
**缓解**：接口层强制调用方身份与权限校验、后台启动 Activity 的默认限制、把高危能力收敛到需用户确认的路径、持续审计 OEM 自有服务。
**信号**：应用从未获得前台却出现界面跳转、锁屏状态下的异常界面、`dumpsys activity` 中出现异常任务栈。

### 5.3 任务栈与界面劫持

**特征**：伪造「用户正在使用的应用」的观感，诱导授权或输入；StrandHogg 2.0（`CVE-2020-0096`）是这一类的代表，公告侧类型多为 EoP，实际用途是钓鱼与授权绕过。
**缓解**：任务栈归属校验、`TaskFragment`/启动来源检查、权限弹窗与生物识别绑定的来源确认、对「覆盖界面」类授权做超时与可见提示。
**信号**：`Settings.canDrawOverlays()` 结果为真但用户未授权、异常的前台应用切换记录。

### 5.4 设备管理员 / Profile Owner / 辅助功能

**特征**：不是内存破坏，而是**授权机制被绕过或被合法滥用**。`CVE-2025-48633` 的公开描述即「`DevicePolicyManagerService.java` 的 `hasAccountsOnAnyUser` 存在逻辑错误，可能在设备配置完成后添加 Device Owner」，类型 EoP，被 KEV 收录。
**缓解**：Device Owner 授予路径的时序校验、限制 provisioning 之后的授予能力、企业侧关闭「允许用户自行 enroll」、定期清点设备管理员与辅助功能列表。
**信号**：`adb shell dumpsys device_policy` 中出现未知 Device Owner / Profile Owner；`settings get secure enabled_accessibility_services` 非空且与用户认知不符；通知读取权限被新增。

### 5.5 文件、URI 与路径过滤绕过

**特征**：过滤器与真实路径解析之间存在语义差异。`CVE-2024-43093` 的描述是「`ExternalStorageProvider.java` 的 `shouldHideDocument` 因 Unicode 归一化问题绕过路径过滤」，被 KEV 收录为 Framework EoP。
**缓解**：统一使用规范化的真实路径比较、对编码/Unicode 变体做归一化后再判断、ContentProvider 只暴露白名单目录、Scoped Storage 优先。
**信号**：应用出现异常的外部存储访问记录、文件选择器中出现敏感目录内容。

### 5.6 整数溢出与长度计算导致的越界

**特征**：尺寸计算错误造成的内存破坏，落点决定类型。`CVE-2025-48595` 的公开描述同时具备两种属性：「`possible way to achieve code execution due to an integer overflow` … `local escalation of privilege`」，KEV 以 Integer Overflow 收录（2026-06）。
**缓解**：边界运算使用安全类型与显式校验、关键解析器改用内存安全语言、编译期加固（FORTIFY、栈保护）、模糊测试覆盖数值边界。
**信号**：解析不可信输入时的崩溃聚类、异常大的尺寸字段日志。

### 5.7 SELinux 策略与工程接口遗留

**特征**：策略错误（过早 `permissive`、错误标签继承、未清理的调试域）或量产版本残留工程模式，让一次本地立足直接跳过第 3 段。
**缓解**：策略集中评审与最小化、`neverallow` 约束、量产前移除调试接口与工程 APK、以 `enforcing` 为发布门槛。
**信号**：`getenforce` 返回非 `Enforcing`；日志中大量 `avc: denied` 或突然完全消失（被 permissive 掩盖）；`ps -A -Z` 出现预期外的域。

### 5.8 合法高危授权的「等效提权」

无障碍服务、通知监听、设备管理器、`SYSTEM_ALERT_WINDOW`、被授予的签名级权限，以及应用内自建 JS 桥的过度授权，都不属于「漏洞」，但攻击效果与 EoP 重叠。企业巡检时这部分往往比 CVE 列表更能解释真实异常。

## 6. 链条如何组装，缓解分别拦在哪

典型完整链条（与研究报告第 6 节一致，这里标出每一环的拦截手段）：

| 环节 | 攻击者需要什么 | 主要拦截手段 |
| --- | --- | --- |
| 入口 | 一个可达的解析/协议/组件 | 关闭自动获取与自动解析、收敛导出组件、协议栈沙箱 |
| 落地执行 | 在目标进程内执行 | seccomp 系统调用白名单、SELinux 域限制、进程拆分 |
| 信息泄露 | 绕过随机化 | KASLR、PAC/MTE、只读内存布局、日志与调试接口清理 |
| 平台提权 | system 能力 | 服务侧权限校验、后台启动限制、授权需用户确认 |
| 内核提权 | 任意读写 | 内核缓解措施（CFI/SCS/PAN/KPTI）、驱动接口最小化、及时跟进 LTS |
| 持久化 | 改分区/关校验 | AVB + dm-verity、回滚保护、签名校验、解锁状态可被服务端识别 |
| 交付控制 | 长期驻留与外传 | Play Protect、网络策略、MDM 基线与远程擦除、完整性校验 |

**观察**：2015 年之前一条媒体解析 RCE 就能直接威胁用户；如今几乎都需要「入口 + 提权」两到三环组合，原因是中间几层的沙箱与完整性措施被持续加固。也因此，**在野利用更集中在 Framework 逻辑缺陷与厂商/固件组件**——这些位置的加固程度低于内核与解析器。

## 7. 案例复盘（公开记录，均已修复）

下表内容取自 CISA KEV 条目与 CVE 公开记录，可按编号回溯原文。

| CVE | 组件/位置 | 公开描述要点 | 类型 | KEV 收录 |
| --- | --- | --- | --- | --- |
| CVE-2025-48595 | Framework | 整数溢出导致可获得代码执行，进而本地提权 | RCE + EoP | 2026-06-02 |
| CVE-2025-48633 | Framework（`DevicePolicyManagerService`） | 配置完成后可被添加为 Device Owner 的逻辑错误 | EoP | 2025-12-02 |
| CVE-2025-48572 | Framework | 权限绕过导致后台拉起 Activity | EoP | 2025-12-02 |
| CVE-2025-48543 | Android Runtime | UAF 使 Chrome 沙箱可逃逸并攻击 `system_server` | EoP | 2025-09-04 |
| CVE-2024-43093 | Framework（`ExternalStorageProvider`） | Unicode 归一化问题绕过文件路径过滤 | EoP | 2024-11-07 |
| CVE-2024-36971 | Kernel（网络路径） | `__dst_negative_advice()` 竞态导致 UAF | RCE（KEV 标注） | 2024-08-07 |
| CVE-2024-32896 | Pixel 固件 | 逻辑错误导致绕过，需用户交互 | EoP | 2024-06-13 |
| CVE-2024-29748 | Pixel 固件 | 权限处理不当导致绕过，需用户交互 | EoP | 2024-04-04 |
| CVE-2023-35674 | Framework（`WindowState`） | 逻辑错误导致后台拉起 Activity | EoP | 2023-09-13 |
| CVE-2023-20963 | Framework | 提权类缺陷（KEV 标注 Privilege Escalation） | EoP | 2023-04-13 |
| CVE-2021-1048 | Kernel（`eventpoll.c`） | UAF 破坏内存，无需额外执行权限即可本地提权 | EoP | 2022-05-23 |
| CVE-2020-0041 | Kernel（`binder.c`） | 边界检查错误导致越界写 | EoP | 2021-11-03 |
| CVE-2019-2215 | Kernel（`binder.c`） | UAF，应用可提权至内核；需恶意本地应用或另一漏洞配合 | EoP | 2021-11-03 |
| Stagefright 系列（2015） | 媒体框架 | 特制 MP4 经 MMS 触发，`mediaserver` 权限过大 | RCE | —（机制推动者） |
| BlueBorne 系列（2017） | Linux 蓝牙栈 | 无需配对即可近场攻击 | RCE | — |
| StrandHogg 2.0（`CVE-2020-0096`） | Activity/任务栈管理 | 任务劫持实施钓鱼与授权绕过 | EoP | — |
| Dirty Pipe（`CVE-2022-0847`） | 内核 `pipe` | 页缓存改写导致提权 | EoP | — |

**三条读法要点**

1. **KEV 里的 Android 条目以 Framework/Kernel EoP 为主**，纯远程代码执行很少——这与「远程攻击面被收敛、在野利用走本地链条」的判断一致。
2. **2023 年之后被利用的编号集中在 Framework 与厂商固件**，而不是内核内存破坏：加固措施改变了攻击者的成本分布。
3. **一条 CVE 的描述往往同时提到两种能力**（如 `CVE-2025-48595`：代码执行 + 本地提权），分级时按最坏落点判断，不要被单一类型列限制。

## 8. 自查与检测

### 8.1 设备侧可核对项

| 目标 | 手段 |
| --- | --- |
| 补丁基线 | `adb shell getprop ro.build.version.security_patch`，与当月公告 SPL 对比 |
| 完整性状态 | `getprop ro.boot.verifiedbootstate`、`getprop ro.boot.flash.locked`、`getprop ro.boot.veritymode` |
| 强制访问控制 | `adb shell getenforce`（应为 `Enforcing`）；日志中 `avc: denied` 的出现与缺失都要看 |
| 进程与域 | `adb shell ps -A -Z`，核对是否存在预期外的高权限域或常驻进程 |
| 授权滥用 | `dumpsys device_policy`（Device Owner/Profile Owner）、`settings get secure enabled_accessibility_services`、设备管理员与通知监听列表 |
| 崩溃落地 | `/data/tombstones`、`logcat -b crash`：媒体/图形/协议栈进程反复崩溃是强信号 |
| 内核异常 | `dmesg` 中 UAF、引用计数、越界访问类告警与随机重启记录 |
| Mainline 模块 | 设置中的「Google Play 系统更新」/模块版本，与公告「Google Play 系统更新」段落对比 |

### 8.2 企业侧建议采集的指标

- 每台设备的 SPL、解锁状态、`getenforce` 结果、Android 版本与厂商构建号
- Device Owner / Profile Owner 列表、辅助功能启用列表、通知读取权限列表的变化历史
- 关键系统进程（`system_server`、图形、蓝牙、媒体）崩溃频率异常
- Play Integrity 判定结果与其评估方式（用于识别绕过风险），异常设备触发复查
- 高敏感场景中「长期未更新 + Framework 高危条目未修」的组合设备清单

### 8.3 判读注意

1. **`avc: denied` 突然消失**通常比出现更糟——可能意味着相关域被放宽。
2. 崩溃日志只能证明「有异常输入被处理」，不能直接证明被利用；要与 SPL 落后项、授权变化、外联行为共同判读。
3. 用户报告的「后台自启界面」「锁屏出现异常弹窗」是 5.2/5.3 类缺陷的典型外部表现，值得单独建一条上报通道。

## 9. 处置优先级建议

| 条件 | 建议窗口 |
| --- | --- |
| KEV 收录 + 组件在自家产品中存在 + 设备 SPL 早于公告 | 立即纳入紧急补丁流程（≤7 天），并核查是否已有异常迹象 |
| 公告级别「严重」，或描述含远程可达且无需用户交互 | 7–14 天 |
| Framework EoP，需本地恶意应用触发 | 下一个常规更新周期，同时评估与已知入口的组合风险 |
| 仅 DoS / 仅信息泄露且无绕过价值 | 排期修复，纳入季度评审 |
| 仅 OEM 私有服务 / 预装应用可触发 | 由厂商侧确认后单独跟踪，勿套用 AOSP 结论 |

补充两条经验规则：**同一组件短期内反复出现同类缺陷时，优先做架构级整改（拆分进程、重写解析器），而不是按月追补丁**；**内核类漏洞的修复验证要看厂商 SPL 追加项（`YYYY-MM-DD`），只看月度 `YYYY-MM` 容易漏判**。

## 10. 分角色清单

**设备厂商 / ROM 维护者**

- 建立「入口组件清单」：所有可被不可信输入触达的解析器与协议栈，标注其 SELinux 域、seccomp 策略与可访问接口
- 对 Framework 与自有服务强制调用方身份校验评审；后台拉起界面、Device Owner 授予等路径要有时序与来源校验
- 内核跟进 LTS 与厂商补丁；量产前清理工程模式、调试域与 `permissive` 残留
- 持续模糊测试覆盖 4.1–4.6，并把 tombstone 聚类结果回灌为测试用例

**应用开发者**

- 显式声明 `android:exported`，不导出即关闭；对导出组件做权限与来源校验
- WebView：不用 `addJavascriptInterface` 暴露敏感对象，禁用 `file://` 与混合内容，校验加载域名
- 不做动态下载执行；依赖项与第三方 SDK 定期审计，关注其 IPC 与解析接口
- 密钥走 Keystore/KeyMint，避免可导出的凭据；敏感操作绑定用户确认与来源确认

**企业与机构**

- 把 SPL 与解锁状态纳入 MDM 合规基线，设定关键漏洞 ≤14 天的修复 SLA
- 定期清点 Device Owner、Profile Owner、辅助功能与通知监听权限；禁止用户自助 enroll
- 对高敏感岗位设备启用完整性校验，异常设备走隔离与取证流程
- 建立「异常界面/后台拉起」类用户上报通道，弥补日志侧难以取证的问题

**个人用户**

- 保持月度更新，重点看 SPL 是否在推进，而不是系统版本号
- 不随意解锁 Bootloader；解锁会显著削弱完整性保护
- 不安装来源不明应用，谨慎授权无障碍与「显示在其他应用上层」
- 不再使用的设备：恢复出厂并注销账号；老旧无补丁设备避免用于支付与身份类操作

**安全研究者 / 应急响应**

- 分析时按「入口 → 落地限制 → 抬升路径 → 持久化」分段定位，不要只看单条 CVE 的级别
- 记录证据时保留 SPL、构建号、SELinux 状态与授权列表，便于判断被利用的是哪一环
- 面向未授权系统的测试一律不做；仅在自有设备或获书面授权的环境中验证

## 11. 附录

### 附录 A：一张速查表

| 我看到 | 第一判断 | 第二动作 |
| --- | --- | --- |
| 公告类型 `RCE` + 无需用户交互 | 入口级风险，最高优先 | 确认解析/协议组件在自家产品中是否可达 |
| 公告类型 `EoP` + 本地触发 | 需与入口组合 | 检查自家导出组件与授权路径是否提供第 1 段立足点 |
| KEV 收录 | 已有真实利用 | 立即核对 SPL，启动紧急流程并做异常排查 |
| 组件为 Framework/`system_server` | 离平台能力近 | 优先于同类内核条目处理 |
| 描述含「logic error」「permissions bypass」 | 逻辑缺陷，加固措施拦不住 | 审查判定分支与来源校验，补回归测试 |
| 描述含 UAF / 越界写 | 内存破坏 | 确认内核缓解措施是否开启，评估修复覆盖 |
| 仅 OEM 条目 | 与 AOSP 结论无关 | 走厂商公告与 SPL 追加项核对 |

### 附录 B：常见 CWE 与本文模式的对应

| 根因模式 | 常见 CWE |
| --- | --- |
| 解析器越界读写 | CWE-125 / CWE-787 |
| 长度与尺寸计算错误 | CWE-190 / CWE-680 |
| 对象生命周期错误 | CWE-416 |
| 竞态 | CWE-362 / CWE-367 |
| 权限判定缺失或错误 | CWE-862 / CWE-280 / CWE-306 |
| 路径与 URI 校验不足 | CWE-22 / CWE-176 |
| 签名与凭据校验缺陷 | CWE-287 / CWE-347 |
| 界面伪造与劫持 | CWE-451 / CWE-1022 |

### 附录 C：公开资料

- Android 安全公告索引：<https://source.android.com/docs/security/bulletin>
- Android 安全概览与缓解措施：<https://source.android.com/docs/security>
- CISA 已知被利用漏洞目录（KEV）：<https://www.cisa.gov/known-exploited-vulnerabilities-catalog>
- OSV Android 数据：<https://osv.dev/list?ecosystem=Android>
- CVE 记录镜像：<https://github.com/CVEProject/cvelistV5>
- Android 开源项目代码库：<https://android.googlesource.com/>
- OWASP MASTG / MASVS：<https://mas.owasp.org/>
- Google Android 漏洞赏金计划：<https://bughunter.withgoogle.com/>

## 联系方式

咨询 Android 系统详情请联系 https://t.me/one00190

---

*本手册仅用于安全学习与防御建设，案例均来自公开记录且已修复。文中不含任何可直接用于攻击的实现细节，请勿将相关内容用于未授权测试。*
