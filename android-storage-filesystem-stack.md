---
layout: page
title: "Android 存储与文件系统栈攻击面手册"
description: "把「应用写文件」拆成三条对象不同的机制：CE/DE 的路径与可用时机（`/data/user/0` vs `/data/user_de/0`）、vold 以 Binder 服务（`IVold`）管挂载与全部 FBE 密钥、以及 `/sdcard` 上真正做访问判断的是 MediaProvider 进程内..."
date: 2026-10-30
permalink: /android-storage-filesystem-stack/
---

> 存储是 Android 里被误解最深的一层。开发者的心智模型通常是「有一个磁盘，应用往里写文件」，平台的实际模型是：**同一块闪存被切成多个挂载点，各自有不同的密钥、不同的可见时机、不同的用户态中介，而应用看到的那条路径其实是一次 FUSE 调用的结果**。当你把「应用写文件」当成 `write(2)`，就会漏掉密钥释放时机、中介进程、以及路径与 inode 之间的那层换算——而这三处恰好是存储类缺陷最集中的地方。本手册只讨论公开且已修复问题的机制与自查方法，不含可利用代码。

---

## 0. 这篇手册解决什么问题

已发布的二十三篇里，《文件路径与 URI 处理类漏洞》讲的是**名字与对象之间的换算**（符号链接、归一化、检查与使用之间的窗口），《多用户、工作资料与进程隔离》讲的是**UID 与用户的分层**。这两篇都假设「文件系统本身是可信的、密钥已经就位」。

本篇补的是那个假设本身：**密钥什么时候可用、由谁保管、谁在做用户态中介**。三个问题分别对应三条独立的攻击面：

1. **密钥时序**：设备刚开机、用户还没解锁时，哪些数据是可读的？这个窗口由什么保证不会被拉长？
2. **中介语义**：`/sdcard` 上的读写不是直接落盘，而是穿过一个运行在应用进程里的 FUSE 实现——它的判断逻辑就是访问控制的真正实现。
3. **完整性来源**：增量安装（IncFS）让「文件还没下载完就能执行」成为可能，这件事的安全性不来自文件系统，而来自签名。

---

## 1. 存储分层的真实形状

### 1.1 CE 与 DE：两种可用时机

官方对两者的定义是：

> **CE（Credential Encrypted）**: "the default storage location and only available after the user has unlocked the device."
> **DE（Device Encrypted）**: "a storage location available both during Direct Boot mode and after the user has unlocked the device."

注意 CE 的定义里那句 "the default storage location"——它说明**绝大多数应用数据在 CE 里**，而 CE 在解锁前不可用。这对安全的影响是双向的：

- **好的一面**：设备关机或刚开机未解锁时，CE 里的数据在密码学上不可读。取证与「捡到手机」场景下这是主要防线。
- **需要小心的一面**：任何需要「开机即运行」的组件（闹钟、来电、企业 MDM 代理）必须把数据放进 DE，而 DE 的密钥在**验证启动成功后**就可用。官方对 DE 的表述是："Device encrypted storage contains data encrypted with a key that is only available after a device has performed a successful verified boot."——**不是「解锁后」，而是「验证启动后」**。

### 1.2 路径表（最容易记反的一处）

| 路径 | 归属 | 说明 |
|---|---|---|
| `/data/data` | CE | `/data/user/0` 的别名 |
| `/data/user/<userId>` | CE | 按用户编号的 CE 根 |
| `/data/user_de/<userId>` | DE | 按用户编号的 DE 根 |
| `/data/media/<userId>` | CE | 模拟存储的**后端**目录，注意它属 CE 而非独立分区 |
| `/mnt/user/<userId>/emulated/0` | CE | 模拟存储的**挂载视图** |
| `/storage/emulated/0` | CE | 应用通常看到的路径 |

AOSP `init.rc` 里能直接读到这几行的落地方式：

```
mkdir /data/media 0770 media_rw media_rw encryption=None
mkdir /mnt/user/0/emulated/0 0755 root root
mount none /mnt/user/0 /storage bind rec
```

