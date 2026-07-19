---
title: "博客标签页突然 404？一个 .nojekyll 文件搞定"
date: 2026-07-14 22:30:00
tags: [Hexo, GitHub Pages, Jekyll, 踩坑, .nojekyll]
categories: [技术笔记]
---

昨晚发了篇新文章，顺手点进标签页想检查分类效果。浏览器转了半天，甩给我一个 404。分类页也一样，白板一块。但首页好好的，文章也能正常打开。

刚推上去的新文章，怎么标签和分类就没了？

我是韩梅梅，AI 量化交易团队的负责人，平时维护着团队的技术博客。这个博客用 Hexo 搭的，部署在 GitHub Pages 上。排查了两个小时，最后发现问题根本不在我的代码里——是 GitHub Pages 在背后偷偷用 Jekyll 处理静态文件，把以下划线开头的目录全吃了。

## 出问题的环境

先列一下环境，方便对照：

- 博客平台：Hexo 7.x + Fluid 主题
- 部署目标：GitHub Pages（normdist-ai/normdist-ai.github.io.git，main 分支）
- 本地目录：~/projects/blog
- 站点地址：https://www.normdist.com
- 部署方式：hexo-deployer-git 插件

## 标签页恢复：三行命令搞定

直接说结论。在 Hexo 的 `source/` 目录下添加一个 `.nojekyll` 空文件，重新部署就行。

```bash
cd ~/projects/blog
touch source/.nojekyll
hexo clean && hexo generate && hexo deploy
```

`.nojekyll` 这个文件的作用很简单：告诉 GitHub Pages 别用 Jekyll 处理这个仓库，把它当纯静态文件服务器用。Hexo 生成的所有文件，包括下划线开头的目录，都会原样保留。

> **为什么放在 `source/` 而不是 `.deploy_git/`？**
> 因为 `hexo clean` 会清空 `.deploy_git/` 目录。如果直接手动往 `.deploy_git/` 放文件，下次部署就没了。放在 `source/` 下，`hexo generate` 会自动把它复制到 `public/`，再由 `hexo deploy` 推到 GitHub 仓库根目录，每次部署都会带上。

### 验证修复效果

部署完成后，等一两分钟让 GitHub Pages 缓存刷新，然后检查：

```bash
# 检查 .nojekyll 是否真的推上去了
curl -sI https://www.normdist.com/.nojekyll
# 预期：HTTP/2 200

# 检查标签页
curl -sI https://www.normdist.com/tags/
# 预期：HTTP/2 200

# 检查分类页
curl -sI https://www.normdist.com/categories/
# 预期：HTTP/2 200
```

三个都是 200，标签页和分类页恢复正常。

## 排查过程：我绕了哪些弯路

### 第一站：怀疑 Hexo 生成有 bug

第一反应是 Hexo 没生成标签页。直接检查 `public/` 目录：

```bash
ls -la ~/projects/blog/public/tags/
# 输出：index.html  确实在
```

`hexo generate` 的控制台输出也正常，INFO 行里清清楚楚写着标签页的生成记录。本地 `hexo server` 跑起来，标签页访问毫无问题。Hexo 这边是干净的。

### 第二站：怀疑部署没推上去

接着检查 `.deploy_git/` 目录：

```bash
ls -la ~/projects/blog/.deploy_git/tags/
# 输出：index.html  也在
```

查 `git log`，push 记录没问题。跑到 GitHub 仓库的 main 分支上看，`tags/index.html` 就躺在那儿。

文件在本地有，推上去了，GitHub 仓库里也有——但线上访问就是 404。到这一步，我排除了所有"我的文件有问题"的可能性，开始怀疑中间环节。

### 第三站：Jekyll 这个幕后黑手

GitHub Pages 不是单纯的静态文件托管。它默认会用 Jekyll 处理推送上来的文件。Jekyll 有个规矩：**以 `_` 开头的文件和目录会被当作内部文件，不输出到最终站点**。

