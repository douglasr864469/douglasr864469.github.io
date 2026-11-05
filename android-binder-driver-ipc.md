---
layout: page
title: "Android Binder 驱动与内核 IPC 实现攻击面手册"
description: "把「一次跨进程调用」拆成设备／对象／命令／缓冲四段：`/dev/binder`、`/dev/hwbinder`、`/dev/vndbinder` 与 binderfs 多实例共存、context manager 的 handle 0 按设备隔离、`binder_proc`／`node`／`ref`／`b..."
date: 2026-11-05
permalink: /android-binder-driver-ipc/
---

> 应用进程之间几乎每一次跨进程调用，最终都会落到同一个内核驱动：`drivers/android/binder.c`。它把「谁在调用谁」「传了哪些句柄」「缓冲区归谁所有」这些语义搬进内核，用一套事务、引用计数与死亡通知机制串起来。也正因为如此，Binder 的内存生命周期、缓冲区核算与锁顺序一旦出错，影响就不是某个应用崩溃，而是跨进程的越界、释放后使用乃至内核提权。这份手册沿「对象与命令 → 归因与 SELinux → 缓冲区与引用计数 → 公开缺陷 → 缓解演进 → 怎么防」拆解驱动层攻击面。只讨论公开且已修复的问题，不含利用代码。

## 0. 使用方式与边界

面向三类读者：

- **应用与框架开发者**：理解一次 Binder 调用在内核里发生了什么，为什么 `oneway` 也会失败、为什么会话死亡需要 `DeathRecipient`。
- **系统与内核工程师**：区分 `/dev/binder`、`binderfs`、`hwbinder`、`vndbinder` 四种设备，知道线程池、异步空间与引用计数的真实名称。
- **安全评估者**：在看到 Binder 相关公告时，判断缺陷落在事务解析、缓冲区核算、引用计数、锁与生命周期，还是 SELinux 策略。

三条自我约束：

1. 只引用公开且已修复的问题；CVE 描述以 NVD、Android Security Bulletin 与 Project Zero 的分析为准。
2. 不写利用步骤、不写 PoC、不写可复现的事务序列。
3. 查不到一手来源的说法放进「未核实与失效来源」。

与已发布手册的分工：

- 《Android 框架 IPC 与组件导出攻击面手册》讲 `Binder` 客户端 API、`ServiceManager` 查找与组件导出；本文讲**驱动**如何实现这些语义。
- 《Android SELinux 策略与 seccomp 沙箱手册》讲策略语言与域隔离；本文只讲 Binder 类权限与归因。
- 《Android 内存安全与分配器手册》讲通用内存缺陷模式；本文只讲 Binder 特有的缓冲区与引用计数。

## 1. 设备与对象模型

### 1.1 四种设备

| 设备 | 路径 | 使用者 |
|------|------|--------|
| binder | `/dev/binder` | 普通应用与框架 |
| hwbinder | `/dev/hwbinder` | HAL（HIDL） |
| vndbinder | `/dev/vndbinder` | vendor 进程 |
| binderfs | `/dev/binderfs/` | 多实例场景，按需创建 |

binderfs 在 Linux 5.0 进入上游，支持多实例；每个实例只有一个 `binder-control` 节点，通过 `BINDER_CTL_ADD` 动态创建实际设备。AOSP 的 `init.rc` 以 `mount binder binder /dev/binderfs stats=global` 挂载，并把 `binder`、`hwbinder`、`vndbinder` 以 0666 权限链接到 `/dev`。

注意：**0666 不等于「谁都能随便用」**。文件权限只是第一道门，真正决定谁能调用谁的是 SELinux 的 `binder` 类权限。

### 1.2 核心结构

| 结构 | 含义 |
|------|------|
| `binder_proc` | 一个打开 Binder 设备的进程 |
| `binder_thread` | 该进程中的 Binder 线程 |
| `binder_node` | 内核中的一个 Binder 对象 |
| `binder_ref` | 某进程对某个 node 的引用 |
| `binder_buffer` | 事务缓冲区，定义在 `binder_alloc.h` |
| `binder_transaction` | 一次事务的运行时表示 |
| `binder_ref_death` | 死亡通知的接收记录 |

