---
layout: page
title: "Android 系统漏洞安全研究资料"
description: "从防御视角梳理 Android 系统攻击面、漏洞类型、典型案例、编号与公告体系、底层架构与研究方法的公开研究资料，含一份总览报告、十一份专题手册与一篇底层漏洞研究文章。"
permalink: /
---

> 全部文章可免费阅读；Markdown 源文件见 [https://github.com/douglasr864469/android-security-report](https://github.com/douglasr864469/android-security-report)，同一份资料同步发布在 GitLab 与 Bitbucket。

从防御视角梳理 Android 系统攻击面、漏洞类型、典型案例、编号与公告体系、底层架构与研究方法的公开研究资料，含一份总览报告、十一份专题手册与一篇底层漏洞研究文章。

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
