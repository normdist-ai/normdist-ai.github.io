---
title: 'Windows CDP 远程调试：从踩坑到局域网打通'
date: 2026-06-11 10:00:00
tags: [CDP, Chrome, Edge, 远程调试, Windows, 自动化, 爬虫]
categories: [技术笔记]
---

## 起因

最近在做 AI Agent 项目，需要让 Agent 能控制浏览器——打开网页、填写表单、抓取数据。Chrome DevTools Protocol（CDP）是标准的解决方案，但实际操作起来，Windows 上到处是坑。

目标很简单：在 Windows 上开启 Chrome/Edge 的 CDP 端口，让局域网内的另一台机器能远程控制浏览器。

听起来一行命令的事，实际上踩了三个大坑。

## 坑一：CDP 端口根本不起

网上教程几乎都写着这样启动：

```bash

1

chrome.exe --remote-debugging-port=9222

```

双击快捷方式，浏览器倒是启动了，但访问 `http://localhost:9222/json`——什么也没有。

翻了一通日志，发现一行关键报错：

```cpp

1
2

DevTools remote debugging requires a non-default data directory.
Specify this using --user-data-dir.

```

**新版 Chrome/Edge 强制要求指定一个独立的用户数据目录**，否则 CDP 直接拒绝启动。这个细节很多教程都没提。

**解决方案：** 快捷方式的目标改成：

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

# Edge
"C:\Program Files (x86)\Microsoft\Edge\Application\msedge.exe" ^
  --remote-debugging-port=9222 ^
  --user-data-dir="%USERPROFILE%\edge-cdp-profile"

# Chrome
"C:\Program Files\Google\Chrome\Application\chrome.exe" ^
  --remote-debugging-port=9222 ^
  --user-data-dir="%USERPROFILE%\chrome-cdp-profile"

```

启动后访问 `http://localhost:9222/json`，终于看到 JSON 输出了。

> 

⚠️ **注意：** 启动前必须关闭所有浏览器窗口和后台进程，否则启动参数不会生效。

## 坑二：局域网访问不了

本机 `localhost:9222` 通了，但从局域网另一台机器访问 `http://<内网IP>:9222/json`——连接被拒绝。

网上有人说加 `--remote-debugging-address=0.0.0.0` 就行。**实际上这个参数在新版 Chromium 中已经失效了**，不管你怎么设，CDP 都只绑定 `127.0.0.1`。

这是 Chromium 的安全限制——不让 CDP 端口暴露到网络上，防止恶意利用。所以启动命令里根本不需要加这个参数，加了也没用。

**解决方案：Windows 端口转发。**

用 `netsh interface portproxy`（需要管理员权限的 PowerShell）：

```powershell

1
2
3
4
5
6

# 将外部对 9223 端口的访问，转发到本机 127.0.0.1:9222
netsh interface portproxy add v4tov4 ^
  listenaddress=0.0.0.0 ^
  listenport=9223 ^
  connectaddress=127.0.0.1 ^
  connectport=9222

```

这样局域网其他设备访问 `http://<内网IP>:9223/json`，就会被转发到本机的 `127.0.0.1:9222`，CDP 就能连上了。

> 

外部端口选 9223 而不是 9222，是为了跟 CDP 原始端口区分，方便后续管理和防火墙配置。

## 坑三：Windows 防火墙拦截

端口转发配好了，局域网还是连不上。最后一个拦路虎：**Windows 防火墙**。

默认情况下，Windows 防火墙会阻止所有外部入站连接。需要手动放行：

```powershell

1
2
3
4
5
6

netsh advfirewall firewall add rule ^
  name="CDP 9223" ^
  dir=in ^
  action=allow ^
  protocol=TCP ^
  localport=9223

```

## 验证

全部配好之后，局域网内的任何设备访问：

```dts

1

http://<内网IP>:9223/json

```

返回类似这样的 JSON，就说明成功了：

```json

1
2
3
4
5
6
7
8

[ {
   "description": "",
   "id": "ABCDEF123456",
   "title": "新建标签页",
   "type": "page",
   "url": "edge://newtab/",
   "webSocketDebuggerUrl": "ws://192.168.x.x:9223/devtools/page/ABCDEF123456"
} ]

```

拿到 `webSocketDebuggerUrl`，就可以通过 WebSocket 连接来控制浏览器了——导航、截图、执行 JavaScript、读写 Cookie，全套 CDP 能力都能用。

## 实际效果

配通之后，AI Agent 可以从局域网内的 Linux 服务器远程操控 Windows 上的 Edge 浏览器：

- **打开网页、搜索内容**——通过 `Page.navigate` + `Runtime.evaluate` 提取页面文本

- **截图**——通过 `Page.captureScreenshot` 获取页面视觉

- **读取登录态**——通过 `Network.getAllCookies` 获取 Cookie，省去在服务器端重新登录的麻烦

- **自动操作**——点击按钮、填写表单，通过 `Runtime.evaluate` 注入 JS 完成

对于需要登录态的网站（知乎、B站、YouTube 等），直接复用浏览器上的登录状态，比在服务器上管理 Cookie 方便得多。

## 踩坑总结

问题
原因
解决方案

CDP 端口未监听
缺少 `--user-data-dir` 参数
指定独立的用户数据目录

局域网访问被拒
Chromium 新版限制 CDP 只绑 127.0.0.1，`0.0.0.0` 参数已失效
`netsh portproxy` 端口转发（9223→9222）

端口转发后仍连不上
Windows 防火墙拦截
添加入站规则放行 9223 端口

## 清理

如果不再需要远程调试，记得清理端口转发和防火墙规则：

```powershell

1
2
3
4
5

# 删除端口转发
netsh interface portproxy delete v4tov4 listenaddress=0.0.0.0 listenport=9223

# 删除防火墙规则
netsh advfirewall firewall delete rule name="CDP 9223"

```

## 后记

整个排查过程花了两个多小时。第一个坑（`--user-data-dir`）浪费的时间最多，因为报错信息容易被忽略；第二个坑（`0.0.0.0` 失效）是 Chromium 的安全策略变更，网上很多教程还在推荐这个参数，但实际上已经不管用了；第三个坑（防火墙）反而是最常规的。

最终方案其实很简单——两个启动参数 + 两条 netsh 命令。但中间的排查过程才是真正花时间的地方。写下来，希望后面的人能少走弯路。