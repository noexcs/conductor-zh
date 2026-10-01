---
description: "在哪里获取 Conductor 的帮助——Slack、GitHub Discussions、issue，以及如何报告安全漏洞。"
---

# 获取帮助

选择与你需求匹配的渠道。选错了，主要代价就是等待的时间。

<div class="grid cards" markdown>

-   **Slack**

    实时提问、快速解除阻塞，与其他用户交流。[加入 Slack 社区](https://join.slack.com/t/orkes-conductor/shared_invite/zt-3dpcskdyd-W895bJDm8psAV7viYG3jFA)。

-   **GitHub Discussions**

    “怎么做”类问题、设计提案，以及值得日后查找的内容。[发起一个讨论](https://github.com/conductor-oss/conductor/discussions)。

-   **GitHub Issues**

    可复现的 bug，以及已在讨论中达成一致的功能。[提交 issue](https://github.com/conductor-oss/conductor/issues)。

-   **社区论坛**

    横跨更广泛 Conductor 社区的长篇讨论。[community.orkes.io](https://community.orkes.io/)。

</div>

## 选择哪个渠道？

| 你想要 | 使用 |
|---|---|
| 询问某事如何工作 | [Discussions](https://github.com/conductor-oss/conductor/discussions) 或 [Slack](https://join.slack.com/t/orkes-conductor/shared_invite/zt-3dpcskdyd-W895bJDm8psAV7viYG3jFA) |
| 报告一个可复现的 bug | [Issues](https://github.com/conductor-oss/conductor/issues) |
| 提出功能 | 先在 [Discussions](https://github.com/conductor-oss/conductor/discussions) 讨论，达成一致后再提 issue |
| 让 pull request 获得评审 | 打开 PR；如果一直没动静，就在 Slack 中提一下 |
| 报告安全漏洞 | 私下报告——见下文 |

**请不要通过创建 issue 来提问。** issue 跟踪器中的问题会挤占可操作的 bug，而且同样的问题在 Discussions 中往往回答得更快。

## 写好 bug 报告

一个被修复的 bug 和一个被搁置的 bug 之间的差别，几乎总是出在报告上：

- **你做了什么**，精确到足以复现——工作流定义、API 调用、配置。
- **发生了什么**，包含实际的错误和堆栈跟踪，而不是转述。
- 你期望的**是什么**。
- **你的环境**：Conductor 版本、`conductor.db.type`、`conductor.queue.type`，以及你是如何运行的。
- 如果你能做到，**在分支上写一个失败的测试**。没有什么比这更能缩短往返时间。

配置比人们想象的更重要。有几类 bug 只在特定组合下才会出现——某个数据库搭配另一种队列后端，或容器化服务器搭配另一台主机上的客户端——因此省略后端信息的报告可能根本无法复现。

## 安全问题 { #security-issues }

不要在公开的 issue、讨论或 Slack 频道中报告漏洞。请遵循 [`SECURITY.md`](https://github.com/conductor-oss/conductor/blob/main/SECURITY.md) 中的私下披露流程，以便在细节公开之前修复就能上线。

## 相关页面

- [贡献概览](index.md)
- [贡献指南](../contributing.md)
- [行为准则](code-of-conduct.md)
- [FAQ](../../devguide/faq.md)
- [调试工作流](../../devguide/how-tos/Workflows/debugging-workflows.md)
