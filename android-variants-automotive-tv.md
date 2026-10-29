---
layout: page
title: "Android 变体设备安全面手册：车机、电视与穿戴"
description: "变体不是「同一系统换个壳」：车机用户模型必须从 headless system user 讲起、VHAL 与 CarService 这层车机特有信号抽象的真实边界、变体在官方披露体系里的位置、AAOS 公告逐条台账与 CISA KEV 中车机条目全部缺席的核对结果；含只用已核实命令/属性的自查清单与企业..."
date: 2026-10-27
permalink: /android-variants-automotive-tv/
---

> 本手册按「变体边界与代码来源 → 用户与并发模型 → VHAL 与 CarService → 构建目标与公告覆盖 → 可观测面」的顺序，梳理 Android Automotive（车机）、Android TV 与 Wear OS 三类变体在公开一手来源中**成文存在**的安全事实。核心论点是：**变体设备的公开可核实信息比手机稀疏得多，这属于观测缺口（observability gap），而不是风险缺口（risk gap）**——公告里没有 TV 章节、KEV 里没有车机条目，不等于这些设备没有漏洞，只等于面向公众的披露通道本身不覆盖它们。全篇只做防御与教育性描述，不含任何攻击步骤、利用构造或绕过操作。

---

## 0. 使用方式与边界

### 0.1 本手册能给你什么

- **一份「变体 vs. 手机」的披露通道差异表**：手机有 Android Security Bulletin（ASB）+ Pixel 公告 + 厂商公告三层；车机多出一条独立的 AAOS 月度公告通道；TV 与 Wear 在 ASB 里**连章节都没有**。
- **一组可以逐字引用的实名**：真实的文档入口、真实的用户类型措辞、真实的 VHAL 参考实现类名、真实的构建目标名。变体设备领域最大的问题是「看起来很像真的」的编造名字，本手册的作用是把**证实过的**和**没证实的**分开。
- **一份 17 条车机公开 CVE 的逐条台账**，含公告真实章节名、类型缩写、严重级别、AOSP 版本、A-bug 编号与公告日期。
- **一个 KEV 覆盖结论**：18 个车机相关 CVE 在 CISA KEV 中**全部未收录**，以及一个高频误传（Tesla TCU 的 adbd 条目）的纠正。
- **企业侧处置建议与只读自查清单**，只列官方文档原文出现过的命令与属性。

### 0.2 本手册不给你什么

- 不给任何攻击链、payload、提权步骤或针对车辆总线的操作细节。车辆是物理安全攸关系统，本手册只服务防御、审计与教育。
- 不给出「TV 漏洞」「Wear 漏洞公告条目」。**这类条目在本次逐期扫描中根本不存在**（见 §4），把它们写出来就是幻觉。
- 不把未核实的名字当事实。所有查无出处的名称集中在文末「未核实与来源失效」节，正文只在**纠正误传**的语境中提及它们。

### 0.3 证据基线与时间戳

- 核验日期：**2026-10-27**。
- ASB 扫描范围：**2021-06 → 2026-10** 逐期（48+ 期），URL 形如 `https://source.android.com/docs/security/bulletin/YYYY-MM-01`（注意：**公告 URL 没有 `/YYYY/` 目录层**，且当前每月只有 `-01` 一页，`-05` 结尾的 URL 已不存在）。
- AAOS 公告扫描范围：`aaos/2021-01-01 … aaos/2026-02-01`（索引页 62 条），其中 2024-01 → 2026-02 共 26 期被逐条计数。
- CISA KEV 快照：`catalogVersion 2026.10.26`、`count 1758`（去重后 1772 个 CVE ID）。
- AOSP 代码来源：`android.googlesource.com` 的分支列表（`platform/manifest/+refs`，3584 个 `refs/heads/*`）、工程清单（`?format=JSON`，199 个 `device/*` 工程）与目录树正文。

---

## 1. 变体边界与代码来源

### 1.1 三条边界，先记住结论

| 变体 | 是否 AOSP 可构建的独立分支族 | 真实代码/配置落点 | AOSP 文档章节 |
|---|---|---|---|
| Android Automotive | **是**，有独立发布分支族 | `device/generic/car`、`device/google_car`、`hardware/interfaces/automotive/*` | `/docs/automotive`（200+ 子页） |
| Android TV | **否**，是主线 AOSP + TV 产品配置 | `device/google/atv` | `/docs/devices/tv`（**不是** `/docs/tv`） |
| Wear OS | 只有发布快照，没有开发主线 | **AOSP 无任何 `device/*wear*` 工程** | **无 AOSP 文档章节** |

### 1.2 Automotive：唯一有专属发布分支族的变体

一手来源 `https://android.googlesource.com/platform/manifest/+refs`（共 3584 个 `refs/heads/*`）中实测存在：

```
android-automotiveos-14.0.0_lts2 … android-automotiveos-14.0.0_lts7
android-automotiveos-15.0.0_lts1 … android-automotiveos-15.0.0_lts5
android14-qpr1-automotiveos-release
android15-automotiveos-release / android15-automotiveos-dev
ub-automotive-master
ub-automotive-master-20221102 … ub-automotive-master-20260720（带日期快照）
```

对安全研究者的意义：车机有**可独立追溯的源码基线**。当你要判断「某台车机的 `android.car` 代码对应哪一版」时，`android-automotiveos-<ver>_lts<n>` 是真实可指的坐标，而不是手机分支的近似。AAOS 的发行入口见 `https://source.android.com/docs/automotive/start/releases`（子页 `aaos-24q3 … aaos-26q2`、`r_release`、`s_qpr3_release`、`t_release`、`u_qpr1_release`、`u_udc_release`）。

### 1.3 Wear OS：分支名存在，但它是「产品快照」而不是「AOSP 变体」

实测的 wear 分支族：

```
android-wear-5.0.0_r1 / 5.1.0_r1 / 5.1.1_r1 / 7.1.1_r1 / 8.0.0_r1,r2 / 8.1.0_r1
android-wear-9.0.0_r2 … android-wear-9.0.0_r34    ← 最后一个「整包系统」形态的 wear 分支族
android-wear-11.0.0-gpl_r1 … r5
android-wear-13.0.0-gpl_r1 … r6
android-wear-14.0.0-gpl_r1 … r3
android-wear-15.0.0-gpl_r1 … r3 / android-wear-15.0.0_r1
nougat-mr1-wear-release / oreo-mr1-wear-release
android-wear-n-preview-1,2 / android-wear-o-preview-3,4 / android-wear-p-preview-2
```

**可安全陈述的边界结论**：

1. `platform/manifest` 里**没有**名为 `wear` 的开发主线分支，Wear 只有 `android-wear-*` 发布快照。
2. Android 11 之后的 Wear 分支名统一带 `-gpl` 后缀；`-gpl` 的官方定义**本次未核实**（见文末），因此不要写成「`-gpl` 就是内核源码」。
3. AOSP 里**没有** `device/*wear*` 产品工程（199 个 `device/*` 逐条核对），`source.android.com` **没有** Wear 文档章节（`/docs/wear`、`/docs/devices/wear`、`/docs/devices/wear/about`、`/docs/devices/wear/os`、`/docs/devices/tech/wear` 全部实测 404；`/docs/devices` 侧栏只有 `admin`、`cuttlefish`、`tv`）。

因此：**Wear OS 的系统镜像与更新节奏属于 Google 产品文档一侧，不是 AOSP 可构建目标。** 手册与报告里凡是写「Wear OS 是 AOSP 的一条分支」的表述，都应改成「Wear OS 有 AOSP 发布快照分支，但不构成可构建的 AOSP 产品工程」。

### 1.4 Android TV：不是分支，是「主线 AOSP + TV 包集合与产品配置」

3584 个分支里，TV 侧只有历史遗留的两条：

```
android-live-tv
android-live-tv-l-mr1
```

除此之外只有 AndroidX 客户端库分支（**不是系统分支**）：`androidx-leanback-release`、`androidx-tv-foundation-release`、`androidx-tv-material-release`、`androidx-androidx-tv-material-release`、`androidx-tvprovider-release`。

**这里要钉死一个高频误传**：`Leanback` 是 **AndroidX 库名**（`androidx-leanback-release`）与 AOSP 示例工程名（`device/google/atv/LeanbackSampleApp`），**不是公告里的子系统/组件名**。它在 ASB 与 AAOS 公告正文中一次都没有出现。

### 1.5 Android Go：配置层的东西，没有代码分支

