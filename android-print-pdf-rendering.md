---
layout: page
title: "Android 打印、PDF 与文档渲染通路攻击面手册"
description: "把「打印」拆成发起／绑定／写入／暂存／投递／输出六段：`PrintService` 是公开基类（`BIND_PRINT_SERVICE` 限绑定方不限实现）、`onWrite()` 写入 spooler 管道 FD 且不可 seek、「另存为 PDF」是假打印机 + SAF 而非打印服务；PDF 侧讲清..."
date: 2026-11-04
permalink: /android-print-pdf-rendering/
---

> 打印在移动端看起来是个边缘功能，但它把三样东西连在了一起：应用私有数据、系统打印后台（spooler）、以及一个可能完全不受应用控制的输出目标。一次「打印为 PDF」会穿过 `PrintManager`、`PrintDocumentAdapter`、spooler 的管道、SAF 文档选择器，最后落到某个 `content://` 上；而 PDF 本身的解析又由 pdfium 与 Skia 分别承担。这条链上既有文件描述符的交接，也有跨应用 URI 的授权，还有解析器的内存安全边界。这份手册沿「发起 → 生成 → 暂存 → 输出 → 解析」拆解机制、公开缺陷与防御。只讨论公开且已修复的问题，不含可利用代码。

## 0. 使用方式与边界

这份手册面向三类读者：

- **应用开发者**：需要理解 `PrintDocumentAdapter.onWrite()` 拿到的文件描述符归谁所有、为什么打印结束后不能再写、以及「打印为 PDF」与应用自己生成 PDF 的区别。
- **系统与平台工程师**：需要区分 `PrintManager`（客户端入口）、`PrintService`（打印服务实现）、spooler（系统打印后台）与 `PdfRenderer`/`PdfDocument`（PDF 读写）四组组件。
- **安全评估者**：需要在看到「打印/PDF」公告时，判断问题落在打印服务绑定、spooler 的 IPC、pdfium 解析还是 URI 授权，而不是把所有带 `print` 或 `pdf` 的 CVE 混为一类。

本文的三条自我约束：

1. 只引用公开且已修复的问题；CVE 编号、落点与描述以 NVD 为准，公告分类与级别以 Android Security Bulletin 为准。
2. 不写利用步骤、不写 PoC、不写可直接复现的攻击参数。
3. 查不到一手来源的说法一律进「未核实与失效来源」。

与已发布手册的分工：

- 《Android 文件路径与 URI 处理攻击面手册》讲 URI 规范化与授权的一般规则；本文只讲打印与「另存为 PDF」这两条具体通路。
- 《Android 媒体解码与容器解析攻击面手册》讲媒体容器与解码器；PDF 是另一类文档格式，本文只讲 PDF 的读写边界，不重复媒体解析。
- 《Android 图形与显示栈攻击面手册》讲渲染与合成；本文只把 PDF 页面渲染作为输出环节引用。

## 1. 一次打印的完整旅程

把「用户点了打印」拆开，至少经过八个阶段：

| 阶段 | 主要位置 | 做的事 | 安全判断点 |
|------|----------|--------|------------|
| 1. 发起 | Activity | `PrintManager.print()` 创建打印任务 | 只有 Activity 能发起；任务名与适配器 |
| 2. 选择 | 系统打印界面 | 选打印机、纸张、份数、颜色 | 目标是否是用户预期 |
| 3. 绑定 | `PrintService` | 系统绑定已启用的打印服务 | `BIND_PRINT_SERVICE` 限绑定方，不限实现方 |
| 4. 请求文档 | spooler → 应用 | 通过 `PrintDocumentAdapter` 请求布局与写入 | 页面范围、取消信号 |
| 5. 写入 | 应用进程 | `onWrite()` 把 PDF 写入给定 FD | FD 是管道，关闭即结束 |
| 6. 暂存 | spooler | 把管道内容落到 spool 文件 | 临时文件的访问控制与清理 |
| 7. 投递 | spooler → 打印服务 | 打印服务通过 `PrintDocument.getData()` 读取 | 只读管道，必须读完并关闭 |
| 8. 输出 | 打印机 / 文件 | 物理打印，或「另存为 PDF」写入 SAF URI | 目标 URI 的授权与覆盖行为 |

