---
description: "为 Conductor 做贡献——报告 issue、提交 pull request 以及参与社区的指南。"
---
# 贡献
感谢你对 Conductor 的关注！
本指南帮助你找到贡献、提问和报告 issue 的最有效方式。

行为准则
-----

请阅读我们的[行为准则](https://orkes.io/orkes-conductor-community-code-of-conduct)。

我有问题！
-----

我们有一个专门的[讨论论坛](https://github.com/conductor-oss/conductor/discussions)，用于提出“how to”类问题并讨论想法。如果你正在考虑创建一个功能请求或着手一个 Pull Request，讨论论坛是很好的起点。
*请不要通过创建 issue 来提问。*

我想贡献！
------

我们欢迎 Pull Requests，并且已经收到了许多出色的社区贡献！
创建和评审 Pull Request 需要花费大量时间。本节帮助你获得顺畅的 Pull Request 体验。

稳定分支是 [main](https://github.com/conductor-oss/conductor/tree/main)。

请仅针对 [main](https://github.com/conductor-oss/conductor/tree/main) 创建用于贡献的 pull request。

在编写任何代码之前，先在[讨论论坛](https://github.com/conductor-oss/conductor/discussions)上讨论你正在考虑的新功能，这是个很好的主意。实现一个功能往往不止一种方式。对不同选项进行一些讨论，有助于塑造出最佳方案。直接以 Pull Request 开始，存在不得不做大量修改的风险。不过有时那确实是最好的方式！用代码展示一个想法会非常有帮助；但请留意它可能是白做的工作。我们一些最好的 Pull Request 正是源自多个相互竞争的实现，这些实现帮助把它打磨到完美。

另外，请记住并非每个功能都适合 Conductor。以下几点值得考量：

* 它是否为用户增加了复杂度，或可能造成困惑？
* 它是否以任何方式破坏向后兼容性（这很少可被接受）
* 它是否需要引入新的依赖（对核心模块而言这很少可被接受）
* 该功能应该是可选启用还是默认启用。对于集成新的 Queuing recipe 或持久化模块，一个可选项启用的独立模块才是正确的选择。  
* 该功能应该实现在主 Conductor 仓库中，还是建立独立仓库更好？尤其是与其他系统的集成，独立仓库往往才是正确的选择，因为它们的生命周期会不同。

当然，对于更小的 bug 修复和改进，流程可以更轻量。

我们会尽力及时响应 Pull Requests。请留意，由于开源项目天然具有分布式的特性，受时区、周末以及我们手头正在做的其他事情的影响，对某个 PR 的响应可能会花一些时间。

我想报告一个 issue
-----

如果你发现了 bug，非常欢迎你创建一个 issue。请附上清晰的复现步骤，更好的做法是在分支中附上一个测试用例。请为 issue 起一个描述性的标题，这有助于整理 issue。

我有一个新功能的绝佳想法
----
Conductor 的许多功能都源自社区的想法。如果你认为缺少了什么，或某些使用场景可以支持得更好，请告诉我们！你可以通过在[讨论论坛](https://github.com/conductor-oss/conductor/discussions)上发起讨论来实现。请提供尽可能多的相关上下文，说明这个功能为什么有用、在何时有用。提供上下文对于“支持 XYZ”类 issue 尤为重要，因为我们可能不熟悉“XYZ”是什么以及它为何有用。如果你对如何实现这个功能有想法，也请一并附上。

一旦我们确定了方向，就该通过创建一个新 issue 来总结这个想法了。

## 代码风格
我们使用 [spotless](https://github.com/diffplug/spotless) 来强制执行项目一致的代码风格，因此请在代码变更后运行 `gradlew spotlessApply` 修复任何违规。

## 许可证 { #license }

通过贡献代码，你同意按照 APLv2 的条款对你的贡献进行授权：https://github.com/conductor-oss/conductor/blob/main/LICENSE

所有文件均以 Apache 2.0 许可证发布，如果你的新文件中没有许可证头，将自动添加以下许可证头：

```
/**
 * Copyright $YEAR Conductor authors.
 *
 * Licensed under the Apache License, Version 2.0 (the "License"); you may not use this file except in compliance with
 * the License. You may obtain a copy of the License at
 *
 * http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software distributed under the License is distributed on
 * an "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied. See the License for the
 * specific language governing permissions and limitations under the License.
 */
```
