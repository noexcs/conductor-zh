---
description: "为 Conductor 做贡献——给仓库加 star 并 fork，找到适合新手的 issue，并了解项目如何在各仓库之间组织。"
---

# 为 Conductor 做贡献

Conductor 采用 Apache 2.0 许可证，并以开放方式开发。服务器功能、SDK、文档和 CLI 全部位于公开仓库中，最终交付内容的很大一部分来自社区。

<div class="grid cards" markdown>

-   **给仓库加 Star**

    最快的帮助方式，也是大多数人发现本项目的方式。给 [conductor-oss/conductor](https://github.com/conductor-oss/conductor) 点一个 Star。

-   **Fork 并构建**

    在首次修改之前，先在本地克隆、构建并运行服务器。从[从源码构建](../../devguide/running/source.md)开始。

-   **找一个适合新手的 issue**

    经过分诊、容易上手，且具备足够起步上下文的 issue。浏览 [good first issue](https://github.com/conductor-oss/conductor/labels/good%20first%20issue)。

-   **动手前先问**

    对任何非平凡的需求，先发起一个 [discussion](https://github.com/conductor-oss/conductor/discussions)。这能避免返工。

</div>

## 贡献方式

写代码是最明显的一种，但并非唯一重要的方式。

| | 去向 |
|---|---|
| **修复 bug** | 拥有该代码的仓库——参见[仓库](repositories.md) |
| **添加持久化或队列后端** | 服务器仓库中的新模块，通过配置可选启用 |
| **改进 SDK** | 该语言自己的仓库 |
| **修正或扩展文档** | 服务器仓库中的 `docs/` |
| **报告 bug** | 附带复现步骤的 [Issues](https://github.com/conductor-oss/conductor/issues) |
| **提出功能** | 先在 [Discussions](https://github.com/conductor-oss/conductor/discussions) 讨论，再创建 issue |
| **解答问题** | [Discussions](https://github.com/conductor-oss/conductor/discussions) 或 [Slack](get-help.md) |
| **报告安全漏洞** | 私下报告——参见[获取帮助](get-help.md#security-issues) |

文档贡献值得特别指出。这里的文档源自对源码的推导，而非凭记忆编写，因此修正文档通常意味着打开 controller 或 SDK 方法，把页面修正为与代码实际行为一致。这使得文档成为特别好的首次贡献：你在修复一个真实问题的同时，也学会了代码库。

## 提交第一个 pull request 之前

1. **在本地构建。** [从源码构建](../../devguide/running/source.md)，然后运行 `./gradlew test`。
2. **非平凡的需求先讨论。** 先讨论过的功能才是能合并的功能。参见[贡献指南](../contributing.md)。
3. **阅读项目约定。** 接口优先设计、`core` 中的 DAO 接口、Spotless 格式化、无 mock 测试——[最佳实践](best-practices.md)。
4. **以 `main` 为目标。** 它是稳定分支，也是唯一的 PR 目标。

## 相关页面

- [仓库](repositories.md)
- [贡献指南](../contributing.md)
- [最佳实践](best-practices.md)
- [行为准则](code-of-conduct.md)
- [获取帮助](get-help.md)
