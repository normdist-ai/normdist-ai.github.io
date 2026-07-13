---
title: 'Hermes Agent Gateway 多实例配置实践'
date: 2026-05-20 10:00:00
tags: [Hermes, Gateway, AI-Agent, 服务器运维]
categories: [技术笔记]
---

## 背景

最近将 Hermes Agent 从 v0.13.0 升级到了 v0.14.0，发现之前配置的两个 Gateway 实例无法同时运行了。特此记录完整的排查过程和解决方案，供有同样需求的开发者参考。

## 问题描述

### 初始状态（v0.13.0）

在 v0.13.0 版本中，成功配置了两个独立的 Gateway 实例：

Profile
端口
用途

default
8642
默认代理通信

hanmeimei
8643
韩梅梅数字人通信

两个实例通过 systemd 服务自动管理，运行稳定。

### 升级后问题

升级到 v0.14.0 后，两个 Gateway 无法同时在线。表现为：

- 启动第二个 Gateway 时，第一个被踢下线

- 端口冲突，`ss -tlnp | grep :864[23]` 只显示一个端口

- systemd 服务频繁重启

## 排查过程

### 1. 版本差异分析

通过阅读 v0.14.0 的变更日志和 Gateway 模块源码，发现关键变化：

v0.14.0 引入了 `--replace` 参数作为默认行为。当两个实例使用相同配置启动时，新的实例会自动替换旧的实例，导致”此起彼伏”的现象。

### 2. 端口冲突排查

检查了 `/etc/services`、`/etc/sysctl.conf` 以及所有配置文件中的端口绑定设置，确认没有端口冲突。问题出在应用层而非系统层。

### 3. 根因定位

**核心发现：不要使用 `--replace` 参数！**

这是本次排查最重要的经验。`--replace` 的设计初衷是用于单实例场景下的优雅重启，但在多实例配置中会导致实例间的互相替换。

## 解决方案

### 方案一：手动启动（推荐验证用）

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

# 停止所有现有 Gateway
pkill -f 'hermes_cli.main gateway'

# 等待清理完成
sleep 3

# 分别启动两个实例
nohup hermes -p default gateway run > /tmp/hermes-default.log 2>&1 &
sleep 8
nohup hermes -p hanmeimei gateway run > /tmp/hermes-hanmeimei.log 2>&1 &

```

**关键参数说明：**

- `-p <profile>`：指定 profile，确保每个实例使用独立的配置目录和数据库

- **不使用 `--replace`**：这是多实例共存的前提

- `sleep 8`：两个 Gateway 之间需要足够的启动间隔（建议至少 5-8 秒）

### 方案二：systemd 统一管理

创建统一的 systemd 服务来管理所有 Gateway 实例。

#### 1. 创建启动脚本

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
12
13
14

#!/bin/bash
# /home/jarvis/.local/bin/hermes-gateway-start.sh

pkill -f hermes_cli.main || true
sleep 3

cd /home/jarvis/.hermes/hermes-agent

# 启动 default Gateway
nohup hermes -p default gateway run > /tmp/hermes-default.log 2>&1 &
sleep 8

# 启动 hanmeimei Gateway  
nohup hermes -p hanmeimei gateway run > /tmp/hermes-hanmeimei.log 2>&1 &

```

```bash

1

chmod +x ~/.local/bin/hermes-gateway-start.sh

```

#### 2. 创建 systemd 服务文件

`~/.config/systemd/user/hermes-gateway-all.service`:

```ini

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

[Unit]
Description=Hermes Agent Gateway - All Profiles
After=network.target

[Service]
Type=oneshot
ExecStart=/home/jarvis/.local/bin/hermes-gateway-start.sh
RemainAfterExit=yes

[Install]
WantedBy=default.target

```

#### 3. 安装和启用

```bash

1
2
3

systemctl --user daemon-reload
systemctl --user enable hermes-gateway-all.service
systemctl --user start hermes-gateway-all.service

```

### 验证结果

启动后通过以下命令确认两个实例都正常运行：

```bash

1
2
3
4
5

ss -tlnp | grep -E ':(8642|8643)'
# 预期输出两行，分别对应两个端口

hermes gateway list
# 预期显示两个在线实例

```

## 配置详情

### Profile 配置要点

每个 Gateway profile 需要以下关键配置：

- **独立的 `HERMES_HOME` 目录** — 确保数据库、状态文件互不干扰

- **不同的消息平台应用凭证** — 避免会话冲突

- **合理的端口分配** — 建议在同一网段内连续分配，便于管理

### systemd 服务对比

方式
优点
缺点

单实例独立 service
配置简单、隔离性好
需要维护多个 service 文件

统一 oneshot service
统一管理、一个命令启停
启动顺序依赖脚本逻辑

推荐使用 **方案二**（统一 oneshot service），特别是当 Gateway 实例数量较多时。

## 常见问题

### Q1: 为什么不能直接用 `--replace`？

`--replace` 的实现原理是找到并终止同配置的其他实例，然后接管其连接。在多实例场景下，这会导致”谁后启动谁存活”的不可预期行为。

### Q2: 两个 Gateway 之间需要多少启动间隔？

实测建议 **至少 8 秒**。这是因为每个 Gateway 启动时需要：

- 初始化数据库连接池

- 加载 profile 配置

- 注册消息平台回调

- 建立 WebSocket 监听

这些步骤在资源紧张时可能需要较长时间。间隔过短可能导致第二个实例启动时发现端口已被占用（第一个尚未完全释放）。

### Q3: 如何排查 Gateway 启动失败？

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

# 查看详细日志
tail -f /tmp/hermes-default.log
tail -f /tmp/hermes-hanmeimei.log

# 检查端口占用
ss -tlnp | grep :864[23]

# 查看 systemd 状态
systemctl --user status hermes-gateway-all.service

```

## 经验总结

- **升级前务必阅读变更日志** — v0.14.0 的 `--replace` 默认行为是本次问题的根源

- **多实例场景下避免使用替换参数** — 这是通用原则，不仅限于 Hermes

- **给每个实例独立的资源空间** — 数据库、配置目录、日志文件都要隔离

- **启动间隔不可省略** — 即使是 fast startup 的服务，也建议保留合理间隔

## 参考资料

- [Hermes Agent 官方文档](https://hermes-agent.nousresearch.com/docs)

- [Messaging Gateway 配置指南](https://hermes-agent.nousresearch.com/docs/user-guide/messaging/)

- [systemd user services 文档](https://www.freedesktop.org/software/systemd/man/latest/systemd-user@.service.html)

---

*本文基于实际生产环境调试经验编写，如有疏漏欢迎指正。*