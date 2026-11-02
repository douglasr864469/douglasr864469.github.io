---
layout: page
title: "Android 竞态条件与 TOCTOU 手册"
description: "把「检查与使用不同步」拆成三种后果互不相同的缺陷：引用计数与解锁的顺序错误导致 UAF（CVE-2025-38349）、锁顺序与 GC 路径导致死锁（CVE-2024-35970）、路径组件在解析期间被替换（`LOOKUP_NO_SYMLINKS`／`LOOKUP_BENEATH` 依赖 `rename..."
date: 2026-11-02
permalink: /android-race-conditions-toctou/
---

> 「并发缺陷」在中文资料里常被写成一句万能解释：出问题了就说「有竞态」，修好了就说「加了锁」。这两句话都不构成分析。真实情况是：并发缺陷在 Android 上分成**三种后果完全不同的形态**——引用计数与等待队列被提前释放导致的 UAF、锁顺序错误导致的死锁、以及路径/属性在「检查」与「使用」之间被换掉导致的越权；它们的判据、可观测信号、缓解手段互不替代。本手册沿一手文档把这条线钉死：`refcount_t` 与 `atomic_t` 的差别官方只写成**内存序**（所以不能拿它当「防溢出」的护身符）；RCU 的安全性来自一条**时间保证**（只在读侧临界区内成立）；lockdep 检查的是**锁的使用状态与锁顺序**，且动态、有盲区；内核承认「路径组件在解析期间变化」这类竞态，并用 seqlock 检测、用 `openat2()` 的 resolve 标志限制；公告的 Type 列**不会写** race condition，机制信息要去 CVE 记录里找。定位是防御与教育：只引用公开且已修复的信息，不含任何可利用代码，也不给出任何触发步骤。

---

## 0. 使用方式与边界

**本手册的证据基础。** 官方口径来自 `source.android.com`（一律 `?hl=en` 抓英文原文）、`docs.kernel.org` 与 `cveawg.mitre.org` 的 CVE 记录；英文引文逐字给出，中文结论紧随其后，并在需要处写明**引文支撑到哪一步为止**。凡是官方页面没写的内容（`refcount_t` 的饱和语义、lockdep 的配置名、`RESOLVE_*` 常量名、公告里的 race 字样），一律进文末「未核实与失效来源」，正文不写。

**本手册刻意不做的事。**

- 不给任何触发时序、重试次数、线程调度参数或 PoC。竞态类缺陷的「利用写法」一旦具体化就是攻击工具，本手册只讨论判据、机制与观测点。
- 不把「加了锁」当作结论。lockdep 能查的是锁顺序与使用状态，查不到临界区内的数据竞争（§4、§7）；把两者混为一谈是这份手册最想纠正的误读。
- 不把公告的 Type 列当根因。2025-12 内核表里 Type 只有 `EoP` 与 `ID`，竞态的机制信息在 CVE 记录的 description 与 CNA 评估文本里（§6）。
- 不讨论 SELinux 策略写法与 seccomp 白名单构造，那部分归 [SELinux 策略与 seccomp 沙箱手册](/android-selinux-seccomp-sandbox/)；不讨论 binder 接口面本身，归 [框架 IPC 与组件导出攻击面手册](/android-framework-ipc-export/)；不讨论分配器硬化与 MTE 的 tag 语义，归内存安全方向的既有材料，本手册只在 §7 引用 Scudo 的诊断文案。

**与相邻手册的分工**：见 §11。

---

## 1. 三个词：竞态、数据竞争、TOCTOU

这三个词经常被当成同义词用，但它们的判据不同，对应的修法也不同。本手册采用下面这组**操作性定义**——注意：这不是某份官方文档的「定义」，而是把官方文档里实际描述的三种现象分开命名，便于后文对号入座。

| 词 | 判据（操作性） | 典型后果 | 对应官方材料 |
| --- | --- | --- | --- |
| 竞态（race condition） | 结果依赖两个上下文之间未被约束的执行交错 | 可以是 UAF、死锁、越权，也可以只是行为不确定 | CVE 记录里的 race 语义、`openat2` 的动机句 |
| 数据竞争（data race） | 两个上下文访问同一内存、至少一个写、且没有同步 | 内存破坏、读到撕裂值 | lockdep 的**盲区**说明（它查的是锁，不是数据） |
| TOCTOU | 「检查」与「使用」之间存在窗口，窗口内对象状态可变 | 越权、逃逸目录、用错对象 | `d_lookup` 的 detect-not-prevent 与 `openat2` 的 resolve 标志 |

**为什么必须先分开。** 因为「修法」完全落在不同层：数据竞争靠同步原语与内存序（§2）；TOCTOU 靠把「检查」与「使用」合并成一次原子语义的操作，或者用序列号在窗口内变更时重试（§5）；生命周期类的竞态靠引用计数、RCU 宽限期或对象状态机（§3、§6）。用「加个锁」去修 TOCTOU，通常只是把窗口缩小，并没有消除；用「重试」去修生命周期问题，则会变成偶发崩溃。

**一个容易被忽略的事实：竞态类缺陷的「失败」往往是无害的。** CVE-2025-38349 的 CNA 评估文本写得很直白：

> "the attacker creates and controls both sides of the race, closing the epoll fd in one thread while closing a monitored file in another (pinnable to separate CPUs), and a lost attempt is completely benign so the race can be retried indefinitely until won."

> — <https://cveawg.mitre.org/api/cve/CVE-2025-38349>（`cna.metrics` 的评估文本，不是 `descriptions` 字段）

