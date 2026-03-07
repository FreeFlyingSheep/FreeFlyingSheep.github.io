---
title: "Agent 基本概念"
date: 2026-03-06T16:09:21+08:00
description: ""
menu:
  sidebar:
    name: "Agent 基本概念"
    identifier: ai-agent-basic
    parent: ai-agent
    weight: 10
tags: ["AI", "Agent"]
categories: ["AI"]
---

[AI 学习笔记系列](/posts/ai/ai)的第七篇，介绍 Agent 相关的基本概念，包括 Agent 的定义、推理与行动、工具调用、记忆以及架构设计范式和评估方法。

<!--more-->

## 智能体（Agent）

当模型不再只负责生成文本，而是会与世界交互、带有记忆（Memory，见后文）时，系统形态就从模型扩展为 Agent。
自动驾驶汽车、游戏 AI、聊天机器人、机器人等都是 Agent 的例子。

Agent 不一定要使用 LLM。
拿语言类 Agent 来说，历史上有一些例子：

- 纯文本：ELIZA
- 世界交互（仍属虚拟）：Siri / Alexa
- 机器人控制：SayCan

这些都更像反射式行为，不进行推理。

*备注：模型能力和系统能力很容易混淆。很多看起来很强的 Agent，强在系统编排（把提示词、工具、检索和流程组合成稳定系统），而不只是模型本身。*

LLM 当然可以作为 Agent，进行推理（如 CoT）或行动（如 RAG，RAG 的具体内容参见后续文章）。

**推理（Reasoning，更新信念）+ 行动（Acting，与世界交互）= ReAct**。
把推理本身作为模型可用动作之一，我们可以选择推理与行动交替，也可以每步让模型自由选择任意动作。
ReAct 是 Agent 的一个重要设计范式，很多后续方法都是在其基础上发展起来的。
ReAct 不需要训练，只需要 Prompt。

*备注：不需要额外训练并不意味着零成本，提示词设计和工具编排依然会显著影响最终效果。*

## 推理（Reasoning）

## 行动（Acting）

## 工具调用（Tool Calling）

## 记忆（Memory）

## Agent 架构设计范式

## Agent 框架

## Agent 评估
