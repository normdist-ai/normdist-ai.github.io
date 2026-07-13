---
title: 'Hermes Agent Gateway 多实例配置实践'
date: 2026-05-20 10:00:00
tags: [AI写真, ComfyUI, FaceID, 表情控制, Stable Diffusion]
categories: [技术笔记]
---

## 背景

用 AI 生成人物写真时，最头疼的问题之一是：**脸像了，但表情僵了**。

我用 ComfyUI + FaceID（IP-Adapter FaceID Plus V2）生成韩梅梅的虚拟写真。FaceID 的核心能力是**保持面部一致性**——不管换什么场景、穿什么衣服，脸都是同一个人。但代价也很明显：**表情被「锁死」了**。默认输出永远是平静、中性的表情，笑容若有若无。

这篇文章分享我如何突破这个限制，让虚拟人物真正「笑」起来。

## 核心问题

FaceID 的工作原理是通过面部嵌入（face embedding）约束生成结果。权重越高，面部越像参考图——但同时也越接近参考图的表情。参考图如果是平静脸，生成的就永远是平静脸。

**矛盾点**：我们需要同时满足两个目标——

- **保持面部特征**（像这个人）

- **改变表情**（让她笑）

这两个目标本质上是冲突的。面部嵌入既包含身份数据也包含表情数据，FaceID 无法只保留前者而丢弃后者。

## 解决方案：FaceDetailer + 表情注入

我的方案是**两阶段生成**：

1
2
3

第一阶段：FaceID 生成（保持身份 + 尝试注入表情）
    ↓
第二阶段：FaceDetailer 精修（独立提示词强化表情）

```

关键在于第二阶段。FaceDetailer 是 Impact-Pack 提供的节点，它会：

- 用 bbox 模型检测脸部区域

- 在检测到的脸部区域做局部重绘（inpainting）

- **使用独立的 CLIP 提示词**控制重绘内容

这个「独立的 CLIP 提示词」就是突破口。

### 具体实现

#### 第一步：FaceID 参数调整

当目标是「笑容」时，适当降低 FaceID 权重，给表情留出空间：

```python

1
2
3
4
5

# 平时（无表情要求）
faceid = {"weight": 1.0, "weight_faceidv2": 0.7, "end_at": 0.9}

# 需要笑容时（降低权重，释放表情自由度）
faceid = {"weight": 0.8, "weight_faceidv2": 0.55, "end_at": 0.8}

```

同时在主提示词中注入表情关键词：

```mipsasm

1
2

big grin, wide toothy smile, laughing joyfully, showing teeth,
a beautiful young Chinese woman, ...

```

这一步能产生「有笑容倾向」的基础图，但还不够明显——因为 FaceID 仍在约束。

#### 第二步：FaceDetailer 表情强化

这是关键一步。为 FaceDetailer 构建专用的表情提示词：

```python

1
2
3
4
5

fd_expression_prompt = (
    f"(big grin, wide toothy smile, laughing joyfully, showing teeth:1.4), "
    f"beautiful detailed face, detailed eyes, "
    f"natural skin texture, perfect facial features, {prompt}"
)

```

注意几个要点：

- **权重 `(expression:1.4)`**：比普通提示词更高的权重，强制模型关注表情

- **保留面部质量关键词**：`beautiful detailed face, natural skin texture` 等，防止重绘时破坏面部细节

- **拼接主提示词**：保留场景上下文，避免脸部与背景脱节

FaceDetailer 参数配置：

```python

1
2
3
4
5
6

{
    "enabled": True,
    "denoise": 0.35,    # 不宜太高，否则会「画走」脸
    "cfg": 4.0,         # 略低于主采样，避免过度锐化
    "steps": 20,        # 足够收敛即可
}

```

#### 为什么这个方案有效

- **解耦了身份和表情**：FaceID 负责身份，FaceDetailer 负责表情，各司其职

- **局部操作**：FaceDetailer 只修改脸部区域，不影响身体、衣服、背景

- **低 denoise 精修**：0.35 的 denoise 意味着只做小幅调整，在已有面部基础上「添加笑容」，而不是重新画一张脸

- **独立提示词通道**：FaceDetailer 的提示词不影响主体生成，不会破坏构图和场景

## 效果对比

### Before：toothy smile（单纯靠主提示词）

表情变化微弱，FaceID 权重过高锁住了表情。看起来像是嘴角微微上扬，但不是真正的笑容。

### After：big grin + FaceDetailer 表情增强