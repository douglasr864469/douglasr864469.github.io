---
layout: page
title: "Android 性能计数器 perf_event 攻击面手册"
description: "讲 `perf_event_open` 的权限模型与 `perf_event_paranoid` 各档语义，说明 AOSP 在 `init.rc` 中如何设定该值、`security.perf_harden` 属性起什么作用，以及现代平台上 `traced_perf` 一类受限采集服务的分工；指出「应用..."
date: 2026-11-17
permalink: /android-kernel-perf-events/
---

> 面向 Android 内核维护者、性能工具开发者与系统安全工程师的防御性说明
> 版本：2026-11
> 适用范围：讨论 `perf_event_open` 的观测能力、Android 权限门禁、公开缺陷与防护。**不包含**侧信道攻击代码、数据窃取或提权步骤。

---

## 0. 性能计数器为什么属于安全边界

`perf_event_open` 是 Linux 的性能事件接口。它能统计 CPU 周期、指令数、缓存命中、分支预测，也能挂接 tracepoint、kprobe、uprobe 与 breakpoint。对性能分析来说，这是一双「看进内核与硬件内部」的眼睛。

安全风险来自同一能力：

- 它可以观测**其他进程**与**内核**的行为；
- 某些事件会泄漏内核地址、调用栈或内存访问模式；
- 高精度计数器可以被用于缓存侧信道与隐蔽信道；
- 历史实现中存在整数溢出、竞态与越界写，直接影响内核完整性。

Android 的限制思路不是「取消性能分析」，而是把权限收窄到受信任的性能工具与系统服务，让普通应用默认拿不到这双眼睛。

---

## 1. 机制与权限

### 1.1 能观测什么

- **硬件事件**：CPU cycles、instructions、cache references/misses、branch misses 等。
- **软件事件**：context switches、page faults、cpu clock 等。
- **tracepoint**：内核静态埋点。
- **kprobe/uprobe**：动态插桩。
- **breakpoint**：硬件断点。
- 可按 PID、CPU、cgroup 过滤；可继承到子进程；可输出采样记录。

### 1.2 perf_event_paranoid

`/proc/sys/kernel/perf_event_paranoid` 控制非特权访问。内核注释给出的语义是：

| 值 | 语义 |
|---|---|
| `-1` | 不限制 |
| `0` | 禁止非特权 raw tracepoint |
| `1` | 禁止 CPU 事件 |
| `2` | 禁止内核 profiling（默认） |

部分旧内核使用 `3` 表示「仅 root」。取值越高越严格。

### 1.3 能力与 seccomp

新内核引入 `CAP_PERFMON`，把性能监控从 `CAP_SYS_ADMIN` 中拆出。即便如此，Android 仍通过 SELinux 与 seccomp 做域级限制。

---

## 2. Android 上的实际门禁

### 2.1 init.rc 的 paranoid 设置

AOSP 的 `init.rc` 会在内核支持 perf_event SELinux hooks 时把 `perf_event_paranoid` 设为 `-1`，把访问控制交给 SELinux；对较旧内核则可能使用 `3`（仅 root），并允许 shell 通过 `security.perf_harden` 调整到 `1`。

这说明：**不能只看 paranoid 数值判断安全**。在 Android 上，`-1` 并不等于「所有人可用」，而是「由 SELinux 精细决定谁可用」。

### 2.2 SELinux 授权

AOSP 策略中，`traced_perf`（Perfetto 的性能采集组件）、simpleperf、system_server、lmkd、gpuservice、profcollectd 等域有相应 allow 规则。普通 `appdomain` 没有宽泛授权，默认拒绝。

### 2.3 应用侧

普通第三方应用既没有性能监控的系统权限，也不在 SELinux 的 perf 授权域中。应用想做性能分析，通常需要：

- 使用 `profileable` 构建变体与 Android Studio/Perfetto 的受控通路；
- 在调试设备上由 shell 或系统工具采集；
- 不能假设 `perf_event_open` 在量产应用进程中可用。

---

## 3. 公开案例台账（逐条核实归属）

| CVE | 机制 | 缺陷类型 | 影响与修复 | 备注 |
|---|---|---|---|---|
| CVE-2013-2094 | `perf_swevent_init` | 整数类型错误导致越界/提权 | 修复于 3.8.9 一线 | 经典 perf 子系统提权案 |
| CVE-2019-3901 | `perf_event_open` | 竞态导致信息泄漏 | setuid 程序场景 | 可从受保护进程泄漏数据 |
| CVE-2022-1729 | perf 事件上下文 | 竞态导致释放后使用 | 后续稳定版修复 | 可导致提权 |
| CVE-2023-6931 | `read_size` 计算 | 整数溢出导致堆越界写 | 后续稳定版修复 | 采样读取路径 |

