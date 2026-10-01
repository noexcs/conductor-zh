---
hide:
  - navigation
  - toc
description: Conductor 是一个用于构建生产级 AI 智能体和工作流的开源平台。最初由 Netflix Engineering 创建——与云无关、与语言无关、与部署方式无关。
---

<div class="home-wrapper">

<div class="hero">
  <div class="hero-actions">
    <a href="quickstart/index.html" class="btn-primary">快速开始<span class="btn-arrow">&rarr;</span></a>
    <p class="home-skills-line">正在使用 AI 编程智能体？安装 <a href="devguide/how-tos/conductor-skills.html">Conductor Skills</a>。</p>
  </div>
</div>

<div class="home-section home-section--alt">
  <div class="section-header-inline">
    <h2>快速开始</h2>
  </div>
  <div class="integration-action-grid integration-action-grid--three">
    <a class="integration-action-card" href="quickstart/index.html">
      <span class="home-card-icon"><svg viewBox="0 0 24 24" width="22" height="22" fill="none" stroke="currentColor" stroke-width="1.7" stroke-linecap="round" stroke-linejoin="round"><polyline points="4 17 10 11 4 5"/><line x1="12" y1="19" x2="20" y2="19"/></svg></span>
      <span class="integration-action-card__title">快速入门指南</span>
      <span>在本地运行 Conductor，注册工作流和智能体，并端到端地执行它。</span>
      <span class="home-card-cta">从这里开始 &rarr;</span>
    </a>
    <a class="integration-action-card" href="https://developer.orkescloud.com/">
      <span class="home-card-icon"><svg viewBox="0 0 24 24" width="22" height="22" fill="none" stroke="currentColor" stroke-width="1.7" stroke-linecap="round" stroke-linejoin="round"><path d="M17.5 19a4.5 4.5 0 1 0-.4-9A7 7 0 1 0 4 14.9"/><path d="M12 12v9"/><path d="m8 17 4-4 4 4"/></svg></span>
      <span class="integration-action-card__title">免费 Orkes Conductor 开发者版</span>
      <span>使用免费的托管版 Conductor 快速上手。</span>
      <span class="home-card-cta">免费开始 &rarr;</span>
    </a>
    <a class="integration-action-card" href="devguide/how-tos/conductor-skills.html">
      <span class="home-card-icon"><svg viewBox="0 0 24 24" width="22" height="22" fill="none" stroke="currentColor" stroke-width="1.7" stroke-linecap="round" stroke-linejoin="round"><path d="m12 3 1.9 5.1L19 10l-5.1 1.9L12 17l-1.9-5.1L5 10l5.1-1.9Z"/><path d="m19 15 .8 2.2L22 18l-2.2.8L19 21l-.8-2.2L16 18l2.2-.8Z"/></svg></span>
      <span class="integration-action-card__title">Conductor Skills</span>
      <span>正在使用 AI 编程智能体？安装 Conductor Skills，让它能够构建和运维工作流。</span>
      <span class="home-card-cta">安装 Skills &rarr;</span>
    </a>
    <a class="integration-action-card" href="devguide/ai/cookbook/index.html">
      <span class="home-card-icon"><svg viewBox="0 0 24 24" width="22" height="22" fill="none" stroke="currentColor" stroke-width="1.7" stroke-linecap="round" stroke-linejoin="round"><path d="M2 3h6a4 4 0 0 1 4 4v14a3 3 0 0 0-3-3H2z"/><path d="M22 3h-6a4 4 0 0 0-4 4v14a3 3 0 0 1 3-3h7z"/></svg></span>
      <span class="integration-action-card__title">AI Cookbook</span>
      <span>完整可运行的 AI 工作流配方：智能体、工具、审批与交付。</span>
      <span class="home-card-cta">打开 Cookbook &rarr;</span>
    </a>
    <a class="integration-action-card" href="devguide/running/deploy.html">
      <span class="home-card-icon"><svg viewBox="0 0 24 24" width="22" height="22" fill="none" stroke="currentColor" stroke-width="1.7" stroke-linecap="round" stroke-linejoin="round"><ellipse cx="12" cy="5" rx="9" ry="3"/><path d="M3 5v14a9 3 0 0 0 18 0V5"/><path d="M3 12a9 3 0 0 0 18 0"/></svg></span>
      <span class="integration-action-card__title">自托管</span>
      <span>安装最新发布版，使用 Docker、共享持久化和生产就绪的拓扑部署 Conductor OSS。</span>
      <span class="home-card-cta">部署 OSS &rarr;</span>
    </a>
    <a class="integration-action-card" href="devguide/cookbook/index.html">
      <span class="home-card-icon"><svg viewBox="0 0 24 24" width="22" height="22" fill="none" stroke="currentColor" stroke-width="1.7" stroke-linecap="round" stroke-linejoin="round"><path d="M4 19.5A2.5 2.5 0 0 1 6.5 17H20"/><path d="M6.5 2H20v20H6.5A2.5 2.5 0 0 1 4 19.5v-15A2.5 2.5 0 0 1 6.5 2z"/></svg></span>
      <span class="integration-action-card__title">设计模式</span>
      <span>面向微服务、定时器、事件驱动工作流和 AI 编排的参考模式。</span>
      <span class="home-card-cta">浏览模式 &rarr;</span>
    </a>
  </div>
