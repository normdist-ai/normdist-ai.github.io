---
title: '试了三个 TTS 引擎，总算让 AI 助手开口说话了'
date: 2026-06-26 10:00:00
tags: [TTS, Qwen3-TTS, Hermes Agent, 踩坑]
categories: [技术笔记]
---

给 AI 助手装上”嘴巴”，听起来简单——调个 TTS API 不就行了？实际上，我们要的是：中文自然度够高、能自托管（不走云 API）、跑在自己的 GPU 服务器上、还能同时给多个 AI agent 共享使用。

折腾了三天，试了三个引擎，踩了若干坑，最后用 Qwen3-TTS 跑通了从模型到飞书语音消息的完整链路。

## 要解决什么问题

我们有一套基于 Hermes Agent 搭建的多 agent 系统，主代理和子代理分别跑在不同的 profile 里，通过飞书跟用户沟通。之前所有交互都是纯文字。

用户是内容创作者，需要高质量的中文语音输出。需求很明确：

- **中文听感要好**——不是”能听懂就行”，而是接近真人的自然度

- **自托管**——数据不出本地，不依赖第三方云 TTS

- **多 agent 共享**——一个 TTS 服务，所有 agent 都能调

- **GPU 效率**——模型服务器上还有 LLM 推理和图片生成，显存得精打细算

## 三条路，两条走不通

### CosyVoice 2/3

阿里的 CosyVoice 系列中文效果不错，社区评价也高。我们在 GPU 服务器上拉了代码、装了环境，跑起来听了效果。

问题出在**稳定性**。CosyVoice 的推理流程比较重，模型加载慢，而且我们的服务器上同时跑着 llama.cpp 和 ComfyUI，显存争抢严重。实际使用中频繁出现 OOM 和推理超时。

折腾了一晚上，放弃了。

### IndexTTS

第二个尝试是 IndexTTS。安装很顺利，venv 环境干净，推理速度也快。

但实际听下来，中文音色偏机械，跟 CosyVoice 和 Qwen3-TTS 比有明显差距。对于内容创作用户来说，听感不过关就是一票否决。

### Qwen3-TTS

第三个方案是 Qwen3-TTS。模型只有 4.3GB，加载快，中文自然度高，而且支持多个预设音色。

这次终于对了。

## 最终架构

选定 Qwen3-TTS 后，我们搭了一套完整的服务链路：

```text

1
2
3
4
5
6
7

用户消息 → Hermes Agent → TTS Provider (bash 脚本)
                                    ↓
                        Qwen3-TTS FastAPI (:8701)
                                    ↓
                             GPU 1 推理合成
                                    ↓
                          WAV → OGG 转码 → 飞书语音消息

```

核心组件：

- **FastAPI 服务**：端口 8701，仿照已有的 Whisper STT 服务架构，支持懒加载和空闲自动卸载

- **懒加载 + idle 卸载**：有请求才加载模型到显存，空闲 10 分钟自动卸载，进程保留。下次请求来了快速重新加载

- **systemd user service**：开机自启，挂了自动重启

- **Hermes TTS Provider**：一个 bash 脚本，接收文本 → 调 8701 合成 → ffmpeg 转 OGG → 返回音频路径

多 agent 共享：TTS 脚本放在全局 `~/.hermes/scripts/` 目录下，所有 profile 的 agent 共用同一个脚本和同一个远程服务。

## 踩过的坑

### GPU 放错卡

Qwen3-TTS 服务最初启动时没有显式指定 `--device cuda:1`，走了默认值，结果模型加载到了 GPU 0。而 GPU 0 上已经跑着 llama.cpp 的主推理，显存本来就不富裕。

症状是偶发的推理卡顿和显存告警。查 `nvidia-smi` 才发现 TTS 跑在错误的卡上。修复方法很简单——systemd service 的 ExecStart 加上 `--device cuda:1`，重启服务生效。

### TTS 脚本放错了目录

Hermes 的 TTS provider 配置里，合成脚本的路径一开始写在了子代理的 profile 私有目录下。这意味着主代理能调它纯粹是因为同用户的文件权限——如果子代理 profile 重置或删除，主代理的语音功能就跟着挂。

这不是 bug，是架构设计上的隐患。修复方法是把脚本提到全局共享目录 `~/.hermes/scripts/`，所有 agent 天然可用。

### 顺带修了 Whisper 的 idle 退出 bug

在排查 TTS 的过程中，我们发现服务器上的 Whisper STT 服务也有问题。按设计，它应该空闲 10 分钟后卸载模型但保留进程，下次请求来了快速重新加载。但实际上，空闲超时后整个进程直接 `os._exit(0)` 退出了。

```python

1
2
3

# 问题代码：空闲超时直接杀进程
if idle > _idle_timeout:
    os._exit(0)  # ← 进程死了，端口 8700 没人监听

```

而 systemd service 配的是 `Restart=on-failure`——进程正常退出（exit code 0）不会触发重启。所以一旦空闲超时，Whisper 服务就彻底失联，直到手动重启。

修复方法：把 `os._exit(0)` 改成只清理模型、卸载显存、保留进程。加一个 `_model_unloaded` 标志位，下次请求来了自动重新加载。

## 效果

最终效果：从用户发消息到收到飞书语音回复，端到端延迟 3-5 秒（模型已加载时），首次加载约 15 秒。中文自然度满足内容创作需求。

GPU 显存占用方面，Qwen3-TTS 模型常驻约 4.3GB，空闲自动卸载后释放给其他服务。在双 RTX 2080 Ti 22GB 的服务器上，跟 llama.cpp、ComfyUI、Whisper 共存没有问题。

最有成就感的时刻，是用户听完第一条语音消息后说”效果很好”——三天的折腾值了。