---
layout: page
title: "Android 无障碍、输入法与自动填充通道手册"
description: "把「辅助功能能看到别人应用里的内容」拆成三条授权方式不同的通道：`BIND_ACCESSIBILITY_SERVICE`／`BIND_INPUT_METHOD`／`BIND_AUTOFILL_SERVICE` 都是 `signature` 级，约束的是「谁能绑定」而非「能看多少」；输入法经 `Input..."
date: 2026-11-02
permalink: /android-accessibility-ime-autofill/
---

> 这三样功能有一个共同点：它们**本来就要看到别的应用里的内容**。无障碍服务读的是别人界面的节点树，输入法拿的是用户正在敲的每一个字符，自动填充服务看到的是当前界面的全部字段。这三件事都不是越权，而是产品定义——没有它们，读屏、输入、免手填就都不成立。真正值得分析的因此不是「它们为什么能看到」，而是三件事：**授权是怎么给的**（一次点击，长期有效）、**系统用什么手段把「看到」收窄**（受限设置、敏感视图标志、可选的认证）、**这条设计边界在历史上从哪里漏过**（公告里十一条已修复缺陷，全部落在系统侧的校验、状态计算与默认服务选择 UI 上，没有一条是「功能设计错了」）。本手册的主线就是把「按设计可见」「按配置可见」「因缺陷可见」三件事分开；把前两者当成漏洞，或把第三者说成设计缺陷，都会得出错误的加固结论。

---

## 0. 使用方式与边界

**本手册的证据基础。** API 与常量取自 `developer.android.com` 的参考页与指南（一律抓英文原文）；权限的保护级别、组件声明形态取自 `frameworks/base` 的 `core/res/AndroidManifest.xml` 与各 `*Service.java`；修复细节取自公告表所指向的 AOSP 提交（提交页可读到改了哪个文件、提交摘要）；公告数据一律**逐格取证**——只引用表里实际出现的 `CVE` / `References` / `Type` / `Severity` / `Updated AOSP versions` 五列原文，表里没有的组件名，明确标注来源是修复提交的文件路径而不是公告。Google Play 的政策表述取自 `support.google.com` 的政策页原文。

**本手册刻意不做的事。**

- 不给任何绕过用户授权的手段：不写如何静默启用无障碍服务、如何伪造默认输入法或默认自动填充服务、如何在受限设置下打开开关。这些是攻击步骤，不是分析。
- 不给节点树遍历、手势注入、字段抓取的代码。相关能力的「存在性」用 API 名称与版本说明，不用可运行片段。
- 不把 Play 政策当运行时强制。政策约束的是「能不能上架、会不会被下架」，系统侧的执行是另一套东西（§2.5、§7）。
- 不把恶意软件滥用当成系统漏洞。前者没有 CVE，也不在本手册的案例表里；两者的处置路径完全不同（§7）。
- 不重复覆盖层的完整攻击面。触摸事件如何被判定为「被遮挡」、Android 12 的不可信触摸阻断细节，归 [输入子系统与事件注入攻击面手册](/android-input-event-injection/)，本手册只讲它与三条通道的交叉（§5）。

**与相邻手册的分工**：见 §10。

---

## 1. 三条通道的一句话对照

先把三条通道放在同一张表里，差别一眼可见。注意「绑定权限」这一列约束的对象：它约束的是**谁能绑定这个服务**，不是**这个服务能看到什么**。

| 维度 | 无障碍服务 | 输入法 | 自动填充 |
| --- | --- | --- | --- |
| 绑定权限 | `android.permission.BIND_ACCESSIBILITY_SERVICE` | `android.permission.BIND_INPUT_METHOD` | `android.permission.BIND_AUTOFILL_SERVICE` |
| 保护级别 | `signature` | `signature` | `signature` |
| 服务声明的 action | `android.accessibilityservice.AccessibilityService` | `android.view.InputMethod` | `android.service.autofill.AutofillService` |
| meta-data 名 | `android.accessibilityservice` | `android.view.im` | `android.autofill` |
| 谁授权 | 用户在「设置 → 无障碍」里启用 | 用户选择并启用输入法 | 用户选定默认自动填充服务 |
| 能看到什么 | 节点树：文本、内容描述、类型、屏幕坐标、是否可编辑、是否密码 | `InputConnection` 送来的文本与 `EditorInfo` 上下文；全屏提取视图里显示编辑器文本 | 前台应用所有窗口的 `AssistStructure` |
| 生效时长 | 启用后长期，除非用户关闭 | 作为当前输入法期间 | 作为默认服务期间 |
| 典型 API 基线 | `AccessibilityNodeInfo` API 14 | `InputMethodService` / `InputConnection` API 3 | `AutofillManager` API 26 |
| 官方专门收紧 | targetSdk ≥ 23；Android 13 受限设置；Android 14 `accessibilityDataSensitive` | 无「必须忽略密码字段」的强制规则 | 无「按敏感度过滤字段」的强制规则 |

**这张表最重要的一行是「谁授权」。** 三条通道用的都是 `signature` 级权限，这意味着普通应用无法把别人的服务绑过来当自己的用——但这**完全不意味着**普通应用无法「成为」这样一条通道。任何开发者都能写一个无障碍服务、一个输入法、一个自动填充服务，只要用户在设置里把它选上，它在设计范围内看到的内容与系统自带的实现没有任何区别。权限在这里回答的是「调用方够不够格」，而不是「被调用方能看多少」。

由此得到本手册的第一个判断：

> 三条通道的风险不来自「权限被绕过」，而来自「授权被获取之后，缺少细粒度的、可持续的约束」。

后面的章节按通道依次展开：先讲它被设计成能看什么（这是无法加固的部分），再讲系统加了哪些收窄手段（这是可以加固的部分），最后讲历史上漏在哪里（这是公告里已经修掉的部分）。

---

## 2. 无障碍服务：一次授权，长期读屏

### 2.1 声明四件套与签名级权限

一个无障碍服务的声明由四样东西构成，缺一样系统就不会把它当成无障碍服务：

```xml
<service
    android:name=".MyAccessibilityService"
    android:permission="android.permission.BIND_ACCESSIBILITY_SERVICE"
    android:exported="true">
    <intent-filter>
        <action android:name="android.accessibilityservice.AccessibilityService" />
    </intent-filter>
    <meta-data
        android:name="android.accessibilityservice"
        android:resource="@xml/accessibility_service_config" />
</service>
```

四条各自的角色需要分清：

- `android:permission` 写在 **service** 上，是「谁能绑定我」的约束。`BIND_ACCESSIBILITY_SERVICE` 在 AOSP 的 `core/res/AndroidManifest.xml` 里是 `signature` 级——只有与系统同签名的调用方能持有。也就是说，只有 `system_server` 能绑定这个服务，普通第三方应用绑不上。
- `<intent-filter>` 里的 action 是**身份标识**：系统按 `android.accessibilityservice.AccessibilityService` 去找服务，名字不对就不认。
- `<meta-data android:name="android.accessibilityservice">` 指向服务的配置文件（XML 资源），能拿到什么能力由这个文件决定（§2.2）。
- `android:exported="true"`：服务必须可被系统跨进程绑定。targetSdk 31 起，带 intent-filter 的组件必须显式声明 `android:exported`，这是 Android 12 的通用要求，不是无障碍专属。

