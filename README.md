# Information Tracking Agent

多平台知识萃取 Agent — 从信息洪流中捕获决策级信号，写入 Obsidian，推送 Telegram。

[English version](README_EN.md)

---

## 为什么造这个轮子

RSS 阅读器遍地都是。但它们做的事本质上一样：把所有东西搬到你面前，让你自己筛。

问题是你筛不过来。200 篇文章里 190 篇是噪音，剩下 10 篇里可能只有 1 篇能改变一个决策。你花在筛上的时间，比读那 1 篇还多。

所以这个项目不是又一个 RSS 阅读器。它是一个对"信息不对称"的对抗工具——你告诉它你在意什么，它替你读完 200 篇，把那 1 篇递到你手边。

而它做这件事的方式，不是靠规则匹配，而是靠一个在读了几百篇内容后、比你更清楚你下一篇想看什么的数字孪生。

---

## 设计哲学

这个系统经历了三代进化。每一代的核心区别不是功能多少，而是"谁在做决策"。

### V1：勤奋但愚笨的计算器

V1 是传统后端工程师的直觉产物——确定性流水线，Cron 定时唤醒，盲扫 7 天数据，逐个处理，推送。

它最大的问题不是慢，是**静默流血**。LLM API 挂了，系统不报错，自作聪明用字数长短猜分数。200 篇文章全拿到 7.0 高分通过初筛，深度分析环节全部静默失败。表面上全速运转，实际产出为零。

对 AI 系统来说，最可怕的不是罢工，是出错时没人知道。

### V2：从脚本到 Agent 的转折

V2 的核心信仰只有一个：**交出微观控制权**。

传统程序员思维总想掌控每一个边缘情况，但恰恰限制了 AI 最擅长的自适应能力。所以 V2 做了三件事：

**增量感知。** 调度器不再盲扫。它带着记忆醒来——知道自己上次读到了哪一页，只带回新东西。`RSS_DAYS_LIMIT = 7` 这种硬编码被彻底移除。有记忆时抓增量，没记忆时根据调度间隔动态计算回滚窗口。

**认知资源分层。** 旗舰模型不再做粗筛。系统模仿人类的"直觉反应"和"深度思考"——粗筛用轻量模型（行政助理），深挖用旗舰模型（智囊团）。这不是优化，是算力管理的哲学。

**Fail Loudly。** 全面推翻"API 失败就按字数打分"的降级逻辑。模型调用中断，立即触发红色告警，写入失败队列。宁可系统挂起，绝不放行数据污染知识库。

### V3：具备持续进化能力的数字生命体

V3 把"进化"本身变成了系统的能力。

**记忆分层。** 短期焦点捕获你最近 1-2 周的密集关切，自动调高相关信号的萃取权重。核心画像承载你稳定的价值观和思维模型。两者不是静态配置，而是通过你的阅读行为持续演化。

**从注入到进化。** 系统不被动接受配置。它通过历史萃取记录——你标记为"值得重读"的文章、你反复搜索的主题——反推你不断流动的兴趣点，并提议更新关注方向。

**技能内生化。** 这是最本质的架构变化。V2 的微信抓取像一只小龙虾——大螯是外挂的，每次要抓东西得先走出房间启动一台收割机，收完再捡起纸读。V3 把抓取能力直接植入进程内——不再是"指挥机器人干活的包工头"，而是一个正在不断生长出新器官的数字生命体。抓不到内容它会"痛"，碰到反爬机制它会"绕路"，工具链断了它会"自检"。

---

## 它怎么工作

```
信息源 (URL / RSS / IM)
  |
  v
QueueStore  --------  任务队列，SQLite 持久化
  |
  v
RuntimeGuard  ------  运行时隔离，拒绝污染路径和幽灵进程
  |
  v
Fetch  -------------  多平台抓取，9+ 渠道自适应
  |
  v
Score  -------------  四层评分引擎
  |                     L1 信号强度 x L2 来源可信度 x L3 时效性
  |                     = 客观质量
  |                     最终分 = 70% 客观质量 + 30% 个性化匹配 (L4)
  v
Extract  ------------  LLM 深度萃取，输出结构化知识卡片
  |
  v
Output  -------------  Obsidian Markdown + Telegram 推送
```

