---
layout: page
title: "Android 内核 io_uring 攻击面与缓解手册"
description: "讲共享环、注册缓冲区、超时与取消、文件描述符注册四类机制如何在并发与生命周期上制造缺陷；按内核配置、SELinux 域授权、seccomp 允许清单三层判断暴露条件，指出「内核支持 io_uring」不等于「应用可调用」；列 CVE-2021-3491、CVE-2021-41073、CVE-2022-2..."
date: 2026-11-17
permalink: /android-kernel-io-uring/
---

> 面向 Android 内核维护者、系统安全工程师与设备厂商的防御性说明
> 版本：2026-11
> 适用范围：讨论 Linux io_uring 子系统在 Android 设备上的暴露条件、已公开缺陷类型、检测与缓解。**不包含**任何利用代码、绕过沙箱的方法或提权步骤。

---

## 0. 为什么要把 io_uring 单独拿出来讲

io_uring 是 Linux 5.1 引入的异步 I/O 接口。它把提交队列与完成队列放在用户态与内核共享的内存环里，应用只需少量系统调用就能批量提交读写、网络、超时等操作。对高吞吐服务来说，这大幅减少了系统调用与上下文切换开销。

但对 Android 这种「应用默认不可信」的系统而言，它带来两个问题：

1. **内核攻击面变大。** 共享环、注册缓冲区、超时、取消、文件描述符注册等机制都要处理并发与生命周期，历史上多次出现越界、类型混淆与释放后使用。
2. **暴露条件不直观。** 应用能不能调用 io_uring，不取决于 Java API，而取决于内核配置、SELinux 域授权与 seccomp 允许清单三层。只看「内核支持 io_uring」就下结论，通常会误判。

这篇手册把「机制—暴露条件—公开案例—缓解」分开讲，便于在设备审计时逐层核对。

---

## 1. 组件与执行路径

### 1.1 三个核心系统调用

- `io_uring_setup(entries, params)`：创建 io_uring 实例，返回一个文件描述符。内核与用户态通过 `mmap` 共享提交队列（SQ）与完成队列（CQ）。
- `io_uring_enter(fd, to_submit, min_complete, flags, sig, sigsz)`：提交请求、等待完成，或同时做两件事。
- `io_uring_register(fd, opcode, arg, nr_args)`：注册缓冲区、文件、事件 fd 等长期资源。

此外还有 `io_uring` 的 `IORING_OP_*` 操作码，覆盖 read/write、send/recv、accept、timeout、cancel、splice、openat 等。操作码越多，内核侧需要校验的参数组合越多。

### 1.2 共享内存环

SQ/CQ 的索引、数组与 SQE/CQE 结构位于用户态可写、内核可读（或反向）的共享页中。内核必须把用户提供的索引、数组下标、缓冲区地址与长度全部视为不可信输入。任何一次漏检，都可能让用户态越界读写内核内存。

### 1.3 Android 上的三层门禁

| 层 | 作用 | 需要核对的事实 |
|---|---|---|
| 内核配置 | 决定 io_uring 是否编进内核 | `CONFIG_IO_URING` 实际生效值 |
| SELinux | 决定某个域能否创建/使用 io_uring anon inode | `class io_uring`、`io_uring_use()` 授权 |
| seccomp | 决定进程能否发起三个系统调用 | bionic 允许清单是否包含 `io_uring_*` |

三层是「与」关系：任何一层拒绝，普通应用都无法直接使用 io_uring。

---

## 2. 已核实的 Android 暴露情况

### 2.1 内核配置

对 AOSP 公共内核仓库的 `gki_defconfig` 核对结果：

- android12-5.10、android13-5.15、android14-6.1 的 `gki_defconfig` 中**没有** `CONFIG_IO_URING` 条目；
- 同一源码树的 `init/Kconfig` 中，该选项的默认值为 `y`，但只在 EXPERT 配置界面显示提示。

这意味着：**不能用「defconfig 没写」推断「一定没启用」**。Kconfig 默认值、厂商 fragment、`savedefconfig` 与最终 `.config` 都可能改变结果。设备审计时应直接读取运行内核的配置（若允许）或构建产物 `.config`，而不是只读 defconfig。