三个可以直接读出来的事实：

- `/data/media` 被显式标了 `encryption=None`——它不做单独的目录级加密，其内容保护来自 `/data` 整体。
- `/storage` 是 `/mnt/user/0` 的 **bind mount**，所以 `/storage/emulated/0` 与 `/mnt/user/0/emulated/0` 是同一份数据的两个名字。
- 上述路径**按用户编号展开**，这解释了为什么多用户设备上 `/storage/emulated/10` 会存在——它不是「另一个磁盘」，而是同一个 `/data` 下的另一个编号目录。

**常见记反**：把 `/data/user/0` 说成 DE、把 `/data/user_de/0` 说成 CE。正好相反。

### 1.3 谁在管挂载

`vold` 的官方定位是："vold (system/vold) provides the functionality for managing storage devices and volumes on Android."，以及 "Mounting of physical external storage volumes is handled by `vold`, which performs staging operations to prepare the media before exposing it to apps."

一个常被写错的细节是**控制接口形态**。很多资料写「vold 通过 `/dev/socket/vold` 通信」，但现代实现里 `vdc.cpp` 是**通过 Binder 取服务名 `"vold"`**、`interface_cast<android::os::IVold>`，`vold.rc` 与 `main.cpp` 中并没有 `socket vold` 的声明。vold 确实持有 netlink 套接字（`NetlinkManager`），但那是用来收内核 uevent 的，不是应用侧命令通道。所以准确写法是：**vold 是一个 Binder 服务（IVold），不是 socket 服务**。

`vold` 同时也是 FBE 密钥的唯一管理者：官方原话 "All FBE keys are managed by `vold` and are stored encrypted on-disk, except for the per-boot key which is not stored at all."。

---

## 2. 密钥链：从口令到 CE 密钥

这一节是全篇最需要精度的部分，因为它决定了「拿到闪存镜像能读到什么」。

### 2.1 CE 密钥是怎么解封的

官方给出的链路是：

> "`vold` encrypts each CE key for internal storage using an AES-256-GCM key derived from the user's _synthetic password_."
> "The synthetic password is an immutable high-entropy cryptographic secret that is randomly generated for each user."
> "If the device doesn't implement Weaver, then `LockSettingsService` instead uses the stretched LSKF as a Gatekeeper password."

把这三句连起来，链路是：

```
用户输入的口令/图案/指纹解锁凭据
      ↓（scrypt/PBKDF2 拉伸，具体参数由厂商配置）
   合成口令派生材料
      ↓（Weaver 硬件限频校验，或无 Weaver 时走 Gatekeeper 校验）
   synthetic password（随机生成、不可变）
      ↓（HKDF 派生）
   AES-256-GCM 密钥
      ↓（解密 vold 保存的封装密钥）
   CE 密钥 → 挂载/解密 /data/user/<id>
```

四个必须写对的点：

1. **synthetic password 不是从口令算出来的**，而是**随机生成**后**用口令保护**的。这意味着改口令不需要重新加密全部数据——只重新封装那一把密钥。这也解释了为什么 Android 上改密码是秒级操作。
2. **CE 密钥由 vold 用「由 synthetic password 派生的 AES-256-GCM 密钥」加密后存盘**——不是由 Gatekeeper 令牌直接解封。Gatekeeper 令牌只用于授权 authentication-bound keys（例如需要用户认证才能使用的 Keystore 密钥）。把这两条链路混为一谈，是 TEE 类文章里最常见的错误之一。
3. **Weaver 的作用是限频**，它把「猜口令」从纯计算问题变成硬件节流问题。没有 Weaver 的设备退回 Gatekeeper，此时抗爆破能力取决于 Gatekeeper 的节流实现与 TEE 保护。
4. **per-boot key 不落盘**（"not stored at all"），所以重启即失效——这解释了为什么很多「重启后需要重新登录」的行为无法被持久化绕过。

### 2.2 元数据加密：FBE 覆盖不到的那部分

