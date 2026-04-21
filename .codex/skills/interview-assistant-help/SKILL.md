---
name: "interview-assistant-help"
description: "Provides onboarding and usage help for the interview_assistant project. Invoke when the user asks how to use this project, wants a quick start, asks what features it has, or effectively means /help."
---

# Interview Assistant Help

用于向用户清晰介绍当前面试助手项目的功能、输入材料、能力边界和推荐起手方式。

## Mission

- 让新用户在 1 次回复内理解这个项目能做什么
- 让用户知道两类核心输入分别服务什么流程
- 让用户知道应该从哪个入口开始
- 让用户能直接复制命令或用自然语言开始

## When To Invoke

- 用户输入 `/help`
- 用户询问“怎么用”“使用方法”“如何开始”“有哪些功能”
- 用户看起来是第一次使用这个项目

## Core Inputs

回复时要先让用户理解这两个核心输入：

1. `简历.docx`
   - 用于综合面试、项目深挖和薄弱点分析
2. `题库/*.md`
   - 用于抽题复习、题目保存和覆盖分析

## Output Contract

默认按下面结构组织回复：

1. 一句话说明项目用途
2. 说明 4 个核心能力
3. 明确两类核心输入的作用
4. 给出 3-5 步快速开始
5. 给出可复制命令示例
6. 给出适合新手的推荐入口

## What To Explain

优先说明这 4 个核心能力，并明确边界：

1. 模拟面试
   - `/mock-interview`
   - 先确认模式、方向、难度、题量和是否即时点评
   - 出题前要参考联网高频题信号
   - 综合面试结合简历项目，技术面试默认不主动引用项目经历
2. 抽题复习
   - `/review-quiz`
   - 只从本地 `题库/*.md` 抽题
   - 不联网生成新题
3. 保存题目
   - `/save-interview`
   - 把当前题目按方向保存到 `题库/*.md`
   - 先检查相似题，再决定融合更新、追加新题或保持不变
4. 薄弱点分析
   - `/weak-analysis`
   - 对比简历与题库覆盖，输出补题优先级
   - 只做分析与建议，不直接替代面试或抽题流程

## Scope Boundaries

- `/mock-interview`：练联网高频新题，技术面默认不主动混入项目经历，综合面试结合简历项目
- `/review-quiz`：只复习本地题库已有题目，不联网找新题
- `/save-interview`：把当前题目和答案归档到 `题库/*.md`
- `/weak-analysis`：只做覆盖分析和补题建议，不直接出题

## Quick Start

默认按 3-5 步给用户快速开始：

1. 把 `简历.docx` 放到项目根目录
2. 确认 `题库` 目录存在，并已有或准备建立分类文件
3. 先从下面任选一个入口开始：
   - `/mock-interview`
   - `/review-quiz`
   - `/weak-analysis`
4. 若用户不想记命令，也可直接用自然语言描述需求

## Example Commands

优先给这些可复制示例：

- `/help`
- `/mock-interview`
- `/mock-interview 技术面试 Redis 标准一面`
- `/mock-interview 技术面试 MySQL 深入追问`
- `/review-quiz`
- `/weak-analysis`

同时可补充自然语言示例：

- `开始综合面试`
- `来一场 JVM 技术面，标准一面`
- `抽几道并发题考我`
- `分析一下我现在的薄弱点`

## Interaction Rules

- 用户像第一次使用时，优先用清晰、简洁、面向新手的表达
- 如果涉及用户选择，必须把候选项明确列出来
- 不要一开始输出内部实现细节或过长说明
- 若用户没有明确方向，优先推荐从 `/mock-interview` 或 `/review-quiz` 开始

## Quality Bar

- 回复必须让用户明确区分 4 个能力的边界
- 回复必须明确提到 `简历.docx` 和 `题库/*.md`
- 回复必须提供至少 3 个可直接复制的示例
- 回复必须让用户能在读完后立即开始第一步

## Do

- 优先用“新手也能直接照做”的表达方式
- 明确告诉用户每个能力分别做什么
- 明确告诉用户两个核心输入分别服务什么流程
- 给出命令示例和自然语言示例两种入口

## Don't

- 不要一上来堆内部实现细节
- 不要混淆 `/mock-interview`、`/review-quiz`、`/weak-analysis` 的边界
- 不要只给抽象介绍而不给可执行下一步
