---
title: "非定时任务衰减律：'记得做'等于'不会做'"
date: 2026-07-15 10:00:00
tags: [AI Agent, 踩坑, 任务管理, 高可用]
categories: [技术笔记]
---

凌晨三点，运维群里有人喊了一声：“完了。”我们的 AI Agent 没有在半小时后补发健康检查——工程师 A 记得要改那个心跳偏差修复逻辑，觉得发行版更新完再搞也不迟。结果两周后，Agent 把-1 发给了生产数据库。

**“记得做”等于“不会做”**——这是我们掏了真金白银买来的教训。

AI Agent 系统里，最优先级的任务从来不是定时任务，而是那些“回头处理一下”的事情。而恰恰是这些不定时任务，会以一种极其隐蔽的方式，自动消失。我管这叫 **非定时任务衰减律**：任务被记住的那一刻，就开始衰减。

---

## 前置条件

本文场景发生在我们的 AI Agent 生产系统上，环境如下：

- Agent 框架：自研 + 开源组件（Hermes Agent Gateway）
- 后端：Python 3.10 + FastAPI
- 数据库：PostgreSQL 15 + Redis 7
- 硬件：ModelBase 服务器，双卡 RTX 2080 Ti 魔改版（单卡 22 GB，总显存 44 GB）
- 推理引擎：llama.cpp，加载 Qwen 系列模型

> 硬件是老伙计了：RTX 2080 Ti 是魔改版，单卡显存 22 GB，双卡共 44 GB。Turing 架构不支持原生 FP8/BF16，均靠模拟实现。

---

## 最终方案：建立持续跟踪机制

先看我们怎么把这个问题堵上的。

核心思路是：**把“记得做”硬编码进状态机，不依赖人的记忆，也不依赖 Agent 的短期上下文。**

### 操作步骤

#### 1. 任务分级，明确定义“衰减风险任务”

把任务按触发方式分成三类：

| 类型 | 触发条件 | 衰减风险 |
|------|----------|----------|
| 定时任务 | 固定 Cron/Crontab | 低 |
| 事件驱动任务 | 实时事件触发 | 中 |
| **非定时任务** | 人为判断“回头做” | **高** |

非定时任务必须单独进入一个**硬跟踪表**。

#### 2. 创建持续跟踪表

在 PostgreSQL 里劈一张专门的表：

```sql
CREATE TABLE pending_tasks (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  task_name TEXT NOT NULL,
  description TEXT,
  created_by TEXT,
  deadline TIMESTAMPTZ,
  next_reminder_at TIMESTAMPTZ DEFAULT NOW() + INTERVAL '30 minutes',
  status TEXT DEFAULT 'pending',
  reminder_interval INTERVAL DEFAULT '30 minutes',
  max_reminder_count INTEGER DEFAULT 10,
  reminder_count INTEGER DEFAULT 0
);
```

重点不是建表，是 **每 30 分钟提醒一次** 这个逻辑。这不是人性话术，是系统逻辑。

#### 3. 专有一个提醒 Worker

启用一个后台 worker，轮询 `pending_tasks` 表：

```python
# reminder_worker.py 核心逻辑（简化）
import asyncio
import asyncpg
from hermes_agent import notify_agent

async def reminder_loop():
    conn = await asyncpg.connect(DATABASE_URL)
    while True:
        rows = await conn.fetch(
            "SELECT * FROM pending_tasks WHERE status = 'pending' AND next_reminder_at < NOW()"
        )
        for row in rows:
            # 发送提醒给对应 Agent 或工程师
            await notify_agent(row['task_name'], row['description'])
            # 更新提醒计次
            await conn.execute(
                "UPDATE pending_tasks SET next_reminder_at = NOW() + reminder_interval, reminder_count = reminder_count + 1 WHERE id = $1",
                row['id']
            )
            if row['reminder_count'] >= row['max_reminder_count']:
                await conn.execute(
                    "UPDATE pending_tasks SET status = 'escalated' WHERE id = $1",
                    row['id']
                )
        await asyncio.sleep(60)
```

#### 4. Agent 内部也注册一个“遗忘检查”

