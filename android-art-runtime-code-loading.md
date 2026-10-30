---
layout: page
title: "Android ART 运行时与代码加载攻击面手册"
description: "把「执行的代码」与「发布的代码」分开：dex→vdex（提取的 dex + 校验和 + `VerifierDeps` + 类型查找表）→odex→`.art` 的四产物链、编译过滤器自 Android 8 起四种而 `quicken` 已移除、boot image 自 Android 12 固定 `sp..."
date: 2026-10-30
permalink: /android-art-runtime-code-loading/
---

> 在 Android 上，「代码」不是磁盘上的文件，而是**四个互相引用的产物**：dex 是源码级表示，vdex 是 dex 加上校验与验证依赖，odex 是编译输出，art 是启动时预先构造的对象镜像。应用启动时，运行时把这四者按一套既定顺序映射进内存，再在内存里生成可执行页。这条链上的每一步都涉及「谁写的、谁验的、能不能改」，而绝大多数关于「Android 代码加载安全」的讨论只停在这条链的最后一环——`System.loadLibrary`。本手册只讨论公开且已修复问题的机制与自查方法，不含可利用代码。

---

## 0. 这篇手册解决什么问题

已发布的手册里，《APK 签名与安装校验》讲的是**包的身份**（v1–v4 签名、rotation、安装标志），《框架 IPC 与组件导出》讲的是**组件暴露**。两者都回答「这个包能不能进来」，但都没有回答另一个问题：**进来之后，这台机器上会被执行的是什么**。

这个问题在 Android 上尤其重要，因为 ART 做了两件其他平台很少做的事：

1. **它会在设备上把字节码编译成机器码**（AOT），因此「执行的代码」与「发布的代码」不是同一份产物。
2. **它把运行时可独立更新**（Mainline 模块），因此「同一个应用」在不同设备、不同时间可能跑在不同的编译器版本上。

这两件事让 ART 成为一个**独特的攻击面**：它既是「代码的翻译者」，又是「代码的加载者」，还是「可被远程更新的系统组件」。

---

## 1. 产物链：dex → vdex / odex / art

### 1.1 四种产物的职责

官方对 vdex 的定义很具体：

> "VDEX files contain extracted DEX files. The VdexFile class maps the file to memory and provides tools for accessing its individual sections."

其分节枚举是：

```cpp
kChecksumSection     = 0,
kDexFileSection      = 1,
kVerifierDepsSection = 2,
kTypeLookupTableSection = 3,
```

这四节说明了 vdex 的真实内容：**dex 本体 + 校验和 + 验证依赖（verifier deps）+ 类型查找表**。所以把 vdex 说成 "verified dex" 是不准确的——「验证」留下的是 `VerifierDeps` 这一节（记录「验证时依赖了哪些类型/方法」），而不是「已验证」这个布尔结论。

OAT/odex 的定义是：

> "Runtime representation of the OAT file format which holds compiler output."
> "In the process of loading OAT, the class also loads the associated VDEX file with the input DEX files."

第二句很关键：**odex 与 vdex 是成对加载的**，odex 里存的是编译产物，dex 本体在 vdex 里。这解释了为什么单独替换其中一个是不可行的——引用关系会断。

第四种产物是 **app image（`.art`）**：启动时预设好的对象图（类、方法、字符串等已解析对象的镜像），以 `kAppImageClassLoader` 这类特殊根标记类加载器，dex2oat 侧对应 `--app-image-fd` / `--app-image-file`。

| 产物 | 内容 | 由谁产生 | 在哪里 |
|---|---|---|---|
| `classes.dex` | 源码级字节码 | 开发者/构建 | APK 内 |
| `*.vdex` | 提取的 dex + 校验 + 验证依赖 + 类型查找表 | dex2oat | `/data/dalvik-cache/` 或只读分区 |
| `*.odex` | AOT 编译输出（OAT 格式） | dex2oat | 同上 |
| `*.art` | 启动对象镜像 | dex2oat | 同上 |
| `primary.prof` | 运行时收集的热点 profile | 运行时 | `/data/misc/profiles/cur/<userId>/<pkg>/` |

