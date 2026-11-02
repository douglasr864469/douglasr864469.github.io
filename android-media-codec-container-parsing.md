---
layout: page
title: "Android 媒体解码与容器解析攻击面手册"
description: "把「打开一个媒体文件」拆成取数／解复用／解码／输出四段：Android 7.0 把单体 `mediaserver` 拆成 `mediaextractor`（UID 1040）、`media.codec`、`audioserver`、`cameraserver` 等独立进程，Android 10 又把软解..."
date: 2026-11-03
permalink: /android-media-codec-container-parsing/
---

> 手机每天要打开别人给的图片、语音、视频和直播流。这些字节在到达屏幕或扬声器之前，必须先经过两段完全不同的代码：一段把「容器」拆成音视频轨道（解复用），另一段把压缩码流还原成像素或采样（解码）。这两段代码都要逐字节处理不可信输入，而且历史上贡献了 Android 用户态最集中的一批严重漏洞。这份手册沿着「数据从哪来 → 谁在拆容器 → 谁在解码 → 进程和沙箱怎么划 → 出问题时落在哪」这条链，逐段说明当前实现的边界、公开缺陷的落点，以及防御方真正能检查的东西。只讨论公开且已修复的问题，不含可利用代码。

## 0. 使用方式与边界

这份手册面向三类读者：

- **应用开发者**：需要知道 `MediaPlayer`、`MediaExtractor`、`MediaCodec`、`MediaMetadataRetriever`、`MediaParser` 各自把工作放在哪个进程，哪些 API 会把不可信字节拉回自己的进程。
- **系统与平台工程师**：需要知道媒体进程怎么拆、拆完之后各自还剩哪些权限，以及为什么「软件解码器」和「硬件解码器」的补丁节奏完全不同。
- **安全评估者**：需要在拿到一份「媒体解析漏洞」公告时，判断它影响的是容器解析、软件解码、厂商解码器还是应用进程内的解析，而不是把所有媒体 CVE 都当成同一类。

本文的三条自我约束：

1. 只引用公开且已修复的问题；CVE 的组件归属以 Android Security Bulletin 的章节与行文为准，公告没有写明的代码库会单独标注来源。
2. 不写利用步骤、不写 PoC、不写可复现的构造参数；成因只描述到「哪一类长度或索引没有校验」。
3. 查不到一手来源的说法一律进「未核实与失效来源」，不写进正文。

与已发布手册的分工：

- 《摄像头、麦克风与屏幕捕获面手册》讲的是「谁能采集」，这一篇讲「采集之后或外部传入的媒体字节怎么被解析」。
- 《音频栈与媒体会话手册》讲的是路由、焦点与 `MediaSession`，不涉及码流解析。
- 《图形与显示栈手册》讲的是合成与显示通路；本文只讲到解码输出缓冲区交出去为止。
- 《内存安全与分配器手册》讲的是通用内存缺陷类型；本文只讲媒体路径上这类缺陷为什么反复出现。
- 《SELinux 与 seccomp 沙箱手册》讲机制本身；本文只给媒体域的具体权限矩阵。

## 1. 一条媒体文件的四段旅程

把「播放一个视频」拆开，至少有四个阶段，每个阶段的输入都来自外部，但代码、进程和失败后果不同：

| 阶段 | 典型实现 | 默认所在进程 | 输入 | 主要风险 |
|------|----------|--------------|------|----------|
| 取数 | `IDataSource` / `MediaDataSource` / `ContentResolver` | 应用进程或 `media.extractor` | 文件、fd、网络流、应用回调 | 数据源边界与长度谎报 |
| 解复用（拆容器） | `MediaExtractor` / `MediaParser` | `media.extractor`（默认）或应用进程 | MP4、Matroska、Ogg、TS 等容器 | 长度、索引、嵌套结构解析 |
| 解码 | `MediaCodec` → Codec2 / OMX | `media.swcodec` 或厂商 HAL 进程 | 压缩码流、CSD、参数集 | 码流语法、参考帧、缓冲区边界 |
| 输出 | `Surface` / `AudioTrack` / `ImageReader` | 应用进程、`surfaceflinger`、`audioserver` | 图形缓冲、PCM | 句柄与生命周期，通常不是解析问题 |

两个容易混淆的概念：

- **容器不是编码**。MP4 只是一个盒子，里面可以装 H.264、H.265、AV1、AAC、Opus。容器解析器出错，和具体编码无关；解码器出错，也和容器格式无关。安全公告里两者都可能被写成「Media Framework」，所以读公告时必须看它引用的代码库。
- **解析不等于解码**。有些格式（例如未压缩 PCM、部分 MJPEG）几乎不需要解码，但容器解析仍然存在；反过来，裸码流可以没有容器，但解码器仍要处理不可信语法。

这条链上还有一个常被忽略的事实：**应用可以选择把哪一段拉回自己的进程**。默认路径把解复用放在 `media.extractor`、把软件解码放在 `media.swcodec`；一旦应用使用自定义 `MediaDataSource`，或者改用 `MediaParser`，解析就会在应用自己的沙箱里发生。这不是缺陷，而是设计取舍，后文会分别说明。

## 2. 进程模型：从单体 mediaserver 到六个服务

### 2.1 Android 7.0 的拆分

在 Android 7.0 之前，媒体播放、录音、相机和 DRM 大量集中在 `mediaserver` 一个进程里。这个进程以 `media` 身份运行，能访问图形设备、视频设备和应用数据；一旦其中任何一个解析器被攻破，攻击者拿到的就是这一整套权限。

Android 7.0 做了拆分。官方文档的原话是：

> "Android 7.0 splits the `mediaserver` process into several new processes"

文档在「Architectural changes」下列出的服务是：MediaServer、MediaCodecService、MediaDrmServer、AudioServer、CameraServer、ExtractorService。拆分的目的不是「让漏洞消失」，而是**把不同信任级别的解析代码放进不同的权限集合**：

- 容器解析器面对最多变的输入，因此放进权限最小的 ExtractorService；
- 编解码器需要图形内存，但不需要应用数据，因此单独成进程；
- 音频与相机各自独立，避免媒体解析影响实时通路。

