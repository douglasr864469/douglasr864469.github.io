---
layout: page
title: "Android 崩溃诊断与日志信息暴露面手册"
description: "把「崩溃会留下什么」拆成四条互不替代的留存通道：`debuggerd`/`crash_dump`/`tombstoned` 的原生崩溃路径、`dropbox` 的系统级事件 blob、`bugreport` 这个最大的聚合外泄面、`logcat`/`logd`/`READ_LOGS` 的读取边界；另含内..."
date: 2026-10-29
permalink: /android-diagnostics-log-exposure/
---

> 本手册沿「原生崩溃诊断 → dropbox 事件 → bugreport 采集 → 日志读取边界 → 内核侧留存 → 隐私成文位置」这条链路，把 Android 上各类诊断产物的真实组件名、真实路径、真实权限与策略出处整理成一张可复核的防御地图。它面向蓝队、设备合规人员与系统研究者，回答「系统在我设备上留下了什么、谁能读到、官方在哪里成文承认过这些内容可能含私密信息」。全文不含任何敏感数据提取操作；涉及内容时只写到数据类别层面，不给载荷、不给利用路径。所有事实均附一手链接，未能核实的表述隔离在末尾 §11。

---

## 0. 使用方式与边界

### 0.1 证据口径

正文只用三类可复核材料：AOSP `refs/heads/main` 分支的源码行（`system/core/debuggerd/`、`system/core/rootdir/init.rc`、`frameworks/base/services/core/java/com/android/server/DropBoxManagerService.java`、`system/sepolicy/+/private/{file_contexts,genfs_contexts,dumpstate.te,logd.te}`）；`source.android.com` 现行文档页；内核与外部一手（`kernel.org` admin-guide、`https://source.android.com/docs/security/bulletin/YYYY-MM-01`、NVD REST、CISA KEV JSON）。文档与代码冲突时两栏并列标注，不择一伪造。

### 0.2 边界

不覆盖任何绕过 SELinux、提权读取他人诊断文件、或从 tombstone/pstore 镜像还原密钥与口令的操作。本手册中「暴露面」定义为信息密度与访问控制之间的差值：一份 tombstone 同时含全线程回溯、寄存器、原始栈转储与模块映射；一份 bugreport 是系统日志、事件日志、内核日志的聚合。两者都是防御方应盘点的对象，不是攻击素材。AOSP `main` 的事实不等于某支厂商 ROM 的事实，涉及 `user` / `userdebug` 差异时以设备实测与本机 sepolicy 版本为准。

### 0.3 阅读顺序

§1 建立「原生崩溃如何落盘」主干；§2–§3 是事件存储与 bugreport 聚合；§4–§5 是读取边界与内核留存；§6 回答官方在哪里成文承认隐私风险；§7 是纠偏清单（从二手博客抄过结论的先对照它）；§8 是公开案例与方法论；§9 是可观测面自查。

---

## 1. 原生崩溃诊断：`debuggerd` / `crash_dump` / `tombstoned`

### 1.1 三件套的分工

社区文本常把 `debuggerd` 当成单体守护进程叙述，现行 AOSP 拆成三个协作单元（`system/core/debuggerd/Android.bp` 的真实模块名可对照）：

- **`debuggerd`** —— 入口可执行文件与客户端库（`debuggerd`、`libdebuggerd`、`libdebuggerd_handler`、`libdebuggerd_client`）。承担 CLI 语义与进程内 handler 注册。
- **`crash_dump`** —— 真正执行 ptrace 的二进制（另有 `crash_dump.policy`）。崩溃时附着故障进程收集现场。
- **`tombstoned`** —— socket 服务与落盘队列，持有 `/data/tombstones` 写入队列并按序号轮转命名。
- 周边：`pbtombstone`（tombstone protobuf 转文本，`/system/bin/pbtombstone u:object_r:pbtombstone_exec:s0`）、`llkd`（低层内核死亡守护）。

`debuggerd` 帮助文案本身就区分两种产物：`"-b, --backtrace    just a backtrace rather than a full tombstone"`。「取一条回溯」与「产一份完整 tombstone」在官方语义里信息量与落盘行为都不同。

### 1.2 tombstone 的内容类别

官方信号范围原文：`The signals reported by debuggerd are SIGABRT, SIGBUS, SIGFPE, SIGILL, SIGSEGV, and SIGTRAP.`

相对 crash dump 的增量，官方成文清单：

> "The tombstone contains the same information as the crash dump, plus a few extras. For example, it includes backtraces for all threads (not just the crashing thread), the floating point registers, raw stack dumps, and memory dumps around the addresses in registers."

据此可列出暴露面清点用的类别表（只到类别，不含载荷）：

| 类别 | 官方成文形态 | 暴露面含义 |
|---|---|---|
| 终止原因 | signal 名、`Abort message`、`Cause:`（文档示例含 seccomp 拒绝） | 透露实现细节与安全策略命中点 |
| 寄存器 | `r0..r11` / `x0..x29`、`sp`、`lr`、`pc`、`cpsr` | `pc`/`lr` 可推断模块基址 |
| 全线程回溯 | `backtrace:` 帧 | 内部函数与库版本 = 组件指纹 |
| 进程标识 | `pid: N, tid: N, name: X`、`>>> <process> <<<` | 进程名常带业务模块信息 |
| 故障地址 | `fault addr` | 与映射表联用可推断内存布局 |
| 栈/内存镜像 | raw stack dumps、memory dumps around register addresses | 最高敏感层：栈上残留数据落在这里 |

文档示例 `Tombstone written to: /data/tombstones/tombstone_06` 说明日志里那一行只是指针，载荷在 `/data/tombstones`。测试工具实名：官方页写着平台源码含 `crasher` 与 `crasher64`（在 `system/core/debuggerd/` 下 `mm` 可得）。

### 1.3 保留数量：文档与代码冲突（必查点）

- 文档口径：`docs/core/tests/debug/native-crash` 仍写 `debuggerd will keep up to 10 tombstones, cycling through the numbers 00 to 09 and overwriting existing tombstones as necessary.`
- 代码口径：`tombstoned/tombstoned.cpp` 为
  `static CrashQueue queue("/data/tombstones", "tombstone_" /* file_name_prefix */, GetIntProperty("tombstoned.max_tombstone_count", 32), ...)`；同一区域另有 `GetIntProperty("tombstoned.max_anr_count", 64)`。

