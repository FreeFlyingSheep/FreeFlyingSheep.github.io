---
title: "MCP 相关概念"
date: 2026-03-08T12:08:50+08:00
description: ""
menu:
  sidebar:
    name: "MCP 相关概念"
    identifier: ai-agent-mcp
    parent: ai-agent
    weight: 20
tags: ["AI", "Agent", "MCP"]
categories: ["AI"]
---

[AI 学习笔记系列](/posts/ai/ai)的第八篇，介绍 MCP 相关的基本概念，包括对工具调用的进一步探讨，MCP 的定义与设计原则，以及 MCP 服务器框架。

<!--more-->

## 再探工具调用（Tool Calling）

之前文章提到了工具调用的基本概念，这里进一步探讨一些相关的细节。

**模型本身不会执行工具，真正执行工具的是外部程序。**

模型能做的只是根据上下文生成一段结构化的调用意图，比如当用户询问 `帮我查一下上海明天的天气` 时（假设今天是 2026 年 3 月 8 日），模型可能会生成如下的调用意图（也就是 Tool Call）：

```json
{
  "name": "get_weather",
  "arguments": {
    "city": "Shanghai",
    "date": "2026-03-09"
  }
}
```

即模型判断这个问题需要调用一个名为 `get_weather` 的工具，并且参数应该是这些。

然后外部运行时会先用解析器（Parser）解析这段调用意图，把它还原为结构化的工具调用对象。
再由执行器（Executor）真正执行对应工具，比如调用天气 API。

综上所述，一个完整的工具调用流程可能是这样的：

1. 用户提问
2. 工具定义会通过 prompt、消息模板或 API 的工具描述接口提供给模型
3. 模型生成 tool call
4. 外部程序解析 tool call
5. 外部程序执行对应工具
6. 工具结果被回填给模型
7. 模型根据结果生成最终回答

### 工具调用的支持

有些模型介绍会提到该模型支持工具调用，这里的“支持”通常不只是指模型能生成结构化的调用意图，还包括它能较稳定地判断何时调用工具、选择正确工具、生成符合 schema 的参数，并在工具结果回填后继续完成回答。

*备注：这里提到的“模型”，如果严格来说，不再指的是狭义的 LLM 本体（裸模型），而是封装了工具调用、上下文构造、结果回填、记忆和调度逻辑的模型服务，甚至可以看作一个轻量级的 agent 系统。*

即使是一些不支持工具调用的模型，也可以通过 Prompt Engineering 的方式让模型生成类似的结构化文本，然后由外部程序进行解析，但这通常不如直接支持工具调用的模型高效和准确，这点很容易理解。
很多所谓“不支持”的模型，其实不是完全不能做，而是缺少专门对齐和配套的运行时（Runtime）。

*备注：工具调用并不是一个完全二元的能力，很多模型都能在 prompt 约束下模仿工具调用，但稳定性和可用性差异很大。*

对于那些声称支持工具调用的模型，它们的优化可能采用了这些方式：

- Prompt 约束：，是把工具定义、参数 schema、输出格式要求直接写进 prompt 里，这种方式最简单，但最不稳定
- 训练与对齐优化：通过 SFT、RL、DPO 等方法优化模型在工具调用场景下的表现，让模型更准确地生成符合规范的调用意图
- 推理与运行时优化：通过结构化输出约束、schema 校验、模板设计、工具选择机制等方式提升整体稳定性

### 工具调用的解析

不同的模型所生成的调用意图可能在格式上有所不同，因此解析器需要针对不同模型的输出进行适配。
有些场景下 tool call 并不是 parser 从自由文本里“猜”出来的，而是模型服务本身已经把它作为结构化字段返回。

解析器会做如下的工作：

- 识别输出里有没有工具调用
- 提取工具名
- 提取参数
- 校验格式
- 处理脏输出
- 必要时做纠错
- 如果存在多个工具调用，还需要正确拆分

常见的不同模型的调用意图格式可能有：

- OpenAI function-call 风格
- JSON block
- XML tag
- 特定模板文本
- 某个模型自定义的 chat template

在一些 API 中，tool call 会直接以结构化字段返回，而在另一些本地部署或开源模型场景中，解析器则需要从模型生成的文本中提取和纠正这些内容。

### 工具调用的回填

