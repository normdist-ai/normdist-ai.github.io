---
title: 'Agent-to-Agent MCP 桥接：让两个 AI Agent 互相调用工具'
date: 2026-07-05 10:00:00
tags: [AI Agent, MCP, Agent 间通信, 工具共享, 桥接方案, 跨平台]
categories: [技术笔记]
---


## 背景：一个 Linux Agent，一个 Windows Agent

一个运行在 Linux 上的 AI Agent（下称”主 Agent”），已经有了完整工具链：文件读写、代码执行、定时任务、数据库查询。但它够不到隔壁一台 Windows 主机——那台机器上有另一个 Agent（下称”远程 Agent”），能做主 Agent 做不了的事：控制 Edge 浏览器登录态、操作 Windows 桌面应用、读取本地系统信息。

两个 Agent 各有专长，但彼此隔离。需求很直接：**让主 Agent 能像调用本地工具一样，调用远程 Agent 的工具。**

这不是 HTTP API 互调，也不是消息队列。这是 Agent 层面的”工具共享”——主 Agent 的 LLM 推理过程中，可以直接发现远程 Agent 暴露的工具，拿到工具描述，然后决定调用哪个、传什么参数，最后拿回结果继续推理。整个过程对 LLM 是透明的，就像那些工具装在本地一样。

## 技术选型：为什么是 MCP

### 候选方案

**方案一：REST API 直连**
远程 Agent 起一个 HTTP 服务，暴露 REST 接口。主 Agent 用 `requests.post()` 调用。

问题：每个工具要手写接口文档、手写参数校验、手写错误处理。LLM 拿不到结构化的工具描述（JSON Schema），无法自动发现和调用。本质上是”人肉 MCP”。

**方案二：自定义 JSON-RPC**
定义一套自己的 RPC 协议，主 Agent 发请求，远程 Agent 回响应。

问题：跟 MCP 解决同一个问题，但 MCP 已经是开放标准，有 Python/TypeScript SDK，有现成的工具发现机制。重复造轮子没有收益。

**方案三：MCP（Model Context Protocol）**
Anthropic 在 2024 年提出的开放协议，专门解决”LLM 如何使用外部工具”的问题。核心能力：

- **工具发现**：客户端连接服务端后，自动获取所有工具的名称、描述、参数 Schema

- **标准化调用**：JSON-RPC 2.0 over stdio/SSE，所有工具调用走同一个协议

- **SDK 成熟**：Python `mcp` 包有 `FastMCP` 装饰器，几行代码注册一个工具

MCP 原生支持两种传输：**stdio**（本地管道）和 **SSE**（网络流）。对于跨机器场景，两种都能用，但实现路径不同。

### 最终选择：SSH 隧道 + stdio

选了 SSH 隧道 + stdio 模式，而不是 SSE 直连。原因：

- **安全**：Windows 主机在内网，不暴露公网端口。SSH 隧道加密传输，且能复用已有的 SSH 密钥认证

- **简单**：不需要在 Windows 上额外配置防火墙入站规则、TLS 证书

- **stdio 天然适合**：MCP 的 stdio 传输就是”通过 stdin/stdout 传 JSON-RPC”，SSH 的 `-T` 模式（禁用 TTY）恰好把远程进程的 stdin/stdout 透明地接到本地

## 实现：三层架构

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

主 Agent (Linux)                    远程 Agent (Windows)
┌──────────────┐                    ┌──────────────────────────┐
│ config.yaml  │                    │ hermes_mcp_server.py     │
│ mcp_servers: │   SSH -T 管道      │   (FastMCP, stdio)       │
│   remote:    │◄─────────────────► │                          │
│     command  │   JSON-RPC 2.0     │ Tools:                   │
│     bash     │   over stdin/stdout│ • check_health           │
│     tunnel.sh│                    │ • get_system_info        │
└──────────────┘                    │ • run_command            │
                                    │ • browser_navigate       │
                                    │ • browser_screenshot     │
                                    │ • ...（共 13 个）         │
                                    └──────────────────────────┘