译：攻击者同时驱动竞争的两侧、一次没命中完全无害，因此可以无限重试直到命中。这条文本的价值不在于某个 CVE，而在于它揭示了**竞态类缺陷的可利用性评估与「触发条件苛刻」不是一回事**：单次失败无副作用的缺陷，重试成本可以低到忽略不计。边界：这是该 CVE 的评分解释文本，只针对 epoll/close 这一调用面，不能外推为「所有内核竞态都能在任意沙箱里反复重试」。

---

## 2. 引用计数：官方只把 `refcount_t` 的差别写成内存序

Android 内核与大量驱动用引用计数管理对象生命周期。一个常见误解是「换成 `refcount_t` 就防住了引用计数溢出导致的 UAF」。官方文档并没有这么说。

> "The goal of refcount_t API is to provide a minimal API for implementing an object’s reference counters."
> "While a generic architecture-independent implementation from lib/refcount.c uses atomic operations underneath, there are a number of differences between some of the refcount_*() and atomic_*() functions with regards to the memory ordering guarantees."

> — <https://docs.kernel.org/core-api/refcount-vs-atomic.html>

译：`refcount_t` 的目标是给对象引用计数提供一个最小 API；通用实现底层仍走原子操作，而它与 `atomic_*()` 的差异**集中在内存序保证**上。

官方给出的两组替换关系，正好说明差异在哪：

> "atomic_dec() --> refcount_dec()"
> "Memory ordering guarantee changes: fully unordered --> RELEASE ordering"
> "atomic_inc_not_zero() --> refcount_inc_not_zero()"
> "fully ordered --> control dependency on success for stores"

译：把 `atomic_dec()` 换成 `refcount_dec()`，不是改个名字，而是把原本**完全无序**的递减改成带 **RELEASE** 语义；`refcount_inc_not_zero()` 则把全序放宽为「成功拿到引用时的控制依赖」，文档的假设是顺序性由「拿到了对象指针」这一事实本身提供。

**这三句话能推出什么、不能推出什么。**

- 能推出：引用计数的正确性不只是「数字对」。递减与释放之间的可见性顺序错了，就可能出现「对象已被释放、另一个上下文仍认为自己持有引用」；递增侧的顺序错了，则可能拿到一个尚未完全发布的对象。
- 不能推出：`refcount_t` 在数值接近上限时会饱和、在减到零后再减会拒绝复用。本页通篇只讲内存序，对 `saturat`、`overflow` 零命中（见未核实 A）。**因此本手册不写「`refcount_t` 防溢出」这句话**；如果要在别处引用饱和语义，必须另找官方出处。
- 也不能推出：只靠引用计数就能防住所有生命周期竞态。§6 的第一个案例恰恰是「递减与解锁的顺序」而不是计数本身出错。

**与公告读法的关系。** CVE-2025-38349 的 CNA 描述是一句教科书式的顺序错误：

> "eventpoll: don't decrement ep refcount while still holding the ep mutex"
> "Jann Horn points out that epoll is decrementing the ep refcount and then doing a mutex_unlock(&ep->mtx); afterwards. That's very wrong, because it can lead to a use-after-free."

> — <https://cveawg.mitre.org/api/cve/CVE-2025-38349>

译：epoll 在**仍持有** `ep->mtx` 时先递减引用计数，随后才解锁；一旦这次递减释放了最后一个引用，对象已经被释放，而代码还要去解这把锁——UAF 由此产生。注意这里的因果：不是「计数算错了」，而是「释放动作被提前到了临界区内部」。这类缺陷的修法是把递减挪到解锁之后，而不是换一个计数器类型。

---

## 3. RCU：安全性是一条时间保证，不是「无锁就安全」

RCU 是内核里最常见的「读多写少」机制，也是最容易被误读成「读侧完全不加锁所以肯定安全」的地方。

> "Concurrent RCU readers can then continue accessing the old versions, and can dispense with the atomic operations, memory barriers, and communications cache misses that are so expensive on present-day SMP computer systems, even in absence of lock contention."
> "So how the heck can a reclaimer tell when a reader is done, given that readers are not doing any sort of synchronization operations???"

> — <https://docs.kernel.org/RCU/whatisRCU.html>

译：并发 RCU 读者可以继续访问旧版本，并省掉原子操作、内存屏障与缓存未命中；于是官方自己提出那个关键问题——读者不做任何同步操作，回收者凭什么知道读者结束了？

答案是一条时间保证：

> "This temporal primitive is used by a reader to inform the reclaimer that the reader is entering an RCU read-side critical section."
> "Any RCU-protected data structure accessed during an RCU read-side critical section is guaranteed to remain unreclaimed for the full duration of that critical section."

译：读者用它告知回收者「我进入了 RCU 读侧临界区」；**在临界区内访问的、受 RCU 保护的数据结构，在整个临界区期间保证不会被回收**。反过来说，一旦释放被提前到临界区之外、或者指针逃出了临界区，这条保证就失效——这才是「看起来很安全的无锁读」变成 UAF 的机制层面原因。边界：该保证只覆盖「确实发生在临界区内」的访问；不能据此说「RCU 保护的对象永远不会 UAF」。

**宽限期的边界比直觉窄。** 官方用了两句几乎重复的话来强调：

> "It does this by blocking until all pre-existing RCU read-side critical sections on all CPUs have completed. Note that synchronize_rcu() will not necessarily wait for any subsequent RCU read-side critical sections to complete."
> "To reiterate, synchronize_rcu() waits only for ongoing RCU read-side critical sections to complete, not necessarily for any that begin after synchronize_rcu() is invoked."

译：`synchronize_rcu()` 只等**调用那一刻已经在执行**的读侧临界区全部结束，之后新开始的读者不在等待范围内。所以「`synchronize_rcu()` 之后就没有读者了」是一个错误推论；正确的推论是「之前已经在读的那批读者都走完了，旧对象此刻可以释放」。

