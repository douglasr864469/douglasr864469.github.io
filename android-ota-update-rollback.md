---
layout: page
title: "Android 系统更新链路与回滚防护手册"
description: "更新引擎与 A/B、Virtual A/B 两条通道（`update_engine` 对应用隐藏、公开面为 `SystemUpdateManager`）、载荷 `CrAU` magic 与 `payload_properties.txt` 四字段、`otacerts.zip` 验签、回滚索引与降级判定的..."
date: 2026-10-27
permalink: /android-ota-update-rollback/
---

> 本手册沿一条主线展开：更新引擎 `update_engine` 与 A/B 槽位 → 载荷校验 `CrAU`、四字段与 `otacerts.zip` → 快照合并 Virtual A/B、`snapuserd` 与 COW → 回滚与降级防护 → 恢复模式 `recovery` → 打包交付 releasetools 与 target_files → 与工厂重置、FRP 的交叉。定位是防御与教学：只解释成文机制与公开且已修复的案例归属，不提供任何绕过步骤、攻击代码或载荷样本。凡在本研究范围内未取得一手出处的组件名、接口名与流行说法，一律不写入正文，只在 §11 以否定结论列出。

---

## 0. 使用方式与边界

- **读者**：OTA 平台维护者、厂商安全负责人、设备侧审计人员。
- **不做的事**：不给利用代码、不给绕过路径、不给真实载荷样本；§8 只收录公开、已修复且能在安全公告中定位的条目。
- **一手来源**：AOSP（Gitiles `refs/heads/main`，抓取日 2026-10-27）与 source.android.com 文档页。公告 URL 统一用 `https://source.android.com/docs/security/bulletin/YYYY-MM-01`（Pixel 为 `.../bulletin/pixel/YYYY-MM-01`），不使用带年份目录的形式。
- **KEV 口径**：§8 的「未收录」仅表示 CISA KEV 目录当时无该条目；**未收录 ≠ 无真实利用**。
- **范围**：只谈系统级 OTA 的载荷语义与回滚语义；运行时密钥、模块级更新、侧信道命令见 §12。

## 1. 更新引擎：`update_engine` 与 A/B 槽位

### 1.1 槽位、开关与不变式

- A/B（seamless）更新为系统保留两套槽位（后缀 `_a`/`_b`）：写非活动槽，切换失败由引导回落旧槽；Android 10 起 Virtual A/B 取代 Legacy A/B（`/docs/core/ota/ab`）。
- 文档的完成判据是三条件不变式：新槽须**可引导**、**校验通过**且为**活动槽**；`bootable` 表示槽可引导，`successful` 表示该槽上次启动成功。
- 引导传递：bootloader 注入 `androidboot.slot_suffix`，第一阶段经 `/firmware/android/slot_suffix` 读取；`update_engine.rc` 写成 `on property:ro.boot.slot_suffix=* → enable update_engine`——没有槽位概念的设备上该守护进程根本不启动。
- `bootloader_message.h` 记录的跨组件教训：`slot_suffix` 曾由 recovery/uncrypt 清除，属 boot-control 共享状态，改动要同时评估 bootloader、recovery、update_engine 三方。
- 收尾：`update_verifier.cpp` 在标记新槽成功后 `SetProperty("ota.warm_reset", "0")`。

### 1.2 守护进程与接口面

- 文档原文定位：`The update_engine source is located in system/update_engine`。
- 服务名实名：`ServiceName() { return "android.os.UpdateEngineService"; }`；`service_contexts` 另有 `update_engine_service` 与 `update_engine_stable_service`。历史 ChromeOS 侧 D-Bus 面为 `org.chromium.UpdateEngine` / `UpdateEngineInterface`（`UpdateEngine.conf`）。
- `IUpdateEngine`（包 `android.os`）成文方法：`applyPayload`、`applyPayloadFd`、`bind`、`unbind`、`suspend`、`resume`、`cancel`、`resetStatus`、`setShouldSwitchSlotOnReboot`、`resetShouldSwitchSlotOnReboot`、`verifyPayloadApplicable`、`cleanupSuccessfulUpdate`、`triggerPostinstall`。回调为 oneway：`onStatusUpdate(int status_code, float percentage)`、`onPayloadApplicationComplete(int error_code)`。
- 状态枚举 `UpdateStatusConstants` 为 0–9，其中 `ATTEMPTING_ROLLBACK = 8`；错误码存在 10/11/12/51/52/60/61 等实名条目，降级拒绝对应 `kPayloadTimestampError`。
- 框架侧：`android.os.UpdateEngine` 是 `@hide`，在 `core/api/current.txt` 中 0 命中，即第三方应用没有可依赖的公开更新 API；可依赖面是 `@SystemApi` 的 `SystemUpdateManager`（`KEY_*`/`STATUS_*`，`retrieveSystemUpdateInfo()` 权限为 `anyOf={READ_SYSTEM_UPDATE_INFO, RECOVERY}`）。
- 持久化：`kNonVolatileDirectory = "/data/misc/update_engine"`；日志在 `/data/misc/update_engine_log`；元数据默认 `/data/ota_package/metadata`。SELinux 面见 `private/update_engine.te` 与 `file_contexts`。
- 行为随版本变动：存在 aconfig 开关（如 `minor_changes_2025q4`），结论应绑定具体 build。

