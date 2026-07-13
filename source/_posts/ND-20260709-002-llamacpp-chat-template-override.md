---
title: '不改 GGUF 文件，用 chat-template-file 修复 llama.cpp Jinja 模板错误'
date: 2026-07-09 10:00:00
tags: []
categories: [技术笔记]
---

## 问题：HTTP 400 “Unable to generate parser for this template”

本地部署的 Qwen3.5-35B-A3B 模型（Ornith 微调版），通过 llama.cpp server 提供 API 服务。某天发现部分请求稳定返回 400 错误：

```stata

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

HTTP 400
{
  "error": {
    "code": 400,
    "message": "Unable to generate parser for this template. Automatic parser generation failed:
While executing CallExpression at line 79, column 24 in source:
... {{- raise_exception('No user query found in messages.') }}
...
Error: Jinja Exception: No user query found in messages.",
    "type": "invalid_request_error"
  }
}

```

错误信息看起来很明确——模板里有个 `raise_exception` 被触发了。但奇怪的是，这个错误**不是偶发的**，而是**100% 可复现**的，只在特定条件下触发。

## 复现：精确定位触发条件

为了搞清楚到底是什么触发了这个错误，设计了一组结构化测试矩阵，覆盖各种消息组合：

#
请求格式
结果
说明

1
`[{user: "hi"}]`
✅ 200
正常

2
`[{system: "..."}, {user: "hi"}]`
✅ 200
正常

3
`[{user: ""}]`
✅ 200
空 content 也算

4
`[{system: "..."}]`
❌ **400**
无 user 角色

5
`[{assistant: "..."}]`
❌ **400**
无 user 角色

6
`[{system: "..."}, {assistant: "..."}]`
❌ **400**
无 user 角色

**规律极其清晰**：只要 messages 数组里**没有任何一条 `role: user` 的消息**，就 100% 触发 400 错误。

每组测试跑了 5-20 次，结果完全一致，排除了偶发网络问题的可能。

## 根因分析：两层问题

深入分析后，发现问题其实有两层。

### 第一层：Qwen chat_template 中的 raise_exception

Qwen3.5 系列模型的 `chat_template.jinja` 中内置了校验逻辑，会在模板渲染阶段检查消息合法性。其中包含多个 `raise_exception()` 调用：

```jinja

1
2
3
4
5
6
7
8
9

{# 检查是否有 user 消息 #}
{% if not user_messages %}
    {{- raise_exception('No user query found in messages.') }}
{%- endif %}

{# 检查 system 消息位置 #}
{% if system_in_middle %}
    {{- raise_exception('System message must be at the beginning.') }}
{%- endif %}

```

这些校验在标准的 Python Jinja2 引擎中工作正常——引擎遇到 `raise_exception` 时会抛出异常。但在 llama.cpp 中情况不同。

### 第二层：llama.cpp 的 minijinja 不完整支持 Jinja2

llama.cpp 内置了一个轻量级 Jinja 引擎（minijinja），用于渲染 chat template。这个引擎**不支持 `raise_exception()` 这种 CallExpression 语法**。

关键区别在于：

- **Python Jinja2**：遇到 `raise_exception` 时抛出异常，被上层捕获后返回语义化的错误信息

- **llama.cpp minijinja**：在**解析模板阶段**就尝试编译这个表达式，发现不支持的语法直接报 `Unable to generate parser for this template`

也就是说，这不是”模板逻辑判断没有 user 消息所以报错”，而是”模板引擎根本无法解析包含 `raise_exception` 的模板”。

### 为什么有 user 消息时不报错？

这是一个让人困惑的点。如果 minijinja 不支持 `raise_exception`，为什么有 user 消息时是好的？

实际测试发现：minijinja **能解析** `raise_exception` 语法，但在执行到该语句时行为和标准 Jinja2 不同。当 messages 中有 user 消息时，`{% if not user_messages %}` 条件为 false，`raise_exception` 这行**永远不会被执行**，所以不报错。当没有 user 消息时，条件为 true，引擎尝试执行 `raise_exception`，触发异常。

这说明 minijinja 是**惰性求值**的——它不会在加载模板时就执行所有分支，只有运行时走到那个分支才会出问题。

## 解决方案探索

### 方案一：修改 GGUF 文件（被否决）

最直接的方法是用 `gguf-py` 等工具直接修改 GGUF 文件中内嵌的 `chat_template` 元数据，删除所有 `raise_exception` 调用。

**优点**：一劳永逸，修改的是源头。
**缺点**：

- 需要重新上传修改后的 GGUF（35GB 文件，传输耗时）

- GGUF 文件被修改后，模型校验和失效