### 2.2 进程、用户与 UID 对照

下表只列能在一手源码中核实的身份。UID 数值取自 AOSP 的 `android_filesystem_config.h`：

| 进程 / 服务 | 运行用户 | UID | 来源 |
|-------------|----------|-----|------|
| `mediaserver` | `media` | 1013 | `AID_MEDIA 1013 /* mediaserver process */` |
| `mediaextractor` | `mediaex` | 1040 | `AID_MEDIA_EX 1040 /* mediaextractor process */` |
| `audioserver` | `audioserver` | 1041 | `AID_AUDIOSERVER 1041 /* audioserver process */` |
| `media.swcodec` / `vendor.media.omx` | `mediacodec` | 1046 | `AID_MEDIA_CODEC 1046 /* mediacodec process */` |
| `cameraserver` | `cameraserver` | 1047 | `AID_CAMERASERVER 1047 /* cameraserver process */` |

注意两点：

1. 源码里**没有** `AID_MEDIACODEC` 这个拼写，实名是 `AID_MEDIA_CODEC`；`AID_MEDIA_EX` 也常被漏写。
2. 这些数值是 AOSP 的约定，厂商可以增加自己的服务，但不应改动已有 AID 的语义。看到 `ps` 里某个媒体进程的 UID 不是上表之一时，先怀疑它是厂商扩展，而不是直接判定异常。

### 2.3 Android 10 的第二次拆分：software codec 独立

Android 7.0 把编解码器放进 `media.codec`，但软件解码器和硬件解码器仍共享同一个进程。Android 10 又把软件编解码器移了出去。官方博客的原话：

> "In Android Q, we moved software codecs out of the main mediacodec service into a constrained sandbox."
>
> "software codecs are extracted from the media.codec process, and moved back to system side."

新的服务定义在 `apex/mediaswcodec.rc` 中：

```
service media.swcodec /apex/com.android.media.swcodec/bin/mediaswcodec
    user mediacodec
    group camera drmrpc mediadrm
    ioprio rt 4
```

这条 rc 里有三个值得记住的细节：

- 可执行文件位于 **APEX**（`com.android.media.swcodec`）而不是 `/system/bin`，意味着软件编解码器可以随模块更新，而不必等待完整系统升级。
- 运行身份是专用的 `mediacodec` 用户，不是 `media`；
- 进程带实时 I/O 优先级（`ioprio rt 4`），说明它仍承担对时延敏感的软解任务。

与此同时，`media.codec` 并没有消失。AOSP 当前源码里，`services/mediacodec/main_codecservice.cpp` 仍会把进程名改为 `media.codec`，并向 `IOmxStore` 注册 OMX 服务；与之配套的 rc 定义的是：

```
service vendor.media.omx /vendor/bin/hw/android.hardware.media.omx@1.0-service
    user mediacodec
    group camera drmrpc mediadrm
```

也就是说：**软件解码在 `media.swcodec`，OMX 硬件解码服务仍以 `media.codec` / `vendor.media.omx` 形态存在**。把「Android 10 之后没有 media.codec 了」当成事实是错的。

### 2.4 一次播放的进程往返

以 `MediaPlayer` 播放本地 MP4 为例，默认路径大致是：

1. 应用把 URI 或 fd 交给 `MediaPlayer`；
2. `MediaPlayerService`（在 `mediaserver`）建立会话；
3. 解复用交给 `media.extractor`：应用或 `mediaserver` 向它要一个 `IMediaExtractor`，再按轨道拿 `IMediaSource`；
4. 解码器由 `MediaCodec` 按名称选择：`c2.*` 走 Codec2，`omx.*` 走 OMX；
5. 软件解码在 `media.swcodec` 完成，硬件解码在厂商 HAL 进程完成；
6. 解码输出以图形缓冲或 PCM 交给 `SurfaceFlinger` / `AudioFlinger`。

每一次跨进程都要过 Binder 和共享内存。**媒体框架的安全设计，本质上就是把这些跨进程边界画在「解析代码」和「高权限资源」之间**。

## 3. 沙箱：SELinux、seccomp 与「最不特权」的真实含义

### 3.1 三个域的权限矩阵

只看进程名还不够，真正决定「被攻破之后能做什么」的是 SELinux 域。以下规则取自 AOSP `system/sepolicy/private/` 下的 `.te` 文件（master 分支）：

| 能力 | `mediaextractor` | `mediaswcodec` | `mediaserver` |
|------|------------------|----------------|---------------|
| 读应用数据文件 | `app_data_file` / `privapp_data_file` 的 `getattr read` | 无 | `append getattr ioctl lock map read write` |
| 打开应用数据文件 | **禁止**（neverallow `data_file_type:file open`） | 无 | 允许（受 DAC 约束） |
| 读外部存储 | `sdcard_type` / `fuse` 的 `getattr read` | 无 | — |
| 图形设备 | 无 | `gpu_device` 读写 + `dmabuf_system_heap_device` 读 | `gpu_device` 读写 |
| 视频设备 | 无 | **无** | `video_device` 读写 |
| TCP / UDP 套接字 | neverallow | neverallow | 允许（受策略约束） |
| 执行未标记文件 | neverallow | — | — |

这张表解释了两件常被误解的事：

1. **`mediaextractor` 能读应用数据，但不能自己打开应用数据文件。** 它拿到的是别人已经打开的 fd；`read` 权限允许它从 fd 读，`open` 被 neverallow 挡住。所以「解析器能读 /data/data」这种说法只在「有人把 fd 递进来」的前提下成立。
2. **`mediaswcodec` 看不到应用数据和视频设备。** 它只有 GPU 与 dmabuf 堆的访问权，外加 Codec2/OMX/allocator 的 HAL 客户端身份。这正是「软件解码器更收紧」的落地方式，而不是一句宣传语。

### 3.2 「最不特权」的官方定义

`source.android.com` 的框架加固页面对 ExtractorService 有一句明确判断：