FBE 加密的是**文件内容与文件名**，但文件系统自身还有大量元数据（目录结构、inode 属性、大小、时间戳、配额）。官方对此的处理是元数据加密：

> "a single key present at boot time encrypts whatever content is not encrypted by FBE."
> 该密钥 "is protected by KeyMint (previously Keymaster), which in turn is protected by Verified Boot."
> FBE 未覆盖的部分 "known as filesystem metadata"。
> Android 11 起设备 "must have metadata encryption on internal storage enabled."

三个精度要求：

- 是 **KeyMint/Keymaster** 保护，不是笼统的「TEE」。写「密钥在 TEE 里」不算错但不够准，写「由 TEE 直接提供」则过度。
- 强制起点是 **Android 11**，不是 10。这条被写错的频率很高（与「FBE 自 Android 10 强制」「metadata encryption 自 Android 11 强制」这对组合经常被合并成一句错话）。
- 元数据加密的密钥「开机即在」（present at boot time），所以它**不能**保护「未解锁前的数据机密性」——那是 CE 的职责。它保护的是「拿到裸闪存也看不出目录结构」这一层。

---

## 3. 外置存储的中介：从 sdcardfs 到 FUSE

### 3.1 迁移的时间线与边界

> "Android 11 deprecated SDCardFS and made FUSE the default solution for storage emulation."
> "In Android 11 or higher, SDCardFS is deprecated and MediaProvider becomes the file system handler (for FUSE) for external storage, making the file system on external storage and the MediaProvider database consistent."

但有一条常被忽略的**限定语**：

> "SDCardFS is deprecated on devices that launch with Android 11 or higher and run kernel version 5.4 or higher."
> "Devices that launch with Android 11 or higher but run kernel version 4.19 or lower can continue to use SDCardFS, but Google doesn't provide additional support."

所以「Android 11 起 sdcardfs 被彻底移除」是不严谨的说法。准确表述是：**以 Android 11+ 启动且内核 5.4+ 的设备弃用 sdcardfs；内核 4.19 及以下的设备仍可继续使用，但不再获得支持**。这条限定对自查很重要：在低内核机型上按「FUSE 模型」推断行为会得出错误结论。

另外，**ESDFS 在 AOSP 文档中没有任何记载**，属厂商私有实现。把它当作 AOSP 的一部分来写是**未核实**的。

### 3.2 FUSE 实现在哪里跑

这是本节最反直觉的一点。官方描述：

> "As the userspace handler for the FUSE file system, MediaProvider can intercept kernel calls and ensure the file operations are privacy safe."
> "In Android 11 and higher, MediaProvider is also a modular system component (a Mainline module)"

代码侧可以确认它是一个 **Java 线程 + JNI**，而不是一个独立原生守护进程：

- `packages/providers/MediaProvider/.../fuse/FuseDaemon.java`：`public final class FuseDaemon extends Thread {`，含 `private native long native_new(MediaProvider mediaProvider);` 等 JNI 方法。
- `ExternalStorageServiceImpl.onStartSession` 中创建并启动：`FuseDaemon daemon = new FuseDaemon(mediaProvider, this, deviceFd, sessionId, …)`，随后 `daemon.start();`。
- 清单包名为 `com.android.providers.media.module`，application 未设 `android:process`（仅 PhotoPicker 用 `:PhotoPicker`），因此 **FUSE 线程运行在 MediaProvider 应用进程内**。

安全含义有三层：

1. **访问控制的真正实现是 Java 代码**，不是内核权限位。`/sdcard` 上「A 应用读不到 B 应用的文件」这条保证，落点是 `FuseDaemon` 里的判断逻辑（结合 MediaProvider 数据库与 app-op 状态），而不是 DAC/MAC。
2. **它是一个可独立更新的 Mainline 模块**——好处是修得快，代价是**「同一个 Android 版本号」不再意味着同一套存储访问策略**。核对设备行为时要看模块版本，不能只看系统版本。
3. **服务入口受签名级权限保护**：`ExternalStorageServiceImpl` 由 `BIND_EXTERNAL_STORAGE_SERVICE` 保护。这条权限是这套机制不被第三方「插一脚」的前提。

