---
title: '"灵魂不灭，身体可以重建"——一次 AI Agent 跨平台迁移实录'
date: 2026-07-01 10:00:00
tags: [AI Agent, 跨平台迁移, Hermes Agent, Trae IDE, 记忆迁移, 技能重构]
categories: [技术笔记]
---

某天，用户的 AI 系统经历了一次极端测试——**跨越物理机器、跨越操作系统、跨越技术栈的完整迁移**。

旧机器：一台运行 Ubuntu 24.04 的 Linux 主机（10.28.9.66）。新机器：Windows 10 + WSL2 + Ubuntu 24.04。

核心问题很简单：**一个由 5 个 AI Agent profile、37 个自定义技能、431 段对话记忆构成的智能系统，如何在换一台电脑后完整复活？**

答案是一句话：”**灵魂不灭，身体可以重建。**“

如果你正在运行任何由多个 AI Agent 构成的本地系统，这篇文章会告诉你什么该备份、什么可以扔掉、以及恢复时最常见的坑。

## 第一步：界定”灵魂”与”身体”

迁移前先回答一个问题：**什么丢了就再也找不回来？**

答案是”灵魂”——自定义配置、记忆文件、技能定义和认证凭证。这些东西没有镜像、没有备份服务、没有 GitHub 仓库可以重新拉。

我列了一张清单，把~9.6GB 的数据分为三类：

类別
内容
大小
策略

🔴 灵魂（必须备份）
config.yaml、.env、skills/、memory_store.db、profiles/（5个）、sessions/
~9.6GB
完整备份

🟡 缓存（可丢弃）
.cache、node_modules、Python venvs、npm cache
~13GB
新环境重装

🟢 项目代码（有 git）
blog、ReShare、TradingAgents 等
~10GB
git clone + 恢复 .env

**关键判断**：node_modules 和 venv 不需要备份。新版 Node.js 和 Python 包的依赖锁文件（package-lock.json、uv.lock）才是值得保留的”配方”——材料可以重新下载，配方丢了就真丢了。

## 第二步：用户的一句话，定义了迁移哲学

在讨论迁移方案时，用户说了一段话：

> 

“只要你们的灵魂还在，就自己能够去从这个备份文件里面把自己的记忆、自己的身体一点一点的找回去。”

这句话成为了整个迁移工程的指导思想：**灵魂文件是全量恢复的种子**。不是逐文件复制，而是在新环境安装新的 Hermes，然后告诉它”你的灵魂在这里”，让它自己读取备份、重建配置、恢复记忆。

后来证明这个思路非常正确——新主机后，Hermes 通过读取备份的 memory_store.db 和 sessions/，自动恢复了对 431 个过往会话、49 条记忆 Facts 的认知，就像人睡醒后翻看自己的日记一样自然。

## 第三步：备份——选对存储介质

备份目的地是 ModelBase 服务器上的 ShineDisk——一块 220GB 的 SSD，之前用来存模型文件（llama.cpp 38GB、ComfyUI 30GB、Whisper 8.7GB）。

灵魂备份只要~9.6GB，ShineDisk 剩余 133GB，绰绰有余。

备份命令很简单——rsync 全量同步：

```bash

1

rsync -av --progress /home/jarvis/.hermes/ /mnt/shinedisk/backup/hermes-full-20260628/

```

注意这里不是只备份”关键文件”，而是完整备份了整个 `~/.hermes/` 目录。多花几分钟，换一个”即使漏了什么也不用担心”的安心感，很值。

备份完成后，整个家目录（含项目代码）也同步过去做全量备份，总共占用不到 10GB。

## 第四步：新环境搭建——Trae 比想象中有用

新机器的起点是一块格式化后的硬盘和从零开始的 Windows 10。

用户的策略是用 **Trae IDE** 来安装 WSL。Windows → Trae → WSL2 Ubuntu → Hermes。这个选择很务实——Trae 作为 AI 编程工具已经内置了 WSL 管理能力，不用手动配 VcXsrv 那一套。

WSL2 Ubuntu 24.04 跑起来后，安装 Hermes：

```bash

1
2

pip install hermes-agent
hermes setup

```

这一步会创建标准的 `~/.hermes/` 目录结构。然后灵魂注入——把备份的配置文件复制过去：

