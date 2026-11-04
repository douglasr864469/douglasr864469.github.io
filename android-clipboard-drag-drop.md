---
layout: page
title: "Android 剪贴板、拖放与跨应用数据传递攻击面手册"
description: "把「复制粘贴」拆成写入／持有／读取／授权／呈现五段：`ClipboardService` 在 system_server、Android 10 起仅默认输入法与焦点应用可读且无助理豁免、`newUri()` 不授权而粘贴时临时授读、拖放四类 URI 标志默认临时；另含首次读取 toast 与 `EXTR..."
date: 2026-11-04
permalink: /android-clipboard-drag-drop/
---

> 复制粘贴是用户最不设防的动作：一段验证码、一张身份证照片、一句聊天内容，都会经过系统剪贴板；拖放则把同样的数据以手势的形式送进另一个应用。Android 在 `ClipboardService` 上叠了十几年的限制——后台读取收窄、首次读取提示、敏感内容预览打码、跨资料隔离——但每一次放宽或收紧都伴随新的边界问题。这份手册沿「写入 → 持有 → 读取 → 授权 → 呈现」这条链，说明剪贴板与拖放的真实进程边界、URI 授权的生命周期、公开缺陷模式与防御手段。只讨论公开且已修复的问题，不含可利用代码。

## 0. 使用方式与边界

这份手册面向三类读者：

- **应用开发者**：需要理解为什么后台读不到剪贴板、`ClipData.newUri()` 不会自动授权、拖放权限为什么在 Activity 销毁后失效。
- **系统与平台工程师**：需要区分 `ClipboardService`、`UriGrantsManagerService`、`ContentProvider` 与 `MediaProvider` 在一条「数据传递」链上的不同职责。
- **安全评估者**：需要在看到「剪贴板」公告时，判断问题落在剪贴板服务、URI 授权服务、内容提供者还是媒体库，而不是把所有涉及「复制」的 CVE 都归为内容泄露。

本文的三条自我约束：

1. 只引用公开且已修复的问题；CVE 编号、落点与描述以 NVD 为准，公告分类与级别以 Android Security Bulletin 为准。
2. 不写利用步骤、不写 PoC、不写可直接复现的攻击参数。
3. 查不到一手来源的说法一律进「未核实与失效来源」。

与已发布手册的分工：

- 《Android 文件路径与 URI 处理攻击面手册》讲路径规范化与 `content://` 的一般处理；本文只讲剪贴板与拖放这两条「临时授权」通道。
- 《Android 多用户、工作资料与沙箱隔离手册》讲用户与资料边界；本文只讲剪贴板跨用户可见性与工作资料隔离的交互。
- 《Android 通知、状态栏与通知监听攻击面手册》讲通知内容暴露；本文把「验证码被后台应用读取」作为剪贴板侧的场景，不重复通知监听。

## 1. 一次复制粘贴的完整旅程

把「用户按了复制」拆开，至少经过七个阶段：

| 阶段 | 主要位置 | 做的事 | 安全判断点 |
|------|----------|--------|------------|
| 1. 构造 | 源应用进程 | 组装 `ClipData`（文本、URI、Intent、HTML） | 是否包含敏感内容；URI 是否可被他人解析 |
| 2. 提交 | 源应用 → `IClipboard` | 调用 `setPrimaryClip()` | 调用者是否有权转授 URI 权限 |
| 3. 持有 | `ClipboardService`（system_server） | 保存当前剪贴板、记录所有者与权限 | 用户边界；替换时撤销旧授权 |
| 4. 通知 | 剪贴板服务 → 监听者 | 触发 `OnPrimaryClipChangedListener` | 谁有资格收到变更通知 |
| 5. 读取 | 目标应用 → `IClipboard` | `getPrimaryClip()` / `hasPrimaryClip()` | 前台/焦点/默认输入法等资格判断 |
| 6. 授权 | 剪贴板服务 → URI 授权服务 | 为 `content://` 授予临时读权限 | 授权对象、时长、撤销时机 |
| 7. 呈现 | 目标应用 / SystemUI | 粘贴、预览、敏感内容打码 | 是否泄露给非预期观察者 |

