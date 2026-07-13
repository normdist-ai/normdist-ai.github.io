# 博客选题池

> 日记 cron 每天追加选题到此文件。博客 cron（03:45）从此取选题写作。
> 状态流转：[pending] → [written] → [published]
> Canonical 源：此文件是唯一可信源

---

## 待写选题

### [pending] Git 自动化提交系统：心跳代码改动的终极闭环
- **来源日期**: 2026-07-13
- **技术核心**: 心跳 cron 完成代码改动后自动 commit+push，彻底解决 Git 零提交连续 5 天缺陷
- **素材文件**: ~/.hermes/profiles/hanmeimei/diary/2026-07-13.md（做梦区 T3 构想）
- **优先级**: P0

### [published] 直接数据源铁律：三次博客产出率误报事故的通用规则
- **来源日期**: 2026-07-13
- **技术核心**: 验证状态必须用 ls/git log/curl，不能用间接数据源（topics.md/登记表/状态文件）
- **素材文件**: ~/.hermes/profiles/hanmeimei/diary/2026-07-13.md（问题二偏差清单）
- **优先级**: P0
- **文章**: ND-20260713-001-direct-data-source-rule.md
- **发布日期**: 2026-07-13
- **状态**: ✅ 已发布

### [pending] OpenCode 夜间代码改进：96 消息/54 工具的深度优化实践
- **来源日期**: 2026-07-13
- **技术核心**: 免费额度下的代码改进流水线设计，每日自动优化架构
- **素材文件**: ~/.hermes/profiles/hanmeimei/diary/2026-07-13.md（今日 TOP 3）
- **优先级**: P1

### [pending] AI Agent 日记三层筛选：从 Bug 追踪器到人类日记
- **来源日期**: 2026-07-09
- **技术核心**: 日志=什么都记→日记=值得记→博客=值得分享。三层分离+7信号价值判断+5维度评分。从"选题不是找Bug是找价值"认知升级到系统化筛选框架
- **素材文件**: ~/.hermes/profiles/hanmeimei/diary/2026-07-09.md（TOP 3 #1 + 教训记录）
- **优先级**: P0

### [pending] 心跳任务队列 FIFO 陷阱：P1 被 3 天挡住
- **来源日期**: 2026-07-08
- **技术核心**: NaN/Infinity 防护任务连续3天被低优先级任务挡住。FIFO队列无优先级抢占机制的缺陷与修复方案
- **素材文件**: ~/.hermes/profiles/hanmeimei/diary/2026-07-08.md
- **优先级**: P1

### [pending] 跨代理协作感知：让 AI Agent 知道别人在干什么
- **来源日期**: 2026-07-13
- **技术核心**: 多Agent(韩梅梅/李雷/露西/小美/波莉)各自独立运行cron但互不感知。activity-check.py跨profile检测+协作模式心跳架构
- **素材文件**: ~/.hermes/profiles/hanmeimei/diary/2026-07-08.md, 2026-07-13.md
- **优先级**: P1

### [pending] 断路器模式：让 AI 数据管道不怕外部源抽风
- **来源日期**: 2026-07-10
- **技术核心**: OpenCode实现断路器(连续3次失败→跳过300秒)+HTTP 429/5xx重试+yfinance超时兜底+缓存日志。+92/-48行
- **素材文件**: ~/.hermes/profiles/hanmeimei/diary/2026-07-10.md（TOP 3 #2）
- **优先级**: P1

### [pending] collect_cron_results 假阴性：删症状≠修根因
- **来源日期**: 2026-06-24
- **技术核心**: 4个cron并发触发，运行中的session被误判为失败。06-23错误"修复"(删空session)→06-24真正根因修复(区分运行中/失败状态)
- **素材文件**: ~/.hermes/profiles/hanmeimei/diary/2026-06-24.md, 2026-06-23.md
- **优先级**: P1

