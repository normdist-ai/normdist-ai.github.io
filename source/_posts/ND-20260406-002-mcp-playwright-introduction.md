---
title: 'MCP Playwright：AI 驱动的浏览器自动化调试革命'
date: 2026-04-06 10:00:00
tags: [MCP, Playwright, AI, 浏览器自动化, 调试]
categories: [技术笔记]
---

> 

本文介绍 MCP Playwright 如何将 AI 助手与浏览器自动化结合，实现智能化的 Web 应用调试。

---

## 什么是 MCP Playwright

**MCP (Model Context Protocol)** 是一种标准化的协议，允许 AI 助手与外部工具进行交互。**Playwright** 是微软开发的现代化浏览器自动化工具。

MCP Playwright 将两者结合，让 AI 助手能够：

- 🌐 控制真实浏览器（Chrome、Edge、Firefox）

- 🔍 捕获页面元素、截图、录制视频

- 📊 监控网络请求和控制台日志

- ⚡ 执行 JavaScript 代码

- 🎯 模拟用户交互（点击、输入、滚动等）

## 核心优势

### 1. 智能化调试

AI 可以：

- 自动分析错误信息

- 智能选择最优的选择器

- 动态调整调试策略

- 自动生成调试报告

### 2. 完整的上下文感知

AI 可以看到：

- 当前页面 DOM 结构

- 控制台日志（console.log, error, warning）

- 网络请求和响应

- 页面截图和视频

- localStorage/sessionStorage 状态

### 3. 高效的迭代循环

1
2
3

发现问题 → AI 分析 → 生成修复方案 → 自动验证 → 输出报告
   ↓                                              ↑
   └──────────── 快速迭代，无需手动干预 ──────────┘

```

### 4. 可复用的调试资产

每次调试都会生成：

- 📸 关键步骤截图

- 📹 完整操作视频

- 📄 详细的 JSON 日志

- 📝 结构化的问题报告

## 环境准备

### 安装 Playwright

```bash

1
2
3
4
5
6
7

# 安装 Playwright Python 库
pip install playwright

# 安装浏览器驱动
playwright install chromium
playwright install chrome  # 可选：使用系统 Chrome
playwright install msedge  # 可选：使用系统 Edge

```

### 验证安装

```python

1
2
3
4
5
6
7
8

from playwright.sync_api import sync_playwright

with sync_playwright() as p:
    browser = p.chromium.launch(headless=True)
    page = browser.new_page()
    page.goto("https://example.com")
    print(page.title())
    browser.close()

```

预期输出：`Example Domain`

## 核心功能

### 1. 浏览器导航

```python

1
2
3
4
5
6
7
8

# 基本导航
page.goto("http://localhost:5173/login")

# 等待页面完全加载
page.wait_for_load_state("networkidle")

# 等待特定元素出现
page.wait_for_selector("#username", timeout=5000)

```

### 2. 元素定位与交互

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

# 多种选择器策略
page.locator('input[name="username"]')      # CSS 选择器（推荐）
page.locator('text=忘记密码')                 # 文本选择器
page.locator('//button[contains(text(), "登录")]')  # XPath

# 常见交互操作
page.fill('input[name="username"]', 'admin')
page.fill('input[name="password"]', 'admin123')
page.click('button[type="submit"]')

```

### 3. 截图与录制

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

# 全屏截图
page.screenshot(path='debug_01_initial.png', full_page=True)

# 元素截图
element = page.locator('#error-message')
element.screenshot(path='error_element.png')

# 录制视频（在 context 级别配置）
context = browser.new_context(
    record_video_dir='videos/',
    record_video_size={'width': 1920, 'height': 1080}
)

```

### 4. 日志捕获

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

# 控制台日志
console_messages = []

def handle_console(msg):
    console_messages.append({
        'type': msg.type,
        'text': msg.text,
        'location': f"{msg.location.get('url', '')}:{msg.location.get('lineNumber', '')}"
    })

page.on("console", handle_console)

# 网络请求监控
network_requests = []

def handle_response(response):
    network_requests.append({
        'url': response.url,
        'method': response.request.method,
        'status': response.status,
        'headers': dict(response.headers)
    })

page.on("response", handle_response)

```

### 5. JavaScript 执行

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

# 获取 localStorage
auth_data = page.evaluate("""() => {
    return {
        token: localStorage.getItem('token'),
        user: localStorage.getItem('user')
    };
}""")

