---
description: "智能体在 Conductor 中的含义，以及智能体轮次如何作为持久化工作流任务运行——带有工具、审批和完整的执行历史。"
---

# 智能体与 AI

## 什么是智能体？

智能体是一个使用 LLM 来决定下一步做什么的程序。它不再遵循固定的步骤序列，而是按轮次工作：模型读取目标与迄今的上下文，然后提出下一个行动。这个行动可能是一次工具调用、一个需要人回答的问题，或者最终答案。每个行动的结果都会成为下一轮的上下文，循环持续进行，直到目标达成。

<section class="agent-runtime-hero" aria-label="Conductor 智能体轮次循环">
  <svg class="agent-runtime-hero__diagram" viewBox="0 0 520 312" role="img" aria-labelledby="agent-runtime-diagram-title agent-runtime-diagram-description">
    <title id="agent-runtime-diagram-title">Conductor 智能体轮次循环</title>
    <desc id="agent-runtime-diagram-description">LLM 提出下一个行动。Conductor 校验该提案、持久化状态并调度工作。工作者、MCP 工具、远程 agent 和人员负责执行。结果被保存，并开启下一轮。</desc>
    <defs>
      <marker id="agent-runtime-arrow-runtime" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse">
        <path d="M 0 0 L 10 5 L 0 10 z" class="agent-runtime-hero__arrowhead agent-runtime-hero__arrowhead--runtime" />
      </marker>
    </defs>
    <rect x="110" y="16" width="300" height="56" rx="12" class="agent-runtime-hero__brain-box" />
    <text x="260" y="39" text-anchor="middle" class="agent-runtime-hero__runtime-title agent-runtime-hero__runtime-title--brain">LLM 决定下一步</text>
    <text x="260" y="59" text-anchor="middle" class="agent-runtime-hero__detail">提出工具调用、问题或答案</text>
    <path d="M 260 72 V 92" class="agent-runtime-hero__arrow agent-runtime-hero__arrow--runtime" marker-end="url(#agent-runtime-arrow-runtime)" />
    <rect x="80" y="94" width="360" height="72" rx="14" class="agent-runtime-hero__runtime-box" />
    <text x="260" y="119" text-anchor="middle" class="agent-runtime-hero__runtime-title">Conductor</text>
    <text x="260" y="138" text-anchor="middle" class="agent-runtime-hero__runtime-detail">校验提案 · 应用审批</text>
    <text x="260" y="155" text-anchor="middle" class="agent-runtime-hero__runtime-detail">持久化状态 · 调度工作</text>
    <path d="M 260 166 V 188" class="agent-runtime-hero__arrow agent-runtime-hero__arrow--runtime" marker-end="url(#agent-runtime-arrow-runtime)" />
    <rect x="31" y="192" width="110" height="56" rx="9" class="agent-runtime-hero__hands-card" />
    <text x="86" y="217" text-anchor="middle" class="agent-runtime-hero__label">工作者</text>
    <text x="86" y="234" text-anchor="middle" class="agent-runtime-hero__detail">你的代码</text>
    <rect x="147" y="192" width="110" height="56" rx="9" class="agent-runtime-hero__hands-card" />
    <text x="202" y="217" text-anchor="middle" class="agent-runtime-hero__label">MCP 工具</text>
    <text x="202" y="234" text-anchor="middle" class="agent-runtime-hero__detail">工具 + 数据</text>
    <rect x="263" y="192" width="110" height="56" rx="9" class="agent-runtime-hero__hands-card" />
    <text x="318" y="217" text-anchor="middle" class="agent-runtime-hero__label">远程 agent</text>
    <text x="318" y="234" text-anchor="middle" class="agent-runtime-hero__detail">A2A 协议</text>
    <rect x="379" y="192" width="110" height="56" rx="9" class="agent-runtime-hero__hands-card" />
    <text x="434" y="217" text-anchor="middle" class="agent-runtime-hero__label">人员</text>
    <text x="434" y="234" text-anchor="middle" class="agent-runtime-hero__detail">评审 + 输入</text>
    <path d="M 260 254 V 276 H 24 V 44 H 102" class="agent-runtime-hero__turn-loop" marker-end="url(#agent-runtime-arrow-runtime)" />
    <text x="272" y="296" text-anchor="middle" class="agent-runtime-hero__compile">结果被保存并开启下一轮</text>
  </svg>
</section>

在 Conductor 中，该循环以持久化工作流的形式运行。模型的提案是数据，而不是命令。Conductor 对其进行校验，应用任何必需的审批，然后才调度工作。工作本身作为普通任务运行，使用工作流已经具备的同一套构件：你的工作者、MCP 工具、远程 agent 和人员。由于每个结果都在下一轮开始前被持久化，崩溃、部署或长时间等待都不会丢失智能体的进度。

## 三种构建方式

这些路径是互补的。一个生产工作流可以在同一个持久化图中同时使用原生 AI 任务、调用已编译的 Conductor Agent，并把专家工作委托给远程 A2A agent。

