---
title: '通用网页项目原型设计规范'
date: 2026-04-04 10:00:00
tags: []
categories: [技术笔记]
---


## 📋 目录

- [概述](#%E6%A6%82%E8%BF%B0)

- [目录结构](#%E7%9B%AE%E5%BD%95%E7%BB%93%E6%9E%84)

- [访问方式](#%E8%AE%BF%E9%97%AE%E6%96%B9%E5%BC%8F)

- [工作流程](#%E5%B7%A5%E4%BD%9C%E6%B5%81%E7%A8%8B)

- [设计规范](#%E8%AE%BE%E8%AE%A1%E8%A7%84%E8%8C%83)

- [文件命名](#%E6%96%87%E4%BB%B6%E5%91%BD%E5%90%8D)

- [版本管理](#%E7%89%88%E6%9C%AC%E7%AE%A1%E7%90%86)

- [最佳实践](#%E6%9C%80%E4%BD%B3%E5%AE%9E%E8%B7%B5)

---

## 概述

### 目的

建立一套独立于正式代码的原型设计系统，支持快速迭代和多方案对比，确保原型开发不影响现有项目代码。

### 原则

- **完全隔离**：原型代码与正式代码物理分离

- **快速预览**：通过浏览器直接访问原型页面

- **多方案对比**：一次性提供多个设计方案供选择

- **灵活部署**：选定方案后可快速集成到正式项目

---

## 目录结构

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
10
11
12
13
14
15
16
17
18

项目根目录/
├── .prototype/                # 原型设计专用文件夹 ⭐ 推荐
│   ├── prototype/              # 原型资源目录
│   │   ├── index.html          # 原型导航首页
│   │   ├── login/              # 登录页面原型
│   │   │   ├── login-1.html    # 方案 1
│   │   │   ├── login-2.html    # 方案 2
│   │   │   └── ...
│   │   └── assets/             # 原型公共资源
│   └── serve_prototype.py      # 原型服务器脚本（可选）
│
├── public/                    # 静态资源目录 (备选方案)
│   └── prototype/              # 原型资源目录
│       ├── index.html          # 原型导航首页
│       └── login/              # 登录页面原型
│
└── docs/
    └── PROTOTYPE_GUIDE.md      # 本文档

```

---

## 访问方式

### 方案 A：独立端口（推荐）⭐

使用独立的 HTTP 服务器，完全隔离于开发服务器。

**启动命令 (Python):**

```bash

1
2

cd .prototype
python -m http.server 5174

```

**访问地址:**

```awk

1
2
3
4

http://localhost:5174                    # 导航页
http://localhost:5174/login/login-1.html  # 登录方案 1
http://localhost:5174/login/login-2.html  # 登录方案 2
...

```

**优势:**

- ✅ 完全独立，不受开发服务器影响

- ✅ 可以并行运行开发和原型设计

- ✅ 更接近生产环境的静态文件服务

- ✅ 自动禁用缓存，方便调试

---

### 方案 B：开发服务器 Public 目录

将原型文件放在项目的 `public` 目录下。

**访问地址:**

```awk

1
2

http://localhost:3000/prototype/index.html
http://localhost:3000/prototype/login/login-1.html

```

**优势:**

- ✅ 只需启动一个服务器

- ✅ 开发服务器自动处理静态资源

- ✅ 无需额外配置

**劣势:**

- ⚠️ 与正式项目共享端口

- ⚠️ 可能受到热更新影响

---

## 工作流程

### 1. 需求沟通

- 明确设计目标

- 确定功能要点

- 约定风格偏好

### 2. 方案设计

设计师（AI 或人工）根据需求设计多个方案（建议 3-10 个）。

### 3. 原型实现

在 `.prototype/prototype` 目录下创建独立的 HTML 文件。

### 4. 评审选择

- 通过浏览器预览所有方案

- 对比不同设计风格

- 选择最满意的方案

### 5. 正式集成

将选定的方案集成到正式项目中：

- 转换为 React/Vue/Angular 组件

- 添加业务逻辑

- 接入 API 接口

- 代码审查和测试

### 6. 清理归档

- 保留选定的原型作为参考

- 删除未选定的原型（或移动到 archive 目录）

---

## 设计规范

### HTML 结构要求

每个原型 HTML 文件应该是**自包含的**，包含所有必要的 CSS 和 JavaScript。

```html

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

<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>页面标题 - 方案 X</title>

    <!-- 使用 CDN 引入公共库 -->
    <script src="https://cdn.tailwindcss.com"></script>

    <!-- 自定义样式 -->
    <style>
        /* 方案特定的样式 */
    </style>
</head>
<body>
    <!-- 页面内容 -->

    <!-- 交互脚本 -->
    <script>
        // 模拟数据和交互逻辑
    </script>
</body>
</html>

```

### 技术要求

#### CSS 框架

- **推荐**: Tailwind CSS (CDN)

- **备选**: Bootstrap, Bulma

- **禁止**: 依赖本地构建的样式文件

#### JavaScript 库

- **推荐**: Vanilla JS (原生)

- **可选**: Alpine.js, Vue.js (CDN)

- **避免**: React/Angular (需要构建)

#### 图标库

- **推荐**:

- Heroicons (SVG inline)

- Font Awesome (CDN)

- Remix Icon (CDN)

#### 字体

- **推荐**: 系统默认字体栈

- **可选**: Google Fonts (CDN)

### 数据模拟

使用硬编码的模拟数据，不依赖真实 API：

```javascript

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

const mockData = {
    user: {
        username: 'admin',
        email: 'admin@example.com'
    },
    items: [
        { id: 1, name: '项目 A', status: 'active' },
        { id: 2, name: '项目 B', status: 'pending' }
    ]
};

```

### 交互模拟

模拟表单提交、页面跳转等交互，但不真正调用后端：

```javascript

1
2
3
4
5
6
7

// 模拟登录
function handleLogin(username, password) {
    // 显示加载状态
    // 延迟 1 秒模拟网络请求
    // 返回成功/失败
    // 跳转到原型页面
}

```

---

## 文件命名

### 命名规则

```d

1

{module}-{version}.html

```

- **module**: 页面/功能模块名称（小写，连字符分隔）

- **version**: 版本号（数字，从 1 开始）

### 示例

✅ **正确:**

- `login-1.html`

- `dashboard-2.html`

- `user-profile-1.html`

- `product-list-3.html`

❌ **错误:**

- `Login-1.html` (大写)

- `login_v1.html` (下划线)

- `login1.html` (缺少连字符)

- `test.html` (无意义名称)

### 版本递增

每个新方案递增版本号：

- 第一个方案：`login-1.html`

- 第二个方案：`login-2.html`

- 第三个方案：`login-3.html`

---

## 版本管理

### Git 配置

在 `.gitignore` 中添加：

```plaintext

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

# ❌ 不要忽略原型文件 - 需要持久保留
# .prototype/prototype/*.html

# 或者只忽略临时测试文件
.prototype/prototype/**/test-*.html
.prototype/prototype/**/*.tmp

# 允许提交所有正式原型文件
!.prototype/prototype/index.html
!.prototype/prototype/login/*.html
!.prototype/prototype/dashboard/*.html

```

### 保留策略 ⭐⭐⭐

#### 方案 A：完全保留（推荐）✅

**保留所有原型文件作为设计资产:**

- ✅ 所有探索性设计方案

- ✅ 选定的最终方案 (标记为 `selected` 或 `final`)

- ✅ 有代表性的备选方案

- ✅ 实验性的创新设计

**优点:**

- 完整的设计历史记录

- 方便未来回顾和参考

- 展示设计决策过程

- 避免重复劳动

**文件组织:**

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

.prototype/prototype/
├── README.md                    # 说明文档
├── index.html                   # 导航首页
├── login/
│   ├── README.md               # 登录页面设计说明
│   ├── login-1.html            # 现代简约 ⭐ 选定方案
│   ├── login-2.html            # 商务专业
│   ├── login-3.html            # 科技感
│   ├── login-4.html            # 极简主义
│   └── ...                     # 保留所有方案
└── dashboard/
    └── ...

```

#### 方案 B：选择性保留

只保留代表性方案，删除明显不合适的早期尝试。

**保留清单:**

- ✅ 最终选定的方案 (必须保留)

- ✅ 最有竞争力的 2-3 个备选方案

- ✅ 最具创意的设计尝试

- ❌ 明显的失败尝试 (可删除)

- ❌ 重复度高的微调版本 (可合并)

#### 方案 C：归档策略

对于长期未使用的原型，可以移动到归档目录:

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

.prototype/
├── prototype/                  # 活跃原型
│   ├── login/
│   └── dashboard/
└── archive/                    # 历史归档
    ├── 2026-Q1/
    │   └── login-designs/     # 2026 年 Q1 的登录设计
    └── 2026-Q2/
        └── old-concepts/      # 旧的设计理念

```

### 推荐策略 ⭐⭐⭐

**采用「完全保留」策略:**

- **所有原型文件都应该保留** - 它们是团队的设计资产

- **使用注释标记重要信息** - 在 HTML 中添加设计说明

- **定期整理但不删除** - 可以移动位置，但不要删除

- **Git 提交时保留历史** - 每次设计迭代都提交到 Git

```html

1
2
3
4
5
6
7
8

<!--
  设计方案：login-2.html - 商务专业风格
  创建日期：2026-04-04
  设计师：AI Assistant
  状态：备选方案 (未被选中)
  特点：左右分屏布局，深蓝色主题，适合传统企业用户
  备注：虽然未被选中，但分屏设计理念值得参考
-->

```

---

## 最佳实践

### 1. 保持简洁

原型应该专注于视觉设计和交互流程，不要过度复杂化。

✅ **推荐:**

- 单个 HTML 文件控制在 500 行以内

- 使用内联样式和脚本

- 模拟 2-3 个典型场景

❌ **避免:**

- 拆分成多个文件

- 实现完整业务逻辑

- 处理所有边界情况

### 2. 响应式设计

确保原型在不同设备上都能正常显示：

```html

1

<meta name="viewport" content="width=device-width, initial-scale=1.0">

```

使用 Tailwind 的响应式类：

```html

1

<div class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-4">

```

### 3. 可访问性

遵循基本的可访问性原则：

```html

1
2
3
4
5
6
7
8
9

<!-- 语义化标签 -->
<header>, <nav>, <main>, <footer>

<!-- 表单标签 -->
<label for="username">用户名</label>
<input id="username" type="text">

<!-- Alt 文本 -->
<img src="logo.png" alt="项目 Logo">

```

### 4. 性能优化

- 使用 CDN 资源

- 压缩内联资源（可选）

- 避免大尺寸图片

### 5. 注释说明

为复杂的设计决策添加注释：

```html

1
2
3
4
5
6
7
8

<!--
  设计方案说明：
  - 风格：现代简约
  - 配色：蓝色主题
  - 特点：卡片式布局，大字体

  适用场景：现代化企业系统
-->

```

### 6. 统一风格 ⭐

**推荐**: 原型中心首页使用**侧边栏导航式**布局。

**原因:**

- ✅ 专业性强，符合企业级系统定位

- ✅ 信息架构清晰，易于理解

- ✅ 易于扩展新模块

- ✅ 与正式项目布局一致

**参考实现:** 见 `.prototype/prototype/index.html`

### 7. 持久保留原则 ⭐⭐⭐

**重要：** 所有原型设计方案应该**持久保留**,即使已经选定了最终方案。

**保留原因:**

- 📚 **设计过程记录** - 保留完整的设计探索过程

- 🔄 **未来参考** - 后续迭代时可以参考历史方案

- 🎨 **灵感来源** - 不同方案可能在未来项目中发挥作用

- 📊 **对比价值** - 便于向新成员展示决策过程

- 🔍 **快速验证** - 需求变更时可以快速回顾备选方案

---

## 原型导航页

创建 `index.html` 作为所有原型的入口。

### 推荐风格：侧边栏导航式 ⭐

**设计理念:**

- 左侧固定导航栏，清晰的信息架构

- 右侧内容区域，展示原型卡片网格

- 专业感强，符合企业级系统定位

- 易于扩展到更多模块

**布局结构:**

```gams

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

┌─────────────────────────────────────┐
│  Sidebar  │    Main Content Area    │
│           │                         │
│  - Logo   │  Welcome Banner         │
│  - 登录页  │  ┌──────┬──────┬──────┐│
│  - 仪表盘  │  │ Card │ Card │ Card ││
│  - 用户页  │  ├──────┼──────┼──────┤│
│  - 设置页  │  │ Card │ Card │ Card ││
│           │  └──────┴──────┴──────┘│
│  [Footer] │  [Coming Soon Modules]  │
└───────────┴─────────────────────────┘

```

**核心特点:**

- **侧边导航** - 使用毛玻璃效果的白色侧边栏

- **渐变背景** - 紫色到紫红色的渐变主题（或根据项目定制）

- **卡片网格** - 响应式网格布局展示各个方案

- **悬停动画** - 卡片悬停时的提升效果

- **标签标识** - 使用标签显示方案数量和状态

**示例代码结构:**

```html

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
41
42
43
44
45
46
47
48
49
50
51
52
53
54
55
56
57
58
59
60
61
62

<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta charset="UTF-8">
    <title>原型设计中心</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <style>
        body {
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
        }
        .sidebar {
            background: rgba(255, 255, 255, 0.95);
            backdrop-filter: blur(10px);
        }
        .nav-item:hover {
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
            color: white;
            transform: translateX(8px);
        }
    </style>
</head>
<body class="min-h-screen flex">
    <!-- 左侧导航栏 -->
    <aside class="sidebar w-72 shadow-2xl">
        <div class="p-8 border-b">
            <h1 class="text-2xl font-bold">项目名称</h1>
            <p class="text-sm text-gray-600">原型设计中心</p>
        </div>

        <nav class="p-4 space-y-2">
            <a href="#login" class="nav-item active block px-4 py-3 rounded-lg">
                🔐 登录页面 <span class="ml-auto text-xs">10 个方案</span>
            </a>
            <a href="#dashboard" class="nav-item block px-4 py-3 rounded-lg">
                📊 仪表盘 <span class="ml-auto text-xs">即将推出</span>
            </a>
        </nav>
    </aside>

    <!-- 右侧内容区 -->
    <main class="flex-1 p-8 overflow-auto">
        <div class="max-w-6xl mx-auto">
            <!-- 欢迎区域 -->
            <div class="mb-8">
                <h2 class="text-4xl font-bold text-white">欢迎使用原型设计中心</h2>
            </div>

            <!-- 原型卡片网格 -->
            <div class="bg-white rounded-2xl shadow-xl p-8">
                <h3 class="text-2xl font-bold mb-6">🔐 登录页面系列</h3>
                <div class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-6">
                    <a href="login/login-1.html" class="group block bg-gradient-to-br from-purple-50 to-indigo-50 rounded-xl p-6 hover:shadow-lg transition-all transform hover:-translate-y-1">
                        <h4 class="font-semibold text-gray-900">现代简约</h4>
                        <p class="text-sm text-gray-600">紫色渐变 + 毛玻璃效果</p>
                    </a>
                    <!-- 更多卡片... -->
                </div>
            </div>
        </div>
    </main>
</body>
</html>

```

---

## 从原型到正式代码

### 转换步骤

- **创建组件**
```javascript

1
2
3
4
5
6

// React 示例
import { useState } from 'react';

export default function Login() {
    // 基于选定原型的结构和逻辑
}

```

```plaintext

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

<!-- Vue 示例 -->
<template>
    <!-- 基于选定原型的 HTML 结构 -->
</template>

<script>
export default {
    // 基于选定原型的逻辑
}
</script>

```

- **迁移样式**

- Tailwind CDN → Tailwind 配置文件

- 内联样式 → CSS Modules / Styled Components

- 或保持 Tailwind CDN 开发阶段使用

- **替换模拟数据**

- Mock Data → API 调用

- 硬编码 → 状态管理

- **添加路由**

- 直接跳转 → React Router / Vue Router / Angular Router

- 页面刷新 → SPA 导航

- **集成测试**

- 功能测试

- 兼容性测试

- 性能优化

---

## 工具和资源

### 推荐工具

- **设计灵感**: Dribbble, Behance, Pinterest

- **配色方案**: Coolors, Adobe Color

- **图标资源**: Heroicons, Feather Icons

- **字体搭配**: Google Fonts, Fontshare

### CDN 资源

```html

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

<!-- Tailwind CSS -->
<script src="https://cdn.tailwindcss.com"></script>

<!-- Alpine.js (可选) -->
<script defer src="https://cdn.jsdelivr.net/npm/alpinejs@3.x.x/dist/cdn.min.js"></script>

<!-- Font Awesome -->
<link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.4.0/css/all.min.css">

<!-- Google Fonts -->
<link href="https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600;700&display=swap" rel="stylesheet">

```

---

## 质量控制清单

### 原型检查项

-  HTML 结构语义化

-  响应式布局正常

-  交互逻辑流畅

-  配色方案协调

-  字体大小合适

-  图标清晰

-  无明显 bug

-  文件大小合理 (< 1MB)

### 集成检查项

-  组件拆分合理

-  类型定义完整（TypeScript）

-  API 接口对接正确

-  样式无冲突

-  性能指标达标

-  跨浏览器兼容

-  单元测试通过

---

## 变更日志

版本
日期
变更内容
作者

1.0
2026-04-04
初始版本，建立通用网页项目原型设计工作规范
xiaorui

---

## 总结

本文档提供了一套通用的网页项目原型设计规范，适用于各种前端项目。通过建立独立于正式代码的原型设计系统，支持快速迭代和多方案对比，同时确保所有设计方案都能持久保留作为团队的设计资产。

**核心价值:**

- 提高设计效率

- 降低试错成本

- 保留设计资产

- 促进团队协作

- 加速项目交付

---

**最后更新**: 2026-04-04
**维护者**: xiaorui
**版本**: 1.0.0