这里有一个在中文资料里反复出现的错误：把 `BIND_ACCESSIBILITY_SERVICE` 写成 `signature|privileged`。**AOSP 的现行定义是 `signature`**。这个错误的实际危害不大（两种写法在「只有系统能绑」这一点上同效），但它会让人误以为「特权应用也能绑」，进而在分析厂商定制时得出错误结论。

### 2.2 能力位：能拿到什么由 XML 声明决定

无障碍服务不是一个「开了就全开」的开关，它的能力由配置里的属性逐项声明。与「读内容」直接相关的几个：

| XML 属性 | 对应常量 | API | 作用 |
| --- | --- | --- | --- |
| `canRetrieveWindowContent` | `CAPABILITY_CAN_RETRIEVE_WINDOW_CONTENT` | 14 | 读取窗口内容的前提；不开则拿不到节点树 |
| `flagRetrieveInteractiveWindows` | `FLAG_RETRIEVE_INTERACTIVE_WINDOWS` | 21 | 取得所有**可交互窗口**而不只是当前活动窗口；以 `canRetrieveWindowContent` 为前提 |
| `canRequestFilterKeyEvents` | `CAPABILITY_CAN_REQUEST_FILTER_KEY_EVENTS` | 18 | 以按键过滤器形式接收按键事件 |
| `flagRequestFilterKeyEvents` | `FLAG_REQUEST_FILTER_KEY_EVENTS` | 18 | 请求开启上述过滤；需 `canRequestFilterKeyEvents` |
| `canPerformGestures` | `CAPABILITY_CAN_PERFORM_GESTURES` | 24 | 允许 `dispatchGesture()` 注入手势 |

这张表要读出两件事。

第一，**「读窗口内容」是一个显式的、可审计的声明**。用户启用服务时，系统设置页会把该服务声明的能力列出来；一个只声明了「响应无障碍事件」的服务，理论上拿不到节点树。这是 Android 在这条通道上最早的分级设计。

第二，**`flagRetrieveInteractiveWindows` 把范围从「当前窗口」扩大到「所有可交互窗口」**。这是读屏类应用需要的（悬浮窗、对话框、分屏、多窗口），但它同时意味着服务能看到不只一个界面的内容。API 21 引入这个标志时，官方把它挂在 `canRetrieveWindowContent` 之下作为**前提条件**，而不是独立的权限——也就是说，一个已经获得「读窗口内容」的服务，再声明这个标志时，用户界面上不会有第二次、更重的确认。

### 2.3 节点树与动作：读、点、滑

`AccessibilityNodeInfo`（API 14）提供的读取面包括 `getText()`、`getContentDescription()`、`getBoundsInScreen()`、`isEditable()`、`isPassword()`。动作面包括：

- `performAction(ACTION_CLICK)`（API 16）——在节点上触发点击，不需要真实触摸；
- `performAction(ACTION_SET_TEXT)`（API 21）——直接改写可编辑节点的文本；
- `performGlobalAction(int)`（API 16）——返回、主页、最近任务、通知栏等全局动作；
- `dispatchGesture()`（API 24）——按路径注入手势，需 `canPerformGestures`。

这里必须强调一个常被混淆的点：**节点树不是截图**。它传递的是 View 层自己上报的语义信息（文本、内容描述、控件类型、在屏幕上的矩形），与像素无关、与渲染无关。因此「给界面加 `FLAG_SECURE` 就安全了」对无障碍读屏不成立——`FLAG_SECURE` 的官方语义是「窗口内容不出现在截图中、也不出现在非安全显示器上」，它管的是像素外流，不管 View 层语义外流。Android 直到 14 才补上专门针对这条通道的标志（§2.5）。

### 2.4 密码字段只是一个标志

`AccessibilityNodeInfo.isPassword()` 返回的是**被读方自己上报的一个布尔值**：输入框声明了密码类型的 `inputType`，节点就带这个标志。它不是一个「框架会自动隐去文本」的开关。

这一点在中文资料里被误传得很广：「`isPassword` 为真时无障碍服务读不到密码」。本轮对官方 API 参考、无障碍指南与 Android 14 行为变更页的检索，**没有找到任何「框架在 `isPassword` 为真时隐去文本」的成文承诺**。因此正确的表述是：

> `isPassword` 是给读取方看的**提示**，让合规的实现主动跳过；它不是系统侧的强制脱敏。

真正带强制语义的是 Android 14 的 `accessibilityDataSensitive`（§2.5），而它保护的也不是「密码字段」这一类，而是**应用主动声明为无障碍敏感的视图子树**。两者的差别很关键：前者靠自觉，后者靠声明。

### 2.5 版本收紧的四个节点

Android 在这条通道上做过四次收紧，方向都是「把入口变窄」，而不是「把已启用服务的视野变窄」：

**（1）targetSdk 门槛。** 官方无障碍指南要求服务的 `targetSdkVersion` 不低于 23；低于该值的旧服务在新系统上不再作为可启用的候选。这条门槛的意义是让服务必须适配运行时权限模型——它挡的是「多年不更新的老实现」，不是恶意实现。

**（2）Android 13（API 33）的受限设置。** 从浏览器、文件管理器等本地来源安装的应用，其无障碍服务默认不能直接启用；用户需要进入「设置 → 应用 → 该应用 → 右上角菜单 → 允许受限设置」才能打开开关。这是 Android 13 引入的一套**针对侧载应用**的统一限制，无障碍只是被覆盖的能力之一（同一套限制也覆盖通知监听等）。它的效果是把「点一下安装包就能拿到读屏权限」变成「必须再走一次带解释的设置流程」。

**（3）Android 14（API 34）的 `View.setAccessibilityDataSensitive()`。** 应用可以把某棵视图子树标记为「无障碍敏感」，此后只有**声明自己是无障碍工具**的服务才能读到它。这里的关键是配套的 `AccessibilityServiceInfo.isAccessibilityTool()`（API 31）：服务在配置里把自己标记为 `isAccessibilityTool=true`，系统据此区分「读屏工具」与「其他目的的服务」。Google 在 Android Developers Blog 里的表述是：没有显式声明自己是合法无障碍工具（`isAccessibilityTool=true`）的应用，将被拒绝访问被标记为敏感的视图。换句话说，这是一条**由被读方声明、由读取方自我归类**的双边约定——它能挡住「顺手拿了无障碍权限的普通应用」，挡不住「把自己标成工具的服务」。

**（4）Google Play 的政策声明（2021-11-03 起）。** 政策页原文：

> "Starting November 3, 2021, apps that target API level 31 and include an AccessibilityService will need to complete a new policy declaration in Play Console."

> — <https://support.google.com/googleplay/android-developer/answer/10964491>

这条政策的作用是让「为什么需要无障碍」变成上架时必须回答的问题。它是**商店侧**的约束：不上架、或不上 Play 的渠道，这条不适用；已经装上的应用也不会因为政策变化而被系统收回权限。把它当成运行时防护是常见的误读（§9 第 12 条）。

