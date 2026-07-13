---
title: '免费 AI 编程工具链：OpenCode + 三层约束让免费模型写出生产级代码'
date: 2026-07-05 10:00:00
tags: [AI 编程, OpenCode, 免费模型, 编程工具链, 约束策略, 生产级代码]
categories: [技术笔记]
---


## 背景：免费模型能写代码吗？

市面上有大量”免费 AI 编程工具”——Cursor、Windsurf、Trae-CN……它们提供免费的模型额度，让开发者零成本用 AI 写代码。但免费模型（GLM、DeepSeek、MiniMax 等）的代码能力参差不齐，直接用经常翻车：改了不该改的文件、删除已有功能、留下半成品代码。

问题不是模型不够强，而是**没有给模型足够的约束**。同一个免费模型，没有约束时改出 bug，加了约束后一次到位。区别在于：你是否把”项目架构””改动范围””编码规范”这些上下文喂给了模型。

本文记录一套经过反复验证的工具链：用 OpenCode CLI 作为执行器，配合三层 AGENTS.md 约束机制，让免费模型产出生产级代码——零 API 费用，可控可验收。

## 工具链全景

```scss

1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19

┌─────────────────────────────────────────────────────┐
│  AI Agent (编排者)                                    │
│  策划 → 委派 → 检查 → 处置 (PDCA)                     │
└──────────────────────┬──────────────────────────────┘
                       │ opencode run -f TASK.md
                       ▼
┌─────────────────────────────────────────────────────┐
│  OpenCode CLI (执行器)                                │
│  合并三层 AGENTS.md → 注入 system prompt               │
│  ├─ 全局 AGENTS.md (编码行为准则)                      │
│  ├─ 项目 AGENTS.md (架构认知)                          │
│  └─ TASK 文件 (精准指令)                               │
└──────────────────────┬──────────────────────────────┘
                       │ 调用免费模型
                       ▼
┌─────────────────────────────────────────────────────┐
│  免费模型 (DeepSeek V4 Flash / GLM / MiniMax)         │
│  在约束下生成代码                                      │
└─────────────────────────────────────────────────────┘

```

### 核心组件

组件
角色
费用

OpenCode CLI
代码执行器，支持多种模型
免费（开源）

免费模型
实际写代码的 LLM
免费（有速率限制）

AGENTS.md
项目上下文 + 编码规范
需手动编写

TASK 文件
单次任务的精准指令
需手动编写

关键分工：**AI Agent 负责策划和检查，OpenCode 负责执行**。Agent 不直接写代码（节省自己的 Token 额度），而是写任务规格书（TASK-*.md），交给 OpenCode 执行，然后验收。

## 三层约束机制：核心创新

OpenCode 启动时会读取多个 AGENTS.md 文件，通过**并集（Union）合并**注入 LLM 的 system prompt。这不是覆盖，是叠加——三层约束同时生效。

### 第一层：全局 AGENTS.md（行为准则）

路径：`~/.config/opencode/AGENTS.md`

内容：Karpathy 四原则——编码前思考、简洁优先、精准修改、目标驱动执行。

作用：告诉模型**怎么改代码**——不准乱动没坏的东西，只改要求改的，改完必须验证。

```markdown

1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16

## Karpathy 四原则

### 1. 编码前思考
列出所有需要改动的文件，想清楚每个文件改什么，不动什么。

### 2. 简洁优先
只做最小改动。不创建新文件，不添加新功能，不搞抽象。

### 3. 精准修改
只改任务清单列出的内容。绝对不碰禁止清单里的文件。
- 不改相邻代码、注释、格式
- 不改没坏的东西
- 不顺手优化或重构

### 4. 目标驱动执行
每改完一个文件就验证语法。全部改完运行 git diff --stat 确认。

```

### 第二层：项目 AGENTS.md（架构认知）

路径：`<project>/AGENTS.md`

内容：技术栈、API 端点清单、目录结构、数据库模型、关键约定。

作用：告诉模型**改哪里**——知道项目的模块边界、已有的设计模式、哪些文件不能碰。

一个典型的项目 AGENTS.md 会包含：

```markdown

1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16

## 项目概述
- 名称：交易系统
- 技术栈：FastAPI + React + SQLite
- 架构铁律：策略脚本不得在 api.py 添加端点

## API 端点清单（50 个）
| 方法 | 端点 | 说明 |
|------|------|------|
| GET  | /api/portfolio | 持仓概览 |
| POST | /api/trade     | 执行交易 |
...

## 禁止改动的文件
- app/models.py — 数据模型
- app/config.py — 配置
- frontend/ — 任何前端文件

```

### 第三层：TASK 文件（精准指令）

路径：`<project>/TASK-YYYYMMDD-feature.md`

内容：本次任务的具体改动清单、禁止改动清单、验收标准。

作用：告诉模型**这次只改这些**——精确到文件名和行为。

```markdown

1
2
3
4
5
6
7
8
9
10
11
12
13
14

# TASK: 数据源配置化

## 改动清单
1. app/config.py — 新增 DATA_SOURCE 配置项
2. app/services/data_fetcher.py — 读取配置选择数据源

## 禁止改动清单（绝对不碰）
- app/api.py
- app/models.py
- frontend/

## 验收标准
- git diff --stat 只显示 2 个文件被修改
- python -c "import ast; ast.parse(open('app/config.py').read())"

```

