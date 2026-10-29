---
layout: page
title: "Android 供应链与构建完整性手册"
description: "为什么「代码从哪来、产物怎么被验证」也是安全边界：AOSP 源码的官方获取口径与发布节奏、官方认可的 Git tag PGP 验签、`make dist` 与 `make` 两条构建刷机路径的差异、`super`/`lpmake` 动态分区里必须保持物理的三类、产物侧签名锚点、SBOM 的官方生成路径与..."
date: 2026-10-29
permalink: /android-supply-chain-build-integrity/
---

> 本手册把「Android 源码从哪里发布、构建产物如何被验证、刷机包如何分区与签名」当成一条可审计的供应链来写：AOSP 源码的官方形态与获取口径、2025→2026 的发布节奏变化、Git tag 的 PGP 验签、`make dist`（target_files / img.zip）与 `make`（PRODUCT_OUT）两条官方构建与刷机路径的差别、动态分区与 `lpmake`、产物侧的签名锚点（`vbmeta`、GKI kernel build-time key pair、OTA release signing keys）、SBOM 生成，以及 defender 视角下的治理动作与可观测点。面向 ROM/固件维护者、企业 IT 与 MDM 管理员、供应链审计人员和安全学习者。内容全部来自 source.android.com、googlesource、NVD、CISA KEV 等公开一手来源，只写已公开且已修复的信息，不含任何可利用步骤或投毒细节；未能核实的表述集中列在最后一节，不写入正文。

---

## 0. 使用方式与边界

- **只写公开、已修复的信息**：案例仅引用官方安全公告与 NVD 已公开描述；不给 PoC、不给利用链、不给绕过验签 / 重签名 / 篡改发布物的任何操作细节。
- **一手来源纪律**：本手册出现的组件实名、分区名、工具名、命令与英文引号句，全部来自核验稿对页面的抓取（抓取日 2026-10-27）并附链接；引号内为原文逐字摘录。
- **公告链接形式**：一律写成 `https://source.android.com/docs/security/bulletin/YYYY-MM-01`，**没有** `/YYYY/` 目录层（这是核验时确认的正确 URL 规则）；且只用于真实核实过的 CVE。
- **KEV 口径**：未命中 CISA KEV 的条目写「CISA KEV 未收录」，并始终附注「未收录 ≠ 无真实利用」。KEV 是行政目录（服务于美国联邦机构的 remediation 义务），收录与否由 CISA 判定。
- **分工边界**：AVB 四色状态机与 Rollback Protection、A/B 槽位与 payload 校验、降级防护归《Android 系统更新链路与回滚防护手册》和 Verified Boot 侧；本篇只写「构建产物如何被签名/校验」与「发布渠道与镜像完整性」这一侧。
- **观测缺口要明说**：Overlay/RRO、`privapp-permissions` 允许清单、VTS/CTS/STS 框架自身这几个方向，在已解析的公告行里**没有任何 CVE 条目**。本手册如实写出这一缺口，不编造案例。观测缺口 ≠ 风险缺口。
- **未核实内容**：素材核验稿标注「查无官方出处 / 未核实 / folklore」的条目，一律不进入正文事实，只出现在末尾「未核实与来源失效」一节。

## 1. 为什么「代码从哪来、产物怎么被验证」也是安全边界

Android 安全讨论通常集中在运行时代码缺陷（框架、HAL、内核）。供应链视角把问题往前推两层，这两层都有独立的攻击面，也都有独立的官方锚点可以审计：

- **来源层**：你构建或刷入的东西，是否真的来自 Google 发布的那份源码，并且对应你想要的那个 tag / 构建号。
- **产物层**：编译产物（分区镜像、`super`、OTA 包）在交付链路上被谁签名、用什么密钥签名、设备端拿什么锚点校验。

官方文档对这两层的成文支撑（均一手核实）：

- 来源层上，官方对「来源可疑」的处置方式是**让使用者自己验签**，而不是发布镜像状态类通告：
  > "(optional) Verify code legitimacy — If you're concerned about the legitimacy of the source code, such as whether it came from Google, you can verify the Git tags for the branch."
- 产物层有三条官方签名锚点（§5、§7 展开）：`vbmeta` 承载全部分区的 Verified Boot 信息；GKI 模块由 Google 用 kernel build-time key pair 签名；OTA 包 "everything is signed with the test key. For user devices, generate and use your own private keys as detailed in Signing builds for release."
- 规范层上，CDD 自己写明「代码即规范」：
  > "Since Android is a single corpus of open-source code, the code itself is the comprehensive \"specification\" of the platform and its APIs."
  防御含义：任何绕过 AOSP 源码、只信二进制的做法都没有规范锚点——这正是审计时要追问「这份产物对应哪个源码 tag」的依据。


## 2. AOSP 源码的官方形态与获取口径

**官方对「源码在哪」的定义**（一手原句）：

> "The Android source is located in a collection of Git repositories hosted by Google. The Git repository includes the entire history of the Android source, including changes to the source and when the changes were made."

**官方 `repo init` 命令**（逐字符抄录；2026 年默认分支已是 `android-latest-release`）：