盘点时应按可被属性覆盖的 **32（tombstone）/ 64（ANR）** 估计，而不是 10；同时必须写明「文档页尚未同步」。两个属性键 `tombstoned.max_tombstone_count`、`tombstoned.max_anr_count` 来自源码字面量，可核实；流传的 `debug.tombstone*` 属性名查无出处，见 §11。

### 1.4 目录、属主与加密实名

`system/core/rootdir/init.rc` 一手行：

```
mkdir /data/tombstones 0775 system system encryption=Require
```

其上注释 `# Start tombstoned early to be able to store tombstones.`。同批相关行：`mkdir /data/anr 0775 system system encryption=Require`、`mkdir /data/system/dropbox 0700 system system`（0700，非属主在 Unix 层即不可读）、`mkdir /data/vendor/tombstones 0771 root root`、`mkdir /data/vendor/tombstones/wifi 0771 wifi wifi`。

`private/file_contexts` 实名：

```
/data/tombstones(/.*)?                     u:object_r:tombstone_data_file:s0
/data/vendor/tombstones/wifi(/.*)?         u:object_r:tombstone_wifi_data_file:s0
/data/anr(/.*)?                            u:object_r:anr_data_file:s0
/data/system/dropbox(/.*)?                 u:object_r:dropbox_data_file:s0
/dev/socket/tombstoned_crash               u:object_r:tombstoned_crash_socket:s0
/dev/socket/tombstoned_java_trace          u:object_r:tombstoned_java_trace_socket:s0
/dev/socket/tombstoned_intercept           u:object_r:tombstoned_intercept_socket:s0
```

「谁能读 tombstone」的完整答案是四层叠加：目录 mode + 属主 `system:system` + `encryption=Require` + SELinux 类型 `tombstone_data_file`。只看权限位会得出错误结论。tombstone **文件**权限位本轮未取到一手行（tombstoned 用 `rename_tombstone_fd()` 把临时 fd 重命名进 `dir_fd`，无 `open(..., 0NNN)` 形式命中），见 §11。

### 1.5 ANR 是同一条管道的另一支

Java 侧 trace 走 `/dev/socket/tombstoned_java_trace`（`tombstoned_java_trace_socket`）与 `tombstoned.max_anr_count`（默认 64），目录仍是 `/data/anr`（`anr_data_file`）。「Android 12 之后 ANR trace 搬到别处/只留在 logcat」的说法本轮无一手页面支撑，见 §11。

---

## 2. dropbox：系统级事件 blob 存储

### 2.1 定位与实名

客户端类 `android.os.DropBoxManager`，注解 `@SystemService(Context.DROPBOX_SERVICE)`。类 javadoc 原文给了最准确的定义：

> "Enqueues chunks of data (from various sources -- application crashes, kernel log records, etc.). The queue is size bounded and will drop old data if the enqueued data exceeds the maximum size. You can think of this as a persistent, system-wide, blob-oriented "logcat"."

以及一句对隐私评估关键的承认：`DropBoxManager entries are not sent anywhere directly, but other system services and debugging tools may scan and upload entries for processing.` —— **dropbox 不主动外发，但它是别的外发组件的池子**。

服务端文件实名要注意目录层：`frameworks/base/services/core/java/com/android/server/DropBoxManagerService.java`，**不在 `server/os/`**（`server/os/…`、`server/am/…` 变体均无此文件）。启动点见 `SystemServer.java`：`t.traceBegin("StartDropBoxManager"); mSystemServiceManager.startService(DropBoxManagerService.class);`。存储路径来自同文件构造函数 `this(context, new File("/data/system/dropbox"), FgThread.get().getLooper());`

### 2.2 tag 的真实生成方式

错误密度最高的一点：tag 是**小写、由「进程类别 + 事件类型」拼接**的字符串，不是大写常量。一手行在 `ActivityManagerService.java`：

```java
final String dropboxTag = processClass(process) + "_strictmode";
final String dropboxTag = processClass(process) + "_" + eventType;
```

可列举 tag 的一手来源只有两处：

1. `core/res/res/values/config.xml` 的 `<string-array name="config_dropboxLowPriorityTags">`：`data_app_strictmode`、`data_app_wtf`、`keymaster`、`netstats`、`system_app_strictmode`、`system_app_wtf`、`system_server_strictmode`、`system_server_wtf`。
2. 服务端常量 `DISABLED_BY_DEFAULT_TAGS = List.of("data_app_wtf", "system_app_wtf", "system_server_wtf")`，注释 `Tags that we should drop by default.` —— 这三类 wtf 默认**不落**。

防御结论：默认关闭集合与低优先级集合都是可被改动的开关，合规盘点应确认目标设备上 wtf 系列是否仍处于关闭态。

### 2.3 API、限额与读取权限

API 实名：`addText(tag, data)`、`addData(tag, data, flags)`、`addFile(tag, fd, flags)`、`isTagEnabled(tag)`、`getNextEntry(tag, millis, callingPackage)`。常量：`EXTRA_TAG = "tag"`、`EXTRA_TIME = "time"`、`EXTRA_DROPPED_COUNT = "android.os.extra.DROPPED_COUNT"`、`ACTION_DROPBOX_ENTRY_ADDED`。flags 文案：`Flag value: Content is human-readable UTF-8 text (can be combined with IS_GZIPPED)` / `Flag value: Content can be decompressed with java.util.zip.GZIPOutputStream.`。容量常量 `PROTO_MAX_DATA_BYTES = 256 * 1024`、`COMPRESS_THRESHOLD_BYTES = 16_384`：超 16 KiB 才压缩、单条上限 256 KiB，决定了 dropbox 目录「增长到尺寸边界后截断」的行为模型。

权限实名（`core/res/AndroidManifest.xml`）：`android.permission.READ_DROPBOX_DATA` 为 `signature|privileged|development`，注释 `Allows an application to access the data in Dropbox. Not for use by third-party applications.`，且带 `@FlaggedApi("com.android.server.feature.flags.enable_read_dropbox_permission")`（权限本身还受 feature flag 门控）；`android.permission.PEEK_DROPBOX_DATA` 为 `signature`。

### 2.4 `dumpsys dropbox` 的真实子命令

help 文案逐字：

```
Dropbox (dropbox) dump options:
  [-h|--help] [-p|--print] [-f|--file] [timestamp]
    -p|--print: print full contents of each entry
    -f|--file: print path of each entry's file
    --proto: dump data to proto
Usage: dumpsys dropbox [--print|--file] [YYYY-mm-dd] [HH:MM:SS] [tag]
```

