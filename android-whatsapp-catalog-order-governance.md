---
layout: page
title: "WhatsApp 商业目录、商品消息与订单治理：群发里的「货」不能随手拼"
description: "讲清 `catalog_id`、`retailer_id` 与 `product_retailer_id` 的关系，单品/多品/目录三种消息的结构差异（多品最多 30 个商品、单品没有 `action.name`），入站 `order` 对象的字段与「通用场景没有 `order_status` 回执」这..."
date: 2026-11-11
permalink: /android-whatsapp-catalog-order-governance/
---

当群发从「发一段文案」升级为「发一件商品」，问题就从文案质量变成了**目录一致性**：同一件商品在目录、消息、订单回执与库存系统里有四个不同的标识，任何一个环节对不上，用户看到的就不是你实际要卖的东西。规模化发送会把这种错配放大到几千个会话，退款、投诉与封号随之而来。

本文梳理 WhatsApp 商业目录与商品消息的标识体系、消息结构、入站订单对象、订单状态与支付边界，以及围绕目录的滥用与治理。**不讨论刷单、伪造订单或规避商业政策的方法**。

---

## 0. 使用方式与边界

- 事实以 Meta 官方文档中目录（Catalogs）、单品/多品/目录消息、订单 webhook 与 Payments 相关页面为准，抓取时间为 2026 年 10 月。
- 官方未写明的上限与政策细节标注「未核实」。
- 面向商家、代运营、BSP 与风控人员，只讨论一致性与防护。

---

## 1. 四个标识，别混用

- `catalog_id`：商业目录的 ID，一个 WABA 可以关联一个目录；
- `retailer_id`：商品在目录内的唯一标识，通常对应商家的 SKU 或内容 ID；
- `product_retailer_id`：消息与订单对象中对某个商品的**引用**，值等于该商品的 `retailer_id`；
- `product_id`：Meta 侧的商品标识，一般由系统生成，不应被商家当作业务主键。

实际事故大多来自把 `product_id` 当成 SKU 使用，或者在换目录时沿用了旧 `retailer_id`。前者会导致对账失败，后者会导致用户下单买到错误商品。

---

## 2. 三种商品消息

**单品消息**（已核实）：`type=interactive`、`interactive.type=product`，`action` 中放 `catalog_id` 与 `product_retailer_id`。它没有 `action.name`，这是与目录消息最容易写错的差异。

**多品消息**（已核实）：`interactive.type=product_list`，`header` 必填，`action` 中含 `catalog_id` 与 `sections[]`；每个 section 有 `title` 与 `product_items`，单条消息最多 **30 个商品**。section 数量的官方上限未见明确文档（未核实）。

**目录消息**（已核实）：`interactive.type=catalog_message`，`action.name=catalog_message`，可选 `thumbnail_product_retailer_id`；正文上限 1024 字符，页脚上限 60 字符，打开后展示整个目录。

群发场景里三种消息的取舍：

- 单品适合「已知用户意向」的追加销售，转化高但覆盖面窄；
- 多品适合促销清单，必须控制商品数，超过 30 个只能拆消息或改用目录消息；
- 目录消息适合「让用户自己逛」，但转化路径长，且目录内容变更会即时影响已发消息的展示。

---

## 3. 入站订单对象

用户在多品消息里完成选择后，企业会收到 `order` 类型的 webhook。订单对象包含（已核实）：

- `catalog_id`；
- `text`：用户附言；
- `product_items[]`：每项含 `product_retailer_id`、`quantity`、`item_price`、`currency`。

注意两件事：**订单里的价格是用户下单时的价格**，不是目录当前价格；**没有通用订单状态回执类型**。Meta 并未提供一个跨市场的 `order_status` webhook 让商家回传发货状态（已核实）。巴西的 Payments 场景另有 `order_details` 与 `order_status`，其订单状态枚举为 `pending`、`processing`、`partially_shipped`、`shipped`、`completed`、`canceled`，支付状态为 `pending`、`captured`、`failed`（已核实）。把巴西的字段当成全球通用能力，是集成阶段常见的误判。

---

## 4. 价格、库存与目录漂移

群发最典型的翻车路径是：活动文案按旧价撰写 → 目录已调价 → 用户点开商品看到新价 → 下单金额与文案不符 → 投诉。

治理上要把「文案、目录、订单、库存」当成一个事务处理：

