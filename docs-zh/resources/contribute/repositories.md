---
description: "Conductor OSS 的仓库——服务器、SDK、CLI 和 skills——以及你希望修改的代码归属哪一个。"
---

# 仓库

Conductor 分布在 [conductor-oss](https://github.com/conductor-oss) 组织下的多个仓库中。弄清楚你的修改归属哪个仓库，可以避免一次被退回的 pull request。

## 服务器与文档 { .wide-first-col }

| 仓库 | 内容 |
|---|---|
| [conductor-oss/conductor](https://github.com/conductor-oss/conductor) | 服务器：核心引擎、系统任务、持久化模块、REST 与 gRPC API、UI，以及位于 `docs/` 下的本文档站点 |

服务器端几乎一切都在这里，包括持久化和队列后端（`postgres-persistence`、`mysql-persistence`、`redis-persistence`、`cassandra-persistence`、`sqlite-persistence`）、存储模块以及 AI/agent 模块。

文档与它所描述的代码在同一个仓库，这是有意为之：对某个端点的修改与其文档页面的修改应属于同一个 pull request。

## 客户端 SDK

每种语言的 SDK 都是独立仓库，有各自的发布节奏。

| 语言 | 仓库 | 文档 |
|---|---|---|
| Java | [conductor-oss/java-sdk](https://github.com/conductor-oss/java-sdk) | [Java SDK](../../documentation/clientsdks/java-sdk.md) |
| Python | [conductor-oss/python-sdk](https://github.com/conductor-oss/python-sdk) | [Python SDK](../../documentation/clientsdks/python-sdk.md) |
| JavaScript | [conductor-oss/javascript-sdk](https://github.com/conductor-oss/javascript-sdk) | [JavaScript SDK](../../documentation/clientsdks/js-sdk.md) |
| Go | [conductor-oss/go-sdk](https://github.com/conductor-oss/go-sdk) | [Go SDK](../../documentation/clientsdks/go-sdk.md) |
| C# | [conductor-oss/csharp-sdk](https://github.com/conductor-oss/csharp-sdk) | [C# SDK](../../documentation/clientsdks/csharp-sdk.md) |
| Ruby | [conductor-oss/ruby-sdk](https://github.com/conductor-oss/ruby-sdk) | [Ruby SDK](../../documentation/clientsdks/ruby-sdk.md) |
| Rust | [conductor-oss/rust-sdk](https://github.com/conductor-oss/rust-sdk) | [Rust SDK](../../documentation/clientsdks/rust-sdk.md) |
| Clojure | [conductor-oss/clojure-sdk](https://github.com/conductor-oss/clojure-sdk) | — |

对 worker 如何轮询、重试或序列化负载的修改属于 SDK 仓库。对服务器接受哪些内容的修改属于服务器仓库。任何修改线上契约的内容两者都涉及，且服务器的修改应先合并，这样 SDK 才有可对接的对象。

## 工具 { .wide-first-col }

| 仓库 | 内容 |
|---|---|
| [conductor-oss/conductor-cli](https://github.com/conductor-oss/conductor-cli) | `conductor` CLI——工作流与任务管理、agents、调度、本地服务器控制 |
| [conductor-oss/conductor-skills](https://github.com/conductor-oss/conductor-skills) | 供与 Conductor 协作的编码 agent 使用的 skills |

## 我的修改归属哪个仓库？

| 你要修改的内容 | 仓库 |
|---|---|
| 引擎行为、系统任务、operator | `conductor` |
| REST 或 gRPC 端点 | `conductor` |
| 持久化或队列后端 | `conductor` |
| 文档页面 | `conductor`，位于 `docs/` 下 |
| UI | `conductor`，位于 `ui/` 下 |
| worker 轮询、重试、客户端侧转移 | 对应语言的 SDK 仓库 |
| CLI 命令或参数 | `conductor-cli` |
| 客户端与服务端之间的线上契约 | 先 `conductor`，后各 SDK |

## 相关页面

- [贡献概览](index.md)
- [贡献指南](../contributing.md)
- [最佳实践](best-practices.md)
