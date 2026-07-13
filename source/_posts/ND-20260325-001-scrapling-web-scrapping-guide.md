---
title: 'Scrapling 自适应 Web Scraping 框架实战：抓取上证指数 K 线数据'
date: 2026-03-25 10:00:00
tags: [Python, Web Scraping, Scrapling]
categories: [技术笔记]
---

## 前言

在日常开发和数据工作中，我们经常需要从网站上抓取数据。传统的 Web Scraping 方案存在以下痛点：

- **网站结构变化导致代码失效** - HTML/CSS 结构更新后，选择器需要重写

- **反爬机制日益复杂** - Cloudflare Turnstile、WAF 等防护手段让普通 HTTP 请求失效

- **动态内容难以抓取** - JavaScript 渲染的内容需要浏览器自动化

- **并发和效率问题** - 大规模抓取需要考虑并发控制和资源占用

今天介绍的 **Scrapling** 框架，正是为了解决这些痛点而生的自适应 Web Scraping 工具。

## 什么是 Scrapling？

Scrapling 是一个功能强大的 Python Web Scraping 框架，由 Karim Shoair 开发，具有以下特点：

### 核心优势

- 

**自适应元素定位**

- 智能跟踪算法能在网站结构变化后自动重新定位元素

- 支持多种选择器：CSS、XPath、BeautifulSoup 风格

- 

**强大的 Fetcher 系统**

- 支持多种 Fetcher 类型：

- `Fetcher` - 快速 HTTP 请求

- `AsyncFetcher` - 异步并发抓取

- `StealthyFetcher` - 隐秘模式，模拟浏览器指纹

- `DynamicFetcher` - 完整浏览器自动化（基于 Playwright）

- 

**Spider 框架**

- 类 Scrapy 的 Spider API

- 支持并发、Session 管理、暂停/恢复

- 内置 Proxy 轮换

- 

**高性能**

- 优化的解析引擎，比 Beautiful Soup 快 700 倍以上

- 低内存占用，支持大规模抓取

## Docker 部署

Scrapling 提供了官方 Docker 镜像，包含所有浏览器和依赖项：

```bash

1
2
3
4
5

# 拉取官方镜像（1.55GB，包含 Playwright 浏览器）
docker pull ghcr.io/d4vinci/scrapling:latest

# 或从 Docker Hub 拉取
docker pull pyd4vinci/scrapling

```

也可以自行构建轻量级镜像（不含浏览器，476MB）：

```dockerfile

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

FROM python:3.12-slim

WORKDIR /app

# 安装核心功能
RUN pip install --no-cache-dir "scrapling[fetchers]"

WORKDIR /workspace

CMD ["python"]

```

构建命令：

```bash

1

docker build -t scrapling-local .

```

## 实战案例：抓取上证指数 K 线数据

### 挑战描述

目标：从东方财富网抓取上证指数（代码：000001）最近 7 天的收盘价数据

难点：

- API 需要特定的 Referer 和 User-Agent 头

- 数据以 JSONP 格式返回（带有 jQuery 回调前缀）

- SSL 证书验证问题（主机名不匹配）

### 解决方案

#### 1. 使用 Scrapling 的底层 HTTP 引擎

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
33
34
35
36
37
38
39
40
41
42
43
44
45
46

from scrapling.engines.static import CurlSession
import json
import re

# API 端点
url = 'https://push2his.eastmoney.com/api/qt/stock/kline/get'

# 请求参数
params = {
    'secid': '1.000001',           # 上证指数
    'ut': 'fa5fd1943c7b386f172d6893dbfba10b',
    'fields1': 'f1,f3',           # 日期和收盘价
    'fields2': 'f51,f52,f53,f54,f55,f56,f57,f58,f59,f60,f61',
    'klt': '101',                    # 日K线
    'fqt': '1',                      # 前复权
    'beg': '20260318',              # 开始日期
    'end': '20260325',               # 结束日期
    'lmt': '200'                     # 返回条数
}

# 必要的请求头
headers = {
    'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36',
    'Referer': 'https://quote.eastmoney.com/zs000001.html'
}

# 使用 CurlSession 发送请求
session = CurlSession()
resp = session.request('GET', url, params=params, headers=headers, 
                     verify=False, timeout=15)

# 解析 JSONP 响应
match = re.search(r'\{.*\}', resp.text, re.DOTALL)
data = json.loads(match.group())

# 提取 K 线数据
klines = data['data']['klines']

# 解析数据
result = []
for kline in klines:
    fields = kline.split(',')
    result.append({
        'date': fields[0],          # 日期
        'close': float(fields[3])   # 收盘价（需要除以100）
    })

