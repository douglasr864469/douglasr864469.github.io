---
layout: page
title: "WhatsApp Flows 数据交换安全：嵌入式表单的加密、鉴权与滥用边界"
description: "把嵌入式表单拆成消息下发、客户端渲染、数据回传三段信任边界：`flow_action` 只有 `navigate` 与 `data_exchange`（没有 `complete`）、健康检查是 POST ping、请求体用 AES-128-GCM 加密并以 RSA-OAEP-SHA256 包裹密钥、响应..."
date: 2026-11-11
permalink: /android-whatsapp-flows-data-exchange-security/
---

群发消息的竞争点正在从「发得出去」转向「收得回来」。一条模板消息只能带来点击，而 Flows（嵌入式表单）让用户在不离开聊天的前提下填写预约、收集意向、完成问卷、选择门店。对做规模化触达的团队来说，Flows 是最容易收集结构化数据的入口；对攻击者来说，它也是最容易伪造、重放与越权的地方。

本文从防御视角拆解 Flows 的消息结构、数据交换端点、加密与公钥管理、鉴权与完整性，以及围绕它的常见误解。**不提供绕过平台限制、伪造签名或批量刷表单的方法**。

---

## 0. 使用方式与边界

- 事实以 Meta 官方文档（developers.facebook.com）中 Flows 的发送指南、端点实现指南与 Flow JSON 参考为准，抓取时间为 2026 年 10 月。
- 文中「已核实」指官方页面有明确表述；官方未载明的数值与行为标注「未核实」，不凭经验补全。
- 面向企业开发、平台治理与安全评估人员，只讨论防护与审计方法。

---

## 1. Flows 在群发链路中的位置

一条带 Flow 的消息仍然是 `interactive` 类型消息，区别在于用户点击按钮后不会跳转浏览器，而是由 WhatsApp 客户端渲染表单，表单中的每一步交互以加密请求发回企业的数据交换端点（data exchange endpoint）。

这条链路有三个信任边界：

1. **消息下发**：企业 → Cloud API → 用户，受模板审核与限额约束；
2. **表单渲染**：Flow JSON 由企业定义、Meta 托管，客户端按 JSON 渲染；
3. **数据回传**：客户端 → 企业端点，靠公钥加密保证只有企业能读。

群发规模越大，第三段越关键：端点一旦被伪造请求打穿，收集到的名单、手机号、身份证件就会直接进入攻击者手里。

---

## 2. 消息结构与关键字段

发送 Flow 的消息结构（已核实）：

- `type=interactive`，`interactive.type=flow`，`action.name=flow`；
- 参数包含 `flow_message_version="3"`、`flow_id` 或 `flow_name`、`flow_token`、`flow_cta`、`flow_action`、`flow_action_payload`、`mode`；
- `flow_action` 取 `navigate` 或 `data_exchange`（不存在 `complete` 这个动作值，常见误传）；
- `flow_action_payload` 含起始屏 `screen` 与初始 `data`；
- `mode` 取 `published` 或 `draft`，生产环境只能用 `published`；
- `flow_cta` 上限 30 个字符。

`flow_token` 由企业生成，官方默认值是 `"unused"`。它的作用是**把一次表单会话与业务侧的一次交互关联起来**，用于判重、续填与幂等。官方没有给出 TTL，这意味着有效期必须由企业在服务端自己定义——不定义就等于允许无限期重放。

---

## 3. 端点握手：是 POST，不是 GET

一个广泛流传的实现错误是把健康检查写成 GET。官方要求数据交换端点对如下 JSON 的 **POST** 请求返回 `{"data":{"status":"active"}}`：

```json
{"version":"3.0","action":"ping"}
```

握手失败会直接影响 Flow 的发布与运行。对安全评估来说，这一步的意义是：端点必须区分「元数据请求」与「数据请求」，且不能在 ping 响应里泄露任何业务信息（如租户列表、数据库版本、内部错误栈）。

---

## 4. 加密：RSA-OAEP 包 AES-GCM

Flow 数据请求体包含三个字段（已核实）：

- `encrypted_flow_data`：AES-128-GCM 密文，认证标签 128 位附在末尾；
- `encrypted_aes_key`：用企业 RSA 公钥以 RSA-OAEP-SHA256（2048 位）加密的 AES 密钥；
- `initial_vector`：128 位初始向量。

响应规则有一个容易踩的细节：**响应的 IV 是把请求 IV 逐字节取反（0xFF 异或）**，附加认证数据（AAD）为空，输出为 base64 的「密文 + 标签」，Content-Type 为 `text/plain`。解密失败时返回 HTTP 421。

这套方案本身没有问题，问题出在实现：

- 复用 IV 或把 IV 当常量写死，会让 AES-GCM 的认证性失效；
- 把解密异常统一吞掉并返回 200，会让重放与伪造请求难以被发现；
- 把明文请求体写进日志（包括用户填写的手机号、证件号），等于绕过了整套加密设计；
- RSA 私钥放在应用代码仓库或镜像里，公钥轮换就失去意义。

---

## 5. 公钥登记与轮换

企业公钥通过 `POST /{PHONE_NUMBER_ID}/whatsapp_business_encryption` 上传，`GET` 同路径可查询状态，返回 `signature_status` 为 `VALID` 或 `MISMATCH`。该操作需要 system user token 与 `whatsapp_business_messaging` 权限（已核实）。

治理要点：

- 公钥与私钥必须成对登记、成对轮换，**先登记新公钥、再切换端点解密、最后下线旧私钥**；
- 定期核对 `signature_status`，出现 `MISMATCH` 意味着客户端加密所用公钥与企业实际私钥不一致，所有 Flow 提交都会失败——这既可能是运维事故，也可能是有人替换了公钥；
- 私钥访问需要审计：谁导出过、何时用于解密、是否存在离线副本；
- 单一公钥服务多个租户时，密钥泄露的影响面是全体客户，必须按租户隔离密钥或至少隔离解密服务。