故 `dumpsys dropbox --print <tag>` 成立（tag 是位置参数）。service 级命令实名：`help`、`set-rate-limit PERIOD`、`add-low-priority TAG`、`remove-low-priority TAG`、`restore-defaults`。help 中**无** `--zip`；`file_contexts` 中也无独立 `/system/bin/dropbox` 二进制。

### 2.5 旁支留存：watchdog 与应用退出信息

`services/core/java/com/android/server/Watchdog.java` 实名：`TIMEOUT_HISTORY_FILE = "/data/system/watchdog-timeout-history.txt"`、`PROP_FATAL_LOOP_COUNT = "framework_watchdog.fatal_count"`、`framework_watchdog.fatal_window.second`、线程 `"watchdog"` 与 `"watchdog.monitor"`；AMS 另有真实属性键 `debug.trigger.watchdog`（注释：`If debug.trigger.watchdog is set to 1, sleep 10 minutes with the AM lock held, which would cause a watchdog kill.`）。`/data/system/watchdog-timeout-history.txt` 是常被忽略的长期运行异常痕迹文件。

`AppErrors.java` 实名：`ApplicationExitInfo.REASON_ANR`、`ApplicationErrorReport.TYPE_ANR`、`report.anrInfo`（`.activity/.cause/.info`）、`Settings.Secure.ANR_SHOW_BACKGROUND`、`MetricsProto.MetricsEvent.ACTION_APP_ANR`。意义：ANR/崩溃不只躺在 `/data/anr`，还会以 `ApplicationExitInfo` 结构化形式提供给组件，构成第二类读取面。

---

## 3. bugreport：单一最大的聚合外泄面

### 3.1 内容构成与用途

现行页 `docs/core/tests/debug/read-bug-reports`（旧猜测路径 `/docs/core/tests/debug/bugreport` 已 404）：

> "A bugreport reflects multiple parallel timelines: system log, event log, kernel log, and multiple specialized timelines for broadcasts, battery stats, etc. Unfortunately, timelines are often reported using different time bases."

用途：`Bugreports can help you identify what's causing Application Not Responding (ANR) errors and deadlock events.` 文档中真实出现的标签类别：`am_anr`、`am_proc_died`、`am_proc_start`、`am_low_memory`、`am_focused_activity`、`screen_toggled`、`ANR in <package>`；文件名形态 `bugreport-YYYY-MM-DD-HH-MM-SS.txt`；内核行示例 `init: Starting service 'bugreport'...`（存在名为 `bugreport` 的 init service）。

观测者效应（防御要点）原文两处：`(Keep in mind the bugreport being gathered can influence thrashing indicators.` 与 `most activity isn't very useful as the act of taking the bugreport loads the system substantially.` 结论：采集动作本身会改变内存抖动/CPU 争用指标，不要把现场 bugreport 当中性观测。

### 3.2 `dumpstate` 的策略实名与采集面

`file_contexts` 中与 bugreport 采集相关的行**只有两条**：`/system/bin/dumpstate u:object_r:dumpstate_exec:s0`、`/dev/socket/dumpstate u:object_r:dumpstate_socket:s0`。`private/dumpstate.te`：`typeattribute dumpstate coredomain;`、`init_daemon_domain(dumpstate)`、`tmpfs_domain(dumpstate)`、`net_domain(dumpstate)`、`r_dir_file(dumpstate, domain)`。最后一条语义即「对所有 domain 的目录/文件可读」，是「bugreport 是聚合面」的策略级证据。

| 采集目标 | `private/dumpstate.te` 一手依据（注释 + 规则） |
|---|---|
| dropbox（只读） | `# /data/system/dropbox for dropbox entries`；`allow dumpstate dropbox_data_file:dir r_dir_perms;` / `:file r_file_perms;` |
| `/data/anr`（可写） | `# Create and write into /data/anr/`；`allow dumpstate anr_data_file:dir rw_dir_perms;` / `:file create_file_perms;` |
| pstore 内核控制台 | `# Read /sys/fs/pstore/console-ramoops`；`allow dumpstate pstorefs:dir r_dir_perms;` / `:file r_file_perms;` |
| 崩溃现场拦截 | `# Connect to tombstoned to intercept dumps.`；`unix_socket_connect(dumpstate, tombstoned_intercept, tombstoned)` |
| perfetto | `# For starting (and killing) perfetto --save-for-bugreport.` 与 `/data/misc/perfetto-traces/bugreport/*.pftrace`；`allow dumpstate perfetto_traces_bugreport_data_file:dir rw_dir_perms;` |
| 对其他进程 | `allow dumpstate { appdomain system_server zygote app_zygote }:process signal;`；`allow dumpstate domain:process getattr;` |
| 域转换 | `domain_auto_trans(dumpstate, perfetto_exec, perfetto)`、`(dumpstate, vdc_exec, vdc)`、`(dumpstate, derive_sdk_exec, derive_sdk)` |

两条注释值得单列：`# Connect to tombstoned to intercept dumps.` 是「诊断链路可被系统组件读取」的成文依据（实现侧实名 `InterceptManager`、`rename_tombstone_fd()`），一次 bugreport 会把本来写成 tombstone 的现场纳入聚合；`# Create and write into ... (which is labelled as dumpstate_tmpfs) to avoid leaking a FD to the bugreport` 说明官方在策略文件里明确承认需要防止 FD 泄漏进 bugreport，即承认它是一个需隔离的外泄容器。

### 3.3 权限与同意路径

`core/res/AndroidManifest.xml` 逐个核实到：`android.permission.DUMP`（`signature|privileged|development`）、`android.permission.TRIGGER_SHELL_BUGREPORT`（`signature`）、`android.permission.CAPTURE_CONSENTLESS_BUGREPORT_ON_USERDEBUG_BUILD`、`android.permission.CAPTURE_CONSENTLESS_BUGREPORT_DELEGATED_CONSENT`、`android.permission.MANAGE_DEVICE_POLICY_BUGREPORT`。Java 侧：`services/core/java/com/android/server/os/BugreportManagerService.java`、`BugreportManagerServiceImpl.java`。

「consentless」这组权限的存在本身就是防御结论：**AOSP 成文区分了需要用户同意与不需要用户同意两条采集路径**，评估时应检查设备是否可能命中后者。流传的 `android.permission.BUGREPORT` / `BUGREPORTING` 本轮未在 manifest 中核实到，见 §11。

