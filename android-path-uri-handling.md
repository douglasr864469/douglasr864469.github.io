---
layout: page
title: "Android 文件路径与 URI 处理类漏洞手册"
description: "把「把用户给的名字当成自己认识的对象」拆成三种逃逸与三条防线：同一个 inode 的多个名字（含 `isolateAppData()` 在私有 mount namespace 里 `symlink(\"/data/data\", \"/data/user/0\")`）、`canonical` 与 `normal..."
date: 2026-10-29
permalink: /android-path-uri-handling/
---

> 路径不是字符串，URI 不是路径，文件描述符才是对象本身。这份手册把「把用户给的名字当成自己认识的对象」这一类缺陷拆开讲：同一个文件可以有多个名字、`..` 与符号链接在解析时才生效、前缀比较少了分隔符就完全失效、SAF 授权随重启消失而前缀授权会把整棵子树交出去、解压时归档里的名字从不被 `java.util.zip` 归一化。全文站在防御与设计一侧，只讨论公开且已修复问题的机制与自查方法，不含可利用代码。

---

## 0. 这篇手册解决什么问题

已发布的二十篇里，`框架 IPC 与组件导出攻击面` 讲的是「谁能调到你」，`多用户隔离` 讲的是「你被关在哪个笼子里」，但两者都默认了一个前提：**你拿到的东西已经是你要的那个东西**。而路径与 URI 类缺陷破坏的正是这个前提——调用方给的是一串字符，你把它当成了自己目录下的那个文件。

这类缺陷有四个特点，值得单独成册：

1. **跨组件通用**。它不依赖某个具体 API 的漏洞，只要代码里存在「字符串 → 对象」的转换就可能出现；同一段写法在 `ContentProvider.openFile()`、`FileProvider`、解压、下载、缓存清理、递归删除里都会复现。
2. **检查与使用天然分离**。任何「先判断这个路径在不在我的目录里，再打开它」的写法都存在时间窗，因为判断的对象是名字、使用的对象是名字，而名字指向的东西可以被换掉。
3. **防御手段容易被误用**。`getCanonicalPath()`、`startsWith()`、`Uri.parse()` 这三个工具每一个都能写对、也能写错，且写错后的代码看起来非常像写对了。
4. **平台在为它做减法**。Scoped Storage、SAF、`FileUriExposedException`、`MANAGE_EXTERNAL_STORAGE` 的边界合起来，是在把「按路径授权的模型」换成「按类型与对象授权的模型」。理解这条演进线，比记住任何单个 API 都有用。

阅读建议：第 1–2 章建立名字空间的心智模型；第 3–5 章是三种最常见的实现级错误（归一化、链接、前缀）；第 6–7 章讲平台侧模型的变化；第 8–10 章是三个你必须亲自把关的边界（`FileProvider`、自写 `openFile()`、解压）；第 11–13 章是自查清单与相邻手册的边界。

---

## 1. 为什么「路径」不是字符串

### 1.1 一个对象，多个名字

Linux 的目录项（dentry）指向 inode，同一个 inode 可以有多个名字，也可以有多个**完全不同的路径**同时指向它：

- **符号链接**：名字 A 指向名字 B，解析时替换。
- **硬链接**：两个名字直接指向同一 inode，对文件而言没有「主名」。
- **绑定挂载（bind mount）**：把某棵子树挂到另一个位置，两个路径看到同一份数据。
- **mount namespace**：不同进程看到的挂载树可以不同，因此**同一个路径字符串在不同进程里可以指向不同的对象**。

最后一条在 Android 上不是理论。AOSP 的 `isolateAppData()`（`frameworks/base/core/jni/com_android_internal_os_Zygote.cpp`）在应用进程的私有 mount namespace 里做数据隔离，其中有一行：

```c
int result = symlink("/data/data", "/data/user/0");
```

`symlink()` 的参数顺序是 `(target, linkpath)`，即在**这个 namespace 内** `/data/user/0` 被建成指向 `/data/data` 的链接；同一段代码还先把 `/data/data` 与 `/data/user` 覆盖成 tmpfs，再逐包隔离出各自的可见视图。

这里要吸取的不是「谁指向谁」（不同版本、不同进程里方向都可能不同），而是：

> **路径字符串的解释依赖解释者所处的 namespace。** 你在自己的进程里算出的绝对路径，交给另一个进程后不保证指向同一个对象。

### 1.2 三种标识，三种信任级别

处理「文件」时其实有三种标识，安全性依次升高：

| 标识 | 例子 | 谁解析 | 风险 |
|---|---|---|---|
| 路径字符串 | `/data/user/0/com.x/files/a.db` | 每一个使用者各自解析 | 解析时可被链接替换、namespace 不同则指向不同 |
| URI | `content://com.x.fileprovider/root/a.db` | 由 provider 解析并授权 | 授权范围、前缀语义、provider 自己的校验决定一切 |
| 文件描述符 | `ParcelFileDescriptor` / `FileDescriptor` | 内核已解析完成 | 对象已绑定，后续无法被换掉 |

结论直接写成规范：

> **在你这一侧，凡是「检查过再用」的代码，都应该把检查对象与使用对象收敛成同一个已解析的句柄（FD 或已 `open` 的流）。** 只校验名字而不持有对象，就一定会留下 TOCTOU 窗口。

### 1.3 这条规则在你控制不了别人的时候仍然有效

你无法阻止另一个应用往共享目录里放符号链接，也无法阻止系统在你两次调用之间改名文件。所以第 4 章讲的不是「如何完全避免」，而是「如何把窗口压缩到内核替你保证的程度」：`openat()`/`O_NOFOLLOW`、以目录 FD 为基准的逐段解析、以及「先打开再校验 `fstat` 结果」的顺序。

---

## 2. 三个名字空间

Android 上「文件」出现在三套互不相同、但会互相映射的名字空间里。把它们的边界画清楚，后面所有错误都能归位。

### 2.1 应用私有空间

| 目录 | API | 典型路径 | 谁能读 |
|---|---|---|---|
| 内部文件 | `getFilesDir()` | `/data/user/0/<pkg>/files` | 仅本应用（同 UID + SELinux 域） |
| 内部缓存 | `getCacheDir()` | `/data/user/0/<pkg>/cache` | 同上 |
| 外部专属 | `getExternalFilesDir(null)` | `/storage/emulated/0/Android/data/<pkg>/files` | Android 11 起其他应用不可访问 |
| 外部缓存 | `getExternalCacheDir()` | `/storage/emulated/0/Android/data/<pkg>/cache` | 同上 |

