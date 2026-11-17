---
layout: page
title: "Android 系统属性服务（property_service）攻击面手册"
description: "把属性系统当成「配置总线 + 控制通道 + 共享内存」三合一对象：讲清读走只读映射、写只能经 `/dev/socket/property_service` 这一条路径，`PROP_MSG_SETPROP2` 与早期消息类型的差异；逐条拆 `ro.`／`persist.`／`ctl.`／`sys.`／`d..."
date: 2026-11-17
permalink: /android-system-property-service/
---

> 面向 Android 系统安全工程师、ROM 维护者与设备厂商的防御性说明
> 版本：2026-11
> 适用范围：讨论 init 提供的系统属性读写通道在权限边界、策略校验与持久化上的设计，以及审计时应关注的失配点。**不包含**任何利用代码、绕过 SELinux 的方法或提权步骤。

---

## 0. 为什么系统属性值得单独审计

Android 的系统属性看起来只是一堆 `key=value` 字符串，但它同时承担了三种角色：

1. **配置总线**：`ro.build.version.sdk`、`ro.product.model`、各种 `persist.*` 开关决定了框架与 HAL 的行为分支。
2. **控制通道**：`ctl.start`、`ctl.stop` 这类名字不是普通配置，而是让 init 启停服务的指令。
3. **跨进程状态**：属性区是多个进程共享映射的只读内存，写入必须经过唯一一个服务进程。

一个可写属性如果被错误地开放给低权限域，等价于让该域间接影响 init 的行为。审计时常见的误区是只看「这个属性有没有被应用读到」，而忽略了「谁能写它、写进去之后会被谁当作指令执行」。

---

## 1. 机制：属性是怎么读写的

### 1.1 读：共享内存映射

属性值保存在 init 维护的一块共享内存里，按 SELinux 上下文分文件存放于 `/dev/__properties__/` 下，例如 `properties_serial` 以及以属性上下文命名的文件。`libc` 在进程启动早期把这些文件**只读**映射进地址空间，之后 `__system_property_get` 之类的调用只是在本进程内存里查表，不产生系统调用。

这意味着：

- 读属性的开销极低，任何进程都能读自己有权映射的那部分区域；
- 属性区更新由 init 负责，客户端看到的是某个时刻的快照，不是实时同步的全局变量；
- 能否读到某个属性，取决于该上下文文件是否可被本域映射，而不是取决于属性名本身。

### 1.2 写：唯一的 socket 入口

所有写入走 `/dev/socket/property_service`，类型是 `SOCK_STREAM`。客户端通过 `__system_property_set` 建立连接并发送消息，消息带有一个类型字段：

- 早期的 `PROP_MSG_SETPROP`，服务端不回结果；
- 后来的 `PROP_MSG_SETPROP2`，服务端会回一个结果码，客户端据此判断成败。

服务端在 `system/core/init/property_service.cpp` 中完成三步校验：名字合法性检查、SELinux 权限检查（调用方域是否有对该属性类型的 `set` 权限）、以及针对特殊前缀的附加规则。任一环节失败就拒绝，且不会给出「为什么失败」的细粒度原因——这是有意的，避免把策略结构暴露给探测者。

### 1.3 前缀语义

属性名不是自由的字符串，前缀决定了它的生命周期与处理路径：

| 前缀 | 语义 | 审计要点 |
| --- | --- | --- |
| `ro.` | 只读，首次设置后不可再改 | 只在启动早期可写；后期写入失败属于预期行为 |
| `persist.` | 持久化，跨重启保留 | 落盘位置与文件权限；写入频率是否被滥用为隐蔽通道 |
| `ctl.` | 控制指令，由 init 特殊处理 | `ctl.start` / `ctl.stop` / `ctl.restart` 会真正拉起或停止服务 |
| `sys.` | 普通可写运行时状态 | 是否被低权限域写入了影响安全决策的值 |
| `vendor.`、`ro.vendor.` | 厂商命名空间 | 厂商策略与 AOSP 策略是否一致 |
| `debug.` | 调试开关 | 量产机上是否仍可写 |

`persist.` 属性在早期实现中逐文件存放于 `/data/property/`，新版改为聚合存储（`persistent_properties` 的 protobuf 形式），因此审计时要按实际版本确认落盘形态，不能照抄旧文档。

---

## 2. Android 上的暴露条件

### 2.1 SELinux：属性上下文决定一切

`system/sepolicy/private/property_contexts` 把属性名映射到安全上下文类型，匹配方式有精确匹配与前缀匹配两种。策略侧通过 `set_prop(domain, type)` 与 `get_prop(domain, type)` 宏授权。

因此判断「某个域能不能写某属性」的正确方法是三件事一起看：

1. 属性名在 `property_contexts` 中命中哪个类型（注意前缀匹配的先后顺序，最长匹配优先）；
2. 调用方域是否有对该类型的 `set` 权限；
3. 该属性是否属于 `ctl.` 等会被 init 特殊解释的名字。

只看第 1 步就会得出错误结论：属性被归到某个类型，不代表任何域都能写它；反过来，某域能写一个类型，也不代表它能写该类型下所有名字——精确条目会覆盖前缀条目。

### 2.2 `ctl.` 的附加校验

`ctl.start` 与 `ctl.stop` 是唯一能直接驱动 init 的属性。init 在处理它们时会额外做一次权限检查：调用方不仅要对该控制属性有权限，通常还需要对目标服务自身的类型具备相应权限，或本身就是 init 域。这条附加规则经常被忽略，导致审计时把「属性可写」误判成「可启停任意服务」。

实际的校验函数与判定细节在不同 Android 版本间有过调整，部署前应回到本设备对应的 `property_service.cpp` 核对，不要依赖本手册的概述。