**读者与回收者之间没有显式通信。**

> "For example, RCU readers and updaters need not communicate at all, but RCU provides implicit low-overhead communication between readers and reclaimers, namely, in step (b) above."

译：读者与更新者之间**根本不需要通信**，安全释放依赖的是 RCU 提供的隐式、低开销通信（宽限期），而不是任何一把读者可见的锁。这解释了为什么「在 RCU 场景里再加一把读锁」往往是徒劳：问题不在读侧缺锁，而在释放点是否落在保证范围内。

---

## 4. 锁：lockdep 查的是「锁顺序与使用状态」，不是数据竞争

内核的锁校验器（lockdep）经常被当成「并发正确性证明器」。官方对它的描述要克制得多。

> "The validator tracks the ‘usage state’ of lock-classes, and it tracks the dependencies between different lock-classes."
> "Lock usage indicates how a lock is used with regard to its IRQ contexts, while lock dependency can be understood as lock order, where L1 -> L2 suggests that a task is attempting to acquire L2 while holding L1."
> "The validator maintains a continuing effort to prove lock usages and dependencies are correct or the validator will shoot a splat if incorrect."

> — <https://docs.kernel.org/locking/lockdep-design.html>

译：校验器跟踪**锁类（lock-class）的使用状态**，以及**锁类之间的依赖**——而依赖就是「锁顺序」：`L1 -> L2` 表示某个任务在持有 L1 时去取 L2。它在运行时持续尝试证明这些用法与依赖是对的，一旦不对就打印 splat（告警栈）。

**锁顺序错误的后果是死锁，而且 detector 能找出任意复杂度的环。**

> "Furthermore, two locks can not be taken in inverse order: <L1> -> <L2> <L2> -> <L1> because this could lead to a deadlock - referred to as lock inversion deadlock - as attempts to acquire the two locks form a circle which could lead to the two contexts waiting for each other permanently."
> "The validator will find such dependency circle in arbitrary complexity, i.e., there can be any other locking sequence between the acquire-lock operations; the validator will still find whether these locks can be acquired in a circular fashion."

译：两把锁不能以相反顺序获取，否则形成**锁反转死锁**——两个上下文永久互等。校验器在依赖图上找环，环中间夹着多少其它加锁序列都能找出来。边界：这一条只覆盖**死锁（可用性）**，与「数据被并发改写（数据竞争）」是两回事。

**锁与中断上下文的交叉同样是死锁来源。**

> "The first rule comes from the fact that a hardirq-safe lock could be taken by a hardirq context, interrupting a hardirq-unsafe lock - and thus could result in a lock inversion deadlock. Likewise, a softirq-safe lock could be taken by an softirq context, interrupting a softirq-unsafe lock."
> "The validator detects and reports lock usage that violates these single-lock state rules."

译：一把可能在硬中断上下文被取的锁（hardirq-safe），如果同时存在「关中断情况下取它」的路径（hardirq-unsafe），就会因为「取锁过程中被中断、中断里再取同一把锁」而自锁；软中断同理。校验器会报告违反这些**单锁使用状态**规则的用法。

**lockdep 的两个限制，写评估时最容易被漏掉。**

第一，它是动态校验器：只看实际发生过的加锁序列，不证明所有路径都正确。

第二，它需要人工标注嵌套层级，否则对「同一锁类的多个实例按固定层级取用」这种合法模式会误判：

> "In these cases there is an inherent “natural” ordering between the two objects (defined by the properties of the hierarchy), and the kernel grabs the locks in this fixed order on each of the objects."
> "The validator does not automatically detect this natural ordering, as the locking rule behind the ordering is not static."

译：对象之间存在由层级结构定义的「天然顺序」，内核按固定顺序取锁；但校验器**不会自动识别**这种天然顺序，因为顺序背后的规则不是静态可推的。所以 lockdep 是「需要工程约定配合的检测器」，不是全自动证明器。

**本手册不写 lockdep 的配置名。** 该页全文不含任何 `CONFIG_` 字符串（见未核实 B），因此「哪个配置打开 lockdep」「哪个配置打开 debugobjects」这类说法在本手册里没有出处，正文不出现。

---

## 5. 路径解析：TOCTOU 的内核侧范式是「检测 + 重试」，不是「阻止」

路径解析是 TOCTOU 最经典的发生地：解析过程中某个路径组件被改名、被替换成符号链接、或者挂载点被挪走，最终「检查时看到的对象」与「使用时打开的对象」不是同一个。内核文档直接承认这类场景存在。

> "In order to allow userspace to protect itself against certain race conditions and attack scenarios involving changing path components, a series of flags are available which apply restrictions to all path components encountered during path lookup. These flags are exposed through openat2()’s resolve field."

> — <https://docs.kernel.org/filesystems/path-lookup.html>

译：为了让用户态能自保，内核提供一组**对所有路径组件生效**的限制标志，经 `openat2()` 的 `resolve` 字段暴露。注意原文的限定词是 "certain race conditions and attack scenarios"——它承诺的是「有一类竞态与攻击场景可以被限制」，不是「所有 TOCTOU 都能被消灭」。

**三条限制标志的语义（内核文档用的是内部名 `LOOKUP_*`）。**

> "LOOKUP_NO_SYMLINKS blocks all symlink traversals (including magic-links)."
> "LOOKUP_BENEATH blocks any path components which resolve outside the starting point of the resolution. This is done by blocking nd_jump_root() as well as blocking “..” if it would jump outside the starting point. rename_lock and mount_lock are used to detect attacks against the resolution of “..”. Magic-links are also blocked."
> "LOOKUP_IN_ROOT resolves all path components as though the starting point were the filesystem root."

