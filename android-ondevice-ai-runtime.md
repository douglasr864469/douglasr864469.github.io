---
layout: page
title: "Android 设备端 AI 推理运行时攻击面手册"
description: "把「模型文件」当作文档型攻击面来审：NNAPI 自 Android 15 弃用（官方建议迁移 TF Lite GPU runtime）、控制面走 Binder 而张量走共享内存、HAL 驱动在 vendor 进程、TFLite 以 `TFL3` 标识且权重是模型自己写的偏移+长度；另含形状乘法溢出、Bu..."
date: 2026-11-01
permalink: /android-ondevice-ai-runtime/
---

> 每一份模型文件，本质上都是一份**结构复杂、由 native 代码解析、并且会直接影响内存布局的文档**。它和 PDF、MP4、字体没有区别——只不过解析器常常跑在比应用更高权限的地方：厂商的 NPU 驱动进程、系统的推理服务、GPU 的着色器编译器。过去十年媒体格式踩过的坑，模型格式正在重走一遍。本手册只讨论公开且已修复问题的机制与自查方法，不含可利用代码。

---

## 0. 这篇手册解决什么问题

已发布的手册里，《ART 运行时与代码加载》讲的是「代码怎么被执行」，《图形与显示栈》讲的是「像素怎么流动」。本篇讲一个新的栈：**张量怎么从模型文件变成一次推理**。

三件必须先说清的事：

1. **NNAPI 已经弃用**。官方在 NNAPI 页面顶部写明：NNAPI is deprecated，在 Android 15 弃用；官方建议性能敏感场景迁移到 TF Lite GPU runtime 等替代方案。仍在用 NNAPI 的设备与代码不会立刻失效，但「新设备会用 CPU backend」是官方预期。
2. **模型是输入，不是代码**。它由 native 解析器处理，天然属于不可信输入。
3. **推理经常发生在应用进程之外**。HAL 驱动在 vendor 进程里，这意味着模型解析的缺陷可能落在比应用更高的权限域。

---

## 1. 架构：三层运行时与两种格式

### 1.1 三条并存的路径

Android 上的设备端推理不是一条链路，而是三条：

| 路径 | 组成 | 谁提供 |
|------|------|--------|
| 平台加速接口 | NNAPI（`libneuralnetworks`）+ HAL 驱动 | AOSP + 厂商 |
| 应用内置运行时 | TFLite / LiteRT 解释器 + delegates | 应用自带 |
| 厂商 SDK | QNN、NeuroPilot 等 | 芯片厂商 |

三者不是替代关系。最常见的组合是「应用带 LiteRT，把算子委派给 GPU delegate 或 NNAPI delegate」。所以一次推理可能同时经过三方代码。

### 1.2 NNAPI 的结构

```
应用进程                    vendor 进程                   硬件
────────                   ───────────                   ────
ANeuralNetworksModel
ANeuralNetworksCompilation ──(Binder)──▶ IDevice / IPreparedModel
ANeuralNetworksExecution                     └─ IBurst.execute()
  └─ 输入/输出 buffer ──(共享内存)────────────────▶ NPU / DSP / GPU
```

要点：

- **控制面走 Binder**（`IDevice`、`IPreparedModel`、`IBurst`），**数据面走共享内存**（ashmem 或 `AHardwareBuffer`）。张量不会经过 Binder 序列化。
- HAL 形态：HIDL 是 `android.hardware.neuralnetworks@1.0` 到 `@1.3`；AIDL 形态提供 `IDevice` / `IPreparedModel` / `IBurst`。具体服务实例名因厂商与版本而异（未核实）。
- **优先级**：`ANeuralNetworksCompilation_setPriority()` 可以要求 HIGH / MEDIUM / LOW。这会影响调度，但**不影响隔离**——同进程内的模型之间没有内存隔离。
- **Burst**：把多次执行绑在一起以减少开销。Burst 复用了同一组 buffer slot，这让「上一次执行的输出」与「下一次执行的输入」可能在物理上重叠。

### 1.3 模型格式：FlatBuffers

NNAPI 与 TFLite 都用 FlatBuffers 表示模型。TFLite 文件有一个非常稳定的特征：**偏移 4 处是标识符 `TFL3`**（FlatBuffers 的 file identifier）。这也是判断「这是不是一个 TFLite 模型」最快的办法。

TFLite 的 `Model` 表包含这些关键字段：