### 2.3 厂商定制是最常见的失配来源

AOSP 的属性集合相对克制，风险多出现在厂商新增的属性上：把安全相关开关做成 `persist.` 属性、把 SELinux 模式或调试级别放在可写属性里、为了让某个闭源服务能读配置而放宽 `property_contexts` 的类型归属。这些改动单看每一处都「只是为了跑通功能」，合起来就形成了一个可被低权限域触碰的控制面。

### 2.4 与 `vendor_init` 的关系

`vendor_init` 是厂商脚本运行的独立域，能写一批 AOSP 域不能写的 `vendor.` 属性。审计时应确认 `vendor_init` 的规则集合有没有被扩大到覆盖敏感属性，以及厂商是否通过 `vendor_init` 把本应由 init 完成的动作下放。

---

## 3. 公开案例台账

本手册检索范围内，**没有找到直接针对 AOSP `property_service` 的独立 CVE**。这一点本身就是值得记录的结论：属性服务的核心校验逻辑多年来相对稳定，公开披露更多集中在「厂商把敏感状态放进属性」这类配置问题上。

| 记录 | 性质 | 说明 | 核实状态 |
| --- | --- | --- | --- |
| 社区常引用的 CVE-2021-25369 | 三星内核信息泄漏 | 与本主题不构成直接对应，常被误当作「属性服务漏洞」引用 | 未核实其与本主题的关联 |
| 厂商自定义属性放宽 SELinux 归属 | 配置失配 | 多个定制 ROM 出现过为跑通功能而扩大 `set_prop` 授权的情况 | 属模式归纳，无单一编号 |
| `ctl.` 滥用讨论 | 设计边界 | 公开讨论多为「若某域同时具备控制属性与服务权限会怎样」的假设分析 | 未核实是否存在真实在野案例 |

需要强调的是：**「查不到 CVE」不等于「没有风险」**。属性服务的风险主要来自策略配置，而策略配置不属于 CVE 的覆盖范畴，它不会进入安全公告，也不会被补丁跟踪系统记录。这也是它容易被长期忽视的原因。

---

## 4. 防护与检测

### 4.1 策略层

- 对 `property_contexts` 做定期比对：把厂商新增条目与 AOSP 基线做差集，逐条确认必要性。
- 检查所有 `set_prop` 授权，重点关注授予对象是 `untrusted_app`、`shell`、`vendor_init` 的条目。
- 对 `ctl.` 相关类型单独列出清单，确认没有任何非 init 域同时具备控制属性权限与目标服务权限。

### 4.2 运行期检测

- 监控 `persist.` 属性的写入频率与来源域，异常高频写入往往是隐蔽通道的信号。
- 记录 `ctl.start` / `ctl.stop` 的调用者，与预期调用方清单比对。
- 审计 init 的拒绝日志：属性写入被拒会留下记录，大量拒绝可能意味着有进程在探测。

### 4.3 构建期检查

- 在 CI 中加一条规则：`property_contexts` 的新增条目必须附带说明与审批，禁止无理由的前缀放宽。
- 对 `debug.` 前缀属性做量产机扫描，确认调试开关没有被遗留为可写。

---

## 5. 常见误传与纠正

1. **「属性区可写就能改任何属性」**——错。属性区是只读映射，写入只能走 socket 且必须过 SELinux 校验。
2. **「能写某类型就能写该类型下所有属性」**——错。精确匹配条目会覆盖前缀条目，同名类型下不同属性可能有不同归属。
3. **「`ctl.start` 可写就能启停任意服务」**——错。还有一层针对目标服务类型的附加校验。
4. **「`ro.` 属性绝对不可改」**——不准确。`ro.` 在启动早期可写一次，之后才锁定；早期写入路径是正常的构建/启动流程。
5. **「持久化属性一定按文件存放在 `/data/property/`」**——只对旧版本成立，新版是聚合存储。
6. **「属性服务出过 CVE，所以它是高危面」**——本手册未检索到直接 CVE，风险来自策略配置而非核心代码。

---

## 6. 未核实项

- 各 Android 版本 `CheckControlPropertyPerms` 的具体判定条件差异，未逐版本核对源码。
- 新版持久化属性的 protobuf 结构细节与损坏恢复行为，未核实。
- 是否存在真实在野的 `ctl.` 滥用案例，未找到公开证据。
- CVE-2021-25369 与本主题的关联性，未核实，正文已按「不构成直接对应」处理。

---

## 7. 参考资料

- AOSP `system/core/init/property_service.cpp`：<https://android.googlesource.com/platform/system/core/+/refs/heads/main/init/property_service.cpp>
- AOSP `system/sepolicy/private/property_contexts`：<https://android.googlesource.com/platform/system/sepolicy/+/refs/heads/main/private/property_contexts>
- AOSP `system/sepolicy` `te_macros`（`set_prop` / `get_prop` 定义）：<https://android.googlesource.com/platform/system/sepolicy/+/refs/heads/main/public/te_macros>
- bionic 属性实现：<https://android.googlesource.com/platform/bionic/+/refs/heads/main/libc/bionic/system_property_api.cpp>
- Android 属性系统概述：<https://source.android.com/docs/core/architecture/init/property-system>

---

## 联系方式

咨询详情请联系 https://t.me/one00190

## 免责声明

本手册面向防御与合规用途，仅讨论系统属性服务的权限模型、策略失配模式与检测方法。文中不包含利用代码、绕过 SELinux 或提权步骤。策略结构与实现细节随 Android 版本变化，请以设备实际源码与官方文档为准；标注为「未核实」的内容不得作为安全结论的依据。
