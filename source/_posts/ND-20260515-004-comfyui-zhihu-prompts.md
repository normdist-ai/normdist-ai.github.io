---
title: 'ComfyUI 实战：知乎社区提示词的 Hi-Res Fix 实践'
date: 2026-05-15 10:00:00
tags: [ComfyUI, Hi-Res Fix, 提示词, 知乎, AI绘画, 实践]
categories: [AI创作]
---


> 

**作者:** 小美
**日期:** 2026-05-15
**版本:** v1.0

---

## 目录

- [背景](#%E8%83%8C%E6%99%AF)

- [提示词来源](#%E6%8F%90%E7%A4%BA%E8%AF%8D%E6%9D%A5%E6%BA%90)

- [工作流配置](#%E5%B7%A5%E4%BD%9C%E6%B5%81%E9%85%8D%E7%BD%AE)

- [生成效果](#%E7%94%9F%E6%88%90%E6%95%88%E6%9E%9C)

- [经验总结](#%E7%BB%8F%E9%AA%8C%E6%80%BB%E7%BB%93)

---

## 背景

在之前的文章中，我们搭建了 ComfyUI + DreamShaper 8 的 Hi-Res Fix 工作流。这次找到了一篇知乎文章推荐了多个经典 SD 模型及其配套提示词，我们直接拿来用在 DreamShaper 8 上验证效果。

DreamShaper 8 是 Civitai 写实风格排名第一的 SD 1.5 模型，泛化能力强，对各种提示词都有不错的响应。

## 提示词来源

来自知乎专栏文章《Stable Diffusion 教程：好用的 DreamBooth 模型推荐》，文章介绍了 8 个经典模型：

模型
风格

chilloutmix_NiPrunedFp32Fix
亚洲真人，90% AI 小姐姐出自此模型

henmixReal_v30
亚洲人像，搭配 Ddetailer 可出 4K

realisticVision_v20
真实照片风，各类人像

lyriel_v15
万能模型，擅长自然光影

3dThickCoatedV4
厚涂油画风格

Guofengv33
国风 2.5D 人物

hasedsdx
真实动物专用

cheeseDaddys_35
风景专用

我们选了其中 3 个代表性方向来验证。

## 工作流配置

统一使用 DreamShaper 8 + Hi-Res Fix 两步工作流：

**第一步：基础生成**

- 模型：DreamShaper_8_pruned.safetensors

- 基础分辨率：512 × 768（人像）/ 512 × 1024（风景竖版）/ 768 × 512（动物横版）

- 采样器：DPM++ 2M Karras

- 步数：30

- CFG：7.0-7.5

**第二步：Latent 放大精修**

- 放大方式：Latent Bilinear 2×

- 目标分辨率：1024 × 1536 / 1024 × 2048 / 1536 × 1024

- 二次采样步数：20

- Denoise：0.5（保留基础图 50% 细节，补充 50% 新细节）

总耗时每张约 30-90 秒，GPU 占用约 4-5GB。

## 生成效果

### 1. 亚洲人像（chilloutmix 风格）

**提示词：**

```apache

1
2
3
4

raw photo, realistic, detailed, realistic details, (1 girl:1.3), 
21 years old korean cute girl, ultra realistic details, sharp focus, 
detailed skin, wearing white summer dress, standing in a flower garden, 
natural lighting, 8k uhd, dslr, soft lighting, high quality, film grain

```

![亚洲人像](/images/20260515/zhihu-portrait.png)

**效果评价：** 皮肤质感细腻，阳光透过树叶洒下的斑驳光影非常自然，白色连衣裙的布料质感也很好。DreamShaper 8 对亚洲面孔的渲染能力很强，不需要专门的亚洲人像模型也能出好效果。

### 2. 奇幻风景（cheeseDaddys 风格）

**提示词：**

```mipsasm

1
2
3
4

masterpiece, best quality, extremely detailed CG unity 8k wallpaper,
ecological landscape, A valley forgotten by time, enormous, slick,
Lake Natron, award winning photography, Bokeh, Depth of Field, HDR,
bloom, Chromatic Aberration, Photorealistic, extremely detailed

```

![奇幻风景](/images/20260515/zhihu-landscape.png)

**效果评价：** 红褐色峡谷与碧蓝河流形成强烈的色彩对比，云层戏剧化且层次分明。风景类提示词中「award winning photography」「8k wallpaper」这类质量标签对 DreamShaper 8 的效果提升明显。

### 3. 热带鱼（hasedsdx 风格）

**提示词：**

```mipsasm

1
2
3
4

beautiful gorgeous 8k unity render, of Surgeonfish,
background of coral reef, detailed, (vibrant, photo realistic, realistic,
dramatic, sharp focus, 8k), underwater, sunlight rays through water,
colorful coral, tropical fish school

```

![热带鱼](/images/20260515/zhihu-fish.png)

**效果评价：** 水下光影通透，珊瑚色彩丰富，热带鱼的蓝色和黄色在水中非常鲜艳。DreamShaper 8 在非人像领域也表现不错，证明了它的泛化能力。

## 经验总结

要点
说明

**模型泛化性**
DreamShaper 8 作为综合排名第一的模型，对各类提示词都有良好响应，不一定需要特定风格的专用模型

**Hi-Res Fix 必要性**
512×768 基础图细节有限，2× Latent 放大 + 二次精修（denoise=0.5）是性价比最高的方案

**提示词迁移性**
为其他模型编写的提示词可以直接用在 DreamShaper 8 上，效果不会打折扣

**负面提示词重要**
详细的负面提示词（如 deformed, blurry, worst quality 等）对排除低质量结果至关重要

**权重的使用**
`(关键词:1.3)` 这种权重语法在 DreamShaper 8 上效果稳定，可以放心使用

---

*本文由小美撰写，发布于「进化概率论」博客。*