## 2. 载荷校验：`CrAU`、四字段与 `otacerts.zip`

- 载荷头前 4 字节是 `char[4]` 的 `"CrAU"`，实名常量 `const char kDeltaMagic[4] = {'C','r','A','U'};`（`payload_consumer/payload_constants.{h,cc}`）。
- `payload_properties.txt` 的四个字段（`common/constants.h` 中 `AB_OTA_PAYLOAD_PROPERTIES`）：`FILE_SIZE`、`FILE_HASH`、`METADATA_SIZE`、`METADATA_HASH`；另一处成文键名是 `AB_OTA_PAYLOAD`。
- 同文件其他属性键实名：`SPL_DOWNGRADE`、`SWITCH_SLOT_ON_REBOOT`、`RUN_POST_INSTALL`、`VABC_NONE`、`DISABLE_VABC`。
- 验签入口：`delta_performer.cc` 的 `CreatePayloadVerifier()` 优先走 `PayloadVerifier::CreateInstanceFromZipPath`（读 `otacerts.zip`），否则退回 `CreateInstance(public_key)`；`payload_verifier.cc` 对 Manifest Signature 与 Payload Signature 做**多签名循环**——一份载荷可同时携带多把 OTA 公钥的签名。
- 拒绝语义是明文：README 写「If the signature cannot be verified, the update is rejected.」
- 信任根位置：`otacerts.zip` 经 `Android.bp` 依赖进入镜像，`update_payload_key/README` 称其 "baked into the system image"；签名侧密钥由 `sign_builds` 的 `--package_key (-k)` 提供（`/docs/core/ota/sign_builds`）。
- 防御要点：公钥随镜像公开分发不构成写权限风险，信任边界是**私钥只存在于发布基础设施**，因此密钥治理问题应回到打包与签名环节排查。
- 流式：Android 8.0 起支持，约 100 KiB 元数据先行、正文边下边校验，故头部与哈希链的解析健壮性是独立风险面（§8 的 CVE-2018-9556 即在此面）。

## 3. 快照合并：Virtual A/B、`snapuserd` 与 COW

- Virtual A/B 在不为每个分区保留整套第二副本的前提下保持 seamless 语义：旧数据留在原分区，新增写进入 **snapshot/COW** 文件（`/docs/core/ota/virtual_ab`、`/virtual_ab/implement`）。
- COW 记录的写操作是四类成文操作：Copy、Replace、Zero、XOR。
- 合并由用户态守护进程 `snapuserd` 经 **dm-user** 把快照呈现为块设备；SELinux 有独立的 `private/snapuserd.te`；第一阶段 init 以 fd 传递套接字并在正确上下文重启该进程。
- 状态机：`MergeStatus` 是 BootControl HAL 的枚举；官方文档明确描述跨重启的合并续做（merge resume）。合并收敛的成文表述是"把 dm-verity 折叠到 dm-linear（`system_base`）之上并移除 dm-user"。
- misc 区独立消息结构：`misc_virtual_ab_message`，`MISC_VIRTUAL_AB_MESSAGE_VERSION 2`，`MISC_VIRTUAL_AB_MAGIC_HEADER 0x56740AB0`；与 BCB 同区但各自版本化，是排查"更新卡在合并中"的第一手凭据。
- 属性面：`ro.virtual_ab.*` 一组由 `aosp/dynamic_partition_control_android.cc` 读取。
- 主机侧：fastboot 的 `snapshot-update merge` / `snapshot-update cancel` 与 `getvar snapshot-update-status`；合并保护清单中明确"设备锁定时 `snapshot-update cancel` 无效"（invalid when the device is locked）。
- 交付要求：GMS 自 Android 11 起要求 Virtual A/B。
- 合并只是数据可见性收敛，不改变槽位活动性与 AVB 下限。