### 3.4 用户可见文案

`packages/apps/Settings/res/values/strings.xml` 实名行：`enterprise_privacy_bug_reports` = `Most recent bug report`（企业隐私披露页把「最近一次 bugreport」列为披露项）；`share_remote_bugreport_dialog_message` = `Your IT admin requested a bug report to help troubleshoot this device. Apps and data may be shared, and your device may temporarily slow down.`；`share_remote_bugreport_dialog_message_finished` = `…Apps and data may be shared.`

---

## 4. 日志读取边界：`logcat` / `logd` / `READ_LOGS`

### 4.1 buffer 是枚举，不是属性

`system/logging/liblog/include/android/log.h` 的 `enum log_id_t` 实名：`LOG_ID_MIN = 0`、`LOG_ID_MAIN = 0`、`LOG_ID_RADIO = 1`、`LOG_ID_EVENTS = 2`、`LOG_ID_SYSTEM = 3`、`LOG_ID_CRASH = 4`、`LOG_ID_STATS = 5`、`LOG_ID_SECURITY = 6`、`LOG_ID_KERNEL = 7`、`LOG_ID_MAX`、`LOG_ID_DEFAULT = 0x7FFFFFFF`。buffer 名由该表派生（`logcat.cpp` 调 `android_log_id_to_name(static_cast<log_id_t>(i))`），故 `logcat -b crash` 成立。CLI 文案：`-b BUFFER, --buffer=BUFFER`、`Unknown -b buffer '%s'.`。`security` buffer 被代码特判：`if (allow_security || name != "security")`，即它需要额外许可 —— **各 buffer 并不等价**。

### 4.2 详细度配置面

`liblog/properties.cpp` 注释实名：`log.tag.<tag>`、`persist.log.tag.<tag>`、`log.tag`、`persist.log.tag`，并明确 `We do not support ro.log.tag*`；`static const char* log_namespace = "persist.log.tag.";`。官方页 `understanding-logging`：`log.tag.* properties are reset on reboot. There are persistent variants that remain across reboots as well.`、`adb shell setprop persist.log.tag.`、`Android uses the liblog library for general system logging.`。防御要点：`persist.log.tag.*` 跨重启存活，一次调试期把 tag 调到 `V` 后可能长期滞留，属典型配置漂移项，应纳入基线核查。

### 4.3 `READ_LOGS`：官方最直接的成文承认

`core/res/AndroidManifest.xml` 逐字：说明 `Allows an application to read the low-level system log files.`；警告 `<p>Not for use by third-party applications, because Log entries can contain the user's private information.`；保护级 `android:protectionLevel="signature|privileged|development"`。`development` 意味着只经开发/adb 途径授予，第三方应用拿不到。

### 4.4 构建类型边界写在策略文件里

`private/logd.te` 顶部注释 `# only on userdebug or eng builds`，其下多条 `userdebug_or_eng(` 包裹的规则，涉及 `-crash_dump`、`-llkd`、`-misc_logd_file`、`-logpersist`（logcatd 持久化日志）、`-su`。即「logpersist / su / coredump 类日志增强只在 userdebug 与 eng 开放」是策略宏而非传言。另有 `neverallow { domain userdebug_or_eng(`-crash_dump -llkd') } logd:process ptrace;`，注释 `# ... and nobody may ptrace me (except on userdebug or eng builds)`。内核日志面：`r_dir_file(logd, proc_kmsg)`、`allow logd kmsg_device:chr_file { getattr w_file_perms };`、`allow logd pstorefs:dir search;`、`allow logd pstorefs:file r_file_perms;`；以及 `dontaudit domain runtime_event_log_tags_file:file { map open read };`。

`file_contexts`：`/system/bin/logd u:object_r:logd_exec:s0`、`/dev/socket/logd`、`/dev/socket/logdr`（读）、`/dev/socket/logdw`（写）、`/data/misc/logd(/.*)? u:object_r:misc_logd_file:s0`；`/system/bin/logcatd` 存在但标签是 `logcat_exec`（易踩的命名不一致）。

关于 `ro.build.type`：它是通用属性，但「`ro.build.type` 直接决定日志详细度」这层绑定本轮未取到一手条款。已核实的成文绑定只有三条：`userdebug_or_eng` 宏、`log.tag.*` / `persist.log.tag.*` 机制、`READ_LOGS` 的 `development` 级别。`logd` 内按 UID 做 `READ_LOGS` 判定的源文件（常被提及的 `Access.cpp`）本轮未确认。

---

## 5. 内核侧留存：`pstore` / `ramoops` / `dmesg_restrict` / `kptr_restrict`

### 5.1 上一次开机的内核日志现在住在 pstore

`rootdir/init.rc`（main）已不再出现 `last_kmsg`，替代块逐字：

```
# pstore/ramoops previous console log
mount pstore pstore /sys/fs/pstore nodev noexec nosuid
chown system log /sys/fs/pstore
chmod 0550 /sys/fs/pstore
chown system log /sys/fs/pstore/console-ramoops
chmod 0440 /sys/fs/pstore/console-ramoops
chown system log /sys/fs/pstore/console-ramoops-0
chmod 0440 /sys/fs/pstore/console-ramoops-0
chown system log /sys/fs/pstore/pmsg-ramoops-0
chmod 0440 /sys/fs/pstore/pmsg-ramoops-0
```

结论：现行 AOSP 由 `/sys/fs/pstore/console-ramoops[-0]` 与 `pmsg-ramoops-0`（属主 `system:log`，mode `0440`）承担「上一次开机的内核控制台日志」；`/proc/last_kmsg` 在 main 的 `init.rc`、`file_contexts`、`genfs_contexts` 中均无引用（负证据）。官方弃用公告页未找到，故此结论以负证据给出，不作正式弃用声明。

内核一手文档 `admin-guide/ramoops.html`：`Reading the data — The dump data can be read from the pstore filesystem. The format for these files is dmesg-ramoops-N, where N is the record number in memory. To delete a stored record from RAM, simply unlink the respective pstore file.` 同页：`The memory area is divided into record_size chunks (also rounded down to power of two)...`，设备树实名 `compatible = "ramoops";` 与 `record-size = <0x4000>; console-size = <0x4000>;`；ftrace 存档入口 `record_ftrace`（debugfs `/sys/kernel/debug/pstore`）。Android 侧读取者实名：`logd`（`allow logd pstorefs:file r_file_perms;`）与 `dumpstate`（`# Read /sys/fs/pstore/console-ramoops`）。另有 `genfscon proc /kmsg u:object_r:proc_kmsg:s0`。

