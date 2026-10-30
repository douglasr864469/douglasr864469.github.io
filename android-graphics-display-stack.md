---
layout: page
title: "Android 图形与显示栈攻击面手册"
description: "把「一块像素到屏幕」的三段结构讲透：BufferQueue 由消费者创建并拥有且可与生产者跨进程、SurfaceFlinger 是独立 system 进程且是唯一能修改显示内容的服务、HWC 决定合成路径；另含 gralloc 的 HIDL 谱系自 `@2.0` 起（gralloc1 并非 HIDL）、..."
date: 2026-10-30
permalink: /android-graphics-display-stack/
---

> Android 的图形栈是「所有应用共享的一条管道」：每个窗口把自己的像素写进一块缓冲区，SurfaceFlinger 把所有可见缓冲区合成一帧，硬件合成器再把这一帧送到屏幕。这条管道的每个接缝都在**不同进程、不同权限域、甚至不同厂商代码**之间。它是设备上唯一一条「所有应用的数据都必须流经同一个特权进程」的路径——也正因为如此，它既是隐私泄漏的高价值目标，又是权限提升的高价值目标。本手册只讨论公开且已修复问题的机制与自查方法，不含可利用代码。

---

## 0. 这篇手册解决什么问题

已发布的手册里，《摄像头、麦克风与屏幕捕获面》讲的是**捕获授权**（谁能录屏、谁被同意），《HAL 与 Treble 边界》讲的是**跨分区接口**。两者都触及图形栈，但都没有进入管道内部：一个讲「能不能拿」，一个讲「接口怎么定义」。

本篇补的是中间那段：**一块像素从应用到屏幕要经过谁的手**。三个问题：

1. **谁拥有缓冲区？** 答案是「消费者拥有队列，生产者只是借用」——这个所有权方向决定了谁能读别人的像素。
2. **谁在合成？** 合成发生在一个独立特权进程里，它必须信任来自应用的大量元数据。
3. **谁能看屏幕？** 「安全窗口」与「受保护显示」是两个不同的概念，混用会导致错误的隐私结论。

---

## 1. 管道：从应用到屏幕

### 1.1 三段结构

```
应用进程                      SurfaceFlinger 进程                显示硬件
─────────                     ───────────────────                ────────
Surface（生产端）  ──Binder──▶  Layer（消费端）
  │                              │
  └─ dequeueBuffer               ├─ 收集可见 layer
     （拿到空 buffer）            ├─ 询问 HWC 如何合成
  └─ 写入像素                    ├─ GPU 合成 or 硬件叠加
  └─ queueBuffer                 └─ 提交到显示
     （交还给队列）
```

官方对 BufferQueue 的定义点明了所有权关系：

> "The `BufferQueue` class connects components that generate buffers of graphical data (_producers_) to components that accept the data for display or further processing (_consumers_)."
> "Consumers create and own the `BufferQueue` data structure and can exist in different processes than their producers."

**第一句的后半句是全篇最重要的一句话**：消费者**创建并拥有**队列，且**可以与生产者不在同一进程**。这意味着：

- 应用（生产者）拿到的缓冲区是「借来的」；
- 队列的所有权在 SurfaceFlinger（消费者）一侧；
- 因此**「应用能否读到别的应用的像素」这个问题，答案取决于它能否拿到那个队列的句柄**，而不是取决于内存权限位。

跨进程那一层接口的官方描述是：

> "This class defines the Binder IPC interface for the producer side of a queue of graphics buffers."
> "The underlying mechanism is a BufferQueue, which implements BnGraphicBufferProducer."

且有一句历史提示："This class was previously called `ISurfaceTexture`."——看到旧资料里的 `ISurfaceTexture`，应当知道它就是今天的 `IGraphicBufferProducer`。

### 1.2 SurfaceFlinger 的位置

> "SurfaceFlinger accepts, composes, and sends buffers to the display."
> "After SurfaceFlinger has collected all buffers for visible layers, it asks the Hardware Composer (HWC) how it should perform composition."
> "SurfaceFlinger is the only service that can modify the content of the display."