> "The extractor service is the least privileged of all the services"

同页还说明它「不能直接读文件描述符，而是通过 Binder 接口向 `mediaserver` 要数据」。这与当前 AIDL 接口并不矛盾：现代接口既支持传入 `IDataSource`（由客户端提供数据），也支持 `makeIDataSource(in FileDescriptor fd, long offset, long length)` 直接把 fd 交给解析进程。区别在于**谁来承担取数**：

- 传 fd：解析进程自己读，SELinux 的 `read` 权限生效，`open` 仍被 neverallow 挡住；
- 传回调式数据源：数据从客户端进程按需搬过来，解析进程只处理内存里的字节。

### 3.3 seccomp 与 minijail

`mediaextractor` 与 `mediaswcodec` 都由 minijail 启动并加载 seccomp 策略：解析服务的入口在 `services/mediaextractor/main_extractorservice.cpp`，软解服务在 `services/mediacodec/main_swcodecservice.cpp`，策略文件随各自 APEX 分发。效果是：即使攻击者在解析器里拿到了代码执行，可用的系统调用集合仍被限制在「读数据、分配内存、发 Binder 回调」这一小组。

### 3.4 沙箱的边界在哪里

沙箱不是万能的，它有三个明确的边界：

- **同一进程内的所有解析器共享命运。** `media.extractor` 里同时装着 MP4、Matroska、Ogg、TS 等解析器；一个解析器的内存破坏可以影响同进程的其他解析器状态。
- **fd 是能力，不是数据。** 一旦 fd 被递进解析进程，解析器就能读该文件；沙箱阻止的是「自己去开新文件」，不是「读已经打开的文件」。
- **输出缓冲区要交给别人。** 解码器必须把图形缓冲或 PCM 交出去，这条通路依赖 gralloc 与 Binder 句柄，生命周期错误会以另一种形式出现。

## 4. 容器解析面：MediaExtractor 与 media.extractor

### 4.1 默认是远程解析

`MediaExtractor` 的 Java 层看起来是一个本地对象，但默认实现会先找 `media.extractor` 服务。AOSP 的 `MediaExtractorFactory.cpp` 里有一个属性开关：

```cpp
if (!property_get_bool("media.stagefright.extractremote", true)) {
    // local extractor
} else {
    // remote extractor
    sp<IBinder> binder = defaultServiceManager()->getService(String16("media.extractor"));
}
```

默认值是 `true`，所以**在正常设备上，解复用发生在 `media.extractor` 进程**。把该属性改成 `false` 会让解析回到调用进程，这通常只用于调试或特殊产品配置；它不是应用可以随便切换的开关。

### 4.2 接口与数据通路

远程解析的核心接口在 `IMediaExtractorService.aidl`：

```aidl
IMediaExtractor makeExtractor(IDataSource source, @nullable @utf8InCpp String mime);
IDataSource makeIDataSource(in FileDescriptor fd, long offset, long length);
```

调用关系是：

1. 客户端准备 `IDataSource`（fd 或回调）；
2. 向服务要 `IMediaExtractor`；
3. 用 `IMediaExtractor` 枚举轨道，拿到每条轨道的 `IMediaSource`；
4. 从 `IMediaSource` 读样本（sample）交给解码器。

`IMediaSource` 返回的是「压缩样本 + 时间戳 + 格式信息」，不是解码后的帧。**解析器只负责切分，不理解编码语义**；这也是为什么容器解析漏洞和解码器漏洞的修复团队、补丁节奏、影响面都不同。

### 4.3 应用自定义数据源：把回调拉回应用进程

`MediaDataSource` 是公开 API，应用可以实现 `readAt(position, buffer, offset, size)`，让框架按需向应用要字节。这时数据通路的形状变成：

- 解析仍在 `media.extractor`；
- 但每次读取都会回调到应用进程（通过 Binder 或 JNI 桥）；
- 应用可以选择返回短读、谎报长度、在读取过程中改变内容。

这带来两个安全含义：

- **解析器仍要防御「数据源不老实」**，不能假设 `readAt` 一定返回请求的字节数；
- **应用自己承担了数据源的正确性**，如果应用把不可信远端数据直接喂给解析器，解析器崩溃会表现为媒体服务重启，而不是应用崩溃。

### 4.4 MediaMetadataRetriever 的额外一跳

`MediaMetadataRetriever` 用于取时长、封面、帧缩略图。它的默认路径是：

1. 应用调用 `setDataSource`；
2. 请求进入 `mediaserver` 的 `MediaPlayerService`；
3. `StagefrightMetadataRetriever` 在 `mediaserver` 内创建 `MediaExtractor`；
4. 该 `MediaExtractor` 再连到 `media.extractor` 完成实际解析。

所以「只取一张封面」并不比「播放视频」少经过解析器；它只是少了渲染与音频通路。评估一条「图片/视频预览」入口时，不能因为它不播放就认为它不碰解析代码。

### 4.5 MediaParser：API 30 的另一种选择

`MediaParser` 是比 `MediaExtractor` 更晚的公开 API。开发者文档标注：

> "Added in API level 30"

也就是 Android 11，而不是常被误传的 Android 12。它**没有取代** `MediaExtractor`，两者并存；`MediaParser` 提供的是「应用自己驱动解析」的模型：应用实现 `InputReader` 提供字节、实现 `OutputConsumer` 接收样本与元数据。

关键在于进程模型：**`MediaParser` 默认在应用进程内完成解析**。官方参考文档没有明说这一点（它只描述输入/输出接口），但 AOSP 的实现直接在应用进程里构造各类 extractor（例如 Matroska 分支返回 `MatroskaExtractor`），native 侧只用于指标上报。这个取舍是清晰的：

- 好处：解析不再需要跨进程，延迟与拷贝更低，应用可以精细控制数据源；
- 代价：解析器的内存破坏发生在应用沙箱内，而不是被限制在 `media.extractor` 里。对应用而言，这通常仍比「解析器在系统进程里被攻破」更可控；但如果应用本身持有高价值权限（例如能访问大量用户媒体），攻击面就从系统进程转移到了应用。