### 5.2 `dmesg_restrict` / `kptr_restrict` 的真实管控点

流传甚广的错误必须纠正：**「Android 在 init.rc 里设置 dmesg_restrict / kptr_restrict」本轮核实不成立**。`init.rc` 的 `write /proc/sys/...` 清单含 `write /proc/sys/kernel/panic_on_oops 1`、`write /proc/sys/kernel/randomize_va_space 2`、多条 `write /proc/sys/kernel/sched_*`、`write /proc/sys/vm/mmap_min_addr 32768`，但**不含**这两项（负证据）。真实管控来自两层：

1. 内核 sysctl 语义（`admin-guide/sysctl/kernel.html`）：`dmesg_restrict — This toggle indicates whether unprivileged users are prevented from using dmesg(8) to view messages from the kernel's log buffer. When dmesg_restrict is set to 0 there are no restrictions. When dmesg_restrict is set to 1, users must have CAP_SYSLOG to use dmesg(8).` 同页写明 `DMESG_RESTRICT sets the default value of dmesg_restrict.`，即默认值由内核编译开关决定；`kptr_restrict` 亦列于同一 sysctl 页（本轮未取到逐字定义段，只引用「同页存在」）。
2. SELinux 类型（`private/genfs_contexts`）：`genfscon proc /sys/kernel/dmesg_restrict u:object_r:proc_security:s0`、`genfscon proc /sys/kernel/kptr_restrict u:object_r:proc_security:s0`；同组归 `proc_security` 的还有 `modules_disabled`、`randomize_va_space`，`panic_on_oops` 归 `proc_panic`。即 sysctl 节点本身由 `proc_security` 管控，不是任意 app/shell 可读写。

`CONFIG_SECURITY_DMESG_RESTRICT` 在 Android GKI defconfig 中的具体行本轮未核实（尝试路径全部 404），见 §11；请勿当作「已确认开启」。

### 5.3 跟踪类产物落盘位置

`private/file_contexts`：`/data/local/traces(/.*)? u:object_r:trace_data_file:s0`、`/data/misc/perfetto-traces(/.*)? u:object_r:perfetto_traces_data_file:s0`、`/data/misc/perfetto-traces/bugreport(.*)? u:object_r:perfetto_traces_bugreport_data_file:s0`、`/data/misc/perfetto-traces/profiling(/.*)? u:object_r:perfetto_traces_profiling_data_file:s0`。debug 索引确认存在的子页：`ftrace`、`systrace`、`strace`、`gdb`、`rescue-party`、`scoped-vendor-logging`、`storaged`、`native-memory`、`harness`、`eval_perf`。二进制实名另含 `/system/bin/perfetto`、`/system/bin/traced`、`/system/bin/traced_perf`、`/system/bin/traced_probes`、`/system/bin/traced_relay`；socket 含 `/dev/socket/traced_consumer`、`/dev/socket/traced_producer`、`/dev/socket/traced_perf`。`traced` 系列的域名与策略细节本轮未逐项核实，正文只给 `file_contexts` 层实名。

---

## 6. 「日志含敏感信息」的官方成文位置

常见误区是以为这句话写在 Security Overview。实际一手成文有三处：

1. `core/res/AndroidManifest.xml` 的 `READ_LOGS` 注释：`Not for use by third-party applications, because Log entries can contain the user's private information.`
2. AOSP Settings `res/values/strings.xml` L1992 `nfc_verbose_vendor_log_summary`：`Include additional device-specific vendor logs in bugreports, which may contain private information.` —— 官方成文承认「厂商日志进 bugreport 且可能含私密信息」。
3. 远程 bugreport 同意文案：`Apps and data may be shared.`，配合企业披露条目 `enterprise_privacy_bug_reports` = `Most recent bug report`。

侧面成文另两处：`DropBoxManager` javadoc 承认收集源含 `application crashes, kernel log records, etc.` 且 `other system services and debugging tools may scan and upload entries for processing.`；官方文档承认 bugreport 是 `system log, event log, kernel log` 的聚合。防御结论：**聚合面 = 单个最大暴露面**，「我只看了某一个日志」的判断不成立。

反向结论：Security Overview（`docs/security/overview/`，可访问）本轮 grep 未命中「日志/崩溃文件可能含敏感信息」的专门句子（命中内容为沙箱、TrustZone、开源审阅等）；CDD（`docs/compatibility/16/android-16-cdd`）未定位到相关条款号。故「Security Overview 里说日志含敏感信息」属未核实引用，见 §11。Crashlytics / ACRA 是第三方崩溃上报 SDK，非 AOSP 机制，本轮未取到 Google 官方条款。

---

## 7. 常见误传纠正

以下 16 条均为「网上广泛流传、本轮核实不成立或无法支撑」的表述。列出它们是本手册最重要的防御性价值之一：错误的暴露面认知会直接导致错误的基线。

