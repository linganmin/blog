---
title: 基于 ChatGLM3-6B 的微调实践
categories: LLM
keywords: 'ChatGPT, LangChain, HuggingFace, LLM'
description: LangChain 快速入门
cover: >-
  https://graph.linganmin.cn/231214/a8efd72bfa4ccbbf13ea1ae91dc334ce?x-oss-process=image/format,webp/quality,q_60
top_img: >-
  https://graph.linganmin.cn/231214/a8efd72bfa4ccbbf13ea1ae91dc334ce?x-oss-process=image/format,webp/quality,q_60
abbrlink: 41948abf
---


因为一些特殊的原因需要使用到大模型，然后便尝试微调清华大学开源的大模型`ChatGLM3-6B`，本文记录之。

`ChatGLM3`是智谱AI和清华大学 KEG 实验室联合发布的新一代对话预训练模型。详细文档参考[ChatGLM3官网仓库](https://github.com/THUDM/ChatGLM3/tree/main)。

## 环境

- 硬件
  - CPU
    - 8C
  - 内存
    - 32G
  - GPU
    - NVIDIA V100 *1（16G）
- 系统
  - ubuntu22.04
- 软件
  - pytorch2.1.0
  - tensorflow2.14.0
  - tensorflow-gpu==2.8.0
  - Python3.10
  - cu118

## ChatGLM3 部署

### 下载仓库

```bash
git clone https://github.com/THUDM/ChatGLM3
cd ChatGLM3
```

### 安装依赖

```bash
pip install -r requirements.txt
```

### 下载模型

因为众所周知的原因，国内从`Hugging Face`下载模型会相当慢，所以我们选择从国内的`modelscope`下载。

![1](https://graph.linganmin.cn/231214/42f0064108501ab32615fdd1e63be4bb?x-oss-process=image/format,webp/quality,q_60)

![2](https://graph.linganmin.cn/231214/6afc7f1d4c3cf33109eb03ec00bab87a?x-oss-process=image/format,webp/quality,q_60)

```bash
git clone https://www.modelscope.cn/ZhipuAI/chatglm3-6b.git THUDM/chatglm3-6b # 将模型clone到 THUDM 目录下
```

![3](https://graph.linganmin.cn/231214/9ab1ff4eb7cb9dd3236f1d53eda4127d?x-oss-process=image/format,webp/quality,q_60)

## 使用模型

### 命令行对话 Demo

在当前目录下新建文件`cli.py`

```python
import os
import platform
from transformers import AutoTokenizer, AutoModel
import torch

MODEL_PATH = os.environ.get('MODEL_PATH', 'THUDM/chatglm3-6b')
TOKENIZER_PATH = os.environ.get("TOKENIZER_PATH", MODEL_PATH)
DEVICE = 'cuda' if torch.cuda.is_available() else 'cpu'

# for Mac Computer like M1
# You Need Use Pytorch compiled with Metal
# DEVICE = 'mps'

# for AMD gpu likes MI100 (Not Official Steady Support yet)
# You Need Use Pytorch compiled with ROCm
# DEVICE = 'cuda'

# for Intel gpu likes A770 (Not Official Steady Support yet)
# You Need Use Pytorch compiled with oneDNN and install intel-extension-for-pytorch
# import intel_extension_for_pytorch as ipex
# DEVICE = 'xpu'

# for Moore Threads gpu like MTT S80 (Not Official Steady Support yet)
# You Need Use Pytorch compiled with Musa
# DEVICE = 'musa'



tokenizer = AutoTokenizer.from_pretrained(TOKENIZER_PATH, trust_remote_code=True)
if 'cuda' in DEVICE: # AMD, NVIDIA GPU can use Half Precision
    model = AutoModel.from_pretrained(MODEL_PATH, trust_remote_code=True).to(DEVICE).eval()
else: # CPU, Intel GPU and other GPU can use Float16 Precision Only
    model = AutoModel.from_pretrained(MODEL_PATH, trust_remote_code=True).float().to(DEVICE).eval()

os_name = platform.system()
clear_command = 'cls' if os_name == 'Windows' else 'clear'
stop_stream = False

welcome_prompt = "欢迎使用 ChatGLM3-6B 模型，输入内容即可进行对话，clear 清空对话历史，stop 终止程序"


def build_prompt(history):
    prompt = welcome_prompt
    for query, response in history:
        prompt += f"\n\n用户：{query}"
        prompt += f"\n\nChatGLM3-6B：{response}"
    return prompt


def main():
    past_key_values, history = None, []
    global stop_stream
    print(welcome_prompt)
    while True:
        query = input("\n用户：")
        if query.strip() == "stop":
            break
        if query.strip() == "clear":
            past_key_values, history = None, []
            os.system(clear_command)
            print(welcome_prompt)
            continue
        print("\nChatGLM：", end="")
        current_length = 0
        for response, history, past_key_values in model.stream_chat(tokenizer, query, history=history, top_p=1,
                                                                    temperature=0.01,
                                                                    past_key_values=past_key_values,
                                                                    return_past_key_values=True):
            if stop_stream:
                stop_stream = False
                break
            else:
                print(response[current_length:], end="", flush=True)
                current_length = len(response)
        print("")


if __name__ == "__main__":
    main()

```

启动

```bash
python cli.py
```

![5](https://graph.linganmin.cn/231214/59e66462efe62db5fdfeca6063fc219b?x-oss-process=image/format,webp/quality,q_60)

![6](https://graph.linganmin.cn/231214/a8efd72bfa4ccbbf13ea1ae91dc334ce?x-oss-process=image/format,webp/quality,q_60)

## 微调

### 数据格式

ChatGLM3 支持多轮对话微调样例和输入输出格式微调，下文我们使用输入输出格式进行微调，格式如下：

```json
[
  {
    "prompt": "<prompt text>",
    "response": "<response text>"
  }
  // ...
]
```

### 修改微调代码

```bash
cd finetune_chatmodel_demo/ && vim  scripts/finetune_pt.sh
```

```bash
# ...

BASE_MODEL_PATH=../THUDM/chatglm3-6b # 修改模型路径
DATASET_PATH=../formatted_data/train.json # 修改语料路径

#...
```

下面是我准备的简单的语料

```json
[
    {
        "prompt": "你是谁",
        "response": "我是小下人工智能助手，我可以帮你写代码、写邮件、做旅游计划哦"
    },
    {
        "prompt": "简单总结一下《三体》的故事",
        "response": "《三体》是刘慈欣创作的科幻小说三部曲的第一部，故事涉及宇宙、科技、文明发展和人类命运等多个层面。以下是《三体》的简单总结：故事的背景设定在中国文化大革命期间，主人公叶文洁目睹了父亲被枪决，深受创伤。在绝望中，她向外太空发射了一条包含了地球文明的信息的信号。 这个信号被三体文明的居民接收到，他们的星球面临毁灭性的灾难，于是决定入侵地球，寻找新的家园。三体人利用先进的科技，通过一款名为“三体”的虚拟现实游戏，筛选出地球上最具天赋和忠诚的玩家，将其引导成为地球上的“黑暗森林”棋子。在故事中，人类社会被三体人的计划和干涉逐渐影响和改变。一些人类个体因为对人类社会的绝望而投靠三体人，而其他人则努力抵抗。小说以深刻的哲学思考和科学探讨为特色，涉及到宇宙维度、人类文明的选择、道德困境等多个层面，展现了一个宏大而引人深思的故事。"
    },
    {
        "prompt": "你怎么评价《三体》里面的章北海将军",
        "response": "章北海将军是《三体》中的中国军方高级军官，他在小说中扮演了一定的角色。他的形象呈现了在面临未知威胁时，国家领导层和军方可能面临的压力和决策。章北海将军是一个为国家和人民着想的人物，他在面对不同的困境时试图寻找最佳的解决方案。在三体危机爆发后，他积极参与应对计划，试图保护地球免受外来文明的入侵。然而，在小说中，由于三体文明的高度先进和独特性，人类几乎无法找到真正有效的对抗方法。章北海将军的形象反映了作者刘慈欣对人类文明面临外部威胁时的无奈和无力的思考。整个小说中，作者通过不同的角色，包括章北海将军，揭示了人类社会在面对未知、超越自身理解的外来力量时的困境和挑战。"
    },
    {
        "prompt": "章北海是否决定了恒星级宇宙飞船研究方向",
        "response": "在《三体》中，章北海将军并没有直接决定恒星级宇宙飞船的研究方向。实际上，对于恒星级宇宙飞船的研究方向，主要是由三体危机爆发后成立的“红岸基地”科研团队来推动的。这个团队是由各领域的科学家和专家组成，致力于寻找对抗三体文明入侵的解决方案。章北海将军在小说中是国防部的高级官员，他与政府和军方有关，负责协调和支持科研工作。尽管他在小说中对于应对三体危机有一定的作用，但对于具体的科研方向和技术细节，他并不是直接的决策者。整个科研过程更多地受到科学家和研究人员的指导和推动。"
    }
]

```

### 开始微调

```bash
bash scripts/finetune_pt.sh 
```

![7](https://graph.linganmin.cn/231215/96121d06be69979b323078b934dc427c?x-oss-process=image/format,webp/quality,q_60)

![8](https://graph.linganmin.cn/231215/49a265c121ebfee6ff71f74bf78c1faf?x-oss-process=image/format,webp/quality,q_60)

![9](https://graph.linganmin.cn/231215/92ca9a391b38f69b7c89625a6364e36b?x-oss-process=image/format,webp/quality,q_60)

![](https://graph.linganmin.cn/231215/ef3132e6e3fba98de7bc86e0f0ef5800?x-oss-process=image/format,webp/quality,q_60)

### 推理验证

```bash
python inference.py \
    --pt-checkpoint "path to p-tuning checkpoint" \
    --model ../THUDM/chatglm3-6b 
```

![10](https://graph.linganmin.cn/231215/9dca4dc4a4f1fd87120f093db9a83719?x-oss-process=image/format,webp/quality,q_60)

> 因为神经网络的灾难性遗忘问题，微调后的模型往往会失去在通用领域的对话能力或者因数据较少而缺乏泛化能力。参考这篇文章[学一个忘一个？人工智能遭遇“灾难性遗忘”，克服“失忆”有何良策？](https://picture.iczhiku.com/weixin/message1587593113355.html)。

> 灾难性遗忘问题很难解决，最好的方法是把数据和通用域的训练数据一起进行训练。