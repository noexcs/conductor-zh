---
description: "Conductor 是什么以及它如何工作：一个持久化地编排工作流、工作者和 AI 智能体的开源引擎。"
---

# 核心概念

## 什么是 Conductor？

**Conductor 是一个持久化运行工作流的开源编排引擎。** 工作流是一系列任务，可以分支、循环和并行运行。Conductor
决定下一个运行的任务，记录每一步的结果，并在步骤失败时重试或恢复。崩溃或重启永远不会丢失进度。

职责在 Conductor 服务器和你自己的代码之间划分：

- **服务器负责编排。** 它作为独立服务运行，可自托管或托管在云端。它
  调度任务，强制执行重试和超时，并在每一步之后持久化状态。编排
  逻辑不留在你的应用代码中。
- **你的工作者负责执行。** 它们运行在你自己的基础设施中，在你已经部署的服务、容器或
  函数内部。业务逻辑是用任何拥有 Conductor SDK 的语言编写的普通函数。工作者轮询服务器获取任务并上报结果，因此它们不需要入站端口。
- **系统任务是内置的。** 它们在服务器内部运行。诸如 HTTP 调用、
  事件和 LLM 调用等常见步骤无需工作者代码。

```mermaid
flowchart LR
    def["工作流定义<br/>(JSON 或代码)"] --> engine
    subgraph server["Conductor 服务器"]
        engine["调度任务、持久化每次状态转换，<br/>处理重试、超时和流程控制"]
    end
    engine -- "任务入队" --> queue[["任务队列"]]
    workers["你的工作者<br/>(任意语言)"] -- "轮询工作" --> queue
    workers -- "上报结果" --> engine
```

工作流定义是 JSON。在源代码控制中对其进行版本管理、从代码生成，或让 LLM
在运行时创建和修改它们。

AI 工作以同样的方式运行。LLM 调用、工具使用和智能体都是工作流任务，拥有与其他
每一步相同的重试、持久化和可观测性。

## Conductor 能做什么？

