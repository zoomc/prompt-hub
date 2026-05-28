# BroZoomOut 金融情报服务 — 全面升级技术方案

> 版本: v1.0 | 日期: 2026-05-28 | 状态: 规划中

## 一、项目定位

BroZoomOut 是 ZoomLab 的核心金融 intelligence 引擎，负责：
- 多市场数据采集与分析（A股/港股/美股/韩股/基金/外汇）
- AI 驱动的综合研判（风险模式、置信度、策略建议）
- 结构化输出供前端消费

**差异化定位**：同花顺没有"AI 综合研判 + 策略建议 + 风险预警"，这是 BroZoomOut 的护城河。

---

## 二、现状分析

### 2.1 已有能力

| 模块 | 能力 | 数据源 |
|------|------|--------|
| 市场数据 | 全球指数行情（11 个）、宏观资产（BTC/原油/美元） | yfinance |
| 风险研判 | risk_mode（risk_off/neutral/risk_on）、置信度、波动率 | AI + 规则 |
| 策略建议 | 现金/权益/防御配比、短期/中期动作 | AI 生成 |
| 新闻聚合 | RSS（Google Finance/CNBC/FT中文） | feedparser |
| 认知图谱 | 叙事时间线、认知差异、重放分析 | 自研 |
| 历史快照 | 每日/半日快照存储与查询 | SQLite |

### 2.2 关键缺口

| 缺口 | 影响 | 优先级 |
|------|------|--------|
| 无板块数据 | 用户看不到行业/概念板块涨跌 | P0 |
| 无个股推荐 | 用户无法获得具体投资标的 | P1 |
| 无新闻分级 | 所有新闻混在一起，无重要性区分 | P0 |
| 无利好利空结构化 | 用户需要自己从新闻中提取 | P1 |
| 无研报摘要 | 券商研报是重要决策依据，完全缺失 | P1 |
| 无 AI 预测 | 只有当前状态，没有未来推演 | P2 |
| 无基金推荐 | 基金是重要资产类别，未覆盖 | P2 |
| 历史深度不足 | 快照只有 1-3 天，无法看趋势 | P0 |
| 置信度无上下文 | 只有数字，没有 delta 变化 | P0 |

---

## 三、升级方案

### Phase 1 — 数据层增强（1-2 周）

#### 3.1 板块动向模块

**目标**：实时展示行业和概念板块涨跌排行

**数据源**：
- 腾讯财经 HTTP API（`http://qt.gtimg.cn`）— 免翻墙、免 key、Docker 友好
- 备选：东方财富 push2.eastmoney.com（需 Surge DIRECT 规则）

**API 设计**：
```
GET /api/brozoomout/sectors
Query: ?type=industry|concept&limit=15&market=cn
Response: {
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
        leading_stock_change: 3.2
      }
    ]
  }
}
```

**调度频率**：交易时段每 30 分钟

#### 3.2 新闻分级模块

**目标**：对新闻按重要性分级，提取利好利空事件

**数据源扩展**：
- 现有：Google Finance RSS、CNBC、FT中文
- 新增：华尔街见闻 RSS、财联社电报 API

**分级逻辑**：
1. AI 判断重要性（重大/关注/一般）
2. 提取利好利空标签
3. 关联受影响板块

**API 设计**：
```
GET /api/brozoomout/news
Query: ?days=1&importance=major|notable|all&limit=20
Response: {
  ok: true,
  data: {
    items: [
      {
        id: "news_xxx",
        title: "央行宣布降准 50 个基点",
        source: "wallstreetcn",
        published_at: "2026-05-28T10:00:00Z",
        importance: "major",
        sentiment_events: [
          {
            event: "央行降准",
            type: "bullish",
            impact: "high",
            affected_sectors: ["银行", "房地产"],
            reason: "释放流动性，利好金融和地产"
          }
        ],
        ai_summary: "央行降准 50bp，释放约 1 万亿流动性..."
      }
    ]
  }
}
```

**调度频率**：每 1 小时

#### 3.3 个股推荐模块

**目标**：基于量化筛选 + AI 分析，推荐值得关注的个股

**筛选逻辑**：
1. 量价异动（成交量突增、价格突破）
2. 技术指标（RSI 超卖/超买、MACD 金叉/死叉）
3. 板块联动（热门板块龙头股）
4. AI 综合评估