关键点是：**打印通路里传递的是文件描述符，不是文件路径，也不是 URI**。应用把 PDF 字节写进 spooler 给的管道，spooler 再把内容交给打印服务；整个过程中应用并不需要（也不应该）知道最终文件落在哪里。理解这一点，才能正确判断「打印泄露」到底发生在哪一段。

第二个关键点是：**「打印为 PDF」不是 `PrintService`**。它由 spooler 内部的 `PrintActivity` 伪造一个名为 `PDF printer` 的打印机，确认后用 `ACTION_CREATE_DOCUMENT` 让用户选择保存位置，再通过 SAF 的 `content://` 打开输出流写出。它走的是打印 UI，但输出端是文档提供者，不是打印服务。

## 2. 对象模型与进程边界

### 2.1 关键对象

| 对象 | 所在层 | 作用 |
|------|--------|------|
| `PrintManager` | 公开 SDK | 发起打印；只有 Activity 能调用 `print()` |
| `PrintAttributes` | 公开 SDK | 介质尺寸、分辨率、颜色模式、边距 |
| `PrintDocumentAdapter` | 公开 SDK（API 19） | 应用实现：布局、写入、取消 |
| `PrintService` | 公开 SDK（`android.printservice`） | 第三方可实现的打印服务基类 |
| `PrintJob`（`android.print`） | 公开 SDK | 客户端看到的任务状态 |
| `PrintJob`（`android.printservice`） | 公开 SDK | 服务侧任务，`getDocument()` 取文档 |
| `PrintDocument` | `android.printservice` | 服务侧文档句柄，`getData()` 返回只读管道 |
| `RemotePrintDocument` | spooler 内部 | 通过 `WriteCommand` 把 sink 交给适配器 |
| `PdfRenderer` | 公开 SDK（API 21） | 读取并渲染 PDF，底层 pdfium |
| `PdfDocument` | 公开 SDK（API 19） | 生成 PDF，底层 Skia |

### 2.2 进程与权限边界

| 进程/组件 | 身份 | 能看到什么 | 能做什么 |
|-----------|------|------------|----------|
| 应用进程 | 应用 uid | 自己的文档内容 | 通过适配器写入 spooler 管道 |
| spooler | 系统进程 | 所有打印任务的暂存内容 | 暂存、转发、触发「另存为 PDF」 |
| 打印服务 | 第三方/系统应用 | 被绑定后收到的文档流 | 读取文档、与打印机通信 |
| 文档提供者 | 提供者应用 | 被写入的 URI | 接收 PDF 字节并持久化 |
| `PdfRenderer` | 应用进程内 | 调用方提供的 FD | 解析并渲染页面 |
| `PdfDocument` | 应用进程内 | 调用方写入的内容 | 生成 PDF 字节 |

`BIND_PRINT_SERVICE` 是 `signature` 级权限，它限制的是**谁可以被系统绑定**，不是**谁能实现打印服务**。任何应用都可以声明一个 `PrintService` 子类并在 manifest 中注册；用户需要在系统设置里启用它，之后它才会出现在打印目标列表里。把「签名权限」误读为「只有系统能写打印服务」，是评估这一层时最常见的错误。

## 3. `onWrite` 的文件描述符语义

`PrintDocumentAdapter` 的核心是三个回调：

- `onStart()`：任务开始，可返回打印属性；
- `onLayout()`：计算页数，返回 `PrintDocumentInfo`；
- `onWrite()`：把指定页范围写成 PDF，写入给定的 `ParcelFileDescriptor`，完成后回调 `onWriteFinished()`；
- `onFinish()`：任务结束，释放资源。

