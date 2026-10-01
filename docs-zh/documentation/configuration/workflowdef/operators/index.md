---
description: "Conductor 工作流操作符概览 — Fork、Switch、Do While、Dynamic Fork、Sub Workflow 和 Terminate 等控制流原语。"
---

# 操作符

操作符是 Conductor 中内置的原语，允许你定义工作流的控制流。它们类似于编程中的 _for 循环_、_if-else 选择_ 等结构。Conductor 支持大多数编程原语，因此你可以创建各种高级工作流。

以下是 Conductor OSS 中可用的操作符：

| 操作符                        | 描述         |
| -------------------------- | ----------------------------------------- |
| [Do While](do-while-task.md)         | Do-while 循环 / For 循环      | 
| [Dynamic](dynamic-task.md)           | 函数指针           | 
| [Dynamic Fork](dynamic-fork-task.md) | 动态并行执行 |
| [Fork](fork-task.md)                 | 静态并行执行  | 
| [Join](join-task.md)                 | 等待所有选定分支完成 |
| [Exclusive Join](exclusive-join-task.md) | 继续执行第一个选定的分支 |
| [Set Variable](set-variable-task.md)     | 工作流变量声明           |
| [Start Workflow](start-workflow-task.md) | 入口点   | 
| [Sub Workflow](sub-workflow-task.md) | 子程序  | 
| [Switch](switch-task.md)             | Switch / If..then...else 选择     | 
| [Terminate](terminate-task.md)       | 退出                       |

## 已弃用的迁移指南

`DECISION` 已弃用。新工作流请使用 [Switch](switch-task.md)。`EXCLUSIVE_JOIN` 当前有效，并已在上述文档中说明。
