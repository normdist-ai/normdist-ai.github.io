---
title: '一个 60 倍的接口提速：两层隐藏 Bug 如何让 API 延迟从 10 秒降到 0.16 秒'
date: 2026-07-03 10:00:00
tags: [性能优化, API 优化, N+1 查询, Bug 排查, 量化交易, 接口提速]
categories: [技术笔记]
---

持仓接口 `/api/positions` 冷查询要 10 秒，热缓存 3 秒。目标是 < 1 秒。

修复后冷查询 0.16 秒，热缓存 0.035 秒，提升 60 倍。

根因不是 N+1 查询，不是数据库缺索引，而是两层 bug 叠加。

## 现象

一个量化记账平台，后端 FastAPI + SQLAlchemy + SQLite，数据源是自建的行情服务（ReShare）。持仓列表端点 `/api/positions` 会拉实时行情算市值、盈亏、今日涨跌——典型的”数据库读持仓 + 逐个拉行情”模式。

6 只持仓，接口耗时 10 秒。用浏览器开发工具看到请求挂起在那不动。

## 第一层发现：慢在 get_prev_close

用排除法定位。把持仓接口的逐项逻辑注释掉，发现拿实时报价那块（`prefetch_prices` → `get_latest_price`）还好，真正卡的是 `get_prev_close`——它负责算今日涨跌额和涨跌幅。

每只持仓调用一次 `get_prev_close`，6 只串行跑下来就 10 秒。一只约 1.5-3 秒。

`get_prev_close` 的实现是这样的（简化后）：

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

def get_prev_close(self, symbol: str) -> Optional[dict]:
    """获取前一交易日收盘价和涨跌幅"""
    hist = self.get_stock_history(symbol, days=10)   # ← 走 /history/get
    if hist is not None and len(hist) >= 2:
        last_close = hist.iloc[-1]["close"]
        prev_close = hist.iloc[-2]["close"]
        change_pct = (
            ((last_close - prev_close) / prev_close) * 100 if prev_close > 0 else 0
        )
        return {
            "code": symbol,
            "prev_close": round(last_close, 2),
            "change_pct": round(change_pct, 2),
        }
    return None

```

它在调 ReShare 的 `/api/v1/history/get` 端点拉历史 K 线。直接对行情服务做基准测试：

端点
耗时

`/api/v1/history/get?symbol=161226.SZ`（LOF 基金）
**30 秒**（超时挂起）

`/api/v1/quote/latest?symbol=161226.SZ`
**0.045 秒**

`quote/latest` 是实时报价端点，已经直接返回了 `change_percent` 和 `prev_close`，根本不需要拉历史 K 线。

那为什么不一开始就用快的端点？

## 第二层发现：为什么 quote 路径”看起来失败”了

这是真正关键的发现。`get_prev_close` 其实有两条路径：先试 quote，失败了再 fallback 到 history。但 quote 路径每次都”失败”——不是行情服务的问题，是代码处理的问题。

ReShare 对某些品种（比如 LOF/ETF）的 `quote/latest` 返回 JSON 里，`turnover` 字段是 `null`：

```json

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

{
    "symbol": "161226.SZ",
    "price": 1.725,
    "change_percent": 0.0,
    "volume": 122537288,
    "turnover": null,
    "high": 1.729,
    "low": 1.67,
    "prev_close": 1.725
}

```

解析这段 JSON 的代码：

```python

1

"amount": float(body.get("turnover", 0)),

```

看起来用了 `get("turnover", 0)` 做兜底，应该很安全。但这里有个 Python 经典坑：

> 

**`dict.get(key, default)` 仅在 key 不存在时返回 default。如果 key 存在但值为 `None`，返回的是 `None`，不是 default。**

`"turnover": null` → `body.get("turnover", 0)` → 返回 `None` → `float(None)` → 抛 `TypeError` → 被外层的 `except Exception: return None` 吞掉 → `get_quote_latest` 返回 `None`。

于是链路变成：

```pgsql

1
2
3

get_prev_close → 试 quote → quote 内部 TypeError → 返回 None
              → fallback 到 history → /history/get 对 LOF 挂起 3 秒
              → 每只持仓串行 × 3 秒 = 10 秒

```

两层 bug 叠加。单独修任何一层都做不到 60 倍：

- 只修 quote 路径优先级：history 仍然是慢的 fallback，一旦 quote 失败还是 3 秒

- 只修 None 解析：get_prev_close 还是优先调 history，慢路径仍然占主导

## 修复

两件事一起改。

### 修复一：`_safe_float` 处理 None / NaN

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

def _safe_float(val, default=0.0):
    """安全转 float，None/NaN 返回 default"""
    try:
        if val is None:
            return default
        f = float(val)
        return default if (f != f) else f   # NaN check: NaN != NaN
    except (TypeError, ValueError):
        return default

```

`f != f` 是个很老的技巧：IEEE 754 里只有 `NaN` 不等于自身，所以 `f != f` 为 True 就说明是 NaN。

