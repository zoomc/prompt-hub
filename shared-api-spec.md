# Shared API Specification — BroZoomOut & ZoomLab

> Version: 2.0 | Date: 2026-05-28
> 目标：定义两个项目共用的所有 API 端点规范，覆盖同花顺/东方财富 70-80% 功能。
> 与 ZoomLab v5.0 对齐，支持 6 大 Tab 页、30+ 页面、Dashboard 9 大模块。

---

## 目录

1. [通用约定](#1-通用约定)
2. [市场数据 API](#2-市场数据-api)
3. [资金流向 API](#3-资金流向-api)
4. [研报与新闻 API](#4-研报与新闻-api)
5. [AI 分析 API](#5-ai-分析-api)
6. [用户数据 API](#6-用户数据-api)
7. [板块热力图 API](#7-板块热力图-api)
8. [K 线与技术指标 API](#8-k-线与技术指标-api)
9. [宏观经济 API](#9-宏观经济-api)
10. [Dashboard API](#10-dashboard-api)
11. [A股/港股扩展 API](#11-a股港股扩展-api)
12. [美股扩展 API](#12-美股扩展-api)
13. [韩国股票 API](#13-韩国股票-api)
14. [基金扩展 API](#14-基金扩展-api)
15. [AI 分析扩展 API](#15-ai-分析扩展-api)
16. [附录 A: 通用类型定义](#附录-a-通用类型定义)
17. [附录 B: 限流状态码](#附录-b-限流状态码)
18. [附录 C: 数据降级策略](#附录-c-数据降级策略)

---

## 1. 通用约定

### 1.1 基础 URL

| 环境 | BroZoomOut (Python) | ZoomLab Go Backend |
|------|---------------------|---------------------|
| 本地 | `http://localhost:8000` | `http://localhost:8080` |
| 生产 | `https://api.zoomlab.cc` | `https://api.zoomlab.cc` |

### 1.2 认证

```http
Authorization: Bearer <jwt-token>
```

- ZoomLab Go 后端使用 Google OAuth + JWT
- BroZoomOut 直接暴露内部端点（仅限内网）

### 1.3 统一响应格式

#### 成功响应

```json
{
  "ok": true,
  "empty": false,
  "data": { ... },
  "meta": {
    "request_id": "req_A1B2C3D4E5F6",
    "snapshot_id": "snap_2026-05-28T15:30:00Z",
    "generated_at": "2026-05-28T15:30:00Z",
    "source_quality": "computed",
    "freshness": "fresh",
    "locale": "zh-CN"
  }
}
```

#### 空响应

```json
{
  "ok": true,
  "empty": true,
  "data": { "total": 0, "items": [] },
  "meta": {
    "request_id": "req_A1B2C3D4E5F6",
    "empty_reason": "NO_ITEMS"
  }
}
```

#### 错误响应（RFC 7807 Problem Details）

```json
{
  "type": "https://brozoomout/errors/brozoomout-no-snapshot",
  "title": "无可用快照",
  "status": 503,
  "detail": "上游数据源不可用且无历史快照可回退。",
  "instance": "/api/brozoomout/dashboard",
  "error_code": "BROZOOMOUT_NO_SNAPSHOT",
  "retryable": true
}
```

### 1.4 缓存策略

| 数据类型 | Cache-Control | 说明 |
|---------|---------------|------|
| 实时行情 | `no-cache` | 每次请求从上游获取 |
| 板块数据 | `max-age=300` | 5 分钟缓存 |
| 研报/新闻 | `max-age=600` | 10 分钟缓存 |
| AI 分析结果 | `max-age=1800` | 30 分钟缓存 |
| 历史快照 | `max-age=3600` | 1 小时缓存 |
| 宏观数据 | `max-age=3600` | 1 小时缓存 |
| Dashboard AI | `max-age=600` | 10 分钟缓存 |
| 股东/分红/评级 | `max-age=3600` | 1 小时缓存 |
| 融资融券/期权 | `max-age=60` | 1 分钟缓存 |
| 大宗交易/做空 | `max-age=120` | 2 分钟缓存 |

### 1.5 限流策略

| 端点类型 | 限流 | 突发 |
|---------|------|------|
| 行情查询 | 60 req/min | 10 req/s |
| AI 分析 | 10 req/min | 2 req/s |
| AI 预测/推荐 | 10 req/min | 2 req/s |
| AI 组合优化/压力测试 | 5 req/min | 1 req/s |
| 写操作 | 30 req/min | 5 req/s |
| 报告查询 | 120 req/min | 20 req/s |
| Dashboard 查询 | 30 req/min | 5 req/s |
| 板块/基金/股东查询 | 30 req/min | 5 req/s |

### 1.6 错误码体系

| 前缀 | 含义 |
|------|------|
| `BROZOOMOUT_*` | BroZoomOut 核心错误 |
| `MARKET_*` | 行情数据错误 |
| `AI_*` | AI 分析错误 |
| `AI_PREDICTION_*` | AI 预测错误 |
| `AI_RISK_*` | AI 风险提示错误 |
| `AI_OPPORTUNITY_*` | AI 机会信号错误 |
| `AI_RECOMMENDATION_*` | AI 推荐错误 |
| `AI_DECISION_*` | AI 决策错误 |
| `AI_MULTI_PREDICTION_*` | AI 多时间维度预测错误 |
| `AI_ANALYSIS_*` | AI 综合分析错误 |
| `AI_OPTIMIZATION_*` | AI 组合优化错误 |
| `AI_STRESS_TEST_*` | AI 压力测试错误 |
| `SECTOR_*` | 板块数据错误 |
| `SHAREHOLDER_*` | 股东信息错误 |
| `DIVIDEND_*` | 分红送转错误 |
| `MARGIN_*` | 融资融券错误 |
| `BLOCK_TRADE_*` | 大宗交易错误 |
| `INSTITUTIONAL_*` | 机构持仓错误 |
| `ANALYST_*` | 分析师评级错误 |
| `OPTIONS_*` | 期权链错误 |
| `KR_*` | 韩国股票错误 |
| `FUND_*` | 基金错误 |
| `NEWS_*` | 新闻错误 |
| `USER_*` | 用户数据错误 |
| `RATE_LIMIT_*` | 限流错误 |

---

## 2. 市场数据 API

### 2.1 指数行情 — 腾讯 `qt.gtimg.cn`

#### 端点

```
GET /api/v1/market/indices
```

#### 数据源

- **URL**: `http://qt.gtimg.cn/q={codes}`
- **编码**: UTF-8
- **分隔符**: `~`

#### 支持的指数代码

| 代码 | 市场 | 名称 |
|------|------|------|
| `sh000001` | A股 | 上证指数 |
| `sz399001` | A股 | 深证成指 |
| `sz399006` | A股 | 创业板指 |
| `sh000300` | A股 | 沪深300 |
| `sh000688` | A股 | 科创50 |

#### 响应格式

```json
{
  "data": [
    {
      "market": "A股",
      "symbol": "sh000001",
      "name": "上证指数",
      "price": 3356.72,
      "change_percent": 1.23,
      "volume": 325467890000,
      "snapshot_time": "2026-05-28T15:30:00+08:00"
    }
  ],
  "status": "ok",
  "success_count": 5,
  "failed_count": 0,
  "duration": 0.342
}
```

#### 字段映射（腾讯格式）

```
[0]=market_code [1]=name [2]=code [3]=price [4]=prev_close
[5]=open [6]=volume [30]=datetime [31]=change_amount
[32]=change_pct [33]=high [34]=low
```

#### 缓存

- `Cache-Control: no-cache`（实时数据）
- 内存缓存 TTL: 5s（防止频繁请求）

---

### 2.2 个股行情 — 新浪 `hq.sinajs.cn`

#### 端点

```
GET /api/v1/market/quote/{symbol}
```

#### 参数

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `symbol` | string | 是 | 股票代码，如 `sh600519`、`hkHSTECH` |

#### 数据源

- **URL**: `http://hq.sinajs.cn/list={symbols}`
- **编码**: GBK
- **Referer**: `http://finance.sina.com.cn`

#### 响应格式

```json
{
  "data": {
    "market": "港股",
    "symbol": "^HSTECH",
    "name": "恒生科技指数",
    "price": 4523.18,
    "change_percent": -2.34,
    "volume": 1234567890,
    "snapshot_time": "2026-05-28T15:30:00+08:00"
  }
}
```

#### 字段映射（新浪格式）

```
[1]=name [6]=price [8]=change_pct [12]=volume
MIN_FIELDS=13
```

---

### 2.3 K 线数据 — yfinance

#### 端点

```
GET /api/v1/market/kline/{symbol}
```

#### 参数

| 参数 | 类型 | 必填 | 默认值 | 说明 |
|------|------|------|--------|------|
| `symbol` | string | 是 | - | 股票/指数代码 |
| `period` | string | 否 | `1mo` | `1d`/`5d`/`1mo`/`3mo`/`6mo`/`1y`/`2y`/`5y`/`max` |
| `interval` | string | 否 | `1d` | `1m`/`5m`/`15m`/`30m`/`1h`/`1d`/`1wk`/`1mo` |

#### 数据源

- **库**: `yfinance`
- **方法**: `yf.Ticker(symbol).history(period, interval)`

#### 响应格式

```json
{
  "data": {
    "symbol": "AAPL",
    "name": "Apple Inc.",
    "currency": "USD",
    "klines": [
      {
        "date": "2026-05-28",
        "open": 198.50,
        "high": 201.30,
        "low": 197.20,
        "close": 200.85,
        "volume": 52340000,
        "adj_close": 200.85
      }
    ],
    "meta": {
      "period": "1mo",
      "interval": "1d",
      "count": 22
    }
  }
}
```

#### 缓存

- `Cache-Control: max-age=300`
- 内存缓存 TTL: 5 分钟

---

### 2.4 全球指数 — yfinance

#### 端点

```
GET /api/v1/market/global-indices
```

#### 支持的指数

| 代码 | 市场 | 名称 |
|------|------|------|
| `^HSI` | 港股 | 恒生指数 |
| `^GSPC` | 美股 | S&P500 |
| `^IXIC` | 美股 | NASDAQ |
| `^DJI` | 美股 | Dow Jones |
| `^KS11` | 韩国 | KOSPI |

#### 响应格式

```json
{
  "data": [
    {
      "market": "美股",
      "symbol": "^GSPC",
      "name": "S&P500",
      "price": 5892.45,
      "change_percent": 0.87,
      "volume": 3245678900,
      "snapshot_time": "2026-05-28T15:30:00Z"
    }
  ],
  "status": "ok",
  "success_count": 5,
  "failed_count": 0,
  "duration": 1.234
}
```

---

### 2.5 宏观资产 — yfinance

#### 端点

```
GET /api/v1/market/macro-assets
```

#### 支持的资产

| 代码 | 市场 | 名称 |
|------|------|------|
| `DX-Y.NYB` | 宏观 | 美元指数 |
| `GC=F` | 宏观 | 黄金 |
| `CL=F` | 宏观 | 原油 |
| `BTC-USD` | 宏观 | BTC |

#### 响应格式

与全球指数相同。

---

### 2.6 市场总览

#### 端点

```
GET /api/v1/market/overview
```

#### 响应格式

```json
{
  "market_sentiment": "neutral",
  "risk_score": 55,
  "indices": [ ... ],
  "hot_sectors": [
    {
      "sector_name": "半导体",
      "change_percent": 3.45,
      "volume": 12345678900,
      "rank": 1,
      "snapshot_time": "2026-05-28T15:30:00+08:00"
    }
  ],
  "macro_assets": [ ... ],
  "source_health": [
    {
      "provider": "tencent_indices",
      "status": "ok",
      "success_count": 5,
      "failed_count": 0,
      "error_message": "",
      "duration": 0.342
    }
  ],
  "updated_at": "2026-05-28T15:30:00+08:00"
}
```

---

## 3. 资金流向 API

### 3.1 个股资金流向 — 东财 `push2his`

#### 端点

```
GET /api/v1/money-flow/stock/{symbol}
```

#### 参数

| 参数 | 类型 | 必填 | 默认值 | 说明 |
|------|------|------|--------|------|
| `symbol` | string | 是 | - | 股票代码（如 `000001`） |
| `days` | int | 否 | `5` | 历史天数（1-30） |

#### 数据源

- **URL**: `https://push2his.eastmoney.com/api/qt/stock/fflow/daykline/get`
- **参数**: `lmt=0&klt=101&secid={secid}&fields1=f1,f2,f3,f7&fields2=f51,f52,f53,f54,f55,f56,f57,f58,f59,f60,f61,f62,f63,f64,f65`
- **需缓存**: TTL 30 分钟

#### 响应格式

```json
{
  "data": {
    "symbol": "000001",
    "name": "平安银行",
    "flow_data": [
      {
        "date": "2026-05-28",
        "main_net_inflow": 125678900,
        "main_net_inflow_pct": 12.34,
        "retail_net_inflow": -5678900,
        "retail_net_inflow_pct": -0.56,
        "total_volume": 10234567890
      }
    ],
    "summary": {
      "main_inflow_5d": 567890123,
      "main_inflow_5d_pct": 8.76,
      "trend": "inflow"
    }
  }
}
```

### 3.2 北向资金 — 东财 `kamt.rtmin`

#### 端点

```
GET /api/v1/money-flow/northbound
```

#### 数据源

- **URL**: `https://push2his.eastmoney.com/api/qt/kamt.rtmin/get`
- **参数**: `fields1=f1,f2,f3,f4&fields2=f51,f54,f52,f58,f53,f62,f56,f57,f60,f61`
- **241 数据点**: 每分钟一个数据点（9:30-15:00）
- **需缓存**: TTL 5 分钟

#### 响应格式

```json
{
  "data": {
    "date": "2026-05-28",
    "sh_net_inflow": 2345678900,
    "sz_net_inflow": 1567890123,
    "total_net_inflow": 3913569023,
    "sh_buy": 12345678900,
    "sh_sell": 10000000000,
    "sz_buy": 8765432100,
    "sz_sell": 7197542077,
    "timeline": [
      {
        "time": "09:30",
        "sh_net": 123456789,
        "sz_net": 87654321,
        "total_net": 211111110
      }
    ],
    "summary": {
      "is_south_inflow": true,
      "inflow_level": "moderate",
      "trend": "accelerating"
    }
  }
}
```

### 3.3 主力资金流向

#### 端点

```
GET /api/v1/money-flow/main-force
```

#### 参数

| 参数 | 类型 | 必填 | 默认值 | 说明 |
|------|------|------|--------|------|
| `market` | string | 否 | `all` | `all`/`cn`/`hk` |
| `sort_by` | string | 否 | `net_inflow` | `net_inflow`/`volume`/`change_pct` |
| `limit` | int | 否 | `20` | 返回数量（1-50） |

#### 响应格式

```json
{
  "data": {
    "date": "2026-05-28",
    "market_total": {
      "main_net_inflow": 5678901234,
      "main_buy_count": 1234,
      "main_sell_count": 876,
      "net_inflow_pct": 3.45
    },
    "sectors": [
      {
        "sector_name": "半导体",
        "main_net_inflow": 1234567890,
        "main_net_inflow_pct": 5.67,
        "top_stocks": [
          { "symbol": "002049", "name": "紫光国微", "net_inflow": 56789012 }
        ]
      }
    ],
    "stocks": [
      {
        "symbol": "600519",
        "name": "贵州茅台",
        "main_net_inflow": 345678901,
        "main_net_inflow_pct": 8.90,
        "change_pct": 2.34,
        "price": 1856.78
      }
    ]
  }
}
```

---

## 4. 研报与新闻 API

### 4.1 研报摘要 — 东财 `reportapi`

#### 端点

```
GET /api/v1/research/reports
```

#### 参数

| 参数 | 类型 | 必填 | 默认值 | 说明 |
|------|------|------|--------|------|
| `type` | string | 否 | `all` | `all`/`daily`/`half_day` |
| `limit` | int | 否 | `20` | 返回数量 |

#### 数据源

- **URL**: `https://reportapi.eastmoney.com/report/list`
- **需缓存**: TTL 10 分钟

#### 响应格式

```json
{
  "data": {
    "reports": [
      {
        "id": "rpt_001",
        "title": "半导体行业周报：AI芯片需求持续增长",
        "source": "中信证券",
        "author": "张三",
        "rating": "买入",
        "target_price": 150.00,
        "summary": "AI芯片需求持续增长，看好国产替代进程。",
        "ai_summary": "中信证券看好半导体AI芯片赛道，目标价150元。",
        "published_at": "2026-05-28T10:00:00+08:00",
        "sector": "半导体",
        "sentiment": "bullish",
        "confidence": 0.85
      }
    ],
    "total": 156,
    "page": 1
  }
}
```

### 4.2 新闻聚合 — RSS

#### 端点

```
GET /api/v1/news/feed
```

#### 参数

| 参数 | 类型 | 必填 | 默认值 | 说明 |
|------|------|------|--------|------|
| `source` | string | 否 | `all` | `all`/`google_finance`/`cnbc_finance`/`ft_chinese` |
| `limit` | int | 否 | `30` | 返回数量 |

#### 数据源

| 源 | URL | 超时 |
|----|-----|------|
| Google Finance | `https://news.google.com/rss/search?q=stock+market+finance&hl=zh-CN` | 45s |
| CNBC | `https://search.cnbc.com/rs/search/combinedcms/view.xml?partnerId=wrss01&id=10000664` | 45s |
| FT Chinese | `https://www.ftchinese.com/rss/news` | 45s |

#### 响应格式

```json
{
  "data": {
    "items": [
      {
        "title": "Fed signals potential rate cut in September",
        "source": "cnbc_finance",
        "url": "https://...",
        "summary": "Federal Reserve officials signaled...",
        "published": "2026-05-28T14:30:00Z",
        "ai_importance": "major",
        "ai_sentiment": "neutral",
        "ai_summary": "美联储暗示9月可能降息，市场关注通胀数据。"
      }
    ],
    "total": 30,
    "fetched_at": "2026-05-28T15:30:00Z"
  },
  "status": "ok"
}
```

### 4.3 新闻分级（AI）

#### 端点

```
POST /api/v1/news/classify
```

#### 请求

```json
{
  "items": [
    {
      "title": "...",
      "summary": "...",
      "source": "cnbc_finance"
    }
  ]
}
```

#### 响应

```json
{
  "data": {
    "classifications": [
      {
        "title": "...",
        "importance": "major",
        "sentiment": "bullish",
        "impact_score": 85,
        "tags": ["货币政策", "美联储"],
        "summary": "一句话总结..."
      }
    ]
  }
}
```

#### AI 分级标准

| 级别 | 定义 | 典型场景 |
|------|------|---------|
| `major` | 重大影响，直接影响市场走势 | 央行政策、重大财报、地缘事件 |
| `notable` | 值得关注，可能影响特定板块 | 行业政策、公司新闻 |
| `general` | 一般信息，参考价值较低 | 市场评论、分析文章 |

### 4.4 经济日历 — Fair Economy

#### 端点

```
GET /api/v1/news/economic-calendar
```

#### 参数

| 参数 | 类型 | 必填 | 默认值 | 说明 |
|------|------|------|--------|------|
| `date_from` | string | 否 | 今天 | 开始日期 |
| `date_to` | string | 否 | 7天后 | 结束日期 |

#### 响应格式

```json
{
  "data": {
    "events": [
      {
        "date": "2026-05-28",
        "time": "20:30",
        "country": "US",
        "event": "GDP (Q1)",
        "forecast": "2.1%",
        "previous": "1.9%",
        "actual": null,
        "importance": "high",
        "impact": "bullish"
      }
    ]
  }
}
```

---

## 5. AI 分析 API

### 5.1 AI 预测 — LLM + 历史数据

#### 端点

```
POST /api/v1/ai/forecast
```

#### 请求

```json
{
  "symbol": "000001",
  "market": "cn_stock",
  "horizon": "1m",
  "context_days": 30
}
```

#### 响应

```json
{
  "data": {
    "symbol": "000001",
    "name": "平安银行",
    "forecast": {
      "bullish": {
        "probability": 0.35,
        "target_price": 15.80,
        "reasoning": "若宏观政策转向宽松，银行股有望受益于信贷扩张。"
      },
      "base": {
        "probability": 0.45,
        "target_price": 14.20,
        "reasoning": "维持当前估值水平，跟随大盘波动。"
      },
      "bearish": {
        "probability": 0.20,
        "target_price": 12.50,
        "reasoning": "若经济数据持续走弱，银行资产质量面临压力。"
      }
    },
    "confidence": 0.65,
    "key_factors": ["宏观经济政策", "信贷数据", "房地产市场"],
    "disclaimer": "AI 预测仅供参考，不构成投资建议。",
    "generated_at": "2026-05-28T15:30:00Z"
  }
}
```

### 5.2 AI 基金推荐 — yfinance 净值 + LLM 策略分析

#### 端点

```
GET /api/v1/ai/fund-recommendations
```

#### 参数

| 参数 | 类型 | 必填 | 默认值 | 说明 |
|------|------|------|--------|------|
| `risk_profile` | string | 否 | `balanced` | `conservative`/`balanced`/`aggressive` |
| `category` | string | 否 | `all` | `equity`/`bond`/`hybrid`/`index`/`qdii` |
| `limit` | int | 否 | `10` | 返回数量 |

#### 响应

```json
{
  "data": {
    "recommendations": [
      {
        "fund_code": "110011",
        "fund_name": "易方达中小盘混合",
        "category": "hybrid",
        "nav": 6.78,
        "nav_change_1d": 1.23,
        "nav_change_30d": 5.67,
        "sharpe_ratio": 1.23,
        "max_drawdown": -12.34,
        "style_drift": false,
        "ai_analysis": "该基金在成长风格中表现稳健，夏普比率优于同类。",
        "recommendation": "适合中等风险偏好投资者",
        "risk_level": "medium"
      }
    ],
    "market_context": {
      "risk_mode": "neutral",
      "volatility_level": "medium",
      "macro_bias": "neutral"
    },
    "disclaimer": "基金推荐仅供参考，不构成投资建议。",
    "generated_at": "2026-05-28T15:30:00Z"
  }
}
```

### 5.3 AI 个股分析

#### 端点

```
GET /api/v1/ai/stock-analysis/{symbol}
```

#### 参数

| 参数 | 类型 | 必填 | 默认值 | 说明 |
|------|------|------|--------|------|
| `symbol` | string | 是 | - | 股票代码 |
| `include_peers` | bool | 否 | `true` | 是否包含同行比较 |

#### 响应

```json
{
  "data": {
    "symbol": "600519",
    "name": "贵州茅台",
    "fundamentals": {
      "pe_ttm": 28.5,
      "pb": 9.8,
      "roe": 31.2,
      "revenue_growth": 15.6,
      "net_profit_growth": 18.3,
      "dividend_yield": 1.52
    },
    "technical": {
      "rsi_14": 62.3,
      "macd_signal": "bullish",
      "ma_trend": "above_20ma",
      "support": 1800,
      "resistance": 1920
    },
    "ai_summary": "茅台基本面稳健，估值合理偏高，技术面偏多。",
    "peers": [
      {
        "symbol": "000858",
        "name": "五粮液",
        "pe_ttm": 22.1,
        "valuation": "undervalued_vs茅台"
      }
    ],
    "generated_at": "2026-05-28T15:30:00Z"
  }
}
```

---

## 6. 用户数据 API

### 6.1 自选股管理

#### 获取自选股

```
GET /api/v1/user/watchlist
```

#### 添加自选股

```
POST /api/v1/user/watchlist
```

**请求**:

```json
{
  "asset_code": "000001",
  "market": "cn_stock",
  "alias": "平安银行",
  "tags": ["银行", "蓝筹"]
}
```

#### 删除自选股

```
DELETE /api/v1/user/watchlist/{asset_id}
```

#### 响应格式

```json
{
  "data": {
    "items": [
      {
        "asset_id": "wl_000001",
        "asset_code": "000001",
        "asset_name": "平安银行",
        "market": "cn_stock",
        "alias": "平安银行",
        "tags": ["银行", "蓝筹"],
        "signal": "Hold",
        "signal_strength": "medium",
        "trigger_reason": "趋势中性，建议持有",
        "risk_level": "low",
        "added_at": "2026-05-20T10:00:00+08:00"
      }
    ],
    "total": 15
  }
}
```

### 6.2 持仓管理

#### 端点

```
GET    /api/v1/user/positions
POST   /api/v1/user/positions
PUT    /api/v1/user/positions/{position_id}
DELETE /api/v1/user/positions/{position_id}
```

#### 请求/响应

```json
{
  "data": {
    "items": [
      {
        "position_id": "pos_001",
        "asset_code": "600519",
        "asset_name": "贵州茅台",
        "market": "cn_stock",
        "quantity": 100,
        "avg_cost": 1800.00,
        "current_price": 1856.78,
        "pnl": 5678.00,
        "pnl_percent": 3.16,
        "weight": 25.5
      }
    ],
    "total_pnl": 12345.67,
    "total_pnl_percent": 2.34,
    "total_value": 567890.12
  }
}
```

### 6.3 收藏与笔记

#### 端点

```
GET    /api/v1/user/favorites
POST   /api/v1/user/favorites
DELETE /api/v1/user/favorites/{favorite_id}

GET    /api/v1/user/notes
POST   /api/v1/user/notes
PUT    /api/v1/user/notes/{note_id}
DELETE /api/v1/user/notes/{note_id}
```

#### 收藏请求

```json
{
  "item_type": "report",
  "item_id": "rpt_001",
  "title": "半导体行业周报",
  "tags": ["研报", "半导体"]
}
```

#### 笔记请求

```json
{
  "asset_code": "600519",
  "title": "茅台跟踪笔记",
  "content": "关注Q2业绩发布，预计营收增长15%+",
  "tags": ["跟踪", "白酒"]
}
```

---

## 7. 板块热力图 API

### 7.1 概念板块列表 — 新浪 `newFLJK`

#### 端点

```
GET /api/v1/sectors/concepts
```

#### 数据源

- **URL**: `http://vip.stock.finance.sina.com.cn/quotes_service/api/json_v2.php/Market_Center.getHQNodeStockCount?node=new_fljk`
- **板块数量**: 163 个概念板块

#### 响应格式

```json
{
  "data": {
    "sectors": [
      {
        "code": "BK0655",
        "name": "人工智能",
        "change_percent": 3.45,
        "volume": 12345678900,
        "turnover_rate": 5.67,
        "stock_count": 156,
        "top_stocks": [
          { "symbol": "002230", "name": "科大讯飞", "change_percent": 5.67 }
        ]
      }
    ],
    "total": 163,
    "snapshot_time": "2026-05-28T15:30:00+08:00"
  }
}
```

### 7.2 板块热力图

#### 端点

```
GET /api/v1/sectors/heatmap
```

#### 参数

| 参数 | 类型 | 必填 | 默认值 | 说明 |
|------|------|------|--------|------|
| `type` | string | 否 | `concept` | `concept`/`industry` |
| `limit` | int | 否 | `50` | 返回数量 |

#### 响应格式

```json
{
  "data": {
    "heatmap": [
      {
        "name": "人工智能",
        "value": 3.45,
        "color": "#ff4444",
        "size": 85,
        "stocks_count": 156,
        "leaders": ["科大讯飞", "海康威视"]
      }
    ],
    "stats": {
      "up_count": 98,
      "down_count": 55,
      "flat_count": 10,
      "avg_change": 1.23
    },
    "snapshot_time": "2026-05-28T15:30:00+08:00"
  }
}
```

### 7.3 板块详情

#### 端点

```
GET /api/v1/sectors/{sector_code}
```

#### 响应

```json
{
  "data": {
    "code": "BK0655",
    "name": "人工智能",
    "change_percent": 3.45,
    "volume": 12345678900,
    "turnover_rate": 5.67,
    "stock_count": 156,
    "stocks": [
      {
        "symbol": "002230",
        "name": "科大讯飞",
        "price": 58.90,
        "change_percent": 5.67,
        "volume": 2345678900,
        "main_net_inflow": 123456789
      }
    ],
    "related_sectors": ["芯片", "机器人"],
    "news": [ ... ],
    "ai_analysis": "人工智能板块受政策利好驱动，短期有望继续走强。"
  }
}
```

---

## 8. K 线与技术指标 API

### 8.1 技术指标计算

#### 端点

```
GET /api/v1/technical/indicators/{symbol}
```

#### 参数

| 参数 | 类型 | 必填 | 默认值 | 说明 |
|------|------|------|--------|------|
| `indicators` | string | 否 | `ma,macd,rsi` | 逗号分隔的指标列表 |
| `period` | string | 否 | `3mo` | 历史周期 |

#### 支持的指标

| 指标 | 参数 | 说明 |
|------|------|------|
| MA | `ma_periods=5,10,20,60` | 移动平均线 |
| EMA | `ema_periods=12,26` | 指数移动平均 |
| MACD | `macd_fast=12,macd_slow=26,macd_signal=9` | MACD |
| RSI | `rsi_period=14` | 相对强弱指标 |
| Bollinger | `bb_period=20,bb_std=2` | 布林带 |
| KDJ | `kdj_period=9` | KDJ 指标 |
| Volume MA | `vol_ma_period=20` | 成交量均线 |

#### 响应

```json
{
  "data": {
    "symbol": "600519",
    "indicators": {
      "ma": {
        "ma5": 1845.23,
        "ma10": 1832.56,
        "ma20": 1810.78,
        "ma60": 1756.34
      },
      "macd": {
        "dif": 12.34,
        "dea": 8.56,
        "histogram": 3.78,
        "signal": "bullish"
      },
      "rsi": {
        "rsi14": 62.3,
        "signal": "neutral"
      },
      "bollinger": {
        "upper": 1920.45,
        "middle": 1810.78,
        "lower": 1701.11,
        "position": "middle"
      }
    },
    "generated_at": "2026-05-28T15:30:00Z"
  }
}
```

---

## 9. 宏观经济 API

### 9.1 宏观数据

#### 端点

```
GET /api/v1/macro/indicators
```

#### 参数

| 参数 | 类型 | 必填 | 默认值 | 说明 |
|------|------|------|--------|------|
| `country` | string | 否 | `CN` | `CN`/`US`/`EU`/`JP` |
| `indicators` | string | 否 | `all` | 逗号分隔 |

#### 支持的指标

| 指标 | 代码 | 数据源 |
|------|------|--------|
| CPI | `cpi` | 东财 |
| PMI | `pmi` | 东财 |
| GDP | `gdp` | 东财 |
| M2 | `m2` | 东财 |
| 社融 | `social_financing` | 东财 |
| PPI | `ppi` | 东财 |

#### 响应

```json
{
  "data": {
    "country": "CN",
    "indicators": [
      {
        "name": "CPI",
        "label": "消费者物价指数",
        "latest_value": 2.1,
        "previous_value": 1.8,
        "change": 0.3,
        "period": "2026-04",
        "unit": "%",
        "yoy_change": 0.3,
        "trend": "rising"
      },
      {
        "name": "PMI",
        "label": "制造业采购经理指数",
        "latest_value": 51.2,
        "previous_value": 50.8,
        "change": 0.4,
        "period": "2026-04",
        "unit": "",
        "threshold": 50,
        "above_threshold": true,
        "trend": "expanding"
      }
    ],
    "snapshot_time": "2026-05-28T15:30:00+08:00"
  }
}
```

### 9.2 数据源健康检查

#### 端点

```
GET /api/v1/health
```

#### 响应

```json
{
  "status": "healthy",
  "checks": {
    "database": "ok",
    "tencent_api": "ok",
    "yfinance": "ok",
    "sina_api": "ok",
    "rss_feeds": "ok",
    "ai_gateway": "ok"
  },
  "scheduler": {
    "daily_market": {
      "last_run": "2026-05-28T15:30:00Z",
      "status": "completed",
      "duration": 12.345
    },
    "half_day_market": {
      "last_run": "2026-05-28T11:30:00Z",
      "status": "completed",
      "duration": 8.765
    }
  },
  "uptime": 864000
}
```

---

## 10. Dashboard API

> 与 ZoomLab v5.0 Dashboard 9 大模块对齐

### 10.1 AI 今日预测

#### 端点

```
GET /api/v1/dashboard/ai-prediction
```

#### 响应格式

```json
{
  "data": {
    "summary": "今日市场偏多，但需关注美联储议息会议结果。",
    "confidence": 0.72,
    "scenarios": {
      "bullish": { "probability": 0.35, "target_price": 3450, "reasoning": "..." },
      "base": { "probability": 0.45, "target_price": 3380, "reasoning": "..." },
      "bearish": { "probability": 0.20, "target_price": 3280, "reasoning": "..." }
    },
    "key_factors": [
      { "factor": "宏观经济政策", "weight": 0.35, "impact": "positive" },
      { "factor": "美联储议息", "weight": 0.25, "impact": "neutral" },
      { "factor": "北向资金", "weight": 0.20, "impact": "positive" }
    ],
    "disclaimer": "AI 预测仅供参考，不构成投资建议。",
    "generated_at": "2026-05-28T09:30:00+08:00"
  }
}
```

#### 数据源

- **数据源**: LLM (OpenAI API) + 历史行情 (yfinance) + 宏观数据
- **缓存**: `max-age=600` (10 分钟)
- **限流**: 10 req/min
- **错误码**: `AI_PREDICTION_UNAVAILABLE`

---

### 10.2 AI 风险提示

#### 端点

```
GET /api/v1/dashboard/ai-risks
```

#### 响应格式

```json
{
  "data": {
    "risk_level": "medium",
    "risk_score": 55,
    "alerts": [
      {
        "id": "risk_001",
        "type": "macro",
        "severity": "high",
        "title": "美联储议息会议结果不确定",
        "description": "市场对加息预期存在分歧，可能导致波动加剧。",
        "affected_markets": ["us_stock", "cn_stock"],
        "affected_sectors": ["金融", "科技"],
        "created_at": "2026-05-28T09:00:00+08:00"
      }
    ],
    "trend": {
      "direction": "rising",
      "change_7d": 5,
      "change_30d": 12
    }
  }
}
```

#### 数据源

- **数据源**: LLM 风险评估 + 市场情绪指标 + 历史风险事件
- **缓存**: `max-age=300` (5 分钟)
- **限流**: 10 req/min
- **错误码**: `AI_RISK_UNAVAILABLE`

---

### 10.3 AI 机会信号

#### 端点

```
GET /api/v1/dashboard/ai-opportunities
```

#### 响应格式

```json
{
  "data": {
    "signals": [
      {
        "id": "opp_001",
        "type": "sector_rotation",
        "strength": "high",
        "title": "半导体板块资金持续流入",
        "description": "近5日半导体板块主力资金净流入超50亿，领涨股涨幅超8%。",
        "related_stocks": ["002049", "603986"],
        "confidence": 0.78,
        "time_horizon": "short_term",
        "created_at": "2026-05-28T09:15:00+08:00"
      }
    ],
    "performance_history": {
      "total_signals_30d": 45,
      "win_rate": 0.62,
      "avg_return": 2.3
    }
  }
}
```

#### 数据源

- **数据源**: LLM + 板块资金流向 + 技术指标
- **缓存**: `max-age=300` (5 分钟)
- **限流**: 10 req/min
- **错误码**: `AI_OPPORTUNITY_UNAVAILABLE`

---

### 10.4 AI 推荐股票

#### 端点

```
GET /api/v1/dashboard/ai-recommendations
```

#### 参数

| 参数 | 类型 | 必填 | 默认值 | 说明 |
|------|------|------|--------|------|
| `market` | string | 否 | `all` | `all`/`cn`/`hk`/`us`/`kr` |

#### 响应格式

```json
{
  "data": {
    "recommendations": [
      {
        "code": "600519",
        "name": "贵州茅台",
        "market": "cn_stock",
        "recommendation": "strong_buy",
        "confidence": 0.82,
        "target_price": 1950,
        "current_price": 1856.78,
        "upside": 5.02,
        "reasoning": "基本面稳健，估值合理，技术面偏多。",
        "risk_level": "low",
        "sector": "白酒",
        "generated_at": "2026-05-28T09:30:00+08:00"
      }
    ],
    "total": 10
  }
}
```

#### 数据源

- **数据源**: LLM + 量化筛选 + 基本面/技术面分析
- **缓存**: `max-age=300` (5 分钟)
- **限流**: 10 req/min
- **错误码**: `AI_RECOMMENDATION_UNAVAILABLE`

---

### 10.5 AI 买卖决策

#### 端点

```
GET /api/v1/dashboard/ai-decisions
```

#### 参数

| 参数 | 类型 | 必填 | 默认值 | 说明 |
|------|------|------|--------|------|
| `action` | string | 否 | `all` | `all`/`buy`/`sell`/`hold` |

#### 响应格式

```json
{
  "data": {
    "decisions": [
      {
        "id": "dec_001",
        "code": "600519",
        "name": "贵州茅台",
        "action": "buy",
        "strength": "strong",
        "entry_price": 1850,
        "stop_loss": 1780,
        "take_profit": 1980,
        "risk_reward_ratio": 1.86,
        "reasoning": "技术面突破均线压制，基本面稳健。",
        "time_horizon": "1-2周",
        "disclaimer": "仅供参考，不构成投资建议。",
        "generated_at": "2026-05-28T09:30:00+08:00"
      }
    ],
    "disclaimer": "所有决策由 AI 生成，仅供参考，不构成投资建议。投资有风险，入市需谨慎。"
  }
}
```

#### 数据源

- **数据源**: LLM + 技术指标 + 资金流向
- **缓存**: `max-age=300` (5 分钟)
- **限流**: 10 req/min
- **错误码**: `AI_DECISION_UNAVAILABLE`

---

### 10.6 近24小时新闻

#### 端点

```
GET /api/v1/dashboard/news
```

#### 参数

| 参数 | 类型 | 必填 | 默认值 | 说明 |
|------|------|------|--------|------|
| `category` | string | 否 | `all` | `all`/`macro`/`industry`/`company`/`anomaly` |
| `hours` | int | 否 | `24` | 时间范围（小时） |

#### 响应格式

```json
{
  "data": {
    "top_highlights": [
      {
        "id": "news_001",
        "title": "美联储暗示9月可能降息",
        "source": "cnbc_finance",
        "importance": "major",
        "sentiment": "bullish",
        "summary": "美联储官员暗示9月可能降息，市场关注通胀数据。",
        "affected_markets": ["us_stock"],
        "published_at": "2026-05-28T14:30:00Z"
      }
    ],
    "timeline": [
      {
        "time_group": "14:00-15:00",
        "items": [ ... ]
      }
    ],
    "stats": {
      "total": 35,
      "major": 3,
      "notable": 12,
      "general": 20
    }
  }
}
```

#### 数据源

- **数据源**: RSS (Google Finance / CNBC / FT) + AI 分级
- **缓存**: `max-age=120` (2 分钟)
- **限流**: 60 req/min
- **错误码**: `NEWS_UNAVAILABLE`

---

## 11. A股/港股扩展 API

### 11.1 板块热力图数据

#### 端点

```
GET /api/v1/cn/sectors/heatmap
```

#### 参数

| 参数 | 类型 | 必填 | 默认值 | 说明 |
|------|------|------|--------|------|
| `type` | string | 否 | `concept` | `concept`/`industry` |
| `market` | string | 否 | `cn_stock` | `cn_stock`/`hk_stock` |

#### 数据源

- **数据源**: 新浪 `newFLJK` (概念板块) / AKShare (行业板块)
- **缓存**: `max-age=300` (5 分钟)
- **限流**: 30 req/min
- **错误码**: `SECTOR_HEATMAP_UNAVAILABLE`

#### 响应格式

```json
{
  "data": {
    "sectors": [
      {
        "code": "BK0655",
        "name": "人工智能",
        "change_percent": 3.45,
        "volume": 12345678900,
        "fund_flow": 567890123,
        "stock_count": 156,
        "up_count": 120,
        "down_count": 30,
        "leading_stock": { "code": "002230", "name": "科大讯飞", "change_percent": 5.67 },
        "historical_return": { "1d": 3.45, "5d": 8.2, "1m": 15.6, "3m": 22.3 }
      }
    ],
    "stats": { "up_count": 98, "down_count": 55, "flat_count": 10 },
    "snapshot_time": "2026-05-28T15:30:00+08:00"
  }
}
```

---

### 11.2 板块内个股

#### 端点

```
GET /api/v1/cn/sectors/:code/stocks
```

#### 参数

| 参数 | 类型 | 必填 | 默认值 | 说明 |
|------|------|------|--------|------|
| `code` | string | 是 | - | 板块代码 |
| `sort_by` | string | 否 | `change_percent` | `change_percent`/`volume`/`fund_flow` |
| `limit` | int | 否 | `20` | 返回数量 |

#### 数据源

- **数据源**: 东财板块成分股 API
- **缓存**: `max-age=120` (2 分钟)
- **限流**: 30 req/min
- **错误码**: `SECTOR_STOCKS_UNAVAILABLE`

---

### 11.3 股东信息

#### 端点

```
GET /api/v1/cn/stocks/:code/shareholders
```

#### 响应格式

```json
{
  "data": {
    "code": "600519",
    "name": "贵州茅台",
    "shareholders": [
      {
        "rank": 1,
        "name": "贵州省人民政府国有资产监督管理委员会",
        "shares": 562124800,
        "percentage": 44.73,
        "change": 0,
        "type": "state"
      }
    ],
    "top10_total_percentage": 72.56,
    "change_quarter": "2026Q1"
  }
}
```

#### 数据源

- **数据源**: 东财股东研究 API
- **缓存**: `max-age=3600` (1 小时)
- **限流**: 30 req/min
- **错误码**: `SHAREHOLDER_UNAVAILABLE`

---

### 11.4 分红送转

#### 端点

```
GET /api/v1/cn/stocks/:code/dividends
```

#### 响应格式

```json
{
  "data": {
    "code": "600519",
    "dividends": [
      {
        "year": "2025",
        "dividend_per_share": 27.25,
        "bonus_shares": 0,
        "record_date": "2026-06-15",
        "payment_date": "2026-06-30",
        "dividend_yield": 1.47
      }
    ],
    "consecutive_years": 20,
    "avg_dividend_yield": 1.8
  }
}
```

#### 数据源

- **数据源**: 东财分红送转 API
- **缓存**: `max-age=3600` (1 小时)
- **限流**: 30 req/min
- **错误码**: `DIVIDEND_UNAVAILABLE`

---

### 11.5 融资融券

#### 端点

```
GET /api/v1/cn/stocks/:code/margin
```

#### 响应格式

```json
{
  "data": {
    "code": "600519",
    "margin_buy_balance": 1234567890,
    "margin_sell_balance": 567890123,
    "net_balance": 666677767,
    "trend": "increasing",
    "history": [
      { "date": "2026-05-27", "buy": 1200000000, "sell": 550000000 },
      { "date": "2026-05-28", "buy": 1234567890, "sell": 567890123 }
    ]
  }
}
```

#### 数据源

- **数据源**: 东财融资融券 API
- **缓存**: `max-age=60` (1 分钟)
- **限流**: 60 req/min
- **错误码**: `MARGIN_UNAVAILABLE`

---

### 11.6 大宗交易

#### 端点

```
GET /api/v1/cn/stocks/:code/block-trades
```

#### 响应格式

```json
{
  "data": {
    "code": "600519",
    "trades": [
      {
        "date": "2026-05-28",
        "price": 1850.00,
        "volume": 10000,
        "amount": 18500000,
        "buyer": "机构专用",
        "seller": "机构专用",
        "premium_discount": -0.36
      }
    ],
    "summary": {
      "total_trades_5d": 8,
      "total_amount_5d": 150000000,
      "net_buy": true
    }
  }
}
```

#### 数据源

- **数据源**: 东财大宗交易 API
- **缓存**: `max-age=120` (2 分钟)
- **限流**: 30 req/min
- **错误码**: `BLOCK_TRADE_UNAVAILABLE`

---

## 12. 美股扩展 API

### 12.1 机构持仓 (13F)

#### 端点

```
GET /api/v1/us/stocks/:code/institutional
```

#### 响应格式

```json
{
  "data": {
    "code": "AAPL",
    "name": "Apple Inc.",
    "holdings": [
      {
        "institution": "Berkshire Hathaway",
        "shares": 89464800,
        "percentage": 0.58,
        "change_quarterly": 2.3,
        "value_usd": 17892960000,
        "filing_date": "2026-05-15"
      }
    ],
    "top10_total_percentage": 32.5,
    "institutional_ownership": 60.2
  }
}
```

#### 数据源

- **数据源**: SEC EDGAR 13F Filing + Finnhub
- **缓存**: `max-age=3600` (1 小时)
- **限流**: 30 req/min
- **错误码**: `INSTITUTIONAL_UNAVAILABLE`

---

### 12.2 分析师评级

#### 端点

```
GET /api/v1/us/stocks/:code/analysts
```

#### 响应格式

```json
{
  "data": {
    "code": "AAPL",
    "analyst_count": 35,
    "consensus": "buy",
    "average_target": 210.50,
    "high_target": 250.00,
    "low_target": 165.00,
    "ratings": {
      "strong_buy": 12,
      "buy": 15,
      "hold": 6,
      "sell": 2,
      "strong_sell": 0
    },
    "recent_changes": [
      {
        "analyst": "Morgan Stanley",
        "from": "hold",
        "to": "overweight",
        "target_from": 195,
        "target_to": 220,
        "date": "2026-05-25"
      }
    ]
  }
}
```

#### 数据源

- **数据源**: Finnhub Analyst API + MarketBeat
- **缓存**: `max-age=3600` (1 小时)
- **限流**: 30 req/min
- **错误码**: `ANALYST_UNAVAILABLE`

---

### 12.3 期权链

#### 端点

```
GET /api/v1/us/stocks/:code/options
```

#### 参数

| 参数 | 类型 | 必填 | 默认值 | 说明 |
|------|------|------|--------|------|
| `expiry` | string | 否 | `nearest` | 到期日 |
| `type` | string | 否 | `all` | `all`/`call`/`put` |

#### 响应格式

```json
{
  "data": {
    "code": "AAPL",
    "current_price": 200.85,
    "expiry_dates": ["2026-06-20", "2026-07-18", "2026-08-15"],
    "options": [
      {
        "strike": 200,
        "expiry": "2026-06-20",
        "type": "call",
        "bid": 5.20,
        "ask": 5.35,
        "last": 5.25,
        "volume": 12345,
        "open_interest": 45678,
        "implied_volatility": 0.28,
        "delta": 0.52,
        "gamma": 0.03,
        "theta": -0.08
      }
    ]
  }
}
```

#### 数据源

- **数据源**: CBOE + Yahoo Finance Options
- **缓存**: `max-age=60` (1 分钟)
- **限流**: 60 req/min
- **错误码**: `OPTIONS_UNAVAILABLE`

---

## 13. 韩国股票 API

### 13.1 外国投资者持股

#### 端点

```
GET /api/v1/kr/stocks/:code/foreign-investors
```

#### 响应格式

```json
{
  "data": {
    "code": "005930",
    "name": "Samsung Electronics",
    "foreign_ownership": 55.2,
    "foreign_shares": 3298000000,
    "change_1d": -0.3,
    "change_5d": 1.2,
    "change_1m": -2.5,
    "trend": "decreasing",
    "history": [
      { "date": "2026-05-27", "ownership": 55.5 },
      { "date": "2026-05-28", "ownership": 55.2 }
    ]
  }
}
```

#### 数据源

- **数据源**: KRX 外国投资者持股 API
- **缓存**: `max-age=120` (2 分钟)
- **限流**: 30 req/min
- **错误码**: `KR_FOREIGN_HOLDING_UNAVAILABLE`

---

### 13.2 做空数据

#### 端点

```
GET /api/v1/kr/stocks/:code/short-selling
```

#### 响应格式

```json
{
  "data": {
    "code": "005930",
    "short_volume": 12345678,
    "short_ratio": 2.5,
    "short_balance": 6789012345,
    "change_1d": 5.6,
    "trend": "increasing",
    "history": [
      { "date": "2026-05-27", "volume": 11000000, "ratio": 2.2 },
      { "date": "2026-05-28", "volume": 12345678, "ratio": 2.5 }
    ]
  }
}
```

#### 数据源

- **数据源**: KRX 做空数据 API
- **缓存**: `max-age=60` (1 分钟)
- **限流**: 60 req/min
- **错误码**: `KR_SHORT_SELLING_UNAVAILABLE`

---

## 14. 基金扩展 API

### 14.1 基金经理信息

#### 端点

```
GET /api/v1/funds/:code/manager
```

#### 响应格式

```json
{
  "data": {
    "code": "110011",
    "managers": [
      {
        "name": "张坤",
        "title": "基金经理",
        "start_date": "2020-01-15",
        "experience": "8年",
        "managed_funds": 3,
        "total_aum": 56789012345,
        "career_return": 45.6,
        "sharpe_ratio": 1.45,
        "max_drawdown": -18.5,
        "best_performing_fund": "易方达蓝筹精选",
        "best_return": 67.8
      }
    ]
  }
}
```

#### 数据源

- **数据源**: 天天基金 基金经理 API
- **缓存**: `max-age=3600` (1 小时)
- **限流**: 30 req/min
- **错误码**: `FUND_MANAGER_UNAVAILABLE`

---

### 14.2 基金评级

#### 端点

```
GET /api/v1/funds/:code/ratings
```

#### 响应格式

```json
{
  "data": {
    "code": "110011",
    "ratings": {
      "晨星": { "overall": 4, "return": 4, "risk": 3 },
      "银河": { "overall": 5, "return": 5, "risk": 4 },
      "海通": { "overall": 4, "return": 4, "risk": 4 }
    },
    "sharpe_ratio": 1.23,
    "max_drawdown": -12.34,
    "sortino_ratio": 1.56,
    "information_ratio": 0.89
  }
}
```

#### 数据源

- **数据源**: 天天基金 + 晨星中国
- **缓存**: `max-age=3600` (1 小时)
- **限流**: 30 req/min
- **错误码**: `FUND_RATING_UNAVAILABLE`

---

### 14.3 机构持有比例

#### 端点

```
GET /api/v1/funds/:code/institutional
```

#### 响应格式

```json
{
  "data": {
    "code": "110011",
    "institutional_holding": 35.6,
    "individual_holding": 64.4,
    "top_institutions": [
      {
        "name": "中国工商银行",
        "percentage": 8.5,
        "shares": 123456789,
        "change_quarterly": 1.2
      }
    ],
    "change_quarterly": 2.5,
    "trend": "increasing"
  }
}
```

#### 数据源

- **数据源**: 天天基金 基金持有人 API
- **缓存**: `max-age=3600` (1 小时)
- **限流**: 30 req/min
- **错误码**: `FUND_INSTITUTIONAL_UNAVAILABLE`

---

## 15. AI 分析扩展 API

### 15.1 多时间维度预测

#### 端点

```
GET /api/v1/ai/prediction/multi-timeframe
```

#### 参数

| 参数 | 类型 | 必填 | 默认值 | 说明 |
|------|------|------|--------|------|
| `code` | string | 是 | - | 股票代码 |
| `market` | string | 否 | `cn_stock` | 市场 |

#### 响应格式

```json
{
  "data": {
    "code": "600519",
    "name": "贵州茅台",
    "predictions": {
      "1d": {
        "direction": "up",
        "change_percent": 1.2,
        "confidence": 0.75,
        "target_price": 1879.26,
        "reasoning": "技术面突破均线压制，短期偏多。"
      },
      "1w": {
        "direction": "up",
        "change_percent": 3.5,
        "confidence": 0.68,
        "target_price": 1921.77,
        "reasoning": "基本面稳健，市场情绪好转。"
      },
      "1m": {
        "direction": "up",
        "change_percent": 8.2,
        "confidence": 0.62,
        "target_price": 2009.03,
        "reasoning": "中长期看好白酒消费升级趋势。"
      },
      "3m": {
        "direction": "neutral",
        "change_percent": 2.5,
        "confidence": 0.55,
        "target_price": 1903.20,
        "reasoning": "需关注宏观经济和政策变化。"
      }
    }
  }
}
```

#### 数据源

- **数据源**: LLM + 历史行情 + 技术指标
- **缓存**: `max-age=600` (10 分钟)
- **限流**: 10 req/min
- **错误码**: `AI_MULTI_PREDICTION_UNAVAILABLE`

---

### 15.2 综合分析评分

#### 端点

```
GET /api/v1/ai/analysis/comprehensive
```

#### 参数

| 参数 | 类型 | 必填 | 默认值 | 说明 |
|------|------|------|--------|------|
| `code` | string | 是 | - | 股票代码 |
| `market` | string | 否 | `cn_stock` | 市场 |

#### 响应格式

```json
{
  "data": {
    "code": "600519",
    "name": "贵州茅台",
    "overall_score": 78,
    "dimensions": {
      "technical": { "score": 82, "signal": "bullish", "details": "..." },
      "fundamental": { "score": 85, "signal": "strong", "details": "..." },
      "fund_flow": { "score": 70, "signal": "neutral", "details": "..." },
      "news": { "score": 75, "signal": "positive", "details": "..." }
    },
    "radar_chart": {
      "technical": 82,
      "fundamental": 85,
      "fund_flow": 70,
      "news": 75,
      "valuation": 68
    },
    "recommendation": "buy",
    "confidence": 0.78,
    "generated_at": "2026-05-28T15:30:00+08:00"
  }
}
```

#### 数据源

- **数据源**: LLM + 技术指标 + 基本面数据 + 资金流向 + 新闻
- **缓存**: `max-age=600` (10 分钟)
- **限流**: 10 req/min
- **错误码**: `AI_ANALYSIS_UNAVAILABLE`

---

### 15.3 组合优化建议

#### 端点

```
POST /api/v1/ai/portfolio/optimize
```

#### 请求

```json
{
  "positions": [
    { "code": "600519", "market": "cn_stock", "weight": 25 },
    { "code": "000858", "market": "cn_stock", "weight": 15 },
    { "code": "601318", "market": "cn_stock", "weight": 20 }
  ],
  "risk_profile": "balanced",
  "constraints": {
    "max_single_weight": 30,
    "min_single_weight": 5,
    "target_return": null
  }
}
```

#### 响应

```json
{
  "data": {
    "current_portfolio": {
      "expected_return": 12.5,
      "volatility": 18.3,
      "sharpe_ratio": 0.68,
      "max_drawdown": -22.5
    },
    "optimized_portfolio": {
      "expected_return": 14.2,
      "volatility": 16.8,
      "sharpe_ratio": 0.85,
      "max_drawdown": -18.2
    },
    "rebalance_suggestions": [
      {
        "code": "600519",
        "name": "贵州茅台",
        "current_weight": 25,
        "suggested_weight": 30,
        "action": "increase",
        "reasoning": "基本面稳健，估值合理，建议增配。"
      }
    ],
    "efficient_frontier": [
      { "risk": 12.5, "return": 8.5 },
      { "risk": 15.0, "return": 11.2 },
      { "risk": 18.0, "return": 14.0 }
    ],
    "disclaimer": "组合优化仅供参考，不构成投资建议。"
  }
}
```

#### 数据源

- **数据源**: LLM + 历史收益率 + 协方差矩阵
- **缓存**: 无（POST 请求）
- **限流**: 5 req/min
- **错误码**: `AI_OPTIMIZATION_UNAVAILABLE`

---

### 15.4 压力测试

#### 端点

```
POST /api/v1/ai/portfolio/stress-test
```

#### 请求

```json
{
  "positions": [
    { "code": "600519", "market": "cn_stock", "weight": 25 },
    { "code": "000858", "market": "cn_stock", "weight": 15 }
  ],
  "scenarios": ["market_crash", "rate_hike", "sector_rotation", "custom"]
}
```

#### 响应

```json
{
  "data": {
    "results": {
      "market_crash": {
        "name": "市场崩盘",
        "description": "大盘下跌20%，波动率翻倍",
        "portfolio_impact": -18.5,
        "worst_stock": { "code": "000858", "impact": -22.3 },
        "recovery_days": 45,
        "var_95": -15.2
      },
      "rate_hike": {
        "name": "加息25bp",
        "description": "央行加息25个基点",
        "portfolio_impact": -5.2,
        "worst_stock": { "code": "601318", "impact": -8.5 },
        "recovery_days": 15,
        "var_95": -4.8
      }
    },
    "correlation_matrix": {
      "600519": { "000858": 0.85, "601318": 0.42 },
      "000858": { "600519": 0.85, "601318": 0.38 },
      "601318": { "600519": 0.42, "000858": 0.38 }
    },
    "disclaimer": "压力测试结果仅供参考，实际市场情况可能与模拟情景存在差异。"
  }
}
```

#### 数据源

- **数据源**: LLM + 历史收益率 + 情景模拟
- **缓存**: 无（POST 请求）
- **限流**: 5 req/min
- **错误码**: `AI_STRESS_TEST_UNAVAILABLE`

---

## 附录 A: 通用类型定义

```typescript
// 市场标识
type MarketKey = 'cn_stock' | 'hk_stock' | 'us_stock' | 'kr_stock' | 'fund' | 'fx';

// 风险模式
type RiskMode = 'risk_off' | 'neutral' | 'risk_on';

// 波动率级别
type VolatilityLevel = 'low' | 'medium' | 'high';

// 宏观偏向
type MacroBias = 'defensive' | 'neutral' | 'aggressive';

// 信号类型
type SignalType = 'Hold' | 'Reduce' | 'Add' | 'Buy' | 'Sell' | 'Watch';

// 信号强度
type SignalStrength = 'low' | 'medium' | 'high';

// 风险级别
type RiskLevel = 'low' | 'medium' | 'high';

// 新闻重要性
type NewsImportance = 'major' | 'notable' | 'general';

// 新闻情感
type NewsSentiment = 'bullish' | 'neutral' | 'bearish';

// 时间周期
type Horizon = 'short_term' | 'mid_term' | 'long_term';

// 风险暴露级别
type RiskExposureLevel = 'low' | 'low_to_medium' | 'medium' | 'medium_to_high' | 'high';
```

## 附录 B: 限流状态码

当请求被限流时，返回 HTTP 429：

```json
{
  "type": "https://brozoomout/errors/rate-limit-exceeded",
  "title": "请求过于频繁",
  "status": 429,
  "detail": "请在 30 秒后重试。",
  "instance": "/api/v1/market/overview",
  "error_code": "RATE_LIMIT_EXCEEDED",
  "retryable": true,
  "retry_after": 30
}
```

## 附录 C: 数据降级策略

当上游数据源不可用时：

1. **缓存优先**: 使用内存/Redis 缓存的最近数据
2. **快照回退**: 使用最近一次成功的快照数据
3. **部分降级**: 仅标记不可用的数据源，其余正常返回
4. **标记过期**: 设置 `freshness: "stale"` 或 `freshness: "expired"`

```json
{
  "ok": true,
  "empty": false,
  "data": { ... },
  "meta": {
    "freshness": "stale",
    "source_quality": "cached",
    "degraded_sources": ["sina_hkstech"]
  }
}
```
