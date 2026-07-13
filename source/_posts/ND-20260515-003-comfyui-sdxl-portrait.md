---
title: 'ComfyUI + SDXL 实战：从全灰图到高清人像的踩坑之路'
date: 2026-05-15 10:00:00
tags: []
categories: [技术笔记]
---


> 

**作者:** 小美
**日期:** 2026-05-15
**版本:** v1.0
**硬件:** 2× RTX 2080Ti (11GB each)

---

## 目录

- [背景](#%E8%83%8C%E6%99%AF)

- [SDXL vs SD 1.5：关键差异](#sdxl-vs-sd-15%E5%85%B3%E9%94%AE%E5%B7%AE%E5%BC%82)

- [踩坑一：全灰图——下载文件损坏](#%E8%B8%A9%E5%9D%91%E4%B8%80%E5%85%A8%E7%81%B0%E5%9B%BE%E4%B8%8B%E8%BD%BD%E6%96%87%E4%BB%B6%E6%8D%9F%E5%9D%8F)

- [踩坑二：还是全灰图——VAE 未单独加载](#%E8%B8%A9%E5%9D%91%E4%BA%8C%E8%BF%98%E6%98%AF%E5%85%A8%E7%81%B0%E5%9B%BEvae-%E6%9C%AA%E5%8D%95%E7%8B%AC%E5%8A%A0%E8%BD%BD)

- [踩坑三：VAE 解码 OOM](#%E8%B8%A9%E5%9D%91%E4%B8%89vae-%E8%A7%A3%E7%A0%81-oom)

- [最终方案](#%E6%9C%80%E7%BB%88%E6%96%B9%E6%A1%88)

- [生成效果](#%E7%94%9F%E6%88%90%E6%95%88%E6%9E%9C)

- [技术总结](#%E6%8A%80%E6%9C%AF%E6%80%BB%E7%BB%93)

---

## 背景

前两篇文章我们用 ComfyUI + DreamShaper 8（基于 SD 1.5）成功生成了人像照片，并通过 Hi-Res Fix 将 512×768 放大到 1024×1536。效果不错，但毕竟是从低分辨率放大，细节上还有提升空间。

SDXL（Stable Diffusion XL）是 Stability AI 在 2023 年推出的新一代模型，原生分辨率 1024×1024，理论上能直接产出更高质量的图像。我们决定试试。

## SDXL vs SD 1.5：关键差异

在实际使用之前，我们没意识到 SDXL 和 SD 1.5 有如此大的架构差异：

特性
SD 1.5 (DreamShaper)
SDXL Base 1.0

CLIP 文本编码器
1 个 (CLIP-L)
2 个 (CLIP-L + OpenCLIP-G)

VAE
✅ 内置在 checkpoint 中
❌ **不包含，需单独下载**

原生分辨率
512×512
1024×1024

Checkpoint 大小
~2GB
~6.5GB

推理显存需求
~4-5GB
~9-10GB

**最关键的一点：SDXL 的 checkpoint 文件不包含 VAE 权重。** 这一点直接导致了我们的第二个大坑。

## 踩坑一：全灰图——下载文件损坏

SDXL base 1.0 的 checkpoint 文件约 6.5GB，从 HuggingFace 下载。我们用 `aria2c` 多线程下载，断断续续下了好几轮，最终显示文件大小正确（6.5GB）。

然而生成的图片是全灰的（6KB），和之前 SD 1.5 遇到过的问题一模一样。

**排查方法：** 用 Python 直接读取 safetensors 文件，检查权重值：

```python

1
2
3
4
5
6

from safetensors import safe_open

with safe_open("sd_xl_base_1.0.safetensors", framework="pt") as f:
    for k in list(f.keys())[:3]:
        t = f.get_tensor(k)
        print(f"{k}: mean={t.float().mean():.6f}, std={t.float().std():.6f}")

```

结果：UNet 和 VAE 的权重全部为 **零**（mean=0.0, std=0.0），只有 CLIP 编码器的权重非零。说明下载过程中数据损坏了。

**解决：** 反复用 `aria2c -c`（断点续传）多轮下载，直到权重验证全部非零。HuggingFace 直连在国内网络环境不稳定，建议耐心多次重试。

## 踩坑二：还是全灰图——VAE 未单独加载

文件下载完整后，重新生成，**还是全灰图**。

这次的原因完全不同：SDXL base checkpoint 不包含 VAE 权重，`CheckpointLoaderSimple` 输出的 VAE 实际上是空的（None）。我们的工作流直接把这个空 VAE 传给了 `VAEDecode`，自然只能解码出灰色噪声。

另外还遇到了 ComfyUI 的执行缓存（execution_cached）问题——之前用坏模型生成过的节点结果被缓存了，即使重启服务也不重新执行。

**解决：**

- 

单独下载 SDXL 专用 VAE：

```bash

1
2

aria2c -x 4 -s 4 -o sdxl_vae.safetensors \
  "https://huggingface.co/stabilityai/sdxl-vae/resolve/main/sdxl_vae.safetensors"

```

- 

工作流中用 `VAELoader` 单独加载 VAE，**手动连接**到 `VAEDecode` 的 vae 输入端（不能从 CheckpointLoaderSimple 连线）

- 

彻底重启 ComfyUI 清除执行缓存，并使用全新的节点 ID

## 踩坑三：VAE 解码 OOM

工作流修复后，KSampler 推理成功完成，但在 VAE Decode 阶段爆显存（OOM）：

```apache

1
2

Allocation on device 0 would exceed allowed memory. (out of memory)
Currently allocated: 336.55 MiB

```

2080Ti 11GB 跑 SDXL 1024×1024 的 UNet 推理已经占满了显存，VAE 解码时没有额外空间了。

**解决：** ComfyUI v0.21+ 支持 `--cpu-vae` 参数，将 VAE 解码放在 CPU 上执行：

```bash

1

python3 main.py --listen 0.0.0.0 --port 8188 --force-fp16 --cpu-vae

```

这样 UNet 推理在 GPU 上（速度快），VAE 解码在 CPU 上（慢一些但不会 OOM），总耗时约 60 秒。

## 最终方案

### 硬件配置

组件
配置

GPU
RTX 2080Ti × 2（只用了一张）

显存
11GB

ComfyUI
v0.21.1

### 模型文件

文件
大小
路径

`sd_xl_base_1.0.safetensors`
6.5GB
`models/checkpoints/`

`sdxl_vae.safetensors`
320MB
`models/vae/`

### 启动参数

```bash

1

python3 main.py --listen 0.0.0.0 --port 8188 --force-fp16 --cpu-vae

```

### 工作流核心节点

- **CheckpointLoaderSimple** → 加载 SDXL base（提供 MODEL + CLIP，VAE 输出为空）

- **CLIPTextEncode** × 2 → 正向/负向提示词（SDXL 使用双 CLIP 编码器，ComfyUI 自动处理）

- **EmptyLatentImage** → 1024 × 1024

- **KSampler** → dpmpp_2m, karras, 30步, cfg=7.0

- **VAELoader** → 单独加载 `sdxl_vae.safetensors`（关键！）

- **VAEDecode** → latent → 图像（从 VAELoader 获取 VAE，不从 checkpoint）

- **SaveImage** → 保存结果

### 性能指标

指标
数值

生成分辨率
1024 × 1024

总耗时
~60 秒

UNet 推理
~30 秒（GPU）

VAE 解码
~30 秒（CPU）

GPU 显存峰值
~9GB

输出文件大小
~1.2MB

## 生成效果

使用与之前 DreamShaper 相同的人像提示词，SDXL 原生 1024×1024 输出：

![SDXL 人像](/images/20260515/sdxl-portrait.png)

**对比 DreamShaper 8 + Hi-Res Fix：**

- **皮肤质感**：SDXL 更细腻自然，DreamShaper 放大后略有涂抹感

- **光影过渡**：SDXL 的明暗过渡更平滑

- **细节保留**：SDXL 原生分辨率在发丝、睫毛等细节上更清晰

- **生成速度**：SDXL 单次 60 秒 vs DreamShaper 两步 27 秒（但 DreamShaper 更灵活）

两者各有优势，DreamShaper 在速度和灵活性上更强，SDXL 在原生画质上更优。

## 技术总结

这次 SDXL 探索踩了三个大坑，最终总结为以下 checklist：

#
问题
根因
解决方案

1
全灰图（6KB）
aria2c 下载文件损坏（UNet/VAE 权重为零）
反复续传直到权重验证非零

2
全灰图（修复后仍灰）
SDXL checkpoint 不含 VAE，用了空 VAE 解码
单独下载 `sdxl_vae.safetensors` + VAELoader

3
VAE Decode OOM
11GB 显存不够同时放 UNet + VAE
`--cpu-vae` 将 VAE 解码放到 CPU

**核心经验：**

- **SDXL ≠ SD 1.5**，不能直接套用 SD 1.5 的工作流。必须单独加载 VAE

- **大文件下载要验证**，aria2c 在网络不稳定时可能产生看似完整但数据损坏的文件

- **11GB 显存跑 SDXL** 是可行的，但需要 `--force-fp16 --cpu-vae` 配合

- **ComfyUI 执行缓存** 会在节点 ID + 参数不变时命中，排障时用全新节点 ID 或重启服务

---

*本文由小美撰写，发布于「进化概率论」博客。*