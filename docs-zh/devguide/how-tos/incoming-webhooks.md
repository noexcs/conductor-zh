---
description: 配置传入 Webhook 以验证 HTTP 回调、启动工作流、恢复 WAIT_FOR_WEBHOOK 任务，或两者兼做。
---

# 传入 Webhook

<section class="concept-hero concept-hero--event-bus" aria-label="传入 Webhook">
  <div class="concept-hero__content">
    <p>一个<strong> Webhook</strong>是外部服务在其一侧发生事件时调用的 HTTP 端点。Conductor 首先验证调用方的签名，持久化地记录该投递，然后要么启动一个新工作流，要么恢复一个正等待 <code>WAIT_FOR_WEBHOOK</code> 的工作流。本页涵盖端点配置、验证以及两种投递模式。</p>
  </div>
  <svg class="concept-hero__graphic event-hero__graphic" viewBox="0 0 440 190" role="img" aria-labelledby="webhook-svg-title webhook-svg-desc" xmlns="http://www.w3.org/2000/svg">
    <title id="webhook-svg-title">传入 Webhook 处理流程</title>
    <desc id="webhook-svg-desc">外部提供商向传入 Webhook 发送 HTTP 回调。经过验证和持久化处理后，配置可以启动工作流、恢复 Wait for Webhook 任务，或两者兼做。</desc>
    <defs><marker id="webhook-arrow" markerWidth="8" markerHeight="8" refX="7" refY="4" orient="auto"><path d="M0,0 L8,4 L0,8 z" fill="currentColor"/></marker></defs>
    <rect x="14" y="68" width="98" height="54" rx="10" class="concept-hero__node event-hero__node--broker"/><text x="63" y="91" text-anchor="middle" class="concept-hero__label">提供商</text><text x="63" y="108" text-anchor="middle" class="concept-hero__detail">HTTP 回调</text>
    <path d="M112 95 H148" class="concept-hero__line" marker-end="url(#webhook-arrow)"/>
    <rect x="156" y="58" width="119" height="74" rx="10" class="concept-hero__node concept-hero__node--accent"/><text x="216" y="83" text-anchor="middle" class="concept-hero__label">传入 Webhook</text><text x="216" y="100" text-anchor="middle" class="concept-hero__detail">验证 + 持久化</text><text x="216" y="117" text-anchor="middle" class="concept-hero__detail">配置的投递</text>
    <path d="M275 81 H312" class="concept-hero__line" marker-end="url(#webhook-arrow)"/>
    <path d="M275 109 H295 V146 H312" class="concept-hero__line" marker-end="url(#webhook-arrow)"/>
    <rect x="320" y="55" width="106" height="44" rx="10" class="concept-hero__node event-hero__node--action"/><text x="373" y="82" text-anchor="middle" class="concept-hero__label">启动工作流</text>
    <rect x="320" y="124" width="106" height="44" rx="10" class="concept-hero__node event-hero__node--action"/><text x="373" y="145" text-anchor="middle" class="concept-hero__label">恢复 WAIT</text><text x="373" y="160" text-anchor="middle" class="concept-hero__detail">FOR_WEBHOOK</text>
  </svg>
</section>

## 端点与生命周期

Webhook 投递使用以下相对于 Conductor API 基础 URL 的路由：

| 方法 | 路由 | 用途 |
|---|---|---|
| `POST` | `/webhook/{id}` | 接收回调体、查询参数和请求头 |
| `GET` | `/webhook/{id}` | 处理提供商的 URL 验证或 ping 请求 |
| `POST` | `/metadata/webhook` | 创建 Webhook 配置 |
| `GET` | `/metadata/webhook` | 列出配置 |
| `GET` | `/metadata/webhook/{id}` | 读取一个配置 |
| `PUT` | `/metadata/webhook/{id}` | 更新一个配置 |
| `DELETE` | `/metadata/webhook/{id}` | 删除一个配置 |

例如，如果 API 基础 URL 是 `https://conductor.example.com/api`，就给提供商 `https://conductor.example.com/api/webhook/<webhook-id>`。

