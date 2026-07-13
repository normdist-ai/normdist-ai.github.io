---
title: 'A2A 协议实战：两个 AI Agent 协作升级路由器技能'
date: 2026-07-09 10:00:00
tags: [A2A协议, AI Agent, 多Agent协作, MCP桥接, 路由器管理]
categories: [技术笔记]
---


## 背景：从工具共享到任务协作

在上一篇文章中，我们解决了 Agent 间的工具共享问题——通过 MCP 桥接，Linux 上的主 Agent 可以调用 Windows Agent 的 13 个工具。但那是单向的：主 Agent 调远程工具，远程 Agent 只是被动的执行者。

真正的多 Agent 协作（Agent-to-Agent，A2A）需要更深一层：**两个 Agent 像同事一样，围绕同一个任务讨论方案、分工执行、互相校验。**

这篇文章记录的就是这样一次实践：两个 AI Agent 通过 A2A 协议，协作升级一个路由器管理技能的全部过程。

## 需求：iKuai 路由器技能升级

iKuai 是一款国产软路由系统，提供 Web 管理界面。此前已经有一个旧版技能（下称 v2），能做基础操作：查询连接状态、管理端口转发、查看流量统计。但功能有限，且与新固件的 API 有兼容性问题。

升级目标（v3）：

- 修复与新固件 API 的兼容性（认证流程变更）

- 增加多 WAN 口负载均衡查询

- 增加 QoS 规则批量管理

- 重构错误处理（旧版大量裸 except）

问题在于：路由器的调试需要**实地操作**——SSH 到路由器、抓 HTTP 包、测试 API 响应。主 Agent 在 Linux 上运行，够不到路由器的管理界面。但它有一个 Windows 端的搭档 Agent，那台机器可以直接访问路由器的管理网段。

这就是 A2A 登场的场景。

## 架构：A2A 协作的三步闭环

```armasm

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

主 Agent (Linux)                    Windows Agent
┌──────────────────┐               ┌──────────────────┐
│                  │  1. 发起讨论   │                  │
│ 分析 v2 技能代码  │ ────────────► │ 接收任务上下文    │
│ 列出升级需求      │               │                  │
│                  │  2. 实地验证   │                  │
│                  │ ◄──────────── │ SSH 连路由器      │
│                  │               │ 抓包测试 API      │
│ 审查验证结果      │  3. 反馈方案   │ 返回 API 文档     │
│ 编写 v3 技能代码  │ ◄──────────── │ 和测试数据        │
└──────────────────┘               └──────────────────┘

```

三步闭环：

- **主 Agent 发起**：分析旧技能代码，梳理升级清单，通过 A2A 通道将任务上下文发送给 Windows Agent

- **Windows Agent 验证**：在本地操作路由器（SSH 连接、抓包、API 测试），将实测数据返回

- **主 Agent 收尾**：根据实测数据，完成代码编写和技能发布

这不是简单的”调一个工具拿结果”，而是多轮往返的协作：讨论→验证→反馈→再验证。

## 实践过程

### 第一步：技能代码审查

主 Agent 先读取旧版 v2 技能的全部代码。核心发现：

**认证流程过时**：

```python

1
2
3
4
5
6
7

# v2 的认证方式（旧固件）
session = requests.Session()
session.post(f"http://{router_ip}/login", data={
    "username": user,
    "password": pwd
})
# 新固件改了——用 Token + CSRF

```

v2 用的是简单的 Session Cookie 认证，但新固件引入了 CSRF Token 机制。旧代码登录后拿到的 Cookie 在后续请求中被 403 拒绝，用户以为技能坏了，实际是认证流程变了。

**错误处理裸奔**：

```python

1
2
3
4
5
6

# v2 的错误处理
try:
    resp = session.get(url)
    return resp.json()
except:
    return {}  # 吞掉所有异常，返回空字典

```

路由器 API 超时、返回 HTML 错误页（而不是 JSON）、网络不通——这些场景全部被裸 `except` 吞掉。调用方拿到空字典，无从判断是真没数据还是出了错。

**功能缺口**：v2 只实现了查询类操作，没有管理类操作（增删改 QoS 规则、端口转发规则）。

### 第二步：A2A 通道 — 发起协作请求

主 Agent 通过 MCP 桥接通道，向 Windows Agent 发起协作：

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

任务：验证 iKuai 新固件 API 的认证流程和关键接口
背景：旧技能 v2 的 Session Cookie 认证在新固件上失效
需求：
  1. 登录路由器管理界面，抓取登录请求的完整 HTTP 流程
  2. 验证新认证方式（Token / CSRF）
  3. 测试以下 API 的请求/响应格式：
     - WAN 口状态查询
     - QoS 规则列表
     - 端口转发规则列表

```

这一步通过 A2A 通道传输的不是代码，而是**任务描述和上下文**。主 Agent 把”需要验证什么”讲清楚，Windows Agent 负责在本地执行。

### 第三步：Windows Agent 实地验证

Windows Agent 接到任务后，执行了以下操作（44 条消息，23 次工具调用）：

- **SSH 连接路由器**：确认网络可达，获取固件版本号

- **浏览器抓包**：打开路由器管理界面，用 CDP 开发者工具抓取登录请求

- **API 探测**：逐个测试关键接口，记录请求格式和响应结构

关键发现（Windows Agent 返回的验证结果）：

**新认证流程**：

```pgsql