要点：

- **私有空间的边界是 UID + 目录权限 + SELinux 三重**，不是「路径里有没有 `..`」。`..` 写在私有目录里并不会把你带到别人家，因为你没有那个 UID 的权限。
- `getFilesDir()` 与 `getDataDir()` 不是同一层：后者指向 `/data/user/0/<pkg>` 本身，里面还包含 `databases/`、`shared_prefs/`、`no_backup/`。把「整个 dataDir」暴露给外部输入是常见的事故起点。
- Android 11 起，`Android/data/<pkg>` 下的内容对其他应用**不可见**，即使用 `DownloadManager` 下载到那里也一样（官方在 storage use-cases 页明确写了这一点）。所以「放到外部专属目录让兄弟应用读」这条老做法在 11+ 上直接失效。

### 2.2 共享存储空间

同一块共享存储至少有三种写法：

```text
/sdcard
/storage/emulated/0
/mnt/user/0/primary
```

它们是同一份数据的多个视图（前两者在多数设备上是链接或绑定）。这带来两个后果：

1. **用字符串判断「是不是同一个目录」必然出错**。`/sdcard/Download` 与 `/storage/emulated/0/Download` 在语义上是同一个目录，在字符串上不相等。
2. **前缀比较必须基于同一套名字**。如果你存的是 `/storage/emulated/0`，而对端给的是 `/sdcard/...`，`startsWith` 会直接判负——这类「误判为不安全」的错误比「误判为安全」更隐蔽，因为它会让功能莫名其妙地失败，开发者往往改成「更宽松」的判断来修复。

访问模型上，Android 10（API 29）起对 target 29 的应用默认启用 Scoped Storage，并保留 `android:requestLegacyExternalStorage="true"` 作为临时退出；**target 30 之后，系统在 Android 11 设备上忽略该属性**，必须适配。官方文档对此的措辞是「After you update your app to target Android 11 (API level 30), the system ignores the `requestLegacyExternalStorage` attribute when your app is running on Android 11 devices」。另有 `android:preserveLegacyExternalStorage="true"`，它只为「升级迁移」保留旧模型，且**用户卸载重装后即失效**，不能当作长期开关。

### 2.3 文档提供者空间

SAF 给的 `content://` 没有路径语义。它由 authority + document id 组成，document id 的格式**由提供者自己定义**，可以是 `primary:Pictures/a.jpg`，也可以是一串数据库主键。因此：

- 不能从 `content://` 推出真实路径，也不应该尝试。
- 不能把 document id 当路径拼接后再用 `File` 打开——这在 `ExternalStorageProvider` 上「碰巧能用」，在云盘类 provider 上必然坏掉，而且给了一个把用户输入当路径解析的入口。
- 正确的抽象是 `DocumentsContract` 的文档/树 API 与 `ContentResolver.openFileDescriptor()`。

---

## 3. 规范化：能解决什么、解决不了什么

### 3.1 三个「看起来差不多」的 API

| 方法 | 做了归一化 | 解析符号链接 | 访问文件系统 |
|---|---|---|---|
| `File.getAbsolutePath()` | 否 | 否 | 否 |
| `File.getPath()` / `Paths.get(...).normalize()` | 是（纯字符串折叠 `..`、`.`） | **否** | 否 |
| `File.getCanonicalPath()` / `getCanonicalFile()` | 是 | **是** | 是（可能抛 `IOException`） |

三者常被混用。最危险的一种混用是：**用 `normalize()` 的结果做安全判断，却用原字符串去打开文件**——归一化折叠了 `..`，但打开时文件系统仍会按原字符串解析链接与 `..`，两者不同源。

`getCanonicalPath()` 也不等于「安全」：它解析的是**调用那一刻**的链接状态。解析完成后你得到一个字符串，再去 `new FileInputStream(path)`，中间的那一段时间正是第 4 章要处理的窗口。

### 3.2 归一化必须覆盖的六类差异

写「同一路径判定」时，下面每一项都曾被当成漏洞或功能缺陷：

1. **`..` 与 `.`**：`/root/a/../../etc`。
2. **重复分隔符与尾随分隔符**：`/root//a`、`/root/a/`。
3. **符号链接**：只有 canonical 解析能消掉（见第 4 章）。
4. **大小写**：FAT/exFAT 格式的 SD 卡通常大小写不敏感，`/sdcard/Download` 与 `/sdcard/download` 可能是同一目录，而 `String.equals` 判负。
5. **Unicode 归一化**：NFC 与 NFD 两种编码在视觉上相同、字节不同；某些文件系统会做归一化，Java 字符串不会。
6. **结尾的空白与点**：FAT 系文件系统会忽略名字结尾的空格和点，`a.jpg ` 与 `a.jpg` 可能落到同一个文件。

还有一个**不属于归一化但必须一起考虑**的问题：路径长度的上限。拼接用户输入前不限制长度，会得到 `ENAMETOOLONG`；如果代码里对错误「按截断处理」，就会把校验过的长路径截成另一个真实路径——这是校验与使用不一致的另一种形态。

### 3.3 规范性结论

> **归一化是必要不充分条件。** 归一化 + 前缀比较（带分隔符）能挡住「用 `..` 逃出目录」这一类，但挡不住「检查后被替换」，也挡不住「同一对象另一个名字」。前者要靠第 4 章的 FD 化，后者要靠「只在自己定义的名字空间内比较」——即比较之前先把自己这一侧的名字也转换成 canonical 形式，而不是拿两种不同来源的字符串硬比。

---

## 4. 符号链接与检查/使用分离

### 4.1 谁能创建链接

| 位置 | 同 UID 可写 | 其他应用可写 | 说明 |
|---|---|---|---|
| `/data/user/0/<pkg>/...` | 是 | 否 | 本应用自己的代码可以造链接，威胁来自「用户可控输入 + 递归处理」 |
| `/sdcard` 共享目录 | 视权限 | 视权限 | 任何拿到写权限的应用都可以放链接，受害者在「按路径处理」时才会踩到 |
| `/data/local/tmp` | adb shell | adb shell | 调试场景下常见，不要在生产代码里把它当临时目录 |