```

### 第一层：SSH 隧道脚本（主 Agent 侧）

这是整个桥接的”管道”。一个 bash 脚本，用 `ssh -T` 连到远程，启动远程的 MCP server 进程：

```bash

1
2
3
4
5
6
7

#!/bin/bash
exec sshpass -p 'PASSWORD' ssh \
  -o StrictHostKeyChecking=no \
  -o ServerAliveInterval=30 \
  -T \
  'user@10.x.x.x' \
  'python "C:\\Users\\RemoteAgent\\hermes_mcp_server.py"'

```

关键点：

- **`-T`：禁用 TTY 分配。没有这个 flag，SSH 会分配伪终端，stdin/stdout 被终端控制字符污染，JSON-RPC 消息会乱码。这是**最容易踩的坑——MCP 初始化静默失败，没有任何报错，只是工具发现返回空列表。

- **`ServerAliveInterval=30`**：SSH 保活。MCP 会话是长连接，没有这个参数，NAT 或防火墙会在几分钟空闲后掐断连接，下次工具调用时报 “broken pipe”。

- **`exec`**：用 `exec` 替换当前 shell 进程，而不是子进程。这样 SSH 进程的 stdin/stdout 直接就是脚本的 stdin/stdout，MCP 客户端能正确接管管道。

### 第二层：MCP Server（远程 Agent 侧）

用 `FastMCP` 写服务端，比手写 JSON-RPC 协议处理简洁得多：

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

from mcp.server.fastmcp import FastMCP

mcp = FastMCP("remote-agent")

@mcp.tool()
def check_health() -> dict:
    """Health check — returns hostname, timestamp, status."""
    import socket, time
    return {
        "hostname": socket.gethostname(),
        "timestamp": time.time(),
        "status": "ok"
    }

@mcp.tool()
def get_system_info() -> dict:
    """OS, CPU, RAM, disk usage details."""
    import psutil
    return {
        "cpu_percent": psutil.cpu_percent(),
        "memory": psutil.virtual_memory()._asdict(),
        "disk": psutil.disk_usage('/')._asdict(),
    }

@mcp.tool()
def run_command(command: str, timeout: int = 30) -> dict:
    """Execute PowerShell command via subprocess."""
    import subprocess
    result = subprocess.run(
        ["powershell.exe", "-Command", command],
        capture_output=True, text=True, timeout=timeout
    )
    return {
        "stdout": result.stdout,
        "stderr": result.stderr,
        "returncode": result.returncode
    }

if __name__ == "__main__":
    mcp.run()

```

**为什么用 FastMCP 而不是原始 `stdio_server`？**

MCP SDK 1.26+ 对 capabilities 字段做了强制校验。如果用原始的 `Server` 类手动注册工具，必须手动构造 `ServerCapabilities()` 对象，漏掉任何字段都会报：

```elm

1

ValidationError: Field required [type=missing]

```

FastMCP 自动处理 capabilities 声明，装饰器语法也比手动注册干净。**结论：无脑选 FastMCP。**

### 第三层：客户端配置（主 Agent 侧）

在主 Agent 的配置文件中注册这个 MCP 服务端：

```yaml

1
2
3
4
5
6
7
8

mcp_servers:
  remote-agent:
    command: bash
    args:
      - /path/to/mcp-ssh-tunnel.sh
    timeout: 60
    connect_timeout: 30
    enabled: true

```

主 Agent 启动时，会执行这个 bash 脚本，建立 SSH 连接，然后通过 MCP 协议发现远程 Agent 的全部工具。从此，LLM 在推理时看到的工具列表里，多了 `remote-agent_check_health`、`remote-agent_run_command`、`remote-agent_browser_navigate` 等条目——和本地工具毫无区别。

## 13 个工具：系统能力 + 浏览器能力

远程 Agent 最终注册了 13 个工具，分两类：

### 系统工具（7 个）

工具
功能

`check_health`
健康检查——主机名、时间戳、状态

`get_system_info`
OS、CPU、内存、磁盘详情

`run_command`
执行 PowerShell 命令

`list_files`
浏览 Windows 文件系统

`read_file`
读取文本文件（带行号）

