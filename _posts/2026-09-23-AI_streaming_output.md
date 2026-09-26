---
layout: post
title: "让模型流式输出，Python + openai 库实现"
Author: "qp91mn64"
Created: "2026-09-23"
Last Modified: "2026-09-26"
---

# 让模型流式输出，Python + openai 库实现

首次发布：2026-09-23  
最近更新：2026-09-26

流式输出的核心特点是，能实时看到模型一个字一个字往外吐的过程，不用像非流式输出（直接请求模型）那样等半天了。

为避免歧义，这里的“模型输出”包括模型思考和最终回答的部分。

## 目录

- [打开流式输出](#打开流式输出)
- [获取模型回答](#获取模型回答)
- [实时显示输出](#实时显示输出)
- [查看token用量](#查看token用量)
- [小结](#小结)
- [备注](#备注)

## 打开流式输出

设置 `stream=True` 即可：

```Python
from openai import OpenAI
client = OpenAI(base_url = "http://localhost:11434/v1", api_key = "ollama")
messages = [{"role": "system", "content": "You are a helpful assistant."},
            {"role": "user", "content": "Hello World!"}]
stream = client.chat.completions.create(
    model="qwen3.5:0.8b",
    messages=messages,
    stream=True
)
```

为了区分，变量名用 stream。

## 获取模型回答

如何从中获取模型的回答呢？难道还是像刚才那样：

```Python
content = stream.choices[0].message.content
```

不好意思，报错了：

```
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
AttributeError: 'Stream' object has no attribute 'choices'
```

怎么办呢？要不通过 `__doc__` 看一看有没有说明：

```Python
print(stream.__doc__)
```

得到：

> Provides the core interface to iterate over a synchronous stream response.

`iterate over`？难道这个东西可迭代？

```Python
for i, x in enumerate(stream):
    print(x)
    break
```

会打印出类似于这样的东西：

```
ChatCompletionChunk(id='chatcmpl-207', choices=[Choice(delta=ChoiceDelta(content='', function_call=None, refusal=None, role='assistant', tool_calls=None, reasoning='Okay'), finish_reason=None, index=0, logprobs=None)], created=1790164049, model='qwen3.5:0.8b', object='chat.completion.chunk', service_tier=None, system_fingerprint='fp_ollama', usage=None)
```

还是使用笨方法展开：

```
ChatCompletionChunk(
    id='chatcmpl-207',
    choices=[
        Choice(
            delta=ChoiceDelta(
                content='',
                function_call=None,
                refusal=None,
                role='assistant',
                tool_calls=None,
                reasoning='Okay'
            ),
            finish_reason=None,
            index=0,
            logprobs=None
        )
    ],
    created=1790164049,
    model='qwen3.5:0.8b',
    object='chat.completion.chunk',
    service_tier=None,
    system_fingerprint='fp_ollama',
    usage=None
)
```

与不使用流式输出（也叫作“非流式输出”）的结果有些不同，最明显的区别是，少了很多东西，`stream.choices[0]` 里面没有 `message`，反而有个 `delta`，而且里面的 `reasoning` 只有很少的内容。此外 `object` 是 `'chat.completion.chunk'` 而不是 `'chat.completion'`，也没有 tokens 用量信息可以查看（有专门的方式能查看，见[查看token用量](#查看token用量)一节）。

此外注意一点：`stream` 里面每个 `ChatCompletionChunk` （为简便起见，下文称之为 chunk）只能访问一次，如果你不小心遍历了整个 `stream`，再次遍历就什么也找不到了。

不妨把所有的 `delta` 里面的东西拼在一起试一试，看一看能不能得到完整的输出：

```Python
... # 见上文
full_reasoning = ""
full_content = ""
stop = False
for x in stream:
    d = x.choices[0].delta
    # 坑：模型思考完，delta 就没有 reasoning 属性，还访问就报错
    if hasattr(d, "reasoning"):
        if d.reasoning:
            full_reasoning += d.reasoning
    if d.content:
        full_content += d.content
    if x.choices[0].finish_reason:
        print("模型终止输出，原因：", x.choices[0].finish_reason)
        stop = True
    if stop:
        print("stop")
print("完整思考内容：", full_reasoning)
print("完整回答：", full_content)
```

其中为了拿到模型思考内容，以及确认 `finish_reason`，加了些东西。

得到的结果大概是这样（不好意思内容有点多）：

```
模型终止输出，原因： stop
stop
完整思考内容： , the user says "Hello World!". I just made that up.

Hmm, I need to understand what they're asking me. Wait, the user says "Hello World!". Oh, maybe they want me to output something in response to that statement. Wait, but I already generated "Hello World!" as an AI. Wait, that doesn't make sense. Let me think.

Wait, the user just said "Hello World!" as a greeting. Maybe they want a response that includes that word. Or maybe they want me to greet them in English. But wait, as an AI model, my output should probably follow my instructions when given prompts. But if I only follow the user's request, I need to wait for more instructions. Wait, but the user's message is a greeting. But maybe they want me to say Hello World! in my response. Or maybe they want me to confirm I'm an AI. Wait, but I need to make sure I'm not making up the word "Hello". Wait, the user's message is just "Hello World!". But I need to check the instruction. If the user wants me to respond, maybe I should just output "Hello World!" as my response. Or perhaps they want me to check if I'm correct. Wait, but I'm an AI. Wait, but the user said "Hello World!". Maybe they want me to say "Hello World!" in the same way I am an AI. But I need to make sure I'm not hallucinating. Wait, but the user is testing me. So maybe I should just output "Hello World!" as my response. But wait, I'm an AI. Wait, but I need to check whether I should reply with this greeting or not. Wait, the user's input is "Hello World!". But according to the system instructions, I should not make up facts. Wait, but if the user says "Hello World!", maybe they want me to output that. So I should output "Hello World!" as my response. Or perhaps they want me to acknowledge that I am an AI. But since the user is just saying "Hello World!", maybe they want me to say "Hello World!" in my own response. So I should output "Hello World!" as my response. Alternatively, maybe they want me to respond with a message like "Hello!" since I'm an AI. But if the user is a bot, maybe I should respond with a generic message. But wait, the user's message is "Hello World!". If I respond with "Hello World!", that might be okay. But if they ask a question, I should answer properly. But since the user is just saying "Hello World!", maybe I should just say "Hello World!" in my response. So the answer would be "Hello World!" as my response. Alternatively, maybe I should just output "Hello World!" in my response. So the final answer.
完整回答： Hello World!

That's it! How can I assist you today? 😊
```

其中“完整思考内容”与“完整回答”看起来都比较自然，除了前者缺少开头，而这也说明同一个 `stream` 里面同一个 chunk 不能被重复遍历两次；`模型终止输出，原因： stop` 一行以及接下来只有一个单独成行的 `stop` 则表明，只要出现 `stop`，就不会再出现新的内容。

## 实时显示输出

确认能拿到完整输出之后，现在的目标是，一个字一个字地显示。难道一个字一个字地打印即可？就像这样：

```Python
from openai import OpenAI
client = OpenAI(base_url="http://localhost:11434/v1", api_key="ollama")
messages = [{"role": "system", "content": "You are a helpful assistant."},
            {"role": "user", "content": "Hello World!"}]
stream = client.chat.completions.create(
    model="qwen3.5:0.8b",
    messages=messages,
    stream=True
)
print("模型思考内容：", end="")
answer = False
# 加上思考内容，让效果更明显
# end=""防换行
for c in stream:
    d = c.choices[0].delta
    if hasattr(d, "reasoning"):
        print(d.reasoning, end="")
    else:
        if not answer:
            print()
            print("模型回答：", end="")
            answer = True
    if d.content:
        print(d.content, end="")
```

问题是，如果你把上面的代码复制粘贴为 Python 脚本，再运行，结果是，等了一会儿没任何输出，然后完整的回答突然就出现了，而不是我们想看的一个字一个字地显示的效果。怎么办呢？

接着查资料（问 AI），得到的结果是，这个是输出缓冲问题，解决方案是禁用缓冲：

- 方案一：如果你用命令行运行 Python 脚本，加个 `-u` 参数：`python -u <your_file_name>.py`。

- 方案二：在 Python 代码的 `print` 里面设置 `flush=True`。

这两种方案都可以，不过一般用方案二，**设置 `flush=True`**，这样运行脚本会方便一点：

```Python
from openai import OpenAI
client = OpenAI(base_url="http://localhost:11434/v1", api_key="ollama")
messages = [{"role": "system", "content": "You are a helpful assistant."},
            {"role": "user", "content": "Hello World!"}]
stream = client.chat.completions.create(
    model="qwen3.5:0.8b",
    messages=messages,
    stream=True
)
print("模型思考内容：", end="", flush=True)
answer = False
# 加上思考内容，让效果更明显
# end=""防换行
for c in stream:
    d = c.choices[0].delta
    if hasattr(d, "reasoning"):
        print(d.reasoning, end="", flush=True)
    else:
        if not answer:
            print()
            print("模型回答：", end="", flush=True)
            answer = True
    if d.content:
        print(d.content, end="", flush=True)
```

你可以自行比较不同方案之间的差异，看一看是否得到了想要的一个字一个字显示的效果。如果现象不够明显，可以尝试更大的模型，例如 `qwen3.5:4b` （注意内存是否足够）。

## 查看token用量

很多时候我们想知道消耗了多少tokens，怎么查看呢？

直接找？

```Python
... # 开启流式输出，见上文
for c in stream:
    if c.usage:
        print(c.usage)
```

不好意思，什么也打印不出来——哪怕最后一个 chunk，模型都终止回答了。

怎么办呢？只好接着查资料（问 AI），得到的结果是，在打开流式输出的时候，必须显式声明包含 usage：

```Python
stream_options={"include_usage": True}
```

完整的请求则是这样的：

```Python
stream = client.chat.completions.create(
    model="qwen3.5:0.8b",
    messages=messages,
    stream=True,
    stream_options={"include_usage": True}
)
```

那么 usage 在哪里找？不妨挨个计数：

```Python
... # 开启流式输出，见上文
count = 0  # 计数
print("模型思考内容：", end="", flush=True)
for c in stream:
    if c.usage:
        print()
        print(c)
        print("用量明细", c.usage)
    elif c.choices:
        if (c.choices[0].finish_reason):
            print()
            print('终止回答原因：',c.choices[0].finish_reason)
            print('c.usage',c.usage)
            print(c)
        d = c.choices[0].delta
        if hasattr(d, "reasoning"):
            print(d.reasoning, end="", flush=True)
        else:
            if not answer:
                print()
                print("模型回答：", end="", flush=True)
                print()
                answer = True
        if d.content:
            print(d.content, end="", flush=True)
    print(" ", count)
    count += 1
```

注意这里面有个坑，`usage` 不是 `None` 的时候，`choices` 是空的，还读取模型回答就会报错，解决方法是先读取 `usage` 或者先检查 `choices` 里面有没有东西。

结果大概像这样：

```
?  239
 😊  240

终止回答原因： stop
c.usage None
ChatCompletionChunk(id='chatcmpl-3', choices=[Choice(delta=ChoiceDelta(content='', function_call=None, refusal=None, role='assistant', tool_calls=None), finish_reason='stop', index=0, logprobs=None)], created=1790404714, model='qwen3.5:0.8b', object='chat.completion.chunk', service_tier=None, system_fingerprint='fp_ollama', usage=None)
  241

ChatCompletionChunk(id='chatcmpl-3', choices=[], created=1790404714, model='qwen3.5:0.8b', object='chat.completion.chunk', service_tier=None, system_fingerprint='fp_ollama', usage=CompletionUsage(completion_tokens=250, prompt_tokens=24, total_tokens=274, completion_tokens_details=None, prompt_tokens_details=None))
用量明细 CompletionUsage(completion_tokens=250, prompt_tokens=24, total_tokens=274, completion_tokens_details=None, prompt_tokens_details=None)
  242
```

这里请自行展开。可以看到，在这里，**提供用量明细的 chunk 出现在最后面，而且不包含模型输出内容**，其中的 `usage` 就像非流式输出返回的那样，提供了输入输出 tokens 用量。

所以查看 token 用量的方式是：

1. 在请求模型的时候，设置 `stream_options={"include_usage": True}`；
2. 检查 `usage` 是不是 `None`，如果不是，则读取token用量；

此外，保险起见，在读取模型输出之前，先检查 `choices` 是不是空列表。

不知道你注意到没有，`finish_reason='stop'` 对应的 chunk，`content` 是空字符串，已经没内容了。由此推测，接收到的 chunk 的顺序是：模型思考内容 -> 模型最终回答内容 -> 通过 `finish_reason` 提示模型终止输出，而且不再包含模型输出 -> 如果有设置，携带 `usage`

最终的示例代码：

```Python
from openai import OpenAI
client = OpenAI(base_url="http://localhost:11434/v1", api_key="ollama")
messages = [{"role": "system", "content": "You are a helpful assistant."},
            {"role": "user", "content": "Hello World!"}]
stream = client.chat.completions.create(
    model="qwen3.5:0.8b",
    messages=messages,
    stream=True,
    stream_options={"include_usage": True}
)
answer = False
print("模型思考内容：", end="", flush=True)
for c in stream:
    if c.usage:
        usage = c.usage
    elif c.choices:
        d = c.choices[0].delta
        if hasattr(d, "reasoning"):
            print(d.reasoning, end="", flush=True)
        else:
            if not answer:
                print()
                print("模型回答：", end="", flush=True)
                print()
                answer = True
        if d.content:
            print(d.content, end="", flush=True)
print()
print("输入tokens：", usage.prompt_tokens)
print("输出tokens：", usage.completion_tokens)
print("总tokens：", usage.total_tokens)
```

## 小结

在请求模型的时候，设置 `stream=True` 即可打开流式输出；

用循环遍历得到的结果，然后从其 `.choices[0].delta` 读取模型思考内容 `reasoning` 和回答 `content`；

在 `print` 里面使用 `flush=True` 来实现逐字显示。

要读取 token 用量，先设置 `stream_options={"include_usage": True}`，然后在 `usage` 里面读取，注意提供用量信息的 chunk 里面不含模型输出。

## 备注

“一个字一个字地显示”是出于行文方便，而不是说，在流式输出模式，每次读取的模型输出都是一个字词。

一个疑问：[查看token用量](#查看token用量)一节里面，挨个计数找出 usage 在哪个 chunk 的时候，实测发现，输出 tokens 比 chunk 总数略多，而且模型输出中间存在空行，是为什么？

不会的问 AI；至于代码，参考了 AI 给出的示例。

含有大段的 AI 生成内容：源自 qwen3.5:0.8b。