译：不允许任何符号链接穿越（含 magic-link）；不允许任何组件解析到起点之外（同时封掉跳到根、以及会越界的 `..`，并用 `rename_lock`/`mount_lock` **检测**解析 `..` 期间发生的攻击）；把起点当作根来解释所有组件。

这里有一个关键设计取向：`LOOKUP_BENEATH` 的做法不是「保证不发生竞争」，而是**用锁检测竞争是否发生**。

**d_lookup 的写法把这个范式说得更清楚。**

> "If a filename search happened to be looking at a dentry that was moved in this way, it might end up continuing the search down the wrong chain, and so miss out on part of the correct chain."
> "The name-lookup process (d_lookup()) does not try to prevent this from happening, but only to detect when it happens."
> "rename_lock is a seqlock that is updated whenever any dentry is renamed. If d_lookup finds that a rename happened while it unsuccessfully scanned a chain in the hash table, it simply tries again."

译：名字查找**不试图阻止**并发 rename，它只**检测**「扫描期间发生过 rename」，检测到就重来。`rename_lock` 是一个 seqlock，任何 dentry 改名都会更新它。

**从这三句可以提炼出一条通用模式**：面对「检查时与使用时状态可能不同」的问题，内核侧的答案是「给状态加版本号（序列号），在窗口内发现版本变了就重试」。这条模式对应用层同样适用——例如对文件做「先检查属性、再打开」的两步操作，本身就是一个可以被替换的窗口。

**两个必须写清楚的边界。**

- 这些标志只约束**解析阶段**。文件已经打开、拿到 fd 之后再去用，属于「使用阶段」，不在解析限制的覆盖范围内。把 `openat2` 当成「所有文件类 TOCTOU 的解药」是过度解读。
- 内核文档里是 `LOOKUP_*`；用户态 `openat2(2)` 用的 `RESOLVE_NO_SYMLINKS` / `RESOLVE_BENEATH` / `RESOLVE_IN_ROOT` 这组常量名**没有**出现在 `docs.kernel.org` 的该页，本次也未取得 man 页一手来源（见未核实 C）。因此本手册在讲用户态时只说「`openat2()` 的 `resolve` 字段」，不写 `RESOLVE_*` 常量的逐条语义。

---

## 6. 公告里的并发缺陷：三个案例，三种后果

下面三条都来自已修复的公开记录。选择它们的原因是后果形态不同：一个是 UAF（引用计数顺序）、一个是死锁（GC 锁顺序）、一个是 UAF（等待队列生命周期）。三条的公告表都**不含** "race condition" 字样——这不是公告写错了，而是 Type 列的分类维度本来就不是「并发/非并发」。

### 6.1 CVE-2025-38349：递减引用计数与解锁的顺序 → UAF

公告表（2025-12-01，Kernel 章节，按列逐格）：

| CVE | References | Type | Severity | Subcomponent |
| --- | --- | --- | --- | --- |
| CVE-2025-38349 | A-432751421 / Upstream kernel [2] | EoP | High | EPoll |

CVE 记录（Linux CNA）的 description 给出根因：

> "eventpoll: don't decrement ep refcount while still holding the ep mutex"
> "Jann Horn points out that epoll is decrementing the ep refcount and then doing a mutex_unlock(&ep->mtx); afterwards. That's very wrong, because it can lead to a use-after-free."

读法：公告只告诉你「这是一个 EPoll 子组件的 EoP、High」；**机制在 CVE 记录里**——递减引用计数发生在解锁之前，一旦这次递减把最后一个引用放掉，对象已经可以被回收，而代码接下来还要解那把锁。修复方向是调整顺序，而不是换计数器类型。

**竞态属性的第二层证据**在 `cna.metrics` 的评估文本里（§1 已引第一条）：攻击者同时控制竞争两侧、单次失败无害、可无限重试；且不需要任何特权：

> "PR:L - Only the ability to issue ordinary syscalls as any unprivileged user is required; epoll needs no capability and is permitted even inside restrictive seccomp sandboxes and containers."

边界：这段只说明**该 CVE 的调用面**在受限 seccomp 沙箱内仍然可达；不能据此写「seccomp 挡不住竞态类提权」这种一般性结论。但它在评估层面有一个实用含义——**用「沙箱已限制了系统调用」来论证某类竞态不可利用，是站不住的**。

### 6.2 CVE-2024-35970：GC 路径上的锁顺序 → 死锁

公告表（2025-12-01，Kernel 章节）：

| CVE | References | Type | Severity | Subcomponent |
| --- | --- | --- | --- | --- |
| CVE-2024-35970 | A-432804305 / Upstream kernel [2]–[29] | EoP | High | Net |

CVE 记录（Linux CNA）：

> "af_unix: Clear stale u->oob_skb."
> "syzkaller started to report deadlock of unix_gc_lock after commit 4090fa373f0e (\"af_unix: Replace garbage collection algorithm.\"), but it just uncovers the bug that has been there since commit 314001f0bf92 (\"af_unix: Add OOB support\")."

读法有三层：

1. **并发缺陷的后果不只有 UAF**。这一条描述的是 `unix_gc_lock` 的死锁——属可用性与锁顺序问题；公告的 Type 列却写 `EoP`。这说明 Type 列是**影响分类**（能被利用到哪种效果），不是根因分类；把 Type 当根因读，会得出「这是提权漏洞所以一定是内存破坏」的错误判断。
2. **syzkaller 这类模糊测试是并发缺陷的主要发现渠道**：Syzbot 在 GC 算法被替换后开始报告死锁，但根因早在此前的 OOB 支持提交里就埋下了。也就是说，新提交既可能引入新缺陷，也可能**只是让旧缺陷变得可触发**。
3. **上游修复与 Android 公告之间存在时间差**：CVE 编号是 2024 年的，却出现在 2025-12 的 Android 公告里。对设备评估的含义是：CVE 编号年份不能作为「Android 已修」的依据。