| 字段 | 内容 | 风险点 |
|------|------|--------|
| `version` | schema 版本 | 版本与解析器不匹配时的兼容分支 |
| `operator_codes` | 算子及其版本 | 未知算子的回退路径 |
| `subgraphs` | 张量与算子图 | 图结构可以任意嵌套 |
| `buffers` | 权重数据（偏移 + 长度） | **偏移/长度是模型自己写的** |
| `metadata` | 自定义元数据 | 解析方常忽略校验 |
| `signature_defs` | 命名输入输出 | 名称可重复、可超长 |

**这张表的最后一行是整篇的核心**：权重不是「文件里的固定结构」，而是「指向文件内部的偏移 + 长度」。解析器必须自己验证这两个数是否落在文件范围内、是否互相重叠、是否满足对齐要求。任何一处漏检，就是一次越界读。

### 1.4 Delegate：把模型的一部分交给别人

TFLite 的 delegate 机制允许把子图交给别的执行器：

- **GPU delegate**：把算子编译成 GPU 计算内核，走 OpenGL ES 或 OpenCL。模型的形状、步长、量化参数都会进入 GPU 驱动。
- **NNAPI delegate**：把子图转成 NNAPI 模型，交给平台的 HAL 驱动。
- **XNNPACK**：CPU 侧的高性能实现。

delegate 的意义是性能，代价是**把模型可控的参数带进了另一层驱动**。GPU 驱动与 NPU 驱动都是闭源的、都在比应用更高的权限域里，且通常没有独立的安全更新通道。

---

## 2. 攻击面：五类

### 2.1 模型解析：新的「文档漏洞」温床

媒体格式的历史教训是清晰的：一旦某个格式被广泛使用，解析器就会成为攻击目标。模型格式正在复制这条路，而且更难：

- 结构比媒体容器更复杂（图、张量、算子、属性四层嵌套）；
- 尺寸计算涉及乘法（`H × W × C × sizeof(dtype)`），**整数溢出**是常态而非例外；
- 权重偏移由模型提供，天然可指向文件任意位置；
- 解析器常常同时存在于应用进程与厂商 HAL 进程。

### 2.2 形状与步长：算术就是攻击面

一个张量的内存占用不是「读一个数」，而是「一串乘法」。攻击者可以把每一个维度都设成极大值或零，触发：

- 乘法溢出后分配过小的缓冲区；
- 零维度导致的除零或空指针解引用；
- 步长与维度不一致导致的行/列越界。

这类缺陷的特征是：**看起来完全合法的模型，只是数字很极端**。基于「文件大小」或「字段范围」的粗粒度校验挡不住。

### 2.3 HAL 驱动：高权限的闭源解析器

NPU 驱动通常在 vendor 进程内，通过 HIDL/AIDL 接收模型与张量描述。它的问题有三个叠加因素：

1. **权限高**：能访问 DMA、IOMMU 映射、硬件寄存器。
2. **闭源**：外部无法审计。
3. **更新慢**：通常随整包或厂商补丁更新，不随 AOSP 月度补丁。

所以「模型解析漏洞」在应用进程里的影响是「本应用崩溃」，在 HAL 进程里的影响可能是「设备级内存破坏」。

### 2.4 共享内存与 buffer 别名

张量数据走共享内存，意味着：

- 同一块物理内存可能同时被标为「这次执行的输入」和「上次执行的输出」；
- Burst 复用 slot，使重叠成为常态；
- 如果驱动对「输入输出是否重叠」缺少校验，就会出现「计算中途数据被覆盖」——结果错误，但不崩溃，极难发现。

这类问题不是内存安全，而是**正确性安全**：模型输出被悄悄篡改，而所有校验都通过。

### 2.5 模型供应链

模型从哪里来，决定了它是否可信：

- 打包在 APK 里 → 随 APK 签名一起受保护；
- 运行时从网络下载 → **平台不强制签名**，完整性由应用自己负责；
- 由系统组件（如 AI Core 类服务）下发 → 依赖该服务的信任模型。

第二类是实际中最常见的。一个「支持热更新模型」的功能，等于在应用里开了一个**可远程更新的解析器输入入口**，其风险等级与「远程下发可执行代码」相当，但往往按「下发配置」对待。

---

## 3. 成因分析

### 3.1 把模型当「数据」，没当「输入」

模型通常被当作资源文件（assets）对待：不校验、不设上限、不做结构验证。而它实际上是一个由 native 代码深度解析的二进制格式。**分类错误导致防护缺位**，这是最根本的成因。

### 3.2 性能压力推动「少校验」

推理对延迟极敏感，每一层校验都要花时间。于是出现一种普遍取舍：在热路径上只做「必要」检查。而「必要」由实现者判断，边界往往被压到很窄。