```
repo init --partial-clone --no-use-superproject -b android-latest-release -u https://android.googlesource.com/platform/manifest
```

- `-b` 只接受分支名或 tag 名：
  > "The `-b` option is used to identify the branch you're initializing. For a list of branches and tag names, see Source code tags and builds."
  > "Use tag names or branch names with the `-b` option."
- `-u` 指定 manifest（描述各 Git project 在工作目录中位置的 XML）：
  > "In this example, the name of the manifest file isn't specified, so the command uses the default manifest file (`default.xml`)."

**同步命令**：

```
repo sync -c -j8
```

**manifest 格式的权威定义不在 AOSP 文档，而在 git-repo 仓库**：AOSP 页面写 "For information on manifest file format, see repo Manifest Format."，链接目标为 `https://gerrit.googlesource.com/git-repo/+/main/docs/manifest-format.md`。

**三个域名的官方分工**（核验到的真实表述，也是本篇纠正误传的基础）：

> "Find the Android Gerrit review interface at android-review.googlesource.com and the code navigation interface at android.googlesource.com."

即：`android.googlesource.com` 是代码浏览/导航入口（同时是 manifest 仓库宿主），`android-review.googlesource.com` 是 Gerrit 评审入口，`gerrit.googlesource.com` 承载的是 **git-repo 工具本身**。



入口 `https://cs.android.com/android/platform/superproject`（该 URL 出现在 `/docs/setup/download` 页面正文）。Repo 启动器官方描述为 Python 脚本："The Repo Launcher provides a Python script that initializes a checkout …"，并用 manifest 聚合 Git project（"Repo uses manifest files to aggregate Git projects into the Android superproject."）。

**专有二进制侧**（同一页面官方口径，审计时不能忽略）：

> "The binaries and their matching makefiles are installed in the vendor/ hierarchy of the source tree."
> "AOSP can run on Cuttlefish emulators directly, but AOSP can't be used on hardware without additional device-specific proprietary libraries."

第三方组件源码索取：

> "To obtain the source code for third-party components where the open source license grants you the right to receive the source, see Obtain Android source."

**官方下载页实名**（`/docs/setup/reference/build-numbers`，原文照抄，注意 AOSP 页上确实写作 "Drive binaries"）：`Factory Images for Nexus and Pixel Devices`、`Drive binaries for Nexus and Pixel Devices`、`Full OTA images for Nexus and Pixel Devices`；已核实的真实外链域为 `developers.google.com/android/images/`。

**tag 与分支的真实命名规则**（`build-numbers` 表格 2026-10 抓样）：`main` 是开发分支名；`android-<ver>_rN` 与 `android-security-<ver>_rN` 是 tag，例：`android-security-17.0.0_r1`（build ID 15848949，SPL 2026-05-05）、`android-17.0.0_r1`（构建码 CP2A.260605.016，SPL 2026-06-05）、`android-16.0.0_r4`（BP4A.251205.006）。tag 末尾的 `_rN` 是同一版本的第 N 次 AOSP 推送（重推/补推），**不是**补丁级别。


**两版之间「到底改了什么」的官方取证命令**：

```
repo forall -pc 'git log --no-merges --oneline branch-1..branch-2'
```


## 3. 发布节奏：`android-latest-release` 与 2026 年起的 Q2/Q4 推送

**2026 年新政（官方横幅原句，最重要的一条）**：

> "Effective in 2026, to align with our trunk stable development model and ensure platform stability for the ecosystem, we will publish source code to AOSP in Q2 and Q4. For building and contributing to AOSP, use `android-latest-release`. The `android-latest-release` manifest branch will always reference the most recent release pushed to AOSP."

**上一轮变化（2025-03-27）官方原句**：

> "Starting March 27, 2025, the latest release branch will always be referenced by the new android-latest-release manifest, which can be used directly with Repo. … The android-latest-release manifest is set to the latest AOSP release branch, android17-release."

**核对结论（针对「Android 15+ 是否只发布季度快照」这一常见问法）**：官方成文口径**不是**「季度安全快照」，而是两条叠在一起：① 2025 年起用 `android-latest-release` 指向最新 release 分支；② 2026 年起 AOSP **只在 Q2 与 Q4 推源码**。安全补丁级别（SPL）仍然按月发布（见 `/docs/whatsnew/latest-security-bulletins` 与每月公告本身）。因此「Android 15+ 只有季度快照」属于**过度简化**：季度化的是 AOSP 源码推送节奏，不是 SPL 与公告节奏。

对 defender 的三点实际影响：

- **「设备构建号对应哪次 AOSP 推送」变成可核对问题**：`android-latest-release` → 当前 `android17-release` → `/docs/setup/reference/build-numbers` 表里的 `Build ID ↔ Tag ↔ Version ↔ Security patch level ↔ 构建码`（如 `android-17.0.0_r1 ↔ CP2A.260605.016 ↔ SPL 2026-06-05`）官方映射。这张表是「设备所见 vs 已发布产物」比对的权威锚点。
- **不能用 AOSP 推送频率推断补丁基线**：一次 AOSP 推送覆盖两个季度的平台改动，而月度 SPL 公告持续发布；判定基线永远是 SPL 与公告，不是「最近有没有 AOSP 推送」。
- **公告侧关于「补丁何时可在 AOSP 查见」有官方措辞**，可直接用于判定时间窗：
  > "Source code patches for these issues will be released to the Android Open Source Project (AOSP) repository in the next 48 hours. We will revise this bulletin with the AOSP links when they are available."
  合作伙伴侧另有：
  > "Android partners are notified of all issues at least a month before publication."


