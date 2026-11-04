---
layout: page
title: "Android 字体、文本整形与渲染栈攻击面手册"
description: "把「一段文字到像素」拆成字符处理／选字体／整形／字形加载／栅格化／排版六段：`FontManagerService` 只在 system_server 分发映射，Minikin/HarfBuzz/FreeType 运行在应用进程；讲清「字体更新 APK」不成立（字体文件 + PKCS#7 + fs-ve..."
date: 2026-11-04
permalink: /android-font-text-rendering/
---

> 每一帧界面里都有文字。文字从字符串到像素，要经过字符集与双向文本处理、字体族选择与回退、字形整形、栅格化与合成五个阶段；这条链上运行着 Minikin、HarfBuzz、FreeType 与 Skia，任何一个环节把「字体文件里的数字」当成可信值，都可能变成越界读、越界写甚至远程代码执行。字体又是一种特殊资产：它可以被系统更新、被应用内置、被下载、被用户替换，因而同时具备「高权限数据」与「低门槛输入」两种属性。这份手册沿「字体从哪来 → 怎么被选中 → 怎么被解析 → 怎么被渲染 → 怎么防」拆解机制与公开缺陷。只讨论公开且已修复的问题，不含可利用代码。

## 0. 使用方式与边界

这份手册面向三类读者：

- **应用开发者**：需要理解 `Typeface.Builder`、`createFromAsset/File`、downloadable fonts 的能力边界，以及为什么「内置一个字体」也可能引入解析风险。
- **系统与平台工程师**：需要区分 `FontManagerService`（system_server）与真正的整形/栅格化库（运行在应用进程），理解字体更新为什么不是「装一个 APK」。
- **安全评估者**：需要在看到「字体/文本」公告时，判断问题落在 FreeType、Minikin、HarfBuzz、资源解析还是字体管理服务，而不是把所有带 `font` 的 CVE 混为一类。

本文的三条自我约束：

1. 只引用公开且已修复的问题；CVE 编号、落点与描述以 NVD 为准，公告分类与级别以 Android Security Bulletin 为准。
2. 不写利用步骤、不写 PoC、不写可直接复现的攻击参数。
3. 查不到一手来源的说法一律进「未核实与失效来源」。

与已发布手册的分工：

- 《Android 图形与显示栈攻击面手册》讲合成、Surface 与 GPU；本文只讲文字从字符串到字形位图这一段。
- 《Android 媒体解码与容器解析攻击面手册》讲媒体容器；字体文件是另一类结构化二进制，本文只讲字体表解析。
- 《Android 供应链与构建完整性手册》讲构建与签名；本文只讲字体更新的签名与校验机制。

## 1. 一段文字的渲染旅程

把 `TextView.setText("你好")` 拆开，至少经过七个阶段：

| 阶段 | 主要位置 | 做的事 | 安全判断点 |
|------|----------|--------|------------|
| 1. 字符处理 | 应用进程（ICU / `AndroidBidi`） | 归一化、双向文本分段、脚本判定 | 输入长度、异常码点 |
| 2. 选字体 | Minikin | 按语言、字重、变体、回退链选 `Typeface` | 回退链是否可被外部影响 |
| 3. 整形 | HarfBuzz（`hb_shape`） | 把码点序列转成字形与位置 | 字体表数据是否可信 |
| 4. 字形加载 | FreeType | 解析 `glyf`/`CFF`/`COLR` 等表，生成轮廓/位图 | 表长度、偏移、嵌套深度 |
| 5. 栅格化 | Skia + FreeType 后端 | 轮廓转位图或直接合成 | 尺寸、变换、缓存 |
| 6. 排版 | Minikin / `Layout` | 断行、字距、行高、对齐 | 极端宽度、溢出 |
| 7. 合成 | `libhwui` / RenderThread | 把文字图层合入帧 | 图层边界、裁剪 |

关键点是：**系统服务只负责「告诉应用用哪个字体」，真正的解析发生在应用进程里**。`FontManagerService` 运行在 system_server，负责字体映射与更新；它通过共享内存把只读映射交给应用，应用进程内的 `libhwui` 再链到 `libminikin`、`libharfbuzz_ng`、`libft2`。因此一个字体解析漏洞的影响面是「所有渲染该字体的应用进程」，而不是「只有系统进程」。