`platform/manifest` 中**不存在任何 `android-go*` 分支**。逐词检索 `go$`、`-go-`、`android-go`、`golite` 只命中 `aml_go_*`（AOSP 内部自动化快照）与 `androidx-*google-shortcuts-release` 等无关分支。本次也未能在 `build/make` 与 `device/generic/goldfish` 树里定位到 `_go` 产品配置文件（`build/make/+/refs/heads/main` 在 gitiles 上直接 404）。结论：**Android Go（低内存机型）是构建目标/配置层面的形态，本手册不写任何 `_go` lunch 目标名。**

### 1.6 `hardware/interfaces/automotive/` 的真实子目录

一手目录树实测，8 个子目录：`audiocontrol`、`can`、`evs`、`ivn_android_device`、`occupant_awareness`、`remoteaccess`、`sv`、`vehicle`。

安全含义很直接：**车载 CAN 总线（`can`）、环视/驻车相机（`evs`）、乘员感知（`occupant_awareness`）、远程访问（`remoteaccess`）、车辆信号服务（`sv`）都是 AOSP 里成文存在的原生 HAL 面**，它们运行在比 Java 框架更靠近车辆物理信号的层次上；而 `vehicle` 就是 §3 的 VHAL。

### 1.7 真实文档入口与抓取方法学

| 变体 | 真实入口（实测 200 + 服务端渲染正文） |
|---|---|
| Automotive | `https://source.android.com/docs/automotive` |
| Automotive 发行 | `https://source.android.com/docs/automotive/start/releases` |
| Android TV | `https://source.android.com/docs/devices/tv` |
| Wear OS | **不存在**（无任何 AOSP 文档章节） |
| CDD | `https://source.android.com/docs/compatibility/cdd`；更新页 `/docs/whatsnew/latest-cdd` |
| AAOS 公告 | `https://source.android.com/docs/security/bulletin/aaos` + `/aaos/YYYY-MM-01` |

**两条必须传给下一个研究者的抓取经验**：

- `source.android.com` 需要带浏览器 UA（或 `Accept: text/plain`）才能拿到服务端渲染正文；否则只有 JS 外壳，会把真实页面误判成「查无命中」。404 页固定约 110 KB，容易反过来把 404 误判成「有内容」，所以要同时看状态码。
- `android.googlesource.com` 取目录/文件用 `<path>/?format=TEXT` + `base64 -d`；仓库根路径需要 `platform/` 前缀（如 `platform/hardware/interfaces`），但 `device/google_car`、`device/google/atv` 可以裸路径访问。**已知陷阱**：gitiles 能列出某个目录（200），但对同目录下**单个文件**请求 `?format=TEXT` 时可能返回 404（本次 `VehiclePropertyAccess.aidl` 就是这样）——「列目录成功」不等于「能取文件内容」，也不等于「文件不存在」。

### 1.8 实测 404、不要再写进报告的 Automotive 路径

`/docs/automotive/foundation/user`、`/docs/automotive/setup`、`/docs/automotive/system/upgrades`、`/docs/automotive/remote-access`、`/docs/automotive/vhal/debug`、`/docs/automotive/ext/car-properties`、`/docs/automotive/reference`、`/docs/automotive/user`、`/docs/automotive/services`、`/docs/automotive/can-bus`、`/docs/automotive/properties`、`/docs/automotive/vhal-impl`、`/docs/automotive/performance`、`/docs/automotive/compatibility/cdd`、`/docs/core/updates`、`/docs/setup/running`、`/docs/devices/android-low-ram`、`/docs/devices/low-ram`。

其中最值得点名的两条：

- **`/docs/automotive/compatibility/cdd` 不存在（404）**：Automotive 侧栏里的「Compatibility Definition Document (CDD)」链接指向**通用 CDD**，没有车机专属 CDD 落地页。
- **`android.auto.go` 是幻觉域名**：`nslookup` 返回 `Non-existent domain`（NXDOMAIN），`https://android.auto.go` 连接失败（`http=000`）。AOSP 文档站从来不是这个域名，而是 `source.android.com/docs/...`。任何报告里出现 `android.auto.go` 都应视为编造。

---

## 2. 车机用户与并发模型：Headless system user 才是起点

车机与手机在「谁在使用这台设备」这个问题上模型完全不同。做车机日志分析、权限审计或事件响应时，把手机的用户直觉套上去会得到错误结论。

### 2.1 官方三类用户（一手：`/docs/automotive/users_accounts/multi_user`）

原文口径：Android Automotive 依赖 Android 的多用户支持来提供「共享设备体验」，每个用户对应一个不同的**自然人**；Android 10 及以上支持以下类型：

- **Headless system user（无头系统用户）**：在后台运行并承载所有系统服务。原文强调 "For Automotive, the system user isn't intended to be used, nor directly accessed, by a physical person."——它不是给自然人用的，也不该被自然人直接访问。
- **Regular user（常规用户）**：原文 "In Automotive, all regular users are secondary users."——车机上**所有常规用户都是 secondary 用户**。
- **Guest user（访客用户）**：借车的人属于临时用户；**一台设备同时只能定义一个 guest**。

同页的关键限定，必须逐字记住：

> "Note: The primary user flag (`FLAG_PRIMARY`) isn't yet supported in the Automotive headless system user model."

也就是说，**车机没有移动端意义上的「主用户」flag**。你在手机侧习惯的「主用户 = 机主」推断在车机上不成立。

### 2.2 角色与默认限制（同页原文）

- **Admin**：默认可以执行所有任务，并可把 admin 角色授予其他用户。
- **Non-admin**：默认**不能**恢复出厂、**不能**给自己授予 admin 角色、**不能**删除其他用户。
- **Guest**：默认既不能恢复出厂也不能删除其他用户；默认还受更多限制——不能修改（增删）账号、不能安装应用、不能设置锁屏；且 guest 默认是 **ephemeral（临时的）**。

**成文标识核验结论（防止造名）**：官方措辞是 **Headless system user / Regular user / Guest user**，角色是 **Admin / Non-admin / Guest**。真实成文的类型标识是 AOSP 资源里的 user type 全名：

- `android.os.usertype.full.SECONDARY`
- `android.os.usertype.full.GUEST`

出处：`frameworks/base/core/res/res/xml/config_user_types.xml`（overlay 形如 `<full-type name="android.os.usertype.full.SECONDARY">` / `<full-type name="android.os.usertype.full.GUEST">`，可带 `default-restrictions no_debugging_features="true" no_install_unknown_sources="true"`）。

`SYSTEM_USER` / `VISITOR` 这类常量名**本次未找到任何出处**，只在文末「未核实与来源失效」中作为误传纠正列出，正文不使用。

### 2.3 前台/后台与并发上限（一手：`/docs/automotive/users_accounts/user_system`，页面标题 "Foreground and background user system handling"）

可核到的成文事实：

- **必须始终有一个前台用户**。
- 从 Android 10 起，车机默认配置为**最多 3 个用户同时运行**，由 `config_multiuserMaxRunningUsers` 控制。原文推论："Therefore, in addition to the headless system user (**User 0**), only one foreground user and one background user can be configured."——即 **User 0（系统用户）+ 1 个前台 + 1 个后台**。
- 用户被移到后台时会 **stopped 但不 locked**；达到上限时，最久未使用的后台用户会被 stopped 并 locked（`config_multiuserDelayUserDataLocking`）。
- **Garage Mode** 期间会重启那些 stopped-and-unlocked 的后台用户。
- **Guest 只能在前台运行**：切出后 guest 被停止，且不能作为后台用户运行。
- 前后台切换时，该用户的所有 activity 与前台服务被终止；但**一方与 OEM 系统应用的 persistent 服务**只要该（已进入后台的）用户没被 stop 就继续运行，这些 persistent 后台进程不会被终止。

安全解读（防御视角）：最后一条是车机日志里「明明切换了用户，为什么还有进程活着」的官方解释。做车机取证/审计时，**「用户已切走」不等于「该用户的所有代码都停了」**； persistent 服务的归属要按 OEM 系统应用清单核对，而不是按当前前台用户推断。

### 2.4 调试授权：CDD 义务与加密令牌门控（一手：`/docs/automotive/security/secure_dev_options`）

原文三条关键点：

> "Under the Android Compatibility Definition Document, OEMs must provide a way to enable app development. However, providing mobile-like developer options within cars leaves those cars vulnerable to attack. Access to developer options can now be gated by an OEM using an authenticated cryptographic token mechanism."

