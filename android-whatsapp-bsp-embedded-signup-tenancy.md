---
layout: page
title: "WhatsApp BSP 与 Embedded Signup 多租户治理：一个应用服务一千个客户"
description: "把服务商接入拆成授权、令牌与回调三条隔离线：Embedded Signup v4 的 `config_id` 与 `code`（30 秒有效期）、postMessage 事件枚举、v2/v3 已于 2026-10-15 弃用；区分 System User token 与客户维度 token 的泄露影响面..."
date: 2026-11-11
permalink: /android-whatsapp-bsp-embedded-signup-tenancy/
---

当群发规模超过单商家自己接 API 的能力，业务就会落到 BSP（Business Solution Provider）或技术提供商手里。一个应用、一套回调地址、一群客户，所有消息都从同一扇门进出。门开得越大，一次配置错误或一个令牌泄露的代价就越高——受影响的不再是一个号码，而是平台上所有租户。

本文拆解 Embedded Signup 的接入流程、令牌模型、webhook 路由与多租户隔离、吞吐与配额，以及服务商侧的治理清单。**不讨论绕过平台审核、冒用他人资质或伪造入驻信息的方法**。

---

## 0. 使用方式与边界

- 事实以 Meta 官方文档中 Embedded Signup、访问令牌、解决方案提供商 webhook 管理与吞吐/消息限额页面为准，抓取时间为 2026 年 10 月。
- 官方未载明的兼容矩阵标注「未核实」。
- 面向 BSP、代运营、企业 IT 与平台风控，只讨论隔离、审计与防护。

---

## 1. 三个角色与信任边界

- **客户（商家）**：拥有 WABA、号码与商业资产；
- **BSP / 技术提供商**：持有应用、集成接入、代发消息；
- **Meta**：托管 Cloud API 与审核。

信任边界有三处：客户授权服务商的范围、服务商应用内的租户隔离、服务商到 Meta 的令牌权限。任何一处模糊，都会出现「A 商家能看见 B 商家消息」这类事故。

---

## 2. Embedded Signup 流程（v4）

当前版本的关键参数（已核实）：

- `FB.init({ appId })`；
- `FB.login` 传 `config_id`、`response_type='code'`、`override_default_response_type=true`，以及 `extras.setup`；
- `sessionInfoVersion` 只在 v2 使用；`featureType` / `features` 只在 v2/v3 可选；v4 中为可选或不再需要；
- 弹出窗通过 `postMessage` 回传，`type=WA_EMBEDDED_SIGNUP`；
- `event` 取值包括 `FINISH`、`FINISH_ONLY_WABA`、`FINISH_WHATSAPP_BUSINESS_APP_ONBOARDING`、`FINISH_GRANT_ONLY_API_ACCESS`、`CANCEL`、`ERROR`；
- `data` 中可拿到 `waba_id`、`phone_number_id`、`business_id`、`catalog_ids`；
- 授权码 `code` 的有效期是 **30 秒**；
- v2 与 v3 已于 **2026-10-15** 弃用（已核实）。

30 秒的有效期意味着换取令牌必须在客户端回调后立刻发起，且要有失败重试与用户重走的路径。把 code 缓存到队列里稍后处理，几乎必然失败。

---

## 3. 令牌模型

- **System User token**：长期有效，可管理多个资产，适合服务商后台的批量运维；
- **Business Integration System User token**：用 Embedded Signup 的 code 换取，按客户隔离，是代发消息的推荐形态；
- **临时 user token**：有效期数小时，只适合调试。

换取方式（已核实）：

```
GET /oauth/access_token?client_id=...&client_secret=...&code=...
```

返回 `access_token` 与 `token_type=bearer`；官方页面没有 `grant_type` 与 `expires_in` 字段。

治理上最重要的一条：**面向客户的发送必须用客户维度的 token，而不是服务商的全局 System User token**。全局 token 一旦泄露，攻击者可以操作平台上所有客户的号码；客户维度 token 泄露，影响面被限制在单个租户。

---

## 4. Webhook 路由与租户隔离

Cloud API 的 webhook 默认是**应用级**的：所有订阅 WABA 的事件都发往应用配置的回调地址（已核实）。订阅按 WABA 进行：

- `POST /{WABA_ID}/subscribed_apps` 订阅；
- `GET` 查询当前订阅；
- `DELETE` 退订。

路由依据是回调体中的 `entry[].id`（WABA）与 `metadata` 中的 `phone_number_id`；官方也支持按客户覆盖 callback URL（已核实）。

多租户实现的最低要求：

1. 回调入口先验签，再解析；
2. 按 `phone_number_id` 查租户映射，映射缺失时进入隔离队列而不是默认租户；
3. 每个租户的消息写入独立分区或独立表，禁止跨租户查询；
4. 覆盖 callback URL 的客户使用独立密钥与独立队列；
5. 退订事件必须同步清理路由表，避免号码转手后消息继续落到旧租户。