1. **`DropBoxManager.TAG_APP_CRASH` / `TAG_SYSTEM_BOOT` / `TAG_WATCHDOG`**：现行 main 分支不存在这些大写常量（对该文件 grep `TAG_` 只命中日志用的 `private static final String TAG = "DropBoxManager";`）。真实机制是 `dropboxTag = processClass(process) + "_" + eventType`（小写拼接）。
2. **大写 tag 字面量 `SYSTEM_BOOT` / `APP_CRASH` / `SYSTEM_TOMBSTONE` / `SYSTEM_LAST_KMSG` / `WATCHDOG`**：本轮在 `DropBoxManagerService.java`、`AppErrors.java`、`SystemServer.java` 均未取到一手写入点。可引用的只是 `config.xml` 与 `DISABLED_BY_DEFAULT_TAGS` 里的小写 tag。
3. **「debuggerd 只保留 10 个 tombstone，00–09 循环覆盖」**：这是文档页历史文案；代码默认 `tombstoned.max_tombstone_count = 32`、`tombstoned.max_anr_count = 64`。引用时必须分别标注文档与代码。
4. **`debug.tombstone*` 属性**：查无出处（`tombstoned.cpp` / `debuggerd.cpp` 中 grep `"debug` 未见此类键）。
5. **`dumpsys dropbox --zip`**：不在 help 文案中；真实选项是 `-p|--print`、`-f|--file`、`--proto`，加日期/时间/tag 位置参数。
6. **`adb shell dropbox` / 独立 `/system/bin/dropbox` CLI**：`file_contexts` 中 grep 不到该路径，不成立。
7. **`android.permission.BUGREPORT` / `android.permission.BUGREPORTING` 当作 manifest 权限**：本轮 grep `name="android.permission` 未见二者。真实的是 `DUMP`、`TRIGGER_SHELL_BUGREPORT`、`CAPTURE_CONSENTLESS_BUGREPORT_ON_USERDEBUG_BUILD`、`CAPTURE_CONSENTLESS_BUGREPORT_DELEGATED_CONSENT`、`MANAGE_DEVICE_POLICY_BUGREPORT`。
8. **`bugreportz` 拥有独立 SELinux 域与专用 socket**：现行 `file_contexts` 与 Android 12 策略快照 `prebuilts/api/32.0/private/file_contexts` 都只有 `/system/bin/dumpstate`（`dumpstate_exec`）与 `/dev/socket/dumpstate`（`dumpstate_socket`）两行；`dumpstate.te` 仅留注释 `# For collecting bugreports.`。
9. **「init.rc 设置 `dmesg_restrict` / `kptr_restrict`」**：init.rc 的 `/proc/sys` 写入清单里没有这两项（负证据）；实际由内核 sysctl 语义（kernel.org：`users must have CAP_SYSLOG to use dmesg(8)`）+ SELinux `genfscon proc /sys/kernel/dmesg_restrict u:object_r:proc_security:s0` 管控。
10. **`STAGE_*` 崩溃阶段常量**：本轮未在相关文件 grep 到，不成立为事实。
11. **「`DropBoxManagerService` 在 `server/os/` 下」**：真实位置是 `services/core/java/com/android/server/DropBoxManagerService.java`（`server/` 根）。
12. **`android.log_id_*` 属性**：不存在；实名是 C 枚举 `LOG_ID_*` 与函数 `android_log_id_to_name()`。
13. **「Security Overview 说崩溃文件可能含敏感信息」**：本轮 grep 未命中；真正的成文位置是 `READ_LOGS` 注释与 Settings 的两条文案（§6）。
14. **公告 URL 写成带年份目录层**：`/docs/security/bulletin/2024/01-01` 实测 404；正确形式是 `https://source.android.com/docs/security/bulletin/2024-01-01`。同理 `/docs/core/tests/debug/native-crashes`、`/debug/bugreport`、`/debug/logcat`、`/devices/tech/debug/*` 等旧路径已失效。
15. **「公告按 daemon 名（`logd`/`dumpstate`/`tombstoned`）列出组件」**：现行表头的组件是以**章节名**呈现（`Android Framework`、`System`、`Google Play System`），Type 列用缩写 `ID` / `EoP` / `RCE`；具体源文件名只出现在 per-CVE 描述里。
16. **「不在 CISA KEV = 没有被真实利用」**：KEV 只收录 CISA 认定「有合理理由推断已在野利用」的条目，且以 Windows/Linux/macOS/网络设备生态为主，Android 条目稀少。不在列仅等于「CISA 未作该认定」，在野利用信号要回到公告页措辞判断。

补充一条工具名纠正：社区流传的 `DumpReader` 类工具查无出处，不应写入任何正文。

---

## 8. 公开案例与「观测缺口」的方法论结论

### 8.1 可完整核实的只有 3 条

1. **`CVE-2023-40085`** —— 公告 `2023-06-01`，表行原文 `CVE-2023-40085 A-269271098 ID High 12, 12L, 13`：类型 `ID`（Information Disclosure）、严重级别 `High`、AOSP 版本 `12, 12L, 13`、bug 追踪号 `A-269271098`。NVD（vulnStatus `Analyzed`）原文：`In convertSubgraphFromHAL of ShimConverter.cpp, there is a possible out of bounds read due to a missing bounds check. This could lead to local information disclosure with no additional execution privileges needed. User interaction is not needed for exploitation.`
   与本主题的**关系与边界**：它属于「本地信息泄露 + 无需额外权限 + 无需用户交互」这一类，可作为诊断/聚合面暴露的**类比案例**；但组件是 NNAPI shim（`ShimConverter.cpp`），**不是** dropbox/tombstone 本体 —— 把它写成崩溃诊断漏洞就是错的。
2. **`CVE-2024-0019`** —— 公告 `2024-01-01`，表行原文 `CVE-2024-0019 A-294104969 ID High 12, 12L, 13, 14`：类型 `ID`、`High`、AOSP `12, 12L, 13, 14`、`A-294104969`。NVD（vulnStatus `Modified`）原文：`In setListening of AppOpsControllerImpl.java, there is a possible way to hide the microphone privacy indicator when restarting systemUI due to a missing check for active recordings. This could lead to local denial of service with no additional execution privileges needed. User interaction is needed for exploitation.`
   **必须如实标注的核验冲突**：NVD 描述落点是「AppOps 麦克风隐私指示器 / 本地 DoS」，与公告行内的 `ID` 类型标签不完全一致（连「是否需要用户交互」也与公告口径不同）。两个来源要同时给出，不可择一伪造。
3. **`CVE-2022-22706`** —— Mali GPU 内核驱动，出现在公告 `2023-06-01` 的在野利用提示句中，公告原文措辞：`Note : There are indications that CVE-2022-22706 may be under limited, targeted exploitation.` 经 CISA KEV JSON 命中验证：**已在 KEV 列表内**。引用「在野利用」时必须停留在 `indications … may be under limited, targeted exploitation` 这一成文级别，不要升级为断言。

### 8.2 KEV 核对结果（3 条逐一）

数据源 `https://www.cisa.gov/sites/default/files/feeds/known_exploited_vulnerabilities.json`（本轮抓取 1,810,928 字节、约 1758 条）。结果：`CVE-2024-0019` 未收录；`CVE-2023-40085` 未收录；`CVE-2022-22706` 已收录（与 2023-06-01 公告措辞互相印证）。结合 §7 第 16 条：**两条未收录不等于两条无真实利用**，只等于 CISA 未作该认定。

### 8.3 方法论结论：这是观测缺口，不是风险缺口