四次收紧放在一起看，结论很清楚：**入口越来越难拿，但拿到之后的视野没有变窄**。一个被用户主动启用、声明了 `canRetrieveWindowContent` 与 `flagRetrieveInteractiveWindows`、且把自己标成 `isAccessibilityTool=true` 的服务，在 Android 14 上依然能读到绝大多数界面内容。真正的收窄手段只有一个：应用主动把子树标记为 `accessibilityDataSensitive`。

### 2.6 公告里的两个无障碍缺陷

无障碍子系统本身也出过系统侧缺陷，两条都在 Framework 表里，修的都是**服务对象的管理逻辑**，不是「读得太多」：

**CVE-2023-21109**（2023-05 公告，Framework 表）：`EoP` / `High` / 影响 `11, 12, 12L, 13`。对应的修复提交改了三个文件——`core/java/android/accessibilityservice/AccessibilityService.java`、`core/java/android/accessibilityservice/AccessibilityServiceInfo.java`、`services/accessibility/java/com/android/server/accessibility/AccessibilityManagerService.java`，提交摘要是「Checks if AccessibilityServiceInfo is within parcelable size」，提交说明里写着：解析服务 XML 时如果对象过大就跳过该服务，服务自我更新时如果过大则报错。根因模式是**跨进程对象的体积未校验**：`AccessibilityServiceInfo` 要经 Binder 在系统与应用之间传递，而解析侧对它的规模没有上界。

**CVE-2025-26462**（2025-06 公告，Framework 表）：`EoP` / `High` / 影响 `13, 14, 15`。修复提交只改一个文件：`services/accessibility/java/com/android/server/accessibility/AccessibilityServiceConnection.java`，摘要是「fix: Unbind an A11yService from onNullBinding」，提交说明写明：`onBind()` 返回 null 的服务不再继续被 `system_server` 持有绑定。根因模式是**生命周期状态未收敛**——绑定失败的路径没有把连接对象清理掉，留下一个「名义上绑着、实际上没有服务」的连接。

两条都不是「无障碍服务读到了不该读的东西」，而是「管理无障碍服务的那层代码自己出了错」。这个区分很重要：它说明这条通道的**系统侧**攻击面主要落在对象校验与生命周期上，而不是权限判定上——权限判定本身（signature + 用户启用）在这十几年里没有出过体系性问题。

---

## 3. 输入法：功能上必须看见明文

### 3.1 绑定链条

输入法的声明形态与无障碍服务同构：

```xml
<service
    android:name=".MyInputMethod"
    android:permission="android.permission.BIND_INPUT_METHOD"
    android:exported="true">
    <intent-filter>
        <action android:name="android.view.InputMethod" />
    </intent-filter>
    <meta-data android:name="android.view.im" android:resource="@xml/method" />
</service>
```

`BIND_INPUT_METHOD` 同样是 `signature` 级（不是 `signature|privileged`），action 常量与 meta-data 名定义在 `android.view.inputmethod.InputMethod` 里（`SERVICE_INTERFACE` 与 `SERVICE_META_DATA`）。同样地：**任何应用都能实现一个输入法**，用户选中之后，它就是合法的输入法，系统不会因为它「不是系统自带」而少给它一点信息。

### 3.2 输入法不是「猜」到文本，是框架交给它

这是本手册最需要钉死的一条事实。输入法与编辑器之间的数据通路是 `InputConnection` 与 `EditorInfo`：

- `EditorInfo`（API 3）携带 `inputType`、`imeOptions`、`packageName`、`fieldId`、`hintText`——也就是说，输入法**知道当前字段属于哪个应用、是什么类型、有没有提示文本**；
- `InputConnection.commitText()`、`sendKeyEvent()`、`getExtractedText()`（均 API 3）让输入法提交文本、发送按键、读取光标周围的上下文；
- `InputMethodService` 的 `onStartInputView()`（API 3）在输入会话开始时被调用，此时 `EditorInfo` 已经就绪。

换句话说：**用户正在输入的内容，是框架主动递到输入法进程里的**。这不是漏洞，是输入法的功能前提——输入法要做联想、纠错、候选，就必须看到已经输入的内容。

由此可以推出一条看起来很绝对、但确实成立的结论：

> 只要某个输入法是用户当前的输入法，它就能看到用户在该输入法里敲的明文。所谓「防输入法记录密码」，在这个前提下没有技术手段可以保证；能做的只有换一个可信的输入法，或者不让该字段获得焦点。

这条结论不是危言耸听，而是把「功能边界」讲清楚。后面两节讲的两件事——全屏提取视图与密码字段的官方措辞——都是在同一个前提下讨论「系统还做了哪些收窄」。

### 3.3 全屏提取视图：文本出现在输入法自己的窗口里

`InputMethodService` 有两个与「全屏输入」相关的 API，都是 API 3：`setExtractViewShown()` 与 `isFullscreenMode()`。官方文档把全屏输入法窗口描述为「intended to cover the entire application display」，并说明提取视图（extract view）会在输入法自己的窗口里显示编辑器的文本，同时「allow some of the application to be seen behind」。

这句话的工程含义比字面更重：

- 横屏、小屏或某些控件里，编辑器会请求「提取模式」，此时**文本被渲染进输入法进程的视图层级**，而不是留在应用窗口里；
- 提取视图覆盖整个显示区域，应用自己的界面退到后面；
- 因此在这段时间里，输入法进程不仅「通过 `InputConnection` 知道文本」，还「在自己的界面上画出文本」。

这放大了「输入法进程被攻破」的后果半径：从「流经的文本可能被记录」变成「屏幕上正在显示的内容由输入法进程负责绘制」。但它**没有改变授权模型**——文本本来就已经交给输入法了。

顺带纠正一个常被写错的 API：`InputMethodService.onExtractViewShown` **不存在**。与提取模式相关的回调是 `onUpdateExtractingVisibility()`、`onUpdateExtractingViews()`；查询状态用 `isExtractViewShown()`。中文资料里把它写成 `onExtractViewShown` 的很多，按名字去写代码会直接编译失败。

### 3.4 密码字段：只有 `InputType` 变体与一句 Caution

输入法判断「这是不是密码框」靠的是 `EditorInfo.inputType` 里的变体位：

- `InputType.TYPE_TEXT_VARIATION_PASSWORD`（API 3）
- `InputType.TYPE_NUMBER_VARIATION_PASSWORD`（API 11）

注意这两个是 **`InputType` 的常量**，不是 `EditorInfo` 的标志位。写成 `EditorInfo.TYPE_TEXT_VARIATION_PASSWORD` 是错的。

官方对输入法处理密码的表述，是一句 Caution 而不是一条强制规则：

> "Hide the password in your UI, both in the input view and in the candidates view. Don't store passwords on a device."

> — <https://developer.android.com/develop/ui/views/touch-and-input/creating-input-method>

