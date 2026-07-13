---
title: 'SSH 启动桌面应用失败：Windows Session 0 隔离与跨会话启动解法'
date: 2026-07-05 10:00:00
tags: [Windows, SSH, Session 0, 跨会话启动, 远程控制, 桌面应用, 故障排查]
categories: [技术笔记]
---


## 问题：SSH 能执行命令，但启动的应用”看不见”

一个 AI Agent 通过 SSH 连接到 Windows 主机，执行 `powershell.exe Start-Process notepad.exe`，命令返回成功，任务管理器里能看到 notepad.exe 进程。但 RDP 连上去看桌面——什么都没有。记事本窗口没出现在屏幕上。

同样的问题出现在：启动 IDE、启动浏览器、启动任何带 GUI 的桌面应用。SSH 能启动进程，但进程跑在”看不见的地方”。

## 根因：Session 0 隔离

Windows 的会话（Session）模型是理解这个问题的钥匙。

### 两种会话

会话
特点
谁在里面

**Session 0**
无桌面、无 UI、无 GUI 交互
系统服务（sshd、Windows Update 等）

**Session 1+**
有桌面、有 UI、UIA 可用
登录的用户（本地登录/RDP）

用户登录 → 创建 Session 1（或更高编号）。用户注销 → Session 销毁。RDP 断开（关窗口）→ Session 保留在内存，只是非 Active 状态。

### SSH 落在哪

SSH 服务（sshd）运行在 **Session 0**。通过 SSH 执行的命令，默认继承 SSH 进程的会话——也是 Session 0。

在 Session 0 里启动 GUI 应用，应用确实会跑起来（进程存在），但它创建的窗口属于 Session 0 的桌面——那是一个**用户看不到的隐藏桌面**。所以任务管理器看得到进程，RDP 桌面看不到窗口。

这就是 Windows 的 Session 0 隔离机制：从 Vista 开始，系统服务被强制隔离到无 UI 的 Session 0，防止服务弹出窗口干扰用户桌面。这个安全设计的副作用就是——SSH 启动的 GUI 应用全部”隐身”。

## 解法：PsExec 跨会话启动

### Sysinternals PsExec64

PsExec 是 Sysinternals 工具集的一员，专门做远程/跨会话进程启动。关键参数：

```nix

1

PsExec64.exe -i 2 -d "C:\path\to\app.exe"

```

参数
含义

`-i`
interactive——在指定会话的桌面上启动（而不是 Session 0）

`2`
Session ID（用户的交互会话，通常是 1 或 2）

`-d`
不等待进程结束就返回（异步启动）

`-i 2` 是核心：告诉 PsExec 把进程启动到 Session 2 的桌面上——那是用户登录后看到的桌面。进程创建的窗口会正常显示。

### 确定目标 Session ID

Session ID 不是固定的。单用户登录通常是 Session 1，RDP 第二个会话可能是 Session 2。查询方式：

```powershell

1
2
3
4
5
6
7

# 方法一：query user
query user
# 输出：
# USERNAME SESSIONNAME ID STATE
# admin    rdp-tcp#0   2 Active

# 方法二：通过任务管理器"用户"选项卡看 Session ID

```

拿到 ID 后，在 SSH 命令中传入：

```bash

1
2

sshpass -p 'PASSWORD' ssh -T user@10.x.x.x \
  'PsExec64.exe -accepteula -i 2 -d "C:\Users\admin\app.exe"'

```

`-accepteula` 自动接受 PsExec 的许可协议（首次运行会弹窗，无人值守时必须加这个）。

### 实际案例：远程启动 IDE

需求：AI Agent 通过 SSH 在 Windows 上启动一个 IDE（Trae），让用户能在 RDP 桌面看到并交互。

```bash

1
2

sshpass -p 'PASSWORD' ssh -T 'user@10.x.x.x' \
  'PsExec64.exe -accepteula -i 2 -d "C:\Users\admin\AppData\Local\Trae\Trae.exe"'

```

启动后，RDP 连接能看到 IDE 窗口正常出现在桌面上。用户可以直接鼠标键盘操作。

## Session 0 隔离的更广泛影响

这个问题不只影响”启动 GUI 应用”。理解 Session 模型后，很多 Windows 远程操作的”诡异行为”都能解释：

### 哪些操作受 Session 影响

操作
Session 0 可用？
需要 Session 1+？

执行 PowerShell 命令
✅
—

文件读写
✅
—

网络请求
✅
—

启动 GUI 应用
❌（进程跑但窗口不可见）
✅

UIAutomation（UIA）
❌（无桌面=无 UI 元素树）
✅

截图（屏幕）
❌（截到的是黑屏）
✅

操作浏览器 Cookie
✅（直接读文件）
✅（也可以通过 CDP）

### RDP 断开 vs 注销

这是运维 Windows 远程机器最容易忽略的区别：

- **断开（Disconnect）**：关掉 RDP 窗口或锁屏。Session 保留在内存，状态 Active→Disc。所有 Session 1+ 的进程继续运行。

- **注销（Sign Out）**：Session 销毁。所有 Session 1+ 的进程被杀。需要重新登录才能恢复。

**实践建议：RDP 用完关窗口或锁屏，绝对不要注销。** 一旦注销，SSH 还在（Session 0 不受影响），但所有桌面应用、浏览器登录态、GUI 工具全部死亡，只能重新登录重建。

## 备选方案对比

PsExec 不是唯一选择，但对 AI Agent 场景最合适：

方案
原理
优点
缺点

**PsExec -i**
跨会话进程注入
简单、稳定、异步
需提前安装 Sysinternals

计划任务 + ONLOGON
登录时触发
原生支持
只在登录时触发，不能按需启动

自动登录 + 启动项
重启自动登录
重启后恢复
安全性低，不能动态启动

RDP 主动操作
模拟用户点击
最接近真人
实现复杂，需 RDP 客户端

## 总结

问题
根因
解法

SSH 启动的 GUI 应用不可见
Session 0 隔离——进程在无 UI 的系统会话
PsExec64 `-i <session_id>` 跨会话启动

浏览器自动化在注销后失效
注销销毁 Session 1+，UIA 无桌面可用
RDP 断开而非注销，保持 Session 存活

Session 0 隔离是 Windows 远程操作的核心心智模型。理解了它，90% 的”SSH 能执行命令但应用不正常”问题都能秒解。

记住一行命令：

```css

1

PsExec64.exe -accepteula -i 2 -d "path\to\app.exe"

```

`-i 2` 是灵魂——把进程送进用户看得见的桌面。