1. 发送前对参与活动的 `product_retailer_id` 做一次快照，记录价格与库存；
2. 文案中的价格由快照生成，而不是人工填写；
3. 订单回调到达时，与快照比对，不一致则进入人工复核而不是直接发货；
4. 活动期间禁止批量改价；如必须改，先下线相关消息；
5. 目录删除商品前，确认没有在途会话引用该 `retailer_id`。

---

## 5. 滥用面

**目录投毒**：代运营或第三方工具获得目录写权限后，批量替换商品图片、标题与链接，把流量导向站外钓鱼页或仿品。防护是目录变更双人审批与变更日志。

**商品消息钓鱼**：伪造「订单待支付」消息，附上站外收款链接。WhatsApp 内的商品消息本身不携带支付链接，任何要求跳转到陌生域名付款的行为都应视为高风险。

**订单对象伪造**：向企业自建回调地址直接 POST 伪造的 `order` 对象。Cloud API 的 webhook 有 `X-Hub-Signature-256`，自建中间层若未验签就转发，会让伪造订单进入 ERP。

**禁售品流转**：Commerce Policies 列出武器、处方药、仿品、赌博、活体动物等禁售类别（已核实其存在，具体 24 类清单以官方页面为准）。群发内容与目录内容都受约束，换一个 `retailer_id` 并不会让禁售品变得合规。

**跨租户串目录**：BSP 用一个应用服务多个商家时，如果按 `phone_number_id` 路由却没校验 `catalog_id` 归属，A 商家的订单可能落进 B 商家的系统。

---

## 6. 治理清单

- 建立 `retailer_id ↔ SKU ↔ 目录 ↔ 仓库` 的映射表，变更走审批；
- 目录写权限只授予必要角色，代运营使用独立账号并留痕；
- webhook 一律验签，自建转发层不得跳过；
- 订单回调做幂等（同一订单重复推送不得重复发货）；
- 价格快照与订单金额自动比对，异常进人工队列；
- 禁售品关键词与类目双重扫描，覆盖文案与目录；
- 群发前预检：商品数量、section 结构、`catalog_id` 归属、库存状态；
- 保留发送批次与商品快照的关联，便于事后追溯。

---

## 7. 常见误传纠正

- **「多品消息可以放任意多个商品」**：单条 `product_list` 上限 30 个商品。
- **「订单里有价格，所以可以按订单价结算」**：订单价是下单时快照，仍需与目录、促销规则核对。
- **「商家可以回传发货状态给 WhatsApp」**：通用场景没有 `order_status` 回执；相关字段属于巴西 Payments。
- **「目录消息有 action.name 和单品消息一样」**：单品消息没有 `action.name`，目录消息有。
- **「换了目录 ID，旧 retailer_id 还能用」**：`retailer_id` 只在所属目录内唯一，换目录必须重建映射。
- **「商品链接是 WhatsApp 提供的，所以一定安全」**：商品消息本身不带站外支付链接，出现即为高风险。

---

## 8. 未核实与待观察

- 多品消息中 section 数量上限，官方文档未明确。
- Commerce Policies 的价格一致性与库存条款细节，抓取页面未完整呈现。
- 各国 Payments 的订单状态字段是否完全一致，未核实。
- 目录同步失败的告警机制与重试策略，官方文档未给出统一建议。

---

## 参考资料

- Meta：单品商品消息
  https://developers.facebook.com/documentation/business-messaging/whatsapp/catalogs/single-product-messages
- Meta：多品商品消息
  https://developers.facebook.com/documentation/business-messaging/whatsapp/catalogs/multi-product-messages
- Meta：目录消息
  https://developers.facebook.com/documentation/business-messaging/whatsapp/catalogs/catalog-messages
- Meta：订单 webhook 参考
  https://developers.facebook.com/documentation/business-messaging/whatsapp/webhooks/reference/messages/order
- Meta：巴西 Payments 订单状态
  https://developers.facebook.com/documentation/business-messaging/whatsapp/payments/payments-br/orders
- Meta：Commerce Policies
  https://www.facebook.com/policies_center/commerce

---

## 联系方式

咨询 Android 系统详情请联系 https://t.me/one00190

---

*本文仅供安全学习与防御研究使用，不含可利用代码；所述行为请遵守当地法律与目标系统授权范围。文中事实以官方文档与公告为准，无法核实之处已单列说明。*
