# BroZoomOut 金融情报服务 — 剩余工作技术方案

> 版本: v2.0 | 日期: 2026-05-28 | 状态: 规划中
> P0 已完成：主题修复 + 指数滚动条 + 置信度药丸

---

## 一、已完成 vs 待做

| 阶段 | 内容 | 状态 |
|------|------|------|
| P0 | 主题修复（dark: 前缀） | ✅ commit `fcb372a` |
| P0 | MarketTicker 指数滚动条 | ✅ commit `fcb372a` |
| P0 | DashboardHero 置信度药丸 | ✅ commit `fcb372a` |
| **P1** | **板块数据源 + 热力图** | 📋 待做 |
| **P1** | **新闻分级 + 利好利空** | 📋 待做 |
| **P1** | **个股推荐** | 📋 待做 |
| **P1** | **研报摘要** | 📋 待做 |
| **P2** | **AI 预测** | 📋 待做 |
| **P2** | **基金推荐** | 📋 待做 |
| **P2** | **Dashboard 端点增强** | 📋 待做 |
| **P3** | **历史趋势数据扩充** | 📋 待做 |
| **P3** | **调度任务扩展** | 📋 待做 |

---

## 二、现有 API 清单（已实现）

```
GET  /api/brozoomout/summary              # 总览摘要
GET  /api/brozoomout/dashboard             # 完整仪表盘
GET  /api/brozoomout/markets/{market}      # 单市场详情
GET  /api/brozoomout/watchlist             # 关注列表
POST /api/brozoomout/watchlist             # 添加关注
DEL  /api/brozoomout/watchlist/{asset_id}  # 删除关注
GET  /api/brozoomout/strategy              # 策略建议
POST /api/brozoomout/strategy/preview      # 策略预览
POST /api/brozoomout/snapshots/refresh     # 刷新快照
GET  /api/brozoomout/snapshots/{id}        # 查询快照

GET  /api/v1/market/overview               # 市场概览（11 指数）
GET  /api/v1/market/daily-report           # 日报
GET  /api/v1/market/half-day-report        # 半日报

GET  /api/v1/experience/feed               # 认知 Feed
GET  /api/v1/experience/timeline           # 时间线
GET  /api/v1/experience/morning-brief      # 晨报
GET  /api/v1/experience/diff               # 认知差异
```

---

## 三、待做模块详细设计

### P1-1：板块动向模块

**目标**：实时展示行业/概念板块涨跌排行

**新增文件**：
```
app/data/providers/tencent_finance.py   # 腾讯财经 HTTP 采集
app/services/sector_service.py          # 板块数据服务
app/api/v1/sectors.py                   # 板块 API
app/jobs/sector_scan_job.py             # 定时扫描任务
```

**数据源**：腾讯财经 HTTP API
- 行业板块：`http://qt.gtimg.cn/q=sh000001`（免翻墙、免 key、Docker 友好）
- 备选：东方财富 push2.eastmoney.com（需 Surge DIRECT 规则）

**API 设计**：
```
GET /api/brozoomout/sectors
Query:
  type      string  industry|concept  默认 industry
  limit     int     返回数量          默认 15
  market    string  cn|all            默认 cn

Response:
{
  ok: true,
  data: {
    type: "industry",
    updated_at: "2026-05-28T15:30:00Z",
    sectors: [
      {
        name: "银行",
        change_pct: 2.15,
        volume: 1234567890,
        leading_stock: "招商银行",
        leading_stock_change: 3.2,
        turnover_rate: 0.85
      }
    ]
  }
}
```

**调度**：交易时段（9:30-15:00）每 30 分钟

**注意**：
- 腾讯 API 返回 GBK 编码，需 `decode('gbk')`
- 请求间隔 ≥ 1 秒，避免限流
- 缓存 5 分钟，避免重复请求

---

### P1-2：新闻分级模块

**目标**：聚合新闻 + AI 分级 + 利好利空结构化提取

**新增文件**：
```
app/data/providers/rss_provider.py      # 扩展现有（新增源）
app/services/news_service.py            # 新闻服务
app/services/sentiment_service.py       # 利好利空提取
app/api/v1/news.py                      # 新闻 API
app/jobs/news_digest_job.py             # 定时聚合任务
```

**数据源扩展**：
- 现有：Google Finance RSS、CNBC、FT中文
- 新增：华尔街见闻 RSS、财联社电报 API