### 4.6 格式覆盖面与补丁节奏

容器解析器支持的格式越多，攻击面越大。常见容器包括 MP4/3GP（ISO BMFF）、Matroska/WebM、Ogg、MPEG-TS/PS、FLV、WAV、FLAC、MP3、AMR 等。补丁节奏取决于代码位置：

- 在 AOSP `frameworks/av` 或 `external/` 下的解析器，随每月安全公告与 Mainline 模块更新；
- 厂商自加的解析器或私有格式支持，补丁由厂商决定，可能显著滞后；
- 应用自带的解析器（例如某些播放器内置 demuxer）完全由应用自己负责，系统补丁覆盖不到。

## 5. 解码面：Codec2、OMX 与厂商进程

### 5.1 组件选择：前缀决定实现

`MediaCodec.createByCodecName()` 接受组件名，AOSP 按前缀分派：

- `c2.*` → `CCodec`，走 Codec2（`android.hardware.media.c2` HAL）；
- `omx.*` → `ACodec`，走 OMX（`IOmx` / `IOmxStore`）。

这条规则来自 `frameworks/av/media/libstagefright/MediaCodec.cpp` 中的名称判断，而不是某个「全局默认」配置。因此：

- 说「Android 10 起 Codec2 完全取代 OMX」**没有官方依据**；AOSP 仍保留 ACodec 与 `vendor.media.omx`，厂商可以继续提供 OMX 组件。
- 说「软解一定是 Codec2」也不准确：软解组件通常是 `c2.android.*` / `libcodec2_soft_*`，但历史上也存在 OMX 软件组件。真正决定沙箱位置的是「软件还是硬件」，不是「Codec2 还是 OMX」。

### 5.2 软解、硬解与安全解码器

三类解码器的落点不同：

| 类型 | 典型组件 | 所在进程 | 更新方式 |
|------|----------|----------|----------|
| 软件解码 | `c2.android.*` / `libcodec2_soft_*` | `media.swcodec` | 随 Media Codecs APEX 更新 |
| 硬件解码 | 厂商 `c2.*` / `omx.*` | 厂商 HAL 进程 | 随 vendor 分区 / 厂商 OTA |
| 安全解码 | 受 DRM 保护的解码器 | DRM / 安全世界相关进程 | 随 vendor 与 DRM 模块 |

Android 7.0 的框架加固文档在描述当时的拆分的时说：「Non-secure decoders and software encoders live in the codec process」，而安全解码器与硬件编码器仍留在 `mediaserver`。这是 7.0 的状态，不应直接套到今天的设备上；现代实现中安全解码通常与 DRM HAL 绑定，具体进程名由厂商决定。**本文只把「软件解码在 `media.swcodec`」当作稳定事实，其余按版本区分。**

### 5.3 缓冲与图形内存

解码器的输入是压缩样本，输出是图形缓冲（视频）或 PCM（音频）。视频输出的关键对象是：

- Codec2 侧的 `C2Buffer` / `C2GraphicBlock`；
- gralloc 分配的 `GraphicBuffer` / native handle；
- 通过 `IAllocator` / `IGraphicBufferAllocator` 跨进程传递。

这也是 `mediaswcodec.te` 里出现 `hal_allocator`、`hal_graphics_allocator` 与 `dmabuf_system_heap_device` 的原因：软解进程必须能分配和映射图形内存，但不需要视频设备节点。**权限矩阵是跟着数据通路设计的，不是跟着「媒体」这个词设计的。**

### 5.4 MediaCodecList 能告诉你什么

应用可以枚举设备上的编解码器，并读取它们的安全相关属性：

```java
for (MediaCodecInfo info : new MediaCodecList(MediaCodecList.ALL_CODECS).getCodecInfos()) {
    Log.i("codec", info.getName()
            + " sw=" + info.isSoftwareOnly()
            + " hw=" + info.isHardwareAccelerated()
            + " vendor=" + info.isVendor());
}
```

三个方法的语义（均为 API 29 加入）：

- `isSoftwareOnly()`：是否纯软件实现。官方 javadoc 明确写道："Software-only codecs are more secure as they run in a tighter security sandbox. On the other hand, software-only codecs do not provide any performance guarantees."
- `isHardwareAccelerated()`：是否硬件加速。javadoc 同时提醒该属性由厂商提供，"it cannot be tested for correctness"——也就是说，它不能作为安全判断的硬依据。
- `isVendor()`：组件由平台提供还是厂商提供。厂商组件的补丁节奏通常跟随厂商 OTA。

API 36 起还增加了 `getSecurityModel()`，返回两种模型：

- `SECURITY_MODEL_SANDBOXED`（0）：编解码器运行在沙箱进程中，恶意内容的影响被限制在沙箱内；
- `SECURITY_MODEL_MEMORY_SAFE`（1）：编解码器不在沙箱进程中，但用内存安全语言（javadoc 举例为 Rust）实现。

这是官方第一次把「沙箱隔离」与「内存安全实现」并列成可查询的安全模型。它的实际覆盖取决于厂商与组件，不能假设所有设备都返回有意义的值。

### 5.5 为什么「软解更安全」但不等于「软解没漏洞」

官方的判断有数据支撑。2019 年 Android 开发者博客在解释这次拆分时给了三个比例：

> "approximately 80% of the critical/high severity vulnerabilities in media components occurred in software codecs"

> "Incorrect or missing memory bounds checking on arrays account for about 34% of Android's userspace vulnerabilities."

> "Use-after-free (UAF), integer overflows, and out of bounds (OOB) reads/writes comprise 90% of vulnerabilities"

也就是说，软件编解码器既是漏洞最集中的地方，也是最容易被沙箱收住的地方——因为它是纯用户态、只处理字节、不需要直接碰硬件。把它放进独立进程，等于承认「这里一定会出错」，但把出错的后果限制在「读不到应用数据、没有视频设备、没有网络」的范围内。

反过来，硬件解码器的漏洞不会因为「在厂商进程里」就消失；它只是把补丁责任转移给了厂商。评估一台设备时，**「软解更安全」描述的是沙箱，不是「没有缺陷」**。