> "Important: We strongly recommend that OEMs configure **`no_debugging_features`**, **`no_install_unknown_sources`**, and **`no_install_unknown_sources_globally`** by default across all user profiles on production builds to prevent unauthorized app sideloading and Android Debug Bridge (adb) debugging exposure."

这段是车机与手机在「开发者选项」上最实质的差异：**手机的开发者选项是给机主的，车机的开发者选项是一块被明确写成「会让车辆易受攻击」的暴露面**，所以 Google 给的建议是「在所有用户配置、所有生产构建上默认关闭」。

可核到的实名与实现位置（同页原文，逐词核到，可直接引用）：

- `config_defaultFirstUserRestrictions`（`frameworks/base/core/res/res/values/config.xml`）
- `frameworks/base/core/res/res/xml/config_user_types.xml`
- `UserManager.DISALLOW_DEBUGGING_FEATURES`
- `packages/apps/Car/Settings/src/com/android/car/settings/system/BuildNumberPreferenceController.java`（版本号 7 次点击倒数；`Build.IS_USER` 且命中 `DISALLOW_DEBUGGING_FEATURES` 时阻断）
- **Debugging Restriction Controller (DRC)** 参考实现：AOSP `packages/apps/Car/DebuggingRestrictionController`；集成指南 `/docs/automotive/security/drc`；令牌格式为 **JSON Web Signature (JWS, RFC 7515)**。

### 2.5 车机专属安全文档族（全部实测 200）

`/docs/automotive/security`（注意：概览页正文只有一段通用安全声明，**没有 Automotive 专属条款**——手册应说明「车机的成文安全要求分散在各子页」）、`secure_dev_options`、`macsec`（Enable MACsec for Ethernet features）、`drc`、`vehicle-bound`（Use vehicle bound file encryption）、`mfg_guide`（Manufacturer guide for long-term Android security）、`vehicle_system_isolation`、`update`（OTA updates）；另有 `/docs/automotive/usb-reset`（USB Port Reset API）、`/docs/automotive/watchdog`（含 `wd_flash_memory` / `wd_performance_health` / `wd_system_health`）、`/docs/automotive/flash-wear`（Flash wear management）、`/docs/automotive/feature-control`（Car service feature control）、`/docs/automotive/tools/fuzz`（Enable fuzzers）与 `/docs/automotive/tools`（CATBox / Spectatio / Torq / sys-perf / network-simulation / ui-frameworks）、`/docs/automotive/dev-tools`。

对企业安全团队的直接价值：**以太网 MACsec、车绑定文件加密、系统隔离、看门狗、闪存磨损管理**这五页，就是车机长期运行安全基线的官方出处；把它们当作车机安全评估的检查表比去找「车机漏洞列表」有效得多。

---

## 3. VHAL 与 CarService：车机特有的信号抽象层

### 3.1 版本与迁移状态（一手：`/docs/automotive/vhal`，"Vehicle HAL (VHAL) overview"）

> "In Android 13 and higher, VHAL migrated to AIDL at **`IVehicle.aidl`**"
> "In Android 12 and lower, VHAL is defined with HIDL language in **`IVehicle.hal`**"
> "Note: All new features, including new property definitions, will be added to the AIDL interface only."
> "**CarService and Android native components** support HIDL and AIDL backend and prefer the AIDL backend when it's available."

指导级别也是原文成文的：**STRONGLY RECOMMEND** 现有 HIDL 实现迁移到 AIDL；**MUST** 新 VHAL 实现使用 AIDL。

AIDL 冻结版本目录实测（`automotive/vehicle/aidl/aidl_api/android.hardware.automotive.vehicle/`）：`1`、`2`、`3`、`current`，即 HAL 包名 **`android.hardware.automotive.vehicle`**、AIDL 版本 1/2/3。HIDL 侧目录 `automotive/vehicle/2.0/` 实测存在（对应 `android.hardware.automotive.vehicle@2.0`）。

对审计的含义：**同一车型年可能同时存在 HIDL 与 AIDL 两套后端**。做车机资产清点时，判后端类型要用 §8 的 `dumpsys`/`lshal` 只读手段，而不是靠 Android 版本猜。

### 3.2 参考实现的真实类名（一手：gitiles 目录树 `automotive/vehicle/aidl/impl/`）

`impl/` 下实测有 `3` 与 `current`。`impl/current/vhal/src/` 内容：

- **`DefaultVehicleHal.cpp`**、**`VehicleService.cpp`**、`ConnectedClient.cpp`、`SubscriptionManager.cpp`、`fuzzer.cpp`
- `impl/current/vhal/include/`：`DefaultVehicleHal.h`、`ConnectedClient.h`、`SubscriptionManager.h`
- 服务定义文件：`impl/current/vhal/vhal-default-service.rc`、`vhal-default-service.xml`
- 同层还有 `impl/current/{default_config,fake_impl,grpc,hardware,proto,utils}`，以及 `rust_impl`、`proto`、`vts`、`aidl_test`、`emu_metadata`、`generated_lib`

**结论与证伪**：当前 AOSP 树里**看不到 `VhalAidlBridge` 这个类**。桥接层实名是 **`DefaultVehicleHal`**，服务入口是 **`VehicleService`**。`VhalController` 在 `impl/1.0` 等历史版本中是否存在本次未核实（只遍历了 `impl/3` 与 `impl/current`）。写手册请用 `DefaultVehicleHal`，**不要用 `VhalAidlBridge`**。

顺带一条对「变体有没有 fuzz 面」的回答：`impl/current/vhal/src/fuzzer.cpp` 与 `/docs/automotive/tools/fuzz`（Enable fuzzers）同时存在，说明 **VHAL 是 AOSP 明确纳入 fuzzing 覆盖的接口**，这一点比手机侧很多 HAL 更成文。

### 3.3 属性访问路径与真实文件清单

访问路径原文（`/docs/automotive/vhal/property-configuration`）："Access through **`CarPropertyManager`** (for Java components) or through the **`libvhalclient`** (for native)."

Java 侧读属性的具体 API 也是原文（`/docs/automotive/vhal/special-properties`）："To read this property from an Android app, use **`CarPropertyManager.getIntProperty`**. In Android 13 and higher, use **`VehiclePropertyIds.GENERAL_SAFETY_REGULATION_COMPLIANCE`** as the property…"

接口侧原文片段（`/docs/automotive/vhal/vhal-interface`）：`VehiclePropConfigs getPropConfigs(in int[] props)` — "Returns a list of Property Configurations for given property IDs."；`void getValues(IVehicleCallback…)`。

属性定义目录（`automotive/vehicle/aidl_property/android/hardware/automotive/vehicle/`）与接口目录（`aidl/android/hardware/automotive/vehicle/`）均实测存在，文件名为一手目录列表。可核到的**文件名**包括：属性侧 `VehicleProperty.aidl`、`VehiclePropertyAccess.aidl`、`VehiclePropertyChangeMode/Status/Type/Group.aidl`、`VehicleArea.aidl` 与 `VehicleArea{Seat,Wheel,Door,Window,Mirror}.aidl`、`VehicleDisplay.aidl`、`VehicleGear.aidl`、`VehicleIgnitionState.aidl`、`VehicleVendorPermission.aidl`、`PropIdAreaId.aidl`、`VehicleAreaConfig.aidl`；接口侧 `IVehicle.aidl`、`IVehicleCallback.aidl`、`VehiclePropConfig(s)/Value(s).aidl`、`GetValue*/SetValue*Request(s)/Result(s).aidl`、`StatusCode.aidl`、`SubscribeOptions.aidl`、`VehiclePropError(s).aidl`、`RawPropValues.aidl`、`SupportedValuesListResult(s).aidl`、`MinMaxSupportedValueResult(s).aidl`。

**诚实边界**：本次**未能取到 `VehicleProperty.aidl` / `VehiclePropertyAccess.aidl` 的文件正文**（gitiles 单文件 `?format=TEXT` 404，同目录列表 200）。因此本手册**不引用 `ACCESS_READWRITE` / `ACCESS_READ_ONLY` / `ACCESS_WRITE_ONLY` 的常量拼写，也不引用 `VEHICLE_SPEED` 之类的属性名**，只引用真实文件名与文档措辞；文档实测到的真实字面例只有 `SEAT_MEMORY_SELECT`、area ID `ROW_1_LEFT`、`INFO_VIN`、`GENERAL_SAFETY_REGULATION_COMPLIANCE`。

### 3.4 用户、OBD2 与 ADAS 都被建模进 VHAL —— 变体攻击面成文的依据