## 4. 来源验证：官方认可的 Git tag PGP 验签

官方成文做法是两步命令（均为一手原句）：

```
gpg --import keyfile.asc
git tag -v TAG_NAME
```

> "Run the following command to input a public key into your GnuPG key database. The key is used to sign annotated tags that represent releases."
> "After importing the keys, you can verify any tag by running this command:"

**公钥从哪里取**：`/docs/setup/download` 页面本身印有 AOSP 用于签署 annotated tag 的 PGP 公钥块（`-----BEGIN PGP PUBLIC KEY BLOCK-----`，UID 为 `The Android Open Source Project <initial-contribution@android.com>`）。核验者应从**该页面本身**取键，而不是从第三方镜像取键——否则验签的信任根就变成了待验证对象，整条链失去意义。

**这套做法能证明什么**：

- 某个 **annotated tag 对象**由该 PGP 密钥签名，且签名有效；
- 你使用的公钥与 Google 在同一页面公布的一致；
- 结合 `repo init -b <tag>`，可证明你 checkout 得到的 commit 集合与该 tag 指向的 commit 集合一致；
- 结合 `repo forall -pc 'git log --no-merges --oneline tagA..tagB'`，可把「两个 AOSP 版本之间的差异」变成可列举、可留档的取证结果。

**不能证明什么**（这一区分是本节要点）：

- **不覆盖未打 tag 的对象**：分支 HEAD、某个下载来的 zip、某台设备里现存的镜像、第三方渠道分发的 factory image 都不在 tag 验签的作用域内；这些要靠 §5 的产物路径与 §7 的签名锚点。
- **不证明语义正确**：验签只锚定「来源 + 完整性」，不锚定「代码里没有恶意逻辑」。入库侧官方成文的门槛只有一句 "automatically merged if they pass code review"；评审的具体门禁细节（`+2`、`verified` 标签）本次查无成文描述，本手册不写。
- **不替代产物侧校验**：镜像是否被替换要看 `vbmeta`/Verified Boot，OTA 包要看 release signing keys（§7）。


## 5. 两条官方构建与刷机路径：`make dist` 与 `make`

这是供应链侧最容易被混淆的一点，官方给了一句**直接区分二者**的原句：

> "You must flash the images in PREVIOUS-tardis-target_files.zip or PREVIOUS-tardis-img.zip (both built with make dist, to be flashed with fastboot update), instead of the ones under the PRODUCT_OUT directory (built with make, which will be flashed with fastboot flashall)."

按官方口径拆开读：

| 产出方式 | 产物 | 官方刷机方式 | 在链路中的角色 |
|---|---|---|---|
| `make dist` | `*-target_files.zip` | 不直接刷 | 重打包/签名的**中间权威产物**：分区镜像 + 元数据，供 `ota_from_target_files` 消费 |
| `make dist` | `*-img.zip` | `fastboot update` | 交付/测试用刷机镜像包 |
| `make` | `PRODUCT_OUT` 目录下的散装镜像 | `fastboot flashall` | 开发调试产物，不属于发布链路 |

**dist 构建的官方命令**：

```
. build/envsetup.sh && lunch tardis-eng
mkdir dist_output
make dist DIST_DIR=dist_output
```
> "make dist builds a full OTA package (in $OUT). The resultant .zip file contains everything needed to construct OTA packages for the tardis device."

**从 target_files 造 OTA 包**（工具实名 `build/make/tools/releasetools`）：

```
ota_from_target_files dist_output/tardis-target_files.zip ota_update.zip
ota_from_target_files -i PREVIOUS-tardis-target_files.zip dist_output/tardis-target_files.zip incremental_ota_update.zip
```

**供应链侧三条要点**：

- **增量包与「起点构建」强绑定**，这是发布纪律而不是实现细节：
  > "Distribute an incremental package only to devices that run exactly the same previous build used as the incremental package's starting point."
- **测试密钥与发布密钥的分界**是官方明文写的：
  > "ota_update.zip is now ready to be sent to test devices (everything is signed with the test key). For user devices, generate and use your own private keys as detailed in Signing builds for release."
- **审计含义**：如果一台「量产」设备的产物只能追到 `PRODUCT_OUT` 散装镜像 + `fastboot flashall` 路径，而追不到 `make dist` 产出的 target_files，就说明它没有经过发布链路应有的重打包与签名环节；这是 ROM 完整性自查里最先该问的问题。
- 差分包工具 `imgdiff` 的位置与归属在本次素材中**没有任何一手来源**，因此本手册不把它当作已核实名字使用（见末尾清单）。

## 6. 分区与动态分区：`super`、`lpmake` 与必须保持物理的三类



**官方三分类与 Treble/VINTF**：