### 三层缺一不可

症状
缺哪层
根因

改了不该改的文件
第一层（全局）
模型不知道”只改要求改的”

改了错误的模块
第二层（项目）
模型不知道项目结构

改了多余的行号
第三层（TASK）
模型不知道本次任务的精确范围

实测案例：同一个任务，Round 1 全局 AGENTS.md 为空（四原则未注入），模型改了一堆不该改的文件，前端白屏。Round 2 补全三层约束后，模型只改了预期的 3 个文件，1 小时完成，零 regression。

## 免费模型实测：哪个能用

OpenCode 内置多个免费模型，能力差异巨大：

模型
编码能力
适用场景

DeepSeek V4 Flash
★★★★★
复杂多文件任务、全局重构

GLM-4.6 (big-pickle)
★★★☆☆
简单单文件改动

MiniMax M3
★★★☆☆
通用任务

### 关键教训：模型选择决定返工量

同一个 bug 修复任务：

- **GLM-4.6 做**：改了一半，留下重复的 return 代码块（dead code），还误改了 CSS 文件导致前端白屏。需要人工补修。

- **DeepSeek V4 Flash 做**：正确诊断根因（缺少入口文件导致 tree-shaking），一次创建完整修复（4 处改动），构建验证部署一条龙。

结论：**编码任务无脑选 DeepSeek V4 Flash。** GLM-4.6 只用于最简单的单文件改动。跳过模型选择直接用默认模型 = 流程违规。

### 模型可用性会波动

免费模型随时可能返回 400 “Model does not exist” 或 429 速率限制。正确流程：

- `opencode models` 列出当前可用模型

- 按能力排序选最强的

- 首选模型不可用，直接换次强的，**不要问用户**

- 所有免费模型都不可用时，才考虑付费 provider

## 代码审查实战：report-only 模式

除了写代码，免费模型还能做代码审查。最佳实践是 **report-only 模式**：让模型只报告问题，不改代码。

```bash

1
2
3

opencode run "审查 app/api.py，只报告 bug 和安全问题，不要改代码。维度：逻辑错误、SQL注入、异常处理缺失、性能瓶颈。每个问题标注行号和严重级别（P0/P1/P2）。" \
  --model opencode/deepseek-v4-flash-free \
  -f review-checklist.md

```

为什么要分离审查和修复？

模式
模型行为
风险

审查+修复一起做
边查边改
容易改过头，误删功能

只审查（report-only）
只输出报告
无风险，报告质量高

单独修复
按报告逐项修
可控，每步可验收

实测：report-only 模式下，GLM-4.6 在 api.py（2000+ 行）中找到 9 个 bug，含 3 个 P1 全部准确。然后再写单独的修复 TASK，逐项修复，每步验收。

## 完整工作流

```ada

1
2
3
4
5
6
7
8
9
10
11
12
13
14
15

1. AI Agent 分析需求
   ↓
2. 编写 TASK-*.md（改动清单 + 禁止清单 + 验收标准）
   ↓
3. opencode models  →  选最强免费模型
   ↓
4. opencode run "阅读 TASK-xxx.md 并执行" --model <best-free>
   ↓
5. AI Agent 验收：
   - git diff --stat（只改了预期文件？）
   - 语法检查（ast.parse / tsc）
   - 功能测试
   ↓
6. 不通过 → 补充 TASK 回到步骤 4
   通过 → 更新文档 + 提交

```

### 典型调用示例

```bash

1
2
3
4
5
6
7
8
9
10
11
12
13

# 检查模型可用性
opencode run "Reply with: OK" --model opencode/deepseek-v4-flash-free

# 执行任务（附带 TASK 文件）
opencode run "阅读 TASK-20260705-data-source-config.md 并按说明执行" \
  --model opencode/deepseek-v4-flash-free \
  -f TASK-20260705-data-source-config.md \
  --workdir /home/user/project

# 验收
cd /home/user/project
git diff --stat
python -m py_compile app/config.py

```

## 踩坑总结

坑
症状
修复

全局 AGENTS.md 路径写错
四原则不生效，模型乱改
`~/.config/opencode/`（不是 `~/.opencode/`）

模型选错
复杂任务改一半留垃圾
无脑用 DeepSeek V4 Flash

TASK 文件在 workdir 外
OpenCode 拒绝读取
TASK 放项目目录内

`~` 路径解析错误
文件写到错误位置
一律用绝对路径

审查修复一起做
误删已有功能
分离审查（report-only）和修复

## 总结

维度
方案

执行器
OpenCode CLI（开源、免费）

约束机制
三层 AGENTS.md 并集合并

首选模型
DeepSeek V4 Flash（免费最强）

代码审查
report-only 模式，审查与修复分离

成本
$0（全部使用免费模型额度）

免费模型不是不能用，关键在于**约束**。三层 AGENTS.md 机制把”项目架构””编码规范””任务范围”三层上下文叠加注入，让免费模型在充分约束下的产出质量，逼近甚至超过无约束的付费模型。

核心心法：**Agent 策划，OpenCode 执行，免费模型干活，人工验收。** 每一层各司其职，成本为零。