### 2.2 SELinux

AOSP 的 SELinux 策略定义了 `class io_uring` 与 `io_uring_use()` 宏。当前公开策略中，只有少数域获得授权，例如 fastbootd、snapuserd、statsd。`app_neverallows.te` 对 `all_untrusted_apps` 设置了针对 `anon_inode` 的 neverallow，因此普通第三方应用无法创建或使用 io_uring 的 anon inode。

这一层是 Android 与通用 Linux 发行版最大的差别：即使内核启用 io_uring，应用沙箱默认也拿不到入口。

### 2.3 seccomp

bionic 的 `SECCOMP_ALLOWLIST_COMMON.TXT` 与 `SYSCALLS.TXT` 中**没有** `io_uring_setup`、`io_uring_enter`、`io_uring_register`。由 zygote 派生的应用进程会继承 seccomp 过滤器，这三条系统调用会被拒绝。

因此，一个普通应用的调用链通常是：

```
Java/Kotlin 代码
  └─ 原生库（若有）
      └─ syscall(io_uring_setup)
          ├─ seccomp: 拒绝
          ├─ SELinux: 即使 seccomp 放行，也会因域无 io_uring 授权而拒绝
          └─ 内核: 只有前两层都放行才到达
```

### 2.4 仍需要关注的进程

系统原生守护进程、厂商服务、调试与工厂工具可能拥有不同的 seccomp/SELinux 配置。只要某个进程被授权使用 io_uring，内核缺陷就重新变得可达。审计时应列出「实际持有 io_uring 授权的域」，而不是只检查应用。

---

## 3. 公开案例台账（逐条核实归属）

| CVE | 机制 | 缺陷类型 | 影响与修复 | 备注 |
|---|---|---|---|---|
| CVE-2021-3491 | `IORING_OP_PROVIDE_BUFFERS` | 堆越界写（CWE-787） | 修复于 5.12.4 / 5.11.21 / 5.10.37 | 用户提供的缓冲区数量/长度未正确截断 |
| CVE-2021-41073 | `PROVIDE_BUFFERS` 生命周期 | 类型混淆/释放问题（CWE-763） | 影响 5.10–5.14.6 一线 | 与缓冲区注册状态有关 |
| CVE-2022-29582 | `IORING_OP_TIMEOUT` | 竞态导致释放后使用（CWE-362） | 修复于 5.17.3 / 5.16.20 / 5.15.34 / 5.10.111 | 并发取消与超时路径 |
| CVE-2023-2598 | 注册固定缓冲区 | 越界访问物理内存（CWE-416/787） | 修复于 6.3.2 | 固定缓冲区与页管理假设不一致 |

以上均为 **Linux 内核通用缺陷**，不是 Android 专属漏洞。是否影响某台 Android 设备，取决于该设备内核是否包含 io_uring、是否合入对应稳定版修复，以及是否存在可调用 io_uring 的域。把它们直接写成「Android 提权漏洞」会夸大适用范围。

### 3.1 一个常见的归因错误

二手资料常把「Android 内核基于 Linux」等同于「所有 Linux 内核 CVE 都能在 Android 上触发」。实际判断至少要问三个问题：

1. 该内核版本是否包含出问题的代码路径？
2. 该设备是否启用了对应配置（如 io_uring）？
3. 攻击者所在域是否有权调用入口系统调用？

三问中任一为否，就不能把 CVE 直接算作该设备的可用攻击面。

---

## 4. 防护与检测

### 4.1 内核侧

- **能不用就不用。** 如果产品没有明确的 io_uring 需求，构建时关闭 `CONFIG_IO_URING`，从根源移除攻击面。
- **及时跟进稳定版。** io_uring 的修复通常先进入稳定分支，再进入各 GKI 分支。设备厂商应核对自身内核版本是否包含上表修复。
- **使用运行时开关。** Linux 6.6 起提供 `kernel.io_uring_disabled`：
  - `0`：不限制；
  - `1`：只允许指定组的进程创建新实例；
  - `2`：全面禁止创建新实例。

  老内核没有该开关时，只能依赖配置裁剪、seccomp 与 SELinux。

### 4.2 Android 策略侧

