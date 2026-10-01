---
description: "针对常见编排问题的完整、可运行的 Conductor 工作流定义：并行、Saga、定时器、事件以及 AI 智能体模式。"
---

# 设计模式

<section class="concept-hero concept-hero--cookbook">
  <div class="concept-hero__content">
    <p>设计模式是针对常见编排问题的完整、可运行的工作流定义。每一页聚焦一个问题，例如并行扇出、Saga、定时器或人工审批，并给出一个可注册、可运行、可适配到你自己的任务中的可用定义。本部分涵盖工作流模式。Agentic 模式与智能体配方位于 <a href="../ai/cookbook/index.html">AI Cookbook</a>。</p>
  </div>
  <svg class="concept-hero__graphic cookbook-hero__graphic" viewBox="0 0 440 205" role="img" aria-label="Cookbook recipes for services, parallel work, timers, events, AI, and code flow into a durable Conductor workflow">
    <defs><marker id="cookbook-arrow" markerWidth="8" markerHeight="8" refX="7" refY="4" orient="auto"><path d="M0,0 L8,4 L0,8 Z" fill="currentColor" /></marker></defs>
    <g class="cookbook-hero__sources">
      <rect x="18" y="18" width="100" height="38" rx="9" /><text x="68" y="42" text-anchor="middle">服务</text>
      <rect x="18" y="82" width="100" height="38" rx="9" /><text x="68" y="106" text-anchor="middle">事件</text>
      <rect x="18" y="146" width="100" height="38" rx="9" /><text x="68" y="170" text-anchor="middle">AI &amp; LLMs</text>
    </g>
    <path d="M118 37 H162 M118 101 H162 M118 165 H162" class="concept-hero__line" marker-end="url(#cookbook-arrow)" />
    <rect x="170" y="52" width="118" height="98" rx="12" class="concept-hero__node concept-hero__node--accent" />
    <path d="M196 82 H262 M196 104 H262 M196 126 H262" class="concept-hero__line concept-hero__line--inside" />
    <text x="229" y="75" text-anchor="middle" class="concept-hero__label">Cookbook 配方</text>
    <text x="229" y="142" text-anchor="middle" class="concept-hero__detail">JSON 或代码</text>
    <path d="M288 101 H335" class="concept-hero__line" marker-end="url(#cookbook-arrow)" />
    <rect x="342" y="68" width="82" height="66" rx="12" class="concept-hero__outcome-box" />
    <path d="M363 101 l10 10 22 -24" class="concept-hero__check" />
    <text x="383" y="148" text-anchor="middle" class="concept-hero__detail">持久化执行</text>
  </svg>
</section>