属性包里实测存在（目录列表）：`CreateUserRequest/Response/Status.aidl`、`SwitchUserRequest/Response/Status.aidl`、`SwitchUserMessageType.aidl`、`RemoveUserRequest.aidl`、`InitialUserInfoRequest/Response(.Action).aidl`、`UserInfo.aidl`、`UsersInfo.aidl`、`UserIdentification*.aidl`、`Obd2*.aidl`、`DiagnosticFloatSensorIndex.aidl`、`DiagnosticIntegerSensorIndex.aidl`、`ImpactSensorLocation.aidl`、`VehicleSeatOccupancyState.aidl`、`VehicleAutonomousState.aidl`、`ElectronicStabilityControlState.aidl`、`AutomaticEmergencyBrakingState.aidl`、`LaneKeepAssistState.aidl`、`DriverDistractionState.aidl`、`EvChargeState.aidl`、`TrailerState.aidl`、`WindshieldWipersState.aidl`。

配套文档页实测存在：`/docs/automotive/users_accounts/user_hal`（"User HAL properties"）、`accounts`（Manage users and accounts）、`disable_packages`（Remove packages for the system user）、`/docs/automotive/vhal/adas-properties`（ADAS vehicle properties）。

**这就是「变体特有安全面」最硬的一条成文证据**：车机的用户创建/切换（CreateUser / SwitchUser / RemoveUser 请求-响应对）与车辆诊断（OBD2）、主动安全状态（ESC、AEB、车道保持、驾驶员分心、座椅占用、碰撞传感位置）**共享同一层 AIDL 属性模型**。审计时必须把「谁能订阅/写这些属性」当作与「谁能装应用」同级的权限问题，而不是把它当成遥测细节。

相机侧真实子系统：`/docs/automotive/camera/acs`（Android Camera Service，含 `multi-client-cameras`、`camera2-apis`、`camera2-migration`）与 `/docs/automotive/camera/evs`（`camera-hal`、`camera_control`、`camera_system`、`display_proxy`、`event_frame`、`frame_metadata`、`multi_camera`）；AOSP 目录 `automotive/occupant_awareness` 对应乘员感知。

### 3.5 簇屏（Instrument Cluster）：权限与虚拟显示的实名（一手：`/docs/automotive/displays/cluster_api`）

> "…only apps with the **`android.car.permission.CAR_INSTRUMENT_CLUSTER_CONTROL`** permission can send data to the car."
> "Hold the **`BIND_INSTRUMENT_CLUSTER_RENDERER_SERVICE`** system permission. This guarantees that only the Instrument Cluster rendering service included as part of the Android system …"

同页其余实名：完整权限串 `android.car.permission.BIND_INSTRUMENT_CLUSTER_RENDERER_SERVICE`；服务须声明 `android:singleUser="true"`；实现类 `ClusterRenderingService` / `InstrumentClusterRenderingService`；示例常量 `DISPLAY_WIDTH = 1280`、`DISPLAY_HEIGHT = 720`、`DISPLAY_DPI = 320`、`DISPLAY_UNOBSCURED_LEFT/TOP/RIGHT/BOTTOM`；通过 `mDisplayManager.createVirtualDisplay(mUniqueId, DISPLAY_WIDTH, DISPLAY_HEIGHT, DISPLAY_DPI, …)` 建立簇屏虚拟显示；`ClusterActivityService` + `Rect unobscuredBounds` + `boolean visible` 上报遮挡区；文档要求实现 `Service#dump()` 以便 `dumpsys` 读取状态。页面亦出现 "CarService bootstraps all car-specific services and provides access to these services through a …"。

安全含义：簇屏是**驾驶安全攸关的显示通道**，Google 把它锁在两个 `android.car.permission.*` 权限与一个 `singleUser` 系统服务之后。企业侧做权限审查时，`CAR_INSTRUMENT_CLUSTER_CONTROL` 的授予清单应视为高危权限集的一部分。

**注意**：`CLUSTER` / `DISPLAY` 作为 VHAL 枚举的字面值本次未取到正文，属未核实项；上面引用的是**权限串与 API 名**，不要把它们当作 VHAL 属性枚举。

### 3.6 SDV 与多 VM 趋势（存在性核实）

`/docs/automotive/sdv`、`/docs/automotive/ivi` 实测 200。VHAL 概览页原文：

> "Software-Defined Vehicle (SDV) Signal Architecture: For service-oriented vehicle signal definitions across multi-VM topologies and SOME/IP networks, see **Vehicle Services Interface Definition Language (VSIDL)** and the AAOS SDV System Architecture."

子页实测存在：`sdv/core-areas/vsidl`、`sdv/some-ip-integration`、`sdv/sdv-system-architecture`、`sdv/sdv-gateway-on-ivi`、`sdv/display-safety`、`sdv/display-safety/get-started#safety-monitor`、`sdv/display-safety/driverui`。AOSP 侧 `automotive/sv`、`automotive/can`、`automotive/ivn_android_device` 与之呼应。另有 `docs/automotive/virtualization`（`architecture` / `reference_platform` / `tools`）与 `docs/automotive/security/vehicle_system_isolation`。

趋势判断（防御侧）：车机正在从「一个 Android 实例」变成「多 VM 中的 Android guest + 车辆服务网关」。**威胁模型也要跟着变**：IVI 与关键域之间的 gateway（`sdv/sdv-gateway-on-ivi`）、SOME/IP 网络、以及跨 VM 的信号定义（VSIDL）会成为新的边界审查对象，而不是 Framework 权限 alone。

---

## 4. 构建目标与公告覆盖：变体在「官方披露体系」里的真实位置

### 4.1 真实 lunch / product 目标名（一手：各 `AndroidProducts.mk` 原文）

**Automotive（`device/google_car/AndroidProducts.mk`，分支 `android-automotiveos-15.0.0_lts5`）**：`PRODUCT_MAKEFILES` 列出 `bluejay_car/aosp_bluejay_car.mk`、`cheetah_car/…`、`oriole_car/…`、`panther_car/…`、`raven_car/…`、`redfin_car/…`、`sunfish_car/…`、`husky_car/…`、`tangorpro_car/…`；`COMMON_LUNCH_CHOICES` 同构，形如：

```
aosp_bluejay_car-trunk_staging-userdebug
aosp_cheetah_car-trunk_staging-userdebug
aosp_oriole_car-trunk_staging-userdebug
aosp_panther_car-trunk_staging-userdebug
aosp_raven_car-trunk_staging-userdebug
aosp_redfin_car-trunk_staging-userdebug
aosp_sunfish_car-trunk_staging-userdebug
aosp_husky_car-trunk_staging-userdebug
aosp_tangorpro_car-trunk_staging-userdebug
```

目录名实测为 `bluejay_car`、`cheetah_car`、`common`、`husky_car`、`oriole_car`、`panther_car`、`raven_car`、`redfin_car`、`sunfish_car`、`tangorpro_car` + `AndroidProducts.mk`、`METADATA`、`OWNERS`。**`_car` 是真实存在的 product flavor**（Pixel 机型目录被复用为 AAOS 开发平台）。

**Automotive 通用（`device/generic/car/AndroidProducts.mk`，分支 `refs/heads/main`）**：`gsi_car_arm64.mk`、`gsi_car_x86_64.mk`、`sdk_car_arm64.mk`、`sdk_car_md_arm64.mk`、`sdk_car_md_x86_64.mk`、`sdk_car_cw_x86_64.mk`、`sdk_car_x86_64.mk`；lunch 例：`gsi_car_arm64-trunk_staging-userdebug`、`sdk_car_arm64-trunk_staging-userdebug`、`sdk_car_md_arm64-trunk_staging-userdebug`、`sdk_car_cw_x86_64-trunk_staging-userdebug`、`sdk_car_x86_64-trunk_staging-userdebug`。目录内另有 `car_x86_64/`、`emulator/`、`emulator_car64_arm64/`、`emulator_car64_x86_64/`、`hals/`、`setup/`、`tools/`、`common/`。

**Android TV（`device/google/atv/products/` 目录列表实测）**：`aosp_tv_arm64`、`aosp_tv_x86`、`gsi_tv_arm`、`gsi_tv_arm64`、`sdk_atv64_arm64`、`sdk_atv_x86`。注意命名形式：**是 `aosp_tv_*` / `sdk_atv*` 前缀式，不是 `xxx_tv` 后缀式**。同目录还有 `atv_base.mk`、`atv_system.mk`、`atv_vendor.mk`、`atv_mainline_system.mk`、`atv_system_ext.mk`、`atv_emulator_vendor.mk`、`atv_generic_system.mk`、`atv_logpersist.mk`、`atv_product.mk`、`gsi_tv_base.mk`，以及资源受限证据 **`atv_lowram_defaults.mk`** 与 **`lowram_boot_profiles/`**。