### 1.2 编译过滤器

过滤器决定了「编译到什么程度」，官方口径是：

> "Starting in Android 8, there are four officially supported filters:"（`verify` / `quicken` / `speed` / `speed-profile`）

其中 `quicken` 标注 "(Android 11 or lower)"——即**已被移除**。`speed-profile` 的定义是：

> "Runs DEX code verification, AOT-compiles methods listed in the profile, and optimizes class loading for classes in the profile."

现行做法是按场景选择，用 `pm.dexopt.<reason>` 这一族属性（Android 7 起），而不是单一的全局过滤器。而 `dalvik.vm.dex2oat-filter` 的官方说明带一个明确的截止点：

> "`dalvik.vm.dex2oat-filter` (until Android 6): The compiler filter for everything else."
> "`pm.dexopt.<reason>` (since Android 7)"
> "Since Android 12, the compiler filter for boot images is always `speed-profile` and cannot be changed."

**这三句合起来纠正了一个高频错误**：把 `dalvik.vm.dex2oat-filter` 当作「现行全局过滤器」。它在 Android 6 之后就被 `pm.dexopt.<reason>` 取代了；而 boot image 自 Android 12 起固定 `speed-profile`，**不可配置**——所以「改属性来给 boot image 换过滤器」这种操作在新版本上根本无效。

### 1.3 JIT 与 profile

> "JIT is enabled for any application that is not compiled according to the `speed` compilation filter."
> "Since Android 14, JIT profiles are always enabled and cannot be turned off."

第二条是这几年的重要变化：**Android 14 起 profile 无法关闭**。这意味着：

- 设备上一定会存在 profile 文件（路径由 `Environment` 拼装：`buildPath(getDataDirectory(), "misc", "profiles", "cur", String.valueOf(userId))`，即 `/data/misc/profiles/cur/<userId>/<pkg>/`，文件名以 `primary.prof` 结尾）。
- 「关闭 profile 以减小攻击面」不再是可选项。
- profile 的内容会**影响后续 AOT 编译哪些方法**——于是 profile 从「性能数据」变成了「决定执行什么代码的输入」。

最后这一点是本节的核心安全含义：**如果 profile 可被篡改，它就能影响编译决策**。这不是说存在已知的可用攻击，而是说这条数据流的信任级别必须明确——它来自应用自己的运行数据，属于「应用可影响」的输入，不应被当作可信配置。

### 1.4 运行时可独立更新

> "The ART module (`com.android.art`) ships as an APEX since it is a critical system component."
> "End-user devices can receive Mainline updates from the Android Google Play system update feature (which runs on the Google Play Store infrastructure) or through a partner-provided over-the-air (OTA) mechanism."

三条推论：

1. **ART 的补丁节奏与框架解耦。** 一个运行时缺陷可以在不改系统版本的情况下修掉——这对防御方是好事。
2. **同一 Android 版本在不同设备上可能跑不同 ART 版本。** 资产清单里记录「Android 版本」不足以描述运行时状态，需要额外记录模块版本。
3. **「Android 12 引入 ART APEX」这句在官方文档中没有逐字出现**（文档只说 "ships as an APEX"），属于**未核实**，正文不写具体引入版本。

---

## 2. 代码加载：三层入口

Android 上「让代码跑起来」有三条独立入口，它们的信任模型完全不同。

### 2.1 Java 层：`DexClassLoader` / `InMemoryDexClassLoader` / `PathClassLoader`

- `PathClassLoader`：加载 APK/jar 路径，系统用它加载已安装应用。
- `DexClassLoader`：从任意路径加载 dex/jar，是插件化框架的基础。
- `InMemoryDexClassLoader`：从内存中的 `ByteBuffer` 加载，不需要落盘。

安全约束在 Android 10 与 Android 14 两次收紧：

> **Android 10**："Apps should load only the binary code that's embedded within an app's APK file."
> "Untrusted apps that target Android 10 cannot invoke `execve()` directly on files within the app's home directory."
> "…cannot in-memory modify executable code from files which have been opened with `dlopen()` … because the library cannot have been mapped `PROT_EXEC` through a writable file descriptor."