**「唯一能修改显示内容的服务」**这句话解释了两件事：

1. 为什么所有应用画面都必须在它这里汇合——不是架构选择，而是硬件事实（只有一个显示控制器）。
2. 为什么它是提权的高价值目标——拿到它的执行权，等于拿到「屏幕上显示什么」的完全控制权（可覆盖任何界面，包括系统对话框）。

它是**独立进程**，不是 `system_server` 的一部分。init 服务定义可以直接读到：

```
service surfaceflinger /system/bin/surfaceflinger
    user system
    group graphics drmrpc readproc
```

这条很值得记住：`user system` 说明它不以 root 运行，`group graphics` 说明它通过组权限访问图形设备。**「图形栈全在 root 下」是常见的错误印象**——实际是一个受限的 system 进程，靠 SELinux 与组权限取得所需能力。这也意味着针对它的提权链需要先突破进程隔离，而不是「直接就是 root」。

### 1.3 合成决策：HWC 与 GPU

合成有两条路径：

- **硬件合成（HWC）**：显示控制器直接叠加多个图层，不消耗 GPU，最省电。
- **GPU 合成（client composition）**：SurfaceFlinger 用 GPU 把多个图层画到一张 buffer 上。

选哪条由 HWC 决定（"it asks the Hardware Composer (HWC) how it should perform composition"）。安全含义在于：**这条决策依赖应用提供的图层元数据**（格式、变换、混合模式、裁剪区、数据空间等）。元数据越复杂，校验面越大。这也是 gralloc 元数据能力（见 §2.2）之所以重要的原因——它把「格式之外的语义」也纳入了跨进程传递。

---

## 2. 缓冲区：gralloc、dma-buf 与 GPU 内存

### 2.1 从 gralloc1 到 gralloc4（一条常被写错的谱系）

官方对 gralloc 的定位是：

> "The Gralloc memory allocator performs buffer allocations and is implemented through two vendor-specific HIDL interfaces (see `hardware/interfaces/graphics/allocator/` and `hardware/interfaces/graphics/mapper/`)."

gralloc4 的新增能力：

> "Gralloc 4 adds support for getting and setting buffer metadata on a buffer."（文件首行为 `package android.hardware.graphics.mapper@4.0;`）

**必须纠正的谱系错误**：很多人写「gralloc1 → gralloc3 → gralloc4」是一条 HIDL 演进线。事实是：

- mapper HAL 目录只有 `2.0 / 2.1 / 3.0 / 4.0 / stable-c`（`stable-c` 为 AIDL 冻结版）；
- **gralloc1 不是 HIDL**，它仍是 legacy libhardware（`hardware/libhardware/include/hardware/gralloc1.h` 至今仍在树内）；
- HIDL 化的起点是 `@2.0`，不是 `@1.0`。

写成 HIDL 谱系会直接导致版本判断错误——例如「Android 9 用 gralloc1 HIDL」这类说法不成立。

### 2.2 Composer HAL 的版本形态

HWC2 对应 HIDL `android.hardware.graphics.composer@2.x`；HWC3 对应 AIDL 包 `android.hardware.graphics.composer3`（`Android.bp` 中 `name: "android.hardware.graphics.composer3"`，源集 `android/hardware/graphics/composer3/*.aidl`）。

实测目录是 `2.1 / 2.2 / 2.3 / 2.4 / aidl`——**没有 2.0**。所以「HWC2 就是 composer@2.0」是错的；HIDL 侧的起点是 2.1。

### 2.3 ION → dma-buf heaps

> "In Android 12, GKI 2.0 replaces the ION allocator with DMA-BUF heaps"
> "The `android12-5.10` branch of the Android Common Kernel disabled `CONFIG_ION` on **March 1, 2021**."
> "Each DMA-BUF heap is a character device that's present at `/dev/dma_heap/<heap_name>`."