`device/google/atv` 根目录实测还有 `TvProvision`、`LeanbackSampleApp`、`TvSampleLeanbackLauncher`、`MdnsOffloadCmdService`、`MdnsOffloadManagerService`、`mdns_offload`、`blur_sysprop_notifier`、`atv-component-overrides.xml`、`permissions`、`sepolicy`、`overlay`、`sdk`、`emulator_x86`、`Generic.kl`、`FrameworkPackageStubs`。

**`_wear` flavor：实测不成立。** AOSP 199 个 `device/*` 工程里没有任何 wear 目录，因此**没有** `aosp_*_wear` 之类 lunch 目标可引用。

车机的资源约束同样成文，只是形式不同：`/docs/automotive/flash-wear`（Flash wear management）与 `/docs/automotive/watchdog/wd_flash_memory`（Monitor flash memory usage）。CDD 层面：`secure_dev_options` 原文直接以 CDD 作为义务来源（"Under the Android Compatibility Definition Document, OEMs must provide a way to enable app development."）；但 **CDD 中「无蜂窝/无显示/资源受限」的专门条款正文本次未展开取到**，本手册不引用具体 CDD 小节号。

### 4.2 核心结论：变体在公告体系里没有专属方向

逐期扫描 **2021-06 → 2026-10 的 Android Security Bulletin**（48+ 期），提取全部 `<h2>/<h3>` 章节标题：

> **从未出现 `Automotive`、`Android TV`、`Television`、`Wear`、`Vehicle` 章节。**

实测出现过的组件章节只有：`Framework`、`Platform`、`System`、`Android Runtime`、`Media Framework`、`Platform apps`、`Platform service`、`System UI`、`Packages`、`Kernel` / `Kernel components`、`Google Play system updates`、`Additional vulnerability details`，以及厂商段 `Arm components`、`Qualcomm components`、`Qualcomm closed-source components`、`MediaTek components`、`Unisoc components`、`Imagination Technologies` 等。

关键字全文检索同样为空：`CarService`、`Vehicle HAL`、`VHAL`、`android.car`、`infotainment`、`Leanback`、`Television`、`Watchdog` 在公告正文里**一次都没有出现**；`wear` 的命中全部来自 `software` 这个词。

由此得到的三条结论（这就是本篇最重要的诚实性要求）：

1. **Android Automotive 有独立公告通道，但章节名仍只用通用组件名**。一手通道 `…/docs/security/bulletin/aaos`，实测章节包括 `AAOS CarSettings`、`AAOS`、`Platform apps`、`Platform service`、`Media Framework`、`System UI`、`Packages`、`System`、`Kernel components`。
2. **Wear OS 未纳入 Android Security Bulletin**：ASB 无 Wear 章节、无 Wear 产品口径；叠加 §1.3（AOSP 无 Wear 文档章节、无 `device/*wear*` 工程）。正确表述是「**Wear OS 的安全更新不走 AOSP/ASB 文档通道，需引用 Google 产品侧文档**」，并明确标注 Google 产品侧文档 URL 本次未取得。
3. **Android TV 同样在 ASB 中没有任何专属章节**，AOSP 侧只有 `docs/devices/tv` 与 `device/google/atv` 构建目标。

因此：**任何「TV 漏洞清单」「Wear 漏洞公告条目」都是编造。** 变体设备看起来「漏洞少」，是因为公开披露通道本身不覆盖它们——这是**观测缺口，不是风险缺口**。

### 4.3 车机与手机的更新通道差异（一手：`/docs/security/bulletin/aaos`）

> "Monthly device updates are an important tool to keep Android users safe and protect their devices. This page contains the available **Android Automotive OS (AAOS) Security Bulletins** which **supplement** the Android Security Bulletins with additional security patches, providing fixes for possible issues affecting devices running AAOS."
> "Important: The Android Automotive security bulletins are published on the **first Monday of the month** unless that Monday falls on a holiday. If the first Monday of the month is a holiday the bulletins will be published on the following work day."
> "Sources: **AAOS devices start receiving OTA updates on the same day the monthly bulletin is released.** Patches listed in the AAOS security bulletins can come from various sources, including: **the car manufacture**, the Android Open Source Project (AOSP), the upstream Linux kernel, and system-on-chip (SOC) manufacturers."
> "Note: Please contact your device supplier for device firmware images."

（`car manufacture` 是官方原文拼写，引用时不要「纠正」。）

SPL 口径（一手 `aaos/2025-06-01`）："The full AAOS update comprises the security patch level of 2025-06-05 or later from the June 2025 Android Security Bulletin **in addition to** all issues in this bulletin." + "Security patch levels of 2025-06-01 or later address all issues associated with the 2025-06-01 security patch level. Device manufacturers that include these updates should set the patch string level to: `[ro.build.version.security_patch]:[2025-06-01]`"。

**差异结论**：手机 = 单一 ASB（+ 厂商 / Pixel 公告）；车机 = **ASB + 独立 AAOS 月度公告**（补充补丁、由整车厂分发固件、公告首发日即开始 OTA）。评估车机补丁状态时，必须同时看 SPL 与「当期 AAOS 公告是否有内容」。

### 4.4 量化事实：AAOS 公告条目极其稀疏（本次逐条实测）

2024-01 → 2026-02 的 **26 期 AAOS 公告**中，只有 **2024-09-01 一期含 1 条 CVE**；其余 25 期正文均为 "There are no Android Automotive OS security patches in the … Bulletin."（例：`aaos/2025-10-01`）。

2021–2023 含条目的期次与条数：`2021-10-01`(1)、`2021-12-01`(3)、`2022-01-01`(3)、`2022-02-01`(2)、`2022-05-01`(1)、`2022-07-01`(2)、`2022-10-01`(1)、`2022-12-01`(8)、`2023-01-01`(8)、`2023-02-01`(1)、`2023-07-01`(1)。

**这条数据要用在它该在的位置**：AAOS 公告条目稀疏，说明绝大多数车机补丁其实是通过 ASB 的**通用组件条目**下发的（同一条 Framework/System 漏洞，手机和车机一起受影响）。它**不能**被读成「车机攻击面小」。

---

## 5. 公开案例：AAOS 公告里的 17 条逐条台账

**URL 形式**：完整链接 = 前缀 `https://source.android.com/docs/security/bulletin/` + 表中「公告」列（如 `aaos/2021-10-01`）；**没有年份目录层**。手机侧同理：`https://source.android.com/docs/security/bulletin/2025-06-01`。**Type 缩写取自公告自身定义**："RCE = Remote code execution, EoP = Elevation of privilege, ID = Information disclosure, DoS = Denial of service, N/A = Classification not available"。公告日期 = AAOS 公告 URL 的 `YYYY-MM-01` 与页面 Published 行。

**章节名说明（关键）**：下表「真实章节」列取自公告页的 `<h2>/<h3>`。车机条目实际落在 `Platform apps`、`System`、`Media Framework`、`Packages`、`Platform service`、`Kernel components` 等**既有章节**，以及 `AAOS` / `AAOS CarSettings` 这类产品级章节名下——**公告体系里不存在 `Automotive` 章节**（§4.2）。

