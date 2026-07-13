---
title: 'Hermes Agent 模型配置完全指南——从单模型到多模型路由'
date: 2026-05-03 10:00:00
tags: [Hermes Agent, 模型配置, provider, 模型路由, fallback]
categories: [技术笔记]
---


> 

**作者:** 小瑞
**日期:** 2026-05-03
**版本:** v1.0
**适用对象:** AI Agent 开发者、模型运维人员

---

## 目录

- [背景](#%E8%83%8C%E6%99%AF)

- [Hermes Agent 的模型体系](#hermes-agent-%E7%9A%84%E6%A8%A1%E5%9E%8B%E4%BD%93%E7%B3%BB)

- [配置 Provider：连接你的模型](#%E9%85%8D%E7%BD%AE-provider%E8%BF%9E%E6%8E%A5%E4%BD%A0%E7%9A%84%E6%A8%A1%E5%9E%8B)

- [模型别名：简化切换](#%E6%A8%A1%E5%9E%8B%E5%88%AB%E5%90%8D%E7%AE%80%E5%8C%96%E5%88%87%E6%8D%A2)

- [Delegation 模型覆盖：子代理用什么模型](#delegation-%E6%A8%A1%E5%9E%8B%E8%A6%86%E7%9B%96%E5%AD%90%E4%BB%A3%E7%90%86%E7%94%A8%E4%BB%80%E4%B9%88%E6%A8%A1%E5%9E%8B)

- [运行时切换](#%E8%BF%90%E8%A1%8C%E6%97%B6%E5%88%87%E6%8D%A2)

- [完整配置示例](#%E5%AE%8C%E6%95%B4%E9%85%8D%E7%BD%AE%E7%A4%BA%E4%BE%8B)

- [踩坑记录](#%E8%B8%A9%E5%9D%91%E8%AE%B0%E5%BD%95)

- [总结](#%E6%80%BB%E7%BB%93)

---

## 背景

在使用 Hermes Agent 搭建个人 AI 助手时，一个常见需求是：**日常对话用本地模型省钱，复杂任务交给云端大模型**。我们的环境是这样的：

模型
来源
用途

Qwen 3.6 35B
本地 LM Studio
日常对话，主力模型

GLM-5
智谱 BigModel API
复杂推理、代码分析

今天我们就来一步步配置这个多模型方案。

---

## Hermes Agent 的模型体系

Hermes 的模型配置是一个三层结构：

```arduino

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

┌─────────────────────────────────────────┐
│           config.yaml（主配置）            │
│  model.default + model.provider          │
├─────────────────────────────────────────┤
│       model_aliases（别名快捷映射）        │
│  qwen → LM Studio / glm → BigModel      │
├─────────────────────────────────────────┤
│       delegation（子代理模型覆盖）         │
│  delegate_task 产生的子进程用什么模型      │
└─────────────────────────────────────────┘

```

理解这三层，就掌握了 Hermes 模型路由的全部。

---

## 配置 Provider：连接你的模型

### Provider 是什么

Provider 就是模型提供商。Hermes 支持以下内置 provider：

Provider
说明
需要的密钥

`openrouter`
OpenRouter 聚合平台
`OPENROUTER_API_KEY`

`anthropic`
Anthropic 直连
`ANTHROPIC_API_KEY`

`nous`
Nous Portal OAuth
`hermes login`

`zai`
智谱 GLM
`GLM_API_KEY`

`lmstudio`
本地 LM Studio
无需密钥

`custom`
任意 OpenAI 兼容端点
自定义

### 配置本地 LM Studio（Qwen）

LM Studio 是一等公民 provider，配置最简单：

```yaml

1
2
3
4
5
6
7

model:
  default: "qwen3.6-35b-a3b-256k"
  provider: "lmstudio"
  # LM Studio 默认连接 http://127.0.0.1:1234/v1
  # 如果改了端口或加了认证：
  # base_url: "http://10.28.9.6:1234/v1"
  # api_key: "your-key"  # 也可以放在 .env 文件中

```

### 配置云端 BigModel（GLM-5）

智谱 BigModel 用 `custom` provider，需要手动指定 base_url：

```yaml

1
2

# 在 .env 中设置密钥
# GLM_API_KEY=your-bigmodel-api-key

```

在 `config.yaml` 中：

```yaml

1
2
3
4

model:
  default: "GLM-5"
  provider: "custom"
  base_url: "https://open.bigmodel.cn/api/coding/paas/v4"

```

> 

**提示：** 密钥推荐放在 `~/.hermes/.env` 文件中，而不是直接写在 config.yaml 里。Hermes 启动时会自动加载 .env 文件。

---

## 模型别名：简化切换

当你要在多个模型间频繁切换时，`model_aliases` 是最优雅的方案。它让你用短名字代替完整配置。

### 配置方法

```yaml

1
2
3
4
5
6
7
8
9

model_aliases:
  qwen:
    model: "qwen3.6-35b-a3b-256k"
    provider: custom
    base_url: "http://10.28.9.6:1234/v1"
  glm:
    model: GLM-5
    provider: custom
    base_url: "https://open.bigmodel.cn/api/coding/paas/v4"

```

### 使用方法

配置好之后，在对话中直接输入：

```bash

1

/model glm

```

就能瞬间切换到 GLM-5。再输入 `/model qwen` 就切回来。Hermes 会自动从别名中解析出 provider、base_url 等信息。

**关键细节：** 别名的优先级高于 models.dev 目录（Hermes 内置的模型数据库），所以你甚至可以用别名路由到目录中不存在的端点（比如自建 Ollama 服务器）。

---

## Delegation 模型覆盖：子代理用什么模型

### 什么是 Delegation

Hermes 的 `delegate_task` 工具可以生成子代理来并行处理任务。默认情况下，子代理继承父代理的模型配置。

### 为什么要覆盖

实际场景中，你可能希望：

- **父代理**用便宜的本地模型做调度

- **子代理**用强大的云端模型做具体工作

### 配置方法

```yaml

1
2
3
4
5
6
7
8

delegation:
  # 所有子代理统一用 GLM-5
  model: "GLM-5"
  provider: "custom"
  base_url: "https://open.bigmodel.cn/api/coding/paas/v4"

  # 或者只指定 provider，让 Hermes 自动解析
  # provider: "openrouter"  # 使用 OpenRouter 上的模型

```

### 解析优先级

Hermes 内部的 `_resolve_delegation_credentials` 函数按以下顺序解析：

- **`delegation.base_url`** → 如果存在，强制使用 `custom` provider

- **`delegation.provider`** → 使用预定义的 runtime resolver（openrouter、nous、zai 等）

- **继承父代理** → 如果都不设置，子代理和父代理用同一个模型

```stylus

1
2
3
4
5

delegation.base_url 存在？
    ├─ 是 → custom provider + 指定的 base_url
    └─ 否 → delegation.provider 存在？
                ├─ 是 → resolve_runtime_provider() 自动解析
                └─ 否 → 继承父代理的 provider + model

```

---

## 运行时切换

除了配置文件，Hermes 还支持运行时切换模型：

### /model 命令

```bash

1
2
3

/model glm          # 使用别名切换
/model qwen         # 切换到本地 Qwen
/model claude-sonnet-4  # 直接指定模型名

```

### 辅助任务的模型路由

对于视觉分析、网页提取等辅助任务（auxiliary tasks），Hermes 有独立的 fallback 链：

```markdown

1
2
3
4
5

1. 主 Provider（如果支持该功能）
2. OpenRouter
3. Nous Portal
4. 原生 Anthropic
5. Custom Endpoint

```

这个链条是硬编码在 `auxiliary_client.py` 中的，确保辅助任务始终能找到可用的模型。

---

## 完整配置示例

以下是一个双模型配置的完整示例（`~/.hermes/config.yaml`）：

```yaml

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
20
21
22
23
24
25
26
27
28

# ═══════════════════════════════════
# 主模型：本地 Qwen（省钱、快速）
# ═══════════════════════════════════
model:
  default: "qwen3.6-35b-a3b-256k"
  provider: "custom"
  base_url: "http://10.28.9.6:1234/v1"

# ═══════════════════════════════════
# 别名：快速切换
# ═══════════════════════════════════
model_aliases:
  qwen:
    model: "qwen3.6-35b-a3b-256k"
    provider: custom
    base_url: "http://10.28.9.6:1234/v1"
  glm:
    model: GLM-5
    provider: custom
    base_url: "https://open.bigmodel.cn/api/coding/paas/v4"

# ═══════════════════════════════════
# 子代理：用云端 GLM 处理复杂任务
# ═══════════════════════════════════
delegation:
  model: "GLM-5"
  provider: "custom"
  base_url: "https://open.bigmodel.cn/api/coding/paas/v4"

```

对应的 `.env` 文件：

```ini

1
2

# BigModel API Key（GLM-5 用）
GLM_API_KEY=your-bigmodel-api-key-here

```

---

## 踩坑记录

### 1. LM Studio 端口问题

LM Studio 默认端口是 `1234`。如果在局域网另一台机器上运行，需要：

- 确保 LM Studio 开启了 CORS

- 在 config.yaml 中写明完整地址（如 `http://10.28.9.6:1234/v1`）

- 注意末尾的 `/v1` 不能少

### 2. custom provider 的 api_key 传递

使用 `custom` provider 时，API Key 的传递方式取决于目标服务：

场景
Key 来源

LM Studio（无认证）
不需要 key

BigModel
`GLM_API_KEY` 环境变量

自建 Ollama Cloud
`api_key` 字段或环境变量

### 3. 别名不会自动路由

`model_aliases` 只是一个快捷方式，**不会自动根据任务复杂度选择模型**。如果你想要”智能路由”，需要：

- 手动切换：遇到难题时 `/model glm`

- 配置 delegation：让子代理用更强的模型

### 4. delegation.provider 只支持预定义 Provider

`delegation.provider` 字段只接受预定义的 provider slug（如 `openrouter`、`nous`、`zai`）。如果要指向自定义端点，必须使用 `delegation.base_url` + `delegation.model` 组合。

---

## 总结

Hermes Agent 的模型配置虽然灵活，但核心就三个概念：

- **Provider** — 连接哪个模型服务

- **model_aliases** — 给常用模型起短名字，方便切换

- **delegation** — 控制子代理用什么模型

掌握这三点，就能轻松玩转多模型环境。我们的配置策略是：

> 

**本地 Qwen 做日常对话，云端 GLM 做复杂推理，delegation 让子代理自动使用更强的模型。**

这样既省钱又保证了能力上限。希望这篇指南能帮到同样在配置 Hermes Agent 多模型方案的朋友。

---

*本文由小瑞撰写，发布于「进化概率论」博客。*