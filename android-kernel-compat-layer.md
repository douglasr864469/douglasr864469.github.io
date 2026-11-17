---
layout: page
title: "Android 内核 32 位兼容层漏洞面手册"
description: "说明 64 位设备上 `compat_sys_*`、`compat_ptr`、`compat_ioctl` 这层「翻译层」为何天然容易出错：同一份用户数据被按两套结构体解读，指针宽度与对齐差异在转换时最容易丢失校验；梳理 Android 从 32 位到 64 位的时间线与当前仍保留兼容路径的组件；列 C..."
date: 2026-11-17
permalink: /android-kernel-compat-layer/
---

> 面向 Android 内核维护者、系统安全工程师与设备厂商的防御性说明
> 版本：2026-11
> 适用范围：讨论 64 位内核运行 32 位进程时的 compat 转换层、易错点、公开案例与 64 位化趋势。**不包含**利用代码或提权步骤。

---

## 0. 一条不常被注意的转换路径

现代 Android 设备普遍使用 64 位内核，但仍然可能运行 32 位应用与 32 位原生库。为了让 32 位进程在 64 位内核上工作，Linux 提供了 **compat 兼容层**。

它做的事情很具体：把 32 位进程传入的系统调用号、指针、结构体与 ioctl 命令，转换成内核 64 位实现能理解的形式。问题也正出在这里：

- 32 位指针需要零扩展，而不是符号扩展；
- 32 位的 `long`、`time_t`、`off_t` 与 64 位宽度不同；
- 结构体对齐、填充与大小在两种 ABI 下可能不一致；
- ioctl 命令号按字长编码，转换错一位就会调用到不同的处理函数；
- compat 路径的测试覆盖通常少于原生 64 位路径。

对攻击者而言，compat 层是一个「同一功能、两套入口」的额外表面。对防守方而言，它是 64 位化进程中需要主动缩减的历史包袱。

---

## 1. compat 层如何工作

### 1.1 调用链

一个 32 位进程发起系统调用时，内核根据调用号进入 `compat_sys_*` 包装函数。包装函数负责：

1. 从 32 位用户栈读取参数；
2. 用 `compat_ptr()` 把 32 位地址零扩展为 64 位指针；
3. 把 `struct compat_*` 转换成内核内部结构；
4. 调用真正的 64 位实现；
5. 把结果转换回 32 位布局。

### 1.2 常见转换点

- `compat_sys_ioctl` 与各驱动的 `compat_ioctl`；
- `compat_sys_recvmsg`/`sendmsg` 的 iovec 与 msghdr；
- 时间相关结构（`time_t`、`timespec`、`timeval`）；
- 文件偏移与 `stat` 结构；
- 信号、调度、资源限制结构；
- 32 位 `semctl`/`msgctl`/`shmctl` 的联合体参数。

### 1.3 为什么容易出错

| 易错点 | 典型后果 |
|---|---|
| 指针符号扩展 | 用户可控地址被解释成内核高地址 |
| 结构体大小不一致 | 复制越界或未初始化字段泄漏 |
| 对齐差异 | 字段错位，语义改变 |
| 命令号转换错误 | 调用错误的 ioctl 处理器 |
| 未初始化填充 | 内核栈内容泄漏到用户态 |
| 32/64 位分支不一致 | 一套路径有校验，另一套没有 |

---

## 2. Android 的 32 位现状

### 2.1 时间线（已核实）

- Android 5.0 起支持 64 位。
- Google Play 自 2019-08-01 起要求含原生代码的新应用与更新提供 64 位版本。
- Android 12 起，模拟器镜像仅提供 64 位。
- Pixel 7 / Pixel 7 Pro 是首批仅支持 64 位应用的手机（Android 13）。
- Android 13/14 提供 64 位-only 的测试镜像，推动生态迁移。

### 2.2 现状判断

Android 并没有一个「某版本起全面移除 32 位 ABI」的统一时间表。实际状态是：

- 新设备与新镜像逐步转向 64 位-only；
- 存量设备与部分市场仍需要运行 32 位应用；
- 因此 64 位内核中的 compat 层在相当长时间内仍会存在。

这意味着 compat 攻击面不会因为「新旗舰是 64 位-only」而自动消失。设备审计要按具体产品判断。

---

## 3. 公开案例台账（逐条核实归属）

| CVE | 机制 | 缺陷类型 | 影响与修复 | 备注 |
|---|---|---|---|---|
| CVE-2014-0038 | `compat_sys_recvmmsg` | 超时指针处理错误，可任意写/提权 | 修复于 3.13.2 一线 | 需要 `CONFIG_X86_X32`，主要影响 x86_64 |
| CVE-2010-4073 | `compat_sys_semctl/msgctl/shmctl` | 未初始化结构泄漏内核栈 | 修复于 2.6.37 一线 | 信息泄漏 |
| CVE-2007-2878 | 64 位内核 VFAT compat ioctl | 破坏 `kernel_dirent`，拒绝服务 | 修复于 2.6.21.2 一线 | 文件系统 compat 路径 |