### 6.3 CVE-2019-2215：等待队列的生命周期与 epoll 引用解绑

公告表（2019-10-01，Kernel 章节；注意该期表头是 **Component**，不是近年的 **Subcomponent**）：

| CVE | References | Type | Severity | Component |
| --- | --- | --- | --- | --- |
| CVE-2019-2215 | A-141720095 | EoP | High | Binder |

CVE 记录（Google Android CNA）：

> "A use-after-free in binder.c allows an elevation of privilege from an application to the Linux Kernel."
> "Android ID: A-141720095"

Project Zero 的公开分析给出了机制：

> "The bug is a use-after-free (UAF) in the Binder driver. The binder_thread struct, defined in drivers/android/binder.c, has the member wait of the wait_queue_head_t struct type. wait is still referenced by a pointer in epoll, even after the binder_thread struct containing it is freed."
> "Normally, the wait queue used for polling on a file is guaranteed to be alive until the file's release handler is called. Rare cases require the use of POLLFREE. In contrast, the Binder driver only worked if you constantly removed and re-added the epoll watch. This is the underlying bug and the use-after-free is a symptom of that."

> — <https://googleprojectzero.blogspot.com/2019/11/bad-binder-android-in-wild-exploit.html>

译：`binder_thread` 结构体内嵌一个 `wait_queue_head_t` 类型的等待队列成员；当 `binder_thread` 被释放后，epoll 里仍有一个指针指向这个内嵌成员。常规约定是「用于轮询的等待队列在文件 release 之前必须保持存活」，少数场景需要 `POLLFREE` 来通知；而 Binder 驱动只有在「不断移除并重新添加 epoll watch」时才不出问题。官方判断：**这才是根因，UAF 只是症状**。

**这条案例的两个写作用途。**

- 它是「生命周期错配」最干净的范例：对象 A 内嵌的对象 B 被外部长期持有指针，而 A 的释放不受 B 的引用约束。要修的不是某个锁，而是**把 B 的生命周期绑定到持有者的生命周期**（或按约定发出 `POLLFREE`）。
- 它同时是一条**误传纠正**：Project Zero 全文对 `race`/`races`/`racy` 整词检索命中为 0（见未核实 H），文章用的是「生命周期保证被破坏」的措辞。因此不能写「Project Zero 称其为竞态条件」；本手册把它归入「生命周期类的并发缺陷」，是因为它的触发依赖两个上下文的执行交错，而不是因为原文用了 race 这个词。

**三条案例的归纳。**

| 案例 | 并发后果 | 出错的那一步 | 修复方向 |
| --- | --- | --- | --- |
| CVE-2025-38349 | UAF | 递减引用计数早于解锁 | 调整释放顺序 |
| CVE-2024-35970 | 死锁 | GC 路径锁顺序/滞留守卫 | 清理陈旧状态、修锁序 |
| CVE-2019-2215 | UAF | 内嵌等待队列的生命周期未随宿主绑定 | 生命周期绑定 / `POLLFREE` |

---

## 7. 检测与缓解：谁能看见并发缺陷的「后果」，谁看不见

并发缺陷本身很难被工具直接判定；能观测到的通常是它的**后果**（越界、UAF、死锁、对象状态机异常）。下面这张表只写有官方出处的能力，不按常识补全。

| 手段 | 它真正检查什么 | 对并发缺陷能做什么 | 明确的边界 |
| --- | --- | --- | --- |
| lockdep | 锁类的使用状态与锁依赖（锁顺序） | 发现锁反转死锁、hardirq/softirq 交叉违规 | 动态；只看发生过的序列；不查临界区内数据竞争；天然嵌套需人工标注 |
| debugobjects | 内核对象生命周期的状态机 | 报告「使用已释放/已销毁对象」「初始化活跃对象」等模式 | 只覆盖接入 `debug_object_*` 的对象类型；按需开启 |
| KASAN（`CONFIG_KASAN_HW_TAGS` 等三档） | 内存访问错误 | 让 UAF/越界在发生时暴露；HW_TAGS 被官方定位为可用于现场 | 不判定线程执行顺序，不能证明「没有竞态」 |
| Scudo chunk header 校验 | 同一 chunk header 是否被并发改写 | 在破坏发生并被访问时终止进程并留信号 | 缓解而非检测器；命中即终止；只在访问时触发 |
| `openat2()` resolve 限制 | 路径解析期间的组件变化 | 限制符号链接穿越与越界解析 | 只管解析阶段，不管已打开 fd 的后续使用 |
| RCU 宽限期 / 引用计数纪律 | 释放时机是否落在保证范围内 | 从设计上消除一类生命周期竞态 | 不是工具；写错释放点照样出事 |

**逐条的官方依据与读法。**

debugobjects 的定位：

> "debugobjects is a generic infrastructure to track the life time of kernel objects and validate the operations on those."
> "Usage of freed/destroyed objects"
> "debugobjects is not changing the data structure of the real object so it can be compiled in with a minimal runtime impact and enabled on demand with a kernel command line option."

> — <https://docs.kernel.org/core-api/debug-objects.html>

译：它跟踪内核对象的生命周期并校验对对象的操作，列出的第一类错误模式就是「使用已释放/已销毁的对象」；它不改动真实对象的数据结构，因此运行开销小，可按需用内核命令行开启。边界：只覆盖接入了 `debug_object_*` 的对象类型；该页**没有**出现 `CONFIG_DEBUG_OBJECTS` 字样（见未核实 G），所以本手册不写配置名。