> "System partitions are partitions that are updated when updating the OS and other features. The system, boot, and init_boot are core system partitions."
> "Vendor partitions contain device and hardware-specific code that might never be updated after initial release. The vendor, vendor_boot, and odm partitions are core vendor partitions."
> "Nonupdatable partitions are partitions whose contents are either not updated or updated with user data."
> "Code in system and vendor partitions can interact using a stable interface called the vendor interface (VINTF)."

**一手核实到的分区实名清单**（审计产物的目录基线）：`system`、`system_ext`、`system_dlkm`、`product`、`vendor`、`vendor_boot`、`vendor_dlkm`、`odm`、`odm_dlkm`、`boot`、`init_boot`、`vbmeta`、`recovery`、`radio`、`pvmfw`、`generic_bootloader`；非可更新侧 `cache`、`userdata`、`metadata`；`dtbo` 由动态分区页核实（bootloader 必读、须为物理分区）。几条与产物来源直接相关的官方定义：

> "system_dlkm partition. This partition contains GKI modules."
> "vbmeta partition. This partition contains the Verified Boot information for all of the partitions. This information verifies that the images installed in each partition is trusted."
> "vendor partition. This partition contains any binary that is vendor specific and not generic enough to distribute to AOSP."





**动态分区与 `super`**（`/docs/core/ota/dynamic_partitions/implement`）：

> "Dynamic partitions are implemented by the device-mapper (dm) module in the Linux kernel. The super partition contains …"
> "Because dynamic partitions are implemented in userspace, partitions needed by the bootloader can't be made dynamic. For example, boot, dtbo, and vbmeta are read by the bootloader, and so must remain as physical partitions."
> "The supported dynamic partitions are: System / Vendor / Product / System Ext / ODM"
> "The super partition block device must be marked with the label super_block_device."

**官方点名的构建工具是 `lpmake`**：

> "…the build system (via lpmake, which generates the super partition image), assumes that a 1 MiB alignment …"

审计 OTA/镜像重打包脚本时，`lpmake` 与 `super_block_device` 是官方给出的两个名字锚点：`super` 镜像由谁生成、按什么对齐生成、里面动态分区的尺寸表是否与 target_files 元数据一致，都从这两处入手。动态分区镜像与发布产物的接口也有原句："These images are placed in the OTA folder in target_files.zip."

总页对动态分区的定位补充："Devices running Android 11 and higher can support dynamic partitions, which are a userspace partitioning system for Android that lets you create, resize, or destroy partitions during over-the-air (OTA) updates."


## 7. 产物侧签名锚点与规范锚点

素材核实的、属于「构建产物侧」的成文事实只有三条，本手册只写这三条（AVB 四色状态与回滚保护归更新/Verified Boot 侧）：

1. **`vbmeta`**："This partition contains the Verified Boot information for all of the partitions. This information verifies that the images installed in each partition is trusted."（见 §6）——设备端判断「镜像是否被替换」的锚点。
2. **GKI 模块由 Google 签名**（`/docs/core/architecture/partitions/gki-partitions`，本篇核心一手句）：
   > "GKI modules are signed by Google using the kernel build-time key pair and are compatible only with the GKI they're built with."
   > "Note: Any device launching with Android 13 and higher must include a system_dlkm partition."
   官方成文的 `BoardConfig.mk` 开关（逐字符，非杜撰）：
   ```
   BOARD_USES_SYSTEM_DLKMIMAGE := true
   BOARD_SYSTEM_DLKMIMAGE_FILE_SYSTEM_TYPE := $(TARGET_RO_FILE_SYSTEM_TYPE)
   TARGET_COPY_OUT_SYSTEM_DLKM := system_dlkm
   BOARD_GOOGLE_SYSTEM_DYNAMIC_PARTITIONS_PARTITION_LIST := system_dlkm
   AB_OTA_PARTITIONS += system_dlkm
   ```
3. **OTA 包签名密钥**：测试密钥 vs 厂商自有私钥（§5 原句）。

`dm-verity` 在 `/docs/security/features/verifiedboot`（HTTP 200）确有成文提及 "dm-verity kernel feature."，但该页本次只取到片段行，故此处仅作**定位引用**，不作定义引用。KMI / `CONFIG_` / symbols 一类机制名本次未核实，不写。

**规范锚点**（为什么审计必须回到源码）：

> "An Android-compatible device must adhere to the requirements of the compatibility definition document (CDD), and pass the compatibility test suite (CTS)."
> "The CDD represents the \"policy\" aspect of Android compatibility."
> "If you want to build a device compatible with a given Android version, start by checking out the source code for that version, and then read the corresponding CDD and stay within its guidelines."

各版本 CDD 是独立成文页（`/docs/compatibility/9/android-9-cdd` … `/docs/compatibility/17/android-17-cdd`，导航实名，最新为 Android 17）。测试面实名与定位（`/docs/compatibility/vts`）：

> "The Android Vendor Test Suite (VTS) provides extensive testing on the following: Kernel / Hardware abstraction layer (HAL)"

