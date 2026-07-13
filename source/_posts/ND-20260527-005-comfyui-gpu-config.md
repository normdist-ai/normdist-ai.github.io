---
title: 'ComfyUI指定GPU出图配置指南'
date: 2026-05-27 10:00:00
tags: [ComfyUI, GPU, AI绘画, 配置]
categories: [技术笔记]
---

## 为什么需要指定 GPU？

在多 GPU 服务器上运行 ComfyUI 时，经常遇到以下场景：

- **GPU 资源隔离**——GPU 0 被 LM Studio、TensorRT-LLM 等大模型服务占用，ComfyUI 需要跑在另一张卡上

- **显存大小差异**——不同型号 GPU 显存不同，大尺寸出图（如 SDXL、1024×1024）需要更大显存的显卡

- **多用户并发**——团队共用服务器时，不同服务分配到固定 GPU，避免争抢资源

本文记录三种指定 ComfyUI 使用特定 GPU 的方法，以及实际踩坑经验。

---

## ⚡ 方法一：systemd 服务配置（推荐）

如果你的 ComfyUI 通过 systemd 服务启动（这也是最稳定的方式），这是**首选方案**。

### 查看当前配置

```bash

1
2
3
4
5

# 用户级服务
cat ~/.config/systemd/user/comfyui.service

# 或系统级服务
cat /etc/systemd/system/comfyui.service

```

### 修改 GPU 指定

编辑服务文件，在 `[Service]` 段添加环境变量：

```ini

1
2
3

[Service]
Environment=CUDA_VISIBLE_DEVICES=1   # ← 关键行：使用物理 GPU 1
ExecStart=/path/to/ComfyUI/venv/bin/python3 main.py --listen 0.0.0.0 --port 8188

```

### 重启并验证

```bash

1
2
3
4
5
6

# 重载配置 + 重启服务
systemctl --user daemon-reload
systemctl --user restart comfyui

# 确认进程跑在目标 GPU 上
nvidia-smi | grep python

```

> 

**优点**：配置持久化，重启自动生效，优先级最高。
**适用场景**：生产环境、长期运行的 ComfyUI 服务。

---

## 🔧 方法二：命令行参数 `--cuda-device`

ComfyUI 原生支持通过命令行指定 GPU：

```bash

1

python main.py --listen 0.0.0.0 --port 8188 --cuda-device 1

```

### ⚠️ 重要陷阱

**`CUDA_VISIBLE_DEVICES` 环境变量的优先级高于 `--cuda-device`！**

如果环境中已设置 `CUDA_VISIBLE_DEVICES=0`，即使传入 `--cuda-device 1`，ComfyUI 实际仍使用 GPU 0。排查方法：

```bash

1
2

# 查看进程实际的环境变量
cat /proc/<PID>/environ | tr '\0' '\n' | grep CUDA

```

> 

**优点**：快速测试、临时切换。
**缺点**：容易被环境变量覆盖，优先级最低。
**适用场景**：开发调试、一次性验证。

---

## 🌍 方法三：环境变量 `CUDA_VISIBLE_DEVICES`

启动时通过环境变量限制 ComfyUI 可见的 GPU：

```bash

1
2
3
4
5

# 单卡指定
CUDA_VISIBLE_DEVICES=1 python main.py --listen 0.0.0.0 --port 8188

# nohup 后台运行
CUDA_VISIBLE_DEVICES=1 nohup python main.py --listen 0.0.0.0 --port 8188 > comfyui.log 2>&1 &

```

### GPU 索引重映射机制

`CUDA_VISIBLE_DEVICES` 有一个容易忽略的特性——**它会重新编号 GPU 设备**：

设置
代码中 `cuda:0` 对应物理卡
代码中 `cuda:1` 对应物理卡

`CUDA_VISIBLE_DEVICES=0`
GPU 0
—

`CUDA_VISIBLE_DEVICES=1`
GPU 1
—

`CUDA_VISIBLE_DEVICES=0,1`
GPU 0
GPU 1

`CUDA_VISIBLE_DEVICES=1,0`
**GPU 1**
GPU 0