堆的名字可以直接从代码里读到：`allocator->Alloc("system", size)` 与 `allocator->Alloc("system-uncached", size)`。分配走 ioctl：

```c
#define DMA_HEAP_IOCTL_ALLOC _IOWR(DMA_HEAP_IOC_MAGIC, 0x0, struct dma_heap_allocation_data)
```

**一处文档自身的笔误值得指出**：source.android.com 该页正文有一处把节点写成 `/dev/dma_heap/system_uncached`（下划线），而 `ueventd.rc` 与 `file_contexts` 里实际是 `system-uncached`（连字符）。自查脚本按文档写会永远匹配不到节点。

切换的安全含义：ION 是所有图形/媒体/相机共享的一个全局分配器，dma-buf heap 是按用途分立的多个设备节点。**分立带来更细的权限粒度**——不同堆可以有不同的 SELinux 标签、不同的大小约束、不同的缓存属性。这正是 §4.2 要讲的「权限按堆切分」的基础。

### 2.4 GPU 设备节点：一个被严重误传的事实

流传最广的说法是「GPU 驱动只能通过厂商图形 HAL 访问，应用碰不到」。SELinux 策略直接否证了它。AOSP `system/sepolicy/private/app.te` 中是：

```
allow { appdomain -isolated_app_all } gpu_device:chr_file rw_file_perms;
```

即：**普通应用域（除 isolated app 外）对 `gpu_device` 标签的字符设备拥有读写权限**。同一文件里 dma-heap 的授权是只读：

```
allow { appdomain -isolated_app_all } dmabuf_system_heap_device:chr_file r_file_perms;
```

这两个 allow 合起来刻画了真实模型：**应用自己就能打开 GPU 设备节点并下发 ioctl，图形 HAL 是「给框架用的抽象层」，不是「应用与 GPU 之间必须经过的闸门」**。这解释了为什么 GPU 驱动的内存破坏缺陷严重度普遍偏高——触发者不需要特殊权限。

节点实名也需要精度：

- Imagination 在 AOSP `file_contexts` 中的实名是 **`/dev/pvrsrvkm`**（`/dev/pvrsrvkm  u:object_r:gpu_device:s0`），不是 `/dev/pvr`。
- `/dev/dri/*` 在 `ueventd.rc` 中是 `0666 root graphics`。
- **`/dev/kgsl-3d0`（Qualcomm）与 `/dev/mali0`（Arm）在 AOSP 平台 `file_contexts` 中没有条目**——它们由 vendor sepolicy 打 `gpu_device` 标签。因此「AOSP 中这些路径被显式定义」属**未核实**，不能说成平台事实。

---

## 3. 隐私面：三个容易混淆的概念

### 3.1 `FLAG_SECURE`

官方的定义非常克制：

> "Window flag: treat the content of the window as secure, preventing it from appearing in screenshots or from being viewed on non-secure displays."
> `public static final int FLAG_SECURE = 0x00002000;`

同段还提到媒体投影路径："a window hosting a sensitive view will be marked as secure during media projection, preventing it from being viewed on non-secure displays and during screen share."

**必须明确的边界**：这在文档里只描述「它能拦住什么」，**没有任何一句承诺「它拦不住什么」**。所以「FLAG_SECURE 不防 root、不防被攻陷的系统」这句话是**推论**，不能作为引文使用。而且这个推论要说得更准：

- 它拦的是**采集路径**（截屏、录屏、非安全显示）；
- 它拦不住**已经能在 SurfaceFlinger 或应用进程内读帧的人**——因为帧本身仍存在于合成链路上；
- 它**不是权限**，没有对应的 `android.permission.*` 项。它的「权限面」在别处（见 §3.3）。

### 3.2 安全视频路径与 DRM 等级

Android 侧的 DRM 安全等级标识符是：

```
SECURITY_LEVEL_HW_SECURE_ALL = 5
SECURITY_LEVEL_HW_SECURE_DECODE = 4
```