本轮对 `dumpstate`、`dropbox`、`tombstone`、`debuggerd`、`logd`、`logcat`、`pstore`、`last_kmsg`、`watchdog`、`bugreport`、`diagnot*` 做了公告全文 grep：`2024-01-01`（正文 26,893 字符）0 命中、`2023-06-01` 0 命中、`2020-09-01`（`ID` 行 15 条）0 命中、`2024-11-01`（`ID` 行 4 条）0 命中；`2021-06-05`、`2019-11-05` 只取到 10,593 字符的短页（Pixel 专属/旧格式，正文不完整），无法判定。

由此六个方向本轮**未检索到条目**：`bugreport`/`dumpstate` 权限缺陷、`dropbox`/`tombstone` 信息泄露、`logd`/`logcat` 策略缺陷、`debuggerd` 提权、`pstore`/`last_kmsg` 读取、`DumpReader` 型工具（该名称本身查无出处）。

正确的读法是：这是**观测缺口（observation gap）而非风险缺口（risk gap）**。公告不以 daemon 命名条目、组件按章节归类、定位要靠 per-CVE 描述里的文件名（`ShimConverter.cpp` 型），加之 source.android.com 需带 `Accept: text/plain` 才返回服务端渲染正文（本轮多次「查无命中」的真实原因）—— 这些是**检索手段的局限**。**不得据此宣称崩溃诊断与日志这一面没有漏洞**；反过来也不得伪造 8–12 条带组件名的案例来填补空白。

---

## 9. 可观测面与自查

### 9.1 一手文案支撑的读取面（防御用途）

- `adb shell dumpsys dropbox --print <tag>` —— 由 help 文案 `[-h|--help] [-p|--print] [-f|--file] [timestamp]` 与 `Usage: dumpsys dropbox [--print|--file] [YYYY-mm-dd] [HH:MM:SS] [tag]` 支撑；`-f|--file` 只打印**路径**，适合做「设备上留了多少条」的计数自查而不触碰内容。service 级命令 `add-low-priority` / `remove-low-priority` / `set-rate-limit` / `restore-defaults` 是策略回收入口。
- `logcat -b crash` 等 buffer —— 由 `LOG_ID_*` 枚举支撑；注意 `security` buffer 的代码特判。
- `getprop` 相关真实键：`tombstoned.max_tombstone_count`、`tombstoned.max_anr_count`、`debug.trigger.watchdog`、`framework_watchdog.fatal_count`、`framework_watchdog.fatal_window.second`、`log.tag.<TAG>`、`persist.log.tag.<TAG>`、`log.tag`、`persist.log.tag`。
- `debuggerd -b <pid>` —— 帮助文案 `"-b, --backtrace    just a backtrace rather than a full tombstone"`，即「只取回溯、不产 tombstone」，自查时比产完整 tombstone 的信息外溢更小。

### 9.2 留存盘点（一手 mode/属主）

| 位置 | 一手依据 | 关注点 |
|---|---|---|
| `/data/tombstones` | `mkdir /data/tombstones 0775 system system encryption=Require` | 上限 32（文档仍称 10）；`tombstone_data_file` |
| `/data/anr` | `mkdir /data/anr 0775 system system encryption=Require` | 上限 64；`anr_data_file`；`dumpstate` 可写 |
| `/data/system/dropbox` | `mkdir /data/system/dropbox 0700 system system` | `dropbox_data_file`；`dumpstate` 只读 |
| `/sys/fs/pstore` | `chmod 0550`，`console-ramoops[-0]` / `pmsg-ramoops-0` `chmod 0440`，属主 `system:log` | 跨重启残留的上次内核控制台 |
| `/data/vendor/tombstones[/wifi]` | `0771 root:root` / `0771 wifi:wifi` | 厂商侧独立留存 |
| `/data/system/watchdog-timeout-history.txt` | `Watchdog.java` 实名 | 致命循环历史 |
| `/data/misc/perfetto-traces/bugreport`、`/data/local/traces` | `file_contexts` 实名 | 跟踪产物会进 bugreport |

user 版可读性的正确写法：上述 mode/属主只覆盖 Unix 权限层，非属主 UID（含 adb shell 域）能否读取还取决于 SELinux 类型与 `logd.te` 的 `userdebug_or_eng` 门。本轮**未取到 shell 域对这些类型的 allow/neverallow 具体行**，因此结论只能是「以设备实测 + 本机 sepolicy 版本为准」，不断言 user 版一定可读或一定不可读。

### 9.3 五条基线动作

1. 按 **32/64** 的默认上限核对 tombstone/ANR 留存，别按文档的 10 估容量。
2. 核查 `persist.log.tag.*` 是否有调试期遗留（跨重启存活）。
3. 核查 `data_app_wtf` / `system_app_wtf` / `system_server_wtf` 是否仍处于 `DISABLED_BY_DEFAULT_TAGS` 默认关闭态。
4. 把 bugreport 当聚合外泄容器对待：清点时假定它含 dropbox、`/data/anr`、`console-ramoops`、经 `tombstoned_intercept` 拦截的崩溃现场与 `--save-for-bugreport` 的 perfetto trace。
5. 检查是否存在 consentless 路径（`CAPTURE_CONSENTLESS_BUGREPORT_*` 两权限与 MDM `MANAGE_DEVICE_POLICY_BUGREPORT`）被启用的可能。

---

## 10. 与其他手册的分工

本手册只回答「诊断产物与日志留在哪里、谁能读、官方在哪里成文承认过隐私风险」，不涉及定级、攻击链路与设备迁移操作。与《Android 漏洞定级与安全性评估》（[android-vulnerability-severity-assessment.md](/android-vulnerability-severity-assessment/)）的边界：本手册给公告表头实名（`ID` / `EoP` / `RCE`）与在野利用措辞的取证口径，**不**判定 CVSS 或给漏洞打分，定级结论一律以该手册为准。与《ADB 备份与数据迁移》（[android-adb-backup-transfer.md](/android-adb-backup-transfer/)）的边界：本手册只描述 `READ_LOGS`、`DUMP`、`TRIGGER_SHELL_BUGREPORT` 等权限的保护级与采集面，**不**展开 adb 操作流程与备份数据抽取，adb 侧的实操与限制归该手册。崩溃/诊断组件的漏洞类型定位见《底层漏洞类型图谱》（[android-low-level-vulnerability-types.md](/android-low-level-vulnerability-types/)），基带与 TEE 侧的诊断产物不在本手册范围（见《基带与电话栈》《TEE 与密钥认证》），主线组件与 APEX 边界见《Mainline APEX 模块》，多用户下的诊断文件隔离见《多用户与沙箱隔离》。

