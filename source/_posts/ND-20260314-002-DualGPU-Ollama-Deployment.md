---
title: 'Ubuntu 双显卡部署 Ollama 并启用 NVIDIA RTX 3060 GPU 加速完整指南'
date: 2026-03-14 10:00:00
tags: []
categories: [技术笔记]
---

**编号**：ND-20260314-002
**版本**:1.0
**更新**: 2026-03-14
**作者**: 小瑞

本文档完整记录了在一台搭载 AMD Ryzen 7 5700G 集成显卡 和 NVIDIA GeForce RTX 3060 12GB 独立显卡 的服务器上，从零开始安装、配置，并最终成功部署支持 GPU 加速 和远程访问 的 Ollama 大模型服务的全流程。

## 核心方法论：PDCA（计划-执行-检查-处理）

文档特点：

✅ 生产环境验证通过
✅ 包含实战问题与解决方案
✅ 可完全复现的部署手册

---

## 一、硬件清单与准备

### 1.1 硬件配置

组件
型号/规格
用途说明

**CPU**
AMD Ryzen 7 5700G (8核16线程)
系统运算核心，内置 Radeon Graphics

**集成显卡**
AMD Radeon Graphics (内置于5700G)
主显示输出，负责操作系统图形界面渲染

**独立显卡**
NVIDIA GeForce RTX 3060 12GB
专用计算卡，100% 用于 Ollama 大模型推理加速

**系统盘**
128GB NVMe SSD
安装 Ubuntu 系统、核心软件

**数据盘**
800GB SATA SSD/HDD
专门用于存储 Ollama 大模型文件（路径：/data/.ollama/models），实现系统与数据分离

**内存**
128GB DDR4
为模型加载和系统运行提供充足缓冲

### 1.2 BIOS 关键设置

在安装操作系统前，需进入服务器主板 BIOS 进行以下设置，以确保硬件以最佳状态运行：

设置项
推荐配置
说明

**启动模式**
设置为 UEFI（禁用 Legacy/CSM）
有利于系统稳定性和未来升级

**安全启动 (Secure Boot)**
建议暂时禁用
避免安装第三方驱动（如 NVIDIA 驱动）时遇到签名问题

**显存分配**
为集成显卡 (AMD) 分配 2GB 显存
确保其有足够资源流畅驱动高分辨率显示器

**PCIe 嵌源分配**
确保连接 NVIDIA 独立显卡的 PCIe 插槽运行在 Gen4 或 Auto 模式
以获取最佳总线带宽

### 1.3 操作系统安装 (Ubuntu Server 24.04.4 LTS)

#### 制作安装介质

从官网下载 Ubuntu 24.04.4 LTS Server 镜像，使用 Rufus 等工具制作 U 盘启动盘。

#### 安装过程

从 U 盘启动，选择”安装 Ubuntu”。

#### 关键分区步骤（手动分区）

挂载点
位置
大小
文件系统
说明

**/**
/dev/nvme0n1p2
50GB+
LVM
系统根目录

**swap**
/dev/sdb1
128GB
swap
交换分区，物理内存的 25%-50%

**/data**
/dev/sdb2
800GB
ext4
专门存放 Ollama 模型数据

**用户创建**
最后创建用户（例如 jarvis），并设置强密码

---

## 二、基础系统与双显卡驱动配置

### 2.1 系统更新与基础工具

```bash

1
2

sudo apt update && sudo apt upgrade -y
sudo apt install -y net-tools curl wget vim htop lm-sensors

```

### 2.2 配置 AMD 集成显卡驱动（用于显示）

AMD 集成显卡驱动 (amdgpu) 通常已被内核包含。只需确保加载：

```bash

1
2

sudo modprobe amdgpu
echo "amdgpu" | sudo tee -a /etc/modules

```

### 2.3 安装 NVIDIA 驱动与 CUDA 工具包（用于计算）

**目标**：为 RTX 3060 安装驱动，但不让其接管显示输出。

#### 添加驱动仓库并安装

```bash

1
2
3
4
5
6

# 添加 NVIDIA 官方 PPA
sudo add-apt-repository ppa:graphics-drivers/ppa -y
sudo apt update

# 安装驱动（选择与 CUDA 12.x 兼容的版本，如550或更高）
sudo apt install -y nvidia-driver-550

```

#### 验证驱动安装

```bash

1
2

# 重启后，运行以下命令
nvidia-smi

```

**期望输出**：

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
|===============================+======================+======================+======================+
|   0  NVIDIA GeForce ... Off | 00000000:10:00.0 Off |                    N/A      |
+-----------------------------------------------------------------------------------------+

```

**关键观察**：

- `Disp.A: Off` 表示该显卡未承担显示输出任务 ✅

- 驱动版本 570.211.01 满足 Ollama 要求的 ≥531 版本 ✅

- CUDA 版本 12.8 支持现代 GPU 加速 ✅

### 2.4 安装 CUDA Toolkit

```bash

1
2
3
4
5
6
7

# 下载 CUDA Keyring
wget https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2404/x86_64/cuda-keyring_1.1-1_all.deb
sudo dpkg -i cuda-keyring_1.1-1_all.deb
sudo apt update

