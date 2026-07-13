---
title: 'MoA 踩坑实录：多模型协同推理的"无限循环"陷阱'
date: 2026-07-11 10:00:00
tags: [MoA, Mixture-of-Agents, 多模型协同, 无限循环, 踩坑记录]
categories: [技术笔记]
---


## 背景：什么是 MoA？

MoA（Mixture of Agents）是一种多模型协同推理架构。不同于传统 MoE（Mixture of Experts）在**单个模型内部**路由专家，MoA 在**多个独立模型之间**进行路由和聚合。

核心流程：

```excel

1
2
3
4
5
6
7

用户输入
   │
   ├──→ 参考模型 1（生成独立回答）
   ├──→ 参考模型 2（生成独立回答）
   ├──→ 参考模型 N（生成独立回答）
   │
   └──→ 聚合器（综合所有参考模型的回答，输出最终结果）

```

每个参考模型独立生成对同一个问题的回答，然后由一个聚合器模型（通常是更高智商的模型）综合所有回答，产出最终输出。

这套架构在 Agent 框架中可以通过配置文件轻松启用。看起来很美好——多个模型各抒己见，聚合器择优合并，理论上应该比单模型更准确、更全面。

但实际部署时，一个隐藏的陷阱让它在特定配置下变成了”无限循环”的噩梦。

## 问题：同一个配置，两个结果

某 AI Agent 平台支持多 profile（配置文件）隔离。不同的 profile 可以有不同的模型配置、MoA 参考模型组合等。

同一天，同一个 MoA 架构在两个 profile 上的表现：

Profile
参考模型数
结果

Profile A（基础配置）
2 个（均为云端 API）
✅ 首次对话成功

Profile B（增强配置）
3 个（含 1 个本地模型）
❌ 无限循环

Profile A 的 MoA 配置：

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

moa:
  presets:
    default:
      reference_models:
        - provider: custom:new-api
          model: sensenova-6.7-flash-lite
        - provider: custom:new-api
          model: agnes-2.0-flash
      aggregator:
        provider: custom:bigmodel
        model: GLM-5
      reference_max_tokens: 600
      max_tokens: 4096
      enabled: true

```

Profile B 在此基础上**多加了一个本地部署的开源模型**：

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

moa:
  presets:
    default:
      reference_models:
        - provider: custom:new-api
          model: sensenova-6.7-flash-lite
        - provider: custom:new-api
          model: Qwen/Qwen3.5-35B-A3B-Ornith   # ← 新增：本地模型
        - provider: custom:new-api
          model: agnes-2.0-flash
      aggregator:
        provider: custom:bigmodel
        model: GLM-5

```

多加一个参考模型，看起来只是多了一次 API 调用，怎么就变成了无限循环？

## 根因分析：MoA 如何放大 API 不稳定性

### 单模型 vs MoA 的失败成本

先理解关键区别——单模型和 MoA 在遇到 API 失败时的行为完全不同：

**单模型模式**：

```tap

1
2

用户输入 → 1 次 API 调用 → 成功/失败
  失败 → 重试 1 次 → 成功/失败

```

最坏情况：2 次 API 调用（1 次原始 + 1 次重试）。

**MoA 模式（3 个参考模型）**：

```tap

1
2
3
4
5
6

用户输入 → 4 次 API 调用（3 参考 + 1 聚合）
  参考模型 1 失败 → 重试 → 成功/失败
  参考模型 2 失败 → 重试 → 成功/失败
  参考模型 3 失败 → 重试 → 成功/失败
  聚合器等待所有参考完成 → 超时 → 整体重试
    → 再次触发 4 次调用 → ...

```

最坏情况：每次整体重试触发 4 次调用，N 次重试 = **4N 次 API 调用**。

这就是核心问题：**MoA 把单点的 API 失败放大了 4 倍**。

### 触发条件：本地模型冷启动延迟

Profile B 的第三个参考模型 `Qwen/Qwen3.5-35B-A3B-Ornith` 是通过 llama.cpp 本地部署的，而不是纯云端 API。本地部署带来一个特殊问题——**冷启动延迟**。

llama.cpp 的模型加载策略支持 `--sleep-idle-seconds` 参数：模型空闲一段时间后自动从 GPU 显存卸载以节省资源。下次调用时需要重新加载模型到显存，这个过程可能耗时 10-30 秒甚至更久。

调用链路：

```actionscript

1

Agent → new-api（API 代理层）→ llama.cpp（本地推理）→ GPU

```

当以下条件**同时满足**时，无限循环被触发：

- **模型处于休眠状态**：llama.cpp 的 `--sleep-idle-seconds=3600`（1 小时空闲后休眠），模型已从显存卸载

- **冷启动耗时超过 new-api 超时阈值**：new-api 代理层有请求超时限制（通常 30-60 秒），而模型重新加载到显存可能需要更久

- **Agent 框架的自动重试机制**：MoA 收到超时错误后自动重试整个推理流程

- **重试时模型仍未加载完成**：第二次调用时模型可能仍在加载中，再次超时

于是循环就形成了：

```smali

1
2
3

调用 → 模型加载中 → new-api 超时 → MoA 重试
  → 调用 → 模型还在加载（或刚加载完又被并发请求打断）→ 再次超时
  → MoA 再次重试 → ...

```

### 为什么 Profile A 不受影响？

Profile A 的两个参考模型（sensenova-6.7-flash-lite 和 agnes-2.0-flash）都是纯云端 API：

- **无冷启动**：云端 API 常驻运行，响应延迟稳定在 1-3 秒

- **无代理层超时**：直接走 new-api → 云端 API，中间不经过本地推理引擎

- **MoA 整体耗时可控**：2 个参考模型并行调用，每个 2-3 秒，聚合器再 3-5 秒，总共 8-10 秒完成