KASAN 的三档：

> "Generic KASAN, enabled with CONFIG_KASAN_GENERIC, is the mode intended for debugging, similar to userspace ASan."
> "Hardware Tag-Based KASAN or HW_TAGS KASAN, enabled with CONFIG_KASAN_HW_TAGS, is the mode intended to be used as an in-field memory bug detector or as a security mitigation."
> "Software Tag-Based KASAN or SW_TAGS KASAN, enabled with CONFIG_KASAN_SW_TAGS, can be used for both debugging and dogfood testing, similar to userspace HWASan."

> — <https://docs.kernel.org/dev-tools/kasan.html>

译：GENERIC 用于调试（类比 ASan）；**HW_TAGS 是唯一被官方写成「可用于现场的内存缺陷检测器或安全缓解」的档位**；SW_TAGS 介于两者之间（调试 + dogfood，类比 HWASan）。写评估时要注意：KASAN 检的是**内存访问错误**，不是竞态；它能让竞态造成的 UAF 暴露，但不会告诉你「这两个线程的执行顺序错了」。

Scudo 的诊断文案是 `source.android.com` 上少见的、把内存错误与「缺锁」显式关联起来的一处：

> "race on chunk header: Two different threads are attempting to manipulate the same header at the same time. This is usually symptomatic of a race-condition or general lack of locking when performing operations on that chunk."

> — <https://source.android.com/docs/security/test/scudo>

译：同一个 chunk header 被两个线程同时改写，通常是该 chunk 上的操作存在竞态或缺少加锁。**但必须同时引用同一页的定位句**，否则会把它误读成检测器：

> "Scudo is more of a mitigation than a fully fledged memory error detector like AddressSanitizer (ASan)."

译：Scudo 更接近缓解措施，而不是 ASan 那样的完整内存错误检测器；官方另写明检测到堆破坏或可疑行为时会终止进程。所以它对并发缺陷的作用是「破坏真发生了、且被访问到的时候，给出一个信号并终止」，不是「检测竞态」。

**一个反向的边界：seccomp 不是并发缺陷的防线。** §6.1 的 CNA 文本已经说明，该 CVE 的调用面「即使在被收紧的 seccomp 沙箱与容器内也是允许的」。这是单条 CVE 的事实，不是一般规律——但它足以提醒评估者：**用「攻击者受 seccomp 限制」来排除内核竞态类提权，需要逐条核对调用面**。

---

## 8. 成因模式：七种落点

把前六节的材料按「出错的那一步」归类，Android 生态里最常见的落点是下面七种。每一种后面标出对应的官方依据，方便回查。

1. **检查—使用窗口**：先检查属性/权限/路径，再执行操作，中间状态可变。官方依据：`d_lookup()` 明确「只检测不阻止」rename 竞争（§5）；`openat2()` 的 resolve 标志正是为这类场景设计的。
2. **释放动作早于临界区结束**：在仍持锁/仍在临界区时递减引用计数或释放对象。官方依据：CVE-2025-38349 的 CNA 描述（§2、§6.1）。
3. **内嵌对象的生命周期未与宿主绑定**：外部（如 epoll）长期持有指向宿主内部成员的指针，宿主释放后指针悬空。官方依据：CVE-2019-2215 的 Project Zero 分析（§6.3）。
4. **回收/GC 路径的锁顺序与滞留守卫**：垃圾回收与正常路径共享锁，顺序或状态清理不当导致死锁。官方依据：CVE-2024-35970 的 CNA 描述（§6.2）。
5. **锁与中断上下文交叉**：同一把锁既可能在中断里取、又可能在开中断时取。官方依据：lockdep 的 hardirq-safe / hardirq-unsafe 规则（§4）。
6. **无锁读逃出临界区**：RCU 的读者把指针带出临界区，或在临界区外释放旧对象。官方依据：RCU 的时间保证与宽限期范围（§3）。
7. **把「检测」当「预防」**：seqlock 重试、`LOOKUP_BENEATH` 的锁检测、以及各种「检查后再看一眼」的写法，都是**检测**机制；它们缩小窗口但不消除窗口。官方依据：`d_lookup()` 的 "does not try to prevent this from happening, but only to detect when it happens."（§5）

**这七种模式的共同点**：判据都不是「有没有加锁」，而是「对象在窗口内的状态是否被约束」。这也是为什么 §7 的工具表里没有一把「万能锁」。

---

## 9. 观测点与自查

**对设备厂商 / 内核维护者。**

- 把 lockdep 的 splat 当作**必须处理的报告**，而不是噪音：它查的是锁顺序与上下文使用状态（§4），漏掉一条就可能在量产设备上变成死锁。
- 对「同一锁类多实例按层级取用」的合法模式显式标注嵌套层级，否则会得到误报；lockdep 不会自动识别天然顺序（§4）。
- 对象生命周期类问题优先用 debugobjects 的状态机暴露，而不是靠 KASAN 撞运气：后者只在错误访问真实发生时才有报告（§7）。
- 现场检测档位只有 KASAN 的 HW_TAGS 被官方写作 "in-field"（§7）；把它与调试档混为一谈会算错性能预算。
- SELinux/seccomp 收紧**不构成**对内核竞态类提权的防护，评估时要逐条核对调用面（§7 末段）。

**对应用与系统服务开发者。**