- 如果将来从 HuggingFace 重新下载模型，修改会丢失

- 不利于维护——每次更新模型都要重新改

考虑到这些缺点，决定寻找不改 GGUF 的替代方案。

### 方案二：换推理引擎（备选）

Qwen3.5 模型同时支持 vLLM 和 SGLang，这两个引擎使用 Python Jinja2，天然支持 `raise_exception`。

**优点**：根治问题，无需任何 workaround。
**缺点**：

- 需要重新部署整个推理服务

- vLLM 对 MoE 模型的显存管理和 llama.cpp 不同，需要重新调参

- 当前 llama.cpp 运行稳定（~106 t/s），切换引擎有风险

### 方案三：llama.cpp `--chat-template-file`（最终选定）

在 llama.cpp 源码中发现了两个关键启动参数：

- `--chat-template JINJA_TEMPLATE`：直接传入自定义 Jinja 模板字符串

- `--chat-template-file JINJA_TEMPLATE_FILE`：从外部文件读取自定义模板

这两个参数会**覆盖** GGUF 文件中内嵌的 `chat_template`，实现运行时模板替换。

```bash

1
2
3

llama-server --model ornith.gguf \
  --jinja \
  --chat-template-file /path/to/fixed-template.jinja

```

**优点**：

- 不碰 GGUF 文件

- 模板文件可以随时修改，只需重启服务

- 配置在 models.ini 中管理，版本可控

- 社区已有验证（HuggingFace 上的修复讨论）

**缺点**：

- 需要重启 llama-server 服务

- 需要准备修复版模板文件

这个方案最干净、侵入性最低，最终选定。

## 实施：四步修复

### 第一步：准备修复版模板

从 HuggingFace 上 unsloth 的 Qwen3.5 仓库获取 `chat_template.jinja`。unsloth 版本已经移除了部分 `raise_exception`，但还残留了 5 处用于内容类型校验的调用。

为了彻底解决问题，用 sed 把所有 `raise_exception('...')` 替换为空操作 `{{ '' }}`：

```bash

1

sed "s/{{- raise_exception('[^']*') }}/{{ '' }}/g" chat_template.jinja > fixed-template.jinja

```

替换后验证：

```bash

1
2

$ grep -c 'raise_exception' fixed-template.jinja
0

```

零残留，模板中的所有 `raise_exception` 调用全部移除。

### 第二步：修改 models.ini

llama.cpp 的 models preset 模式使用 INI 文件管理多个模型。在 Ornith 模型的 section 中添加两行：

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

[Qwen/Qwen3.5-35B-A3B-Ornith]
alias = deepreinforce-ai/ornith-1.0-35b
model = /path/to/ornith-1.0-35b-Q4_K_M.gguf
mmproj = /path/to/mmproj-ornith-35b-f16.gguf
ctx-size = 262144
temp = 0.6
top-p = 0.95
top-k = 20
reasoning = on
jinja = true                                                          # ← 新增
chat-template-file = /path/to/fixed-template.jinja                   # ← 新增

```

**关键技术细节**：models.ini 的 key 映射规则——llama.cpp 源码中的 `get_map_key_opt()` 函数会遍历所有 CLI 参数，用环境变量名（如 `LLAMA_ARG_CHAT_TEMPLATE_FILE`）和去掉 `--` 前缀的参数名（如 `chat-template-file`）作为 INI key。

所以 INI 中写 `chat-template-file` 等价于命令行 `--chat-template-file`，写 `jinja = true` 等价于 `--jinja`。`--jinja` 必须设置，否则 `--chat-template-file` 只接受内置模板名列表，不接受自定义 Jinja 文件。

### 第三步：上传模板并重启服务

```bash

1
2
3
4
5

# 上传修复版模板到服务器
scp fixed-template.jinja server:/path/to/templates/

# 重启 llama-server（通过 systemd）
ssh server "sudo systemctl restart llama-server"

```

重启后，主进程重新读取 `models.ini`，用新的配置参数 spawn Ornith 子进程。

### 第四步：验证启动参数

通过 llama.cpp 的 `/v1/models` API 确认新进程的启动参数：

1
2

✅ --jinja: 已启用
✅ --chat-template-file: /path/to/fixed-template.jinja

```

子进程的完整参数列表中包含了 `--chat-template-file` 和 `--jinja`，确认配置已生效。

## 验证：完整测试矩阵

修复后，用同样的测试矩阵重新跑一遍：

#
请求格式
修复前
修复后

1
`[{user: "hi"}]`
✅ 200
✅ 200

2
`[{system}, {user: "hi"}]`
✅ 200
✅ 200