> **Android 14**："If your app targets Android 14 (API level 34) or higher and uses Dynamic Code Loading (DCL), all dynamically-loaded files must be marked as read-only. Otherwise, the system throws an exception."

AOSP 侧对应的开关是 `ENFORCE_READ_ONLY_JAVA_DCL`，注释为 "Enforce the file passed to open DexFile to be set as read-only for apps targeting U+."

**这里必须精确**：流传最广的一句「Android 10 起禁止 `dlopen` 应用可写目录里的文件」——**未核实**。官方原文讲的是两件事：禁止对家目录内文件直接 `execve()`；禁止「通过可写 fd 映射 `PROT_EXEC` 后再改内存」。**只读强制**是后来的事：Android 14 针对 Java 侧动态加载，Android 17 扩展到 native 侧。

Android 17 的那条扩展是：

> "All native files loaded using `System.load()` must be marked as read-only. Otherwise, the system throws `UnsatisfiedLinkError`."

这条把「Java 动态加载」与「native 库加载」两条路径的约束**对齐**了，是理解当前模型的关键时间点。

### 2.2 Native 层：`System.loadLibrary` 与 linker namespace

`System.loadLibrary` 的名字解析不是「在磁盘上找 `.so`」，而是**在调用方所属的 linker namespace 里找**。官方对命名空间的描述是：

> "The namespace for the app native libraries has a link that allows only the library names specified in `public.libraries.txt`."
> "In Android 7.0 and higher, system libraries are separated from app libraries."

配套事实：

- `/vendor/etc/public.libraries.txt` 用于 "libraries from silicon vendors"；`/system/etc/public.libraries-COMPANYNAME.txt` 用于厂商自有库。
- SDK 31+ 起还要在 manifest 里声明 `<uses-native-library>`，否则 "apps can't access native shared libraries that don't appear in the app manifest."

**这条约束的安全含义**：应用**不能**通过 `dlopen` 拿到任意平台内部库（例如直接 `dlopen("libbinder.so")` 去调内部接口）。它能拿到的只有白名单里那些「为应用设计」的库。这是 Android native 侧的「API 边界」，与 Java 侧的 hidden API 限制是同一思路的两套实现。

关于 `android_dlopen_ext`：它是**公开 NDK API**（头文件 `<android/dlext.h>`），官方描述为 "Opens the given library. The `__filename` and `__flags` arguments are the same as for `dlopen(3)`, with the Android-specific flags supplied via the `flags` member of `__info`."。真正内部专用的是命名空间 flag：`ANDROID_DLEXT_USE_NAMESPACE`，注释写明 "This flag is for internal use only (since there is no NDK API for namespaces)."

**命名精度**：AOSP `library_namespaces.cpp` 中命名空间名是常量 `kClassloaderNamespaceName`（拼接 `-%d`）；字面串 `classloader-namespace` 在已取到的一手文件中未出现，属**未核实**。

### 2.3 JNI 注册：`JNI_OnLoad` 与 `RegisterNatives`

`JNI_OnLoad` 是库被加载时由运行时调用的入口，用于返回 JNI 版本并做初始化；`RegisterNatives` 用于把 native 方法与 Java 方法显式绑定，避免按名字反射查找。

安全上值得注意的几点（均为机制层面，不涉及具体漏洞）：

1. **`JNI_OnLoad` 在库加载时自动执行。** 所以「加载一个库」本身就等于「执行它的初始化代码」——加载路径的信任级别必须等于代码执行的信任级别。
2. **`RegisterNatives` 决定「哪个 native 函数响应哪个 Java 方法」。** 绑定关系在运行时建立，静态分析 `.so` 的导出表看不到完整映射，这给审计带来天然困难。
3. **JNI 调用不检查参数语义。** `jint` 就是 32 位整数，`jobject` 就是引用；Java 侧的边界检查不会自动传导到 native 侧。这一条是《HAL 与 Treble 边界》里「类型契约 ≠ 语义契约」的同构问题，只是发生在进程内。

