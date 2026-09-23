---
layout: post
title: "让模型流式输出（WIP）"
Author: "qp91mn64"
Created: "2026-09-23"
---

# 让模型流式输出（WIP）

流式输出的核心特点是，能实时看到模型一个字一个字往外吐的过程，不用等半天才能看到模型输出。

这里还是以本地模型 qwen3.5:0.8b 为例，说明如何以流式输出的形式呈现模型内容。

如果不清楚如何调用本地模型，请移步 [如何用 Python 和 openai 库调用一个本地模型，得到模型回答](https://qp91mn64.github.io/2026/09/22/how_to_invoke_a_local_model.html)

## 打开流式输出

设置 `stream=True` 即可打开流式输出：

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

与不使用流式输出的结果比较像，最明显的区别是，少了很多东西，`stream.choices[0]` 里面没有 `message`，反而有个 `delta`，而且里面的 `reasoning` 只有很少的内容。此外 `object` 是 `'chat.completion.chunk'` 而不是 `'chat.completion'`，也没有 tokens 用量信息可以查看。

此外注意一点：`stream` 里面每个 `ChatCompletionChunk` 只能访问一次，如果你不小心遍历了整个 `stream`，再次遍历就什么也找不到了。

不妨把所有的 `delta` 里面的东西拼在一起试一试，看一看能不能得到完整的输出：

```Python
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

得到的结果大概是这样：

```
模型终止输出，原因： stop
stop
完整思考内容： , the user says "Hello World!". I just made that up.

Hmm, I need to understand what they're asking me. Wait, the user says "Hello World!". Oh, maybe they want me to output something in response to that statement. Wait, but I already generated "Hello World!" as an AI. Wait, that doesn't make sense. Let me think.

Wait, the user just said "Hello World!" as a greeting. Maybe they want a response that includes that word. Or maybe they want me to greet them in English. But wait, as an AI model, my output should probably follow my instructions when given prompts. But if I only follow the user's request, I need to wait for more instructions. Wait, but the user's message is a greeting. But maybe they want me to say Hello World! in my response. Or maybe they want me to confirm I'm an AI. Wait, but I need to make sure I'm not making up the word "Hello". Wait, the user's message is just "Hello World!". But I need to check the instruction. If the user wants me to respond, maybe I should just output "Hello World!" as my response. Or perhaps they want me to check if I'm correct. Wait, but I'm an AI. Wait, but the user said "Hello World!". Maybe they want me to say "Hello World!" in the same way I am an AI. But I need to make sure I'm not hallucinating. Wait, but the user is testing me. So maybe I should just output "Hello World!" as my response. But wait, I'm an AI. Wait, but I need to check whether I should reply with this greeting or not. Wait, the user's input is "Hello World!". But according to the system instructions, I should not make up facts. Wait, but if the user says "Hello World!", maybe they want me to output that. So I should output "Hello World!" as my response. Or perhaps they want me to acknowledge that I am an AI. But since the user is just saying "Hello World!", maybe they want me to say "Hello World!" in my own response. So I should output "Hello World!" as my response. Alternatively, maybe they want me to respond with a message like "Hello!" since I'm an AI. But if the user is a bot, maybe I should respond with a generic message. But wait, the user's message is "Hello World!". If I respond with "Hello World!", that might be okay. But if they ask a question, I should answer properly. But since the user is just saying "Hello World!", maybe I should just say "Hello World!" in my response. So the answer would be "Hello World!" as my response. Alternatively, maybe I should just output "Hello World!" in my response. So the final answer.
完整回答： Hello World!

That's it! How can I assist you today? 😊
```

其中“完整思考内容”与“完整回答”看起来都比较自然，除了前者缺少开头，而这也说明同一个 `stream` 里面同一个 `ChatCompletionChunk` 不能被重复遍历两次；`模型终止输出，原因： stop` 一行以及接下来只有一个单独成行的 `stop` 则表明，出现 `stop` 就不会再出现新的内容。

## 备注

这篇博客没写完，还差如何实时显示模型输出的部分。

不会的问 AI；至于代码，参考了 AI 给出的示例。