<div class="agent-overview-grid agent-overview-grid--three">
  <a class="agent-overview-card agent-overview-card--link" href="llm-orchestration.html">
    <span class="agent-overview-card__kicker">直接编排</span>
    <strong>声明式 AI 工作流</strong>
    <span>直接组合 LLM、MCP、向量检索、控制流、等待和人工任务。当工作流定义需要暴露完整编排时，选择这种方式。</span>
  </a>
  <a class="agent-overview-card agent-overview-card--link" href="conductor-agents.html">
    <span class="agent-overview-card__kicker">编译图</span>
    <strong>Conductor Agents</strong>
    <span>使用 Conductor SDK 编写，或引入受支持的框架 agent。将其编译为可检查的图并部署，然后作为 <code>AGENT</code> 任务复用。</span>
  </a>
  <a class="agent-overview-card agent-overview-card--link" href="a2a-integration.html">
    <span class="agent-overview-card__kicker">远程委托</span>
    <strong>A2A agents</strong>
    <span>通过持久化 <code>AGENT</code> 任务调用运行在 Agent2Agent 协议之后的 agent。Conductor 管理交接过程，无需在本地编译该 agent。</span>
  </a>
</div>

## 运行原则

当执行契约是显式的，自适应行为才保持可控。这些原则适用于全部三种编写路径。

<div class="agent-overview-grid agent-overview-grid--principles">
  <div class="agent-overview-card">
    <span class="agent-overview-card__number">01</span>
    <strong>模型输出是提案</strong>
    <span>计划和工具参数必须通过 schema 校验、策略、护栏和审批，才能成为可执行的工作。</span>
  </div>
  <div class="agent-overview-card">
    <span class="agent-overview-card__number">02</span>
    <strong>状态属于工作流</strong>
    <span>进度、等待、决策和结果都保存在持久化执行状态中——而不只是 agent 进程的内存里。</span>
  </div>
  <div class="agent-overview-card">
    <span class="agent-overview-card__number">03</span>
    <strong>副作用跨越任务边界</strong>
    <span>工作者和系统任务通过有界、可观察的接口执行已审批的操作，这些接口带有定义好的重试和超时。</span>
  </div>
  <div class="agent-overview-card">
    <span class="agent-overview-card__number">04</span>
    <strong>每一轮都可治理</strong>
    <span>提案、策略结果、审批、输入、输出、重试、时序和终止状态都保持可检查、可恢复。</span>
  </div>
</div>

## 你将获得

Conductor 将同一套持久化执行模型应用于自适应智能体和普通分布式工作流。

<div class="agent-overview-grid agent-overview-grid--outcomes">
  <div class="agent-overview-card"><strong>持久化执行</strong><span>跨崩溃、部署、重试和长时间等待，从已持久化的进度处恢复。</span></div>
  <div class="agent-overview-card"><strong>策略与护栏</strong><span>在执行前校验模型提案，并约束工具、输入、扇出、时间和成本。</span></div>
  <div class="agent-overview-card"><strong>逐轮可观察性</strong><span>检查决策、策略结果、任务数据、时序和失败的持久化记录。</span></div>
  <div class="agent-overview-card"><strong>人工控制</strong><span>暂停而不丢失状态，收集评审或输入，然后恢复同一执行。</span></div>
  <div class="agent-overview-card"><strong>框架与协议互操作</strong><span>在稳定的工作流边界之后使用受支持的 agent 框架、MCP 工具和远程 A2A agent。</span></div>
  <div class="agent-overview-card"><strong>普通工作流编排</strong><span>将 agent 与 API、工作者、分支、调度、通知和补偿逻辑并排放置。</span></div>
</div>

## 从哪里开始

选择与你要构建的东西相匹配的边界，然后只深入你需要的平台部分。

<div class="agent-overview-grid agent-overview-grid--three agent-overview-grid--next">
  <div class="agent-overview-card">
    <span class="agent-overview-card__kicker">构建</span>
    <strong>选择编写路径</strong>
    <span>比较三种 agent 模型，然后学习声明式工作流可用的原生模型与检索任务。</span>
    <span class="agent-overview-card__links"><a href="../concepts/agents.html">智能体概念</a><a href="llm-orchestration.html">LLM 编排</a></span>
  </div>
  <div class="agent-overview-card">
    <span class="agent-overview-card__kicker">集成</span>
    <strong>把 agent 带入持久化图</strong>
    <span>在本地编译由 SDK 或框架编写的 agent，或通过 A2A 调用独立部署的 agent。</span>
    <span class="agent-overview-card__links"><a href="conductor-agents.html">Conductor Agents</a><a href="a2a-integration.html">A2A 集成</a></span>
  </div>
  <div class="agent-overview-card">
    <span class="agent-overview-card__kicker">运维</span>
    <strong>为生产而设计</strong>
    <span>应用参考架构，然后依次走过治理、评估、部署、恢复和运维。</span>
    <span class="agent-overview-card__links"><a href="production-agent-architecture.html">生产架构</a><a href="production-path.html">生产路径</a></span>
  </div>
</div>