关键点是：**剪贴板保存的不是「一段文本」，而是一份带所有者与授权关系的对象**。文本只是最常见的形式；URI 条目会牵出内容提供者，Intent 条目会牵出组件调用。把剪贴板当成纯字符串缓冲区，就会漏掉授权这条主线。

第二个关键点是：**「能写」和「能读」不对称**。Android 10 起对读取做了资格限制，但写入基本不受限——任何应用都可以调用 `setPrimaryClip()` 覆盖当前内容。这意味着「剪贴板劫持」（把用户复制的地址替换成攻击者地址）在平台层面并没有被完全禁止，防御要靠应用自身的校验与用户提示。

## 2. 对象模型与进程边界

### 2.1 关键对象

| 对象 | 所在层 | 作用 |
|------|--------|------|
| `ClipboardManager` | 公开 SDK | 应用侧入口，读写剪贴板、注册变更监听 |
| `IClipboard` | AOSP Binder 接口 | 跨进程调用契约 |
| `ClipboardService` | AOSP 内部（system_server） | 持有当前剪贴板、执行读取资格判断与授权 |
| `ClipData` | 公开 SDK | 容器：`ClipDescription` + 一个或多个 `Item` |
| `ClipDescription` | 公开 SDK | 标签、MIME 类型列表、extras |
| `ClipData.Item` | 公开 SDK | 文本、URI、Intent、HTML 四类载荷 |
| `UriGrantsManagerService` | AOSP 内部（system_server） | URI 权限授予与撤销的中央服务 |
| `ContentProvider` | 公开 SDK | 真正提供 `content://` 数据的组件 |
| `DragEvent` / `DragAndDropPermissions` | 公开 SDK | 拖放事件与拖放期间的临时 URI 授权 |

### 2.2 MIME 与载荷类型

| 构造方式 | 典型 MIME | 说明 |
|----------|-----------|------|
| `ClipData.newPlainText()` | `text/plain` | 纯文本 |
| `ClipData.newHtmlText()` | `text/html` | 带 HTML 标记的文本 |
| `ClipData.newIntent()` | `text/vnd.android.intent` | 携带 `Intent` 的条目 |
| `ClipData.newUri()` | 由提供者决定；无法确定时 `text/uri-list` | URI 条目，读取时需解析权限 |

`newUri()` 有一个容易被忽略的细节：它会向内容解析器查询该 URI 的具体类型；查不到时才退回 `text/uri-list`。而 `text/uri-list` **不用于 `content://` URI**——它是给纯文本 URI 列表用的。把两者混为一谈，会导致接收方用错误的解析路径处理数据。

### 2.3 进程与权限边界

| 进程 | 身份 | 能读什么 | 能写什么 |
|------|------|----------|----------|
| 普通应用 | 应用 uid | 仅在有资格时读取当前剪贴板 | 任何时候都可写入/覆盖 |
| 默认输入法 | 输入法进程 | 剪贴板内容 | 剪贴板内容 |
| 焦点应用 | 前台应用 | 剪贴板内容 | 剪贴板内容 |
| system_server | 系统 uid | 所有用户的剪贴板 | 剪贴板、授权、监听注册 |
| shell | shell uid | 受 shell 专属分支约束 | 剪贴板 |
| 内容提供者 | 提供者进程 | 自己的数据 | 自己的数据 |

这张表解释了一个常见困惑：**为什么有的应用在后台能读到剪贴板？** 因为它可能持有输入法身份、处于焦点、或以 shell 身份运行；这不代表 Android 10 的限制失效，而是这些身份本来就在允许列表里。

## 3. 读取资格：Android 10 之后的真实规则

Android 10（API 29）起的规则可以概括为：

- 只有**默认输入法**与**当前获得焦点的应用**可以读取剪贴板；
- AOSP 实现中还存在 shell、Content Capture、Autofill 等系统分支；
- **没有默认助理豁免**——官方文档只列默认输入法，源码中也没有「助理身份直接放行」的判断；
- **写入不受此限制**。

这条规则的目的是防止后台应用静默窃取用户复制的内容。它带来两个工程后果：

1. 依赖剪贴板的后台服务（如跨设备同步）必须改为前台服务或输入法形态，否则读不到数据；
2. 应用不能通过「先申请某个权限」来绕过资格判断——资格是按进程状态与角色判定的，不是按权限判定的。

