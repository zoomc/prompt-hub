# Shared API Specification — BroZoomOut & ZoomLab

> Version: 1.0 | Date: 2026-05-28
> 目标：定义两个项目共用的所有 API 端点规范，覆盖同花顺/东方财富 70-80% 功能。

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

### 1.5 限流策略

| 端点类型 | 限流 | 突发 |
|---------|------|------|
| 行情查询 | 60 req/min | 10 req/s |
| AI 分析 | 10 req/min | 2 req/s |
| 写操作 | 30 req/min | 5 req/s |
| 报告查询 | 120 req/min | 20 req/s |

### 1.6 错误码体系

| 前缀 | 含义 |
|------|------|
| `BROZOOMOUT_*` | BroZoomOut 核心错误 |
| `MARKET_*` | 行情数据错误 |
| `AI_*` | AI 分析错误 |
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
