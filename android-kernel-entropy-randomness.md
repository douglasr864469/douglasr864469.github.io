---
layout: page
title: "Android 熵源与随机数安全边界手册"
description: "从 `getrandom` 的引入讲起，说明 `GRND_INSECURE` 与 `GRND_RANDOM` 的真实语义（后者在新内核上已无实际阻塞效果）；拆启动期熵来源、`kaslr-seed` 的传递、`CONFIG_RANDOM_TRUST_BOOTLOADER` 的信任假设，以及硬件随机源（RN..."
date: 2026-11-17
permalink: /android-kernel-entropy-randomness/
---

> 面向 Android 应用开发者、系统安全工程师与设备厂商的防御性说明
> 版本：2026-11
> 适用范围：讨论随机数接口的语义、Android 启动期熵、历史缺陷与正确用法。**不包含**任何密钥窃取、预测攻击或利用代码。

---

## 0. 随机数错了，后面全错

加密密钥、会话令牌、nonce、盐值、设备标识与证书序列号都依赖随机数。随机数一旦可预测，攻击者不需要破解算法，直接猜出密钥即可。

Android 的随机数链路跨三层：

1. **应用层**：`java.security.SecureRandom`、`Kotlin Random`、`UUID.randomUUID()`、第三方库；
2. **系统层**：Bionic 的 `getrandom`/`/dev/urandom`、OpenSSL/BoringSSL 的 DRBG；
3. **内核层**：CRNG（密码学安全随机数生成器）、启动期种子、硬件 RNG。

任何一层误解都可能导致「看起来随机、实际不安全」。这篇手册把常见接口的语义、Android 启动期熵的来源，以及历史上真实发生过的缺陷分开讲。

---

## 1. 接口语义

### 1.1 getrandom(2)

- Linux 3.17 引入。
- 默认情况下，在 CRNG 完成初始化之前阻塞，初始化之后不阻塞。
- `GRND_NONBLOCK`：不阻塞；若 CRNG 未就绪，返回 `EAGAIN`。
- `GRND_INSECURE`：Linux 5.6 引入；不等待 CRNG 初始化，适合「不需要密码学强度」的场景。
- `GRND_RANDOM`：在现代内核中除参数校验外**没有实际分支**，uapi 注释明确写着「No effect」。它不会让结果「更强」。

### 1.2 /dev/urandom 与 /dev/random

- `/dev/urandom` 读取不阻塞，但在 CRNG 初始化之前可能返回质量不足的数据。
- `/dev/random` 在早期内核中会在熵不足时阻塞；现代内核的行为已经收敛，man 手册建议早期启动场景改用 `getrandom`。
- 不存在「urandom 一定不安全、random 一定安全」的简单二分。

### 1.3 应用层 API

- `java.security.SecureRandom`：Android 上默认从 `/dev/urandom` 取种，不需要应用手动 `setSeed`。手动播种通常是错误做法，可能降低熵而不是增加熵。
- `SecureRandom.getInstanceStrong()`：在 Android 上等价于 AndroidOpenSSL 提供的强算法实现，但它**不映射**为阻塞式 `/dev/random`。
- `java.util.Random`、`kotlin.random.Random`：不是密码学安全随机数，不能用于密钥、令牌或验证码。
- `UUID.randomUUID()`：基于 `SecureRandom` 生成 v4 UUID，适合标识符，但 UUID 本身不是密钥材料。
- 原生代码：优先 `getrandom`，其次 `/dev/urandom`；不要自行实现 PRNG。

---

## 2. Android 启动期熵

### 2.1 启动期为什么特殊

设备刚上电时，内核可能还没有收集到足够的不可预测事件。若此时生成磁盘加密密钥、生成设备证书或初始化 TLS，随机数质量就取决于启动链是否提供了足够种子。

### 2.2 KASLR 种子

启用 `RANDOMIZE_BASE` 时，bootloader 应通过设备树 `/chosen/kaslr-seed` 提供随机 u64，供内核用于地址空间布局随机化。这个种子也参与早期熵池。

### 2.3 信任 bootloader 的种子

`CONFIG_RANDOM_TRUST_BOOTLOADER`（Linux 5.4 引入）决定是否把 bootloader 提供的种子计入初始熵。启用它意味着系统信任 bootloader 的随机性；若 bootloader 实现质量差，这个信任会变成风险。

### 2.4 硬件 RNG

- GKI 配置设置 `CONFIG_HW_RANDOM=y`，但 `/dev/hwrng` 是否存在取决于具体硬件驱动。
- Armv8.5 的 `FEAT_RNG` 提供 `RNDR`/`RNDRRS` 指令；arm64 内核在支持时会使用它们，并通过 SMCCC TRNG 接口获取固件随机数。
- 硬件 RNG 的价值在于缩短启动期熵积累时间，但它不是「万能真随机」；仍需内核正确混合。

---

## 3. 公开案例：2013 年 Android SecureRandom 缺陷

2013 年 8 月，比特币钱包开发者发现部分 Android 钱包生成的密钥可被预测，导致资金被盗。随后 Google 发布说明。

### 3.1 已核实事实

- **CVE-2013-7372**：NVD 描述为 Android 4.4 之前的 Apache Harmony `SHA1PRNG_SecureRandomImpl.engineNextBytes` 存在缺陷，影响包括 4.0–4.3.1 在内的版本。
- 根因是 PRNG 初始化不当：在特定调用路径下，`SHA1PRNG` 的种子没有被正确混合，输出可被预测。
- Google 安全博客确认受影响的不仅是 `SecureRandom` 的 JCA 路径，直接调用 OpenSSL 的代码也可能受影响；Android 4.4 修复。
- bitcoin.org 于 2013-08-11 发布钱包安全公告。