---

## 6. 鉴权与完整性：别把 webhook 的签名头套在端点上

Flow 数据交换端点的请求体是加密的，官方文档没有为它列出任何专用签名头。常被误用的 `X-Hub-Signature-256` 是 **webhook 回调**的签名机制（基于 App Secret 的 SHA256），不是 Flow 端点的鉴权方式（已核实）。

Flow 7.3 与 `data_api_version` 4.0 及以上，请求中会带 `flow_token_signature`（JWT），可用于校验会话来源与完整性；更早版本没有这个字段，端点的安全性完全依赖加密与 `flow_token` 的业务校验。

因此端点的最低防护应包括：

1. 只接受 HTTPS，拒绝任何明文回退；
2. 校验 `flow_token` 是否由本企业签发、是否已使用、是否过期；
3. 对同一 `flow_token` 的并发提交做幂等处理；
4. 限制请求体大小与解密失败速率，防止填充攻击与资源耗尽；
5. 对新版本强制校验 `flow_token_signature`，并验证签发者与受众。

---

## 7. 风险面清单

| 风险 | 触发条件 | 影响 | 防护 |
|---|---|---|---|
| 会话重放 | `flow_token` 无 TTL、无使用记录 | 重复提交、重复发货、重复发券 | 服务端一次性令牌 + 幂等键 |
| 公钥替换 | 私钥泄露或登记接口被越权调用 | 攻击者可解密全部表单数据 | 密钥隔离、变更告警、双人审批 |
| 端点探测 | ping 响应含内部信息 | 暴露架构与租户信息 | 最小化响应、统一错误页 |
| 明文落库 | 解密后直接写日志/监控 | 敏感数据绕过加密设计 | 字段级脱敏、日志白名单 |
| 表单内容注入 | 用户输入直接拼进下游模板/工单 | 存储型 XSS、指令注入 | 输出编码、长度与字符集限制 |
| 大体积载荷 | 单次请求接近 10 MB | 内存耗尽、拖慢端点 | 请求体上限、分片校验 |
| 草稿模式误发 | `mode=draft` 进入生产 | 未审核表单对外 | 环境隔离、发布前检查 |

补充一个常被忽视的点：Flow JSON 本身最大 10 MB，路由分支最多 10 个，数据通道 payload 上限 10 MB（已核实）。屏幕数与组件数的官方上限未见明确文档（未核实），不要以第三方博客的数字做容量规划。

---

## 8. 企业侧防护清单

- **数据最小化**：表单只收业务必需字段，证件号、住址等敏感项默认不收；
- **分层存储**：解密后的数据按敏感级别分库，保留期与访问审批绑定；
- **审计**：记录每次解密的时间、端点实例、`flow_token`、结果码，不记录明文；
- **密钥生命周期**：轮换计划、备份策略、销毁流程写入 runbook；
- **异常监测**：解密失败率、ping 频率、同一 token 重复提交、单 IP 高并发；
- **供应链**：托管 Flow 端点的第三方同样要过安全评估，不能因为「是 Meta 的加密」就免检；
- **合规**：跨境传输、告知同意、未成年人数据按所在辖区单独处理。

---

## 9. 常见误传纠正

- **「健康检查是 GET」**：官方要求 POST，请求体带 `action: "ping"`。
- **「`flow_action` 可以是 `complete`」**：只有 `navigate` 与 `data_exchange`。
- **「Flow 端点靠 X-Hub-Signature-256 验签」**：那是 webhook 的机制；端点的完整性依赖加密与 `flow_token`（新版本另有 JWT）。
- **「加密了就不用管令牌」**：加密解决机密性，不解决重放；`flow_token` 必须由企业做一次性校验。
- **「公钥上传一次就永久有效」**：公钥可轮换，且必须监控 `signature_status`。
- **「Flow 表单数据可以直接进群发名单」**：收集目的与后续营销用途必须分别取得同意，否则就是把合规表单变成违规名单。

---

## 10. 未核实与待观察

- Flow 屏幕数、组件数的官方上限，以及数据端点的超时时间，官方文档未明确载明。
- `flow_token` 的官方建议 TTL 未给出。
- 数据交换端点是否存在未公开的专用签名头，未核实。
- 不同 `data_api_version` 对 JWT 字段的兼容矩阵，未核实到完整表格。

---

## 参考资料

- Meta：Flows 发送指南（Sending a Flow）
  https://developers.facebook.com/documentation/business-messaging/whatsapp/flows/guides/sendingaflow
- Meta：实现 Flow 端点（Implementing your Flow endpoint）
  https://developers.facebook.com/documentation/business-messaging/whatsapp/flows/guides/implementingyourflowendpoint
- Meta：WhatsApp Business Encryption（公钥登记）
  https://developers.facebook.com/documentation/business-messaging/whatsapp/flows/guides/whatsapp-business-encryption
- Meta：Flow JSON 参考
  https://developers.facebook.com/documentation/business-messaging/whatsapp/flows/reference/flowjson
- Meta：Webhook 签名（X-Hub-Signature-256）
  https://developers.facebook.com/docs/graph-api/webhooks/getting-started

---

## 联系方式

咨询 Android 系统详情请联系 https://t.me/one00190

---

*本文仅供安全学习与防御研究使用，不含可利用代码；所述行为请遵守当地法律与目标系统授权范围。文中事实以官方文档与公告为准，无法核实之处已单列说明。*