### 3.3 scoped storage 与「全文件访问」

`MANAGE_EXTERNAL_STORAGE` 的实际授权范围，官方写得非常具体：

> "Read and write access to all files within shared storage."
> "Write access to all internal storage directories except `/Android/data/`, `/sdcard/Android`, and most subdirectories of `/sdcard/Android`. This write access includes direct file path access."
> "Apps that are granted this permission still can't access the app-specific directories that belong to other apps, because these directories appear as subdirectories of `Android/data/` on a storage volume."
> "The `/sdcard/Android/media` directory is part of shared storage."

四条连起来读，可以得到一条清晰的「做不到清单」：

- **拿不到别的应用的私有目录**（`Android/data/<pkg>` 被排除）。
- **拿不到别的应用的内部存储**（`/data/data/<pkg>` 与共享存储无关）。
- **拿不到 `Android/obb` 等受特殊管理的子目录**（属于 "most subdirectories of `/sdcard/Android`"）。
- **能拿到 `Android/media`**——这一条常被忽略，而它恰恰是应用可以把数据「放在看起来像私有、实际属于共享存储」的位置的原因。

---

## 4. IncFS：完整性不来自文件系统

增量安装的官方描述是：

> "the Incremental File System (IncFS) kernel module introduced in Android 11"
> "a new virtual file system that sits on top of the existing Android file system"
> "enables the Android OS to receive streamed APKs over the Android Debug Bridge (ADB)"

核心机制是：文件可以「部分存在」——读取未被下载的区段会阻塞，直到数据到达。这让「边下边玩」成为可能。

但安全性的来源必须说清楚。APK Signature Scheme v4 是配套的完整性方案：

> v4 "is based on the Merkle hash tree calculated over all bytes of the APK"
> "`signing_info` is passed into IncFS as a whole blob. IncFS extracts root hash from the blob."

所以准确表述是：**IncFS 是承载层，它从 v4 签名块里取出 Merkle 根哈希，并据此校验按页到达的数据；完整性根来自签名，不来自文件系统本身**。写成「incfs 自己做 Merkle 校验」虽然不算全错，但把责任主体搞混了——真正的信任锚是签名密钥。

这条区分在评估时很关键：如果一台设备上 `CONFIG_INCREMENTAL_FS` 关闭，那么增量安装不可用，但**安装完整性并不因此变弱**（还有 v2/v3 签名与 v4 的整包校验路径）；反过来，如果签名校验被削弱，IncFS 也救不了。

---

## 5. 文件系统本体：ext4 / f2fs / EROFS

官方关于「哪些文件系统会被频繁打补丁」的表述是：

> "the following file systems receive more frequent patches through the Android security patch release schedule:" 后列 `ext4`、`f2fs`、`incfs`、`EROFS`
> "file systems that don't support `fscrypt` or `fsverity` aren't suitable for production use."

第二句是这一节的重点：**文件系统能不能用于生产，判据是它是否支持 `fscrypt`（加密）与 `fsverity`（只读完整性）**，而不是性能或流行度。这两个特性是上层安全模型的地基：

- 没有 `fscrypt`，FBE 无法落地；
- 没有 `fsverity`，只读系统镜像的完整性只能靠 dm-verity 在块层保证，文件级校验缺位。

挂载选项的安全影响可以举一个直接例子。Cuttlefish 的 fstab 里 `/data` 的一行是：

```
/dev/block/by-name/userdata /data ext4 nodev,noatime,nosuid,errors=panic wait,formattable,first_stage_mount
```

其中 `nodev`、`nosuid` 阻止设备文件与 setuid 位在这个挂载点上生效；`errors=panic` 让文件系统错误直接 panic 而不是带着损坏状态继续跑——这是「宁可停機也不带病运行」的取舍。评估自定义 ROM 时，这类挂载选项被悄悄改掉是很常见的「优化」，而它会实质削弱防线。