第二个关键点是：**字体是数据，不是代码**。但字体解析器是用不安全的语言写的，字体表里又充满长度、偏移、数组下标和嵌套结构。于是「数据」获得了与「代码」相近的攻击价值——尤其是可变字体，它把大量参数化的几何数据放进一个文件里。

## 2. 对象模型与配置

### 2.1 关键组件

| 组件 | 所在层 | 作用 |
|------|--------|------|
| `Typeface` | 公开 SDK | 应用侧字体句柄 |
| `Typeface.Builder` | 公开 SDK（API 26） | 构造字体，支持变体轴 |
| `FontsContract` | 公开 SDK（API 26） | downloadable fonts 的提供者契约 |
| `FontManagerService` | AOSP 内部（system_server） | 字体映射、更新、按用户配置 |
| `Minikin` | native 库 | 选族、回退、度量、断行 |
| HarfBuzz（`libharfbuzz_ng`） | native 库 | 文本整形 |
| FreeType（`libft2`） | native 库 | 字形解析与栅格化后端 |
| Skia | native 库 | 绘制与 PDF 输出 |
| `FontResourcesParser` | 框架 Java | 解析 `res/font` 资源 |

### 2.2 配置与回退链

系统字体由几份配置共同决定：

- `/system/etc/fonts.xml`：字体族与回退顺序；
- `/system/etc/font_fallback.xml`：按语言/变体的回退规则；
- `/system/fonts/`：实际字体文件；
- `/product/etc/fonts_customization.xml`：产品定制。

`fonts.xml` 中的 `lang`、`variant`、`fallbackFor` 属性决定「这段文字该用哪个字体」。emoji 的处理是一个典型例子：`und-Zsye` 语言标签指向 `NotoColorEmoji.ttf`；Android 13 起支持 COLRv1 彩色矢量字体，而更早的 CBDT/CBLC 位图 emoji 仍然在字体表内。**emoji 不是图片解码，而是字体渲染**——这个区分对判断漏洞落点很重要。

Android 15 起 `fonts.xml` 被标记为弃用，变体轴只保留 `wght` 与 `ital`；这意味着自定义字体的表达能力在收窄，配置解析路径也在变化。

## 3. 字体从哪里来：四条来源与各自边界

### 3.1 系统字体

随系统镜像提供，位于 `/system/fonts` 与 `/product/fonts`。它们由构建流程打包，受分区只读保护。系统字体是「可信基线」，但仍可能被字体更新机制替换。

### 3.2 应用内置字体

通过 `resources/font` 或 `assets` 打包，用 `ResourcesCompat.getFont()`、`Typeface.createFromAsset()`、`Typeface.createFromFile()` 加载。**内置字体同样是外部数据**：如果字体文件来自第三方或用户上传，解析风险与应用无关地存在。

### 3.3 downloadable fonts

API 26 引入。应用通过 `FontsContract` 向字体提供者请求字体，提供者返回字体文件与证书。它**只为本应用取字**，不会替换系统字体；要替换系统字体需要 `UPDATE_FONTS` 权限与系统级机制。

### 3.4 系统字体更新

这是最容易被误解的一条。真实的更新机制是：

- 下发的是**字体文件**，不是 APK；
- 每个字体文件带 **PKCS#7 分离签名**；
- 文件落到 `/data/fonts/`，配置在 `/data/fonts/config/config.xml`；
- 使用 **fs-verity** 做完整性校验；
- 签名证书来自 `config_fontManagerServiceCerts` 配置。

因此「字体更新 APK」这一说法不成立：没有 APK 安装环节，也没有 APK 签名校验；它是「带签名的数据文件 + 只读完整性保护」。这也解释了为什么字体更新可以在不重启、不经过包管理器的情况下生效。

## 4. 解析栈：三个库、三种风险

### 4.1 Minikin：排版逻辑

Minikin 负责选族、回退、度量与断行，整形时调用 `hb_shape`。它的历史实现中有 `MinikinFontFreeType`，现行代码改为 `MinikinFontFactory` + `MinikinFontSkia`。Minikin 的风险主要在**逻辑与边界**：语言列表缓存、布局工具、断行计算都可能因为极端输入出现越界或崩溃。

### 4.2 HarfBuzz：整形

