---
description: "引导式教程——通过动手实验分步学习 Conductor，内容涵盖任务工作者、定义与工作流。"
---
# 引导式教程

## 高层步骤
一般来说，要让 Conductor 为你的业务工作流工作，需要以下步骤：

1. 创建任务工作者（worker），以固定间隔轮询已调度的任务
2. 为这些工作者创建任务定义并注册它们。
3. 创建工作流定义

## 开始之前
确保你已经有一个 Conductor 实例在运行。这包括 Server 和 UI 两部分。我们建议遵循 [Docker 说明](../running/deploy.md)。

## 工具
为了进行测试和发起 API 调用，以下工具会很有用：

- Linux cURL 命令
- [Postman](https://www.postman.com) 或类似的 REST 客户端

## 开始吧
我们将首先定义一个使用系统任务（System Tasks）的简单工作流。

[下一步](first-workflow.md)
