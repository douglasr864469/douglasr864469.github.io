---
layout: page
title: "Android 内核 keyring 与凭据缓存安全手册"
description: "讲内核钥匙环的模型与权限位，以及它与 Android 文件加密的关系：fscrypt v1 曾借助钥匙环传递主密钥，v2 与后续版本改走其他路径，Linux 6.1 更直接移除了这部分用法（CVE-2022-49899）；拆 `KEYCTL_*` 系列操作在 Android 上的可达性——Android..."
date: 2026-11-17
permalink: /android-kernel-keyring-credentials/
---

> 面向 Android 内核维护者、文件加密与密钥管理工程师的防御性说明
> 版本：2026-11
> 适用范围：讨论内核 keyring 子系统、它与 Android 文件级加密的关系、公开缺陷与缓解。**不包含**密钥提取、绕过加密或提权步骤。

---

## 0. Android 的密钥不只在 TEE 里

谈到 Android 密钥管理，人们通常先想到 TEE、StrongBox 与 Keystore。但设备上还有另一套**内核软件密钥**体系：keyring。

keyring 子系统负责在内核中保存、索引与授权访问「键」。它被网络、文件系统、加密与认证机制使用。Android 的文件级加密（FBE）在早期实现中就把 fscrypt 主密钥放进 keyring，由内核在读写加密文件时取用。

这意味着：keyring 不仅是「另一个密钥 API」，而是**磁盘加密密钥在运行期的落点之一**。它的权限模型、生命周期与历史缺陷，直接影响设备被解锁后的数据边界。

---

## 1. 模型与权限

### 1.1 keyring 的层级

- 每个进程有 thread keyring、process keyring、session keyring；
- 每个 UID 有 user keyring 与 user-session keyring；
- 特殊 ID 通过 `KEY_SPEC_*` 常量指定，例如 `KEY_SPEC_SESSION_KEYRING`。

键可以链接到多个 keyring，形成搜索路径。`request_key()` 会沿路径搜索，必要时调用 `/sbin/request-key` 由用户态程序实例化。

### 1.2 权限位

每个 key 有一组权限位：

| 位 | 含义 |
|---|---|
| `view` | 查看 key 的元数据 |
| `read` | 读取 key 内容 |
| `write` | 更新 key 内容 |
| `search` | 在 keyring 中搜索 |
| `link` | 把 key 链接到 keyring |
| `setattr` | 修改权限与属性 |

此外还有 possessor 条件：持有 key 的进程可能获得额外权限。权限组合错误是 keyring 历史缺陷的常见来源。

### 1.3 系统调用

- `add_key(type, description, payload, plen, keyring)`：创建或更新 key。
- `request_key(type, description, callout_info, dest_keyring)`：查找，找不到则请求实例化。
- `keyctl(operation, ...)`：搜索、链接、读取、撤销、设置权限等。
- `keyctl(KEYCTL_JOIN_SESSION_KEYRING, name)`：加入或创建会话 keyring。

---

## 2. 与 Android 文件级加密的关系

### 2.1 fscrypt 的两代密钥管理

**v1（早期实现）**：通过 `add_key` 把 fscrypt 主密钥以 `logon` 类型写入 session/user keyring，描述符形如 `fscrypt:<hex>`。内核在文件系统层按描述符查找密钥。

**v2（现行实现）**：改用 `FS_IOC_ADD_ENCRYPTION_KEY` ioctl 添加密钥。主密钥存放在**每个 superblock 的 keyring** 中，名称形如 `fscrypt-<sb->s_id>`；主密钥类型为 `._fscrypt`，每用户授权键类型为 `.fscrypt`。

因此，「Android 把密钥放在一个叫 fscrypt 的全局 keyring 里」是不准确的。具体名称、类型与层级随实现版本变化。

### 2.2 Linux 6.1 的去 keyring 化

