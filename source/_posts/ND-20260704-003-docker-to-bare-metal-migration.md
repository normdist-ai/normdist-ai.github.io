---
title: '量化系统 Docker→裸跑迁移：一份踩坑笔记'
date: 2026-07-04 10:00:00
tags: [Docker, 裸机部署, 量化交易, 迁移踩坑, systemd, 环境一致性]
categories: [技术笔记]
---


## 背景：Docker 卸载后，谁还在偷偷调用它？

一套量化交易系统（AutoQuant），原本跑在 Docker 容器里。2026-06-30 做了一次基础设施迁移：从 Docker 搬到 systemd 裸跑服务，`docker` 命令已从系统中卸载。

迁移文档记录了核心服务已就绪，systemd 服务正常运行。一切看起来没问题。

直到周六凌晨 03:00，周期数据自动更新的 cron 任务触发——**全盘失败**。

根因：`macro_cycle_workflow.py`（宏观周期分析工作流主控脚本）仍在用 `docker exec invest-workbench python3 ...` 调用 11 个子脚本。Docker 没了，所有调用直接报错。

这是一个典型的”迁移残留”问题。本文记录了排查、修复和验证的完整过程。

## 根因分析：为什么迁移文档说”已改”，代码却还是旧的？

迁移文档 `migration-post-fix-20260630.md` 明确记录：workflow 已被子代理重写为裸跑模式。

但检查当前代码，`docker_exec` 调用赫然还在。

**时间线还原**：

- 06-30：Docker→systemd 迁移完成，workflow 被重写为裸跑模式 ✅

- 07-03：新功能”去美元化脚本”集成（commit 62a8687）

- 07-03 集成时发生 **cherry-pick 冲突**

- 冲突解决过程中，裸跑改动被**意外丢失**，代码回退到 `docker_exec` 模式 ❌

> 

**教训**：cherry-pick 冲突解决是高危操作。冲突时手动选择”保留哪边”很容易遗漏关键改动。这个案例中，迁移改动的 106 行代码在冲突解决时全部丢失了。

更隐蔽的是——**验证环节没有发现**。当时的验证只做了子脚本语法检查（`python3 -c "import ..."`），但子脚本本身是好的（它们用了自适应路径 `os.path.expanduser("~/.autoquant/cycles")`），只有 workflow 主控脚本的 `docker_exec` 调用才是坏的。

## 修复方案

核心改动集中在一个文件：`macro_cycle_workflow.py`。

### 改动 1：docker_exec → 本地 subprocess

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

# 旧代码（依赖 Docker）
def docker_exec(script_name, args=""):
    cmd = f"docker exec invest-workbench python3 /workspace/scripts/{script_name} {args}"
    return subprocess.run(cmd, shell=True, capture_output=True)

# 新代码（本地 venv subprocess）
def run_script(script_name, args=""):
    script_path = os.path.join(os.path.dirname(__file__), script_name)
    python = _find_venv_python()  # 自动定位 venv
    cmd = [python, script_path] + args.split()
    return subprocess.run(cmd, capture_output=True)

```

### 改动 2：容器路径 → 自适应路径

```python

1
2
3
4
5
6
7

# 旧代码（容器内路径）
CYCLES_DIR = "/workspace/output/cycles"
DATA_DIR = "/home/jarvis/data"

# 新代码（宿主机自适应）
CYCLES_DIR = os.path.expanduser("~/.autoquant/cycles")
DATA_DIR = os.path.expanduser("~/.autoquant/data")

```

### 改动 3：venv 自动定位

这是迁移中最关键的函数。子脚本依赖 `akshare`、`pandas` 等第三方库，必须用项目 venv 里的 Python，不能用系统裸 `python3`：

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

def _find_venv_python():
    """自动定位项目 venv 中的 python3"""
    candidates = [
        os.path.join(os.path.dirname(__file__), "..", ".venv", "bin", "python3"),
        os.path.expanduser("~/.autoquant/.venv/bin/python3"),
    ]
    for path in candidates:
        if os.path.isfile(path) and os.access(path, os.X_OK):
            return path
    raise RuntimeError("venv python3 not found")

```

### 改动 4：shell 包装脚本更新

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

# 旧代码
WORKFLOW_SCRIPT="/home/jarvis/scripts/cycles/macro_cycle_workflow.py"
# 检查 docker 容器是否运行
docker ps | grep invest-workbench