## 4. 回滚与降级防护

### 4.1 三层拒绝结构

| 层 | 判定凭据 | 拒绝发生处 | 能否被 OTA 自身放松 |
| --- | --- | --- | --- |
| AVB Rollback Protection | `rollback_index[n]` 对比 `stored_rollback_index[n]` | bootloader 移交 HLOS 之前 | 否，存储只增且篡改可证 |
| 构建时间戳 | `ro.build.date.utc`、`ro.<partition>.build.date.utc`、`max_timestamp` | `update_engine` 报 `kPayloadTimestampError` | 仅当 `ro.ota.allow_downgrade` 与 `ro.debuggable` 同时成立 |
| SPL / GKI | 载荷属性 `SPL_DOWNGRADE`；GKI `prevent_downgrade_version` | bootloader 拒绝；内核符号版本下限 | 成文后果是需要数据擦除 |

### 4.2 AVB 回滚保护

- 定义见 `external/avb/README.md` 与 verified-boot 文档页（§13）：每个镜像携带 `rollback_index[n]`，设备保存 `stored_rollback_index[n]`，前者不得小于后者；检查发生在 "prior to transferring control to the HLOS"；A/B 下每槽独立保存，文档示例为 `[42,101]` 与 `[43,103]`。
- 官方 FAQ 直接回答「Do A/B OTAs break AVB2.0's rollback protection?」为 **No**，并说明设备成功启动新镜像后回滚保护随即生效。

### 4.3 构建时间戳路径

- `aosp/hardware_android.cc`：`kPropBuildDateUTC`、`GetPartitionBuildDate()`、`GetBuildTimestamp()`。
- `common/utils.cc`：`IsTimestampNewer()`；`payload_consumer/delta_performer.cc` 汇总 `max_timestamp` 与 `downgrade_detected`，命中即 `ErrorCode::kPayloadTimestampError`。
- `AllowDowngrade()` 的成文条件是**两个属性同时成立**：`ro.debuggable` 为 1 且 `ro.ota.allow_downgrade` 为真——即工程/调试构建才可能放松，用户版默认拒绝。

### 4.4 SPL 与内核侧下限

- `CheckSPLDowngrade()` 的成文注释指出：bootloader 会无条件拒绝 SPL 降级（"bootloader will reject SPL downgrade no matter what"），且一旦触发需要擦除数据（"Data wipe will be required"）。
- GKI 侧 `prevent_downgrade_version` 提供内核组件的降级下限。

### 4.5 与刷机通道的对照

- `fastboot --slot`、`--set-active` 与 `snapshot-update` 只改变槽位活动性与合并状态，不写 `stored_rollback_index`。
- 因此"能写回旧镜像"与"能让旧镜像通过 AVB 启动"是两件事：前者是工具能力，后者是设备状态，不要把前者当成后者的失效。

## 5. 恢复模式：`recovery` 的成文面与硬边界

### 5.1 可见菜单

- `recovery_ui/device.cpp` 的字符串实名包含 `Reboot system now`、`Apply update from ADB`、`Wipe data/factory reset`、`Power off` 等。
- `RemoveMenuItemForAction` 与 `CHECK(!g_menu_actions.empty())` 说明菜单项可被移除；不能把 AOSP 菜单当成设备事实。

### 5.2 指令通道与日志

- 跨重启指令走 misc/BCB；文件面由 `otautil/paths.cpp` 定义：`/cache/recovery/last_log`、`/cache/recovery/log`、`/cache/recovery/last_kmsg`、`/cache/recovery/last_command`；运行期日志 `/tmp/recovery.log`。
- `OPTIONS[]` 是成文选项表；`--retry_count` 属 BCB/命令行选项而**不是**某个文件名（见 §9）。
- 状态回读经 `recovery-persist`/`recovery-refresh`，pstore 见 `recovery_refresh.te`；`README.md` 另成文 `/data/misc/recovery/inject.txt`。
- 框架入口：`RecoverySystem.java` 与 `RecoverySystemService.java`（`publishBinderService(Context.RECOVERY_SERVICE, ...)`、`ServiceManager.checkService("vold")`）。

### 5.3 安装脚本的两种形态