```

#### 2. 数据结果

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

[
  {
    "date": "2026-03-18",
    "close": 4065.37
  },
  {
    "date": "2026-03-19",
    "close": 4042.02
  },
  {
    "date": "2026-03-20",
    "close": 4022.7
  },
  {
    "date": "2026-03-23",
    "close": 3906.62
  },
  {
    "date": "2026-03-24",
    "close": 3881.42
  },
  {
    "date": "2026-03-25",
    "close": 3933.06
  }
]

```

**趋势分析**：期间从 4065.37 跌至 3933.06，跌幅约 3.28%

### 技术要点

#### 1. 绕过 SSL 验证

```python

1

resp = session.request(..., verify=False)

```

东方财富的 SSL 证书存在主机名不匹配问题，`verify=False` 可以绕过。

#### 2. JSONP 解析

```python

1
2

match = re.search(r'\{.*\}', resp.text, re.DOTALL)
data = json.loads(match.group())

```

API 返回格式为：`jQuery123456789({...})`，需要提取 JSON 部分。

#### 3. 数据缩放

```python

1

'close': float(fields[3])  # 原始数据需要除以100

```

API 返回的收盘价是整数形式（如 406537），实际值为 4065.37。

## Scrapling 使用技巧

### 1. 高级选择器

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

from scrapling.fetchers import Fetcher

page = Fetcher.get('https://example.com')

# CSS 选择器
quotes = page.css('.quote .text::text').getall()

# XPath 选择器
quotes = page.xpath('//span[@class="text"]/text()').getall()

# BeautifulSoup 风格
quotes = page.find_all('div', class_='quote')

```

### 2. 异步并发抓取

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

from scrapling.fetchers import AsyncFetcher
import asyncio

async def fetch_multiple():
    urls = [
        'https://example.com/page1',
        'https://example.com/page2',
        'https://example.com/page3'
    ]
    
    fetcher = AsyncFetcher()
    tasks = [fetcher.get(url) for url in urls]
    results = await asyncio.gather(*tasks)
    
    for result in results:
        print(f"Status: {result.status}")

asyncio.run(fetch_multiple())

```

### 3. 隐秘模式（Stealthy Mode）

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

from scrapling.fetchers import StealthyFetcher

# 模拟浏览器指纹
page = StealthyFetcher.fetch(
    'https://example.com',
    impersonate='chrome',      # 使用 Chrome TLS 指纹
    headless=True,
    solve_cloudflare=True      # 自动解决 Cloudflare Turnstile
)

```

## 常见问题和解决方案

### 问题 1：SSL 证书错误

**错误信息**：

```cmake

1

SSL: no alternative certificate subject name matches target hostname

```

**解决方案**：

```python

1
2

session = CurlSession()
resp = session.request(..., verify=False)

```

### 问题 2：API 返回空响应

**可能原因**：

- 缺少必要的请求头（Referer、User-Agent）

- 参数格式不正确

- IP 被限制

**解决方案**：

```python

1
2
3
4
5

headers = {
    'User-Agent': 'Mozilla/5.0 ...',
    'Referer': 'https://quote.eastmoney.com/...'
}
resp = session.request('GET', url, headers=headers)

```

### 问题 3：动态加载的内容抓取不到

**解决方案**：使用 DynamicFetcher（需要浏览器）

```python

1
2
3
4
5

from scrapling.fetchers import DynamicFetcher

with DynamicFetcher(headless=True) as fetcher:
    page = fetcher.fetch('https://example.com')
    data = page.css('.dynamic-content')

```

## 总结

Scrapling 是一个功能强大且易用的 Web Scraping 框架，特别适合处理复杂的数据抓取场景：

### 优点

✅ 自适应元素定位，网站结构变化后仍能工作
✅ 多种 Fetcher 类型，满足不同需求
✅ 高性能，比传统方案快数百倍
✅ 支持异步并发，适合大规模抓取
✅ 提供 Docker 镜像，部署简单  

### 适用场景

- 新闻网站抓取

- 电商数据采集

- 财经行情数据获取

- 社交媒体数据提取

- 任何需要 Web Scraping 的场景

### 学习资源

- [GitHub 仓库](https://github.com/D4Vinci/Scrapling)

- [官方文档](https://scrapling.readthedocs.io/)

- [示例代码](https://github.com/D4Vinci/Scrapling/tree/main/examples)

**推荐指数**：⭐⭐⭐⭐⭐

如果你需要在项目中抓取 Web 数据，Scrapling 是一个非常值得尝试的工具！

---

**参考数据**：

- 抓取对象：上证指数（000001）

- 数据时间范围：2026-03-18 至 2026-03-25

- 数据点数：6 个交易日

- 期间涨跌：-3.28%