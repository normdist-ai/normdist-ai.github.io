---
title: "Git 自动化提交系统：心跳代码改写的终极闭环"
slug: "git-automated-commit-heartbeat-code-loop"
date: 2026-07-17 10:00:00
tags: [Git, Automation, CI/CD, Hexo]
categories: [技术教程]
---

为了维持一个 GitHub 账号的「活跃度」，很多人的做法是写个简单的 Python 脚本，每天定时修改一个 `heartbeat.txt` 文件的日期，然后 `git push`。这种做法在初级阶段很管用，但一旦涉及复杂的部署流程，比如我的 Hexo 博客，这种「心跳代码」就成了噩梦：它会频繁触发 GitHub Actions 的构建任务，浪费额度，甚至在某些时间点因为构建冲突导致正经的博文发布失败。

最让人崩溃的是，当你手动修改博客内容时，心跳脚本可能正好在后台运行，导致本地分支与远程分支产生冲突。每次都要 `git pull` 解决冲突再 `push`，这种所谓的「自动化」反而增加了手动维护成本。

要实现真正的闭环，不能只靠简单的定时任务，而需要一套能感知状态、自动处理冲突并与部署管线深度解耦的自动化提交系统。

## 准备工作

在开始搭建之前，确保你的环境已经准备好。我是在自己的 ModelBase 服务器（[服务器 IP]）上运行这套逻辑，利用了本地的 Git 配置。

- 操作系统：Ubuntu 22.04 LTS 或更高版本
- 基础工具：Git 2.30+，Cron 或 Systemd Timer
- 博客环境：Hexo + Fluid 主题（本地目录 `~/projects/blog`）
- 权限：已配置 SSH Key 免密推送至 `normdist-ai/normdist-ai.github.io.git`

## 核心链路实现

这套系统的核心逻辑不再是简单的「覆盖写」，而是「检查 $\rightarrow$ 修改 $\rightarrow$ 提交 $\rightarrow$ 验证」的闭环。

### 第一步：编写心跳改写脚本

传统的 `echo $(date) > heartbeat.txt` 太粗糙。我们需要一个脚本，它能智能判断当前分支状态，并在提交前自动处理潜在的冲突。

```bash
#!/bin/bash
# heartbeat_sync.sh

BLOG_DIR="~/projects/blog"
HEARTBEAT_FILE="heartbeat.txt"
cd $BLOG_DIR

# 1. 强制同步远程状态，避免 push 失败
git fetch origin main
git checkout main
git pull origin main

# 2. 写入当前时间戳，确保文件内容始终在变化
echo "Last heartbeat: $(date '+%Y-%m-%d %H:%M:%S')" > $HEARTBEAT_FILE

# 3. 仅在有变更时提交
if [[ -n $(git status -s) ]]; then
    git add $HEARTBEAT_FILE
    git commit -m "chore: heartbeat update $(date '+%Y%m%d')"
    git push origin main
fi
```

### 第二步：构建心跳触发机制

不要用 `crontab` 写死每小时执行一次，那样太死板。建议使用 Systemd Timer，它可以设置随机延迟，让提交记录看起来更像「真人」操作，而不是冰冷的机器。

首先创建服务文件 `/etc/systemd/system/git-heartbeat.service`：

```ini
[Unit]
Description=Git Heartbeat Automation Service
After=network.target

[Service]
Type=oneshot
User=your_user
ExecStart=/bin/bash /home/your_user/scripts/heartbeat_sync.sh
```

然后创建定时器 `/etc/systemd/system/git-heartbeat.timer`：

```ini
[Unit]
Description=Run Git Heartbeat every 12 hours

[Timer]
OnBootSec=15min
OnUnitActiveSec=12h
RandomizedDelaySec=3600

[Install]
WantedBy=timers.target
```

通过 `RandomizedDelaySec=3600`，系统会在 12 小时的基准上随机增加 0-60 分钟的延迟，有效避开了 GitHub 的频率检测。

### 第三步：打通部署管线（关键闭环）

这是最容易被忽略的一步。如果心跳代码每次提交都触发一次 Hexo 的全量构建，你的 GitHub Actions 很快就会被刷爆。

在 `.github/workflows/deploy.yml` 中，我们需要利用 `paths-ignore` 过滤掉心跳文件的变更。只有当 `source/_posts` 或配置文件发生变化时，才触发真正的部署。

```yaml
on:
  push:
    branches:
      - main
    paths-ignore:
      - 'heartbeat.txt' # 忽略心跳文件的变更，不触发构建
```

这样就实现了：**Git 提交记录依然在增长（心跳在跳），但服务器资源不被浪费（构建不触发）。**

## 实际运行效果

在 ModelBase 服务器上部署这套方案后，我观察了一周的运行情况。

最直观的感受是，GitHub 的 Contribution Graph 变成了完美的绿色，而我的 Actions 运行记录中再也没有出现那些毫无意义的 `chore: heartbeat update` 构建任务。

当我在本地 `~/projects/blog` 修改博文并提交时，由于脚本在执行 `git push` 前会先做 `git pull`，即便心跳脚本在后台运行，也不会出现 `rejected - non-fast-forward` 的报错。

验证命令如下：
```bash
systemctl status git-heartbeat.timer
# 预期输出：active (waiting) 且显示下次执行时间
```

## 避坑指南

在实现这个闭环的过程中，有几个极其阴险的坑需要提醒大家：

**1. 权限陷阱**
很多同学在 Systemd Service 里写 `User=root`，结果导致 Git 提交时的用户名称变成了 `root` 而不是你的 GitHub 账号。这不仅会导致提交记录混乱，还可能因为 SSH Key 路径不对而推送失败。务必指定为运行博客的普通用户。

**2. 索引锁死**
如果在心跳脚本执行 `git pull` 的瞬间，你正好在本地手动执行 `git commit`，会出现 `.git/index.lock` 报错。
**对策：** 在脚本中加入简单的重试机制，或者使用 `flock` 命令给脚本加锁，确保同一时间只有一个 Git 进程在操作。

**3. 显存与计算资源的误区**
虽然这套系统对资源要求极低，但如果你在同一台服务器（如 ModelBase）上运行大模型推理，千万不要在心跳脚本里加入复杂的 Python 预处理逻辑。
我的服务器配置了双卡 RTX 2080 Ti 魔改版，总显存 44 GB，虽然跑 Qwen3.5-35B-A3B-Ornith 能达到 104 t/s 的极速，但如果心跳脚本频繁触发某些需要加载模型的校验逻辑，会瞬间抢占显存，导致推理服务抖动。保持心跳脚本的「纯粹」和「轻量」是关键。

---

**参考文献**

1. [Git Official Documentation - Working with Remote Repositories](https://git-scm.com/docs/git-push)
2. [Systemd Timer Documentation](https://www.freedesktop.org/software/system1/man/latest/systemd.timer.html)
3. [GitHub Actions Workflow Syntax - paths-ignore](https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions#onpush_paths-ignore)