- A/B：`install/install.cpp` 的 `SetUpAbUpdateCommands` 组装 `/system/bin/update_engine_sideload --payload=file://... --offset=... --headers=... --status_fd=...`，参数默认值见 `aosp/sideload_main.cc`。
- 非 A/B：`SetUpNonAbUpdateCommands` 配合 `UPDATE_BINARY_NAME`（即 `update-binary`）。
- 块级（block-based）OTA 是现行形态；文件级 OTA 官方已弃用——`ota_from_target_files.py` 原文：「We have deprecated the support for file-based OTA since O.」

### 5.4 SELinux 硬边界

- `private/recovery.te` 用 `recovery_only` 宏把规则限定在 recovery 构建，用 `execute_no_trans` 允许不切换域地执行指定程序，并有 neverallow 块约束 recovery 环境下的执行面。
- 可引的 allow 面示例：`dm_user_device:dir`、`sysfs_dm`、`super_block_device_type:blk_file` 的 ioctl、`snapuserd_socket`/`snapuserd` 连接、`metadata_file:dir { getattr mounton }`。

### 5.5 旧机制的成文残留

- 产品名实名带下划线：`recovery_from_boot.p`，补丁由 `imgdiff` 生成；其失效条件在历史分支构建规则中成文（五类）。
- `install-recovery.sh` 属 vendor 侧机制（sepolicy 中 `vendor_install_recovery_exec`），不是 AOSP recovery 的自用路径。
- 分区形态的构建变量实名：`BOARD_USES_RECOVERY_AS_BOOT`、`BOARD_MOVE_RECOVERY_RESOURCES_TO_VENDOR_BOOT`、`BOARD_INCLUDE_RECOVERY_RAMDISK_IN_VENDOR_BOOT`，产物含 `ramdisk-recovery.img`。

## 6. 打包与交付：releasetools 与 target_files

- 真实仓库路径是 `platform/build/tools/releasetools`；`platform/tool/releasetools` 不存在（Gitiles 返回 "Repository is not found"）。关键脚本：`ota_from_target_files.py`、`common.py`、`edify_generator.py`、`non_ab_ota.py`、`make_recovery_patch.py`。文档：`/docs/core/ota/tools`。
- `target_files` 里的 `META/misc_info.txt` 是打包契约：`recovery_api_version` 为必填项，`blockimgdiff_versions=3,4` 决定镜像 diff 的版本协商。
- 动态分区时代交付物是 `super.img`（`/docs/core/ota/dynamic_partitions/implement`），与 §3 的 snapshot 语义在此衔接：布局变化不改变 §2 的验签要求。
- 交付通道与信任的关系：包是否合法只由 §2 的签名链决定；`libcurl_http_fetcher.cc` 与 `certificate_checker.cc` 处理传输层与证书校验，不替代载荷验签。

## 7. 与工厂重置 / FRP 的交叉

- 已核实的唯一 FRP 实名面：`property_contexts` 中 `ro.frp.pst` 标记为 `u:object_r:exported_default_prop:s0 exact string`。FRP 的官方文档页与 `PersistentDataBlockService` 现路径本次未取得出处（见 §11），故本节只写边界不写机制。
- 与更新链的三个真实交点：
  - `Wipe data/factory reset` 是 recovery 菜单项（§5.1），重置与更新失败处置共用同一通道；
  - CVE-2026-0134 落在 `recovery_ui.cpp` 的 `PostWipeData`，说明"擦除之后"的处理同属更新链责任面；
  - §4.4 的 SPL 降级成文后果是"需要数据擦除"，把回滚防护与重置状态在语义上绑定。
- `ota.warm_reset`（§1.1）是更新成功路径上唯一核实过的与 reset 相关的属性名。
- 「解锁（OEM unlock）必须执行 factory reset」这一流行说法本次未成文核实（§11）；可引的实名只有 `ro.oem_unlock_supported`、`ro.oem.key1`。
- FRP／引导加载器方向的公开 CVE 公告列未逐条核对，故不入 §8（§11）；相关刷机命令面属《ADB、备份与数据迁移通道手册》。

## 8. 公开案例

只收录公开、已修复、可在公告页定位的条目。「未收录」指 CISA KEV 目录（`catalogVersion: 2026.10.23`，共 1756 条 `"cveID"`）中没有该 CVE；**CISA KEV 未收录 ≠ 无真实利用**，该目录只反映已确认在野利用的判断。

