---
layout: page
title: "Android 系统漏洞安全研究资料"
description: "从防御视角梳理 Android 系统攻击面、漏洞类型、典型案例、编号与公告体系、底层架构与研究方法的公开研究资料，含一份总览报告、八十四份专题手册与一篇底层漏洞研究文章。"
permalink: /
---

> 全部文章可免费阅读；Markdown 源文件见 [https://github.com/douglasr864469/android-security-report](https://github.com/douglasr864469/android-security-report)，同一份资料同步发布在 GitLab 与 Bitbucket。

从防御视角梳理 Android 系统攻击面、漏洞类型、典型案例、编号与公告体系、底层架构与研究方法的公开研究资料，含一份总览报告、八十四份专题手册与一篇底层漏洞研究文章。

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
- **网络数据路径与内核过滤攻击面手册**：[android-network-data-path.md](/android-network-data-path/)
  把「应用能不能联网」拆成策略与执行两层：策略在 Java（`ConnectivityService`／`NetworkPolicyManager`／`Vpn`），执行在内核（eBPF cgroup 钩子、`xt_bpf` 挂于 `bw_raw_PREROUTING`／`bw_mangle_POSTROUTING`、按 UID 的 firewall 链）；另含 qtaguid→eBPF 的替换与 `bpfloader` 是独立特权进程、VPN 的 allowlist／disallowlist 互斥与 `bindProcessToNetwork` 的旁路效应、`IDnsResolver` 的按网络缓存与 `setAllowBypassPrivateDnsOnNetwork` 的逐 UID 降级、DoT（SDK 28）／DoH3（SDK 30）、neverallow 主体是 `all_untrusted_apps` 而非 `appdomain`；含七类成因与 11 条误传纠正
- **ART 运行时与代码加载攻击面手册**：[android-art-runtime-code-loading.md](/android-art-runtime-code-loading/)
  把「执行的代码」与「发布的代码」分开：dex→vdex（提取的 dex + 校验和 + `VerifierDeps` + 类型查找表）→odex→`.art` 的四产物链、编译过滤器自 Android 8 起四种而 `quicken` 已移除、boot image 自 Android 12 固定 `speed-profile`、Android 14 起 JIT profile 恒开启且不可关闭；另含 ART 作为 `com.android.art` APEX 独立更新的双刃、hidden API 的豁免逻辑不含 debuggable 分支、动态加载的三次收紧（Android 10 禁 `execve`／可写 fd 映射、14 强制 Java 侧只读、17 扩到 `System.load()`）、`public.libraries.txt` 构成的 native 边界；含 dex2oat 案例与 13 条误传纠正
- **图形与显示栈攻击面手册**：[android-graphics-display-stack.md](/android-graphics-display-stack/)
  把「一块像素到屏幕」的三段结构讲透：BufferQueue 由消费者创建并拥有且可与生产者跨进程、SurfaceFlinger 是独立 system 进程且是唯一能修改显示内容的服务、HWC 决定合成路径；另含 gralloc 的 HIDL 谱系自 `@2.0` 起（gralloc1 并非 HIDL）、composer 无 `@2.0`、ION→dma-buf heaps（节点为 `system-uncached`，连字符）、普通应用域对 `gpu_device` 持有 rw 权限这一硬事实、`FLAG_SECURE` 的官方边界与「不是权限」这一事实、`HW_SECURE_ALL`／`HW_SECURE_DECODE` 与 Widevine L1/L2/L3 的区分、`setTransactionState` 三个后果不同的案例；含六条归属纠正与 12 条误传纠正
- **输入子系统与事件注入攻击面手册**：[android-input-event-injection.md](/android-input-event-injection/)
  把「点一下」拆成三层三种身份：内核 evdev 的 24 字节 `input_event` 与 `SYN_REPORT` 提交屏障、system_server 内的 EventHub/InputReader/InputDispatcher、以及应用侧走 socketpair 而非 Binder 的 InputChannel；另含五条注入入口的权限对照（`injectInputEvent` 的 `INJECT_EVENTS`、`adb shell input`、`UiAutomation`、无障碍手势、`uinput`）、注入事件 `deviceId=-1` 的可辨识性边界、Android 12 不可信触摸阻断与 `FLAG_WINDOW_IS_OBSCURED`、`setHideOverlayWindows`、无障碍=完全控制的授权半径；含 13 条误传纠正与 4 组未核实项
- **音频栈与媒体会话攻击面手册**：[android-audio-media-session.md](/android-audio-media-session/)
  把「麦克风开着」拆成四条链路：audioserver 里的 AudioFlinger/AudioPolicyService/AAudioService、共享内存而非 Binder 的 PCM 通路、HAL 从 HIDL 到 AIDL 的形态迁移、以及媒体会话这条独立控制面；另含 `RECORD_AUDIO` 与 AppOps 双层判定、Android 9 起后台麦克风限制、回放捕获「双方同意」模型（`MediaProjection`、`allowAudioPlaybackCapture`、`setAllowedCapturePolicy` 三档）、通知监听器等价于全局媒体控制权；含 13 条误传纠正与 4 组未核实项
- **传感器与定位攻击面手册**：[android-sensors-location.md](/android-sensors-location/)
  把「设备知道你在哪」拆成两条互不相同的通路：免权限 motion 传感器经 BitTube 队列与 `SensorDirectChannel` 旁路上报、定位经 LocationManagerService 与 GNSS HAL；另含权限/精度/频率三个独立维度、Android 12 的 200 Hz 上限与 `HIGH_SAMPLING_RATE_SENSORS`、大概位置的模糊化实现、passive provider 的「搭便车」通道、模拟位置的 AppOps 通路与 `isMock()` 的边界；含 13 条误传纠正与 5 组未核实项
- **设备端 AI 推理运行时攻击面手册**：[android-ondevice-ai-runtime.md](/android-ondevice-ai-runtime/)
  把「模型文件」当作文档型攻击面来审：NNAPI 自 Android 15 弃用（官方建议迁移 TF Lite GPU runtime）、控制面走 Binder 而张量走共享内存、HAL 驱动在 vendor 进程、TFLite 以 `TFL3` 标识且权重是模型自己写的偏移+长度；另含形状乘法溢出、Burst slot 复用导致的 buffer 重叠、delegate 把参数带进 GPU/NPU 驱动、模型热更新等于远程输入入口；含 13 条误传纠正与 5 组未核实项
- **Verified Boot 与 dm-verity 验证链手册**：[android-verified-boot-chain.md](/android-verified-boot-chain/)
  把「验证通过」拆成四段与四个断点：硬件信任根→bootloader→`vbmeta`（`AVB0`）→分区的 hash/hashtree/chain 描述符；另含 AVB 自 Android 8.0（7.0 强化的是 dm-verity）、`HASHTREE_DISABLED`/`VERIFICATION_DISABLED` 两个逃生标志、`veritymode` 的 enforcing/logging/disabled、动态分区 dm-linear 后叠 dm-verity、rollback index 与降级、四种启动状态里 yellow 与 orange 的区别；含 13 条误传纠正与 4 组未核实项
- **虚拟化与隔离边界手册**：[android-virtualization-isolation.md](/android-virtualization-isolation/)
  五层隔离边界逐层取证：应用沙箱的 UID/DAC、Android 8.0 起「all apps」受 seccomp-bpf 约束、Android 9 起按应用的 SELinux 沙箱如何叠加；AVF 自 Android 13 引入且官方明确不取代 TrustZone；pKVM 不是新 hypervisor 而是把「约束宿主」搬进 EL2，普通 KVM 因宿主态关闭 stage-2 给不了 pVM 级机密性；pVM 只保证机密性与完整性，可用性明确例外、OEM 与 SoC 仍在信任集内；Microdroid 用 Bionic/Binder/APEX 换来的富环境如何变成新的接口面；Trusty 与 StrongBox 两条并列边界；公告中 `pKVM` 与 `Kernel Virtual Machine` 两个不同子组件的实证；17 条误传纠正、观测清单与 8 组未核实项
- **原生内存安全与分配器手册**：[android-memory-safety-allocators.md](/android-memory-safety-allocators/)
  四道性质不同的防线：Scudo 的 chunk header 校验与 quarantine（Android 11 起默认，但低内存设备仍用 jemalloc）、MTE 的 tag 传递与 SYNC/ASYNC 失败语义（App 默认关闭、需 bootloader 配合、内核 MTE-KASAN 是另一套开关）、HWASan 与 GWP-ASan 的适用面与 1% 采样、以及靠零初始化（Android 12 起，不含 zygote 与 App）与 Rust（仅覆盖新代码）消除整类缺陷；含 86%→不足 20% 的官方曲线、11 行防护矩阵、公告表头 `Component`/`Subcomponent` 陷阱、18 条误传纠正与 7 组未核实项
- **竞态条件与 TOCTOU 手册**：[android-race-conditions-toctou.md](/android-race-conditions-toctou/)
  把「检查与使用不同步」拆成三种后果互不相同的缺陷：引用计数与解锁的顺序错误导致 UAF（CVE-2025-38349）、锁顺序与 GC 路径导致死锁（CVE-2024-35970）、路径组件在解析期间被替换（`LOOKUP_NO_SYMLINKS`／`LOOKUP_BENEATH` 依赖 `rename_lock`/`mount_lock` 检测、`d_lookup()` 只检测不阻止）；另含 `refcount_t` 与 RCU 的官方语义边界（宽限期只等已存在的读者）、lockdep 的检查对象与「不自动识别天然嵌套顺序」的盲区、三行公告表逐格取证、lockdep/debugobjects/KASAN/Scudo 的可见性矩阵、七种成因模式与 12 条误传纠正
- **无障碍、输入法与自动填充通道手册**：[android-accessibility-ime-autofill.md](/android-accessibility-ime-autofill/)
  把「辅助功能能看到别人应用里的内容」拆成三条授权方式不同的通道：`BIND_ACCESSIBILITY_SERVICE`／`BIND_INPUT_METHOD`／`BIND_AUTOFILL_SERVICE` 都是 `signature` 级，约束的是「谁能绑定」而非「能看多少」；输入法经 `InputConnection` 拿到正在输入的明文、全屏提取视图把文本画进输入法自己的窗口、密码只剩 `InputType` 变体与一句 Caution；自动填充拿到**前台应用**整棵 `AssistStructure`、`importantForAutofill` 只有五个取值（无 `excludeDescendants`）、`FLAG_SECURE` 不阻止自动填充；无障碍侧的收紧是 targetSdk 23 门槛、Android 13 受限设置、Android 14 `accessibilityDataSensitive` 与 `isAccessibilityTool` 的双边约定；含 11 条公告台账（含修复提交落点）、六种成因模式、14 条误传纠正与 10 组未核实项
- **账户、凭据与无密码认证手册**：[android-accounts-credentials-passkeys.md](/android-accounts-credentials-passkeys/)
  把「证明你是你」拆成三条绑定对象不同的链路：`AccountManager` 的账户只由 `name` + `type` 标识、数据在 system_server 而认证器在应用进程，`GET_ACCOUNTS` 自 targetSdk 26 起「不被使用」、真正的门槛是五档可见性（同 UID 恒可见，同签名只是可被用户撤销的 `USER_MANAGED_VISIBLE`），`peekAuthToken` 需与认证器签名匹配；凭据管理器是平台 API（Android 14 起的提供方）与 Jetpack 双轨，密码能力到 API 19、Passkey 到 API 28、库 minSdk 23，创建侧字段是 `registrationResponseJson` 而非 `creationResponseJson`，提供方要覆写的是带 `Request` 的三个抽象方法；Passkey 的信任根是 RP ID 域名，`assetlinks.json` 用 `get_login_creds` 共享凭据、`handle_all_urls` 只用于 App Links；含 14 条账户类 CVE 台账（逐条回到 CVE 描述核对 `AccountManagerService.java` 落点）、六种成因模式、17 条误传纠正与 10 组未核实项
- **蓝牙与 BLE 配对、绑定与权限模型手册**：[android-bluetooth-ble-pairing-permissions.md](/android-bluetooth-ble-pairing-permissions/)
  把「配对就安全了」拆成五段：发现／连接／配对／绑定／授权访问，权限侧讲清 Android 12 三件套为什么是三种能力（`BLUETOOTH_SCAN`／`ADVERTISE`／`CONNECT`）、`neverForLocation` 是应用承诺且会让系统过滤部分 beacon、定位耦合看 `targetSdk` 而非设备版本、CDM 代扫只绕过定位不绕过 `CONNECT`；协议侧讲清 SSP 四种关联模型与 MITM 的真实对应（Just Works 不防中间人）、LE Legacy 与 LE Secure Connections 的分界、LTK/IRK/CSRK 都由栈持有且无公开 API；绑定侧讲清 `BOND_BONDED` 只表示本地存有 link key、`setPairingConfirmation` 需 `BLUETOOTH_PRIVILEGED`、`ACTION_PAIRING_REQUEST` 只能收不能伪造、Android 16 改为断开链路并保留本地绑定、解绑不承诺对端删密钥；GATT 侧讲清 `PERMISSION_READ_ENCRYPTED` 与 `_ENCRYPTED_MITM` 不是一回事、栈只强制安全位；含 11 条公告台账（逐条核对 `System` 节与 `packages/modules/Bluetooth` 补丁路径）、六种成因模式、22 条误传纠正与 12 组未核实项
- **USB 主机、配件与有线外设攻击面手册**：[android-usb-host-accessory-peripherals.md](/android-usb-host-accessory-peripherals/)
  把「插上 USB」拆成三条角色不同的路径：主机模式由手机供电并枚举、配件模式（AOA）由配件供电并枚举、设备侧 gadget 由对端主机枚举；权限侧讲清 `requestPermission` 的授权只到设备断开为止、`UsbManager.ACTION_USB_PERMISSION` 并不存在（官方示例用应用自定义 action）、intent-filter + `device_filter.xml` 会弹「是否打开此应用」而非静默授权、空过滤器匹配所有设备、UVC 设备还需 `CAMERA` 权限；外设侧讲清 HID 直达 `InputReader`、音频走 HAL、RNDIS/NCM 进入网络栈、Android 无官方 UMS，以及 MTP 只在解锁后挂载共享存储、PTP 仅暴露 `DCIM`/`Pictures`、不暴露 `/data`；设备侧讲清 `IUsbGadget` 与 `GadgetFunction` 功能位、configfs 由特权域写入、ADB 的三道门与 `ro.adb.secure`、Android 12 起的 USB 数据信令企业策略；另含 USB-C 角色与 Alt Mode（无 `MODE_DRP`、框架不解析 PD）、14 条已核实公告台账与两条「公告措辞≠真实组件」案例、六种成因模式、12 条误传纠正与 10 组未核实项
- **媒体解码与容器解析攻击面手册**：[android-media-codec-container-parsing.md](/android-media-codec-container-parsing/)
  把「打开一个媒体文件」拆成取数／解复用／解码／输出四段：Android 7.0 把单体 `mediaserver` 拆成 `mediaextractor`（UID 1040）、`media.codec`、`audioserver`、`cameraserver` 等独立进程，Android 10 又把软解移入 APEX 里的 `media.swcodec`（UID 1046，`mediacodec` 用户）；沙箱侧逐条列出 `mediaextractor` 能读 fd 但不能 `open` 应用数据、`mediaswcodec` 只有 GPU/dmabuf 而无应用数据与视频设备、`MediaExtractor` 默认经 `media.stagefright.extractremote` 走远程解析、`MediaParser`（API 30，非 12）默认在应用进程内解析、`MediaMetadataRetriever` 仍需绕经 `mediaserver` 与解析器两跳；解码侧讲清 `c2.`/`omx.` 前缀分派、软解与硬解的补丁节奏差异、`isSoftwareOnly` 与 API 36 的 `getSecurityModel`；含 27 条已核实公告台账（2015 Stagefright 批次、libavc/libhevc/libmpeg2/aac、容器解析与 EoP）、六种成因模式、13 条误传纠正与 14 组未核实项
- **窗口管理、悬浮窗与界面覆盖（tapjacking）攻击面手册**：[android-window-overlay-tapjacking.md](/android-window-overlay-tapjacking/)
  把「用户看到的就是他点到的那一层」拆成四段：窗口分层、触摸命中、覆盖授权、遮挡判定。类型侧列清 `TYPE_SYSTEM_ALERT` 是 2003（2002 才是 `TYPE_PHONE`）、`TYPE_TOAST` 2005、`TYPE_APPLICATION_OVERLAY` 2038 必须叠在 Activity 之上且位于状态栏与输入法之下；权限侧讲清 `SYSTEM_ALERT_WINDOW` 的 `signature|setup|appop|installer|pre23|development` 与 AppOps 兜底、`Settings.canDrawOverlays` 只是建议、Bubbles 与 PiP 不需要该权限、`HIDE_OVERLAY_WINDOWS` 是 `normal` 级；输入侧讲清 `FLAG_WINDOW_IS_OBSCURED` 与 `FLAG_WINDOW_IS_PARTIALLY_OBSCURED` 的差别、`setFilterTouchesWhenObscured` 只认全遮挡、Android 12 起非受信覆盖层的穿透触摸被直接拦截（可信仅限无障碍／输入法／助手，组合不透明度按 `1 - Π(1-opacity)` 计算且 >0.8 才拦）；含 14 条核心公告台账与 16 条应用对话框台账、六种成因模式、16 条误传纠正与 14 组未核实项
- **通知、状态栏与通知监听攻击面手册**：[android-notification-statusbar-listener.md](/android-notification-statusbar-listener/)
  把「一条通知」拆成提交／入队／排序／分发／呈现／交互六段：`NotificationManagerService` 在 system_server 而非 SystemUI、渠道重要性的公开档位与 `IMPORTANCE_MAX` 系统保留、`POST_NOTIFICATIONS` 的预授予与前台服务通知降级、通知监听器的 `signature` 级绑定权限与「授权范围内可见」边界（工作资料监听器被忽略）、`PendingIntent` 可变性自 Android 12 起必须显式而非默认不可变、全屏意图自 Android 14 起限通话/闹钟、Android 15 对不受信监听器的验证码脱敏与投屏隐藏；含 24 条已核实公告台账（20 条通知服务/监听接口 + 4 条锁屏与相邻组件，逐条回到 NVD 的类/方法落点核对）、六种成因模式、14 条误传纠正与 12 组未核实项
- **电源管理、唤醒锁与闹钟调度攻击面手册**：[android-power-wakelock-alarm.md](/android-power-wakelock-alarm/)
  把「熄屏后谁还能跑」拆成四套状态机：`PowerManagerService`（system_server）结算唤醒锁、`AlarmManagerService` 与 `DeviceIdleController` 位于 jobscheduler APEX、`AppStandbyController` 负责分桶；讲清 `WAKE_LOCK` 普通权限与「保活」无关、Doze 忽略唤醒锁、精确闹钟两套权限（`SCHEDULE_EXACT_ALARM` 可撤销 / `USE_EXACT_ALARM` 受政策限制）、`setExactAndAllowWhileIdle` 约 9 分钟量级；含 6 条已核实公告台账、六种成因模式、5 条误传纠正与 5 组未核实项
- **剪贴板、拖放与跨应用数据传递攻击面手册**：[android-clipboard-drag-drop.md](/android-clipboard-drag-drop/)
  把「复制粘贴」拆成写入／持有／读取／授权／呈现五段：`ClipboardService` 在 system_server、Android 10 起仅默认输入法与焦点应用可读且无助理豁免、`newUri()` 不授权而粘贴时临时授读、拖放四类 URI 标志默认临时；另含首次读取 toast 与 `EXTRA_IS_SENSITIVE` 只影响预览、`DRAG_FLAG_GLOBAL_SAME_APPLICATION`（API 35）；含 6 条公告台账（区分 URI 授权服务、内容提供者与剪贴板类）、六种成因模式、6 条误传纠正与 4 组未核实项
- **打印、PDF 与文档渲染通路攻击面手册**：[android-print-pdf-rendering.md](/android-print-pdf-rendering/)
  把「打印」拆成发起／绑定／写入／暂存／投递／输出六段：`PrintService` 是公开基类（`BIND_PRINT_SERVICE` 限绑定方不限实现）、`onWrite()` 写入 spooler 管道 FD 且不可 seek、「另存为 PDF」是假打印机 + SAF 而非打印服务；PDF 侧讲清 `PdfRenderer` 用 pdfium、`PdfDocument` 用 Skia 两条技术栈；含 4 条已核实公告台账、六种成因模式、5 条误传纠正与 5 组未核实项
- **字体、文本整形与渲染栈攻击面手册**：[android-font-text-rendering.md](/android-font-text-rendering/)
  把「一段文字到像素」拆成字符处理／选字体／整形／字形加载／栅格化／排版六段：`FontManagerService` 只在 system_server 分发映射，Minikin/HarfBuzz/FreeType 运行在应用进程；讲清「字体更新 APK」不成立（字体文件 + PKCS#7 + fs-verity + `config_fontManagerServiceCerts`）、可变字体与 COLRv1 的解析风险、downloadable fonts 只为本应用取字；含 7 条公告台账、七种成因模式、5 条误传纠正与 4 组未核实项
- **DRM 与受保护媒体通路攻击面手册**：[android-drm-protected-media.md](/android-drm-protected-media/)
  把「受保护播放」拆成选方案／供应／取密钥／装密钥／建会话／配置解码／解密／显示八段：应用只见不透明请求与会话 ID、现行安全级别是 0–5 而非 L1/L2/L3、`MediaFormat` 无 `KEY_CRYPTO`（用 `FEATURE_SecurePlayback` + `MediaCrypto` + `.secure`）、HDCP 只能查询不能强制；含 5 条公告台账（mediadrm/Widevine/cdm_engine）、六种成因模式、7 条误传纠正与 5 组未核实项
- **企业设备管理、托管配置与工作资料攻击面手册**：[android-enterprise-device-management.md](/android-enterprise-device-management/)
  把「企业管控」拆成角色／权限／配置／资料／策略五层：legacy admin 只执行 XML 声明策略、Device Owner 仅能在初始化阶段设定、`PROVISION_DEVICE_OWNER` 权限不存在（用 `MANAGE_PROFILE_AND_DEVICE_OWNERS` 或 shell、配置消息用 `DISPATCH_PROVISIONING_MESSAGE`）、工作资料即 secondary user；含 Android 12 provisioning 流程变化、4 条公告台账、六种成因模式、5 条误传纠正与 4 组未核实项
- **MediaProjection、屏幕录制与投屏通路攻击面手册**：[android-media-projection-screen-capture.md](/android-media-projection-screen-capture/)
  把「录屏」拆成意图／系统同意／框架服务／显示缓冲四层：`createScreenCaptureIntent` 只生成请求不授权、同意界面在 SystemUI 的 `MediaProjectionPermissionActivity` 而非某个「MediaProjectionService」、会话与令牌由 system_server 的 `MediaProjectionManagerService` 持有；Android 14 起每次会话重新授权、target 34+ 必须 `FOREGROUND_SERVICE_MEDIA_PROJECTION` 前台服务、同意界面区分单应用与整屏两种范围；权限侧区分 `CAPTURE_VIDEO_OUTPUT`（signature）、`DETECT_SCREEN_CAPTURE`／`DETECT_SCREEN_RECORDING`（normal）与 `MANAGE_MEDIA_PROJECTION`；数据侧讲清 `FLAG_SECURE` 安全图层在合成层就黑屏、DRM 行为随级别与输出保护而变、音频是独立授权链；含 5 条公告台账、六种成因模式、5 条误传纠正与 4 组未核实项
- **Binder 驱动与内核 IPC 实现攻击面手册**：[android-binder-driver-ipc.md](/android-binder-driver-ipc/)
  把「一次跨进程调用」拆成设备／对象／命令／缓冲四段：`/dev/binder`、`/dev/hwbinder`、`/dev/vndbinder` 与 binderfs 多实例共存、context manager 的 handle 0 按设备隔离、`binder_proc`／`node`／`ref`／`buffer` 的强引用弱引用与死亡通知；命令侧讲清 `BC_`／`BR_` 语义、oneway 仍占异步配额并会收到 `BR_DEAD_REPLY`／`BR_ONEWAY_SPAM_SUSPECT`、`BINDER_SET_MAX_THREADS` 默认 15 且不存在同名内核接口；归因侧讲清 `sender_euid` 由驱动写入而非调用方自报、SELinux `binder { call transfer set_context_mgr impersonate }` 才是真正的门；缺陷集中在长度偏移核算、释放路径与等待队列；含 6 条公告台账、五种成因模式、5 条误传纠正与 5 组未核实项
- **AppOps 与运行时权限的底层执行路径手册**：[android-appops-permission-enforcement.md](/android-appops-permission-enforcement/)
  把「应用有没有权限」拆成权限状态机与 AppOps 状态机两套：`PermissionManagerServiceImpl` 记录授予标志与来源、`AppOpsService` 在 system_server 按 uid＋包名＋op 记录 `MODE_ALLOWED`／`IGNORED`／`ERRORED`／`DEFAULT`；一次访问要经 checkPermission → Binder → `noteOp`／`startOp` → 数据方判定，只查权限位就放行是典型「检查了但没执行」；讲清 `unsafeCheckOp` 不能当授权依据、`FLAG_PERMISSION_ONE_TIME` 是加在普通权限上的标志而非权限类型、特殊访问入口在设置页；含 4 条公告台账（安装会话与存储过滤等权限服务之外的落点也在此列）、六种成因模式、5 条误传纠正与 4 组未核实项
- **设备标识符、广告 ID 与硬件指纹手册**：[android-device-identifiers-fingerprinting.md](/android-device-identifiers-fingerprinting/)
  把「设备 ID」拆成硬件／系统／服务／推断四层：IMEI 与序列号自 Android 10 起收进 `READ_PRIVILEGED_PHONE_STATE`、`ANDROID_ID` 自 8.0 起按「签名＋用户＋设备」派生（同签名重装不变、换签名会变）、广告 ID 可重置且 Android 12 起可删除并返回全零、Wi-Fi MAC 从恒零到按网络持久随机再到部分场景非持久随机；推断层讲清 Build 字段、传感器清单与 `QUERY_ALL_PACKAGES` 组合后仍具辨识度；含 3 条公告台账、五种成因模式、5 条误传纠正与 5 组未核实项
- **UWB 超宽带测距与安全测距攻击面手册**：[android-uwb-ranging-security.md](/android-uwb-ranging-security/)
  把「测到距离」拆成框架／模块／HAL／协议四层：`android.uwb` 没有公开的 `UwbAdapter` 类（只有 `IUwbAdapter.aidl`）、第三方走 `androidx.core.uwb`、`UWB_RANGING` 危险权限＋前台＋`android.hardware.uwb` 特性构成三道门；安全测距讲清 static STS 对重放/中继偏弱、provisioned STS 依赖安全带外分发、STS 不等于完整的 distance bounding；身份来自带外通道而非 UWB 本身，CCC Digital Key 3.0 正是 BLE 认证＋UWB 测距＋NFC 后备；并如实说明未检索到「越权测距」类公开 CVE；含 4 条公告台账、五种成因模式、5 条误传纠正与 5 组未核实项
- **时间、时区与网络授时的安全边界手册**：[android-time-timezone-ntp.md](/android-time-timezone-ntp/)
  把「现在几点」拆成 RTC／墙钟／单调时钟／授时源四路：`/dev/alarm` 在 Android 8.0 被内核 alarmtimer 取代、AOSP 的 SNTP 客户端无认证且未实现 NTS、NITZ 的信任基础是运营商、Android 12 起默认 NTP 优先于 NITZ；权限侧讲清 `SET_TIME`／`SET_TIME_ZONE` 是 signature|privileged|role、DPM 自动时间接口限 owner、`DISALLOW_CONFIG_DATE_TIME` 是用户限制；影响侧讲清 TLS 的本地校验、Play Integrity 判决须由服务端判新鲜度、密钥证明字段与 AVB 回滚索引不依赖本地时间、TOTP 防线在服务端；含 4 条公告台账、五种成因模式、5 条误传纠正与 5 组未核实项
- **WhatsApp「群发」功能边界与反滥用手册**：[android-whatsapp-bulk-messaging-boundaries.md](/android-whatsapp-bulk-messaging-boundaries/)
  把「群发」拆成广播列表／群组与社区／Business Platform 三条互不兼容的通道：广播列表上限 256 人且只有已保存发送方号码的人才会收到、多个列表相加也不能突破上限；群组上限 1,024 人、社区约 2,000 人，治理与邀请权另算；商业接口则另立 opt-in、24 小时客服窗口（窗口外只能发预先审核的模板）与模板质量评级（GREEN／YELLOW／RED，RED 暂停发送），限额在业务组合级按滚动 24 小时内的去重用户数统计，从 250 起逐级提额；并说明为何「群发软件」在条款、技术、法律与安全四面都走不通，给出账号／消息／接收方／企业四层反滥用机制、Android 客户端侧被篡改客户端与无障碍监听等风险、7 条合规运营清单；含 6 条误传纠正与 5 组未核实项
- **WhatsApp「群控」滥用生态与防御手册**：[android-whatsapp-device-farms-abuse-defense.md](/android-whatsapp-device-farms-abuse-defense/)
  把「群控」拆成账号／执行／编排／变现四层：账号层受《反电信网络诈骗法》第十四条直接规制（电话卡批量插入设备、批量账号与网络地址自动切换系统、批量接收短信/语音验证的平台）、执行层落到无障碍模拟操作／调试桥投屏／录屏识别回注／root 与定制 ROM／云手机与多开／修改版客户端六条路径，逐条给出权限代价与可见痕迹；条款侧逐句引用 WhatsApp 服务条款对自动化注册、批量与自动消息、封禁后另建账号、反向工程与克隆服务的禁止，并说明「养号」「一机一号一IP」「官方 API 白名单」为何都不成立；法律侧并列反电诈法第 14／38／42 条、刑法第 285 条第 3 款与「客多多精准获客」入库参考案例、第 253 条之一与第 287 条之二，并提示《网络安全法》2025 年修正后该禁止性规定已移至第二十九条、自 2026-01-01 施行；含 6 条误传纠正与 5 组未核实项
- **设备农场与云手机的检测工程手册**：[android-device-farm-detection-engineering.md](/android-device-farm-detection-engineering/)
  把「这是不是设备农场」拆成账号／设备／网络／编排四层并给出各层证据与局限：设备层讲清 Play Integrity 的 `deviceRecognitionVerdict` 各标签、`recentDeviceActivity` 三级活跃度、`appAccessRiskVerdict` 对已知与未知「捕获屏幕/控制输入」类应用的判决、`playProtectVerdict` 与 `appLicensingVerdict` 的真实含义，说明为何「完整性通过」不等于「真人」；网络层讲清 IP 信誉、ASN、CGNAT 与 IPv6 下的误判；行为层把「同步性」作为最难伪装的特征，给出时间同步、内容复用、接收方重叠与速率基线四类信号；含误报表、分级处置、合规边界（目的限定、自动化决定、留存与跨境）、5 条误传纠正与 5 组未核实项
- **群发名单与同意管理手册**：[android-bulk-messaging-consent-management.md](/android-bulk-messaging-consent-management/)
  把「他有同意吗」拆成可举证的证据链：三类名单（自有 opt-in／合作方共享／来源不明，第三类无合法基础）分别讨论；同意模型覆盖广播列表、群组与商业接口三条通道，逐项引用 Meta 2024 年 11 月起的 opt-in 规则、《个人信息保护法》第 13／14／17／23／39 条、《通信短信息服务管理规定》2026 修订版与 GDPR 第 4(11)／7 条；证据链给出最小字段表与四档强度，退订给出 7 条落地做法，跨境单列路径；含 5 条误传纠正与 5 组未核实项
- **接收方防护手册**：[android-bulk-messaging-recipient-protection.md](/android-bulk-messaging-recipient-protection/)
  从收到消息的一方出发，把「接触→建立信任→诱导操作→变现」四段攻击链拆开，给出每段的可观察痕迹与有效动作；技术核验讲清号码不是身份、官方标识要看平台结构化标记、链接看域名主体、聊天窗口发来的 APK 一律视为恶意（Android 13 起侧载应用默认限制无障碍与通知监听）；取证强调原始导出而非截图，设备侧自查覆盖关联设备／两步验证／补卡与权限组合；举报入口按平台内、12321、96110 与 110、运营商与银行分流，阻断按接收设置／账号加固／家庭与组织三层给出；含 5 条误传纠正与 5 组未核实项
- **企业群发与群控防护手册**：[android-enterprise-bulk-messaging-defense.md](/android-enterprise-bulk-messaging-defense/)
  把企业侧风险拆成账号接管、终端失守、第三方代运营与内部滥用四个入口：账号侧讲清验证码钓鱼、SIM 补卡、扫码登录与客服社工的路径，以及「投诉回流」这一外部信号；终端侧把群控客户端还原为「无障碍＋通知监听＋悬浮窗＋电池白名单」的权限组合，并结合 Android 13 限制设置与 Play Integrity `appAccessRiskVerdict`；落地侧对比设备所有者与资料所有者两种 Android Enterprise 形态，给出代运营五条硬要求、离职回收清单与应急流程；含员工个人信息保护的合规边界、5 条误传纠正与 5 组未核实项
- **群发、群控与账号买卖的法律边界手册**：[android-bulk-messaging-legal-boundaries.md](/android-bulk-messaging-legal-boundaries/)
  按行政、刑事、民事三层并列责任：行政侧引用《反电信网络诈骗法》第 14／31／38／42 条（卡池、群控盒子、接码平台、出租出借账号）、广告法与 2026 年修订的《通信短信息服务管理规定》；刑事侧梳理第 285 条第 3 款提供侵入非法控制程序工具罪（含「客多多」类群控软件入库案例）、第 253 条之一侵犯公民个人信息罪、第 287 条之一与之二、第 266 条诈骗罪；民事侧讲清《民法典》第 1165 条与第 1194–1197 条、个保法第 69 条过错推定；另给场景定性对照表、企业合规清单与「不知情」抗辩的推定边界；含 5 条误传纠正与 5 组未核实项
- **TLS 与证书信任链手册**：[android-tls-certificate-trust.md](/android-tls-certificate-trust/)
  把一次 HTTPS 连接拆成协议协商、链验证、信任锚判定、主机名匹配与额外策略五道关，逐层讲清 Conscrypt（BoringSSL、`com.android.conscrypt` APEX 自 Android 10、内部实现类非公开 API）、系统信任库（`<subject_hash_old>.<n>` 命名、Android 14 迁入 `/apex/com.android.conscrypt/cacerts`、用户库 `cacerts-added`）、按 targetSdk 分档的默认信任策略（23- 信任用户 CA、24+ 仅系统、28+ 默认禁明文、特权应用 Android 9+ 不信任用户库）、NSC 的 `base-config`／`domain-config`／`pin-set`／`debug-overrides`（pin 过期是停止固定而非连接失败、仅支持 SHA-256）、CT 自 Android 16 选择启用与 Android 17 计划默认开启；归纳五类失效模式（验证被短路、`TrustManagerImpl` 缓存缺陷 CVE-2016-0818、Conscrypt 内存安全、配置漂移、生态层信任错位），含 8 条已核实公告台账、14 条误传纠正与 6 组未核实项
- **WhatsApp 群发投递可观测性手册**：[android-whatsapp-bulk-deliverability-observability.md](/android-whatsapp-bulk-deliverability-observability/)
  把「发出去了」拆成请求／状态／错误／账号与模板四层信号：接口成功只代表请求被接受（官方明确不代表送达）、每条出站消息最多三条 `sent`／`delivered`／`read` 回执且失败回执省略 `contacts`、重试最长 36 小时故必须去重；逐码梳理 18 个错误（`130429`／`130497`／`131026`／`131047`／`131048`／`131049`／`131050`／`131051`／`131056`／`131057`／`132000`／`132001`／`132005`／`132007`／`132012`／`132015`／`132016`／`133010`）并给出「修好再发／退避／等 24 小时／永久停止／先注册」可重试性矩阵；讲清吞吐（默认 80 mps、共享 Business app 20 mps、自动升至 1,000 mps、含入站）、组合级消息限额（2025-10-07 起 250 起步、滚动 24 小时唯一用户数、质量下降不再降额）、号码对速率与每用户营销限额四把闸门；含模板与号码质量枚举差异、指标分母口径、事故分诊树、webhook 验签与去重、15 条误传纠正与 10 组未核实项
- **WhatsApp 模板治理与内容审核手册**：[android-whatsapp-template-governance-review.md](/android-whatsapp-template-governance-review/)
  把「能不能发」变成可管理的内控流程：先分清营销／工具／认证三类模板的分类判据（是否由用户的具体行为触发），再走审核流水线（通常 24 小时内出结论、通过／拒绝／待审三种落点），并逐项对齐结构硬约束（头部 60 字符 1 个变量、正文 1,024 字符且变量按 {{1}} 顺序连续编号不得位于首尾、变量值不得含 # $ %、页脚 60 字符纯文本不支持变量、按钮最多 10 个且 URL 按钮最多 2 个各 1 个变量并位于末尾、每个变量必须有示例值）；认证模板单独讨论仅承载验证码语义、变量 15 字符以内、有效期 1–90 分钟；归纳五类拒绝原因（变量不匹配、重复模板、政策违规、威胁诱导、信息不完整）与整改顺序，说明质量评级 GREEN／YELLOW／RED 如何反噬发送能力（3 小时与 6 小时暂停、节奏暂停需人工恢复、走向停用），给出配额与创建速率的容量规划用法；重点讲清「编辑即原地重审、没有版本回滚」的运维后果与影子模板流程，另附三角色内控、证据留存字段与 BSP 治理边界；含 10 条误传纠正与 5 组未核实项
- **WhatsApp 群组与社区治理手册**：[android-whatsapp-group-community-governance.md](/android-whatsapp-group-community-governance/)
  把群组当成有生命周期的权限系统而不是「拉一群人聊天」：结构层区分单群 1,024 人与社区容器（最多 100 个群、约 2,000 人、最多 20 名管理员、公告群强制存在）；权限层讲清管理员能做与不能做的边界，以及「最后一位管理员离开会自动提升某位成员」这条最容易被忽略的继承规则；生命周期层说明邀请链接可重置但不能撤回已加入者、被移出者进入「过往成员」约 60 天、已送达消息不会因移除而消失、群组没有一键销毁；另单列广播列表（256 人、收件人必须已保存你的号码、部分地区有月度限制）与群组的三项差异，逐句引用服务条款对批量自动化发送、自动拨号、自动化数据收集与抓取的禁止，还原举报群组时提交最近 5 条消息与群组标识且不通知被举报方、举报者通常被移出该群的行为；末附 @all／新成员历史消息／群组状态／活动等 2026 年前后变化、治理清单与事件响应五步；含 10 条误传纠正与 6 组未核实项
- **WhatsApp 多设备与链接设备会话安全手册**：[android-whatsapp-linked-devices-session-security.md](/android-whatsapp-linked-devices-session-security/)
  拆开「一部手机 + 最多 4 台链接设备」的身份体系：配对走二维码或 8 位配对码两条路径，每台设备持有自己的身份密钥、服务器按设备做 fan-out 分发，因此手机离线仍能收发，但手机连续离线约 14 天会强制登出全部链接设备、链接设备约 30 天不活跃会被断开；安全码覆盖账号的全部设备集合，设备增减都会变化，可作为「设备集变更」告警；端到端加密备份与消息加密是两套独立机制，密钥可交给 passkey、自定义密码或 64 位密钥；设备侧可见性有明确边界（能看到设备类型与粗粒度活跃时间，看不到读过哪些消息、无法远程擦除已同步历史）；归纳四类真实风险（二维码／配对码钓鱼、设备遗失与借用、备份密钥失守、非官方客户端）并给出「只扫自己发起配对的二维码、绝不把 8 位码告诉任何人」的操作底线；另记录 iPad 原生应用 2025-05、iPad 直接注册 2026-07、passkey 规模 2026-08 等节点与个人／企业两份清单；含 10 条误传纠正与 6 组未核实项
- **WhatsApp 账号接管与商业账号资产防护手册**：[android-whatsapp-account-takeover-business-assets.md](/android-whatsapp-account-takeover-business-assets/)
  把账号接管还原为四条现实路径（验证码拦截与社工、二维码／配对码劫持、邮箱 6 位码链路、账号买卖与内部风险），并说明邮箱验证不适用于首次注册只服务于已有账号的恢复；验证机制层逐项讲清两步验证从 6 位 PIN 向不少于 8 位字母数字演进、passkey 的抗钓鱼优势与生态依赖、应用锁与聊天锁只作用于终端侧、忘记 PIN 存在约 7 天等待期、无 SIM 卡无法通过短信恢复、号码约 45 天不活跃会被回收；资产层强调 WABA 不可在业务组合之间迁移、Solution Partner 授权上限为 5 家、业务组合完全控制权限最多 10 人且至少 2 人、Business App 链接设备 4 台（订阅后 10 台）且没有细粒度角色分离；防线按身份／设备／组织／监控四层给出可执行项，应急响应给出「先切断再取证」的六步顺序与「申诉是止损手段不是恢复手段」的定位，另附资产清单九项；含 10 条误传纠正与 6 组未核实项
- **WhatsApp 内容举报、转发限制与取证边界手册**：[android-whatsapp-content-reporting-forwarding-fidelity.md](/android-whatsapp-content-reporting-forwarding-fidelity/)
  讲清端到端加密下「平台看不到内容但看得到信号」的真实边界：举报是一次定向披露，提交最近约 5 条消息及其媒体、发送者与群组标识、时间与类型，不通知被举报方、举报者通常被移出该群，且举报与拉黑是两个独立动作、举报只是快照不能当作证据保全；转发限制按三层拆开（新消息一次最多 5 个会话、已转发标签、经 5 次以上转发的消息只能转 1 个会话并显示「已多次转发」），并说明它只是产品层减速，复制粘贴、截图与录屏均可绕过；时效功能讲清消失消息 24 小时／7 天／90 天可选且只作用于设置之后的新消息、转发副本与备份中的副本不会随之消失，view once 官方自己承认无法阻止另一台设备拍摄或录屏；导出与取证区分用户侧导出（可含或不含媒体、网页版不含媒体、导出文件是明文）与平台侧能力（不持有可读明文、不提供批量导出与已删消息恢复）；另记录高级聊天隐私 2025-04、严格账号设置 2026-01、端上诈骗提醒 2026-08 三项变化与三条治理原则；含 10 条误传纠正与 6 组未核实项
- **WhatsApp 商业消息数据治理与审计手册**：[android-whatsapp-business-data-governance-audit.md](/android-whatsapp-business-data-governance-audit/)
  回答合规评审必问的三件事：谁是控制者（商户为控制者、Meta 按 DPT 为处理者、BSP 通常是再处理者，数据主体权利的第一责任方是企业）、平台留多久（Cloud API 媒体约 30 天、标识符与状态在最后一次状态更新后约 30 天、No Storage 模式约 1 小时，On-Premises 的本地 60 分钟只对历史架构有意义且该 API 已于 2025-10-23 停止服务、合作终止后约 90 天内删除或去标识化）、能不能查到（平台只有 Admin Center 的安全审计日志与只记录管理动作的活动日志，没有消息级审计 API、没有消息撤回或删除接口、媒体删除只作用于媒体对象）；另讲清跨境传输的两条路径与 DPF 在 2025-09-03 裁决中被维持但仍应保留 SCC 备选、系统用户令牌最长 60 天需纳入轮换、数据主体请求的能力映射表与「无法按单个用户删除已投递消息」这一必须写进隐私声明的约束、2024-11 起一般性 opt-in 仍需可举证、2026-06 Business Agent 对处理范围的扩展，末附九项合规档案清单；含 10 条误传纠正与 6 组未核实项
- **WhatsApp 反滥用生态与透明度手册**：[android-whatsapp-abuse-ecosystem-transparency.md](/android-whatsapp-abuse-ecosystem-transparency/)
  把「平台怎么治理」整理成可核对的地图：先分四层主体（产品层检测处置、合同层条款与商业政策、监管层 DSA／OSA 等、司法层调取与保全）并指出申诉与监管投诉不互通；处置阶梯按警告、短期限制、中期限制、无限期限制、永久封禁五级展开，强调处置往往先于通知、部分限制可随质量恢复自助解除、同一业务组合下的账号可能被关联处置；透明度部分说清没有全球统一报告，数据分散在印度月度合规报告、欧盟 DSA 报告（公开版本曾仅覆盖 Channels 相关处置）、Meta 社区标准执行报告与官方博客，并给出「主动处置占比高、申诉量远小于封禁量、举报只覆盖滥用一小部分」的读法与引用时必须标注口径的提醒；执法协作区分常规与紧急请求、明确平台不持有可读明文、90 天数据保全只是暂停删除而非披露；另记录 DMA 互操作 2024-03／2025-09／2027-09 三阶段、英国 OSA 2025-03 生效、诈骗产业链以规模摊薄损失故治理须打到基础设施层，末附企业合规动作清单；含 10 条误传纠正与 7 组未核实项
- **WhatsApp 商业账号权限治理手册**：[android-whatsapp-business-account-permissions.md](/android-whatsapp-business-account-permissions/)
  把「谁能发」拆成个人账号、商业账号（WABA）与企业资产组合三层资产，指出只清理一层会留下影子发信人；归纳四个权限放大器（共享登录、链接设备、第三方工具长期授权、系统用户令牌）并给出运营／客服主管／技术集成／资产管理员四档权限矩阵，强调「能改配置的人少于能发消息的人」与成员/令牌双人复核；离职回收按 T-7／T-1／T+0／T+7 排序，单列「平台侧移除成员但第三方授权仍有效」这一最易漏项；审计部分说明平台只能做到事件级而非意图级，要求先保证一人一身份、关键动作双人复核，并定期做成员、协作账号、服务商子账号与令牌的四表对账；含 5 条误传纠正与 4 组未核实项
- **WhatsApp 加密备份与密钥治理手册**：[android-whatsapp-encrypted-backup-key-governance.md](/android-whatsapp-encrypted-backup-key-governance/)
  回答「备份是不是第二份明文」：默认云备份与端到端加密备份是两种安全状态，后者需用户显式开启并持有 passkey、密码或 64 位密钥之一；讲清 2021 年 Meta 工程博客公开的密钥分片与 HSM 设计只保证正常运维状态下平台无法解密，不覆盖设备被控、凭据泄露与客户端篡改；单列三个高风险模式（共用账号与共用凭据、把生产会话恢复到个人设备再导出、离职后云端备份残留），并给出「删除云端备份、吊销旧设备、轮换凭据」三件套；恢复部分引用官方限制：换机须同号且新机未注册、可在旧设备改号、聊天不能直接迁到新号码；末附六项治理清单、5 条误传纠正与 4 组未核实项
- **WhatsApp 频道与广播列表治理手册**：[android-whatsapp-channels-broadcast-governance.md](/android-whatsapp-channels-broadcast-governance/)
  把四种一对多形态放在一张表里对照（广播列表、群组、频道、商业平台模板消息），核心差别不是「能发多少人」而是关系是否双向、身份是否暴露、内容是否进入公共空间；频道部分说明管理员单向发布、关注关系不暴露号码、内容长期留存且可被转发，因此适合公告而不适合客服；广播列表被指出是「伪订阅」——没有订阅关系、收件人只是被放进列表，合法基础仍是事先同意；群组部分讲清成员可拉人、彼此可见号码与截图外传的连带风险；末给按关系强度而非人数排序的五步选型法、5 条误传纠正与 4 组未核实项
- **WhatsApp 号码迁移与数据可携手册**：[android-whatsapp-number-migration-portability.md](/android-whatsapp-number-migration-portability/)
  指出账号风险常集中在迁移窗口：按换机同号、换号同机、换号换机三条路径对照官方机制与风险，明确「新机迁移前不能已注册」「聊天不能直接迁到新号码、须先在旧设备改号」两条硬限制；拆开迁移期的四类攻击面（验证码通道、旧设备残留、备份错配、双注册风控）；商业号码部分单列 WABA 绑定、模板归属、质量评级、API 配置与第三方号码映射的影响；区分导出、迁移、平台间可携三层概念，提醒「法律上有可携权」不等于产品可一键迁移；末附迁移前中后 13 步清单、5 条误传纠正与 4 组未核实项
- **WhatsApp Cloud API 回调安全手册**：[android-whatsapp-cloud-api-webhook-security.md](/android-whatsapp-cloud-api-webhook-security/)
  把回调拆成订阅握手、载荷验签、至少一次投递、幂等与重放五段：verify token 只是握手暗号不是认证凭据；每次 POST 都必须用 App Secret 对原始请求体字节计算 HMAC-SHA256，与 `X-Hub-Signature-256` 头常量时间比较，先验签后解析；官方明确失败会以递减频率重试最长 7 天并产生重复通知，因此端点要快速返回 2xx、先入队后处理，并以「业务主键 + 单向状态机」判重，避免重复通知导致状态回退或重复发送；另讲清重放的时间窗与事件去重、App Secret 轮换期间的双密钥验签、日志脱敏与五类告警；含 5 条误传纠正与 4 组未核实项
- **群控/群发工具供应链风险手册**：[android-whatsapp-third-party-tool-supply-chain.md](/android-whatsapp-third-party-tool-supply-chain/)
  按接入方式而非功能多少给工具排风险谱系（官方 API、官方客户端人工操作、自动化驱动官方客户端、非官方修改版、云控/设备农场），指出每往右一步省下的成本都以账号资产与客户数据为抵押；引用官方立场说明非官方应用可能携带恶意软件、违反条款并导致账号受限；给出采购前的十二个书面问题（主体资质、接入通道、数据流、子处理者、签名更新、退出与责任）与六项技术验证（签名、权限、网络出口、更新机制、SDK、账号隔离）；合同部分覆盖 DPA、审计权、违规通知、变更通知与删除证明，事件响应按隔离、取证、评估、通报、轮换、复盘六步，特别提醒不要急着重装覆盖证据；含 5 条误传纠正与 4 组未核实项
- **WhatsApp 未成年人保护与年龄门槛手册**：[android-whatsapp-minor-protection-age-assurance.md](/android-whatsapp-minor-protection-age-assurance/)
  梳理 13 岁下限与部分地区的更高门槛、2018 年欧盟升至 16 岁与 2024 年英国/欧盟降回 13 岁的变化、2026 年家长管理账号与青少年监护工具的公开信息（细节标注未核实）；把群组邀请与频道列为未成年人最主要的陌生人接触面，给出家长/学校/企业/平台四方动作，指出年龄声明靠自报、技术限制只能降低概率，末附 5 条误传纠正与 4 组未核实项
- **WhatsApp Flows 数据交换安全手册**：[android-whatsapp-flows-data-exchange-security.md](/android-whatsapp-flows-data-exchange-security/)
  把嵌入式表单拆成消息下发、客户端渲染、数据回传三段信任边界：`flow_action` 只有 `navigate` 与 `data_exchange`（没有 `complete`）、健康检查是 POST ping、请求体用 AES-128-GCM 加密并以 RSA-OAEP-SHA256 包裹密钥、响应 IV 逐字节取反且失败返回 421、公钥登记在 `/{PHONE_NUMBER_ID}/whatsapp_business_encryption` 并用 `signature_status` 监控；讲清 `X-Hub-Signature-256` 属于 webhook 而非 Flow 端点、`flow_token` 无官方 TTL 必须自建一次性校验、7.3/4.0 起另有 JWT 校验；含重放、公钥替换、明文落库等七类风险表与 6 条误传纠正。
- **WhatsApp 商业目录与订单治理手册**：[android-whatsapp-catalog-order-governance.md](/android-whatsapp-catalog-order-governance/)
  讲清 `catalog_id`、`retailer_id` 与 `product_retailer_id` 的关系，单品/多品/目录三种消息的结构差异（多品最多 30 个商品、单品没有 `action.name`），入站 `order` 对象的字段与「通用场景没有 `order_status` 回执」这一边界（相关字段属于巴西 Payments，状态枚举 pending 至 canceled）；给出价格快照与库存漂移的治理流程、目录投毒与跨租户串目录的防护、webhook 验签与订单幂等要求；含 6 条误传纠正与 4 组未核实项。
- **Click-to-WhatsApp 归因与 referral 防伪手册**：[android-whatsapp-ctwa-attribution-integrity.md](/android-whatsapp-ctwa-attribution-integrity/)
  指出 `referral` 只出现在广告点击后的首条消息，逐字段说明 `source_url`/`source_id`/`source_type`/`media_type`/`ctwa_clid`（Status 广告位不返回 clid），并把 `ctwa_clid` 到 Conversions API 的路径讲清（dataset、`action_source=business_messaging`、14 个业务消息事件）；强调 7 天窗口只适用于哈希客户信息路径、Meta 不负责去重，拆开伪造 referral、`source_url` 注入、跨租户串单与隐私过度收集四类风险；含 6 条误传纠正与 4 组未核实项。
- **WhatsApp BSP 与 Embedded Signup 多租户治理手册**：[android-whatsapp-bsp-embedded-signup-tenancy.md](/android-whatsapp-bsp-embedded-signup-tenancy/)
  把服务商接入拆成授权、令牌与回调三条隔离线：Embedded Signup v4 的 `config_id` 与 `code`（30 秒有效期）、postMessage 事件枚举、v2/v3 已于 2026-10-15 弃用；区分 System User token 与客户维度 token 的泄露影响面，说明 webhook 默认是应用级、需按 WABA 订阅并按 `phone_number_id` 路由，给出 80→1,000 条/秒与 `TIER_250`…`TIER_UNLIMITED` 配额要点；含七类多租户风险表与客户侧验收七问。
- **WhatsApp 媒体与文档分享安全手册**：[android-whatsapp-media-document-sharing-security.md](/android-whatsapp-media-document-sharing-security/)
  把素材通道拆成上传、媒体 ID、临时 URL 三段，列出 30 天/7 天/5 分钟三个有效期与 image 5 MB、video/audio 16 MB、document 100 MB、sticker 100/500 KB 的限额；给出 EXIF 与元数据剥离、按文件头嗅探类型、宏与脚本清除、压缩炸弹检测、转码与哈希登记的流水线，讲清 `caption`/`filename` 的注入风险与入站媒体的隔离要求；含 6 条误传纠正与 4 组未核实项。
- **WhatsApp 会话保留、导出与合规删除手册**：[android-whatsapp-retention-export-deletion.md](/android-whatsapp-retention-export-deletion/)
  把「数据留多久」拆成个人端、频道与企业端三条线：导出聊天与请求账号信息是两件不同的事（后者约 3 天生成且不含消息），为所有人删除限 2 天、删号 90 天内完成但备份与去标识日志可能更久；临时消息 24 小时/7 天/90 天只作用于新消息、Keep 撤销期限存在 25 天与约 30 天两种口径、View once 可被翻拍或录屏；企业侧讲清 Cloud API 默认 30 天、No Storage 约 1 小时、Coexistence 可同步 6 个月单聊、条款明确不提供归档或备份；附可执行保留政策的五个维度与 6 条误传纠正。
- **WhatsApp 安全码与密钥透明手册**：[android-whatsapp-security-code-key-transparency.md](/android-whatsapp-security-code-key-transparency/)
  讲清安全码是每个单聊独立的 60 位数字与二维码、入口在联系人的 Encryption 页、换机/重装/增删链接设备都会触发变化、通知开关在 Account > Security notifications、未验证不影响收发；说明群组没有群安全码、群消息依赖 Sender Key 且成员离开后重建；梳理 2023-04-13 公布的密钥透明（只追加 AKD、约 5 分钟一个 epoch、S3 审计证明 WORM 保留 5 年、开源 AKD 库、客户端显示成功/待定/失败），并拆解「换机话术」等社工风险与 6 条误传纠正。
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