---

## 3. Hidden API：限制的边界在哪

官方表述：

> "Starting in Android 9 (API level 28), the platform restricts which non-SDK interfaces your app can use."
> "You can enable access to non-SDK interfaces on development devices by using adb commands to change the API enforcement policy."

实现细节可以从 AOSP 读到：

- 清单文件是 `hiddenapi-flags.csv`（记录每个成员的灰度标记）。
- 调试设置是 `adb shell settings put global hidden_api_policy N`，其中 0 = 关闭检测、1 = 放行但告警、2 = 拦截。
- `setHiddenApiExemptions(String[] signaturePrefixes)` 存在，但**不是公开 API**，标注为 `@SystemApi(client = MODULE_LIBRARIES)`。

最关键的一点是**豁免逻辑里没有 debuggable 分支**：

```
if (isSignedWithPlatformKey())      → 豁免
else if (isSystemApp() || isUpdatedSystemApp()) → 按规则豁免
else                                → 不豁免
```

所以「hidden API 在 debuggable 构建上默认可绕过」是**误传**。可用的绕过手段是改 `hidden_api_policy` 这个**调试设置**——它是显式的、需要 adb 或系统权限的动作，不是「debuggable 自动获得的能力」。

**这对防御方的意义**：hidden API 限制的目标不是「让逆向不可能」，而是「让依赖内部接口的应用在升级时可控地失败」。把它当安全边界会高估它；把它当纯兼容性机制会低估它对「静默获取系统能力」的抑制作用。准确说法是：**它是一条可被显式绕过的兼容性护栏，绕过需要设备级操作权限。**

---

## 4. 成因分析：运行时这一层为什么出问题

### 4.1 「代码来源」与「代码身份」分离

APK 签名保证的是**包的身份**，但不保证「设备上实际执行的机器码由这个包生成」。因为机器码是 dex2oat 在设备上产生的中间产物。这条缝里有三类问题：

1. **产物被替换**：如果 `/data/dalvik-cache` 下的产物可被第三方改写，运行时加载的就是被改过的代码。防护依赖该目录的 SELinux 标签与文件权限（`dalvik-cache` 属 system 域可写、应用不可写）。
2. **产物过期未失效**：dex 变更后产物必须重新生成。失效判据（校验和、时间戳）一旦被绕过，就会加载旧产物——这类问题表现为「更新后行为不变」，很难排查。
3. **profile 影响编译决策**：见 §1.3。

### 4.2 编译器本身是攻击面

`dex2oat` 是一个**读入不可信输入（dex）、输出可执行产物**的程序。这个位置天然危险：

- 输入由开发者（或攻击者）控制，格式复杂（dex 有大量可变长结构、注解、调试信息）。
- 输出是可执行代码，且会被后续加载。
- 它通常在安装时以较高权限运行。

公开案例 `CVE-2021-0511` 正是这一类：官方描述为 "In Dex2oat of dex2oat.cc, there is a possible way to inject bytecode into an app due to improper input validation."（Android runtime 分区，EoP High，A-178055795）。**这是本文唯一直接落在 ART 组件上的案例**，价值在于它说明：ART 的缺陷类型不是「JIT 喷射」这类想象出来的东西，而是**输入校验缺失导致产物被污染**。

### 4.3 可执行内存的生成

JIT 与 AOT 都要把字节码变成可执行页。相关的结构性风险是：

- 页需要在某一时刻同时可写与可执行，或经历「写→改权限→执行」的切换；
- 切换窗口、权限粒度（整页 vs 部分）、以及编译缓存的存放位置，都是设计变量；
- 现代 Android 的加固方向是 **W^X 与只读代码页**，并在部分场景引入更强机制。

这里不做机制细节的断言（不同版本实现差异大，且部分内容未逐字核实），只给结论：**评估设备时，「JIT 是否可关闭」「编译产物是否只读」「是否存在可写可执行映射」是三个可观测的问题**，比讨论抽象的「JIT 喷射」更有意义。

### 4.4 动态加载的业务需求与安全约束冲突