- 任何「先检查、再使用」的两步文件操作都要当成 TOCTOU 候选：检查与使用之间，路径、符号链接、挂载点都可能变化（§5）。
- 需要「解析结果在窗口内不变」的语义时，优先使用一次解析完成全部约束的接口，而不是「解析后自己再校验一遍」；内核侧的约束标志见 §5，用户态封装是否提供需按平台版本核对。
- 跨线程共享的对象，释放动作不要发生在临界区内部；引用计数的递减顺序与解锁顺序要对齐（§2、§6.1）。
- 使用 RCU 时，不要假设 `synchronize_rcu()` 之后就没有读者：它只等调用时刻之前已存在的临界区（§3）。

**对安全评估者。**

- 公告的 Type 列不是根因分类：同一个 `EoP` 可能对应 UAF，也可能对应死锁（§6.2）。要读 CVE 记录的 description 与 CNA 评估文本。
- 竞态类缺陷的可利用性不能只看「触发条件是否苛刻」：单次失败无副作用的缺陷可以无限重试（§1 的 AC:L 文本）。
- 「沙箱限制了系统调用」不能单独作为排除依据（§7 末段）。
- CVE 编号年份不代表 Android 已修：CVE-2024-35970 出现在 2025-12 的 Android 公告里（§6.2）。

**对个人用户与企业。**

- 并发缺陷没有用户可以自行观测的稳定信号：日志里通常只看到后果（崩溃、重启、异常终止），看不到根因。可行的处置仍是跟进补丁级别与厂商公告，而不是依赖本地检测。
- 企业侧评估设备时，把「内核与系统组件的补丁月份」作为硬指标；竞态类条目在公告里与其它 EoP 混排，无法凭标题区分。

---

## 10. 常见误传纠正

1. **「`refcount_t` 能防引用计数溢出导致的 UAF。」** 官方对照页只讨论内存序（§2）；饱和语义在该页零命中。计数类型不解决「释放顺序」问题，CVE-2025-38349 就是顺序错误。
2. **「RCU 读侧完全不加锁，所以不存在同步问题。」** 读者确实不需要原子操作与屏障，但必须进入 `rcu_read_lock()` 标记的临界区；保证也只在该临界区内成立（§3）。
3. **「`synchronize_rcu()` 返回后所有读者都结束了。」** 它只等调用时刻之前已在执行的临界区（§3）。
4. **「lockdep 能查所有并发缺陷。」** 它查锁使用状态与锁顺序；临界区内的数据竞争不在其检查范围（§4、§7）。
5. **「lockdep 是静态证明器。」** 它是动态校验器，只看实际发生过的序列，且需要人工标注嵌套层级（§4）。
6. **「KASAN 能检测竞态。」** 它检测的是内存访问错误；竞态造成的 UAF 会暴露，但执行顺序错误本身不会被判定（§7）。
7. **「Scudo 是内存错误检测器，能发现数据竞争。」** 官方定位是缓解措施；命中即终止进程，且只在破坏被访问时触发（§7）。它的 header 文案提到 race，是**诊断解释**，不是检测能力承诺。
8. **「Project Zero 把 CVE-2019-2215 称为竞态条件。」** 原文对 race/races/racy 整词检索为 0；它讲的是等待队列生命周期与 epoll 引用的错配（§6.3、未核实 H）。
9. **「公告里能直接筛出竞态类漏洞。」** 公告 Type 列只写 EoP/ID/RCE/DoS 等影响分类；2025-12 内核表逐行核对无 race 字样（未核实 D）。
10. **「加了 seccomp 沙箱，内核竞态类提权就打不到了。」** CVE-2025-38349 的 CNA 文本明确该调用面在受限 seccomp 沙箱内仍可达（§7）；能否阻断要逐条核对调用面。
11. **「`openat2()` 的限制标志就等于消灭了文件类 TOCTOU。」** 它只约束解析阶段；已打开 fd 的后续使用不在范围内（§5）。
12. **「CVE 编号是 2024 年，说明 Android 早就修了。」** CVE-2024-35970 出现在 2025-12 的 Android 公告里（§6.2）。

---

## 11. 与其他手册的分工

| 问题 | 归本手册 | 归哪本手册 |
| --- | --- | --- |
| 并发缺陷的判据、锁顺序、RCU 宽限期、路径解析期 TOCTOU | ✅ | — |
| 按组件定位的缺陷类型图谱（binder、驱动、图形、媒体各自的面） | 仅并发维度 | [底层漏洞类型图谱手册](/android-low-level-vulnerability-types/) |
| 内核分支、KMI、模块交付与补丁通道 | 仅作为评估背景 | [内核与 GKI 驱动攻击面手册](/android-kernel-gki-drivers/) |
| binder 接口面与组件导出 | 仅作案例 | [框架 IPC 与组件导出攻击面手册](/android-framework-ipc-export/) |
| SELinux 策略与 seccomp 的强制点 | 仅指出其边界 | [SELinux 策略与 seccomp 沙箱手册](/android-selinux-seccomp-sandbox/) |
| 公告字段、SPL、CVE 与 OSV 的对应 | 仅公告 Type 列的读法 | [漏洞编号与公告体系手册](/android-cve-and-security-bulletins/) |
| 定级与可利用性要素 | 仅「可重试」这一条 | [安全性评估与定级手册](/android-vulnerability-severity-assessment/) |
| 通用缺陷类型清单（不按并发维度切分） | 仅并发相关条目 | [通用系统漏洞类型手册](/android-common-system-vulnerabilities/) |
| 从漏洞到提权的链路与影响面 | — | [远程代码执行与权限提升手册](/android-rce-and-privilege-escalation/) |

---

## 参考

**官方一手来源（本手册逐字引用过）**