# 安装 CUDA Toolkit
sudo apt install -y cuda-toolkit-12-5

```

#### 验证安装

```bash

1

ls -la /usr/lib/x86_64-linux-gnu/libcuda*

```

---

## 三、配置 NVIDIA 仅用于计算（核心步骤）

**目标**：确保 NVIDIA 独立显卡 100% 专用于 Ollama 计算，不参与显示合成。

### 3.1 创建配置文件，禁止 NVIDIA 驱动参与显示合成

```bash

1
2
3
4
5
6
7
8

# 创建配置文件
sudo tee /etc/modprobe.d/nvidia-compute-only.conf << 'EOF'
options nvidia modeset=0
options nvidia NVreg_EnablePCIeGen3=1
EOF

# 更新 initramfs
sudo update-initramfs -u

```

### 3.2 验证双显卡状态

**将显示器连接至主板的 HDMI/DP 接口（来自 AMD 集成显卡）**

重启后，应能正常进入图形界面。

在终端运行：

```bash

1
2
3
4
5

# 1. NVIDIA 状态（应显示空闲）
nvidia-smi

# 2. AMD 状态（应显示为当前渲染）
glxinfo | grep "OpenGL renderer"

```

**期望结果**：

- `nvidia-smi`：显示 RTX 3060 信息，但 “Processes” 列表应为空（或仅 Ollama 进程）

- `glxinfo`：应显示 “AMD” 或 “Radeon”，确认 AMD 负责渲染

---

## 四、Ollama 服务安装与核心配置

### 4.1 核心原则

- **手动离线安装**：支持 GPU 的版本

- **Systemd 管理**：采用 override 方式进行配置

- **数据分离**：模型存储在独立数据盘

### 4.2 获取并安装 Ollama（手动/离线方式）

#### 下载安装包

访问 [Ollama 官方 GitHub Releases 页面](https://github.com/ollama/ollama/releases)，下载支持 GPU 的 Linux 版本：

**下载页面**：[https://github.com/ollama/ollama/releases](https://github.com/ollama/ollama/releases)

**操作说明**：
在页面中找到最新的稳定版本（如 v0.16.3 或更高），下载名为 `ollama-linux-amd64.tar.zst` 的文件。

#### 上传并安装到服务器

```bash

1
2
3

# 假设安装包已上传至用户家目录
sudo tar -xaf ~/ollama-linux-amd64.tar.zst -C /usr/local/bin/
sudo chmod 755 /usr/local/bin/ollama

```

### 4.3 创建专业的 Systemd 服务配置（采用 override 最佳实践）

为避免直接修改易出错的主服务文件，并便于未来维护升级，我们采用 systemd 推荐的 override.conf 方式进行配置。这是从初期配置错误中总结出的关键经验。

#### 创建主服务单元文件

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

sudo tee /etc/systemd/system/ollama.service << 'EOF'
[Unit]
Description=Ollama Service
After=network-online.target

[Service]
Type=exec
ExecStart=/usr/local/bin/ollama serve
User=ollama
Group=ollama
Restart=always
RestartSec=3

[Install]
WantedBy=multi-user.target
EOF

```

#### 创建配置覆盖目录和核心优化文件

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

sudo mkdir -p /etc/systemd/system/ollama.service.d

sudo tee /etc/systemd/system/ollama.service.d/override.conf << 'EOF'
[Service]
# ==== 网络与访问控制 ====
# 允许远程主机连接（Cherry Studio、Open WebUI 等必需）
Environment="OLLAMA_HOST=0.0.0.0"

# ==== 存储路径 ====
# 将模型存储在独立的 800GB 数据盘，路径为 /data/.ollama/models，实现系统与数据分离
Environment="OLLAMA_MODELS=/data/.ollama/models"

# ==== GPU 加速核心优化 ====
Environment="OLLAMA_GPU_LAYERS=100" # 最大化利用GPU进行计算
Environment="OLLAMA_NUM_PARALLEL=4" # 并行请求数，根据CPU核心数调整

# ==== 资源与安全 ====
LimitNOFILE=65536
NoNewPrivileges=true
PrivateTmp=true
# 确保服务进程有权访问数据目录
ReadWritePaths=/data/.ollama
EOF

```

**配置参数说明**：

参数
说明
官方文档来源

`OLLAMA_HOST=0.0.0.0`
允许远程访问
GitHub GPU 文档

`OLLAMA_MODELS=/data/.ollama/models`
模型存储路径（独立数据盘）
自定义配置

`OLLAMA_GPU_LAYERS=100`
最大化 GPU 层数
官方性能优化

`OLLAMA_NUM_PARALLEL=4`
限制并行请求数
官方配置

#### 创建数据目录并授权

```bash

1
2

sudo mkdir -p /data/.ollama/models
sudo chown -R ollama:ollama /data/.ollama

```

#### 启用并启动服务

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

# 启用并启动服务
sudo systemctl enable --now ollama

# 查看服务状态
sudo systemctl status ollama

```

### 4.4 配置用户环境（镜像加速与参数）