### 5.6 编码器也解析输入吗

编码器的输入是原始帧（YUV、PCM），不是压缩码流，所以它不面对「恶意码流语法」这一类问题。但编码器仍会读取应用提供的 `MediaFormat` 参数（分辨率、帧率、码率、颜色格式、CSD），并可能根据这些参数分配缓冲区。异常参数导致的整数溢出或超大分配，仍是编码路径上的现实问题。安全公告中「Media Framework」的条目并不全是解码器，读的时候要看具体代码库。

## 6. 应用层入口对照与权限

### 6.1 入口对照表

| API | 解析位置 | 解码位置 | 典型用途 |
|-----|----------|----------|----------|
| `MediaPlayer` | `media.extractor` | `media.swcodec` / 厂商 HAL | 播放音视频 |
| `MediaExtractor` | `media.extractor`（默认） | 不涉及 | 自取样本、自定义播放器 |
| `MediaCodec` | 不涉及 | `media.swcodec` / 厂商 HAL | 编解码、转码 |
| `MediaMetadataRetriever` | `mediaserver` + `media.extractor` | 取缩略图时可能解码 | 元数据、封面、帧图 |
| `MediaParser` | **应用进程** | 不涉及 | 应用自驱动解析 |
| `MediaDataSource` + 上述 API | 取决于 API；数据回调到应用 | — | 网络流、加密流、自定义源 |

### 6.2 权限

读取本地媒体文件的权限随目标 SDK 变化：

- targetSdk 33 及以上：使用细分媒体权限 `READ_MEDIA_VIDEO`、`READ_MEDIA_AUDIO`（图片是 `READ_MEDIA_IMAGES`），官方行为变更页明确要求用它们替代 `READ_EXTERNAL_STORAGE`；
- targetSdk 32 及以下：仍使用 `READ_EXTERNAL_STORAGE`；
- 访问网络媒体：需要 `INTERNET`；
- 通过 `ContentResolver` 打开其他应用的内容：依赖 URI 授权（`FLAG_GRANT_READ_URI_PERMISSION`），而不是媒体权限本身。

`MediaExtractor.setDataSource(FileDescriptor)` 会把 fd 直接交给解析进程。它**不经过 MediaProvider** 这一点可以从接口签名看出来（`makeIDataSource(in FileDescriptor fd, ...)`），但「是否因此绕过权限」不能简单下结论：fd 的取得本身仍受权限、授权与 SELinux 约束，官方文档也没有「绕过」的表述。本文只描述通路，不把它当作权限绕过。

## 7. 公开缺陷台账（2015–2025）

以下条目全部已修复，描述只到成因类别。**2015–2018 年的公告表格没有 Component 列**，组件名只出现在章节标题、描述或引用的代码库里；因此表中单列「代码位置」，并注明它来自公告描述、NVD 还是 AOSP commit。这不是吹毛求疵，而是避免把「章节名」当成「代码库」。

### 7.1 2015 年：Stagefright 批次

2015-08-01 公告的媒体章节包含多条 Critical RCE，受影响版本集中在 Android 5.1 及以前：

| CVE | 级别 | 受影响版本 | 代码位置 | 成因类别 |
|-----|------|------------|----------|----------|
| CVE-2015-1538 | Critical | ≤ 5.1 | `SampleTable.cpp`（NVD） | sample-to-chunk 参数整数溢出 |
| CVE-2015-1539 | Critical | ≤ 5.1 | `ESDS.cpp`（NVD） | 描述符长度下溢 |
| CVE-2015-3824 | Critical | ≤ 5.1 | `tx3g` atom 处理（NVD） | atom 尺寸相加溢出 |
| CVE-2015-3826 | Low | 5.0 / 5.1 | 3GPP 元数据（NVD） | 无界读取，仅导致崩溃 |
| CVE-2015-3827 | Critical | ≤ 5.1 | `covr` atom 处理（NVD） | 整数下溢 |
| CVE-2015-3828 | Critical | ≥ 5.0 | 3GPP 处理（NVD） | 长度判断缺失导致下溢 |
| CVE-2015-3829 | Critical | ≥ 5.0 | `covr` 处理（NVD） | chunk 尺寸溢出 |
| CVE-2015-3832 | Critical | ≤ 5.1 | `MPEG4Extractor.cpp`（NVD） | NAL 长度导致缓冲溢出 |
| CVE-2015-3864 | Critical | ≤ 5.1 | `MPEG4Extractor::parseChunk`（NVD） | 前次修复不彻底后的变体 |

2015-09-01 公告收录了 CVE-2015-3864；2015-10-01 公告不再包含上表条目。一个常被忽略的细节是：这一批并不全是 Critical——CVE-2015-3826 只是 Low 级拒绝服务。把整批都描述成「远程代码执行」会夸大实际影响。

### 7.2 2016 年：Mediaserver 与 libvpx

| CVE | 公告月份 | 章节 | 级别 | 受影响版本 | 成因类别 |
|-----|----------|------|------|------------|----------|
| CVE-2016-0815 | 2016-03 | Mediaserver | Critical RCE | 4.4.4–6.0.1 | 分片读取越界（`MPEG4Source::fragmentedRead`） |
| CVE-2016-0816 | 2016-03 | Mediaserver | Critical RCE | 6.0 / 6.0.1 | 特制媒体文件导致内存破坏 |
| CVE-2016-1621 | 2016-03 | libvpx | Critical RCE | 4.4.4–6.0 | 编码器/解码器边界处理 |

CVE-2016-2504 经常被误传为 Stagefright 条目。它确实存在于 2016-08 公告，但属于 Qualcomm GPU 驱动的 Critical 提权（Quadrooter 批次之一），与 libstagefright 无关。

### 7.3 编解码器库：2019–2023

这些条目在公告中多写作「Media Framework」，具体库名来自公告引用的 AOSP commit 或 NVD：

