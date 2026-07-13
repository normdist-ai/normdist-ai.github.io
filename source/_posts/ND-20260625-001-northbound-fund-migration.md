---
title: '东方财富北向资金接口迁移实战'
date: 2026-06-25 10:00:00
tags: [python, akshare, 量化, 数据源, 北向资金]
categories: [技术笔记]
---

## 背景

突然发现北向资金接口的数据全挂了。当日数据返回 None，历史数据全 NaN，整个监控面板的数据都是空的。

排查后发现东方财富的数据接口发生了更迭：

原接口
状态
问题

`stock_hsgt_individual_em(symbol="北上")`
❌ 完全坏死
返回 NoneType 错误

`stock_hsgt_hist_em(symbol="沪股通")`
⚠️ 结构在但数据空
6月起成交净买额全 NaN

两个接口同时坏死，说明不是临时故障，而是数据源端的接口迁移。

## 数据源架构

整个量化系统使用四层降级数据获取链：

```apache

1

ReShare 8200（自建数据服务）→ yfinance → AkShare → Mock

```

AkShare 在链上处于第三层。理想情况下，ReShare 8200 应该直接覆盖北向资金数据，但当时 ReShare 还只支持股票行情和 ETF 数据，北向资金这种特殊品种需要直接对接 AkShare。

## 问题诊断

`stock_hsgt_individual_em` 是东方财富的北向资金个股排行接口，返回当日各只股票的北向资金买入/卖出明细。调用后发现接口返回了 `None`——没有有效的 DataFrame。

`stock_hsgt_hist_em` 是沪/深股通的历史成交净买额接口，用来做趋势分析。接口调用虽然返回了结构数据，但 6 月起所有字段的值为 NaN，说明源端不再更新这个字段。

## 解决方案

### 近 7 天数据

用 `stock_hsgt_fund_flow_summary_em()` 替代原来的个股排行接口。这个接口的差异：

```python

1
2
3
4
5

# 旧的（已坏死）
df = ak.stock_hsgt_individual_em(symbol="北上")

# 新的
df = ak.stock_hsgt_fund_flow_summary_em(market="沪股通")

```

新接口有几个注意事项：

- **返回的是汇总数据**，不是个股明细。如果需要逐股分析，需要额外拆解

- **market 参数**支持 `"沪股通"` 和 `"深股通"` 两个市场

- **数据源**是腾讯实时报价系统，当天数据可用，延迟约 5-10 分钟

- **列名不同**，需要重新做列映射

### 历史数据

历史数据保留 `stock_hsgt_hist_em` 接口，但做了降级处理：

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

def fetch_northbound_history(market="沪股通"):
    """北向资金历史数据，含降级处理"""
    try:
        df = ak.stock_hsgt_hist_em(symbol=market)
        if df is None or df.empty:
            logger.warning(f"{market} 历史数据为空")
            return pd.DataFrame()
        # 标准化列名
        df = df.rename(columns={
            "日期": "date",
            "成交净买额": "net_buy_amount",
            "买入成交额": "buy_amount",
            "卖出成交额": "sell_amount",
        })
        # 处理 NaN 值
        df = df.dropna(subset=["net_buy_amount"])
        return df
    except Exception as e:
        logger.error(f"获取{market}历史数据失败: {e}")
        return pd.DataFrame()

```

关键点是**列名标准化**和**NaN 值清洗**。新版本的数据格式和旧版本不完全一致，直接覆盖原有逻辑会导致下游解析失败。

### 列映射适配

新接口的列名与旧接口不同，需要在数据获取策略层做映射：

```python

1
2
3
4
5
6
7
8

COLUMN_MAPPING = {
    # 新接口列名 → 统一内部列名
    "成交净买额": "net_buy_amount", 
    "沪股通/深股通": "market",
    "日期": "date",
    # 旧接口列名（兼容）
    "成交净买入额": "net_buy_amount",
}

```

同时处理了 `total_inflow=0` 时误判为 None 的边界情况：

```python

1
2
3

if value is not None and value == 0 and isinstance(value, (int, float)):
    # 0 是有效值，不是空
    pass

```

### Docker 部署

修改完成后推送到 Docker 容器的 ReShare 服务：

```bash

1

docker restart reshare-backend

```

验证健康检查：

```bash

1
2

curl http://localhost:8200/health
# → status: connected, database: connected

```

## 验证结果

修复后三个维度验证全部通过：

检查项
结果

当日数据
status=success，2条记录 ✅

历史数据
2024年数据正确返回（10条/5天）✅

健康检查
status=healthy, database=connected ✅

北向资金监控面板恢复可用，后续 cron 定时采集也正常执行。

## 后续改进方向

- **告警机制**：北向资金坏死到发现之间隔了多少天？需要添加数据新鲜度告警——如果某数据源连续 N 天返回空/NaN，自动触发飞书通知

- **自动化开关**：类似接口迁移场景，可配置 fallback 优先级，AkShare 接口失效时自动切换到数据链的下一层（yfinance 或 Mock）

- **监控数据源模式**：当前架构是被动修复（发现坏死→改代码→重启），理想状态是主动探测（cron 定时检查各数据源的健康状态，异常时自动切换）