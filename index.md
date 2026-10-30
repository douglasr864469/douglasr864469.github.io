---
layout: page
title: "Android 系统漏洞安全研究资料"
description: "从防御视角梳理 Android 系统攻击面、漏洞类型、典型案例、编号与公告体系、底层架构与研究方法的公开研究资料，含一份总览报告、二十四份专题手册与一篇底层漏洞研究文章。"
permalink: /
---

> 全部文章可免费阅读；Markdown 源文件见 [https://github.com/douglasr864469/android-security-report](https://github.com/douglasr864469/android-security-report)，同一份资料同步发布在 GitLab 与 Bitbucket。

从防御视角梳理 Android 系统攻击面、漏洞类型、典型案例、编号与公告体系、底层架构与研究方法的公开研究资料，含一份总览报告、二十四份专题手册与一篇底层漏洞研究文章。

## 内容

- **总览报告**：[android-system-vulnerability-report.md](/android-system-vulnerability-report/)
  启动链与信任模型、攻击面分析、漏洞类型、8 个已修复典型案例、利用链与缓解、防御建议
- **系统底层漏洞研究文章**：[android-low-level-vulnerability-research.md](/android-low-level-vulnerability-research/)
  分层架构与安全边界、四类底层漏洞成因（内存安全／权限边界／Binder／设备驱动）、源码审计与模糊测试等研究方法、复现与影响评估要求、防护机制与研究挑战
- **底层漏洞类型图谱手册**：[android-low-level-vulnerability-types.md](/android-low-level-vulnerability-types/)
  按组件定位缺陷坐标：内核内存与用户态交换、驱动与 ioctl、ashmem/memfd 与 ION/dma-buf heaps、Binder 三域与 UAPI 接口面、图形与媒体 HAL、无线与近场协议、TEE 与配置性暴露；含 18 条已核验归属的公开案例表与 9 条常见误传纠正
- **通用漏洞类型手册**：[android-common-system-vulnerabilities.md](/android-common-system-vulnerabilities/)
  分层攻击面速览、11 类缺陷清单（识别特征与缓解）、自查观测点与判读、分角色行动清单、类型速查表
- **漏洞编号与公告体系手册**：[android-cve-and-security-bulletins.md](/android-cve-and-security-bulletins/)
  CVE / Android bug ID / OSV / CWE / 厂商编号的对应关系、SPL 机制、Android 与厂商公告分层结构、逐字段读公告、查询与核对实操
- **远程代码执行与权限提升专题手册**：[android-rce-and-privilege-escalation.md](/android-rce-and-privilege-escalation/)
  执行上下文与影响半径、提权三段式、RCE 与 EoP 根因模式、链条组装与缓解矩阵、17 条公开案例复盘、自查检测与处置优先级
- **安全性评估与定级手册**：[android-vulnerability-severity-assessment.md](/android-vulnerability-severity-assessment/)
  Android 严重级别与 CVSS 3.1/4.0 口径对照、安全上下文阶梯与级别修饰符、OSV 与 CISA KEV 字段读法、可利用性要素清单、P0–P3 处置矩阵、设备暴露面自评与加固基线、10 条常见误读
- **TEE、密钥管理与设备认证手册**：[android-tee-key-attestation.md](/android-tee-key-attestation/)
  沿「信任根 → 密钥存储 → 凭据认证 → 状态证明 → 存储加密」一条链展开：AVB 2.0 与四色状态、Rollback Protection、KeyMint 三档安全等级与 StrongBox 要求、Gatekeeper 与 HardwareAuthToken 实名、密钥证明的 RootOfTrust ASN.1 结构与 verifiedBootHash 用法、FBE 的 DE/CE 与 CE 密钥实际解封路径；含 19 条逐条核实归属的公开案例、17 条常见误传纠正与真实可读的属性观测点
- **Mainline 与 APEX 模块更新手册**：[android-mainline-apex-modules.md](/android-mainline-apex-modules/)
  把 Mainline 能力与 APEX 容器两个常被混用的概念分开：逐字模块清单与容器形态、APEX 双重签名与版本回滚语义、Mainline 与 Play 两条更新通道及扁平化 APEX 例外、公告中 Mainline 条目的实际形态、SPL 与模块版本不等价的双轴处置；含已核实条目与两处公告空白
- **生物识别与锁屏凭据手册**：[android-biometric-lockscreen.md](/android-biometric-lockscreen/)
  现行强度分档 Class 3/2/1 与按用途的组织方式、CDD 生物识别条款门槛、框架 API 与 HAL 版本实名、LSKF→scrypt→synthetic password→CE 密钥的实际链路与 Weaver 限频；含厂商指纹 TA/驱动侧已核实案例与误传纠正
- **基带与电话子系统手册**：[android-baseband-telephony.md](/android-baseband-telephony/)
  应用处理器与基带两侧的分工及两条修复通道、RIL AIDL 子域实名与内核侧宿主接口（QMI/MHI/QRTR/SMD/BAM/CCCI）、高通/联发科/三星公告的组件命名轴、SPL 与基带固件版本的真实关系、Project Zero 暴露面结论与逐条核实的 modem 类 CVE 归属
- **多用户、工作资料与进程隔离手册**：[android-multiuser-sandbox-isolation.md](/android-multiuser-sandbox-isolation/)
  用户类型与 UID 计算公式、DISALLOW_* 策略常量的真实宿主类、SELinux 域与 seccomp 的现行官方措辞、isolated process / USAP / AVF 三层进程隔离边界；含 10 条多用户与 Zygote 侧已核实案例、15 条常见误传纠正与可执行的实况观测清单
- **ADB、备份与数据迁移通道手册**：[android-adb-backup-transfer.md](/android-adb-backup-transfer/)
  adbd 授权判定与 RSA-2048/SHA-256 指纹真相、ro.adb.secure 的实际读取条件、无线调试配对引导与 7 天授权超时、fastboot 的 AOSP 与 OEM 命令边界、备份与 D2D 迁移的两条独立控制面（dataExtractionRules、bmgr、ro.backup.disable）；含 10 条通道侧已核实案例与 15 条误传纠正
- **系统更新链路与回滚防护手册**：[android-ota-update-rollback.md](/android-ota-update-rollback/)
  更新引擎与 A/B、Virtual A/B 两条通道（`update_engine` 对应用隐藏、公开面为 `SystemUpdateManager`）、载荷 `CrAU` magic 与 `payload_properties.txt` 四字段、`otacerts.zip` 验签、回滚索引与降级判定的真实条件、`recovery` 成文边界、releasetools 与 target_files 打包、FRP 交叉；含逐条核实归属的公开案例、误传纠正与真实可读属性，未能核实项单列
- **APK 签名与安装校验手册**：[android-apk-signing-install.md](/android-apk-signing-install/)
  v1–v4 方案官方口径与保护范围、v3 proof-of-rotation 属性与 `apksigner rotate` 参数、Android 13 起才认 lineage、`PackageManager` 安装标志与 46 个 `INSTALL_FAILED_*` 实名、`GET_SIGNATURES` 顺序不稳定与 `signingInfo` 迁移、剥离 v2+ 签名的拒绝规则；含 13 条案例与 13 条误传纠正
- **框架 IPC 与组件导出攻击面手册**：[android-framework-ipc-export.md](/android-framework-ipc-export/)
  组件导出与权限属性、640 条 `<protected-broadcast>` 实名清单口径、PendingIntent 与后台活动启动（BAL）规则的现行形态、ContentProvider 与四类 URI 授权标志、Deep Link 与 App Links 验证、Task 劫持的官方术语与 opt-in 边界；含逐条核实归属的案例表、18 条误传纠正与自查观测点
- **SELinux 策略与 seccomp 沙箱手册**：[android-selinux-seccomp-sandbox.md](/android-selinux-seccomp-sandbox/)
  SELinux 与 seccomp 两条强制轴的分工（对象访问矩阵 vs 系统调用集合）；Android 5.0/6.0/8.0/9.0 四步强化的官方逐字口径；`system/sepolicy` 的 public/private/vendor/reqd_mask/prebuilts 真实布局与「public 不许写 allow」这条成文纪律；签名→seinfo→seapp_contexts 的三级映射与 `levelFrom` 取值约束；`untrusted_app*` 按 targetSdk 降档、isolated/ephemeral/sdk_sandbox 分流；988 条 neverallow 中与 app 沙箱直接相关的 20 余条逐行出处（ptrace 双向禁止、capability、tee_device、三个 binder 域）；bionic `libc/seccomp/seccomp_policy.cpp` 的 `SECCOMP_RET_TRAP` 与 primary/secondary 架构对；`SEPOLICY_VERSION=30` 与 202404/202504 两条版本轴；freeze/treble/apex 策略检查与 CTS `SELinuxHostTest` 实名；16 条常见误传纠正，未核实项与未做 CVE 核对均单列
- **内核与 GKI 驱动攻击面手册**：[android-kernel-gki-drivers.md](/android-kernel-gki-drivers/)
  官方根本没有「GKI 2.0」这个说法，现行口径是一条双条件强制句（Android 12 起且内核 ≥5.10 必须用 GKI）；`androidNN-K.K` 分支命名与 Android 12–17 的多内核并存矩阵、KMI 世代号写在 `build.config.common`、冻结后影响 KMI 的 CONFIG 变更被构建拒绝；Android 13 起 `boot.img` 只剩内核、generic ramdisk 迁往 `init_boot`、`system_dlkm` 成为强制动态分区；两条签名主体的真实分工——认证 GKI `boot.img` 未做 verified boot 签名、OEM 必须用设备专属 AVB key 重签，而 `system_dlkm` 内的模块由 Google 的 kernel build-time key pair 签名；vendor hook 的 `android_vh_`/`android_rvh_` 实名与「挂受限 hook 的模块永远无法卸载」这条处置约束；可一手实名的加固项只有 `CONFIG_CFI_CLANG`、`CONFIG_CFI_PERMISSIVE`、`CONFIG_SHADOW_CALL_STACK` 三项；公告内核表列头实名 `CVE / References / Type / Severity / Subcomponent` 与 Subcomponent 取值（pKVM、IOMMU、Net、EPoll、XFRM、Mali、PowerVR-GPU、preloader）、19 行逐字案例台账、`Kernel LTS` 表才是内核修复的权威核对位置；18 条常见误传纠正，DDK 与符号表文件名、syzkaller、历史 CVE 措辞等 12 组未核实项单列
- **崩溃诊断与日志信息暴露面手册**：[android-diagnostics-log-exposure.md](/android-diagnostics-log-exposure/)
  把「崩溃会留下什么」拆成四条互不替代的留存通道：`debuggerd`/`crash_dump`/`tombstoned` 的原生崩溃路径、`dropbox` 的系统级事件 blob、`bugreport` 这个最大的聚合外泄面、`logcat`/`logd`/`READ_LOGS` 的读取边界；另含内核侧 `pstore`/`ramoops`/`dmesg_restrict`/`kptr_restrict` 的留存与抑制、「日志含敏感信息」的官方成文位置、误传纠正与「观测缺口 ≠ 风险缺口」这条方法论结论
- **变体设备安全面手册（车机／电视／穿戴）**：[android-variants-automotive-tv.md](/android-variants-automotive-tv/)
  变体不是「同一系统换个壳」：车机用户模型必须从 headless system user 讲起、VHAL 与 CarService 这层车机特有信号抽象的真实边界、变体在官方披露体系里的位置、AAOS 公告逐条台账与 CISA KEV 中车机条目全部缺席的核对结果；含只用已核实命令/属性的自查清单与企业侧处置建议
- **供应链与构建完整性手册**：[android-supply-chain-build-integrity.md](/android-supply-chain-build-integrity/)
  为什么「代码从哪来、产物怎么被验证」也是安全边界：AOSP 源码的官方获取口径与发布节奏、官方认可的 Git tag PGP 验签、`make dist` 与 `make` 两条构建刷机路径的差异、`super`/`lpmake` 动态分区里必须保持物理的三类、产物侧签名锚点、SBOM 的官方生成路径与它能回答／不能回答的问题；含产物与发布链路侧已修复案例、分角色处置清单与误传纠正
- **文件路径与 URI 处理类漏洞手册**：[android-path-uri-handling.md](/android-path-uri-handling/)
  把「把用户给的名字当成自己认识的对象」拆成三种逃逸与三条防线：同一个 inode 的多个名字（含 `isolateAppData()` 在私有 mount namespace 里 `symlink("/data/data", "/data/user/0")`）、`canonical` 与 `normalize` 的分工与六类归一化差异、检查与使用之间的窗口（`O_NOFOLLOW`／FD 化／Java 层缺 `RESOLVE_BENEATH` 的边界）、`startsWith` 少斜杠与 `Path.resolve` 绝对路径覆盖（并纠正「Java `new File(root, input)` 会被绝对路径跳出」这条误传）；另含 Scoped Storage 与 `MANAGE_EXTERNAL_STORAGE` 的三条做不到、SAF 授权随重启失效与 `FLAG_GRANT_PREFIX_URI_PERMISSION` 的前缀溢出、`FileProvider` 七个根与 `belongsToRoot()` 的 `rootPath + '/'`、自写 `openFile()` 的三个校验点、Zip Slip 七类变体、12 条自查清单与 8 组未核实项
- **无线与近场通信攻击面手册（Wi-Fi／蓝牙／NFC／UWB）**：[android-wireless-nearby-stacks.md](/android-wireless-nearby-stacks/)
  把「射频邻近即可攻击」拆成四套独立边界：Wi-Fi 是网络栈（`com.android.wifi` 的模块化有版本差异，Android 11 时 Wi-Fi 服务仍在 system_server 内）、蓝牙是外设与音频总线（Android 12 三个运行时权限与 `neverForLocation` 会过滤部分 beacon 的副作用）、NFC 贴近即读且卡片模拟的唯一授权点是「默认服务」选择、UWB 的 HAL 由 FiRa UCI 约束；另含 Wi-Fi 三面 HAL（Vendor／Supplicant／Hostapd）与 AIDL 化时间线、HID 注入与「配对≠在场」、邪恶双胞胎与 EAP 证书校验、NDEF 解析边界、2017/2021/2023 三种公告表形态、七类成因、12 条误传纠正与 8 组未核实项
- **摄像头、麦克风与屏幕捕获面手册**：[android-media-capture-privacy.md](/android-media-capture-privacy/)
  把「能录你」拆成三条对象不同的链路：相机与麦克风受权限 + app-op 双层控制（`AppOpsManager.startOp/stopOp/noteOp`、Android 9 起后台受限、Android 12 起有设备级开关与状态栏指示器），屏幕捕获则是**每次会话一次同意**（Android 14 起复用 `createScreenCaptureIntent()` 的 Intent 或在同一 `MediaProjection` 上二次 `createVirtualDisplay` 抛 `SecurityException`，未注册 `Callback#onStop` 抛 `IllegalStateException`）；另含应用级共享与状态栏胶囊的版本限定（Android 15 QPR1）、`FLAG_SECURE` 的真实边界、`AudioPlaybackCapture` 的被捕获方允许模型、Media framework 公告条目的读法、七种成因与 12 条误传纠正
- **HAL 与 Treble 边界攻击面手册**：[android-hal-treble-boundary.md](/android-hal-treble-boundary/)
  把「厂商代码怎么被系统调用」拆成四道互不替代的关卡：VINTF 声明只回答「提供什么／要求什么」而非访问控制、binderized 与 passthrough 的地址空间差别（SP-HAL 库被加载进框架进程）、servicemanager 与 hwservicemanager 两个不互通的注册中心、VNDK 自 Android 15 弃用与 linker namespace 的 `public.libraries.txt` 白名单；另含 HIDL（Android 10 弃用）→ AIDL for HALs（Android 11）的实例名形态差异、`checkvintf` 实为构建期 host 工具而设备侧二进制叫 `vintf`、`lshal` 看不到 AIDL HAL 的盲区、五类结构性成因、八条 CVE 归属纠正表与 13 条误传纠正
- **存储与文件系统栈攻击面手册**：[android-storage-filesystem-stack.md](/android-storage-filesystem-stack/)
  把「应用写文件」拆成三条对象不同的机制：CE/DE 的路径与可用时机（`/data/user/0` vs `/data/user_de/0`）、vold 以 Binder 服务（`IVold`）管挂载与全部 FBE 密钥、以及 `/sdcard` 上真正做访问判断的是 MediaProvider 进程内的 Java `FuseDaemon` 线程；另含 sdcardfs 弃用的内核版本限定（5.4+，4.19 及以下仍可用）、metadata encryption 自 Android 11 强制且由 KeyMint 保护、synthetic password 随机生成而非由口令推导、IncFS 的 Merkle 根来自 APK v4 签名而非文件系统；含 F2FS 案例、六条归属纠正与 13 条误传纠正
- **适用对象**：安全学习者、企业安全管理者、Android 开发者、设备厂商与 ROM 维护者

## 说明

- 资料仅基于公开、已修复的漏洞信息，用于教育与防御目的
- 不包含可直接用于攻击的利用代码
- 案例与时间线均来自公开资料，具体修复版本请以厂商公告为准

## 参考

- [Android Security Bulletin](https://source.android.com/docs/security/bulletin)
- [Android Security Overview](https://source.android.com/docs/security)
- [OWASP MASTG](https://mas.owasp.org/MASTG/)

## 联系方式

咨询 Android 系统详情请联系 https://t.me/one00190

---

*本仓库内容仅供安全学习与防御研究使用。*