| CVE | 公告月份 | 代码库 | 级别 | 受影响版本 |
|-----|----------|--------|------|------------|
| CVE-2019-1989 | 2019-03 | `external/libavc` | Critical RCE | 7.0–9 |
| CVE-2019-2028 | 2019-04 | `external/libmpeg2` | Critical RCE | 7.0–9 |
| CVE-2019-2107 | 2019-07 | `external/libhevc` | Critical RCE | 7.0–9 |
| CVE-2019-2108 | 2019-09 | `external/libhevc` | Critical RCE | 10 |
| CVE-2020-0032 | 2020-03 | `external/libavc` | Critical RCE | 8.0–10 |
| CVE-2021-0325 | 2021-02 | `external/libavc` | High RCE | 8.1–11 |
| CVE-2023-21282 | 2023-08 | `external/aac` | Critical RCE | 11–13 |

这张表能看出一个趋势：**同一类库会跨年份反复出现**，因为解码器要处理的是语法极其复杂的码流，修完一个字段还会有下一个字段。2021-02 起 libavc 的同类条目降为 High，说明分层加固（IntSan、BoundSan、CFI）开始改变严重级别的分布，而不是「漏洞停止出现」。

### 7.4 容器解析：样本表与元数据

容器解析的缺陷集中在三类位置：

- **样本表 / chunk 偏移**：`stsc`、`stco`、`stsz` 等表项决定「哪个字节属于哪一帧」，一旦偏移与长度相乘溢出，后续读取就越界。CVE-2015-1538 是这一类。
- **描述符嵌套**：ESDS 描述符带长度字段，长度下溢会让解析器把后续字节当成新描述符。CVE-2015-1539 是这一类。
- **元数据 atom**：`covr`、`tx3g` 等 atom 的尺寸计算。CVE-2015-3824/3827/3829 属于这一类。
- **Matroska**：CVE-2020-0491 在 NVD 中被描述为 `MatroskaExtractor` 的拒绝服务；公告章节仍只写「Media Framework」，没有点名 Matroska。这说明「公告没写格式」不等于「只影响某个格式」。

### 7.5 提权与「媒体进程被攻破之后」

媒体解析进程本身被攻破后，攻击者拿到的是该进程的权限。历史上也有专门针对这一层的提权条目：

| CVE | 公告月份 | 章节 | 级别 | 受影响版本 |
|-----|----------|------|------|------------|
| CVE-2023-20933 | 2023-02 | Media Framework | High EoP | 10–13 |
| CVE-2023-40077 | 2023-12 | Framework（`frameworks/av`） | Critical EoP | 11–14 |
| CVE-2024-0049 / 0050 / 0051 | 2024-03 | Media Framework | High EoP | 视条目而定 |
| CVE-2025-48548 / 48549 | 2025-09 | Media Framework | High | 视条目而定 |

这些条目提醒我们：**「解析器被沙箱隔离」和「沙箱不可逃逸」是两件事**。沙箱降低的是单次内存破坏的后果，不是保证后续没有提权路径。

### 7.6 2023 年之后的趋势

2023 年之后，媒体章节的 Critical RCE 明显减少，更多是 EoP 与信息泄露（High）。2023-08 公告的 CVE-2023-21282（`external/aac`）是近年少见的 Critical RCE，且公告明确写了 "User interaction is needed for exploitation."。2024–2025 年抽样的媒体条目则基本是 EoP/ID 级别。

这不代表媒体解析变「安全」了，而是说明：

- 高危内存破坏被分层加固压低了可利用性；
- 攻击面从「远程代码执行」转向「本地提权与信息泄露」；
- 旧的、不再维护的格式解析器仍是风险点（补丁只覆盖受支持版本）。

### 7.7 读媒体公告的方法

1. **先看章节，再看行文，最后看引用。** 2019 年后的公告有独立的 Media Framework 章节；2023 年起很多条目并入 Framework。章节名只表示归类，不表示代码库。
2. **2015–2018 年没有 Component 列。** 不要从公告表格里「读出」组件名；组件名来自章节描述、NVD 或 commit。
3. **注意「远程」的定义。** 2015-08 公告的措辞是 "could enable remote code execution … through multiple methods such as email, web browsing, and MMS when processing media files."——「远程」指投递方式与处理时机，不等于「收到即执行」。2023-08 公告则直接写明需要用户交互。
4. **区分 RCE 与 DoS。** 同一批次里可能混有 Low 级 DoS（如 CVE-2015-3826），不要把整批都算成 RCE。
5. **区分「受影响版本」与「修复版本」。** 公告列的是受影响范围，修复通常随当月补丁级别下发；旧设备是否真的拿到补丁，要看厂商。

## 8. 六种成因模式

把公开条目按成因归类，可以看出媒体路径上反复出现的六种模式。它们不是「漏洞列表」，而是审查代码时的检查清单。

### 8.1 长度字段的算术

容器与码流里到处是长度、偏移、计数。常见错误包括：

- 两个 32 位长度相加或相乘后溢出；
- 长度字段为 0 或 `SIZE_MAX` 时走了未覆盖的分支；
- 用有符号数接收无符号长度，负数被当成巨大值；
- 先按长度分配，再按另一个长度读取。

官方给出的比例是「数组边界检查缺失/错误约占 Android 用户态漏洞的 34%」，这类问题在媒体解析器中尤其集中。

### 8.2 索引表与「先信后验」

样本表、chunk 偏移表、cue 表都是「用文件里的数字去索引内存」。如果解析器先用表项算出偏移，再去读取，而没有验证偏移 + 长度 ≤ 文件长度，就会越界。更隐蔽的版本是**先信后验**：先用未验证的值分配缓冲区，再在别处校验，但分配已经发生。

### 8.3 嵌套结构与递归

容器可以嵌套容器（MP4 的 box、Matroska 的 EBML、TS 的 PSI 表），描述符也可以嵌套（ESDS）。递归深度、嵌套层数、子结构数量如果没有上限，会导致栈耗尽或超大分配。这类问题常表现为 DoS，但在特定布局下也可能变成内存破坏。

### 8.4 码流语法与参考帧