---

## 11. 未核实与来源失效

以下条目**不得**作为事实引用，正文若出现同名内容均已在 §7 作为误传纠正处理。

1. `DropBoxManager.TAG_APP_CRASH` / `TAG_SYSTEM_BOOT` / `TAG_WATCHDOG` 等大写常量 —— 现行 main 分支查无出处。
2. 大写 dropbox tag（`SYSTEM_BOOT`、`APP_CRASH`、`SYSTEM_TOMBSTONE`、`SYSTEM_LAST_KMSG`、`WATCHDOG`）的一手写入点 —— 未找到。
3. `debug.tombstone*` 属性名 —— 查无出处。
4. tombstone **文件**的权限位（0600/0640 等）—— 未取到一手行（`rename_tombstone_fd()` 路径无 `open(..., 0NNN)`）。
5. `dumpsys dropbox --zip`、独立 `/system/bin/dropbox` CLI —— 均无一手依据。
6. `android.permission.BUGREPORT` / `android.permission.BUGREPORTING` —— 未在 manifest 核实到。
7. `STAGE_*` 崩溃阶段常量 —— 未 grep 到。
8. `bugreportz` 的独立 SELinux 域与专用 socket —— 现行与 `prebuilts/api/32.0` 快照 `file_contexts` 均无，视为不成立。
9. 「init.rc 设置 `dmesg_restrict` / `kptr_restrict`」—— 负证据（写入清单不含二者）。
10. `CONFIG_SECURITY_DMESG_RESTRICT` 在 GKI defconfig 的具体行 —— 未取到（`android15-6.6/arch/arm64/configs/gki_defconfig`、`kvm_x86_64-gki_defconfig`、`rockpi4_linux_android_defconfig` 全 404）。
11. Security Overview / CDD 中「日志与崩溃文件含敏感信息」的具体条款位置 —— 未定位（已核实的成文位置见 §6）。
12. `dumpstate` / `bugreportz` 源码目录实名（候选 `frameworks/native/cmds/dumpstate/`）—— 未取到列表验证；`system/core/libdumpstate/DumpstateUtil.cpp` 404。
13. 「Android 12+ 起 ANR trace 迁移或只留 logcat」—— 无一手页面（`/docs/core/performance/anrs`、`/docs/google-swc/anrs` 404）。
14. `android.log_id_*` 属性写法、`DumpReader` 工具名、「Debug trails」文档标题 —— 均查无出处。
15. `adb bugreport` 命令的一手文案、开发选项「Take bug report」精确字符串、`logd` 内 `READ_LOGS` 判定源文件、`traced` 系列的 SELinux 域名、`kptr_restrict` 的逐字定义段、`ro.build.type` 与日志详细度的成文绑定 —— 本轮未核实。
16. §8.3 六个方向的公告案例与对应 KEV 结论 —— 本轮未检索到条目。
17. 已失效 URL（实测 404）：`/docs/core/tests/debug/{native-crashes,bugreport,system-debugging,dumpstate,understanding-tombstones,reading-crash-reports,logcat,adb}`、`/docs/sdk/tools/native-debugger`、`/docs/security/bulletin/2024/01-01`、`/docs/security/bulletin/2024-03-05`、`/devices/tech/debug/*` 系列、`system/core/debuggerd/README.md`、`server/os/DropBoxManagerService.java`。
18. 方法论提醒：`source.android.com` 需带 `Accept: text/plain`（或浏览器 UA）才返回服务端渲染正文，否则只得到 JS 外壳；`googlesource` 用 `?format=TEXT` + `base64 -d`；NVD 无 API key 时限流。

---

## 12. 参考

- 官方文档：https://source.android.com/docs/core/tests/debug · https://source.android.com/docs/core/tests/debug/native-crash · https://source.android.com/docs/core/tests/debug/read-bug-reports · https://source.android.com/docs/core/tests/debug/understanding-logging · https://source.android.com/docs/core/tests/debug/scoped-vendor-logging · https://source.android.com/docs/security/overview/
- AOSP 源码：https://android.googlesource.com/platform/system/core/+/refs/heads/main/debuggerd/Android.bp · .../debuggerd/tombstoned/tombstoned.cpp · .../debuggerd/debuggerd.cpp · .../rootdir/init.rc · https://android.googlesource.com/platform/system/sepolicy/+/refs/heads/main/private/file_contexts · .../private/genfs_contexts · .../private/dumpstate.te · .../private/logd.te · .../prebuilts/api/32.0/private/file_contexts
- 框架源码：https://android.googlesource.com/platform/frameworks/base/+/refs/heads/main/core/java/android/os/DropBoxManager.java · .../services/core/java/com/android/server/DropBoxManagerService.java · .../core/res/res/values/config.xml · .../core/res/AndroidManifest.xml · .../services/core/java/com/android/server/Watchdog.java · .../services/core/java/com/android/server/am/ActivityManagerService.java · .../services/core/java/com/android/server/am/AppErrors.java · .../services/java/com/android/server/SystemServer.java · https://android.googlesource.com/platform/packages/apps/Settings/+/refs/heads/main/res/values/strings.xml
- 日志组件：https://android.googlesource.com/platform/system/logging/+/refs/heads/main/liblog/include/android/log.h · .../liblog/properties.cpp · .../logcat/logcat.cpp
- 内核与外部：https://www.kernel.org/doc/html/latest/admin-guide/ramoops.html · https://www.kernel.org/doc/html/latest/admin-guide/sysctl/kernel.html · https://www.cisa.gov/sites/default/files/feeds/known_exploited_vulnerabilities.json · https://services.nvd.nist.gov/rest/json/cves/2.0?cveId=CVE-2023-40085 · https://services.nvd.nist.gov/rest/json/cves/2.0?cveId=CVE-2024-0019
- 安全公告：https://source.android.com/docs/security/bulletin/2024-01-01 · https://source.android.com/docs/security/bulletin/2023-06-01 · https://source.android.com/docs/security/bulletin/2024-11-01

## 13. 联系方式

咨询 Android 系统详情请联系 https://t.me/one00190

*本手册仅供安全学习与防御研究使用；所有组件事实、接口名称与案例归属均取自公开一手来源并附链接，未能核实的表述已单列于 §11 而非写入正文。*
