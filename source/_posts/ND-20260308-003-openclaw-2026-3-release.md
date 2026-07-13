---
title: 'OpenClaw2026.3 发布：AI代理能力再升级'
date: 2026-03-08 10:00:00
tags: [OpenClaw, AI代理, 安全模块, 版本更新]
categories: [技术笔记]
---

**编号**：ND-20260308-003
**版本**:1.0
**更新**: 2026-03-08
**作者**: 小瑞

**编号**：ND-20260308-002
**版本**: 1.0
**最后更新**: 2026-03-08
**维护人**: xiaorui

## 1. 概述

OpenClaw 在 2026 年 3 月发布了重要更新（版本 2026.3.2），带来了多项新功能和性能优化。本文将介绍这次更新的主要内容，帮助用户了解 OpenClaw 的最新能力。

---

## 2. 核心更新亮点

### 2.1 OpenClaw Hosting（测试版）

一键部署完全托管的 AI 代理服务，无需自行配置服务器：

### 2.2 安全加固

`config.get`

### 2.3 AI 能力扩展

功能
说明

**Kilo Gateway**
支持默认模型路由（包括 Claude Opus 4.6）

**Vercel AI Gateway**
模型简写规范化

**Moonshot/Kimi**
支持 `web_search` 和视频提供商流程

**Per-agent 参数覆盖**
支持缓存行为调优

`web_search`

---

## 3. 工作流增强

### 3.1 子代理功能

新增 `/subagents spawn` 命令，支持创建子代理进行复杂任务分解。

`/subagents spawn`

### 3.2 平台集成

平台
新增功能

**iOS**
分享扩展支持

**Slack**
实时流式传输

**Telegram**
内联按钮样式

**iMessage**
BlueBubbles 通道稳定性改进

### 3.3 监控升级

---

## 4. 记忆系统优化

### 4.1 Voyage AI 集成增强

`input_type`

### 4.2 上下文管理

---

## 5. 版本管理

### 5.1 自更新功能

OpenClaw 支持自更新：

`1 2 3 4 5`
`# 运行 /update 命令 /update # 或让代理检查更新 "check for updates"`

`# 运行 /update 命令`
`/update`

`# 或让代理检查更新`
`"check for updates"`

### 5.2 OpenClaw Reset

重新安装 OpenClaw：

---

## 6. 使用建议

### 6.1 推荐更新场景

### 6.2 注意事项

---

## 7. 总结

OpenClaw 2026.3 版本带来了显著的性能提升和安全加固，特别是：

这些改进使 OpenClaw 成为更可靠、更强大的 AI 代理平台。

---

## 8. 参考资料

**维护记录**：

日期
版本
变更内容

2026-03-08
1.0
初始版本，介绍 OpenClaw 2026.3 更新

---