`write_file`
写入文件（自动创建目录）

`get_processes`
进程列表（按 CPU/内存排序）

### 浏览器工具（6 个）

工具
功能

`browser_list_tabs`
列出所有浏览器标签页

`browser_navigate`
导航到 URL，返回标题+正文

`browser_screenshot`
页面截图（base64 JPEG）

`browser_get_page_text`
获取当前页面文本

`browser_get_cookies`
提取 Cookie（可按域名过滤）

`browser_execute_js`
执行任意 JavaScript

浏览器工具基于 CDP（Chrome DevTools Protocol）端口 9222，复用用户在远程机器上已登录的浏览器会话——这是最有价值的能力：主 Agent 可以操作远程的登录态，做那些需要”真人登录”才能做的事。

## 踩坑记录

### 坑一：SSH 不加 `-T` 导致 MCP 初始化静默失败

症状：`hermes mcp test remote-agent` 显示 “Connected” 但 “Tools discovered: 0”。

根因：SSH 默认分配 TTY，终端控制字符（`\r\n`、ANSI 转义序列）混入 JSON-RPC 消息流，MCP 客户端解析 JSON 失败，但 SDK 把解析错误吞掉了，没有向上抛。

修复：SSH 命令加 `-T` flag。

### 坑二：MCP SDK 版本不匹配导致 capabilities 校验失败

症状：远程 Agent 启动正常，但主 Agent 连接后立刻断开，日志里有 `ValidationError: Field required [type=missing]`。

根因：远程机器上的 `mcp` 包版本 < 1.26，使用了旧的 `Server` 类，没有声明 `capabilities`。SDK 1.26+ 强制要求这个字段。

修复：`pip install mcp>=1.26.0`，或改用 `FastMCP`（自动声明 capabilities）。

### 坑三：psutil 未安装导致系统工具报错

症状：`get_system_info` 和 `get_processes` 调用后返回 500 错误。

根因：远程机器是干净的 Python 环境，没装 `psutil`。FastMCP 的工具函数在执行时才 import，import 失败直接抛异常。

修复：`pip install psutil`。建议在 MCP server 启动时统一 import，把依赖缺失问题前置暴露。

## 从 A2A 到多 Agent 协作

这个 MCP 桥接解决的是”工具共享”问题——主 Agent 能调用远程 Agent 的工具。但真正的 Agent-to-Agent（A2A）协作不止于此。

在这次实践中，主 Agent 不只是机械地调用远程工具，还会**根据远程 Agent 返回的信息，自主决定下一步操作**。比如：

- 主 Agent 调用 `remote-agent_browser_navigate` 打开一个需要登录的页面

- 发现页面跳转到登录页（说明 session 过期）

- 主 Agent 通知用户”远程浏览器需要重新登录”

- 用户在远程桌面完成登录后，主 Agent 重新调用工具，继续操作

这个循环里，主 Agent 承担了**编排者（Orchestrator）**的角色：它不执行具体操作，但决定调用哪些工具、按什么顺序、如何处理异常。远程 Agent 则是**能力提供者（Capability Provider）**：它不思考，只执行被调用的工具。

这种分工，比”一个巨型 Agent 做所有事”的架构更清晰，也更容错——远程 Agent 挂了不影响主 Agent 的其他能力，主 Agent 的 LLM 更换不影响远程 Agent 的工具实现。

## 总结

维度
方案

传输层
SSH 隧道 + `-T`（禁用 TTY）

协议层
MCP over stdio（JSON-RPC 2.0）

服务端
FastMCP（自动声明 capabilities）

客户端
主 Agent config.yaml 注册 mcp_server

工具数
13 个（7 系统 + 6 浏览器）

安全
SSH 密钥认证 + 内网不暴露公网

核心技术不是什么新发明——SSH 管道是 30 年前的东西，MCP 是 2024 年的标准。真正有价值的是**把它们组合起来**，用最小复杂度实现了 Agent 间的工具共享。

三个最容易踩的坑：SSH `-T` flag、MCP SDK capabilities 校验、远程依赖（psutil）前置安装。踩过一次就不会再踩。