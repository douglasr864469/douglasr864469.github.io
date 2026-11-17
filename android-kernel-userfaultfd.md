---
layout: page
title: "Android 内核 userfaultfd 与缺页处理攻击面手册"
description: "讲缺页处理被用户态接管后的边界：区分 `UFFD_USER_MODE_ONLY` 与全权限模式，说明 `unprivileged_userfaultfd` 默认 0 的语义变化及其对旧应用的影响；核对 GKI 开启 `CONFIG_USERFAULTFD`、bionic seccomp 仍放行该系统调用..."
date: 2026-11-17
permalink: /android-kernel-userfaultfd/
---

> 面向 Android 内核维护者、系统安全工程师与运行时开发者的防御性说明
> 版本：2026-11
> 适用范围：讨论 userfaultfd 的缺页接管机制、它在 Android 上的暴露条件、公开缺陷与缓解。**不包含**任何竞态利用代码或提权步骤。

---

## 0. 一个「合法」的缺页接口为什么危险

正常进程访问尚未映射或尚未换入的页面时，内核会自行分配物理页、从 swap 或文件读入数据，然后恢复执行。整个过程对用户态透明。

userfaultfd 改变了这件事：它允许用户态注册一段虚拟地址范围，当这段范围发生缺页时，内核不自行处理，而是把缺页事件交给用户态线程，由该线程决定「提供哪一页」「复制哪一页」或「继续等待」。

对开发者来说，这是实现用户态分页、热迁移、快照与垃圾回收的工具。对安全研究来说，它提供了一个**可以精确暂停内核线程**的能力：当内核正在 `copy_from_user` 或 `copy_to_user` 的中途发生缺页，内核线程会停下来等用户态回应。这个停顿把原本极短的竞态窗口放大到可控长度。

因此，userfaultfd 本身通常不是漏洞，而是**利用原语**。真正需要盯住的是：哪些进程能拿到它，以及内核在暂停/恢复路径上是否正确处理并发状态。

---

## 1. 机制与系统调用

### 1.1 基本流程

1. 调用 `userfaultfd(flags)` 创建 userfaultfd 对象，得到文件描述符。
2. 通过 `ioctl(UFFDIO_API)` 协商 API 与特性。
3. 通过 `ioctl(UFFDIO_REGISTER)` 注册一段虚拟地址范围，并指定要接管的事件类型（缺页、写保护、minor/major 缺页等）。
4. 用户态线程读取 fd 上的事件，得知「哪个地址、什么类型、哪个 faulting 线程」。
5. 用 `UFFDIO_COPY`、`UFFDIO_ZEROPAGE`、`UFFDIO_CONTINUE`、`UFFDIO_WRITEPROTECT` 等 ioctl 提供页面或改变状态。
6. 内核恢复被暂停的线程。

### 1.2 与常规缺页的区别

| 维度 | 常规缺页 | userfaultfd 接管 |
|---|---|---|
| 处理者 | 内核 | 用户态线程 |
| 时延 | 微秒级 | 取决于用户态调度，可被人为拉长 |
| 可观测性 | 通常不可见 | 每次缺页都有事件 |
| 风险 | 内核缺页路径缺陷 | 竞态窗口放大 + 用户态与内核状态不一致 |

### 1.3 非特权限制

`/proc/sys/vm/unprivileged_userfaultfd` 控制非特权使用：

- 默认值 `0`：没有 `CAP_SYS_PTRACE` 的进程只能处理**用户态缺页**，调用时必须带 `UFFD_USER_MODE_ONLY`；
- 值 `1`：取消该限制；
- 某些内核还提供 `/dev/userfaultfd`，按设备权限控制。

这条 sysctl 是 Linux 侧的第一道闸门，但它不是 Android 的唯一闸门。

---

## 2. Android 上的暴露情况

### 2.1 内核配置

AOSP 公共内核的 GKI 配置中，android12-5.10、android13-5.15、android14-6.1 的 `gki_defconfig` 均设置 `CONFIG_USERFAULTFD=y`。也就是说，**能力本身在 GKI 中是编入的**。

### 2.2 seccomp