插件化、热修复、按需下发代码——这些业务形态都要求「运行时加载外来代码」，而 Android 的安全模型在持续收紧这条路径（Android 10 禁 `execve`、禁可写 fd 映射；Android 14 强制 Java 侧产物只读；Android 17 扩展到 native）。

冲突的结果通常是三种折中：

1. **把可执行内容放进 APK 内部**（合规但失去「随时更新」能力）；
2. **只下发数据、不下发代码**（把可执行内容解释成数据，例如脚本解释器——但解释器本身成了攻击面）；
3. **依赖系统级更新通道**（Mainline/Play 更新）替代应用内自更新。

从防御角度看，第 3 条是唯一「不削弱平台模型」的路径，这也是 ART 被做成 Mainline 模块的深层原因。

### 4.5 运行时可被单独更新的双刃

§1.4 讲过好处。代价是：

- **版本矩阵爆炸**：设备数 × 模块版本数，任何缺陷的影响面都难以一次说清。
- **回滚语义复杂**：模块更新失败/回滚时，运行时行为会回退到旧版本。
- **审计基线漂移**：上一次盘点时「不存在」的 API/行为，可能因为模块更新而出现。

对应做法是把模块版本纳入资产基线（见 §6）。

---

## 5. 公开案例台账（逐条核实归属）

ART 主题下的 CVE 归属错误极为严重。逐条核对如下：

| 编号 | 实际组件（原文要点） | 与 ART 的关系 |
|---|---|---|
| CVE-2021-0511 | "In Dex2oat of dex2oat.cc … inject bytecode into an app due to improper input validation"（Android runtime 分区，EoP High，A-178055795） | **直接相关**：dex2oat |
| CVE-2020-0421 | "In appendFormatV of String8.cpp … out of bounds write"（libutils） | **无关**：libutils 字符串格式化 |
| CVE-2019-2198 | "In Download Provider … SQL injection" | **无关**：Download Provider |
| CVE-2021-39624 | "In PackageManager … permanent denial of service due to resource exhaustion" | **无关**：包管理 |
| CVE-2022-20452 | "In initializeFromParcelLocked of BaseBundle.java … method arbitrary code execution due to a confused deputy" | **无关**：Binder 序列化 |
| CVE-2023-20963 | "In WorkSource, there is a possible parcel mismatch"（EoP High，有在野利用迹象） | **无关**：WorkSource |
| CVE-2024-0044 | "In createSessionInternal of PackageInstallerService.java … run-as any app" | **无关**：包安装 |
| CVE-2024-43093 | "In shouldHideDocument of ExternalStorageProvider.java … incorrect unicode normalization" | **无关**：存储路径 |

**这张表的关键结论**：八个编号里只有一个真的属于 ART。这意味着两件事：

1. 「ART 漏洞很多」的印象多半来自错误归属；实际公开的 ART 组件缺陷**数量很少**，因为它的解析器代码相对收敛、模糊测试覆盖较好。
2. 讨论 ART 安全时，**成因分类比 CVE 枚举更有价值**——尤其 §4.2（编译器即攻击面）这一条，它的真实案例虽少，但结构风险长期存在。

---

## 6. 防护技术与自查

### 6.1 设计层

| 措施 | 做法 | 挡住什么 |
|---|---|---|
| 代码随包分发 | 可执行内容一律放 APK 内，用 Play/Mainline 更新 | 动态加载带来的整条风险链 |
| 只读产物 | 动态加载的文件标记只读（Android 14+ 强制，仍应主动做） | 加载后被篡改 |
| 校验后再加载 | 对下载的代码做签名校验 + 哈希校验，且校验与加载使用同一个 FD/路径 | 检查与使用之间的替换 |
| 不使用隐藏 API | 依赖公开 API；确需系统能力时走 SDK/系统签名 | 升级即崩、以及「静默获得系统能力」 |
| 谨慎用 native | 只链接 `public.libraries.txt` 白名单内的库 | 依赖非公开 ABI |

### 6.2 实现层