---

## 5. 吞吐与配额

已核实的要点：

- 默认吞吐 **80 条/秒/号码**，满足条件后可升至 **1,000 条/秒**（无限额档、24 小时内 10 万独立用户、质量不低于 YELLOW）；
- 与 WhatsApp Business App 共用的号码吞吐为 **20 条/秒**；
- 超限错误码 `130429`，升级过程中的限流错误码 `131057`；
- 消息限额档位为 `TIER_250`、`TIER_2K`、`TIER_10K`、`TIER_100K`、`TIER_UNLIMITED`，按 24 小时独立用户计数。

对服务商的含义：吞吐是**按号码**而非按应用计算的，一个租户的突发流量不会直接挤占另一个租户，但共享基础设施（队列、数据库、发送 worker）会。容量规划必须按租户做配额，而不是按全局总量做平均。

---

## 6. 多租户风险面

| 风险 | 触发条件 | 影响 | 防护 |
|---|---|---|---|
| 全局令牌泄露 | System User token 落在代码/镜像/日志 | 全平台号码被接管 | 客户维度 token、密钥托管、最小权限 |
| webhook 串租户 | 仅按号码路由且映射可被覆盖 | 消息落入错误租户 | 双键校验（WABA + 号码）、映射变更审计 |
| 号码转手残留 | 客户退租未清理订阅与路由 | 新机主消息发给旧商家 | 退订联动、定期对账 |
| 入驻信息伪造 | 代运营代填资质 | 违规主体进入平台 | 实名核验、授权留存、抽检 |
| 共享应用秘钥 | 所有租户共用同一 App Secret | 一处泄露全平台可伪造回调 | 按客户覆盖回调与密钥 |
| 配额争抢 | 单租户刷满共享 worker | 其他租户延迟 | 按租户限流与队列隔离 |
| 审计缺失 | 无操作日志 | 事故无法定责 | 令牌使用、配置变更、导出全留痕 |

---

## 7. 客户侧验收清单

选择 BSP 时，至少确认以下问题有明确答案：

- 发送用的是客户维度令牌还是服务商全局令牌？
- webhook 是否验签，转发层是否会跳过？
- 客户数据存放在哪个区域，保留多久，退租后如何删除？
- 谁可以导出会话与客户名单，导出是否审批？
- 号码迁移或终止合作时，WABA 与号码能否完整交还？
- 是否有独立的审计日志可供客户查阅？
- 出现串租户事件时的通知时限与赔偿条款？

这些问题比「价格多少、通道稳不稳」更能决定长期风险。

---

## 8. 常见误传纠正

- **「Embedded Signup 返回的 code 可以稍后兑换」**：有效期 30 秒。
- **「v2/v3 仍然可用」**：官方已于 2026-10-15 弃用。
- **「webhook 可以按 WABA 天然隔离」**：默认是应用级回调，隔离要靠订阅与路由设计。
- **「吞吐是按应用算的」**：默认 80 条/秒是按号码计。
- **「System User token 权限越大越省事」**：权限越大，泄露影响面越大。
- **「客户退租后号码自动解绑」**：订阅与路由需要显式清理。

---

## 9. 未核实与待观察

- Embedded Signup 各版本对 `featureType` / `features` 的完整兼容矩阵，未核实。
- `state` 参数是否在 v4 支持，官方页面未载。
- 各租户维度 token 的轮换周期与失效行为，未核实到统一说明。
- 号码质量枚举的完整取值，官方抓取页面只见到 YELLOW。

---

## 参考资料

- Meta：Embedded Signup 实现
  https://developers.facebook.com/documentation/business-messaging/whatsapp/embedded-signup/implementation
- Meta：Embedded Signup 版本
  https://developers.facebook.com/documentation/business-messaging/whatsapp/embedded-signup/versions
- Meta：访问令牌
  https://developers.facebook.com/documentation/business-messaging/whatsapp/access-tokens
- Meta：解决方案提供商 webhook 管理
  https://developers.facebook.com/documentation/business-messaging/whatsapp/solution-providers/manage-webhooks
- Meta：吞吐
  https://developers.facebook.com/documentation/business-messaging/whatsapp/throughput
- Meta：消息限额
  https://developers.facebook.com/documentation/business-messaging/whatsapp/messaging-limits

---

## 联系方式

咨询 Android 系统详情请联系 https://t.me/one00190

---

*本文仅供安全学习与防御研究使用，不含可利用代码；所述行为请遵守当地法律与目标系统授权范围。文中事实以官方文档与公告为准，无法核实之处已单列说明。*
