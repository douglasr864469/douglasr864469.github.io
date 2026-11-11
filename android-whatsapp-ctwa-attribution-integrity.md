---
layout: page
title: "Click-to-WhatsApp 归因与 referral 防伪：群发线索从哪来、能信几分"
description: "指出 `referral` 只出现在广告点击后的首条消息，逐字段说明 `source_url`/`source_id`/`source_type`/`media_type`/`ctwa_clid`（Status 广告位不返回 clid），并把 `ctwa_clid` 到 Conversions API ..."
date: 2026-11-11
permalink: /android-whatsapp-ctwa-attribution-integrity/
---

做 WhatsApp 群发的团队普遍关心「一条消息带来多少转化」，而广告侧的团队关心「哪个广告带来多少会话」。当两者用同一套数据说话，就会出现一个被反复忽略的问题：**首条消息里的 referral 字段只能证明「这条会话由某个广告点击开启」，不能证明用户是谁、也不能证明后续成交归属**。

本文拆解 Click-to-WhatsApp（CTWA）的 referral 结构、`ctwa_clid` 与 Conversions API 的对接方式、归因窗口与去重边界，以及围绕归因数据的伪造与串单风险。**不提供刷转化、伪造点击或规避平台计费的方法**。

---

## 0. 使用方式与边界

- 事实以 Meta 官方文档中 CTWA、referral 参考与 Business Messaging Conversions API 页面为准，抓取时间为 2026 年 10 月。
- 官方未给出的窗口与有效期标注「未核实」。
- 面向投放、客服系统与数据治理人员，只讨论可验证性与防护。

---

## 1. referral 在哪里出现

用户点击广告后打开 WhatsApp，发出的**第一条消息**会携带 `referral` 对象。它不是用户输入，而是客户端注入的元数据；后续消息不再重复携带。这意味着：

- 归因信息只存在于会话的第一个入口；
- 如果首条消息在企业系统里被丢弃、被合并或未被解析，归因永久丢失；
- 用户从广告进入后隔几天再主动发消息，新会话不再有 referral。

工程上必须把首条消息的 `referral` 与 `wa_id`、`message_id`、时间戳一起持久化，并在客服工单创建时写入不可变字段，而不是放在可编辑的备注里。

---

## 2. 字段清单与含义

已核实的 referral 字段：

| 字段 | 含义 | 备注 |
|---|---|---|
| `source_url` | 广告点击后跳转的 URL | 常含广告参数，需当作不可信输入处理 |
| `source_id` | 广告或帖子的 ID | 用于回查投放平台 |
| `source_type` | 来源类型，当前为 `ad` | 非广告入口不出现 |
| `headline` | 广告标题 | 展示用，不可用于业务判断 |
| `body` | 广告正文 | 同上 |
| `media_type` | `image` 或 `video` | 决定后续媒体字段 |
| `image_url` | 图片广告的素材地址 | 仅图片广告出现 |
| `video_url` | 视频广告的素材地址 | 仅视频广告出现 |
| `thumbnail_url` | 视频广告的缩略图 | 仅视频广告出现 |
| `ctwa_clid` | 点击 ID，用于回传转化 | WhatsApp Status 广告位不返回 |

`source_id_type` 与 `ref` 两个字段在一些第三方文档里被广泛引用，但官方 referral 参考页未列出（未核实），不要把它们写进对接契约。

---

## 3. ctwa_clid 与转化回传

`ctwa_clid` 要放在 Conversions API 的 `user_data` 中，与 `whatsapp_business_account_id` 同级（已核实）。对接要点：

- 先为 WABA 创建 dataset：`POST /{WABA_ID}/dataset`；
- 回传时 `action_source=business_messaging`、`messaging_channel=whatsapp`；
- 官方列出的业务消息事件共 14 个，包括 `Purchase`、`LeadSubmitted`、`InitiateCheckout`、`AddToCart`、`ViewContent`、`OrderCreated`、`OrderShipped`、`OrderDelivered`、`OrderCanceled`、`OrderReturned`、`CartAbandoned`、`QualifiedLead`、`RatingProvided`、`ReviewProvided`；
- 事件时间应与业务系统里的真实发生时间一致，不能用「回传时刻」。

