---
title: 博客助手自我进化：技能解耦与三层回退链设计
date: 2026-07-19 23:58:32
tags: [技能开发, 架构设计, MOA, 回退链, blog-helper]
---

重命名了一下 `moa-plus` 技能，`blog-helper` 直接罢工——`init.sh` 报错说找不到路径，`draft.py` 和 `review.py` 也跟着挂。代码里埋了一个硬编码的定时炸弹。

## 问题：技能之间不该拉拉扯扯

`blog-helper` 是博客创作与发布流水线技能，需要调用 MOA 类深度思考能力来审稿和写稿。但改造前，代码里到处都是对 `moa-plus` 路径的硬编码依赖：

- `init.sh` 检查 `~/.hermes/skills/devops/moa-plus/scripts/moa_plus.py` 是否存在
- `draft.py` 和 `review.py` 里直接写死 `Path.home() / ".hermes" / "skills" / "devops" / "moa-plus" / "scripts" / "moa_plus.py"`
- `SKILL.md` 的标题写着"写稿+审核：超级大脑 (moa-plus)"

这种设计违反 Hermes Agent 的核心原则：**每个技能都应该是相对独立的，串联它们的是代理，而不是技能内部的硬编码依赖**。一旦 `moa-plus` 迁移路径、重命名或升级结构，`blog-helper` 就会直接挂掉，没有回旋余地。

另外还混淆了两个概念。MOA（Mixture of Agents，混合代理）是多 LLM 代理协作；MoE（Mixture of Experts，混合专家）是单模型内部稀疏激活。**blog-helper 调用的是 MOA，不是 MoE**。概念混淆间接导致了硬编码思维——把它当成一个固定"组件"而非"可替换技能"。

## 解决方案：三层回退链 + 技能独立性

一句话点醒设计者："`init.sh` 为什么要去找 `moa-plus` 的路径？它不应该。MOA 调用应该是可选项：检测到就调，没有要回退，不能卡死。"

改造分三步走。

### 第一步：描述层解耦

`SKILL.md` 不再具名 `moa-plus`，改为描述"调用 MOA 类深度思考能力"。代理在编排时根据实际可用技能灵活匹配，而不是绑死一个特定技能名。

同时，把审稿和校对的标准固化成两份独立的指南文件：

- `references/review-guide.md`：4 类问题（事实/逻辑/计算/无来源）+ 3 级严重度
- `references/proofreading-guide.md`：6 类检查项（AI 味/隐私/格式/front matter/命名/参考文献）

模型在没有 MOA 的情况下也能按指南独立工作——相当于把"大脑"的决策逻辑外置到文档中，而不是依赖外部技能。

### 第二步：代码层解耦

脚本运行时通过 `_find_moa_script()` 函数动态检测 MOA 路径，不再硬编码。检测逻辑有三层，优先级递减：

1. **用户显式配置**：`config.yaml` 的 `llm.thinking_script` 字段，最灵活
2. **默认搜索路径**：`~/.hermes/skills/*/moa-plus/scripts/moa_plus.py`，兼容旧位置
3. **模糊匹配兜底**：`*/*moa*/scripts/*.py`，并检查文件内容是否包含 `"generate"` 函数签名，确保找到的是真正的 MOA 脚本而非同名文件

即使技能迁移或改名，只要符合基本模式就能自动发现。

### 第三步：三层回退链设计

这是整个改造的骨架，保证助手在任何情况下都能运行而不是崩溃：

```text
主路径：MOA 类思考技能（运行时检测到 → 调用）
   ↓ 如果不可用或调用失败
回退 1：单模型直调（按审稿/校对指南独立工作）
   ↓ 如果配置缺失或调用失败
兜底  ：输出 prompt 到文件，提示代理按指南人工处理
```

每个层级都独立且可降级，不依赖上一层。具体实现上，`draft.py` 和 `review.py` 各自封装三个函数（主路径/单模型/兜底），按策略顺序调用。单模型回退时读取 `config.yaml` 中的 `fallback_base_url`/`fallback_model` 配置；兜底时生成 `.txt` prompt 文件，内容包含当前任务、指南引用和指令，控制台打印提示让代理人工编排。

兜底层输出的是结构化 prompt 文件，不是简单的错误日志。代理拿到文件后可以直接喂给任何可用的模型，不影响后续流程。

## 实测验证：三层都跑通了

改完后在临时 HOME 全新环境分别测试每条路径。

### draft.py 三层回退

| 层级 | 结果 |
|------|------|
| 主路径 MOA | 正常生成草稿 3573 字符 |
| 单模型回退（故意指向不存在的 MOA 路径） | `deepseek-v4-flash` 生成 1691 字符草稿，逻辑完整 |
| 兜底（删除 fallback 配置） | 输出 `writing-prompt-*.txt` 文件，包含全部上下文和指令 |

### review.py 三层回退

| 层级 | 结果 |
|------|------|
| 主路径 MOA | 正常审稿 |
| 单模型回退 | `deepseek-v4-flash` 按审稿指南审稿，返回 0 问题 |
| 兜底 | 仅输出预检规则结果和指引，提示代理按 `review-guide.md` 人工审稿 |

### init.sh 实测

- 临时 HOME 全新环境下通过，不再检查 `moa-plus` 路径
- config 模板新增 `writing_mode` 和 `review_mode` 字段

## 改动清单

| 文件 | 改动 |
|------|------|
| `SKILL.md` | 标题去 `moa-plus` 具名；新增代理调用指引；references 索引加两份指南 |
| `references/review-guide.md` | 新建审稿指南（4 类问题 + 3 级严重度） |
| `references/proofreading-guide.md` | 新建校对指南（6 类检查项） |
| `scripts/init.sh` | 删 `moa-plus` 路径检查；config 模板新增 `writing_mode`/`review_mode` |
| `scripts/draft.py` | 硬编码路径 → `_find_moa_script()`；单函数 → 三层回退链 |
| `scripts/review.py` | 同上 |

## 韧性设计的本质

这次改造重新理解了"技能独立性"——**技能是乐高积木，不是拼图**。每块积木不依赖其他积木的形状，代理负责把它们拼成想要的样子。而不是像拼图那样，每块只能配特定的邻居。

三层回退链的设计哲学是：**永远假设外部依赖不可靠**。MOA 可能升级、迁移、甚至被移除，但 `blog-helper` 不能因此崩溃。每一层都是独立的"逃生通道"，越往下越简单、越可靠。

审稿指南和校对指南的引入，本质上是**把专家知识固化到文档层级**，让模型在没有 MOA 时也能按规则工作——类似"知识蒸馏"，把外部技能的逻辑写进 prompt，让模型自己推理。

下一步打算把回退链的配置抽象成通用的 YAML 声明，让每个技能都能声明自己的回退策略，而不用在代码里写死 fallback 逻辑。这或许会成为 Hermes Agent 技能系统的一个通用模式。

## 参考文献

1. Hermes Agent 技能系统设计原则
2. blog-helper 技能源码（本次改造）
3. 本次开发会话中的口头指导：技能独立性、MOA vs MoE 概念区分、回退链设计（2026-07-19）