值得强调：**符号链接的攻击面不等于「别人能不能写我的目录」**。即使目录完全私有，只要你的代码接受「相对路径 / 文件名」这类输入并做递归处理（打包、清理、备份、解压），链接就可能由你的另一个功能、或用 `adb` 调试、或由被诱导的用户操作创造出来。

### 4.2 三种典型写法及其失败点

```java
// 写法 A：先校验、后打开（经典 TOCTOU）
File f = new File(baseDir, userInput);
if (!f.getCanonicalPath().startsWith(baseDir.getCanonicalPath())) return;
InputStream in = new FileInputStream(f);   // 中间可能已被替换成指向 /etc 的链接
```

```java
// 写法 B：只做字符串前缀（连归一化都没有）
if (!userPath.startsWith("/data/user/0/com.x/files/")) return;
new File(userPath).delete();
```

```java
// 写法 C：递归处理时不判断链接，直接跟随
void wipe(File dir) {
  for (File c : dir.listFiles()) {
    if (c.isDirectory()) wipe(c); else c.delete();
  }
}
```

失败点分别是：A 的窗口、B 的 `..` 与链接（字符串层面完全合法）、C 的「目录链接把你带到目录外」。

### 4.3 把窗口压到内核层

可用的手段按强度排列：

1. **以 FD 为准**：先 `open` 拿到 FD，之后所有操作基于 FD（`fstat` 校验类型与属主、`read`/`write`、`fchmod`），不再接受名字。这是唯一能真正关闭窗口的做法。
2. **`O_NOFOLLOW`**：打开最后一段时若为符号链接直接失败。`android.system.Os.open(path, flags, mode)` 可以传 `OsConstants.O_NOFOLLOW`。
3. **以目录 FD 为基准逐段解析**：打开基目录得到 dirfd，再 `openat(dirfd, "name", O_NOFOLLOW)`。Java 层没有直接的 `openat`，但可以用 `Files.newByteChannel(path, Set.of(LinkOption.NOFOLLOW_LINKS))` 表达「不跟随最后一段」的语义。
4. **打开后校验**：`Os.fstat(fd)` 检查 `st_dev`/`st_ino` 是否是你预期的那个对象，或者检查 `st_uid`。这条在无法 `O_NOFOLLOW` 时是兜底。

需要明确 Java 层的能力边界：`java.nio.file.Files` 支持 `NOFOLLOW_LINKS`，但没有 `openat2()` 的 `RESOLVE_BENEATH` 这类「整条路径都不许出基目录」的原子语义。**如果你的威胁模型要求「不许逃出基目录」的强保证，Java 层做不到原子化**，要么接受「打开后校验」，要么下到 native 层。

### 4.4 递归处理的三条规矩

- 遍历前用 `Files.isSymbolicLink(p)` / `Os.lstat()` 判断，遇到链接**默认跳过**，需要跟随时显式开启。
- 删除时先 `NOFOLLOW` 打开再判断类型，不要用 `File.isDirectory()`（它会跟随链接）。
- 复制/打包同理：跟随链接会把「目录外的东西」装进包里，等于把私有文件带出去。

---

## 5. 前缀比较：最常见的实现级错误

### 5.1 少了分隔符

```java
// 错
if (child.getCanonicalPath().startsWith(root.getCanonicalPath())) { /* 认为在 root 内 */ }
```

`/sdcard/foo` 与 `/sdcard/foobar` 通过这个判断，但后者不在前者里。正确写法是先把 root 的尾部分隔符去掉，再比较 `root + "/"`：

```java
String r = root.getCanonicalPath();
if (!r.endsWith("/")) r = r + "/";
if (!childCanonical.startsWith(r)) return;
```

`androidx.core.content.FileProvider` 的实现就采用这个形态：它的 `belongsToRoot()` 会去掉尾部斜杠后再比较 `filePath.startsWith(rootPath + '/')`，注释与实现的目的正是为了让 `/data/files2` 不匹配 `/data/files`。

### 5.2 两侧名字空间不一致

即使比较方法正确，两侧字符串来源不同也会失效：

- 一侧是 `Environment.getExternalStorageDirectory()`（历史上常返回 `/sdcard` 风格），另一侧是 `MediaStore` 查出来的 `_data`（通常是 `/storage/emulated/0/...` 形态）。
- 一侧来自 `getCanonicalPath()`（已解链接），另一侧是原始 `Uri` 的 path 段（未解）。

规范：**比较前，两侧都必须经过同一个函数处理**，并且这个函数要说明它做什么（归一化？解链接？大小写折叠？）。代码评审时，凡是看到 `startsWith(` 后面跟着一个「不是刚刚被 canonical 过的」变量，都应该停下来看。

### 5.3 相对拼接的两个误解（含一条误传纠正）

第一个误解：**`..` 会被自动处理**。不会。`new File(root, "../x")` 在 Java 侧保留 `..`，只有文件系统解析时才逃出 root。

第二个是流传很广的误传：

> 「`new File(root, userInput)` 在 `userInput` 是绝对路径时会跳出 root。」

这条描述的是 **Python `os.path.join`** 的行为。Java 的 `File(File parent, String child)` 会把 child 一律按相对部分处理（内部行为是 `parent + child`，不归一化），所以 `/etc/passwd` 这种输入得到的是 `<root>/etc/passwd` 而不是 `/etc/passwd`——**在 Java 侧这条传说不成立**。

真正会「绝对路径覆盖基路径」的是 `java.nio`：

```java
Path p = base.resolve(userInput);   // userInput 是绝对路径时，resolve 直接返回它
```

`Path#resolve` 的文档语义明确：如果参数是绝对路径，返回该参数本身。因此混用 `File` 与 `Path` 的代码里，逃逸向量是**两种**：`..`（两者都有）与绝对路径覆盖（仅 `Path.resolve`）。

结论：拼接之后必须做一次「canonical + 带分隔符前缀」检查，无论用哪种 API。**不要靠记住某一种 API 的逃逸规则来写安全代码**，这才是可维护的做法。

### 5.4 双重解码

`Uri.getPath()` 返回的是已解码的路径段，`Uri.toString()` 是编码形态。如果代码先对 `toString()` 做一次 `URLDecoder.decode`，再交给 `new File`，就会把 `%252e%252e` 解成本不存在的东西、或把 `%2e%2e` 再解一次得到 `..`。规范：**只解码一次，且在确立「这一层是原始字节」之后立即解码，之后全程用解码后的值**；不要把「已解码值」再传回会被自动解码的 API。