以上均为 Linux 内核缺陷。Android 设备是否受影响，取决于内核版本、perf 配置、SELinux 授权以及攻击者能否调用 `perf_event_open`。

### 3.1 侧信道研究

USENIX Security 2016 的 ARMageddon 研究指出，Android 上 `perf_event_open` 可被无特权访问，并被用于缓存攻击与隐蔽信道。该研究推动 Android 收紧权限。引用这类研究时要注意时间点：它描述的是当时的权限模型，不应直接套用到现代 Android。

---

## 4. 防护与检测

### 4.1 内核与配置

- 保持 `perf_event_paranoid` 在较严格档位；若使用 `-1` + SELinux 模式，必须确认策略完整。
- 跟进上表 CVE 的稳定版修复，核对 GKI 与厂商内核。
- 评估是否需要 `CONFIG_PERF_EVENTS`；完全关闭会损失性能工具，但能移除攻击面。
- 限制 kprobe/uprobe 的动态创建权限。

### 4.2 Android 策略

- 只给性能工具域授予 perf 权限，不给 `untrusted_app` 或宽泛的 `system_app`。
- 在 seccomp 层评估拦截 `perf_event_open`，同时验证 Perfetto/simpleperf 的合法路径。
- 审计 `perf_event_open` 调用与 AVC 拒绝记录，关注异常高频采样与跨进程 PMU 访问。
- 对 `profileable` 应用与 debuggable 构建做单独评估，不要与量产构建混用策略。

### 4.3 检测思路

```text
1. 读取 /proc/sys/kernel/perf_event_paranoid
2. 列出拥有 perf_event 相关 SELinux allow 的域
3. 检查目标进程 seccomp 是否拦截 perf_event_open
4. 审计 perf_event_open 的系统调用与 AVC 记录
5. 检查是否存在非性能工具域的高频 PMU 采样
6. 对照内核版本与上表修复
```

---

## 5. 常见误传纠正

1. **「perf_event_paranoid=-1 就是完全不设防」**——在 Android 上通常意味着访问控制交给 SELinux；数值本身不说明最终权限。
2. **「普通应用可以随意调用 perf_event_open」**——现代 Android 的 SELinux 默认拒绝应用域，seccomp 也可能拦截。
3. **「perf 只能做性能分析，没有安全影响」**——它能观测内核与其他进程，也是侧信道与信息泄漏原语。
4. **「所有 perf CVE 都能在 Android 上触发」**——需要内核版本、配置与调用域同时满足。
5. **「关闭 perf 不影响系统」**——Perfetto、simpleperf、lmkd 等可能依赖它；关闭前必须验证。

---

## 6. 未核实项

- 不同设备、内核与 init 流程结束后的 `perf_event_paranoid` 实际值；没有统一的固定值。
- 普通应用是否在 SELinux 之外还被 seccomp 拦截 `perf_event_open`；SELinux 已默认拒绝，但 seccomp 层未逐一核实。
- 上表 CVE 与 Android 安全公告的逐条对应关系；部分 CVE 只在内核稳定分支修复，未必单独出现在 Android 公告中。
- 各厂商内核对 `security.perf_harden` 与 perf 策略的修改。

以上项目需在目标设备上实测或查阅厂商源码。

---

## 7. 参考资料

- perf_event_open 手册：<https://man7.org/linux/man-pages/man2/perf_event_open.2.html>
- 内核 perf 核心源码：<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/events/core.c>
- AOSP init.rc：<https://android.googlesource.com/platform/system/core/+/refs/heads/main/rootdir/init.rc>
- AOSP traced_perf 策略：<https://android.googlesource.com/platform/system/sepolicy/+/refs/heads/main/private/traced_perf.te>
- CVE-2013-2094：<https://nvd.nist.gov/vuln/detail/CVE-2013-2094>
- CVE-2019-3901：<https://nvd.nist.gov/vuln/detail/CVE-2019-3901>
- CVE-2022-1729：<https://access.redhat.com/security/cve/cve-2022-1729>
- CVE-2023-6931：<https://nvd.nist.gov/vuln/detail/CVE-2023-6931>
- ARMageddon 论文：<https://www.usenix.org/system/files/conference/usenixsecurity16/sec16_paper_lipp.pdf>

---

## 联系方式

咨询详情请联系 https://t.me/one00190

## 免责声明

本手册面向防御与合规用途，仅讨论 perf_event 的观测能力、Android 权限门禁、公开缺陷与防护。文中不包含侧信道攻击代码、数据窃取或提权步骤。内核配置与 SELinux 策略会随版本变化，请以设备实际构建与官方文档为准；标注为「未核实」的内容不得作为安全结论的依据。
