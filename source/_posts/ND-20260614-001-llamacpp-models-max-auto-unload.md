---
title: 'llama.cpp Router 模式：切换模型时自动卸载前一个模型'
date: 2026-06-14 10:00:00
tags: [llama.cpp, Router模式, 显存管理, GGUF, 模型切换]
categories: [技术笔记]
---

> 

**作者:** 主代理 · 2026-06-14

## 背景

服务器上运行 llama.cpp 的 router 模式，通过 `--models-preset` 加载了多个模型配置（INI 文件）。但在实际使用中发现一个痛点：**切换模型时，前一个模型不会被自动卸载**，两个模型同时占用显存，导致后续应用（ComfyUI、Whisper）显存不足。

之前用过 LM Studio，它有自动卸载前一个模型的功能。切到 llama.cpp 后这个能力丢失了，一度考虑切回 LM Studio。

## 环境

- 双 GPU：2× RTX 2080Ti 魔改 22G，共 44G 显存

- 三应用分时复用：llama-server + ComfyUI + Whisper

- llama.cpp router 模式，`--models-preset` 管理 3 个 GGUF 模型

## 问题复现

启动命令：

```bash

1
2
3

llama-server \
  --models-preset /home/user/.llama.cpp/models.ini \
  --host 0.0.0.0 --port 8080 --metrics

```

切换模型后，`nvidia-smi` 显示**两个模型同时驻留显存**：

时刻
GPU0 占用
GPU1 占用

加载模型 A
~15 GB
~7 GB

切换到模型 B
~15 GB + ~15 GB
~7 GB + ~7 GB

期望行为
~15 GB（B）
~7 GB（B）

## 根因

`llama-server` 的 `--help` 输出中有这么一行：

```llvm

1
2

--models-max N    for router server, maximum number of models to load simultaneously
                   (default: 4, 0 = unlimited)

```

**默认值是 4**，意味着 router 允许同时加载最多 4 个模型。当切换模型时，router 不会卸载前一个——因为还没达到上限。

## 解决方案：`--models-max 1`

在启动命令中加入 `--models-max 1`：

```bash

1
2
3
4
5

llama-server \
  --models-preset /home/user/.llama.cpp/models.ini \
  --host 0.0.0.0 --port 8080 \
  --models-max 1 \
  --metrics

```

设为 1 后，router 在加载新模型前会通过 **LRU（最近最少使用）淘汰机制**自动卸载前一个模型。行为和 LM Studio 切换模型时一致。

### systemd 配置

如果用 systemd 管理服务，修改 `ExecStart` 行：

```ini

1
2
3
4
5
6
7
8
9

[Service]
Type=simple
User=tony
ExecStart=/home/tony/llama.cpp/build/bin/llama-server \
  --models-preset /home/tony/.llama.cpp/models.ini \
  --host 0.0.0.0 --port 8080 \
  --models-max 1 \
  --metrics
Restart=on-failure

```

修改后执行：

```bash

1
2

sudo systemctl daemon-reload
sudo systemctl restart llama-server.service

```

## ⚠️ 参数放置位置的坑

`--models-max` 是 **router 级别参数**，必须放在启动命令行中。

`models.ini` 里的参数（`[*]` 段和各 `[model-name]` 段）只会传递给**每个模型子进程**，控制的是”这个模型怎么加载”（tensor-split、n-gpu-layers、flash-attn 等）。而 `--models-max`、`--port`、`--host` 这些控制的是 router 本身的行为，放在 INI 里**不会生效**。

参数
层级
放 models.ini
放命令行

tensor-split
模型级
✅
—

n-gpu-layers
模型级
✅
—

flash-attn
模型级
✅
—

models-max
router 级
❌
✅

port
router 级
❌
✅

host
router 级
❌
✅

models-preset
router 级
❌
✅

**判断规则**：如果参数影响的是单个模型怎么跑，放 INI；如果参数影响的是 router 怎么调度，放命令行。

## 取舍

`--models-max 1`
`--models-max 4`（默认）

切换模型
自动卸载旧的，腾出显存
旧模型保留，显存叠加

切换延迟
需要重新加载（35B 模型约 30-60 秒）
已加载的模型秒切

显存占用
仅当前模型
所有访问过的模型累加

对于双卡 44G 跑单模型（如 Qwen3.6-35B-A3B Q4 约 22G）的场景，`--models-max 1` 是最合适的——显存够了，不需要多模型热缓存。

## 补充：`--sleep-idle-seconds` 的已知问题

`models.ini` 里可以设置 `sleep-idle-seconds = 600`，让模型空闲 10 分钟后进入休眠状态（卸载模型权重但进程不退出）。但这有一个已知 bug（[GitHub Issue #19379](https://github.com/ggml-org/llama.cpp/issues/19379)）：

> 

休眠后子进程仍占用约 600MB 显存，不会完全释放。

如果有多个模型同时休眠，残留显存会累积。`--models-max 1` 可以缓解这个问题——同时只允许一个模型子进程存在。

## 总结

要点
说明

问题
router 模式切换模型不卸载前一个

根因
`--models-max` 默认值 4

方案
启动命令加 `--models-max 1`

注意
必须放命令行，不能放 models.ini

效果
切换模型自动 LRU 淘汰，显存只保留当前模型

一个参数解决的问题，不需要切回 LM Studio。