---
title: 'Karpathy 四原则驱动代码重构：3 文件 1 小时零 regression'
date: 2026-07-04 10:00:00
tags: [代码重构, Karpathy, 重构原则, 量化交易, 最佳实践, 可读性]
categories: [技术笔记]
---


## 背景

量化交易系统的数据源是硬编码在代码里的——ReShare、yfinance、akshare 三个数据源的开关和地址散落在多个文件中。想临时关闭某个数据源？得翻代码改逻辑。

需求很简单：把数据源配置提取到统一配置文件，通过 config 字典控制每个数据源的 enabled 状态和 base_url。

但”简单需求”最容易翻车——改着改着顺手重构、碰了不该碰的文件、引入 regression。本文记录了一次严格遵守 Karpathy 四原则的重构实践：3 个文件、1 小时完成、零 regression。

## Karpathy 四原则

这是一套由 Andrej Karpathy 提出的代码修改纪律，核心理念是”最小化人类干预”：

### 原则一：编码前思考

> 

列出所有需要改动的文件，想清楚每个文件改什么、不改什么。

不急于动手。先把改动清单写到纸上（或文档里），每个文件精确到”第几行改成什么”。

### 原则二：简洁优先

> 

只做最小改动。不创建新文件，不添加新功能，不搞抽象。

这次的目标是”配置化”，不是”重构数据获取层”。后者是另一个任务。混在一起做，两边都做不好。

### 原则三：精准修改

> 

只改清单上列出的内容。不碰禁止清单上的文件。不改相邻代码、注释、格式。不改没坏的东西。不顺手优化。

### 原则四：目标驱动执行

> 

每改完一个文件就验证。全部改完运行 diff 确认范围。

## 实践：任务文档的结构

这次重构的核心工具是一份**任务文档**——在编码前就写好，作为执行过程中的”宪法”：

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
17
18
19
20
21
22
23

## 改动清单

### 文件 1: data_fetcher.py
1. 删第4行 import yfinance as yf（顶层导入）
2. 在 import time 后添加 from app.config import Config
3. BASE_URL 改为 Config.DATA_SOURCES["reshare"]["base_url"]
   TIMEOUT 改为 Config.DATA_SOURCES["reshare"]["timeout"]
4. _get_real_history() 中 yfinance 块：加 config 检查 + inline import
5. _get_real_history() 中 akshare 块：加 config 检查
...

### 文件 2: portfolio_service.py
1. 顶部添加 from app.config import Config
2. _get_latest_price() 中加 config 检查

### 文件 3: api.py
1. 找到所有 reshare-backend 字符串，改为实际地址

## 禁止改动清单（绝对不碰）
- app/config.py — 已改完
- frontend/ 下的任何文件
- app/data/ 下的任何文件
- scripts/ 下的任何文件

```

这份文档有几个关键设计：

**精确到行级的改动指令**——不是”把 yfinance 改成配置化”，而是”第4行删除、第5行后新增、第30行的常量替换为 Config.xxx”。这样执行时不需要思考”怎么改”，只需要”按指令改”。

**显式的禁止清单**——列出绝对不能碰的文件和目录。这比”注意别改其他文件”有效得多。

**验收标准写在前面**——在动手之前就定义好”做完”的标准：

```bash

1
2
3
4

python -m py_compile data_fetcher.py      # 无语法错误
python -m py_compile portfolio_service.py  # 无语法错误
python -m py_compile api.py               # 无语法错误
git diff --stat                           # 只改了这3个文件

```

## 执行过程

### 步骤 1：编码前思考（10 分钟）

读完三个目标文件，把每一处需要改动的位置精确记录到任务文档中。共识别出 10 处改动点（文件1）、2 处（文件2）、1 处（文件3）。

### 步骤 2-4：按文件逐一修改 + 即时验证（40 分钟）

**改 data_fetcher.py**：

核心改动是把硬编码的数据源调用改为 config 检查：

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

# 改动前（硬编码，无法控制开关）
import yfinance as yf
try:
    df = yf.download(ticker, ...)
except:
    pass

# 改动后（配置化，可通过 config 开关）
if not Config.DATA_SOURCES["yfinance"]["enabled"]:
    raise Exception("yfinance disabled")
import yfinance as yf  # inline import，不用时不加载
try:
    df = yf.download(ticker, ...)
except:
    pass

```

每改完一个文件立即验证：

```bash

1

python -m py_compile data_fetcher.py  # ✅ 通过

```

**改 portfolio_service.py** 和 **api.py** 同理，各 2 处改动。

### 步骤 5：全局验证（10 分钟）

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

# 三个文件语法检查
python -m py_compile data_fetcher.py       # ✅
python -m py_compile portfolio_service.py   # ✅
python -m py_compile api.py                 # ✅

# 确认只改了预期文件
git diff --stat
# data_fetcher.py      | 28 +++--
# portfolio_service.py |  3 +
# api.py               |  2 +-
# 3 files changed

```

`git diff --stat` 显示**恰好改了 3 个文件**，与改动清单完全一致。零意外改动。

### 步骤 6：提交

```bash

1
2

git add data_fetcher.py portfolio_service.py api.py
git commit -m "refactor: 数据源统一配置化，支持 enabled 开关"

```

## 成效对比

维度
传统”自由发挥”
四原则约束

耗时
2-3 小时
1 小时

改动文件数
3-5 个（含顺手优化）
3 个（精确）

regression 数
1-2 个
0

返工次数
1-2 次
0

代码审查
需要识别哪些是”需求改动”哪些是”顺手改动”
diff 清晰，一眼看完

## 核心经验

### 1. 任务文档是四原则的载体

口头说”遵循四原则”没有约束力。必须把改动清单、禁止清单、验收标准写成文档，执行时对照检查。

### 2. “禁止清单”比”改动清单”更重要

人的本能是”看到能优化的就想优化”。显式的禁止清单（”绝对不碰 config.py、frontend/、data/“）是防止越界的护栏。

### 3. inline import 配合 config 检查是优雅的开关模式

```python

1
2
3

if not Config.DATA_SOURCES["yfinance"]["enabled"]:
    raise Exception("yfinance disabled")
import yfinance as yf  # 不用时完全不加载

```

这样既实现了开关，又避免了”顶层 import 一个不用的库”的浪费。

### 4. git diff –stat 是最后的安全网

不管中间过程多仔细，提交前必须看 `git diff --stat`。如果出现了清单之外的文件，说明中间有意外改动，必须回溯。

---

> 

重构的质量不取决于”改了多少代码”，而取决于”没改多少代码”。最小化改动范围、最大化验证覆盖——这才是零 regression 的秘诀。Karpathy 四原则不是理论，是可以量化执行的工程纪律。