### 3.3 delegate 把信任边界推到了驱动里

应用开发者通常认为「我只是调了个 delegate」。实际上模型结构、张量参数、量化系数都流进了 GPU/NPU 驱动。**信任边界被无声地外推**，而开发者没有感知。

### 3.4 厂商驱动的更新节奏与 AOSP 不同步

NPU 驱动属于 vendor 分区，安全补丁由芯片厂商发布。设备的安全补丁级别（SPL）可能与驱动实际版本不一致——这是整个 Android 生态的已知结构性问题，在 AI 栈上尤其突出，因为这部分代码很新。

### 3.5 NNAPI 弃用带来迁移期风险

弃用意味着两条路径会长期并存：老代码继续用 NNAPI，新代码迁到 LiteRT 或厂商 SDK。迁移期最常见的问题是**同一份模型在两个运行时上行为不一致**，以及**旧路径不再获得修复**。官方对 NNAPI 的表述是「可以继续使用，但预期多数设备未来使用 CPU backend」——这句话的含义是：即使不迁移，也不要指望硬件加速仍然可用。

### 3.6 缺少统一的模型完整性标准

APK 有签名方案 v2/v3/v4，媒体有 DRM，而模型没有对应的平台级完整性机制。每个应用自己决定要不要校验哈希、要不要绑定版本。结果是同一个「模型热更新」功能，在不同应用里的安全强度相差一个数量级。

---

## 4. 公开状态台账

| 项目 | 状态 |
|------|------|
| NNAPI | 自 Android 8.1（API 27）可用；**Android 15 弃用**；官方建议迁移到 TF Lite GPU runtime 等方案 |
| NNAPI HAL | HIDL `@1.0`–`@1.3`；AIDL 形态提供 `IDevice`/`IPreparedModel`/`IBurst` |
| TFLite / LiteRT | 应用自带运行时；`.tflite` 以 `TFL3` 为文件标识；提供 GPU / NNAPI / XNNPACK delegate |
| 平台级模型签名 | 未见 AOSP 强制机制（未核实是否有面向系统组件的内部机制） |

与前面几篇同理，本手册**不对模型解析相关 CVE 做逐条组件归属**：缺陷分布在推理运行时、GPU 驱动、NPU 固件三层，公告组件名（如 `Media Framework`、`GPU Driver`）与具体补丁之间缺少稳定映射。

---

## 5. 防护技术

### 5.1 应用侧

- **把模型当不可信输入**：加载前用 FlatBuffers 的 verifier 校验结构，再校验每个张量的元素数不超过上限。
- **对下载的模型做完整性校验**：至少校验哈希，最好做签名；把「模型版本」与「应用版本」绑定，拒绝降级。
- **限制解析器所在进程的权限**：能在普通应用进程里解析，就不要放到特权服务里。
- **不要在热路径上省略边界校验**：把校验放在加载期一次性完成，而不是每次推理。
- **对 delegate 失败做降级**：GPU/NPU 不可用时回退到 CPU，不要假设硬件路径一定可用（尤其在 NNAPI 弃用之后）。

### 5.2 平台与厂商侧

- **分层隔离**：模型解析放在权限最低的进程，HAL 只接收「已校验」的描述结构。
- **HAL 模糊测试**：对 `IDevice`/`IPreparedModel`/`IBurst` 的接口做结构化模糊测试，重点覆盖极端形状、零维度、重叠 buffer。
- **IOMMU/SMMU 强制启用**，让 NPU 的 DMA 不能越界访问系统内存。
- **内存安全加固**：CFI、shadow call stack；支持 MTE 的 SoC 应在推理进程启用。
- **给模型解析器建立独立的补丁通道**，不要等整包更新。

### 5.3 企业侧

- 盘点「会下载模型」的应用：这是最容易被忽略的远程输入面。
- 把「模型来源域名」纳入网络策略，而不是只按应用放行。
- 对涉及敏感数据的推理（语音、图像），要求模型与运行时都来自受控渠道。

---

## 6. 自查与观测清单

```bash
# 1. NNAPI / 推理服务
service list | grep -i neuralnetworks
dumpsys | grep -i nnapi            # 部分版本提供，未核实

# 2. 应用内的模型文件
adb shell pm path <package>
# 拉取 APK 后检查 assets 目录中的 .tflite / .bin / .onnx 文件

# 3. 运行时库
adb shell lsof -p <pid> | grep -i "neural\|tflite\|nnapi"

# 4. 下载的模型（若应用缓存到私有目录）
adb shell run-as <package> ls -l files/ 2>/dev/null

# 5. HAL 进程
adb shell ps -A | grep -i "vendor.*nn\|neural"
```