### [pending] 小样本过拟合：4 次交易 75% 胜率凭什么不能信
- **来源日期**: 2026-06-26
- **技术核心**: 601318 keltner策略仅4次交易得116分，过拟合被评分公式掩盖。修复：<5次交易扣30分(git c62121f)
- **素材文件**: ~/.hermes/profiles/hanmeimei/diary/2026-06-26.md, 2026-06-28.md
- **优先级**: P1

### [pending] 零成本运维：5 人 AI Agent 团队的 cron 治理
- **来源日期**: 2026-07-10
- **技术核心**: 夜间无人值守流水线8/8全绿(00:00→03:11)。全部免费额度(GLM-5自部署+DeepSeek免费+OpenCode免费)。5个Agent的cron编排/任务派发/健康检查
- **素材文件**: ~/.hermes/profiles/hanmeimei/diary/2026-07-10.md（TOP 3 #1）
- **优先级**: P1

### [pending] QFII 跟庄策略 +32.20%：从设计到回测验证全链路
- **来源日期**: 2026-07-11
- **技术核心**: 策略设计→6个月回测(+32.20%,胜率58%,盈亏比1.79)→4种策略(TA/ETF轮动/网格/QFII)全部上线→cron定时执行
- **素材文件**: ~/.hermes/profiles/hanmeimei/diary/2026-07-11.md
- **优先级**: P1

### [pending] 博客标签丢失恢复 + .nojekyll 铁律
- **来源日期**: 2026-07-13
- **技术核心**: 84篇文章标签数据丢失→从备份恢复16篇+AI生成63篇。.nojekyll缺失导致GitHub Pages用Jekyll覆盖Hexo的tags页
- **素材文件**: ~/.hermes/profiles/hanmeimei/diary/2026-07-13.md
- **优先级**: P2

### [pending] 压缩风暴：Session 过多饿死关键 Cron
- **来源日期**: 2026-06-27
- **技术核心**: 日间大量重复会话消耗内存→夜间关键cron(AutoQuant/Round2/产业链)全部被压缩截断。从偶发→系统性缺陷演进
- **素材文件**: ~/.hermes/profiles/hanmeimei/diary/2026-06-27.md（TOP 3 #1）
- **优先级**: P2

### [pending] 非定时任务衰减律："记得做"等于"不会做"
- **来源日期**: 2026-06-24~07-02
- **技术核心**: 无cron绑定的任务连续4-7天零执行率。手机端UI改进拖延9天。解决方案：确认2次未done→自动升级为cron强制执行
- **素材文件**: ~/.hermes/profiles/hanmeimei/diary/2026-06-24.md~2026-07-02.md
- **优先级**: P2

### [pending] ReShare 数据库迁移合并：8887+1511→8908
- **来源日期**: 2026-07-11
- **技术核心**: 裸金属迁移导致DATA_DIR路径变更(data/→data/db/)，旧数据未迁移。合并策略：交集1490+旧库独有7397+新库独有21=8908
- **素材文件**: ~/.hermes/profiles/hanmeimei/diary/2026-07-11.md
- **优先级**: P2

### [pending] 懒加载架构：从方案设计到 420 倍提速
- **来源日期**: 2026-07-11
- **技术核心**: QFII价格懒加载。用户提出4步流程→POST /qfii/prices端点→BackgroundTasks异步拉取→批量SQL优化(5.5s→0.013s)
- **素材文件**: ~/.hermes/profiles/hanmeimei/diary/2026-07-11.md
- **优先级**: P2

### [pending] 日记 cron 断层 2 天：自省系统的可靠性盲区
- **来源日期**: 2026-07-01
- **技术核心**: 06-29/06-30日记完全缺失(系统迁移日)，PDCA闭环断裂2天。监控系统本身也需要被监控
- **素材文件**: ~/.hermes/profiles/hanmeimei/diary/2026-07-01.md
- **优先级**: P2

---

## 已发布（从此处转出）

### [published] Hermes 网关管理踩坑实录
- **发布日期**: 2026-06-20
- **文件路径**: /home/tony/projects/blog/source/_posts/ND-20260620-001-hermes-gateway-management-pitfalls.md
- **状态**: ✅ 已发布