这句话的对象是输入法**作者**，语气是建议。系统没有一条机制能在运行时检查「这个输入法有没有把密码存下来」。相比之下，`IME_FLAG_NO_PERSONALIZED_LEARNING`（API 26）看起来像一条防线，但文档自己就写明了它不是保证——它只是请求输入法不要用这个字段的内容做个性化学习，而「some IMEs may not respect it」。

所以在这条通道上，密码字段的防护实质上是：**框架告诉输入法这是密码，然后信任输入法照做**。这是一个诚实的描述，也是一条应该在评估报告里写清楚的事实。

### 3.5 唯一一条「设计上就不让输入法看见」的路径

Android 11 引入了内联自动填充（inline autofill）。官方功能页的表述是：建议由框架直接呈现在输入法的候选区域附近，而**在用户选中某条建议之前，输入法并不知道建议的内容**。

这是三条通道里唯一一条「从设计上就把输入法排除在数据流之外」的机制。它的边界同样要写清楚：它只覆盖「自动填充提供建议」这一个场景；用户手输的内容依然经过 `InputConnection`；它也不改变输入法能看到当前字段 `EditorInfo` 的事实（输入法仍知道「这里是一个地址字段」）。

### 3.6 公告里的四个输入法缺陷

输入法子系统的公开缺陷集中在两处：系统服务 `InputMethodManagerService` 的校验与状态，以及输入法元数据的资源管理。

| CVE | 公告 | 类型 / 级别 | 影响版本 | 修复落点（据提交文件路径） |
| --- | --- | --- | --- | --- |
| CVE-2022-20394 | 2022-10（System 表） | `ID` / `High` | 10, 11, 12, 12L | `InputMethodManagerService.getInputMethodWindowVisibleHeight`，缺权限检查 |
| CVE-2023-21192 | 2023-06 Pixel | `EoP` / `Moderate` | 13 | `InputMethodManagerService.setInputMethodWithSubtypeIdLocked`，输入校验 |
| CVE-2025-48603 | 2025-12（Framework 表） | `DoS` / `High` | 13, 14, 15, 16 | `InputMethodInfo.java`，资源耗尽 |
| CVE-2023-21336 | Android 14 发布公告（Framework 表） | `ID` / `Moderate` | — | 输入法相关；NVD 记为 CWE-203 |

**CVE-2022-20394 值得单独讲。** 它泄漏的是「输入法窗口的可见高度」。这个值本身听起来无害，但它是一个**可观测的侧信道**：键盘是否弹出、弹出多高，能反映用户是否正在输入、输入界面处于什么状态。缺权限检查把它变成任何应用都能问的值。这类缺陷的教训是：在输入法这条通道上，**元数据（窗口可见性、当前输入法、子类型）本身就是信息**，不能因为「不是文本」就当作低敏感。

**CVE-2023-21336** 在 Android 14 的发布公告里只有 `ID` / `Moderate` 两格，被归在 Framework 表；NVD 把它记为 CWE-203（通过可观察差异导致的信息暴露）。公告没有给出「哪一段代码」的说明，本手册因此不推测它的具体机制，只把它作为「输入法通道存在侧信道类缺陷」的一个编号证据。

**CVE-2025-48603** 是 `DoS`：`InputMethodInfo` 相关的资源耗尽。它与 §2.6 的 CVE-2023-21109 形态相似——都是**元数据对象**的问题，一个管体积上界，一个管资源消耗。这两条放在一起说明：输入法/无障碍这类「配置驱动」的子系统，其配置对象本身就是一个需要设上界的输入面。

---

## 4. 自动填充：前台应用的整棵视图树

### 4.1 绑定与 API 基线

自动填充服务的声明与上两条同构：

```xml
<service
    android:name=".MyAutofillService"
    android:permission="android.permission.BIND_AUTOFILL_SERVICE"
    android:exported="true">
    <intent-filter>
        <action android:name="android.service.autofill.AutofillService" />
    </intent-filter>
    <meta-data android:name="android.autofill" android:resource="@xml/autofill_service" />
</service>
```

`BIND_AUTOFILL_SERVICE` 为 `signature` 级。API 基线：`AutofillManager` 与 `AutofillService.onFillRequest()` / `onSaveRequest()` 均为 API 26；`AssistStructure` / `AssistStructure.ViewNode` 是 API 23（它最初为 assistant 设计，autofill 复用了这套结构）；`AutofillId` 与 `AutofillManager.requestAutofill()` 为 API 26。

### 4.2 它看到的是什么：不是「一个字段」，是整屏

自动填充服务的输入不是「用户点中的那个输入框」，而是 `FillRequest` 里附带的一整棵 `AssistStructure`。官方 javadoc 的措辞是「A ViewStructure representing **all views in the screen** is created」；`AutofillManager` 的说明是从「the root views of **all application windows**」派发。

这两句话需要限定，否则会被读成「能看到设备上所有应用」：

- 它是**当前前台应用**的窗口树——包括该应用弹出的对话框、悬浮层等属于同一应用的窗口；
- 它**不是**其他后台应用的窗口，也不是系统 UI 的窗口；
- 它是「屏幕上的视图结构」，因此分屏、多窗口下可能出现多个应用的窗口，但这仍然是「当前可见」的范围。

限定之后，结论依然很强：**一个自动填充服务拿到的，是当前界面的全部字段结构，而不是只与地址/信用卡相关的字段**。它是否只挑自己需要的字段、是否丢弃其余内容，完全靠服务自己遵守。框架在这一层没有「按字段敏感度过滤」的机制。

### 4.3 数据集认证是可选项，不是强制

`FillResponse` 里可以放多个 `Dataset`，每个 `Dataset.Builder` 可以调 `setAuthentication()` 挂一个认证流程（`IntentSender` 或 `BiometricPrompt` 的认证回调）。设了认证之后，数据集在界面上是**锁定态**。官方 javadoc 对此的说明是：

> "Since the data set is 'locked' until the user authenticates it, typically this data set name is masked."

> — <https://developer.android.com/reference/android/service/autofill/Dataset.Builder>

关键在于「可选」两个字：框架**不会**因为某个字段「看起来敏感」而强制要求认证。它不判断字段敏感度——判断权在服务手里。也就是说，「自动填充服务在填密码前会先要求指纹」这件事，是**该服务自己的产品决定**，不是平台的保证。评估一个自动填充服务时，这一条必须单独确认。

### 4.4 `importantForAutofill` 的五个取值

被填充方（也就是普通应用）控制「这个视图要不要参与自动填充」的手段是 `android:importantForAutofill`（API 26）与 `View.setImportantForAutofill()`。取值有五个：

| 取值 | 语义 |
| --- | --- |
| `auto` | 由系统判断（默认） |
| `yes` | 参与自动填充 |
| `no` | 本视图不参与，**但其子视图仍可参与** |
| `yesExcludeDescendants` | 本视图参与，后代全部排除 |
| `noExcludeDescendants` | 本视图与后代全部排除 |

**没有 `excludeDescendants` 这个取值。** 这是一个高频错误：想排除一整棵子树时写成 `excludeDescendants`，构建期就会失败；想用 `no` 达到同样效果，则只会排除当前视图，子视图照旧被填充。