### 3.2 教训

1. **「用了 SecureRandom」不等于安全。** 具体 provider、算法与初始化路径都重要。
2. **默认实现可能随版本变化。** 旧设备上的默认 provider 与新设备不同。
3. **不要把随机数缺陷当成应用层小问题。** 它直接影响密钥与资金。

### 3.3 与当前 Android 的关系

现代 Android 的 `SecureRandom` 默认走 `/dev/urandom`，上述 Harmony `SHA1PRNG` 缺陷已在新版本中修复。但旧设备、定制 ROM 与第三方运行时仍可能保留旧实现。兼容性测试与低版本设备审计时仍需关注。

---

## 4. 防护与检测

### 4.1 应用开发

- 密钥、令牌、验证码、盐值一律使用 `SecureRandom` 或平台密码学库的安全随机接口。
- **不要**调用 `setSeed`、不要用时间戳/设备 ID/用户 ID 作为种子。
- 不要用 `Random`、`Math.random()`、`System.currentTimeMillis()` 组合生成安全值。
- 需要长期密钥时，优先使用 Android Keystore 生成，而不是在应用层生成后自行保存。
- 对第三方库做依赖审计，确认其随机数来源。

### 4.2 系统与设备

- 保持 CRNG 初始化前的阻塞语义；不要在启动极早期用 `GRND_INSECURE` 生成密钥。
- 核对 bootloader 是否提供 `kaslr-seed`，以及 `CONFIG_RANDOM_TRUST_BOOTLOADER` 的取舍是否与威胁模型一致。
- 确认硬件 RNG 驱动存在且被内核使用；没有硬件 RNG 的设备应评估启动期熵积累策略。
- 对工厂测试与恢复模式中的密钥生成流程单独审计，这些环境常被忽略。

### 4.3 检测思路

```text
1. 读取 /proc/sys/kernel/random/entropy_avail（若可读）与池大小
2. 检查 /dev/hwrng 是否存在及权限
3. 核对内核配置中的 CONFIG_RANDOM_TRUST_BOOTLOADER 与 CONFIG_HW_RANDOM
4. 检查 bootloader 是否传入 kaslr-seed
5. 审计应用是否调用 setSeed 或使用非安全 Random
6. 对旧版本设备核对 SecureRandom provider 实现
```

---

## 5. 常见误传纠正

1. **「/dev/random 永远比 /dev/urandom 安全」**——现代内核中两者的阻塞行为已经收敛；man 手册建议早期启动使用 `getrandom`，并称 urandom 为首选来源之一。
2. **「getrandom 一定会阻塞」**——只在 CRNG 初始化前阻塞；初始化后默认不阻塞。
3. **「GRND_RANDOM 提供更强随机性」**——现代内核中它没有实际效果。
4. **「SecureRandom 默认就是密码学安全的，所以随便用」**——安全性取决于 provider、算法、初始化与版本；旧设备上曾有真实缺陷。
5. **「手动 setSeed 能增加安全性」**——通常相反；应用提供的种子往往低熵。
6. **「UUID 可以当密钥」**——UUID v4 适合标识，不是为密钥材料设计的。

---

## 6. 未核实项

- GKI 是否设置 `CONFIG_RANDOM_TRUST_BOOTLOADER`；defconfig 中未见明确条目。
- 具体设备是否存在 `/dev/hwrng` 节点，以及是否把 `RNDR` 作为应用可见熵源。
- Android 是否强制应用使用某一特定 `SecureRandom` provider；官方仅作推荐。
- 各厂商 bootloader 的 `kaslr-seed` 实现质量与种子的真实熵。
- 旧版本定制 ROM 中 `SHA1PRNG` 修复的回移情况。

以上项目需在目标设备与固件上核实，不得作为通用结论。

---

## 7. 参考资料

- getrandom 手册：<https://man7.org/linux/man-pages/man2/getrandom.2.html>
- random(4) 手册：<https://man7.org/linux/man-pages/man4/random.4.html>
- Linux 5.6 发布说明：<https://kernelnewbies.org/Linux_5.6>
- 内核 uapi random.h：<https://raw.githubusercontent.com/torvalds/linux/master/include/uapi/linux/random.h>
- CVE-2013-7372：<https://nvd.nist.gov/vuln/detail/CVE-2013-7372>
- Google 安全博客：<https://android-developers.googleblog.com/2013/08/some-securerandom-thoughts.html>
- bitcoin.org 2013-08-11 公告：<https://bitcoin.org/en/alert/2013-08-11-android>
- Android 启动链文档：<https://source.android.com/docs/core/architecture/bootloader>
- arm64 archrandom 实现：<https://raw.githubusercontent.com/torvalds/linux/master/arch/arm64/include/asm/archrandom.h>
- Android 弱 PRNG 风险说明：<https://developer.android.com/privacy-and-security/risks/weak-prng>

---

## 联系方式

咨询详情请联系 https://t.me/one00190

## 免责声明

本手册面向防御与合规用途，仅讨论 Android 随机数接口、启动期熵、历史缺陷与正确用法。文中不包含密钥预测、窃取或利用代码。内核配置与设备实现会随版本变化，请以官方文档与实测为准；标注为「未核实」的内容不得作为安全结论的依据。
