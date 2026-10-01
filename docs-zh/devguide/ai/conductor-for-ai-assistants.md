---
description: 面向构建或运维 Conductor 工作流和 agent 的 AI 编码助手的权威、有源码依据的指南。
---

# 面向 AI 助手的 Conductor

当 AI 编码助手需要帮助构建、评审、运行或运维 Conductor 工作流时，把本页作为权威的起点。

## Conductor 是什么

Conductor 是一个面向工作流、自适应 agent 和 AI 系统的开源持久化执行平台。工作流是一张带版本的任务图。Conductor 持久化执行状态并协调任务调度；工作者和内置系统任务执行工作。

Conductor 支持两条互补的 AI 路径：

- **原生 AI 工作流：** 在工作流定义中组合 LLM、MCP、向量、人工审批和控制流系统任务。
- **框架编写的 agent：** 把受支持的 SDK 或框架 agent——如 OpenAI Agents、Google ADK、LangChain 或 LangGraph——编译为 Conductor 图，然后在更大的工作流中使用。

产品地图参见[智能体与 AI 概览](index.md)，受支持的框架参见[框架 agent 配方](agent-framework-recipes.md)。

## 安全编写规则

1. 操作匹配时优先使用内置系统任务。不要用 HTTP 包装器或自定义 worker 替代原生 LLM、MCP、向量、审批、等待、转换或控制流任务。
2. 每个外部副作用都必须是幂等的。Conductor 的任务投递是至少一次，任务在失败或超时后可能被重新投递。
3. 约束自适应执行。使用循环迭代上限、任务和工作流超时、有界扇出，以及已审批的能力选择。
4. 不要把凭证放进工作流输入或 prompt。改用合适的服务端集成、密钥设施或工作者环境。
5. 把生成的工作流定义当作不可信数据。用 `workflowDef` 启动之前，先校验其结构和能力白名单。
6. 重要写入前要求审批。直接使用 `HUMAN`，或使用 SDK agent 的工具审批配置。
7. 有意保持输出小。大对象存外部，通过工作流传引用。

## 选择正确的起点

| 目标 | 从这里开始 |
|---|---|
| 创建一个持久化服务工作流 | [第一个工作流](../../quickstart/first-workflow.md) |
| 构建受治理的 plan/act/evaluate 循环 | [持久化自适应图](dynamic-workflows.md) |
| 引入现有框架 agent（LangChain、ADK 等） | [框架 Agent](agent-framework-recipes.md) |
| 添加策略和审批 | [Agent 护栏](agent-guardrails.md) |
| 测试路由、工具和输出质量 | [Agent 评估](agent-evals.md) |
| 设计生产 agent 系统 | [生产 Agent 架构](production-agent-architecture.md) |
| 核对任务和 API 字段 | [工作流定义参考](../../documentation/configuration/workflowdef/index.md) |

## 持久化执行词汇表

- **工作流定义：** 带版本的任务图；运行中的工作流使用它启动时的那份定义版本。
- **任务：** 一个工作单元。内置系统任务在平台内运行；`SIMPLE` 任务由已注册的工作者执行。
- **工作流输出：** 用 `outputParameters` 从任务输出组装出的稳定契约。
- **重试：** 任务范围的恢复。重试一个失败的 `DO_WHILE` 会重启该循环的迭代历史。
- **暂停与审批：** `WAIT` 和 `HUMAN` 保持持久化执行状态，直到被解决。
- **动态任务 / 扇出：** `DYNAMIC` 在运行时选择任务；`FORK_JOIN_DYNAMIC` 创建运行时分支，后接 `JOIN`。
- **循环保留：** `keepLastN` 通过有意删除较旧的迭代历史，为长循环限定存储。

## 给出建议前先验证

把源码当作规范。查 Java 任务和 API 实现以确认运行时语义，然后查相关 SDK 源码以确认 SDK 编写的 agent、护栏和 eval 行为。只要配置好的服务端和集成可用，就运行 JSON 语法检查、严格文档构建、链接校验和本地执行。

机器可读的发现从 [llms.txt](../../llms.txt) 开始。精选的 [llms-full.txt](../../llms-full.txt) 由仓库清单中列出的源页面生成。