3
`[{user: ""}]`
✅ 200
✅ 200

4
`[{system: "..."}]`
❌ 400
✅ **200**

5
`[{assistant: "..."}]`
❌ 400
✅ **200**

6
`[{system}, {assistant}]`
❌ 400
✅ **200**

**全部通过。** 之前 100% 失败的三个场景，现在全部返回 200。正常的请求不受影响。

同时通过 API 代理层（new-api）验证了端到端链路：代理 → llama.cpp → 模板渲染 → 正常响应，全链路通畅。

## 坑点总结

这次排查过程中踩了几个坑，记录下来：

### 坑 1：误判错误来源

最初以为错误来自上游 API 服务商（SiliconFlow），因为错误信息里有 `raise_exception`。但实际测试发现，免费模型和本地模型的错误信息**完全不同**：

- **免费模型（上游）**：模板能正常解析，`raise_exception` 是正常的运行时校验

- **本地模型（llama.cpp）**：minijinja 引擎不支持 `raise_exception`，解析/执行阶段崩溃

两个问题根因不同，解决方法也不同。通过对比 `Server` HTTP 头可以快速区分：上游返回的是服务商标识，本地返回的是 `Server: llama.cpp`。

### 坑 2：unsloth 修复版模板不够彻底

unsloth 的修复版移除了最常触发的两处 `raise_exception`（user 检查和 system 位置检查），但保留了 5 处用于内容类型校验的调用（图片/视频类型检查、空消息检查）。

修复后测试发现，只含 assistant 消息的请求仍然报错——因为触发了 `'No messages provided.'` 这处残留的 `raise_exception`。最终用 sed 全量替换，0 残留才彻底解决。

**教训**：使用社区的修复方案时，不能只看 star 数和讨论热度，必须实际测试覆盖所有边界场景。

### 坑 3：models.ini 配置不生效（需要重启主进程）

修改 `models.ini` 后，尝试通过 `/models/unload` API 卸载模型，期望自动重新加载。但卸载后重新 spawn 的子进程**仍然使用旧参数**——因为主进程在启动时读取 `models.ini` 并缓存在内存中，单纯卸载模型不会触发重新读取。

**解决方法**：通过 `systemctl restart llama-server` 重启整个主进程，让它重新解析 `models.ini`。

### 坑 4：models.ini 的 key 命名规则

models.ini 不是随便起 key 名的。通过阅读 llama.cpp 源码，发现 INI key 必须匹配以下两种格式之一：

- 环境变量名：`LLAMA_ARG_CHAT_TEMPLATE_FILE`

- 去掉 `--` 的参数名：`chat-template-file`

写 `chat_template_file`（下划线）或 `chatTemplateFile`（驼峰）都不行，会报 `option not recognized` 错误。

## 技术要点速查

### llama.cpp chat template override 方法总结

方法
命令
适用场景

`--chat-template`
`--chat-template chatml`
使用内置模板（ChatML、Llama3 等 50+ 种）

`--chat-template`
`--chat-template "{{...}}"`
直接传入 Jinja 字符串（需加 `--jinja`）

`--chat-template-file`
`--chat-template-file ./template.jinja`
从文件加载（需加 `--jinja`）

`--no-jinja`
`--no-jinja --chat-template chatml`
完全绕过 Jinja 引擎（可能丢失 tool-call）

### 查找 llama.cpp 支持的内置模板

```bash

1

llama-server --help | grep -A 100 "chat-template"

```

常见内置模板：`chatml`、`llama3`、`deepseek3`、`gemma`、`mistral-v3`、`phi3`、`zephyr` 等。

### models.ini 中添加自定义模板

```ini

1
2
3
4

[YourModel]
model = /path/to/model.gguf
jinja = true
chat-template-file = /path/to/fixed-template.jinja

```

记住三点：

- `jinja = true` 必须设置（否则只接受内置模板名）

- key 用连字符格式（`chat-template-file`），不是下划线

- 修改后需重启 llama-server 主进程

## 结论

llama.cpp 的 minijinja 引擎对 Jinja2 的支持不是完整的，`raise_exception` 只是其中一个不兼容的点。当模型 chat_template 使用了 minijinja 不支持的特性时，通过 `--chat-template-file` 在运行时覆盖模板，是不修改 GGUF 文件的最佳解决方案。

这个方案的核心优势是**关注点分离**：模型权重（GGUF）和对话模板（jinja 文件）独立管理，模板可以随时更新而不用动几十 GB 的模型文件。对于需要频繁迭代 prompt 格式的场景（比如调整 tool-calling 格式、修改 system prompt 处理逻辑），这种分离特别有价值。