**AI 分级 Prompt**：
```
你是一个金融新闻分析师。请对以下新闻进行分级：

新闻标题：{title}
新闻内容：{content}

请返回 JSON：
{
  "importance": "major|notable|general",
  "sentiment_events": [
    {
      "event": "事件描述",
      "type": "bullish|bearish|neutral",
      "impact": "high|medium|low",
      "affected_sectors": ["银行", "地产"]
    }
  ],
  "ai_summary": "一句话总结"
}
```

**API 设计**：
```
GET /api/brozoomout/news
Query:
  days         int     天数            默认 1
  importance   string  major|notable|all  默认 all
  limit        int     返回数量        默认 20

Response:
{
  ok: true,
  data: {
    items: [
      {
        id: "news_20260528_001",
        title: "央行宣布降准 50 个基点",
        source: "wallstreetcn",
        url: "https://...",
        published_at: "2026-05-28T10:00:00Z",
        importance: "major",
        sentiment_events: [
          {
            event: "央行降准 50bp",
            type: "bullish",
            impact: "high",
            affected_sectors: ["银行", "房地产"]
          }
        ],
        ai_summary: "央行降准 50bp，释放约 1 万亿流动性，利好金融和地产"
      }
    ],
    stats: {
      total: 45,
      major: 3,
      notable: 12,
      bullish: 5,
      bearish: 3
    }
  }
}
```

**调度**：每 1 小时

---

### P1-3：个股推荐模块

**目标**：基于量化筛选 + AI 分析，推荐值得关注的个股

**新增文件**：
```
app/data/providers/stock_screener.py    # 量化筛选器
app/services/stock_pick_service.py      # 个股推荐服务
app/api/v1/stock_picks.py               # 个股推荐 API
app/jobs/stock_pick_job.py              # 每日盘前扫描
```

**筛选逻辑**：
1. **量价异动**：成交量 > 5 日均量 2 倍
2. **技术指标**：RSI < 30（超卖）或 RSI > 70（超买）
3. **MACD 金叉/死叉**
4. **板块联动**：热门板块龙头股
5. **AI 综合评估**：将以上指标送 LLM 生成推荐理由

**API 设计**：
```
GET /api/brozoomout/stock-picks
Query:
  market   string  cn|hk|us|kr|all  默认 all
  signal   string  buy|sell|watch|all  默认 all
  limit    int     返回数量          默认 10

Response:
{
  ok: true,
  data: {
    generated_at: "2026-05-28T09:00:00Z",
    picks: [
      {
        code: "600036",
        name: "招商银行",
        market: "cn_stock",
        signal: "buy",
        signal_strength: "medium",
        current_price: 38.50,
        target_price: 42.00,
        stop_loss: 36.00,
        upside_pct: 9.1,
        reason: "银行板块领涨，RSI 32 超卖反弹，MACD 金叉，基本面稳健",
        metrics: {
          rsi: 32,
          macd_signal: "golden_cross",
          volume_ratio: 2.1,
          pe_ratio: 6.8
        },
        related_news: ["央行降准利好银行板块"]
      }
    ]
  }
}
```

**调度**：每日盘前 9:00

---

### P1-4：研报摘要模块

**目标**：抓取券商研报，AI 生成一句话总结

**新增文件**：
```
app/data/providers/eastmoney_research.py  # 东财研报 API
app/services/research_service.py          # 研报服务
app/api/v1/research.py                    # 研报 API
app/jobs/research_digest_job.py           # 每日摘要任务
```

**数据源**：东方财富研报列表 API

**API 设计**：
```
GET /api/brozoomout/research
Query:
  days    int     天数      默认 1
  limit   int     数量      默认 10

Response:
{
  ok: true,
  data: {
    reports: [
      {
        id: "rep_20260528_001",
        title: "招商银行(600036)深度报告：零售之王再出发",
        broker: "中金公司",
        analyst: "张三",
        target_price: 45.0,
        rating: "推荐",
        current_price: 38.5,
        upside_pct: 16.9,
        published_at: "2026-05-28",
        ai_summary: "中金看好招行零售转型，目标价 45 元，对应 12x PE，维持推荐",
        related_stock: "600036",
        related_sectors: ["银行"]
      }
    ]
  }
}
```

**调度**：每日 10:00

---

### P2-1：AI 预测模块

**目标**：基于当前状态 + 历史数据，生成未来推演

**新增文件**：
```
app/services/forecast_service.py   # AI 预测服务
app/api/v1/forecast.py             # 预测 API
app/jobs/forecast_job.py           # 每日收盘后生成
```

