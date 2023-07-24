---
title: Chapter Zero Of Learn LangChain 
categories: LLM
keywords: 'ChatGPT, LangChain, HuggingFace, LLM'
description: >-
  LangChain 是一个用于开发由语言模型驱动的应用程序的框架，他主要拥有 2 个能力：1. 数据感知（可以将 LLM 模型与外部数据源进行连接）; 2. 代理（允许与 LLM 模型进行交互）。
cover: >-
  https://graph.linganmin.cn/230724/883c7d735b53fe65141cd7bc53314b86?x-oss-process=image/format,webp/quality,q_10
top_img: >-
  https://graph.linganmin.cn/230724/883c7d735b53fe65141cd7bc53314b86?x-oss-process=image/format,webp/quality,q_60
abbrlink: 86b0ac2d
---

## LangChain是什么

`LangChain`是一个强大的框架，旨在帮助开发人员使用语言模型构建端到端的应用程序。它提供了一套工具、组件和接口，可简化创建由大型语言模型 (`LLM`) 和聊天模型提供支持的应用程序的过程。

简单来说，可以理解`LangChain`相当于开源版的`LLM`插件，它提供了丰富的大语言模型工具，支持在开源模型的基础上快速增强模型的能力。`LangChain`可以轻松管理与语言模型的交互，将多个组件链接在一起，并集成额外的资源，例如 API 和数据库。

## LangChain有哪些模块

`LangChain`为以下模块提供标准的、可扩展的接口和外部集成。

### Model I/O

> 任何语言模型应用程序的核心都是模型，`LangChain`提供了与任何语言模型交互的构建模块。

- Prompt
  - 模板化、动态选择和管理模型输入
- Language models
  - 通过通用接口调用语言模型
- Output parsers
  - 从模型输出中提取信息

![model-io流程图](https://graph.linganmin.cn/230724/a926366e277cbd4f64041286ed2cbaa2?x-oss-process=image/format,webp/quality,q_60)

### Data connection

> 许多`LLM`需要访问用户特殊数据，但这部分数据不属于集群训练的一部分，`LangChain`提供了通过`加载`,`转换`,`存储`和数据查询的构建模块。

- Document loaders
  - 从许多不同来源加载文档
- Document transformers
  - 拆分文档、将文档转换为问答格式、删除冗余文档等
- Text embedding models
  - 将非结构化文本转换为浮点数列表
- Vector stores
  - 存储和搜索嵌入的数据
- Retrievers
  - 查询数据

![Data connection](https://graph.linganmin.cn/230724/7f7996f61bc246dd01b4b6b10e6cab75?x-oss-process=image/format,webp/quality,q_60)

### Chains

> 单独使用`LLM`对于一个简单的应用来说是很好的，但是面对更复杂的应用程序需要链接（彼此链接或者于其他组件链接）`LLMS`。
> `LangChain`为此类`链式应用`提供了`Chain Interface`，我们一般将一个`Chain`定义为对组件的调用序列。
> 将组件组合在一个`Chain`上的想法简单但功能强大，它极大的简化了复杂应用程序的实现，并使其更加模块化，这反过来又使调试、维护和改进应用程序变得更加容易。

### Agents

> `Agent`的核心思想是使用`LLM`开选择要采取的一系列操作，在`Chains`中，一系列操作被 hardcoded 在代码中。在`Agents`中，语言模型被用作推理引擎来确定要采取哪些操作以及按什么顺序。

下面是几个关键组件：

- Agent
  - 由语言模型和提示词提供的负责决定下一步采取什么步骤的类（class），可以理解为它可以动态的帮我们选择和调用chain或者已有的工具
- Tools
  - Agent 调用的函数，包括下面两点
    - 让 Agent 可以正确运行的工具
    - 让 Agent 可以清楚理解的该工具的描述
- Toolkits
  - 通常，Agent 可以使用工具集要比使用单个工具更重要，为此，LangChain 提供了工具包的概念（完成特定目标所需的工具组）
- AgentExecutor
  - AgentExecutor 是 Agent 的运行时。

![agent](https://graph.linganmin.cn/230724/0f4e4657ba34f41d4807a8b9dc925932?x-oss-process=image/format,webp/quality,q_60)

### Memory

> 默认情况下，`Chains`和`Agents`是无状态的，这意味着它们独立处理每个传入查询（就像底层 LLM 和聊天模型本身一样），在某些应用程序中，例如聊天机器人，必须记住以前的短期和长期的交互/会话，`Memory class`便是做这件事的。
> `LangChain`提供两种形式的内存组件。首先，`LangChain`提供了帮助帮助工具来管理和操作以前的聊天信息，这些都是模块化的，其次`LangChain`提供了将这些实用程序合并到链中的简单的方法。

### Callbacks

> `LangChain`提供了一个回调系统，允许你`Hook`到`LLM`应用程序的各个阶段，这对于日志记录、监控、流出库以及其他任务是非常有用的。

---

以上便是最最基础的关于`LangChain`和其主要模块的简单介绍，下一篇会使用`LangChain`+`ChatGPT`构建一系列简单应用以熟悉`LangChain`的各个核心模块加理解。