传入请求在被接受处理之前会先经过验证。记录下来的事件和队列使投递在工作者重启之后仍然持久；随后处理会评估配置、启动所配置的工作流，并匹配符合条件的 `WAIT_FOR_WEBHOOK` 任务。诊断投递问题时，检查 Webhook/事件记录以及由此产生的工作流或任务状态。

## 选择投递模式

Webhook 配置可以对一次经验证的回调应用以下一种或两种效果：

- **启动：** 启动每个配置的接收工作流。
- **恢复：** 匹配并推进符合条件的 `WAIT_FOR_WEBHOOK` 任务。
- **两者：** 从同一次持久化回调中启动所配置的工作流并恢复匹配的等待。

根据你需要创建或推进的状态来选择模式；Webhook 不是事件处理器的动作分发器。

## 在不暴露密钥的情况下配置

配置指明验证器、可选的期望请求头、接收工作流的版本或要启动的工作流，以及匹配行为。把验证器材料放在平台密钥存储中并引用它；永远不要把签名密钥、HMAC 密钥或私钥的字面值放进工作流或文档示例中。

```json
{
  "name": "payment-provider-callback",
  "sourcePlatform": "Custom",
  "verifier": "HMAC_BASED",
  "headerKey": "X-Provider-Signature",
  "secretValue": "${workflow.secrets.PAYMENT_WEBHOOK_SECRET}",
  "receiverWorkflowNamesToVersions": {
    "process_payment_callback": 1
  }
}
```

使用你的环境支持的密钥引用形式，而不是把真实密钥复制进配置。回调的载荷和请求头同样要当作潜在敏感数据处理。

## 验证器选择

| 验证器 | 验证输入 | GET 质询 / ping 行为 |
|---|---|---|
| `HEADER_BASED` | 每个配置的请求头必须恰好出现一次且等于其配置值。 | 没有提供商质询行为。 |
| `SIGNATURE_BASED` | 配置的请求头包含 `sha256=` 加上用配置的密钥对原始请求体计算的 HMAC-SHA-256。 | 没有提供商质询行为。 |
| `HMAC_BASED` | 配置的请求头携带原始请求体的 HMAC-SHA-256；配置的密钥在验证前会先做 Base64 解码。 | 没有提供商质询行为。 |
| `SLACK_BASED` | `X-Slack-Signature`、`X-Slack-Request-Timestamp` 和原始请求体；时间戳会做重放窗口检查。 | URL 验证期间返回 Slack 的 JSON `challenge` 值。 |
| `STRIPE` | `Stripe-Signature`、原始请求体以及 Stripe 签名密钥。 | 没有提供商质询行为。 |
| `TWITTER` | 配置的签名请求头和原始请求体，使用 Twitter HMAC 编码。 | 对 `crc_token`，返回用配置的密钥签名的 `response_token`。 |
| `SENDGRID` | SendGrid 事件 Webhook 的签名和时间戳请求头、原始请求体，以及配置的 ECDSA 公钥。 | 没有提供商质询行为。 |

验证是一道安全边界，不是针对任意工作流动作的授权模型。把每个 Webhook 配置限制在它真正需要的工作流和任务匹配范围内。

## Webhook 与事件处理器

[事件处理器](consume-route-events.md) 订阅 broker 事件，可以分发其文档中记载的动作。传入 Webhook 接收 HTTP，只执行 Webhook 配置中的启动工作流和 `WAIT_FOR_WEBHOOK` 匹配行为。不要把 Webhook 建模为调用 `complete_task`、`fail_task`、`terminate_workflow` 或 `update_workflow_variables` 动作的方式。

如果是 broker 消息而非 HTTP 回调，请使用[消费与路由事件](consume-route-events.md)。要直接完成已知工作流中当前的 `WAIT`，请使用[向工作流发送信号](../cookbook/sending-signals.md)。

## 后续步骤

<div class="event-next-steps">
  <a href="consume-route-events.html">路由 broker 事件 →</a>
  <a href="../cookbook/sending-signals.html">向 WAIT 任务发送信号 →</a>
  <a href="event-bus.html">返回概览 →</a>
</div>
