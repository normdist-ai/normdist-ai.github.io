---
title: 'AutoQuant 开发周报 - 2026/05/24'
date: 2026-05-24 10:00:00
tags: [AutoQuant, 开发周报, 量化交易, v2.0]
categories: [技术笔记]
---


## 📊 本周数据概览

指标
数值

Git 提交
2 次

文件修改
63 个

代码新增
9,653 行

代码删除
496 行

版本升级
v1.7.0 → v2.0.0

## 🎯 重点改进

### 1. 架构重构：从 Streamlit 单体到 FastAPI + React SPA

本周最大的里程碑是 AutoQuant 完成了从 v1.x（Streamlit 单体应用）到 v2.0（前后端分离架构）的全面迁移。

旧架构痛点：

- Streamlit 服务端渲染，每次交互需重新执行脚本

- 前端交互能力受限，用户体验停留在数据展示层面

- 部署运维复杂，前后端耦合紧密

新架构优势：

- FastAPI REST API（端口 8100）：提供约 20 个标准化 REST 端点

- React + Vite SPA：9 个前端页面实现完整交互体验，深色主题设计

- 同端口部署：FastAPI 和前端共享 8100 端口

### 2. 数据源重构：ReShare 8200 为主力的降级链

将主力数据源从 AkShare 切换到 ReShare 8200，构建完整降级链路：

- P1（主力）：ReShare 8200 - 本地部署，毫秒级响应

- P2：yfinance - 国际数据覆盖广

- P3：AkShare - A股数据丰富

- P4：Mock - 开发测试兜底

### 3. 技术指标模块：10 个核心指标

新增 app/indicators/ 目录，实现 10 个主流技术分析指标：

- 趋势类：SMA、EMA

- 动量类：RSI、MACD、KDJ、Stochastic

- 波动类：布林带(Bollinger Bands)、ATR

- 量能类：OBV、VWAP

### 4. 回测引擎与 AI 信号系统

模块
代码量
功能

backtest_engine.py
366行
历史数据回测框架，自动计算收益率/夏普比率/最大回撤

signal_engine.py
194行
AI驱动的交易信号生成

auto_trader.py
222行
定时策略执行 + 风控（止损/止盈）

notification.py
384行
多渠道消息推送系统

### 5. 每日自动分析脚本

daily_trade.py（316行）实现自动化每日扫描、技术面分析和交易决策。

## 🐛 遇到的问题

### BacktestEngine 字符串键 Bug（已修复）

改进报告中发现的严重问题：backtest_engine.py 中多处使用变量名而非字符串字面量访问 DataFrame，导致 NameError。v2.0 提交中通过定义列常量全面修复。

## 📈 性能提升

维度
v1.x
v2.0

前端交互
Streamlit 重渲染
SPA 即时响应

API 响应
无 REST API
~20个标准化端点

数据源速度
AkShare（网络延迟）
ReShare 8200（本地毫秒级）

## 🎯 下周计划

-  API 密钥加密存储（P0 优先级）

-  单元测试体系建设（从 0% 覆盖率起步）

-  前端深色主题优化

## 💡 经验总结

- **架构升级要果断**：Streamlit 适合快速原型，功能丰富后前后端分离是必然选择。v2.0 迁移换来长远的可维护性和扩展性。

- **数据源本地化是关键**：ReShare 8200 将 API 调用延迟从数百毫秒降到毫秒级。

- **指标模块化价值大**：10个技术指标拆分为独立文件，每个模块可单独测试和扩展。

- **改进报告驱动质量提升**：通过持续改进报告系统性追踪问题，确保发布后仍有明确的质量提升方向。