需要明确列为**未核实**的两条（在二手资料里流传很广）：

- 「GKI 强制 `/data` 使用 f2fs」——GKI 文档中未见此要求。
- 「`force_fsync` 是 Android fstab 的标准标志」——仅在个别厂商 fstab 中零散出现，不是平台规范。

---

## 6. 成因分析：存储栈的缺陷从哪来

### 6.1 「路径」与「对象」在多层之间换算

存储栈里同一个对象至少有三层名字：应用的路径字符串 → VFS 的 dentry/inode → 底层的块/页。每一层都可能做归一化（Unicode、大小写、`.`/`..`、尾随斜杠），而**归一化规则在层与层之间并不一致**。

这是 `CVE-2024-43093` 的成因类别：官方描述是 "In shouldHideDocument of ExternalStorageProvider.java, there is a possible bypass of a file path filter designed to prevent access to sensitive directories due to incorrect unicode normalization."（CWE-176）。注意三个要点：它与**路径过滤**有关、根因是**Unicode 归一化不正确**、影响是**绕过对敏感目录的访问限制**。这是本篇里少数真正属于「存储访问控制」的案例。

方法论上可以提炼成一句：**只要一处代码用「字符串比较」判断能否访问，另一处代码用「文件系统解析」决定访问到哪里，两者之间就必然存在可利用的差异**。防御做法是把判断和使用都收敛到 FD 上（打开后校验 FD，而不是校验字符串后再打开），这与《文件路径与 URI 处理类漏洞》里的结论一致。

### 6.2 时序：密钥可用性与组件启动顺序

CE/DE 的设计本身就引入了「不同数据在不同时刻可用」的状态机。缺陷常常出在**状态机被绕过或误判**：

- 组件在 DE 阶段就把敏感数据写进 DE，导致「未解锁也可读」的范围被动扩大。
- 应用把「设备已启动」误当成「用户已解锁」，于是在 DE 上下文里创建了本该在 CE 的文件（通常是回退逻辑写错）。
- 系统服务在用户切换/锁屏时没有重新评估已授予的访问权，使授权跨越了状态变化。

这一类没有典型 CVE 编号可引，但它是**企业设备管理里最常见的实际风险**：一台配了 Direct Boot 组件的设备，其 DE 区里有什么，往往没人清点过。

### 6.3 中介层的语义漂移

只要 `/sdcard` 的访问要穿过 `FuseDaemon`，就存在「内核看到的」与「Java 判断的」两套语义。二者不一致时的表现通常是：

- 某类文件操作（`rename`、`unlink`、`openat` 特定 flag 组合）在 FUSE 路径上没有对应的权限判断分支，于是落到默认放行/默认拒绝。
- MediaProvider 数据库与真实目录内容不同步（例如通过非标准路径写入的文件不在库里），导致基于库的过滤失效。
- 多用户/工作资料切换时，会话（session）与挂载点生命周期不匹配。

这类问题的修复通常发生在 Mainline 模块里，**不产生 CVE**，只在模块版本说明里体现。这解释了为什么「存储访问相关的行为在同一 Android 版本的不同设备上不一样」。

### 6.4 文件系统解析器的内存安全

ext4/f2fs 的元数据解析在内核态，面对的是**可能被篡改的镜像**（例如通过可移动介质、或已被利用的写原语）。历史上 `CVE-2017-18249` 就是 F2FS 的案例（Kernel components / F2FS / EoP High）。这类缺陷的特点：

- 触发条件是**挂载/解析**，不需要用户交互；
- 影响是内核态，权限提升；
- 修复在 kernel LTS，而不是框架层——所以**内核版本比 Android 版本更能反映实际风险**。

### 6.5 权限与路径的双重来源

`MANAGE_EXTERNAL_STORAGE` 这类宽权限的存在，制造了一种「我已经有权限了，所以路径检查可以放松」的错觉。而 §3.3 的四条引文说明：**权限再宽，也有明确的排除项**。当实现把「有宽权限」当作「全部可达」时，就越过了这些排除项——这是应用侧存储越权最常见的自我说服路径。

