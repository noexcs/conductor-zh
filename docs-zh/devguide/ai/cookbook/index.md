---
description: 面向知识、工具、智能体、审批与交付的生产级 Conductor AI 工作流入门模板。
---

# AI 配方手册（AI Cookbook）

<section class="ai-cookbook-hero" aria-label="AI Cookbook overview">
  <div class="ai-cookbook-hero__content">
    <p>本页的每个配方都是一个完整、可运行的 AI 工作流。注册定义、运行它，然后换成你自己的模型、工具和数据。这些配方的构建方式与生产环境的运行方式一致：循环有上限，工具访问受白名单限制，高风险步骤等待人工审批，每次运行都记录发生了什么。</p>
  </div>
  <svg class="ai-cookbook-hero__diagram" viewBox="0 70 540 300" role="img" aria-labelledby="ai-cookbook-diagram-title ai-cookbook-diagram-description">
    <title id="ai-cookbook-diagram-title">AI 配方手册生产级入门模型</title>
    <desc id="ai-cookbook-diagram-description">两类配方汇入一个持久化的生产级入门模板。该模板通过策略控制连接模型与工具，并产出可检查的结果。</desc>
    <defs>
      <marker id="ai-cookbook-arrow" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto">
        <path d="M 0 0 L 10 5 L 0 10 z" class="ai-cookbook-hero__arrowhead" />
      </marker>
      <marker id="ai-cookbook-arrow-control" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto">
        <path d="M 0 0 L 10 5 L 0 10 z" class="ai-cookbook-hero__arrowhead ai-cookbook-hero__arrowhead--control" />
      </marker>
    </defs>
    <rect x="20" y="82" width="220" height="72" rx="10" class="ai-cookbook-hero__family ai-cookbook-hero__family--workflows" />
    <text x="40" y="108" class="ai-cookbook-hero__label">智能体工作流</text>
    <text x="40" y="126" class="ai-cookbook-hero__detail">由图决定执行内容</text>
    <text x="40" y="142" class="ai-cookbook-hero__detail">LLM · MCP · agents · humans</text>
    <rect x="20" y="174" width="220" height="72" rx="10" class="ai-cookbook-hero__family ai-cookbook-hero__family--agents" />
    <text x="40" y="200" class="ai-cookbook-hero__label">AI 智能体</text>
    <text x="40" y="218" class="ai-cookbook-hero__detail">智能体拥有自己的循环</text>
    <text x="40" y="234" class="ai-cookbook-hero__detail">SDK · 护栏 · 记忆</text>
    <path d="M240 118 H282 M240 210 H282" class="ai-cookbook-hero__arrow" marker-end="url(#ai-cookbook-arrow)" />
    <rect x="290" y="88" width="228" height="152" rx="15" class="ai-cookbook-hero__starter" />
    <text x="404" y="120" text-anchor="middle" class="ai-cookbook-hero__starter-title">生产级入门模板</text>
    <rect x="314" y="138" width="180" height="31" rx="7" class="ai-cookbook-hero__model" />
    <text x="404" y="158" text-anchor="middle" class="ai-cookbook-hero__label">模型 / 工具 / 智能体</text>
    <path d="M404 169 V191" class="ai-cookbook-hero__arrow ai-cookbook-hero__arrow--control" marker-end="url(#ai-cookbook-arrow-control)" />
    <rect x="314" y="197" width="180" height="27" rx="7" class="ai-cookbook-hero__policy" />
    <text x="404" y="215" text-anchor="middle" class="ai-cookbook-hero__policy-text">策略 · 审批 · 限额</text>
    <path d="M404 240 V270" class="ai-cookbook-hero__arrow ai-cookbook-hero__arrow--control" marker-end="url(#ai-cookbook-arrow-control)" />
    <rect x="290" y="281" width="228" height="78" rx="13" class="ai-cookbook-hero__outcome" />
    <path d="M326 318 l10 10 22 -25" class="ai-cookbook-hero__check" />
    <text x="432" y="313" text-anchor="middle" class="ai-cookbook-hero__label">可检查的结果</text>
    <text x="432" y="332" text-anchor="middle" class="ai-cookbook-hero__detail">证据 · 状态 · 媒体引用</text>
  </svg>
</section>

## 智能体工作流

工作流图本身就是智能体。模型负责推理，但真正执行什么由 Conductor 决定：由 `SWITCH`、`DO_WHILE`、`FORK_JOIN` 和 `HUMAN` 组合起的 LLM、MCP 与智能体任务。可用动作的白名单写在定义里，而不是提示词里，因此模型无法自行扩大自身的影响范围。