---

## 6. Scoped Storage：把「路径权限」换成「类型权限」

这一章讲的是平台侧模型的变化，因为它决定了「路径类缺陷还能造成多大伤害」。

### 6.1 时间线（官方口径）

| 版本 | 行为 |
|---|---|
| Android 10（API 29） | target 29 默认启用 scoped storage；可用 `android:requestLegacyExternalStorage="true"` 临时退出（默认 false） |
| Android 11（API 30） | target 30 的应用在 11 设备上**忽略** `requestLegacyExternalStorage`；要求适配 |
| Android 11 迁移 | `android:preserveLegacyExternalStorage="true"` 只为升级时迁移数据保留旧模型，卸载重装后失效 |
| Android 11+ | target 30 时 `WRITE_EXTERNAL_STORAGE` 不再影响存储访问能力（官方文档的表述是它「不影响你的应用的存储访问」） |

`WRITE_EXTERNAL_STORAGE` 这条最容易被误读：它不是「被废弃」，而是**在 target 30 的模型里不再是决定因素**——决定因素是文件类型、归属与是否拿到 `MediaStore`/SAF 授权。

### 6.2 `MANAGE_EXTERNAL_STORAGE` 的边界

官方对该权限的描述是「读写共享存储内的所有文件、访问 `MediaStore.Files` 表、访问 OTG 与 SD 卡根目录」，并明确写了三条**做不到**：

1. **仍然进不去其他应用的 `Android/data/<pkg>`**（这些目录在共享存储上表现为 `Android/data` 的子目录，但受保护）。
2. 走 SAF 时，**它不会让你访问本无权访问的文件**：文档原话是「you can only access a file or directory if you can do so without having the `MANAGE_EXTERNAL_STORAGE` permission」。
3. 受 Play 政策约束，只有文件管理器、备份恢复、杀毒、文档管理、设备间迁移等类别才被允许申请。

判断是否已获授权用 `Environment.isExternalStorageManager()`，引导用户去系统设置用 `Settings.ACTION_MANAGE_ALL_FILES_ACCESS_PERMISSION`。测试时官方给的命令是：

```bash
adb shell appops set --uid <PACKAGE_NAME> MANAGE_EXTERNAL_STORAGE allow
```

### 6.3 `MediaStore` 的写入契约

- 新建/更新媒体**不要**用 `_data`（DATA 列）拼路径：官方明确「To create or update a media file, don't use the value of the `DATA` column. Instead, use `DISPLAY_NAME` and `RELATIVE_PATH`」。
- 写大文件用 `IS_PENDING = 1` 占位，写完置 0，未置 0 前别人看不到。
- `RELATIVE_PATH` 是**提示**，不是绝对路径；它相对于该媒体类型的公共目录。

### 6.4 对漏洞分析的意义

理解这一层之后，可以得出两条判断规则：

> **规则一：在共享存储上，「路径穿越」的收益被压到了「访问我自己也有权访问的文件」。** 真正的越权需要突破 provider 层（第 8–9 章），而不是路径字符串。
>
> **规则二：在 provider 内部，路径穿越依然成立且依然高危。** MediaProvider、DocumentsUI、下载管理器这类组件持有系统级权限，它们内部的 `openFile()` 如果拼接了用户可控的路径段，后果与 Scoped Storage 无关。

---

## 7. SAF 与 URI 授权：权限是「对象 + 动作 + 时间」

### 7.1 授权的时间维度

Storage Access Framework 的核心不是「选到了文件」，而是「选到了那一刻被授予的一个临时权限」。官方文档的原话是：系统给你的 URI 权限授予「lasts until the user's device restarts」——**重启即失效**。

想跨重启保留，必须显式「取走」持久授权：

```kotlin
val takeFlags = intent.flags and
    (Intent.FLAG_GRANT_READ_URI_PERMISSION or Intent.FLAG_GRANT_WRITE_URI_PERMISSION)
contentResolver.takePersistableUriPermission(uri, takeFlags)
```

官方同时给出了一条重要限制：「Even after calling `takePersistableUriPermission()`, your app doesn't retain access to the URI if the associated document is moved or deleted.」——文档被移动或删除后，授权随之作废，必须重新向用户申请。

由此得出设计上的三条：

1. **不要把持久 URI 当作「永久可用的文件路径」**。它随时可能因为用户重命名、移动、删除、或换设备而失效，代码必须处理 `SecurityException` 与 `FileNotFoundException`，并回退到重新选择。
2. **不要把「本次授权」当成「长期授权」**。不做 `takePersistableUriPermission` 就拿到的授权，在重启后一定失效；很多「上次能打开这次打不开」的缺陷根因就在这里。
3. **授权是按 UID 记的**。多用户/工作资料里同一包名的两个实例是不同的 UID，授权不共享；这一点与 `多用户隔离` 那篇里的 uid 规则一致。

### 7.2 前缀授权：一次给出一整棵子树

`Intent.FLAG_GRANT_PREFIX_URI_PERMISSION`（值为 `0x80`）的含义是：授予的权限**覆盖以该 URI 为前缀的所有 URI**。`ACTION_OPEN_DOCUMENT_TREE` 正是靠它把「选中的目录及其所有子目录」一次性授权出去。

风险点在于这条语义的两面性：

- 对用户而言，选一个目录就是选一棵树，这是预期行为。
- 对代码而言，**前缀匹配只看字符串前缀**。如果你的 provider 里 `content://auth/tree/1` 与 `content://auth/tree/10` 是两棵不同的树，前缀授权可能把两者都覆盖。自写 provider 时，document id 的命名必须避免「一个是另一个的字符串前缀」这种关系，否则授权会溢出到不该给的节点。

另一个相关的常量是 `FLAG_GRANT_PERSISTABLE_URI_PERMISSION`：它表示「这个授权允许被持久化」，与 `FLAG_GRANT_READ/WRITE_URI_PERMISSION` 组合使用。它本身不授予任何访问权，只决定「能不能被 `takePersistableUriPermission` 收下」。

### 7.3 `ACTION_OPEN_DOCUMENT_TREE` 在 Android 11+ 的收缩

