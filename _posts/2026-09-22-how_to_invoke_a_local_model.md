---
layout: post
title: "如何用 Python 和 openai 库调用一个本地模型，得到模型回答"
Author: "qp91mn64"
Created: "2026-09-22"
Last_modified: "2026-09-23"
---

首次发布：2026-09-22  
最近更新：2026-09-23

# 如何用 Python 和 openai 库调用一个本地模型，得到模型回答

使用工具：Python，openai，Ollama

其中 openai 是 OpenAI 官方 Python SDK。目前很多大模型都兼容 OpenAI 格式，故换个网址，配置好 API Key 就能用。

[Ollama](https://ollama.com/) 则是一个跑本地模型很方便的工具，就是运行本地模型对电脑配置有一定要求，模型越大，要求越高，运行速度也越慢。

模型：qwen3.5:0.8b，相对轻量，Apache 2.0 许可证，商用友好。

这里默认读者有 Python 基础，而且已经装好 Ollama。

## 导入有关模块

```Python
from openai import OpenAI
```

## 配置客户端（Client）

“客户端”？什么东西？

不明白，不过这个东西有一个作用是不用手写 HTTP 请求，只需要写：

```Python
client = OpenAI(
    base_url = "http://localhost:11434/v1",
    api_key = "ollama"
)
```

其中 `base_url` 是个网址，对于本地的 Ollama，如果没有专门配置，默认端口 11434，这里采用 OpenAI 兼容的 `http://localhost:11434/v1`（注意不是 `https`）即可，而 `api_key` 是传入的密钥，对于 Ollama，不需要放真实密钥，放个 "ollama" 就够了。

然后常用聊天补全的形式调用，也存在其他不同的调用方式。

## 聊天补全（Chat Completion）

这种形式的特点是，传入的消息放在一个列表里面，而且有 "role"、"content" 字段，其中 "role" 有 "system"、"user"、"tool"、"assistant" 之分：

- "system"：放在开头第一条消息，对应的 "content" 通常叫“系统提示词”

- "user"：放用户输入

- "tool"：工具调用结果

- "assistant"：模型之前的回复。是的，不叫 "model"，叫 "assistant"（直译为助手）。

先写系统提示词：

```Python
messages = [
    {
        "role": "system",
        "content": "You are a helpful assistant."
    }
]
```

然后把用户输入放上，这里为了演示方便，就不用实际的输入：

```Python
messages.append(
    {
        "role": "user",
        "content": "Hello World!"
    }
)
```

接着就可以传给模型了：

```Python
response = client.chat.completions.create(
    model="qwen3.5:0.8b"  # 实际用的什么模型就写什么模型名称
    messages=messages  # 消息列表
)
```

`client.chat.completions.create`？怎么这么复杂？`client` 是之前创建的“客户端”（客户端是什么？）的实例，怎么又冒出 `chat`，`completions`，`create`？

不明白，接着查资料（问 AI，偷懒），得到的结果是， `chat` 专门处理多轮对话，`completions` 是其中的“补全”部分，而 `create` 的作用是发起新的请求。所以合在一起，`client.chat.completions.create`的作用是发起新的聊天补全请求。

那么里面传的参数什么意思呢？

- `model`：模型名称，写实际的模型名即可。
- `messages`：刚才用的消息列表。

补充一些其他参数：

- `tools`：工具定义，好的工具能扩展 AI 的能力边界，写 Agent 经常用到。
- `reasoning_effort`：思考强度，目前很多模型都有思考模式，如果不需要，用 "none" 关闭。
- `max_tokens`：最大输出 token 数量，注意包括模型的思考内容，如果这个数字太小，模型输出会被截断或者看不到输出。

token 又是什么？通常翻译为“词元”，复数 tokens，可以看作一个单位，就像用“米”衡量长度，“秒”衡量时间一样，token 衡量的是模型输入/输出有“多少”。为什么要管这个？因为如果你调用的不是本地模型，相应的 API（大致理解为一种接口，能传参数）往往是按照 token 计费的。

## 调用模型

现在把上述代码片段放在一起，是不是就能得到模型的回答了呢：

```Python
from openai import OpenAI

client = OpenAI(
    base_url = "http://localhost:11434/v1",
    api_key = "ollama"
)
messages = [
    {
        "role": "system",
        "content": "You are a helpful assistant."
    }
]
messages.append(
    {
        "role": "user",
        "content": "Hello World!"
    }
)
response = client.chat.completions.create(
    model="qwen3.5:0.8b",  # 实际用的什么模型就写什么模型名称
    messages=messages,  # 消息列表
)
```

执行代码，稍微等一等，然后查看 `response` 的内容，大致长这样（取自实际测试结果）：

```
ChatCompletion(id='chatcmpl-523', choices=[Choice(finish_reason='stop', index=0, logprobs=None, message=ChatCompletionMessage(content='Hello! How can I help you out today? 😊', refusal=None, role='assistant', annotations=None, audio=None, function_call=None, tool_calls=None, reasoning='The user is greeting me with "Hello World!" which is a classic "Hello World" programming greeting. I need to respond to this greeting in a friendly and helpful way.\n\nSince I\'m an AI language model without the ability to generate content, I should respond warmly and offer assistance. I\'ll acknowledge the greeting and express how I can help with whatever they need.'))], created=1790085557, model='qwen3.5:0.8b', object='chat.completion', service_tier=None, system_fingerprint='fp_ollama', usage=CompletionUsage(completion_tokens=90, prompt_tokens=24, total_tokens=114, completion_tokens_details=None, prompt_tokens_details=None))
```

怎么看？哪里是我们要找的模型回答？

以 `id` 为例，实际上是 `response` 的一个属性，通过 `response.id` 即可查看取值。

有个笨方法是，找个文本编辑器，新建文件，把这一大段东西丢进去，然后遇到形如 `ChatCompletion(` 里面带一堆参数的括号，以及分隔不同参数的逗号就换行，用缩进区分不同层次：

```
ChatCompletion(
    id='chatcmpl-523', 
    choices=[
        Choice(
            finish_reason='stop',
            index=0,
            logprobs=None,
            message=ChatCompletionMessage(
                content='Hello! How can I help you out today? 😊',
                refusal=None,
                role='assistant',
                annotations=None,
                audio=None,
                function_call=None,
                tool_calls=None,
                reasoning='The user is greeting me with "Hello World!" which is a classic "Hello World" programming greeting. I need to respond to this greeting in a friendly and helpful way.\n\nSince I\'m an AI language model without the ability to generate content, I should respond warmly and offer assistance. I\'ll acknowledge the greeting and express how I can help with whatever they need.'
            )
        )
    ],
    created=1790085557,
    model='qwen3.5:0.8b',
    object='chat.completion',
    service_tier=None,
    system_fingerprint='fp_ollama',
    usage=CompletionUsage(
        completion_tokens=90,
        prompt_tokens=24,
        total_tokens=114,
        completion_tokens_details=None,
        prompt_tokens_details=None
    )
)
```

现在看起来就清晰多了，我们关心的模型输出在 `message`，即 `response.choices[0].message`（注意 `choices` 实际上是个列表）里面，`content` 就是最终输出，`reasoning` 就是模型思考内容（是的，qwen3.5:0.8b 是个思考模型），`role` 和 `tool_calls` 在多轮上下文拼接的时候会用到，前者对应消息类型，后者表明模型是否发起了工具调用请求。

顺便查看一下其他东西：

- `response.choices[0].finish_reason`：大意是终止原因，`'stop'` 说明模型答完了。有时候会显示 `'length'`，同时模型输出被截断（见补充内容），这个时候请检查上下文容量。
- `response.model`：模型名称，与我们实际用的模型一致。
- `response.usage`：可以查看用了多少 token，其中：
  - `completion_tokens`：输出 tokens 数量，包括模型思考和最终回答。
  - `prompt_tokens`：输入，或者说提示词，对应的 tokens 数量。
  - `total_tokens`：两者加起来。

注意：由于模型本身的特点，每次得到的结果都会不尽相同。

## 获取最终输出

加一行即可：

```Python
content=response.choices[0].message.content
```

如果想打印出来而不是手动查看：

```Python
print("模型回答："，content)
```

现在执行代码就会直接看到模型的最终回答了。

## 如果输出 tokens 不够用（补充）

为了演示模型输出 tokens 不够用会发生什么，取 `max_tokens=30`，只有刚才的例子输出 tokens 数量（90）的零头：

```Python
response2 = client.chat.completions.create(
    model="qwen3.5:0.8b",
    messages=messages,
    max_tokens=30,  # 通常不够模型思考的
)
```
response2 = client.chat.completions.create(model="qwen3.5:0.8b",messages=messages,max_tokens=30)
然后读取结果：

```Python
print("模型思考内容：", response2.choices[0].message.reasoning)
print("模型回答：", response2.choices[0].message.content)
```

结果是这样的，模型思考内容被截断，回答则是空的：

```
模型思考内容： The user's message is "Hello World!" which is the greeting and the initial output of the "Hello World" in the Python interpreter. It could
模型回答：
```

如果进一步查看终止原因以及 tokens 用量：

```Python
print("终止原因：", response2.choices[0].finish_reason)
print("输入tokens：", response2.usage.prompt_tokens)
print("输出tokens：", response2.usage.completion_tokens)
print("总tokens：", response2.usage.total_tokens)
```

会得到：

```
终止原因： length
输入tokens： 24
输出tokens： 30
总tokens： 54
```

终止的原因是 `'length'`，实际输出 tokens 数量等于设置的最大输出 tokens 数量（30），也能验证模型输出被截断。

## 备注

不会的问 AI；至于代码，参考了 AI 给出的示例。