在 Agent 的顶层决策循环里加一条规则——如果某项任务存在超过 6 小时且未进入下一步，自动报给人类监督员：

```python
# Agent 决策层
if task.age_hours > 6 and task.status == 'pending':
    self.escalate_to_human(task)
```

### 验证

用了一个周末去验证效果。注入 30 个“假装非定时任务”，跟踪一周：

- **无跟踪机制**：27 个被遗忘（90% 衰减率）
- **有跟踪机制**：29 个被正常处理（3% 衰减率）

实际跑了一周生产，只漏了一个——原因是那条任务创建时 deadline 设到了下个月。

---

## 踩坑记录

### 坑一：任务复核——“回头确认”就再也回不了头

**现象**：Agent 在处理完定时任务后，收到了一个“顺便帮我看一下日志里这条异常”的请求。它把异常记录到了上下文，标记为“待复核”。然后它切换到了另一个用户对话。等到切换回来，那个上下文已经丢了，日志异常自动沉没了。

**报错**：无。一切看起来正常，只是任务消失了。

**根因**：Agent 的上下文窗口受限于 Token 数量和历史长度。当新对话进来，旧对话的“待复核”标记被清理了。这本质上是 **短期记忆的必然衰减**。

**解法**：把“待复核”这类非定时任务，也插入到 `pending_tasks` 表中。Agent 每次决策前先查这张表。这样任务不只活在上下文里，也活在数据库里。

### 坑二：日志排查——“我记得刚才看了日志”

**现象**：有一次调试心跳偏差，工程师 A 在 Agent 的 Web 界面上看到了告警，口头说“我回头修这个”。他确实也把命令记在了笔记里。一周后，告警再次弹出。他懵了——明明记得修过。

**根因**：工程师 A 的“修过”是指“看过日志，确认了根因”。但在他的工作流中，“看了日志”和“修复代码”之间，隔着两顿饭、一次代码 review、一次非计划会议。两顿饭之后的记忆，比两行代码更不可靠。

**解法**：哪怕只看了一眼日志，也要在系统里创建一个 `pending_task` ——“调心跳偏差修复，状态：_尝试中_”。这比任何笔记都靠谱。

---

## 为什么“记得做”等于“不会做”

这个规律背后有一个简单的原理：**定时任务有外部触发器（时钟），非定时任务只有内部记忆（人脑或 Agent 上下文）。而内部记忆是单向衰减的。**

你在某个时刻决定“回头做”——那一刻，你实际上在等另一个外部事件来触达它（比如“修完这个 feature 后”、“发行版上线后”）。但外部事件不会识别“修完后该做 A”这个关联，除非你把关联写进系统。

回到我们的 Agent 系统。Agent 没有“我明天记得调那个参数”的能力——它只有上下文。当你让它“记一下那个参数 alpha 要调成 0.8”，它会在当前会话里记住。一旦你换了一个窗口，或者重启了服务，那个参数就消失在 Token 序列的阴影里了。

**不是 Agent 不靠谱，是人觉得 Agent 会记住一切——但 Agent 只记住了你最后说的三句话。**

所以，解决方案不是让 Agent 记住更多，而是让系统自己追踪所有的“非定时任务”。就像你给陌生人发了一条消息，他不是“记得回复”，而是你设了一个闹钟“10分钟后催一次”。

---

让系统记住比让人记住更可靠。让时钟工作，而不是让记忆工作。

**参考文献**

1. [ND-20260627-001: “记得做”等于“没做”——AI Agent自主任务执行通道的断裂与修复](https://www.normdist.com) — 本文的姊妹篇，详细拆解 Agent 执行通道的断裂案例。
2. [ND-20260520-001: Hermes Agent Gateway 多实例配置实践](https://www.normdist.com) — 本文的 Agent 网关配置参考。
3. [ND-20260312-001: AI Agent 协作：技术实现与应用](https://www.normdist.com) — 本文的 Agent 协作背景参考。
4. PostgreSQL 文档：`CREATE TABLE` with default values and intervals. — 数据库表结构参考。
5. Hermes Agent 项目文档（内部）：Agent 决策循环与上下文管理。 — Agent 上下文衰减机制的技术背景。