关键细节：

1. **FD 是 spooler 创建的管道**。`RemotePrintDocument.WriteCommand` 用 `createPipe()` 建管道，把写端交给适配器，自己从读端读入 spool 文件。应用写完后必须关闭写端，否则 spooler 会一直等待。
2. **不能假设 FD 可 seek**。它是管道，不是普通文件；依赖 `seek()` 的 PDF 生成库会失败。
3. **写入是分页的**。`onWrite()` 可能被多次调用，每次针对不同页范围；适配器应只写请求的页。
4. **取消要及时**。`onWrite` 收到取消信号后应尽快停止并关闭 FD，否则 spool 文件会残留半成品。

服务侧读取是另一条管道：`PrintDocument.getData()` 会新建一个管道并返回只读读端，打印服务必须读完并关闭。两条管道之间由 spooler 的暂存文件连接——这就是为什么「打印内容」在系统里会短暂落盘，也是为什么 spool 目录的访问控制属于打印安全的一部分。

## 4. 「另存为 PDF」的真实路径

很多资料把「另存为 PDF」描述成一种打印服务，实际上它是 spooler 的一个内置 UI 流程：

1. `PrintActivity` 构造一个伪造的 `PrinterId`，描述里写着 `PDF printer`；
2. 用户在打印界面确认后，流程不走真实打印机，而是发 `ACTION_CREATE_DOCUMENT`；
3. 用户在 SAF 界面选择保存位置，得到 `content://` URI；
4. spooler 用 `openOutputStream(uri)` 打开输出流，把暂存内容写进去。

这条路径的安全含义有三点：

- **输出位置由用户决定**，应用无法指定；这降低了「打印内容被写到应用可读位置」的风险，但也意味着用户可能保存到云盘或共享目录。
- **URI 授权是 SAF 授予的**，不是应用转授的；提供者需要正确处理写入与覆盖。
- **文件名与 MIME 由系统与用户交互决定**，应用不应假定最终文件名。

对评估者而言，看到「打印泄露」类问题，应先确认泄露发生在暂存、服务读取还是最终保存，而不是笼统归因于「打印框架」。

## 5. PDF 的读与写：两条完全不同的技术栈

### 5.1 `PdfRenderer`：读，基于 pdfium

- API 21 引入；打开时要求 FD **可 seek**，否则抛异常；
- JNI 层使用 pdfium（`fpdfview.h`、`FPDF_*` 系列接口）；
- `Page.render()` 只接受 `ARGB_8888` 位图；
- API 35 增加 `write()`，可以把页面写回 PDF。

因为底层是 pdfium，PDF 解析的内存安全边界本质上是 pdfium 的边界。Android 公告中与 PDF 解析相关的条目通常修在 `external/pdfium`，而不是框架 Java 层。

### 5.2 `PdfDocument`：写，基于 Skia

- API 19 引入；用 `Page.canvas` 绘制，`writeTo(OutputStream)` 输出；
- JNI 层使用 Skia 的 `SkPDF::MakeDocument`，**不是** pdfium。

这意味着「Android 的 PDF 读写都用 pdfium」是错误说法：读用 pdfium，写用 Skia。两者共享的只是「PDF 文件格式」这一层，不共享解析器代码。评估时把写入侧的问题归到 pdfium，会找错组件。

### 5.3 `WebView` 打印

`WebView.createPrintDocumentAdapter()` 在 4.4 就已存在；5.0 废弃无参版本并增加接受文件名的重载。它返回的是一个适配器，把网页内容排版成 PDF 交给打印框架——**WebView 本身不是打印服务**，也不直接写文件。跨应用传递 `file://` URI 自 API 24 起会对 target N+ 抛 `FileUriExposedException`，所以打印链路里若涉及外部文件，应使用 `content://` 与 `openFileDescriptor(uri, "r")`。

## 6. 公开公告台账

