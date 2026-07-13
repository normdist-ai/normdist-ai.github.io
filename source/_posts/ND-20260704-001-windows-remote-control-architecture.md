---
title: 'AI Agent 如何远程控制 Windows：SSH + MCP + CDP 三层方案选型'
date: 2026-07-04 10:00:00
tags: []
categories: [技术笔记]
---


## 背景：一个 Linux Agent 想操作 Windows

一个运行在 Linux 服务器上的 AI Agent，已经能通过 CDP（Chrome DevTools Protocol）远程操控 Windows 上的 Edge 浏览器。现在需求升级了——不只是浏览器，还要控制**整个 Windows 操作系统**：文件管理、应用启动、系统设置、桌面 GUI 交互。

核心约束很明确：**所有方案必须从 Linux 远程控制 Windows**，不能在 Windows 上跑 Agent 主进程。

这是一个典型的”跨 OS 远程控制”架构选型问题。本文记录了完整的调研过程和最终决策。

## 候选方案全景扫描

市面上能做”AI 控制 GUI”的方案不少，但加上”从 Linux 远程控制 Windows”这个约束后，可选范围迅速收窄。以下是逐一评估。

### 方案一：Microsoft UFO-Agent（UI Automation 原生方案）

[UFO-Agent](https://github.com/microsoft/UFO) 是微软官方出品的 Windows GUI 自动化 Agent，MIT 开源。

**架构亮点**：UFO 不依赖截图和视觉模型，而是直接调用 Windows 内置的 **UI Automation (UIA) API**——这是 Windows 的无障碍接口（Accessibility API）。通过 UIA 树获取应用程序的 UI 元素结构（按钮、菜单、输入框），再通过 UIA 直接操作这些元素。

双 Agent 架构：

- **AppAgent**：分析用户意图，从已安装应用中选择合适的工具

- **ActAgent**：在选定应用内部执行具体操作（点击、输入、滚动）

**远程控制评估**：

UFO 支持 RESTful API 模式，可以在 Windows 上作为 HTTP 服务运行：

```bash

1
2
3
4
5
6
7
8

# Windows 侧部署
pip install ufo-agent
python -m ufo.restful_api --host 0.0.0.0 --port 8000

# Linux 侧调用
curl -X POST http://<windows-ip>:8000/api/task \
  -H "Content-Type: application/json" \
  -d '{"task": "打开文件管理器，在D盘创建一个名为test的文件夹"}'

```

维度
评估

可行性
✅ 高度可行（HTTP Bridge）

延迟
低（局域网 <50ms，LLM 调用 2-5s/步）

精确度
⭐⭐⭐⭐⭐（原生 UIA，元素级定位）

依赖
需 GPT-4o API Key

成熟度
**8/10**

**优势**：精确度极高（远超截图方案），微软官方维护，文档完善。
**劣势**：强依赖 GPT 模型 API，仅支持 Windows。

### 方案二：OmniParser v2（视觉解析中间件）

[OmniParser](https://github.com/microsoft/OmniParser) 同样来自微软，但定位不同——它不是完整的 Agent，而是**屏幕解析中间件**：把截图转换为结构化的 UI 元素描述。

工作链路：截图 → YOLOv8 图标检测 + OCR 文本识别 → 元素聚合 + 坐标映射 → 结构化 JSON → 交给任意 LLM 决策。

**关键定位**：OmniParser 是任何视觉 Agent 的「眼睛」，但不是完整方案。需要自行组装”截图采集 → OmniParser 解析 → LLM 决策 → 输入注入”的完整链路。

纯视觉方案的好处是**跨平台**，但劣势是需要自己搭建完整的基础设施，且截图精度不如 UIA。

成熟度：**7/10**（作为组件优秀，作为方案需大量集成工作）。

### 方案三：Anthropic Computer Use

Claude 的 Computer Use 通过”反复截图 → 分析 → 输出操作指令 → 执行 → 再截图验证”的循环驱动。

问题在于：**官方参考实现是 Linux Docker 容器**，不原生支持 Windows 桌面控制。要适配 Windows，需要自建 VNC 桥接或截图+执行服务。

维度
评估

延迟
高（10-20s/步）

精确度
⭐⭐⭐（截图不如 UIA）

工程量
大量自建基础设施

适合需要 Claude 高级推理能力的复杂场景，但对大多数 Windows 操作来说”杀鸡用牛刀”。

### 方案四：OpenAI Operator / CUA

OpenAI 的 CUA 在**云端虚拟环境**中操作，不是控制用户本地的 Windows。对”远程控制本地 Windows 桌面”这一需求基本不适用。排除。

### 方案五：其他开源项目

- **UI-TARS**（字节跳动）：端到端视觉 GUI Agent，开源模型可本地部署，平台无关。模型能力强但工程化需自建。**6/10**

- **OS-Copilot / FRIDAY**：学术研究项目，主要面向 Linux/macOS。**5/10**

- **OpenAdapt**：RPA 录制回放 + LLM，早期项目。**4/10**

- **Self-Operating Computer**：概念验证级别。**4/10**

## 核心对比：五种远程控制通道

把视角从”用哪个 Agent”切换到”用什么远程通道”，对比更清晰：

方案
通道
延迟/步
精确度
复杂度
安全性
综合评分

**A. UFO HTTP API**
REST/HTTP
3-5s
⭐⭐⭐⭐⭐
★★☆
★★★☆
**9**

B. SSH + PowerShell
SSH
1-3s
⭐⭐⭐⭐
★★★☆
★★★★
**7**

C. VNC + 视觉 Agent
VNC
8-20s
⭐⭐⭐
★★★★
★★☆
**5**

D. 自建截图+执行 HTTP
HTTP
5-10s
⭐⭐⭐
★★★★★
★★★☆
**5**

E. RDP + 工具链
RDP
8-15s
⭐⭐⭐
★★★★★
★★★☆
**4**

## 最终决策：分层控制策略

单一方案无法覆盖所有场景。最终采用了**分层策略**——按操作复杂度选择不同通道：

```nestedtext

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

Layer 3: 复杂 GUI 交互（操作 Excel、配置系统设置）
   ────▶ UFO Agent (HTTP API) — UIA 精确操作

Layer 2: 简单 GUI 操作（启动程序、窗口管理）
   ────▶ SSH + PowerShell (Start-Process 等)

Layer 1: 命令行/脚本操作（文件管理、系统查询）
   ────▶ SSH + PowerShell + 原生命令

Layer 0: 浏览器操作（已有能力）
   ────▶ CDP (Chrome DevTools Protocol)

```

**为什么分层而不是全用 UFO？**

- SSH 方案**零成本、立即可用**（Windows 自带 OpenSSH Server，只需在可选功能中开启），适合 80% 的命令行操作

- UFO 需要 GPT API Key 和 Python 环境部署，但 GUI 交互场景精度最高

- CDP 浏览器控制已就绪，无需重复建设

**落地路线**：

```subunit

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

Phase 1 (1-2天): SSH 基础通道
├── Windows 启用 OpenSSH Server
├── 配置 SSH 密钥认证
├── 验证文件操作、进程管理

Phase 2 (3-5天): UFO Agent 集成
├── Windows 安装 UFO + GPT-4o 配置
├── 启动 REST API 服务
├── 验证 GUI 自动化

Phase 3 (1-2周): 统一抽象层
├── SSH + UFO + CDP 自动路由
├── 错误恢复 + 操作审计

```

## 选型决策树

```ruby

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

                 需要控制 Windows？
               ┌──── 是 ────┐
               ▼            ▼
         需要GUI交互?    仅命令行操作?
             │              │
             ▼              ▼
       ┌─────────┐    SSH + PowerShell
       │ 预算充足?│    (零成本，立即可用)
       │(GPT API)│
       └────┬────┘
     ┌──────┴──────┐
     ▼             ▼
  是(GPT)       否(本地模型)
     │             │
     ▼             ▼
┌─────────┐   ┌─────────────┐
│ UFO     │   │ UI-TARS +   │
│ Agent   │   │ 自建截图/   │
│ (HTTP)  │   │ 执行桥接    │
└─────────┘   └─────────────┘

```

## 五条核心结论

- 

**UFO Agent 是 Windows GUI 控制的最佳选择**：微软官方、MIT 开源、原生 UIA 精确度远超截图方案、提供 HTTP API 可被 Linux 远程调用。

- 

**分层策略优于单一方案**：SSH 处理命令行（快、稳、免费），UFO 处理 GUI（精确但需 API 费用），CDP 处理浏览器（已有）。

- 

**OmniParser 是有价值的补充**：当 UIA 无法识别非标准 UI（Canvas 渲染、游戏内界面）时，视觉方案作为兜底。

- 

**Anthropic/OpenAI 方案不推荐作为主力**：均为截图+视觉方案，延迟高、精度不如 UIA、需大量自建基础设施。

- 

**SSH 方案立即可落地**：仅需在 Windows 上启用 OpenSSH Server，今天就能验证文件操作和系统管理能力。

---

> 

当一个 AI Agent 需要跨操作系统工作时，正确的做法不是寻找一个”万能方案”，而是为每类操作选择最合适的通道，再用统一抽象层串联起来。分层，才是远程控制的正确打开方式。