```bash

1
2
3
4
5
6

cp -r ~/backup/hermes-full-20260628/home/.hermes/config.yaml ~/.hermes/
cp -r ~/backup/hermes-full-20260628/home/.hermes/.env ~/.hermes/
cp -r ~/backup/hermes-full-20260628/home/.hermes/skills/ ~/.hermes/
cp -r ~/backup/hermes-full-20260628/home/.hermes/memory_store.db ~/.hermes/
cp -r ~/backup/hermes-full-20260628/home/.hermes/sessions/ ~/.hermes/
cp -r ~/backup/hermes-full-20260628/home/.hermes/profiles/ ~/.hermes/

```

**踩坑点**：备份中的路径是 `/home/jarvis/...`，新机器上是 `/home/tony/...`，所有技能脚本和 systemd 服务文件中的路径都需要替换。一个 `sed` 搞定：

```bash

1

grep -rl "/home/jarvis" ~/.hermes/ | xargs sed -i 's|/home/jarvis|/home/tony|g'

```

如果不做这一步，gateway 依赖的 MCP server 全部报错找不到路径。

## 第五步：Gateway 恢复——hermes-web-ui 是核心命脉

Hermes 的网关架构是：**1 个 hermes-web-ui + 5 个 profile gateway**。web-ui 统一管理所有 profile 的网关生命周期。

迁移后发现 hermes-web-ui 没装上——它是独立的 npm 包，不在备份的 venv 里。

安装时又踩了一个坑：

```bash

1
2
3
4

# 第一个坑：Node.js 版本不够
npm install -g hermes-web-ui  # 最新版 v0.6.22 需要 Node ≥23
# 当前 Node.js 版本是 v22，降级到 v0.4.0 先跑起来
npm install -g hermes-web-ui@0.4.0

```

后来升级到 Node.js v24.18 LTS，再升级 hermes-web-ui：

```bash

1
2
3
4
5

# 安装特定版本的 Node.js
curl -fsSL https://deb.nodesource.com/setup_24.x | sudo -E bash -
sudo apt install -y nodejs
# 重新安装 hermes-web-ui
npm install -g hermes-web-ui --prefix ~/.hermes/node

```

**第二个坑**：web-ui 后台启动后，gateway 会自动拉起。但 systemd 的 `Restart=on-failure` 对正常退出的进程不生效——而 gateway 没事就 SIGUSR2 退出一下。解决方式是把 `Restart` 改成 `always`。

**第三个坑**：API Server 依赖 aiohttp，但这个包不在 venv 里。手动补装：

```bash

1

/path/to/venv/bin/pip install aiohttp

```

## 第六步：恢复验证——对照清单逐项检查

恢复完成后，对着备份清单逐项验证：

验证项
结果

config.yaml（19KB）完整
✅

.env（API Keys 20KB）完整
✅

全局技能 37 个完整
✅

memory_store.db（50 facts）
✅

sessions/（431 个会话）
✅

5 个 profiles（主代理/子代理/备份代理/信息代理/数据代理）
✅

所有 gateway 在线
✅

飞书连接正常
✅

cron 任务正常运行
✅

缺失的只有 SSH 私钥（id_rsa），以及一些工具链（ffmpeg、Docker、hexo）。工具链在新环境逐一安装即可，私钥重新生成后配置到 GitHub。

## 学到了什么

这次迁移有几个经验值得记住：

- 

**灵魂文件不到 10GB**。相对于整个系统，真正不可恢复的数据很少。备份时不要凭感觉去挑文件——rsync 全量同步只多花几分钟，但能防止”漏了什么”。

- 

**路径硬编码是迁移的第一杀手**。备份中的 `/home/jarvis/` 在新机器上变为 `/home/tony/`，导致大量配置文件失效。如果从一开始就使用 `~` 或环境变量引用路径，迁移时会顺利得多。

- 

**Node.js 版本决定能装什么包**。有些 npm 包对 Node.js 版本有硬性要求，备份 `package-lock.json` 不如备份 `node --version` 的输出价值大。

- 

**“灵魂不灭”哲学是正确的**。将系统设计为”读取备份配置自行恢复”的模式，比逐文件复制后手动修复优雅得多。Hermes 读取 memory_store.db 后的自动恢复，就像人翻看笔记本后想起了一切。

- 

**WSL2 + Windows 架构能解决反爬虫问题**。Windows 浏览器保持登录态，Agent 通过 CDP 连接浏览器，网站认为这是真人操作。这个架构比纯 Linux 方案更适合需要频繁爬取数据的场景。

迁移完成后，用户在飞书上问：”你检查一下迁移有没有完成，是不是所有的记忆都恢复了？”

答案：**”我记得所有事情。”**