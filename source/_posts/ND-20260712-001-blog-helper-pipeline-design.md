---
title: '从零搭建 AI Agent 博客流水线：blog-helper 6 阶段设计实录'
date: 2026-07-12 12:30:00
tags: [AI Agent, Hermes, 博客, 自动化]
categories: [技术笔记]
---

你有没有过这样的经历：一篇技术文章写完，发布后才发现某个数字写错了。我上周就踩了这个坑——双卡总显存写成了单卡的数字，读者评论里指出来，尴尬得不行。

手动写博客的问题在于，写稿、审核、校对全压在一个人身上，注意力是稀缺资源，长文基本不可能不出错。于是我用 Hermes Agent 搭了一个叫 `blog-helper` 的技能——6 阶段流水线，多模型协作审稿，事实库防错。这篇文章记录整个设计过程和取舍。

## 前置条件

如果你想复现这个方案，需要：

- **Hermes Agent**（`hermes-agent` 或 `hermes-agent` 框架）

- **一个可用的 LLM API**（DeepSeek / OpenRouter / 本地模型都行）

- **Hexo 博客**（或任何静态站点生成器）

- **SearXNG 或类似搜索引擎**（用于调研阶段）

不需要额外的服务、数据库或外部依赖。所有脚本都是独立的 Python 文件，每个都能单独运行。

## 做了什么

`blog-helper` 是一个 6 阶段的博客创作流水线，从选题到多平台发布，每个阶段对应一个独立脚本：

- **选题管理**：FIFO 选题池，手动维护选题，AI 管理队列

- **定向调研**：自动判定文章类型（6 种模板），按模板章节生成调研清单

- **编写**：注入事实库 + 模板规范，调用 LLM 生成初稿

- **审核**：MoA 多模型独立审稿，多数投票定级

- **校对**：正则预检 + AI 格式检查

- **发布**：调度器 + 平台适配器，未来支持知乎、公众号等

核心思路：**把创作拆成阶段，让 AI 在关键节点做检查员，人做最终决策者。**

## 技术选型与架构

市面上有不少 AI 内容管道方案。我调研了几个主流思路：

方案
特点
我的取舍

**n8n 工作流**
可视化编排，多 Agent 协作
太重，不适合技术博客这种轻量场景

**CrewAI / AutoGen**
多 Agent 框架，角色分工
适合复杂任务编排，但博客流程不需要这么重的框架

**LangGraph**
状态机驱动，擅长循环
学习成本高，博客流程是线性管道

**blog-helper（本方案）**
独立脚本 + 文件流转
简单、可调试、每个脚本可独立运行

**为什么选独立脚本而不是框架？**

- **可调试**：每个脚本 `--help` 后单独运行，出问题定位快

- **可替换**：审稿模型从 DeepSeek 换成 GLM，只改一个配置

- **可组合**：审核发现问题 → 修改 → 重新审核，自然形成修订循环

- **零外部依赖**：Python 3 + 标准库 + requests，不需要装额外框架

架构上，draft 输出到 `archive/`（中转站），publish 从 archive 读取。这个设计让”写”和”发”解耦——写稿时不发布，发布时从 archive 选文章。

```text

1
2

选题 → 调研 → 编写 → 审核 → 校对 → 发布
                        ↕ 修订循环

```

## 效果

流水线跑出来的实际效果：

**错误拦截**：以我第一篇文章为例，review.py 抓到了 3 个 MUST_FIX：

- 预检引擎拦截了把单卡显存错误写成”官方默认值”的问题——魔改版是 22 GB，事实库规则直接拦截

- DeepSeek-V4-Flash 发现了开头数据与后文的逻辑矛盾

- 审稿模型发现代码块缺少语言标注

**效率提升**：以前一篇技术文章从写稿到发布大概 2-3 小时，现在：

阶段
耗时
说明

选题+调研
5 分钟
FIFO 选题 + 自动调研清单

编写
2 分钟
LLM 生成初稿

审核+校对
3 分钟
多模型审稿 + 正则检查

修订
视情况
通常 1-2 轮

发布
1 分钟
自动 hexo generate + deploy

**总计约 15-20 分钟**（含修订），比手动写快 5-10 倍。

## 踩过的坑

### 坑一：LLM 输出空响应导致崩溃

draft.py 调用 LLM 后，如果模型返回 `choices: null`（DeepSeek API 偶尔出现），脚本直接崩溃。

**解法**：加空响应保护，检测到 null 时提示用户手动补充核心字段：

```python

1
2
3
4
5

choices = response.get("choices")
if not choices:
    print("⚠️ 模型返回空响应，请手动补充 core 字段", file=sys.stderr)
    core = input("输入技术核心描述: ")
    topic_data["core"] = core

```

### 坑二：LLM 从 prompt 里”跑题”

`from_topic` 模式下，如果 `topic_data` 缺少 `angle`/`type`/`outline`/`key_points` 字段，模型会忽略选题，写其他内容。

**解法**：`from_topic` 分支会补齐缺失字段，`core` → `angle` 映射。强制锚定选题段落放在 prompt 最前面。

### 坑三：DeepSeek-V4-Pro 转发异常

new-api 转发 V4-Pro 时出现异常，导致长输出被截断为 `null choices`。

**解法**：临时切到 V4-Flash。这个问题需要排查 new-api 配置，后续跟进。

### 坑四：旧技能和新技能并存

最初在旧 `hexo-blog` 技能上改，后来决定从零创建 `blog-helper`。旧技能保留待验证后删除。

**教训**：改技能前先备份，确认新方案稳定后再清理旧方案。

## 下一步

当前还不够完善的地方：

- **标题优化**：draft.py 生成的标题偏平淡，计划加 `topic-title.py` 独立脚本

- **DeepSeek-V4-Pro 不可用**：需排查 new-api 配置

- **example 学习参照**：已建目录，待从知乎等平台收集高赞文章作为风格参照

- **多平台发布**：Hexo 已通，知乎、公众号适配器待开发

---

如果你也在维护技术博客，不一定非要搭完整流水线。哪怕只加一个事实库预检，就能省掉很多尴尬的修改时间。

---

**参考文献**

- [Grammarly — How to Write a Blog Post](https://www.grammarly.com/blog/content-marketing/how-to-write-a-blog-post/)

- [Kontent.ai — Content Production Process](https://kontent.ai/blog/content-production-process)

- [Hermes Agent 官方文档](https://hermes-agent.nousresearch.com/docs)

- [Mixture of Agents 论文](https://arxiv.org/abs/2406.04692)

- [Multi-Agent AI Content Pipeline (Valentin Chmara)](https://valentinchmara.com/blog/building-an-ai-content-team-automating-my-content-creation-with-multi-agent-systems)

- [Multi-Agent SEO Blog Writing (n8n)](https://n8n.io/workflows/8654-multi-agent-seo-optimized-blog-writing-system-with-hyperlinks-for-e-commmerce/)

- 本文所有脚本源码：`~/.hermes/skills/content/blog-helper/scripts/`