<div class="wcc-widget" role="tablist">
  <div class="wcc-left">
    <div class="wcc-item wcc-active" data-wcc="0" role="tab" aria-selected="true" tabindex="0">
      <div class="wcc-header"><span class="wcc-title">创建工作流</span><span class="wcc-chevron"></span></div>
      <div class="wcc-body">定义由多个任务组成的工作流，这些任务按特定顺序执行。 <a href="../../documentation/configuration/workflowdef/index.html">了解更多</a></div>
    </div>
    <div class="wcc-item" data-wcc="1" role="tab" aria-selected="false" tabindex="0">
      <div class="wcc-header"><span class="wcc-title">分支你的流程</span><span class="wcc-chevron"></span></div>
      <div class="wcc-body">使用 switch-case 运算符做出分支决策。 <a href="../../documentation/configuration/workflowdef/operators/switch-task.html">了解更多</a></div>
    </div>
    <div class="wcc-item" data-wcc="2" role="tab" aria-selected="false" tabindex="0">
      <div class="wcc-header"><span class="wcc-title">运行循环</span><span class="wcc-chevron"></span></div>
      <div class="wcc-body">使用 Do-While 循环运算符迭代一组任务。 <a href="../../documentation/configuration/workflowdef/operators/do-while-task.html">了解更多</a></div>
    </div>
    <div class="wcc-item" data-wcc="3" role="tab" aria-selected="false" tabindex="0">
      <div class="wcc-header"><span class="wcc-title">并行化你的任务</span><span class="wcc-chevron"></span></div>
      <div class="wcc-body">使用静态或动态 Fork 并行执行任务。 <a href="../../documentation/configuration/workflowdef/operators/fork-task.html">了解更多</a></div>
    </div>
    <div class="wcc-item" data-wcc="4" role="tab" aria-selected="false" tabindex="0">
      <div class="wcc-header"><span class="wcc-title">在外部运行你的任务</span><span class="wcc-chevron"></span></div>
      <div class="wcc-body">在微服务、无服务器函数或应用程序中使用外部工作者来实现任务。 <a href="workers.html">工作者</a> · <a href="../../documentation/clientsdks/index.html">SDKs</a></div>
    </div>
    <div class="wcc-item" data-wcc="5" role="tab" aria-selected="false" tabindex="0">
      <div class="wcc-header"><span class="wcc-title">使用内置任务</span><span class="wcc-chevron"></span></div>
      <div class="wcc-body">使用内置任务执行常见操作，例如调用 HTTP 端点、写入事件队列和执行内联代码。 <a href="../../documentation/configuration/workflowdef/systemtasks/index.html">了解更多</a></div>
    </div>
    <div class="wcc-item" data-wcc="6" role="tab" aria-selected="false" tabindex="0">
      <div class="wcc-header"><span class="wcc-title">使用 LLM 任务</span><span class="wcc-chevron"></span></div>
      <div class="wcc-body">使用 LLM 任务构建 AI 驱动的工作流，包括智能体工作流。 <a href="../ai/index.html">了解更多</a></div>
    </div>
    <div class="wcc-item" data-wcc="13" role="tab" aria-selected="false" tabindex="0">
      <div class="wcc-header"><span class="wcc-title">编排智能体</span><span class="wcc-chevron"></span></div>
      <div class="wcc-body">构建并运行 Conductor Agent，或将已部署的及远程的 A2A 智能体编排为持久的工作流步骤。 <a href="agents.html">了解更多</a></div>
    </div>
    <div class="wcc-item" data-wcc="7" role="tab" aria-selected="false" tabindex="0">
      <div class="wcc-header"><span class="wcc-title">人工介入（Human in the Loop）</span><span class="wcc-chevron"></span></div>
      <div class="wcc-body">使用 Human 任务在你的工作流中插入人工步骤。 <a href="../../documentation/configuration/workflowdef/systemtasks/human-task.html">Human 任务</a> · <a href="../../documentation/configuration/workflowdef/systemtasks/wait-task.html">Wait 任务</a></div>
    </div>
    <div class="wcc-item" data-wcc="8" role="tab" aria-selected="false" tabindex="0">
      <div class="wcc-header"><span class="wcc-title">处理故障</span><span class="wcc-chevron"></span></div>
      <div class="wcc-body">设置超时和速率限制来管理任务和工作流的故障。 <a href="../how-tos/Workflows/handling-errors.html">了解更多</a></div>
    </div>
    <div class="wcc-item" data-wcc="9" role="tab" aria-selected="false" tabindex="0">
      <div class="wcc-header"><span class="wcc-title">回放任何工作流</span><span class="wcc-chevron"></span></div>
      <div class="wcc-body">从头开始、从任意任务开始回放已完成或失败的工作流，或只重试失败的步骤——即使在数月之后。完整的执行历史始终保留。 <a href="../how-tos/Workflows/debugging-workflows.html">了解更多</a></div>
    </div>
    <div class="wcc-item" data-wcc="10" role="tab" aria-selected="false" tabindex="0">
      <div class="wcc-header"><span class="wcc-title">与应用集成</span><span class="wcc-chevron"></span></div>
      <div class="wcc-body">使用 Kafka、NATS、SQS、AMQP 和 webhook 的事件驱动触发器，将 Conductor 接入你的生态系统。 <a href="../cookbook/event-driven.html">了解更多</a></div>
    </div>
    <div class="wcc-item" data-wcc="11" role="tab" aria-selected="false" tabindex="0">
      <div class="wcc-header"><span class="wcc-title">可视化调试</span><span class="wcc-chevron"></span></div>
      <div class="wcc-body">从 Conductor UI 跟踪和调试工作流。查看输入、拉取日志，并从任意点重启。 <a href="../../quickstart/first-workflow.html">快速开始</a></div>
    </div>
    <div class="wcc-item" data-wcc="12" role="tab" aria-selected="false" tabindex="0">
      <div class="wcc-header"><span class="wcc-title">水平扩展</span><span class="wcc-chevron"></span></div>
      <div class="wcc-body">在负载均衡器后面运行多个服务器实例，配合共享后端实现高可用。 <a href="../running/deploy.html">部署指南</a></div>
    </div>
    <div class="wcc-controls" aria-label="Feature navigation">
      <button class="wcc-control" data-wcc-direction="previous" type="button">
        <span class="wcc-control-icon" aria-hidden="true">←</span><span>上一页</span>
      </button>
      <button class="wcc-control" data-wcc-direction="next" type="button">
        <span>下一页</span><span class="wcc-control-icon" aria-hidden="true">→</span>
      </button>
    </div>
  </div>
  <div class="wcc-right" role="tabpanel">
    <!-- 0: Create Workflows — linear -->
    <svg class="wcc-diagram wcc-visible" data-wcc-diagram="0" viewBox="0 0 220 420" xmlns="http://www.w3.org/2000/svg">
      <circle cx="110" cy="30" r="22" fill="#e2e8f0" stroke="#4a5568" stroke-width="1.5"/><text x="110" y="35" text-anchor="middle" font-size="11" fill="#2e3545" font-family="sans-serif">开始</text>
      <line x1="110" y1="52" x2="110" y2="80" stroke="#a0aec0" stroke-width="1.5" marker-end="url(#wcc-arrow)"/>
      <rect x="35" y="80" width="150" height="40" rx="6" fill="#e2e8f0" stroke="#4a5568" stroke-width="1.5"/><text x="110" y="105" text-anchor="middle" font-size="12" fill="#2e3545" font-family="sans-serif">任务 A</text>
      <line x1="110" y1="120" x2="110" y2="155" stroke="#a0aec0" stroke-width="1.5" marker-end="url(#wcc-arrow)"/>
      <rect x="35" y="155" width="150" height="40" rx="6" fill="#e2e8f0" stroke="#4a5568" stroke-width="1.5"/><text x="110" y="180" text-anchor="middle" font-size="12" fill="#2e3545" font-family="sans-serif">任务 B</text>
      <line x1="110" y1="195" x2="110" y2="230" stroke="#a0aec0" stroke-width="1.5" marker-end="url(#wcc-arrow)"/>
      <rect x="35" y="230" width="150" height="40" rx="6" fill="#e2e8f0" stroke="#4a5568" stroke-width="1.5"/><text x="110" y="255" text-anchor="middle" font-size="12" fill="#2e3545" font-family="sans-serif">任务 C</text>
      <line x1="110" y1="270" x2="110" y2="305" stroke="#a0aec0" stroke-width="1.5" marker-end="url(#wcc-arrow)"/>
      <circle cx="110" cy="327" r="22" fill="#e2e8f0" stroke="#4a5568" stroke-width="1.5"/><text x="110" y="332" text-anchor="middle" font-size="11" fill="#2e3545" font-family="sans-serif">结束</text>
      <defs><marker id="wcc-arrow" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="6" markerHeight="6" orient="auto-start-reverse"><path d="M 0 0 L 10 5 L 0 10 z" fill="#a0aec0"/></marker></defs>
    </svg>
    <!-- 1: Branch — switch -->
    <svg class="wcc-diagram" data-wcc-diagram="1" viewBox="0 0 260 420" xmlns="http://www.w3.org/2000/svg">
      <circle cx="130" cy="30" r="22" fill="#e2e8f0" stroke="#4a5568" stroke-width="1.5"/><text x="130" y="35" text-anchor="middle" font-size="11" fill="#2e3545" font-family="sans-serif">开始</text>
      <line x1="130" y1="52" x2="130" y2="80" stroke="#a0aec0" stroke-width="1.5" marker-end="url(#wcc-arrow)"/>
      <rect x="55" y="80" width="150" height="40" rx="6" fill="#e2e8f0" stroke="#4a5568" stroke-width="1.5"/><text x="130" y="105" text-anchor="middle" font-size="12" fill="#2e3545" font-family="sans-serif">任务 A</text>
      <line x1="130" y1="120" x2="130" y2="150" stroke="#a0aec0" stroke-width="1.5" marker-end="url(#wcc-arrow)"/>
      <polygon points="130,150 170,185 130,220 90,185" fill="#e2e8f0" stroke="#4a5568" stroke-width="1.5"/><text x="130" y="188" text-anchor="middle" font-size="10" fill="#2e3545" font-family="sans-serif">Switch</text><text x="130" y="200" text-anchor="middle" font-size="10" fill="#2e3545" font-family="sans-serif">分支</text>
      <line x1="90" y1="185" x2="50" y2="185" stroke="#a0aec0" stroke-width="1.5"/><line x1="50" y1="185" x2="50" y2="260" stroke="#a0aec0" stroke-width="1.5" marker-end="url(#wcc-arrow)"/>
      <line x1="170" y1="185" x2="210" y2="185" stroke="#a0aec0" stroke-width="1.5"/><line x1="210" y1="185" x2="210" y2="260" stroke="#a0aec0" stroke-width="1.5" marker-end="url(#wcc-arrow)"/>
      <rect x="0" y="260" width="100" height="40" rx="6" fill="#e2e8f0" stroke="#4a5568" stroke-width="1.5"/><text x="50" y="285" text-anchor="middle" font-size="12" fill="#2e3545" font-family="sans-serif">任务 B</text>
      <rect x="160" y="260" width="100" height="40" rx="6" fill="#e2e8f0" stroke="#4a5568" stroke-width="1.5"/><text x="210" y="285" text-anchor="middle" font-size="12" fill="#2e3545" font-family="sans-serif">任务 C</text>
      <line x1="50" y1="300" x2="50" y2="340" stroke="#a0aec0" stroke-width="1.5"/><line x1="50" y1="340" x2="130" y2="340" stroke="#a0aec0" stroke-width="1.5"/>
      <line x1="210" y1="300" x2="210" y2="340" stroke="#a0aec0" stroke-width="1.5"/><line x1="210" y1="340" x2="130" y2="340" stroke="#a0aec0" stroke-width="1.5"/>
      <line x1="130" y1="340" x2="130" y2="360" stroke="#a0aec0" stroke-width="1.5" marker-end="url(#wcc-arrow)"/>
      <circle cx="130" cy="382" r="22" fill="#e2e8f0" stroke="#4a5568" stroke-width="1.5"/><text x="130" y="387" text-anchor="middle" font-size="11" fill="#2e3545" font-family="sans-serif">结束</text>
    </svg>
    <!-- 2: Run Loops — do-while highlighted -->
    <svg class="wcc-diagram" data-wcc-diagram="2" viewBox="-30 0 330 440" xmlns="http://www.w3.org/2000/svg">
      <circle cx="110" cy="30" r="22" fill="#e2e8f0" stroke="#4a5568" stroke-width="1.5"/><text x="110" y="35" text-anchor="middle" font-size="11" fill="#2e3545" font-family="sans-serif">开始</text>
      <line x1="110" y1="52" x2="110" y2="80" stroke="#a0aec0" stroke-width="1.5" marker-end="url(#wcc-arrow)"/>
      <rect x="35" y="80" width="150" height="40" rx="6" fill="#e2e8f0" stroke="#4a5568" stroke-width="1.5"/><text x="110" y="105" text-anchor="middle" font-size="12" fill="#2e3545" font-family="sans-serif">任务 A</text>
      <line x1="110" y1="120" x2="110" y2="150" stroke="#a0aec0" stroke-width="1.5" marker-end="url(#wcc-arrow)"/>
      <polygon points="110,150 150,185 110,220 70,185" fill="#e2e8f0" stroke="#4a5568" stroke-width="1.5"/><text x="110" y="188" text-anchor="middle" font-size="10" fill="#2e3545" font-family="sans-serif">Switch</text><text x="110" y="200" text-anchor="middle" font-size="10" fill="#2e3545" font-family="sans-serif">分支</text>
      <line x1="70" y1="185" x2="30" y2="185" stroke="#a0aec0" stroke-width="1.5"/><line x1="30" y1="185" x2="30" y2="270" stroke="#a0aec0" stroke-width="1.5" marker-end="url(#wcc-arrow)"/>
      <rect x="-20" y="270" width="100" height="40" rx="6" fill="#e2e8f0" stroke="#4a5568" stroke-width="1.5"/><text x="30" y="295" text-anchor="middle" font-size="12" fill="#2e3545" font-family="sans-serif">任务 B</text>
      <line x1="150" y1="185" x2="210" y2="185" stroke="#a0aec0" stroke-width="1.5"/><line x1="210" y1="185" x2="210" y2="250" stroke="#a0aec0" stroke-width="1.5" marker-end="url(#wcc-arrow)"/>
      <rect x="155" y="250" width="120" height="75" rx="8" fill="rgba(6,214,160,0.10)" stroke="#06d6a0" stroke-width="2"/><text x="215" y="272" text-anchor="middle" font-size="10" font-weight="600" fill="#06d6a0" font-family="sans-serif">Do While 循环</text>
      <rect x="170" y="282" width="90" height="32" rx="5" fill="#e2e8f0" stroke="#4a5568" stroke-width="1.5"/><text x="215" y="303" text-anchor="middle" font-size="11" fill="#2e3545" font-family="sans-serif">任务 C</text>
      <path d="M 260 298 C 280 298 280 268 260 268" stroke="#06d6a0" stroke-width="1.5" fill="none" marker-end="url(#wcc-arrow-teal)"/>
      <line x1="30" y1="310" x2="30" y2="370" stroke="#a0aec0" stroke-width="1.5"/><line x1="30" y1="370" x2="110" y2="370" stroke="#a0aec0" stroke-width="1.5"/>
      <line x1="215" y1="325" x2="215" y2="370" stroke="#a0aec0" stroke-width="1.5"/><line x1="215" y1="370" x2="110" y2="370" stroke="#a0aec0" stroke-width="1.5"/>
      <line x1="110" y1="370" x2="110" y2="390" stroke="#a0aec0" stroke-width="1.5" marker-end="url(#wcc-arrow)"/>
      <circle cx="110" cy="412" r="22" fill="#e2e8f0" stroke="#4a5568" stroke-width="1.5"/><text x="110" y="417" text-anchor="middle" font-size="11" fill="#2e3545" font-family="sans-serif">结束</text>
      <defs><marker id="wcc-arrow-teal" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="6" markerHeight="6" orient="auto-start-reverse"><path d="M 0 0 L 10 5 L 0 10 z" fill="#06d6a0"/></marker></defs>
    </svg>
    <!-- 3: Parallelize — fork/join -->
    <svg class="wcc-diagram" data-wcc-diagram="3" viewBox="-25 0 370 440" xmlns="http://www.w3.org/2000/svg">
      <circle cx="140" cy="30" r="22" fill="#e2e8f0" stroke="#4a5568" stroke-width="1.5"/><text x="140" y="35" text-anchor="middle" font-size="11" fill="#2e3545" font-family="sans-serif">开始</text>
      <line x1="140" y1="52" x2="140" y2="80" stroke="#a0aec0" stroke-width="1.5" marker-end="url(#wcc-arrow)"/>
      <rect x="65" y="80" width="150" height="40" rx="6" fill="#e2e8f0" stroke="#4a5568" stroke-width="1.5"/><text x="140" y="105" text-anchor="middle" font-size="12" fill="#2e3545" font-family="sans-serif">任务 A</text>
      <line x1="140" y1="120" x2="140" y2="148" stroke="#a0aec0" stroke-width="1.5" marker-end="url(#wcc-arrow)"/>
      <polygon points="140,148 180,178 140,208 100,178" fill="#e2e8f0" stroke="#4a5568" stroke-width="1.5"/><text x="140" y="181" text-anchor="middle" font-size="10" fill="#2e3545" font-family="sans-serif">Switch</text><text x="140" y="193" text-anchor="middle" font-size="10" fill="#2e3545" font-family="sans-serif">分支</text>
      <line x1="100" y1="178" x2="40" y2="178" stroke="#a0aec0" stroke-width="1.5"/><line x1="40" y1="178" x2="40" y2="240" stroke="#a0aec0" stroke-width="1.5" marker-end="url(#wcc-arrow)"/>
      <line x1="140" y1="208" x2="140" y2="240" stroke="#a0aec0" stroke-width="1.5" marker-end="url(#wcc-arrow)"/>
      <line x1="180" y1="178" x2="255" y2="178" stroke="#a0aec0" stroke-width="1.5"/><line x1="255" y1="178" x2="255" y2="230" stroke="#a0aec0" stroke-width="1.5" marker-end="url(#wcc-arrow)"/>
      <rect x="-10" y="240" width="100" height="40" rx="6" fill="#e2e8f0" stroke="#4a5568" stroke-width="1.5"/><text x="40" y="265" text-anchor="middle" font-size="12" fill="#2e3545" font-family="sans-serif">任务 B</text>
      <rect x="90" y="240" width="100" height="40" rx="6" fill="#dc2626" stroke="#dc2626" stroke-width="1.5"/><text x="140" y="265" text-anchor="middle" font-size="12" fill="#fff" font-family="sans-serif" font-weight="600">任务 D</text>
      <rect x="200" y="230" width="120" height="70" rx="8" fill="rgba(6,214,160,0.10)" stroke="#06d6a0" stroke-width="2"/><text x="260" y="250" text-anchor="middle" font-size="10" font-weight="600" fill="#06d6a0" font-family="sans-serif">Do While 循环</text>
      <rect x="215" y="258" width="90" height="32" rx="5" fill="#e2e8f0" stroke="#4a5568" stroke-width="1.5"/><text x="260" y="279" text-anchor="middle" font-size="11" fill="#2e3545" font-family="sans-serif">任务 C</text>
      <line x1="40" y1="280" x2="40" y2="340" stroke="#a0aec0" stroke-width="1.5"/><line x1="40" y1="340" x2="140" y2="340" stroke="#a0aec0" stroke-width="1.5"/>
      <line x1="140" y1="280" x2="140" y2="340" stroke="#a0aec0" stroke-width="1.5"/>
      <line x1="260" y1="300" x2="260" y2="340" stroke="#a0aec0" stroke-width="1.5"/><line x1="260" y1="340" x2="140" y2="340" stroke="#a0aec0" stroke-width="1.5"/>
      <line x1="140" y1="340" x2="140" y2="370" stroke="#a0aec0" stroke-width="1.5" marker-end="url(#wcc-arrow)"/>
      <circle cx="140" cy="392" r="22" fill="#e2e8f0" stroke="#4a5568" stroke-width="1.5"/><text x="140" y="397" text-anchor="middle" font-size="11" fill="#2e3545" font-family="sans-serif">结束</text>
    </svg>
    <!-- 4: External Workers -->
    <svg class="wcc-diagram" data-wcc-diagram="4" viewBox="0 0 320 440" xmlns="http://www.w3.org/2000/svg">
      <circle cx="90" cy="30" r="22" fill="#e2e8f0" stroke="#4a5568" stroke-width="1.5"/><text x="90" y="35" text-anchor="middle" font-size="11" fill="#2e3545" font-family="sans-serif">开始</text>
      <line x1="90" y1="52" x2="90" y2="80" stroke="#a0aec0" stroke-width="1.5" marker-end="url(#wcc-arrow)"/>
      <rect x="30" y="80" width="120" height="40" rx="6" fill="#e2e8f0" stroke="#4a5568" stroke-width="1.5"/><text x="90" y="105" text-anchor="middle" font-size="12" fill="#2e3545" font-family="sans-serif">任务 A</text>
      <line x1="150" y1="100" x2="200" y2="100" stroke="#a0aec0" stroke-width="1.5" stroke-dasharray="5,3" marker-end="url(#wcc-arrow)"/>
      <rect x="200" y="80" width="110" height="40" rx="6" fill="#06d6a0" stroke="#05c792" stroke-width="1.5"/><text x="255" y="98" text-anchor="middle" font-size="10" fill="#fff" font-weight="600" font-family="sans-serif">工作者 A</text><text x="255" y="112" text-anchor="middle" font-size="9" fill="#fff" font-family="sans-serif">微服务</text>
      <line x1="90" y1="120" x2="90" y2="160" stroke="#a0aec0" stroke-width="1.5" marker-end="url(#wcc-arrow)"/>
      <rect x="30" y="160" width="120" height="40" rx="6" fill="#e2e8f0" stroke="#4a5568" stroke-width="1.5"/><text x="90" y="185" text-anchor="middle" font-size="12" fill="#2e3545" font-family="sans-serif">任务 B</text>
      <line x1="150" y1="180" x2="200" y2="180" stroke="#a0aec0" stroke-width="1.5" stroke-dasharray="5,3" marker-end="url(#wcc-arrow)"/>
      <rect x="200" y="160" width="110" height="40" rx="6" fill="#f59e0b" stroke="#d97706" stroke-width="1.5"/><text x="255" y="178" text-anchor="middle" font-size="10" fill="#fff" font-weight="600" font-family="sans-serif">工作者 B</text><text x="255" y="192" text-anchor="middle" font-size="9" fill="#fff" font-family="sans-serif">无服务器</text>
      <line x1="90" y1="200" x2="90" y2="240" stroke="#a0aec0" stroke-width="1.5" marker-end="url(#wcc-arrow)"/>
      <rect x="30" y="240" width="120" height="40" rx="6" fill="#e2e8f0" stroke="#4a5568" stroke-width="1.5"/><text x="90" y="265" text-anchor="middle" font-size="12" fill="#2e3545" font-family="sans-serif">任务 C</text>
      <line x1="150" y1="260" x2="200" y2="260" stroke="#a0aec0" stroke-width="1.5" stroke-dasharray="5,3" marker-end="url(#wcc-arrow)"/>
      <rect x="200" y="240" width="110" height="40" rx="6" fill="#3b82f6" stroke="#2563eb" stroke-width="1.5"/><text x="255" y="258" text-anchor="middle" font-size="10" fill="#fff" font-weight="600" font-family="sans-serif">工作者 C</text><text x="255" y="272" text-anchor="middle" font-size="9" fill="#fff" font-family="sans-serif">遗留应用</text>
      <line x1="90" y1="280" x2="90" y2="320" stroke="#a0aec0" stroke-width="1.5" marker-end="url(#wcc-arrow)"/>
      <circle cx="90" cy="342" r="22" fill="#e2e8f0" stroke="#4a5568" stroke-width="1.5"/><text x="90" y="347" text-anchor="middle" font-size="11" fill="#2e3545" font-family="sans-serif">结束</text>
    </svg>
    <!-- 5: Built-in Tasks -->
    <svg class="wcc-diagram" data-wcc-diagram="5" viewBox="0 0 260 420" xmlns="http://www.w3.org/2000/svg">
      <circle cx="130" cy="30" r="22" fill="#e2e8f0" stroke="#4a5568" stroke-width="1.5"/><text x="130" y="35" text-anchor="middle" font-size="11" fill="#2e3545" font-family="sans-serif">开始</text>
      <line x1="130" y1="52" x2="130" y2="80" stroke="#a0aec0" stroke-width="1.5" marker-end="url(#wcc-arrow)"/>
      <rect x="30" y="80" width="200" height="40" rx="6" fill="#e2e8f0" stroke="#4a5568" stroke-width="1.5"/><text x="130" y="105" text-anchor="middle" font-size="11" fill="#2e3545" font-family="sans-serif">HTTP：调用 API 端点</text>
      <line x1="130" y1="120" x2="130" y2="160" stroke="#a0aec0" stroke-width="1.5" marker-end="url(#wcc-arrow)"/>
      <rect x="30" y="160" width="200" height="40" rx="6" fill="#e2e8f0" stroke="#4a5568" stroke-width="1.5"/><text x="130" y="185" text-anchor="middle" font-size="11" fill="#2e3545" font-family="sans-serif">Event：写入 Kafka</text>
      <line x1="130" y1="200" x2="130" y2="240" stroke="#a0aec0" stroke-width="1.5" marker-end="url(#wcc-arrow)"/>
      <rect x="30" y="240" width="200" height="40" rx="6" fill="#e2e8f0" stroke="#4a5568" stroke-width="1.5"/><text x="130" y="265" text-anchor="middle" font-size="11" fill="#2e3545" font-family="sans-serif">Inline：执行 JS</text>
      <line x1="130" y1="280" x2="130" y2="320" stroke="#a0aec0" stroke-width="1.5" marker-end="url(#wcc-arrow)"/>
      <circle cx="130" cy="342" r="22" fill="#e2e8f0" stroke="#4a5568" stroke-width="1.5"/><text x="130" y="347" text-anchor="middle" font-size="11" fill="#2e3545" font-family="sans-serif">结束</text>
    </svg>
    <!-- 6: LLM Tasks -->
    <svg class="wcc-diagram" data-wcc-diagram="6" viewBox="0 0 240 340" xmlns="http://www.w3.org/2000/svg">
      <circle cx="120" cy="30" r="22" fill="#e2e8f0" stroke="#4a5568" stroke-width="1.5"/><text x="120" y="35" text-anchor="middle" font-size="11" fill="#2e3545" font-family="sans-serif">开始</text>
      <line x1="120" y1="52" x2="120" y2="80" stroke="#a0aec0" stroke-width="1.5" marker-end="url(#wcc-arrow)"/>
      <rect x="20" y="80" width="200" height="40" rx="6" fill="#e2e8f0" stroke="#4a5568" stroke-width="1.5"/><text x="120" y="105" text-anchor="middle" font-size="11" fill="#2e3545" font-family="sans-serif">搜索新闻索引</text>
      <line x1="120" y1="120" x2="120" y2="160" stroke="#a0aec0" stroke-width="1.5" marker-end="url(#wcc-arrow)"/>
      <rect x="20" y="160" width="200" height="40" rx="6" fill="#e2e8f0" stroke="#4a5568" stroke-width="1.5"/><text x="120" y="185" text-anchor="middle" font-size="11" fill="#2e3545" font-family="sans-serif">获取上下文答案</text>
      <line x1="120" y1="200" x2="120" y2="240" stroke="#a0aec0" stroke-width="1.5" marker-end="url(#wcc-arrow)"/>
      <circle cx="120" cy="262" r="22" fill="#e2e8f0" stroke="#4a5568" stroke-width="1.5"/><text x="120" y="267" text-anchor="middle" font-size="11" fill="#2e3545" font-family="sans-serif">结束</text>
    </svg>
    <!-- 7: Human in the Loop -->
    <svg class="wcc-diagram" data-wcc-diagram="7" viewBox="0 0 280 380" xmlns="http://www.w3.org/2000/svg">
      <circle cx="140" cy="30" r="22" fill="#e2e8f0" stroke="#4a5568" stroke-width="1.5"/><text x="140" y="35" text-anchor="middle" font-size="11" fill="#2e3545" font-family="sans-serif">开始</text>
      <line x1="140" y1="52" x2="140" y2="80" stroke="#a0aec0" stroke-width="1.5" marker-end="url(#wcc-arrow)"/>
      <polygon points="140,80 180,115 140,150 100,115" fill="#e2e8f0" stroke="#4a5568" stroke-width="1.5"/><text x="140" y="118" text-anchor="middle" font-size="11" fill="#2e3545" font-family="sans-serif">Switch</text>
      <line x1="100" y1="115" x2="50" y2="115" stroke="#a0aec0" stroke-width="1.5"/><line x1="50" y1="115" x2="50" y2="190" stroke="#a0aec0" stroke-width="1.5" marker-end="url(#wcc-arrow)"/>
      <line x1="180" y1="115" x2="230" y2="115" stroke="#a0aec0" stroke-width="1.5"/><line x1="230" y1="115" x2="230" y2="190" stroke="#a0aec0" stroke-width="1.5" marker-end="url(#wcc-arrow)"/>
      <rect x="0" y="190" width="100" height="45" rx="6" fill="#e2e8f0" stroke="#4a5568" stroke-width="1.5"/><text x="50" y="210" text-anchor="middle" font-size="10" fill="#2e3545" font-family="sans-serif">默认</text><text x="50" y="224" text-anchor="middle" font-size="10" fill="#2e3545" font-family="sans-serif">审批</text>
      <rect x="180" y="190" width="100" height="45" rx="6" fill="#f59e0b" stroke="#d97706" stroke-width="1.5"/><text x="230" y="210" text-anchor="middle" font-size="10" fill="#fff" font-weight="600" font-family="sans-serif">人工</text><text x="230" y="224" text-anchor="middle" font-size="10" fill="#fff" font-weight="600" font-family="sans-serif">审批</text>
      <line x1="50" y1="235" x2="50" y2="280" stroke="#a0aec0" stroke-width="1.5"/><line x1="50" y1="280" x2="140" y2="280" stroke="#a0aec0" stroke-width="1.5"/>
      <line x1="230" y1="235" x2="230" y2="280" stroke="#a0aec0" stroke-width="1.5"/><line x1="230" y1="280" x2="140" y2="280" stroke="#a0aec0" stroke-width="1.5"/>
      <line x1="140" y1="280" x2="140" y2="310" stroke="#a0aec0" stroke-width="1.5" marker-end="url(#wcc-arrow)"/>
      <circle cx="140" cy="332" r="22" fill="#e2e8f0" stroke="#4a5568" stroke-width="1.5"/><text x="140" y="337" text-anchor="middle" font-size="11" fill="#2e3545" font-family="sans-serif">结束</text>
    </svg>
    <!-- 8: Handle Failures -->
    <svg class="wcc-diagram" data-wcc-diagram="8" viewBox="0 0 300 420" xmlns="http://www.w3.org/2000/svg">
      <circle cx="110" cy="30" r="22" fill="#e2e8f0" stroke="#4a5568" stroke-width="1.5"/><text x="110" y="35" text-anchor="middle" font-size="11" fill="#2e3545" font-family="sans-serif">开始</text>
      <line x1="110" y1="52" x2="110" y2="80" stroke="#a0aec0" stroke-width="1.5" marker-end="url(#wcc-arrow)"/>
      <rect x="35" y="80" width="150" height="40" rx="6" fill="#e2e8f0" stroke="#4a5568" stroke-width="1.5"/><text x="110" y="105" text-anchor="middle" font-size="12" fill="#2e3545" font-family="sans-serif">任务 A</text>
      <circle cx="200" cy="90" r="12" fill="#f59e0b" stroke="#d97706" stroke-width="1.5"/><text x="200" y="94" text-anchor="middle" font-size="10" fill="#fff" font-weight="bold" font-family="sans-serif">!</text>
      <text x="220" y="94" font-size="9" fill="#4a5568" font-family="sans-serif">失败时重试</text>
      <line x1="110" y1="120" x2="110" y2="160" stroke="#a0aec0" stroke-width="1.5" marker-end="url(#wcc-arrow)"/>
      <rect x="35" y="160" width="150" height="40" rx="6" fill="#e2e8f0" stroke="#4a5568" stroke-width="1.5"/><text x="110" y="185" text-anchor="middle" font-size="12" fill="#2e3545" font-family="sans-serif">任务 B</text>
      <text x="200" y="175" font-size="14" fill="#4a5568" font-family="sans-serif">&#9201;</text>
      <text x="220" y="178" font-size="9" fill="#4a5568" font-family="sans-serif">x 秒后</text><text x="220" y="190" font-size="9" fill="#4a5568" font-family="sans-serif">超时</text>
      <line x1="110" y1="200" x2="110" y2="240" stroke="#a0aec0" stroke-width="1.5" marker-end="url(#wcc-arrow)"/>
      <rect x="35" y="240" width="150" height="40" rx="6" fill="#e2e8f0" stroke="#4a5568" stroke-width="1.5"/><text x="110" y="265" text-anchor="middle" font-size="12" fill="#2e3545" font-family="sans-serif">任务 C</text>
      <line x1="185" y1="260" x2="220" y2="260" stroke="#dc2626" stroke-width="1.5" stroke-dasharray="5,3" marker-end="url(#wcc-arrow-red)"/>
      <text x="228" y="258" font-size="9" fill="#dc2626" font-family="sans-serif">失败时</text>
      <line x1="110" y1="280" x2="110" y2="320" stroke="#a0aec0" stroke-width="1.5" marker-end="url(#wcc-arrow)"/>
      <circle cx="110" cy="342" r="22" fill="#e2e8f0" stroke="#4a5568" stroke-width="1.5"/><text x="110" y="347" text-anchor="middle" font-size="11" fill="#2e3545" font-family="sans-serif">结束</text>
      <defs><marker id="wcc-arrow-red" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="6" markerHeight="6" orient="auto-start-reverse"><path d="M 0 0 L 10 5 L 0 10 z" fill="#dc2626"/></marker></defs>
    </svg>
    <!-- 9: Replay Any Workflow -->
    <svg class="wcc-diagram" data-wcc-diagram="9" viewBox="-30 0 400 420" xmlns="http://www.w3.org/2000/svg">
      <circle cx="110" cy="30" r="22" fill="#e2e8f0" stroke="#4a5568" stroke-width="1.5"/><text x="110" y="35" text-anchor="middle" font-size="11" fill="#2e3545" font-family="sans-serif">开始</text>
      <line x1="110" y1="52" x2="110" y2="80" stroke="#a0aec0" stroke-width="1.5" marker-end="url(#wcc-arrow)"/>
      <rect x="35" y="80" width="150" height="40" rx="6" fill="#e2e8f0" stroke="#4a5568" stroke-width="1.5"/><text x="110" y="105" text-anchor="middle" font-size="12" fill="#2e3545" font-family="sans-serif">任务 A</text>
      <line x1="110" y1="120" x2="110" y2="160" stroke="#a0aec0" stroke-width="1.5" marker-end="url(#wcc-arrow)"/>
      <rect x="35" y="160" width="150" height="40" rx="6" fill="#e2e8f0" stroke="#4a5568" stroke-width="1.5"/><text x="110" y="185" text-anchor="middle" font-size="12" fill="#2e3545" font-family="sans-serif">任务 B</text>
      <line x1="110" y1="200" x2="110" y2="240" stroke="#a0aec0" stroke-width="1.5" marker-end="url(#wcc-arrow)"/>
      <rect x="35" y="240" width="150" height="40" rx="6" fill="#dc2626" stroke="#dc2626" stroke-width="1.5"/><text x="110" y="265" text-anchor="middle" font-size="12" fill="#fff" font-weight="600" font-family="sans-serif">任务 C</text>
      <text x="110" y="300" text-anchor="middle" font-size="10" fill="#dc2626" font-family="sans-serif" font-weight="600">FAILED</text>
      <!-- Restart arrow -->
      <path d="M 35 260 C -20 260 -20 30 60 30" stroke="#06d6a0" stroke-width="2" fill="none" stroke-dasharray="5,3" marker-end="url(#wcc-arrow-teal)"/>
      <text x="-8" y="150" text-anchor="middle" font-size="9" fill="#06d6a0" font-family="sans-serif" font-weight="600" transform="rotate(-90 -8 150)">重新开始</text>
      <!-- Rerun arrow -->
      <path d="M 185 260 C 240 260 240 180 185 180" stroke="#3b82f6" stroke-width="2" fill="none" stroke-dasharray="5,3" marker-end="url(#wcc-arrow-blue)"/>
      <text x="248" y="220" text-anchor="start" font-size="9" fill="#3b82f6" font-family="sans-serif" font-weight="600">重新执行</text>
      <!-- Retry arrow -->
      <path d="M 185 250 C 300 250 300 240 185 240" stroke="#f59e0b" stroke-width="2" fill="none" stroke-dasharray="5,3" marker-end="url(#wcc-arrow-amber)"/>
      <text x="290" y="258" text-anchor="start" font-size="9" fill="#f59e0b" font-family="sans-serif" font-weight="600">重试</text>
      <defs>
        <marker id="wcc-arrow-blue" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="6" markerHeight="6" orient="auto-start-reverse"><path d="M 0 0 L 10 5 L 0 10 z" fill="#3b82f6"/></marker>
        <marker id="wcc-arrow-amber" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="6" markerHeight="6" orient="auto-start-reverse"><path d="M 0 0 L 10 5 L 0 10 z" fill="#f59e0b"/></marker>
      </defs>
    </svg>
    <!-- 10: Integrate -->
    <svg class="wcc-diagram" data-wcc-diagram="10" viewBox="-20 0 350 350" xmlns="http://www.w3.org/2000/svg">
      <rect x="75" y="20" width="150" height="50" rx="8" fill="#06d6a0" stroke="#05c792" stroke-width="1.5"/><text x="150" y="42" text-anchor="middle" font-size="11" fill="#fff" font-weight="600" font-family="sans-serif">Conductor</text><text x="150" y="58" text-anchor="middle" font-size="10" fill="#fff" font-family="sans-serif">工作流引擎</text>
      <line x1="75" y1="45" x2="20" y2="115" stroke="#a0aec0" stroke-width="1.5" marker-end="url(#wcc-arrow)"/>
      <line x1="120" y1="70" x2="90" y2="115" stroke="#a0aec0" stroke-width="1.5" marker-end="url(#wcc-arrow)"/>
      <line x1="180" y1="70" x2="210" y2="115" stroke="#a0aec0" stroke-width="1.5" marker-end="url(#wcc-arrow)"/>
      <line x1="225" y1="45" x2="280" y2="115" stroke="#a0aec0" stroke-width="1.5" marker-end="url(#wcc-arrow)"/>
      <rect x="-10" y="115" width="70" height="36" rx="6" fill="#e2e8f0" stroke="#4a5568" stroke-width="1.5"/><text x="25" y="138" text-anchor="middle" font-size="10" fill="#2e3545" font-family="sans-serif">Kafka</text>
      <rect x="70" y="115" width="70" height="36" rx="6" fill="#e2e8f0" stroke="#4a5568" stroke-width="1.5"/><text x="105" y="138" text-anchor="middle" font-size="10" fill="#2e3545" font-family="sans-serif">NATS</text>
      <rect x="170" y="115" width="70" height="36" rx="6" fill="#e2e8f0" stroke="#4a5568" stroke-width="1.5"/><text x="205" y="138" text-anchor="middle" font-size="10" fill="#2e3545" font-family="sans-serif">SQS</text>
      <rect x="250" y="115" width="70" height="36" rx="6" fill="#e2e8f0" stroke="#4a5568" stroke-width="1.5"/><text x="285" y="138" text-anchor="middle" font-size="10" fill="#2e3545" font-family="sans-serif">AMQP</text>
      <line x1="150" y1="70" x2="150" y2="200" stroke="#a0aec0" stroke-width="1.5" marker-end="url(#wcc-arrow)"/>
      <rect x="80" y="200" width="140" height="36" rx="6" fill="#e2e8f0" stroke="#4a5568" stroke-width="1.5"/><text x="150" y="223" text-anchor="middle" font-size="10" fill="#2e3545" font-family="sans-serif">Webhooks</text>
    </svg>
    <!-- 11: Debug Visually -->
    <svg class="wcc-diagram" data-wcc-diagram="11" viewBox="0 0 300 420" xmlns="http://www.w3.org/2000/svg">
      <circle cx="110" cy="30" r="22" fill="#e2e8f0" stroke="#4a5568" stroke-width="1.5"/><text x="110" y="35" text-anchor="middle" font-size="11" fill="#2e3545" font-family="sans-serif">开始</text>
      <line x1="110" y1="52" x2="110" y2="80" stroke="#a0aec0" stroke-width="1.5" marker-end="url(#wcc-arrow)"/>
      <rect x="35" y="80" width="150" height="40" rx="6" fill="#e2e8f0" stroke="#4a5568" stroke-width="1.5"/><text x="110" y="105" text-anchor="middle" font-size="12" fill="#2e3545" font-family="sans-serif">任务 A</text>
      <line x1="110" y1="120" x2="110" y2="160" stroke="#a0aec0" stroke-width="1.5" marker-end="url(#wcc-arrow)"/>
      <polygon points="110,160 150,195 110,230 70,195" fill="#e2e8f0" stroke="#4a5568" stroke-width="1.5"/><text x="110" y="198" text-anchor="middle" font-size="10" fill="#2e3545" font-family="sans-serif">Switch</text>
      <line x1="110" y1="230" x2="110" y2="260" stroke="#a0aec0" stroke-width="1.5" marker-end="url(#wcc-arrow)"/>
      <rect x="35" y="260" width="150" height="40" rx="6" fill="#dc2626" stroke="#dc2626" stroke-width="1.5"/><text x="110" y="285" text-anchor="middle" font-size="12" fill="#fff" font-weight="600" font-family="sans-serif">任务 D</text>
      <rect x="180" y="165" width="110" height="60" rx="8" fill="#fff" stroke="#a0aec0" stroke-width="1" stroke-dasharray="4,3"/>
      <text x="235" y="183" text-anchor="middle" font-size="9" fill="#4a5568" font-family="sans-serif">查看输入、</text>
      <text x="235" y="195" text-anchor="middle" font-size="9" fill="#4a5568" font-family="sans-serif">拉取日志、</text>
      <text x="235" y="207" text-anchor="middle" font-size="9" fill="#4a5568" font-family="sans-serif">从此处</text>
      <text x="235" y="219" text-anchor="middle" font-size="9" fill="#4a5568" font-family="sans-serif">重启</text>
      <line x1="180" y1="195" x2="155" y2="195" stroke="#a0aec0" stroke-width="1" stroke-dasharray="3,3"/>
      <line x1="110" y1="300" x2="110" y2="340" stroke="#a0aec0" stroke-width="1.5" marker-end="url(#wcc-arrow)"/>
      <circle cx="110" cy="362" r="22" fill="#e2e8f0" stroke="#4a5568" stroke-width="1.5"/><text x="110" y="367" text-anchor="middle" font-size="11" fill="#2e3545" font-family="sans-serif">结束</text>
    </svg>
    <!-- 12: Scale -->
    <svg class="wcc-diagram" data-wcc-diagram="12" viewBox="0 0 320 300" xmlns="http://www.w3.org/2000/svg">
      <rect x="90" y="10" width="140" height="40" rx="8" fill="#06d6a0" stroke="#05c792" stroke-width="1.5"/><text x="160" y="35" text-anchor="middle" font-size="11" fill="#fff" font-weight="600" font-family="sans-serif">负载均衡器</text>
      <line x1="120" y1="50" x2="60" y2="80" stroke="#a0aec0" stroke-width="1.5" marker-end="url(#wcc-arrow)"/>
      <line x1="200" y1="50" x2="260" y2="80" stroke="#a0aec0" stroke-width="1.5" marker-end="url(#wcc-arrow)"/>
      <rect x="10" y="80" width="100" height="55" rx="6" fill="#e2e8f0" stroke="#4a5568" stroke-width="1.5"/><text x="60" y="102" text-anchor="middle" font-size="10" fill="#2e3545" font-weight="600" font-family="sans-serif">实例 1</text><text x="60" y="116" text-anchor="middle" font-size="9" fill="#4a5568" font-family="sans-serif">API 服务器</text><text x="60" y="128" text-anchor="middle" font-size="9" fill="#4a5568" font-family="sans-serif">+ Sweeper</text>
      <rect x="210" y="80" width="100" height="55" rx="6" fill="#e2e8f0" stroke="#4a5568" stroke-width="1.5"/><text x="260" y="102" text-anchor="middle" font-size="10" fill="#2e3545" font-weight="600" font-family="sans-serif">实例 2</text><text x="260" y="116" text-anchor="middle" font-size="9" fill="#4a5568" font-family="sans-serif">API 服务器</text><text x="260" y="128" text-anchor="middle" font-size="9" fill="#4a5568" font-family="sans-serif">+ Sweeper</text>
      <line x1="60" y1="135" x2="60" y2="175" stroke="#a0aec0" stroke-width="1.5" marker-end="url(#wcc-arrow)"/>
      <line x1="260" y1="135" x2="260" y2="175" stroke="#a0aec0" stroke-width="1.5" marker-end="url(#wcc-arrow)"/>
      <line x1="160" y1="195" x2="160" y2="195" stroke="#a0aec0" stroke-width="1.5"/>
      <rect x="30" y="175" width="260" height="55" rx="8" fill="rgba(6,214,160,0.08)" stroke="#06d6a0" stroke-width="1.5"/>
      <text x="160" y="195" text-anchor="middle" font-size="10" font-weight="600" fill="#06d6a0" font-family="sans-serif">共享后端</text>
      <text x="160" y="215" text-anchor="middle" font-size="9" fill="#4a5568" font-family="sans-serif">数据库  ·  队列  ·  索引  ·  锁</text>
    </svg>
    <!-- 13: Agents -->
    <svg class="wcc-diagram" data-wcc-diagram="13" viewBox="0 0 320 330" xmlns="http://www.w3.org/2000/svg">
      <rect x="85" y="15" width="150" height="42" rx="7" fill="#e2e8f0" stroke="#4a5568" stroke-width="1.5"/><text x="160" y="40" text-anchor="middle" font-size="11" fill="#2e3545" font-family="sans-serif">Conductor 工作流</text>
      <line x1="160" y1="57" x2="160" y2="90" stroke="#a0aec0" stroke-width="1.5" marker-end="url(#wcc-agent-arrow)"/>
      <rect x="95" y="90" width="130" height="46" rx="7" fill="#06d6a0" stroke="#05c792" stroke-width="1.5"/><text x="160" y="117" text-anchor="middle" font-size="12" fill="#fff" font-weight="600" font-family="sans-serif">AGENT 任务</text>
      <line x1="125" y1="136" x2="70" y2="185" stroke="#a0aec0" stroke-width="1.5" marker-end="url(#wcc-agent-arrow)"/>
      <line x1="195" y1="136" x2="250" y2="185" stroke="#a0aec0" stroke-width="1.5" marker-end="url(#wcc-agent-arrow)"/>
      <rect x="10" y="185" width="120" height="55" rx="7" fill="#e2e8f0" stroke="#4a5568" stroke-width="1.5"/><text x="70" y="208" text-anchor="middle" font-size="10" fill="#2e3545" font-weight="600" font-family="sans-serif">Conductor Agent</text><text x="70" y="224" text-anchor="middle" font-size="9" fill="#4a5568" font-family="sans-serif">构建并运行</text>
      <rect x="190" y="185" width="120" height="55" rx="7" fill="#e2e8f0" stroke="#4a5568" stroke-width="1.5"/><text x="250" y="208" text-anchor="middle" font-size="10" fill="#2e3545" font-weight="600" font-family="sans-serif">A2A 智能体</text><text x="250" y="224" text-anchor="middle" font-size="9" fill="#4a5568" font-family="sans-serif">远程编排</text>
      <line x1="70" y1="240" x2="70" y2="275" stroke="#a0aec0" stroke-width="1.5"/><line x1="250" y1="240" x2="250" y2="275" stroke="#a0aec0" stroke-width="1.5"/><line x1="70" y1="275" x2="250" y2="275" stroke="#a0aec0" stroke-width="1.5"/><line x1="160" y1="275" x2="160" y2="300" stroke="#a0aec0" stroke-width="1.5" marker-end="url(#wcc-agent-arrow)"/>
      <circle cx="160" cy="310" r="16" fill="#e2e8f0" stroke="#4a5568" stroke-width="1.5"/><text x="160" y="314" text-anchor="middle" font-size="9" fill="#2e3545" font-family="sans-serif">完成</text>
      <defs><marker id="wcc-agent-arrow" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="6" markerHeight="6" orient="auto"><path d="M 0 0 L 10 5 L 0 10 z" fill="#a0aec0"/></marker></defs>
    </svg>
  </div>
