# BroZoomOut 升级计划 — 专业金融分析仪表盘

> Version: 2.0 | Date: 2026-05-28
> 目标：将 BroZoomOut 从信息聚合页面升级为专业金融分析仪表盘，对标同花顺/东方财富 70-80% 功能覆盖度。
> 与 ZoomLab v5.0 对齐，支持 6 大 Tab 页、30+ 页面、Dashboard 9 大模块。

---

## 目录

1. [现状分析](#1-现状分析)
2. [升级总览](#2-升级总览)
3. [P1 核心面板](#3-p1-核心面板)
4. [P2 AI 智能](#4-p2-ai-智能)
5. [P3 交互体验](#5-p3-交互体验)
6. [P4 深度数据](#6-p4-深度数据)
7. [Dashboard 9 大模块详细设计](#7-dashboard-9-大模块详细设计)
8. [A股/港股模块扩展](#8-a股港股模块扩展)
9. [美股模块扩展](#9-美股模块扩展)
10. [韩国股票模块扩展](#10-韩国股票模块扩展)
11. [基金模块扩展](#11-基金模块扩展)
12. [AI 分析模块扩展](#12-ai-分析模块扩展)
13. [部署架构](#13-部署架构)
14. [验收标准总表](#14-验收标准总表)

---

## 1. 现状分析

### 1.1 当前架构

```
finance-intelligence-service (Python/FastAPI)
├── app/api/v1/          # API 端点
│   ├── bro_zoomout.py   # 核心仪表盘 API (12 个端点)
│   ├── market.py        # 市场数据 API (3 个端点)
│   ├── experience.py    # 认知体验 API (6 个端点)
│   ├── reports.py       # 报告 API (9 个端点)
│   ├── jobs.py          # 调度触发 API (2 个端点)
│   └── health.py        # 健康检查 (1 个端点)
├── app/data/providers/  # 数据源
│   ├── tencent_provider.py   # A股指数 (腾讯 API)
│   ├── sina_provider.py      # 港股 (新浪 API)
│   ├── yfinance_provider.py  # 全球指数+宏观资产
│   ├── akshare_provider.py   # A股指数+板块 (备用)
│   ├── rss_provider.py       # 新闻聚合
│   └── news_provider.py      # 新闻入口
├── app/services/        # 业务逻辑
│   ├── bro_zoomout/     # 核心服务
│   │   ├── snapshot_builder.py   # 快照管理
│   │   ├── summary_builder.py    # 摘要构建
│   │   ├── market_analyzer.py    # 市场分析
│   │   ├── strategy_generator.py # 策略生成
│   │   ├── watchlist_engine.py   # 自选股引擎
│   │   ├── risk_explainer.py     # 风险解释
│   │   ├── regime_evaluator.py   # 市场状态评估
│   │   └── constants.py          # 常量定义
│   ├── experience/      # 认知体验服务
│   ├── analysis_service.py       # 数据分析
│   ├── market_service.py         # 市场数据服务
│   ├── report_service.py         # 报告服务
│   └── ... (20+ 服务)
├── app/jobs/            # 调度任务
│   ├── daily_market_job.py       # 每日任务 (15:30)
│   └── half_day_market_job.py    # 半日任务 (11:30)
├── app/core/            # 核心模块
│   ├── scheduler.py      # APScheduler 调度器
│   ├── llm_client.py     # LLM 客户端
│   ├── cache.py          # 缓存
│   ├── rate_limiter.py   # 限流
│   └── circuit_breaker.py # 熔断器
└── app/models/          # 数据模型
    ├── schemas.py              # Pydantic 模型
    ├── bro_zoomout_schemas.py  # BroZoomOut 模型
    └── db_models.py            # SQLAlchemy 模型
```

### 1.2 当前数据流

```
外部数据源 → Provider → Service → Job/API → Database/Cache → Response
   ↓
腾讯 qt.gtimg.cn ──→ tencent_provider ──→ fetch_a_stock_indices()
新浪 hq.sinajs.cn ──→ sina_provider ──→ fetch_hkstech()
yfinance ──→ yfinance_provider ──→ fetch_global_indices() + fetch_macro_assets()
RSS ──→ rss_provider ──→ fetch_rss_news()
```

### 1.3 当前 API 端点清单

| 端点 | 方法 | 功能 | 状态 |
|------|------|------|------|
| `/api/v1/brozoomout/summary` | GET | 总览摘要 | ✅ 已实现 |
| `/api/v1/brozoomout/dashboard` | GET | 仪表盘 | ✅ 已实现 |
| `/api/v1/brozoomout/markets/{market}` | GET | 分市场分析 | ✅ 已实现 |
| `/api/v1/brozoomout/watchlist` | GET/POST | 自选股管理 | ✅ 已实现 |
| `/api/v1/brozoomout/watchlist/{id}` | DELETE | 删除自选股 | ✅ 已实现 |
| `/api/v1/brozoomout/strategy` | GET | 策略建议 | ✅ 已实现 |
| `/api/v1/brozoomout/strategy/preview` | POST | 策略预览 | ✅ 已实现 |
| `/api/v1/brozoomout/snapshots/refresh` | POST | 刷新快照 | ✅ 已实现 |
| `/api/v1/brozoomout/snapshots/{id}` | GET | 查看快照 | ✅ 已实现 |
| `/api/v1/market/overview` | GET | 市场总览 | ✅ 已实现 |
| `/api/v1/market/daily-report` | GET | 每日报告 | ✅ 已实现 |
| `/api/v1/market/half-day-report` | GET | 半日报告 | ✅ 已实现 |
| `/api/v1/experience/timeline` | GET | 认知时间线 | ✅ 已实现 |
| `/api/v1/experience/replay-summary` | GET | 回放摘要 | ✅ 已实现 |
| `/api/v1/experience/morning-brief` | GET | 晨间简报 | ✅ 已实现 |
| `/api/v1/experience/feed` | GET | 信息流 | ✅ 已实现 |
| `/api/v1/experience/diff` | GET | 认知差异 | ✅ 已实现 |
| `/api/v1/experience/narrative-timeline` | GET | 叙事时间线 | ✅ 已实现 |
| `/api/v1/reports/latest` | GET | 最新报告 | ✅ 已实现 |
| `/api/v1/reports/history` | GET | 报告历史 | ✅ 已实现 |
| `/api/v1/reports/context` | GET | 报告上下文 | ✅ 已实现 |
| `/api/v1/reports/cognitive-context` | GET | 认知上下文 | ✅ 已实现 |
| `/api/v1/reports/regime-history` | GET | 市场状态历史 | ✅ 已实现 |
| `/api/v1/reports/timeline` | GET | 报告时间线 | ✅ 已实现 |
| `/api/v1/reports/narratives` | GET | 叙事上下文 | ✅ 已实现 |
| `/api/v1/reports/claims` | GET | 主张列表 | ✅ 已实现 |
| `/api/v1/reports/replay` | GET | 认知回放 | ✅ 已实现 |
| `/api/v1/jobs/daily-market` | POST | 触发日报 | ✅ 已实现 |
| `/api/v1/jobs/half-day-market` | POST | 触发半日报 | ✅ 已实现 |

### 1.4 功能差距分析

| 功能域 | 同花顺/东方财富 | BroZoomOut 当前 | 覆盖度 |
|--------|---------------|----------------|--------|
| 实时行情 | ✅ 全市场 | ✅ 指数+港股 | 40% |
| K 线图表 | ✅ 日/周/月+指标 | ❌ 无 | 0% |
| 板块热力图 | ✅ 163 概念板块 | ❌ 无 | 0% |
| 资金流向 | ✅ 主力/北向/散户 | ❌ 无 | 0% |
| 个股详情 | ✅ 行情+财务+估值 | ❌ 无 | 0% |
| 研报 | ✅ 摘要+评级 | ⚠️ 有基础 | 30% |
| 新闻 | ✅ AI 分级 | ⚠️ 有 RSS | 40% |
| AI 分析 | ✅ 多维度 | ✅ 认知系统 | 60% |
| 策略建议 | ✅ 个性化 | ✅ 已实现 | 80% |
| 自选股 | ✅ 全功能 | ✅ 已实现 | 70% |
| 宏观数据 | ✅ CPI/PMI/GDP | ❌ 无 | 0% |
| 经济日历 | ✅ 完整 | ❌ 无 | 0% |

**当前覆盖度**: ~30% | **目标覆盖度**: 75%

---

## 2. 升级总览

### 2.1 升级路线图

```
P1 核心面板 (2 周)     → 覆盖度 30% → 50%
    ↓
P2 AI 智能 (2 周)      → 覆盖度 50% → 65%
    ↓
P3 交互体验 (1 周)     → 覆盖度 65% → 70%
    ↓
P4 深度数据 (3 周)     → 覆盖度 70% → 75%
```

### 2.2 技术栈

| 层级 | 技术 | 说明 |
|------|------|------|
| API 框架 | FastAPI 0.115+ | 已有 |
| 数据库 | SQLAlchemy + SQLite | 已有，可升级 PostgreSQL |
| 缓存 | 内存缓存 → Redis | P1 阶段升级 |
| 调度 | APScheduler | 已有 |
| K 线计算 | pandas + ta-lib | P4 新增 |
| 数据源 | 腾讯/新浪/yfinance/东财 | 已有 + 新增 |
| LLM | OpenAI API | 已有 |

---

## 3. P1 核心面板

### 3.1 板块热力图

**功能描述**: 展示 163 个概念板块的涨跌幅热力图，颜色编码涨跌（红涨绿跌），支持点击查看详情。

#### API 端点设计

```
GET /api/v1/sectors/heatmap
GET /api/v1/sectors/concepts
GET /api/v1/sectors/{sector_code}
```

#### 数据源

- **主数据源**: 新浪 `newFLJK` — 163 个概念板块
- **备用数据源**: AKShare `stock_board_industry_name_em()` — 行业板块
- **URL 模式**: `http://vip.stock.finance.sina.com.cn/quotes_service/api/json_v2.php/Market_Center.getHQNodeStockCount?node=new_fljk`

#### 数据流图

```
新浪 newFLJK API
    ↓
app/data/providers/sina_sector_provider.py (新增)
    ↓
app/services/sector_service.py (新增)
    ↓
app/api/v1/sectors.py (新增)
    ↓
内存缓存 (TTL: 5 分钟)
    ↓
JSON Response
```

#### 后端服务设计

**文件**: `app/data/providers/sina_sector_provider.py` (新增)

```python
# 核心逻辑:
# 1. 请求新浪 newFLJK 获取板块列表
# 2. 解析 JSON 响应，提取板块代码、名称、涨跌幅
# 3. 对每个板块，获取 TOP 5 成分股
# 4. 构建热力图数据结构
# 5. 缓存 5 分钟
```

**文件**: `app/services/sector_service.py` (新增)

```python
# 核心逻辑:
# 1. 调用 sina_sector_provider 获取原始数据
# 2. 计算颜色编码 (红涨绿跌，渐变色阶)
# 3. 排序：按涨跌幅绝对值排序（最活跃在前）
# 4. 计算板块统计：上涨/下跌/平盘数量
# 5. 返回热力图数据结构
```

#### 调度任务设计

**文件**: `app/jobs/sector_scan_job.py` (新增)

```python
# 调度: 每 30 分钟执行一次 (交易时段)
# 逻辑:
# 1. 调用 sina_sector_provider 获取全量板块数据
# 2. 存入 SectorSnapshot 表
# 3. 更新板块排名
# 4. 检测异动板块 (涨跌幅 > 3%)
```

#### 验收标准

- [ ] 163 个概念板块全部展示
- [ ] 颜色编码：涨红跌绿，渐变色阶
- [ ] 点击板块可查看详情（成分股、新闻、资金流向）
- [ ] 每 30 分钟自动更新
- [ ] 响应时间 < 500ms（缓存命中）
- [ ] 降级策略：新浪不可用时使用 AKShare

---

### 3.2 新闻分级

**功能描述**: AI 自动对新闻进行分级（major/notable/general），标注利好利空，提供一句话总结。

#### API 端点设计

```
POST /api/v1/news/classify
GET  /api/v1/news/feed?importance=major
```

#### 数据源

- **新闻源**: 已有 RSS (Google Finance / CNBC / FT)
- **AI 分级**: LLM (OpenAI API)

#### 数据流图

```
RSS Feed (已有)
    ↓
rss_provider.py (已有)
    ↓
news_classifier.py (新增) ← LLM API
    ↓
news_service.py (增强)
    ↓
app/api/v1/news.py (新增)
    ↓
内存缓存 (TTL: 10 分钟)
```

#### 后端服务设计

**文件**: `app/services/news_classifier.py` (新增)

```python
# 核心逻辑:
# 1. 接收新闻列表 (title, summary, source)
# 2. 构建 prompt: 分级 + 情感分析 + 一句话总结
# 3. 调用 LLM API (使用 prompt_templates.py)
# 4. 解析返回结果
# 5. 缓存分类结果

# Prompt 模板:
CLASSIFY_PROMPT = """
你是一位金融新闻分析师。请对以下新闻进行分级和分析。

新闻标题: {title}
新闻摘要: {summary}
来源: {source}

请返回 JSON 格式:
{
  "importance": "major|notable|general",
  "sentiment": "bullish|neutral|bearish",
  "impact_score": 0-100,
  "tags": ["标签1", "标签2"],
  "summary": "一句话总结"
}
"""
```

#### 调度任务设计

**文件**: `app/jobs/news_aggregation_job.py` (增强)

```python
# 现有: rss_provider 获取新闻
# 新增:
# 1. 调用 news_classifier 对新闻进行 AI 分级
# 2. 优先推送 major 级别新闻
# 3. 缓存分类结果
# 4. 每 15 分钟执行一次
```

#### 验收标准

- [ ] 所有新闻自动分级为 major/notable/general
- [ ] 利好/利空标注准确率 > 80%
- [ ] 一句话总结清晰准确
- [ ] major 级别新闻置顶显示
- [ ] 每 15 分钟自动更新
- [ ] LLM 调用失败时降级为未分级

---

### 3.3 个股推荐

**功能描述**: 基于量化筛选 + AI 评估，推荐值得关注的个股。

#### API 端点设计

```
GET /api/v1/ai/stock-picks
```

#### 参数

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `strategy` | string | `value` | `value`/`growth`/`momentum`/`dividend` |
| `market` | string | `all` | `all`/`cn`/`hk`/`us` |
| `limit` | int | `10` | 推荐数量 |

#### 数据流图

```
yfinance (实时行情) + AKShare (财务数据)
    ↓
stock_screener.py (新增) ← 量化筛选
    ↓
stock_analyzer.py (新增) ← AI 评估
    ↓
app/api/v1/ai.py (增强)
```

#### 后端服务设计

**文件**: `app/services/stock_screener.py` (新增)

```python
# 量化筛选逻辑:
# 1. 基础筛选: 市值 > 50亿, 日均成交额 > 1亿
# 2. 价值策略: PE < 20, PB < 3, ROE > 15%
# 3. 成长策略: 营收增速 > 20%, 净利增速 > 25%
# 4. 动量策略: 20日涨幅 > 5%, 成交量放大
# 5. 红利策略: 股息率 > 3%, 连续3年分红
```

**文件**: `app/services/stock_analyzer.py` (新增)

```python
# AI 评估逻辑:
# 1. 获取筛选结果 (约 20 只)
# 2. 构建 prompt: 基本面 + 技术面 + 行业分析
# 3. 调用 LLM API 进行综合评估
# 4. 返回推荐列表 (每只附 AI 分析)
```

#### 验收标准

- [ ] 支持 4 种策略：价值/成长/动量/红利
- [ ] 推荐列表附带 AI 分析说明
- [ ] 筛选逻辑透明可追溯
- [ ] 推荐结果每日更新
- [ ] 推荐理由清晰易懂

---

### 3.4 研报摘要

**功能描述**: AI 对研报进行一句话总结，标注评级和目标价。

#### API 端点设计

```
GET /api/v1/research/digest
GET /api/v1/research/reports/{report_id}
```

#### 数据源

- **研报**: 东财 `reportapi` (已有基础)
- **AI 摘要**: LLM

#### 后端服务设计

**文件**: `app/services/research_digest_service.py` (新增)

```python
# 核心逻辑:
# 1. 获取东财研报列表
# 2. 对每篇研报调用 LLM 生成一句话摘要
# 3. 提取评级、目标价、核心观点
# 4. 缓存摘要结果 (TTL: 24 小时)
# 5. 按重要性排序返回
```

#### 验收标准

- [ ] 每篇研报有一句话 AI 摘要
- [ ] 评级和目标价清晰展示
- [ ] 每日自动更新
- [ ] 摘要长度 < 50 字

---

## 4. P2 AI 智能

### 4.1 AI 预测（三场景）

**功能描述**: 基于 LLM + 历史数据，提供牛市/基准/熊市三场景预测。

#### API 端点设计

```
POST /api/v1/ai/forecast
GET  /api/v1/ai/forecast/{symbol}
```

#### 数据流图

```
yfinance (历史K线) + 当前行情 + 宏观数据
    ↓
forecast_service.py (新增)
    ↓
LLM API (三场景 prompt)
    ↓
forecast_cache.py (Redis, TTL: 30min)
    ↓
JSON Response
```

#### 后端服务设计

**文件**: `app/services/forecast_service.py` (新增)

```python
# 核心逻辑:
# 1. 获取目标股票 90 天历史数据
# 2. 计算技术指标 (MA/MACD/RSI)
# 3. 获取宏观环境 (风险模式/波动率)
# 4. 构建三场景预测 prompt
# 5. 调用 LLM API
# 6. 解析并缓存结果

# Prompt 模板:
FORECAST_PROMPT = """
基于以下数据，对 {symbol} ({name}) 进行三场景预测:

历史数据: {history}
技术指标: {indicators}
当前价格: {current_price}
宏观环境: {macro_context}

请返回 JSON:
{
  "bullish": { "probability": 0-1, "target_price": float, "reasoning": "..." },
  "base": { "probability": 0-1, "target_price": float, "reasoning": "..." },
  "bearish": { "probability": 0-1, "target_price": float, "reasoning": "..." },
  "confidence": 0-1,
  "key_factors": ["..."]
}
"""
```

#### 验收标准

- [ ] 三场景概率之和 = 100%
- [ ] 目标价基于历史数据合理推断
- [ ] 每个场景有清晰的理由说明
- [ ] 预测结果缓存 30 分钟
- [ ] 置信度基于数据质量计算

---

### 4.2 AI 基金推荐

**功能描述**: 基于 yfinance 净值数据 + LLM 策略分析，推荐适合的基金。

#### API 端点设计

```
GET /api/v1/ai/fund-recommendations
```

#### 数据流图

```
yfinance (基金净值) + 市场风险模式
    ↓
fund_analyzer.py (新增)
    ↓
LLM API (策略分析)
    ↓
JSON Response
```

#### 后端服务设计

**文件**: `app/services/fund_analyzer.py` (新增)

```python
# 核心逻辑:
# 1. 获取基金净值数据 (yfinance)
# 2. 计算风险指标: 夏普比率、最大回撤、波动率
# 3. 检测风格漂移: 对比持仓风格 vs 基金名称
# 4. 根据当前风险模式匹配基金类型
# 5. LLM 生成策略分析
# 6. 返回推荐列表

# 风险指标计算:
# - 夏普比率 = (年化收益 - 无风险利率) / 年化波动率
# - 最大回撤 = max(peak - trough) / peak
# - 风格漂移 = 持仓风格与基金名称的差异度
```

#### 验收标准

- [ ] 夏普比率 > 1 的基金优先推荐
- [ ] 最大回撤 < 20% 的基金标记为低风险
- [ ] 风格漂移检测准确
- [ ] 推荐附带 AI 策略分析
- [ ] 根据用户风险偏好个性化推荐

---

### 4.3 Dashboard 增强

**功能描述**: 增强仪表盘展示置信度趋势和风险模式变化。

#### 新增组件

| 组件 | 功能 | 数据源 |
|------|------|--------|
| 置信度趋势 | 30 天置信度折线图 | 历史快照 |
| 风险模式变化 | 风险模式切换时间线 | 历史快照 |
| 波动率指标 | 波动率仪表盘 | 实时计算 |
| 市场情绪 | 情绪指标 (恐惧/贪婪) | 综合计算 |

#### 验收标准

- [ ] 置信度趋势图展示 30 天数据
- [ ] 风险模式变化时间线清晰
- [ ] 波动率仪表盘实时更新
- [ ] 市场情绪指标直观易懂

---

## 5. P3 交互体验

### 5.1 历史趋势数据扩充

**功能描述**: 扩充历史数据至 30 天快照，支持趋势分析。

#### 数据流图

```
每日/半日 Job
    ↓
快照存储 (内存 → SQLite)
    ↓
30 天数据聚合
    ↓
趋势分析 API
```

#### 后端服务设计

**文件**: `app/services/history_service.py` (新增)

```python
# 核心逻辑:
# 1. 每次 Job 执行后存储快照到 SQLite
# 2. 提供 30 天历史数据查询
# 3. 计算趋势指标: 均值、标准差、变化率
# 4. 识别模式: 持续上涨/下跌/震荡
```

#### 验收标准

- [ ] 存储 30 天历史快照
- [ ] 支持按日期范围查询
- [ ] 趋势分析准确
- [ ] 历史数据可回溯

---

### 5.2 调度任务扩展

**功能描述**: 新增板块扫描、新闻聚合、研报摘要等调度任务。

#### 新增调度任务

| 任务 | 调度 | 功能 |
|------|------|------|
| `sector_scan_job` | 每 30 分钟 (交易时段) | 板块数据更新 |
| `news_aggregation_job` | 每 15 分钟 | 新闻聚合+AI分级 |
| `research_digest_job` | 每日 9:00 | 研报摘要生成 |
| `stock_picks_job` | 每日 9:30 | 个股推荐更新 |
| `snapshot_archive_job` | 每日 16:00 | 历史快照归档 |

#### 验收标准

- [ ] 所有调度任务按计划执行
- [ ] 任务执行有日志记录
- [ ] 任务失败有重试机制
- [ ] 任务状态可通过 API 查询

---

### 5.3 交互增强

**功能描述**: 增强前端交互体验。

#### 新增功能

| 功能 | 说明 |
|------|------|
| 板块联动 | 点击热力图板块 → 显示成分股 |
| 数据下钻 | 点击个股 → 显示详情页 |
| 实时刷新 | WebSocket 推送行情更新 |
| 空状态优化 | 每个组件有友好的空状态 |
| 加载骨架屏 | 数据加载时显示骨架屏 |

#### 验收标准

- [ ] 板块联动交互流畅
- [ ] 数据下钻逻辑清晰
- [ ] 实时刷新无延迟
- [ ] 空状态有引导文案
- [ ] 骨架屏与真实内容布局一致

---

## 6. P4 深度数据

### 6.1 K 线图

**功能描述**: 支持日/周/月 K 线，技术指标叠加（MA/MACD/RSI）。

#### API 端点设计

```
GET /api/v1/market/kline/{symbol}
GET /api/v1/technical/indicators/{symbol}
```

#### 数据源

- **K 线数据**: yfinance `history()`
- **技术指标**: 纯 Python 计算

#### 后端服务设计

**文件**: `app/services/technical_service.py` (新增)

```python
# 技术指标计算 (纯 Python):
# - MA: 简单移动平均
# - EMA: 指数移动平均
# - MACD: DIF - DEA
# - RSI: 相对强弱指标
# - Bollinger: 布林带
# - KDJ: 随机指标

# 依赖: pandas (已有)
# 无需 ta-lib，纯 Python 实现
```

**文件**: `app/services/kline_service.py` (新增)

```python
# 核心逻辑:
# 1. 调用 yfinance 获取 K 线数据
# 2. 计算技术指标
# 3. 构建 K 线响应格式
# 4. 缓存 5 分钟
```

#### 验收标准

- [ ] 支持日/周/月 K 线
- [ ] MA5/10/20/60 均线叠加
- [ ] MACD 柱状图
- [ ] RSI 超买超卖区间
- [ ] 布林带上下轨
- [ ] KDJ 金叉死叉信号
- [ ] 响应时间 < 1s

---

### 6.2 技术指标计算

**功能描述**: 纯 Python 实现常用技术指标。

#### 支持的指标

| 指标 | 计算方法 | 用途 |
|------|---------|------|
| MA | `df['close'].rolling(window).mean()` | 趋势判断 |
| EMA | `df['close'].ewm(span).mean()` | 趋势判断 |
| MACD | DIF = EMA12 - EMA26, DEA = EMA9(DIF) | 买卖信号 |
| RSI | 100 - 100/(1+RS), RS = avg_gain/avg_loss | 超买超卖 |
| Bollinger | MA ± 2*STD | 波动区间 |
| KDJ | K = (C-L9)/(H9-L9)*100 | 超买超卖 |

#### 验收标准

- [ ] 所有指标计算结果与同花顺一致 (误差 < 0.01)
- [ ] 计算时间 < 100ms
- [ ] 支持自定义参数

---

### 6.3 资金流向分析

**功能描述**: 主力/散户/北向资金流向分析。

#### API 端点设计

```
GET /api/v1/money-flow/stock/{symbol}
GET /api/v1/money-flow/northbound
GET /api/v1/money-flow/main-force
```

#### 数据源

- **个股资金流向**: 东财 `push2his` (需缓存 30 分钟)
- **北向资金**: 东财 `kamt.rtmin` (需缓存 5 分钟)
- **主力资金**: 综合计算

#### 后端服务设计

**文件**: `app/data/providers/eastmoney_provider.py` (新增)

```python
# 核心逻辑:
# 1. 请求东财 push2his API
# 2. 解析资金流向数据
# 3. 缓存 30 分钟
# 4. 支持降级到缓存数据
```

**文件**: `app/services/money_flow_service.py` (新增)

```python
# 核心逻辑:
# 1. 调用 eastmoney_provider 获取数据
# 2. 计算主力净流入/流出
# 3. 分析北向资金趋势
# 4. 生成资金流向分析文本
```

#### 验收标准

- [ ] 个股资金流向数据准确
- [ ] 北向资金 241 数据点完整
- [ ] 主力/散户资金分类清晰
- [ ] 资金流向趋势分析准确

---

### 6.4 个股详情页

**功能描述**: 行情+财务+估值+研报的综合详情页。

#### API 端点设计

```
GET /api/v1/stock/detail/{symbol}
GET /api/v1/stock/financials/{symbol}
GET /api/v1/stock/valuation/{symbol}
```

#### 数据流图

```
yfinance (行情+财务) + 东财 (研报) + AI (分析)
    ↓
stock_detail_service.py (新增)
    ↓
JSON Response
```

#### 后端服务设计

**文件**: `app/services/stock_detail_service.py` (新增)

```python
# 核心逻辑:
# 1. 获取实时行情 (yfinance)
# 2. 获取财务数据: 营收、净利、ROE、PE、PB
# 3. 获取估值数据: PE/PB/PS 历史分位
# 4. 获取相关研报
# 5. AI 生成综合分析
# 6. 返回详情页数据
```

#### 验收标准

- [ ] 行情数据实时更新
- [ ] 财务数据最近 4 个季度
- [ ] 估值分位准确
- [ ] 研报列表按时间排序
- [ ] AI 分析全面客观

---

### 6.5 宏观数据

**功能描述**: CPI/PMI/GDP 等宏观经济指标。

#### API 端点设计

```
GET /api/v1/macro/indicators
GET /api/v1/macro/calendar
```

#### 数据源

- **宏观指标**: 东财宏观数据 API
- **经济日历**: Fair Economy

#### 后端服务设计

**文件**: `app/data/providers/macro_provider.py` (新增)

```python
# 核心逻辑:
# 1. 请求东财宏观数据 API
# 2. 解析 CPI/PMI/GDP/M2 等指标
# 3. 缓存 1 小时
# 4. 支持多国家: CN/US/EU/JP
```

**文件**: `app/services/macro_service.py` (新增)

```python
# 核心逻辑:
# 1. 调用 macro_provider 获取数据
# 2. 计算同比/环比变化
# 3. 判断趋势: 上升/下降/持平
# 4. 生成宏观环境分析
```

#### 验收标准

- [ ] CPI/PMI/GDP 数据准确
- [ ] 数据更新频率符合官方发布节奏
- [ ] 同比/环比计算正确
- [ ] 趋势判断准确

---

## 7. Dashboard 9 大模块详细设计

> 与 ZoomLab v5.0 Dashboard 对齐

### 7.1 MarketOverview 市场概览

**功能描述**: 展示全球 7 大指数（上证/深证/创业板/恒生/S&P/NASDAQ/KOSPI）+ 恐惧贪婪指数仪表盘。

#### API 调用

| 端点 | 用途 | 缓存 |
|------|------|------|
| `GET /api/v1/market/overview` | 全球指数概览 | 60s |
| `GET /api/v1/market/sentiment` | 市场情绪 | 300s |

#### 数据流

```
腾讯 qt.gtimg.cn ──→ tencent_provider ──→ A股指数
新浪 hq.sinajs.cn ──→ sina_provider ──→ 港股指数
yfinance ──→ yfinance_provider ──→ 美股/韩国指数
    ↓
market_service.py ──→ 聚合计算
    ↓
GET /api/v1/market/overview ──→ 响应
```

#### 前端组件

| 组件 | 功能 | 数据源 |
|------|------|--------|
| MarketOverviewCard | 卡片容器 | API |
| IndexCard × 7 | 单个指数卡片 | API |
| SentimentGauge | 恐惧贪婪仪表盘 | API |

#### 调度任务

| 任务 | 调度 | 功能 |
|------|------|------|
| `daily_market_job` | 每日 15:30 | 更新指数数据 |
| `half_day_market_job` | 每日 11:30 | 半日更新 |

#### 验收标准

- [ ] 7 个全球指数正确展示
- [ ] 恐惧贪婪指数仪表盘准确
- [ ] 60 秒自动刷新
- [ ] 响应时间 < 1s

---

### 7.2 PortfolioSummary 持仓概览

**功能描述**: 显示用户持仓总资产、日收益、行业分布饼图。

#### API 调用

| 端点 | 用途 | 缓存 |
|------|------|------|
| `GET /api/v1/user/positions` | 持仓列表 | 30s |

#### 数据流

```
用户持仓数据
    ↓
position_service.py ──→ 聚合计算总资产/日收益
    ↓
前端渲染 SectorPieChart
```

#### 前端组件

| 组件 | 功能 |
|------|------|
| PortfolioSummaryCard | 卡片容器 |
| AssetSummary | 总资产/日收益/收益率 |
| SectorPieChart | 行业分布饼图 |

#### 验收标准

- [ ] 总资产计算准确
- [ ] 日收益实时更新
- [ ] 行业分布饼图正确
- [ ] 响应时间 < 1s

---

### 7.3 NewsFeed 新闻流

**功能描述**: 近 24 小时重要信息，支持分类筛选（全部/宏观/行业/公司/异动），Top 3 重要卡片 + 时间线列表。

#### API 调用

| 端点 | 用途 | 缓存 |
|------|------|------|
| `GET /api/v1/dashboard/news` | 近24小时新闻 | 120s |

#### 数据流

```
RSS Feed (Google Finance / CNBC / FT)
    ↓
rss_provider.py ──→ 新闻聚合
    ↓
news_classifier.py ← LLM API ──→ AI 分级
    ↓
GET /api/v1/dashboard/news ──→ 响应
```

#### 前端组件

| 组件 | 功能 |
|------|------|
| NewsFeedCard | 卡片容器 |
| CategoryFilterTabs | 分类筛选 Tab |
| TopHighlightCards | Top 3 重要卡片 |
| TimelineList | 时间线列表 |

#### 调度任务

| 任务 | 调度 | 功能 |
|------|------|------|
| `news_aggregation_job` | 每 15 分钟 | 新闻聚合+AI分级 |

#### 验收标准

- [ ] 近 24 小时新闻正确展示
- [ ] 分类筛选功能正常
- [ ] Top 3 重要卡片正确高亮
- [ ] 时间线按时间分组
- [ ] 响应时间 < 1s

---

### 7.4 AIDailyPrediction AI 今日预测

**功能描述**: AI 预测摘要 + 置信度徽章 + 三场景概率分布条形图 + 关键影响因素列表。

#### API 调用

| 端点 | 用途 | 缓存 |
|------|------|------|
| `GET /api/v1/dashboard/ai-prediction` | AI 今日预测 | 600s |

#### 数据流

```
LLM API + 历史行情 + 宏观数据
    ↓
forecast_service.py ──→ 三场景预测
    ↓
GET /api/v1/dashboard/ai-prediction ──→ 响应
```

#### 前端组件

| 组件 | 功能 |
|------|------|
| AIDailyPredictionCard | 卡片容器 |
| PredictionSummary | 预测摘要 + 置信度 |
| ThreeScenarioChart | 三场景概率条形图 (Recharts BarChart) |
| KeyFactorsList | 关键因素列表 + 权重条形图 |

#### 调度任务

| 任务 | 调度 | 功能 |
|------|------|------|
| `ai_prediction_job` | 每日 9:00 | 更新今日预测 |

#### 验收标准

- [ ] 预测摘要清晰准确
- [ ] 置信度徽章正确显示
- [ ] 三场景概率之和 = 100%
- [ ] 关键因素权重可视化
- [ ] 响应时间 < 5s

---

### 7.5 AIRiskAlerts AI 风险提示

**功能描述**: 风险等级徽章 + 30 天趋势折线图 + 风险因素卡片列表。

#### API 调用

| 端点 | 用途 | 缓存 |
|------|------|------|
| `GET /api/v1/dashboard/ai-risks` | AI 风险提示 | 300s |

#### 数据流

```
LLM 风险评估 + 市场情绪指标 + 历史风险事件
    ↓
risk_service.py ──→ 风险评估
    ↓
GET /api/v1/dashboard/ai-risks ──→ 响应
```

#### 前端组件

| 组件 | 功能 |
|------|------|
| AIRiskAlertsCard | 卡片容器 |
| RiskLevelBadge | 风险等级徽章 |
| RiskTrendChart | 风险趋势折线图 (Recharts AreaChart) |
| RiskFactorCards | 风险因素卡片列表 |

#### 调度任务

| 任务 | 调度 | 功能 |
|------|------|------|
| `risk_assessment_job` | 每日 9:00 | 更新风险评估 |

#### 验收标准

- [ ] 风险等级徽章正确显示
- [ ] 30 天趋势图准确
- [ ] 风险因素卡片列表完整
- [ ] 响应时间 < 5s

---

### 7.6 AIOpportunitySignals AI 机会信号

**功能描述**: 机会等级徽章 + 类型筛选 Tab + 机会卡片列表 + 历史表现。

#### API 调用

| 端点 | 用途 | 缓存 |
|------|------|------|
| `GET /api/v1/dashboard/ai-opportunities` | AI 机会信号 | 300s |
| `GET /api/v1/ai/opportunity-performance` | 机会历史表现 | 3600s |

#### 数据流

```
LLM + 板块资金流向 + 技术指标
    ↓
opportunity_service.py ──→ 机会识别
    ↓
GET /api/v1/dashboard/ai-opportunities ──→ 响应
```

#### 前端组件

| 组件 | 功能 |
|------|------|
| AIOpportunitySignalsCard | 卡片容器 |
| OpportunityBadge | 机会等级徽章 |
| TypeFilterTabs | 类型筛选 Tab |
| OpportunityCards | 机会卡片列表 |

#### 调度任务

| 任务 | 调度 | 功能 |
|------|------|------|
| `opportunity_scan_job` | 每 30 分钟 (交易时段) | 扫描机会信号 |

#### 验收标准

- [ ] 机会等级徽章正确显示
- [ ] 类型筛选功能正常
- [ ] 机会卡片列表完整
- [ ] 历史表现数据准确
- [ ] 响应时间 < 5s

---

### 7.7 AIStockRecommendations AI 推荐股票

**功能描述**: 市场筛选 Tab [全部/A股/港股/美股/韩国] + 推荐股票卡片网格（按推荐强度排序）。

#### API 调用

| 端点 | 用途 | 缓存 |
|------|------|------|
| `GET /api/v1/dashboard/ai-recommendations` | AI 推荐股票 | 300s |

#### 数据流

```
LLM + 量化筛选 + 基本面/技术面分析
    ↓
recommendation_service.py ──→ 推荐生成
    ↓
GET /api/v1/dashboard/ai-recommendations ──→ 响应
```

#### 前端组件

| 组件 | 功能 |
|------|------|
| AIStockRecommendationsCard | 卡片容器 |
| MarketFilterTabs | 市场筛选 Tab |
| StockCardGrid | 推荐股票卡片网格 |

#### 调度任务

| 任务 | 调度 | 功能 |
|------|------|------|
| `stock_picks_job` | 每日 9:30 | 更新个股推荐 |

#### 验收标准

- [ ] 市场筛选功能正常
- [ ] 推荐股票按强度排序
- [ ] 每只股票附带 AI 分析
- [ ] 响应时间 < 5s

---

### 7.8 AITradingDecisions AI 买卖决策

**功能描述**: 操作筛选 Tab [全部/买入/卖出/持有] + 决策卡片列表 + 免责声明。

#### API 调用

| 端点 | 用途 | 缓存 |
|------|------|------|
| `GET /api/v1/dashboard/ai-decisions` | AI 买卖决策 | 300s |

#### 数据流

```
LLM + 技术指标 + 资金流向
    ↓
decision_service.py ──→ 决策生成
    ↓
GET /api/v1/dashboard/ai-decisions ──→ 响应
```

#### 前端组件

| 组件 | 功能 |
|------|------|
| AITradingDecisionsCard | 卡片容器 |
| ActionFilterTabs | 操作筛选 Tab |
| DecisionCardList | 决策卡片列表 |
| Disclaimer | 免责声明 |

#### 调度任务

| 任务 | 调度 | 功能 |
|------|------|------|
| `trading_decision_job` | 每日 9:30 | 更新买卖决策 |

#### 验收标准

- [ ] 操作筛选功能正常
- [ ] 决策卡片包含止损/止盈
- [ ] 免责声明清晰可见
- [ ] 响应时间 < 5s

---

### 7.9 Watchlist 自选股

**功能描述**: 自选股列表，支持添加/删除，显示实时行情。

#### API 调用

| 端点 | 用途 | 缓存 |
|------|------|------|
| `GET /api/v1/user/watchlist` | 自选股列表 | 30s |
| `POST /api/v1/user/watchlist` | 添加自选 | - |
| `DELETE /api/v1/user/watchlist/:id` | 删除自选 | - |

#### 数据流

```
用户操作 (添加/删除)
    ↓
watchlist_service.py ──→ 数据库操作
    ↓
实时行情 API ──→ 行情更新
    ↓
GET /api/v1/user/watchlist ──→ 响应
```

#### 前端组件

| 组件 | 功能 |
|------|------|
| WatchlistCard | 卡片容器 |
| WatchlistItem × N | 自选股条目 |

#### 验收标准

- [ ] 自选股列表正确展示
- [ ] 添加/删除功能正常
- [ ] 实时行情更新
- [ ] 响应时间 < 1s

---

## 8. A股/港股模块扩展

### 8.1 板块热力图数据源

**功能描述**: 展示 163 个概念板块的涨跌幅热力图，颜色编码涨跌（红涨绿跌），支持点击查看详情。

#### API 端点设计

```
GET /api/v1/cn/sectors/heatmap
GET /api/v1/cn/sectors/:code/stocks
```

#### 数据源

- **主数据源**: 新浪 `newFLJK` — 163 个概念板块
- **备用数据源**: AKShare `stock_board_industry_name_em()` — 行业板块
- **URL 模式**: `http://vip.stock.finance.sina.com.cn/quotes_service/api/json_v2.php/Market_Center.getHQNodeStockCount?node=new_fljk`

#### 数据流图

```
新浪 newFLJK API
    ↓
app/data/providers/sina_sector_provider.py (新增)
    ↓
app/services/sector_service.py (新增)
    ↓
app/api/v1/sectors.py (新增)
    ↓
内存缓存 (TTL: 5 分钟)
    ↓
JSON Response
```

#### 后端服务设计

**文件**: `app/data/providers/sina_sector_provider.py` (新增)

```python
# 核心逻辑:
# 1. 请求新浪 newFLJK 获取板块列表
# 2. 解析 JSON 响应，提取板块代码、名称、涨跌幅
# 3. 对每个板块，获取 TOP 5 成分股
# 4. 构建热力图数据结构
# 5. 缓存 5 分钟
```

**文件**: `app/services/sector_service.py` (新增)

```python
# 核心逻辑:
# 1. 调用 sina_sector_provider 获取原始数据
# 2. 计算颜色编码 (红涨绿跌，渐变色阶)
# 3. 排序：按涨跌幅绝对值排序（最活跃在前）
# 4. 计算板块统计：上涨/下跌/平盘数量
# 5. 返回热力图数据结构
```

#### 调度任务设计

**文件**: `app/jobs/sector_scan_job.py` (新增)

```python
# 调度: 每 30 分钟执行一次 (交易时段)
# 逻辑:
# 1. 调用 sina_sector_provider 获取全量板块数据
# 2. 存入 SectorSnapshot 表
# 3. 更新板块排名
# 4. 检测异动板块 (涨跌幅 > 3%)
```

#### 验收标准

- [ ] 163 个概念板块全部展示
- [ ] 颜色编码：涨红跌绿，渐变色阶
- [ ] 点击板块可查看详情（成分股、新闻、资金流向）
- [ ] 每 30 分钟自动更新
- [ ] 响应时间 < 500ms（缓存命中）
- [ ] 降级策略：新浪不可用时使用 AKShare

---

### 8.2 板块内个股数据源

**功能描述**: 点击板块后展示板块内个股列表，支持按涨跌幅/成交量/资金流向排序。

#### API 端点设计

```
GET /api/v1/cn/sectors/:code/stocks
```

#### 数据源

- **数据源**: 东财板块成分股 API
- **缓存**: 2 分钟

#### 数据流图

```
东财板块成分股 API
    ↓
app/data/providers/eastmoney_provider.py (增强)
    ↓
app/services/sector_service.py (增强)
    ↓
内存缓存 (TTL: 2 分钟)
    ↓
JSON Response
```

#### 验收标准

- [ ] 板块内个股列表完整
- [ ] 支持按涨跌幅/成交量/资金流向排序
- [ ] 响应时间 < 500ms

---

### 8.3 股东信息数据源

**功能描述**: 展示个股股东信息，包括前十大股东、持股比例、变化情况。

#### API 端点设计

```
GET /api/v1/cn/stocks/:code/shareholders
```

#### 数据源

- **数据源**: 东财股东研究 API
- **缓存**: 1 小时

#### 数据流图

```
东财股东研究 API
    ↓
app/data/providers/eastmoney_provider.py (增强)
    ↓
app/services/shareholder_service.py (新增)
    ↓
内存缓存 (TTL: 1 小时)
    ↓
JSON Response
```

#### 验收标准

- [ ] 前十大股东信息完整
- [ ] 持股比例计算准确
- [ ] 变化情况清晰展示
- [ ] 响应时间 < 1s

---

### 8.4 分红送转数据源

**功能描述**: 展示个股分红送转历史，包括每股分红、送股比例、登记日期等。

#### API 端点设计

```
GET /api/v1/cn/stocks/:code/dividends
```

#### 数据源

- **数据源**: 东财分红送转 API
- **缓存**: 1 小时

#### 数据流图

```
东财分红送转 API
    ↓
app/data/providers/eastmoney_provider.py (增强)
    ↓
app/services/dividend_service.py (新增)
    ↓
内存缓存 (TTL: 1 小时)
    ↓
JSON Response
```

#### 验收标准

- [ ] 分红送转历史完整
- [ ] 每股分红金额准确
- [ ] 连续分红年数正确
- [ ] 响应时间 < 1s

---

### 8.5 融资融券数据源

**功能描述**: 展示个股融资融券数据，包括融资余额、融券余额、趋势分析。

#### API 端点设计

```
GET /api/v1/cn/stocks/:code/margin
```

#### 数据源

- **数据源**: 东财融资融券 API
- **缓存**: 1 分钟

#### 数据流图

```
东财融资融券 API
    ↓
app/data/providers/eastmoney_provider.py (增强)
    ↓
app/services/margin_service.py (新增)
    ↓
内存缓存 (TTL: 1 分钟)
    ↓
JSON Response
```

#### 验收标准

- [ ] 融资融券数据准确
- [ ] 趋势分析清晰
- [ ] 历史数据完整
- [ ] 响应时间 < 1s

---

### 8.6 大宗交易数据源

**功能描述**: 展示个股大宗交易记录，包括交易价格、成交量、买卖方信息。

#### API 端点设计

```
GET /api/v1/cn/stocks/:code/block-trades
```

#### 数据源

- **数据源**: 东财大宗交易 API
- **缓存**: 2 分钟

#### 数据流图

```
东财大宗交易 API
    ↓
app/data/providers/eastmoney_provider.py (增强)
    ↓
app/services/block_trade_service.py (新增)
    ↓
内存缓存 (TTL: 2 分钟)
    ↓
JSON Response
```

#### 验收标准

- [ ] 大宗交易记录完整
- [ ] 交易价格/成交量准确
- [ ] 买卖方信息清晰
- [ ] 响应时间 < 1s

---

## 9. 美股模块扩展

### 9.1 机构持仓数据源 (13F)

**功能描述**: 展示美股机构持仓信息（13F Filing），包括机构名称、持股数量、持仓变化。

#### API 端点设计

```
GET /api/v1/us/stocks/:code/institutional
```

#### 数据源

- **数据源**: SEC EDGAR 13F Filing + Finnhub
- **缓存**: 1 小时

#### 数据流图

```
SEC EDGAR 13F Filing API
    ↓
app/data/providers/sec_provider.py (新增)
    ↓
app/services/institutional_service.py (新增)
    ↓
内存缓存 (TTL: 1 小时)
    ↓
JSON Response
```

#### 后端服务设计

**文件**: `app/data/providers/sec_provider.py` (新增)

```python
# 核心逻辑:
# 1. 请求 SEC EDGAR 13F Filing API
# 2. 解析 XML/JSON 响应
# 3. 提取机构名称、持股数量、持仓变化
# 4. 缓存 1 小时
```

**文件**: `app/services/institutional_service.py` (新增)

```python
# 核心逻辑:
# 1. 调用 sec_provider 获取原始数据
# 2. 计算持仓变化百分比
# 3. 排序：按持股数量降序
# 4. 返回机构持仓数据
```

#### 验收标准

- [ ] 机构持仓信息完整
- [ ] 持仓变化计算准确
- [ ] 响应时间 < 2s

---

### 9.2 分析师评级数据源

**功能描述**: 展示美股分析师评级，包括评级共识、目标价、评级变化。

#### API 端点设计

```
GET /api/v1/us/stocks/:code/analysts
```

#### 数据源

- **数据源**: Finnhub Analyst API + MarketBeat
- **缓存**: 1 小时

#### 数据流图

```
Finnhub Analyst API + MarketBeat
    ↓
app/data/providers/analyst_provider.py (新增)
    ↓
app/services/analyst_service.py (新增)
    ↓
内存缓存 (TTL: 1 小时)
    ↓
JSON Response
```

#### 验收标准

- [ ] 分析师评级信息完整
- [ ] 目标价计算准确
- [ ] 评级变化清晰展示
- [ ] 响应时间 < 2s

---

### 9.3 期权链数据源

**功能描述**: 展示美股期权链数据，包括行权价、到期日、隐含波动率、希腊字母。

#### API 端点设计

```
GET /api/v1/us/stocks/:code/options
```

#### 数据源

- **数据源**: CBOE + Yahoo Finance Options
- **缓存**: 1 分钟

#### 数据流图

```
CBOE + Yahoo Finance Options API
    ↓
app/data/providers/options_provider.py (新增)
    ↓
app/services/options_service.py (新增)
    ↓
内存缓存 (TTL: 1 分钟)
    ↓
JSON Response
```

#### 后端服务设计

**文件**: `app/data/providers/options_provider.py` (新增)

```python
# 核心逻辑:
# 1. 请求 CBOE/Yahoo Finance Options API
# 2. 解析期权链数据
# 3. 提取行权价、到期日、隐含波动率、希腊字母
# 4. 缓存 1 分钟
```

#### 验收标准

- [ ] 期权链数据完整
- [ ] 隐含波动率计算准确
- [ ] 希腊字母（Delta/Gamma/Theta）正确
- [ ] 响应时间 < 2s

---

## 10. 韩国股票模块扩展

### 10.1 外国投资者持股数据源

**功能描述**: 展示韩国股票外国投资者持股信息，包括持股比例、变化趋势。

#### API 端点设计

```
GET /api/v1/kr/stocks/:code/foreign-investors
```

#### 数据源

- **数据源**: KRX 外国投资者持股 API
- **缓存**: 2 分钟

#### 数据流图

```
KRX 外国投资者持股 API
    ↓
app/data/providers/krx_provider.py (新增)
    ↓
app/services/foreign_holding_service.py (新增)
    ↓
内存缓存 (TTL: 2 分钟)
    ↓
JSON Response
```

#### 后端服务设计

**文件**: `app/data/providers/krx_provider.py` (新增)

```python
# 核心逻辑:
# 1. 请求 KRX 外国投资者持股 API
# 2. 解析 JSON 响应
# 3. 提取持股比例、变化趋势
# 4. 缓存 2 分钟
```

#### 验收标准

- [ ] 外国投资者持股信息完整
- [ ] 持股比例计算准确
- [ ] 变化趋势清晰展示
- [ ] 响应时间 < 2s

---

### 10.2 做空数据源

**功能描述**: 展示韩国股票做空数据，包括做空量、做空比例、趋势分析。

#### API 端点设计

```
GET /api/v1/kr/stocks/:code/short-selling
```

#### 数据源

- **数据源**: KRX 做空数据 API
- **缓存**: 1 分钟

#### 数据流图

```
KRX 做空数据 API
    ↓
app/data/providers/krx_provider.py (增强)
    ↓
app/services/short_selling_service.py (新增)
    ↓
内存缓存 (TTL: 1 分钟)
    ↓
JSON Response
```

#### 验收标准

- [ ] 做空数据完整
- [ ] 做空比例计算准确
- [ ] 趋势分析清晰
- [ ] 响应时间 < 2s

---

## 11. 基金模块扩展

### 11.1 基金经理数据源

**功能描述**: 展示基金经理信息，包括履历、管理规模、历史业绩。

#### API 端点设计

```
GET /api/v1/funds/:code/manager
```

#### 数据源

- **数据源**: 天天基金 基金经理 API
- **缓存**: 1 小时

#### 数据流图

```
天天基金 基金经理 API
    ↓
app/data/providers/tiantianfund_provider.py (新增)
    ↓
app/services/fund_manager_service.py (新增)
    ↓
内存缓存 (TTL: 1 小时)
    ↓
JSON Response
```

#### 后端服务设计

**文件**: `app/data/providers/tiantianfund_provider.py` (新增)

```python
# 核心逻辑:
# 1. 请求天天基金 基金经理 API
# 2. 解析 JSON 响应
# 3. 提取经理履历、管理规模、历史业绩
# 4. 缓存 1 小时
```

#### 验收标准

- [ ] 基金经理信息完整
- [ ] 管理规模计算准确
- [ ] 历史业绩数据正确
- [ ] 响应时间 < 2s

---

### 11.2 基金评级数据源

**功能描述**: 展示基金评级信息，包括晨星、银河、海通评级，以及夏普比率、最大回撤等指标。

#### API 端点设计

```
GET /api/v1/funds/:code/ratings
```

#### 数据源

- **数据源**: 天天基金 + 晨星中国
- **缓存**: 1 小时

#### 数据流图

```
天天基金 + 晨星中国 API
    ↓
app/data/providers/tiantianfund_provider.py (增强)
    ↓
app/services/fund_rating_service.py (新增)
    ↓
内存缓存 (TTL: 1 小时)
    ↓
JSON Response
```

#### 验收标准

- [ ] 基金评级信息完整
- [ ] 夏普比率/最大回撤计算准确
- [ ] 多家评级机构数据一致
- [ ] 响应时间 < 2s

---

### 11.3 机构持有比例数据源

**功能描述**: 展示基金机构持有比例，包括机构名称、持股比例、变化情况。

#### API 端点设计

```
GET /api/v1/funds/:code/institutional
```

#### 数据源

- **数据源**: 天天基金 基金持有人 API
- **缓存**: 1 小时

#### 数据流图

```
天天基金 基金持有人 API
    ↓
app/data/providers/tiantianfund_provider.py (增强)
    ↓
app/services/fund_institutional_service.py (新增)
    ↓
内存缓存 (TTL: 1 小时)
    ↓
JSON Response
```

#### 验收标准

- [ ] 机构持有比例信息完整
- [ ] 持股比例计算准确
- [ ] 变化情况清晰展示
- [ ] 响应时间 < 2s

---

## 12. AI 分析模块扩展

### 12.1 多时间维度预测数据源

**功能描述**: 提供 1天/1周/1月/3月 多时间维度的股票预测，包括方向、置信度、目标价。

#### API 端点设计

```
GET /api/v1/ai/prediction/multi-timeframe
```

#### 数据源

- **数据源**: LLM + 历史行情 + 技术指标
- **缓存**: 10 分钟

#### 数据流图

```
LLM API + 历史行情 (yfinance) + 技术指标
    ↓
app/services/multi_forecast_service.py (新增)
    ↓
内存缓存 (TTL: 10 分钟)
    ↓
JSON Response
```

#### 后端服务设计

**文件**: `app/services/multi_forecast_service.py` (新增)

```python
# 核心逻辑:
# 1. 获取目标股票 90 天历史数据
# 2. 计算技术指标 (MA/MACD/RSI)
# 3. 构建多时间维度预测 prompt
# 4. 调用 LLM API
# 5. 解析并缓存结果

# Prompt 模板:
MULTI_FORECAST_PROMPT = """
基于以下数据，对 {symbol} ({name}) 进行多时间维度预测:

历史数据: {history}
技术指标: {indicators}
当前价格: {current_price}

请返回 JSON:
{
  "1d": { "direction": "up/down/neutral", "change_percent": float, "confidence": 0-1, "reasoning": "..." },
  "1w": { "direction": "up/down/neutral", "change_percent": float, "confidence": 0-1, "reasoning": "..." },
  "1m": { "direction": "up/down/neutral", "change_percent": float, "confidence": 0-1, "reasoning": "..." },
  "3m": { "direction": "up/down/neutral", "change_percent": float, "confidence": 0-1, "reasoning": "..." }
}
"""
```

#### 验收标准

- [ ] 4 个时间维度预测完整
- [ ] 方向/置信度/目标价准确
- [ ] 每个维度有清晰的理由说明
- [ ] 响应时间 < 5s

---

### 12.2 综合分析评分数据源

**功能描述**: 提供股票综合分析评分，包括技术面/基本面/资金流/新闻四个维度的评分和雷达图。

#### API 端点设计

```
GET /api/v1/ai/analysis/comprehensive
```

#### 数据源

- **数据源**: LLM + 技术指标 + 基本面数据 + 资金流向 + 新闻
- **缓存**: 10 分钟

#### 数据流图

```
LLM API + 技术指标 + 基本面数据 + 资金流向 + 新闻
    ↓
app/services/comprehensive_analysis_service.py (新增)
    ↓
内存缓存 (TTL: 10 分钟)
    ↓
JSON Response
```

#### 后端服务设计

**文件**: `app/services/comprehensive_analysis_service.py` (新增)

```python
# 核心逻辑:
# 1. 获取技术指标 (MA/MACD/RSI)
# 2. 获取基本面数据 (PE/PB/ROE)
# 3. 获取资金流向数据
# 4. 获取相关新闻
# 5. LLM 生成综合分析评分
# 6. 返回四维评分 + 雷达图数据
```

#### 验收标准

- [ ] 四个维度评分完整
- [ ] 综合评分计算准确
- [ ] 雷达图数据正确
- [ ] 响应时间 < 5s

---

### 12.3 组合优化数据源

**功能描述**: 基于用户持仓生成组合优化建议，包括再平衡建议、有效前沿、压力测试。

#### API 端点设计

```
POST /api/v1/ai/portfolio/optimize
```

#### 数据源

- **数据源**: LLM + 历史收益率 + 协方差矩阵
- **缓存**: 无（POST 请求）

#### 数据流图

```
用户持仓数据 + 历史收益率 (yfinance)
    ↓
app/services/portfolio_optimizer.py (新增)
    ↓
LLM API (组合优化分析)
    ↓
JSON Response
```

#### 后端服务设计

**文件**: `app/services/portfolio_optimizer.py` (新增)

```python
# 核心逻辑:
# 1. 获取用户持仓数据
# 2. 获取历史收益率数据
# 3. 计算协方差矩阵
# 4. LLM 生成组合优化建议
# 5. 返回当前组合 vs 优化组合对比
```

#### 验收标准

- [ ] 组合优化建议合理
- [ ] 再平衡建议清晰
- [ ] 有效前沿数据正确
- [ ] 响应时间 < 5s

---

### 12.4 压力测试数据源

**功能描述**: 基于用户持仓进行压力测试，模拟市场崩盘、加息、板块轮动等场景。

#### API 端点设计

```
POST /api/v1/ai/portfolio/stress-test
```

#### 数据源

- **数据源**: LLM + 历史收益率 + 情景模拟
- **缓存**: 无（POST 请求）

#### 数据流图

```
用户持仓数据 + 历史收益率 (yfinance)
    ↓
app/services/stress_tester.py (新增)
    ↓
LLM API (情景模拟分析)
    ↓
JSON Response
```

#### 后端服务设计

**文件**: `app/services/stress_tester.py` (新增)

```python
# 核心逻辑:
# 1. 获取用户持仓数据
# 2. 获取历史收益率数据
# 3. 构建压力测试场景
# 4. LLM 生成压力测试分析
# 5. 返回各场景影响 + 相关性矩阵
```

#### 验收标准

- [ ] 压力测试场景完整
- [ ] 各场景影响计算准确
- [ ] 相关性矩阵正确
- [ ] 响应时间 < 5s

---

## 13. 部署架构

### 13.1 当前部署

```
Docker (Python 3.13-slim)
├── uvicorn (port 8000)
├── SQLAlchemy + SQLite
├── APScheduler
└── 内存缓存
```

### 13.2 升级后部署

```
Docker Compose
├── finance-intelligence-service (Python)
│   ├── uvicorn (port 8000)
│   ├── SQLAlchemy + PostgreSQL
│   ├── APScheduler
│   └── Redis (缓存)
├── redis (port 6379)
└── postgres (port 5432)
```

### 13.3 性能优化

| 优化项 | 方案 | 预期效果 |
|--------|------|---------|
| 缓存 | Redis 替代内存缓存 | 减少 50% 外部请求 |
| 数据库 | PostgreSQL 替代 SQLite | 并发性能提升 10x |
| 连接池 | httpx 连接池 | 减少连接开销 |
| 异步 | 全面异步化 | 吞吐量提升 3x |

---

## 14. 验收标准总表

### P1 核心面板 (2 周)

| 模块 | 验收标准 | 负责 |
|------|---------|------|
| 板块热力图 | 163 板块展示，颜色编码，点击详情 | 后端 |
| 新闻分级 | AI 分级准确率 > 80%，一句话总结 | 后端 |
| 个股推荐 | 4 种策略，AI 分析附带 | 后端 |
| 研报摘要 | 一句话摘要，评级目标价 | 后端 |

### P2 AI 智能 (2 周)

| 模块 | 验收标准 | 负责 |
|------|---------|------|
| AI 预测 | 三场景概率和为 100%，理由清晰 | 后端 |
| AI 基金推荐 | 夏普比率/最大回撤计算准确 | 后端 |
| Dashboard 增强 | 置信度趋势图，风险模式时间线 | 后端 |

### P3 交互体验 (1 周)

| 模块 | 验收标准 | 负责 |
|------|---------|------|
| 历史趋势 | 30 天快照存储，趋势分析 | 后端 |
| 调度扩展 | 5 个新调度任务按计划执行 | 后端 |
| 交互增强 | 联动/下钻/实时刷新 | 前端 |

### P4 深度数据 (3 周)

| 模块 | 验收标准 | 负责 |
|------|---------|------|
| K 线图 | 日/周/月 K，技术指标叠加 | 后端 |
| 技术指标 | 6 种指标，误差 < 0.01 | 后端 |
| 资金流向 | 主力/散户/北向数据准确 | 后端 |
| 个股详情 | 行情+财务+估值+研报 | 后端 |
| 宏观数据 | CPI/PMI/GDP 数据准确 | 后端 |

### Dashboard 9 大模块 (与 ZoomLab v5.0 对齐)

| 模块 | 验收标准 | 负责 |
|------|---------|------|
| MarketOverview | 7 个全球指数 + 恐惧贪婪指数 | 后端 |
| PortfolioSummary | 总资产/日收益/行业分布饼图 | 后端 |
| NewsFeed | 近24小时新闻 + 分类筛选 + Top 3 | 后端 |
| AIDailyPrediction | 预测摘要 + 三场景概率 + 关键因素 | 后端 |
| AIRiskAlerts | 风险等级 + 30天趋势 + 风险因素 | 后端 |
| AIOpportunitySignals | 机会等级 + 类型筛选 + 机会卡片 | 后端 |
| AIStockRecommendations | 市场筛选 + 推荐股票网格 | 后端 |
| AITradingDecisions | 操作筛选 + 决策卡片 + 免责声明 | 后端 |
| Watchlist | 自选股列表 + 实时行情 | 后端 |

### A股/港股模块扩展

| 模块 | 验收标准 | 负责 |
|------|---------|------|
| 板块热力图 | 163 板块展示，颜色编码，点击详情 | 后端 |
| 板块内个股 | 成分股列表，支持排序 | 后端 |
| 股东信息 | 前十大股东，持股比例，变化情况 | 后端 |
| 分红送转 | 分红历史，每股分红，连续年数 | 后端 |
| 融资融券 | 融资余额，融券余额，趋势分析 | 后端 |
| 大宗交易 | 交易记录，价格，成交量，买卖方 | 后端 |

### 美股模块扩展

| 模块 | 验收标准 | 负责 |
|------|---------|------|
| 机构持仓 (13F) | 机构名称，持股数量，持仓变化 | 后端 |
| 分析师评级 | 评级共识，目标价，评级变化 | 后端 |
| 期权链 | 行权价，到期日，隐含波动率，希腊字母 | 后端 |

### 韩国股票模块扩展

| 模块 | 验收标准 | 负责 |
|------|---------|------|
| 外国投资者持股 | 持股比例，变化趋势 | 后端 |
| 做空数据 | 做空量，做空比例，趋势分析 | 后端 |

### 基金模块扩展

| 模块 | 验收标准 | 负责 |
|------|---------|------|
| 基金经理 | 履历，管理规模，历史业绩 | 后端 |
| 基金评级 | 晨星/银河/海通评级，夏普比率 | 后端 |
| 机构持有比例 | 机构名称，持股比例，变化情况 | 后端 |

### AI 分析模块扩展

| 模块 | 验收标准 | 负责 |
|------|---------|------|
| 多时间维度预测 | 1天/1周/1月/3月 预测完整 | 后端 |
| 综合分析评分 | 四维评分 + 雷达图 | 后端 |
| 组合优化 | 优化建议 + 再平衡 + 有效前沿 | 后端 |
| 压力测试 | 场景模拟 + 影响计算 + 相关性矩阵 | 后端 |
