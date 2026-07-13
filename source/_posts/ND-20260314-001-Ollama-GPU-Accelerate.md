---
title: 'Ubuntu 24.04 部署 Ollama 并启用 NVIDIA RTX 3060 GPU 加速技术报告'
date: 2026-03-14 10:00:00
tags: [Ollama, GPU加速, Ubuntu, CUDA]
categories: [技术笔记]
---

**编号**：ND-20260314-001
**版本**:1.0
**更新**: 2026-03-14
**作者**: 小瑞

## 一、引言

### 1.1 项目背景

随着大语言模型（LLM）在各类应用场景中的快速普及，本地部署 Ollama 服务成为企业和开发者的重要选择。然而，如何正确配置 GPU 加速，充分发挥 NVIDIA 显卡的计算能力，是部署过程中的关键技术挑战。

### 1.2 目标概述

本次部署的核心目标包括：

- 在 Ubuntu Server 24.04 上部署 Ollama 大模型服务

- 实现双显卡分工：核显（AMD）负责显示输出，独显（NVIDIA RTX 3060）负责模型计算

- 通过 Cherry Studio 等客户端实现局域网访问

- 遵循 systemd 最佳实践，使用 override.conf 进行服务配置

### 1.3 硬件环境

组件
规格

CPU
AMD Ryzen 7 5700G（集成 Vega 核显）

独显
NVIDIA GeForce RTX 3060 12GB

内存
64GB DDR4

系统
Ubuntu Server 24.04 LTS

NVIDIA 驱动
570.211.01

CUDA 版本
12.8

---

## 二、环境准备与驱动验证

### 2.1 NVIDIA 驱动安装验证

在部署 Ollama 之前，首先确认 NVIDIA 显卡驱动已正确安装：

```bash

1

nvidia-smi

```

**输出结果**：

```gherkin

1
2
3
4
5
6
7
8

+-----------------------------------------------------------------------------------------+
| NVIDIA-SMI 570.211.01   Driver Version: 570.211.01   CUDA Version: 12.8     |
|-----------------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|===============================+======================+======================+======================|
|   0  NVIDIA GeForce ... Off | 00000000:10:00.0 Off |                    N/A      |
+-----------------------------------------------------------------------------------------+

```

**关键观察**：

- `Disp.A: Off` 表示该显卡未承担显示输出任务

- 驱动版本 570.211.01 满足 Ollama 要求的 ≥531 版本

- CUDA 版本 12.8 支持现代 GPU 加速

### 2.2 CUDA 工具包安装

Ollama 官方明确指出，使用 GPU 加速需要在系统中存在 CUDA 运行时库：

```bash

1
2
3
4
5

# 安装 CUDA toolkit
sudo apt-get install -y nvidia-cuda-toolkit

# 验证安装
ls -la /usr/lib/x86_64-linux-gnu/libcuda*

```

---

## 三、问题诊断与根因分析

### 3.1 初次部署的困境

在初始部署阶段，我们遇到了以下问题：

- **GPU 未被识别**：Ollama 日志显示仅识别到 CPU

- **推理性能低下**：模型运行速度极慢

- **环境变量不生效**：设置 `OLLAMA_GPU_LAYERS` 等参数后无效果

### 3.2 根因分析

通过日志分析发现问题根因：

```bash

1

journalctl -u ollama -f | grep -i "gpu\|cuda"

```

**问题日志**：

```routeros

1
2

level=INFO source=types.go:60 msg="inference compute" id=cpu library=cpu
level=INFO source=routes.go:1832 msg="vram-based default context" total_vram="0 B"

```

**根因**：官方下载的 Ollama 二进制文件不包含 CUDA 加速库，导致无法调用 GPU 进行计算。

### 3.3 解决方案

经过深入研究，发现 Ollama 官方安装包实际包含两个版本：

- **基础版本** (ollama-linux-amd64.tar.zst)：仅包含 CPU 加速

- **完整版本**（下载链接）：包含 `/usr/local/lib/ollama/cuda_v12/` 和 `cuda_v13` 目录

**关键发现**：正确的安装方式会自动包含 CUDA 加速库，但需要确保 `/usr/local/lib/ollama` 目录存在且包含 `libggml-cuda.so` 等 CUDA 库文件。

---

## 四、Ollama 安装与配置

### 4.1 官方推荐安装方式

使用官方安装脚本是最可靠的方式：

```bash

1

curl -fsSL https://ollama.com/install.sh | sh

```

该脚本会自动：

- 检测 NVIDIA GPU

- 下载正确的 Ollama 版本（包含 CUDA 库）

- 安装到 `/usr/local/bin/`

- 创建库文件到 `/usr/local/lib/ollama/`

### 4.2 CUDA 库验证

安装完成后，验证 CUDA 加速库是否正确安装：