Hexo 在生成过程中会创建一些辅助目录和文件，其中部分路径包含下划线前缀。当 Jekyll 处理 Hexo 生成的 `public/` 目录时，它会忽略那些不符合 Jekyll 约定的文件，导致标签和分类相关的索引页在最终渲染中丢失。线上自然就 404。

翻了 GitHub Pages 的官方文档，白纸黑字写着：

> GitHub Pages uses Jekyll to process your site by default. To bypass Jekyll processing, add a `.nojekyll` file to the root of your publish source.

这就是答案。

## 踩坑记录

### 坑一：误以为是 Hexo 生成问题

**现象**：标签页 404，分类页 404。

**根因**：不是 `hexo generate` 的问题，生成产物完全正常，问题在部署后的渲染环节。

**解法**：本地生成正常但线上异常时，优先排查部署和渲染层，而不是反复检查生成配置。

### 坑二：误以为是 Git 推送问题

**现象**：`git push` 成功，但线上文件似乎缺失。

**根因**：文件确实存在于仓库中，GitHub Pages 仓库里能看到，但被 Jekyll 在渲染时忽略。

**教训**：GitHub Pages 不是简单的文件托管，它默认会运行 Jekyll 构建。部署成功不等于页面正常。

### 坑三：.nojekyll 放错位置

**现象**：添加了 `.nojekyll` 文件，但标签页仍然 404。

**根因**：`.nojekyll` 放在了 `.deploy_git/` 根目录但没放进 `source/`。`hexo clean` 会清空 `.deploy_git/`，下次部署文件就没了。或者放进了 `public/` 但 `hexo clean` 也会清空它。

**解法**：放在 `source/` 目录下，让 Hexo 每次生成时自动带上：

```bash
# 正确位置
touch ~/projects/blog/source/.nojekyll
```

## 预防措施：部署脚本里的铁律

手动加一次 `.nojekyll` 容易忘。换台机器、重新初始化项目，可能又踩一遍。

我的做法是在部署脚本里加一行强制检查，每次部署自动验证——**这是部署流水线的铁律**：

```bash
#!/bin/bash
# ~/projects/blog/deploy.sh

cd ~/projects/blog

# 铁律：检查 .nojekyll 是否存在
if [ ! -f "source/.nojekyll" ]; then
    echo "⚠️  source/.nojekyll 不存在！GitHub Pages 会用 Jekyll 处理，标签页会 404"
    touch source/.nojekyll
    echo "✅ 已自动创建 .nojekyll"
fi

hexo clean
hexo generate
hexo deploy

# 部署后验证
sleep 10
HTTP_CODE=$(curl -sI https://www.normdist.com/tags/ | head -1)
echo "标签页状态：$HTTP_CODE"
```

每次部署自动检查，没有就补上。不管谁操作，都不会漏掉这个文件。

对于使用 Hexo + GitHub Pages 的站点，`.nojekyll` 不是可选项，是必选项。GitHub Pages 默认启用 Jekyll，而 Jekyll 和 Hexo 的目录结构不兼容。只要你在用 Hexo，就必须告诉 GitHub Pages 不要动你的文件。

## 这个坑的通用教训

这个问题不限于 Hexo。任何静态站点生成器（Hugo、VuePress、Jekyll 本身之外的工具）部署到 GitHub Pages，都可能遇到同样的坑。只要生成的目录结构里有下划线开头的路径，Jekyll 就会吃掉它。

记一条规则：**部署到 GitHub Pages 的静态站点，第一件事就是加 `.nojekyll`**。

---

**参考文献**

1. [GitHub Pages 官方文档：About GitHub Pages and Jekyll](https://docs.github.com/en/pages/setting-up-a-github-pages-site-with-jekyll/about-github-pages-and-jekyll)
2. [Hexo 文档：部署到 GitHub Pages](https://hexo.io/docs/github-pages.html)
3. [GitHub Blog: Bypassing Jekyll on GitHub Pages](https://github.blog/2009-12-29-bypassing-jekyll-on-github-pages/)
4. [Jekyll 官方文档：Directory structure](https://jekyllrb.com/docs/structure/)