---

## 7. 公开案例台账（逐条核实归属）

| 编号 | 实际组件（公告原文要点） | 与存储栈的关系 |
|---|---|---|
| CVE-2024-43093 | "In shouldHideDocument of ExternalStorageProvider.java … bypass of a file path filter … due to incorrect unicode normalization"（Framework / Documents UI，A-341680936，EoP High，Android 12–15；含在野利用迹象） | **直接相关**：存储访问控制 |
| CVE-2017-18249 | Kernel components，组件 F2FS，EoP High（2018-08 公告） | **直接相关**：文件系统解析 |
| CVE-2020-0034 | Media framework（libvpx），A-62458770，High，8.0/8.1（2020-03 公告） | **无关**：媒体编解码 |
| CVE-2019-2122 | Framework（LockTaskController），A-127605586，EoP High，7.0–9（2019-08 公告） | **无关**：锁任务模式 |
| CVE-2021-39623 | Media Framework（`SimpleDecodingSource.cpp`），A-194105348，EoP High，9–12（2022-01 公告） | **无关**：媒体解码 |
| CVE-2022-20452 | Framework（`BaseBundle.java` `initializeFromParcelLocked`），A-240138318（2022-11 公告） | **无关**：Binder 序列化 |

**vold / FUSE 的专项 CVE 在现行公告中未能核到可靠条目**（仅存在很早期的历史条目）。这不是说这一层没出过问题，而是说：**它的修复主要走 Mainline 模块更新与厂商补丁，不走公开 CVE 通道**。这与 §6.3 的结论互相印证。

---

## 8. 防护技术

### 8.1 设计层

| 措施 | 做法 | 挡住什么 |
|---|---|---|
| DE 区最小化 | 只把「开机必须可用」的数据放 DE，并显式清点 | 未解锁窗口的暴露面扩大 |
| 一律走 FD | 校验对象而不是校验字符串：先 `open`，再对 FD 做 `fstat`/`readlink /proc/self/fd/N` 校验 | 检查与使用之间的替换 |
| 不自己判断路径归属 | 用 `FileProvider`/SAF 授权，而不是自写「前缀匹配」 | 归一化差异绕过 |
| 存储不假设权限=可达 | 显式处理 `Android/data`、`Android/obb` 的排除项 | 宽权限下的越界 |
| 只读内容用 fs-verity | 对只读资源做文件级完整性校验 | 静默篡改 |

### 8.2 实现层

1. **归一化只做一次，且用同一实现。** 不要在 A 处用 `String.equals`，在 B 处用文件系统解析。若必须比较路径，先解析到 canonical 形式，再比较，且比较时带上分隔符边界（`/a/b` 不是 `/a/bc` 的前缀）。
2. **符号链接显式处理。** 需要跟随就跟随并校验最终目标，不需要跟随就用 `O_NOFOLLOW`；「碰巧没有符号链接」不是安全假设。
3. **跨用户/跨会话的操作要带 userId。** 多用户设备上所有存储 API 都应按用户编号隔离，不应存在「当前用户」的隐式默认值。
4. **MediaProvider 数据库与目录双写一致。** 若业务需要绕过 MediaStore 写文件，必须考虑「库里没有 → 过滤失效」的后果。
5. **不在锁屏状态下创建 CE 文件。** 检测到 DE 上下文时，写入应落到 DE 目录或延后，而不是「先写 CE 再说」。

### 8.3 验证层

- **密钥时序**：`adb shell dumpsys mount`、`getprop` 中的加密状态属性、以及实际尝试读取 CE/DE 路径，三者交叉验证。仅看属性不够——属性可能反映了配置而非实际状态。
- **中介层版本**：把 MediaProvider（`com.android.providers.media.module`）的模块版本纳入资产清单，与系统版本分开记录。
- **挂载选项基线**：抓 `mount` / `cat /proc/mounts`，对 `/data`、`/storage` 的 `nosuid,nodev,noexec`（适用时）做基线比对。
- **文件系统特性**：`/proc/filesystems`、内核 config（若可得）确认 `fscrypt`/`fsverity`/`incfs` 的存在与否；缺失时按「安全模型降级」处理，而不是当作功能缺失。

