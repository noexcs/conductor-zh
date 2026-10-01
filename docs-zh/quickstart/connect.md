---
description: 将 Conductor CLI、SDK 和代理运行时连接到 Developer Edition 或本地服务器。
---

# 连接到 Conductor

<section class="integration-hero integration-hero--workflow" aria-label="Recommended: Orkes Developer Edition" markdown="1">

## 推荐：Orkes Developer Edition

创建一个免费的[账户](https://developer.orkescloud.com/)、[应用](https://orkes.io/content/access-control-and-security/applications#configuring-applications)和[访问密钥](https://orkes.io/content/sdks/authentication#retrieving-access-keys)（在[Orkes Developer Edition](https://developer.orkescloud.com/)中）。然后设置以下环境变量。

```bash
export CONDUCTOR_SERVER_URL=https://developer.orkescloud.com/api
export CONDUCTOR_AUTH_KEY=<your-access-key>
export CONDUCTOR_AUTH_SECRET=<your-access-secret>
```

之后，你就可以配置本地 CLI 和核心 SDK。

</section>

## 安装 CLI

CLI 用于向所选的 Conductor 服务器注册工作流并启动执行。

```bash
npm install -g @conductor-oss/conductor-cli
```

## 本地服务器替代方案

当你需要一个自管的开发服务器时使用。它要求 Java 21+ 和 Node.js。

```bash
conductor server start
export CONDUCTOR_SERVER_URL=http://localhost:8080/api
conductor workflow list
```

## AI 与代理凭据

为 AI 工作流和代理配置模型访问与凭据。

- **Developer Edition：** 在[集成](https://orkes.io/content/category/integrations/ai-llm)下为你的模型提供商添加一个集成
- **本地服务器：** 在启动服务器之前[导出提供商密钥](../devguide/ai/llm-orchestration.md#supported-llm-providers)，以便服务器继承该密钥。例如：

    ```bash
    export OPENAI_API_KEY=<your-openai-api-key>
    conductor server start
    ```

## Docker

你也可以通过[官方 Docker 容器](https://hub.docker.com/r/conductoross/conductor)运行 Conductor。

```bash
docker run --rm -p 8080:8080 conductoross/conductor:latest
export CONDUCTOR_SERVER_URL=http://localhost:8080/api
conductor workflow list
```

## 下一步

当 Conductor 已启动并可访问时，选择你要构建的内容。

<div class="integration-action-grid integration-action-grid--four">
  <a class="integration-action-card" href="first-worker.html">
    <span class="integration-action-card__title">你的第一个工作流 &amp; 工作者</span>
    <span>用你选择的语言编写并运行一个持久化工作流。</span>
  </a>
  <a class="integration-action-card" href="first-agent.html">
    <span class="integration-action-card__title">运行你的第一个代理</span>
    <span>使用 Python、Java、TypeScript/JavaScript 或 C# 编写并运行一个 Conductor Agent。</span>
  </a>
  <a class="integration-action-card" href="framework-agents.html">
    <span class="integration-action-card__title">带一个框架代理</span>
    <span>通过 Conductor 运行现有的 OpenAI Agents、LangChain、LangGraph 或 Google ADK 代理。</span>
  </a>
  <a class="integration-action-card" href="first-workflow.html">
    <span class="integration-action-card__title">无代码</span>
    <span>使用 CLI 和 JSON 注册并运行工作流</span>
  </a>
</div>