HarfBuzz 把码点序列与字体表结合，输出字形与位置。它的输入同时来自「文本」与「字体」，两者都可能是攻击者可控的。公开公告中直接标注 HarfBuzz 的条目很少，但这不等于它没有缺陷——更常见的情况是缺陷被归入上层组件或统一记为字体解析问题。

### 4.3 FreeType：字形解析

FreeType 是 Skia 的字形后端（`SkFontHost_FreeType.cpp`），负责解析 `glyf`、`CFF`、`COLR`、`SVG` 等表。它是字体攻击面中最核心的部分：表结构复杂、历史长、被大量格式共享。近年与 Android 相关的字体内存漏洞，多数落在 FreeType 的表解析路径上。

一个常见的误解是「只要是最新的 FreeType 就安全」。实际上 Android 使用的是平台内置版本，修复随安全公告合入；应用无法替换系统 FreeType。评估时应以设备的补丁级别为准，而不是看上游版本号。

## 5. 公开公告台账

| CVE | NVD 落点 | 类型 / 级别 | 修复 | 分类 |
|-----|----------|-------------|------|------|
| [CVE-2025-27363](https://nvd.nist.gov/vuln/detail/CVE-2025-27363) | FreeType 可变字体越界写 | RCE / High | 2025-05 | System |
| [CVE-2023-21287](https://nvd.nist.gov/vuln/detail/CVE-2023-21287) | 类型混淆（FreeType） | RCE / High | 2023-08 | Framework |
| [CVE-2023-20958](https://nvd.nist.gov/vuln/detail/CVE-2023-20958) | FreeType `ttcolr.c` 的 `read_paint` | ID / High | 2023-03 | Framework |
| [CVE-2022-20472](https://nvd.nist.gov/vuln/detail/CVE-2022-20472) | Minikin `LocaleListCache.cpp` | RCE / Critical | 2022-12 | Framework |
| [CVE-2021-0313](https://nvd.nist.gov/vuln/detail/CVE-2021-0313) | Minikin `LayoutUtils.cpp` | DoS / Critical | 2021-01 | Framework |
| [CVE-2020-15999](https://nvd.nist.gov/vuln/detail/CVE-2020-15999) | NVD 只写 Chrome | RCE / Moderate | 2021-01 | Framework |
| [CVE-2018-9410](https://nvd.nist.gov/vuln/detail/CVE-2018-9410) | `FontUtils.cpp` 的 `analyzeAxes` | ID / High | 2018-07 | Framework |

公告链接：

- 2025-05：https://source.android.com/security/bulletin/2025-05-01
- 2023-08：https://source.android.com/security/bulletin/2023-08-01
- 2023-03：https://source.android.com/security/bulletin/2023-03-01
- 2022-12：https://source.android.com/security/bulletin/2022-12-01
- 2021-01：https://source.android.com/security/bulletin/2021-01-01
- 2018-07：https://source.android.com/security/bulletin/2018-07-01

读表要点：

1. **落点分成两组**：FreeType 表解析（越界写、类型混淆、越界读）与 Minikin 排版逻辑（缓存、布局）。把两者统称「字体漏洞」会掩盖修复方向。
2. **CVE-2020-15999 的 NVD 描述只提 Chrome**，但 Android 公告在 2021-01 收录了它；判断是否为 Android 平台问题应以公告为准。
3. **级别与影响类型不必一致**：同样是 Critical，CVE-2022-20472 是 RCE，CVE-2021-0313 是 DoS。

## 6. 成因模式

### 6.1 表长度与偏移未校验

字体文件由多个表组成，表头声明长度与偏移。解析器若按声明值读取而不校验文件实际大小，就会出现越界读；若按声明值写入缓冲区而不校验目标容量，就会出现越界写。CVE-2025-27363 的可变字体越界写属于这一类。

### 6.2 可变字体参数化带来的组合爆炸

可变字体用变体轴（如 `wght`、`wdth`）对几何数据做插值。攻击者可以提供极端轴值，让解析器走到平时不会执行的路径。防御上要对轴值做范围裁剪，并对插值结果做边界检查。

### 6.3 彩色字体表的嵌套结构

COLR/CPAL 表用「绘制图元」描述彩色字形，图元可以嵌套引用。`read_paint` 一类函数如果未限制嵌套深度或引用范围，就会在递归/循环中越界（CVE-2023-20958）。这类问题在 COLRv1 普及后更值得关注。

### 6.4 类型混淆

同一段内存被当作不同结构解释，通常源于标志位与实际数据不匹配。CVE-2023-21287 被描述为 FreeType 的类型混淆。修法是把「判别字段」与「结构选择」绑定为一次校验，而不是分散在各分支。

### 6.5 缓存键不完整

Minikin 的语言列表缓存、字形缓存都以「输入组合」为键。如果键漏掉了影响结果的维度（如变体轴、语言变体），就会命中错误条目，轻则显示错误，重则越界（CVE-2022-20472）。防御上，缓存键必须覆盖所有影响输出的输入。

### 6.6 极端布局参数

`LayoutUtils` 一类工具在计算断行、宽度、行高时，若遇到极端字号、超长文本、零宽字符组合，可能产生负值或溢出（CVE-2021-0313）。这类问题通常表现为崩溃（DoS），但在特定路径上也可能影响内存安全。

### 6.7 更新链的信任假设

字体更新依赖签名与 fs-verity。若实现中把「配置里的证书」当成唯一信任源，而配置本身可被更低权限修改，就会形成信任链缺口。反之，若校验顺序错误（先落盘后验签），也会留下窗口。

## 7. 版本加固时间线

| 版本 / API | 变化 |
|------------|------|
| 8 / 26 | downloadable fonts；`Typeface.Builder` |
| 9 / 28 | 无字体相关变更条目 |
| 12 / 31 | `FontManager` 支持免 OTA 更换字体（AOSP 12 起有对应服务） |
| 13 / 33 | COLRv1 彩色矢量 emoji |
| 14 / 34 | 字体非线性缩放至 200% |
| 15 / 35 | `fonts.xml` 弃用；变体轴仅保留 `wght` / `ital` |

趋势是**字体系统从「编译期固定」转向「运行时可更新」**：downloadable fonts 让应用按需取字，`FontManager` 让系统免 OTA 换字，彩色字体让字体文件承载更多结构。能力越强，解析器的输入面越大。

## 8. 观测点与自查

### 8.1 应用侧

- 是否加载来自外部（用户上传、网络下载、第三方 SDK）的字体文件？加载前是否校验来源与大小？
- 使用 `Typeface.Builder` 设置变体轴时，是否把轴值限制在合理范围？
- downloadable fonts 是否校验提供者与证书？是否处理了提供者返回异常字体的情况？
- 是否在 `try/catch` 中处理字体解析失败，避免启动崩溃？

### 8.2 平台侧

- 检查 `/data/fonts/config/config.xml` 的签名与 fs-verity 状态；
- 核对 `config_fontManagerServiceCerts` 是否只包含预期证书；
- 审计 `fonts.xml` / `font_fallback.xml` 的变更，尤其是产品定制目录；
- 关注系统补丁级别与 FreeType/Minikin 的公告修复。

### 8.3 评估侧

- 先定位落点：FreeType、Minikin、HarfBuzz、资源解析、字体管理服务；
- 区分「解析崩溃」「信息泄露」「RCE」「更新链绕过」；
- 不要因为公告标题写「Font」就假定是应用层 API 问题。

## 9. 防御要点

1. **把字体当不可信输入**：外部字体先做格式与大小校验，再交给渲染栈。
2. **限制变体轴范围**：不要直接把用户输入映射为轴值；按字体声明的最小/最大值裁剪。
3. **避免超大字体与超长文本组合**：对字体文件大小、文本长度、字号设置上限。
4. **失败可降级**：字体解析失败时回退到系统字体，而不是让界面崩溃。
5. **更新链校验顺序**：先验签、再落盘、最后启用；启用前确认 fs-verity 状态。
6. **最小化自定义字体**：能使用系统字体时不要引入第三方字体；确需引入时固定版本并记录来源。
7. **关注补丁级别**：字体漏洞的修复随系统公告到达，长期不更新的设备是主要风险。

## 10. 常见误传纠正

### 误传一：任意 FreeType CVE 都算 Android 的漏洞

不准确。只有 Android 公告列出的条目才应计入平台风险；例如 CVE-2020-15999 的 NVD 描述只提 Chrome，但 Android 在 2021-01 公告中收录了它。判断标准是公告，不是上游项目。

### 误传二：downloadable fonts 可以替换系统字体

错。它只为本应用获取字体；替换系统字体需要 `UPDATE_FONTS` 权限与系统级更新机制。

### 误传三：字体更新是一个 APK

错。字体更新下发的是字体文件，带 PKCS#7 分离签名，并用 fs-verity 保护完整性；没有 APK 安装与 APK 签名环节。

### 误传四：可变字体只是运行时代码

不准确。可变字体是「带变体轴的字体数据」，不是可执行代码；但它的解析器存在内存漏洞（如 CVE-2025-27363 的越界写）。

### 误传五：emoji 是位图解码，不属于字体栈

错。emoji 通过字体渲染：`und-Zsye` 指向 `NotoColorEmoji.ttf`，Android 13 起使用 COLRv1 彩色矢量表，更早的 CBDT/CBLC 位图也仍在字体表内。

## 11. 未核实与失效来源

- Android 9 是否包含字体更新能力存在矛盾：Android 9.0 页面没有该条目，而字体文档标注为 Android 12；本文按「AOSP 12 起有对应服务」表述，历史归属未完全核实。
- `FontManager` 的公开 API level 未核实：公开页面 404，源码中仅为 `@SystemApi`。
- HarfBuzz 与 `FontManagerService` 在 2019–2026 公告中的独立条目检索为 0 命中，但这不代表不存在，可能被归入其他组件。
- 各厂商对自定义字体与字体商店的实现差异未逐一核实。

## 12. 参考资料

- Typeface.Builder（官方参考）：
  https://developer.android.com/reference/android/graphics/Typeface.Builder
- downloadable fonts（官方指南）：
  https://developer.android.com/guide/topics/ui/look-and-feel/downloadable-fonts
- Android 13 彩色矢量字体：
  https://developer.android.com/about/versions/13/features#color-vector-fonts
- Android 14 非线性字体缩放：
  https://developer.android.com/about/versions/14/features#non-linear-font-scaling
- 自定义字体回退（官方文档）：
  https://source.android.com/docs/core/fonts/custom-font-fallback
- libhwui 构建配置（AOSP）：
  https://android.googlesource.com/platform/frameworks/base/+/main/libs/hwui/Android.bp#115
- FontCollection（Minikin，AOSP）：
  https://android.googlesource.com/platform/frameworks/minikin/+/main/libs/minikin/FontCollection.cpp#65
- MinikinSkia（AOSP）：
  https://android.googlesource.com/platform/frameworks/base/+/main/libs/hwui/hwui/MinikinSkia.cpp#31
- SkFontHost_FreeType（AOSP）：
  https://android.googlesource.com/platform/external/skia/+/main/src/ports/SkFontHost_FreeType.cpp
- FontResourcesParser（AOSP）：
  https://android.googlesource.com/platform/frameworks/base/+/main/core/java/android/content/res/FontResourcesParser.java#236
- SystemFonts（AOSP）：
  https://android.googlesource.com/platform/frameworks/base/+/main/graphics/java/android/graphics/fonts/SystemFonts.java#56
- FontManagerService（AOSP）：
  https://android.googlesource.com/platform/frameworks/base/+/main/services/core/java/com/android/server/graphics/fonts/FontManagerService.java#69
- UpdatableFontDir（AOSP）：
  https://android.googlesource.com/platform/frameworks/base/+/main/services/core/java/com/android/server/graphics/fonts/UpdatableFontDir.java#568
- NVD 条目（本文台账）：
  https://nvd.nist.gov/vuln/detail/CVE-2025-27363
  https://nvd.nist.gov/vuln/detail/CVE-2023-21287
  https://nvd.nist.gov/vuln/detail/CVE-2023-20958
  https://nvd.nist.gov/vuln/detail/CVE-2022-20472
  https://nvd.nist.gov/vuln/detail/CVE-2021-0313
  https://nvd.nist.gov/vuln/detail/CVE-2020-15999
  https://nvd.nist.gov/vuln/detail/CVE-2018-9410

---

## 联系方式

咨询 Android 系统详情请联系 https://t.me/one00190

---

*本手册仅供安全学习与防御研究使用；所有机制表述、公告列值与 CVE 描述均取自公开一手来源并附链接，未能核实的部分已单列于「未核实与失效来源」而非写入正文。*