其中每一种都带有让该模式可安全投入真实运行的控制手段——有界循环、强制执行的白名单、显式的拒绝路径，或人工关卡。

| 配方 | 结果 | 构建基础 |
|---|---|---|
| [RAG 智能体](rag-agent.md) | 检索、评估上下文能否作答、重试；无依据时拒绝回答而非硬答。 | `DO_WHILE`, `LLM_SEARCH_INDEX` |
| [MCP 工具调用](mcp-tool-calling.md) | 发现工具、筛选候选，并对照白名单复查模型的选择。 | `LIST_MCP_TOOLS`, `CALL_MCP_TOOL`, `SWITCH` |
| [A2A 智能体编排](a2a-orchestration.md) | 并行委派给两个远程 A2A 智能体，汇合后综合结果。 | `GET_AGENT_CARD`, `FORK_JOIN`, `AGENT` |
| [HITL 工作流](hitl-approval.md) | 起草动作，暂停等待人工，仅在明确批准后发送。 | `HUMAN`, `SWITCH`, `HTTP` |
| [带护栏的 LLM](llm-guardrails.md) | 用模式筛查、策略检查和一次有界修复为模型调用设界。 | `INLINE`, `SWITCH`, `TERMINATE` |
| [深度研究智能体](deep-research.md) | 分解目标、扇出搜索、逐轮审查覆盖度、渲染 PDF。 | `DO_WHILE`, `FORK_JOIN_DYNAMIC`, `GENERATE_PDF` |
| [A2A 委派](remote-a2a-delegation.md) | 通过 A2A 将请求交给他人运营的智能体。 | `AGENT` (`a2a`) |

## AI 智能体

智能体拥有自己的推理循环：它决定调用哪个工具、何时结束。你可以用 Python、TypeScript、Java 或 C# 的 Conductor SDK 编写智能体，也可以通过 Conductor SDK 接入用 LangChain 或 Google ADK 编写的智能体。Conductor 提供循环自身给不了的东西——每次工具调用都是一个可持久化、可单独重试的任务，审批与取消是智能体无法绕过的边界。

| 配方 | 结果 | 构建基础 |
|---|---|---|
| [工具调用智能体](agent-tool-calling.md) | 声明两个工具，让模型自行选择。 | SDK `Agent` + `@tool` |
| [带护栏的智能体](agent-guardrails.md) | 检查智能体自身输出，规则不通过时重试。 | `RegexGuardrail`, `@guardrail` |
| [多智能体交接](agent-handoff.md) | 主管智能体委派给最合适的专家智能体。 | `Strategy.HANDOFF` |
| [带记忆的智能体](agent-memory.md) | 跨会话按相关性召回事实，而非回放。 | `SemanticMemory` |
| [带 CLI 工具的智能体](agent-cli-tools.md) | 执行真实的 shell 命令，并受白名单限制。 | `cli_allowed_commands` |
| [大规模并行智能体](agent-scatter-gather.md) | 扇出到 100 个子智能体并综合结果。 | `scatter_gather()` |
| [Conductor 智能体](reusable-conductor-agent.md) | 从另一个工作流调用已稳定部署的能力。 | `AGENT` (`conductor`) |
| [LangChain 调查员](langchain-entitlement-investigator.md) | 用 LangChain 编写，通过 Conductor SDK 调用。 | `AGENT` (`conductor`) |
| [ADK 分诊](google-adk-order-triage.md) | 用 ADK 编写，通过 Conductor SDK 调用。 | `AGENT` (`conductor`) |
| [专家评审](parallel-specialist-review.md) | 用持久化扇出与汇合收集独立评审。 | `AGENT`, `FORK_JOIN`, `JOIN` |
| [智能体审批](human-approved-action.md) | 在持久化审批边界处暂停已部署的智能体。 | `AGENT`, `SWITCH`, `HUMAN` |
| [智能体取消](conductor-agent-cancellation.md) | 将父级终止传播给长时间运行的已部署智能体。 | `AGENT`, `FORK_JOIN`, `TERMINATE` |

所有定义都依赖 Conductor 对重试和超时的默认值，因此 JSON 保持可读——在提供商配额或影响范围有要求的地方添加显式限制。不要把文档、媒体和长证据放进工作流负载中；改为传递对象存储或 Files API 的引用。