| 链路组件 | CVE | 缺陷位置（一手） | 公告归属 | KEV |
| --- | --- | --- | --- | --- |
| update_verifier | CVE-2017-0475 | `bootable/recovery/update_verifier`（标记成功前校验） | [2017-03-01](https://source.android.com/docs/security/bulletin/2017-03-01) · A-31914369 | 未收录 |
| update_engine | CVE-2017-13265 | update_engine（Pixel 线） | [pixel/2018-03-01](https://source.android.com/docs/security/bulletin/pixel/2018-03-01) | 未收录 |
| 载荷头解析 | CVE-2018-9556 | `ParsePayloadHeader`，`payload_metadata.cc` | [2018-12-01](https://source.android.com/docs/security/bulletin/2018-12-01) | 未收录 |
| AVB 校验 | CVE-2020-0151 | `avb_vbmeta_image_verify` | [pixel/2020-06-01](https://source.android.com/docs/security/bulletin/pixel/2020-06-01) | 未收录 |
| AVB 校验 | CVE-2020-0152 | `avb_vbmeta_image_verify` | [pixel/2020-06-01](https://source.android.com/docs/security/bulletin/pixel/2020-06-01) | 未收录 |
| 侧载安装 | CVE-2020-0204 | `InstallPackage`，`package.cpp`（TOCTOU） | [pixel/2020-06-01](https://source.android.com/docs/security/bulletin/pixel/2020-06-01) | 未收录 |
| 写入路径 | CVE-2021-0377 | `DeltaPerformer::Write` | [pixel/2021-03-01](https://source.android.com/docs/security/bulletin/pixel/2021-03-01) | 未收录 |
| 事件处理 | CVE-2021-39632 | `inotify_cb`，`events.cpp` | [2022-01-01](https://source.android.com/docs/security/bulletin/2022-01-01) | 未收录 |
| dm-verity | CVE-2022-20571 | `drivers/md/dm-android-verity.c` | [pixel/2022-12-01](https://source.android.com/docs/security/bulletin/pixel/2022-12-01) | 未收录 |
| dm-verity | CVE-2022-20572 | `drivers/md/dm-verity-target.c` | [pixel/2022-12-01](https://source.android.com/docs/security/bulletin/pixel/2022-12-01) | 未收录 |
| 擦除后处理 | CVE-2026-0134 | `PostWipeData`，`recovery_ui.cpp` | [pixel/2026-06-01](https://source.android.com/docs/security/bulletin/pixel/2026-06-01) | 未收录 |

- 归属提醒：CVE-2017-0475 属 `update_verifier`，与同批公告中的 CVE-2017-0474（Mediaserver）不是同一问题，二者常被混用。
- `snapuserd` 组件在 CVE 检索中命中为 0，只说明公开描述文本不含该词，不表示无缺陷历史（§11）。

## 9. 常见误传纠正

1. 「NVD 描述里能搜到 update_engine」：`keywordSearch=update_engine`、`=snapuserd`、`="in update_verifier"` 均返回 `totalResults: 0`（对照 `system_server` 有 15 条，说明关键词机制可用）。文件级归属须引公告表格与 NVD 原文。
2. 把 CVE-2017-0475 与 CVE-2017-0474 混为一谈：前者是 recovery `update_verifier`，后者属 Mediaserver。
3. 「A/B 更新破坏了 AVB 回滚保护」：官方 FAQ 的成文答案是 No。
4. 「公告 URL 用带年份目录的形式」：该形式对月度公告实测 404；月度公告与 Pixel 公告的可用形式见 §13。
5. 「`PAYLOAD_MAGIC` 与 `HEADER_SIZE` 是实名」：真实名字是 `kDeltaMagic`，以及 `FILE_SIZE`/`FILE_HASH`/`METADATA_SIZE`/`METADATA_HASH`。
6. 「`android.os.UpdateEngine` 是公开 SDK」：它是 `@hide`，`current.txt` 0 命中。
7. 「监听 `UPDATE_ENDED`/`UPDATE_STATUS` 广播即可」：AOSP 无此广播，真实机制是 `IUpdateEngineCallback` 的两个 oneway 回调。
8. 「`update_engine_client --markboot_successful` 可用」：本次抓取的参数表无此项；`markBootSuccessful` 只在 `update_verifier` 的 C++ 调用中确认存在。
9. 「`platform/tool/releasetools`」：仓库不存在，实为 `platform/build/tools/releasetools`。
10. 「`recovery-from-boot.p`」：实名带下划线 `recovery_from_boot.p`；`install-recovery.sh` 属 vendor 侧而非 AOSP recovery 自用。
11. 「`UPDATE_RECOVERY`、`free_cache`、`/metadata/recovery`、`--sideload_package` 是 AOSP 现行名」：在 `refs/heads/main` 查无出处。
12. 「`Ro.OemLock`、`ro.boot.virtual_ab_enable`、`ota_downgrade`、`applyPayloadFromBsdiff`、`IUpdateEngine.schedule` 是接口」：均非 AOSP 成文名。
13. 「传输层 TLS 校验等于载荷可信」：`certificate_checker.cc` 属 HTTP 层，不替代 §2 的签名验证。
14. 「VABC 是官方定义的缩写」：文档以 "Virtual A/B compression" 描述该特性，但未显式展开这三个字母；实名只有 `VABC_NONE`/`DISABLE_VABC`。
15. 「KEV 未收录就说明没被利用」：见 §8 口径说明。

## 10. 可观测面与自查

以下均为本次核实到的实名。

- **槽位与活动性**：`getprop ro.boot.slot_suffix`；`fastboot getvar slot-suffix`、`fastboot --slot`、`fastboot --set-active`。
- **Virtual A/B**：`getprop` 过滤 `ro.virtual_ab.`（该前缀下属性实名存在）；`fastboot snapshot-update merge|cancel`、`fastboot getvar snapshot-update-status`。
- **降级判定输入**：`ro.build.date.utc`、`ro.<partition>.build.date.utc`、`ro.ota.allow_downgrade`、`ro.debuggable`；失败侧看 `kPayloadTimestampError`；载荷侧看 `SPL_DOWNGRADE`。
- **文件与日志**：`/data/misc/update_engine`、`/data/misc/update_engine_log`、`/data/ota_package`、`/cache/recovery/*`（`last_log`、`log`、`last_kmsg`、`last_command`）、`/tmp/recovery.log`、`/data/misc/recovery/inject.txt`、`/dev/socket/recovery`。
- **服务面**：`android.os.UpdateEngineService`、`update_engine_stable_service`（`service_contexts`）；`dumpsys` 服务名字符串未逐字核实，不作结论。
- **合法调用面清单**：`update_engine_client` 参数表可当巡检白名单——`--update --payload --offset --size --headers --verify --allocate --metadata --switch_slot --trigger_postinstall --suspend --resume --cancel --reset_status --follow --merge`；`--metadata` 默认 `/data/ota_package/metadata`；`--follow` 退出码 0/1 可用于脚本判定；`--switch_slot` 只接受 `true`/`false`。
- **SELinux 巡检**：比对设备侧策略与 AOSP `update_engine.te`、`snapuserd.te`、`recovery.te`（含 `recovery_only` 与 neverallow 块）是否被厂商放大。
- **打包巡检**：`target_files` 的 `META/misc_info.txt` 是否声明 `recovery_api_version` 与 `blockimgdiff_versions=3,4`；交付物是否与 `super.img` 布局一致。
- **合并卡住排查**：依次看 `misc_virtual_ab_message`（版本 2、magic `0x56740AB0`）、HAL 的 `MergeStatus`、`snapshot-update-status`；锁定状态下 `cancel` 无效属预期。
- **重置面**：`ota.warm_reset`、`ro.frp.pst` 的取值与 SELinux 标签。

## 11. 未核实与来源失效

以下条目未取得一手证据，正文只作否定结论；破折号后是查证范围。

1. `PAYLOAD_MAGIC` 常量名——被核验文件中无此串，实名是 `kDeltaMagic`（`payload_constants.{h,cc}`）。
2. `HEADER_SIZE` 作为 payload_properties 字段——`constants.h`、`payload_properties.cc`、`brillo_update_payload` 均无。
3. `update_engine_public_key`、`TARGET_PUBLIC_KEY`——`Android.bp`、`payload_verifier.{h,cc}`、`payload_signer.cc`、`update_payload_key/README` 均无；已核实的等价机制是 `otacerts.zip` + `CreateInstanceFromZipPath`。
4. `Ro.OemLock`——无此名；相近实名仅 `ro.oem_unlock_supported`、`ro.oem.key1`（property_contexts）。
5. `ro.boot.virtual_ab_enable`——property_contexts、file_contexts、update_engine 源码均无。
6. `ota_downgrade`、`ota_reject_downgrade`——不存在；真实机制是时间戳比较与 `kPayloadTimestampError`。
7. `UPDATE_ENDED`、`UPDATE_STATUS` 广播——AOSP 代码与 sepolicy 无此字符串。
8. `applyPayloadFromBsdiff`——`IUpdateEngine.aidl`、`UpdateEngine.conf`、`common/constants.h` 三处均无（ChromeOS 历史接口，非 AOSP 出处）。
9. `IUpdateEngine.schedule`——aidl 方法表无此方法；D-Bus 侧相近成员也未出现在 AOSP `UpdateEngine.conf`。
10. `update_engine_client --markboot_successful`——本次抓取的参数表无此项。
11. Pixel 及厂商对 re-downgrade 的成文拒绝规则——未找到公开页面；只有通用机制可引（`stored_rollback_index`、`ro.ota.allow_downgrade`、`SPL_DOWNGRADE`、锁定时 `snapshot-update cancel` 无效）。
12. `VABC` 缩写的官方展开——文档以 "Virtual A/B compression" 描述特性，未显式定义该缩写。
13. `dumpsys` 服务名字符串——`Context.RECOVERY_SERVICE` 的常量值与 update_engine 的 binder 服务名均未逐字核对（代码行已定位）。
14. `/proc/mounts` 中 super/snapshot 的具体字符串样本——未取一手样本；仅 sepolicy 侧 type 实名可引。
15. `omaha` 属 ChromeOS 的一手出处——AOSP 查无此名，ChromeOS 文档未抓取。
16. FRP 的官方文档实名与机制、`PersistentDataBlockService` 现路径——`/docs/privacy/frp` 与 `/docs/security/features/frp` 均 404，frameworks 路径不存在。
17. `SETFRPOWNER`、`GETFRPOWNER`、`GETFRPOWNERCERT`——AOSP 查无；「仅 OEM 示例」这一断言本身亦无来源。
18. 「OEM unlock 必须执行 factory reset」——未成文核实（boot-flow 页存在但未抽取该句）。
19. `/metadata/recovery` 下的 staging 文件、`--sideload_package`、`free_cache`、`UPDATE_RECOVERY`——`main` 与 `pie-release`/`marshmallow-release` 试查均无出处。
20. `imgdiff` 宿主工具的仓库位置——三处候选路径 404（`system/imgdiff`、`system/extras/imgdiff`、`system/core/tools`）。
21. `platform/tool/releasetools`——仓库名不存在（"Repository is not found"），实名 `platform/build/tools/releasetools`。
22. Android 11+「recovery 位于 boot/vendor_boot 的 `recovery_ramdisk`」的官方文档原文——未核实；boot-image-header 正文只有 `recovery_dtbo`、`recovery_acpio`，vendor-boot-partition 页 404。构建变量实名可引（§5.5）。
23. FRP／引导加载器方向 12 条候选 CVE 的公告页表格列（组件归属、Pixel 列、日期）——未逐条核对，故不入 §8。
24. 失效 URL（实测 404 或不存在）：`/docs/core/ota/ab_implement`、`/ab_tools`（正确 `/docs/core/ota/ab/ab_implement`）、`/ab/ab_updatable_devices`、`/ab_part-a`、`/docs/core/ota/static`、`/docs/core/ota/payload`、`/downdatable-partitions`、`/docs/security/features/verifiedboot/downgrade-protection`（正确 `verified-boot#rollback-protection`）、`/updatable-bootable-devices`、update_engine 仓 `docs/`、`binder_bindings/aidl/...`（正确 `binder_bindings/android/os/*.aidl`）、`payload_generator/bootable_payload_writer.cc`、`common/flags.h`（正确 `flags/update_engine_flags.aconfig`），以及带年份目录的公告 URL 形式。
25. AOSP 月度公告正文只有表格（CVE／References／Type／Severity／Subcomponent），无散文；文件级归属必须引 NVD 原文。

## 12. 与其他手册的分工

- **《TEE、密钥管理与设备认证手册》**：运行时信任根与密钥使用。本手册只把 AVB 的 `stored_rollback_index` 当作**镜像回滚下限的存储凭据**引用，不涉密钥使用面。
- **《Mainline 与 APEX 模块更新手册》**：组件级更新通道与模块语义。本手册处理**系统级** OTA 载荷与槽位切换；GKI `prevent_downgrade_version` 是交集，此处仅作内核侧降级下限。
- **《ADB、备份与数据迁移通道手册》**：侧信道与刷机命令面（adb、备份恢复、迁移对设备状态的影响）。本手册只写 `update_engine_sideload` 与 `update_engine_client` 作为链路内部件的成文接口。
- **本手册**：系统级 OTA 的载荷构造与校验、快照合并语义、回滚与降级的三层拒绝，以及 recovery 与 releasetools 的成文边界。

## 13. 参考

文档（均加前缀 `https://source.android.com`）：
`/docs/core/ota/ab` · `/docs/core/ota/ab/ab_implement` · `/docs/core/ota/ab/ab_faqs` · `/docs/core/ota/virtual_ab` · `/docs/core/ota/virtual_ab/implement` · `/docs/core/ota/tools` · `/docs/core/ota/sign_builds` · `/docs/core/ota/dynamic_partitions/implement` · `/docs/security/features/verifiedboot` · `/docs/security/features/verifiedboot/verified-boot` · `/docs/security/features/verifiedboot/avb` · `/docs/security/bulletin`

公告页：
`https://source.android.com/docs/security/bulletin/2017-03-01` · `.../2018-12-01` · `.../2022-01-01` · `.../pixel/2018-03-01` · `.../pixel/2020-06-01` · `.../pixel/2021-03-01` · `.../pixel/2022-12-01` · `.../pixel/2026-06-01`

update_engine 源码（均加前缀 `https://android.googlesource.com/platform/system/update_engine/+/refs/heads/main/`）：
`README.md` · `update_engine.rc` · `Android.bp` · `UpdateEngine.conf` · `binder_bindings/android/os/IUpdateEngine.aidl` · `binder_bindings/android/os/IUpdateEngineCallback.aidl` · `aosp/binder_service_android.h` · `common/constants.h` · `common/utils.cc` · `payload_consumer/payload_constants.{h,cc}` · `payload_consumer/payload_verifier.{h,cc}` · `payload_consumer/delta_performer.cc` · `payload_generator/payload_properties.cc` · `payload_generator/payload_signer.cc` · `aosp/hardware_android.cc` · `aosp/dynamic_partition_control_android.cc` · `aosp/update_engine_client_android.cc` · `aosp/sideload_main.cc` · `aosp/platform_constants_android.cc` · `flags/update_engine_flags.aconfig` · `update_payload_key/README` · `libcurl_http_fetcher.cc` · `certificate_checker.cc`

其他源码仓（前缀 `https://android.googlesource.com/platform/`）：
`bootable/recovery/+/refs/heads/main/`：`recovery.cpp` · `recovery_ui/device.cpp` · `recovery_ui/recovery_ui.cpp` · `install/install.cpp` · `install/wipe_data.cpp` · `otautil/paths.cpp` · `update_verifier/update_verifier.cpp` · `bootloader_message/include/bootloader_message/bootloader_message.h` · `README.md`
`build/+/refs/heads/main/`：`core/Makefile` · `tools/releasetools/`（`ota_from_target_files.py`、`common.py`、`edify_generator.py`、`non_ab_ota.py`、`make_recovery_patch.py`）
`frameworks/base/+/refs/heads/main/`：`core/java/android/os/UpdateEngine.java` · `SystemUpdateManager.java` · `RecoverySystem.java` · `core/api/current.txt` · `services/core/java/com/android/server/recoverysystem/RecoverySystemService.java`
`system/sepolicy/+/refs/heads/main/`：`private/update_engine.te` · `private/snapuserd.te` · `private/recovery.te` · `public/recovery.te` · `private/recovery_persist.te` · `private/recovery_refresh.te` · `private/update_verifier.te` · `private/file_contexts` · `private/service_contexts` · `private/property_contexts` · `vendor/file_contexts` · `public/file.te`
`external/avb/+/refs/heads/main/README.md` · `system/core/+/refs/heads/main/fastboot/fastboot.cpp`

漏洞数据：
`https://nvd.nist.gov/vuln/detail/CVE-2017-0475`（同形式可查 CVE-2017-13265、CVE-2018-9556、CVE-2020-0151、CVE-2020-0152、CVE-2020-0204、CVE-2021-0377、CVE-2021-39632、CVE-2022-20571、CVE-2022-20572、CVE-2026-0134） · `https://services.nvd.nist.gov/rest/json/cves/2.0?cveId=CVE-2018-9556` · `https://www.cisa.gov/sites/default/files/feeds/known_exploited_vulnerabilities.json`

## 14. 联系方式

咨询 Android 系统详情请联系 https://t.me/one00190

*本手册仅供安全学习与防御研究使用；所有组件事实、接口名称与案例归属均取自公开一手来源并附链接，未能核实的表述已单列于 §11 而非写入正文。*