Linux 6.1 起（并回移到 stable 5.10.154、5.15.78、6.0.8），fscrypt **不再使用 keyrings 子系统**保存主密钥，改为内部数据结构。公开原因是 keyring 路径存在释放后使用与死锁风险（参见 CVE-2022-49899 的记录）。

这对 Android 的意义是：

- 新内核上的 fscrypt 密钥生命周期由 fscrypt 自己管理，不再经过通用 keyring；
- 旧内核仍可能使用 keyring 路径，需要按版本审计；
- 不能把「keyring CVE」直接等同于「Android FBE 可被绕过」，还要看内核版本与实现路径。

### 2.3 硬件包裹密钥

Android 的 FBE 主密钥通常由硬件包裹（hardware-wrapped keys），TEE 或安全硬件参与解封。内核 keyring 保存的是解封后的运行期材料，而不是可直接落盘的明文。两者是不同层次，不能混为一谈。

---

## 3. 公开案例台账（逐条核实归属）

| CVE | 机制 | 缺陷类型 | 影响与修复 | 备注 |
|---|---|---|---|---|
| CVE-2016-0728 | `join_session_keyring()` | 引用计数处理缺陷导致 UAF/提权 | 影响 <4.4.1 | NVD 表述为对象引用处理不当；公开分析称可通过反复 join 造成 32 位引用计数溢出，不宜简单写成「整数溢出」 |
| CVE-2017-7374 | 撤销 ext4/f2fs/ubifs 加密 key | fs/crypto 释放后使用 | 修复于 4.10.7 一线 | 与 fscrypt 密钥撤销路径相关 |
| CVE-2017-17807 | `request_key()` | 目标 keyring 缺少写权限检查 | 修复于 4.14.6 一线 | 权限模型缺陷 |
| CVE-2024-50301 | `key_task_permission` | slab 越界读 | 修复于 5.10.230 / 5.15.172 / 6.1.117 / 6.6.61 等 | 权限判定路径 |

以上均为 Linux 内核缺陷。Android 设备是否受影响，取决于内核版本、是否启用 keyring、以及攻击者所在域是否可调用相关系统调用。

### 3.1 归因注意事项

- CVE-2016-0728 常被写成「Android 提权漏洞」。它是 Linux 内核缺陷，Android 设备可能承载，但需要具体版本与配置才能判断。
- CVE-2017-7374 与文件加密相关，但不等于「FBE 被破解」；它影响的是密钥撤销路径的生命周期。
- 不要把 keyring 缺陷与 Android Keystore/TEE 缺陷混为一谈，二者是不同的密钥体系。

---

## 4. Android 的访问控制

### 4.1 seccomp

Android O 起，应用进程的 seccomp 过滤器阻止 key 控制类系统调用。CTS 中有用例验证 `add_key`/`keyctl` 被拦截。这意味着普通第三方应用无法直接操作内核 keyring。

### 4.2 SELinux

keyring 的 `key` 类默认拒绝。AOSP 没有为普通应用提供宽泛的 key 权限；只有少数系统域按需授权。审计时应检查是否存在厂商策略扩大了 `key` 类权限。

### 4.3 调试接口

`/proc/keys` 与 `/proc/key-users` 可展示 key 元数据与配额。这些接口的可见性受权限与 SELinux 限制；在量产设备上应确认普通应用不可读。

---

## 5. 防护与检测

### 5.1 内核与系统

- 跟进上表 CVE 的稳定版修复，核对 GKI 分支与厂商内核。
- 新内核优先使用不依赖通用 keyring 的 fscrypt 实现；旧内核需单独评估 keyring 路径。
- 限制 key 配额，防止资源耗尽。
- 对 `request_key` 的 callout 程序做白名单与路径校验，避免被诱导执行不可信程序。

### 5.2 Android 策略

