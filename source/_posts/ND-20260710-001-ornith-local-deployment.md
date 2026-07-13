---
title: 'Ornith 开源模型本地部署：替代 Qwen3.6 的完整实战'
date: 2026-07-10 10:00:00
tags: [Ornith, 本地部署, llama.cpp, Qwen3.6, 开源模型]
categories: [技术笔记]
---

## 背景：为什么需要替代 Qwen3.6

本地部署大语言模型，硬件永远是第一约束。双卡 RTX 2080 Ti（22GB 魔改版）的显存总量约 22GB，面对 35B 参数的 MoE 模型，可用选项其实不多。

此前使用的 Qwen3.6-35B-A3B 在多数场景下表现尚可，但有两个问题一直悬而未决：

- **推理速度不理想**：在本地硬件上，Qwen3.6 的 token 生成速度始终在 40-60 t/s 区间，长输出场景下延迟明显

- **社区活跃度下降**：Qwen3.6 发布后，社区微调版本和优化方案较少，后续迭代前景不明

某天，社区中出现了一个新的名字——**Ornith**。基于 Qwen3.5-35B-A3B 优化训练，社区基准测试显示多项指标超越 Qwen3.6。更关键的是，这个模型完全兼容现有的 llama.cpp 部署方案，不需要额外适配。

## 部署全流程

### 硬件与软件环境

组件
规格

显卡
双 RTX 2080 Ti（22GB 魔改版）

推理引擎
llama.cpp（models-preset 多模型模式）

量化格式
Q4_K_M GGUF

代理层
new-api（透明转发，注册模型名）

张量分片
tensor-split 0.7:0.3

### 第一步：下载与量化

Ornith 的 GGUF 量化版可以从 Hugging Face 直接下载。选择 Q4_K_M 量化级别——这是一个平衡点：Q4_K_M 相比 Q4_K_S 保留更多关键层精度，相比 Q5/Q6 则节省约 4-6GB 显存。

```bash

1
2
3

# 下载 Ornith GGUF 模型
wget https://huggingface.co/deepreinforce-ai/ornith-1.0-35b-GGUF/resolve/main/ornith-1.0-35b-Q4_K_M.gguf \
  -O /mnt/nvme/models/ornith-1.0-35b-Q4_K_M.gguf

```

### 第二步：llama.cpp 配置（models.ini）

本地 llama.cpp 运行在 **models-preset 模式**下，通过 INI 文件管理多个模型。Ornith 的配置如下：

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

[Qwen/Qwen3.5-35B-A3B-Ornith]
alias = deepreinforce-ai/ornith-1.0-35b
model = /mnt/nvme/.../ornith-1.0-35b-Q4_K_M.gguf
mmproj = /mnt/nvme/.../mmproj-ornith-35b-f16.gguf
ctx-size = 262144
temp = 0.6
reasoning = on
jinja = true
chat-template-file = /home/user/llama.cpp/models/qwen35_ornith_fixed.jinja

```

注意 `tensor-split = 0.7,0.3` 在全局配置中设定——这是双卡张量分片的关键参数。0.7:0.3 的比例意味着第一张卡承担 70% 的计算量，第二张卡承担 30%。这个比例根据显存大小和 PCIe 带宽反复调优得出。

### 第三步：new-api 模型注册

在 new-api 代理层将 Ornith 注册为 `Qwen/Qwen3.5-35B-A3B-Ornith`。这样做的好处是：

- 可追溯：模型名包含基座和优化版本信息

- 兼容性：客户端无需修改，只需换模型名

- 区分度：和原版 Qwen3.5 区分开来，方便对比测试

```yaml

1
2
3
4
5

# new-api 配置节选
model:
  name: "Qwen/Qwen3.5-35B-A3B-Ornith"
  provider: "local-llama"
  base_url: "http://localhost:8080/v1"

```

### 第四步：重启与验证

修改 models.ini 后，必须重启 llama.cpp 主进程（systemd service）才能生效。局部卸载或 reload API 都不行——主进程缓存了 preset 配置。

```bash

1

sudo systemctl restart llama-server

```

验证方式：发送测试请求，用 `GET /v1/models` 确认子进程命令行参数中包含 `--jinja` 和 `--chat-template-file`。

## 踩坑实录：Chat Template 兼容性

部署中遇到的最大问题不是性能，而是 **chat template 兼容性**。

Ornith 的 GGUF 内嵌 jinja 模板中包含 `raise_exception()` 调用——这是 Qwen 系列模板的常规做法，用于校验输入格式。但 llama.cpp 的 minijinja 引擎实现不完全兼容标准 Python Jinja2，导致 `raise_exception()` 在解析阶段就崩溃。

**错误现象**：

```json

1
2
3
4
5
6
7

HTTP 400: {
  "error": {
    "message": "Unable to generate parser for this template. 
    Automatic parser generation failed: 
    ...raise_exception('No user query found in messages.')..."
  }
}