评分合约：

```python
objective_quality = L1 * L2 * L3
final_score       = 0.70 * objective_quality + 0.30 * L4
score             = final_score * 10  # 0-10 分制
```

每一个环节的失败都有类型化处理——`retry_scheduled`、`rejected`、`failed_terminal`。绝不把失败标记为成功。

---

## 支持的平台

| 平台 | 抓取方式 | 说明 |
|------|---------|------|
| YouTube | yt-dlp | 视频元数据与字幕 |
| Twitter/X | xreach / MCP | 推文与线程 |
| Reddit | Jina Reader | 帖子与评论 |
| V2EX | Jina Reader | 讨论帖 |
| Hacker News | Jina Reader | 技术讨论 |
| 微信公众号 | Agent-Reach | 文章正文（内置技能） |
| 小宇宙 FM | Jina Reader | 播客元数据 |
| RSS/Atom | stdlib xml | 订阅源聚合 |
| Web | Jina Reader | 任意 URL |

**MCP 扩展：** 系统架构支持通过 MCP 协议接入外部信息源管理服务（如 Mindspace MCP），实现搜索、频道管理和文章检索的扩展，无需修改核心代码。

**Web 搜索：** 支持 Exa API 直接调用或通过 MCP 模式工作，支持分类过滤、域名限制、时效筛选。

---

## 快速开始

### 安装

```bash
git clone https://github.com/bor799/information-tracking-agent.git
cd information-tracking-agent
pip install -e .
```

### 配置

```bash
cp config/config.example.yaml config/config.local.yaml
cp .env.example .env
```

编辑 `.env` 填入 API key（至少需要一个 LLM 提供商）。详细配置见 [docs/CONFIGURATION.md](docs/CONFIGURATION.md)。

### 运行

```bash
# 入队一条 URL
information-tracking-agent enqueue-url "https://example.com/article"

# 处理队列中所有待处理任务（staging 模式）
information-tracking-agent worker-once --limit 10

# 查看队列状态
scripts/control.sh status
```

---

## 项目结构

```
src/knowledge_extractor_v3/
  __init__.py
  shadow_runner.py       # CLI 入口
  pipeline.py            # 处理管线编排
  worker.py              # 队列消费与批处理
  scheduler.py           # 定时源调度
  queue_store.py         # SQLite 任务队列
  runtime_guard.py       # 运行时隔离与指纹
  config_loader.py       # 分层配置加载
  models.py              # 数据模型
  health.py              # 健康检查
  prompt_registry.py     # Prompt 版本管理
  prompt_parser.py       # LLM 输出解析
  llm/                   # LLM 提供商抽象
  fetchers/              # 多渠道抓取适配器
  sources/               # 信息源注册与去重
  outputs/               # Obsidian + Telegram 输出
```

40 个源文件，24 个测试文件，224+ 测试用例。

---

## 测试

```bash
pip install -e ".[dev]"
python -m pytest -q              # 224+ tests
python -m compileall src tests   # 编译检查
```

CI 自动运行于 Python 3.11 / 3.12 / 3.13。

---

## 文档

| 文档 | 说明 |
|------|------|
| [配置指南](docs/CONFIGURATION.md) | 环境变量、YAML 配置、安全默认值 |
| [系统架构](docs/architecture.md) | 数据流、队列合约、Prompt 合约 |
| [错误架构](docs/error-architecture.md) | 9 类历史错误与 V3 结构性防御 |
| [搜索能力](docs/SEARCH_CAPABILITIES.md) | Exa 搜索、MCP 模式 |
| [RSS 架构分析](docs/RSS_FETCH_ARCHITECTURE_ANALYSIS.md) | RSS 抓取失败分析与优化 |

---

## 命令参考

```bash
# 入队
information-tracking-agent enqueue-url <URL>

# 处理
information-tracking-agent worker-once --limit N --mode live|staging

# 系统管理
scripts/control.sh status    # 查看状态
scripts/control.sh recover   # 恢复常驻服务
```

---

## License

MIT