这五个值的设计本身说明了一件事：**平台把「要不要被读」的决定权交给了被读方**。一个应用如果不想让自己的某个字段进入 `AssistStructure`，正确做法是用 `noExcludeDescendants` 把整棵子树排除；而如果它既想参与自动填充、又想在无障碍侧保持私密，那是另一个开关（`accessibilityDataSensitive`），两者不通用。

### 4.5 `FLAG_SECURE` 不是自动填充的开关

这条误传在移动安全评估里出现频率极高：「界面上加 `FLAG_SECURE`，截图、录屏、自动填充都挡掉了」。

官方对 `FLAG_SECURE` 的定义只覆盖前两项：窗口内容不出现在截图中、不出现在非安全显示器上。自动填充的数据源是 `AssistStructure`，属于另一条通道；本轮对自动填充相关官方文档的检索，**没有找到任何「`FLAG_SECURE` 会阻止自动填充读取」的承诺**。

因此正确的表述是：

> `FLAG_SECURE` 是防像素外流的开关，不是防视图语义外流的开关。用它来论证「自动填充读不到这个界面」属于对文档的过度外推。

如果确实要让某棵子树不被自动填充读取，用的是 §4.4 的 `noExcludeDescendants`；要让无障碍服务读不到，用的是 Android 14 的 `accessibilityDataSensitive`。三个开关，三个不同的作用域，混用是评估报告里最常见的错误之一。

### 4.6 公告里的三个自动填充缺陷

自动填充是三条通道里最年轻的一条（API 26），公开缺陷也最少，但三条各有代表性：

**CVE-2024-31310**（2024-06 公告，Framework 表）：`EoP` / `High` / 影响 `12, 12L, 13, 14`。修复提交落在 `services/autofill/java/com/android/server/autofill/AutofillManagerServiceImpl.java`，摘要是「Add in check for intent filter when setting/updating service」，提交说明提到用 `CtsAutoFillServiceTestCases` 验证。这条缺陷的类型是**服务资格校验不完整**：在设置或更新自动填充服务时，缺少对 intent filter 的检查，于是「什么样的服务能被设为默认」这件事出现了绕过空间。它说明了一个结构性事实——自动填充的授权入口是「设为默认」，因此**默认服务的资格判定本身就是安全边界**。

**CVE-2023-20976**（2023-06 Pixel 公告）：`EoP` / `Moderate` / 影响 `13`。修复落点是 `DefaultAutofillPicker.getConfirmationMessage`。从方法名与提交位置可以看出，问题出在**默认服务选择界面的确认文案**上：确认对话框所呈现的信息与实际将要生效的选择之间不一致，属于典型的 UI 欺骗面。这类缺陷的级别通常不高（Moderate），但它的意义很直接：三条通道的授权都是「用户点一下」，那么**那一下所依据的界面就必须是可信的**。

**CVE-2020-0031**（2020-03 公告，Framework 表）：`ID` / `High` / 影响 `10`。修复指向 `Session.triggerAugmentedAutofillLocked`，缺陷类型是「不恰当地显示敏感信息」。增强型自动填充（augmented autofill）是让服务在应用没有实现标准自动填充时也能提供建议的机制；这条缺陷说明，**建议的呈现时机与位置**同样是一个需要校验的面——在错误的界面上弹出正确的建议，等于把敏感信息泄漏给了不该看到的人。

三条合起来看，自动填充的攻击面几乎全部集中在**授权与呈现**这一侧（谁能被设为默认、确认界面说什么、建议在哪里显示），而不是「读取字段」那一侧。这与它的设计一致：读取整屏本来就是它的功能。

---

## 5. 覆盖层：三条通道共同的横向风险

覆盖层（overlay）不增加三条通道的视野，但它改变了「用户以为自己在点什么」与「系统实际把事件给了谁」之间的一致性。三条通道都依赖用户的操作（启用服务、选择输入法、点选建议），因此覆盖层与它们的交叉点都在**授权那一刻**。

### 5.1 两个标志位

- `MotionEvent.FLAG_WINDOW_IS_OBSCURED`（API 9）：事件发生的位置被其他窗口部分或全部遮挡时置位；
- `MotionEvent.FLAG_WINDOW_IS_PARTIALLY_OBSCURED`（API 29）：区分「部分遮挡」与「完全遮挡」；
- `android:filterTouchesWhenObscured` / `View.setFilterTouchesWhenObscured()`（API 9）：被遮挡时丢弃触摸，由应用自愿启用。

这三个都是**应用侧可选的**判定依据，而不是平台的默认拦截。

### 5.2 Android 12 的不可信触摸阻断

Android 12（API 31）把这件事变成了默认行为。官方 risk 页的表述是：Android 12 及更高版本默认阻止来自其他 UID 的非可信覆盖层的触摸事件，从而阻止完全遮挡类攻击；对于窗口动画等场景，只阻断不透明度达到一定阈值的覆盖层。

配套的应用侧 API 是 `Window.setHideOverlayWindows(boolean)`（API 31）：调用后，系统在该窗口之上隐藏非系统覆盖层。这里有一个命名陷阱：**`HIDE_OVERLAY_WINDOWS` 是权限名（normal 级），不是方法名**；方法叫 `setHideOverlayWindows`。与之相关的系统标志是 `SYSTEM_FLAG_HIDE_NON_SYSTEM_OVERLAY_WINDOWS`。

### 5.3 两条公告案例

**CVE-2021-0438**（2021-04 公告，Framework 表）：`EoP` / `High` / 影响 `8.1, 9, 10`。修复涉及 `InputDispatcher.cpp` 与 `WindowManagerService.java`，缺陷是 `FLAG_OBSCURED` 的计算不正确——也就是说，**「这个窗口被遮挡了」这个判定本身出过错**。当判定错误时，所有依赖它的下游逻辑（包括应用侧的 `filterTouchesWhenObscured`）都会失效。

**CVE-2024-31324**（2024-06 公告，Framework 表）：`EoP` / `High` / 影响 `12, 12L, 13, 14`。修复落在 `WindowState.hide`，缺陷是在屏幕旋转的时序下，遮挡状态没有被正确维持，从而绕过点按劫持防护。这条的价值在于提醒：**遮挡判定是状态机，不是一次性计算**；旋转、折叠、多窗口切换都会让状态在时间轴上出现窗口。

### 5.4 与三条通道的交叉

把覆盖层放回本手册的语境，只有三个交叉点值得记：

1. **启用无障碍服务**：设置页的开关如果被覆盖层遮挡，用户点到的可能不是那个开关。Android 12 的默认阻断覆盖了「完全遮挡」这一类，但覆盖层与三条通道的关系不止于此。
2. **选择输入法/默认自动填充服务**：选择器是一个系统界面，选择结果却决定长期的数据流向。CVE-2023-20976 正是这一类「选择界面本身不可信」的缺陷。
3. **点选自动填充建议**：建议浮层与输入法候选区在视觉上相邻，用户很难分辨自己点的是哪一个。

触摸事件在被遮挡时如何判定、`setHideOverlayWindows` 的完整语义、Android 12 行为变更的逐条细节，见 [输入子系统与事件注入攻击面手册](/android-input-event-injection/) 的对应章节；本手册不重复。