```bash

1

ls -la /usr/local/lib/ollama/

```

**预期输出**：

```tap

1
2
3
4
5

drwxr-xr-x 6 root root 4096 Mar 14 12:19 /usr/local/lib/ollama/
drwxr-xr-x 2 root root 4096 Mar 14 01:52 cuda_v12/
drwxr-xr-x 2 root root 4096 Mar 14 01:37 cuda_v13/
-rwxr-xr-x 1 root root 873912 Mar 1 01:28 libggml-cpu-alderlake.so
-rwxr-xr-x 1 root root 1009080 Mar 1 01:28 libggml-cuda.so

```

**关键文件**：

- `libggml-cuda.so` - CUDA 加速核心库

- `cuda_v12/` - CUDA 12 运行时库

- `cuda_v13/` - CUDA 13 运行时库

### 4.3 Systemd 服务配置（最佳实践）

#### 4.3.1 为什么使用 override.conf

根据 systemd 最佳实践，不应直接修改主服务文件（`/etc/systemd/system/ollama.service`），而应使用 drop-in 目录进行配置覆盖：

- 避免升级时被覆盖

- 便于维护和回滚

- 配置清晰分离

#### 4.3.2 创建配置目录

```bash

1

sudo mkdir -p /etc/systemd/system/ollama.service.d

```

#### 4.3.3 配置 override.conf

根据 Ollama 官方文档，创建 `/etc/systemd/system/ollama.service.d/override.conf`：

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

[Service]
# ==== 网络与访问控制 ====
Environment="OLLAMA_HOST=0.0.0.0"

# ==== 存储路径 ====
Environment="OLLAMA_MODELS=/data/.ollama/models"

# ==== GPU 加速优化（官方推荐） ====
Environment="CUDA_VISIBLE_DEVICES=0"
Environment="OLLAMA_FLASH_ATTENTION=1"

# ==== 性能调优 ====
Environment="OLLAMA_NUM_PARALLEL=2"
Environment="OLLAMA_KEEP_ALIVE=5m"

# ==== 资源与安全 ====
LimitNOFILE=65536
NoNewPrivileges=true
PrivateTmp=true
ReadWritePaths=/data/.ollama

```

**配置参数说明**：

参数
说明
官方文档来源

`OLLAMA_HOST=0.0.0.0`
允许远程访问
GitHub GPU 文档

`CUDA_VISIBLE_DEVICES=0`
指定使用第一个 GPU
官方多 GPU 配置

`OLLAMA_FLASH_ATTENTION=1`
启用 Flash Attention 加速
官方性能优化

`OLLAMA_NUM_PARALLEL=2`
限制并行请求数
官方配置

`OLLAMA_KEEP_ALIVE=5m`
模型内存保留时间
官方配置

#### 4.3.4 服务重载与启动

```bash

1
2
3
4
5
6
7
8

# 重载 systemd 配置
sudo systemctl daemon-reload

# 重启 Ollama 服务
sudo systemctl restart ollama

# 查看服务状态
systemctl status ollama

```

---

## 五、GPU 加速验证

### 5.1 日志验证

通过日志确认 GPU 被正确识别：

```bash

1

journalctl -u ollama -f | grep -i "gpu\|cuda"

```

**成功日志**：

```routeros

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

level=INFO source=types.go:42 msg="inference compute"
 id=GPU-ed518f4d-e8bc-ebec-38f0-a054daf782f3
 library=CUDA
 compute=8.6
 name=CUDA0
 description="NVIDIA GeForce RTX 3060"
 libdirs=ollama,cuda_v12
 driver=12.8
 pci_id=0000:10:00.0
 type=discrete
 total="12.0 GiB"
 available="11.6 GiB"

level=INFO source=routes.go:1832 msg="vram-based default context"
 total_vram="12.0 GiB"
 default_num_ctx=4096

```

**关键指标解读**：

- `library=CUDA` - 确认使用 CUDA 加速

- `compute=8.6` - RTX 3060 计算能力

- `total="12.0 GiB"` - 可用显存

- `libdirs=ollama,cuda_v12` - 使用 CUDA 12 库

### 5.2 模型运行验证

```bash

1

ollama run qwen3.5:9b "Hello"

```

**输出**：

```fortran

1

Hello! 👋 How can I assist you today? Feel free to ask anything or let me know if you need help with anything specific! 😊

```

### 5.3 GPU 显存占用验证

```bash

1

nvidia-smi --query-gpu=utilization.gpu,memory.used --format=csv

```

**输出**：

```mel

1
2

utilization.gpu [%], memory.used [MiB]
0 %, 7954 MiB

