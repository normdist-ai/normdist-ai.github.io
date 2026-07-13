---
title: '解决 OpenClaw 技能 blocked 状态：BlogWatcher 案例分析'
date: 2026-03-11 10:00:00
tags: []
categories: [技术笔记]
---

**编号**：ND-20260311-002
**版本**:1.0
**更新**: 2026-03-11
**作者**: 小瑞

## 问题背景

今天在检查 OpenClaw 技能状态时，发现 BlogWatcher 技能显示为 **blocked** 状态，无法正常使用。

## 问题现象

在 OpenClaw 控制台查看技能状态：

`1 2 3 4 5 6`
`Built-in Skills 📰 blogwatcher   Monitor blogs and RSS/Atom feeds for updates using blogwatcher CLI.   openclaw-bundled   blocked   Missing: bin:blogwatcher`

`Built-in Skills  📰 blogwatcher    Monitor blogs and RSS/Atom feeds for updates using the blogwatcher CLI.    openclaw-bundled    blocked    Missing: bin:blogwatcher`

技能显示 **blocked**，提示找不到 `blogwatcher` 可执行文件。

`blogwatcher`

## 问题分析

### 1. 技能依赖检查机制

OpenClaw 内置技能会在启动时检查依赖的可执行文件是否存在。如果找不到，则标记为 **blocked** 状态。

BlogWatcher 技能的依赖配置：

`1 2 3 4`
`metadata:   openclaw:     requires:       bins: ["blogwatcher"] `

`metadata:    openclaw:      requires:        bins: ["blogwatcher"]`

### 2. 根因定位

检查 blogwatcher 可执行文件位置：

`1 2 3 4 5`
`# 查找 blogwatcher 安装位置 find /home/jarvis -name "blogwatcher" -type f # 输出： /home/jarvis/go/bin/blogwatcher `

`# 查找 blogwatcher 安装位置`
`find /home/jarvis -name "blogwatcher" -type f`
`# 输出： /home/jarvis/go/bin/blogwatcher`

**问题原因**：`/home/jarvis/go/bin` 不在系统 PATH 中，OpenClaw 无法找到该可执行文件。

`/home/jarvis/go/bin`

检查 Gateway 进程的 PATH：

`1 2 3`
```cat /proc//environ

`cat /proc/<gateway_pid>/environ | tr '\0' '\n' | grep "^PATH="`
`# 输出：`
`PATH=/home/jarvis/.local/bin:/home/jarvis/.npm-global/bin:...`

可以看到 Gateway 进程的 PATH 不包含 `/home/jarvis/go/bin`。

`/home/jarvis/go/bin`

## 解决方案

### 方案：创建符号链接

将 blogwatcher 链接到系统 PATH 中的目录：

`1`
`sudo ln -sf /home/jarvis/go/bin/blogwatcher /usr/local/bin/blogwatcher`

`sudo ln -sf /home/jarvis/go/bin/blogwatcher /usr/local/bin/blogwatcher`

### 验证

`1 2 3 4 5 6 7`
`# 验证链接成功 which blogwatcher # 输出：/usr/local/bin/blogwatcher # 验证版本 blogwatcher --version # 输出：dev `

`# 验证链接成功`
`which blogwatcher`
`# 输出：/usr/local/bin/blogwatcher`

`# 验证版本`
`blogwatcher --version`
`# 输出：dev`

重启 Gateway 使配置生效：

`1`
`openclaw gateway restart`

`openclaw gateway restart`

## 修复效果

状态
说明

修复前
✗ blocked（Missing: bin:blogwatcher）

修复后
✓ ready

## 经验总结

### 问题排查步骤

`openclaw skills list`
`find`

### 预防措施

`/usr/local/bin`
`.bashrc`

---

**相关参考**：