强引用与弱引用分别记录在 `binder_node.local_strong_refs` 与 `local_weak_refs`，引用本身记录在 `binder_ref`。死亡通知通过 `BC_REQUEST_DEATH_NOTIFICATION` 注册、`BC_DEAD_BINDER_DONE` 确认，事件以 `BR_DEAD_BINDER` 投递。

### 1.3 ServiceManager 与 context manager

`servicemanager` 启动时调用 `ProcessState::becomeContextManager()`，最终发出 `BINDER_SET_CONTEXT_MGR`，成为 handle 0 的 context manager。驱动对 handle 0 的解析直接取该 `binder_device` 上的 `binder_context_mgr_node`，因此 **context manager 是按设备隔离的**：`/dev/binder`、`/dev/hwbinder`、`/dev/vndbinder` 各有自己的 handle 0。

ServiceManager 本身并不转发调用，它只是一个「服务名 → handle」的目录。真正的调用发生在客户端拿到 handle 之后。

## 2. 命令与事务语义

### 2.1 BC_ 与 BR_

客户端写入驱动的命令以 `BC_` 开头，驱动返回的事件以 `BR_` 开头：

| 命令 | 作用 |
|------|------|
| `BC_TRANSACTION` / `BC_REPLY` | 发起事务 / 回复事务 |
| `BC_FREE_BUFFER` | 释放事务缓冲区 |
| `BC_ENTER_LOOPER` | 进入 Binder 线程池 |
| `BC_INCREFS` / `BC_ACQUIRE` / `BC_RELEASE` / `BC_DECREFS` | 弱引用与强引用的增减 |
| `BC_REQUEST_DEATH_NOTIFICATION` / `BC_DEAD_BINDER_DONE` | 注册死亡通知 / 确认已处理 |
| `BR_TRANSACTION` / `BR_REPLY` | 投递事务 / 回复 |
| `BR_DEAD_REPLY` / `BR_FAILED_REPLY` | 目标已死亡 / 事务失败 |
| `BR_SPAWN_LOOPER` | 请求进程补充 Binder 线程 |

### 2.2 oneway 不是「发出去就不管」

带 `TF_ONE_WAY` 标志的事务不等待成功回复，但它仍然：

- 占用异步缓冲区配额（`free_async_space` 约为缓冲区大小的一半）；
- 在目标死亡或失败时返回 `BR_DEAD_REPLY` / `BR_FAILED_REPLY`；
- 过量发送会触发驱动的异步洪泛检测（`BR_ONEWAY_SPAM_SUSPECT`）。

把 oneway 当作「无成本广播」，是应用侧最常见的误判，也是驱动侧需要限流的原因。

### 2.3 线程池

`BINDER_SET_MAX_THREADS` 设置进程的最大 Binder 线程数，libbinder 的默认值是 15。驱动在需要时通过 `BR_SPAWN_LOOPER` 请求补充线程。并不存在名为 `binder_set_max_threads` 的内核接口——这是常见记错。

## 3. 归因与 SELinux 边界

### 3.1 uid/pid 从哪来

事务进入驱动时，`binder_transaction` 把 `current_euid()` 记录为 `sender_euid`，并与 `sender_pid` 一起随 `binder_transaction_data` 下发。服务端调用 `Binder.getCallingUid()` 读到的就是这份 `sender_euid`。

这意味着：**归因的可信度取决于驱动写入的字段，而不是调用方自报**。但也意味着，任何让这份字段与实际调用者不一致的缺陷，都会直接破坏服务端的权限判断。

### 3.2 SELinux 的 binder 类

SELinux 的 `binder` 类权限包括：

- `call`：允许发起事务；
- `transfer`：允许在事务中传递 Binder 或文件描述符；
- `set_context_mgr`：允许注册为 context manager；
- `impersonate`：允许以其他身份操作。

`binder_call`、`binder_transfer_binder` 不是权限名，而是 LSM 钩子（`security_binder_transaction`、`security_binder_transfer_binder`、`security_binder_transfer_file`）的语义，最终映射到上述权限。Android 策略里常用的 `binder_use()` 宏就是一组授权的集合。

### 3.3 context manager 的门槛