</div>

<style>
.wcc-widget{display:flex;gap:0;border:1px solid var(--c-cloud,#e2e8f0);border-radius:var(--r-md,10px);overflow:hidden;margin:1.5rem 0 2rem;min-height:420px;background:var(--c-white,#fff)}
.wcc-left{flex:0 0 52%;display:flex;flex-direction:column;border-right:1px solid var(--c-cloud,#e2e8f0);min-height:420px}
.wcc-right{flex:1;display:flex;align-items:center;justify-content:center;padding:2rem;background:var(--c-snow,#f8fafc)}
.wcc-item{display:none;flex:1;cursor:default;padding:2rem;animation:wcc-slide-in .22s ease}
.wcc-item.wcc-active{display:block;background:var(--c-fog,#f1f4f8)}
.wcc-header{display:flex;align-items:center;justify-content:space-between}
.wcc-title{font-family:var(--font-body,sans-serif);font-size:1.1rem;font-weight:600;color:var(--c-teal,#06d6a0)}
.wcc-chevron{display:none}
.wcc-body{padding-top:1rem;font-size:.9rem;line-height:1.65;color:var(--c-slate,#4a5568)}
.wcc-body a{color:var(--c-teal,#06d6a0);text-decoration:none;font-weight:500}
.wcc-body a:hover{text-decoration:underline}
.wcc-controls{display:flex;align-items:center;justify-content:space-between;gap:.75rem;padding:1rem 1.25rem;border-top:1px solid var(--c-cloud,#e2e8f0);background:var(--c-white,#fff)}
.wcc-control{display:inline-flex;align-items:center;gap:.55rem;padding:.45rem .7rem;border:0;background:transparent;color:var(--c-charcoal,#2e3545);font-family:var(--font-body,sans-serif);font-size:.8rem;font-weight:600;line-height:1;white-space:nowrap;cursor:pointer}
.wcc-control-icon{display:inline-flex;width:2rem;height:2rem;align-items:center;justify-content:center;border-radius:50%;background:var(--c-blue,#3b82f6);color:#fff;font-size:1.2rem;font-weight:700;line-height:1;transition:transform .15s ease,background .15s ease}
.wcc-control:hover:not(:disabled){color:var(--c-blue,#3b82f6)}
.wcc-control:hover:not(:disabled) .wcc-control-icon{background:#2563eb;transform:translateX(2px)}
.wcc-control[data-wcc-direction="previous"]:hover:not(:disabled) .wcc-control-icon{transform:translateX(-2px)}
.wcc-control:disabled{opacity:.4;cursor:not-allowed}
.wcc-diagram{display:none;max-width:100%;max-height:400px;width:auto;height:auto}
.wcc-diagram.wcc-visible{display:block}
@keyframes wcc-slide-in{from{opacity:0;transform:translateX(12px)}to{opacity:1;transform:translateX(0)}}
@media(max-width:768px){.wcc-widget{flex-direction:column}.wcc-left{flex:none;border-right:none;border-bottom:1px solid var(--c-cloud,#e2e8f0);min-height:260px}.wcc-item{padding:1.5rem}.wcc-right{min-height:300px}.wcc-controls{padding:1rem}}
</style>

<script>
document.addEventListener("DOMContentLoaded",function(){var items=Array.prototype.slice.call(document.querySelectorAll(".wcc-item"));var diagrams=document.querySelectorAll(".wcc-diagram");var controls=document.querySelectorAll(".wcc-control");var activeIndex=Math.max(0,items.findIndex(function(item){return item.classList.contains("wcc-active")}));function show(index,focus){activeIndex=Math.max(0,Math.min(index,items.length-1));items.forEach(function(item,itemIndex){var active=itemIndex===activeIndex;item.classList.toggle("wcc-active",active);item.setAttribute("aria-selected",String(active));item.setAttribute("aria-hidden",String(!active));item.tabIndex=active?0:-1});var selected=items[activeIndex];var diagramIndex=selected.getAttribute("data-wcc");diagrams.forEach(function(diagram){diagram.classList.toggle("wcc-visible",diagram.getAttribute("data-wcc-diagram")===diagramIndex)});controls.forEach(function(control){control.disabled=control.getAttribute("data-wcc-direction")==="previous"?activeIndex===0:activeIndex===items.length-1});if(focus)selected.focus()}items.forEach(function(item,itemIndex){item.addEventListener("keydown",function(event){if(event.key==="ArrowRight"){event.preventDefault();show(activeIndex+1,true)}else if(event.key==="ArrowLeft"){event.preventDefault();show(activeIndex-1,true)}else if(event.key==="Home"){event.preventDefault();show(0,true)}else if(event.key==="End"){event.preventDefault();show(items.length-1,true)}});item.addEventListener("click",function(){show(itemIndex,false)})});controls.forEach(function(control){control.addEventListener("click",function(){show(activeIndex+(control.getAttribute("data-wcc-direction")==="next"?1:-1),true)})});show(activeIndex,false)});
</script>

## 核心构建块

- **[工作流](workflows.md)** — 流程的蓝图。工作流是一个 JSON 文档，
  描述任务有向图、其依赖关系、输入/输出映射和故障处理策略。
- **[任务](tasks.md)** — Conductor 工作流的基本构建块。任务可以是系统
  任务（由引擎执行）或工作者任务（由轮询获取工作的外部工作者执行）。
- **[工作者](workers.md)** — 执行 Conductor 工作流中任务的代码。工作者是
  语言无关的进程，轮询 Conductor 服务器，执行业务逻辑并
  上报结果。
- **[智能体](agents.md)（`AGENT` 任务）** — 调用已部署的 Conductor Agent 或远程 A2A
  智能体，作为工作流内部的持久化步骤。

## 支持的平台与集成

Conductor 开箱即用的支持内容速查：

| 领域 | 支持 |
|---|---|
| [工作者 SDK](../../documentation/clientsdks/index.md) | Java, Python, Go, JavaScript, C#, Clojure, Ruby, Rust |
| [LLM 提供商](../ai/llm-orchestration.md#supported-llm-providers) | 14+，包括 OpenAI、Anthropic、Gemini、Bedrock、Mistral 和 Azure OpenAI |
| [工具调用](../ai/mcp-guide.md) | MCP（Model Context Protocol） |
| [向量数据库](../ai/llm-orchestration.md) | Pinecone, pgvector, MongoDB Atlas |
| [事件代理](../how-tos/event-bus.md) | Kafka, NATS JetStream, SQS, AMQP, Azure Service Bus |
| [持久化后端](../running/deploy.md) | PostgreSQL, MySQL, Redis, Cassandra, SQLite |

## 深入阅读

- [架构](../architecture/index.md) — 系统设计与组件
- [持久化执行](../../architecture/durable-execution.md) — 故障语义与状态持久化
- [智能体与 AI](../ai/index.md) — LLM 编排模式与智能体工作流
