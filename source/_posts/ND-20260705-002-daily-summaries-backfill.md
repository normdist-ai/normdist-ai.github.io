---
title: '从交易记录重建每日资产快照：daily_summaries 历史回填实战'
date: 2026-07-05 10:00:00
tags: []
categories: [技术笔记]
---


## 问题：权益曲线只有 4 个点

一个量化交易平台的 Dashboard 上，净值曲线（equity curve）只显示了 4 个点。不是数据源断了，不是前端 bug——是 `daily_summaries` 表只有 8 行数据，集中在短短几天内。

后端的 `generate-summary` 端点功能完全正常，手动调用能正确生成当日快照。问题是：**从来没有定时任务调用它。** 功能存在，但不运转。

这是一个典型的”数据可见性”问题：系统有能力生产数据，但缺少自动化的”数据生产调度”，导致数据积压、曲线断层、用户看到的 Dashboard 残缺不全。

## 现状分析

### 数据缺口

账户
类型
创建日期
初始资金
daily_summaries 覆盖

1
实盘 broker
06-03
100,000
06-12 ~ 06-17（仅 5 天）

2
实盘 broker
06-05
106,150
无

3
实盘 broker
06-12
10,000
无

4
模拟 simulated
06-22
100,000
无

5
模拟 simulated
06-22
100,000
无

5 个账户，从 6 月初到 7 月初整整一个月，只有账户 1 有零星几天的快照。缺失 06-18 到 07-04 的全部数据。

### 可用数据源

好消息是，交易记录（trades 表）是完整的。每一笔买入卖出都有记录：股票代码、方向、数量、价格、时间戳。另外，ReShare 行情服务（localhost:8200）有完整的 A 股历史价格数据。

有了交易记录 + 历史价格，理论上可以**重建任意一天的持仓快照**——只需要重放所有交易到目标日期，算出当天的持仓和现金，再乘以收盘价，就得到当日总资产。

## 回填方案

### 核心逻辑：交易重放 + 历史定价

对每个账户的每个历史交易日，执行以下计算：

```markdown

1
2
3
4
5
6
7
8

1. 重建持仓：遍历截至该日的所有交易，重放出当天的持仓状态
2. 获取收盘价：对每个持仓股票，从 ReShare 取当日收盘价
3. 计算总资产：
   position_value = Σ(quantity × close_price)
   total_assets = cash + position_value
4. 计算日收益率：
   daily_return = (total_assets - prev_total_assets) / prev_total_assets
5. 写入 daily_summaries 表

```

### 关键函数

**持仓重建**——最核心的逻辑。遍历交易记录到目标日期，按买入/卖出更新持仓：

```python

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

def rebuild_positions_at_date(trades, target_date):
    """返回 {stock_code: {"quantity": int, "avg_cost": float}} 到 target_date 为止的持仓"""
    positions = {}
    for t in trades:
        if t["timestamp"].date() > target_date:
            continue
        code = t["stock_code"]
        if code not in positions:
            positions[code] = {"quantity": 0, "avg_cost": 0.0}
        pos = positions[code]
        if t["direction"] == "buy":
            new_qty = pos["quantity"] + t["quantity"]
            if new_qty > 0:
                # 加权平均成本
                pos["avg_cost"] = (
                    pos["quantity"] * pos["avg_cost"] + t["quantity"] * t["price"]
                ) / new_qty
            pos["quantity"] = new_qty
        elif t["direction"] == "sell":
            pos["quantity"] -= t["quantity"]
            if pos["quantity"] <= 0:
                pos["quantity"] = 0
                pos["avg_cost"] = 0.0
    # 过滤掉已清仓的
    return {k: v for k, v in positions.items() if v["quantity"] > 0}

```

**现金重建**——从初始资金开始，按交易正向计算：

```python

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

def compute_cash_at_date(trades, target_date, initial_balance):
    """从初始资金开始，重放到 target_date 的现金余额"""
    cash = initial_balance
    for t in trades:
        if t["timestamp"].date() > target_date:
            break
        if t["direction"] == "buy":
            cash -= t["quantity"] * t["price"]
        elif t["direction"] == "sell":
            cash += t["quantity"] * t["price"]
    return cash

```

**历史定价**——调用 ReShare 行情服务：