**实现逻辑**：
1. 收集当前 snapshot（risk_mode、confidence、indices）
2. 收集 30 天历史 snapshot 趋势
3. 构建 prompt 送 LLM
4. 结构化输出三场景预测

**LLM Prompt**：
```
你是一个金融市场分析师。基于以下数据，生成未来 {horizon} 的市场预测。

当前状态：
- 风险模式：{risk_mode}
- 置信度：{confidence}%
- 波动率：{volatility}
- 关键指数：{indices_summary}

历史趋势（近 30 天）：
{history_summary}

请返回 JSON：
{
  "prediction": "一段话预测",
  "confidence": 65,
  "key_factors": ["因素1", "因素2"],
  "scenarios": [
    {"name": "牛市", "probability": 25, "description": "...", "impact": "..."},
    {"name": "基准", "probability": 50, "description": "...", "impact": "..."},
    {"name": "熊市", "probability": 25, "description": "...", "impact": "..."}
  ]
}
```

**API 设计**：
```
GET /api/brozoomout/forecast
Query:
  horizon   string  1w|1m  默认 1w

Response:
{
  ok: true,
  data: {
    horizon: "1w",
    generated_at: "2026-05-28T15:30:00Z",
    prediction: "下周市场大概率维持震荡，关注美联储议息会议结果",
    confidence: 65,
    key_factors: ["美联储议息会议", "A 股量能萎缩", "北向资金流向"],
    scenarios: [
      {
        name: "牛市",
        probability: 25,
        description: "美联储鸽派 + 北向大幅流入 + 政策利好",
        impact: "risk_mode → risk_on, confidence > 70"
      },
      {
        name: "基准",
        probability: 50,
        description: "维持现状，窄幅震荡",
        impact: "risk_mode → neutral, confidence 40-60"
      },
      {
        name": "熊市",
        probability: 25,
        description: "美联储鹰派 + 地缘风险升级",
        impact: "risk_mode → risk_off, confidence < 30"
      }
    ]
  }
}
```

**调度**：每日收盘后 15:30

---

### P2-2：基金推荐模块

**目标**：同类基金 Top N 推荐

**新增文件**：
```
app/data/providers/fund_provider.py    # 基金数据采集
app/services/fund_pick_service.py      # 基金推荐服务
app/api/v1/fund_picks.py               # 基金推荐 API
app/jobs/fund_pick_job.py              # 每日更新
```

**API 设计**：
```
GET /api/brozoomout/fund-picks
Query:
  type    string  equity|bond|hybrid|index|all  默认 all
  limit   int     数量                          默认 10

Response:
{
  ok: true,
  data: {
    generated_at: "2026-05-28T09:00:00Z",
    funds: [
      {
        code: "161725",
        name: "招商中证白酒指数",
        type: "index",
        return_1m: 5.2,
        return_3m: 12.8,
        return_6m: -3.5,
        risk_level: "medium",
        aum: "150 亿",
        manager: "侯昊"
      }
    ]
  }
}
```

**调度**：每日 9:00

---

### P2-3：Dashboard 端点增强

**修改文件**：`app/api/v1/bro_zoomout.py`（get_dashboard 函数）

**在现有响应中增加字段**：

```json
{
  "data": {
    "...existing fields...",

    "indices": [
      {"name": "上证指数", "symbol": "000001", "price": 4093.73, "change_pct": -1.25}
    ],

    "sector_movers": {
      "top_gainers": [{"name": "银行", "change_pct": 2.15}],
      "top_losers": [{"name": "半导体", "change_pct": -3.2}]
    },

    "news_highlights": [
      {"title": "央行降准", "importance": "major", "sentiment": "bullish"}
    ],

    "confidence_delta": -2,
    "risk_mode_changed": false,
    "market_count_by_sentiment": {
      "bullish": 2,
      "neutral": 3,
      "bearish": 1
    }
  }
}
```

---

### P3-1：历史快照扩充

**修改文件**：`app/api/v1/bro_zoomout.py`（archive 相关）

**改动**：
- 当前只返回 1-3 条快照
- 支持 `?days=30` 参数，返回最多 30 天历史
- 每条快照增加 `market_summary`（当日主要指数涨跌）

---

### P3-2：调度任务扩展

