     1|# BroZoomOut 金融情报服务 — 剩余工作技术方案
     2|
     3|> 版本: v3.0 | 日期: 2026-05-28 | 状态: 规划中
     4|> P0 已完成：主题修复 + 指数滚动条 + 置信度药丸
     5|
     6|---
     7|
     8|## 一、已完成 vs 待做
     9|
    10|| 阶段 | 内容 | 状态 |
    11||------|------|------|
    12|| P0 | 主题修复（dark: 前缀） | ✅ commit `fcb372a` |
    13|| P0 | MarketTicker 指数滚动条 | ✅ commit `fcb372a` |
    14|| P0 | DashboardHero 置信度药丸 | ✅ commit `fcb372a` |
    15|| **P1** | **板块数据源 + 热力图** | 📋 待做 |
    16|| **P1** | **新闻分级 + 利好利空** | 📋 待做 |
    17|| **P1** | **个股推荐** | 📋 待做 |
    18|| **P1** | **研报摘要** | 📋 待做 |
    19|| **P2** | **AI 预测** | 📋 待做 |
    20|| **P2** | **基金推荐** | 📋 待做 |
    21|| **P2** | **Dashboard 端点增强** | 📋 待做 |
    22|| **P3** | **历史趋势数据扩充** | 📋 待做 |
    23|| **P3** | **调度任务扩展** | 📋 待做 |
    24|
    25|---
    26|
    27|## 二、现有 API 清单（已实现）
    28|
    29|```
    30|GET  /api/brozoomout/summary              # 总览摘要
    31|GET  /api/brozoomout/dashboard             # 完整仪表盘
    32|GET  /api/brozoomout/markets/{market}      # 单市场详情
    33|GET  /api/brozoomout/watchlist             # 关注列表
    34|POST /api/brozoomout/watchlist             # 添加关注
    35|DEL  /api/brozoomout/watchlist/{asset_id}  # 删除关注
    36|GET  /api/brozoomout/strategy              # 策略建议
    37|POST /api/brozoomout/strategy/preview      # 策略预览
    38|POST /api/brozoomout/snapshots/refresh     # 刷新快照
    39|GET  /api/brozoomout/snapshots/{id}        # 查询快照
    40|
    41|GET  /api/v1/market/overview               # 市场概览（11 指数）
    42|GET  /api/v1/market/daily-report           # 日报
    43|GET  /api/v1/market/half-day-report        # 半日报
    44|
    45|GET  /api/v1/experience/feed               # 认知 Feed
    46|GET  /api/v1/experience/timeline           # 时间线
    47|GET  /api/v1/experience/morning-brief      # 晨报
    48|GET  /api/v1/experience/diff               # 认知差异
    49|```
    50|
    51|---
    52|
    53|## 三、待做模块详细设计
    54|
    55|### P1-1：板块动向模块
    56|
    57|**目标**：实时展示行业/概念板块涨跌排行
    58|
    59|**新增文件**：
    60|```
    61|app/data/providers/tencent_finance.py   # 腾讯财经 HTTP 采集
    62|app/services/sector_service.py          # 板块数据服务
    63|app/api/v1/sectors.py                   # 板块 API
    64|app/jobs/sector_scan_job.py             # 定时扫描任务
    65|```
    66|
    **数据源**：新浪 `newFLJK` API（已验证可用）
    - 概念板块：`https://vip.stock.finance.sina.com.cn/quotes_service/api/json_v2.php/Market_Service.getHQNodeStockCount?node=new_FLJK`
    - 返回 163 个概念板块，含涨跌幅、领涨股
    - 无反爬限制，Docker 直达
    - **替代东财被封的 push2.eastmoney.com/clist**
    70|
    71|**API 设计**：
    72|```
    73|GET /api/brozoomout/sectors
    74|Query:
    75|  type      string  industry|concept  默认 industry
    76|  limit     int     返回数量          默认 15
    77|  market    string  cn|all            默认 cn
    78|
    79|Response:
    80|{
    81|  ok: true,
    82|  data: {
    83|    type: "industry",
    84|    updated_at: "2026-05-28T15:30:00Z",
    85|    sectors: [
    86|      {
    87|        name: "银行",
    88|        change_pct: 2.15,
    89|        volume: 1234567890,
    90|        leading_stock: "招商银行",
    91|        leading_stock_change: 3.2,
    92|        turnover_rate: 0.85
    93|      }
    94|    ]
    95|  }
    96|}
    97|```
    98|
    99|**调度**：交易时段（9:30-15:00）每 30 分钟
   100|
   101|**注意**：
   102|- 腾讯 API 返回 GBK 编码，需 `decode('gbk')`
   103|- 请求间隔 ≥ 1 秒，避免限流
   104|- 缓存 5 分钟，避免重复请求
   105|
   106|---
   107|
   108|### P1-2：新闻分级模块
   109|
   110|**目标**：聚合新闻 + AI 分级 + 利好利空结构化提取
   111|
   112|**新增文件**：
   113|```
   114|app/data/providers/rss_provider.py      # 扩展现有（新增源）
   115|app/services/news_service.py            # 新闻服务
   116|app/services/sentiment_service.py       # 利好利空提取
   117|app/api/v1/news.py                      # 新闻 API
   118|app/jobs/news_digest_job.py             # 定时聚合任务
   119|```
   120|
   121|**数据源扩展**：
   122|- 现有：Google Finance RSS、CNBC、FT中文
   123|- 新增：华尔街见闻 RSS、财联社电报 API
   124|
   125|**AI 分级 Prompt**：
   126|```
   127|你是一个金融新闻分析师。请对以下新闻进行分级：
   128|
   129|新闻标题：{title}
   130|新闻内容：{content}
   131|
   132|请返回 JSON：
   133|{
   134|  "importance": "major|notable|general",
   135|  "sentiment_events": [
   136|    {
   137|      "event": "事件描述",
   138|      "type": "bullish|bearish|neutral",
   139|      "impact": "high|medium|low",
   140|      "affected_sectors": ["银行", "地产"]
   141|    }
   142|  ],
   143|  "ai_summary": "一句话总结"
   144|}
   145|```
   146|
   147|**API 设计**：
   148|```
   149|GET /api/brozoomout/news
   150|Query:
   151|  days         int     天数            默认 1
   152|  importance   string  major|notable|all  默认 all
   153|  limit        int     返回数量        默认 20
   154|
   155|Response:
   156|{
   157|  ok: true,
   158|  data: {
   159|    items: [
   160|      {
   161|        id: "news_20260528_001",
   162|        title: "央行宣布降准 50 个基点",
   163|        source: "wallstreetcn",
   164|        url: "https://...",
   165|        published_at: "2026-05-28T10:00:00Z",
   166|        importance: "major",
   167|        sentiment_events: [
   168|          {
   169|            event: "央行降准 50bp",
   170|            type: "bullish",
   171|            impact: "high",
   172|            affected_sectors: ["银行", "房地产"]
   173|          }
   174|        ],
   175|        ai_summary: "央行降准 50bp，释放约 1 万亿流动性，利好金融和地产"
   176|      }
   177|    ],
   178|    stats: {
   179|      total: 45,
   180|      major: 3,
   181|      notable: 12,
   182|      bullish: 5,
   183|      bearish: 3
   184|    }
   185|  }
   186|}
   187|```
   188|
   189|**调度**：每 1 小时
   190|
   191|---
   192|
   193|### P1-3：个股推荐模块
   194|
   195|**目标**：基于量化筛选 + AI 分析，推荐值得关注的个股
   196|
   197|**新增文件**：
   198|```
   199|app/data/providers/stock_screener.py    # 量化筛选器
   200|app/services/stock_pick_service.py      # 个股推荐服务
   201|app/api/v1/stock_picks.py               # 个股推荐 API
   202|app/jobs/stock_pick_job.py              # 每日盘前扫描
   203|```
   204|
   205|**筛选逻辑**：
   206|1. **量价异动**：成交量 > 5 日均量 2 倍
   207|2. **技术指标**：RSI < 30（超卖）或 RSI > 70（超买）
   208|3. **MACD 金叉/死叉**
   209|4. **板块联动**：热门板块龙头股
   210|5. **AI 综合评估**：将以上指标送 LLM 生成推荐理由
   211|
   212|**API 设计**：
   213|```
   214|GET /api/brozoomout/stock-picks
   215|Query:
   216|  market   string  cn|hk|us|kr|all  默认 all
   217|  signal   string  buy|sell|watch|all  默认 all
   218|  limit    int     返回数量          默认 10
   219|
   220|Response:
   221|{
   222|  ok: true,
   223|  data: {
   224|    generated_at: "2026-05-28T09:00:00Z",
   225|    picks: [
   226|      {
   227|        code: "600036",
   228|        name: "招商银行",
   229|        market: "cn_stock",
   230|        signal: "buy",
   231|        signal_strength: "medium",
   232|        current_price: 38.50,
   233|        target_price: 42.00,
   234|        stop_loss: 36.00,
   235|        upside_pct: 9.1,
   236|        reason: "银行板块领涨，RSI 32 超卖反弹，MACD 金叉，基本面稳健",
   237|        metrics: {
   238|          rsi: 32,
   239|          macd_signal: "golden_cross",
   240|          volume_ratio: 2.1,
   241|          pe_ratio: 6.8
   242|        },
   243|        related_news: ["央行降准利好银行板块"]
   244|      }
   245|    ]
   246|  }
   247|}
   248|```
   249|
   250|**调度**：每日盘前 9:00
   251|
   252|---
   253|
   254|### P1-4：研报摘要模块
   255|
   256|**目标**：抓取券商研报，AI 生成一句话总结
   257|
   258|**新增文件**：
   259|```
   260|app/data/providers/eastmoney_research.py  # 东财研报 API
   261|app/services/research_service.py          # 研报服务
   262|app/api/v1/research.py                    # 研报 API
   263|app/jobs/research_digest_job.py           # 每日摘要任务
   264|```
   265|
   266|**数据源**：东方财富研报列表 API
   267|
   268|**API 设计**：
   269|```
   270|GET /api/brozoomout/research
   271|Query:
   272|  days    int     天数      默认 1
   273|  limit   int     数量      默认 10
   274|
   275|Response:
   276|{
   277|  ok: true,
   278|  data: {
   279|    reports: [
   280|      {
   281|        id: "rep_20260528_001",
   282|        title: "招商银行(600036)深度报告：零售之王再出发",
   283|        broker: "中金公司",
   284|        analyst: "张三",
   285|        target_price: 45.0,
   286|        rating: "推荐",
   287|        current_price: 38.5,
   288|        upside_pct: 16.9,
   289|        published_at: "2026-05-28",
   290|        ai_summary: "中金看好招行零售转型，目标价 45 元，对应 12x PE，维持推荐",
   291|        related_stock: "600036",
   292|        related_sectors: ["银行"]
   293|      }
   294|    ]
   295|  }
   296|}
   297|```
   298|
   299|**调度**：每日 10:00
   300|
   301|---
   302|
   303|### P2-1：AI 预测模块
   304|
   305|**目标**：基于当前状态 + 历史数据，生成未来推演
   306|
   307|**新增文件**：
   308|```
   309|app/services/forecast_service.py   # AI 预测服务
   310|app/api/v1/forecast.py             # 预测 API
   311|app/jobs/forecast_job.py           # 每日收盘后生成
   312|```
   313|
   314|**实现逻辑**：
   315|1. 收集当前 snapshot（risk_mode、confidence、indices）
   316|2. 收集 30 天历史 snapshot 趋势
   317|3. 构建 prompt 送 LLM
   318|4. 结构化输出三场景预测
   319|
   320|**LLM Prompt**：
   321|```
   322|你是一个金融市场分析师。基于以下数据，生成未来 {horizon} 的市场预测。
   323|
   324|当前状态：
   325|- 风险模式：{risk_mode}
   326|- 置信度：{confidence}%
   327|- 波动率：{volatility}
   328|- 关键指数：{indices_summary}
   329|
   330|历史趋势（近 30 天）：
   331|{history_summary}
   332|
   333|请返回 JSON：
   334|{
   335|  "prediction": "一段话预测",
   336|  "confidence": 65,
   337|  "key_factors": ["因素1", "因素2"],
   338|  "scenarios": [
   339|    {"name": "牛市", "probability": 25, "description": "...", "impact": "..."},
   340|    {"name": "基准", "probability": 50, "description": "...", "impact": "..."},
   341|    {"name": "熊市", "probability": 25, "description": "...", "impact": "..."}
   342|  ]
   343|}
   344|```
   345|
   346|**API 设计**：
   347|```
   348|GET /api/brozoomout/forecast
   349|Query:
   350|  horizon   string  1w|1m  默认 1w
   351|
   352|Response:
   353|{
   354|  ok: true,
   355|  data: {
   356|    horizon: "1w",
   357|    generated_at: "2026-05-28T15:30:00Z",
   358|    prediction: "下周市场大概率维持震荡，关注美联储议息会议结果",
   359|    confidence: 65,
   360|    key_factors: ["美联储议息会议", "A 股量能萎缩", "北向资金流向"],
   361|    scenarios: [
   362|      {
   363|        name: "牛市",
   364|        probability: 25,
   365|        description: "美联储鸽派 + 北向大幅流入 + 政策利好",
   366|        impact: "risk_mode → risk_on, confidence > 70"
   367|      },
   368|      {
   369|        name: "基准",
   370|        probability: 50,
   371|        description: "维持现状，窄幅震荡",
   372|        impact: "risk_mode → neutral, confidence 40-60"
   373|      },
   374|      {
   375|        name": "熊市",
   376|        probability: 25,
   377|        description: "美联储鹰派 + 地缘风险升级",
   378|        impact: "risk_mode → risk_off, confidence < 30"
   379|      }
   380|    ]
   381|  }
   382|}
   383|```
   384|
   385|**调度**：每日收盘后 15:30
   386|
   387|---
   388|
   389|### P2-2：基金推荐模块
   390|
   391|**目标**：同类基金 Top N 推荐
   392|
   393|**新增文件**：
   394|```
   395|app/data/providers/fund_provider.py    # 基金数据采集
   396|app/services/fund_pick_service.py      # 基金推荐服务
   397|app/api/v1/fund_picks.py               # 基金推荐 API
   398|app/jobs/fund_pick_job.py              # 每日更新
   399|```
   400|
   401|**API 设计**：
   402|```
   403|GET /api/brozoomout/fund-picks
   404|Query:
   405|  type    string  equity|bond|hybrid|index|all  默认 all
   406|  limit   int     数量                          默认 10
   407|
   408|Response:
   409|{
   410|  ok: true,
   411|  data: {
   412|    generated_at: "2026-05-28T09:00:00Z",
   413|    funds: [
   414|      {
   415|        code: "161725",
   416|        name: "招商中证白酒指数",
   417|        type: "index",
   418|        return_1m: 5.2,
   419|        return_3m: 12.8,
   420|        return_6m: -3.5,
   421|        risk_level: "medium",
   422|        aum: "150 亿",
   423|        manager: "侯昊"
   424|      }
   425|    ]
   426|  }
   427|}
   428|```
   429|
   430|**调度**：每日 9:00
   431|
   432|---
   433|
   434|### P2-3：Dashboard 端点增强
   435|
   436|**修改文件**：`app/api/v1/bro_zoomout.py`（get_dashboard 函数）
   437|
   438|**在现有响应中增加字段**：
   439|
   440|```json
   441|{
   442|  "data": {
   443|    "...existing fields...",
   444|
   445|    "indices": [
   446|      {"name": "上证指数", "symbol": "000001", "price": 4093.73, "change_pct": -1.25}
   447|    ],
   448|
   449|    "sector_movers": {
   450|      "top_gainers": [{"name": "银行", "change_pct": 2.15}],
   451|      "top_losers": [{"name": "半导体", "change_pct": -3.2}]
   452|    },
   453|
   454|    "news_highlights": [
   455|      {"title": "央行降准", "importance": "major", "sentiment": "bullish"}
   456|    ],
   457|
   458|    "confidence_delta": -2,
   459|    "risk_mode_changed": false,
   460|    "market_count_by_sentiment": {
   461|      "bullish": 2,
   462|      "neutral": 3,
   463|      "bearish": 1
   464|    }
   465|  }
   466|}
   467|```
   468|
   469|---
   470|
   471|### P3-1：历史快照扩充
   472|
   473|**修改文件**：`app/api/v1/bro_zoomout.py`（archive 相关）
   474|
   475|**改动**：
   476|- 当前只返回 1-3 条快照
   477|- 支持 `?days=30` 参数，返回最多 30 天历史
   478|- 每条快照增加 `market_summary`（当日主要指数涨跌）
   479|
   480|---
   481|
   482|### P3-2：调度任务扩展
   483|
   484|| 任务 | 频率 | 文件 | 新增依赖 |
   485||------|------|------|----------|
   486|| `sector_scan_job` | 每 30 分钟（交易时段） | `app/jobs/sector_scan_job.py` | 腾讯 API |
   487|| `news_digest_job` | 每 1 小时 | `app/jobs/news_digest_job.py` | RSS + LLM |
   488|| `stock_pick_job` | 每日 9:00 | `app/jobs/stock_pick_job.py` | yfinance + LLM |
   489|| `research_digest_job` | 每日 10:00 | `app/jobs/research_digest_job.py` | 东财 API + LLM |
   490|| `forecast_job` | 每日 15:30 | `app/jobs/forecast_job.py` | LLM + 历史数据 |
   491|| `fund_pick_job` | 每日 9:00 | `app/jobs/fund_pick_job.py` | 基金数据源 |
   492|
   493|---
   494|
   495|## 四、新增模块结构
   496|
   497|```
   498|app/
   499|├── api/v1/
   500|│   ├── sectors.py              # 🆕 板块 API
   501|