官方注释分别是：

> `HW_SECURE_ALL`: "DRM key management, crypto operations, decoding of content and all handling of the media (compressed and uncompressed) is handled within a hardware backed trusted execution environment."
> `HW_SECURE_DECODE`: "…crypto operations and decoding of content are performed within a hardware backed trusted execution environment."

**必须纠正的说法**：Android 框架里**没有** `SECURITY_LEVEL_L1/L2/L3` 这样的常量。「L1/L2/L3」是 Widevine 侧的术语，在 source.android.com 的 DRM 页面中并未定义，也无法在 AOSP 中找到对应常量。把两者等同，是一个流传极广的错误。

ClearKey 在 AOSP 里以 HAL 服务名出现：`android.hardware.drm-service.clearkey`（`ICryptoFactory/clearkey`、`IDrmFactory/clearkey`）。

这条区分的安全价值：`HW_SECURE_ALL` 表示**连未压缩帧都不出 TEE**，这是「防录屏」能达到的最强形态；而仅 `HW_SECURE_DECODE` 只保证解码在 TEE 内，解码后的帧仍可能进入普通图形管道。评估「能不能录到受保护内容」时，看的是前者，不是「有没有 DRM」。

### 3.3 `SurfaceControl` 与权限门

`SurfaceControl` 是应用可用的类，但**高权方法都有权限门**。AOSP `AndroidManifest.xml` 中的声明是：

```xml
<permission android:name="android.permission.ACCESS_SURFACE_FLINGER"
            android:protectionLevel="signature" />
```

注释写明："Allows an application to use SurfaceFlinger's low level features. Not for use by third-party applications."。

`SurfaceControl` 中若干 `@hide` 方法逐个标注 "The caller must hold the ACCESS_SURFACE_FLINGER permission."（例如 `setTrustedOverlay`、`setDropInputMode`、`setCanOccludePresentation`）。这些方法的存在说明：**「谁能覆盖在谁上面」「谁能挡住输入」是受签名级权限保护的能力**，这也是 tapjacking 类问题在这一层被反复讨论的原因。

### 3.4 跨进程共享 buffer

`HardwareBuffer` 的官方描述是：

> "HardwareBuffer allows sharing buffers across different application processes. In particular, HardwareBuffers may be mappable to memory accessible to various hardware systems, such as the GPU, a sensor or context hub, or other auxiliary processing units."

即跨进程共享是**设计能力**（通过 Parcelable / fd 传递）。需要标注**未核实**的一点：官方文档中**没有**「恶意应用无法获取他人 buffer」这样的隔离声明。可表述的机制事实是：共享需要先拿到句柄，而句柄的传递受 Binder 权限与显式授权约束——不要把「拿不到」写成文档承诺。

---

## 4. 成因分析：图形栈的缺陷从哪来

### 4.1 长生命周期的跨进程对象

一块图形缓冲区的生命周期跨越：应用申请 → 写入 → 交队列 → SF 持有 → 合成使用 → 归还 → 应用复用。中间任何一步的时序错乱都会产生经典的内存安全问题：

- **已在用却被回收**：应用在 SF 仍在合成时提前释放/复用缓冲区（通过错误的 `release`/`cancel` 时序）。
- **重复归还**：同一 buffer 被归还两次，导致队列状态与所有权记录不一致。
- **同步对象误用**：fence 的归属与等待语义搞错，导致「读到上一帧」或「读到未写完的帧」。

这类问题的特征是**不产生越界写，但产生信息泄漏或渲染错误**，因此常被归类为「信息泄露」而非「内存破坏」。这也是图形栈 CVE 里「信息泄露」占比偏高的原因。

### 4.2 元数据校验面

gralloc4 的核心新增能力是「在 buffer 上存取元数据」。元数据包含格式、尺寸、用途、数据空间等语义信息，由生产者写入、消费者读取。这带来一个经典问题：**消费者信任生产者写的元数据**。