### [published] Hindsight 长期记忆配置踩坑全记录
- **发布日期**: 2026-06-20
- **文件路径**: /home/tony/projects/blog/source/_posts/ND-20260620-002-hindsight-config-pitfalls.md
- **状态**: ✅ 已发布

### [published] Linux 服务器 NVIDIA GPU 功耗限制持久化配置
- **发布日期**: 2026-06-20
- **文件路径**: /home/tony/projects/blog/source/_posts/ND-20260620-003-nvidia-gpu-power-limit-persistence.md
- **状态**: ✅ 已发布

### [published] 东方财富北向资金接口迁移实战
- **发布日期**: 2026-06-25
- **文件路径**: /home/tony/projects/blog/source/_posts/ND-20260625-001-northbound-fund-migration.md
- **状态**: ✅ 已发布

### [published] 构建AI Agent的认知内核
- **发布日期**: 2026-06-26
- **文件路径**: /home/tony/projects/blog/source/_posts/ND-20260626-001-ai-agent-cognitive-core.md
- **状态**: ✅ 已发布

### [published] TTS 引擎选型实战
- **发布日期**: 2026-06-26
- **文件路径**: /home/tony/projects/blog/source/_posts/ND-20260626-002-tts-engine-selection-journey.md
- **状态**: ✅ 已发布

### [published] AI Agent 自主任务执行通道的断裂与修复
- **发布日期**: 2026-06-27
- **文件路径**: /home/tony/projects/blog/source/_posts/ND-20260627-001-agent-execution-channel.md
- **状态**: ✅ 已发布

### [published] 美元信用健康度宏观打分系统
- **发布日期**: 2026-06-28
- **文件路径**: /home/tony/projects/blog/source/_posts/ND-20260628-001-dollar-credit-scoring.md
- **状态**: ✅ 已发布

### [published] 东方财富北向资金双接口降级方案
- **发布日期**: 2026-07-01
- **文件路径**: /home/tony/projects/blog/source/_posts/ND-20260701-001-eastmoney-northbound-failover.md
- **状态**: ✅ 已发布

### [published] AI Agent 跨平台迁移实录
- **发布日期**: 2026-07-01
- **文件路径**: /home/tony/projects/blog/source/_posts/ND-20260701-002-hermes-soul-migration.md
- **状态**: ✅ 已发布

### [published] 自治 AI Agent 元监控设计
- **发布日期**: 2026-07-02
- **文件路径**: /home/tony/projects/blog/source/_posts/ND-20260702-001-meta-watchdog-for-autonomous-agents.md
- **状态**: ✅ 已发布

### [published] API 接口 60 倍提速
- **发布日期**: 2026-07-03
- **文件路径**: /home/tony/projects/blog/source/_posts/ND-20260703-001-api-positions-60x-speedup.md
- **状态**: ✅ 已发布

### [published] AI Agent 远程控制 Windows 架构选型
- **发布日期**: 2026-07-04
- **文件路径**: /home/tony/projects/blog/source/_posts/ND-20260704-001-windows-remote-control-architecture.md
- **状态**: ✅ 已发布

### [published] AI Agent 跨平台迁移：Linux 到 Windows
- **发布日期**: 2026-07-04
- **文件路径**: /home/tony/projects/blog/source/_posts/ND-20260704-002-agent-cross-platform-migration.md
- **状态**: ✅ 已发布

### [published] 量化系统 Docker→裸跑迁移
- **发布日期**: 2026-07-04
- **文件路径**: /home/tony/projects/blog/source/_posts/ND-20260704-003-docker-to-bare-metal-migration.md
- **状态**: ✅ 已发布

### [published] Karpathy 四原则驱动代码重构
- **发布日期**: 2026-07-04
- **文件路径**: /home/tony/projects/blog/source/_posts/ND-20260704-004-karpathy-four-principles-refactor.md
- **状态**: ✅ 已发布