- 保持应用域对 `keyctl`/`add_key` 的 seccomp 拦截。
- 不让第三方应用读取 `/proc/keys`。
- 检查厂商策略是否给 `key` 类意外授权，尤其是 `system_app`、`vendor_app` 与调试域。
- 对文件加密密钥的添加、撤销与轮换做审计，记录调用域与时间。

### 5.3 设备审计清单

```text
1. 确认内核是否启用 keyring 与 fscrypt
2. 判断 fscrypt 实现走 keyring 还是内部结构（按内核版本）
3. 检查应用 seccomp 是否拦截 keyctl/add_key
4. 检查 SELinux key 类授权范围
5. 检查 /proc/keys 的可见性
6. 对照内核版本与上表修复
```

---

## 6. 常见误传纠正

1. **「内核 keyring 就是 Android Keystore」**——不是。Keystore 是应用框架 + TEE/StrongBox 的密钥服务；keyring 是内核软件密钥容器。
2. **「FBE 密钥一直存在名为 fscrypt 的全局 keyring 中」**——v1 使用 session/user keyring 的 `fscrypt:<hex>` 描述符；v2 使用 per-superblock keyring；6.1 起不再使用 keyring。
3. **「keyring 里的密钥是明文磁盘密钥」**——Android FBE 主密钥通常由硬件包裹，keyring 保存的是运行期材料。
4. **「CVE-2016-0728 是整数溢出」**——NVD 的表述是对象引用处理不当；公开分析提到 32 位引用计数溢出，但不宜简化为单一整数溢出。
5. **「普通应用可以读 /proc/keys」**——受权限与 SELinux 限制，量产设备上通常不可读。

---

## 7. 未核实项

- AOSP 文档未明文写出 fscrypt keyring 的名称/类型；文中的名称来自内核源码，需按版本核对。
- 各 Android GKI 分支是否已合入 Linux 6.1 的去 keyring 改写，未逐一核实。
- `request_key` 是否同样被 Android seccomp 拦截，未核实。
- 是否存在专门针对 keyctl 的 SELinux 明文规则；当前判断是 `key` 类默认拒绝为主边界。
- 各厂商内核对 key 配额与 `/proc/keys` 可见性的修改。

以上项目需在目标设备与内核源码上核实。

---

## 7.1 参考资料

- 内核 keyring 文档：<https://docs.kernel.org/security/keys/core.html>
- keyrings(7) 手册：<https://man7.org/linux/man-pages/man7/keyrings.7.html>
- Android 硬件包裹密钥：<https://source.android.com/docs/security/features/encryption/hw-wrapped-keys>
- fscrypt 文档（5.8）：<https://www.kernel.org/doc/html/v5.8/filesystems/fscrypt.html>
- fscrypt keyring 源码（5.4）：<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/fs/crypto/keyring.c?h=v5.4>
- CVE-2022-49899：<https://www.cve.org/CVERecord?id=CVE-2022-49899>
- CVE-2016-0728：<https://nvd.nist.gov/vuln/detail/CVE-2016-0728>
- Red Hat CVE-2016-0728：<https://access.redhat.com/security/cve/cve-2016-0728>
- CVE-2017-7374：<https://nvd.nist.gov/vuln/detail/CVE-2017-7374>
- CVE-2017-17807：<https://nvd.nist.gov/vuln/detail/CVE-2017-17807>
- CVE-2024-50301：<https://www.cve.org/CVERecord?id=CVE-2024-50301>
- Android O seccomp：<https://android-developers.googleblog.com/2017/07/seccomp-filter-in-android-o.html>

---

## 联系方式

咨询详情请联系 https://t.me/one00190

## 免责声明

本手册面向防御与合规用途，仅讨论内核 keyring 的模型、Android 文件加密关系、公开缺陷与缓解。文中不包含密钥提取、绕过加密或提权步骤。内核实现与 Android 策略会随版本变化，请以设备实际构建与官方文档为准；标注为「未核实」的内容不得作为安全结论的依据。