</div>

<div class="home-section">
  <div class="section-header-inline">
    <h2>用任意语言编写代码</h2>
  </div>
  <div class="home-sdk-grid">
    <a class="home-sdk-card" href="documentation/clientsdks/python-sdk.html">
      <img src="https://orkes.io/content/img/Python_logo.svg" alt="" />
      <span class="home-sdk-card__meta"><strong>Python</strong><span>conductor-oss/python-sdk</span></span>
      <span class="home-sdk-card__arrow">&rarr;</span>
    </a>
    <a class="home-sdk-card" href="documentation/clientsdks/java-sdk.html">
      <img src="https://orkes.io/content/img/java.svg" alt="" />
      <span class="home-sdk-card__meta"><strong>Java</strong><span>conductor-oss/java-sdk</span></span>
      <span class="home-sdk-card__arrow">&rarr;</span>
    </a>
    <a class="home-sdk-card" href="documentation/clientsdks/js-sdk.html">
      <img src="https://orkes.io/content/img/JavaScript_logo_2.svg" alt="" />
      <span class="home-sdk-card__meta"><strong>TypeScript</strong><span>conductor-oss/javascript-sdk</span></span>
      <span class="home-sdk-card__arrow">&rarr;</span>
    </a>
    <a class="home-sdk-card" href="documentation/clientsdks/csharp-sdk.html">
      <img src="https://orkes.io/content/img/csharp.png" alt="" />
      <span class="home-sdk-card__meta"><strong>.NET</strong><span>conductor-oss/csharp-sdk</span></span>
      <span class="home-sdk-card__arrow">&rarr;</span>
    </a>
    <a class="home-sdk-card" href="documentation/clientsdks/go-sdk.html">
      <img src="https://orkes.io/content/img/Go_Logo_Blue.svg" alt="" />
      <span class="home-sdk-card__meta"><strong>Go</strong><span>conductor-oss/go-sdk</span></span>
      <span class="home-sdk-card__arrow">&rarr;</span>
    </a>
    <a class="home-sdk-card" href="documentation/clientsdks/ruby-sdk.html">
      <img src="https://upload.wikimedia.org/wikipedia/commons/7/73/Ruby_logo.svg" alt="" />
      <span class="home-sdk-card__meta"><strong>Ruby</strong><span>conductor-oss/ruby-sdk</span></span>
      <span class="home-sdk-card__arrow">&rarr;</span>
    </a>
    <a class="home-sdk-card" href="documentation/clientsdks/rust-sdk.html">
      <img src="https://upload.wikimedia.org/wikipedia/commons/d/d5/Rust_programming_language_black_logo.svg" alt="" />
      <span class="home-sdk-card__meta"><strong>Rust</strong><span>conductor-oss/rust-sdk</span></span>
      <span class="home-sdk-card__arrow">&rarr;</span>
    </a>
  </div>
</div>