导航中真实存在的条目还包括 `Trade Federation` 与 `Security Test Suite`。**观测缺口声明**：在已解析的 80+ 份公告正文中，未检索到以 VTS/CTS/STS 测试框架自身为组件段的 CVE 条目（公告不以其为组件段）。这是观测缺口，不等于无风险。

## 8. SBOM：官方生成路径与它能回答的问题

**AOSP 确实有成文页**（任务假设的「AOSP 不提供 SBOM 文档」不成立）：`https://source.android.com/docs/setup/create/create-sbom`，标题 "Create a software bill of materials (SBOM)"。

> "In February 2022, the National Institute of Standards and Technology (NIST) published version 1.1 of the Secure Software Development Framework (SSDF), a set of comprehensive guidelines on secure software development practices in response to the 2021 Cybersecurity Executive Order (EO) 14028. As part of these requirements, the US government might request a software bill of materials (SBOM), which lists components of a software release."
> "Note: SBOMs generated using the steps on this page include the Minimal Elements for a Software Bill of Materials (SBOM) as published by the National Telecommunications and Information Administration (NTIA). Additionally, Android-based SBOM tooling and product SBOMs are compatible with the Software Package Data Exchange (SPDX) 2.3 format …"

**取 CI 预生成 SBOM**（官方步骤实名）：

> "In your browser, navigate to ci.android.com. In the Enter a branch name field, type aosp-android-latest-release."

**本地自行生成**（逐字符命令）：

```
$ source build/envsetup.sh
$ lunch TARGET
$ m sbom   # Generates an SBOM
$ ls out/dist/sbom*
```



**能回答 / 不能回答**：能回答「这次发布包含哪些组件」「厂商加了什么（含二进制与构建/发布工具链）」「两份产物的组件清单差异」「是否可对齐 SPDX 2.3 与 NTIA Minimal Elements」。不能回答「组件有没有被篡改」（那是 §4 验签与 §7 产物签名的职责）、不能回答可利用性。本次素材未核实 SBOM 的覆盖边界（是否包含 vendor 闭源 blob、`radio` 镜像），故不得声称「SBOM 覆盖全部产物」。许可与来源索取的成文入口（导航实名）：`/docs/setup/contribute/licenses`、`/license`、`/opensourcerequest`、`/source/requirements`。

## 9. 公开已修复案例：产物与发布链路侧

以下条目均由本地缓存的 `source.android.com` 公告正文与 NVD CVE API 交叉核实；只写归属与防御含义，不写利用方式。

1. **CVE-2018-9567**（本篇最核心的一条）——公告 `https://source.android.com/docs/security/bulletin/2018-12-01`，行原文 `CVE-2018-9567 | A-65543936* | EoP | High | Bootloader`；NVD CVSS v3 `7.8 HIGH`；Bug 尾带 `*` 表示补丁不在 AOSP。NVD 描述原文：
   > "On Pixel devices there is a bug causing verified boot to show the same certificate fingerprint despite using different signing keys. This may lead to local escalation of privilege if people are relying on those fingerprints to determine what version of the OS the device is running, with System execution privileges need…"
   归属意义：**签名信息被错误呈现**的「构建产物侧」缺陷——如果运维方拿证书指纹判断「这是哪个 OS 版本/谁的签名」，这个锚点历史上确实不可靠。CISA KEV 未收录（未收录 ≠ 无真实利用）。
2. **CVE-2022-22706**（在野利用信号条目）——公告 `https://source.android.com/docs/security/bulletin/2023-06-01` 正文注记：
   > "Note: There are indications that CVE-2022-22706 may be under limited, targeted exploitation."
   NVD 描述原文：
   > "Arm Mali GPU Kernel Driver allows a non-privileged user to achieve write access to read-only memory pages. This affects Midgard r26p0 through r31p0, Bifrost r0p0 through r35p0, and Valhall r19p0 through r35p0."
   NVD `7.8 HIGH`，且带 CISA 字段 `cisaExploitAdd: 2023-03-30`、`cisaActionDue: 2023-04-20`；CISA KEV **收录**（`cveID` 精确命中）。
3. **CVE-2024-26926**——公告 `https://source.android.com/docs/security/bulletin/2024-06-01`，Kernel 段，`EoP` / High，参考 `A-320661088` + `Upstream kernel [2]`，Subcomponent `Binder`；NVD 原文以 "In the Linux kernel, the following vulnerability has been resolved: binder: check offset alignment in binder_get_object() …" 起头。意义：GKI/upstream kernel 路径，公告把它单列并给上游内核引用，是「AOSP 之外补丁来源」的一手例子（对应 "links to patches outside of AOSP"）。KEV 未收录（≠ 无真实利用）。
4. **CVE-2024-0671 / CVE-2024-1065**——同一条 2024-06-01 公告中 Kernel 段之后的**闭源组件**表，Subcomponent 均为 `Mali`，`EoP` / High，参考分别 `A-329094549 *`、`A-329096276 *`（NVD 分别 `6.8 MEDIUM`、`5.9 MEDIUM`，描述均为 Arm Ltd GPU Kernel Driver 的 Use After Free）。意义：这类补丁落在 **vendor 侧内核态驱动**，即 `vendor_dlkm`/`odm_dlkm` 承载的内核模块；两者不在 AOSP，所以「AOSP 有没有补丁」对这类条目回答不了问题，必须回到厂商 OTA。KEV 未收录（≠ 无真实利用）。
5. **CVE-2024-31317**——同一公告 CVE 行原文 `A-316153291 | EoP | High | 12, 12L, 13, 14`（本次唯一完整取到 Updated AOSP versions 列的一条；所属章节名未解析出，故不写章节）。NVD 原文：
   > "In multiple functions of ZygoteProcess.java, there is a possible way to achieve code execution as any app via WRITE_SECURE_SETTINGS due to unsafe deserialization. This could lead to local escalation of privilege with User execution privileges needed. User interaction is not needed for exploitation."
