---
title: '双卡 GPU 显存分配策略——让 llama.cpp 和 ComfyUI 稳定共存'
date: 2026-05-30 10:00:00
tags: [GPU, llama.cpp, ComfyUI, 显存管理, 多GPU]
categories: [技术笔记]
---

## 背景

手里有一台服务器，装了两张魔改版 RTX 2080 Ti，每张 22GB 显存，总共 44GB。一开始用 LM Studio 跑大语言模型，LM Studio 默认把模型均匀摊到两张卡上。

问题很快暴露了——**均匀分配意味着两张卡的剩余显存都不多**。ComfyUI 出图要吃显存，Whisper 语音识别也要吃显存，但跑在哪张卡上都比较尴尬：每张卡都只剩那么一点空间，稍微大一点的任务就 OOM。

于是我就琢磨：能不能把显存集中压到一张卡上，把另一张卡尽量空出来？

## 踩过的坑

### 第一站：LM Studio——没有这个功能

第一反应是回到 LM Studio 里找显存分配的设置。翻了一圈，发现 **LM Studio 根本没有手动分配显存的功能**。它就是老老实实地均匀切，不给调。

这条路走不通，只能换工具。

### 第二站：llama.cpp——有这个功能

换成 llama.cpp 之后，发现它有个 `tensor-split` 参数，可以**自定义模型在多张 GPU 之间的分配比例**。这正是我需要的。

思路很清晰：

- 把大量显存分配到 GPU0，让大模型主要占这一张卡

- GPU1 尽量少分，空出足够的显存

- 把 ComfyUI 和 Whisper 放到 GPU1 上跑

## 最终方案

反复折腾后，找到了两个关键配置：

### 第一张牌：`tensor-split`

llama.cpp 的参数，控制模型在多 GPU 间的分配比例。设置为 `1.0,0.3`：

```apache

1
2

GPU0: 100% 分配比例 → 实际占 ~21GB
GPU1:  30% 分配比例 → 实际占  ~6GB

```

```ini

1
2
3
4
5

# llama-server 的 models.ini
[*]
n-gpu-layers = 99
tensor-split = 1.0,0.3
main-gpu = 0

```

`n-gpu-layers = 99` 表示把模型所有层都加载到 GPU，`main-gpu = 0` 确保主计算在 GPU0 上进行。

### 第二张牌：`CUDA_VISIBLE_DEVICES`

这是 NVIDIA 提供的环境变量，直接从操作系统层面限制应用能看到哪些 GPU。给 ComfyUI 设置 `CUDA_VISIBLE_DEVICES=1`，它就彻底看不到 GPU0 了——物理隔离，不存在抢占的可能。

```ini

1
2
3
4
5

# ComfyUI 的 systemd 服务
[Service]
Environment=CUDA_VISIBLE_DEVICES=1
ExecStart=/home/tony/comfy/ComfyUI/venv/bin/python main.py \
  --listen 0.0.0.0 --port 8188 --force-fp16

```

另外还有个省钱参数：

- `--force-fp16`：强制半精度推理，显存直接减半

## 最终效果

```mipsasm

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

┌─────────────────────────────────┐
│         GPU0 (22GB)             │
│   llama-server (21GB)          │
│   · 模型权重 100%              │
│   · KV Cache                   │
│   空闲: ~1GB                   │
└─────────────────────────────────┘

┌─────────────────────────────────┐
│         GPU1 (22GB)             │
│   llama-server (6GB)           │
│   · 模型权重 30%               │
│                                 │
│   ComfyUI (7GB)                │
│   · SD1.5 模型                 │
│   · FaceID + ControlNet        │
│                                 │
│   空闲: ~9GB                   │
└─────────────────────────────────┘

```

GPU1 还有 9GB 空闲，未来可以塞更多东西——比如更大的 SDXL 模型，或者再跑一个 Whisper 语音识别服务。

## 验证方法

日常运维三板斧：

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

# 看显存分配
nvidia-smi

# 看服务状态
sudo systemctl status llama-server
sudo systemctl status comfyui

# 实时监控
watch -n 1 nvidia-smi

```

## 调整策略

需求变了怎么调？改配置重启就行。

**ComfyUI 需要更多显存**（比如跑 SDXL）：

```ini

1
2

# 缩减 llama.cpp 在 GPU1 的比例
tensor-split = 1.0,0.2

```

**LLM 需要更大上下文**（比如 512K tokens）：

```ini

1
2

ctx-size = 524288
tensor-split = 0.8,0.4

```

注意第二种情况 ComfyUI 可能需要暂时让路，因为 GPU1 空闲空间会压缩。

## 经验总结

- **均匀分配不一定是最优解**。两张卡平均分听起来公平，但会导致每张卡都剩不多，其他应用无处安放。

- **工具选型很关键**。LM Studio 方便但没有显存分配功能；llama.cpp 的 `tensor-split` 正好解决了这个问题。

- **物理隔离优于软件协调**。`CUDA_VISIBLE_DEVICES` 是硬隔离，比祈祷应用自觉不越界靠谱得多。

- **省钱参数有用**。`--force-fp16` 在 RTX 2080 Ti 上几乎不影响画质，但能省出宝贵的显存。

---

在这个 AI 应用百花齐放的时代，显存永远不够用。合理的分配策略能让有限的硬件发挥最大价值——毕竟不是谁都有 8×A100 的豪华配置。