---

## 6. 公告台账与读法

把本手册引用的十一条缺陷放在一张表里，按公告时间排序。**所有列值都取自公告表原文**；「修复落点」一列是提交文件路径，公告表里没有这一列，属于补充信息。

| CVE | 公告 | 表 | Type | Severity | 影响版本 | 修复落点（提交路径） |
| --- | --- | --- | --- | --- | --- | --- |
| CVE-2020-0031 | 2020-03 | Framework | `ID` | High | 10 | `Session.triggerAugmentedAutofillLocked` |
| CVE-2021-0438 | 2021-04 | Framework | `EoP` | High | 8.1, 9, 10 | `InputDispatcher.cpp` / `WindowManagerService.java` |
| CVE-2022-20394 | 2022-10 | System | `ID` | High | 10, 11, 12, 12L | `InputMethodManagerService.getInputMethodWindowVisibleHeight` |
| CVE-2023-21109 | 2023-05 | Framework | `EoP` | High | 11, 12, 12L, 13 | `AccessibilityServiceInfo` / `AccessibilityManagerService` |
| CVE-2023-20976 | 2023-06 Pixel | Pixel | `EoP` | Moderate | 13 | `DefaultAutofillPicker.getConfirmationMessage` |
| CVE-2023-21192 | 2023-06 Pixel | Pixel | `EoP` | Moderate | 13 | `InputMethodManagerService.setInputMethodWithSubtypeIdLocked` |
| CVE-2023-21336 | Android 14 发布公告 | Framework | `ID` | Moderate | — | 未在公告中给出 |
| CVE-2024-31310 | 2024-06 | Framework | `EoP` | High | 12, 12L, 13, 14 | `AutofillManagerServiceImpl.java` |
| CVE-2024-31324 | 2024-06 | Framework | `EoP` | High | 12, 12L, 13, 14 | `WindowState.hide` |
| CVE-2025-26462 | 2025-06 | Framework | `EoP` | High | 13, 14, 15 | `AccessibilityServiceConnection.java` |
| CVE-2025-48603 | 2025-12 | Framework | `DoS` | High | 13, 14, 15, 16 | `InputMethodInfo.java` |

读这张表有三条方法上的提醒：

**第一，表里没有「组件」列。** 2020–2025 的这几种版式都是五列（CVE、References、Type、Severity、Updated AOSP versions），Pixel 公告的第四列是「Updated Android versions」。想定位到具体文件，必须点开 References 里的 AOSP 提交——本手册「修复落点」一列就是这么来的。把「公告里没写组件」当成「不知道修了什么」，是读公告时最常见的偷懒。

**第二，Type 是结果分类，不是根因。** `EoP` 意思是「这个缺陷可被用来提权」，`ID` 是信息泄露，`DoS` 是拒绝服务；它们描述的是**后果**。同一个 `EoP` 背后可能是体积未校验（CVE-2023-21109）、生命周期未收敛（CVE-2025-26462）、资格校验缺失（CVE-2024-31310）或遮挡状态计算错误（CVE-2024-31324）。根因要读提交。

**第三，CWE 不在公告里。** 本手册只在 CVE-2023-21336 一条上引用 NVD 的 CWE-203，因为那一行公告只有 `ID` / `Moderate` 两格、没有机制说明；其余各条不强行贴 CWE 标签。

---

## 7. 成因模式：六种

十一条缺陷不能一条一个解释，归纳下来是六种模式。这六种也适用于评估新出现的同类问题。

**（1）授权是「一次点击、长期有效」。** 三条通道都没有「按会话授权」的机制：启用之后，服务在下次用户手动关闭之前一直有效。没有会话，就没有「这次授权是为了做什么」的上下文，也就没有基于上下文的审计。这是设计选择，不是缺陷；但它决定了这类通道一旦被恶意占用，**发现成本远高于获取成本**。

**（2）功能前提就是读明文。** 输入法必须读用户输入才能联想；自动填充必须读整屏才能匹配字段；无障碍必须读节点树才能朗读。三者都无法通过「少给一点数据」来加固，只能通过「让用户知道谁在读」与「让应用声明哪些不能读」来管理。

**（3）敏感度由被读方声明、由读取方自律。** `isPassword` 是提示，`importantForAutofill` 是被读方的开关，`accessibilityDataSensitive` 是被读方的声明，`setAuthentication` 是读取方的产品决定。四者没有一个是「平台强制判断敏感度」。这条模式解释了为什么同一个「看起来敏感」的界面，在不同设备、不同服务下的实际暴露程度可以差很多。

**（4）系统侧对象校验缺失。** CVE-2023-21109（`AccessibilityServiceInfo` 体积）、CVE-2025-48603（`InputMethodInfo` 资源耗尽）、CVE-2024-31310（intent filter 校验）都是这一类：跨进程传进来的配置对象没有按「不可信输入」对待。

**（5）默认服务选择的 UI 面。** CVE-2023-20976 是纯粹的选择界面缺陷。只要授权入口是「让用户选一个默认服务」，那么这个选择界面就是安全边界的一部分，它的文案、排序、确认步骤都要按安全界面来设计。

**（6）策略与执行分离。** Play 的 2021-11-03 声明、Android 13 的受限设置、targetSdk 门槛，三者分属商店政策、系统入口限制、兼容性门槛。把它们混成一句「系统禁止了滥用」，会得出错误的结论：**政策管上架，受限设置管入口，两者都不管已经启用的服务读什么**。

---

## 8. 观测点与自查

### 8.1 用户侧

三条通道的授权状态在设置里都是可见的，这是最直接的观测点：

- **无障碍**：「设置 → 无障碍 → 已下载/已启用的服务」。这里列出所有已启用的服务及其声明的能力。Android 13 起，侧载应用的开关可能被「受限设置」挡住，需要额外一步解除。
- **输入法**：「设置 → 系统 → 语言和输入法 → 屏幕键盘 / 当前输入法」。切换输入法时会弹出系统确认。
- **自动填充**：「设置 → 系统 → 语言和输入法 → 自动填充服务」。这里显示当前默认服务，也提供「无」这个选项。

三个入口的共同点是：**只能看「谁被启用了」，看不到「它读了多少」**。没有任何系统界面会告诉用户「这个无障碍服务昨天读了多少个界面」。这是第（1）条成因模式的直接后果。

### 8.2 开发者侧

| 目的 | API | API 级别 |
| --- | --- | --- |
| 查询无障碍是否开启 / 已启用哪些服务 | `AccessibilityManager.isEnabled()`、`getEnabledAccessibilityServiceList(int)` | 4 / 14 |
| 判断某服务是否声明为无障碍工具 | `AccessibilityServiceInfo.isAccessibilityTool()` | 31 |
| 把子树标记为无障碍敏感 | `View.setAccessibilityDataSensitive(boolean)` | 34 |
| 控制某视图是否参与自动填充 | `android:importantForAutofill` / `View.setImportantForAutofill()` | 26 |
| 被遮挡时丢弃触摸 | `View.setFilterTouchesWhenObscured()` / `android:filterTouchesWhenObscured` | 9 |
| 在自己的窗口上隐藏非系统覆盖层 | `Window.setHideOverlayWindows(boolean)` | 31 |
| 请求输入法不要做个性化学习 | `EditorInfo.IME_FLAG_NO_PERSONALIZED_LEARNING` | 26 |
| 阻止编辑器进入全屏提取模式 | `EditorInfo.IME_FLAG_NO_EXTRACT_UI` | 3 |