工具调用的结果回填给模型时，也需要注意格式和内容的规范化，确保模型能够正确理解工具的输出。

这一步本质上是在做上下文构造，因此和 Context Engineering 密切相关，回填的内容可能需要做一些处理，比如结果的摘要、结构化、错误处理等，以便模型能够更好地利用这些信息生成最终回答。

## 模型上下文协议（MCP）

假设我们已经有了支持工具调用的模型，我们还需要完成下面的工作，才算真正实现一个工具调用系统：

- 手动定义工具 schema
- 把工具描述塞进 prompt 或 API 请求
- 根据模型输出解析 tool call
- 执行对应工具
- 把结果重新组织后回填给模型

这里面可能遇到的问题有很多，比如：

- 不同工具接入方式不一致
- 不同客户端要重复做适配
- Schema、返回格式、调用方式缺乏统一规范
- 模型服务、IDE、Agent、业务系统之间难以互操作

那么有没有一个统一的标准 / 协议，来规范工具调用的定义、描述、解析和执行呢？
MCP（Model Context Protocol）就是这样一个开源的、连接 AI 应用和外部系统的标准。

MCP 的核心概念包括了 Tools、Resources 和 Prompts。

### 工具（Tools）

Tools 是 MCP 中最接近传统 tool calling 概念的一类能力。

官方规范把 tools 定义为：它们用于让语言模型与外部系统交互，例如查询数据库、调用 API 或执行计算。

MCP server 可以暴露一组可调用的工具，每个工具有自己的名字和输入 schema，模型或上层运行时在合适的时候可以选择调用它们。
例如，一个天气服务可以暴露一个工具：

- get_weather
- 输入参数：city、date

### 资源（Resources）

官方规范把 resources 定义为：服务端可以向客户端暴露的数据，用来给语言模型提供上下文，例如文件、数据库 schema 或应用特定的信息，每个 resource 都由 URI 唯一标识。
例如：

- 一份本地文档
- 一个代码文件
- 某张数据库表的 schema
- 某个页面的抓取结果
- 某条业务记录
- 一个模板化的资源 URI

### 提示词（Prompts）

官方规范把 prompts 定义为：服务端向客户端暴露的提示词模板（Prompt Templates），客户端可以发现它们、获取它们的内容，并传入参数进行定制。
规范还特别强调，prompts 倾向于是用户控制的，也就是更适合作为用户显式选择的交互入口。

例如，一个文件系统服务可以提供：

- read_file
- write_file

这些 prompts 可能带参数，比如：

- 文档 URI
- 输出语言

### 通知消息（Notifications）

MCP 还定义了 Notifications，他们是由服务端（server）主动发送给客户端（client）的消息，用来在状态变化时主动告知对方。
Notifications 是单向通知机制，即发送方（server）发出通知，接收方（client）不需要返回响应。

比如，一个 server 新增了一个 tool，或者某个 resource 的内容更新了，就可以发一个 notification 给 client，告诉它“这里有变化了，可以重新获取一下最新信息”。

### 传输方式（Transport）

Transport 可以理解为 MCP client 和 MCP server 之间具体通过什么通道通信。

MCP 目前主要支持两种标准传输方式：

- **stdio transport**：通过标准输入/输出通信，适合同机上的本地进程之间直接通信
- **Streamable HTTP transport**：通过 HTTP 进行消息传输，适合远程 MCP server，也支持标准的 HTTP 认证方式

而**通信协商**（Transport Negotiation）指的是 client 和 server 在建立连接时，对该用哪种传输方式、如何完成这条连接进行协商和处理。

### MCP 工作流

一次完整的 MCP 交互可能是这样的：

1. 客户端与服务端初始化并协商能力
2. 客户端发现服务端提供的 tools、resources、prompts
3. 用户发起任务
4. 客户端按需读取 resources、调用 tools、应用 prompts
5. 结果被整理进上下文
6. 模型生成最终回答

以上文的天气和文件系统服务为例，假设用户的请求是：`帮我看看上海明天的天气。如果不下雨，我就去图书馆；如果下雨，我就呆在家里。顺便帮我把明天的安排写在 TODO 文件里。`
一次完整的请求流程可能是这样的：