官方明确列了不能再选的目录：

- 内部存储卷的根目录；
- 各 SD 卡卷（厂商认为「可靠」的卷）的根目录；
- `Download` 目录。

同时，不能通过它让用户选择 `Android/data/` 与 `Android/obb/` 下的任何文件。这条收缩的意义是：**「选一个目录拿到全盘」这条路径被平台堵住了**，剩下的宽口径访问只能走 `MANAGE_EXTERNAL_STORAGE`（受 Play 政策约束）或系统角色。

### 7.4 一条常见误传

> 「拿到 SAF 的 URI 就等于拿到了真实路径，可以转成 `/sdcard/...` 再用 `File` 打开。」

不成立，而且方向是错的：`content://` 的 document id 由 provider 定义，可能是数据库主键、可能是云端对象键。在某些 provider 上「碰巧」能从 document id 拼出真实路径，是因为该 provider 恰好用路径做 id（`ExternalStorageProvider` 就是如此），这不是契约。依赖它会在换机型、换 provider、或系统升级后失效；更糟的是，它把「不可信的 id 字符串」重新变成了「路径输入」，正好制造出本篇要防的那类缺陷。

---

## 8. `FileProvider`：把私有文件交给别人的唯一正规通道

### 8.1 前置：为什么不能直接给 `file://`

`android.os.FileUriExposedException` 的类文档说得很直接：当应用把 `file://` 形式的 `Uri` 暴露给另一个应用时抛出，因为「This exposure is discouraged since the receiving app may not have access to the shared path」，建议改用 `content://`，让平台能给接收方授予临时访问权；并且它「is only thrown for applications targeting `Build.VERSION_CODES#N` or higher」。

也就是说：**target N 之后，`file://` 越界暴露在框架层就被拦住了**。这把「自己拼路径给别家」的做法整体推到了 `FileProvider`（或自写 provider）上——而 provider 的路径配置就成了新的、必须审的边界。

### 8.2 清单三件套与七个根

正确配置的最小形态：

```xml
<provider
    android:name="androidx.core.content.FileProvider"
    android:authorities="com.example.app.fileprovider"
    android:exported="false"
    android:grantUriPermissions="true">
    <meta-data
        android:name="android.support.FILE_PROVIDER_PATHS"
        android:resource="@xml/file_paths" />
</provider>
```

`<paths>` 支持的子元素（来自 `androidx.core.content.FileProvider` 源码中的常量）共七个：

| 子元素 | 常量 | 映射到的根 |
|---|---|---|
| `files-path` | `TAG_FILES_PATH` | `context.getFilesDir()` |
| `cache-path` | `TAG_CACHE_PATH` | `context.getCacheDir()` |
| `external-path` | `TAG_EXTERNAL` | `Environment.getExternalStorageDirectory()` |
| `external-files-path` | `TAG_EXTERNAL_FILES` | `getExternalFilesDirs()[0]` |
| `external-cache-path` | `TAG_EXTERNAL_CACHE` | `getExternalCacheDirs()[0]` |
| `external-media-path` | `TAG_EXTERNAL_MEDIA` | `getExternalMediaDirs()[0]` |
| `root-path` | `TAG_ROOT_PATH` | 设备根 `/` |

每个元素有 `name`（URI 里的路径段）与 `path`（相对该根的子目录）。要点：

- **`root-path` 意味着「任何 canonical 路径都能命中某个根」**。它不会让 provider 无视路径校验，但会让「路径是否在授权根内」这项检查失去筛选意义——等价于把整个文件系统纳入可映射范围。生产配置里出现 `root-path` 应当被视为高危配置。
- **`external-path` 的 `path` 留空**等于把整个共享存储根纳入映射，虽然读写仍受权限约束，但一个误配的 `FLAG_GRANT_WRITE` 就会变成「可写共享存储」。
- `external-*` 系列只取**第一个**外部目录（`[0]`）。在多卷设备上，用户数据可能在 `[1]`，功能会失效——这是功能缺陷而非安全问题，但常被误判成「路径穿越」。

### 8.3 它自己怎么做校验

`FileProvider` 的路径策略有三处关键实现，值得当作「正确写法」的参考：

1. **配置阶段 canonical 化**：`addRoot()` 里对每个根执行 `root.getCanonicalFile()`，配置里的相对 `path` 也被拼上后再 canonical 化。这样 `..` 在配置里就失效了。
2. **`getUriForFile()` 先 canonical 化候选文件**，再在全部根中挑**最具体（最长匹配）**的那个；如果没有根包含它，抛 `IllegalArgumentException("Failed to find configured root that contains ...")`。
3. **`getFileForUri()` 反向解析**：`new File(root, path)` 之后 `getCanonicalFile()`，若结果跳出了根，抛异常 `"Resolved path jumped beyond configured root"`；前缀判定用 `belongsToRoot()`，即去尾斜杠后比较 `rootPath + '/'`（因此 `/data/files2` 不会命中 `/data/files`）。

除此之外，`attachInfo()` 会校验 provider 的配置：**不能 exported、必须 `grantUriPermissions="true"`、authority 不能为空**。任何一条不满足都会在 attach 阶段抛异常。这条可以当成「部署即自检」的例子：把安全约束写成启动期断言，比写在文档里可靠。

### 8.4 授权是怎么被「顺带」给出去的

`FileProvider` 只是把文件映射成 URI；真正的权限授予发生在 `Intent` 上：

- 发送方在 `Intent` 上加 `FLAG_GRANT_READ_URI_PERMISSION`（需要写就再加 `FLAG_GRANT_WRITE_URI_PERMISSION`），并把 URI 放在 `data` 或 `ClipData` 里。
- 平台在投递时把这份临时授权给**接收方**。注意：`ClipData` 里的 URI 也会被一并授予——很多人只在 `data` 上设 flag，却把 `EXTRA_STREAM` 设成了含 URI 的 `ClipData`，结果授权范围比预期大。
- 发送方一旦给了 `FLAG_GRANT_WRITE_URI_PERMISSION`，接收方就能改写你给出去的那个文件。**只读场景绝不要顺手把 WRITE 也加上**（常见的「复制一份 flag 组合」写法就会这样）。

三条实践规则：