| # | CVE | 公告 | 真实章节 / 组件 | Type | 严重级别 | AOSP | A-bug | 描述要点（NVD 原文片段） |
|---|---|---|---|---|---|---|---|---|
| 1 | CVE-2021-0583 | `aaos/2021-10-01` | **AAOS CarSettings** | EoP | High | 9, 10 | A-182282956 | "In onCreate of BluetoothPairingDialog" |
| 2 | CVE-2021-1039 | `aaos/2021-12-01` | **Platform apps** | EoP | High | 9, 10, 11, 12 | A-182808318 | "In NotificationAccessActivity of AndroidManifest.xml" |
| 3 | CVE-2021-1040 | `aaos/2021-12-01` | Platform apps | EoP | High | 9, 10, 11, 12 | A-182810085 | NVD 描述本次未取到（见文末未核实） |
| 4 | CVE-2021-1038 | `aaos/2021-12-01` | Platform apps | DoS | High | 9, 10, 11, 12 | A-183411279 | "In UserDetailsActivity of AndroidManifest.xml" |
| 5 | CVE-2021-1035 | `aaos/2022-01-01` | 该期章节名未取到 | EoP | High | 10, 12 | A-195668284 | "In setLaunchIntent of BluetoothDevicePickerPreferenceController.java" |
| 6 | CVE-2021-1036 | `aaos/2022-01-01` | 同上 | EoP | High | 9, 10, 11, 12 | A-182812255 | NVD 描述未取到 |
| 7 | CVE-2021-1037 | `aaos/2022-01-01` | 同上 | ID | High | 9, 10, 11, 12 | A-162951906 | NVD 描述未取到 |
| 8 | **CVE-2021-0524** | `aaos/2022-02-01` | **AAOS** | ID | High | 12 | A-180418334 | "In isServiceDistractionOptimized of **CarPackageManagerService.java**"（`android.car` 包内真实服务类名） |
| 9 | CVE-2021-39677 | `aaos/2022-02-01` | AAOS | ID | High | 11 | A-205097028 | "In startVideoStream() there is a possibility of an OOB Read in the heap" |
| 10 | CVE-2022-20223 | `aaos/2022-12-01` | Platform apps | EoP | High | 10, 11 | A-223578534 | "In assertSafeToStartCustomActivity of AppRestrictionsFragment.java" |
| 11 | CVE-2021-39631 | `aaos/2022-12-01` | **System** | ID | High | 10, 11, 12, 12L, 13 | A-193890833 | "In clear_data_dlg_text of strings.xml" |
| 12 | CVE-2021-0954 | `aaos/2022-12-01` | Platform apps | EoP | High | 11, 12, 12L, 13 | A-143559931 | "In ResolverActivity" |
| 13 | CVE-2022-20353 | `aaos/2023-01-01` | Media Framework（按该期章节顺序） | ID | High | 10, 11 | A-221041256 | "In onSaveRingtone of DefaultRingtonePreference.java" |
| 14 | **CVE-2022-28389** | `aaos/2023-01-01` | **Kernel components → `USB CAN bus driver`**（公告组件列原文即此串） | EoP | **Moderate**（公告口径） | —（上游内核） | A-228694270，标注 `Upstream kernel` | "mcba_usb_start_xmit in drivers/net/can/usb/mcba_usb.c in the Linux kernel through 5.17.1 has a double free." |
| 15 | CVE-2023-20927 | `aaos/2023-02-01` | **Platform service** | EoP | High | 13 | A-244216503 | "In permissions of AndroidManifest.xml" |
| 16 | CVE-2023-21260 | `aaos/2023-07-01` | **Packages** | EoP | High | 11, 12, 12L, 13 | A-259384309 | "In notification access permission dialog box" |
| 17 | **CVE-2024-40667** | `aaos/2024-09-01` | **System** | EoP | High | 13, 14 | A-335585728 | NVD 返回空描述 → 只引用公告表格 |

**严重级别双轨原则**：表中级别为 **AAOS 公告口径**；凡公告与 NVD 评级不一致，正式报告须**双轨并列**（公告级别 + NVD 级别/CVSS），不得用一方覆盖另一方。本次素材**未逐条取回 NVD 的 CVSS 数值**，故本节不做 NVD 级别断言（已列入文末未核实）。可确定的是 #14 公告为 `Moderate`，其余 16 条为 `High`。

**方向覆盖与诚实的空结论**：**Automotive 组件**（CarSettings / `android.car` 包内服务）有案例，见 #1、#2、#4、#8、#10；**车机外设（USB / CAN）**有 #14，是唯一一条「外设 + 车载总线」条目；**车机 Media / 视频流**有 #9（簇屏/媒体流 OOB Read）与 #13。**多用户 / 访客 / system user 方向：未检索到条目**——17 条里没有任何一条描述涉及 `UserManager`、多用户或 guest，AAOS 公告也从未出现 `User` 章节。**VHAL / `CarService` 本身：未检索到条目**——公告中从未出现 `VHAL`、`Vehicle HAL`、`CarService` 字样，只有 #8 以 `CarPackageManagerService.java` 这一 `android.car` 类名出现在 NVD 描述里。**Android TV 专属条目：未检索到**（无 TV 章节，`Leanback` 0 命中）。**Wear OS 专属条目：未检索到**（§4.2）。2024-01 之后 AAOS 公告仅 1 条 CVE（#17）。→ **变体公开案例稀疏，不能据此推断攻击面更小。**

---

## 6. CISA KEV 核对：车机条目全部缺席

一手数据 `https://www.cisa.gov/sites/default/files/feeds/known_exploited_vulnerabilities.json`，实测 `catalogVersion 2026.10.26`、`dateReleased 2026-10-26T22:24:24.6101Z`、`count 1758`、去重后 1772 个 CVE ID，`vendorProject: "Android"` 条目 17 条。对 §5 全部 17 条（另加 2022-12-01 期的 CVE-2021-0481 / CVE-2021-39706 / CVE-2021-39707 / CVE-2022-20144 / CVE-2022-20348 等，共 **18 个车机相关 CVE ID**）逐一精确匹配：

> **结论：18/18 全部 `absent`（未收录）。** 含 CVE-2021-0583、CVE-2021-1035/1036/1037/1038/1039/1040、CVE-2021-0524、CVE-2021-39677、CVE-2021-39631、CVE-2021-0954、CVE-2022-20223、CVE-2022-20353、CVE-2022-28389、CVE-2023-20927、CVE-2023-21260、CVE-2024-40667。

**推论边界（必须写清）**：「不在 KEV」只表示 CISA 未确认该 CVE 在真实事件中被用作利用，**不等于**它未被利用，更不等于车机安全。KEV 主要覆盖手机 / Pixel / Framework / Kernel / 厂商 SoC 等被大规模利用的条目；变体设备长期缺乏遥测与公开事件披露，属**观测缺口**而非**风险缺口**。字段级核对：`vendorProject` 唯一值集合中**没有任何汽车厂商**（`Tesla|Toyota|Honda|BMW|Ford` 作为 vendorProject 命中 **0**）；`product` 字段中**没有** `car` / `vehicle` / `automotive` / `infotainment` / `TCU` / `head unit` 类条目。可安全陈述：**截至 catalogVersion 2026.10.26，CISA KEV 中没有 Google Android Automotive 或整车厂车机条目。**

### 6.1 纠正一个高频前提：CVE-2025-34251 不是 Android 漏洞

NVD 一手原文：

> "Tesla Telematics Control Unit (TCU) firmware prior to v2025.14 contains an authentication bypass vulnerability. The TCU runs the Android Debug Bridge (adbd) as root and …"

CWE 归类 `CWE-269`、`CWE-288`。**归属核实**：这是 **Tesla TCU 固件**的问题，**不是 Google Android / AOSP 的漏洞**；它带「Android」字样只因受影响组件跑了一个 `adbd`——典型**跨厂商命名碰撞**，「Android Debug Bridge」这个词把它误导进了「Android 漏洞」的直觉。任何 Android 漏洞统计与 SPL 口径都不应计入它。**KEV 归属核实（证伪）**：该 CVE 在 KEV JSON 中**无匹配** → 不在 CISA KEV（至少不在 2026.10.26 版）；「KEV 里存在某个 Tesla TCU 的 adbd 条目」这一说法被证伪。

---

## 7. 常见误传纠正