创建用户配置文件，设置国内镜像加速下载：

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

# 创建配置目录
mkdir -p ~/.ollama

cat > ~/.ollama/config.json << 'EOF'
{
  "registry": {
    "mirrors": {
      "registry.ollama.ai": "https://mirrors.aliyun.com/ollama"
    }
  }
}
EOF

```

---

## 五、模型下载、验证与远程访问配置

### 5.1 下载测试模型

先使用国内镜像加速下载一个小模型进行测试：

```bash

1

ollama pull llama3.2:1b

```

### 5.2 验证 GPU 加速是否生效

**这是最关键的验收步骤。**

#### 打开终端 A，实时监控 GPU

```bash

1

watch -n 0.5 nvidia-smi

```

#### 打开终端 B，运行模型推理

```bash

1

ollama run llama3.2:1b "请用中文写一首关于夏日的五言绝句。"

```

#### 验证结果

在终端 A 的监控中，在模型生成文本的几秒内，`Volatile GPU-Util` 应从 0% 显著上升（例如升至 30%-70%），同时 `Memory-Usage` 会增加。

**成功标志**：✅ GPU 显著参与计算，证明 GPU 加速成功。

### 5.3 GPU 显存占用验证

```bash

1

nvidia-smi --query-gpu=utilization.gpu,memory.used --format=csv

```

**输出示例**：

```mel

1
2

utilization.gpu [%], memory.used [MiB]
65 %, 7954 MiB

```

**分析**：

- 显存占用约 7.8GB，符合 llama3.2:1b 模型的显存需求

- GPU 利用率在推理时显著提升，闲置时为 0%

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

**性能提升**：约 **200 倍**（10 分钟 vs 3 秒）

### 5.5 配置远程访问（为 Cherry Studio 等客户端）

#### 确认服务监听地址

```bash

1
2

sudo ss -tlnp | grep 11434
# 必须显示：0.0.0.0:11434

```

#### 配置防火墙（如果启用）

```bash

1
2

sudo ufw allow 11434/tcp
sudo ufw reload

```

#### 在客户端（如 Cherry Studio）配置

**API 地址**：`http://<你的服务器IP>:11434`

配置后，点击”管理”，应能自动获取服务器上的模型列表（如 llama3.2:1b）。

---

## 六、显存管理与模型选择

### 6.1 官方显存建议

根据 Ollama 官方文档：

> 

为确保可接受的性能，模型大小应至少比服务器可用 RAM 小两倍，并且是 GPU 可用显存的 2/3。

### 6.2 计算示例

#### GPU 显存推荐最大模型

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

[Service]
Environment="OLLAMA_HOST=0.0.0.0"
Environment="OLLAMA_MODELS=/data/.ollama/models"
Environment="OLLAMA_GPU_LAYERS=100"
Environment="OLLAMA_NUM_PARALLEL=4"
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
sudo systemctl status ollama

# 查看 GPU 识别
journalctl -u ollama | grep "inference compute"

# 监控 GPU 使用
nvidia-smi -l 1

# 测试模型
ollama run llama3.2:1b "Hello"

```

---

## 九、结论

本次部署成功实现了以下目标：

✅ 在 Ubuntu Server 24.04.4 LTS 上部署 Ollama v0.16.3
✅ 正确配置 NVIDIA RTX 3060 GPU 加速
✅ 实现双显卡分工：核显负责显示，独显负责计算
✅ 通过 Cherry Studio 实现局域网访问
✅ 遵循 systemd 最佳实践，使用 override.conf 配置
✅ 实现系统与数据分离（模型存储在独立 800GB 数据盘）

### 性能提升

- 推理速度从分钟级提升到秒级（**约 200 倍提升**）

- GPU 显存得到有效利用（约 8GB / 12GB）

- 支持更大的模型运行

- 系统稳定性显著提高

### 配置要点

- 使用官方安装脚本确保包含 CUDA 库

- 使用 systemd override.conf 进行配置管理

- 设置 `CUDA_VISIBLE_DEVICES` 指定 GPU

- 创建独立数据盘用于模型存储

- 配置 NVIDIA 仅用于计算（modeset=0）

- 使用国内镜像加速模型下载

---

## 十、参考来源

- [Ollama 官方硬件支持](https://docs.ollama.com/hardware-support)

- [Ollama GitHub GPU 配置](https://github.com/ollama/ollama/blob/main/docs/gpu.md)

- [Ollama Linux 安装](https://docs.ollama.com/linux)

- [NVIDIA 驱动与 CUDA 配置指南](https://docs.nvidia.com/)

---

**文档版本**：2.2
**最后更新**：2026年3月14日
**适用系统**：Ubuntu Server 24.04.4 LTS | Ollama v0.16.3 | NVIDIA RTX 3060 12GB
**核心硬件配置**：AMD CPU with iGPU + NVIDIA dGPU (Compute-Only)
**核心部署经验**：手动离线安装 + Systemd Override 配置

**部署环境**：Ubuntu Server 24.04.4 LTS | Ollama v0.16.3 | NVIDIA RTX 3060 12GB