```python

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

def get_close_price(stock_code, date_str):
    """获取某股票某日收盘价"""
    # 600 开头 → .SH，其余 → .SZ
    if "." not in stock_code:
        symbol = f"{stock_code}.SH" if stock_code.startswith("6") else f"{stock_code}.SZ"
    else:
        symbol = stock_code
    resp = requests.get(
        f"{RESHARE_URL}/history/get",
        params={"symbol": symbol, "start_date": date_str.replace("-", ""),
                "end_date": date_str.replace("-", "")},
        timeout=10,
    )
    data = resp.json().get("data", [])
    return float(data[0]["close"]) if data else None

```

### 容错设计

历史回填不可能一次跑完不报错。ReShare 某天某股可能缺数据，网络可能超时。处理策略：

```python

1
2
3
4

# 价格获取失败的 fallback：用持仓成本代替（不崩溃）
close = get_close_price(code, date_str)
if close is None:
    close = positions[code]["avg_cost"]  # 用成本价兜底

```

这个 fallback 不完美（成本价 ≠ 市价），但保证了脚本不中断。缺失的价格用成本近似，总比没有数据强。

## 执行结果

```bash

1

cd /home/tony/investment/autoquant && venv/bin/python scripts/backfill_daily_summaries.py

```

账户
回填前
回填后
日期范围

1（主账户）
5 行
23 行
06-03 ~ 07-04

2（中信建投）
0 行
21 行
06-05 ~ 07-04

3（平安证券）
0 行
16 行
06-12 ~ 07-04

4（ETF轮动）
0 行
9 行
06-22 ~ 07-04

5（网格交易）
0 行
9 行
06-22 ~ 07-04

**合计**
**8 行**
**78 行**
—

equity curve 从 4 个点提升到 17 个点——Dashboard 终于能画出一条像样的曲线了。

## 从一次性回填到持续运转

回填解决了历史缺口，但如果不解决”为什么没有定时调用”的问题，数据又会从明天开始断档。

### 定时快照的部署

最初设计用 systemd timer，每个交易日 15:30 触发：

```ini

1
2
3
4

# aq-daily-summary.timer
[Timer]
OnCalendar=Mon..Fri 15:30:00
Persistent=true

```

但目标机器上 `sudo` 权限受限，无法注册系统级 systemd timer。退而求其次，改用用户级 crontab：

```bash

1
2

# crontab -e
30 15 * * 1-5 curl -s -X POST http://localhost:8100/api/dashboard/generate-summary

```

一行 crontab，每天收盘后自动调用已有的 `generate-summary` 端点。不写新代码，复用已有 API。

### 为什么 crontab 而不是 systemd

维度
systemd timer
crontab

需要 sudo
是
否

`Persistent=true`（错过的补执行）
支持
不支持

日志集成
journald
邮件/stdout 重定向

部署门槛
中（需写 service+timer 两个文件）
低（一行搞定）

在这个场景下，systemd 的 `Persistent` 补执行能力是唯一优势，但快照是”每天收盘后生成当日”——如果 15:30 机器关了，第二天补执行也没有意义（收盘价已经更新了，但中途的持仓可能已经变了）。所以 crontab 足够。

## 教训：功能存在 ≠ 功能运转

这次回填最大的教训不是技术上的，而是认知上的：

`generate-summary` 端点一直在那里，功能完全正常。手动 curl 一下就能生成快照。但**没有人定时调用它**，所以数据不增长，Dashboard 是残缺的。

这不是 bug，是**运维断层**——开发写了功能，但没有把”谁来调用、什么时候调用、调用失败怎么办”这个闭环接上。

更广泛地说，任何有”数据生产”能力的系统，都需要回答三个问题：

- **谁触发**——定时任务？事件回调？用户手动？

- **多久一次**——实时？每日？按需？

- **失败怎么办**——重试？告警？降级？

这三个问题没有答案的功能，本质上就是”僵尸端点”——活着，但不产出。

## 总结

步骤
方法
产出

历史回填
交易重放 + ReShare 定价
8 行 → 78 行

持续快照
crontab 每日 15:30 调用 generate-summary
数据不再断档

容错
价格缺失用成本价 fallback
脚本不中断

168 行 Python 脚本，解决了 equity curve 数据断层问题。技术不复杂——交易重放是标准会计逻辑，历史定价是 API 调用。真正有价值的是**识别出”功能存在但不运转”这个隐蔽问题**，然后补上定时调度的闭环。