| # | 误传 | 纠正 |
|---|---|---|
| 1 | 车机文档站在 `android.auto.go` | 幻觉域名（NXDOMAIN / `http=000`）；真实入口是 `source.android.com/docs/automotive` |
| 2 | TV 文档在 `/docs/tv` | 真实路径 **`/docs/devices/tv`**；`/docs/tv` 实测 404 |
| 3 | Wear 有自己的 AOSP 文档章节 | 无任何 Wear 章节；`/docs/wear`、`/docs/devices/wear*`、`/docs/devices/tech/wear` 全 404 |
| 4 | Wear OS 是 AOSP 可构建分支 | 只有 `android-wear-*` 发布快照，无开发主线分支、无 `device/*wear*` 工程 |
| 5 | 存在 `aosp_*_wear` / `_wear` flavor | 199 个 `device/*` 工程里没有 wear 目录 → **不成立** |
| 6 | Android TV 是独立 AOSP 分支 | 主线 AOSP + TV 产品配置；TV 侧仅剩 `android-live-tv` 历史遗留分支 |
| 7 | `Leanback` 是公告里的 TV 组件 | 它是 AndroidX 库（`androidx-leanback-release`）与示例工程 `device/google/atv/LeanbackSampleApp`；公告正文 0 命中 |
| 8 | ASB 有 Automotive / TV / Wear 章节 | 2021-06→2026-10 逐期扫描从未出现；车机条目落在 `System`、`Platform apps` 等既有章节 |
| 9 | AAOS 公告每月都列一批 CVE | 2024-01→2026-02 的 26 期中仅 **2024-09 一期含 1 条**，其余均为 "There are no … patches" |
| 10 | VHAL 的 AIDL 桥接类叫 `VhalAidlBridge` | 该类在 AOSP 树中不存在；实名 **`DefaultVehicleHal.cpp`** / **`VehicleService.cpp`**（`impl/current/vhal/src`） |
| 11 | 车机用户叫 `SYSTEM_USER` / `VISITOR` | 查无出处。官方措辞为 Headless system user / Regular user / Guest user；成文标识 `android.os.usertype.full.SECONDARY` / `.GUEST` |
| 12 | 车机有 `FLAG_PRIMARY` 主用户 | 文档明确该 flag 在 Automotive headless system user 模型下「尚未支持」；所有常规用户都是 secondary |
| 13 | `CarUpdateService` 有官方文档页 | 未找到任何成文页（`/docs/automotive/system/upgrades`、`/docs/automotive/upgrade`、`/docs/core/updates` 均 404）；`/docs/automotive/security/update` 属通用 core topics 项目 |
| 14 | `dumpsys car_service` 是官方文档原文 | 未逐字出现；文档原文只有 `cmd car_service enable-feature car_remote_access_service` 与两条 HAL `dumpsys` |
| 15 | 车机上 `pm list users` 有官方示例 | 车机用户文档中未出现 `pm list users` / `pm create-user` 实例 → 未核实 |
| 16 | `VehiclePropertyAccess.aidl` 的 `ACCESS_*` / `SPEED` 字面值可引用 | 文件名真实、正文本次未取到 → 不得引用其常量拼写与属性名 |
| 17 | 存在车机专属 CDD：`/docs/automotive/compatibility/cdd` | 404，不存在；侧栏链接指向通用 `/docs/compatibility/cdd` |
| 18 | 公告 URL 形如 `…/bulletin/2025/2025-06-01` | 无年份目录层；`-05` 结尾 URL 已全部失效，每月只有 `-01` |
| 19 | `-gpl` 分支就是内核源码分支 | 后缀的官方定义未核实，只能陈述分支名事实 |
| 20 | KEV 里有 Tesla TCU 的 adbd 条目 | CVE-2025-34251 确属 Tesla TCU 固件（非 Google Android），且在该 KEV 快照中无匹配 |
| 21 | 「公告沉默 ⇒ 车机安全」 | 披露通道不覆盖 ≠ 风险低，这是观测缺口（§4.2、§6） |
| 22 | Android Go 有自己的代码分支 | `platform/manifest` 无任何 `android-go*` 分支，属构建目标 / 配置层形态 |

---

## 8. 可观测面与自查（只用已核实的 9 条命令/属性）

以下均为官方文档原文摘录，属**只读诊断 / 管理动作**；标注 `userdebug/eng` 者仅工程构建可用，生产 `user` 构建按 §3.1 应已被 `no_debugging_features` 关闭。一手页：`/docs/automotive/vhal/vhal_debug`、`/docs/automotive/remote_access`、`/docs/automotive/displays/cluster_api`、AAOS 公告 Common Q&A。

1. VHAL dump（AIDL，只读）：`adb root && adb shell dumpsys android.hardware.automotive.vehicle.IVehicle/default` —— 原文说明这调用 AIDL 服务隐式继承的 `dump()` 方法。
2. 列出参考 VHAL 支持的调试命令：`adb root && adb shell dumpsys android.hardware.automotive.vehicle.IVehicle/default --help`
3. 读取属性（资产清点用）：`… IVehicle/default --get INFO_VIN` 或 `--get 0x11100100`。`INFO_VIN` 是车辆识别码，属**需要管控的敏感暴露面**，不得进入未脱敏的日志导出。
4. 属性写入（管理动作，仅工程构建）：`… IVehicle/default --set SEAT_MEMORY_SELECT -a ROW_1_LEFT -i 1` —— 文档用它演示 area ID 语义；生产车队不应具备该通道。
5. HIDL 后端 VHAL 调试：`adb root && adb shell lshal debug android.hardware.automotive.vehicle@2.0::IVehicle/default`
6. 一致性验证：VTS 测试 `VtsHalAutomotiveVehicle_TargetTest`（用例 `useAidlBackend` / `useHidlBackend`，对应后端缺失时跳过；同时校验系统属性配置正确性）。
7. CarService 特性开关（userdebug/eng）：`adb shell cmd car_service enable-feature car_remote_access_service`
8. 远程访问服务只读诊断：`adb root && adb shell dumpsys android.hardware.automotive.remoteaccess.IRemoteAccess/default`
9. 补丁级别属性：`ro.build.version.security_patch` —— 成文声明形式取自 AAOS 公告原文："Device manufacturers that include these updates should set the patch string level to: `[ro.build.version.security_patch]:[2025-06-01]`"。

**边界声明**：除上述 9 条外，本手册不给出任何 car / TV / Wear 专属命令或 sysprop。`dumpsys car_service`、车机上的 `pm list users` 实例、`dumpsys media.tv`，以及形如 `ro.car.*` / `ro.tv.*` / `qemu.sf.*` 的键名，本次均**未在一手文档逐字取到**，属未核实。可安全引用的是**构建侧**真实名（不是运行期 sysprop）：`PRODUCT_MAKEFILES`、`COMMON_LUNCH_CHOICES`、`config_multiuserMaxRunningUsers`、`config_multiuserDelayUserDataLocking`、`config_defaultFirstUserRestrictions`。TV 侧可核的真实存在物是构建与包层名（`TvProvision`、`atv-component-overrides.xml`、`permissions`、`sepolicy`、`MdnsOffloadManagerService`、`blur_sysprop_notifier`）与文档页 `docs/devices/tv`；Wear 侧无可引用的 AOSP 文档页，观测面素材整体标注未核实。

---

## 9. 企业侧对车机 / TV / 穿戴的处置建议

1. **更新节奏承诺写进采购与合同**：AAOS 公告发布于**每月第一个周一**（节假日顺延），且 "AAOS devices start receiving OTA updates on the same day the monthly bulletin is released"；但固件由整车厂分发（"Please contact your device supplier for device firmware images."）。要求供应商逐期对齐 **ASB SPL + 当期 AAOS 公告**，并给出落到具体车型的时间差（TAT）。
2. **SPL 双轨核对**：车机补丁状态 = ASB 的 SPL **加上** AAOS 公告全部条目；只核对 `ro.build.version.security_patch` 不足以判定车机已完整更新。
3. **补丁来源分层与责任划分**：AAOS 补丁来源含**整车厂、AOSP、上游 Linux 内核、SoC 厂商**四类；`Upstream kernel` 类条目（如 §5 #14）需单独跟踪内核版本与车型适配。
4. **调试面默认关闭 + 令牌化例外**：按官方强烈建议，在**所有用户配置、所有生产构建**默认启用 `no_debugging_features`、`no_install_unknown_sources`、`no_install_unknown_sources_globally`；确需开发解锁时走 DRC + JWS 认证令牌，并保留审计记录。
5. **隔离与网络暴露面收敛**：以 `security/vehicle_system_isolation`（域隔离）、`security/macsec`（车载以太网加密）、`security/vehicle-bound`（车绑定文件加密）、`usb-reset`、`watchdog` / `flash-wear`（可用性与磨损）为检查表落地；远程访问是**需显式 opt-in** 的功能，未启用就不该在车型上暴露该通道，启用则把任务注入入口纳入变更管理与监控。
6. **高危权限与属性订阅清单审查**：`android.car.permission.CAR_INSTRUMENT_CLUSTER_CONTROL`、`BIND_INSTRUMENT_CLUSTER_RENDERER_SERVICE`、声明 `android:singleUser="true"` 的簇屏渲染服务，以及 VHAL 侧 CreateUser / SwitchUser / RemoveUser、OBD2、ADAS 状态属性，应与「可安装应用」同级纳入权限评审。
7. **日志采集合规**：车机日志要区分 **User 0（headless system user）与前台用户**，不要套用移动端「主用户」假设；理解「已切走的用户仍可能有 persistent 服务在运行」；VIN、账号、位置类信号按个人数据处理并在导出前脱敏；guest 默认 ephemeral，事后取证窗口有限，需在事件响应流程里预先声明。
8. **别把「公告沉默」当风险结论**：TV、Wear 与车机 VHAL/CarService 方向在 ASB/KEV 中缺乏条目，应改为按「厂商更新承诺 + 组件构成 + 暴露面清单」评估，并为 TV / Wear 单独建立厂商侧文档基线（其官方更新节奏说明本次未核实，需向供应商索取书面承诺）。

