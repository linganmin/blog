---
title: Chapter One Of Learn LangChain 快速入门
categories: LLM
keywords: 'ChatGPT, LangChain, HuggingFace, LLM'
description: LangChain 快速入门
cover: >-
  https://graph.linganmin.cn/230727/e64f88e4a73453b4beced9914f69cfaa?x-oss-process=image/format,webp/quality,q_60
top_img: >-
  https://graph.linganmin.cn/230727/e64f88e4a73453b4beced9914f69cfaa?x-oss-process=image/format,webp/quality,q_60
abbrlink: 67196ad2
---

## 安装依赖

```bash
pip install langchain
```

## 环境设置

使用`LangChain`通常需要与一个或多个模型提供者、数据存储、API等集成。在下面的示例中，我们将使用`OpenAI`的模型 API。

### 安装 openapi 依赖包

```bash
pip install openai
```

### 设置环境

通过接口访问`OpenAI`需要一个密钥，你可以通过创建一个账户然后绑定支持的信用卡（需要美国信用卡）就可以生成自己的密钥了, [点击进入 OpenAI API Keys 管理](https://platform.openai.com/account/api-keys)，参见下图：

![openai-payment](https://graph.linganmin.cn/230725/2e6d9420754094d1fba5bae9cd45991e?x-oss-process=image/format,webp/quality,q_60)

![openai-key](https://graph.linganmin.cn/230725/7babb6317798704e5eebb6f32928ab13?x-oss-process=image/format,webp/quality,q_60)

```python
import os

os.environ["OPENAI_API_KEY"] = "..."
```

如果你不想设置环境变量，可以在初始化`OpenAI`的`LLM`时，直接通过名为`openai_api_key`的参数传递秘钥

```python
from langchain.llms import OpenAI

llm = OpenAI(openai_api_key="...")
```

## 构建应用程序

`LangChain`提供了许多可用于构建语言模型应用程序的模块，模块可以在简单的应用程序中独立使用，也可以组合起来用于更复杂的用例。

`LangChain`应用程序的核心构建模块是`LLMChain`，这包括了三个重要的东西：

- LLM
  - 语言模型是这里的核心推理引擎，为了使用`LangChain`，我们需要了解不同类型的语言模型以及如何使用它们
- Prompt Templates
  - 它为语言模型提供指令，这控制了语言模型的输出，因此了解如何构建提示，和不同的提示策略是至关重要的
- Out Parsers
  - 将原始响应从`LLM`转换为更可行的格式，使其易于使用上下游的输出

### LLMS

语言模型有两种类型，在`LangChain`中称为

- LLMS
  - 这种语言模型将字符串作为输入并返回字符串
- ChatModels
  - 这种语言模型将消息列表作为输入并返回消息

#### ChatModels

`LLMS`的输入/输出简单易懂都是一个字符串，但`ChatModels`输入是个`ChatMessage`列表，输出是单个`ChatMessage`,`ChatMessage`有两个必要组件：

- content
  - 这是消息内容
- role
  - 这是`ChatMessage`来源的实体角色

LangChain 提功了几个对象来方便区分不用的角色：

- HumanMessage
  - 来自人类/用户的聊天消息
- AIMessage
  - 来自AI/助手的聊天消息
- SystemMessage
  - 来自系统的聊天消息
- FunctionMessage
  - 来自函数调用的 ChatMessage

`LangChain`为`LLMS`和`ChatModels`提供了标准接口，暴露的标准接口的两个方法：

- predict
  - 接受一个字符串，返回一个字符串
- predict_messages
  - 接受消息列表，返回消息

```python
# 举栗
# 导入一个 LLM 和一个 ChatModel

from langchain.llms import OpenAI
from langchain.chat_models import ChatOpenAI

llm = OpenAI()
chat_model = ChatOpenAI()

```

![model-test](https://graph.linganmin.cn/230727/4aa9010f506190db7b2f1f19653e9b25?x-oss-process=image/format,webp/quality,q_60)

### Prompt templates

大多数 LLM 应用不会将用户输入直接传递到 LLM，它们会将用户输入添加到被称为提示模版的较大文本中，该文本提供有关当前特定任务的附加上下文。

`PromptTemplates`正是帮助解决这个问题，它将从用户输入到格式化的提示符的所有逻辑捆绑在一起，举栗：

```python
from langchain.prompts import PromptTemplate

prompt = PromptTemplate.from_template("我准备开一个卖{product}的店，帮我起个名字")
prompt.format(product="蛋糕")

# 我准备开一个卖蛋糕的店，帮我起个名字
```

与原始字符串格式相比，使用这些格式有几个优点。你可以“部分”去掉变量——例如你一次只能格式化一些变量。您可以将它们组合在一起，轻松地将不同的模板组合到单个提示符中

### Output Parsers

`Output Parsers`是将`LLM`的原始输出转换为可在下游使用的格式，主要有以下类型:

- 将`LLM`文本转换为结构化数据（比如 JSON）
- 将`ChatMessage`转换为字符串
- 将调用返回的信息之外的信息（如 OpenAI 函数调用）转换为字符串

在本入门指南中，我们将编写自己的输出解析器，将逗号分隔列表转换为列表的解析器。

```python
from langchain.schema import BaseOutputParser

class CommaSeparatedListOutputParser(BaseOutputParser):
  """ Pase the output of an LLM call toa comma-separated list. """

  def parse(self, text: str):
    """Parse the output of an LLM call."""
    return text.strip().split(", ")


CommaSeparatedListOutputParser().parse("hi, bye")

# ['hi', 'bye']
```

### LLMChain

下面我们将以上各部分组合成一条`Chain`，该`Chain`将获取输入变量，将这些变量传递给提示模板以创建提示，然后将提示传递给`LLM`，然后通过(可选)的输出解析器输出，这是一种将模块逻辑绑定在一起的方便的方法。

```python
import os

os.environ["OPENAI_API_KEY"] = "xxx" # 换成你自己的key

from langchain.chat_models import ChatOpenAI
from langchain.prompts.chat import (
    ChatPromptTemplate,
    SystemMessagePromptTemplate,
    HumanMessagePromptTemplate,
                                   
)
from langchain.chains import LLMChain
from langchain.schema import BaseOutputParser

class CommaSeparatedListOutputParser(BaseOutputParser):
  """ Pase the output of an LLM call toa comma-separated list. """

  def parse(self, text: str):
    """Parse the output of an LLM call."""
    return text.strip().split(", ")

template = """you are a helpful assistant who generates comma separated lists. A user will pass in a category, and you should generated 5 objects in that category in a comma separated list. ONLY return a comma separated list, and nothing more."""

system_message_prompt = SystemMessagePromptTemplate.from_template(template)

human_template = "{text}"
human_message_prompt = HumanMessagePromptTemplate.from_template(human_template)


chat_prompt = ChatPromptTemplate.from_messages([system_message_prompt,human_message_prompt])

chain = LLMChain(llm = ChatOpenAI(),prompt=chat_prompt,output_parser=CommaSeparatedListOutputParser())


chain.run("colors")

['blue', 'red', 'green', 'yellow', 'orange']

```