**API 设计**：
```
GET /api/brozoomout/stock-picks
Query: ?market=cn|hk|us|kr&signal=buy|sell|watch|all&limit=10
Response: {
  ok: true,
  data: {
    picks: [
      {
        code: "600036",
        name: "招商银行",
        market: "cn_stock",
        signal: "buy",
        signal_strength: "medium",
        current_price: 38.5,
        target_price: 42.0,
        stop_loss: 36.0,
        reason: "银行板块领涨，RSI 超卖反弹，基本面稳健",
        metrics: {
          rsi: 32,
          macd_signal: "golden_cross",
          volume_ratio: 2.1
        }
      }
    ]
  }
}
```

**调度频率**：每日盘前（9:00）

#### 3.4 研报摘要模块

**目标**：抓取券商研报，AI 生成一句话总结

**数据源**：东方财富研报 API

**API 设计**：
```
GET /api/brozoomout/research
Query: ?days=1&limit=10
Response: {
  ok: true,
  data: {
    reports: [
      {
        title: "招商银行(600036)深度报告：零售之王再出发",
        broker: "中金公司",
        target_price: 45.0,
        rating: "推荐",
        published_at: "2026-05-28",
        ai_summary: "中金看好招行零售转型，目标价 45 元，对应 12x PE",
        related_stock: "600036"
      }
    ]
  }
}
```

**调度频率**：每日

#### 3.5 AI 预测模块

**目标**：基于当前状态 + 历史数据，生成未来推演

**实现**：
1. 收集当前 snapshot 数据（risk_mode、confidence、indices）
2. 收集历史 snapshot 趋势（30 天）
3. 构建 prompt 送 LLM
4. 结构化输出三场景预测

**API 设计**：
```
GET /api/brozoomout/forecast
Query: ?horizon=1w|1m
Response: {
  ok: true,
  data: {
    horizon: "1w",
    generated_at: "2026-05-28T18:00:00Z",
    prediction: "下周市场大概率维持震荡，关注美联储议息会议",
    confidence: 65,
    key_factors: ["美联储议息", "A 股量能萎缩", "北向资金流向"],
    scenarios: [
      {
        name: "牛市",
        probability: 25,
        description: "美联储鸽派 + 北向大幅流入",
        impact: "risk_mode → risk_on, confidence > 70"
      },
      {
        name: "基准",
        probability: 50,
        description: "维持现状，窄幅震荡",
        impact: "risk_mode → neutral, confidence 40-60"
      },
      {
        name: "熊市",
        probability: 25,
        description: "美联储鹰派 + 地缘风险升级",
        impact: "risk_mode → risk_off, confidence < 30"
      }
    ]
  }
}
```

**调度频率**：每日收盘后

---

### Phase 2 — 现有端点增强（1 周）

#### 3.6 Dashboard 端点增强

当前 `/api/brozoomout/dashboard` 响应中增加：

```json
{
  "data": {
    "...existing fields...",
    "indices": [
      {"name": "上证指数", "price": 4093.73, "change_pct": -1.25}
    ],
    "sector_movers": {
      "top_gainers": [{"name": "银行", "change_pct": 2.15}],
      "top_losers": [{"name": "半导体", "change_pct": -3.2}]
    },
    "news_highlights": [
      {"title": "央行降准", "importance": "major", "sentiment": "bullish"}
    ],
    "confidence_delta": -2,
    "risk_mode_changed": false
  }
}
```

#### 3.7 Market Detail 端点增强

`/api/brozoomout/markets/{market}` 增加：

```json
{
  "data": {
    "...existing fields...",
    "sparkline": [
      {"date": "2026-05-22", "close": 4150.0},
      {"date": "2026-05-23", "close": 4120.5}
    ],
    "volume_trend": "increasing",
    "related_stocks": [
      {"code": "600036", "name": "招商银行", "change_pct": 3.2}
    ]
  }
}
```

#### 3.8 Archive 端点增强

`/api/brozoomout/archive` 支持查询参数：

```
GET /api/brozoomout/archive?days=30
Response: {
  data: {
    items: [
      {"snapshot_id": "snap_xxx", "as_of": "2026-05-01", "risk_mode": "neutral", "confidence_score": 55}
    ]  // 最多 30 条
  }
}
```

---

### Phase 3 — 调度任务扩展（1 周）

| 任务 | 频率 | 说明 | 新增依赖 |
|------|------|------|----------|
| `sector_scan_job` | 每 30 分钟（交易时段） | 抓取板块数据 | 腾讯 API |
| `news_digest_job` | 每 1 小时 | 聚合新闻，AI 分级 | RSS + LLM |
| `stock_pick_job` | 每日盘前 9:00 | 扫描全市场推荐 | yfinance + LLM |
| `research_digest_job` | 每日 10:00 | 抓取研报，AI 总结 | 东财 API + LLM |
| `forecast_job` | 每日 15:30 | 生成次日/次周预测 | LLM + 历史数据 |