6. **CVE-2021-0703**——NVD 原文 "In SecondStageMain of init.cpp, there is a possible use after free due to incorrect shared_ptr usage…"，`6.8 MEDIUM`，组件是 `init`（分区挂载/早期引导的构建产物侧）。**其公告日期与章节名本次未定位到**（缓存的 2021-09-01 / 2021-09-05 内无该 CVE 行），故此处只给 NVD 归属，不指定公告章节。

**CISA KEV 批量核对**（数据源 catalogVersion `2026.10.26`、`count` 1758，精确匹配 `cveID`；自检正例 CVE-2020-0041 命中 1 次）：CVE-2022-22706 收录；CVE-2018-9567、CVE-2021-0703、CVE-2024-26926、CVE-2024-0671、CVE-2024-1065、CVE-2024-31317 均未收录。**推论边界**：命中 KEV 视为已确认在野利用；未命中不得反向声称「未被利用」，应补充引用公告注记与 NVD 的 `cisaExploitAdd` 字段。

**明确排除，避免误归**：CVE-2023-21127（`readSampleData of NuMediaExtractor.cpp`，RCE/High 8.8，AOSP 11–13，`A-275418191`）属 Media Framework，不是构建/产物面；CVE-2020-0041（`binder_transaction of binder.c`）属 IPC/内核 binder，归框架与 IPC 侧。**该方向未检索到条目的主题**（禁止编造）：Overlay/RRO、`privapp-permissions` 与预装应用机制、VTS/CTS/STS 框架自身、以 `super`/动态分区元数据为缺陷对象的条目（最接近的只有第 1 条）。

## 10. 分角色处置清单（只做观测与治理，不含利用）

- **设备管理 / 企业 IT**：把设备上报的构建码与 SPL 逐项比对 `/docs/setup/reference/build-numbers` 官方表；对 CVE-2022-22706 这类 KEV 命中项按「已确认在野利用」排优先级；对 KEV 未命中但公告带 "indications … exploitation" 注记的条目按同等信号升级处理；不要以「AOSP 本季度没推送」作为「无补丁可跟」的理由（SPL 仍按月）。
- **ROM / 固件维护者**：发布链路必须走 `make dist` → `target_files.zip` → `ota_from_target_files`，并留档每次发布的 target_files；禁止用 `PRODUCT_OUT` 散装镜像 + `fastboot flashall` 产物充当量产交付；用户版不得沿用 test key；确认 `boot`/`dtbo`/`vbmeta` 仍是物理分区、动态分区只使用 System/Vendor/Product/System Ext/ODM 白名单；重打包脚本改动 `super` 布局时以 `lpmake` 与 `super_block_device` 标签为审计锚点；遵守「成组更新」清单（boot+system_dlkm；init_boot+system+system_ext+product）；GKI 与 GKI 模块必须同构建同更新。
- **应用开发者**：理解 ODM 定制走 RRO 而非改代码——官方成文的 RRO 属性是 `android:targetPackage` / `android:targetName`，且 `<application android:hasCode="false" />` 为强制（"Overlays can't overlay code, so they can't have DEX files."）；当资源值与预期不符时，把「是否存在启用中的 overlay 包」列入排查路径，但注意本方向在已解析公告中**无 CVE 条目**（观测缺口 ≠ 风险缺口）。
- **安全学习者**：先用 `git tag -v` 建立「来源」直觉，再用 §5/§6 建立「产物」直觉，最后用 §7/§8 建立「签名与清单」直觉；不要从第三方镜像取 PGP 公钥。

## 11. 常见误传纠正