| 任务 | 频率 | 文件 | 新增依赖 |
|------|------|------|----------|
| `sector_scan_job` | 每 30 分钟（交易时段） | `app/jobs/sector_scan_job.py` | 腾讯 API |
| `news_digest_job` | 每 1 小时 | `app/jobs/news_digest_job.py` | RSS + LLM |
| `stock_pick_job` | 每日 9:00 | `app/jobs/stock_pick_job.py` | yfinance + LLM |
| `research_digest_job` | 每日 10:00 | `app/jobs/research_digest_job.py` | 东财 API + LLM |
| `forecast_job` | 每日 15:30 | `app/jobs/forecast_job.py` | LLM + 历史数据 |
| `fund_pick_job` | 每日 9:00 | `app/jobs/fund_pick_job.py` | 基金数据源 |

---

## 四、新增模块结构

```
app/
├── api/v1/
│   ├── sectors.py              # 🆕 板块 API
│   ├── news.py                 # 🆕 新闻 API
│   ├── stock_picks.py          # 🆕 个股推荐 API
│   ├── research.py             # 🆕 研报 API
│   ├── forecast.py             # 🆕 AI 预测 API
│   ├── fund_picks.py           # 🆕 基金推荐 API
│   └── brozoomout.py           # ✏️ 修改（Dashboard 增强）
├── data/providers/
│   ├── tencent_finance.py      # 🆕 腾讯财经
│   ├── eastmoney_research.py   # 🆕 东财研报
│   ├── stock_screener.py       # 🆕 量化筛选器
│   ├── fund_provider.py        # 🆕 基金数据
│   ├── rss_provider.py         # ✏️ 修改（新增源）
│   └── yfinance_provider.py    # ✏️ 修改（个股筛选）
├── services/
│   ├── sector_service.py       # 🆕
│   ├── news_service.py         # 🆕
│   ├── sentiment_service.py    # 🆕 利好利空提取
│   ├── stock_pick_service.py   # 🆕
│   ├── research_service.py     # 🆕
│   ├── forecast_service.py     # 🆕
│   └── fund_pick_service.py    # 🆕
└── jobs/
    ├── sector_scan_job.py      # 🆕
    ├── news_digest_job.py      # 🆕
    ├── stock_pick_job.py       # 🆕
    ├── research_digest_job.py  # 🆕
    ├── forecast_job.py         # 🆕
    └── fund_pick_job.py        # 🆕
```

---

## 五、数据源可用性

| 数据源 | 协议 | Docker 可达 | 需 Key | 备注 |
|--------|------|-------------|--------|------|
| 腾讯财经 qt.gtimg.cn | HTTP | ✅ | ❌ | 纯 HTTP，Surge 不拦截 |
| yfinance | HTTPS | ⚠️ 需代理 | ❌ | Surge 增强模式下需 HTTP_PROXY |
| RSS Feeds | HTTPS | ⚠️ 需代理 | ❌ | 同上 |
| 东财研报 | HTTPS | ⚠️ 需代理 | ❌ | push2.eastmoney.com 需 Surge DIRECT |
| AI Gateway | HTTP | ✅ host.docker.internal | ❌ | 本地服务 |

---

## 六、LLM 调用规范

所有 AI 分析通过统一客户端 `app/core/llm_client.py`：
- 模型：deepseek-chat（通过 AI Gateway）
- 超时：30 秒
- 重试：2 次，指数退避
- 缓存：相同输入 10 分钟内复用

---

## 七、风险与缓解

| 风险 | 缓解措施 |
|------|----------|
| 东财 API 反爬 | 控制频率 + 缓存 + 备选腾讯 |
| LLM 调用成本 | 盘后批量生成，非实时 |
| RSS 源失效 | 监控 feed 健康，自动降级 |
| 板块数据延迟 | 30 分钟足够决策 |
| 量化筛选误报 | 加入置信度阈值 |

---

## 八、工期估算

| 阶段 | 模块 | 工期 | 前置 |
|------|------|------|------|
| P1 | 板块数据源 + API | 2 天 | 无 |
| P1 | 新闻分级 + 利好利空 | 2 天 | 无 |
| P1 | 个股推荐 | 3 天 | 无 |
| P1 | 研报摘要 | 2 天 | 无 |
| P2 | AI 预测 | 2 天 | P1（需历史数据） |
| P2 | 基金推荐 | 2 天 | 无 |
| P2 | Dashboard 端点增强 | 1 天 | P1 |
| P3 | 历史快照扩充 | 0.5 天 | 无 |
| P3 | 调度任务扩展 | 1 天 | 所有模块 |
| **总计** | | **15.5 天** | |
