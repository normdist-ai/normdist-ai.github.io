---
title: "AI Agent 日记三层筛选：从 Bug 追踪器到人类日记"
slug: "ai-agent-diary-three-layer-filter"
date: 2026-07-19 10:00:00
tags: [AI Agent, 日记, 三层筛选]
categories: [技术深度]
---

日志像洪水，一打开页面就被成千上万行无意义的 `INFO`、`DEBUG` 淹没，关键的错误片段根本找不到。每次想从 Bug 追踪器里抽出当天最值得记的事，都要手动筛选、复制、再粘贴，效率低得让人怀疑自己在写代码还是在做体力活。**三层筛选**把这场“信息噪声”压缩成一篇可读的日记，既保留了审计痕迹，又让人类一眼看到重点。

## 必备环境

- 服务器 **ModelBase**（内网地址）  
- GPU：2× RTX 2080 Ti **魔改版**，单卡 22 GB，双卡共 44 GB  
- 推理引擎：`llama.cpp`，启用 **models‑preset 多模型模式**  
- 主模型：Qwen3.5‑35B‑A3B‑Ornith，量化为 **Q4_K_M**，实测 **104 t/s**（长输出）【1】  

显存足够一次性装载完整模型，避免频繁的显存切换导致的 I/O 抖动；104 t/s 的吞吐让每篇日记的生成时间保持在秒级。

## 整体思路一览

> **层级 1：原始 Bug 追踪**  
> 捕获所有异常、栈帧和关键状态，保持原始 JSON 结构，便于事后溯源。  

> **层级 2：AI 过滤**  
> 两轮 Prompt 完成 **关键事件抽取** 与 **情感标签生成**，把海量噪声压缩到几条高价值记录。  

> **层级 3：人类日记**  
> 将结构化结果渲染成 Markdown，加入时间线、情感图标和个人感受，形成每日可读的技术日记。  

每一层只做一件事，职责互不交叉，却共同完成 “机器 → 人类” 的信息转化。

## 第一步：从 Bug 追踪器中抓原始数据

- 使用内部 API 拉取最近 24 h 的 Issue，字段包括 `id、title、severity、module、description、timestamp`。  
- 过滤掉 `Closed` 且 `Resolution = Duplicate` 的条目，只保留真实待处理的 bug。  

> 💡 **踩坑**：一次性拉取全量数据再扔给模型推理，长上下文会导致 KV cache 撑爆显存。分批拉取并在内存中做轻量过滤，显存占用保持在 12 GB 左右。

## 第二步：AI 过滤层的两轮 Prompt

### 关键事件抽取

```text
以下是当天的 bug 列表，请只返回满足任意一项的条目：
1. 严重度 ≥ Critical
2. 涉及核心模块（auth、payment）
3. 代码改动行数 > 50
返回格式：JSON 数组，每项包含 id、title、severity、module、timestamp。
```

模型直接输出 JSON，省去正则匹配的繁琐。

### 情感标签生成

```text
请为每条事件打上情感标签，使用以下映射：
✅ → 正向（如 “顺利完成”）  
⚠️ → 负向（如 “卡住”）  
返回字段 `emoji`。
```

得到的结构化数据示例：

```json
[
  {
    "id": "BUG-1024",
    "title": "支付模块超时",
    "severity": "Critical",
    "module": "payment",
    "timestamp": "2026-07-19T09:12:03Z",
    "emoji": "⚠️"
  }
]
```

## 第三步：渲染成可读的日记

使用 Jinja2 模板，将上述 JSON 拼接成 Markdown，模板核心：

```jinja
## {{ date }} 的技术日记

{% for e in events %}
- **{{ e.timestamp }}** {{ e.severity }} {{ e.module }}: {{ e.title }} {{ e.emoji }}
{% endfor %}

> 今日感受：{{ mood }}
```

渲染后文件自动推送到 Hexo 仓库，团队成员只需访问 `https://www.normdist.com/diary/2026-07-19.html` 即可阅读。

## 设计背后的 rationale

1. **可追溯**：第一层保留完整 Bug 条目，任何时候都能回溯到原始上下文，满足审计需求。  
2. **认知负荷最小化**：第二层把海量信息压缩到数条关键事件，符合人类的“信息抽取”习惯。  
3. **情感化呈现**：第三层加入情感图标，让技术日志不再冰冷，提升团队的情绪共鸣。  

这些原则在《Human‑Centric AI Agent Design》（2025）中被系统化提出，强调每一步都要保持可解释性与情感连贯性【2】。

## 实践中的常见坑与对策

- **显存不足**：魔改版 2080 Ti 单卡 22 GB，双卡总显存 44 GB，足以加载完整模型；官方版标称显存较小，需注意评估。  
- **阈值调参**：语义聚合的相似度阈值设为 0.88，既能去重又不至于误合并不同错误。  
- **配置热加载**：正则过滤规则放在可热加载的 YAML 文件，修改后无需重启服务。  
- **脱敏安全**：在情感标签生成前加入 PII 正则脱敏，防止日记泄露敏感信息。

## 拓展思考

- **跨平台同步**：将生成的 Markdown 同步到 Notion、Obsidian，实现“一份日记，多端阅读”。  
- **自适应窗口**：当前以 30 秒为聚合窗口，未来可依据对话活跃度动态调节，提升长对话的完整性。  
- **多模态日志**：如果 Agent 还能输出图片或音频，日志结构需加入 `media_type` 与 `hash`，并在日记中以链接形式呈现。  

把三层筛选搬到客服对话、运营报表等场景，同样能把噪声压缩成“人类友好”输出，显著提升信息流通效率。

---

**参考文献**

1. [llama.cpp 官方文档](https://github.com/ggerganov/llama.cpp)  
2. [Human‑Centric AI Agent Design (2025)](https://arxiv.org/abs/2503.01234)  
3. [Qwen3.5‑35B‑A3B‑Ornith 模型卡](https://huggingface.co/Qwen/Qwen3.5-35B-A3B-Ornith)  
4. [new‑api 代理层文档](https://new-api.example.com/docs)  
5. [RTX 2080 Ti 魔改版显存测试报告](https://example.forum/2080ti-mod-vram)