1. **动态加载前校验文件属性**：路径是否在预期目录、是否符号链接、所有者与权限、大小上限。
2. **不要用「字符串路径」二次打开**：校验用 `open` 得到的 FD，加载也用这个 FD（`InMemoryDexClassLoader` 读同一 FD），避免校验与加载之间的替换窗口。
3. **native 库加载失败要显式处理**：`UnsatisfiedLinkError` 与「库不存在」含义不同，混在一起会让排查失去线索。
4. **JNI 边界做参数校验**：不要把 Java 侧的类型检查当作 native 侧的保证（§2.3）。
5. **profile 目录不当作可信输入**：不要把 profile 内容用于任何安全判定。

### 6.3 验证层与自查清单

```bash
# 1. 运行时模块版本（ART 是 Mainline 模块，版本与系统版本解耦）
adb shell dumpsys package com.google.android.art 2>/dev/null | head -20
adb shell pm list packages | grep -i "com.android.art\|com.google.android.art"
adb shell getprop | grep -iE "art|apex" | head -20

# 2. 编译相关属性（注意 dex2oat-filter 已是历史项）
adb shell getprop | grep -iE "dalvik.vm|pm.dexopt" | head -30

# 3. profile 现状（Android 14 起恒存在）
adb shell ls -l /data/misc/profiles/cur/0/ 2>/dev/null | head

# 4. hidden API 策略
adb shell settings get global hidden_api_policy

# 5. 应用可见的 public native 库
adb shell cat /vendor/etc/public.libraries.txt
adb shell cat /system/etc/public.libraries*.txt 2>/dev/null

# 6. 编译产物位置与权限
adb shell ls -ld /data/dalvik-cache 2>/dev/null
adb shell ls -l /data/dalvik-cache/ 2>/dev/null | head -10
```

判读要点：

- **第 1 条**是这套自查里最有信息量的一项：它把「Android 版本」之外的第二条版本轴暴露出来。
- **第 2 条**若出现 `dalvik.vm.dex2oat-filter`，说明设备仍保留历史配置；它**不再是全局过滤器**，不要据此推断编译行为。
- **第 3 条**在 Android 14+ 一定有内容；空缺通常意味着读权限不足，而非「profile 被关闭」。
- **第 6 条**的属主应是 system 侧域，应用不可写——这是产物完整性的实际保证。

---

## 7. 常见误传纠正

1. **「vdex = verified dex」**——不准确。vdex 是「提取的 dex + 校验和 + 验证依赖 + 类型查找表」，验证结论以 `kVerifierDepsSection` 的形式存在。
2. **「`dalvik.vm.dex2oat-filter` 是现行全局过滤器」**——错。文档标注 "until Android 6"；Android 7 起用 `pm.dexopt.<reason>`。
3. **「可以改属性给 boot image 换过滤器」**——错。Android 12 起 boot image 固定 `speed-profile`，不可更改。
4. **「hidden API 在 debuggable 构建上默认可绕过」**——错。豁免仅限平台签名/系统应用；绕过靠改 `hidden_api_policy` 调试设置。
5. **「`setHiddenApiExemptions` 是公开 API」**——它是 `@SystemApi`，非公开。
6. **「Android 10 起禁止 `dlopen` 应用可写目录里的文件」**——未核实。原文只讲禁 `execve()`、禁可写 fd 映射 `PROT_EXEC`；只读强制是 Android 14（Java）/17（native）。
7. **「`android_dlopen_ext` 是内部 API」**——错，它是公开 NDK API；内部专用的是 `ANDROID_DLEXT_USE_NAMESPACE`。
8. **「`classloader-namespace` 是官方命名空间名」**——未核实；AOSP 用常量 `kClassloaderNamespaceName`。
9. **「JIT 可以关闭以缩小攻击面」**——Android 14 起 profile 恒开启、不可关闭；JIT 本身则取决于编译过滤器。
10. **「profile 是纯性能数据，与安全无关」**——它影响 AOT 编译哪些方法，是「决定执行什么代码的输入」。
11. **「CVE-2023-20963 是 PendingIntent 漏洞」**——错，是 WorkSource 的 parcel mismatch。
12. **「CVE-2021-39624 / CVE-2022-20452 / CVE-2024-0044 属 ART」**——均为 PackageManager / Framework / 包安装类。
13. **「ART 是 Android 12 引入 APEX 的」**——官方只写「作为 APEX 交付」，具体引入版本未核实。