这张表要按「作用域」读，而不是按「强度」读：`accessibilityDataSensitive` 只管无障碍，不管自动填充；`importantForAutofill` 只管自动填充，不管无障碍；`FLAG_SECURE` 两者都不管（§4.5）；`setHideOverlayWindows` 管的是覆盖层，不是读取。把任意一个当成「万能开关」，都会在实际评估中漏掉另外两条通道。

还有一条实践建议值得写进自查清单：**`AccessibilityManager.getEnabledAccessibilityServiceList()` 的返回值会暴露用户是否在使用读屏类工具**。如果应用要根据它改变行为（例如降低动画、简化布局），这是合理用法；如果根据它改变**安全相关行为**（例如「检测到无障碍就关闭某些功能」），要意识到这个判断本身是可被观察的，而且会伤及真实的无障碍用户。

### 8.3 企业侧

企业环境里，三条通道的处置方向是「收敛可用集合」：受管设备上限定可用的输入法与默认自动填充服务，并对无障碍服务做白名单。具体到 MDM/EMM 的策略常量名，本轮未逐字核对（见文末「未核实与失效来源」F 项），因此这里只写方向不写常量——写错常量名的加固清单比没有清单更危险。

---

## 9. 常见误传纠正

下面十四条是中文资料里反复出现、且都能用一手来源否定的说法。

| # | 误传 | 事实 |
| --- | --- | --- |
| 1 | `BIND_INPUT_METHOD` 的保护级别是 `signature\|privileged` | AOSP `core/res/AndroidManifest.xml` 里是 `signature` |
| 2 | `BIND_AUTOFILL_SERVICE` 的保护级别是 `signature\|privileged` | 同上，`signature` |
| 3 | `android:importantForAutofill` 可以取 `excludeDescendants` | 只有 `auto`/`yes`/`no`/`yesExcludeDescendants`/`noExcludeDescendants` 五个值 |
| 4 | `TYPE_TEXT_VARIATION_PASSWORD` 是 `EditorInfo` 的标志 | 是 `InputType` 的常量；`EditorInfo` 只提供 `inputType` 字段承载它 |
| 5 | `InputMethodService.onExtractViewShown` 是提取视图的回调 | 该 API 不存在；相关回调是 `onUpdateExtractingVisibility()` / `onUpdateExtractingViews()` |
| 6 | `InputMethodManager.isInputMethodEnabled()` 可以查询某个输入法是否启用 | 该方法不存在；用 `getEnabledInputMethodList()` / `getInputMethodList()` |
| 7 | 自动填充服务对敏感字段必须调 `setAuthentication()` | 认证是可选项，框架不判断字段敏感度，不会强制 |
| 8 | 给界面加 `FLAG_SECURE` 就能阻止自动填充读取 | 官方对 `FLAG_SECURE` 的定义只覆盖截图与非安全显示；无此承诺 |
| 9 | 自动填充服务能看到设备上所有应用的界面 | 只看到**前台应用**的窗口树（含其对话框等窗口），不是所有应用 |
| 10 | `HIDE_OVERLAY_WINDOWS` 是隐藏覆盖层的方法 | 它是 normal 级权限名；方法是 `Window.setHideOverlayWindows()` |
| 11 | `isPassword` 为真时框架会隐去文本 | 官方无此成文承诺；它只是被读方上报的提示位 |
| 12 | Play 的 2021-11-03 无障碍声明等于系统层面的限制 | 那是商店政策；系统侧的执行是受限设置（Android 13）与 `accessibilityDataSensitive`（Android 14） |
| 13 | 任意应用都能用无障碍手势注入 | 需服务声明 `canPerformGestures`（`dispatchGesture` API 24），且用户已启用该服务 |
| 14 | Android 13 起侧载应用的无障碍服务无法启用 | 准确说法是「受限设置」默认关闭，用户可在应用信息里手动解除 |

第 11 条需要补一句：说「没有成文承诺」不等于「框架一定不脱敏」。它可能在某些版本、某些实现上有额外处理，但那属于实现细节而非接口契约，不能写进加固建议。本手册的处理方式是：**正文只写「API 契约里没有这条保证」，并把可能的实现层行为放进文末「未核实与失效来源」的 A 项**。

---

## 10. 与其他手册的分工

| 主题 | 本手册的处理 | 详见 |
| --- | --- | --- |
| 触摸事件分发、事件注入、遮挡判定 | 只讲与三条通道授权的交叉（§5） | [输入子系统与事件注入攻击面手册](/android-input-event-injection/) |
| 组件导出、PendingIntent、默认服务选择器的 IPC 面 | 只讲授权入口 | [框架 IPC 与组件导出攻击面手册](/android-framework-ipc-export/) |
| `FLAG_SECURE`、屏幕捕获、MediaProjection | 只讲它与自动填充的不等价（§4.5） | [摄像头、麦克风与屏幕捕获面手册](/android-media-capture-privacy/)、[图形与显示栈攻击面手册](/android-graphics-display-stack/) |
| 进程隔离、SELinux 域、seccomp | 不涉及：三条通道都在应用域内运行 | [SELinux 策略与 seccomp 沙箱手册](/android-selinux-seccomp-sandbox/) |
| 日志与崩溃转储里的输入内容 | 不涉及：那属于留存通道 | [崩溃诊断与日志信息暴露面手册](/android-diagnostics-log-exposure/) |
| 权限模型与提权链 | 只讲三条通道的授权模型 | [远程代码执行与权限提升手册](/android-rce-and-privilege-escalation/) |

---

## 参考

**官方 API 与指南（本手册逐字引用或逐项核对过）**

