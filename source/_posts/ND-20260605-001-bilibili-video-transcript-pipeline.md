---
title: '双卡2080Ti跑大模型：PCIe带宽是瓶颈吗？一次深度实测'
date: 2026-06-05 10:00:00
tags: [bilibili, whisper, python, 自动化, 音频转写]
categories: [技术笔记]
---

## 需求来了

故事的起点很普通：有个AI助手需要帮用户分析B站视频内容，给个文字摘要就行。

看起来不难——YouTube那边已经有现成的技能了，能搜视频、提字幕、出摘要，跑得很稳。但B站完全是另一回事：API不一样、反爬不一样、生态也不一样。

YouTube的技能架构能不能复用？能，但得重写数据层。

## 先想清楚再动手

分析了一下B站视频的内容获取路径，发现就三条路：

- **字幕API** — B站部分视频有字幕（UP主上传或AI生成），通过API可以直接拿到，秒级响应

- **音频转写** — 下载视频音轨，用Whisper做语音识别，分钟级响应

- **纯元信息** — 标题、简介、弹幕热词，勉强能猜个大概

关键数据：**B站视频有字幕的不超过一半**。也就是说，如果只走字幕API，一大半视频直接歇菜。

所以方案很自然——**三级降级**：先试字幕，没有就转写，转写也不行就只拿元信息。总有一条路能走通。

## 架构：两台机器，三级降级

手边有两台机器：一台日常用的小主机，一台带双GPU的推理服务器。音频下载在小主机完成，Whisper推理必须走GPU服务器——因为小主机没有CUDA。

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

小主机                               GPU 服务器
┌─────────────────┐                 ┌─────────────────┐
│ Level 1: 字幕API│──→ 有？直接返回  │                 │
│       ↓ 没有    │                 │                 │
│ B站 playurl API │                 │                 │
│  ↓ 获取音频流URL │                 │                 │
│ ffmpeg 下载 WAV  │ ─── SCP ──→   │ faster-whisper  │
│                 │                 │  (GPU 1)        │
│                 │ ←─ SSH stdout ─│ 转写结果(JSON)   │
└─────────────────┘                 └─────────────────┘

```

这里有个细节：GPU 0 上跑着ComfyUI（图片生成服务），不能抢显存。好在GPU 1还有18GB空闲，Whisper medium模型只需要约400MB，完全够用。两块GPU各干各的，互不干扰。

## 开始踩坑

### 坑1：yt-dlp被B站412反爬封杀

第一反应是用yt-dlp下载B站音频——这东西号称支持上千个网站，一行命令搞定。

结果：

```subunit

1

ERROR: [BiliBili] xxx: HTTP Error 412: Precondition Failed

```

直接被拦。B站的反爬策略升级非常频繁，yt-dlp社区在追赶，但2026年的B站明显跑在前面。

**那就绕过yt-dlp。**

B站的在线播放器本身就需要获取音视频流——它用的是playurl API，返回DASH格式（音频和视频分离）。既然播放器能用，我们也能用：

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
25
26
27
28
29
30
31
32
33

import requests, subprocess

# 请求DASH格式——音频视频分离
params = {
    'avid': avid,
    'cid': cid,
    'qn': 64,
    'fnval': 16,   # 关键：请求DASH格式
    'fourk': 1
}
resp = requests.get(
    "https://api.bilibili.com/x/player/playurl",
    params=params,
    headers={"User-Agent": "Mozilla/5.0 ..."}
)
data = resp.json()

# DASH格式下，音频是独立的流
audio_url = None
for audio in data['data']['dash']['audio']:
    if audio.get('codecid') == 0:  # AAC编码
        audio_url = audio['baseUrl']
        break

# ffmpeg直接下载音频流，转成Whisper喜欢的WAV格式
subprocess.run([
    'ffmpeg', '-y',
    '-headers', 'User-Agent: Mozilla/5.0 ...',
    '-i', audio_url,
    '-vn', '-acodec', 'pcm_s16le',
    '-ar', '16000', '-ac', '1',  # 16kHz单声道，Whisper最佳输入
    output_path
], capture_output=True)

```

这一招比yt-dlp还稳——毕竟用的是B站播放器自己的API。

### 坑2：Whisper模型下载被限速

GPU服务器在国内，直连HuggingFace下载faster-whisper-large-v3（3GB）基本不可能。hf-mirror有时能用有时不能。

**解法：** aria2c多线程下载，`-x 16 -s 16`把连接数拉满，配合hf-mirror：

```bash

1
2
3

aria2c -x 16 -s 16 \
  "https://hf-mirror.com/Systran/faster-whisper-large-v3/resolve/main/model.bin" \
  -d /path/to/model/dir/

```

最后3GB的文件下了大概十分钟，还算能接受。

### 坑3：模型加载报”construction from null”

large-v3的model.bin下载完了（3.09GB），信心满满地加载，结果：

```pgsql

1

RuntimeError: basic_string: construction from null is not valid

```