`BINDER_SET_CONTEXT_MGR` 在驱动侧的检查主要是：是否已有注册者（有则 `EBUSY`）、SELinux 是否允许 `set_context_mgr`、euid 是否与首注册者一致。它并**不要求** `CAP_SYS_ADMIN`。在 Android 上，真正把范围收窄的是 SELinux：只有 servicemanager 及其 hw/vnd 变体域被允许注册。

## 4. 缓冲区与引用计数：缺陷高发区

Binder 的内存安全缺陷，绝大多数集中在三件事上：

1. **事务缓冲区的大小与偏移核算**：`binder_alloc` 需要根据对象数、偏移数组、数据长度计算所需空间；算错就会越界读写（CVE-2020-0041）。
2. **缓冲区的释放路径**：事务失败、进程退出、线程退出时都要正确归还缓冲区与其中的引用；漏放或重复释放会导致 UAF 或引用计数错乱（CVE-2023-20938、CVE-2023-21255）。
3. **对象生命周期与等待队列**：`binder_thread` 内嵌的等待队列若被外部（如 epoll）持有，线程退出后仍被访问，就会形成 UAF（CVE-2019-2215）。

这类缺陷的共同特征是：**触发条件往往是正常 API 组合**，不需要特殊的调试接口；因此一旦存在，影响面覆盖所有使用 Binder 的进程。

## 5. 公开公告台账