<div class="home-section home-section--alt">
  <div class="section-header-inline">
    <h2>更多资源</h2>
  </div>
  <div class="integration-action-grid integration-action-grid--three">
    <a class="integration-action-card" href="https://orkes.io/blog">
      <span class="home-card-icon"><svg viewBox="0 0 24 24" width="22" height="22" fill="none" stroke="currentColor" stroke-width="1.7" stroke-linecap="round" stroke-linejoin="round"><path d="M12 20h9"/><path d="M16.5 3.5a2.1 2.1 0 0 1 3 3L7 19l-4 1 1-4Z"/></svg></span>
      <span class="integration-action-card__title">博客</span>
      <span>探索技术用例、社区文章、产品更新等更多内容。</span>
      <span class="home-card-cta">阅读博客 &rarr;</span>
    </a>
    <a class="integration-action-card" href="https://orkes.io/customers">
      <span class="home-card-icon"><svg viewBox="0 0 24 24" width="22" height="22" fill="none" stroke="currentColor" stroke-width="1.7" stroke-linecap="round" stroke-linejoin="round"><rect x="2" y="7" width="20" height="14" rx="2"/><path d="M16 21V5a2 2 0 0 0-2-2h-4a2 2 0 0 0-2 2v16"/></svg></span>
      <span class="integration-action-card__title">案例研究</span>
      <span>探索启发性的故事与用例，了解各公司如何使用 Conductor 转变其业务运营。</span>
      <span class="home-card-cta">阅读案例研究 &rarr;</span>
    </a>
    <a class="integration-action-card" href="https://www.youtube.com/@orkesio">
      <span class="home-card-icon"><svg viewBox="0 0 24 24" width="22" height="22" fill="none" stroke="currentColor" stroke-width="1.7" stroke-linecap="round" stroke-linejoin="round"><polygon points="10 8 16 12 10 16 10 8"/><rect x="2" y="4" width="20" height="16" rx="3"/></svg></span>
      <span class="integration-action-card__title">视频</span>
      <span>快速了解 Conductor 的关键功能与能力。</span>
      <span class="home-card-cta">观看视频 &rarr;</span>
    </a>
  </div>
</div>

<div class="home-section">
  <div class="section-header-inline">
    <h2>加入社区</h2>
  </div>
  <div class="integration-action-grid integration-action-grid--three">
    <a class="integration-action-card" href="https://join.slack.com/t/orkes-conductor/shared_invite/zt-3dpcskdyd-W895bJDm8psAV7viYG3jFA">
      <span class="home-card-icon"><svg viewBox="0 0 24 24" width="22" height="22" fill="none" stroke="currentColor" stroke-width="1.7" stroke-linecap="round" stroke-linejoin="round"><path d="M21 11.5a8.4 8.4 0 0 1-8.5 8.4 8.6 8.6 0 0 1-3.9-.9L3 21l2-5.4a8.3 8.3 0 0 1-1-4A8.4 8.4 0 0 1 12.5 3a8.4 8.4 0 0 1 8.5 8.5Z"/></svg></span>
      <span class="integration-action-card__title">社区</span>
      <span>加入公开的 Slack 社区，提问并分享资源。</span>
      <span class="home-card-cta">加入 Slack &rarr;</span>
    </a>
    <a class="integration-action-card" href="resources/contribute/index.html">
      <span class="home-card-icon"><svg viewBox="0 0 24 24" width="22" height="22" fill="none" stroke="currentColor" stroke-width="1.7" stroke-linecap="round" stroke-linejoin="round"><circle cx="18" cy="18" r="3"/><circle cx="6" cy="6" r="3"/><path d="M6 21V9a9 9 0 0 0 9 9"/></svg></span>
      <span class="integration-action-card__title">贡献指南</span>
      <span>提交 pull request、报告 issue，并查看项目的安全与贡献政策。</span>
      <span class="home-card-cta">为 Conductor 贡献 &rarr;</span>
    </a>
    <a class="integration-action-card" href="https://orkes.io/events">
      <span class="home-card-icon"><svg viewBox="0 0 24 24" width="22" height="22" fill="none" stroke="currentColor" stroke-width="1.7" stroke-linecap="round" stroke-linejoin="round"><rect x="3" y="4" width="18" height="18" rx="2"/><line x1="16" y1="2" x2="16" y2="6"/><line x1="8" y1="2" x2="8" y2="6"/><line x1="3" y1="10" x2="21" y2="10"/></svg></span>
      <span class="integration-action-card__title">活动</span>
      <span>在活动上亲眼看我们的演示，或报名参加我们即将进行的一场直播。</span>
      <span class="home-card-cta">查看即将举行的活动 &rarr;</span>
    </a>
  </div>
</div>