<div class="cookbook-grid">

  <a class="cookbook-card" href="microservice-orchestration.html">
    <span class="cookbook-card__icon" aria-hidden="true"><svg viewBox="0 0 24 24"><rect x="3" y="4" width="6" height="6" rx="1"/><rect x="15" y="4" width="6" height="6" rx="1"/><rect x="9" y="14" width="6" height="6" rx="1"/><path d="M9 7h6M12 10v4"/></svg></span>
    <span class="cookbook-card__body"><strong>微服务编排</strong><span>HTTP 服务链、条件分支、使用 Fork/Join 的并行 HTTP 调用。</span></span>
    <span class="cookbook-card__arrow" aria-hidden="true">→</span>
  </a>

  <a class="cookbook-card" href="dynamic-parallelism.html">
    <span class="cookbook-card__icon" aria-hidden="true"><svg viewBox="0 0 24 24"><path d="M5 3v5c0 2 2 3 7 3s7 1 7 3v7"/><path d="M19 3v5c0 2-2 3-7 3s-7 1-7 3v7"/><circle cx="5" cy="3" r="2"/><circle cx="19" cy="3" r="2"/><circle cx="5" cy="21" r="2"/><circle cx="19" cy="21" r="2"/></svg></span>
    <span class="cookbook-card__body"><strong>动态并行</strong><span>动态分叉、按分支执行不同任务、同一任务扇出以及并行子工作流。</span></span>
    <span class="cookbook-card__arrow" aria-hidden="true">→</span>
  </a>

  <a class="cookbook-card" href="wait-and-timers.html">
    <span class="cookbook-card__icon" aria-hidden="true"><svg viewBox="0 0 24 24"><circle cx="12" cy="13" r="8"/><path d="M12 9v5l3 2M9 2h6M12 2v3"/></svg></span>
    <span class="cookbook-card__body"><strong>等待与定时器模式</strong><span>固定延迟、定时执行、外部信号和人工审批。</span></span>
    <span class="cookbook-card__arrow" aria-hidden="true">→</span>
  </a>

  <a class="cookbook-card" href="sending-signals.html">
    <span class="cookbook-card__icon" aria-hidden="true"><svg viewBox="0 0 24 24"><path d="M4 12h13M13 7l5 5-5 5"/><path d="M4 5v14"/></svg></span>
    <span class="cookbook-card__body"><strong>向工作流发送信号</strong><span>从审批界面、webhook 或外部回调完成被阻塞的等待。</span></span>
    <span class="cookbook-card__arrow" aria-hidden="true">→</span>
  </a>

  <a class="cookbook-card" href="task-timeouts-and-retries.html">
    <span class="cookbook-card__icon" aria-hidden="true"><svg viewBox="0 0 24 24"><path d="M20 11a8 8 0 1 0 1 4"/><path d="M20 4v7h-7"/><path d="M9 12l2 2 4-4"/></svg></span>
    <span class="cookbook-card__body"><strong>任务超时与重试</strong><span>指数退避、租约、硬性 SLA 以及防止惊群。</span></span>
    <span class="cookbook-card__arrow" aria-hidden="true">→</span>
  </a>

  <a class="cookbook-card" href="saga-compensation.html">
    <span class="cookbook-card__icon" aria-hidden="true"><svg viewBox="0 0 24 24"><path d="M3 12a9 9 0 0 1 9-9 9 9 0 0 1 9 9"/><path d="M21 12a9 9 0 0 1-9 9 9 9 0 0 1-9-9"/><path d="M3 12h4M17 12h4"/><path d="M7 9l-4 3 4 3M17 9l4 3-4 3"/></svg></span>
    <span class="cookbook-card__body"><strong>Saga 与补偿</strong><span>撤销部分完成的事务：failureWorkflow、逆序回滚、幂等撤销。</span></span>
    <span class="cookbook-card__arrow" aria-hidden="true">&#8594;</span>
  </a>

  <a class="cookbook-card" href="http-poll-long-running-job.html">
    <span class="cookbook-card__icon" aria-hidden="true"><svg viewBox="0 0 24 24"><path d="M21 12a9 9 0 1 1-3-6.7"/><path d="M21 3v6h-6"/><circle cx="12" cy="12" r="2.5"/></svg></span>
    <span class="cookbook-card__body"><strong>轮询长时间运行的任务</strong><span>用一个 HTTP_POLL 任务等待缓慢的第三方 API，配合退避与轮询上限。</span></span>
    <span class="cookbook-card__arrow" aria-hidden="true">&#8594;</span>
  </a>

  <a class="cookbook-card" href="workflow-scheduling.html">
    <span class="cookbook-card__icon" aria-hidden="true"><svg viewBox="0 0 24 24"><rect x="3" y="5" width="18" height="16" rx="2"/><path d="M7 3v4M17 3v4M3 10h18M8 14h3M13 14h3M8 18h3"/></svg></span>
    <span class="cookbook-card__body"><strong>定时工作流</strong><span>Cron 执行、停机补偿、有界窗口与并发运行。</span></span>
    <span class="cookbook-card__arrow" aria-hidden="true">→</span>
  </a>

  <a class="cookbook-card" href="event-driven.html">
    <span class="cookbook-card__icon" aria-hidden="true"><svg viewBox="0 0 24 24"><path d="M4 7h10M4 17h10M14 7l-3-3M14 7l-3 3M20 17l-3-3M20 17l-3 3"/><circle cx="4" cy="7" r="2"/><circle cx="20" cy="17" r="2"/></svg></span>
    <span class="cookbook-card__body"><strong>事件驱动配方</strong><span>以 Kafka、NATS、RabbitMQ 和 SQS 事件启动或推进工作流。</span></span>
    <span class="cookbook-card__arrow" aria-hidden="true">→</span>
  </a>

  <a class="cookbook-card" href="../ai/cookbook/index.html">
    <span class="cookbook-card__icon" aria-hidden="true"><svg viewBox="0 0 24 24"><path d="M7 4h10a3 3 0 0 1 3 3v7a3 3 0 0 1-3 3h-5l-4 3v-3H7a3 3 0 0 1-3-3V7a3 3 0 0 1 3-3Z"/><path d="M9 10h.01M12 10h.01M15 10h.01"/></svg></span>
    <span class="cookbook-card__body"><strong>AI Cookbook</strong><span>Agentic 工作流与用 SDK 编写的智能体：RAG、MCP 工具、护栏、人工审批、深度研究、交接与记忆。</span></span>
    <span class="cookbook-card__arrow" aria-hidden="true">→</span>
  </a>

  <a class="cookbook-card" href="dynamic-workflows.html">
    <span class="cookbook-card__icon" aria-hidden="true"><svg viewBox="0 0 24 24"><path d="m8 8-4 4 4 4M16 8l4 4-4 4M14 5l-4 14"/></svg></span>
    <span class="cookbook-card__body"><strong>以代码编写的动态工作流</strong><span>用 Python 定义的链、分支、并行任务、循环以及生成的定义。</span></span>
    <span class="cookbook-card__arrow" aria-hidden="true">→</span>
  </a>

</div>