bionic 的 `SECCOMP_ALLOWLIST_COMMON.TXT` 包含 `userfaultfd(int) all`。这意味着由 zygote 派生的进程在 seccomp 层允许调用 `userfaultfd`。这一点与 io_uring 不同：io_uring 的三个系统调用不在允许清单里，而 userfaultfd 在。

### 2.3 SELinux

AOSP 为 userfaultfd 定义了按域生成的 anon inode 类型，并通过 `userfaultfd_use()` 宏授权。公开策略中，获得授权的主要是：

- zygote、app_zygote、webview_zygote；
- system_server；
- dex2oat 等运行时组件。

`untrusted_app` 受 `neverallow` 针对 `anon_inode` 的限制，普通第三方应用域默认不能创建 userfaultfd anon inode。

### 2.4 一个需要说明的张力

AOSP ART 中存在「Android S 起默认启用 userfaultfd 用于并发标记压缩垃圾回收（CMC GC）」的提交。运行时确实可能在受控场景下使用 userfaultfd，但公开策略同时又限制 `untrusted_app` 的 anon inode。两者如何在同一进程模型中协调，官方没有给出面向普通应用的完整说明。

因此，正确表述是：

- **能力已编入 GKI；**
- **seccomp 允许普通进程调用；**
- **SELinux 对应用域默认拒绝；**
- **运行时内部是否存在受控使用，需要按版本与实现核实。**

不能简单写成「Android 应用可以用 userfaultfd」，也不能写成「Android 完全禁用了 userfaultfd」。

---

## 3. 公开案例台账（逐条核实归属）

| CVE | 组件 | 缺陷类型 | 影响与修复 | 备注 |
|---|---|---|---|---|
| CVE-2017-15126 | `fs/userfaultfd.c` | 释放后使用（UAF） | 修复于 4.13.6 一线 | 缺页事件与对象生命周期处理不当 |
| CVE-2018-18397 | `UFFDIO_*` ioctl | 访问控制不当 | 修复于 4.19.7 一线 | 非特权路径可执行不应允许的操作 |
| CVE-2021-47461 | writeprotect 与 `exit_mmap` | 竞态 | 后续稳定版修复 | 进程退出与写保护并发 |

以上均为 Linux 内核缺陷。它们是否影响 Android 设备，取决于内核版本是否包含修复、配置是否启用 userfaultfd，以及攻击者所在域是否被授权。

### 3.1 更值得关注的不是 CVE，而是原语

userfaultfd 在公开研究中的主要角色，是配合其他内核缺陷构造稳定利用：

- 在 `copy_from_user` 期间暂停，制造「用户态检查」与「内核使用」之间的时间差；
- 在 `copy_to_user` 期间暂停，观察或改变内核读取到的用户内存；
- 与 `mremap`、`fork`、`exit_mmap`、写保护等路径组合，制造对象生命周期错位。

这类用法本身不依赖 userfaultfd 的实现缺陷，因此不会对应某个「userfaultfd CVE」。把每个相关利用都归因于 userfaultfd 是错误归因。

---

## 4. 防护与检测

### 4.1 内核与系统配置

- 保持 `vm.unprivileged_userfaultfd=0`，并要求非特权调用使用 `UFFD_USER_MODE_ONLY`。
- 若产品不需要用户态分页，考虑在构建时关闭 `CONFIG_USERFAULTFD`；但要注意 ART 的 GC 实现可能依赖它，关闭前必须验证运行时兼容性。
- 跟进上表 CVE 的稳定版修复，核对 GKI 分支与厂商内核。

### 4.2 Android 策略

- 保持 `untrusted_app` 对 userfaultfd anon inode 的 neverallow，不要为兼容性测试放宽。
- 只给确实需要的运行时域授予 `userfaultfd_use()`，并在域定义中写清用途。
- 在 seccomp 层评估是否需要对非运行时域收紧 `userfaultfd`；若收紧，必须验证 ART、WebView 与 dex2oat 不受影响。
- 监控异常：短时间内大量 `userfaultfd` 创建、跨进程 fd 传递、与 `mremap`/`exit_mmap` 同时出现的可疑序列。