1. **能显式指定接收方就显式指定**（显式组件或 `setPackage()`）。用隐式 `ACTION_SEND` + `createChooser` 时，授权在用户选择之后才生效；如果你自己再拼一个「全部应用都能收到」的广播式发送，等于把授权范围交出去。
2. **一次只给一个 URI、只给必要动作**。需要给多个文件时，逐个确认必要性，而不是批量加 flag。
3. **给完不能收回**。URI 授权一旦授予，发送方没有「撤销」的常规手段（除非调用 `revokeUriPermission()`，而它对已投递的授权同样存在时间差）。所以「先想清楚再发」是唯一可控的时点。

---

## 9. 自写 `ContentProvider.openFile()`：provider 自己就是边界

### 9.1 契约

`ContentProvider.openFile(Uri uri, String mode)` 的契约是：**返回指向 `uri` 所代表文件的 `ParcelFileDescriptor`**。框架不会替你校验「这个 uri 对应的文件是否属于你」。因此每一个自写 provider 都必须自己完成三件事：

1. **authority 与结构校验**：`UriMatcher` 命中哪一类路径；段数是否匹配。
2. **名字 → 对象**：把 uri 的路径段解析成自己名字空间内的对象（数据库行、包内文件），而不是把它当路径拼接。
3. **归属校验**：解析出的对象必须属于当前 provider 的包（或属于调用者）。

### 9.2 最常见的三种错误

```java
// 错误一：把 uri 最后一段直接拼进私有目录
File f = new File(getContext().getFilesDir(), uri.getLastPathSegment());
return ParcelFileDescriptor.open(f, MODE_READ_ONLY);   // lastPathSegment 可以是 "..%2F.." 解码后的任意值
```

```java
// 错误二：UriMatcher 只匹配"类"，忽略"参数"
// content://auth/files/* 命中后，把 * 当作 id 查询，但没有校验 id 的合法性
```

```java
// 错误三：解析后不 canonical，直接前缀比较
if (!f.getPath().startsWith(getContext().getFilesDir().getPath())) return null;
```

错误一的问题在于 `getLastPathSegment()` 返回的是**已解码**段，它可能包含 `/`、`..`；错误二的问题在于匹配成功只说明「形状对」，不说明「对象属于你」；错误三的问题在第 5 章：未 canonical 的字符串比较既可能放过 `..`，也可能因为链接而误判。

### 9.3 正确的结构

```java
// 结构示意（防御侧写法，不含任何利用细节）
@Override
public ParcelFileDescriptor openFile(Uri uri, String mode) throws FileNotFoundException {
    switch (matcher.match(uri)) {
        case FILES: {
            String id = uri.getPathSegments().get(1);
            if (!ID_PATTERN.matcher(id).matches()) throw new FileNotFoundException(); // 只接受自己定义的 id 形状
            File base = new File(requireContext().getFilesDir(), "shared");
            File target = new File(base, id + ".bin");
            String rootPath = canonical(base) + "/";
            if (!canonical(target).startsWith(rootPath)) throw new FileNotFoundException();
            return ParcelFileDescriptor.open(target, toMode(mode));
        }
        default: throw new FileNotFoundException();
    }
}
```

三个细节：**id 走白名单形状**（比黑名单过滤 `..` 可靠）；**canonical 之后再比较，且比较串带 `/`**；**对未知模式与未知 uri 一律抛异常**，不要「尽量满足」。

### 9.4 FD 给出之后

`ParcelFileDescriptor` 一旦返回，调用方拿到的就是一个已打开的句柄。此后：

- 你**不能**再通过改路径来限制它：它已经指向某个 inode。
- 你在返回前做的所有校验，都必须在这一刻完成；返回后没有任何补救手段。
- 若返回的是目录 FD（`openFile` 返回目录句柄是允许的），调用方在某些情况下可以做 `readdir`，等于把整个目录列给你没打算给的人。**除非明确需要，不要返回目录 FD。**

`openTypedAssetFile()` 与 `AssetFileDescriptor` 还多一层「区间」语义：`startOffset`/`length` 让调用方读文件的某一段。区间参数同样要校验（负值、越界、求和溢出），否则「只给前 1KB」可能变成「从任意偏移读任意长度」。

---

## 10. 归档解压：Zip Slip 及其变体

### 10.1 根因只有一句

`java.util.zip` 的 `ZipEntry.getName()` **原样返回归档内记录的条目名**，`ZipInputStream`/`ZipFile` 都不做任何路径归一化或校验。因此：

```java
ZipEntry e;
while ((e = zin.getNextEntry()) != null) {
    File out = new File(destDir, e.getName());   // e.getName() 可能是 "../../../data/..."
    // 直接写出去 = 把文件写到 destDir 之外
}
```

### 10.2 正确的校验顺序

```text
1. 拒绝绝对路径（以 "/" 开头）与带盘符的名字（形如 "C:\"）
2. 用 "/" 统一分隔符后，按段拆分；任一段为 ".." 或 "." → 拒绝（或折叠后再校验）
3. 目标 = canonical(destDir) + 相对路径；再 canonical 化
4. 目标必须以 canonical(destDir) + "/" 开头，否则拒绝
5. 若条目表示链接（Zip 的 external attributes 里带 S_IFLNK），默认拒绝
6. 单文件解压后大小、总大小、条目数量都要有上限（防炸弹）
```

第 4 步与第 5 章的规则是同一个：**canonical 之后、带分隔符的前缀比较**。第 5 步的原因：Zip 规范允许条目携带 Unix 权限位，攻击者可以把一个条目标成符号链接，解压后它就指向任意位置；此后你再解压下一个条目，写入路径就被链接带走了。

### 10.3 变体清单

| 变体 | 机制 | 校验点 |
|---|---|---|
| 绝对路径条目 | 名字以 `/` 开头，`new File(dest, name)` 在某些实现里得到绝对路径 | 显式拒绝绝对名 |
| 盘符/UNC | `C:\...`、`\\server\share` | 拒绝含 `:` 或 `\\` 的名字 |
| 链接条目 | 条目本身是符号链接 | 检查 external attributes 的 `S_IFLNK` 位 |
| 嵌套归档 | 解出来的包再解一次，第二层绕过第一层校验 | 对每一层都应用同一套校验 |
| 压缩炸弹 | 极小体积解出极大体积 / 极多条目 | 限制单文件大小、总大小、条目数、递归深度 |
| 覆盖攻击 | 解压覆盖应用自己的配置/可执行内容 | 解压目标目录必须是本次新建的临时目录，且不与代码目录重叠 |
| 大小写/Unicode 碰撞 | `A.txt` 与 `a.txt` 在 FAT 上是同一文件，后者覆盖前者 | 解压前做名字冲突检测（按目标文件系统的规则） |