---

## 四、技术架构

### 4.1 新增模块结构

```
app/
├── api/v1/
│   ├── brozoomout.py          # 现有
│   ├── sectors.py             # 新增：板块 API
│   ├── news.py                # 新增：新闻 API
│   ├── stock_picks.py         # 新增：个股推荐 API
│   ├── research.py            # 新增：研报 API
│   └── forecast.py            # 新增：AI 预测 API
├── data/providers/
│   ├── yfinance_provider.py   # 现有
│   ├── rss_provider.py        # 现有
│   ├── tencent_finance.py     # 新增：腾讯财经 API
│   ├── eastmoney_research.py  # 新增：东财研报 API
│   └── stock_screener.py      # 新增：量化筛选器
├── services/
│   ├── sector_service.py      # 新增
│   ├── news_service.py        # 新增
│   ├── stock_pick_service.py  # 新增
│   ├── research_service.py    # 新增
│   ├── forecast_service.py    # 新增
│   └── sentiment_service.py   # 新增：利好利空提取
└── jobs/
    ├── sector_scan_job.py     # 新增
    ├── news_digest_job.py     # 新增
    ├── stock_pick_job.py      # 新增
    ├── research_digest_job.py # 新增
    └── forecast_job.py        # 新增
```

### 4.2 数据流

```
数据源层                    处理层                    输出层
─────────                ─────────                ─────────
腾讯财经 ──┐
yfinance  ──┤
RSS Feeds ──┼──→ 采集/清洗  ──→  AI 分析/分级  ──→  结构化 API
东财研报  ──┤         │                │
LLM       ──┘         ↓                ↓
                   SQLite 存储    /api/brozoomout/*
```

### 4.3 LLM 调用规范

所有 AI 分析通过统一的 LLM 客户端（`app/core/llm_client.py`）：
- 模型：deepseek-chat（通过 AI Gateway）
- 超时：30 秒
- 重试：2 次，指数退避
- 缓存：相同输入 10 分钟内复用

---

## 五、数据源可用性

| 数据源 | 协议 | Docker 可达 | 需要 Key | 备注 |
|--------|------|-------------|----------|------|
| 腾讯财经 qt.gtimg.cn | HTTP | ✅ | ❌ | 纯 HTTP，Surge 不拦截 |
| yfinance | HTTPS | ⚠️ 需代理 | ❌ | Surge 增强模式下需 HTTP_PROXY |
| RSS Feeds | HTTPS | ⚠️ 需代理 | ❌ | 同上 |
| 东财研报 | HTTPS | ⚠️ 需代理 | ❌ | push2.eastmoney.com 需 Surge DIRECT |
| AI Gateway | HTTP | ✅ host.docker.internal | ❌ | 本地服务 |

---

## 六、风险与缓解

| 风险 | 缓解措施 |
|------|----------|
| 东财 API 反爬 | 控制请求频率，缓存结果，备选腾讯 |
| LLM 调用成本 | 盘后批量生成，非实时 |
| RSS 源失效 | 监控 feed 健康，自动降级 |
| 板块数据延迟 | 30 分钟足够决策，非高频交易 |
| 量化筛选误报 | 加入人工审核队列（可选） |

---

## 七、验收标准

| 指标 | 目标 |
|------|------|
| API 响应时间 | < 2 秒（P95） |
| 数据覆盖率 | 6 个市场 + 行业/概念板块 |
| 新闻分级准确率 | > 80%（人工抽检） |
| 个股推荐胜率 | > 55%（30 天回测） |
| 调度任务成功率 | > 95% |
| LLM 调用成功率 | > 99% |

---

## 八、工期估算

| 阶段 | 内容 | 工期 | 前置 |
|------|------|------|------|
| P0 | 历史深度 + 置信度 delta + 板块数据源 | 3 天 | 无 |
| P1 | 新闻分级 + 利好利空提取 | 3 天 | P0 |
| P1 | 个股推荐 + 研报摘要 | 4 天 | P0 |
| P2 | AI 预测 + 基金推荐 | 3 天 | P1 |
| P3 | Dashboard 增强 + 端点优化 | 2 天 | P1 |
| **总计** | | **15 天** | |