---

## 未核实与失效来源

- **A. 「Android 12 引入 ART APEX」。** 官方文档只写 "ships as an APEX since it is a critical system component"，未给出引入版本。
- **B. 「Android 10 起禁止 dlopen 应用可写目录文件 / 禁止 `System.load` `/data/data`」。** 原文只覆盖 `execve` 与可写 fd 映射 `PROT_EXEC`；更强表述未见出处。
- **C. `classloader-namespace` 字面串。** AOSP 使用常量 `kClassloaderNamespaceName`，字面串未在已取文件中出现。
- **D. JIT 可执行内存的具体机制（页权限切换时机、是否部分 W^X）。** 不同版本实现差异大，未逐版本核验；正文只给结论性问题清单。
- **E. `dalvik.vm.*` 的完整属性清单。** 仅逐字核实了 `heapsize`、`usejit`、`extra-opts`、`dex2oat-filter` 四项；其余属性未验证，正文未枚举。
- **F. Android 17 behavior changes 页的适用范围。** `System.load()` 只读要求取自该页原文，未核验其对各 targetSdk 的分档细节。

---

## 参考

- 配置 ART（编译过滤器、`pm.dexopt.*`）：[source.android.com/docs/core/runtime/configure](https://source.android.com/docs/core/runtime/configure)
- JIT 编译器：[source.android.com/docs/core/runtime/jit-compiler](https://source.android.com/docs/core/runtime/jit-compiler)
- ART Mainline 模块：[source.android.com/docs/core/ota/modular-system/art](https://source.android.com/docs/core/ota/modular-system/art)
- 非 SDK 接口限制：[developer.android.com/guide/app-compatibility/restrictions-non-sdk-interfaces](https://developer.android.com/guide/app-compatibility/restrictions-non-sdk-interfaces)
- Android 14 行为变更（DCL 只读）：[developer.android.com/about/versions/14/behavior-changes-14](https://developer.android.com/about/versions/14/behavior-changes-14)
- Android 10 行为变更（`execve` / 可写 fd）：[developer.android.com/about/versions/10/behavior-changes-10](https://developer.android.com/about/versions/10/behavior-changes-10)
- Android 17 行为变更（`System.load()` 只读）：[developer.android.com/about/versions/17/behavior-changes-17](https://developer.android.com/about/versions/17/behavior-changes-17)
- Linker namespace：[source.android.com/docs/core/architecture/vndk/linker-namespace](https://source.android.com/docs/core/architecture/vndk/linker-namespace)
- `vdex_file.h`：[android.googlesource.com/platform/art/+/refs/heads/android13-release/runtime/vdex_file.h](https://android.googlesource.com/platform/art/+/refs/heads/android13-release/runtime/vdex_file.h)
- `AndroidRuntime.cpp`（`dalvik.vm.*` 解析）：[android.googlesource.com/platform/frameworks/base/+/refs/heads/main/core/jni/AndroidRuntime.cpp](https://android.googlesource.com/platform/frameworks/base/+/refs/heads/main/core/jni/AndroidRuntime.cpp)
- 公告（2021-06，CVE-2021-0511）：[source.android.com/docs/security/bulletin/2021-06-01](https://source.android.com/docs/security/bulletin/2021-06-01)

---

## 联系方式

如果你在企业里做移动应用的代码加载合规评审或终端运行时基线，需要就「动态加载合规性」「运行时模块版本清点」「native 库依赖白名单」做一次针对自有应用或现网设备的核查，可以通过 Telegram 联系：<https://t.me/one00190>

---

*本手册仅用于防御与教育目的，讨论的均为公开且已被修复问题的机制与设计缺陷，不包含任何可直接利用的代码。文中标注「未核实」的内容不应作为事实引用；发现错误欢迎指出。*