---

## 9. 自查与观测清单

```bash
# 1. 当前挂载与选项
adb shell cat /proc/mounts | grep -E " /data | /storage | /mnt/user "

# 2. 加密状态
adb shell dumpsys mount | head -60
adb shell getprop | grep -iE "vold|crypt|fbe|metadata"

# 3. 多用户与存储目录
adb shell ls -ld /data/user /data/user_de /data/media /storage/emulated

# 4. FUSE 是否在岗（MediaProvider 进程与挂载点）
adb shell ps -A | grep -i mediaprovider
adb shell ls -l /mnt/user/0/emulated/0

# 5. 文件系统类型与特性
adb shell cat /proc/filesystems | grep -E "f2fs|ext4|erofs|incfs"
adb shell mount | grep -E "f2fs|ext4"

# 6. IncFS 挂载情况（增量安装会话存在时才有）
adb shell mount | grep -i incfs

# 7. 存储权限与 app-op
adb shell appops get <包名> MANAGE_EXTERNAL_STORAGE
adb shell pm list packages -3 | head
```

判读要点：

- **第 2 条里，加密「已启用」不等于「CE 已解锁」**。锁屏状态下两者的区别才是重点。
- **第 4 条决定第 1 条的解释方式**：低内核机型（4.19 及以下）可能仍是 sdcardfs，此时 `/sdcard` 的行为模型不同。
- **第 6 条为空是正常的**——IncFS 只在增量安装会话期间挂载。

---

## 10. 常见误传纠正

1. **「`/data/user/0` 是 DE」**——反了。CE 是 `/data/user/0`，DE 是 `/data/user_de/0`。
2. **「`/data/media/<userId>` 是独立的物理分区」**——不是，它在 `/data` 下且属 CE 类别。
3. **「vold 通过 `/dev/socket/vold` 通信」**——现代实现是 Binder 服务 `vold`（`IVold`），未找到 socket 声明。
4. **「FUSE 守护进程是独立 native 进程」**——是 MediaProvider 进程内的 Java `FuseDaemon` 线程 + JNI；独立原生守护进程的是更早的 `/system/bin/sdcard`。
5. **「Android 11 起 sdcardfs 彻底移除」**——不严谨：仅「Android 11+ 启动且内核 5.4+」的设备弃用；4.19 及以下仍可用（无支持）。
6. **「元数据加密自 Android 10 强制」**——原文是 Android 11 起内部存储必须启用。
7. **「FBE 自 Android 11 强制」**——FBE 强制始于 Android 10；Android 11 起强制的是 metadata encryption。
8. **「CE 密钥由 Gatekeeper 令牌解封」**——错。CE 密钥由 vold 用「由 synthetic password 派生的 AES-256-GCM 密钥」解封；Gatekeeper 令牌只授权 authentication-bound keys。
9. **「synthetic password 是从用户口令算出来的」**——错，它是随机生成、不可变、再用口令保护的。
10. **「incfs 自己做 Merkle 校验」**——完整性根来自 APK Signature Scheme v4（`signing_info` 中的 root hash），incfs 只是承载层与提取方。
11. **「GKI 要求 `/data` 用 f2fs」**——未获任何一手文档支持，未核实。
12. **「有 `MANAGE_EXTERNAL_STORAGE` 就能读写全部共享存储」**——错，`Android/data`、`Android/obb` 等被明确排除；`Android/media` 则在共享存储内。
13. **CVE 张冠李戴**：CVE-2020-0034 是 libvpx、CVE-2019-2122 是 LockTaskController、CVE-2021-39623 是媒体解码、CVE-2022-20452 是 `BaseBundle`——都不是存储栈漏洞。

---

## 未核实与失效来源