一个常见错误是把「会话建立」当成 `LeadSubmitted` 回传。会话建立只说明有人点了广告，不说明留了线索；把两者混同会系统性高估广告效果，也会让后续的预算分配失真。

---

## 4. 归因窗口与去重

官方仅在**哈希客户信息**路径上明示「过去 7 天」的归因窗口；`ctwa_clid` 路径的窗口未给出数值（未核实）。因此：

- 不要把 7 天当成 `ctwa_clid` 的既成事实写进合同或报表口径；
- 报表应同时给出「点击后 1 天 / 7 天 / 30 天」的分窗口数据，让业务自己看到衰减；
- 跨窗口的重复成交必须去重。Meta 侧不负责去重，需要企业在回传前用业务单号或事件 ID 做幂等（官方未给出 `event_id` 字段要求，未核实）。

---

## 5. 风险面：referral 是「声明」，不是「证明」

**伪造 source_id**：攻击者可以向企业自建的回调地址发送带有任意 `referral` 的伪造 webhook。若中间层未验签，报表里就会出现不存在的广告点击。

**source_url 注入**：`source_url` 可能包含攻击者可控的查询参数。把它直接渲染到客服界面或拼进 SQL、模板、跳转链接，会造成 XSS、SQL 注入或开放重定向。

**归因串单**：BSP 用一个应用服务多个广告主时，如果按 `wa_id` 而不是按 `phone_number_id` + `ctwa_clid` 归属，A 广告主的转化会记到 B 头上。

**越权查询**：把 `ctwa_clid` 当作查询键开放给一线客服，等于把投放数据暴露给所有人；它应只在受控的报表服务中使用。

**隐私过度收集**：为了「提高归因准确率」而收集身份证、精确地址等非必要信息，违反数据最小化，也会让哈希匹配的合规基础变得脆弱。

---

## 6. 治理清单

- 首条消息的 referral 落库为不可变字段，附 `message_id` 与接收时间；
- webhook 必须验签（`X-Hub-Signature-256`），自建转发层不得跳过；
- `source_url`、`headline`、`body` 一律按不可信文本处理，输出编码、长度截断；
- 归因报表按 `phone_number_id` 分租户隔离，禁止跨租户聚合；
- 回传事件只使用真实业务事件，禁止把「会话建立」映射为线索或成交；
- 回传前做幂等，记录已回传的事件键；
- 对所有 `ctwa_clid` 的访问留审计日志；
- 在隐私政策中说明归因数据的用途、保留期与共享对象。

---

## 7. 常见误传纠正

- **「referral 会出现在每条消息里」**：只出现在广告点击后的首条消息。
- **「meta 会自动去重转化」**：官方未提供去重保证，重复回传会重复计数。
- **「ctwa_clid 在 Status 广告位也有」**：官方明确 Status 广告位不返回该字段。
- **「归因窗口统一是 7 天」**：7 天只出现在哈希客户信息路径，`ctwa_clid` 路径未给数值。
- **「referral 里的 headline/body 是可信的广告原文」**：它们是客户端注入的展示字段，必须当不可信输入。
- **「有了 ctwa_clid 就能证明用户身份」**：它标识一次点击，不标识自然人。

---

## 8. 未核实与待观察

- `ctwa_clid` 的有效期与官方归因窗口，未核实。
- Conversions API 是否接受业务侧自定义 `event_id` 做去重，未核实。
- `source_id_type` 与 `ref` 字段是否存在，官方参考页未列。
- 各广告位（Status、Reels、Feed）对 referral 字段的差异矩阵，未核实。

---

## 参考资料

- Meta：消息 webhook 中的 referral 字段
  https://developers.facebook.com/documentation/business-messaging/whatsapp/webhooks/reference/messages/text
- Meta：Business Messaging Conversions API
  https://developers.facebook.com/documentation/ads-commerce/conversions-api/business-messaging
- Meta：Webhook 签名
  https://developers.facebook.com/docs/graph-api/webhooks/getting-started

---

## 联系方式

咨询 Android 系统详情请联系 https://t.me/one00190

---

*本文仅供安全学习与防御研究使用，不含可利用代码；所述行为请遵守当地法律与目标系统授权范围。文中事实以官方文档与公告为准，无法核实之处已单列说明。*
