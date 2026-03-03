---
layout: post
title: ReAct 推理与行动智能体范式详解
categories: agent 
tags: AI Agent
date: 2025/03/29 22:00:00
---

![title](https://image.sideproject.cn/titlex/titlex_154.jpg)

ReAct（Reasoning + Acting）是一种将推理与行动交织的智能体范式，由 Google 和 Princeton 的研究者在 2022 年提出。ReAct 的核心思想是：让模型在解决任务时交替进行"思考"（Reasoning）和"行动"（Acting），通过推理确定下一步行动，通过行动获取观察结果，再基于观察继续推理，形成"Thought-Action-Observation"循环。这种设计使模型能够根据中间结果动态调整策略，而非一次性生成完整计划，提高了复杂任务的完成率和可解释性。ReAct 是当前智能体领域最广泛采用的范式之一，被 LangChain、AutoGPT 等框架实现。

**ReAct 的工作流程**

ReAct 的典型流程为：给定任务和可用工具，模型首先输出 Thought（推理步骤，解释当前状态和下一步计划），然后输出 Action（要调用的工具及参数），系统执行工具并返回 Observation（观察结果），模型将 Observation 作为输入继续生成下一个 Thought 和 Action，如此循环直至模型输出 Final Answer（最终答案）或达到终止条件。例如，对于"某公司 2023 年营收是多少"的问题，模型可能 Thought：需要搜索该公司财报；Action：调用搜索工具；Observation：返回搜索结果；Thought：从结果中提取营收数据；Action：无需再调用工具；Final Answer：2023 年营收为 X 亿元。

**ReAct 的优势与局限**

ReAct 的优势在于：推理与行动交织使模型能够根据观察调整策略，适应动态环境；显式的 Thought 输出提高了可解释性，便于调试和审计；对工具调用的依赖更灵活，可处理多步、多工具任务。局限在于：每步都需要模型生成，token 消耗和延迟较高；模型可能产生无效或错误的 Thought 和 Action，需要良好的提示设计和错误恢复机制。ReAct 适合需要多步推理、工具调用的场景，是理解智能体基础范式的重要参考。

**ReAct 的应用与变体**

ReAct 被广泛应用于问答、搜索、数据分析、代码生成等场景。LangChain 的 ReAct Agent、OpenAI 的 o1 模型中的推理模式，均在一定程度上借鉴了 ReAct 思想。Plan-and-Execute 等变体将"规划"与"执行"分离，先生成完整计划再执行，适合计划相对稳定的任务。ReAct 作为智能体基础范式，其思想将持续影响智能体架构设计。

**ReAct 的提示设计要点**

实现 ReAct 时，提示（Prompt）设计至关重要。需明确说明：可用工具列表及每个工具的功能、输入格式；Thought-Action-Observation 的格式要求；何时输出 Final Answer。良好的 few-shot 示例可显著提高模型的格式遵循和推理质量。工具描述要清晰，避免模型选择错误工具。对于复杂任务，可在提示中建议推理策略（如"先搜索再分析"）。ReAct 的提示工程是智能体开发的重要技能。

**ReAct 与 Plan-and-Execute 的对比**

Plan-and-Execute 先生成完整计划（步骤列表），再逐步执行，每步执行后可根据结果调整后续计划。其优势是计划更全局、可能更高效；劣势是对动态环境的适应不如 ReAct 灵活。ReAct 每步都根据当前观察决定下一步，更灵活但可能缺乏全局观。选择时可根据任务特点：计划相对稳定、可预见的任务，Plan-and-Execute 可能更高效；需要根据中间结果动态调整的任务，ReAct 更合适。两种范式可结合，如混合使用"高层规划+低层 ReAct"。

**ReAct 的实践建议**

使用 ReAct 时，建议设置最大步数限制，避免无限循环。为工具调用添加超时和错误处理。记录完整的 Thought-Action-Observation 轨迹，便于调试和审计。对于生产环境，可考虑 ReAct 的简化版本（如减少 Thought 的详细程度）以降低 token 消耗。ReAct 是理解智能体推理机制的基础，掌握 ReAct 有助于理解更复杂的 Agent 架构。