| CVE | NVD 落点 / 类型 | 级别 | 修复 | 分类 |
|-----|----------------|------|------|------|
| [CVE-2019-2180](https://nvd.nist.gov/vuln/detail/CVE-2019-2180) | `ipp.c` 的 `ippSetValueTag` 越界读 | High | 2019-09 | System / ID |
| [CVE-2019-2228](https://nvd.nist.gov/vuln/detail/CVE-2019-2228) | `array.c` 的 `array_find` 越界读，spooler | High | 2019-12 | System / ID |
| [CVE-2023-45780](https://nvd.nist.gov/vuln/detail/CVE-2023-45780) | Print Service 后台启动 Activity | High | Android 14 | Framework / EoP |
| [CVE-2023-40093](https://nvd.nist.gov/vuln/detail/CVE-2023-40093) | 裁剪内容进入 PDF 输出；修在 `external/pdfium` | High（NVD 5.5） | 2024-02 | System / ID |

公告链接：

- 2019-09：https://source.android.com/security/bulletin/2019-09-01
- 2019-12：https://source.android.com/security/bulletin/2019-12-01
- Android 14：https://source.android.com/security/bulletin/android-14
- 2024-02：https://source.android.com/security/bulletin/2024-02-01

读表要点：

1. **前两条在 native 库**（IPP 解析与数组查找），不在 Java 打印框架；它们说明打印协议解析器本身就是攻击面。
2. **CVE-2023-45780 是权限提升**，落点是打印服务的后台 Activity 启动，与文档内容无关。
3. **CVE-2023-40093 修在 pdfium**，但影响描述是「裁剪内容进入 PDF 输出」，属于信息泄露而非解析崩溃；不要因为它修在 pdfium 就归为 RCE。

## 7. 成因模式

### 7.1 文件描述符生命周期错配

管道 FD 的所有权、关闭时机、取消路径，是打印适配器最容易出错的地方。写端未关会导致 spooler 挂起；读端未读完会导致服务侧阻塞；取消后继续写会产生半成品。安全影响通常体现为拒绝服务或临时文件残留。

### 7.2 解析器边界未校验

IPP 报文与 PDF 对象都包含长度、偏移、数组下标等外部可控字段。CVE-2019-2180 与 CVE-2019-2228 都是「按声明长度读取但未校验实际边界」。防御上，解析器必须把每个长度字段当成不可信输入。

### 7.3 打印服务的身份边界

打印服务是被系统绑定的第三方组件，但它运行在应用 uid 下。如果服务在处理任务时启动 Activity、访问文件或调用其他特权接口，就可能突破后台限制（CVE-2023-45780）。修法通常是显式检查调用来源与前台状态。

### 7.4 裁剪与输出的语义混淆

「裁剪」是页面呈现属性，不是内容删除。如果渲染管线把裁剪区域之外的内容仍然写入 PDF 对象流，就会出现「看起来被裁掉、实际仍可提取」的信息泄露（CVE-2023-40093 的描述指向这一类）。防御上，输出前应按最终可见区域重建内容，而不是只改显示参数。

### 7.5 URI 与 FD 混用

打印链路传 FD，「另存为 PDF」传 URI。把两者混用会导致权限模型错位：FD 不需要 URI 授权，URI 则需要 SAF 或显式授予。代码中若出现「把 URI 当路径打开」或「把 FD 当文件 seek」，往往同时是功能缺陷与安全缺陷。

### 7.6 临时文件清理不彻底

spool 文件保存了完整文档内容。任务取消、服务崩溃、打印服务被禁用时，若清理不彻底，敏感内容会残留。防御上，暂存目录应限制访问、任务结束即删除，并对异常退出做启动时清扫。

## 8. 版本加固时间线

| 版本 / API | 变化 |
|------------|------|
| 4.4 / 19 | 打印框架引入；`PrintDocumentAdapter`；WebView 可创建适配器 |
| 5.0 / 21 | `PdfRenderer` 引入；WebView 适配器增加文件名重载 |
| 7.0 / 24 | 跨应用 `file://` 抛 `FileUriExposedException` |
| 10 / 29 | 分区存储，文档访问转向 `content://` |
| 11 / 30 | target 30 强制分区存储 |
| 12 / 31 | 含 intent-filter 的组件必须显式声明 `exported` |
| 13 / 33 | `PrintManager.isPrintServiceEnabled()`，仅同 UID 可查 |
| 14 / 34 | 公告包含 Print Service 提权条目 |
| 15 / 35 | `PdfRenderer` 增加 `LoadParams` 与 `write()` |

趋势很清楚：**打印的输入输出都在从「路径」转向「URI + FD」**，而 PDF 的能力在从「只读渲染」扩展到「可写回」。每扩展一步，解析器与文件句柄的边界就需要重新审视。

## 9. 观测点与自查

### 9.1 应用侧

- `onWrite()` 是否在所有分支关闭 FD？取消时是否停止写入？
- 是否假设 FD 可 seek？是否假设只调用一次 `onWrite()`？
- 打印内容是否包含屏幕外/裁剪外的敏感数据？
- 使用 `PdfRenderer` 时是否确保 FD 可 seek？是否处理了损坏 PDF 的异常？
- 使用 `PdfDocument` 时是否限制输出大小，避免把超大位图写成巨型 PDF？

### 9.2 平台侧

- `dumpsys print` 查看打印任务、服务绑定与 spool 状态；
- 检查 spool 目录权限与残留文件；
- 审计打印服务的 manifest：是否声明了 `BIND_PRINT_SERVICE`、是否导出必要组件；
- 关注 pdfium 的版本与公告修复级别。

### 9.3 评估侧

- 先定位落点：打印服务、spooler、pdfium、Skia、SAF；
- 区分「文档内容泄露」「解析崩溃」「权限提升」「拒绝服务」；
- 不要把所有 PDF 相关 CVE 都归到 `PdfRenderer`——写入侧走的是 Skia。

## 10. 防御要点

1. **打印内容最小化**：在适配器里按最终可见区域重建页面，不要把隐藏字段、裁剪外内容原样写入。
2. **FD 生命周期显式管理**：用 `try/finally` 关闭；取消路径也要关闭；不要缓存 FD 跨回调使用。
3. **不信任 PDF 输入**：`PdfRenderer` 打开外部 PDF 前校验大小与来源；解析失败要降级，不要崩溃。
4. **打印服务最小权限**：不要在处理打印任务时启动 Activity 或访问无关文件；需要前台交互时走通知。
5. **暂存清理**：任务结束、取消、异常退出三条路径都要清理 spool 文件。
6. **URI 用 SAF 语义**：读写文档用 `openFileDescriptor` / `openOutputStream`，不要拼路径。
7. **关注解析器版本**：pdfium 的修复随安全公告发布，长期不更新的设备是主要风险面。

## 11. 常见误传纠正

### 误传一：Chrome 的 pdfium CVE 就是 Android 的 CVE

不准确。部分 CVE（如 2017-5036 一类）在 NVD 中带 Android CPE，但来源是 Chrome 的 CVE 管理流程，Android 公告并未收录。判断是否为 Android 平台问题，应以 Android Security Bulletin 是否列出为准。

### 误传二：打印服务只能由系统实现

错。`PrintService` 是公开基类，任何应用都可以实现并在 manifest 中注册；`BIND_PRINT_SERVICE` 是签名权限，限制的是「谁能被系统绑定」，不是「谁能实现」。

### 误传三：`PdfRenderer` 只能读

不准确。API 35 已为 `PdfRenderer` 增加 `write()`，可以把页面写回 PDF；它不再只是只读渲染器。

### 误传四：`PdfRenderer` 与 `PdfDocument` 都用 pdfium

错。`PdfRenderer` 的 JNI 使用 pdfium（`FPDF_*`），`PdfDocument` 使用 Skia 的 `SkPDF::MakeDocument`。两者是不同的技术栈。

### 误传五：「另存为 PDF」是一种 `PrintService`，WebView 打印是独立服务

错。「另存为 PDF」是 spooler 内置的假打印机 + SAF 输出；WebView 打印只是返回一个 `PrintDocumentAdapter`，本身不是服务。

## 12. 未核实与失效来源

- `android.printservice.PrintDocument` 与 `PrintJob#getDocument` 没有公开的开发者文档页，本文只能引用 AOSP 源码。
- `PdfRenderer` 是否强制 FD 以 `O_RDONLY` 打开未核实；文档只写「必须可 seek」。
- PDF 相关代码在 Android 16 已移出 `core/jni`，本文的组件归属基于 Android 14.0.0_r1 的源码结构。
- 未找到平台自身「`PdfRenderer` 解析导致 RCE」的公开公告；现有条目多为 pdfium 修复或信息泄露。
- 各厂商打印服务（如 Mopria）的实现差异未逐一核实。

## 13. 参考资料

- PrintManager（官方参考）：
  https://developer.android.com/reference/android/print/PrintManager
- PrintAttributes（官方参考）：
  https://developer.android.com/reference/android/print/PrintAttributes
- PrintService（官方参考）：
  https://developer.android.com/reference/android/printservice/PrintService
- PrintDocumentAdapter（官方参考）：
  https://developer.android.com/reference/android/print/PrintDocumentAdapter
- PdfRenderer（官方参考）：
  https://developer.android.com/reference/android/graphics/pdf/PdfRenderer
- PdfDocument（官方参考）：
  https://developer.android.com/reference/android/graphics/pdf/PdfDocument
- FileUriExposedException（官方参考）：
  https://developer.android.com/reference/android/os/FileUriExposedException
- Android 10 分区存储：
  https://developer.android.com/about/versions/10/privacy/changes#scoped-storage
- Android 11 存储变更：
  https://developer.android.com/about/versions/11/privacy/storage
- Android 12 组件导出：
  https://developer.android.com/about/versions/12/behavior-changes-12#exported
- PrintManager.isPrintServiceEnabled：
  https://developer.android.com/reference/android/print/PrintManager#isPrintServiceEnabled
- RemotePrintDocument（AOSP）：
  https://android.googlesource.com/platform/frameworks/base/+/main/packages/PrintSpooler/src/com/android/printspooler/model/RemotePrintDocument.java
- PrintDocument（AOSP）：
  https://android.googlesource.com/platform/frameworks/base/+/android-14.0.0_r1/core/java/android/printservice/PrintDocument.java
- PrintActivity（AOSP）：
  https://android.googlesource.com/platform/frameworks/base/+/main/packages/PrintSpooler/src/com/android/printspooler/ui/PrintActivity.java
- PdfRenderer JNI（AOSP）：
  https://android.googlesource.com/platform/frameworks/base/+/android-14.0.0_r1/libs/hwui/jni/pdf/PdfRenderer.cpp
- PdfDocument JNI（AOSP）：
  https://android.googlesource.com/platform/frameworks/base/+/android-14.0.0_r1/libs/hwui/jni/pdf/PdfDocument.cpp
- NVD 条目（本文台账）：
  https://nvd.nist.gov/vuln/detail/CVE-2019-2180
  https://nvd.nist.gov/vuln/detail/CVE-2019-2228
  https://nvd.nist.gov/vuln/detail/CVE-2023-45780
  https://nvd.nist.gov/vuln/detail/CVE-2023-40093

---

## 联系方式

咨询 Android 系统详情请联系 https://t.me/one00190

---

*本手册仅供安全学习与防御研究使用；所有机制表述、公告列值与 CVE 描述均取自公开一手来源并附链接，未能核实的部分已单列于「未核实与失效来源」而非写入正文。*