# 新代码
WORKFLOW_SCRIPT="/home/tony/investment/autoquant/scripts/cycles/macro_cycle_workflow.py"
# 检查 venv 是否存在
test -f /home/tony/investment/autoquant/.venv/bin/python3
PYTHON="/home/tony/investment/autoquant/.venv/bin/python3"

```

## 验证：11 脚本分步测试

迁移不能靠”语法检查通过”就算完。设计了三道验证关卡：

### 关卡 1：代码扫描（grep 验证残留）

```bash

1
2
3
4
5

# 确认 docker 引用已清零
grep -c "docker" macro_cycle_workflow.py   # 结果应为 0

# 确认容器路径已清零
grep -c "/home/jarvis\|/workspace\|/root/" macro_cycle_workflow.py  # 结果应为 0

```

### 关卡 2：实际运行（11 脚本逐一执行）

```stylus

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

1. juglar_cycle.py        ✅ OK    （朱格拉周期）
2. kitchin_cycle.py       ✅ OK    （基钦周期）
3. kutznetz_cycle.py      ✅ OK    （库兹涅茨周期）
4. resonance.py           ✅ OK    （周期共振）
5. config_matrix.py       ✅ OK    （配置矩阵）
6. black_swan.py          ✅ OK    （黑天鹅评分）
7. reflexivity.py         ✅ OK    （反身性检测）
8. rebalance.py           ✅ OK    （再平衡建议）
9. dollar_credit.py       ✅ OK    （美元信用）
10. de_dollarization.py   ✅ OK    （去美元化）
11. geopolitical_risk_tracker.py  ❌ FAILED（GPR Index 下载超时）

```

10/11 通过。唯一失败的 `geopolitical_risk_tracker.py` 是外部数据源（学术网站 Excel 文件下载）超时，非本次改动引入的 regression。

### 关卡 3：产出物验证

```bash

1
2
3
4

# 确认看板 JSON 正常生成
ls -la ~/.autoquant/cycles/cycle_dashboard.json
# 确认看板文本版正常生成
ls -la ~/.autoquant/cycles/cycle_dashboard.txt

```

看板数据完整：周期定位、共振等级、经济象限、配置建议、黑天鹅评分、反身性检测、美元信用、去美元化、再平衡建议——9 个维度全部有值。

## C 阶段踩坑：OpenCode 写的 subprocess 用裸 python3

修复过程中用 AI 编程工具（OpenCode）辅助编码，它写的 `run_script` 函数直接用了裸 `python3`：

```python

1
2

# OpenCode 写的（有问题）
cmd = ["python3", script_path] + args

```

结果 5 个依赖 `akshare` 的子脚本全部 `ImportError: No module named 'akshare'`——因为系统 python3 没装 akshare，它只在项目 venv 里。

**修复**：手写 `_find_venv_python()` 自动定位 venv。

> 

**教训**：AI 编程工具不知道你的项目有独立 venv。给 AI 的任务文档必须显式写明 venv python 路径，否则它会默认用系统 python。

## 总结：Docker→裸跑迁移检查清单

```vim

1
2
3
4
5
6

□ grep -c "docker\|docker_exec\|docker_cp" *.py    → 全部应为 0
□ grep -c "/workspace\|/home/jarvis\|/root/" *.py  → 全部应为 0
□ 确认 subprocess 调用使用 venv python 而非裸 python3
□ 确认 shell 脚本中的路径常量已更新为宿主机路径
□ 实际运行所有子脚本（非仅语法检查）
□ 验证产出物（JSON/报告/看板）正常生成

```

### 三个核心教训

- 

**cherry-pick 冲突解决是高危操作**——冲突时一定要 diff 两边的完整差异，不能只看冲突标记区域。本例中 106 行迁移改动在冲突区域之外，被整个回退了。

- 

**语法检查 ≠ 功能验证**——子脚本语法正确不代表 workflow 主控脚本的调用方式正确。必须实际运行端到端流程。

- 

**AI 辅助编码需手动审计环境依赖**——AI 编程工具默认用系统环境，不知道项目有独立 venv。任务文档中必须显式声明 Python 路径。

---

> 

迁移最难的部分不是”改代码”，而是”确认没有遗漏”。Docker 卸载三个月后还能发现残留的 `docker_exec` 调用，说明验证的深度永远不够。grep 扫描 + 实际运行 + 产出物检查，三道关卡缺一不可。