1. Client 与 Server 初始化并协商能力
   1. Host 内部的 MCP client 启动
   2. Client 分别连接相关的 MCP server，例如：
      - weather server
      - filesystem server
   3. Client 与各个 server 完成初始化握手
      1. 确认 MCP 协议版本
      2. 确认双方支持的能力
      3. 建立后续请求/响应通道
2. Client 发现 Server 提供的 tools、resources、prompts
   1. Client 向 weather server 查询可用能力
      - 发现 tool：
        - `get_weather(city, date)`
   2. Client 向 filesystem server 查询可用能力
      - 发现 tools：
        - `read_file(path)`
        - `write_file(path, content)`
      - 发现 resources：
        - `file:///todo/TODO.md`
      - 发现 prompts（如果有）：
        - `append_todo_item`
        - `plan_tomorrow`
3. 用户发起任务
   1. 用户输入请求：
      - 帮我看看上海明天的天气
      - 根据是否下雨决定去图书馆还是呆在家里
      - 把明天的安排写进 TODO 文件
   2. 模型或上层运行时先理解任务
      - 子任务 1：查询上海明天天气
      - 子任务 2：基于天气结果生成明天安排
      - 子任务 3：读取或更新 TODO 文件
4. Client 按需读取 resources、调用 tools、应用 prompts
   1. 调用天气 tool
      1. Client 调用：
         - `get_weather(city="Shanghai", date="2026-03-09")`
      2. Weather server 返回结果，例如：
         - 天气：Cloudy
         - 降雨：false
         - 温度：18°C
   2. 模型或运行时根据天气结果做条件判断
      - 如果 `rain = false`
        - 明天安排 = 去图书馆
      - 如果 `rain = true`
        - 明天安排 = 呆在家里
   3. 读取 TODO 文件（如果需要先查看原内容）
      - Client 读取 resource：
        - `file:///todo/TODO.md`
      - 或调用 tool：
        - `read_file(path="/todo/TODO.md")`
   4. 组织要写入的内容
      - 例如生成一条新的 TODO：
        - `- [ ] 明天去图书馆`
      - 或：
        - `- [ ] 明天呆在家里`
   5. 写入 TODO 文件
      - Client 调用 filesystem server 的 tool：
        - `write_file(path="/todo/TODO.md", content="...更新后的内容...")`
   6. 如果 server 提供了 prompt，也可以辅助组织写入格式
      - 例如使用 prompt：
        - `append_todo_item`
      - 用来统一 TODO 条目的格式
5. 结果被整理进上下文
   - Client 将外部结果整理后回填给模型
     - 天气查询结果
       - 上海明天多云
       - 18°C
       - 不下雨
     - 条件判断结果
       - 明天安排：去图书馆
     - 文件写入结果
       - TODO 文件已更新
6. 模型生成最终回答
   - 模型基于上下文生成面向用户的最终回复，例如：
     - 上海明天不下雨，18°C，适合出门。
     - 所以你明天可以去图书馆。
     - 我已经把“明天去图书馆”写进 TODO 文件了。

### MCP 开发方案

如果要自己写 MCP，常见路线大致有三种。

官方 **MCP SDK** 最贴近协议本身，适合想按规范实现 MCP client 和 server 的场景。
它的定位是构建 MCP servers 和 clients 的标准方式，支持暴露 tools、resources、prompts，构建可连接任意 MCP server 的 client，并支持本地与远程传输方式。

**LangChain MCP** 更像一层适配层，适合已经在用 LangChain 或 LangGraph 的项目，用来把 MCP 能力接进现有 agent 栈。
`langchain-mcp-adapters` 可以把一个或多个 MCP server 上定义的 tools 接进 LangChain agent 工作流。
`MultiServerMCPClient` 还支持连接多个 MCP server，并且默认按无状态方式为每次工具调用创建新的 MCP 会话。

**FastMCP** 则更强调 Python 开发体验，适合快速起步和高效构建 MCP 应用。
它可以快速构建 MCP servers、clients 和 applications，并支持把 Python 函数直接暴露成 MCP tools。
同时也支持从 OpenAPI 或 FastAPI 集成，根据在线文档或函数签名自动生成工具定义和 schema。
它还会自动处理不少底层细节，例如 schema、校验、文档生成，以及 client 侧的 transport negotiation、authentication 和协议生命周期。

*备注：MCP SDK 更适合学协议，LangChain MCP 更适合集成 agent，FastMCP 更适合快速开发。*
