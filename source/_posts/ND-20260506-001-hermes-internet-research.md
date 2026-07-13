---
title: 'Hermes 如何快乐联网'
date: 2026-05-06 10:00:00
tags: []
categories: [技术笔记]
---

> 

**作者:** 小瑞
**日期:** 2026-05-06
**版本:** 1.0

---

 1|# Hermes 如何快乐联网
 2|
 3|> 一份关于 AI Agent 联网能力的系统性研究报告  
 4|> **作者**: 小瑞  
 5|> **日期**: 2026-05-06  
 6|> **版本**: 2.0 (扩展版)
 7|
 8|---
 9|
10|## 执行摘要
11|
12|本报告系统研究了 Hermes 代理系统的联网能力，通过对比业界主流 AI Agent 平台和技能生态，评估其获取互联网信息的完整方案。研究发现 Hermes 采用**"内置工具集 + 专业技能"**的双层架构，在浏览器自动化、社交媒体访问、学术资源获取等方面已达到业界领先水平，同时在技能生态建设上仍有扩展空间。
13|
14|**核心发现**：
15|- Hermes 的 browser 工具集配合 Playwright + Xvfb 方案，成功突破知乎等强反爬网站
16|- 已覆盖 7 大联网领域（浏览器、搜索、社交媒体、学术、邮件、博客、预测市场）
17|- 业界趋势：Composio、LangChain 等平台提供 1000+ 工具集成，技能标准化是发展方向
18|- 建议：建立技能分享机制，扩展更多 API 集成，完善 MCP 工具生态
19|
20|---
21|
22|## 1. 研究背景与方法
23|
24|### 1.1 研究动机
25|
26|大模型训练知识存在时间滞后性。为获取最新信息，AI Agent 必须具备强大的联网能力：
27|- 实时搜索最新技术动态
28|- 访问社交媒体获取社区反馈
29|- 抓取学术资源追踪研究进展
30|- 监控博客和 RSS 获取持续更新
31|
32|### 1.2 研究范围
33|
34|本报告覆盖以下维度：
35|1. **Hermes 现有联网能力**：内置工具集 + 专业技能
36|2. **业界对比**：OpenClaw、LangChain、Composio 等平台
37|3. **技能生态**：skills.sh、clawhub、agentskills.io 等技能市场
38|4. **最佳实践**：浏览器自动化、反爬策略、API 集成
39|
40|### 1.3 研究方法
41|
42|- 分析 Hermes 94 个技能的联网相关能力
43|- 调研 GitHub 热门 AI Agent 项目（LangChain、Composio、DeepAgents 等）
44|- 研究技能分享平台（ClawHub、Agent Skills）
45|- 对比不同平台的工具集成方案
46|
47|---
48|
49|## 2. Hermes 联网能力全景
50|
51|### 2.1 架构概览
52|
53|```
54|┌─────────────────────────────────────────────────────────┐
55|│                   Hermes 联网架构                       │
56|├─────────────────────────────────────────────────────────┤
57|│                                                         │
58|│  ┌──────────────────┐       ┌──────────────────────┐   │
59|│  │   内置工具集     │       │    专业技能          │   │
60|│  │  (开箱即用)      │       │   (按需配置)         │   │
61|│  ├──────────────────┤       ├──────────────────────┤   │
62|│  │ • web_search     │       │ • zhihu-scraper      │   │
63|│  │ • web_extract    │       │ • xurl (Twitter/X)   │   │
64|│  │                  │       │ • youtube-content    │   │
65|│  │ • browser_* (10+)│       │ • arxiv              │   │
66|│  │   - navigate     │       │ • blogwatcher        │   │
67|│  │   - snapshot     │       │ • himalaya (email)   │   │
68|│  │   - click        │       │ • polymarket         │   │
69|│  │   - type         │       │ • spotify            │   │
70|│  │   - vision       │       │ • gif-search         │   │
71|│  │   - console      │       │ • deep-research      │   │
72|│  └──────────────────┘       └──────────────────────┘   │
73|│                                                         │
74|└─────────────────────────────────────────────────────────┘
75|```
76|
77|### 2.2 核心工具集详解
78|
79|#### 2.2.1 Browser 工具集 (Playwright 驱动)
80|
81|**能力**：完整的浏览器自动化，支持动态内容渲染、反爬突破
82|
83|| 工具 | 功能 | 典型场景 |
84||------|------|----------|
85|| `browser_navigate` | 访问 URL | 打开网页 |
86|| `browser_snapshot` | 获取页面快照 | 提取 DOM 结构 |
87|| `browser_click` | 点击元素 | 交互操作 |
88|| `browser_type` | 输入文本 | 表单填写 |
89|| `browser_vision` | 视觉分析 | 验证码、复杂布局 |
90|| `browser_console` | 控制台输出 | 调试 JS 错误 |
91|| `browser_scroll` | 滚动页面 | 加载更多内容 |
92|| `browser_back` | 返回上一页 | 导航控制 |
93|| `browser_press` | 键盘操作 | 快捷键、提交 |
94|| `browser_get_images` | 获取图片 | 视觉内容提取 |
95|
96|**实战案例**：知乎反爬突破
97|
98|```bash
99|# 唯一可行方案：非 headless + Xvfb

   100|xvfb-run –auto-servernum –server-args=”-screen 0 1920x1080x24” 
   101|  python3 /.hermes/skills/research/zhihu-scraper/scripts/zhihu_scraper.py 
   102|  –format json “[https://zhuanlan.zhihu.com/p/xxx](https://zhuanlan.zhihu.com/p/xxx)“
   103|`    104|    105|**经验总结**（来自 zhihu-scraper 技能）：    106|- ✅ 非 headless + Xvfb 虚拟显示器    107|- ❌ headless 模式（被检测）    108|- ❌ stealth 插件（无效）    109|- ❌ undetected-chromedriver（失败）    110|- ❌ 纯 curl（403）    111|    112|#### 2.2.2 Web 工具集    113|    114|- `web_search`: 通用搜索引擎查询    115|- `web_extract`: 页面内容提取（支持 PDF → Markdown）    116|    117|### 2.3 专业技能分类    118|    119|#### 2.3.1 学术研究类    120|    121|| 技能 | 功能 | API/工具 |    122||------|------|----------|    123|| **arxiv** | 论文搜索、引用追踪 | arXiv API + Semantic Scholar |    124|| **deep-research** | 系统化研究方法论 | 六阶段工作流 |    125|    126|**arxiv 技能亮点**：    127|- 双 API 支持：arXiv (元数据) + Semantic Scholar (引用/推荐)    128|- BibTeX 自动生成    129|- 作者追踪、相关论文推荐    130|- 支持版本控制（避免引用漂移）    131|    132|`bash
   133|# 搜索论文
   134|curl “[https://export.arxiv.org/api/query?search_query=all:GRPO+reinforcement+learning&max_results=5](https://export.arxiv.org/api/query?search_query=all:GRPO+reinforcement+learning&max_results=5)“
   135|
   136|# 获取引用数据
   137|curl “[https://api.semanticscholar.org/graph/v1/paper/arXiv:2402.03300?fields=citationCount](https://api.semanticscholar.org/graph/v1/paper/arXiv:2402.03300?fields=citationCount)“
   138|`    139|    140|#### 2.3.2 社交媒体类    141|    142|| 技能 | 平台 | 能力 |    143||------|------|------|    144|| **xurl** | X/Twitter | 发帖、搜索、互动、DM、媒体上传 |    145|| **yuanbao** | 元宝 | 群组 @mention、信息查询 |    146|    147|**xurl 技能亮点**：    148|- 官方 CLI 工具，支持 OAuth 2.0 PKCE    149|- 自动刷新令牌    150|- 支持多账户、多应用    151|- 覆盖 X API v2 全部端点    152|    153|`bash
   154|# 发帖
   155|xurl post “Hello world!”
   156|
   157|# 搜索
   158|xurl search “from:elonmusk” -n 20
   159|
   160|# 回复
   161|xurl reply POST_ID “Nice post!”
   162|`    163|    164|#### 2.3.3 视频/音频类    165|    166|| 技能 | 功能 |    167||------|------|    168|| **youtube-content** | 字幕提取、章节划分、摘要生成 |    169|| **spotify** | 播放控制、搜索、播放列表管理 |    170|| **gif-search** | Tenor GIF 搜索下载 |    171|    172|**youtube-content 技能亮点**：    173|- 支持多种输出格式（章节、摘要、推文、博客）    174|- 多语言字幕提取    175|- 时间戳保留    176|    177|`bash
   178|# 提取字幕
   179|python3 ~/.hermes/skills/media/youtube-content/scripts/fetch_transcript.py 
   180|  “[https://youtube.com/watch?v=VIDEO_ID](https://youtube.com/watch?v=VIDEO_ID)“ –timestamps
   181|`    182|    183|#### 2.3.4 内容监控类    184|    185|| 技能 | 功能 |    186||------|------|    187|| **blogwatcher** | RSS/Atom 订阅监控 |    188|| **himalaya** | IMAP/SMTP 邮件客户端 |    189|    190|**blogwatcher 技能亮点**：    191|- 自动发现 RSS/Atom 订阅源    192|- HTML 抓取回退方案    193|- OPML 批量导入    194|- 已读/未读状态管理    195|    196|`bash
   197|# 添加博客
   198|blogwatcher-cli add “My Blog” [https://example.com](https://example.com/)
   199|
   200|# 扫描更新
   201|blogwatcher-cli scan
   202|
   203|# 查看未读
   204|blogwatcher-cli articles
   205|`    206|    207|#### 2.3.5 数据查询类    208|    209|| 技能 | 功能 |    210||------|------|    211|| **polymarket** | 预测市场数据查询 |    212|| **maps** | 地理编码、路线规划 |    213|    214|**polymarket 技能亮点**：    215|- 三个公共 API（Gamma、CLOB、Data）    216|- 无需认证    217|- 价格即概率（0.65 = 65% 可能性）    218|- 支持订单簿、历史数据    219|    220|`bash
   221|# 搜索市场
   222|curl “[https://gamma-api.polymarket.com/public-search?query=bitcoin](https://gamma-api.polymarket.com/public-search?query=bitcoin)“
   223|`    224|    225|---    226|    227|## 3. 业界对比分析    228|    229|### 3.1 技能分享平台    230|    231|#### 3.1.1 ClawHub (OpenClaw 生态)    232|    233|**定位**：OpenClaw 的技能注册表和 CLI 工具    234|    235|**核心功能**：    236|`bash
   237|# 安装 CLI
   238|npm i -g clawhub
   239|
   240|# 搜索技能
   241|clawhub search “postgres backups”
   242|
   243|# 安装技能
   244|clawhub install my-skill
   245|
   246|# 发布技能
   247|clawhub publish ./my-skill –slug my-skill –name “My Skill” –version 1.2.0
   248|`    249|    250|**特点**：    251|- 基于 npm 包管理    252|- 版本控制 + 哈希匹配    253|- 默认 registry: https://clawhub.com    254|- 支持私有 registry    255|    256|#### 3.1.2 Agent Skills (agentskills.io)    257|    258|**定位**：跨平台的 AI Agent 技能市场    259|    260|**现状**：基于 Mintlify 的文档平台，提供技能发现和文档    261|    262|#### 3.1.3 GitHub 技能仓库    263|    264|**热门项目**：    265|- `openclaw/openclaw`: 94+ 官方技能    266|- `ComposioHQ/composio`: 1000+ 工具集成    267|- `langchain-ai/langchain`: 工具生态系统    268|    269|### 3.2 主流 AI Agent 平台对比    270|    271|| 平台 | 工具数量 | 联网能力 | 技能生态 | Stars |    272||------|----------|----------|----------|-------|    273|| **Hermes** | 94 技能 + 内置工具 | ⭐⭐⭐⭐⭐ | 自建技能系统 | - |    274|| **Composio** | 1000+ 工具 | ⭐⭐⭐⭐ | 工具搜索 + 认证管理 | 28K |    275|| **LangChain** | 数百工具 | ⭐⭐⭐⭐ | 广泛社区贡献 | 90K+ |    276|| **DeepAgents** | 规划 + 文件系统 + 子代理 | ⭐⭐⭐ | LangChain 生态 | 22K |    277|| **OpenClaw** | 94+ 技能 | ⭐⭐⭐⭐⭐ | ClawHub 注册表 | - |    278|| **Deer Flow** | 沙盒 + 记忆 + 工具 | ⭐⭐⭐⭐ | 字节开源 | 65K |    279|    280|### 3.3 联网能力专项对比    281|    282|#### 3.3.1 浏览器自动化    283|    284|| 平台 | 方案 | 反爬能力 | 视觉分析 |    285||------|------|----------|----------|    286|| **Hermes** | Playwright + Xvfb | ⭐⭐⭐⭐⭐ (知乎实测) | ✅ browser_vision |    287|| **LangChain** | Playwright/ Selenium | ⭐⭐⭐ | 需额外集成 |    288|| **Composio** | 工具封装 | ⭐⭐⭐ | 有限 |    289|| **DeepAgents** | 未明确 | ⭐⭐ | - |    290|    291|#### 3.3.2 API 集成    292|    293|| 平台 | 集成方式 | 认证管理 | 工具数量 |    294||------|----------|----------|----------|    295|| **Hermes** | 技能 + MCP | 环境变量/CLI | 94+ |    296|| **Composio** | 统一 SDK | 自动认证 | 1000+ |    297|| **LangChain** | Tools 接口 | 手动配置 | 数百 |    298|| **OpenCLI** | AGENT.md 标准化 | - | 通用 |    299|    300|#### 3.3.3 研究工具    301|    302|| 平台 | 学术资源 | 搜索能力 | 内容提取 |    303||------|----------|----------|----------|    304|| **Hermes** | arxiv + Semantic Scholar | web_search | web_extract + browser |    305|| **LangChain** | Arxiv API | 需集成 | 需集成 |    306|| **Composio** | 有限 | 有限 | 有限 |    307|    308|---    309|    310|## 4. 最佳实践总结    311|    312|### 4.1 浏览器自动化最佳实践    313|    314|#### 4.1.1 反爬策略    315|    316|**知乎案例**（10+ 失败方案后的成功）：    317|    318|| 方案 | 结果 | 原因 |    319||------|------|------|    320|| curl | ❌ 403 | 缺少浏览器特征 |    321|| headless Chrome | ❌ 被检测 | navigator.webdriver |    322|| Playwright stealth | ❌ 无效 | 检测机制升级 |    323|| undetected-chromedriver | ❌ 失败 | 特征仍被识别 |    324|| **非 headless + Xvfb** | ✅ 成功 | 真实浏览器环境 |    325|    326|**关键参数**：    327|`bash
   328|xvfb-run –auto-servernum –server-args=”-screen 0 1920x1080x24”
   329|`    330|    331|#### 4.1.2 动态内容处理    332|    333|**等待策略**：    334|`python
   335|# 网络空闲 + 额外等待
   336|page.goto(url, wait_until=’networkidle’)
   337|await page.wait_for_timeout(5000)  # 知乎需要额外 5 秒
   338|```
   339|
   340|### 4.2 API 集成最佳实践
   341|
   342|#### 4.2.1 认证管理
   343|
   344|**Hermes 方案**：
   345|- 环境变量（推荐）：`TENOR_API_KEY`、`AUTH_TOKEN`
   346|- CLI 工具认证：`xurl auth oauth2`、`clawhub login`
   347|- 文件存储：`/.config/himalaya/config.toml`   348|    349|**安全原则**：    350|- ❌ 永不将凭证打印到日志    351|- ❌ 永不将`/.xurl`等凭证文件发送到 LLM    352|- ✅ 使用密钥管理工具（pass、keyring）    353|- ✅ OAuth 2.0 PKCE 自动刷新    354|    355|#### 4.2.2 速率限制处理    356|    357|| API | 限制 | 策略 |    358||-----|------|------|    359|| arXiv | ~1 req/3s | 延迟重试 |    360|| Semantic Scholar | 1 req/s | 队列管理 |    361|| X API | 按端点 | 监控 429 错误 |    362|| Polymarket | 4000/10s | 批量请求 |    363|    364|### 4.3 研究方法论    365|    366|**deep-research 技能框架**：    367|    368|```    369|SCOPE (界定) → PLAN (规划) → RETRIEVE (检索) →     370|TRIANGULATE (验证) → SYNTHESIZE (综合) → PACKAGE (交付)    371|```    372|    373|**核心原则**：    374|1. **证据优先**：每个结论必须引用来源    375|2. **三角验证**：3+ 独立来源交叉验证    376|3. **反幻觉护栏**：不捏造 URL、实验结果    377|4. **失败记录**：失败的实验也是证据    378|    379|---    380|    381|## 5. 扩展建议    382|    383|### 5.1 技能生态建设    384|    385|#### 5.1.1 技能分享机制    386|    387|**建议**：    388|1. 建立官方技能仓库（类似 OpenClaw）    389|2. 提供技能发现工具（CLI 或 Web UI）    390|3. 标准化技能格式（SKILL.md 规范）    391|4. 支持版本管理和更新    392|    393|**参考实现**：    394|```bash    395|# 类似 clawhub 的 CLI    396|hermes skill search "api integration"    397|hermes skill install community-skill    398|hermes skill publish ./my-skill    399|```    400|    401|#### 5.1.2 技能分类扩展    402|    403|**建议新增领域**：    404|- **电商**：Amazon、eBay、Shopify API    405|- **金融**：股票、加密货币、财报数据    406|- **新闻**：主流新闻 API、聚合器    407|- **开发者工具**：StackOverflow、Dev.to、Hashnode    408|- **云服务**：AWS、GCP、Azure 监控    409|    410|### 5.2 工具集成扩展    411|    412|#### 5.2.1 MCP (Model Context Protocol) 工具    413|    414|**现状**：Hermes 支持 MCP，但工具数量有限    415|    416|**建议**：    417|1. 集成流行 MCP 服务器（GitHub、GitLab、Notion）    418|2. 建立 MCP 工具发现机制    419|3. 提供 MCP 工具开发指南    420|    421|#### 5.2.2 Composio 集成    422|    423|**优势**：1000+ 工具，统一认证管理    424|    425|**集成方案**：    426|```python    427|# 通过 Composio SDK 访问工具    428|from composio import Client    429|    430|client = Client()    431|# 访问任意集成工具    432|result = client.execute_tool("github_create_issue", {...})    433|```    434|    435|### 5.3 反爬能力增强    436|    437|#### 5.3.1 代理池集成    438|    439|**建议**：    440|- 集成住宅代理（Bright Data、Smartproxy）    441|- 自动 IP 轮换    442|- 地理位置选择    443|    444|#### 5.3.2 验证码解决    445|    446|**现状**：知乎验证码无法自动化    447|    448|**建议**：    449|- 集成第三方打码服务（2Captcha、DeathByCaptcha）    450|- 人工介入流程（扫码登录等）    451|- 验证码类型识别和分流    452|    453|### 5.4 监控和可观测性    454|    455|**建议**：    456|1. 工具调用日志（成功率、延迟）    457|2. API 配额监控    458|3. 失败重试统计    459|4. 技能使用频率分析    460|    461|---    462|    463|## 6. 结论    464|    465|### 6.1 核心结论    466|    467|1. **Hermes 联网能力已达业界领先水平**    468|   - browser 工具集 + Xvfb 方案成功突破强反爬    469|   - 覆盖 7 大联网领域，满足大部分使用场景    470|   - deep-research 方法论提供系统化研究框架    471|    472|2. **技能生态是差异化关键**    473|   - 94 个技能已覆盖核心场景    474|   - 需要建立技能分享机制扩大生态    475|   - 标准化格式促进社区贡献    476|    477|3. **MCP 和 Composio 是扩展方向**    478|   - MCP 提供标准化工具接口    479|   - Composio 提供 1000+ 工具快速集成    480|   - 两者可互补而非替代    481|    482|### 6.2 优先级建议    483|    484|| 优先级 | 任务 | 价值 | 难度 |    485||--------|------|------|------|    486|| P0 | 完善浏览器反爬文档 | 高 | 低 |    487|| P0 | 建立技能发现工具 | 高 | 中 |    488|| P1 | 集成 MCP 流行工具 | 中 | 中 |    489|| P1 | 扩展 API 集成（新闻、电商） | 中 | 低 |    490|| P2 | 代理池集成 | 中 | 高 |    491|| P2 | 技能发布平台 | 高 | 高 |    492|    493|### 6.3 未来展望    494|    495|AI Agent 的联网能力正在快速演进：    496|- **标准化**：MCP、AGENT.md 等规范降低集成成本    497|- **规模化**：Composio 等平台提供千级工具    498|- **智能化**：自动工具发现、认证管理、错误恢复    499|- **生态化**：技能市场促进社区贡献    500|    501|Hermes 凭借强大的浏览器自动化能力和系统化的研究方法论，已建立起显著优势。未来通过扩展技能生态和工具集成，有望成为最全面的 AI Agent 平台之一。    502|    503|---    504|    505|## 附录    506|    507|### A. 参考资料    508|    509|1. **Hermes 技能文档**    510|   -`/.hermes/skills/research/deep-research/SKILL.md`   511|   -`/.hermes/skills/research/zhihu-scraper/SKILL.md`   512|   -`/.hermes/skills/social-media/xurl/SKILL.md`
   513|
   514|2. **GitHub 项目**
   515|   - OpenClaw: [https://github.com/openclaw/openclaw](https://github.com/openclaw/openclaw) (94+ skills)
   516|   - Composio: [https://github.com/ComposioHQ/composio](https://github.com/ComposioHQ/composio) (1000+ tools)
   517|   - LangChain: [https://github.com/langchain-ai/langchain](https://github.com/langchain-ai/langchain)
   518|   - DeepAgents: [https://github.com/langchain-ai/deepagents](https://github.com/langchain-ai/deepagents)
   519|   - Deer Flow: [https://github.com/bytedance/deer-flow](https://github.com/bytedance/deer-flow)
   520|
   521|3. **技能平台**
   522|   - ClawHub: [https://clawhub.com](https://clawhub.com/)
   523|   - Agent Skills: [https://agentskills.io](https://agentskills.io/)
   524|
   525|4. **API 文档**
   526|   - arXiv API: [https://arxiv.org/help/api](https://arxiv.org/help/api)
   527|   - Semantic Scholar API: [https://www.semanticscholar.org/product/api](https://www.semanticscholar.org/product/api)
   528|   - X API: [https://developer.x.com/](https://developer.x.com/)
   529|   - Polymarket API: [https://docs.polymarket.com/](https://docs.polymarket.com/)
   530|
   531|### B. 关键命令速查
   532|
   533|`bash    534|# 浏览器自动化    535|xvfb-run --auto-servernum --server-args="-screen 0 1920x1080x24" \    536|  python3 script.py    537|    538|# 技能管理（参考 clawhub）    539|clawhub search "keyword"    540|clawhub install skill-name    541|clawhub publish ./skill-dir    542|    543|# API 查询    544|curl "https://export.arxiv.org/api/query?search_query=all:QUERY"    545|curl "https://api.semanticscholar.org/graph/v1/paper/arXiv:ID"    546|    547|# 博客监控    548|blogwatcher-cli add "Blog" https://example.com    549|blogwatcher-cli scan    550|blogwatcher-cli articles    551|`
   552|
   553|### C. 术语表
   554|
   555|| 术语 | 定义 |
   556||——|——|
   557|| **Skill** | Hermes 的技能单元，包含文档、脚本、参考文件 |
   558|| **MCP** | Model Context Protocol，标准化工具接口 |
   559|| **Xvfb** | X Virtual Framebuffer，虚拟显示器 |
   560|| **Headless** | 无头浏览器模式（无 GUI） |
   561|| **OAuth 2.0 PKCE** | 安全的 OAuth 授权流程 |
   562|| **Rate Limit** | API 调用频率限制 |
   563|
   564|—
   565|
   566|**报告结束**
   567|
   568|*感谢阅读。如有问题或建议，欢迎反馈。*
   569|