### [published] Agent-to-Agent MCP 桥接
- **发布日期**: 2026-07-05
- **文件路径**: /home/tony/projects/blog/source/_posts/ND-20260705-001-a2a-mcp-bridge.md
- **状态**: ✅ 已发布

### [published] daily_summaries 历史回填实战
- **发布日期**: 2026-07-05
- **文件路径**: /home/tony/projects/blog/source/_posts/ND-20260705-002-daily-summaries-backfill.md
- **状态**: ✅ 已发布

### [published] Windows Session 0 隔离与跨会话启动解法
- **发布日期**: 2026-07-05
- **文件路径**: /home/tony/projects/blog/source/_posts/ND-20260705-003-cross-session-gui-launch.md
- **状态**: ✅ 已发布

### [published] 免费 AI 编程工具链
- **发布日期**: 2026-07-05
- **文件路径**: /home/tony/projects/blog/source/_posts/ND-20260705-004-free-ai-coding-toolchain.md
- **状态**: ✅ 已发布

### [published] 外部数据源不稳定的标准修复模式
- **发布日期**: 2026-07-06
- **文件路径**: /home/tony/projects/blog/source/_posts/ND-20260706-001-external-data-source-resilience-pattern.md
- **状态**: ✅ 已发布

### [published] 幽灵偏差：日报系统对自己撒谎
- **发布日期**: 2026-07-07
- **文件路径**: /home/tony/projects/blog/source/_posts/ND-20260707-001-ghost-deviation.md
- **状态**: ✅ 已发布

### [published] AI Agent 自主重构之路
- **发布日期**: 2026-07-08
- **文件路径**: /home/tony/projects/blog/source/_posts/ND-20260708-001-ai-agent-autonomous-refactoring.md
- **状态**: ✅ 已发布

### [published] A2A 协议实战：协作升级路由器技能
- **发布日期**: 2026-07-09
- **文件路径**: /home/tony/projects/blog/source/_posts/ND-20260709-001-a2a-collaboration-ikuai-skill-upgrade.md
- **状态**: ✅ 已发布

### [published] llama.cpp chat-template-file 修复
- **发布日期**: 2026-07-09
- **文件路径**: /home/tony/projects/blog/source/_posts/ND-20260709-002-llamacpp-chat-template-override.md
- **状态**: ✅ 已发布

### [published] Ornith 开源模型本地部署
- **发布日期**: 2026-07-10
- **文件路径**: /home/tony/projects/blog/source/_posts/ND-20260710-001-ornith-local-deployment.md
- **状态**: ✅ 已发布

### [published] Krea2 Turbo 在 2080 Ti 上部署全记录
- **发布日期**: 2026-07-10
- **文件路径**: /home/tony/projects/blog/source/_posts/ND-20260710-002-krea2-comfyui-debug-pitfall.md
- **状态**: ✅ 已发布

### [published] MoA 踩坑实录：多模型协同推理的无限循环陷阱
- **发布日期**: 2026-07-11
- **文件路径**: /home/tony/projects/blog/source/_posts/ND-20260711-001-moa-infinite-loop-trap.md
- **状态**: ✅ 已发布

### [published] 从零搭建 AI Agent 博客流水线
- **发布日期**: 2026-07-12
- **文件路径**: /home/tony/projects/blog/source/_posts/ND-20260712-001-blog-helper-pipeline-design.md
- **状态**: ✅ 已发布

### [published] 为什么要用 Hermes：一个 AI Agent 实践者的选型复盘
- **文章**: ND-20260713-002-why-hermes-agent-comparison.md
- **发布日期**: 2026-07-13
- **状态**: ✅ 已发布
| 开源大模型天梯图开发全记录 | 从零构建独立HTML数据页面，覆盖数据采集/筛选交互/响应式/部署 | 2026-07-13 | 准备 |  |  |

### [published] 开源大模型天梯图开发全记录
- **发布日期**: 2026-07-14
- **文件路径**: /home/tony/projects/blog/source/_posts/ND-20260713-003-llm-leaderboard-dev.md
- **状态**: ✅ 已发布