- 构建输入法（含密码字段 Caution）— <https://developer.android.com/develop/ui/views/touch-and-input/creating-input-method>
- `InputMethod`（`SERVICE_INTERFACE` / `SERVICE_META_DATA`）— <https://developer.android.com/reference/android/view/inputmethod/InputMethod>
- `InputMethodService`（`setExtractViewShown` / `isFullscreenMode`）— <https://developer.android.com/reference/android/inputmethodservice/InputMethodService>
- `InputMethodManager` — <https://developer.android.com/reference/android/view/inputmethod/InputMethodManager>
- `InputConnection` — <https://developer.android.com/reference/android/view/inputmethod/InputConnection>
- `EditorInfo` — <https://developer.android.com/reference/android/view/inputmethod/EditorInfo>
- `InputType`（密码变体位）— <https://developer.android.com/reference/android/text/InputType>
- `AutofillService` — <https://developer.android.com/reference/android/service/autofill/AutofillService>
- `AutofillManager`（「all application windows」）— <https://developer.android.com/reference/android/view/autofill/AutofillManager>
- `AssistStructure`（「all views in the screen」）— <https://developer.android.com/reference/android/app/assist/AssistStructure>
- `Dataset.Builder`（`setAuthentication` 的锁定语义）— <https://developer.android.com/reference/android/service/autofill/Dataset.Builder>
- `AccessibilityServiceInfo`（能力位与 `isAccessibilityTool`）— <https://developer.android.com/reference/android/accessibilityservice/AccessibilityServiceInfo>
- `AccessibilityNodeInfo` — <https://developer.android.com/reference/android/view/accessibility/AccessibilityNodeInfo>
- `AccessibilityManager` — <https://developer.android.com/reference/android/view/accessibility/AccessibilityManager>
- `Window`（`setHideOverlayWindows`）— <https://developer.android.com/reference/android/view/Window>
- Android 11 内联自动填充 — <https://developer.android.com/about/versions/11/features>
- 点击劫持风险页（Android 12 不可信触摸、Android 14 `accessibilityDataSensitive`）— <https://developer.android.com/privacy-and-security/risks/tapjacking>
- Android Developers Blog：区分合法无障碍工具 — <https://developer.android.com/blog/posts/enhancing-android-security-stop-malware-from-snooping-on-your-app-data>
- Google Play：Use of the AccessibilityService API（2021-11-03 声明）— <https://support.google.com/googleplay/android-developer/answer/10964491>

**AOSP 源码与提交**

- `frameworks/base` 核心清单（`BIND_INPUT_METHOD` / `BIND_ACCESSIBILITY_SERVICE` / `BIND_AUTOFILL_SERVICE` 的保护级别）— <https://android.googlesource.com/platform/frameworks/base/+/refs/heads/main/core/res/AndroidManifest.xml>
- CVE-2023-21109 修复提交（`AccessibilityServiceInfo` 体积校验）— <https://android.googlesource.com/platform/frameworks/base/+/2c1f16db893680b0db29ffa222652fea3e5b87e0>
- CVE-2025-26462 修复提交（`onNullBinding` 解绑）— <https://android.googlesource.com/platform/frameworks/base/+/7076b2b2bd3cceea45f5d3dfa5ee279da819c9e1>
- CVE-2024-31310 修复提交（设置/更新服务时校验 intent filter）— <https://android.googlesource.com/platform/frameworks/base/+/74afbb05ca08738f66d82df867bbee66de4884bc>

**安全公告（逐格取证）**

- 2020-03-01 — <https://source.android.com/docs/security/bulletin/2020-03-01>
- 2021-04-01 — <https://source.android.com/docs/security/bulletin/2021-04-01>
- 2022-10-01 — <https://source.android.com/docs/security/bulletin/2022-10-01>
- 2023-05-01 — <https://source.android.com/docs/security/bulletin/2023-05-01>
- Pixel 2023-06-01 — <https://source.android.com/docs/security/bulletin/pixel/2023-06-01>
- Android 14 发布公告 — <https://source.android.com/docs/security/bulletin/android-14>
- 2024-06-01 — <https://source.android.com/docs/security/bulletin/2024-06-01>
- 2025-06-01 — <https://source.android.com/docs/security/bulletin/2025-06-01>
- 2025-12-01 — <https://source.android.com/docs/security/bulletin/2025-12-01>

**相邻手册**：见 §10。

---

## 未核实与失效来源

**A. `isPassword` 为真时框架是否脱敏。** 推测：节点信息里的密码文本会被替换为空或掩码。未核实原因：`AccessibilityNodeInfo` 参考页、无障碍指南与 Android 14 行为变更页全文均未找到「按 `isPassword` 隐去文本」的表述；本轮也没有对 `AccessibilityNodeInfo` 的 `setText` 路径做源码级追踪。**正文因此只写「API 契约里没有这条保证」，不写「框架一定返回明文」，也不写「一定返回掩码」。**

**B. 逐包查询「某应用是否被启用的无障碍服务」。** 推测：存在按包名过滤的公开 API。未核实原因：`AccessibilityManager` 的公开方法列表里只有 `getEnabledAccessibilityServiceList(int)` 与 `getInstalledAccessibilityServiceList()`，未见按包名查询的重载。**正文不写这种查询方式存在。**

**C. 「侧载」的精确定义。** 推测：Android 13 受限设置适用于所有非商店安装。未核实原因：本轮只核对了 Android 13 行为变更页的表述（浏览器、文件管理器等本地来源），未逐条核对 CDD 里 `PACKAGE_SOURCE_LOCAL_FILE` / `PACKAGE_DOWNLOADED_FILE` 的判定细节。**正文因此用「从浏览器、文件管理器等本地来源安装」这种描述性说法，不写判定常量。**

**D. 恶意软件滥用三条通道的案例。** 这类事件在安全厂商报告里很多，但**没有 CVE**，也不对应某个已修复的系统缺陷。本手册的案例表因此完全不收录它们；把它们与 CVE 混排会让读者误以为「系统已经修了」。

**E. Android 14 关于三条通道的其他收紧项。** 推测：除 `accessibilityDataSensitive` 外还有其他限制。未核实原因：本轮只逐条核对了 `accessibilityDataSensitive` 与 `isAccessibilityTool` 两条，未对 Android 14 行为变更页做全文穷举。**正文不声称「只有这两条」。**

**F. 企业策略的具体常量名。** 推测：DevicePolicyManager 或 Android Enterprise 有对应的输入法/无障碍白名单策略。未核实原因：本轮未抓取 `DevicePolicyManager` 参考页与 Android Enterprise 策略文档。**§8.3 因此只写方向，不写方法名或常量名。**

**G. 自动填充服务滥用风险的官方专页。** 推测：`developer.android.com` 上有一篇与点击劫持页并列的「autofill 滥用」风险页。未核实原因：在 autofill 指南与风险页目录中检索未找到；找到的只有「自动填充服务必须加密持久化的敏感数据」这一类要求。**正文因此不引用「官方风险页」，只引用 API 文档与公告。**

**H. `setAccessibilityDataSensitive` 的 XML 属性名。** 推测：存在与该方法对应的布局属性。未核实原因：本轮只核对了方法签名与 API 级别，未核对 `R.attr` 名称。**正文只写方法，不写 `android:` 属性。**

**I. `FLAG_SECURE` 与 `AssistStructure` 的代码级行为。** 正文的结论止于「官方文档没有承诺 `FLAG_SECURE` 阻止自动填充」。至于 AOSP 实现在某些版本上是否因为别的路径（例如 `onProvideAutofillStructure` 的窗口过滤）而顺带挡住，本轮未做完整源码追踪，**不作断言**。

**J. `InputMethodManagerService.setInputMethodWithSubtypeIdLocked` 的具体缺陷形态。** 公告只给 `EoP` / `Moderate` / 影响 `13`，未给提交说明的语义。**正文只写「输入校验」，不推测可利用条件。**

---

## 联系方式

咨询 Android 系统详情请联系 https://t.me/one00190

---

*本手册仅供安全学习与防御研究使用；所有 API 常量、权限保护级别、公告列值与修复落点均取自公开一手来源并附链接，未能核实的部分已单列于「未核实与失效来源」而非写入正文。*