一脸懵。排查了半天，发现是文件结构的问题——手动下载的model.bin和HF cache里的配置文件格式不匹配。faster-whisper的base模型用`vocabulary.txt`，而large-v3用`vocabulary.json`。手动下载时只下了model.bin，配置文件是从base模型目录复制的，编码格式对不上。

**临时方案：** 放弃large-v3，改用medium模型。让HF hub自动下载完整文件结构（config.json + vocabulary.txt + tokenizer.json + model.bin），不再手动拼凑。medium的中文识别率已经够用，后续再处理large-v3的问题。

教训：**模型文件不是只下model.bin就完事了，配置文件和词表必须配套。**

### 坑4：faster-whisper的GPU选择

Whisper必须跑在GPU 1上，不能占GPU 0。

直觉写法：`device="cuda:1"`——报错。

查了一下faster-whisper的API签名，发现`device`和`device_index`是**两个独立参数**：

```python

1
2
3
4
5
6
7
8

from faster_whisper import WhisperModel

model = WhisperModel(
    "medium",
    device="cuda",
    device_index=1,          # GPU索引是独立参数，不能拼在device字符串里
    compute_type="int8_float16"
)

```

小问题，但卡了十分钟。

## 远程转写的核心代码

GPU服务器端的转写脚本很简洁——接收音频文件，加载模型，转写，输出JSON到stdout：

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
25
26
27
28
29
30
31
32
33
34
35
36
37
38
39
40
41
42
43
44
45
46
47

#!/usr/bin/env python3
"""remote_whisper.py — 运行在GPU服务器上"""
import argparse, json, sys, time
from faster_whisper import WhisperModel

def main():
    parser = argparse.ArgumentParser()
    parser.add_argument("audio_file")
    parser.add_argument("--model", default="medium")
    parser.add_argument("--language", default="zh")
    parser.add_argument("--device", default="cuda")
    parser.add_argument("--device-index", type=int, default=1)
    parser.add_argument("--compute-type", default="int8_float16")
    args = parser.parse_args()

    t0 = time.time()
    model = WhisperModel(
        args.model,
        device=args.device,
        device_index=args.device_index,
        compute_type=args.compute_type
    )
    segments, info = model.transcribe(
        args.audio_file,
        language=args.language,
        beam_size=5
    )

    result = [
        {
            "start": round(seg.start, 2),
            "end": round(seg.end, 2),
            "text": seg.text.strip()
        }
        for seg in segments
    ]

    json.dump({
        "language": info.language,
        "duration": round(info.duration, 1),
        "segments": result,
        "model": args.model,
        "transcribe_time": round(time.time() - t0, 1)
    }, sys.stdout, ensure_ascii=False, indent=2)

if __name__ == "__main__":
    main()

```

本地主控脚本通过SCP传文件、SSH执行转写、解析JSON结果：

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
12
13
14
15
16
17

import subprocess, json

# 1. SCP传音频到GPU服务器
subprocess.run([
    "scp", "-o", "StrictHostKeyChecking=no",
    wav_path, f"user@gpu-server:/tmp/whisper-audio/{filename}"
])

# 2. SSH远程执行转写
result = subprocess.run([
    "ssh", "-o", "StrictHostKeyChecking=no",
    "user@gpu-server",
    f"python3 /tmp/remote_whisper.py /tmp/whisper-audio/{filename}"
], capture_output=True, text=True, timeout=300)

# 3. 解析结果
transcript = json.loads(result.stdout)

```

整个远程调用链路：**SCP传文件 → SSH执行 → stdout返回JSON**。简单粗暴，但非常可靠。

## 搜索怎么办

光能提取内容还不够，还得能搜B站视频。

B站的搜索API同样有412反爬问题——看来B站对所有非浏览器请求都不太友好。

**解法：** 本地部署了SearXNG搜索引擎，用`site:bilibili.com`限定搜索范围。从搜索结果里提取BV号，再调B站视频信息API获取元数据（标题、播放量、时长、UP主等）。

SearXNG的搜索结果精度不如B站原生搜索，但胜在稳定——不依赖cookie，不会被封。

## 实测

用一期29分钟的财经类视频做端到端测试：

指标
数据

视频时长
29:05

Level 1 检测
无字幕，自动降级到Level 2

音频大小
55.8 MB (WAV, 16kHz)

Whisper 转写时间
144秒

输出段落数
1263段（含时间戳）

模型
medium, int8_float16, GPU 1

GPU 显存占用
~400MB

从用户甩链接到看到文字摘要，全流程约3分钟。其中大部分时间花在音频下载和传输上，Whisper推理本身很快。

## 回头看

这套方案的核心思路其实就几条：

- **三级降级**——永远有兜底，不会空手而归

- **API直连代替第三方工具**——yt-dlp被拦，但B站自己的playurl API拦不住

- **硬件分工**——CPU活在本地干，GPU活通过SSH远程调用，各司其职

- **够用就好**——medium模型400MB显存换来可接受的中文识别率，不追求完美

从零到跑通，大约一个下午。大部分时间花在踩坑上——B站反爬、模型下载、GPU分配、API格式。但每个坑解决后都有记录，下次不会重蹈覆辙。

如果有人也在做类似的事情，希望这些踩坑记录能帮你省点时间。