如果 SF 或 HWC 直接按元数据里的尺寸/格式去访问 buffer，而实际分配的 buffer 小于元数据声明，就会出现越界。防御要求是**双重校验**：既校验元数据自身合法性，也校验「元数据与真实分配是否一致」。

### 4.3 版本演进中的兼容层

图形 HAL 的版本非常碎：mapper `2.0/2.1/3.0/4.0/stable-c`、composer `2.1–2.4/aidl`。每一代都保留了对旧版的兼容路径，而兼容路径往往**校验最弱**——因为它要接受老设备的「不完整」数据。

评估时的实用口径：不要问「支持 gralloc 几」，要问「这台设备上 mapper 与 composer 分别实现了哪些版本，走的是哪条兼容分支」。

### 4.4 特权进程的输入面

SurfaceFlinger 是「唯一能修改显示内容的服务」，而它的输入来自所有应用。这意味着：

- 每个应用都能向它提交**图层状态变更**（位置、大小、透明度、变换、裁剪、层级）；
- 这些变更通过 Binder 到达，参数需要校验；
- 校验缺失的后果是「应用可以影响不属于自己的图层」——例如遮挡系统 UI、伪造对话框、改变受保护属性。

公开案例 `CVE-2021-1027` 就是这一类：官方描述为 "In setTransactionState of SurfaceFlinger, there is possible arbitrary code execution in a privileged process due to improper casting."。注意三点：入口是 `setTransactionState`（`Transaction` 的 Binder 入口）、根因是**类型转换不当**、后果是**特权进程内任意代码执行**。这是本篇最值得精读的一个案例，因为它把「用户可控参数 → 类型混淆 → 特权进程 RCE」这条链完整走通了。

同类的较新案例是 `CVE-2024-34743`："In setTransactionState of SurfaceFlinger.cpp, there is a possible way to perform tapjacking…"（公告列在 Framework 节），以及 `CVE-2024-40660`（"…possible way to change protected display attributes…"）。三者入口相同、后果不同——这正好说明**同一入口可以产生 RCE、也可以产生 UI 欺骗**，定级差异来自后果而非成因。

### 4.5 GPU 驱动：应用直连的内核面

§2.4 已经说明应用可以直接对 GPU 节点下 ioctl。因此 GPU 驱动的缺陷属于「应用可达的内核攻击面」，其成因与内核驱动类一致（ioctl 参数校验、对象生命周期、命令流校验），但**触发门槛极低**（无需权限）。

一个需要精确定位的案例：`CVE-2025-21479`，Qualcomm 2025 年 6 月公告的标题为 "Incorrect Authorization in Graphics"，描述是 "Memory corruption due to unauthorized command execution in GPU micronode while executing specific sequence of commands."。**注意它与「QSEE」无关**——早期一些资料把它归到可信执行环境，是错的；它属图形/GPU 侧。

### 4.6 显示属性与隐私开关

最后一类成因是「受保护属性可被修改」。受保护属性包括：安全窗口标记、显示的安全状态、输入通道属性。这些属性一旦可被应用改写，后果不是内存破坏，而是**隐私模型失效**（例如受保护内容出现在可被录制的显示上）。`CVE-2024-40660` 属这一类。

---

## 5. 公开案例台账（逐条核实归属）

图形栈是 CVE 张冠李戴最严重的领域之一。逐条核对：

