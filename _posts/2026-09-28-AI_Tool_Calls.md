---
layout: post
title: "如何让模型调用工具，Python + openai 库实现"
Author: "qp91mn64"
Created: "2026-09-28"
Last_Modified: "2026-09-29"
---

# 如何让模型调用工具，Python + openai 库实现

首次发布：2026-09-28  
最近更新：2026-09-29

调用模型：Python 3.12.0，openai 1.109.1，Ollama 0.19.0

模型选择：qwen3.5:0.8b，效果不好就换 qwen3.5:2b 或 qwen3.5:4b （这三个模型都基于 Apache 2.0 许可证）

AI 借助工具可以扩展能力范围。要实现工具调用，首先要让 AI 知道有什么工具，然后当模型发起工具调用请求之后，代码就执行相应工具，得到结果，再把结果回传给模型，模型看到结果再进行下一步行动。

限于篇幅，这里只讨论单次工具调用，非流式输出。

## 目录

- [定义一个工具](#定义一个工具)
  - [从一个字典开始](#从一个字典开始)
  - [第二层](#第二层)
  - [第三层](#第三层)
  - [继续剥洋葱](#继续剥洋葱)
  - [最里层](#最里层)
  - [工具定义示例](#工具定义示例)
  - [执行工具的代码](#执行工具的代码)
- [把工具定义发给模型](#把工具定义发给模型)
- [执行工具](#执行工具)
  - [解析工具请求](#解析工具请求)
  - [运行工具代码](#运行工具代码)
- [回传结果](#回传结果)
  - [回传模型的请求](#回传模型的请求)
  - [回传工具调用结果](#回传工具调用结果)
- [改进方向](#改进方向)
- [不回传请求会怎么样](#不回传请求会怎么样)
- [小结](#小结)
- [备注](#备注)

## 定义一个工具

那么一个工具长什么样呢？又怎么“写”一个工具呢？一般地，传给模型的工具定义是 JSON 格式（一种轻量级的数据交换格式），或者说嵌套字典，放进 `tools` 列表即可；而每个工具对应的执行函数则需要单独编写，不同工具的执行方式千差万别，这里不再赘述。

一个 JSON 格式的定义还是比较复杂的，里面嵌套了好几层，不妨像剥洋葱一样一层层剥开看一看每一层是什么样子的。在接下来的部分，由于是从 Python 的角度看待 JSON 格式的对象，统一使用 Python 里面的数据类型。

### 从一个字典开始

一般地，自定义工具的第一层是这样的，一个对象，在 Python 里面，一个字典：

```Python
tool = {
    "type": "function",  # 类型
    "function": {
        ... # 内嵌层次
    }
}
```

可以简单理解为，模型调用工具的形式通常是函数调用（Funcion Calling），所以类型是 "function"。

### 第二层

这个 "function" 里面有些什么呢？剥开第二层，又是一个字典：

```Python
... # 外部层次
"function": {
    "name": "工具名",
    "description": "简短描述",
    "parameters": {
        ... # 内嵌层次
    }
}
```

这个工具的名称，介绍，以及各种参数。

### 第三层

剥开第三层，还是字典：

```Python
... # 外部层次
"parameters": {
    "type": "object",  # 类型又出现了
    "properties": {
        ... # 内嵌层次，各种参数
    },
    "required": [
        ... # 放"properties"中的参数，表示必填项
    ]
}
```

没有直接出现各种参数，而是又绕了一层。其中 `"type"` 写个 `"object"` 即可，`"properties"` 放各参数定义，`"required"` 实际上就是个列表，里面放 `"properties"` 中必填的参数名称，没出现的则是可选项。

### 继续剥洋葱

```Python
... # 外部层次
"properties": {
    "param1": {  # 假设的参数名：第一个参数
        ... # 内嵌层次
    },
    "param2": {
        ... # 内嵌层次
    },
    ...,
    "param_n": {
        ... # 内嵌层次
    }
}
```

"properties" 里面是并排的各个参数，每个参数的结构是类似的；为了行文方便，参数名是假设的。

一个工具也可能没有参数，此时 `"properties"` 就是个空字典，剥洋葱到此为止，外部一层也没有 `"required"`。

### 最里层

一个参数长这样：

```Python
"param_k": {
    "type": "类型"  # "string", "int"等类型，由参数取值决定
    "description": "描述"  # 对这个参数的描述
    "enum": ["值1", "值2", ..., "值m"]  # 可选：限制取值
}
```

### 工具定义示例

```Python
tool = {
    "type": "function",  # 类型
    "function": {
        "name": "工具名",
        "description": "简短描述",
        "parameters": {
            "type": "object",  # 类型又出现了
            "properties": {
                "param1": {  # 假设的参数名：第一个参数
                    "type": "类型"  # "string", "int"等类型，由参数取值决定
                    "description": "描述"  # 对这个参数的描述
                },
                "param2": {  # 与"param1"类似
                    "type": ...  # 类型
                    "description": ...  # 描述
                    "enum": ["值1", "值2", ..., "值m"]  # 可选：限制取值
                },
                ...,
                "param_n": {
                    ... # 与前面的参数类似
                }
            }
            "required": [
                "param_r1", "param_r2", ..., "param_rk" # 放"properties"中的参数，表示必填项
            ]
        }
    }
}
```

一个工具定义，看着有点吓人，说白了就是个**嵌套了好几层的字典**，其中的值，一般是字典、字符串或者列表类型，列表用于 `"required"` 和各参数的 `"enum"`，字典用于需要展开结构的情况，其余是字符串。至于各层字典，除了 `"function"`，其余都含 `"type"` 键。

单看模板有点抽象，不如看几个具体一点的例子。

#### 示例1：获取当前时间

假设工具名是 `get_current_time`，返回值就是当前时间，无需参数：

```Python
get_current_time_tool = {
    "type": "function",
    "function": {
        "name": "get_current_time",
        "description": "获取当前时间",
        "parameters": {
            "type": "object",
            "properties": {
            },
        },
    },
}
```

#### 示例2：查看库存

```Python
view_inventory_tool = {
    "type": "function",
    "function": {
        "name": "view_inventory",
        "description": "查看商店库存",
        "parameters": {
            "type": "object",
            "properties": {
                "item": {
                    "type": "string",
                    "description": "商品名称，例如瓶装水，饼干，一次只能查找一种商品"
                }
            },
            "required": ["item"]
        },
    },
}
```

#### 示例3：搜索工具

```Python
search_tool = {
    "type": "function",
    "function": {
        "name": "search",
        "description": "查资料，获取信息回答问题",
        "parameters": {
            "type": "object",
            "properties": {
                "query": {
                    "type": "string",
                    "description": "搜索关键词"
                },
            },
            "required": ["query"]
        },
    },
}
```

### 执行工具的代码

只有工具定义，能接收到工具调用请求，却不能实际执行工具，得到结果。

谁来执行？模型吗？不是的，模型本身不能执行。那么只能是程序代码了，比如说，查看商店库存的工具，执行代码可以是这样的（经过简化处理），写个函数即可：

```Python
def execute_view_inventory_tool(arguments):
    """
    查看商店库存工具执行函数
    为了简便，这里的数据是杜撰的，只列出部分商品，具体数据请以实际商店为准
    """
    inventory = {
        "饼干": 40,
        "方便面": 35,
        "瓶装水": 55,
        "洗衣粉": 30,
        "圆珠笔": 50,
        "牙膏": 35,
    }
    item = arguments["item"]
    if item in inventory.keys():
        if inventory[item] > 0: 
            return "找到库存{}，剩余数量{}。".format(item, inventory[item])
        else:
            return "未找到库存{}，请及时补货。".format(item)
    else:
        return "本商店不卖{}。".format(item)
```

后续如果模型请求了 `view_inventory` 工具，调用这个函数即可得到结果。

## 把工具定义发给模型

实际传给模型的是一个列表，所以把有关工具定义放进列表再发出去即可：

```Python
tools = [
    {
        ... # 工具定义
    },
    {
        ... # 工具定义
    },
    ...
]
response = client.chat.completions.create(
    model=model,
    messages=messages,
    tools=tools,
)
```

以上文的 `view_inventory` 工具为例，示例如下：

```Python
from openai import OpenAI
model = "qwen3.5:0.8b"
client = OpenAI(
    base_url="http://localhost:11434/v1",
    api_key="ollama",
)
view_inventory_tool = {
    "type": "function",
    "function": {
        "name": "view_inventory",
        "description": "查看商店库存",
        "parameters": {
            "type": "object",
            "properties": {
                "item": {
                    "type": "string",
                    "description": "商品名称，例如瓶装水，饼干，一次只能查找一种商品"
                }
            },
            "required": ["item"]
        },
    },
}
tools = [
    view_inventory_tool
]
messages = [
    {"role": "system", "content": "你是一家小商店的老板，负责这家商店的经营，目标是保持口碑，留住顾客，获得稳定的利润，让商店能长久地经营下去。目前在售商品有饼干、方便面、瓶装水、圆珠笔、牙膏、洗衣粉。"},
    {"role": "user", "content": "现在商店里面还有饼干吗？"}
]
response = client.chat.completions.create(
    model=model,
    messages=messages,
    tools=tools,
)
print(response)
```

查看返回结果，展开之后大概是这样的：

```
ChatCompletion(
    id='chatcmpl-53',
    choices=[
        Choice(
            finish_reason='tool_calls',
            index=0,
            logprobs=None,
            message=ChatCompletionMessage(
                content='',
                refusal=None,
                role='assistant',
                annotations=None,
                audio=None,
                function_call=None,
                tool_calls=[
                    ChatCompletionMessageFunctionToolCall(
                        id='call_y9wzw3f8',
                        function=Function(
                            arguments='{"item":"饼干"}',
                            name='view_inventory'
                        ),
                        type='function',
                        index=0
                    )
                ],
                reasoning='用户想知道商店内是否有饼干。我需要使用view_inventory工具来查看商店库存。这个工具需要"item"参数，但我不知道具体的商品名称。从问题来看，用户询问的是"饼干"，所以我应该使用"饼干"作为item参数。\n\n让我调用view_inventory工具，参数是"饼干"。'
            )
        )
    ],
    created=1790519298,
    model='qwen3.5:0.8b',
    object='chat.completion',
    service_tier=None,
    system_fingerprint='fp_ollama',
    usage=CompletionUsage(
        completion_tokens=93,
        prompt_tokens=343,
        total_tokens=436,
        completion_tokens_details=None,
        prompt_tokens_details=None
    )
)
```

注意到与直接调用模型，不带工具相比，这次的返回结果多出 `response.choices[0].message.tool_calls`，表明模型发起了工具调用请求:

```
tool_calls=[
    ChatCompletionMessageFunctionToolCall(
        id='call_y9wzw3f8',
        function=Function(
            arguments='{"item":"饼干"}',
            name='view_inventory'
        ),
        type='function',
        index=0
    )
]
```

`tool_calls` 是个列表，里面可能有一个或多个请求，一个请求则包含 `id`、`function`、`type`、`index` 属性，其中 `id` 能区分不同的工具调用请求，回传结果的时候用得到，`function` 里面包括 `name` 工具名称和 `arguments` 参数。

## 执行工具

接下来的部分就需要代码实现了，然后拿到结果。

### 解析工具请求

`arguments` 是个字符串，里面有执行工具所需的参数，怎么拿到参数呢？直接读取字符串？还是提取出其中的字典，直接按照参数名称访问每个键？

这里选择后者——使用 Python 自带的 `json` 库就能处理（详见相关说明文档）：

```Python
import json
argument_string = '{"item":"饼干"}'
argument_json = json.loads(argument_string)
print(argument_json["item"])
```

结果将会打出 `饼干`。

回到具体的请求，则是这样的，注意由于实际模型可能同时发起多个请求，用循环逐个处理：

```Python
... # 见上文
import json
tool_calls = response.choices[0].message.tool_calls
for tool_call in tool_calls:
    name = tool_call.function.name
    arguments = json.loads(tool_call.function.arguments)
    ... # WIP
```

### 运行工具代码

判断模型请求的是哪个工具，然后传入对应参数即可。

为了方便，用字典来管理各个执行函数（不要忘记了到上文找对应的执行函数）：

```Python
... # 见上文
executing_tools = {
    "view_inventory": execute_view_inventory_tool,
}
```

这样读取工具名，传入参数，就能运行相应的函数：

```Python
... # 见上文
import json
tool_calls = response.choices[0].message.tool_calls
for tool_call in tool_calls:
    name = tool_call.function.name
    arguments = json.loads(tool_call.function.arguments)
    result = executing_tools[name](arguments)
    print("工具id", tool_call.id, "执行结果：", result)  # 演示/调试用
    ... # WIP
```

得到的结果大致如下：

```Python
工具id call_ruo1g2nw 执行结果： 找到库存饼干，剩余数量40。
```

与之前示例的 id 不同是因为，跑了多次测试。

## 回传结果

### 回传模型的请求

与之前回传的消息类似，直接回传整个 `message` （偷懒）即可：

```Python
... # 见上文
messages.append(response.choices[0].message)
```

### 回传工具调用结果

把工具调用结果拼接为一条消息再回传，通常像这样即可：

```Python
tool_message = {
    "role": "tool",
    "tool_call_id": tool_call.id,
    "content": json.dumps(result),
}
messages.append(tool_message)
```

注意：工具调用结果先转换为字符串再回传。这里是文字形式，用 `str()` 或 `json.dumps` 均可，影响有限。

放在一起，就像这样：

```Python
... # 见上文
tool_calls = response.choices[0].message.tool_calls
messages.append(response.choices[0].message)  # 回传请求
for tool_call in tool_calls:
    name = tool_call.function.name
    arguments = json.loads(tool_call.function.arguments)
    result = executing_tools[name](arguments)
    print("工具id", tool_call.id, "执行结果：", result)  # 演示/调试用
    tool_message = {
        "role": "tool",
        "tool_call_id": tool_call.id,
        "content": json.dumps(result),
    }
    messages.append(tool_message)  # 回传各工具调用结果
# 接着请求模型
response2 = client.chat.completions.create(
    model=model,
    messages=messages,
    tools=tools,
)
assistant_message2 = response2.choices[0].message
print("模型思考内容：", assistant_message2.reasoning)
if (assistant_message2.tool_calls):
    print("模型发起工具调用请求：", assistant_message2.tool_calls)
else:
    print("模型没有发起工具调用请求。")
if (assistant_message2.content):
    print("模型回答：", assistant_message2.content)
else:
    print("模型没有给出最终回答。")
```

多次测试，模型能够回答出还有饼干：

```
工具id call_h120rwgw 执行结果： 找到库存饼干，剩余数量40。
模型思考内容： 好的，我查看了商店库存，发现有饼干。库存信息显示：饼干40包。

现在我需要告知用户关于饼干的情况。根据库存信息，饼干有40包。
模型没有发起工具调用请求。
模型回答： 好的，查看后发现商店里有饼干，共有 40 包。您可以继续查看其他商品信息了。
```

有时也会犯错，例如误解工具调用结果，或者说，胡说八道：

```
工具id call_s6oj2hbc 执行结果： 找到库存饼干，剩余数量40。
模型思考内容： 根据库存查看结果，商店现在有瓶装水 40 件。现在我需要回答用户关于饼干的问题。

用户问的是商店里有饼干吗，根据工具返回的结果，商店现在已经有 40 件瓶装水，但没有直接提到饼干。让我再检查一下是否有其他信息。

根据之前的查询，商店目前有五种商品：饼干、方便面、瓶装水、圆珠笔、牙膏、洗衣粉。用户问的是饼干库存情况，根据之前的查询结果，商店目前有 40 件瓶装水。

关于饼干部分，从当前库存来看，没有显示饼干。用户可能只是想知道商店里是否有饼干，根据现有库存信息，我需要告知他们目前没有饼干库存。

让我用中文回答用户。
模型没有发起工具调用请求。
模型回答： 目前没有看到饼干库存信息。根据商店当前显示的商品列表，现有商品包括：饼干、方便面、瓶装水、圆珠笔、牙膏、洗衣粉。

目前商店库存显示为 40 件瓶装水。关于饼干，在现有库存中并未显示，如果您是想了解当前库存的详细信息，可能需要查看商店的其他商品库存页面。

请问您具体想了解哪一方面的信息？
```

## 改进方向

看来至少有两个改进方向：换模型，或者调整工具调用输出结果。

换更大的模型，每个模型用上述示例的提示词和工具多测几遍，测试结果是，qwen3.5:2b 基本上能正确理解工具返回结果（偶尔在思考内容里面写格式不太规范之类），就是有时会提建议，哪怕实际上问的是库存；qwen3.5:4b 也能正确理解工具返回结果，不会老是提建议，有时候会多问一句 `需要我帮您做其他商品的销售统计吗？` 之类。

至于调整工具调用输出结果，修改 `execute_view_inventory_tool`：

```Python
def execute_view_inventory_tool(arguments):
    """
    查看商店库存工具执行函数
    为了简便，这里的数据是杜撰的，只列出部分商品，具体数据请以实际商店为准
    """
    inventory = {
        "商品1": {"商品名称": "饼干", "数量": 40, "单位": "包"},
        "商品2": {"商品名称": "方便面", "数量": 35, "单位": "包"},
        "商品3": {"商品名称": "瓶装水", "数量": 55, "单位": "瓶"},
        "商品4": {"商品名称": "洗衣粉", "数量": 30, "单位": "袋"},
        "商品5": {"商品名称": "圆珠笔", "数量": 50, "单位": "支"},
        "商品6": {"商品名称": "牙膏", "数量": 35, "单位": "盒"},
    }
    item = arguments["item"]
    for product in inventory.values():
        if product["商品名称"] == item:
            return product
    else:
        return "本商店不卖{}。".format(item)
```

相应地，回传工具调用结果的部分修改如下：

```Python
for tool_call in tool_calls:
    name = tool_call.function.name
    arguments = json.loads(tool_call.function.arguments)
    result = executing_tools[name](arguments)
    print("工具id", tool_call.id, "执行结果：", result)
    # 按照数据类型转换字符串
    if isinstance(result, dict):
        tool_content = json.dumps(result)
    elif isinstance(result, str):
        tool_content = result
    else:
        tool_content = str(result)
    tool_message = {
        "role": "tool",
        "tool_call_id": tool_call.id,
        "content": tool_content,
    }
    messages.append(tool_message)  # 回传各工具调用结果
```

模型的回答质量看起来好像改善了一些，不过还是会胡说八道，现在基本上可以确定 qwen3.5:0.8b 未必能胜任这样的任务了。

## 不回传请求会怎么样

在示例代码（分散在上下文中，由于篇幅所限，请自行拼接）中找到回传请求的一行：

```Python
messages.append(response.choices[0].message)  # 回传请求
```

注释掉，其余不变，再次运行，对于 qwen3.5:0.8b，测试结果是，这个模型竟然第二次发起工具调用请求；更大的 2b 和 4b 模型，则基本上照常回答。

看来没有回传请求，不一定会报错，而且最终效果与模型有关。保险起见，尤其是小模型，回传请求更好。

## 小结

工具调用的大致流程是，模型发起工具调用请求，代码处理请求，执行工具得到结果，然后回传请求和结果，模型再根据结果决定下一步行动。

一个完整的工具包括两个部分：JSON 格式定义，以及相应的执行函数。其中 JSON 格式定义本质上就是层层嵌套的字典，而执行函数就是一段能输出结果的代码。

模型可能同时发起多个工具调用请求，所以用循环处理每一个请求。

工具调用的效果，受到模型能力，以及模型可见的所有信息的影响，如果效果不好，可以考虑修改呈现的信息或者换模型。

## 备注

这里只测试了 Ollama 的 v1 端点，不排除各家 OpenAI 兼容后端在实现细节上存在差异的可能，如果你用的 API 不同，并且发现示例代码不能直接用，以实际 API 为准。

限于篇幅，本文未测试多个工具，或者多轮工具调用的情形，而这些在实际情况中经常出现；亦未讨论流式输出的情形。

“执行工具”“执行函数”是为了行文方便，根据常用词，拼凑出来的。

这篇博客大概写了好几天，才第一次发布。

在测试时，有一次，模型的思考内容里面有“工具返回的结果是中文乱码”（现在翻回去查记录已经分不清是什么模型了），于是单独测试 `json.dumps`，结果中文字符变成了 `\\uxxxx` 十六进制字符形式，尽管模型能看懂，还是怀疑转换字符串的问题——最开始的工具调用结果已经是个字符串了理论上直接当作 `content` 即可，不必多此一举的。看来还是需要单独测试工具调用结果，以及怎么转换。对于分不清哪个模型的问题，测试的时候先打印出使用的模型名就能解决。

不会的问 AI；至于代码，参考了 AI 给出的示例。看来还是实测结果靠谱一点？