| CVE | 落点 / 类型 | 影响 | 修复 | 说明 |
|-----|-------------|------|------|------|
| [CVE-2019-2215](https://nvd.nist.gov/vuln/detail/CVE-2019-2215) | `binder.c` 等待队列 UAF（EoP） | 内核 3.18/4.4/4.9 多款设备 | 2019-10 前 SPL | 有在野利用；Project Zero 有根因分析 |
| [CVE-2020-0041](https://nvd.nist.gov/vuln/detail/CVE-2020-0041) | `binder_transaction` 的 `num_valid` 计算错误（越界写/EoP） | Android 10，部分 4.9/4.14 设备 | 2020-03 | 事务缓冲区核算缺陷 |
| [CVE-2020-0423](https://nvd.nist.gov/vuln/detail/CVE-2020-0423) | `binder_release_work` 锁缺陷（UAF/EoP） | 影响版本未逐一核实 | 2020-10 | 释放路径与锁顺序 |
| [CVE-2022-20421](https://nvd.nist.gov/vuln/detail/CVE-2022-20421) | `binder_inc_ref_for_node` UAF（EoP） | 影响版本未逐一核实 | 2022-10 | 引用计数 |
| [CVE-2023-20938](https://nvd.nist.gov/vuln/detail/CVE-2023-20938) | `binder_transaction_buffer_release` 引用计数错误（UAF/EoP，可泄露 node 指针） | GKI 5.4/5.10 设备 | 2023-02 | 有公开根因分析 |
| [CVE-2023-21255](https://nvd.nist.gov/vuln/detail/CVE-2023-21255) | 同根因，前次修复不完整 | 同上 | 2023-07 | 修复不完整导致二次公告 |

读表要点：

1. **六年六例，全部与内存生命周期有关**，没有一例是「协议设计」问题。
2. **修复不完整会二次公告**（CVE-2023-20938 → CVE-2023-21255），说明这类缺陷的审计需要覆盖同一根因的其他路径。
3. **影响范围常写成「部分设备」**，因为是否触发取决于内核版本、配置与厂商补丁。

## 6. 缓解演进

Android 内核侧这些年叠了多层缓解：

| 机制 | 作用 | 状态 |
|------|------|------|
| KFENCE | 轻量级采样内存错误检测 | GKI 配置启用（500ms、63 objects） |
| `INIT_ON_ALLOC_DEFAULT_ON` | 分配时清零，削弱未初始化内存利用 | GKI 默认启用 |
| CFI / KCFI | 间接调用目标校验 | Android 9 起，GKI 默认；6.1+ 为 KCFI |
| Shadow Call Stack | 返回地址保护 | GKI 启用 |
| `HARDENED_USERCOPY` | 用户态拷贝边界检查 | GKI 启用 |
| `SLAB_FREELIST_RANDOM` / `HARDENED` | 堆布局随机化与加固 | GKI 启用 |
| 关闭 `SLAB_MERGE_DEFAULT` | 减少跨类型堆叠 | GKI 配置 |

需要区分「defconfig 里存在」与「量产全部启用」：KASAN 通常出现在调试构建（如 `build.config_kasan.gki`），量产未必开启；slab 虚拟隔离未见 GKI 默认启用。把这些混为一谈，会高估或低估实际防护。

此外，libbinder 默认启用 oneway 洪泛检测，这属于拒绝服务缓解，不是内存安全加固。

## 7. 成因模式

### 7.1 长度与偏移的整数核算

事务里同时有数据区、偏移数组与对象数组，长度字段来自用户态。任何一处加法溢出、单位混淆（字节 / 指针数）或对齐遗漏，都会让后续拷贝越界。修复通常不是「加一个 if」，而是重写核算函数并补充边界测试。

### 7.2 释放路径与引用计数

缓冲区里可能嵌着 Binder 引用与文件描述符。释放时要逐个归还；如果中途失败、提前返回或重复处理，就会漏放或双放。CVE-2023-20938 的根因就是释放路径对引用的处理不一致。

### 7.3 锁顺序与等待队列

Binder 的锁层次复杂（proc 锁、node 锁、inner 锁）。等待队列被外部事件机制持有时，线程退出与唤醒之间的竞态会留下悬空指针（CVE-2019-2215）。

### 7.4 修复不完整

同一根因往往有多条触发路径。只修报告里那一条，留下其他路径，就会出现二次公告（CVE-2023-21255）。评估时看到「同一组件连续两个月出公告」，应怀疑修复覆盖不全。

### 7.5 异步配额与拒绝服务

oneway 事务占用异步空间，大量发送会耗尽配额，影响同进程的其他调用。这不是内存破坏，但会导致系统服务不可用。

## 8. 版本与加固时间线

| 时间 / 版本 | 变化 |
|-------------|------|
| Linux 5.0 | binderfs 进入上游，支持多实例 |
| Android 9 | 内核 CFI 引入 |
| Android 11 | `init_on_alloc` 生产启用 |
| Android 12 | GKI 5.10 普及，KFENCE 等配置进入 GKI |
| Android 13 | GKI 5.15/6.1 逐步引入 KCFI |
| 2019–2023 | Binder 驱动连续出现 UAF/越界类公告 |

趋势是**检测与加固前移**：从「发现一个修一个」转向「用 KFENCE、CFI、清零分配降低同类缺陷的可利用性」。

## 9. 观测点与自查

### 9.1 内核与 ROM 侧

- 内核补丁级别是否覆盖 2023-07 及之后的 Binder 公告？
- GKI defconfig 中 KFENCE、`INIT_ON_ALLOC_DEFAULT_ON`、CFI/KCFI、`HARDENED_USERCOPY` 是否启用？
- 是否存在不必要的 binder 设备或 binderfs 实例？是否设置了实例上限？
- 是否保留了厂商私有的 Binder 扩展？这些扩展往往不在 AOSP 审计范围内。

### 9.2 SELinux 侧

- 应用域是否只授予必要的 `binder { call transfer }`？
- `set_context_mgr` 是否只授给 servicemanager 及其变体？
- 是否有 `neverallow` 阻止普通应用域注册 context manager？
- 新增服务是否复用了过宽的域？

### 9.3 应用侧

- 服务端是否校验 `getCallingUid()` / `getCallingPid()`，而不是信任参数里的身份字段？
- 是否限制单次事务大小与 oneway 频率？
- 是否注册 `DeathRecipient` 并处理 `onServiceDied`？
- 是否在 Binder 线程里做重活，导致线程池耗尽？

## 10. 防御要点

1. **跟进 SPL**：Binder 缺陷的修复窗口以月为单位，延迟打补丁等于长期暴露。
2. **最小化 Binder 暴露**：不导出不必要的服务，跨进程接口用签名权限保护。
3. **服务端复核身份**：以 `getCallingUid()` 为准，参数里的 uid 一律视为不可信。
4. **限制事务规模**：对入参长度、对象数、fd 数设上限，拒绝异常组合。
5. **oneway 限流**：对高频 oneway 调用做合并与节流，避免异步配额被单点耗尽。
6. **启用内核缓解**：KFENCE、CFI、清零分配、硬化 usercopy 是降低同类缺陷可利用性的基础。
7. **审计厂商扩展**：私有 Binder 命令与结构体是公告覆盖的盲区。

## 11. 常见误传纠正

### 误传一：Binder 只在内核里，普通应用碰不到

错。应用常态打开 `/dev/binder` 并收发事务；SELinux 的 `binder_use()` 正是为应用域准备的。

### 误传二：binderfs 必须 root 才能访问

错。节点权限是 0666；挂载需要 `CAP_SYS_ADMIN`，但 Android 由 init 完成挂载。访问控制主要落在 SELinux。

### 误传三：oneway 没有返回值，失败也无感

错。失败会收到 `BR_DEAD_REPLY` / `BR_FAILED_REPLY`，并且占用异步配额。

### 误传四：注册 context manager 需要 root 或 `CAP_SYS_ADMIN`

错。驱动侧不检查该能力；Android 通过 SELinux 把 `set_context_mgr` 限定给 servicemanager 域。

### 误传五：修完一个 Binder CVE 就安全了

错。同一根因常有其他路径，CVE-2023-20938 之后两个月就出了 CVE-2023-21255。

## 12. 未核实与失效来源

- CVE-2020-0423、CVE-2022-20421 的精确影响版本未逐一核实，表中只写「未逐一核实」。
- KASAN 在量产 GKI 中的启用比例未核实；本文只确认调试构建存在。
- 「slab 虚拟隔离」是否在部分 OEM 内核启用未核实，不作为通用结论。
- Android 12 之后是否有 Binder 专项加固公告，未检索到可确认条目。
- 各厂商私有 Binder 扩展的审计情况未核实。

## 13. 参考资料

- Binder 驱动（AOSP 内核）：
  https://android.googlesource.com/kernel/common/+/refs/heads/android-mainline/drivers/android/binder.c
- binder_alloc（AOSP 内核）：
  https://android.googlesource.com/kernel/common/+/refs/heads/android-mainline/drivers/android/binder_alloc.h
- GKI defconfig：
  https://android.googlesource.com/kernel/common/+/refs/heads/android-mainline/arch/arm64/configs/gki_defconfig
- init.rc（AOSP）：
  https://android.googlesource.com/platform/system/core/+/refs/heads/main/rootdir/init.rc
- binderfs 文档：
  https://docs.kernel.org/admin-guide/binderfs.html
- 内核 CFI / KCFI：
  https://source.android.com/docs/security/test/kcfi
- Android 11 系统加固：
  https://android-developers.googleblog.com/2020/06/system-hardening-in-android-11.html
- CVE-2019-2215 根因分析：
  https://googleprojectzero.github.io/0days-in-the-wild/0day-RCAs/2019/CVE-2019-2215.html
- CVE-2023-20938 分析：
  https://androidoffsec.withgoogle.com/posts/attacking-android-binder-analysis-and-exploitation-of-cve-2023-20938/
- 公告：
  https://source.android.com/docs/security/bulletin/2020-03-01
  https://source.android.com/docs/security/bulletin/2020-10-01
  https://source.android.com/docs/security/bulletin/2022-10-01
  https://source.android.com/docs/security/bulletin/2023-02-01
  https://source.android.com/docs/security/bulletin/2023-07-01
- NVD 条目（本文台账）：
  https://nvd.nist.gov/vuln/detail/CVE-2019-2215
  https://nvd.nist.gov/vuln/detail/CVE-2020-0041
  https://nvd.nist.gov/vuln/detail/CVE-2020-0423
  https://nvd.nist.gov/vuln/detail/CVE-2022-20421
  https://nvd.nist.gov/vuln/detail/CVE-2023-20938
  https://nvd.nist.gov/vuln/detail/CVE-2023-21255

---

## 联系方式

咨询 Android 系统详情请联系 https://t.me/one00190

---

*本手册仅供安全学习与防御研究使用；所有机制表述、公告列值与 CVE 描述均取自公开一手来源并附链接，未能核实的部分已单列于「未核实与失效来源」而非写入正文。*