### 3.1 首次读取提示

Android 12（API 31）起，应用首次读取其他应用写入的剪贴板时会看到系统 toast。以下情况不弹：

- 读取自己写入的内容；
- 重复读取同一份内容；
- 只读取 `ClipDescription` 而不读取条目数据。

这个提示是**可见性提示**，不是访问控制：它不阻止读取，只是让用户知情。因此它不能替代 Android 10 的资格限制，也不能阻止有资格的应用读取。

### 3.2 预览与敏感标记

Android 13（API 33）为剪贴板增加了预览浮层与编辑入口，并引入 `ClipDescription.EXTRA_IS_SENSITIVE`（字符串常量 `android.content.extra.IS_SENSITIVE`）。这个标记的作用是**让系统在预览时打码**，官方明确说明它不改变剪贴板行为、不增加安全性。把它当成「加密剪贴板」或「禁止读取」是典型误读。

Android 14（API 34）增加 `EXTRA_IS_REMOTE_DEVICE`，用于标记内容来自远程设备（如跨设备剪贴板），供系统做呈现决策。

## 4. URI 授权：剪贴板最容易被误解的部分

### 4.1 `newUri()` 不授权

`ClipData.newUri()` 只是构造一个指向 URI 的条目，**不会**给任何应用授予读取权限。真正的授权发生在粘贴时：`ClipboardService` 会以剪贴板所有者的身份，为读取方授予临时读权限，并在剪贴板被替换时撤销。

这带来两个直接结论：

- 源应用不需要在复制时显式 `grantUriPermission()`；
- 目标应用拿到 URI 后能否读取，取决于粘贴时的那次临时授权，而不是 URI 本身。

如果授权失败（例如提供者已退出、URI 已失效），剪贴板服务会清空内容并返回 `null`，而不是把无效 URI 交给目标应用。

### 4.2 `setPrimaryClip()` 会校验转授权

写入方必须证明自己有权把 URI 转授给未来的读取方。如果调用方对 URI 只有临时权限、且不允许转授，`setPrimaryClip()` 会抛 `SecurityException`。这条校验是 URI 权限模型的一部分，防止应用把「借来的权限」二次分发。

### 4.3 拖放的临时授权

拖放使用另一套标志（API 24 起）：

| 标志 | 值 | 作用 |
|------|----|------|
| `DRAG_FLAG_GLOBAL` | 256 | 允许拖出当前应用 |
| `DRAG_FLAG_GLOBAL_URI_READ` | 1 | 跨应用拖放时授予读权限 |
| `DRAG_FLAG_GLOBAL_URI_WRITE` | 2 | 跨应用拖放时授予写权限 |
| `DRAG_FLAG_GLOBAL_PERSISTABLE_URI_PERMISSION` | 64 | 允许权限跨重启保留 |
| `DRAG_FLAG_GLOBAL_SAME_APPLICATION` | API 35 新增 | 同应用内拖放的全局标志变体 |

目标应用通过 `requestDragAndDropPermissions()` 请求临时权限；`DragAndDropPermissions.release()` 或 Activity 销毁会释放。**默认是临时的**，只有显式加上可持久化标志才可能跨重启——而即便是可持久化标志，也需要目标方主动请求并持有对应授权，不能理解为「拖一次永久可读」。

### 4.4 常见授权缺陷

公开公告里，真正落在 URI 授权链上的条目包括：

- CVE-2020-27097：`UriGrantsManagerService.checkGrantUriPermission` 的权限绕过；
- CVE-2020-27098：同一方法上的访问控制问题，可触达联系人数据；
- CVE-2018-9548：`ContentProvider.java` 缺少 URI 校验；
- CVE-2021-39662：`MediaProvider.checkUriPermission` 缺少检查。

这四条经常被笼统称为「剪贴板漏洞」，但它们的落点分别是**授权服务**与**内容提供者**，与剪贴板服务本身无关。真正落在剪贴板类里的公开条目是 CVE-2023-20953（`ClipboardListener.onPrimaryClipChanged`，恢复出厂/界面绕过）与 CVE-2026-28602（`ClipboardService.setClipboardAccessNotificationsEnabledForUser`，多用户隔离），两者也都不是「内容被后台读取」这一类。