---

## 10. 与其他手册的分工

- 《多用户与工作资料隔离》([android-multiuser-sandbox-isolation.md](/android-multiuser-sandbox-isolation/)) 讲手机的 Secondary / Work Profile / Guest 通用模型；本手册只讲**车机的 headless system user 模型**（User 0 + 1 前台 + 1 后台、`FLAG_PRIMARY` 尚未支持、guest 仅前台且 ephemeral、前后台切换时 persistent 服务的存活规则）。凡涉及车机用户语义以本手册 §2 为准，通用隔离与工作资料语义以该手册为准。
- 《漏洞编号与公告体系》([android-cve-and-security-bulletins.md](/android-cve-and-security-bulletins/)) 负责 CVE 与 `A-<bug id>` 编号规则、SPL 定义、公告结构与 URL 形制的通用解释；本手册只把「变体在公告体系中的位置」作为结论引用（ASB 无 Automotive/TV/Wear 章节、AAOS 是补充通道、公告 URL 无年份目录层、`-05` 已失效）。
- 《基带与电话子系统》([android-baseband-telephony.md](/android-baseband-telephony/)) 覆盖调制解调器与电话栈；车机的远程信息处理（TCU）与 IVN（`ivn_android_device`）在本手册只按 AOSP 目录与远程访问文档描述，基带侧细节一律归该手册。注意本手册 §6.1 已明确：Tesla TCU 的 CVE-2025-34251 **不属于** Android 范畴，不进入任何 Android 公告/SPL 统计。
- 与同批其他专题手册（底层组件漏洞类型、Mainline / APEX 模块、TEE 与密钥认证、生物识别与锁屏、RCE 与权限提升、严重性评估、ADB 与备份传输等，均以书名号指代、不重复展开）的分工原则是：**那边讲手机侧的组件成因与机理，这边讲变体设备的成文边界与可观测缺口**。本手册不重复内存破坏机理与打分方法，相关基础见 [android-common-system-vulnerabilities.md](/android-common-system-vulnerabilities/) 与 [android-vulnerability-severity-assessment.md](/android-vulnerability-severity-assessment/)。

## 11. 未核实与来源失效

以下**已尝试但未能核实**，正文不作为事实引用；复用本素材时必须保留「未核实」标记或直接删去。

1. **`android.auto.go`**：幻觉域名（NXDOMAIN、`http=000`），不得作为任何官方来源引用。
2. **`SYSTEM_USER` / `VISITOR`**：查无出处的常量名；官方措辞见 §2.1（Headless system user / Regular user / Guest user）。
3. **`VhalAidlBridge`**：AOSP 当前树中不存在该类；`VhalController` 是否在 `impl/1.0` 等历史版本中存在亦未核实（本次只遍历 `impl/3`、`impl/current`）。
4. **`CarUpdateService`**：官方文档页与 AOSP 路径均未定位（试 `/docs/automotive/system/upgrades`、`/docs/automotive/upgrade`、`/docs/core/updates`，均 404）。提及它必须另行给出源码出处。
5. **`dumpsys car_service`**：仅逻辑成立（簇屏页要求实现 `Service#dump()` 并提示 "see `dumpsys` for more information"），文档未逐字出现该命令串。
6. **车机上 `pm list users` 的官方示例**：未取到；`users_accounts/accounts` 页正文未展开。
7. **CDD 中「无蜂窝 / 无显示 / 资源受限」条款正文**：只取到 `/docs/compatibility/cdd` 与 `/docs/whatsnew/latest-cdd` 落地页；且 `/docs/automotive/compatibility/cdd` 实测 404，不存在。
8. **Wear OS / Android TV 的安全更新节奏官方说明**：Google 产品侧文档 URL 未取得；`source.android.com` 无 Wear 章节。
9. **`_go` / Android Go 的 AOSP lunch 目标名**：`build/make/+/refs/heads/main`（含 `target/product` 子路径）在 gitiles 404；`device/generic/goldfish/AndroidProducts.mk` 未匹配到 `go` 目标。
10. **`VehicleProperty.aidl` / `VehiclePropertyAccess.aidl` 正文**：`ACCESS_*` 常量拼写、`SPEED` 相关属性名、`CLUSTER` / `DISPLAY` 的 VHAL 枚举字面值均未取到（单文件 `?format=TEXT` 返回 404，而同目录列表返回 200；未改用 `aidl_api/<ver>/current/` 冻结文件重试，轮次耗尽）。
11. **`-gpl` 后缀分支（`android-wear-11/13/14/15.0.0-gpl_r*`）的官方定义**：仅证实分支名字存在。
12. **部分 CVE 的 NVD 英文描述与 CVSS 数值**：CVE-2024-40667、CVE-2021-1040、CVE-2021-1036、CVE-2021-1037（素材另点名 CVE-2021-39738）API 返回空；§5 各条的 NVD 侧级别未逐条取回，因此严重级别「双轨」只给标注原则、不给数值。
13. **AAOS `2022-01-01` 期的组件章节标题**：`<h2>/<h3>` 提取未命中（该期表格结构不同），表中记为「该期章节名未取到」。
14. **`dumpsys media.tv` 与 `ro.car.*` / `ro.tv.*` / `qemu.sf.*` 等变体专属 sysprop**：未在一手文档逐字取到，不得引用。
15. **来源形制提醒（已失效）**：`-05` 结尾公告 URL 全部 404（2022-12-05、2023-12-05、2024-12-05、2025-12-05、2026-01-05 … 2026-10-05）；`hardware/interfaces/…` 裸路径 404（需 `platform/` 前缀）；§1.7 列出的路径清单同样失效。

## 12. 参考

一手（基址 `https://source.android.com`）：`/docs/automotive` 及子页 `users_accounts/{multi_user,user_system,accounts,user_hal,disable_packages}`、`security/{secure_dev_options,drc,macsec,vehicle-bound,vehicle_system_isolation,mfg_guide,update}`、`remote_access`、`vhal`、`vhal/{vhal-interface,property-configuration,special-properties,vhal_debug,adas-properties}`、`displays/cluster_api`、`camera/{acs,evs}`、`sdv`（`core-areas/vsidl`、`some-ip-integration`、`sdv-system-architecture`、`sdv-gateway-on-ivi`、`display-safety`）、`ivi`、`virtualization`、`usb-reset`、`watchdog`、`flash-wear`、`feature-control`、`tools`、`tools/fuzz`、`dev-tools`、`start/releases`；`/docs/devices/tv`；`/docs/compatibility/cdd`、`/docs/whatsnew/latest-cdd`。

公告：`/docs/security/bulletin`、`/docs/security/bulletin/aaos` 及各期 `aaos/2021-10-01`、`aaos/2021-12-01`、`aaos/2022-01-01`、`aaos/2022-02-01`、`aaos/2022-12-01`、`aaos/2023-01-01`、`aaos/2023-02-01`、`aaos/2023-07-01`、`aaos/2024-09-01`、`aaos/2025-06-01`、`aaos/2025-10-01`；ASB 逐期扫描 `https://source.android.com/docs/security/bulletin/YYYY-MM-01`（2021-06 … 2026-10）。

代码：`https://android.googlesource.com/platform/manifest/+refs`；`https://android.googlesource.com/?format=JSON`；`device/google_car/+/android-automotiveos-15.0.0_lts5/AndroidProducts.mk`；`device/generic/car/+/refs/heads/main/AndroidProducts.mk`；`device/google/atv/+/refs/heads/main/products/`；`platform/hardware/interfaces/+/refs/heads/main/automotive/vehicle/`（及 `aidl`、`aidl_api/android.hardware.automotive.vehicle`、`aidl/impl/{3,current}`、`aidl_property`、`2.0`）、`automotive/remoteaccess`。

外部数据：NVD REST API `https://services.nvd.nist.gov/rest/json/cves/2.0?cveId=CVE-…`（含 CVE-2025-34251）；CISA KEV `https://www.cisa.gov/sites/default/files/feeds/known_exploited_vulnerabilities.json`（`catalogVersion 2026.10.26`，`count 1758`）。

## 13. 联系方式

咨询 Android 系统详情请联系 https://t.me/one00190

*本手册仅供安全学习与防御研究使用；所有组件事实、接口名称与案例归属均取自公开一手来源并附链接，未能核实的表述已单列于「未核实与来源失效」一节而非写入正文。*