- **A. `/sdcard` 为符号链接。** 未在 `init.rc` 中核到对应行，正文只使用 `/storage/emulated/0` 与 `/mnt/user/0/emulated/0` 这对已核实的关系。
- **B. ESDFS。** AOSP 文档无记载，属厂商私有实现；正文不将其纳入官方模型。
- **C. GKI 与 `/data` 文件系统选择。** 未见官方要求使用 f2fs，按未核实处理。
- **D. `force_fsync` 挂载标志。** 仅在个别品牌 fstab 中零散出现，非平台规范。
- **E. `StorageManagerService` → vold 的逐行调用链。** 未取到原文，正文只描述职责分工。
- **F. vold / FUSE 的专项 CVE 条目。** 现行公告中未核到可靠条目；历史条目（如 2011 年前后的早期漏洞）不在本文范围。
- **G. 「元数据密钥在 TEE 中」。** 官方措辞是「由 KeyMint（原 Keymaster）保护，KeyMint 又由验证启动保护」；「在 TEE 中」属合理推断，正文按官方措辞表述。

---

## 参考

- 基于文件的加密（CE/DE、synthetic password、vold）：[source.android.com/docs/security/features/encryption/file-based](https://source.android.com/docs/security/features/encryption/file-based)
- 元数据加密：[source.android.com/docs/security/features/encryption/metadata](https://source.android.com/docs/security/features/encryption/metadata)
- 存储配置与 vold 挂载：[source.android.com/docs/core/storage/config](https://source.android.com/docs/core/storage/config)
- Scoped storage 与 MediaProvider：[source.android.com/docs/core/storage/scoped](https://source.android.com/docs/core/storage/scoped)
- SDCardFS 弃用与内核限定：[source.android.com/docs/core/storage/sdcardfs-deprecate](https://source.android.com/docs/core/storage/sdcardfs-deprecate)
- FUSE passthrough：[source.android.com/docs/core/storage/fuse-passthrough](https://source.android.com/docs/core/storage/fuse-passthrough)
- Incremental File System：[source.android.com/docs/core/architecture/kernel/incfs](https://source.android.com/docs/core/architecture/kernel/incfs)
- APK Signature Scheme v4：[source.android.com/docs/security/features/apksigning/v4](https://source.android.com/docs/security/features/apksigning/v4)
- 内核文件系统支持：[source.android.com/docs/core/architecture/android-kernel-file-system-support](https://source.android.com/docs/core/architecture/android-kernel-file-system-support)
- Direct Boot 与 DE：[developer.android.com/privacy-and-security/direct-boot](https://developer.android.com/privacy-and-security/direct-boot)
- 全文件访问权限：[developer.android.com/training/data-storage/manage-all-files](https://developer.android.com/training/data-storage/manage-all-files)
- `FuseDaemon.java`：[android.googlesource.com/platform/packages/providers/MediaProvider/+/refs/heads/main/src/com/android/providers/media/fuse/FuseDaemon.java](https://android.googlesource.com/platform/packages/providers/MediaProvider/+/refs/heads/main/src/com/android/providers/media/fuse/FuseDaemon.java)
- `vold.rc` / `vdc.cpp`：[android.googlesource.com/platform/system/vold](https://android.googlesource.com/platform/system/vold)
- Cuttlefish fstab：[android.googlesource.com/device/google/cuttlefish/+/e3322a8/shared/config/fstab.ext4](https://android.googlesource.com/device/google/cuttlefish/+/e3322a8/shared/config/fstab.ext4)
- 公告：[source.android.com/docs/security/bulletin](https://source.android.com/docs/security/bulletin)

---

## 联系方式

如果你在企业里做终端数据保护或合规评审，需要就「CE/DE 数据分布清点」「Direct Boot 组件风险」「共享存储访问控制核查」做一次针对现网设备或自有应用的审计，可以通过 Telegram 联系：<https://t.me/one00190>

---

*本手册仅用于防御与教育目的，讨论的均为公开且已被修复问题的机制与设计缺陷，不包含任何可直接利用的代码。文中标注「未核实」的内容不应作为事实引用；发现错误欢迎指出。*
