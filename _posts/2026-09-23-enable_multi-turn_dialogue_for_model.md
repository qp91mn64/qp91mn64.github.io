---
layout: post
title: "与模型多轮对话，Python + openai库实现（WIP）"
Author: "qp91mn64"
Created: "2026-09-23"
---

# 与模型多轮对话，Python + openai库实现（WIP）

使用工具：Python，openai，Ollama

模型：qwen3.5:0.8b

## 回顾

还记得怎么调用模型并得到回答吗？

```Python
from openai import OpenAI

client = OpenAI(
    base_url = "http://localhost:11434/v1",
    api_key = "ollama"
)
messages = []
# 系统提示词
system_prompt = {
    "role": "system",
    "content": "You are a helpful assistant."
}
messages.append(system_prompt)
# 用户提示词
user_prompt =     {
    "role": "user",
    "content": "Hello World!"
}
messages.append(user_prompt)
# 传给模型，接收输出
response = client.chat.completions.create(
    model="qwen3.5:0.8b",  # 实际用的什么模型就写什么模型名称
    messages=messages,  # 消息列表
)
# 读取思考内容（如果你嫌模型思考太多，可忽略）
reasoning = response.choices[0].message.reasoning
print("模型思考内容：", reasoning)
# 读取模型回答
content = response.choices[0].message.content
print("模型回答：", content)
```

为了清晰，这里稍微调整了一下。

在调用模型的时候，给模型传递了两种消息：一种叫“系统提示词”，即 `"role": "system"` 对应的消息，通常预先定义好；另一种叫“用户提示词”， 即 `"role": "user"` 对应的消息，可以预先定义好，也可以来自外部（用户）输入。接下来看一下如何接收外部输入，也是为多轮对话做准备。

## 接收外部输入

用 Python 自带的 `input()` 就够了，相当于换一个 `user_prompt` ：

```Python
user_prompt = {
    "role": "user",
    "content": input("请输入：")  # 接收外部输入
}
```

然后传给模型即可，完整示例如下：

```Python
from openai import OpenAI

client = OpenAI(
    base_url = "http://localhost:11434/v1",
    api_key = "ollama"
)
messages = []
# 系统提示词
system_prompt = {
    "role": "system",
    "content": "You are a helpful assistant."
}
messages.append(system_prompt)
# 用户提示词
user_prompt = {
    "role": "user",
    "content": input("请输入：")  # 接收外部输入
}
messages.append(user_prompt)
# 传给模型
response = client.chat.completions.create(
    model="qwen3.5:0.8b",  # 实际用的什么模型就写什么模型名称
    messages=messages,  # 消息列表
)
# 读取思考内容（如果你嫌模型思考太多，可忽略）
reasoning = response.choices[0].message.reasoning
print("模型思考内容：", reasoning)
# 读取模型回答
content = response.choices[0].message.content
print("模型回答：", content)
```

实际使用时，在 `“请输入：”` 后面放你要说的内容，按下回车键，等一等，就可以看到模型的回答了。不妨多试几次，每次向模型提出不同的问题，或者换不同的模型试一试。

## 第二轮对话

与人交流的时候，往往不会只说一句话。现在我们想给模型发两条消息，或者说在看到模型输出之后接着问，怎么办呢？

直接发，难道像这样？

```Python
from openai import OpenAI
model = "qwen3.5:0.8b"  # 模型名称
client = OpenAI(base_url = "http://localhost:11434/v1", api_key = "ollama")
# 第一次输入
user_prompt = {"role": "user", "content": input("请输入：")}
response = client.chat.completions.create(model=model, messages=[user_prompt])  # 简便起见不读取思考内容
print("模型回答：", response.choices[0].message.content)
# 第二次输入
user_prompt2 = {"role": "user", "content": input("请输入：")}
response2 = client.chat.completions.create(model=model, messages=[user_prompt2])
print("模型第二次回答：", response2.choices[0].message.content)
```

这样有一个问题，**模型“不记得”之前的信息**，就像新建对话一样。
