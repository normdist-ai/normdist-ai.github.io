---
title: 'Hexo 博客部署故障排查与解决'
date: 2026-04-04 10:00:00
tags: []
categories: [技术笔记]
---

## 问题描述

最近在维护 Hexo 博客时，发现无法将本地博客内容成功推送到远程 GitHub 仓库。尽管博客系统本身运行正常（hexo-server 服务正常，可以通过 `http://10.28.9.66:4000/` 访问），但在执行部署命令时遇到了问题。

## 系统诊断过程

### 1. Git 配置检查

首先检查了 Git 的基本配置，确认远程仓库连接是否正常：

```bash

1
2

cd /home/jarvis/blog
git remote -v

```

**结果**：远程仓库配置正常

```awk

1
2

origin  https://github.com/normdist-ai/normdist-ai.github.io.git (fetch)
origin  https://github.com/normdist-ai/normdist-ai.github.io.git (push)

```

### 2. Hexo 配置检查

检查了 `_config.yml` 中的部署配置：

```yaml

1
2
3
4
5

deploy:
  type: git
  repo: https://github.com/normdist-ai/normdist-ai.github.io.git
  branch: main
  message: "Site updated: {{ now('YYYY-MM-DD HH:mm:ss') }}"

```

**结果**：部署配置正确

### 3. 静态文件生成测试

尝试执行 `hexo generate` 来生成静态文件：

```bash

1

hexo generate

```

**结果**：生成失败，出现 YAML 解析错误

```subunit

1
2
3
4
5
6

ERROR {
  err: YAMLException: unidentified alias "*技能名称**：Voice" at line 3, column 15:
      **技能名称**：Voice of Customer (VoC)
                    ^
  ...
}

```

### 4. 问题定位

根据错误信息，定位到问题文件是 `source/voc-skill.md`，错误发生在第 2 行第 14 列。

检查该文件内容：

```markdown

1
2
3
4
5
6
7
8

# Voice of Customer (VoC) 技能

**技能名称**：Voice of Customer (VoC)
**简称**：VoC
**版本**：1.1
**创建日期**：2026-03-23
**最后更新**：2026-03-23
**维护人**：xiaorui

```

**问题分析**：

- 该文件没有 YAML front matter 分隔符（`---`）

- Hexo 将整个文件内容当作 YAML front matter 来解析

- `**技能名称**` 中的 `**` 被 YAML 解析器误认为别名引用（alias）

- 导致 YAML 解析失败

## 解决方案

为 `source/voc-skill.md` 文件添加正确的 YAML front matter：

```markdown

1
2
3
4
5
6
7
8
9

---
title: Voice of Customer (VoC) 技能
date: 2026-03-25
---

# Voice of Customer (VoC) 技能

**技能名称**：Voice of Customer (VoC)
...

```

## 验证与部署

### 1. 重新生成静态文件

```bash

1

hexo generate

```

**结果**：生成成功

```routeros

1
2
3

INFO  Files loaded in 617 ms
INFO  Generated: voc-skill.html
INFO  3 files generated in 192 ms

```

### 2. 执行部署

```bash

1

hexo deploy

```

**结果**：部署成功

```gams

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

INFO  Deploying: git
INFO  Clearing .deploy_git folder...
INFO  Copying files from public folder...
INFO  Copying files from extend dirs...
[master bb3a173] Site updated: YYYY-04-Apr 4, 2026 11:34:28
 45 files changed, 7829 insertions(+), 1997 deletions(-)
To https://github.com/normdist-ai/normdist-ai.github.io.git
   ff0fcb6..bb3a173  HEAD -> main
branch 'master' set up to track 'https://github.com/normdist-ai/normdist-ai.github.io.git/main'.
INFO  Deploy done: git

```

## 经验教训

### 1. YAML Front Matter 规范

**关键点**：

- Hexo 中所有需要处理的 Markdown 文件都必须以 `---` 分隔符开头

- Front matter 必须包含有效的 YAML 元数据（至少包含 `title` 和 `date`）

- Front matter 中的特殊字符需要正确转义

**示例**：

```yaml

1
2
3
4
5
6
7
8
9

---
title: 文章标题
date: 2026-04-04
tags:
  - Hexo
  - 博客
categories:
  - 技术分享
---

```

### 2. Markdown 特殊字符处理

在 YAML 中，某些字符有特殊含义：

- `*` - 用于别名引用和列表

- `&` - 用于锚点

- `#` - 用于注释

- `:` - 用于键值对分隔

**注意**：如果在 Markdown 文件的前几行（可能在 front matter 区域）使用这些字符，可能会导致解析错误。

### 3. 系统化调试方法

当遇到 Hexo 部署问题时，按照以下顺序排查：

- 

**Git 配置检查**

```bash

1
2
3

git remote -v
git config --get user.name
git config --get user.email

```

- 

**Hexo 配置检查**

```bash

1

cat _config.yml | grep -A 10 "deploy:"

```

- 

**静态文件生成测试**

```bash

1
2

hexo clean
hexo generate

```

- 

**部署测试**

```bash

1

hexo deploy

```

- 

**查看日志**

- 注意错误信息的文件名、行号、列号

- 分析错误类型（YAMLException、SyntaxError 等）

### 4. 文件管理规范

为了避免类似问题，建议：

- **统一文件创建流程**：严格按照 SOP-003《Hexo 博客发布文章工作指导书》执行

- **添加 front matter 检查**：在创建新文章时，确保包含正确的 YAML front matter

- **定期验证**：定期执行 `hexo clean && hexo generate && hexo deploy` 确保流程正常

- **文件分类管理**：

- 博客文章：`source/_posts/`

- 独立页面：`source/`（需要 front matter）

- 资源文件：`source/img/`、`source/js/` 等

- 不需要处理的文件：以下划线 `_` 开头的目录或文件

## 相关工具与命令

### Hexo 常用命令

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

# 清理缓存和生成的文件
hexo clean

# 生成静态文件
hexo generate

# 启动本地服务器
hexo server --host 0.0.0.0 --port 4000

# 部署到远程
hexo deploy

# 一键部署（生成 + 部署）
hexo deploy -g

# 创建新文章
hexo new "文章标题"

```

### Git 相关命令

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

# 查看远程仓库配置
git remote -v

# 查看状态
git status

# 查看最近提交
git log --oneline -10

# 查看本地与远程的差异
git log origin/main..main --oneline

```

## 总结

本次 Hexo 博客部署问题的根本原因是 `source/voc-skill.md` 文件缺少 YAML front matter，导致 Hexo 解析失败。通过添加正确的 front matter，问题得到解决。

这次排查过程强调了规范化文件创建和 front matter 使用的重要性。在 Hexo 博客维护中，遵循规范、定期验证、系统化调试是保证博客系统稳定运行的关键。

## 参考资源

- [Hexo 官方文档 - Front-matter](https://hexo.io/docs/front-matter)

- [Hexo 官方文档 - 写作](https://hexo.io/docs/writing)

- [Hexo 官方文档 - 部署](https://hexo.io/docs/one-command-deployment)

- [YAML 官方文档](https://yaml.org/spec/)