- `refcount_t` 与 `atomic_t` 的差异（内存序）— <https://docs.kernel.org/core-api/refcount-vs-atomic.html>
- lockdep 设计文档（锁使用状态与锁依赖）— <https://docs.kernel.org/locking/lockdep-design.html>
- RCU 是什么（读侧保证、宽限期）— <https://docs.kernel.org/RCU/whatisRCU.html>
- 路径解析（`openat2()` resolve、`LOOKUP_*`、`rename_lock`）— <https://docs.kernel.org/filesystems/path-lookup.html>
- debugobjects（对象生命周期状态机）— <https://docs.kernel.org/core-api/debug-objects.html>
- KASAN 三档（GENERIC / SW_TAGS / HW_TAGS）— <https://docs.kernel.org/dev-tools/kasan.html>
- Scudo（header 竞态文案、缓解定位）— <https://source.android.com/docs/security/test/scudo>
- Android Security Bulletin 2025-12-01 — <https://source.android.com/docs/security/bulletin/2025-12-01>
- Android Security Bulletin 2019-10-01 — <https://source.android.com/docs/security/bulletin/2019-10-01>
- Linux CNA：CVE-2025-38349 — <https://cveawg.mitre.org/api/cve/CVE-2025-38349>
- Linux CNA：CVE-2024-35970 — <https://cveawg.mitre.org/api/cve/CVE-2024-35970>
- Google Android CNA：CVE-2019-2215 — <https://cveawg.mitre.org/api/cve/CVE-2019-2215>
- Project Zero：Bad Binder（等待队列生命周期）— <https://googleprojectzero.blogspot.com/2019/11/bad-binder-android-in-wild-exploit.html>

**相邻手册**：见 §11。

---

## 未核实与失效来源

**A. `refcount_t` 的溢出/饱和语义**
推测：接近上限时饱和、减到零后再减会拒绝复用，从而把「计数回绕」变成可检测的失败。未核实原因：`refcount-vs-atomic.html` 全文对 `saturat`、`overflow` 零命中（该页只讲内存序）；`core-api/refcount.html` 返回 **404**；`core-api/kref.html` 亦无相关表述。饱和语义很可能只写在头文件注释与 `lib/refcount.c` 中，属源码而非文档站内容。**正文因此不写「`refcount_t` 防溢出」，只写内存序差异。**

**B. lockdep 的开启配置名**
推测：lockdep 由 `CONFIG_PROVE_LOCKING` 开启。未核实原因：`locking/lockdep-design.html` 全文**不含任何 `CONFIG_` 字符串**；`locking/locking.html` 返回 **404**。**正文只描述校验器行为，不写配置名。**

**C. `openat2(2)` 的 `RESOLVE_*` 常量名与逐条语义**
推测：`RESOLVE_NO_SYMLINKS` / `RESOLVE_BENEATH` / `RESOLVE_IN_ROOT` 分别对应第 12 条所述三类限制。未核实原因：`docs.kernel.org` 的路径解析页只使用内部名 `LOOKUP_*`，全文无 `RESOLVE_` 字样；`userspace-api/openat2.html` 返回 **404**；man 页不属本次允许来源，未抓取。**正文只在讲用户态时说「`openat2()` 的 `resolve` 字段」，不写常量名。**

**D. Android 公告表格中的 "race condition" 字样**
推测：某些公告行会标注竞态。未核实原因：对 2025-12-01 内核表逐行核对，Type 列只有 `EoP`/`ID`、Severity 只有 `Critical`/`High`，无竞态字样；2024-07-01 为旧版式（第五列是 "Updated AOSP versions"），同样没有。另：2025-10-01 页面抓取返回 200，但正文只有公告前言与 Q&A，整页无 `CVE-2025` 字符串与漏洞表，**该期无法用于举证**（疑为页面结构差异）。**正文因此反复提醒：Type 列不是根因分类。**

**E. MTS / CTS / STS 中关于 flaky 或并发测试的官方表述**
推测：测试体系文档会提到「时序相关偶发失败需重跑」之类规则。未核实原因：本轮抓取预算用于内核文档与公告，未抓取 `https://source.android.com/docs/security/test/mts?hl=en` 及 CTS 相关页。**正文不写测试体系对并发缺陷的处置规则。**

**F. HWASan / GWP-ASan 页面中的并发表述**
推测：这两份文档可能提到多线程场景。未核实原因：本轮只抓取并核对了 Scudo 页；HWASan 与 GWP-ASan 两页未抓取，故**只写「未核实」，不写「没有」**。

**G. `CONFIG_DEBUG_OBJECTS` 与裸 `CONFIG_KASAN` 的逐字出现**
推测：debugobjects 由 `CONFIG_DEBUG_OBJECTS` 开启、KASAN 总开关为 `CONFIG_KASAN`。未核实原因：`debug-objects.html` 全文无 `CONFIG_` 字符串（只说可按需用内核命令行开启）；`kasan.html` 只出现带后缀的 `CONFIG_KASAN_GENERIC` / `CONFIG_KASAN_SW_TAGS` / `CONFIG_KASAN_HW_TAGS`，**未出现裸 `CONFIG_KASAN`**。**正文只写带后缀的三个，不写总开关名。**

**H. Project Zero 用 "race condition" 描述 CVE-2019-2215**
推测：既然缺陷由 binder_poll 与线程退出并发触发，文章可能称之为竞态。未核实原因：对该文全文做整词检索，`race`/`races`/`racy` 命中均为 **0**（疑似命中的是 `stack trace` 的子串）；文章用的是「等待队列生命周期保证」与「UAF 是症状」。**正文因此只写「生命周期错配」，不写「PZ 称其为竞态」。**

---

## 联系方式

咨询 Android 系统详情请联系 https://t.me/one00190

---

*本手册仅供安全学习与防御研究使用；所有机制表述、公告列值与 CVE 描述均取自公开一手来源并附链接，未能核实的部分已单列于「未核实与失效来源」而非写入正文。*