## 5. 拖放：手势背后的权限模型

拖放的安全模型与剪贴板相似但不同：

1. **起点**：源应用构造 `ClipData` 与 `DragShadowBuilder`，调用 `startDragAndDrop()`；
2. **跨应用**：只有带 `DRAG_FLAG_GLOBAL` 才能离开本应用；
3. **目标**：目标视图在 `onDragEvent()` 中判断是否接受；
4. **授权**：目标调用 `requestDragAndDropPermissions()` 获取 URI 临时权限；
5. **释放**：`release()`、Activity 销毁或拖放结束即失效。

常见的工程缺陷是把 `DRAG_FLAG_GLOBAL` 与 URI 读写标志一起无条件打开：这会让任何能接收拖放的应用都获得对 URI 的临时读写能力。正确做法是按需授予，并在目标端校验来源与数据类型。

另一个边界是**同应用内的全局标志**。API 35 新增 `DRAG_FLAG_GLOBAL_SAME_APPLICATION`，用于区分「同应用内需要全局语义」与「真正跨应用」的场景。它的出现说明平台也意识到：把「全局」当成一个布尔开关，会让权限范围比实际需要更大。

## 6. 公开公告台账

| CVE | NVD 落点 / 类型 | 级别（公告） | 修复 | 分类 |
|-----|----------------|--------------|------|------|
| [CVE-2020-27097](https://nvd.nist.gov/vuln/detail/CVE-2020-27097) | `UriGrantsManagerService.checkGrantUriPermission` 权限绕过（ID） | High | 2020-08 | Framework |
| [CVE-2020-27098](https://nvd.nist.gov/vuln/detail/CVE-2020-27098) | 同名方法，访问联系人（EoP；NVD 记 ID） | High | 2020-08 | Framework |
| [CVE-2018-9548](https://nvd.nist.gov/vuln/detail/CVE-2018-9548) | `ContentProvider.java` 缺 URI 校验（ID） | High | 2018-12 | Framework |
| [CVE-2021-39662](https://nvd.nist.gov/vuln/detail/CVE-2021-39662) | `MediaProvider.checkUriPermission` 缺检查（EoP） | High | 2022-05 | Framework |
| [CVE-2023-20953](https://nvd.nist.gov/vuln/detail/CVE-2023-20953) | `ClipboardListener.onPrimaryClipChanged`：恢复出厂/界面绕过（EoP） | High | 2023-03 | System |
| [CVE-2026-28602](https://nvd.nist.gov/vuln/detail/CVE-2026-28602) | `ClipboardService.setClipboardAccessNotificationsEnabledForUser`：多用户隔离（ID） | High | 2026-09 | Framework |

公告链接：

- 2018-12：https://source.android.com/security/bulletin/2018-12-01
- 2020-08：https://source.android.com/security/bulletin/2020-08-01
- 2022-05：https://source.android.com/security/bulletin/2022-05-01
- 2023-03：https://source.android.com/security/bulletin/2023-03-01
- 2026-09：https://source.android.com/docs/security/bulletin/2026/2026-09-01

读表要点：

1. **前四条不是剪贴板服务**，而是 URI 授权与内容提供者；把它们归入「剪贴板漏洞」会误导修复范围。
2. **后两条虽在剪贴板类，但影响不是「内容泄露」**：一条是恢复出厂/界面保护绕过，一条是多用户提示开关的隔离问题。
3. **级别以公告为准**；NVD 的类型标注与公告可能不同（如 CVE-2020-27098）。

## 7. 成因模式

### 7.1 授权检查缺省

最典型的一类：方法本应校验「调用者是否有权把这个 URI 转授出去」，但实现中漏掉了某个分支（CVE-2020-27097 / 27098 / 2021-39662）。防御上，授权检查应集中在一个函数里，所有入口复用它，而不是在每个调用点各写一遍。

### 7.2 数据与元数据混淆

`ClipDescription` 与 `ClipData.Item` 是两层：只读描述不等于读到了内容，但很多权限判断把「能拿到描述」当成「能拿到数据」。CVE-2018-9548 一类问题正是 URI 校验与数据访问之间的缝隙。

### 7.3 用户边界未贯彻

剪贴板是「每用户一份」还是「全局一份」，决定了跨用户/跨资料是否可见。CVE-2026-28602 落在提示开关的按用户设置上，说明即使是「提示」这种非数据路径，也要贯彻用户隔离。

### 7.4 生命周期假设错误

拖放授权默认随 Activity 销毁释放；剪贴板 URI 授权随内容替换撤销。如果代码假定「授权会一直有效」，在替换/销毁后就会拿到 `SecurityException` 或空数据，进而走进错误分支。

### 7.5 呈现层被当成安全层

`EXTRA_IS_SENSITIVE` 只影响预览打码，首次读取 toast 只影响知情。把呈现提示当成访问控制，是安全设计中最常见的层次错位。

### 7.6 全局标志过度使用

拖放的 `DRAG_FLAG_GLOBAL` 与 URI 读写标志组合，会把权限面扩大到所有能接收拖放的应用。默认最小化、按需开启，是这一层的基本纪律。

## 8. 版本加固时间线

| 版本 / API | 变化 |
|------------|------|
| 10 / 29 | 仅默认输入法与焦点应用可读剪贴板 |
| 11 / 30 | 分享 `content://` 必须显式携带读/写 URI 权限标志 |
| 12 / 31 | 首次读取他人剪贴板弹 toast |
| 13 / 33 | 剪贴板预览与编辑入口；`EXTRA_IS_SENSITIVE` |
| 14 / 34 | `EXTRA_IS_REMOTE_DEVICE`；无剪贴板行为变更条目 |
| 15 / 35 | `DRAG_FLAG_GLOBAL_SAME_APPLICATION`；无剪贴板行为变更条目 |

这张表反映两个方向：**读取侧越来越可见**（提示、预览、敏感标记），**授权侧越来越显式**（Android 11 要求分享 URI 时显式带标志）。对开发者而言，「默认能读/默认能传」的时代已经结束。

## 9. 观测点与自查

### 9.1 应用侧

- 读取剪贴板前是否判断自身是否处于前台/是否为默认输入法？
- 复制 `content://` 时是否确认自己有权转授？是否处理了 `SecurityException`？
- 粘贴 URI 时是否处理「授权失败返回 null」的分支？
- 拖放是否只授予必要的读写标志？是否在结束后 `release()`？
- 敏感内容是否设置了 `EXTRA_IS_SENSITIVE`，并理解它只影响预览？

### 9.2 平台侧

- `dumpsys clipboard` 查看当前剪贴板所有者、条目类型与授权记录；
- 关注 `UriGrantsManagerService` 的授权日志，尤其是跨用户/跨资料的授予；
- 检查输入法进程的剪贴板访问频率，异常高频可能是窃取行为；
- 对工作资料场景，验证剪贴板是否按资料隔离。

### 9.3 评估侧

- 看到「剪贴板」公告，先确认落点：剪贴板服务、URI 授权服务、内容提供者还是媒体库；
- 区分「读取内容」「权限绕过」「用户隔离」「界面保护」四类影响；
- 不要因为标题里有 `Clipboard` 就假定是内容泄露。

## 10. 防御要点

1. **复制敏感内容时加标记**：`EXTRA_IS_SENSITIVE` 至少能让预览打码；但不要把它当加密。
2. **粘贴前校验来源与类型**：不要无条件信任剪贴板里的 URI、Intent 或 HTML。
3. **URI 授权按最小范围**：只授予读或只授予写，避免 `GLOBAL` + 读写全开。
4. **处理授权失败**：把 `null` 与 `SecurityException` 当成正常分支，而不是崩溃路径。
5. **输入法之外的读取要谨慎**：后台同步类功能改走前台服务或专用通道，不要依赖剪贴板。
6. **跨资料默认隔离**：工作资料与个人资料的剪贴板共享应由策略显式开启，而不是默认放开。
7. **监控异常访问**：把剪贴板读取次数、URI 授权次数纳入行为基线，异常升高时告警。

## 11. 常见误传纠正

### 误传一：Android 10 起后台一律不能读剪贴板

不准确。默认输入法与焦点应用可以读，AOSP 还有 shell、Content Capture、Autofill 等分支；限制的是「无资格的后台应用」，不是「所有后台进程」。

### 误传二：默认助理也有豁免

错。官方文档只列默认输入法，AOSP 源码中也没有「助理身份直接放行」的判断。把助理当成第二豁免通道，是没有依据的。

### 误传三：`EXTRA_IS_SENSITIVE` 会强制屏蔽内容

错。官方明确说明它不改变剪贴板行为、不增加安全性，只影响预览时的打码呈现。

### 误传四：拖放 URI 授权是永久的

错。默认是临时授权，`release()` 或 Activity 销毁即释放；只有显式使用可持久化标志才可能跨重启，且仍需目标方主动请求。

### 误传五：`newUri()` 会自动授予永久权限

错。`newUri()` 不授权；粘贴时由剪贴板服务临时授予读权限，内容被替换即撤销。

### 误传六：剪贴板相关 CVE 都是内容泄露

错。CVE-2023-20953 是恢复出厂/界面保护绕过，CVE-2026-28602 是多用户提示开关隔离；前四条则落在 URI 授权与内容提供者，而非剪贴板服务。

## 12. 未核实与失效来源

- 拖放相关 CVE 在 NVD 检索 `drag` 未命中，但这可能是分词与命名差异导致，**不能据此断言不存在**。
- Android 14/15 的行为变更页没有剪贴板/拖放小节；是否另有公告级变更未核实。
- 「默认助理豁免」没有官方或 AOSP 依据；本文按「不存在」处理，但不排除厂商定制。
- NVD 详情页对脚本访问返回 403，本文的 NVD 描述以可访问页面与公告交叉核对为准。

## 13. 参考资料

- 复制粘贴（官方指南）：
  https://developer.android.com/develop/ui/views/touch-and-input/copy-paste
- ClipDescription（官方参考）：
  https://developer.android.com/reference/android/content/ClipDescription
- ClipboardManager（官方参考）：
  https://developer.android.com/reference/android/content/ClipboardManager
- View 与拖放标志（官方参考）：
  https://developer.android.com/reference/android/view/View
- 多窗口拖放（官方指南）：
  https://developer.android.com/develop/ui/views/touch-and-input/drag-drop/multi-window
- Android 10 隐私变更（剪贴板）：
  https://developer.android.com/about/versions/10/privacy/changes#clipboard-data
- Android 11 行为变更（分享 URI）：
  https://developer.android.com/about/versions/11/behavior-changes-all#share-content-uris
- Android 12 行为变更（剪贴板提示）：
  https://developer.android.com/about/versions/12/behavior-changes-all#clipboard-access-notifications
- Android 13 行为变更（敏感内容）：
  https://developer.android.com/about/versions/13/behavior-changes-all#copy-sensitive-content
- ClipboardService（AOSP）：
  https://android.googlesource.com/platform/frameworks/base/+/refs/heads/main/services/core/java/com/android/server/clipboard/ClipboardService.java
- IClipboard.aidl（AOSP）：
  https://android.googlesource.com/platform/frameworks/base/+/refs/heads/main/core/java/android/content/IClipboard.aidl
- ClipData（AOSP）：
  https://android.googlesource.com/platform/frameworks/base/+/refs/heads/main/core/java/android/content/ClipData.java
- ClipboardOverlayController（AOSP）：
  https://android.googlesource.com/platform/frameworks/base/+/refs/heads/main/packages/SystemUI/src/com/android/systemui/clipboardoverlay/ClipboardOverlayController.java
- NVD 条目（本文台账）：
  https://nvd.nist.gov/vuln/detail/CVE-2020-27097
  https://nvd.nist.gov/vuln/detail/CVE-2020-27098
  https://nvd.nist.gov/vuln/detail/CVE-2018-9548
  https://nvd.nist.gov/vuln/detail/CVE-2021-39662
  https://nvd.nist.gov/vuln/detail/CVE-2023-20953
  https://nvd.nist.gov/vuln/detail/CVE-2026-28602

---

## 联系方式

咨询 Android 系统详情请联系 https://t.me/one00190

---

*本手册仅供安全学习与防御研究使用；所有机制表述、公告列值与 CVE 描述均取自公开一手来源并附链接，未能核实的部分已单列于「未核实与失效来源」而非写入正文。*