| 编号 | 实际组件（原文要点） | 与图形栈的关系 |
|---|---|---|
| CVE-2021-1027 | "In setTransactionState of SurfaceFlinger … arbitrary code execution in a privileged process due to improper casting"（Pixel 公告 2021-12-01） | **直接相关**：SurfaceFlinger |
| CVE-2024-34743 | "In setTransactionState of SurfaceFlinger.cpp … perform tapjacking"（2024-08-01，Framework 节） | **直接相关**：SurfaceFlinger |
| CVE-2024-40660 | "…possible way to change protected display attributes…"（2024-11-01） | **直接相关**：显示属性 |
| CVE-2025-21479 | "Incorrect Authorization in Graphics"；"Memory corruption due to unauthorized command execution in GPU micronode…"（Qualcomm 2025-06） | **直接相关**：GPU |
| CVE-2019-2181 | "In binder_transaction of binder.c in the Android kernel…" | **无关**：内核 Binder |
| CVE-2020-0108 | "In postNotification of ServiceRecord.java…" | **无关**：AMS |
| CVE-2021-0473 | "In rw_t3t_process_error of rw_t3t.cc… remote code execution over NFC" | **无关**：NFC |
| CVE-2021-39623 | "In doRead of SimpleDecodingSource.cpp…" | **无关**：媒体解码 |
| CVE-2022-20004 | "In checkSlicePermission of SliceManagerService.java…" | **无关**：Slices |
| CVE-2023-21282 | "In TRANSPOSER_SETTINGS of lpp_tran.h…" | **无关**：媒体解码（AAC） |

**六个被广泛误传的编号全部与图形无关。** 这提示一条方法论：图形栈的缺陷名往往含有 `Surface`、`Composer`、`gralloc`、`GPU` 这类关键词，而误传的编号来自「听起来相关」的邻近领域（Binder、媒体、NFC）。**核对时只认公告里的组件字段，不认 CVE 编号的直觉。**

---

## 6. 防护技术

### 6.1 设计层

| 措施 | 做法 | 挡住什么 |
|---|---|---|
| 敏感界面加 `FLAG_SECURE` | 密码、验证码、支付确认页 | 截屏/录屏采集 |
| 受保护内容用 `HW_SECURE_ALL` 路径 | 要求解码与未压缩帧都不出 TEE | 图形管道内取帧 |
| 不在应用层做「安全覆盖」 | 用系统级窗口类型与签名权限，而不是自绘遮挡 | 覆盖攻击与 tapjacking |
| 元数据双重校验 | 同时校验声明值与实际分配 | 越界与越权访问 |
| GPU HAL 收窄 | 优先通过厂商图形 HAL 暴露能力，减少应用直连 | 驱动缺陷的触发面（受限） |

### 6.2 实现层

1. **`FLAG_SECURE` 要在窗口创建时设置**，不要指望运行中切换能立刻生效于已捕获的会话。
2. **不要用「全屏透明覆盖层」做防截屏**——那是 UI 欺骗的同构手法，且无法阻止采集。
3. **不要把 `HardwareBuffer` 句柄外传**：句柄一旦跨进程传递，接收方即可映射；传递前必须有明确的业务必要性与接收方校验。
4. **合成相关元数据从服务端再校验**：应用侧不要假设「SF 会替我检查」。
5. **对 `SurfaceControl` 的 `@hide` 方法不依赖反射调用**：它们受签名权限保护，反射调用在 SELinux 与权限检查处会失败，且随版本变化。

### 6.3 验证层

- **权限基线**：`ACCESS_SURFACE_FLINGER` 的持有者应只有系统组件；出现第三方持有即为异常。
- **节点基线**：`ls -lZ /dev/kgsl-3d0 /dev/mali0 /dev/pvrsrvkm /dev/dri/*` 记录属主、权限、SELinux 标签，跨版本比对。
- **合成路径**：`dumpsys SurfaceFlinger` 中的 layer 列表与合成类型（HWC/GPU）可作为行为基线。
- **隐私验证**：对受保护界面做实际截屏尝试，确认得到的是空白/黑屏而非内容；这比读文档可靠。

---

## 7. 自查与观测清单

