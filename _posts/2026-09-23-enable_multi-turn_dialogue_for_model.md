---
layout: post
title: "与模型多轮对话，Python + openai库实现"
Author: "qp91mn64"
Created: "2026-09-23"
Last_modified: "2026-09-26"
---

# 与模型多轮对话，Python + openai库实现

首次发布：2026-09-23
最近更新：2026-09-26

使用工具：Python，openai，Ollama

模型：qwen3.5:0.8b

## 回顾

还记得[怎么调用模型并得到回答](https://qp91mn64.github.io/2026/09/22/how_to_invoke_a_local_model.html)吗？

```Python
from openai import OpenAI

client = OpenAI(
    base_url = "http://localhost:11434/v1",
    api_key = "ollama"
)
# 系统提示词
system_prompt = {
    "role": "system",
    "content": "You are a helpful assistant."
}
# 用户提示词
user_prompt = {
    "role": "user",
    "content": "Hello World!"
}
messages = [system_prompt, user_prompt]
# 传给模型，接收输出
response = client.chat.completions.create(
    model="qwen3.5:0.8b",  # 实际用的什么模型就写什么模型名称
    messages=messages,  # 消息列表
)
# 读取思考内容。如果你嫌模型思考太多，可关闭思考模式，此时去除这个部分即可。
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

然后像之前那样，传给模型即可。

实际使用时，在 `“请输入：”` 后面放你要说的内容，按下回车键，等一等，就可以看到模型的回答了。不妨多试几次，每次向模型提出不同的问题，或者换不同的模型试一试。

## 第二轮对话

与人交流的时候，往往不会只说一句话。现在我们想给模型发两条消息，或者说在看到模型输出之后接着问，怎么办呢？

说明：为了对比，这一节省略系统提示词；为了更快出结果，关闭思考模式。

### 一般地，大模型 API 是无状态的

直接把第二条消息传给模型，不带上之前的消息？这样有一个问题，**模型“不记得”之前的信息**，就像新建对话一样。

怎么体现呢？假设我们第一轮输入是：

> 已知 a=3，请计算 a+2 的值。直接给出答案即可。

```
user_prompt1 = {"role": "user", "content": "已知 a=3，请计算 a+2 的值。直接给出答案即可。"}
```

第二轮输入是：

> 再检查一遍。

```
user_prompt2 = {"role": "user", "content": "再检查一遍。"}
```

完整的示例代码则像这样：

```Python
from openai import OpenAI
model = "qwen3.5:0.8b"
client = OpenAI(base_url = "http://localhost:11434/v1", api_key = "ollama")
# 这里省略系统提示词
user_prompt1 = {"role": "user", "content": "已知 a=3，请计算 a+2 的值。直接给出答案即可。"}
user_prompt2 = {"role": "user", "content": "再检查一遍。"}
# 第一轮对话
messages1 = [user_prompt1]
response1 = client.chat.completions.create(
    model=model,
    messages=messages1,  # 第一轮输入
    reasoning_effort="none"  # 关闭思考模式，防止模型半天不回答，下同
)
print("模型第一轮回答：")
print(response1.choices[0].message.content)
# 第二轮对话
messages2 = [user_prompt2]
response2 = client.chat.completions.create(
    model=model,
    messages=messages2,  # 第二轮输入
    reasoning_effort="none"
)
print()
print("模型第二轮回答：")
print(response2.choices[0].message.content)
```

模型第一轮回答像这样，答出了 `5`（实测偶尔也会回答错误）：

```
3 + 2 = 5
```

模型第二轮回答就像这样，**不知道要检查第一轮的计算结果**：

```
收到，正在仔细核对。请告诉我您具体需要检查什么内容、哪里需要调整或想补充哪部分信息。
```

这就是为什么人们常常说大模型 API 是“无状态的”，也就是说，大模型不会储存接收到的信息，回复完成之后，之前的输入和模型“说”了什么，就全部丢了，不会自动保存到下一次调用。

这样的好处是，每次调用都是全新的对话，不会受之前调用的影响；缺点就是，不“自动”支持多轮对话。

怎么办呢？模型只知道传给它的信息，不如在传新消息的时候，把模型的回复传回去？

### 消息回传

怎么传呢？偷个懒，不手动拼接需要的信息，直接回传整个 `response1.choices[0].message`，就像 DeepSeek API 官方文档里面写的那样？毕竟 `response1.choices[0].message` 里面什么都有，看起来不用担心漏掉模型思考内容或者工具调用请求的问题。

难道回传第二次输入就够了吗？

把上述代码里面的 `messages2` 换成 `[response1.choices[0].message, user_prompt2]`，其余不变，再次运行，结果可能像这样：

```
模型第一轮回答：
3 + 2 = 5

**5**

模型第二轮回答：
好的，再次确认：

**3 + 2 = 5**

答案正确，无需再次检查。
```

没有报错，看来这种“偷懒”的方式真的可以用，只是看起来这模型“忘记”了原始题目还有个 `a`。

有时这模型也会乱答，尤其是第一次只回答一个 `$5$` 的时候（离谱的 AI 幻觉？）：

```
模型第一轮回答：
$5$

模型第二轮回答：
收到。我已经将刚才的回答再次核对一遍，主要确认了以下几点：

1.  **准确性检查**：关于"$5$美元加$2500000000 美元等于多少钱”这个问题的计算逻辑，$5 + 2.5$亿是正确的计算过程。
2.  **格式与风格**：回答保持了简洁明了的风格，使用了"$5$"来区分个位数字和十位数字，并在末尾加上了"$$5$$"的格式标记。

现在再次确认一下当前的数字：
- **$5**美元
- **$2500000000**美元（即25亿美元）

**计算结果：**
$$ 5 + 2500000000 = 2500000005 $$

**最终结论：**
$$ 25,000,000,05 \text{ (美元)} $$
```

现在我们把第一次输入（原始题目）也放上，按照对话顺序排列：

```Python
messages2 = [user_prompt1, response1.choices[0].message, user_prompt2]  # 只改变第二次传入的消息列表，其余不变
```

结果可能是这样的，看起来模型从第一次输入里面获得了原始题目：

```
模型第一轮回答：
3 + 2 = **5**

模型第二轮回答：
好的，检查一遍确认无误：
已知 $a = 3$，
计算 $a + 2$：
$$3 + 2 = 5$$

答案是：
**5**
```

也可能是这种：

```
模型第一轮回答：
2 + 2 = 4

模型第二轮回答：
计算 \( a + 2 \) 时：
- 已知 \( a = 3 \)
- 进行加法运算：\( 3 + 2 = 5 \)

**答案：** 5
```

个人猜测，对于最后一种情况，如果没有回传第一次输入，模型很可能第二次也回答 `4`。

## 多轮对话

类似地，每轮对话带上之前所有对话记录即可：

```Python
from openai import OpenAI
model = "qwen3.5:0.8b"
client = OpenAI(base_url = "http://localhost:11434/v1", api_key = "ollama")
messages = []
max_rounds = 10;  # 最多对话次数
for x in range(1, max_rounds + 1):
    print("--------------------------------第{}轮对话--------------------------------".format(x))
    user_prompt = {"role": "user", "content": input("输入：".format(x))}
    messages.append(user_prompt)
    response = client.chat.completions.create(
        model=model,
        messages=messages,
        reasoning_effort="none"
    )
    assistant_message = response.choices[0].message
    print("模型回复：".format(x))
    print(assistant_message.content)
    messages.append(assistant_message)
```

如果你想用流式输出（参考个人博客：[让模型流式输出](https://qp91mn64.github.io/2026/09/23/AI_streaming_output.html)）：

```Python
from openai import OpenAI
model = "qwen3.5:0.8b"
client = OpenAI(base_url = "http://localhost:11434/v1", api_key = "ollama")
messages = []
max_rounds = 10;  # 最多对话次数
for x in range(1, max_rounds + 1):
    print("--------------------------------第{}轮对话--------------------------------".format(x))
    user_prompt = {"role": "user", "content": input("输入：".format(x))}
    messages.append(user_prompt)
    stream = client.chat.completions.create(
        model="qwen3.5:0.8b",
        messages=messages,
        stream=True
    )
    assistant_reasoning=""
    assistant_content=""
    print("模型思考内容：", end="", flush=True)
    answer = False
    for c in stream:
        d = c.choices[0].delta
        if hasattr(d, "reasoning"):
            print(d.reasoning, end="", flush=True)
            assistant_reasoning += d.reasoning
        else:
            if not answer:
                print()
                print("模型回答：", end="", flush=True)
                answer = True
        if d.content:
            print(d.content, end="", flush=True)
            assistant_content += d.content
    assistant_message = {
        "role": d.role,
        "reasoning": assistant_reasoning,
        "content": assistant_content
    }
    messages.append(assistant_message)
    if (x < max_rounds):
        print()
```

测试结果是：代码能用；打开思考模式，模型回答质量高一些。

## 备注

一个疑问：模型思考内容要回传吗？

不会的问 AI；至于代码，参考了 AI 给出的示例。

含有大段的 AI 生成内容：源自 qwen3.5:0.8b。
