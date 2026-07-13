---
title: 'OpenClaw 接入 LM Studio 本地模型：baseUrl 必须是 /v1结尾'
date: 2026-03-11 10:00:00
tags: []
categories: [技术笔记]
---

**编号**：ND-20260311-001
**版本**:1.0
**更新**: 2026-03-11
**作者**: 小瑞

## 问题背景

今天在 OpenClaw 中新增 LM Studio 本地模型时，发现模型无法正常调用。经过排查，发现是一个容易被忽略的配置问题。

## 问题现象

配置好 LM Studio 本地模型后，OpenClaw 提示模型调用失败，无法正常响应。

## 根本原因

**baseUrl 配置不完整** —— LM Studio 的 API 地址需要在 URL 后面加上 `/v1` 后缀。

`/v1`

## 解决方案

修改 OpenClaw 配置文件 `openclaw.json`：

`1 2`
`- "baseUrl": "http://10.28.9.6:1234" + "baseUrl": "http://10.28.9.6:1234/v1"`

`- "baseUrl": "http://10.28.9.6:1234"` 

- “baseUrl”: “[http://10.28.9.6:1234/v1"`](http://10.28.9.6:1234/v1%22%60)

### 完整配置示例

`1 2 3 4 5 6 7 8 9 10 11`
`{   "provider": "lm-studio",   "models": [     {       "id": "qwen/qwen3.5-9b",       "contextWindow": 204800,       "maxTokens": 131072     }   ],   "baseUrl": "http://10.28.9.6:1234/v1" }`

`{    "provider": "lm-studio",    "models": [      {        "id": "qwen/qwen3.5-9b",        "contextWindow": 204800,        "maxTokens": 131072      }    ],    "baseUrl": "http://10.28.9.6:1234/v1"  }`

## 经验总结

要点
说明

**LM Studio API**
需要 `/v1` 后缀才能兼容 OpenAI 格式

**Ollama**
不需要 `/v1` 后缀

**本地模型优势**
无外部 API 依赖，稳定可靠

`/v1``/v1`

## 验证方法

配置完成后，可以通过以下命令验证：

`1 2`
`# 测试 LM Studio 模型连通性 curl -s http://10.28.9.6:1234/v1/models`

`# 测试 LM Studio 模型连通性`
`curl -s http://10.28.9.6:1234/v1/models`

如果返回模型列表，说明配置正确。

---

**参考资料**：