1. 误传「`android.googlesource.com` 只是 Gerrit 的只读镜像，AOSP 另有权威库」。**正确口径**：查无该表述的官方原文；官方只说源码 "hosted by Google"，并区分 review 接口（`android-review.googlesource.com`）与 code navigation 接口（`android.googlesource.com`）；`gerrit.googlesource.com` 承载的是 git-repo 工具本身。
2. 误传「AOSP 镜像曾被投毒 / 有镜像投毒通告」。**正确口径**：查无官方出处；官方对来源可疑的成文处置是让使用者自行验签（§4）。
3. 误传「Gerrit 提交可绕过评审进入 AOSP」。**正确口径**：查无官方出处；成文表述是 "automatically merged if they pass code review"，即评审是入库门槛；`+2`/`verified` 门禁细节未核实，不写。
4. 误传「Android 15+ 之后 AOSP 只发布季度安全快照」。**正确口径**：季度化的是 2026 年起的**源码推送**（Q2/Q4），SPL 与公告仍按月。
5. 误传「tag 末尾 `_rN` 是补丁级别」。**正确口径**：`_rN` 是同一版本第 N 次 AOSP 推送。
6. 误传「预装应用要求见 'Configure system and app partition contents' 页」。**正确口径**：该页标题在本次抓取的任何导航中未出现；预装/权限侧的成文要求应引 CDD（如 `/docs/compatibility/17/android-17-cdd`）。`privapp-permissions` 的文件位置与 XML 格式**未核实**，不得引用。
7. 误传「`android:isDefault` 是 RRO 成文属性」。**正确口径**：成文属性是 `android:targetPackage`、`android:targetName`、`android:hasCode`；静态 RRO 只有「默认启用」有原句，`config_` 系列变量实名未核实。
8. 误传「`fastboot --slot`、`dmctl` 是文档化命令」。**正确口径**：`fastboot update` / `fastboot flashall` 有 `/docs/core/ota/tools` 原句为据，`--slot` 与 `dmctl` 均未核实。
9. 误传「xz/liblzma 后门影响 Android 源码链路」。**正确口径**：AOSP 文档域内查无官方出处——既无承认也无否认；已核实的是获取链路为 Git + Python 写的 repo。只能写「未检索到官方认定」，不能写「已排除」。
10. 误传「Stage2/Turla 类工具利用 Android 构建链」是 Android 供应链结论。**正确口径**：本次未抓取相关报告原文，未核实；官方威胁沟通面只有安全公告与 Advisories（导航 "Advisories … March 2016" 实名存在）。

## 12. 观测点：核实过的属性、命令与路径

只列一手核实到的名字；未核实的一律不写成可执行读法（见末节）。

- **包与分区归属**：`pm list packages` 的真实标志（核实自 AOSP 源码 `PackageManagerShellCommand.java` 的 usage 文本）：`-f`（关联文件）、`-s`（仅系统包）、`-3`、`-d`/`-e`、`-q`、`-i`、`-u`/`-U`、`--show-versioncode`、`--apex-only`、`--factory-only`、`--uid UID`、`--user USER_ID`；官方把 `-l` 标注为 "ignored (used for compatibility with older releases)"，不要再当作「显示权限」。`pm list packages -s -f` 可按 APK 路径判断落在 `/system/…`、`/vendor/…`、`/product/…`。
- **分区与布局**：动态分区白名单只有 System / Vendor / Product / System Ext / ODM，`system_dlkm` 属动态分区；`boot`、`dtbo`、`vbmeta` 必须保持物理分区；重打包与 `super` 镜像以工具 `lpmake`（1 MiB 对齐假定）与标签 `super_block_device` 为审计锚点。
- **来源与版本一致性**：`gpg --import keyfile.asc` + `git tag -v TAG_NAME`；`repo forall -pc 'git log --no-merges --oneline tagA..tagB'`；`/docs/setup/reference/build-numbers` 的 Build ID ↔ Tag ↔ Version ↔ SPL ↔ 构建码映射（如 `android-17.0.0_r1 ↔ CP2A.260605.016 ↔ SPL 2026-06-05`）。
- **组件清单**：`m sbom` → `out/dist/sbom*`；或 `ci.android.com` 输入分支 `aosp-android-latest-release` 取 CI 预生成 SBOM（SPDX 2.3 + NTIA Minimal Elements）。
- **签名锚点**：`vbmeta`（全部分区的 Verified Boot 信息）、GKI 模块的 kernel build-time key pair、OTA 包的 test key 与厂商自有私钥之分。
- **不给读法的**（本次未核实）：`ro.build.fingerprint`、`ro.product.bootimage.*`、`ro.boot.verifiedbootstate`、`adb shell dmctl`、`fastboot --slot`、`/proc/mounts` 里的 `dsnapshot`/`dsnap` 特征串。

## 与其他手册的分工

| 手册 | 关注面 |
|---|---|
| 本手册 | 代码与产物的**来源与完整性**：发布渠道与节奏、tag 验签、两条构建/刷机路径、动态分区与 `lpmake`、产物签名锚点、SBOM |
| 《TEE、密钥管理与设备认证手册》 | **运行时**信任根、密钥层级与设备认证证明 |
| 《底层漏洞类型图谱手册》 | 内核 / HAL / binder 等底层缺陷的**类型**分布与归属 |
| 《Android 系统更新链路与回滚防护手册》 | A/B 槽位、payload 校验、降级防护与 recovery；AVB 四色状态与 Rollback Protection 亦归该侧 |
| 《APK 签名与安装链路手册》 | 应用侧签名与安装；本手册只写系统产物侧签名 |

## 未核实与来源失效

正文未采用以下说法为事实；条目——未能核实的原因（含试过的来源）。