```bash
# 1. 合成服务与进程
adb shell ps -A | grep -i surfaceflinger
adb shell dumpsys SurfaceFlinger --list | head -40

# 2. 显示与安全状态
adb shell dumpsys display | head -60
adb shell dumpsys SurfaceFlinger | grep -iE "secure|composition" | head -20

# 3. GPU 与图形设备节点（含 SELinux 标签）
adb shell ls -lZ /dev/kgsl-3d0 /dev/mali0 /dev/pvrsrvkm 2>/dev/null
adb shell ls -lZ /dev/dri 2>/dev/null

# 4. dma-buf 堆
adb shell ls -l /dev/dma_heap 2>/dev/null

# 5. 图形 HAL 版本
adb shell dumpsys -l | grep -iE "graphics|composer|allocator|mapper"
adb shell service list | grep -iE "graphics|composer"

# 6. 权限持有者（需可读系统包信息）
adb shell dumpsys package | grep -B2 -A2 ACCESS_SURFACE_FLINGER | head -20
```

判读要点：

- **第 3 条**在部分机型上因权限不足返回空，这是正常的；此时用 `getprop` 与 `dumpsys` 侧证。
- **第 4 条**若为空，说明是较老内核（ION 时代），此时按 ION 模型评估，不要按 dma-heap 推断。
- **第 5 条**能把 §2.2 的版本碎片化直接暴露出来——`composer3` 出现即表示 AIDL 化已完成。

---

## 8. 常见误传纠正

1. **「gralloc1 是 HIDL」**——错。HIDL mapper 从 `@2.0` 起；gralloc1 是 libhardware（`gralloc1.h` 仍在树内）。「gralloc1→3→4」的 HIDL 谱系不成立。
2. **「HWC2 = composer@2.0」**——无 `@2.0` 目录，HIDL 版本是 `@2.1`–`@2.4`；HWC3 包名为 `android.hardware.graphics.composer3`。
3. **「GPU 驱动只能经厂商图形 HAL 访问」**——错。`allow { appdomain -isolated_app_all } gpu_device:chr_file rw_file_perms;` 表明普通应用直接持有该节点。
4. **「Imagination 节点是 `/dev/pvr`」**——AOSP 实名 `/dev/pvrsrvkm`。
5. **「dma-heap 节点是 `system_uncached`（下划线）」**——实际是 `system-uncached`（连字符）；source.android.com 该页正文自身有此笔误。
6. **「Widevine L1 对应 `SECURITY_LEVEL_L1`」**——无此常量。真实值是 `SECURITY_LEVEL_HW_SECURE_ALL/DECODE`；L1/L2/L3 是 Widevine 侧术语，未在 Android 官方文档中定义。
7. **「`FLAG_SECURE` 是一个权限」**——不是权限，没有对应 `android.permission.*`；`0x00002000` 是窗口标志位。相关的权限面是 `ACCESS_SURFACE_FLINGER`（signature）。
8. **「`FLAG_SECURE` 官方说明了它防不住 root」**——官方文档只描述它能拦什么，没有反向承诺；「防不住」是推论，不能当引文。
9. **「SurfaceFlinger 跑在 `system_server` 里」**——是独立进程（init 服务 `surfaceflinger`，`user system`）。
10. **「SurfaceFlinger 以 root 运行」**——`user system`，靠 `group graphics` 与 SELinux 取得能力。
11. **「应用拿不到 GPU 设备节点」**——错，见第 3 条。
12. **CVE 张冠李戴**：CVE-2019-2181 是内核 Binder、CVE-2020-0108 是 AMS、CVE-2021-0473 是 NFC、CVE-2021-39623 是媒体解码、CVE-2022-20004 是 SliceManagerService、CVE-2023-21282 是 AAC 解码——均非图形栈。

---

## 未核实与失效来源