`get_quote_latest` 里所有 `float(body.get(xxx, 0))` 换成 `_safe_float(body.get(xxx))`。不仅 `turnover`，`volume`、`open`、`high`、`low` 都做同样处理，避免任何一个字段为 null 就把整个报价拖垮。

### 修复二：get_prev_close 优先级反转

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
26
27
28
29
30
31
32

def get_prev_close(self, symbol: str) -> Optional[dict]:
    """优先用快速 quote，超时或失败时 fallback 到 history"""
    # ① 快路径：quote/latest（~50ms）
    try:
        quote = self.reshare2.get_quote_latest(symbol)
        if quote and quote.get("price"):
            change_pct = float(quote.get("change_pct", 0))
            return {
                "code": symbol,
                "prev_close": round(float(quote["price"]), 2),
                "change_pct": round(change_pct, 2),
            }
    except Exception:
        pass

    # ② 慢路径：history fallback
    try:
        hist = self.reshare2.get_history(symbol, days=5)
        if hist is not None and len(hist) >= 2:
            last_close = float(hist.iloc[-1]["close"])
            prev_close = float(hist.iloc[-2]["close"])
            change_pct = (
                ((last_close - prev_close) / prev_close) * 100 if prev_close > 0 else 0
            )
            return {
                "code": symbol,
                "prev_close": round(last_close, 2),
                "change_pct": round(change_pct, 2),
            }
    except Exception as e:
        print(f"get_prev_close history fallback failed for {symbol}: {e}")
    return None

```

这里有个语义陷阱值得提一句。`/quote/latest` 的 `prev_close` 字段确实是”昨日收盘价”，但消费方 `api.py` 里把返回值的 `prev_close` key 当作”今日收盘价”在算涨跌额：

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

# api.py 消费逻辑（不修改）
prev = data_fetcher.get_prev_close(pos.stock_code)
if prev:
    _pct = float(prev.get("change_pct", 0))
    _close = float(prev.get("prev_close", 0))  # 实际当今日收盘价用
    today_close = _close
    if today_close > 0 and day_change_pct != 0:
        yest_close = today_close / (1 + day_change_pct / 100)
        day_change = round(today_close - yest_close, 2)

```

为了不破坏消费方，`get_prev_close` 走 quote 路径时返回的 `prev_close` key 里填的是 `quote["price"]`（今日现价），而不是 quote 字段本身的 `prev_close`（昨日收盘价）。这样消费方的反推公式仍然成立：`今日收盘 / (1+涨幅) = 昨日收盘`。

变量名误导是真实存在的，但改名要波及多个调用方，留着命名、对齐语义是当下最稳的选择。

## 验证

```bash

1
2

# 冷查询
time curl -s http://localhost:8100/api/positions?account_id=2

```

指标
修复前
修复后

冷缓存
10.0s
0.16s

热缓存
3.1s
0.035s

161226.SZ quote_latest
返回 None
返回完整 dict

回归全套端点：portfolio、dashboard/summary、trades、accounts、signals、stocks、health/consistency 全部正常。

改动只有 33 行。

## 几个复盘点

**1. “兜底参数”的陷阱。** `dict.get(key, default)` 这个写法看着无害，实际只在 key 缺失时才生效。后端拿第三方 JSON，字段存在但为 null 是常态，必须显式处理 None。一行业务代码的隐患，被 try/except 静默吃掉后，能拖垮一个接口的性能。

**2. silent except 是性能 bug 的温床。** `except Exception: return None` 把所有异常归一成”业务可接受的 None”。但上层不区分”数据源真的没数据”和”我自己的代码崩了”，于是 fallback 链被错误地触发。后续接外部数据源，异常处理至少应该分级：网络超时、状态码异常、解析错误、空值——不同错误不同的处理策略。

**3. 接口慢先查外部调用的端点选择。** 数据服务通常提供多种粒度的接口，”全量历史”和”最新一条”性能差几个数量级。一个接口慢，先单独 benchmark 它依赖的下游端点，很快能定位是数据源本身慢还是调用方式不对。

**4. 串行调用是放大器。** 6 只持仓，每只慢 3 秒，串行就 18 秒。即便修不好单次调用，改成并发拉行情（`ThreadPoolExecutor`）也能把延迟从”延迟 × N”降为”延迟 × 1”。这次的修复让单次调用从 3 秒降到 0.05 秒，串行也能接受了；但并发化是下一阶段的优化项。

**5. 改最少的代码，覆盖最多的场景。** 整个修复只动 `data_fetcher.py` 一个文件、两个方法，API 层和前端零改动。不是每个性能问题都需要架构重构——找到真正的瓶颈点，最小化改动面。

---

**附：根因链路全图**

```pgsql

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

ReShare /quote/latest 返回 "turnover": null
  ↓
get_quote_latest() 中 float(body.get("turnover", 0))
  ↓ (body.get 返回 None 而非 0)
float(None) → TypeError
  ↓
except Exception: return None  ← 静默吞掉
  ↓
get_prev_close() 的 quote 路径拿到 None
  ↓
fallback 到 history 路径
  ↓
/history/get 对 LOF/ETF 挂起 3 秒
  ↓
6 只持仓串行 × 3 秒 = 10 秒总延迟

```