### 4.3 运行时侧

- 垃圾回收器等合法使用者应使用最小权限、限定地址范围，并在结束后及时注销与关闭 fd。
- 不要把 userfaultfd 暴露给应用可调用的 Binder 接口或第三方原生库。
- 对注册范围做边界检查，避免覆盖运行时自身的关键内存。

### 4.4 审计清单

```text
1. 读取 vm.unprivileged_userfaultfd 当前值
2. 确认内核是否启用 CONFIG_USERFAULTFD
3. 列出获得 userfaultfd_use() 授权的域
4. 核对 bionic seccomp 允许清单中的 userfaultfd 条目
5. 检查是否有厂商服务向应用传递 userfaultfd
6. 对照内核版本与上表修复
```

---

## 5. 常见误传纠正

1. **「userfaultfd 是 Android 专属漏洞」**——它是 Linux 通用机制，Android 只是承载平台之一。
2. **「seccomp 允许就等于应用能用」**——SELinux 仍然按域拒绝 anon inode；两层都要看。
3. **「Android 完全禁用了 userfaultfd」**——GKI 编入了该能力，ART 也存在受控使用；准确说法是「应用域默认不可用」。
4. **「所有 userfaultfd 相关利用都对应一个 CVE」**——多数是利用原语，不是接口实现缺陷。
5. **「设置 unprivileged_userfaultfd=0 就万事大吉」**——有 `CAP_SYS_PTRACE` 的进程仍可使用；权限边界仍需 SELinux 与 seccomp 配合。

---

## 6. 未核实项

- 普通 `untrusted_app` 运行时是否通过继承 fd 等方式实际使用 userfaultfd；AOSP ART 的 CMC GC 提交与公开 SELinux 限制之间的关系，官方没有面向应用的完整说明。
- 各厂商内核对 `vm.unprivileged_userfaultfd` 默认值的修改情况。
- ART 在具体 Android 版本中启用 CMC GC 的范围与回退策略。
- 是否存在未公开的、向应用暴露 userfaultfd 的厂商接口。

以上项目需在目标设备上实测或查阅厂商源码，不得直接引用为结论。

---

## 7. 参考资料

- userfaultfd 手册：<https://man7.org/linux/man-pages/man2/userfaultfd.2.html>
- 内核用户态缺页文档：<https://docs.kernel.org/admin-guide/mm/userfaultfd.html>
- 内核 sysctl 文档：<https://docs.kernel.org/admin-guide/sysctl/vm.html>
- AOSP 公共内核 gki_defconfig：<https://raw.githubusercontent.com/aosp-mirror/kernel_common/android14-6.1/arch/arm64/configs/gki_defconfig>
- bionic seccomp 允许清单：<https://raw.githubusercontent.com/aosp-mirror/platform_bionic/master/libc/SECCOMP_ALLOWLIST_COMMON.TXT>
- AOSP SELinux `te_macros`：<https://android.googlesource.com/platform/system/sepolicy/+/refs/heads/main/public/te_macros>
- AOSP `app_neverallows.te`：<https://android.googlesource.com/platform/system/sepolicy/+/refs/heads/main/private/app_neverallows.te>
- ART 相关提交：<https://android.googlesource.com/platform/art/+/854cb7d94594f027cf0f056d6cd023e7a00df0cd>
- CVE-2017-15126：<https://nvd.nist.gov/vuln/detail/CVE-2017-15126>
- CVE-2018-18397：<https://nvd.nist.gov/vuln/detail/CVE-2018-18397>
- CVE-2021-47461：<https://nvd.nist.gov/vuln/detail/CVE-2021-47461>

---

## 联系方式

咨询详情请联系 https://t.me/one00190

## 免责声明

本手册面向防御与合规用途，仅讨论 userfaultfd 的机制、Android 暴露条件、公开缺陷与缓解。文中不包含竞态利用代码、提权步骤或绕过沙箱的方法。内核配置与 SELinux 策略会随版本变化，请以设备实际构建与官方文档为准；标注为「未核实」的内容不得作为安全结论的依据。