<div class="faq-section home-section home-section--alt">
  <div class="section-header-inline">
    <h2>常见问题。</h2>
  </div>
  <div class="faq-grid">
    <details class="faq-item">
      <summary>我如何用 Docker 运行 Conductor？</summary>
      <p>运行 <code>docker run -p 8080:8080 conductoross/conductor:latest</code> 即可启动包含所有依赖的 Conductor。服务器将在 <code>http://localhost:8080</code> 可用。对于使用外部持久化的生产部署，请参阅<a href="devguide/running/deploy.html">生产部署指南</a>。</p>
    </details>
    <details class="faq-item">
      <summary>Conductor 是开源的吗？</summary>
      <p>是的。Conductor 是一个完全开源的工作流引擎，采用 Apache 2.0 许可证。你可以在自己的基础设施上自托管它，没有任何供应商锁定。它支持 5 种持久化后端、6 种消息代理，并可在任何 Docker 能运行的地方运行。</p>
    </details>
    <details class="faq-item">
      <summary>这与 Netflix Conductor 是同一个东西吗？</summary>
      <p>是的。Conductor OSS 是 Netflix 将该项目捐赠给开源基金会后，原始 Netflix Conductor 仓库的延续。</p>
    </details>
    <details class="faq-item">
      <summary>这个项目得到积极维护吗？</summary>
      <p>是的。<a href="https://orkes.io">Orkes</a> 是该仓库的主要维护者，并为 Conductor 提供企业级 SaaS 平台，覆盖所有主流云服务商。</p>
    </details>
    <details class="faq-item">
      <summary>Conductor 可以扩展以处理我的工作负载吗？</summary>
      <p>Conductor 的服务器和工作者（worker）可以独立扩展。使用任务域（task domain）、并发限制、持久化配置和指标，使吞吐量与隔离度匹配你的环境。</p>
    </details>
    <details class="faq-item">
      <summary>Conductor 支持持久化执行吗？</summary>
      <p>支持。Conductor 会持久化工作流和任务状态，支持在工作者和基础设施故障后恢复，并暴露重试、超时、暂停、恢复和终止控制。</p>
    </details>
    <details class="faq-item">
      <summary>工作流完成或失败后我能重放它吗？</summary>
      <p>Conductor 支持 restart、rerun 和 retry 控制。执行历史的保留取决于配置，<code>keepLastN</code> 会有意删除较旧的循环迭代。</p>
    </details>
    <details class="faq-item">
      <summary>工作流总是异步的吗？</summary>
      <p>不是。Conductor 虽然擅长异步编排，但在需要即时结果时也支持同步的工作流执行。</p>
    </details>
    <details class="faq-item">
      <summary>我需要使用 Conductor 专用的框架吗？</summary>
      <p>不需要。Conductor 与语言和框架无关。使用你偏好的语言和框架&mdash;SDK 为 Java、Python、JavaScript、Go、C# 等提供了原生集成。</p>
    </details>
    <details class="faq-item">
      <summary>JSON 对复杂工作流来说不是太有限了吗？</summary>
      <p>JSON 让编排保持为机器可读的数据，而业务逻辑和副作用由工作者和内置任务执行。当路径在运行时选择时，请使用经过校验的运行时定义、动态任务和动态分叉。</p>
    </details>
    <details class="faq-item">
      <summary>Conductor 是低代码/无代码平台吗？</summary>
      <p>不是。Conductor 专为编写代码的开发者设计。虽然工作流可以用 JSON 定义，但其强大之处来自用你偏好的编程语言构建工作者和任务。</p>
    </details>
    <details class="faq-item">
      <summary>Conductor 能处理复杂工作流吗？</summary>
      <p>Conductor 就是专门为复杂编排设计的。它支持高级模式，包括嵌套循环、动态分支、子工作流，以及包含数千个任务的工作流。</p>
    </details>
    <details class="faq-item">
      <summary>Netflix Conductor 被弃置了吗？</summary>
      <p>没有。原始的 Netflix 仓库已过渡到 Conductor OSS，它是该项目的新家园。积极的开发和维护在这里继续进行。</p>
    </details>
    <details class="faq-item">
      <summary>Orkes Conductor 与 Conductor OSS 兼容吗？</summary>
      <p>100% 兼容。Orkes Conductor 构建在 Conductor OSS 之上，确保开源版与企业版之间的完全兼容。</p>
    </details>
    <details class="faq-item">
      <summary>Conductor 能编排 AI 智能体和 LLM 吗？</summary>
      <p>可以。Conductor 提供原生 LLM 任务、MCP 工具发现与调用、人工审批，以及用于 RAG 的向量工作流。提供商与能力细节请参阅仍在维护的 Agents &amp; AI 文档。</p>
    </details>
    <details class="faq-item">
      <summary>Conductor 为自适应智能体提供什么？</summary>
      <p>Conductor 将原生 AI 任务和 MCP 任务与持久化循环、分支、扇出、审批、重试、取消以及可检查的执行历史结合起来。从受治理的自适应图开始。</p>
    </details>
  </div>
</div>

</div>