### 10.4 解压后的权限

解压出来的文件权限取决于你的实现：用 `FileOutputStream` 创建得到的是默认 umask 结果；从条目里恢复权限（`setPosixFilePermissions`）则是另一个隐患——把 `0666` 或 `0777` 恢复到共享存储上的文件，等于放宽了别人的访问面。**恢复权限只应发生在自己完全掌控的私有目录，且只恢复 `0644`/`0600` 这类白名单值。**

---

## 11. 编码与时序：两个被低估的维度

### 11.1 URI 的编码歧义

- **`Uri.encode()` 与 `URLEncoder.encode()` 不是一回事**。后者是 form 编码，会把空格变成 `+`；URI 的路径段里空格应当是 `%20`。用 `URLEncoder` 拼路径段，会让名字里带空格的路径在服务端/provider 侧被解释成 `+`，产生「同一个文件两个名字」。
- **authority 不区分大小写**（域名语义），path 区分。比较 authority 时要用小写形式；比较 path 时绝不能折大小写——除非你确认目标文件系统不敏感（见第 3 章）。
- **`Uri.parse()` 不做校验**。它接受几乎任何字符串，`getPath()` 可能是 `null`（比如 `content://auth` 就没有 path）。所有 `getPath()`/`getLastPathSegment()` 的返回值都必须判空。
- **尾随点与用户信息**：URI 语法允许 `userinfo@`、允许 `path/./`；不同解析器对这些成分的处理不一致。跨进程传 URI 时，接收方应只信任**自己重新解析后的结果**，不要信任发送方附带的字符串副本。

### 11.2 时序：不只存在于文件系统

TOCTOU 在本篇的语境里至少有三个层次：

1. **文件层**：检查名字 → 使用名字（第 4 章）。
2. **授权层**：检查授权 → 使用授权。授权可能在两次调用之间被撤销（用户移除 SD 卡、撤销 SAF 授权、多用户切换），因此授权检查要通过**一次真实的打开操作**来完成，而不是「先 `checkUriPermission()` 再 `openFileDescriptor()`」。
3. **配置层**：检查配置 → 使用配置。`FileProvider` 的 `attachInfo()` 把配置校验放在进程启动时，正是为了消除这一层的窗口。

### 11.3 一条实用判据

> 如果一段代码里出现「同一个名字被解析了两次」，就应当把它当作缺陷候选。

这句话覆盖了上面三个层次的全部形态：`getCanonicalPath()` 一次、`new FileInputStream()` 又一次；`checkUriPermission()` 一次、`openFileDescriptor()` 又一次；`getCanonicalFile()` 检查配置一次、运行时又按原配置字符串解析一次。

---

## 12. 自查清单

### 12.1 代码层（逐条 yes/no，任一为 no 就是缺陷候选）

| # | 检查项 |
|---|---|
| 1 | 所有「路径是否在目录内」的判断都基于 `getCanonicalPath()`/`getCanonicalFile()`，且比较串以 `/` 结尾 |
| 2 | 没有任何一处用 `startsWith()` 比较「未 canonical 的字符串」 |
| 3 | 用户可控输入进入路径前，先经过**白名单形状**校验（而不是黑名单过滤 `..`） |
| 4 | 检查与打开之间不存在「再解析一次名字」的间隙；能拿 FD 就拿 FD |
| 5 | 递归遍历/删除前判断了符号链接，默认不跟随 |
| 6 | 解压的每个条目都做了「canonical 目标 + 带 `/` 前缀」校验，并拒绝绝对路径与链接条目 |
| 7 | 解压有大小/条目数/层数上限 |
| 8 | `FileProvider` 的 `<paths>` 里没有 `root-path`，`external-path` 的 `path` 非空；`exported=false` 且 `grantUriPermissions=true` |
| 9 | 给外部应用的 `Intent` 只带 `FLAG_GRANT_READ_URI_PERMISSION`（除非确实需要写），且尽量显式指定接收方 |
| 10 | 自写 provider 的 `openFile()` 对未知 uri/模式一律抛异常，不返回目录 FD |
| 11 | SAF 授权做了持久化决策（要么持久化并处理失效，要么明确只用单次授权） |
| 12 | 共享存储的写入走 `DISPLAY_NAME`/`RELATIVE_PATH`/`IS_PENDING`，不用 `_data` 拼路径 |

### 12.2 设备侧（可在自己的调试设备上跑，不涉及他人数据）

```bash
# 看某个应用私有目录的权限与属主
adb shell ls -ld /data/user/0/<pkg>

# 看某路径是不是链接，以及最终指向
adb shell readlink -f /data/user/0/<pkg>/files/some

# 列出某个目录下的所有符号链接（含指向目录外的）
adb shell find /data/user/0/<pkg> -type l -exec ls -l {} \;

# 确认应用是否被授予“管理所有文件”
adb shell appops get <pkg> MANAGE_EXTERNAL_STORAGE
```

### 12.3 评审时的一句话

> 「这个字符串是谁生成的？它被解析了几次？第二次解析时，它指向的东西还是第一次看到的那个吗？」

---

## 13. 与相邻手册的边界

| 手册 | 分工 |
|---|---|
| [android-framework-ipc-export.md](/android-framework-ipc-export/) | 讲「谁能调到你的组件」。本篇不重复组件导出与权限声明，只讲组件被调到之后**怎么处理传进来的名字** |
| [android-multiuser-sandbox-isolation.md](/android-multiuser-sandbox-isolation/) | 讲 UID、`/data/user/<id>`、工作资料的隔离边界。本篇引用的 uid 规则以该篇为准 |
| [android-apk-signing-install.md](/android-apk-signing-install/) | 讲安装时的签名与校验。安装器读 APK 路径同样受本篇规则约束，但签名校验的细节在该篇 |
| [android-adb-backup-transfer.md](/android-adb-backup-transfer/) | 备份/迁移会批量读取应用数据，路径规则与 `bmgr` 的域限制在该篇 |
| [android-common-system-vulnerabilities.md](/android-common-system-vulnerabilities/) | 通用类型总览。本篇是其中「路径与 URI」这一类的展开 |

