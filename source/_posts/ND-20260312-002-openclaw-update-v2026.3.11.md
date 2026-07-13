---
title: 'OpenClaw v2026.3.11 更新内容'
date: 2026-03-12 10:00:00
tags: []
categories: [技术笔记]
---

**编号**：ND-20260312-002
**版本**:1.0
**更新**: 2026-03-12
**作者**: 小瑞

本文介绍 OpenClaw v2026.3.11 版本的主要更新内容和技术改进。

## 1. Cron/Doctor 改进

### 1.1 Cron 任务隔离

**更新内容**：限制独立的 cron 交付，防止 cron 任务泄漏到其他聊天通道。

**技术细节**：

### 1.2 Doctor 迁移命令

**更新内容**：添加 `openclaw doctor --fix` 命令。

`openclaw doctor --fix`

**功能说明**：

**使用场景**：

`openclaw doctor --fix`

---

## 2. ACP 会话恢复功能

### 2.1 Resume Session ID

**更新内容**：添加可选的 `resumeSessionId` 参数，用于 runtime: “acp”。

`resumeSessionId`

**功能说明**：

**技术细节**：

`resumeSessionId`
`runtime: "acp"`

**使用场景**：

---

## 3. Discord 线程优化

### 3.1 自动归档配置

**更新内容**：添加 `autoArchiveDuration` 通道配置。

`autoArchiveDuration`

**配置选项**：

**功能说明**：

**使用场景**：

**最佳实践**：

---

## 4. Telegram 路由优化

### 4.1 DM 重启

**更新内容**：每个 agent 重启 Telegram DM，而不是每个会话键。

**问题说明**：

`agent:main:main`
`agent:main:telegram:direct:<id>`

### 4.2 Announce 交付优化

**更新内容**：将仅文本的 announce 任务通过实际的外出适配器路由，在完成后代输出后。

**问题说明**：

`delivered: true`

### 4.3 Matrix DM 路由改进

**更新内容**：为损坏的 `m.direct` homeservers 添加更安全的后备检测。

`m.direct`

**功能说明**：

---

## 5. 飞书插件 Onboarding

### 5.1 插件发现缓存清理

**更新内容**：在重新加载注册表之前清除短时插件发现缓存。

**问题说明**：

---

## 6. 版本管理

**版本信息**：

**升级建议**：

`openclaw doctor --fix`
`autoArchiveDuration`

---

## 7. 技术要点总结

更新类别
主要内容

Cron/Doctor
任务隔离、迁移命令

ACP
会话恢复、上下文保留

Discord
线程归档、配置优化

Telegram
DM 重启、交付优化

Matrix
DM 路由改进

飞书
插件 onboarding 优化

---

## 8. 升级说明

### 8.1 升级命令

`1 2`

`# 升级 OpenClaw`
`openclaw gateway update.run`

### 8.2 升级后操作

`1 2`

`# 迁移旧的 cron 配置`
`openclaw doctor --fix`

### 8.3 配置检查

`autoArchiveDuration`

---

## 9. 总结

OpenClaw v2026.3.11 是一个重要的功能更新版本。主要改进包括：

这些改进提升了系统的稳定性、可靠性和用户体验。建议所有用户及时升级到最新版本。

---

*基于 OpenClaw GitHub changelog 编写*