解码器要维护参考帧列表、切片边界、宏块索引。码流可以声明「参考一个不存在的帧」「切片跨越图像边界」「参数集在流中途改变」。解码器如果假设这些关系总是成立，就会在帧缓冲里越界。CVE-2019-1989、CVE-2020-0032 等 libavc 条目属于这一大类。

### 8.5 资源耗尽

分辨率、帧率、采样率、声道数、码率都可以由文件声明。解析器或解码器如果按声明值直接分配（例如「宽 × 高 × 4」），一个精心构造的文件就能触发超大分配，导致进程被杀或系统卡顿。这类问题的修复通常不是「加一个 if」，而是引入上限与增量分配。

### 8.6 生命周期与所有权

解码输出要跨进程交给 Surface 或 AudioTrack，缓冲区可能被映射、共享、异步释放。如果解析器或解码器在缓冲区交出后仍持有指针，就会出现 use-after-free。官方统计里 UAF、整数溢出、OOB 合计占 90%，生命周期问题正是其中一部分。

## 9. 加固演进时间线

| 时间 | 措施 | 官方依据 |
|------|------|----------|
| Android 7.0 | 拆分 `mediaserver`，引入 `media.extractor` / `media.codec` 等独立进程 | framework-hardening |
| Android 7.0 | 在媒体框架启用整数溢出 sanitizer（IntSan） | ubsan 页面："to harden the media framework in Android 7.0" |
| Android 8.1 | 在媒体栈启用 LLVM CFI | cfi 页面："In Android 8.1, we enabled LLVM's implementation of CFI in the media stack." |
| Android 9 | IntSan 覆盖扩大；官方称其在 Pie 中缓解了 11 个 Critical 漏洞 | 开发者博客 |
| Android 10 | 软件编解码器移入 `media.swcodec` 沙箱；BoundSan 覆盖 11 个媒体编解码器 | 开发者博客 / ubsan 页面 |
| Android 11（API 30） | `MediaParser` 公开，应用可选择进程内解析 | MediaParser 参考文档 |
| API 36 | `MediaCodecInfo.getSecurityModel()` 区分「沙箱」与「内存安全实现」 | MediaCodecInfo 参考文档 |

这条时间线的方向很清楚：**先拆进程（限制后果），再加编译器缓解（降低可利用性），最后把「安全模型」变成可查询属性（让应用能做选择）**。三者互补，不能互相替代。

## 10. 观测点与自查

以下是防御方可以做的检查。命令与字段随版本和厂商变化，只作为入口，不作为判据。

### 10.1 进程与身份

```bash
adb shell ps -A -o USER,PID,NAME | grep -Ei 'media|codec'
adb shell getprop | grep -Ei 'media\.|codec'
adb shell ls -Z /apex/com.android.media.swcodec/bin/mediaswcodec
adb shell ls -Z /system/bin/mediaextractor
```

预期看到：

- `mediaextractor` 以 `mediaex`（1040）运行；
- `media.swcodec` 以 `mediacodec`（1046）运行；
- 两者的 SELinux 上下文分别属于 `mediaextractor` / `mediaswcodec` 域；
- 若 `media.stagefright.extractremote` 被改为 `false`，解析会回到调用进程，这属于非默认配置，值得记录。

### 10.2 编解码器清单

应用侧用第 5.4 节的代码枚举 `MediaCodecList`，重点看：

- 是否存在纯软件组件（`isSoftwareOnly()` 为 true）；
- 厂商组件（`isVendor()` 为 true）的比例；
- API 36 设备上 `getSecurityModel()` 的返回值是否合理（全为 0 或全为 1 都值得追问）。

### 10.3 崩溃与日志

媒体解析器崩溃通常表现为：

- `media.extractor` 或 `media.swcodec` 进程重启；
- `tombstone` 中出现 `libstagefright`、`libavc`、`libhevc`、`libcodec2_soft_*` 等库名；
- 应用侧看到 `MediaCodec.CodecException`、`MediaExtractor` 抛 `IOException`，或播放器直接报错。

对安全评估而言，「某个文件能让媒体进程崩溃」本身就是需要记录的输入面证据，即使它不一定能变成代码执行。

### 10.4 自查清单

- [ ] 应用是否把不可信远端数据直接交给 `MediaExtractor` / `MediaParser` / `MediaCodec`？
- [ ] 是否使用自定义 `MediaDataSource`，并假设 `readAt` 的返回值总是等于请求长度？
- [ ] 是否在解析前限制了文件大小、轨道数、分辨率、采样率？
- [ ] 是否依赖厂商解码器处理安全敏感内容？如果是，补丁节奏是否可接受？
- [ ] targetSdk 33+ 是否已迁移到 `READ_MEDIA_VIDEO` / `READ_MEDIA_AUDIO`？
- [ ] 是否把「预览/取封面」当成低风险路径，而忽略了它同样经过解析器？
- [ ] 是否在应用内自带解析器或解码器？如果是，它不在系统补丁覆盖范围内。

## 11. 常见误传纠正

| 误传 | 实际情况 |
|------|----------|
| 「Stagefright 只要收到短信就会自动执行」 | 2015-08 公告的措辞是「在处理媒体文件时」，投递方式包括邮件、浏览、MMS；不等于「收到即执行」 |
| 「Stagefright 影响 9.5 亿设备」 | 这是 Zimperium 2015 年博客中的估算（由 95% 推算），装机量本身无法核实 |
| 「MediaParser 是 Android 12 引入并取代 MediaExtractor」 | API 30（Android 11）；与 MediaExtractor 并存，没有取代关系 |
| 「Codec2 从 Android 10 起完全取代 OMX」 | 未核实；AOSP 仍按 `c2.` / `omx.` 前缀选择实现，OMX 仍可在 vendor 分区使用 |
| 「Android 7 之后 mediaserver 被删除了」 | 仍存在，继续承载 MediaPlayerService、ResourceManagerService 等 |
| 「软件解码跑在应用进程」 | 默认在 `media.swcodec`（`mediacodec`，UID 1046） |
| 「媒体解析一定在隔离进程」 | `MediaParser` 默认在应用进程内解析；`media.stagefright.extractremote=false` 也会回到本地 |
| 「公告写 Media Framework 就是 libstagefright」 | 章节名只是归类；2019 年后的条目可能指向 libavc、libhevc、libmpeg2、aac 等 |
| 「CVE-2016-2504 是 Stagefright」 | 2016-08 公告中的 Qualcomm GPU 驱动提权，与 libstagefright 无关 |
| 「libg729 有对应的媒体公告条目」 | 本次检索未找到；不要把「代码库里存在某解码器」当成「有公开 CVE」 |
| 「媒体漏洞都能通过网页零点击触发」 | 2023-08 公告明确写 "User interaction is needed for exploitation." |
| 「`setDataSource(fd)` 可以绕过 MediaProvider 权限」 | 官方没有此表述；fd 的取得仍受权限与授权约束，本文只描述通路 |
| 「硬件解码一定比软件解码安全」 | 硬解在厂商进程，补丁随厂商 OTA；软解沙箱更紧但官方也明确「不保证性能」 |