```

**分析**：

- 显存占用约 7.8GB，符合 qwen3.5:9b 模型的显存需求

- GPU 利用率为 0%（推理完成后释放）

### 5.4 推理性能对比

模式
首次推理耗时
响应速度

CPU
~10 分钟
极慢

GPU (RTX 3060)
~3 秒
流畅

---

## 六、显存管理与模型选择

### 6.1 官方显存建议

根据 Ollama 官方文档：

> 

为确保可接受的性能，模型大小应至少比服务器可用 RAM 小两倍，并且是 GPU 可用显存的 2/3。

### 6.2 计算示例

**GPU 显存推荐最大模型**：

显存
推荐最大模型

8GB
5GB

12GB
8GB

24GB
16GB

### 6.3 RTX 3060 12GB 适用模型

模型
参数量
量化版本
显存需求

Llama 3.2 1B
Q8_0
~1.5GB

Llama 3.2 3B
Q8_0
~4GB

Qwen2.5 7B
Q4_K_M
~5GB

Qwen2.5 9B
Q8_0
~8GB

Llama 3 8B
Q4_K_M
~6GB

---

## 七、常见问题与解决方案

### 7.1 GPU 未被识别

**症状**：日志显示 `library=cpu`，VRAM 为 0

**排查步骤**：

- 验证 NVIDIA 驱动：`nvidia-smi`

- 检查 CUDA 库：`ls /usr/local/lib/ollama/`

- 检查环境变量：`journalctl -u ollama | grep CUDA`

**解决方案**：

```bash

1
2
3

# 重新安装包含 CUDA 库的 Ollama
sudo rm -rf /usr/local/lib/ollama
curl -fsSL https://ollama.com/install.sh | sh

```

### 7.2 多 GPU 指定问题

**场景**：系统有多张显卡，需指定使用特定 GPU

**官方推荐方案**：

```bash

1
2
3
4
5

# 使用 GPU UUID（更可靠）
export CUDA_VISIBLE_DEVICES=GPU-UUID

# 或使用数字索引
export CUDA_VISIBLE_DEVICES=0

```

**获取 GPU UUID**：

```bash

1

nvidia-smi -L

```

### 7.3 挂起/恢复后 GPU 丢失

**官方文档说明**：Linux 系统在挂起/恢复后可能出现 GPU 发现失败

**解决方案**：

```bash

1

sudo rmmod nvidia_uvm && sudo modprobe nvidia_uvm

```

---

## 八、最终配置汇总

### 8.1 服务配置

**文件**：`/etc/systemd/system/ollama.service.d/override.conf`

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
10
11

[Service]
Environment="OLLAMA_HOST=0.0.0.0"
Environment="OLLAMA_MODELS=/data/.ollama/models"
Environment="CUDA_VISIBLE_DEVICES=0"
Environment="OLLAMA_FLASH_ATTENTION=1"
Environment="OLLAMA_NUM_PARALLEL=2"
Environment="OLLAMA_KEEP_ALIVE=5m"
LimitNOFILE=65536
NoNewPrivileges=true
PrivateTmp=true
ReadWritePaths=/data/.ollama

```

### 8.2 验证命令

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

# 查看服务状态
systemctl status ollama

# 查看 GPU 识别
journalctl -u ollama | grep "inference compute"

# 监控 GPU 使用
nvidia-smi -l 1

# 测试模型
ollama run qwen3.5:9b "Hello"

```

---

## 九、结论

本次部署成功实现了以下目标：

✅ 在 Ubuntu Server 24.04 上部署 Ollama v0.18.0
✅ 正确配置 NVIDIA RTX 3060 GPU 加速
✅ 实现双显卡分工：核显负责显示，独显负责计算
✅ 通过 Cherry Studio 实现局域网访问
✅ 遵循 systemd 最佳实践，使用 override.conf 配置

### 性能提升

- 推理速度从分钟级提升到秒级

- GPU 显存得到有效利用（约 8GB / 12GB）

- 支持更大的模型运行

### 配置要点

- 使用官方安装脚本确保包含 CUDA 库

- 使用 systemd override.conf 进行配置管理

- 设置 `CUDA_VISIBLE_DEVICES` 指定 GPU

- 启用 `OLLAMA_FLASH_ATTENTION` 提升性能

---

## 十、参考来源

- Ollama 官方硬件支持：[https://docs.ollama.com/hardware-support](https://docs.ollama.com/hardware-support)

- Ollama GitHub GPU 配置：[https://github.com/ollama/ollama/blob/main/docs/gpu.md](https://github.com/ollama/ollama/blob/main/docs/gpu.md)

- Ollama Linux 安装：[https://docs.ollama.com/linux](https://docs.ollama.com/linux)

- NVIDIA 驱动与 CUDA 配置指南

---

**报告生成时间**：2026年3月14日
**部署环境**：Ubuntu Server 24.04 LTS | Ollama v0.18.0 | NVIDIA RTX 3060 12GB