**示例**：`CUDA_VISIBLE_DEVICES=1,0` 时，CUDA 将物理 GPU 1 映射为逻辑设备 0。代码里写 `cuda:0`，实际访问的是物理卡 1。这在某些硬编码 `cuda:0` 的模型加载场景中非常有用——不用改代码就能指定用哪张卡。

> 

**优点**：灵活、无需修改服务文件。
**缺点**：需要每次启动时设置，shell 配置中可能意外覆盖。
**适用场景**：手动启动、脚本化部署、需要重映射的场景。

---

## 🕳️ 踩坑记录

### 坑一：`--cuda-device` 参数”无效”

**现象**：用 `--cuda-device 1` 启动，`nvidia-smi` 看到进程仍在 GPU 0。

**原因**：systemd 服务文件中硬编码了 `Environment=CUDA_VISIBLE_DEVICES=0`，环境变量优先级高于命令行参数。

**解决**：修改 systemd 服务文件中的 `CUDA_VISIBLE_DEVICES` 值。

### 坑二：手动设置的环境变量被覆盖

**现象**：终端里执行 `export CUDA_VISIBLE_DEVICES=1` 后启动 ComfyUI，进程实际仍用 GPU 0。

**排查清单**：

```bash

1
2
3
4
5
6
7
8

# 1. 检查 systemd 服务中的环境变量（最常见原因）
grep CUDA ~/.config/systemd/user/*.service /etc/systemd/system/*.service

# 2. 检查 shell 配置文件
grep CUDA ~/.bashrc ~/.profile ~/.zshrc /etc/environment

# 3. 查看运行中进程的实际环境变量
cat /proc/<PID>/environ | tr '\0' '\n' | grep CUDA

```

---

## 🔄 快速切换脚本

经常需要在 GPU 之间切换？一个脚本搞定：

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

#!/bin/bash
# switch-comfyui-gpu.sh — 一键切换 ComfyUI 的 GPU

GPU_ID=$1

if [ -z "$GPU_ID" ]; then
    echo "用法: $0 <gpu_id>"
    echo "示例: $0 1   # 切换到 GPU 1"
    exit 1
fi

SERVICE_FILE=~/.config/systemd/user/comfyui.service

# 修改服务文件中的 CUDA_VISIBLE_DEVICES
sed -i "s/CUDA_VISIBLE_DEVICES=[0-9]/CUDA_VISIBLE_DEVICES=$GPU_ID/" "$SERVICE_FILE"

# 重载配置并重启
systemctl --user daemon-reload
systemctl --user restart comfyui

# 验证结果
sleep 3
echo "✅ ComfyUI 已切换到 GPU $GPU_ID："
nvidia-smi | grep python

```

使用方式：

```bash

1
2
3

chmod +x switch-comfyui-gpu.sh
./switch-comfyui-gpu.sh 1   # 切到 GPU 1
./switch-comfyui-gpu.sh 0   # 切回 GPU 0

```

---

## 📊 方法对比总结

方法
优先级
持久化
可靠性
推荐场景

systemd `Environment`
⭐⭐⭐ 最高
✅ 自动
⭐⭐⭐ 最可靠
生产环境、长期运行

环境变量 `CUDA_VISIBLE_DEVICES`
⭐⭐ 中
❌ 需每次设置
⭐⭐ 需注意覆盖
手动启动、脚本部署

`--cuda-device` 参数
⭐ 最低
❌ 仅本次
⭐ 易被覆盖
快速测试、临时验证

---

## 💡 最佳实践建议

- **生产环境统一用 systemd**：配置一次，永久生效，重启不丢。

- **多 GPU 服务器做固定分配**：LM Studio 占 GPU 0，ComfyUI 占 GPU 1，各跑各的互不干扰。

- **切换前检查环境变量**：`--cuda-device` 不生效时，先查 `CUDA_VISIBLE_DEVICES` 有没有捣乱。

- **善用重映射**：需要让代码中的 `cuda:0` 指向物理 GPU 1 时，用 `CUDA_VISIBLE_DEVICES=1,0` 比改代码方便得多。

---

*本文基于实际多 GPU 服务器部署经验总结。如有其他踩坑经验，欢迎交流。*