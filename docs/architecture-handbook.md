# Daily Stock Analysis — 架构手册

> 本文档基于当前代码库自动分析生成，描述系统整体架构、执行流程与关键链路。
> 
> 如与代码实际行为不符，以代码为准。

---

## 目录

1. [项目定位](#1-项目定位)
2. [目录结构](#2-目录结构)
3. [入口点](#3-入口点)
4. [核心执行流程](#4-核心执行流程)
5. [数据流](#5-数据流)
6. [关键模块说明](#6-关键模块说明)
7. [外部集成](#7-外部集成)
8. [配置系统](#8-配置系统)
9. [Web / 桌面端](#9-web--桌面端)
10. [调度机制](#10-调度机制)
11. [CI/CD 流水线](#11-cicd-流水线)
12. [架构分层总览](#12-架构分层总览)

---

## 1. 项目定位

股票智能分析系统，覆盖 A 股、港股、美股。

**主流程：** 抓取市场数据 → 技术分析 / 新闻检索 → LLM 分析 → 生成报告 → 多渠道推送通知

**关键能力：**
- 支持多数据源自动 fallback（6 个数据提供者）
- 支持多 LLM 提供商（Gemini / DeepSeek / Claude / OpenAI / Ollama / 自定义）
- 支持多通知渠道（10+ 渠道同时推送）
- 支持多搜索引擎聚合新闻
- 提供 Web UI、桌面端、FastAPI、IM 机器人多种交互方式
- 支持回测、投资组合管理、Agent 多步骤分析

---

## 2. 目录结构

```
daily_stock_analysis/
├── main.py                      # 主 CLI 入口，所有运行模式的统一调度
├── server.py                    # uvicorn 独立启动 FastAPI 的薄封装
├── analyzer_service.py          # 独立 analyzer 服务封装
├── webui.py                     # 历史 WebUI 兼容 shim
├── .env.example                 # 所有配置项的规范模板（498 行，分组注释完整）
├── litellm_config.example.yaml  # LiteLLM 路由器 YAML 配置示例
│
├── src/                         # 核心 Python 后端包
│   ├── config.py                # Config 数据类，setup_env()，get_config() 单例
│   ├── logging_config.py        # setup_logging() — 文件 + 控制台日志
│   ├── scheduler.py             # run_with_schedule() — 阻塞式每日调度循环
│   ├── analyzer.py              # GeminiAnalyzer — LLM 抽象层（底层走 LiteLLM）
│   ├── stock_analyzer.py        # 单只股票分析编排
│   ├── market_analyzer.py       # 市场级别分析辅助
│   ├── notification.py          # NotificationService — 多渠道通知分发器
│   ├── search_service.py        # SearchService — 多搜索引擎新闻聚合
│   ├── storage.py               # SQLite 持久化助手
│   ├── auth.py                  # Web 登录认证
│   ├── feishu_doc.py            # 飞书文档创建（FeishuDocManager）
│   ├── md2img.py                # Markdown → 图片渲染
│   │
│   ├── core/                    # 核心流水线 & 业务逻辑
│   │   ├── pipeline.py          # StockAnalysisPipeline — 顶层编排器
│   │   ├── market_review.py     # run_market_review() — 市场复盘 LLM 分析
│   │   ├── trading_calendar.py  # 各市场交易日判断
│   │   ├── config_manager.py    # ConfigManager — 运行时读写 .env
│   │   ├── config_registry.py   # 配置 Schema 注册表
│   │   └── backtest_engine.py   # 回测核心计算引擎
│   │
│   ├── services/                # 应用服务层
│   │   ├── analysis_service.py         # API / Bot 触发分析的入口
│   │   ├── stock_service.py            # 股票 CRUD & 自选股
│   │   ├── history_service.py          # 历史分析查询
│   │   ├── history_comparison_service.py # 相邻分析对比
│   │   ├── backtest_service.py         # 回测运行 & 结果持久化
│   │   ├── portfolio_service.py        # 投资组合管理
│   │   ├── portfolio_risk_service.py   # 风险集中度 / 回撤预警
│   │   ├── portfolio_import_service.py # 从 CSV / 图片导入持仓
│   │   ├── agent_model_service.py      # Agent LLM 模型路由
│   │   ├── system_config_service.py    # 通过 API 读写系统配置
│   │   ├── task_service.py             # 异步任务生命周期
│   │   ├── task_queue.py               # 任务队列实现
│   │   ├── image_stock_extractor.py    # 视觉 LLM：从图片提取股票代码
│   │   ├── name_to_code_resolver.py    # 公司名称 → 股票代码解析
│   │   ├── report_renderer.py          # Jinja2 报告模板渲染
│   │   └── social_sentiment_service.py # 美股社交情绪（Reddit / X）
│   │
│   ├── repositories/            # 数据访问层（SQLite DAO）
│   ├── agent/                   # 多 Agent / 策略 Skill 系统
│   ├── notification_sender/     # 各渠道发送实现
│   ├── schemas/                 # Pydantic 请求/响应 Schema
│   └── utils/                   # 共享工具函数
│
├── data_provider/               # 市场数据抓取器注册表
│   ├── base.py                  # canonical_stock_code()，抓取器基类
│   ├── efinance_fetcher.py      # 东方财富（默认，优先级 0）
│   ├── akshare_fetcher.py       # AkShare（优先级 1）
│   ├── tushare_fetcher.py       # Tushare Pro（优先级 2）
│   ├── pytdx_fetcher.py         # 通达信（优先级 2）
│   ├── baostock_fetcher.py      # BaoStock（优先级 3）
│   ├── yfinance_fetcher.py      # Yahoo Finance（优先级 4，主要用于美股）
│   ├── tickflow_fetcher.py      # TickFlow API（A 股指数增强）
│   └── fundamental_adapter.py  # 跨数据源基本面数据聚合
│
├── api/                         # FastAPI 应用
│   ├── app.py                   # FastAPI 工厂，CORS，静态文件挂载
│   ├── deps.py                  # 依赖注入（config、services）
│   ├── middlewares/             # 认证、限流、日志中间件
│   └── v1/                      # /api/v1 路由处理器
│
├── bot/                         # 聊天机器人 / IM 集成
│   ├── handler.py               # 消息解析与分发
│   ├── dispatcher.py            # Bot 命令 → 服务路由
│   ├── commands/                # 各命令处理器
│   └── platforms/               # 钉钉 Stream、飞书 Stream 适配器
│
├── apps/
│   ├── dsa-web/                 # Vue 3 + Vite SPA（主 Web UI）
│   └── dsa-desktop/             # Electron 桌面端封装
│
├── strategies/                  # YAML 定义的交易策略 Skill 文件
├── templates/                   # Jinja2 报告模板
├── sources/                     # 新闻/研报来源配置
├── scripts/                     # 工具脚本（DB 迁移等）
├── docker/                      # Dockerfile、docker-compose 文件
├── docs/                        # 项目文档
├── data/                        # SQLite 数据库（运行时产物）
├── logs/                        # 日志文件（运行时产物）
├── reports/                     # 生成的报告输出（运行时产物）
└── .github/
    ├── workflows/               # CI/CD 流水线
    └── scripts/ai_review.py     # AI 辅助 PR 审查脚本
```

---

## 3. 入口点

### 3.1 `main.py` — 主 CLI 入口

所有运行模式由 CLI flag 选择（`main()` 从 line 615 开始）：

| Flag | 模式 | 说明 |
|---|---|---|
| _(无 flag)_ | 单次全量分析 | 执行完整分析流程后退出 |
| `--schedule` | 阻塞式每日调度 | 持续运行，按 `SCHEDULE_TIME` 定时触发 |
| `--market-review` | 仅市场复盘 | 不分析个股，只做市场级 LLM 复盘 |
| `--backtest` | 历史回测评估 | 对历史分析建议做回测验证 |
| `--serve` | API 服务 + 分析 | 启动 FastAPI 同时执行分析 |
| `--serve-only` / `--webui-only` | 仅 API 服务 | 启动服务但不自动分析 |

**常用 CLI 参数：**

| 参数 | 类型 | 用途 |
|---|---|---|
| `--debug` | flag | 开启详细日志 |
| `--dry-run` | flag | 仅抓取数据，跳过 LLM 分析 |
| `--stocks 600519,AAPL` | string | 覆盖 STOCK_LIST |
| `--no-notify` | flag | 禁止所有通知推送 |
| `--workers N` | int | 覆盖 MAX_WORKERS 线程数 |
| `--force-run` | flag | 跳过交易日历检查强制运行 |
| `--port` / `--host` | int/str | FastAPI 端口/地址（默认 8000/0.0.0.0） |

**启动引导顺序（main.py lines 31–61）：**
1. 快照初始 `os.environ` 状态（`_INITIAL_PROCESS_ENV`）
2. 调用 `setup_env()` 通过 `python-dotenv` 加载 `.env`
3. 按需配置 HTTP 代理（`USE_PROXY`）
4. env 就绪后才导入 pipeline 相关模块
5. `main()` 内：解析参数 → 加载 `Config` → `setup_logging()` → 分发到对应模式

### 3.2 `server.py` — 独立 FastAPI 入口

55 行薄封装，用于 `uvicorn server:app --reload` 开发模式：

1. 调用 `setup_env()` 和 `get_config()`
2. 调用 `setup_logging(log_prefix="api_server")`
3. 导入并重导出 `api.app:app`
4. 直接运行时：`uvicorn.run("server:app", host="0.0.0.0", port=8000, reload=True)`

---

## 4. 核心执行流程

### 4.1 全量分析流程（`run_full_analysis()`）

```
main()
  └─ run_full_analysis(config, args, stock_codes)
        │
        ├─ 1. config.refresh_stock_list()
        │      从 .env 热重载 STOCK_LIST（支持运行时修改后生效）
        │
        ├─ 2. _compute_trading_day_filter()
        │      └─ trading_calendar.get_open_markets_today()
        │      └─ 按市场过滤：非交易日的股票跳过不分析
        │
        ├─ 3. 实例化 StockAnalysisPipeline(config, max_workers, ...)
        │      （src/core/pipeline.py）
        │
        ├─ 4. pipeline.run(stock_codes, dry_run, ...)
        │      │
        │      ├─ [ThreadPoolExecutor, MAX_WORKERS 线程并发]
        │      │   对每只股票执行：
        │      │    ├─ data_provider  → 抓取 OHLCV、实时行情、基本面数据
        │      │    ├─ SearchService  → 聚合多引擎新闻
        │      │    ├─ GeminiAnalyzer.analyze() → LLM 生成分析结论
        │      │    │     └─ AnalysisResult（评分、建议、趋势判断、完整报告）
        │      │    ├─ repositories/ → 结果写入 SQLite
        │      │    └─ [若 single_notify] NotificationService.send() 立即推送
        │      │
        │      └─ [全部股票完成后]
        │           NotificationService.generate_aggregate_report()
        │           └─ [若非 single_notify] 发送汇总报告
        │
        ├─ 5. 可选：ANALYSIS_DELAY 等待
        │
        ├─ 6. run_market_review(notifier, analyzer, search_service, ...)
        │      ├─ 抓取主要指数数据（沪深300、创业板等）
        │      ├─ SearchService → 宏观/市场新闻
        │      ├─ LLM → 生成市场复盘评述
        │      └─ NotificationService.send()（或合并推送）
        │
        ├─ 7. 合并推送（MERGE_EMAIL_NOTIFICATION=true 时）
        │      将个股报告 + 市场复盘合并为单次推送
        │
        ├─ 8. FeishuDocManager.create_daily_doc()
        │      创建飞书云文档（可选）
        │
        └─ 9. BacktestService.run_backtest()
               对历史分析建议执行回测评估（可选）
```

### 4.2 市场复盘流程（`--market-review`）

```
run_market_review()
  ├─ 获取主要指数行情数据
  ├─ SearchService 抓取宏观/板块新闻
  ├─ GeminiAnalyzer 生成市场评述（max_tokens=8192）
  └─ NotificationService 推送复盘报告
```

### 4.3 回测流程（`--backtest`）

```
BacktestService.run_backtest()
  ├─ 从 SQLite 读取历史分析建议
  ├─ data_provider 获取对应时段行情
  ├─ backtest_engine 计算建议准确率
  └─ 结果持久化到 SQLite
```

### 4.4 API / Bot 触发分析流程

```
HTTP POST /api/v1/analysis/analyze
  └─ analysis_service.trigger_analysis()
        ├─ task_queue.py 异步任务入队
        ├─ task_service.py 管理任务生命周期
        └─ 复用 StockAnalysisPipeline（与 CLI 共用同一套核心逻辑）

Bot 消息（钉钉 / 飞书 Stream）
  └─ bot/handler.py → bot/dispatcher.py → bot/commands/
        └─ analysis_service.trigger_analysis()
```

---

## 5. 数据流

```
.env / CLI 参数
      │
      ▼
Config 单例（src/config.py）
      │
      ├─────────────────────────────────────┐
      ▼                                     ▼
data_provider/ 数据抓取                SearchService 新闻聚合
  优先级 0: efinance（东方财富）          ├─ Bocha（中文优化，AI 摘要）
  优先级 1: akshare                      ├─ MiniMax
  优先级 2: tushare / pytdx（通达信）    ├─ Tavily
  优先级 3: baostock                     ├─ SerpAPI
  优先级 4: yfinance（美股为主）          ├─ Brave Search
                                          └─ SearXNG（自建/公开实例）
      │                                     │
      └──────────────┬──────────────────────┘
                     ▼
        StockAnalysisPipeline.run()
          ├─ [每只股票，线程并发]
          │     ├─ 组装 stock_data 字典
          │     ├─ GeminiAnalyzer.analyze()（via LiteLLM）
          │     │     └─ LLM → AnalysisResult
          │     │           ├─ sentiment_score（浮点情绪分）
          │     │           ├─ operation_advice（买入/持有/卖出）
          │     │           ├─ trend_prediction（趋势判断）
          │     │           └─ 完整 Markdown 报告
          │     └─ repositories/ → SQLite（data/stock_analysis.db）
          │
          └─ [汇总] NotificationService.send()
                ├─ 企业微信 Webhook
                ├─ 飞书 Webhook / App
                ├─ Telegram Bot
                ├─ 邮件（SMTP）
                ├─ 钉钉 Webhook / Stream
                ├─ Discord Webhook / Bot
                ├─ Slack Bot / Webhook
                ├─ Pushover
                ├─ PushPlus
                ├─ Server酱3
                └─ 自定义 Webhook
```

---

## 6. 关键模块说明

### 6.1 `src/core/pipeline.py` — `StockAnalysisPipeline`

顶层编排器，在 `run_full_analysis()` 中实例化，持有：
- `self.notifier` — `NotificationService`
- `self.analyzer` — `GeminiAnalyzer`
- `self.search_service` — `SearchService`

使用 `concurrent.futures.ThreadPoolExecutor`（上限 `MAX_WORKERS`，默认 3）并发处理股票列表，控制外部 API 请求速率。

### 6.2 `src/analyzer.py` — `GeminiAnalyzer`

LLM 抽象层，尽管命名含 Gemini，实际支持所有配置的提供商（底层走 LiteLLM）：
- `is_available()` — 检查是否有可用的 API Key / Channel
- `analyze(stock_data, news, ...)` → `AnalysisResult`
- 支持流式输出、多 Key 负载均衡、跨渠道自动 fallback
- 支持 YAML 格式的 LiteLLM Router 配置

### 6.3 `src/notification.py` — `NotificationService`

多渠道通知分发器：
- `send()` 同时向所有已配置渠道并发推送
- `generate_aggregate_report(results, report_type)` → Markdown 汇总报告
- 委托 `src/notification_sender/` 实现各渠道具体发送逻辑
- 支持 Markdown → 图片转换（对不渲染 Markdown 的渠道）

### 6.4 `src/search_service.py` — `SearchService`

多搜索引擎新闻聚合器，按优先级/轮转策略选择引擎，应用 `NEWS_MAX_AGE_DAYS` 时间过滤和 `NEWS_STRATEGY_PROFILE` 窗口预设。

### 6.5 `src/core/trading_calendar.py`

- `get_open_markets_today()` → 当日开市的市场集合（`'cn'`, `'us'` 等）
- `get_market_for_stock(code)` → 识别股票所属市场
- 确保非交易日的股票自动跳过，除非使用 `--force-run`

### 6.6 `src/core/config_manager.py` — `ConfigManager`

运行时读写 `.env` 文件：
- Web UI 设置页面通过此方式修改配置（如 `SCHEDULE_TIME`、`STOCK_LIST`）
- `_build_schedule_time_provider()` 每次调度循环前重新读取 `.env`
- 实现了无需重启进程即可生效的热重载能力

### 6.7 `src/scheduler.py` — `run_with_schedule()`

阻塞式调度循环：
1. 可选：启动时立即执行一次
2. 使用 `schedule_time_provider` callable 每轮新鲜读取目标时间（支持 WebUI 动态修改）
3. 以 1 秒步长休眠等待目标时间
4. 支持 `background_tasks` — 额外周期任务（如 `agent_event_monitor`）与日常分析并行运行

### 6.8 `data_provider/` — 数据提供者

| 提供者 | 优先级 | 主要覆盖 | 需要配置 |
|---|---|---|---|
| efinance（东方财富）| 0 | A 股，完整 OHLCV | 无 |
| AkShare | 1 | A 股、港股、美股 | 无 |
| Tushare Pro | 2 | A 股，专业数据 | `TUSHARE_TOKEN` |
| TongDaXin（pytdx）| 2 | A 股 | 无 |
| BaoStock | 3 | A 股 | 无 |
| Yahoo Finance（yfinance）| 4 | 美股/全球 | 无 |

单一数据源失败会自动 fallback 到下一优先级，不影响整体流程。

### 6.9 `src/services/` — 服务层

| 服务 | 职责 |
|---|---|
| `analysis_service.py` | API/Bot 触发分析、异步任务管理 |
| `backtest_service.py` | 回测运行、结果持久化 |
| `portfolio_service.py` | 投资组合 CRUD |
| `portfolio_risk_service.py` | 风险集中度 / 回撤预警 |
| `portfolio_import_service.py` | CSV / 图片导入持仓 |
| `image_stock_extractor.py` | 视觉 LLM 从图片提取股票代码 |
| `social_sentiment_service.py` | 美股社交情绪（Reddit/X，仅 US 市场） |
| `report_renderer.py` | Jinja2 模板报告渲染 |
| `task_queue.py` + `task_service.py` | 异步任务队列 & 生命周期 |

### 6.10 `src/agent/` — Agent 系统

当 `AGENT_MODE=true` 时激活，支持：
- 单 Agent（`AGENT_ARCH=single`）
- 多 Agent 编排（`AGENT_ARCH=multi`）
- 可配置 Skill 路由（`AGENT_SKILL_ROUTING`）
- Agent 记忆（`AGENT_MEMORY_ENABLED`）
- 事件监控子调度器（`AGENT_EVENT_MONITOR_ENABLED`，每 N 分钟触发）

策略 Skill 文件定义在 `strategies/` 目录（YAML 格式）。

---

## 7. 外部集成

### 7.1 LLM 提供商（通过 LiteLLM 统一接入）

| 提供商 | 配置变量 | 说明 |
|---|---|---|
| Google Gemini | `GEMINI_API_KEY` / `GEMINI_API_KEYS` | 有免费额度，多用户默认选择 |
| DeepSeek | `DEEPSEEK_API_KEY` | 性价比高 |
| AIHubmix | `AIHUBMIX_KEY` | 聚合商：GPT/Claude/Gemini/GLM/Qwen |
| Anthropic Claude | `ANTHROPIC_API_KEY` | |
| OpenAI / 兼容接口 | `OPENAI_API_KEY` + `OPENAI_BASE_URL` | 任何 OpenAI 兼容端点 |
| Ollama（本地） | `OLLAMA_API_BASE` | 无需 API Key |
| 自定义 YAML | `LITELLM_CONFIG` | 完整 LiteLLM Router 配置 |
| 多渠道 | `LLM_CHANNELS=deepseek,gemini` | Key 轮转 + 自动 fallback |

### 7.2 通知渠道（10+ 渠道同时推送）

| 渠道 | 配置变量 |
|---|---|
| 企业微信 | `WECHAT_WEBHOOK_URL` |
| 飞书（Lark）| `FEISHU_WEBHOOK_URL` / App ID+Secret（Stream 模式）|
| Telegram | `TELEGRAM_BOT_TOKEN` + `TELEGRAM_CHAT_ID` |
| 邮件（SMTP）| `EMAIL_SENDER` + `EMAIL_PASSWORD` + `EMAIL_RECEIVERS` |
| 钉钉 | `DINGTALK_APP_KEY/SECRET` / `CUSTOM_WEBHOOK_URLS` |
| Discord | `DISCORD_WEBHOOK_URL` 或 Bot Token + Channel ID |
| Slack | `SLACK_BOT_TOKEN` + `SLACK_CHANNEL_ID` 或 Webhook |
| Pushover | `PUSHOVER_USER_KEY` + `PUSHOVER_API_TOKEN` |
| PushPlus | `PUSHPLUS_TOKEN` + `PUSHPLUS_TOPIC` |
| Server酱3 | `SERVERCHAN3_SENDKEY` |
| 自定义 Webhook | `CUSTOM_WEBHOOK_URLS`（自动识别钉钉/Discord/Slack/Bark/通用）|

### 7.3 新闻 / 搜索引擎

| 引擎 | 配置变量 | 特点 |
|---|---|---|
| Bocha | `BOCHA_API_KEYS` | 中文优化，AI 摘要 |
| MiniMax | `MINIMAX_API_KEYS` | Web Search |
| Tavily | `TAVILY_API_KEYS` | 通用，支持多 Key |
| SerpAPI | `SERPAPI_API_KEYS` | Google 结果 |
| Brave Search | `BRAVE_API_KEYS` | 独立索引 |
| SearXNG | `SEARXNG_BASE_URLS` | 自建或公开实例 |

### 7.4 IM 机器人（Stream 长连接）

`start_bot_stream_clients()` 在后台线程中启动（main.py line 539）：

- **钉钉 Stream**：`DINGTALK_STREAM_ENABLED=true`，需要 `dingtalk-stream` SDK
- **飞书 Stream**：`FEISHU_STREAM_ENABLED=true`，需要 `lark-oapi` SDK

消息链路：`bot/handler.py` → `bot/dispatcher.py` → `bot/commands/` → `analysis_service.trigger_analysis()`

### 7.5 飞书文档集成

- `src/feishu_doc.py` — `FeishuDocManager`
- 每次分析后自动创建带日期的飞书云文档，包含市场复盘 + 个股仪表盘
- 需要：`FEISHU_APP_ID` + `FEISHU_APP_SECRET`

### 7.6 社交情绪（仅美股）

- 数据来源：`api.adanos.org`（Reddit、X/Twitter、Polymarket）
- 配置：`SOCIAL_SENTIMENT_API_KEY`
- 仅对美股代码激活，A 股 / 港股忽略
- 免费额度：250 次/月

---

## 8. 配置系统

### 8.1 配置文件

- `.env` — 运行时主配置文件（从 `.env.example` 复制后填写）
- `litellm_config.yaml` — LiteLLM 高级路由配置（可选）
- `sources/` — 新闻来源配置（可选）
- `strategies/` — Agent Skill YAML（可选）

### 8.2 主要配置分组

| 组 | 关键变量 |
|---|---|
| 股票列表 | `STOCK_LIST`（逗号分隔的股票代码）|
| 数据源 | `TUSHARE_TOKEN`、`TICKFLOW_API_KEY` |
| LLM | `GEMINI_API_KEY`、`DEEPSEEK_API_KEY`、`LLM_CHANNELS`、`LLM_TEMPERATURE` |
| 搜索 | `BOCHA_API_KEYS`、`TAVILY_API_KEYS`、`SEARXNG_BASE_URLS` |
| 通知 | 各渠道 Token / Webhook（见上）|
| 调度 | `SCHEDULE_ENABLED`、`SCHEDULE_TIME`（HH:MM 24h）|
| 市场复盘 | `MARKET_REVIEW_ENABLED`、`MARKET_REVIEW_REGION`（cn/us/both）|
| 报告 | `REPORT_TYPE`（simple/full/brief）、`REPORT_LANGUAGE`（zh/en）|
| 回测 | `BACKTEST_ENABLED`、`BACKTEST_EVAL_WINDOW_DAYS` |
| Agent | `AGENT_MODE`、`AGENT_ARCH`、`AGENT_SKILLS`、`AGENT_MEMORY_ENABLED` |
| 系统 | `MAX_WORKERS`、`DATABASE_PATH`、`LOG_DIR`、`USE_PROXY` |
| Web UI | `WEBUI_ENABLED`、`WEBUI_PORT`、`ADMIN_AUTH_ENABLED` |

### 8.3 热重载机制

通过 Web UI 设置页面修改配置后：
- `ConfigManager` 直接写入 `.env` 文件
- 调度器每轮循环前通过 `schedule_time_provider` 重读 `.env`
- `_reload_runtime_config()` 在每次计划执行前重新实例化 `Config` 单例
- **无需重启进程即可生效**

---

## 9. Web / 桌面端

### 9.1 `apps/dsa-web/` — Vue 3 SPA

- 技术栈：Vue 3 + Vite + Tailwind CSS
- 构建产物输出至 `static/`，由 FastAPI 作为静态文件对外提供
- 包含单元测试（Vitest）和 E2E 测试（Playwright）
- `WEBUI_AUTO_BUILD=true` 时，启动时自动执行 `npm run build`
- `src/webui_frontend.py` 的 `prepare_webui_frontend_assets()` 管理构建流程

**构建：**
```bash
cd apps/dsa-web
npm ci
npm run lint
npm run build
```

### 9.2 `apps/dsa-desktop/` — Electron 桌面端

- 主进程：`main.js`
- 预加载脚本：`preload.js`（contextBridge 安全通信）
- 本质是包裹 Web UI 的 Electron 容器
- 桌面端 Release 由 `.github/workflows/desktop-release.yml` 自动构建
- 支持 Windows、macOS、Linux 平台原生二进制

---

## 10. 调度机制

系统提供三种互补的调度方式：

### 方式一：内置 Python 调度器（`--schedule` flag）

**代码：** `src/scheduler.py`，由 `main.py` lines 771–821 调用

```
启动
  ├─ [可选] 立即执行一次（SCHEDULE_RUN_IMMEDIATELY=true）
  └─ 循环：
        1. 从 .env 实时读取 SCHEDULE_TIME
        2. 以 1 秒步长休眠等待目标时间
        3. _reload_runtime_config() — 重新加载最新配置
        4. run_full_analysis() — 执行分析
        └─ 重复
```

相关配置：
- `SCHEDULE_TIME=18:00` — 每日触发时间（24 小时制）
- `SCHEDULE_RUN_IMMEDIATELY=true` — 启动时先执行一次再等待

### 方式二：GitHub Actions Cron（`daily_analysis.yml`）

- 使用工作流 `schedule: cron:` 触发
- 无需常驻 Python 进程
- 适合云托管 / Serverless 场景
- 密钥通过 GitHub Secrets 注入

### 方式三：WebUI 驱动的动态配置

Web 设置页面写入 `.env` → 调度器下一个循环自动读取新配置（无需重启）

### Agent 事件监控子调度器

当 `AGENT_EVENT_MONITOR_ENABLED=true` 时，额外启动一个后台任务（`AGENT_EVENT_MONITOR_INTERVAL_MINUTES`，默认 5 分钟），与日常任务并行运行，独立于每日分析周期。

---

## 11. CI/CD 流水线

| 工作流文件 | 触发条件 | 用途 | 是否阻断 |
|---|---|---|---|
| `ci.yml` | Push / PR | 主 CI：ai-governance + backend-gate + docker-build + web-gate | 是 |
| `daily_analysis.yml` | Cron（每日）| 在 GitHub Actions 上执行定时股票分析 | — |
| `docker-publish.yml` | Release / Tag | 构建 Docker 镜像并推送到 Docker Hub | — |
| `ghcr-dockerhub.yml` | Release / Tag | 同步推送到 GHCR + Docker Hub（双注册表）| — |
| `auto-tag.yml` | Push to main | 自动创建语义化版本 Tag（需 commit 含 `#patch`/`#minor`/`#major`）| — |
| `create-release.yml` | Tag 推送 | 创建 GitHub Release（含 CHANGELOG）| — |
| `desktop-release.yml` | Tag 推送 | 跨平台构建 Electron 桌面端二进制 | — |
| `pr-review.yml` | PR | AI 辅助 PR 审查（辅助项，非阻断）| 否 |
| `network-smoke.yml` | 定时 | 外部 API 连通性 Smoke 测试（观测项）| 否 |

**主 CI 四个门禁（`ci.yml`）：**
1. `ai-governance` — 校验 `AGENTS.md` / `CLAUDE.md` / `.github` 指令 / `.claude/skills` 关系
2. `backend-gate` — 执行 `./scripts/ci_gate.sh`（flake8 + pytest）
3. `docker-build` — Docker 构建 + 关键模块导入 Smoke
4. `web-gate` — 前端改动时执行 `npm run lint + build`

---

## 12. 架构分层总览

```
┌─────────────────────────────────────────────────────────────────┐
│  入口层                                                          │
│  main.py（CLI 全模式）· server.py（uvicorn 独立启动）            │
├─────────────────────────────────────────────────────────────────┤
│  接入层                                                          │
│  api/app.py + api/v1/（FastAPI REST API）                       │
│  bot/（钉钉 Stream · 飞书 Stream 长连接机器人）                  │
│  apps/dsa-web/（Vue 3 SPA）· apps/dsa-desktop/（Electron）      │
├─────────────────────────────────────────────────────────────────┤
│  编排层                                                          │
│  src/core/pipeline.py（StockAnalysisPipeline，ThreadPool）      │
│  src/core/market_review.py（市场复盘）                          │
│  src/scheduler.py（每日调度循环）                                │
│  src/core/trading_calendar.py（交易日过滤）                     │
├─────────────────────────────────────────────────────────────────┤
│  服务层                                                          │
│  src/services/（analysis / backtest / portfolio / task / ...）  │
│  src/core/config_manager.py（运行时配置热重载）                  │
├─────────────────────────────────────────────────────────────────┤
│  智能层                                                          │
│  src/analyzer.py（GeminiAnalyzer via LiteLLM）                  │
│  src/search_service.py（多引擎新闻聚合）                        │
│  src/agent/（多 Agent / Strategy Skill 系统）                   │
├─────────────────────────────────────────────────────────────────┤
│  数据层                                                          │
│  data_provider/（6 个抓取器，优先级自动 fallback）               │
│  src/repositories/（SQLite DAO 数据访问对象）                   │
│  src/storage.py（SQLite 持久化助手）                            │
├─────────────────────────────────────────────────────────────────┤
│  通知层                                                          │
│  src/notification.py（NotificationService 多渠道并发推送）       │
│  src/notification_sender/（各渠道具体实现）                     │
│  src/md2img.py（Markdown → 图片，适配不支持 MD 的渠道）          │
├─────────────────────────────────────────────────────────────────┤
│  配置层                                                          │
│  src/config.py（Config 单例 + setup_env）                       │
│  src/core/config_manager.py（运行时 .env 读写）                 │
│  .env（运行时可编辑，每轮调度热重载）                            │
└─────────────────────────────────────────────────────────────────┘
```

---

*生成时间：2026-03-30*  
*基于代码库当前状态分析，如发现与实际代码不符，以代码为准并更新本文档。*