## 12. 与其他手册的分工

- 媒体**采集**（相机、麦克风、屏幕录制）见《摄像头、麦克风与屏幕捕获面手册》；
- 音频**路由与会话**见《音频栈与媒体会话手册》；
- 解码后的**合成与显示**见《图形与显示栈手册》；
- 媒体进程的**SELinux/seccomp 机制**见《SELinux 与 seccomp 沙箱手册》；
- 通用**内存缺陷类型**见《内存安全与分配器手册》；
- 媒体文件作为**应用外部输入**的 URI/fd 处理见《文件路径与 URI 处理类手册》。

## 13. 参考

- Android Open Source Project：`frameworks/av/media/libstagefright/MediaExtractorFactory.cpp`、`media/libmedia/aidl/android/IMediaExtractorService.aidl`、`services/mediaextractor/mediaextractor.rc`、`services/mediacodec/mediaswcodec.rc`、`services/mediacodec/android.hardware.media.omx@1.0-service.rc`、`media/libstagefright/MediaCodec.cpp`
- AOSP `system/sepolicy/private/`：`mediaextractor.te`、`mediaswcodec.te`、`mediaserver.te`
- AOSP `system/core/libcutils/include/private/android_filesystem_config.h`
- source.android.com：`/docs/core/media/framework-hardening`、`/docs/core/media/media-modules`、`/docs/security/test/ubsan`、`/docs/security/test/cfi`
- developer.android.com：`MediaParser`、`MediaCodecInfo`、`MediaExtractor` 参考页；Android 13 行为变更（细分媒体权限）
- Android Developers Blog：Queue hardening enhancements（2019-05）
- Android Security Bulletin：2015-08-01、2015-09-01、2016-03-01、2016-08-01、2019-03-01、2019-04-01、2019-07-01、2019-09-01、2020-03-01、2021-02-01、2023-02-01、2023-08-01、2023-12-01、2024-03-01、2025-09-01
- NVD：上表各 CVE 条目

## 未核实与失效来源

**A. MediaParser 的进程模型**
官方参考文档没有说明解析发生在应用进程还是系统进程。本文的「默认在应用进程内」来自 AOSP 源码证据（应用侧直接构造 extractor），**属于源码推断而非文档表述**。

**B. Codec2 成为默认实现的版本**
未找到「Android 10 起 Codec2 全局默认」的官方原句。本文只写「按组件名前缀选择」，不写默认版本。

**C. OMX 的弃用时间表**
未找到官方弃用声明。AOSP 仍保留 ACodec 与 `vendor.media.omx`，**正文不写「已弃用」**。

**D. `media_codecs.xml` 的读取顺序**
社区常说「先读 APEX 再读 vendor」，本次未在官方文档中取到该表述，**正文不写顺序**，只写「系统组件随 APEX 更新，vendor 组件由厂商提供」。

**E. `setDataSource(fd)` 是否「绕过」MediaProvider**
接口签名显示 fd 直接进入解析进程，但官方没有「绕过权限」的表述，**正文只描述通路**。

**F. 厂商编解码服务的具体名称与身份**
AOSP 标准名是 `vendor.media.omx`；厂商可自定义 `c2` 服务名。**正文不列举具体厂商服务名**。

**G. `AID_MEDIA_CODEC` 的历史版本**
只核对当前 master 的 1046；旧版本是否使用同一数值未逐版核实。

**H. `/dev/dri` 的具体节点**
`mediaswcodec.te` 只出现 `gpu_device`、`dmabuf_system_heap_device` 等标签，**正文不写具体设备节点路径**。

**I. `mediaserver` 读写应用数据的具体用途**
策略允许 `app_data_file` 的读写，但本文未逐项核实哪些功能需要它，**正文只陈述策略事实**。

**J. 2015–2018 年 CVE 的代码位置**
公告当年没有 Component 列；表中的 `SampleTable.cpp`、`ESDS.cpp`、`MPEG4Extractor.cpp` 等来自 NVD 描述与公开 commit。**它们不是公告原文**，正文已注明来源。

**K. `dumpsys media.player` 等命令的字段**
字段随版本与厂商变化，**正文只把命令作为观测入口**。

**L. 「9.5 亿设备」**
Zimperium 原文可核实，但装机量是估算，**正文按估算处理**。

**M. `getSecurityModel()` 的厂商覆盖**
API 36 定义了常量与语义，但各厂商组件是否都返回有意义的值未核实，**正文只说「取决于厂商与组件」**。

**N. 安全解码器当前所在的进程**
Android 7.0 文档描述的是当时状态（安全解码器留在 `mediaserver`）；现代实现可能与 DRM HAL 绑定，**正文按版本区分，不把 7.0 的描述套用到今天**。

---

## 联系方式

咨询 Android 系统详情请联系 https://t.me/one00190

---

*本手册仅供安全学习与防御研究使用；所有机制表述、公告列值与 CVE 描述均取自公开一手来源并附链接，未能核实的部分已单列于「未核实与失效来源」而非写入正文。*