1
2
3
4

1. GET /api/system/auth_token → 获取 CSRF Token
2. POST /api/login (Header: X-CSRF-Token, Body: {user, pwd})
   → 返回 { "token": "xxx", "expires": 3600 }
3. 后续请求 Header 带 Authorization: Bearer xxx

```

**WAN 口状态 API 响应格式**：

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
  "code": 0,
  "data": [{
    "interface": "wan1",
    "status": "connected",
    "ip": "10.x.x.x",
    "rx_bytes": 1234567890,
    "tx_bytes": 987654321
  }]
}

```

这些是**实测数据**，不是文档抄来的。新旧固件的差异只有实地抓包才能发现。

### 第四步：主 Agent 编写 v3 技能

拿到 Windows Agent 返回的 API 文档和测试数据后，主 Agent 开始编写 v3：

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

class iKuaiV3:
    def __init__(self, host, user, pwd):
        self.host = host
        self.session = requests.Session()
        self._authenticate(user, pwd)

    def _authenticate(self, user, pwd):
        """新固件认证：CSRF Token + Bearer Token"""
        # 1. 获取 CSRF Token
        resp = self.session.get(f"http://{self.host}/api/system/auth_token")
        csrf_token = resp.json()["csrf_token"]

        # 2. 登录
        resp = self.session.post(
            f"http://{self.host}/api/login",
            json={"username": user, "password": pwd},
            headers={"X-CSRF-Token": csrf_token}
        )
        result = resp.json()
        if result.get("code") != 0:
            raise AuthError(f"Login failed: {result.get('msg', 'unknown')}")

        self.bearer_token = result["data"]["token"]
        self.session.headers["Authorization"] = f"Bearer {self.bearer_token}"

    def get_wan_status(self):
        """查询 WAN 口状态"""
        resp = self.session.get(f"http://{self.host}/api/network/wan")
        self._check_response(resp)
        return resp.json()["data"]

```

改进点：

- **显式认证**：CSRF Token + Bearer Token 两步认证，认证失败直接抛 `AuthError`

- **错误分层**：网络错误、认证错误、API 业务错误分别用不同异常类型，不再裸 except

- **Token 过期自动续期**：检测到 401 响应时自动重新认证

### 第五步：闭环验证

v3 代码写完后，主 Agent 再次通过 A2A 通道请 Windows Agent 做最终验证：

```markdown

1
2
3
4
5

任务：验证 v3 技能代码在路由器上的实际运行效果
需求：
  1. 在本地运行 v3 代码
  2. 测试 4 个核心接口（认证、WAN 状态、QoS 列表、端口转发）
  3. 故意制造一次 Token 过期，验证自动续期逻辑

```

Windows Agent 在本地跑了一遍 v3 代码，4 个接口全部通过，Token 过期自动续期逻辑验证通过。整个协作闭环完成。

## A2A 协作 vs 传统工具调用

这次实践最大的收获不是 v3 技能本身，而是 A2A 协作模式的验证。跟传统的主 Agent 单向调用工具相比：

维度
传统工具调用
A2A 协作

信息流
单向：主 Agent 发指令，拿结果
双向：讨论、验证、反馈、再验证

决策权
主 Agent 全权决策
双方都有判断权，可以提出异议

任务粒度
一个工具 = 一次调用
一个任务 = 多轮协作

适用场景
明确的操作（读文件、跑命令）
需要探索和验证的复杂任务

关键区别在于：**传统工具调用中，远程 Agent 是手。A2A 协作中，远程 Agent 是同事。**

这次 iKuai 技能升级，Windows Agent 不是简单地”帮我跑个命令”，而是：

- 自主决定怎么验证（先 SSH 再抓包再 API 测试）

- 发现问题主动汇报（认证方式变了不只是报个错，而是分析出完整的替代方案）

- 对主 Agent 的代码提出改进建议（Token 过期续期是 Windows Agent 在测试中发现的需求）

## 反思：什么场景适合 A2A

不是所有任务都需要 A2A。这次实践之后，总结了一个判断框架：

**适合 A2A 的场景**：

- 需要实地操作才能获取信息（抓包、实机测试）

- 信息在协作过程中会变化（验证后发现新问题，需要调整方案）

- 两个 Agent 有互补能力（一个擅长编码，一个擅长运维操作）

**不适合 A2A 的场景**（传统工具调用就够）：

- 标准化的读写操作（文件、数据库）

- 单次就能完成的操作（查询状态、执行命令）

- 不需要多轮往返的简单任务

## 总结

维度
实践结果

协作模式
A2A（双向讨论 + 多轮验证）

协作轮次
2 轮（发起验证 + 闭环验证）

消息量
44 条消息，23 次工具调用

技能版本
v2 → v3（认证重构 + 功能扩展 + 错误处理分层）

核心价值
从”工具共享”升级到”任务协作”

A2A 协作的本质不是技术协议的复杂度——MCP 桥接已经解决了通信问题。真正的挑战是**任务编排**：什么时候该发起协作、怎么描述任务上下文、怎么处理反馈分歧。

这次 iKuai 技能升级是一个理想案例：任务边界清晰、验证可量化、两个 Agent 能力互补。更复杂的 A2A 场景（多 Agent 并行协作、动态任务分配、冲突仲裁）还需要在实践中继续探索。