```

100% 可复现：任何没有 `role: user` 的请求（包括 tools 探测请求）都会触发。这意味着正常的 MoA 聚合请求、system-only 消息都会失败。

**修复方案**：使用 `--chat-template-file` 覆盖 GGUF 内嵌模板，不修改模型文件本身。

```bash

1
2
3
4
5
6
7
8

# 下载修复版模板（来自 unsloth 的 Qwen3.5 模板）
wget https://huggingface.co/unsloth/Qwen3.5-35B-A3B/raw/main/chat_template.jinja \
  -O /opt/models/chat_template_fixed.jinja

# 启动时覆盖
llama-server --model ornith-1.0-35b-Q4_K_M.gguf \
  --jinja \
  --chat-template-file /opt/models/chat_template_fixed.jinja

```

修复版模板删除了两处 `raise_exception`：

- `raise_exception('No user query found in messages.')`

- `raise_exception('System message must be the beginning.')`

对正常输入（有 system + user），渲染输出 byte-for-byte 一致，不影响任何功能。

## 性能实测：104 t/s 背后的 MoE 原理

### 测试结果

测试项
数值

模型参数量
35B（MoE，激活参数约 3.5B）

量化
Q4_K_M

上下文长度
262K

长输出速度
**104 t/s**

首 token 延迟
~1.2s

### 理论分析：为什么 104 t/s 是合理的

Ornith 是 MoE（Mixture of Experts）架构。35B 总参数中，每次推理只激活约 3.5B 参数（10% 激活率）。Q4_K_M 量化后，激活参数大约占用 3.5B × 0.5 bytes ≈ 1.75 GB。

但实际上，推理过程中需要加载的不仅仅是激活参数。MoE 路由机制需要读取所有 expert 的权重头部信息来决定路由。加上 KV cache（262K 上下文约 2-3 GB）、中间激活值，单次推理的实际显存占用约 8-9 GB/token。

对双卡 RTX 2080 Ti（22GB 总显存），理论计算如下：

- 全参数加载（假设没有 MoE 路由优化）：35B × 0.5 bytes ≈ 17.5 GB → 接近显存上限，速度约 40-50 t/s

- MoE 路由优化生效（只读取激活 expert）：8-9 GB/token → 显存压力小，带宽利用率更高

理论全参数吞吐上限约 47 t/s。实测 104 t/s 超出理论值一倍以上——这成为 MoE 路由优化生效的铁证：**llama.cpp 的 MoE 实现只读取被路由选中的 expert 权重，而非加载全部参数**。

### 全网 Benchmark 对比

硬件配置
模型
量级
速度
来源

2×RTX 2080 Ti (22GB)
Ornith-35B
Q4_K_M
**104 t/s**
本文

2×RTX 2080 Ti
Mixtral 8x7B
Q4_K_M
~100 t/s
Level1Techs

2×RTX 2080 Ti
Qwen3.6-35B
Q4_K_M
~42 t/s
社区实测

2×RTX 2080 Ti
DeepSeek-V2-Lite
Q4_K_M
~120 t/s
insiderllm

1×RTX 4090 (24GB)
Ornith-35B
Q4_K_M
~45 t/s
社区估算

2000 系列的双卡方案在 35B 量级 MoE 模型上，天花板大约是 100-130 t/s。Ornith 的 104 t/s 达到了约 70% 的硬件利用率，属于合理的高水平。

## Qwen3.6 vs Ornith：性能对比

维度
Qwen3.6-35B-A3B
Ornith-1.0-35B (基于 Qwen3.5)

推理速度
~42 t/s
**104 t/s** (2.5×)

社区基准
基准线
多项超越 Qwen3.6

模板兼容性
原生支持
需 chat-template-file 修复

社区活跃度
下降
上升（新模型）

部署复杂度
低
中（模板修复 + 首次配置）

## 经验总结

- 

**MoE 的路由优化不是理论概念，而是可实测的**：实测值超过全参理论值，是验证路由优化是否生效最直接的方法。如果实测速度接近全参理论值，说明路由优化可能未生效（比如某些推理引擎的 MoE 实现是全部加载再路由）。

- 

**Chat template 兼容性是跨模型部署的常见坑点**：不同推理引擎（llama.cpp vs vLLM vs SGLang）对 jinja 的解析能力不同。`raise_exception()` 在 Python Jinja2 中正常，在 minijinja 中崩溃。跨模型迁移时，务必检查模板兼容性。

- 

**模型名注册策略很重要**：用 `基座-优化版本` 的命名格式（如 `Qwen3.5-35B-A3B-Ornith`），既保留了可追溯性，又便于在代理层做 A/B 测试。

- 

**双卡 2080 Ti 的 104 t/s 不是极限**：从全网数据看，相同硬件配置下的天花板是 100-130 t/s。如果优化 tensor-split 比例、调整 batch size，还有约 25% 的提升空间。