- 保持应用域的 `io_uring_use()` 未授权状态，不要为了性能试验给第三方应用开口。
- 对确实需要 io_uring 的原生服务，单独定义域并只授予该域，避免复用 `untrusted_app` 或宽泛的 `system_server` 权限。
- 在 seccomp 允许清单中继续排除三个 `io_uring_*` 系统调用；若厂商自定义清单，逐条核对是否被加入。
- 审计 `audit.log` 中的 `io_uring` 相关拒绝记录，以及异常 anon inode 创建。

### 4.3 设备审计清单

```text
1. 读取运行内核配置：CONFIG_IO_URING 是否为 y/m
2. 检查 /proc/sys/kernel/io_uring_disabled（若存在）
3. 列出拥有 io_uring_use() 授权的 SELinux 域
4. 核对目标进程的 seccomp 允许清单
5. 对照内核版本与上表修复版本
6. 检查是否有厂商服务自行放行 io_uring
```

---

## 5. 常见误传纠正

1. **「Android 应用可以直接使用 io_uring」**——公开策略与 bionic 允许清单都不支持这一说法。应用域默认被 SELinux 与 seccomp 双重拦截。
2. **「gki_defconfig 没写 CONFIG_IO_URING，所以 GKI 一定没有 io_uring」**——defconfig 不是最终配置，Kconfig 默认值与厂商 fragment 都可能改变结果。
3. **「io_uring 只影响服务器，与手机无关」**——内核代码是共用的；手机是否受影响取决于配置与域授权，而不是设备形态。
4. **「CVE-2021-3491 是 Android 专属漏洞」**——它是 Linux 内核缺陷，Android 只是可能的承载平台之一。
5. **「关闭 io_uring 会影响所有异步 I/O」**——常规的 epoll、AIO、线程池与阻塞 I/O 不依赖 io_uring；关闭它只影响显式使用该接口的组件。

---

## 6. 未核实项

- GKI 最终二进制是否实际启用 io_uring：defconfig 未写、Kconfig 默认 y 的推导不等同于官方声明。
- 各厂商内核是否通过 fragment 另行启用 io_uring。
- 非 zygote 派生的原生守护进程中，是否存在未公开的 io_uring 使用与授权。
- `kernel.io_uring_disabled` 在各 Android GKI 分支的实际可用性与默认值。

以上项目在设备审计时应以实测为准，不得当作既定事实引用。

---

## 7. 参考资料

- io_uring 系统调用手册：<https://man7.org/linux/man-pages/man2/io_uring_setup.2.html>
- Linux 5.1 发布说明：<https://kernelnewbies.org/Linux_5.1>
- AOSP 公共内核 gki_defconfig：<https://raw.githubusercontent.com/aosp-mirror/kernel_common/android14-6.1/arch/arm64/configs/gki_defconfig>
- AOSP SELinux `te_macros`：<https://android.googlesource.com/platform/system/sepolicy/+/refs/heads/main/public/te_macros>
- AOSP `app_neverallows.te`：<https://android.googlesource.com/platform/system/sepolicy/+/refs/heads/main/private/app_neverallows.te>
- bionic seccomp 允许清单：<https://raw.githubusercontent.com/aosp-mirror/platform_bionic/master/libc/SECCOMP_ALLOWLIST_COMMON.TXT>
- 内核 sysctl 文档：<https://docs.kernel.org/admin-guide/sysctl/kernel.html>
- CVE-2021-3491：<https://nvd.nist.gov/vuln/detail/CVE-2021-3491>
- CVE-2021-41073：<https://nvd.nist.gov/vuln/detail/CVE-2021-41073>
- CVE-2022-29582：<https://nvd.nist.gov/vuln/detail/CVE-2022-29582>
- CVE-2023-2598：<https://nvd.nist.gov/vuln/detail/CVE-2023-2598>

---

## 联系方式

咨询详情请联系 https://t.me/one00190

## 免责声明

本手册面向防御与合规用途，仅讨论 io_uring 在 Android 上的暴露条件、公开缺陷类型与缓解措施。文中不包含利用代码、绕过沙箱或提权步骤。内核配置与策略会随版本变化，请以设备实际构建与官方文档为准；标注为「未核实」的内容不得作为安全结论的依据。