Profile B 加了一个本地模型后：

- 本地模型冷启动 30+ 秒 → 超过 new-api 超时

- 聚合器拿不到完整的 3 个参考回答 → MoA 流程失败

- Agent 自动重试 → 再次触发 4 次调用（包括重新尝试本地模型）

- 模型可能已加载但被并发请求打断 → 再次超时

## 解决方案

### 方案一：移除本地模型，改用纯云端参考模型（推荐）

最简单直接——不把本地模型用作 MoA 参考位：

```yaml

1
2
3
4
5
6

# 回归纯云端配置（与 Profile A 一致）
reference_models:
  - provider: custom:new-api
    model: sensenova-6.7-flash-lite
  - provider: custom:new-api
    model: agnes-2.0-flash

```

**优点**：零冷启动风险，延迟稳定。
**代价**：放弃了本地模型的免费推理算力。

### 方案二：调整 llama.cpp 休眠策略

增大 `--sleep-idle-seconds` 或直接关闭休眠：

```bash

1
2
3
4
5

# 方案 2a：增大休眠时间（模型保持加载更久）
./llama-server --model model.gguf --sleep-idle-seconds=86400  # 24小时

# 方案 2b：完全关闭休眠（常驻显存）
./llama-server --model model.gguf --no-sleep-on-idle

```

**优点**：保留本地模型作为参考位，不浪费显存。
**代价**：显存常驻占用，其他模型可用 VRAM 减少。

### 方案三：增大 new-api 代理层超时

在 new-api 的配置中调整请求超时阈值：

```1c

1
2

# new-api 设置 → 渠道设置 → 超时时间
# 默认通常 30-60 秒，改为 120 秒以覆盖冷启动

```

**优点**：不改模型配置，只改网络层。
**代价**：如果冷启动超过 120 秒（大模型可能），仍然会超时。且增大超时会影响所有走该渠道的请求。

### 方案四：用免费云 API 替代本地模型参考位（最优）

这是在方案一基础上的进阶——如果有些平台提供免费的小额 API 额度（如魔搭 ModelScope），可以把这些高智商云模型配置为 MoA 参考位。

MoA 参考模型每次只需约 600 token（由 `reference_max_tokens: 600` 控制），且只在复杂任务时触发 MoA——这正好匹配”小额度 + 高智商”的免费 API 特性。

```yaml

1
2
3
4
5
6
7
8

# 用免费云 API 替代本地模型
reference_models:
  - provider: custom:new-api
    model: sensenova-6.7-flash-lite
  - provider: custom:modelscope    # 免费高智商模型
    model: deepseek-v3
  - provider: custom:new-api
    model: agnes-2.0-flash

```

**优点**：零成本、零冷启动、高智商参考。
**代价**：免费额度有限，但 MoA 参考位的低消耗（600 token/次）完全够用。

## 教训总结

### 1. MoA 不是简单的”N 次并行调用”

MoA 把 N 个参考模型的回答聚合，看起来只是并行 N 次 API 调用。但**任何一个参考模型失败，整个 MoA 流程失败**。失败后的重试会触发完整的 N+1 次调用（N 参考 + 1 聚合）。

**放大系数 = 参考模型数 + 1**。3 个参考模型 → 放大 4 倍。

### 2. 本地模型和云 API 的稳定性不对等

云 API 的响应延迟是一个相对稳定的分布（P99 通常在 10 秒以内）。本地部署的模型则有冷启动这个”悬崖”——空闲时 0 延迟，唤醒时 30+ 秒延迟。

把两种不同稳定性特征的模型混入 MoA 参考池，等于引入了一个不稳定因素。而 MoA 的放大效应会让这个不稳定因素成为系统的瓶颈。

### 3. 代理层超时是隐性杀手

`Agent → new-api → llama.cpp` 三层架构中，每层都有自己的超时机制。new-api 的默认超时（30-60 秒）可能无法覆盖 llama.cpp 的冷启动时间（30+ 秒）。

**排查多层架构的超时问题，必须逐层验证**，而不是只看最终错误信息。

### 4. 对比测试是发现问题的金钥匙

如果只在 Profile B 上测试 MoA，很可能误判为”MoA 不稳定”或”框架有 bug”。正是因为有 Profile A（纯云端）的成功对照，才能快速定位到”本地模型冷启动”这个根因。

**部署多模型架构时，先用最小配置（纯云端）跑通基线，再逐步加入复杂组件（本地模型等），每步对比验证。**

## MoA 调试检查清单

如果你也遇到 MoA 无限循环问题，按以下步骤排查：

- **检查每个参考模型的独立可达性**：绕过 MoA，直接向每个参考模型发送测试请求，确认是否都能在超时阈值内响应

- **检查本地模型冷启动时间**：`curl` 直接调用 llama.cpp API，测量首次响应时间（如果模型在休眠，会看到明显延迟）

- **检查 new-api 代理层超时配置**：确认 new-api 的渠道超时设置是否能覆盖最慢的参考模型

- **检查 Agent 框架的重试策略**：确认 MoA 失败后的重试次数上限和退避策略

- **对比最小配置**：移除所有本地模型，只保留纯云端参考模型，验证是否恢复正常

## 结论

MoA 是一个强大的多模型协同架构，但它的”多模型放大效应”意味着你在获得更全面回答的同时，也承担了更多的失败风险。

**核心法则：MoA 参考池中每个模型的稳定性，决定了整个 MoA 的稳定性。一个不稳定的参考模型，足以拖垮整个推理流程。**

在使用 MoA 时，优先选择稳定性高的云端 API 作为参考模型。如果必须使用本地模型，务必处理好冷启动延迟与代理层超时的关系，或者在非关键路径上单独使用本地模型，而不是把它塞进 MoA 的参考池。