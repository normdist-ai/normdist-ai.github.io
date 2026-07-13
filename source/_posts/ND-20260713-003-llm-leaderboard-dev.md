---
title: '开源大模型天梯图开发全记录'
date: 2026-07-14 06:56:42
tags: [前端开发, 数据可视化, HTML, Hexo, LLM]
categories: [技术笔记]
---

昨天深夜，我把 169 行 HTML 文件喂给 write_file 工具，以为会看到干净的代码。结果打开一看——每行前面都多了个 `142|` 的前缀，像给每行贴了个条形码。一行还行，169 行全被污染。

这就是做开源大模型天梯图最真实的日常：你以为在写代码，其实是在和数据打架。

起因很简单：想找一个能直观对比开源大模型编程能力的排行榜。不是那种只列了十几个主流模型的简表，而是把 2023 年到现在所有能打的都收进来。翻了一圈，要么只更新到 2024 年初，要么只有总分没有 HumanEval 和 SWE-bench 的细分数据，要么表格在大屏上还缩在 1200px 宽度里，两边留白够放两个广告位。

于是就有了这个天梯图：[https://normdist.com/benchmark/llm.html](https://normdist.com/benchmark/llm.html)。61 个开源/开放模型，按 LiveCodeBench 编程能力排名，附带 HumanEval 和 SWE-bench Verified 参考分，支持按厂商、参数规模、架构、年份多层筛选，移动端做了三档响应式适配。这篇文章记录从零到部署踩过的坑，以及那些踩完才想明白的技术决策。

## 数据采集：两个 API 源头拼出完整画像

起初只想做个排行榜，但现有方案数据量太少。我把目标锁定在 2023-2026 年全部开源模型的编程能力对比，数据采集走了两条路：

- **HuggingFace API**：批量查询 `createdAt` 精确日期和 model card 里的 benchmark 评分。这步对按年份筛选至关重要——同是 Qwen 系列，Qwen2.5-Coder 和 Qwen3-Coder 的发布时间差了半年，编程能力分差出 15 分。
- **LLM Stats**：补充 HumanEval 和 SWE-bench Verified 参考分数。

从 54 个模型迭代到 61 个，每加一个都要验证数据来源是否可靠。HuggingFace 的 `/models/{model_id}` 端点在短时间高频请求下会返回 429，加 1 秒延迟后稳定跑通。61 个模型分两批拉完，每批间隔 30 秒，总共不到 10 分钟。LLM Stats 这边有些模型没有收录，比如 Ornith 全系列和 GLM 全系列，这些需要手动从模型卡 README 里提取 benchmark 结果。最终 61 个模型覆盖了 DeepSeek、Alibaba、Meta、Google、Mistral、Anthropic 等 15 个厂商，时间跨度从 2023 年 2 月 LLaMA 发布到 2026 年 7 月最新的 Qwen3.5-Ornith。

## 独立 HTML 的三个“叛逆”设计

项目方有句话让我记到现在：“别依赖 CDN。”于是这个页面成了 Hexo 博客里的异类——所有 CSS、JS 全部内联，不引用任何外部资源。

### 决策一：skip_render 比插件更干净

Hexo 的渲染管线会把所有 `.md` 和 `.html` 文件套上主题模板，生成导航栏、侧边栏、页脚。但天梯图页面需要完整控制 DOM，表格要撑满 1600px 宽度，不能被主题的 `main-content` 容器限制。方案有两个：写一个 Hexo 插件在 `after_render` 阶段替换部分 HTML，或者直接让源文件跳过渲染。选了后者——在 `_config.yml` 里加一行：

```yaml
skip_render:
  - benchmark/llm.html
```

文件放在 `source/benchmark/llm.html`，`hexo generate` 后原样复制到 `public/benchmark/llm.html`。主题的导航栏通过 `_config.fluid.yml` 的 `menu` 配置手动添加链接，指向 `/benchmark/llm.html`，`target: _blank` 新窗口打开。

### 决策二：零 CDN 依赖，系统字体栈

CSS 和 JS 全写在 `<style>` 和 `<script>` 标签里，没有引外部字体、图标库或 jQuery。字体栈用的是系统默认：

```css
font-family: -apple-system, BlinkMacSystemFont, "PingFang SC", "Microsoft YaHei", Helvetica, Arial, sans-serif;
```

macOS 和 Windows 下分别调用系统 UI 字体，渲染零延迟，不触发网络请求。表格里的 Emoji 用系统自带，不用 Font Awesome。

### 决策三：SVG Logo 内联，无 Logo 用首字母圆形

厂商 Logo 这部分反复改了三次。第一版用 `<img src="...">` 外链，CDN 缓存失效时 Logo 消失只剩空白。第二版改成 Base64 内联，但 8 个 SVG 的 Base64 字符串加起来 15KB，HTML 文件膨胀到 120KB。最终方案：8 个有正式 Logo 的厂商（DeepSeek、Meta、Google、Mistral、Anthropic、Qwen、Zhipu、Moonshot）用 `<svg>` 直接内联，压缩后总共不到 3KB。没有 Logo 的厂商渲染一个首字母圆形 badge，CSS 就能搞定：

```css
.logo-badge {
  width: 24px; height: 24px;
  border-radius: 50%;
  background: #e0e0e0;
  text-align: center;
  line-height: 24px;
  font-size: 12px;
  font-weight: 700;
  color: #555;
}
```

## 多层筛选：同组 OR + 跨组 AND

筛选逻辑参考了 llm-stats.com 的设计，但做了一层扩展：同组内的多个选项是并集，跨组之间是交集。

举个例子：厂商组选“DeepSeek”和“Alibaba”（OR），架构组再选“MoE”（AND），结果是 14 个模型——DeepSeek 和 Alibaba 旗下所有 MoE 模型的并集。如果再加参数组选“≤7B”，就缩小到 3 个模型。实现上，每次 checkbox 状态变化时遍历全部 61 条数据，逐行检查是否满足所有筛选组的条件。数据量小，61 行够用，不用上虚拟滚动或索引加速。

列选择器也折腾了一阵。按钮标签中文化了，叫「⚙ 表头」，用 checkbox 列表控制列的显隐。第一版把面板放在表格容器内部，`position: absolute` 定位。但表格容器加了 `overflow-x: auto` 支持横向滚动，结果面板一打开就被裁剪，只露出上半截。根因是 `overflow-x: auto` 会裁剪超出容器边界的绝对定位子元素。解法是把面板移到表格容器外部，改用 `position: fixed`，然后通过 JS 在按钮点击时计算按钮的 `getBoundingClientRect()`，动态设置面板的 `top` 和 `left`：

```javascript
const rect = btn.getBoundingClientRect();
panel.style.top = (rect.bottom + 4) + 'px';
panel.style.left = (rect.right - 200) + 'px'; // 面板宽 200px，右对齐
```

排序箭头则设计为三态循环：`asc(▲紫色) → desc(▼紫色) → none(灰色)`，每列独立排序，一眼就能看出当前排序状态。

## 移动端适配：三档断点与 sticky 修正

手机上看表格是个噩梦，我设了三档断点：

| 宽度 | 显示列 |
|------|--------|
| >900px | 全列（排名、模型、厂商、参数、架构、LiveCodeBench、HumanEval、SWE-bench、VRAM） |
| ≤900px | 隐藏 HumanEval、SWE-bench、VRAM 列，保留 5 列核心数据 |
| ≤480px | 再隐藏参数、架构列，只剩排名、模型名、LiveCodeBench 分数 |

表头用 `position: sticky; top: 0` 固定在表格容器顶部。最开始设了 `top: 64px` 给 Hexo 导航栏留空间，但独立 HTML 页面不走主题导航，`top: 64px` 会导致表头在滚动时悬浮在奇怪的位置。改成 `top: 0` 后正常。日期列加了 `white-space: nowrap` 防止在小屏上换行成两行，挤占模型名空间。

## 部署与缓存绕过

发布流程简单到只有四步：

1. `hexo clean && hexo generate` 生成静态文件
2. 复制到 `.deploy_git/` 目录
3. `git push` 到 `normdist-ai/normdist-ai.github.io` 的 main 分支
4. GitHub Pages 自动部署，CDN 缓存约 10 分钟

更新频繁时，10 分钟的缓存窗口会导致部分用户看到旧版本。最简单的绕过：在 Hexo 菜单链接后加 `?timestamp` 参数。每次更新后手动换时间戳，读者点导航栏的「天梯」链接时绕过 CDN 缓存。

## 那 169 行的教训

回到开头的事故。**现象**：write_file 写回的文件每行都有 `数字|` 前缀。**根因**：read_file 工具返回的内容带了行号前缀，我直接写回去了。169 行 HTML 全部被污染，表格数据前的 `<tr>` 变成 `1|<tr>`，JavaScript 代码变成 `142|  function filterModels() {`。浏览器打开页面直接报错，整个天梯图不可用。

修复分两步：先用正则全量清洗，`re.sub(r'^\d+\|\s*', '', line, flags=re.MULTILINE)` 去掉所有行号前缀，再手动检查 169 行确保没有误删以数字开头的合法内容。所幸没有——HTML 标签和 JS 代码通常不以数字开头。

这个坑的代价是 169 行代码的重复劳动，但更重要的是——工具返回的格式永远要验证，别假设它和你想的一样。

## 下一步

61 个模型只是开始。计划加入更多编程基准（MBPP、DS-1000），以及按参数规模分组的高亮显示。做数据页面的核心心得：**数据清洗比写代码累十倍，但值得。**

---

**参考文献**

1. [LiveCodeBench 官方](https://livecodebench.github.io/)
2. [LLM Stats - HumanEval & SWE-bench 数据](https://llm-stats.com/)
3. [HuggingFace Models API 文档](https://huggingface.co/docs/hub/api)
4. [Hexo skip_render 配置文档](https://hexo.io/docs/configuration#Directory)
5. [MDN: overflow 与绝对定位裁剪](https://developer.mozilla.org/en-US/docs/Web/CSS/overflow)