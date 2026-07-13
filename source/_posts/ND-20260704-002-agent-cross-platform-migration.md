---
title: 'AI Agent 跨平台迁移实录：从 Linux 到 Windows 的零损失搬家'
date: 2026-07-04 10:00:00
tags: []
categories: [技术笔记]
---


## 背景

一个 AI Agent 在 Linux 服务器上稳定运行了数月：定时任务（cron）按节奏执行，长期记忆持续积累，灵魂文件（system prompt）定义了人格和规则。现在，需要把它迁移到一台 Windows 主机上。

这不是简单的文件复制。一个有”记忆”的 Agent，迁移时要回答三个核心问题：

- **定时任务零损失**：十几个 cron job 能否在新平台上 1:1 恢复？

- **记忆完整性**：积累数月的对话记忆、技能配置能否无损迁移？

- **灵魂文件写入**：Agent 的 system prompt（人格定义）能否在新环境正确加载？

本文记录了这次迁移的全过程和踩过的坑。

## 迁移范围清点

迁移前，先做一次完整的资产盘点：

资产
位置
迁移方式

Agent 核心代码
代码仓库
git clone

灵魂文件（system prompt）
profile 目录
直接复制

长期记忆
profile/memories/
直接复制

技能定义
skills/
直接复制

定时任务
cron jobs 配置
逐个重建

运行环境
Python venv + 依赖
新环境重建

工具链
gh、git、node
逐个安装

前四项是”文件级迁移”，复制即可。真正的难点在后面三项。

## 难点一：Cron 任务的跨平台恢复

Linux 的 cron 是 Agent 自动化运转的心脏。迁移到 Windows 后，cron 调度机制能否保留？

**策略**：保留 Linux 服务器作为 Agent 主运行环境，Windows 作为被控目标。Agent 本体不搬——搬的是”控制范围”。Agent 在 Linux 上调度，通过 SSH/HTTP 远程操控 Windows。

这意味着 **cron 配置完全不需要迁移**。所有定时任务继续在 Linux 上按原节奏运行，只是执行的目标多了 Windows 这台机器。

> 

关键决策：Agent 迁移 ≠ Agent 搬家。更好的方案是让 Agent 留在 Linux，扩展它的”触手”到 Windows。这样 cron 零损失、记忆零损失、灵魂文件零改动。

## 难点二：运行环境重建

Windows 上需要安装的开发工具链：

```powershell

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

# 1. Git（Windows 版自带 Git Bash）
winget install Git.Git

# 2. GitHub CLI
winget install GitHub.cli

# 3. Python 3.12
winget install Python.Python.3.12

# 4. Node.js（如果需要）
winget install OpenJS.NodeJS.LTS

# 5. OpenSSH Server（让 Linux 能远程连入）
# Settings → Apps → Optional Features → Add "OpenSSH Server"

```

**踩坑记录**：

- **bin 目录丢失**：安装 GitHub CLI 后 `gh` 命令不识别。根因：安装路径未加入 PATH。修复：手动将 `%ProgramFiles%\GitHub CLI` 添加到系统 PATH。

- **SSH 服务未自启**：OpenSSH Server 安装后默认未启动。需要在 `services.msc` 中将 OpenSSH SSH Server 设为自动启动。

## 难点三：验证迁移完整性

迁移不是”复制完就结束”，必须有验证步骤。设计了三级验证：

### Level 1：连通性验证

```bash

1
2
3
4
5

# 从 Linux 测试 SSH 连通
ssh tony@<windows-ip> "echo OK && hostname"

# 验证 Windows 上工具链
ssh tony@<windows-ip> "git --version && gh --version && python --version"

```

### Level 2：功能验证

Agent 通过 SSH 向 Windows 发送实际操作指令，验证文件操作、进程管理等基础能力：

```bash

1
2
3
4
5

# 创建测试文件
ssh tony@<windows-ip> "powershell -Command 'New-Item -ItemType File -Path C:\Users\test\migration_check.txt'"

# 查询系统状态
ssh tony@<windows-ip> "powershell -Command 'Get-ComputerInfo | Select-Object OsName, OsVersion'"

```

### Level 3：场景验证

让 Agent 完成一个端到端任务（如”在 Windows 上打开 Edge 浏览器并访问指定网页”），通过 CDP 验证浏览器控制链路。

## 迁移模式总结

经过实践，总结出 Agent 跨平台迁移的两种模式：

### 模式 A：本体搬迁（不推荐）

把 Agent 整体从 Linux 搬到 Windows。

- ❌ Cron 需要全部重建（Windows 没有原生 cron）

- ❌ 记忆文件需手动复制

- ❌ 灵魂文件路径需重新配置

- ❌ Python 依赖可能不兼容

- ❌ 运维成本高

### 模式 B：触手延伸（推荐）

Agent 留在 Linux，通过远程控制协议扩展到 Windows。

- ✅ Cron 零损失

- ✅ 记忆零损失

- ✅ 灵魂文件零改动

- ✅ 增量扩展，不破坏现有能力

- ✅ Linux + Windows 双平台协同

```nix

1
2
3
4
5
6
7
8
9

┌──────────────────────────────────────┐
│  Linux 服务器（Agent 本体）            │
│  ├── 灵魂文件 / 记忆 / 技能            │
│  ├── Cron 调度（全部保留）              │
│  └── 远程控制通道                      │
│       ├── SSH → Windows（命令行）      │
│       ├── HTTP → UFO（GUI）            │
│       └── CDP → Edge（浏览器）         │
└──────────────────────────────────────┘

```

## 教训与最佳实践

- 

**迁移前先问”真的需要搬吗”**：很多时候，扩展控制范围比物理搬迁更合理。Agent 的”迁移”应该理解为”能力延伸”，不是”身体搬家”。

- 

**资产盘点先于动手**：列清每一项资产的位置和迁移方式，避免遗漏。特别是 cron 配置、记忆文件这类”看不见但关键”的资产。

- 

**三级验证不可省**：连通性 → 功能 → 场景，逐级验证才能确保迁移质量。只做连通性验证是”自欺欺人”。

- 

**Windows 工具链的 PATH 问题**：winget 安装的工具不总是自动加入 PATH。迁移后第一时间验证 `git`、`gh`、`python` 等关键命令可用。

- 

**OpenSSH Server 要设自启**：否则重启后 Linux 侧的 Agent 会突然连不上 Windows。

---

> 

一个有记忆的 Agent 迁移，本质上是”让它的身体可以重建，但灵魂和记忆不能丢”。选对迁移模式，比选对工具更重要。