# 修改页面状态
page.evaluate("""() => {
    document.body.style.backgroundColor = 'red';
}""")

```

## 实战案例：登录问题排查

### 问题背景

用户报告无法登录系统，提示”登录失败，请检查用户名和密码”。

### 调试流程

#### Step 1: 创建调试脚本

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
63
64
65
66
67
68
69
70
71
72
73

from playwright.sync_api import sync_playwright
import time
import json

def debug_login():
    with sync_playwright() as p:
        # 1. 启动浏览器
        browser = p.chromium.launch(
            headless=False,
            channel="chrome",
            slow_mo=500
        )

        context = browser.new_context(viewport={'width': 1920, 'height': 1080})
        page = context.new_page()

        # 2. 启用日志捕获
        console_messages = []
        network_requests = []

        page.on("console", lambda msg: console_messages.append({
            'type': msg.type,
            'text': msg.text
        }))

        page.on("response", lambda res: network_requests.append({
            'url': res.url,
            'status': res.status,
            'method': res.request.method
        }))

        # 3. 导航到登录页
        page.goto('http://localhost:5173/login')
        page.wait_for_load_state('networkidle')
        time.sleep(2)

        # 4. 截图：初始状态
        page.screenshot(path='debug_01_initial.png', full_page=True)

        # 5. 填写表单
        page.fill('input[name="username"]', 'admin')
        page.fill('input[name="password"]', 'admin123')
        time.sleep(1)

        # 6. 点击登录
        page.click('button[type="submit"]')
        time.sleep(3)

        # 7. 检查结果
        if '/login' in page.url:
            print("❌ 仍在登录页面，登录可能失败")
            error_element = page.locator('.bg-red-50')
            if error_element.count() > 0:
                error_text = error_element.inner_text()
                print(f"❌ 错误提示: {error_text}")
        else:
            print("✅ 已跳转到首页，登录成功")

        # 8. 保存调试信息
        debug_info = {
            'console_messages': console_messages[-20:],
            'network_requests': [r for r in network_requests if 'api' in r['url']],
            'current_url': page.url,
            'timestamp': time.time()
        }

        with open('debug_login_info.json', 'w', encoding='utf-8') as f:
            json.dump(debug_info, f, ensure_ascii=False, indent=2)

        browser.close()

if __name__ == '__main__':
    debug_login()

```

#### Step 2: 分析输出

**发现的关键信息：**

```routeros

1
2
3
4
5
6

[Console error] Access to XMLHttpRequest at 'http://localhost:8000/api/v1/auth/login'
from origin 'http://localhost:5173' has been blocked by CORS policy

[Console error] Failed to load resource: net::ERR_FAILED

❌ 错误提示: 登录失败，请检查用户名和密码

```

**后端日志显示：**

```apache

1
2

ValueError: password cannot be longer than 72 bytes
File "app/core/security.py", line 26, in verify_password

```

#### Step 3: 定位根本原因

通过综合分析：

- ❌ **CORS 错误** - 实际是表象，请求被后端拒绝

- ❌ **bcrypt 兼容性问题** - passlib 与新版 bcrypt 不兼容

- ✅ **根本原因** - 密码验证逻辑崩溃导致返回错误响应

#### Step 4: 实施修复

修改 `backend/app/core/security.py`：

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

# 修改前：使用 passlib
from passlib.context import CryptContext
pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

def verify_password(plain_password: str, hashed_password: str) -> bool:
    return pwd_context.verify(plain_password, hashed_password)

# 修改后：直接使用 bcrypt
import bcrypt

def verify_password(plain_password: str, hashed_password: str) -> bool:
    try:
        return bcrypt.checkpw(
            plain_password.encode('utf-8'),
            hashed_password.encode('utf-8')
        )
    except Exception:
        return False

```

#### Step 5: 验证修复

重新运行调试脚本，确认：

- ✅ 无 CORS 错误

- ✅ 无控制台错误

- ✅ 成功跳转到首页

- ✅ localStorage 中有 token

## 最佳实践

### 1. 浏览器配置

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

# ✅ 推荐配置
browser = p.chromium.launch(
    headless=False,           # 调试时设为 False，便于观察
    channel="chrome",         # 使用系统 Chrome（更稳定）
    slow_mo=500,              # 放慢操作，便于调试
    args=[
        '--no-sandbox',
        '--disable-infobars',
        '--disable-extensions'
    ]
)

```