1. 「`android.googlesource.com` 只是只读镜像、AOSP 另有权威库」——两页官方文档均无对应原文，属流传说法。
2. 「AOSP 源码被投毒」的事件或说明页——AOSP 文档与导航查无；官方成文处置是让使用者自行验签。
3. Gerrit 入库门禁细节（`+2`、`verified` 标签）——无成文描述。
4. `imgdiff` 的位置与归属——素材内没有任何一手来源，故正文不作已核实工具名使用。
5. `privapp-permissions` 允许清单的文件路径与 XML 格式——已抓页面 grep `privapp-permissions`、`/system/etc/permissions` 均 0 命中；`/docs/core/permissions` 存在但正文未读。
6. 「Configure system and app partition contents」页与「预装应用」成文要求页——导航中无该标题，站内搜索返回 110 KB 空壳页；应改引 CDD。
7. 静态 RRO 的 `config_` 变量实名、`/docs/core/runtime/rro-troubleshoot` 正文——前者未核实，后者未读。
8. GKI 的 `CONFIG_` / symbols / KMI 机制名——仅核实到 `system_dlkm`、kernel build-time key pair、`depmod artifacts`。
9. `vts-latest-versions` 路径——已抓 HTML 内 grep 0 命中。
10. `fastboot --slot`（`/docs/devices/bootloader/flashall` 返回 404）、`adb shell dmctl`（`fs_mgr/README.md?format=TEXT` 无有效输出）、`/proc/mounts` 的 `dsnapshot`/`dsnap` 命名。
11. `getprop` 键 `ro.build.fingerprint`、`ro.product.bootimage.*`、`ro.boot.verifiedbootstate`——`platform/build` 的 `core/sysprop.mk?format=TEXT` 返回 200 但解码/匹配失败（`platform/build/make` 为 404）。
12. Pixel factory image 的具体文件名规则（如 `redfin-*-factory-*.zip`）——未跟到 `developers.google.com/android/images` 正文。
13. 给 OEM 的 sideload 发布渠道成文页——只有「提前至少 1 个月通知」与「公告后 48 小时内推 AOSP」两句时间性承诺。
14. CVE-2021-0703 的公告日期与章节名、CVE-2024-31317 的章节名——缓存的 `2021-09-01`/`2021-09-05` 两份文件内 grep 0 命中；行解析未取到章节名。
15. xz/liblzma 是否影响 Android 源码链路——AOSP 文档域内查无官方出处（既无承认也无否认），正文只写「未检索到官方认定」。
16. Stage2/Turla 类工具利用 Android 构建链——本次未抓取相关威胁报告原文，未核实。
17. 本方向公告条目缺失——Overlay/RRO、`privapp-permissions` 与预装应用机制、VTS/CTS/STS 框架自身、`super`/动态分区元数据：在已解析的 80+ 份公告中均 0 命中（观测缺口 ≠ 风险缺口）。
18. 站点地图部分失效——`sitemap.xml` 可用，但 `sitemap_0..4_of_5.xml` 全部 HTTP 500 / 0 字节（改用页内导航 href 抓取）；`/docs/concepts/architecture/partitions` 为 404，正确路径是 `/docs/core/architecture/partitions`；`/docs/setup/build/downloading` 实际渲染为 `/docs/setup/download`

## 参考

- https://source.android.com/docs/setup/build/downloading
- https://source.android.com/docs/setup/download/source-control-tools
- https://source.android.com/docs/setup/reference/build-numbers
- https://source.android.com/docs/whatsnew/site-updates
- https://source.android.com/docs/core/architecture/partitions
- https://source.android.com/docs/core/ota/dynamic_partitions/implement
- https://source.android.com/docs/core/ota/tools
- https://source.android.com/docs/core/runtime/rros
- https://source.android.com/docs/core/architecture/partitions/gki-partitions
- https://source.android.com/docs/compatibility/overview
- https://source.android.com/docs/compatibility/cdd
- https://source.android.com/docs/compatibility/vts
- https://source.android.com/docs/setup/create/create-sbom
- https://source.android.com/docs/security/features/verifiedboot
- https://source.android.com/docs/security/bulletin/2018-12-01
- https://source.android.com/docs/security/bulletin/2023-06-01
- https://source.android.com/docs/security/bulletin/2024-06-01
- https://source.android.com/docs/security/bulletin/2024-01-01
- https://gerrit.googlesource.com/git-repo/+/main/docs/manifest-format.md
- https://cs.android.com/android/platform/superproject
- https://developers.google.com/android/images/
- https://ci.android.com（分支 `aosp-android-latest-release`）
- AOSP 源码 `PackageManagerShellCommand.java`（`pm list packages` usage 文本，取自 android.googlesource.com）
- https://services.nvd.nist.gov/rest/json/cves/2.0?cveId=（CVE-2018-9567、CVE-2021-0703、CVE-2022-22706、CVE-2023-21127、CVE-2024-0671、CVE-2024-1065、CVE-2024-26926、CVE-2024-31317、CVE-2020-0041）
- https://www.cisa.gov/sites/default/files/feeds/known_exploited_vulnerabilities.json（catalogVersion 2026.10.26，count 1758）

## 联系方式

咨询 Android 系统详情请联系 https://t.me/one00190

*本手册仅供安全学习与防御研究使用；所有组件事实、接口名称与案例归属均取自公开一手来源并附链接，未能核实的表述已单列于「未核实与来源失效」一节而非写入正文。*