判读要点：

- `service list | grep -i neuralnetworks` 为空，说明设备可能没有任何 NNAPI HAL 实例（在 NNAPI 弃用后会越来越常见），此时推理走 CPU 路径。
- 应用私有目录里出现「运行时下载的模型文件」，且没有配套的签名/哈希，属于应当核实的输入面。
- 模型文件大小与内存占用不匹配（例如 10 MB 的模型申请了几百 MB），是形状计算异常的典型信号。

---

## 7. 常见误传纠正

1. **「NNAPI 是模型格式」**——错。它是平台加速接口；模型用 FlatBuffers 表达。
2. **「NNAPI 在 Android 16 才弃用」**——错。官方口径是 **Android 15** 弃用。
3. **「弃用等于立即失效」**——错。仍可使用，但官方预期多数设备未来走 CPU backend。
4. **「模型文件是只读数据，不构成攻击面」**——错。它由 native 解析器处理，属于不可信输入。
5. **「推理一定在云端」**——错。设备端推理是常态，且是隐私设计的一部分。
6. **「GPU delegate 只是加速」**——错。模型参数会进入 GPU 驱动，信任边界被外推。
7. **「NPU 驱动跑在应用进程里」**——错。它在 vendor 进程内，通过 HAL 接口通信。
8. **「张量数据通过 Binder 传」**——错。走共享内存，Binder 只传控制。
9. **「量化模型更安全」**——错。量化改变数据类型，不减少解析面。
10. **「TFLite 与 NNAPI 是同一层」**——错。TFLite 是运行时与格式，NNAPI 是平台加速接口，二者通过 NNAPI delegate 连接。
11. **「模型文件有签名，平台会强制校验」**——未见 AOSP 强制机制，完整性由应用负责。
12. **「限制模型文件大小就能防止内存破坏」**——错。一个几 KB 的模型就能构造极端的形状与偏移。
13. **「推理结果错误一定是模型的精度问题」**——不一定，也可能是 buffer 重叠导致的数据覆盖。

---

## 未核实与失效来源

- AIDL 形态 NNAPI 的服务实例命名规则：未核实。
- 各家 NPU（高通 Hexagon、联发科 APU、Google Edge TPU）驱动的具体隔离配置（IOMMU 是否强制、进程权限）：未核实，不逐厂商断言。
- 平台级模型签名/完整性机制是否存在：未核实。
- AI Core 类系统组件（如 Gemini Nano 相关服务）的接口与信任模型：未核实，本手册不展开。
- 推理相关 CVE 的组件归属：本手册有意不做逐条归属，理由见第 4 节。

---

## 参考

- NNAPI（含弃用声明）：[developer.android.com/ndk/guides/neuralnetworks](https://developer.android.com/ndk/guides/neuralnetworks)
- NNAPI 迁移指南：[developer.android.com/ndk/guides/neuralnetworks/migration-guide](https://developer.android.com/ndk/guides/neuralnetworks/migration-guide)
- NNAPI HAL：[source.android.com/docs/core/interaction/neuralnetworks](https://source.android.com/docs/core/interaction/neuralnetworks)
- LiteRT / TensorFlow Lite：[ai.google.dev/edge/litert](https://ai.google.dev/edge/litert)
- TFLite delegate：[ai.google.dev/edge/litert/performance/delegates](https://ai.google.dev/edge/litert/performance/delegates)
- TFLite schema（FlatBuffers）：[github.com/tensorflow/tensorflow/blob/master/tensorflow/lite/schema/schema.fbs](https://github.com/tensorflow/tensorflow/blob/master/tensorflow/lite/schema/schema.fbs)
- `ANeuralNetworks.h`：[android.googlesource.com/platform/frameworks/ml/+/refs/heads/main/nn/runtime/include/NeuralNetworks.h](https://android.googlesource.com/platform/frameworks/ml/+/refs/heads/main/nn/runtime/include/NeuralNetworks.h)
- FlatBuffers 验证器：[flatbuffers.dev](https://flatbuffers.dev)

---

## 联系方式

如果你在企业里需要就「应用是否在运行时下载模型」「推理路径的隔离强度」「NPU 驱动补丁是否落地」做一次核查，可以通过 Telegram 联系：<https://t.me/one00190>

---

*本手册仅用于防御与教育目的，讨论的均为公开且已被修复问题的机制与设计缺陷，不包含任何可直接利用的代码。文中标注「未核实」的内容不应作为事实引用；发现错误欢迎指出。*