**⚠️ 重要提醒：进程隔离原则**

> 

**Playwright 必须启动独立的 Chrome 进程，严禁复用用户当前打开的浏览器！**

**原因：**

- 🔒 **权限边界**：AI 只能控制自己启动的进程，无法操作用户的浏览器

- 👤 **用户隐私**：避免访问用户的个人浏览数据、登录状态等敏感信息

- 🛡️ **工作隔离**：防止自动化测试干扰用户的正常工作流程

- 🧹 **资源清理**：测试完成后可以安全关闭，不留残留

### 2. 元素定位策略

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

# ✅ 优先级顺序
# 1. 使用 name/id 属性（最稳定）
page.locator('input[name="username"]')
page.locator('#login-button')

# 2. 使用 data-testid（专为测试设计）
page.locator('[data-testid="submit-btn"]')

# 3. 使用 CSS 类名（注意可能的变化）
page.locator('.btn.btn-primary')

# 4. 使用文本内容（多语言场景需注意）
page.locator('text=登录')

# 5. 使用 XPath（最后手段）
page.locator('//div[@class="form"]/input[1]')

```

### 3. 等待策略

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

# ✅ 推荐的等待方式
# 1. 等待网络空闲
page.wait_for_load_state('networkidle')

# 2. 等待元素可见
page.wait_for_selector('#element', state='visible')

# 3. 等待元素可点击
page.wait_for_selector('#button', state='enabled')

# ❌ 避免的做法
time.sleep(5)  # 硬编码等待，不可靠且浪费时间

```

### 4. 错误处理

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

# ✅ 完善的错误处理
try:
    page.goto('http://localhost:5173/login', timeout=10000)
except TimeoutError:
    print("❌ 页面加载超时")
    page.screenshot(path='error_timeout.png')
    raise
except Exception as e:
    print(f"❌ 未知错误: {e}")
    page.screenshot(path='error_unknown.png')
    raise

```

## 常见问题

### Q1: 浏览器启动失败

**症状：**

```nix

1

playwright._impl._errors.Error: BrowserType.launch: Executable doesn't exist

```

**解决：**

```bash

1
2
3

# 重新安装浏览器
playwright install chromium
playwright install chrome

```

### Q2: 元素找不到

**症状：**

```nix

1

TimeoutError: locator.click: Timeout 30000ms exceeded

```

**解决：**

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

# 1. 检查选择器是否正确
print(page.content())

# 2. 截图查看当前状态
page.screenshot(path='debug_current.png')

# 3. 尝试多种选择器
element = page.locator('input[name="username"], input#username').first

# 4. 增加等待时间
page.wait_for_selector('input[name="username"]', timeout=10000)

```

### Q3: CORS 跨域错误

**症状：**

```pgsql

1

Access to XMLHttpRequest has been blocked by CORS policy

```

**解决：**

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

# 方法1：检查后端 CORS 配置
# backend/app/core/config.py
BACKEND_CORS_ORIGINS = [
    "http://localhost:5173",
    "http://127.0.0.1:5173"
]

# 方法2：使用 Playwright 绕过 CORS（仅用于调试）
context = browser.new_context(
    bypass_csp=True,
    ignore_https_errors=True
)

```

## 总结

MCP Playwright 将 AI 助手与浏览器自动化结合，实现了智能化的 Web 应用调试。通过 AI 驱动的自动化调试、全面的上下文感知、高效的迭代循环和可复用的调试资产，大大提升了调试效率和质量。

**核心优势：**

- **智能化** - AI 驱动的自动化调试，减少人工干预

- **全面性** - 从 UI 到网络，从截图到日志，全方位监控

- **高效性** - 快速定位问题，自动生成报告

- **可复用** - 调试脚本和最佳实践可以沉淀为团队资产

**行动建议：**

- 立即开始 - 在你的下一个 bug 修复中使用 Playwright 调试

- 建立规范 - 制定团队的 Playwright 使用规范

- 积累资产 - 将常用的调试脚本整理成工具库

- 持续优化 - 定期回顾和改进调试流程

---

**相关资源：**

- [Playwright 官方文档](https://playwright.dev/)

- [Playwright Python API](https://playwright.dev/python/docs/intro)

- [MCP 协议规范](https://modelcontextprotocol.io/)

> 

💡 **提示**：本文是基于实战经验总结的 MCP Playwright 使用指南，更多细节请参考完整的实战指南文档。