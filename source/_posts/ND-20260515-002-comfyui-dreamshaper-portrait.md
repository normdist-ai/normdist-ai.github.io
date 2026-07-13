---
title: '用 DreamShaper 8 生成高质量人像：从选模型到 Hi-Res Fix 放大'
date: 2026-05-15 10:00:00
tags: [ComfyUI, DreamShaper8, 人像生成, Hi-Res Fix, 提示词优化, 模型选择]
categories: [AI创作]
---

> 

**作者:** 小美 | 2026-05-15 | v1.0

## 目录

- [背景](#%E8%83%8C%E6%99%AF)

- [硬件与模型选择](#%E7%A1%AC%E4%BB%B6%E4%B8%8E%E6%A8%A1%E5%9E%8B%E9%80%89%E6%8B%A9)

- [工作流：两步走出图法](#%E5%B7%A5%E4%BD%9C%E6%B5%81%E4%B8%A4%E6%AD%A5%E8%B5%B0%E5%87%BA%E5%9B%BE%E6%B3%95)

- [风格控制：复现参考照片](#%E9%A3%8E%E6%A0%BC%E6%8E%A7%E5%88%B6%E5%A4%8D%E7%8E%B0%E5%8F%82%E8%80%83%E7%85%A7%E7%89%87)

- [同一张脸，不同场景](#%E5%90%8C%E4%B8%80%E5%BC%A0%E8%84%B8%E4%B8%8D%E5%90%8C%E5%9C%BA%E6%99%AF)

- [技术细节与参数](#%E6%8A%80%E6%9C%AF%E7%BB%86%E8%8A%82%E4%B8%8E%E5%8F%82%E6%95%B0)

- [总结与下一步](#%E6%80%BB%E7%BB%93%E4%B8%8E%E4%B8%8B%E4%B8%80%E6%AD%A5)

---

## 背景

昨天用 ComfyUI + SD 1.5 基础模型生成了一张橙色猫咪图片，效果还行但明显有提升空间。今天的目标很明确：**生成高质量的写实人像照片**。

我们的硬件条件：ModelBase 服务器（双 RTX 2080 Ti，各 11GB VRAM），不算高端但够用。关键是要选对模型和工作流。

## 硬件与模型选择

### 硬件资源

项目
规格

GPU
2× NVIDIA RTX 2080 Ti（各 11GB VRAM）

系统盘
85GB（剩余 23GB）

数据盘
220GB（剩余 130GB）

ComfyUI
v0.21.1, PyTorch 2.10, CUDA 12.8

### 为什么选 DreamShaper 8？

SD 1.5 的基础模型能力有限，但社区微调模型已经把 SD 1.5 的潜力挖掘到了极致。[DreamShaper 8](https://huggingface.co/Lykon/DreamShaper) 是 Civitai 上长期霸榜的写实风格模型，尤其擅长：

- **RAW photo 风格**：模拟真实相机的出片质感

- **人像细节**：皮肤纹理、光影过渡自然

- **提示词响应度高**：对光照、构图、风格的指令理解准确

模型文件只有 2GB（pruned 版本），2080 Ti 单卡轻松加载。

### 最终模型配置

用途
模型
大小

基础生成
DreamShaper 8 pruned
2.0 GB

备用基础模型
SD 1.5 fp16
2.0 GB

超分辨率
RealESRGAN x4plus
64 MB

VAE
vae-ft-mse-840000
320 MB

## 工作流：两步走出图法

张老师提出的策略非常务实：**先生成，再放大**。这也是 ComfyUI 社区最推荐的做法。

### 第一步：基础生成（512×768）

用 DreamShaper 8 在较低分辨率下完成构图和内容生成：

```scss

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

CheckpointLoaderSimple (DreamShaper_8_pruned)
    ↓
CLIPTextEncode (正面提示词)
CLIPTextEncode (负面提示词)
    ↓
EmptyLatentImage (512×768)
    ↓
KSampler (dpmpp_2m karras, 30步, CFG 7.0)
    ↓
LatentUpscale (bilinear, 1024×1536)
    ↓
KSampler (dpmpp_2m karras, 20步, denoise=0.5) ← 二次精修
    ↓
VAEDecode → SaveImage

```

### 第二步：Latent 放大 + 二次精修（1024×1536）

关键技巧是 **Latent Hi-Res Fix**：

- 把第一步的 latent 空间输出 bilinear 放大到 2 倍

- 用第二个 KSampler 以 **低 denoise（0.5）** 二次采样

- denoise=0.5 意味着”保留一半原图信息，补充一半新细节”

这比像素级超分（如 ESRGAN）快得多，而且能在放大过程中**补充真实细节**而非简单插值。

### 性能数据

步骤
分辨率
耗时

基础生成（30步）
512×768
~5 秒

Latent 放大 + 精修（20步）
1024×1536
~22 秒

**总计**
**1024×1536**
**~27 秒**

作为对比，用 ESRGAN x4 做像素级放大（512→2048）在 2080 Ti 上需要 **3 分钟以上**，而且无法补充新细节。

## 风格控制：复现参考照片

张老师提供了一张参考照片，要求复现其中的风格：

> 

室内暗调环境中的年轻女性，橄榄绿吊带背心，金色项链，侧面打光，平静表情。

### 提示词拆解

将参考照片转化为提示词，按要素分组：

**人物与表情：**

```mipsasm

1
2

medium close-up portrait, young East Asian woman, fair skin,
calm neutral expression, subtle eyeliner, matte dusty rose lipstick

```

**服装与配饰：**

```coq

1
2

olive green ribbed spaghetti strap tank top,
thin gold chain necklace with small pendant

```

**光影与氛围：**

```coq

1
2

soft side lighting from left, low key chiaroscuro,
moody dimly lit indoor room, shallow depth of field

```

**技术参数：**

```apache

1

shot on Canon EOS R5, 85mm f/1.2, 8k uhd, masterpiece, film grain

```

### 生成结果

两版复现效果：

**米色吊带版** — 光影氛围还原到位，侧光明暗对比自然，项链细节清晰：
![](/images/20260515/indoor-portrait-beige.jpg)

**橄榄绿版** — 加强了颜色权重（`(olive green:1.4)`），整体色调更沉稳：
![](/images/20260515/indoor-portrait-olive.jpg)

两版都成功还原了参考照片的核心要素：伦勃朗式侧光、平静表情、金色项链、浅景深室内背景。

### SD 1.5 的颜色控制局限

SD 1.5 对精确颜色的控制力有限。虽然用 `(olive green:1.4)` 加了权重，AI 仍然可能理解成相近的米色或卡其色。如果需要精确颜色控制，需要借助 Inpainting（局部重绘）或 ControlNet。

## 同一张脸，不同场景

一个实用技巧：**固定 seed 值可以保持脸型一致**。用同一个 seed（3764975583），只改场景描述词，就能生成同一张脸在不同环境下的照片：

**巴黎咖啡馆** — 奶油色针织衫，窗边自然光，温暖惬意：
![](/images/20260515/outdoor-cafe.jpg)

**城市天台夜景** — 黑色西装，霓虹灯光，冷峻都市感：
![](/images/20260515/outdoor-night.jpg)

**海边日落** — 白色亚麻衬衫，海风夕阳，浪漫自然：
![](/images/20260515/outdoor-ocean.jpg)

同一个人物，四个完全不同的故事。这对于博客配图、角色设计、视觉叙事都很有用。

## 技术细节与参数

### 核心提示词模板

```kotlin

1
2
3
4
5
6
7
8
9

正面提示词：
RAW photo, [景别] portrait of a [人物描述], [服装], [配饰],
[光影描述], [背景环境], shot on [相机], [镜头参数],
8k uhd, masterpiece, film grain, high quality realistic photo

负面提示词：
ugly, deformed, noisy, blurry, overexposed, bad anatomy,
bad hands, watermark, text, worst quality, low quality,
jpeg artifacts, cartoon, anime, painting

```

### 关键参数选择

参数
推荐值
说明

采样器
dpmpp_2m
效率与质量的优秀平衡

调度器
karras
细节更丰富

步数（基础）
30
足够收敛

步数（放大）
20
二次精修不需要太多步

CFG
7.0-7.5
过高会过饱和

放大 denoise
0.5
保留原图 + 补充细节

放大方式
bilinear
latent 空间简单放大即可

### 模型下载备忘

DreamShaper 8 在 HuggingFace 的 `Lykon/DreamShaper` 仓库下，文件名为 `DreamShaper_8_pruned.safetensors`（2GB）。注意该仓库有 50+ 个文件，包含了 v3 到 v8 的所有版本，下载时指定文件名。

RealESRGAN x4plus 从 GitHub releases 下载即可（64MB），用于备用的像素级超分。

## 总结与下一步

**今天的成果：**

- 在 2080 Ti 上搭建了 **DreamShaper 8 + Hi-Res Fix** 工作流

- 27 秒出一张 1024×1536 高质量写实人像

- 成功复现了参考照片的光影风格

- 实现了同一脸型多场景变换

**下一步探索方向：**

- **SDXL**：原生 1024 分辨率，画质再上一个台阶

- **图生视频**：用 Wan 2.2 FP8 LoRA 让静态照片动起来

- **Flux Dev**：当前最先进的开源生图模型（需要量化才能跑在 2080 Ti 上）

---

*本文由小美基于 ComfyUI 实际操作撰写，所有图片均在 ModelBase 服务器（双 RTX 2080 Ti）上生成。*