- **A. `/dev/kgsl-3d0` 与 `/dev/mali0` 的平台定义。** AOSP 平台 `file_contexts` 中无条目，由 vendor sepolicy 打标签；「AOSP 显式定义这些路径」未核实。
- **B. Widevine L1/L2/L3 的官方定义与「L1 必须 TEE + 安全解码器」的规定。** source.android.com 的 DRM 页不含 L1/L2/L3；相关表述未核实。
- **C. 「恶意应用无法获取他人 buffer」的隔离声明。** 官方文档只描述 `HardwareBuffer` 可跨进程共享，未见反向隔离承诺。
- **D. `FLAG_SECURE` 的反向承诺。** 文档只列它能拦什么；「不防 root/被攻陷系统」为推论。
- **E. `SurfaceFlinger` 内部合成决策的细节分支。** 正文只使用「询问 HWC 如何合成」这一官方表述，未逐版本展开。
- **F. `/dev/dri/*` 的实际可用性。** `ueventd.rc` 中为 `0666 root graphics`，但应用能否实际打开还受 SELinux 约束，未逐机型核验。

---

## 参考

- BufferQueue 与 Gralloc：[source.android.com/docs/core/graphics/arch-bq-gralloc](https://source.android.com/docs/core/graphics/arch-bq-gralloc)
- SurfaceFlinger 与 HWC：[source.android.com/docs/core/graphics/arch-sf-hwc](https://source.android.com/docs/core/graphics/arch-sf-hwc)
- 图形概览：[source.android.com/docs/core/graphics](https://source.android.com/docs/core/graphics)
- ION → DMA-BUF heaps：[source.android.com/docs/core/architecture/kernel/dma-buf-heaps](https://source.android.com/docs/core/architecture/kernel/dma-buf-heaps)
- `dma-heap.h` uapi：[git.kernel.org/…/include/uapi/linux/dma-heap.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/plain/include/uapi/linux/dma-heap.h)
- `app.te`（gpu_device 授权）：[android.googlesource.com/platform/system/sepolicy/+/refs/heads/main/private/app.te](https://android.googlesource.com/platform/system/sepolicy/+/refs/heads/main/private/app.te)
- `file_contexts`：[android.googlesource.com/platform/system/sepolicy/+/refs/heads/main/private/file_contexts](https://android.googlesource.com/platform/system/sepolicy/+/refs/heads/main/private/file_contexts)
- `WindowManager.java`（`FLAG_SECURE`）：[android.googlesource.com/platform/frameworks/base/+/refs/heads/main/core/java/android/view/WindowManager.java](https://android.googlesource.com/platform/frameworks/base/+/refs/heads/main/core/java/android/view/WindowManager.java)
- `MediaDrm.java`（安全等级常量）：[android.googlesource.com/platform/frameworks/base/+/refs/heads/main/media/java/android/media/MediaDrm.java](https://android.googlesource.com/platform/frameworks/base/+/refs/heads/main/media/java/android/media/MediaDrm.java)
- `SurfaceControl.java`（权限门）：[android.googlesource.com/platform/frameworks/base/+/refs/heads/main/core/java/android/view/SurfaceControl.java](https://android.googlesource.com/platform/frameworks/base/+/refs/heads/main/core/java/android/view/SurfaceControl.java)
- `HardwareBuffer.java`：[android.googlesource.com/platform/frameworks/base/+/refs/heads/main/core/java/android/hardware/HardwareBuffer.java](https://android.googlesource.com/platform/frameworks/base/+/refs/heads/main/core/java/android/hardware/HardwareBuffer.java)
- Qualcomm 2025 年 6 月公告：[docs.qualcomm.com/product/publicresources/securitybulletin/june-2025-bulletin.html](https://docs.qualcomm.com/product/publicresources/securitybulletin/june-2025-bulletin.html)
- 公告：[source.android.com/docs/security/bulletin](https://source.android.com/docs/security/bulletin)

---

## 联系方式

如果你在企业里做终端隐私合规或图形栈安全评审，需要就「受保护内容是否真正受保护」「GPU 节点暴露面」「合成路径与安全窗口核查」做一次针对现网机型的验证，可以通过 Telegram 联系：<https://t.me/one00190>

---

*本手册仅用于防御与教育目的，讨论的均为公开且已被修复问题的机制与设计缺陷，不包含任何可直接利用的代码。文中标注「未核实」的内容不应作为事实引用；发现错误欢迎指出。*