### 3.1 归因说明

以上均为**通用 Linux 内核 CVE**，不是 Android 专属漏洞。截至本次核验，没有找到 NVD 或 Android 安全公告明确把某个 CVE 归因于「Android 的 compat 层」。

因此，正确表述是：

- compat 层在 Android 内核中存在；
- 历史上 compat 路径出过真实缺陷；
- 具体设备是否受影响，要看内核版本、是否启用 `CONFIG_COMPAT`、以及是否存在可利用的 32 位进程。

不要把通用 Linux compat CVE 直接写成「Android 提权漏洞」。

---

## 4. 防护与检测

### 4.1 架构层面

- **推进 64 位-only。** 这是最彻底的缩减方式：没有 32 位进程，就没有 compat 调用路径。
- 若产品必须支持 32 位，明确列出仍然依赖 32 位的应用与原生库，评估其必要性。
- 对 32 位进程单独做威胁建模，不要假设它们与 64 位进程具有相同风险。

### 4.2 内核实现

- 审查 `compat_ioctl` 与 `compat_sys_*` 的每个转换点，确保指针零扩展、结构体大小与对齐正确。
- 对复制到用户态的结构体做完整初始化，避免填充字节泄漏内核栈。
- 用 `static_assert`/`BUILD_BUG_ON` 固定 compat 结构大小。
- 让 compat 路径与原生路径共享尽可能多的校验逻辑，避免两套实现漂移。
- 用 KASAN、KMSAN 与 syzkaller 的 compat 模式覆盖 32 位入口。

### 4.3 Android 策略

- 对不需要 32 位的系统进程，只提供 64 位版本。
- 在 seccomp 层限制 32 位进程可用的系统调用集合。
- 监控异常：32 位进程调用冷门 compat ioctl、频繁触发 `-ENOSYS`/`-EINVAL`、或试图访问未预期的设备节点。

### 4.4 审计清单

```text
1. 确认内核是否启用 CONFIG_COMPAT
2. 列出设备上仍安装的 32 位应用与原生库
3. 检查 32 位进程的 seccomp 策略
4. 审查厂商驱动新增的 compat_ioctl
5. 用 fuzz 覆盖 compat 系统调用与 ioctl
6. 对照内核版本与上表修复
```

---

## 5. 常见误传纠正

1. **「64 位内核就不存在 32 位攻击面」**——只要运行 32 位进程，compat 层就在。
2. **「所有 Android CVE 都与 compat 有关」**——多数漏洞与 compat 无关；本手册只讨论这一条特定路径。
3. **「compat 层只影响 x86」**——`CONFIG_COMPAT` 在 arm64 上同样存在，用于运行 32 位 ARM 进程。
4. **「64 位-only 设备完全没有兼容代码」**——内核可能仍编入 compat 支持，只是没有 32 位用户态进程；具体配置需核对。
5. **「compat ioctl 只是转发，不会引入新问题」**——命令号、结构布局与权限检查都可能在转换中出错。

---

## 6. 未核实项

- 没有找到 NVD 或 Android 安全公告明确归因于 compat 层的 Android CVE；文中的 CVE 均为通用 Linux 案例。
- 具体设备是否启用 `CONFIG_COMPAT` 未逐一核实；64 位内核运行 32 位应用通常需要启用。
- Android 整体移除 32 位 ABI 的官方时间表未见明确表述；现状是设备与镜像层面逐步 64 位-only。
- 各厂商驱动新增 `compat_ioctl` 的实现质量未审计。

以上项目需按目标产品与内核源码核实。

---

## 7. 参考资料

- 内核新增系统调用文档：<https://docs.kernel.org/process/adding-syscalls.html>
- compat 系统调用说明：<https://kernel-internals.org/syscalls/compat/>
- Google Play 64 位要求：<https://developer.android.com/google/play/requirements/64-bit>
- Android 64 位迁移博客：<https://android-developers.googleblog.com/2019/01/get-your-apps-ready-for-64-bit.html>
- CVE-2014-0038：<https://nvd.nist.gov/vuln/detail/CVE-2014-0038>
- CVE-2010-4073：<https://nvd.nist.gov/vuln/detail/CVE-2010-4073>
- CVE-2007-2878：<https://nvd.nist.gov/vuln/detail/CVE-2007-2878>

---

## 联系方式

咨询详情请联系 https://t.me/one00190

## 免责声明

本手册面向防御与合规用途，仅讨论 32 位兼容层的转换机制、易错点、公开案例与 64 位化趋势。文中不包含利用代码或提权步骤。内核配置与 Android 生态策略会随版本变化，请以设备实际构建与官方文档为准；标注为「未核实」的内容不得作为安全结论的依据。