---

## 参考

- Android Developers，`Data and file storage overview`（访问范围、`MANAGE_EXTERNAL_STORAGE`、`WRITE_EXTERNAL_STORAGE` 在 API 30 的定位）：<https://developer.android.com/training/data-storage>
- Android Developers，`Storage use cases and best practices`（`requestLegacyExternalStorage`、`preserveLegacyExternalStorage` 与「target 30 后被忽略」的原句）：<https://developer.android.com/training/data-storage/use-cases>
- Android Developers，`Access media files from shared storage`（`IS_PENDING`、`RELATIVE_PATH`、`DATA` 列的用法限制）：<https://developer.android.com/training/data-storage/shared/media>
- Android Developers，`Access documents and other files from shared storage`（SAF 授权「lasts until the user's device restarts」、`takePersistableUriPermission`、`ACTION_OPEN_DOCUMENT_TREE` 在 11+ 的禁止目录）：<https://developer.android.com/training/data-storage/shared/documents-files>
- Android Developers，`Manage all files on a storage device`（`MANAGE_EXTERNAL_STORAGE` 的四项授予与三条做不到、`Environment.isExternalStorageManager()`、`appops` 测试命令）：<https://developer.android.com/training/data-storage/manage-all-files>
- Android Developers，`Sharing a file`（`FileProvider` 清单配置与 `file_paths.xml` 示例）：<https://developer.android.com/training/secure-file-sharing/setup-sharing>
- AOSP，`frameworks/base/core/java/android/os/FileUriExposedException.java`（类文档原文：target N 起抛出、建议改用 `content://`）：<https://android.googlesource.com/platform/frameworks/base/+/refs/heads/master/core/java/android/os/FileUriExposedException.java>
- AOSP，`frameworks/base/core/jni/com_android_internal_os_Zygote.cpp`（`isolateAppData()` 中 `symlink("/data/data", "/data/user/0")` 与私有 mount namespace）：<https://android.googlesource.com/platform/frameworks/base/+/master/core/jni/com_android_internal_os_Zygote.cpp>
- AndroidX，`core/core/src/main/java/androidx/core/content/FileProvider.java`（`<paths>` 七个标签常量、`belongsToRoot()` 的 `rootPath + '/'`、`getFileForUri()` 的 `"Resolved path jumped beyond configured root"`、`attachInfo()` 的配置断言）：<https://github.com/androidx/androidx/blob/androidx-main/core/core/src/main/java/androidx/core/content/FileProvider.java>
- AOSP，`frameworks/base/core/java/android/content/Intent.java`（`FLAG_GRANT_PREFIX_URI_PERMISSION`、`FLAG_GRANT_PERSISTABLE_URI_PERMISSION`）：<https://android.googlesource.com/platform/frameworks/base/+/refs/heads/master/core/java/android/content/Intent.java>
- OpenJDK，`java.io.File`、`java.nio.file.Path#resolve`（`File(parent, child)` 不做归一化、`resolve` 在参数为绝对路径时直接返回参数）：<https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/nio/file/Path.html>

---

## 未核实与失效来源

本篇写作时以下各项未被一手来源证实，或只做了有限核对，**不应据以对外引用**：

- **A. `MediaProvider` 与 FUSE 的细节未逐行核对。** 文中提到「共享存储由 MediaProvider 通过挂载层按应用过滤」，这是 AOSP 的架构常识，但本轮没有打开 `packages/providers/MediaProvider` 与 `system/core` 中相关源文件逐行确认挂载参数与过滤逻辑，故正文没有给出任何具体属性名或挂载选项。
- **B. 持久 URI 授权的数量上限未核实。** 社区常引用「每个应用最多 128 个持久授权」，本轮在 `ContentResolver` 与 SAF 文档中都未找到该数字，因此未写入正文。
- **C. 具体 CVE 编号与本类缺陷的对应关系一律未核。** 本篇刻意不列 CVE：路径类问题在公告里常归入 Framework 或 System 组件的 EoP 条目，但**条目描述与「路径穿越」的对应需要逐条读公告正文**，本轮没有做这件事，也不做归属推断。需要逐条台账的，方法见 [android-cve-and-security-bulletins.md](/android-cve-and-security-bulletins/)。
- **D. `/data/data` 与 `/data/user/0` 的链接方向依赖版本与 mount namespace。** 正文只引用了 `isolateAppData()` 里的那一行，并明确说明它是**应用进程私有 namespace 内**的形态；不同 Android 版本、不同进程看到的可能是另一方向。不要把正文这句话推广成「设备上永远是 X 指向 Y」。
- **E. FAT/exFAT 的具体挂载选项未核。** 正文说「外置卡通常大小写不敏感、忽略结尾空格与点」，这是 FAT 系列文件系统的通用行为，但本轮没有确认各厂商 `vold` 的实际挂载参数，因此没有给出 `vfat` 选项名。
- **F. `Files.newByteChannel(..., NOFOLLOW_LINKS)` 在各 API level 的行为未逐一实测。** 正文把它作为「Java 层表达不跟随最后一段」的可用手段，但 Android 各版本 libcore 的差异未验证，落地前应在本项目的 minSdk 上实测。
- **G. `openat2()` / `RESOLVE_BENEATH` 在 Android 内核上的可用性未核。** 正文只说「Java 层没有等价的原子语义」，没有断言内核是否支持；NDK 侧是否可用取决于内核版本与 syscall 白名单，未验证。
- **H. 一处官方表述的转述来源。** 关于「target 30 时 `WRITE_EXTERNAL_STORAGE` 不影响存储访问」，本轮抓取到的 `data-storage` 页面是本地化版本，英文页对应句未在同一次抓取中直接命中，因此正文按语义转述而非逐字引用。

---

## 联系方式

如果你在企业里做 Android 应用或系统组件的安全评审，需要就「路径与 URI 处理」「provider 边界」「解压与归档处理」做一次针对代码库的核查，可以通过 Telegram 联系：<https://t.me/one00190>

---

*本手册仅用于防御与教育目的，讨论的均为公开且已被修复问题的机制与设计缺陷，不包含任何可直接利用的代码。文中标注「未核实」的内容不应作为事实引用；发现错误欢迎指出。*

