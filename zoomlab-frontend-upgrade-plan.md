# ZoomLab 金融模块前端详细升级计划

> 版本: v4.0 | 日期: 2026-05-28 | 状态: 规划中
> 范围: 6 大 Tab 页、30+ 二级页面、6 个三级详情页

---

## 目录

1. [项目概述](#一项目概述)
2. [页面架构总览](#二页面架构总览)
3. [路由设计](#三路由设计)
4. [Tab 1: Dashboard（首页/概览）](#四tab-1-dashboard首页概览)
5. [Tab 2: A股/港股（国内股票）](#五tab-2-a股港股国内股票)
6. [Tab 3: 美股（US Stocks）](#六tab-3-美股us-stocks)
7. [Tab 4: 韩国股票（Korean Stocks）](#七tab-4-韩国股票korean-stocks)
8. [Tab 5: 基金（Funds）](#八tab-5-基金funds)
9. [Tab 6: AI 分析（AI Analysis）](#九tab-6-ai-分析ai-analysis)
10. [全局组件设计](#十全局组件设计)
11. [全局状态管理](#十一全局状态管理)
12. [API 接口清单](#十二api-接口清单)
13. [设计系统与样式规范](#十三设计系统与样式规范)
14. [响应式设计规范](#十四响应式设计规范)
15. [日间/夜间模式规范](#十五日间夜间模式规范)
16. [交互与动画规范](#十六交互与动画规范)
17. [组件清单与层级关系](#十七组件清单与层级关系)
18. [验收标准](#十八验收标准)
19. [实施计划与里程碑](#十九实施计划与里程碑)

---

## 一、项目概述

### 1.1 背景

ZoomLab 是一个面向专业投资者的金融分析 Web 应用。当前已实现 BroZoomOut 详情页（单页 Dashboard 模式），需要扩展为完整的 6 Tab 金融终端界面，覆盖 A 股、港股、美股、韩国股票、基金和 AI 分析六大板块。

### 1.2 现有代码基础

**已实现的核心组件**：
- `BroZoomOutPage.tsx` — 当前的单页 Dashboard，包含 DashboardHero、MarketTicker、MarketAnalysis、WatchlistPanel、StrategyPanel、RiskHistoryPanel
- `broZoomOutClient.ts` — API 客户端，已封装 fetchSummary、fetchDashboard、fetchMarketDetail、fetchWatchlist、previewStrategy、refreshSnapshot
- `types.ts` — 完整的 TypeScript 类型定义，包含 RiskMode、MarketKey、SignalType 等 20+ 类型
- `design-tokens.css` — CSS 自定义属性，已定义完整的颜色、间距、圆角、阴影、动效变量

**现有路由结构**（`App.tsx`）：
```
/              → HomePage
/news          → NewsPage
/news/detail   → NewsDetailPage
/brozoomout    → BroZoomOutPage
/notes         → NotesPage
/settings      → SettingsPage
/status        → StatusPage (admin)
/lab           → LabPage (admin)
```

**现有 UI 组件库**（`components/ui/`）：
- Button, Input, Badge, Card, Tabs, Dialog, DropdownMenu
- Skeleton, Alert, Switch, Checkbox, Tooltip, ScrollArea
- Table, Separator, Label, Avatar, Progress, AlertDialog
- ContextMenu, Carousel, Sonner

**现有设计系统**（`components/design-system/`）：
- PillButton, Badge, SectionHeader, WorkspaceCard

### 1.3 升级目标

| 目标 | 说明 |
|------|------|
| 6 Tab 架构 | Dashboard / A股港股 / 美股 / 韩国股票 / 基金 / AI 分析 |
| 30+ 页面 | 每个 Tab 下 4-5 个二级页面 + 6 个三级详情页 |
| 专业级 UI | 参照 Bloomberg Terminal / TradingView 的信息密度 |
| 双模式 | 日间模式（白底）+ 夜间模式（深色），平滑切换 |
| 响应式 | 桌面 ≥1280px / 平板 768-1279px / 移动 ≤767px |
| 性能 | 骨架屏加载、数据缓存、自动刷新、懒加载 |

---

## 二、页面架构总览

### 2.1 三级页面架构图

```
ZoomLab App
├── PortalShell (全局外壳)
│   ├── Header (顶部导航)
│   │   ├── Logo + BrandMark
│   │   ├── Tab 导航栏 (6 Tab)
│   │   ├── 搜索框 (全局)
│   │   ├── 主题切换 (日/夜)
│   │   └── 用户菜单
│   └── <main>
│       └── <TabContent> (无刷新切换)
│           ├── Tab 1: Dashboard
│           │   ├── MarketOverview (市场概览)
│           │   ├── PortfolioSummary (持仓概览)
│           │   ├── NewsFeed (新闻流)
│           │   ├── AIInsights (AI 洞察)
│           │   └── Watchlist (自选股)
│           ├── Tab 2: A股/港股
│           │   ├── MarketHeatmap (板块热力图)
│           │   ├── SectorMovers (板块动向)
│           │   ├── StockSearch (个股搜索)
│           │   ├── StockRanking (个股排行)
│           │   └── StockDetail (个股详情) ← 三级页面
│           ├── Tab 3: 美股
│           │   ├── USMarketOverview (美股概览)
│           │   ├── USStockSearch (美股搜索)
│           │   ├── USStockRanking (美股排行)
│           │   └── USStockDetail (美股详情) ← 三级页面
│           ├── Tab 4: 韩国股票
│           │   ├── KOSPIOverview (KOSPI 概览)
│           │   ├── KoreanStockSearch (韩国股票搜索)
│           │   ├── KoreanStockRanking (韩国股票排行)
│           │   └── KoreanStockDetail (韩国股票详情) ← 三级页面
│           ├── Tab 5: 基金
│           │   ├── FundCategories (基金分类)
│           │   ├── FundRanking (基金排名)
│           │   ├── FundSearch (基金搜索)
│           │   └── FundDetail (基金详情) ← 三级页面
│           └── Tab 6: AI 分析
│               ├── MarketPrediction (市场预测)
│               ├── StockAnalysis (个股分析)
│               ├── PortfolioOptimization (组合优化)
│               └── RiskAssessment (风险评估)
└── Footer
```

### 2.2 页面数量统计

| Tab | 二级页面 | 三级页面 | 合计 |
|-----|---------|---------|------|
| Dashboard | 5 | 0 | 5 |
| A股/港股 | 4 | 1 (StockDetail) | 5 |
| 美股 | 3 | 1 (USStockDetail) | 4 |
| 韩国股票 | 3 | 1 (KoreanStockDetail) | 4 |
| 基金 | 3 | 1 (FundDetail) | 4 |
| AI 分析 | 4 | 0 | 4 |
| **合计** | **22** | **4** | **26** |

---

## 三、路由设计

### 3.1 路由表

```typescript
// 路由配置
const ROUTES = {
  // 一级：Tab 页
  dashboard: '/',
  domestic: '/domestic',       // A股/港股
  us: '/us',                   // 美股
  korea: '/korea',             // 韩国股票
  funds: '/funds',             // 基金
  ai: '/ai',                   // AI 分析

  // 二级：Tab 内页面
  dashboard_market: '/dashboard/market',
  dashboard_portfolio: '/dashboard/portfolio',
  dashboard_news: '/dashboard/news',
  dashboard_insights: '/dashboard/insights',
  dashboard_watchlist: '/dashboard/watchlist',

  domestic_heatmap: '/domestic/heatmap',
  domestic_sectors: '/domestic/sectors',
  domestic_search: '/domestic/search',
  domestic_ranking: '/domestic/ranking',

  us_overview: '/us/overview',
  us_search: '/us/search',
  us_ranking: '/us/ranking',

  korea_overview: '/korea/overview',
  korea_search: '/korea/search',
  korea_ranking: '/korea/ranking',

  funds_categories: '/funds/categories',
  funds_ranking: '/funds/ranking',
  funds_search: '/funds/search',

  ai_prediction: '/ai/prediction',
  ai_stock: '/ai/stock',
  ai_portfolio: '/ai/portfolio',
  ai_risk: '/ai/risk',

  // 三级：详情页
  stock_detail: '/domestic/stock/:code',
  us_stock_detail: '/us/stock/:code',
  korea_stock_detail: '/korea/stock/:code',
  fund_detail: '/funds/fund/:code',
} as const;
```

### 3.2 路由实现方案

采用 `react-router-dom` v6 的嵌套路由，Tab 切换使用 `<Outlet />` 实现无刷新：

```typescript
// App.tsx 路由结构
<Routes>
  <Route path="/" element={<AuthGuard><PortalShell>...</PortalShell></AuthGuard>}>
    {/* Tab 1: Dashboard */}
    <Route index element={<DashboardPage />}>
      <Route path="market" element={<MarketOverview />} />
      <Route path="portfolio" element={<PortfolioSummary />} />
      <Route path="news" element={<NewsFeed />} />
      <Route path="insights" element={<AIInsights />} />
      <Route path="watchlist" element={<WatchlistPage />} />
    </Route>

    {/* Tab 2: A股/港股 */}
    <Route path="domestic" element={<DomesticPage />}>
      <Route index element={<Navigate to="heatmap" replace />} />
      <Route path="heatmap" element={<MarketHeatmap />} />
      <Route path="sectors" element={<SectorMovers />} />
      <Route path="search" element={<StockSearch />} />
      <Route path="ranking" element={<StockRanking />} />
      <Route path="stock/:code" element={<StockDetail />} />
    </Route>

    {/* Tab 3: 美股 */}
    <Route path="us" element={<USPage />}>
      <Route index element={<Navigate to="overview" replace />} />
      <Route path="overview" element={<USMarketOverview />} />
      <Route path="search" element={<USStockSearch />} />
      <Route path="ranking" element={<USStockRanking />} />
      <Route path="stock/:code" element={<USStockDetail />} />
    </Route>

    {/* Tab 4: 韩国股票 */}
    <Route path="korea" element={<KoreaPage />}>
      <Route index element={<Navigate to="overview" replace />} />
      <Route path="overview" element={<KOSPIOverview />} />
      <Route path="search" element={<KoreanStockSearch />} />
      <Route path="ranking" element={<KoreanStockRanking />} />
      <Route path="stock/:code" element={<KoreanStockDetail />} />
    </Route>

    {/* Tab 5: 基金 */}
    <Route path="funds" element={<FundsPage />}>
      <Route index element={<Navigate to="categories" replace />} />
      <Route path="categories" element={<FundCategories />} />
      <Route path="ranking" element={<FundRanking />} />
      <Route path="search" element={<FundSearch />} />
      <Route path="fund/:code" element={<FundDetail />} />
    </Route>

    {/* Tab 6: AI 分析 */}
    <Route path="ai" element={<AIPage />}>
      <Route index element={<Navigate to="prediction" replace />} />
      <Route path="prediction" element={<MarketPrediction />} />
      <Route path="stock" element={<StockAnalysis />} />
      <Route path="portfolio" element={<PortfolioOptimization />} />
      <Route path="risk" element={<RiskAssessment />} />
    </Route>
  </Route>
</Routes>
```

### 3.3 导航状态管理

```typescript
// Tab 导航状态
interface TabState {
  activeTab: 'dashboard' | 'domestic' | 'us' | 'korea' | 'funds' | 'ai';
  subPage: string;
  history: string[];  // 导航历史栈，支持返回
}

// 导航上下文
interface NavigationContext {
  state: TabState;
  navigateTo: (path: string) => void;
  goBack: () => void;
  canGoBack: boolean;
}
```

---

## 四、Tab 1: Dashboard（首页/概览）

### 4.1 DashboardPage 主页面

**文件**：`features/finance/pages/DashboardPage.tsx`

**页面布局**：
```
┌─────────────────────────────────────────────────────────────┐
│  MarketTicker (全局指数滚动条)                                │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────────────────────────────────────────────────┐   │
│  │  MarketOverview (市场概览)                           │   │
│  │  ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐    │   │
│  │  │ 上证 │ │ 深证 │ │ 创业板│ │ 恒生 │ │ S&P  │    │   │
│  │  │ -1.2%│ │ -0.8%│ │ +0.1%│ │ -1.0%│ │ -0.1%│    │   │
│  │  └──────┘ └──────┘ └──────┘ └──────┘ └──────┘    │   │
│  │  ┌──────┐ ┌──────┐                                │   │
│  │  │NASDAQ│ │KOSPI │  情绪仪表盘                      │   │
│  │  │ -0.2%│ │ +2.2%│  [恐惧] [贪婪] 量表              │   │
│  │  └──────┘ └──────┘                                │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  ┌─────────────────────────────┬─────────────────────────┐ │
│  │  PortfolioSummary           │  NewsFeed               │ │
│  │  (持仓概览)                 │  (新闻流)               │ │
│  │  ┌───────────────────┐     │  🟢 央行降准 10:00      │ │
│  │  │ 总资产 ¥1,234,567 │     │  🔴 美联储鹰派 08:30   │ │
│  │  │ 日收益 +¥12,345   │     │  ⚪ A股缩量 昨日       │ │
│  │  │ 收益率 +1.01%     │     │  ...                   │ │
│  │  └───────────────────┘     │                         │ │
│  │  行业分布饼图               │                         │ │
│  └─────────────────────────────┴─────────────────────────┘ │
│                                                             │
│  ┌─────────────────────────────┬─────────────────────────┐ │
│  │  AIInsights                 │  Watchlist              │ │
│  │  (AI 洞察)                  │  (自选股)               │ │
│  │  今日预测: 震荡偏弱          │  600036 招商银行 +1.2% │ │
│  │  风险提示: 美联储加息        │  300750 宁德时代 -0.5% │ │
│  │  机会信号: 银行板块超卖      │  005930 三星电子 +2.1% │ │
│  └─────────────────────────────┴─────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

**组件树**：
```
DashboardPage
├── MarketTicker (已有)
├── MarketOverviewCard
│   ├── IndexCard × 7 (上证/深证/创业板/恒生/S&P/NASDAQ/KOSPI)
│   └── SentimentGauge (恐惧贪婪指数)
├── TwoColumnLayout
│   ├── LeftColumn
│   │   ├── PortfolioSummaryCard
│   │   │   ├── AssetSummary (总资产/日收益/收益率)
│   │   │   └── SectorPieChart (行业分布)
│   │   └── AIInsightsCard
│   │       ├── PredictionSummary (今日预测)
│   │       ├── RiskAlert (风险提示)
│   │       └── OpportunitySignal (机会信号)
│   └── RightColumn
│       ├── NewsFeedCard
│       │   └── NewsItem × N
│       └── WatchlistCard
│           └── WatchlistItem × N
```

### 4.2 MarketOverview 市场概览

**文件**：`features/finance/components/market/MarketOverviewCard.tsx`

**Props 设计**：
```typescript
interface MarketOverviewCardProps {
  indices: MarketIndex[];
  sentiment: MarketSentiment | null;
  loading: boolean;
  onIndexClick?: (index: MarketIndex) => void;
}

interface MarketIndex {
  market: 'cn' | 'hk' | 'us' | 'kr';
  name: string;
  symbol: string;
  price: number;
  change: number;
  changePercent: number;
  volume: number;
  updated_at: string;
}

interface MarketSentiment {
  fear_greed_index: number;  // 0-100
  label: '极度恐惧' | '恐惧' | '中性' | '贪婪' | '极度贪婪';
  volume_trend: '放量' | '缩量' | '持平';
  northbound_flow: number;  // 北向资金净流入（亿）
}
```

**数据流**：
```
GET /api/v1/market/overview
→ { indices: MarketIndex[], sentiment: MarketSentiment }
→ 缓存 60 秒
→ MarketTicker 和 MarketOverviewCard 共享数据
```

**样式规范**：
- 桌面：7 个 IndexCard 横向排列（4+3 或 7 列）
- 平板：IndexCard 两行排列（4+3）
- 移动端：IndexCard 横向滚动
- 每个 IndexCard：`card-base` 样式，`rounded-lg`，`p-md`
- 价格数字：`tabular-nums`（等宽数字），`font-bold`
- 涨跌颜色：正=`text-emerald-500`，负=`text-red-500`，零=`text-gray-400`
- 恐惧贪婪仪表盘：半圆弧形量表，0-100 渐变色

### 4.3 PortfolioSummary 持仓概览

**文件**：`features/finance/components/portfolio/PortfolioSummaryCard.tsx`

**Props 设计**：
```typescript
interface PortfolioSummaryCardProps {
  portfolio: PortfolioData | null;
  loading: boolean;
  error: string | null;
  onViewDetail?: () => void;
}

interface PortfolioData {
  total_assets: number;
  daily_pnl: number;
  daily_pnl_percent: number;
  total_pnl: number;
  total_pnl_percent: number;
  cash_balance: number;
  positions: PositionSummary[];
  sector_distribution: SectorDistribution[];
  updated_at: string;
}

interface PositionSummary {
  code: string;
  name: string;
  market: MarketKey;
  quantity: number;
  current_price: number;
  cost_price: number;
  market_value: number;
  pnl: number;
  pnl_percent: number;
}

interface SectorDistribution {
  sector: string;
  percentage: number;
  color: string;
}
```

**数据流**：
```
GET /api/v1/portfolio/summary
→ PortfolioData
→ 缓存 30 秒（持仓数据需要实时性）
→ 未登录时显示引导提示
```

**样式规范**：
- 总资产大数字：`text-3xl font-bold tabular-nums`
- 日收益：正=`text-emerald-500`，负=`text-red-500`，前缀 +/- 符号
- 行业分布饼图：使用 Recharts PieChart，颜色映射 8 个行业
- 持仓列表：最多显示 Top 5，点击"查看更多"展开

### 4.4 NewsFeed 新闻流

**文件**：`features/finance/components/news/NewsFeedCard.tsx`

**Props 设计**：
```typescript
interface NewsFeedCardProps {
  items: NewsItem[];
  loading: boolean;
  error: string | null;
  filter: NewsFilter;
  onFilterChange: (filter: NewsFilter) => void;
  onItemClick: (item: NewsItem) => void;
  onLoadMore?: () => void;
}

type NewsFilter = 'all' | 'major' | 'positive' | 'negative';

interface NewsItem {
  id: string;
  title: string;
  source: string;
  published_at: string;
  importance: 'major' | 'notable' | 'general';
  sentiment: 'positive' | 'negative' | 'neutral';
  sentiment_score: number;  // -1 到 1
  ai_summary: string;
  related_stocks: string[];
  impact_sectors: string[];
  url: string;
}
```

**数据流**：
```
GET /api/v1/news?importance=all&limit=20&offset=0
→ { items: NewsItem[], total: number, has_more: boolean }
→ 缓存 120 秒
→ 支持分页加载（滚动到底自动加载）
```

**样式规范**：
- 时间线布局：左侧竖线 + 圆点标记
- 利好新闻：左侧绿色边框 + 🟢 圆点
- 利空新闻：左侧红色边框 + 🔴 圆点
- 中性新闻：左侧灰色边框 + ⚪ 圆点
- 重大新闻：左侧黄色竖线 + 加粗标题
- 顶部筛选：pill-tab 样式按钮组
- 每条新闻：标题 + 时间 + AI 摘要 + 影响板块标签

### 4.5 AIInsights AI 洞察

**文件**：`features/finance/components/ai/AIInsightsCard.tsx`

**Props 设计**：
```typescript
interface AIInsightsCardProps {
  insights: AIInsights | null;
  loading: boolean;
  error: string | null;
}

interface AIInsights {
  prediction: {
    summary: string;
    confidence: number;
    horizon: 'short_term' | 'mid_term';
  };
  risk_alerts: RiskAlert[];
  opportunities: OpportunitySignal[];
  generated_at: string;
}

interface RiskAlert {
  id: string;
  title: string;
  severity: 'high' | 'medium' | 'low';
  description: string;
  related_markets: MarketKey[];
}

interface OpportunitySignal {
  id: string;
  title: string;
  strength: 'strong' | 'moderate' | 'weak';
  description: string;
  related_stocks: string[];
}
```

**数据流**：
```
GET /api/v1/ai/insights
→ AIInsights
→ 缓存 300 秒（AI 分析更新频率较低）
```

**样式规范**：
- 预测摘要：大字体引言样式，居中
- 置信度：药丸标签，颜色映射（高=绿，中=黄，低=红）
- 风险提示列表：红色左边框卡片
- 机会信号列表：绿色左边框卡片
- 生成时间：底部小字灰色

### 4.6 Watchlist 自选股

**文件**：`features/finance/components/watchlist/WatchlistCard.tsx`

**Props 设计**：
```typescript
interface WatchlistCardProps {
  items: WatchlistItem[];
  loading: boolean;
  error: string | null;
  onAdd: (code: string, market: MarketKey) => void;
  onRemove: (itemId: string) => void;
  onItemClick: (item: WatchlistItem) => void;
  onRefresh: () => void;
}

interface WatchlistItem {
  id: string;
  code: string;
  name: string;
  market: MarketKey;
  current_price: number;
  change: number;
  change_percent: number;
  signal: SignalType;
  signal_strength: SignalStrength;
  added_at: string;
}
```

**数据流**：
```
GET /api/brozoomout/watchlist
→ { total: number, items: WatchlistItem[] }
→ 缓存 30 秒
→ 支持 POST 添加、DELETE 删除
```

**样式规范**：
- 卡片列表，每只股票一行
- 信号标签颜色：BUY=绿，SELL=红，HOLD=灰，WATCH=黄
- 涨跌颜色：正=绿，负=红
- 添加按钮：底部 pill 按钮
- 空状态：引导提示 + 添加按钮

---

## 五、Tab 2: A股/港股（国内股票）

### 5.1 DomesticPage 主页面

**文件**：`features/finance/pages/DomesticPage.tsx`

**页面布局**：
```
┌─────────────────────────────────────────────────────────────┐
│  二级导航: [热力图] [板块动向] [个股搜索] [个股排行]           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  MarketHeatmap (板块热力图)                          │   │
│  │  163 个概念板块的方块热力图                           │   │
│  │  红涨绿跌颜色编码                                    │   │
│  │  点击进入板块详情                                     │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  ┌─────────────────────────────┬─────────────────────────┐ │
│  │  SectorMovers               │  StockSearch            │ │
│  │  (板块动向)                 │  (个股搜索)             │ │
│  │  涨幅榜/跌幅榜              │  搜索框 + 结果列表      │ │
│  │  领涨股/领跌股              │  快捷添加自选股          │ │
│  └─────────────────────────────┴─────────────────────────┘ │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  StockRanking (个股排行)                             │   │
│  │  涨幅榜 | 跌幅榜 | 换手率榜 | 成交额榜              │   │
│  │  筛选条件: 行业 / 市值 / PE                          │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### 5.2 MarketHeatmap 板块热力图

**文件**：`features/finance/components/heatmap/MarketHeatmap.tsx`

**Props 设计**：
```typescript
interface MarketHeatmapProps {
  sectors: HeatmapSector[];
  loading: boolean;
  error: string | null;
  onSectorClick: (sector: HeatmapSector) => void;
  viewMode: 'industry' | 'concept';
  onViewModeChange: (mode: 'industry' | 'concept') => void;
}

interface HeatmapSector {
  id: string;
  name: string;
  change_percent: number;
  volume: number;
  leading_stock: string;
  leading_stock_change: number;
  stock_count: number;
  market: 'cn_stock' | 'hk_stock';
}

interface HeatmapColorConfig {
  positiveColor: string;    // 涨的颜色
  negativeColor: string;    // 跌的颜色
  neutralColor: string;     // 平的颜色
  maxPercent: number;       // 最大涨跌幅（用于映射）
}
```

**数据流**：
```
GET /api/v1/market/sectors?type=industry&market=cn_stock
→ { sectors: HeatmapSector[] }
→ 缓存 120 秒
→ 切换 industry/concept 时重新请求
```

**颜色映射算法**：
```typescript
function getHeatmapColor(pct: number, config: HeatmapColorConfig): string {
  const clamped = Math.max(-config.maxPercent, Math.min(config.maxPercent, pct));
  const ratio = clamped / config.maxPercent;

  if (ratio > 0) {
    // 涨：从浅绿到深绿
    const lightness = 85 - ratio * 40;  // 85% → 45%
    return `hsl(142, 70%, ${lightness}%)`;
  } else if (ratio < 0) {
    // 跌：从浅红到深红
    const lightness = 85 + ratio * 40;  // 85% → 45%
    return `hsl(0, 70%, ${lightness}%)`;
  }
  return 'hsl(220, 14%, 97%)';  // 中性灰色
}
```

**布局实现**：
- 使用 CSS Grid，按板块市值加权排列
- 每个方块大小 = f(板块市值)
- 方块内显示：板块名 + 涨跌幅 + 领涨股
- hover 效果：放大 + 阴影 + tooltip 显示详情
- 点击方块：跳转到板块详情页（三级页面）

**响应式**：
- 桌面：完整热力图，方块可点击
- 平板：方块缩小，tooltip 改为点击弹出
- 移动端：改为列表视图，按涨跌幅排序

### 5.3 SectorMovers 板块动向

**文件**：`features/finance/components/sectors/SectorMovers.tsx`

**Props 设计**：
```typescript
interface SectorMoversProps {
  sectors: SectorMover[];
  loading: boolean;
  error: string | null;
  type: 'industry' | 'concept';
  onTypeChange: (type: 'industry' | 'concept') => void;
  onSectorClick: (sector: SectorMover) => void;
}

interface SectorMover {
  name: string;
  change_percent: number;
  volume: number;
  volume_change: number;
  leading_stock: string;
  leading_stock_change: number;
  up_count: number;
  down_count: number;
  flat_count: number;
}
```

**数据流**：
```
GET /api/v1/market/sectors/movers?type=industry&limit=20
→ { sectors: SectorMover[] }
→ 缓存 60 秒
```

**布局实现**：
- 两个 Tab：行业板块 / 概念板块
- 对称条形图：中轴线居中，涨向右（绿），跌向左（红）
- 每条：板块名 + 涨跌幅 + 涨跌家数 + 领涨股
- 排序：涨幅降序（涨在上，跌在下）

**响应式**：
- 桌面：完整条形图
- 平板：条形图缩小
- 移动端：改为简单列表

### 5.4 StockSearch 个股搜索

**文件**：`features/finance/components/search/StockSearch.tsx`

**Props 设计**：
```typescript
interface StockSearchProps {
  results: SearchResult[];
  loading: boolean;
  error: string | null;
  onSearch: (query: string) => void;
  onSelect: (stock: SearchResult) => void;
  onAddWatchlist?: (code: string) => void;
}

interface SearchResult {
  code: string;
  name: string;
  market: MarketKey;
  pinyin: string;
  industry: string;
  list_date: string;
}
```

**数据流**：
```
GET /api/v1/stock/search?q=招商&market=cn_stock&limit=20
→ { results: SearchResult[] }
→ 防抖 300ms，输入停止后才请求
→ 无缓存（搜索结果实时性要求高）
```

**布局实现**：
- 搜索框：顶部全宽，支持代码/名称/拼音
- 搜索结果列表：每行显示代码 + 名称 + 行业
- 快捷操作：右侧"添加自选"按钮
- 空结果：显示"未找到匹配的股票"
- 搜索历史：最近 5 条搜索记录

### 5.5 StockRanking 个股排行

**文件**：`features/finance/components/ranking/StockRanking.tsx`

**Props 设计**：
```typescript
interface StockRankingProps {
  stocks: RankingStock[];
  loading: boolean;
  error: string | null;
  rankingType: RankingType;
  onRankingTypeChange: (type: RankingType) => void;
  filters: RankingFilters;
  onFiltersChange: (filters: RankingFilters) => void;
  onStockClick: (stock: RankingStock) => void;
}

type RankingType = 'gainers' | 'losers' | 'turnover' | 'volume';

interface RankingFilters {
  industry?: string;
  market_cap_min?: number;
  market_cap_max?: number;
  pe_min?: number;
  pe_max?: number;
}

interface RankingStock {
  rank: number;
  code: string;
  name: string;
  current_price: number;
  change: number;
  change_percent: number;
  volume: number;
  turnover: number;
  market_cap: number;
  pe_ratio: number;
  industry: string;
}
```

**数据流**：
```
GET /api/v1/stock/ranking?type=gainers&industry=&limit=50
→ { stocks: RankingStock[] }
→ 缓存 30 秒
→ 切换 rankingType 或 filters 时重新请求
```

**布局实现**：
- 顶部 Tab：涨幅榜 / 跌幅榜 / 换手率榜 / 成交额榜
- 筛选器：行业下拉 + 市值范围 + PE 范围
- 排行表格：排名 + 代码 + 名称 + 现价 + 涨跌幅 + 成交量 + 市值 + PE
- 表头可点击排序
- 点击股票行 → 跳转到 StockDetail（三级页面）

### 5.6 StockDetail 个股详情（三级页面）

**文件**：`features/finance/pages/StockDetailPage.tsx`

**页面布局**：
```
┌─────────────────────────────────────────────────────────────┐
│  ← 返回  招商银行 600036.SH                                 │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────────────────────────────────────────────────┐   │
│  │  基本信息卡片                                        │   │
│  │  现价 ¥38.50  涨跌 +1.25 (+3.35%)                  │   │
│  │  成交量 1.2亿  成交额 46.2亿  换手率 0.48%           │   │
│  │  总市值 9,700亿  PE 6.82  PB 0.95                   │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  K 线图区域                                          │   │
│  │  [日K] [周K] [月K]  指标: [MA] [MACD] [RSI] [KDJ]  │   │
│  │  ┌───────────────────────────────────────────────┐  │   │
│  │  │                                               │  │   │
│  │  │           K 线图表 (Recharts)                  │  │   │
│  │  │                                               │  │   │
│  │  └───────────────────────────────────────────────┘  │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  ┌──────────────────────┬──────────────────────────────┐   │
│  │  财务数据             │  估值分析                     │   │
│  │  利润表/资产负债表/   │  PE/PB/PS/EV/EBITDA          │   │
│  │  现金流量表           │  历史估值走势图               │   │
│  └──────────────────────┴──────────────────────────────┘   │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  资金流向分析                                        │   │
│  │  主力净流入: +2.3亿  散户净流入: -0.8亿              │   │
│  │  北向资金: +1.5亿                                    │   │
│  │  资金流向柱状图 (近 10 日)                           │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  ┌──────────────────────┬──────────────────────────────┐   │
│  │  研报列表             │  相关新闻                     │   │
│  │  中金公司: 推荐       │  最新 5 条相关新闻            │   │
│  │  华泰证券: 买入       │                              │   │
│  └──────────────────────┴──────────────────────────────┘   │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  AI 个股分析                                        │   │
│  │  建议: 买入  置信度: 75%                            │   │
│  │  理由: 银行板块领涨，超卖反弹...                     │   │
│  │  目标价: ¥42.00  止损价: ¥36.00                    │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

**组件树**：
```
StockDetailPage
├── BackNavigation (返回按钮)
├── StockHeader (股票基本信息)
│   ├── StockName (代码+名称)
│   ├── PriceDisplay (现价+涨跌)
│   └── KeyMetrics (成交量/市值/PE/PB)
├── KLineChart (K 线图)
│   ├── TimeframeSelector (日K/周K/月K)
│   ├── IndicatorSelector (MA/MACD/RSI/KDJ)
│   └── Chart (Recharts CandlestickChart)
├── TwoColumnLayout
│   ├── FinancialData (财务数据)
│   │   ├── IncomeStatement (利润表)
│   │   ├── BalanceSheet (资产负债表)
│   │   └── CashFlowStatement (现金流量表)
│   └── ValuationAnalysis (估值分析)
│       ├── ValuationMetrics (PE/PB/PS/EV/EBITDA)
│       └── ValuationChart (历史估值走势)
├── FundFlowAnalysis (资金流向)
│   ├── FlowSummary (主力/散户/北向)
│   └── FlowChart (近 10 日资金流向柱状图)
├── TwoColumnLayout
│   ├── ResearchReports (研报列表)
│   │   └── ReportItem × N
│   └── RelatedNews (相关新闻)
│       └── NewsItem × 5
└── AIStockAnalysis (AI 个股分析)
    ├── Recommendation (买入/持有/卖出)
    ├── ConfidenceScore (置信度)
    ├── AnalysisReason (分析理由)
    └── PriceTarget (目标价/止损价)
```

**Props 设计**：
```typescript
interface StockDetailPageProps {
  code: string;
  market: 'cn_stock' | 'hk_stock';
}

interface StockDetail {
  basic: StockBasicInfo;
  kline: KLineData[];
  financials: FinancialData;
  valuation: ValuationData;
  fund_flow: FundFlowData;
  research: ResearchReport[];
  news: NewsItem[];
  ai_analysis: AIStockAnalysis;
}

interface StockBasicInfo {
  code: string;
  name: string;
  market: MarketKey;
  current_price: number;
  change: number;
  change_percent: number;
  open: number;
  high: number;
  low: number;
  prev_close: number;
  volume: number;
  turnover: number;
  turnover_rate: number;
  market_cap: number;
  pe_ratio: number;
  pb_ratio: number;
  industry: string;
  list_date: string;
}

interface KLineData {
  date: string;
  open: number;
  high: number;
  low: number;
  close: number;
  volume: number;
  ma5?: number;
  ma10?: number;
  ma20?: number;
  ma60?: number;
}

interface FinancialData {
  income: IncomeStatement;
  balance: BalanceSheet;
  cashflow: CashFlowStatement;
}

interface IncomeStatement {
  revenue: number[];
  net_profit: number[];
  gross_margin: number[];
  net_margin: number[];
  periods: string[];
}

interface BalanceSheet {
  total_assets: number[];
  total_liabilities: number[];
  equity: number[];
  debt_ratio: number[];
  periods: string[];
}

interface CashFlowStatement {
  operating: number[];
  investing: number[];
  financing: number[];
  periods: string[];
}

interface ValuationData {
  pe_ratio: number;
  pb_ratio: number;
  ps_ratio: number;
  ev_ebitda: number;
  pe_history: { date: string; value: number }[];
  pb_history: { date: string; value: number }[];
  industry_pe_avg: number;
  industry_pb_avg: number;
}

interface FundFlowData {
  main_flow: number;
  retail_flow: number;
  northbound_flow: number;
  daily_flow: { date: string; main: number; retail: number; northbound: number }[];
}

interface ResearchReport {
  id: string;
  broker: string;
  rating: 'buy' | 'hold' | 'sell' | 'overweight' | 'underweight';
  target_price: number;
  title: string;
  ai_summary: string;
  published_at: string;
}

interface AIStockAnalysis {
  recommendation: 'buy' | 'hold' | 'sell';
  confidence: number;
  reason: string;
  target_price: number;
  stop_loss: number;
  risk_factors: string[];
  catalysts: string[];
}
```

**数据流**：
```
GET /api/v1/stock/detail/:code?market=cn_stock
→ StockDetail
→ 缓存 30 秒
→ K 线数据独立请求: GET /api/v1/stock/kline/:code?period=daily&limit=120
→ 财务数据独立请求: GET /api/v1/stock/financials/:code
→ AI 分析独立请求: GET /api/v1/stock/ai-analysis/:code
```

**交互逻辑**：
- 返回按钮：使用 `useNavigate(-1)` 返回上一页
- K 线图切换：点击日K/周K/月K 重新请求数据
- 指标切换：点击 MA/MACD/RSI/KDJ 在图表上叠加显示
- 研报展开：点击研报卡片展开完整 AI 摘要
- 添加自选：右上角按钮，调用 POST /api/brozoomout/watchlist

---

## 六、Tab 3: 美股（US Stocks）

### 6.1 USPage 主页面

**文件**：`features/finance/pages/USPage.tsx`

**页面布局**：
```
┌─────────────────────────────────────────────────────────────┐
│  二级导航: [概览] [搜索] [排行]                               │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  USMarketOverview (美股概览)                         │   │
│  │  三大指数: S&P 500 / NASDAQ / Dow Jones              │   │
│  │  热门板块: 科技 / 金融 / 医疗 / 能源                  │   │
│  │  市场情绪: 看涨/看跌比例                              │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  ┌─────────────────────────────┬─────────────────────────┐ │
│  │  USStockSearch              │  USStockRanking          │ │
│  │  (美股搜索)                 │  (美股排行)               │ │
│  └─────────────────────────────┴─────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

### 6.2 USMarketOverview 美股概览

**文件**：`features/finance/components/us/USMarketOverviewCard.tsx`

**Props 设计**：
```typescript
interface USMarketOverviewProps {
  indices: USMarketIndex[];
  sectors: USSector[];
  sentiment: USSentiment | null;
  loading: boolean;
  onIndexClick: (index: USMarketIndex) => void;
}

interface USMarketIndex {
  name: 'S&P 500' | 'NASDAQ' | 'Dow Jones';
  symbol: string;
  price: number;
  change: number;
  change_percent: number;
  volume: number;
  updated_at: string;
}

interface USSector {
  name: string;
  change_percent: number;
  leading_stock: string;
}

interface USSentiment {
  bullish_percent: number;
  bearish_percent: number;
  vix: number;
  vix_change: number;
}
```

**数据流**：
```
GET /api/v1/us/market/overview
→ { indices: USMarketIndex[], sectors: USSector[], sentiment: USSentiment }
→ 缓存 60 秒
```

### 6.3 USStockSearch 美股搜索

**文件**：`features/finance/components/us/USStockSearch.tsx`

与 A 股搜索类似，但搜索范围限定为美股市场。支持代码（如 AAPL）和名称搜索。

**数据流**：
```
GET /api/v1/stock/search?q=Apple&market=us_stock&limit=20
→ { results: SearchResult[] }
→ 防抖 300ms
```

### 6.4 USStockRanking 美股排行

**文件**：`features/finance/components/us/USStockRanking.tsx`

与 A 股排行类似，但筛选条件适配美股（行业分类不同）。

**数据流**：
```
GET /api/v1/us/stock/ranking?type=gainers&limit=50
→ { stocks: RankingStock[] }
→ 缓存 30 秒
```

### 6.5 USStockDetail 美股详情（三级页面）

**文件**：`features/finance/pages/USStockDetailPage.tsx`

与 A 股详情页结构相同，但数据字段适配美股：
- 财务报表格式适配 US GAAP
- 估值指标增加 EV/EBITDA
- 新闻来源适配（Bloomberg, Reuters 等）

---

## 七、Tab 4: 韩国股票（Korean Stocks）

### 7.1 KoreaPage 主页面

**文件**：`features/finance/pages/KoreaPage.tsx`

**页面布局**：
```
┌─────────────────────────────────────────────────────────────┐
│  二级导航: [概览] [搜索] [排行]                               │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  KOSPIOverview (KOSPI 概览)                         │   │
│  │  KOSPI 指数走势 + KOSDAQ 指数走势                    │   │
│  │  热门板块: 半导体 / 汽车 / 电池 / 化工               │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  ┌─────────────────────────────┬─────────────────────────┐ │
│  │  KoreanStockSearch          │  KoreanStockRanking      │ │
│  │  (韩国股票搜索)             │  (韩国股票排行)           │ │
│  └─────────────────────────────┴─────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

### 7.2 KOSPIOverview KOSPI 概览

**文件**：`features/finance/components/korea/KOSPIOverviewCard.tsx`

**Props 设计**：
```typescript
interface KOSPIOverviewProps {
  kospi: KoreanMarketIndex;
  kosdaq: KoreanMarketIndex;
  sectors: KoreanSector[];
  loading: boolean;
  onIndexClick: (index: KoreanMarketIndex) => void;
}

interface KoreanMarketIndex {
  name: 'KOSPI' | 'KOSDAQ';
  symbol: string;
  price: number;
  change: number;
  change_percent: number;
  volume: number;
  updated_at: string;
}

interface KoreanSector {
  name: string;
  change_percent: number;
  leading_stock: string;
}
```

**数据流**：
```
GET /api/v1/korea/market/overview
→ { kospi: KoreanMarketIndex, kosdaq: KoreanMarketIndex, sectors: KoreanSector[] }
→ 缓存 60 秒
```

### 7.3 KoreanStockSearch 韩国股票搜索

与 A 股搜索类似，搜索范围限定为韩国市场。支持代码（如 005930.KS）和名称搜索。

### 7.4 KoreanStockRanking 韩国股票排行

与 A 股排行类似，筛选条件适配韩国市场。

### 7.5 KoreanStockDetail 韩国详情（三级页面）

与 A 股详情页结构相同，数据字段适配韩国市场（KRX 格式）。

---

## 八、Tab 5: 基金（Funds）

### 8.1 FundsPage 主页面

**文件**：`features/finance/pages/FundsPage.tsx`

**页面布局**：
```
┌─────────────────────────────────────────────────────────────┐
│  二级导航: [分类] [排名] [搜索]                               │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  FundCategories (基金分类)                           │   │
│  │  [股票型] [债券型] [混合型] [指数型] [QDII]          │   │
│  │  每类显示基金数量 + 平均收益率                        │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  ┌─────────────────────────────┬─────────────────────────┐ │
│  │  FundRanking                │  FundSearch              │ │
│  │  (基金排名)                 │  (基金搜索)               │ │
│  │  收益率排行                 │                          │ │
│  │  夏普比率排行               │                          │ │
│  │  最大回撤排行               │                          │ │
│  └─────────────────────────────┴─────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

### 8.2 FundCategories 基金分类

**文件**：`features/finance/components/funds/FundCategories.tsx`

**Props 设计**：
```typescript
interface FundCategoriesProps {
  categories: FundCategory[];
  loading: boolean;
  onCategoryClick: (category: FundCategory) => void;
}

interface FundCategory {
  id: string;
  name: string;
  fund_count: number;
  avg_return_1m: number;
  avg_return_3m: number;
  avg_return_1y: number;
  icon: string;
}
```

**数据流**：
```
GET /api/v1/fund/categories
→ { categories: FundCategory[] }
→ 缓存 600 秒（基金分类变化慢）
```

**布局实现**：
- 卡片网格：每类一张卡片
- 卡片内容：图标 + 名称 + 基金数量 + 平均收益率
- 收益率颜色：正=绿，负=红
- 点击卡片 → 跳转到该分类的基金列表

### 8.3 FundRanking 基金排名

**文件**：`features/finance/components/funds/FundRanking.tsx`

**Props 设计**：
```typescript
interface FundRankingProps {
  funds: RankedFund[];
  loading: boolean;
  error: string | null;
  rankingType: FundRankingType;
  onRankingTypeChange: (type: FundRankingType) => void;
  category?: string;
  onCategoryChange: (category: string) => void;
  onFundClick: (fund: RankedFund) => void;
}

type FundRankingType = 'return_1m' | 'return_3m' | 'return_6m' | 'return_1y' | 'return_3y' | 'sharpe' | 'max_drawdown' | 'scale';

interface RankedFund {
  rank: number;
  code: string;
  name: string;
  type: string;
  manager: string;
  scale: number;
  return_1m: number;
  return_3m: number;
  return_6m: number;
  return_1y: number;
  return_3y: number;
  sharpe_ratio: number;
  max_drawdown: number;
  volatility: number;
}
```

**数据流**：
```
GET /api/v1/fund/ranking?type=return_3m&category=&limit=50
→ { funds: RankedFund[] }
→ 缓存 300 秒
```

**布局实现**：
- 顶部 Tab：收益率排行 / 夏普比率排行 / 最大回撤排行 / 规模排行
- 筛选器：基金类型下拉
- 表格：排名 + 代码 + 名称 + 类型 + 规模 + 收益率 + 夏普比率
- 收益率颜色：正=绿，负=红
- 点击基金行 → 跳转到 FundDetail（三级页面）

### 8.4 FundSearch 基金搜索

与股票搜索类似，搜索范围限定为基金市场。支持代码（如 161725）和名称搜索。

### 8.5 FundDetail 基金详情（三级页面）

**文件**：`features/finance/pages/FundDetailPage.tsx`

**页面布局**：
```
┌─────────────────────────────────────────────────────────────┐
│  ← 返回  招商中证白酒 161725                                │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────────────────────────────────────────────────┐   │
│  │  基本信息卡片                                        │   │
│  │  类型: 股票型  规模: 150亿  经理: 侯昊              │   │
│  │  成立日期: 2015-05-27  评级: ★★★★☆                 │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  业绩走势图                                          │   │
│  │  [1月] [3月] [6月] [1年] [3年] [成立以来]           │   │
│  │  基金净值 vs 基准指数                                │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  ┌──────────────────────┬──────────────────────────────┐   │
│  │  持仓明细             │  风险分析                     │   │
│  │  前十大重仓股         │  夏普比率: 1.25              │   │
│  │  贵州茅台 15.2%      │  最大回撤: -12.5%            │   │
│  │  五粮液 12.8%        │  年化波动率: 18.3%           │   │
│  │  ...                 │  风险等级: 中高               │   │
│  └──────────────────────┴──────────────────────────────┘   │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  风格漂移检测                                        │   │
│  │  当前风格: 大盘成长  漂移程度: 低                    │   │
│  │  风格雷达图                                          │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  AI 基金推荐                                        │   │
│  │  建议: 持有  置信度: 70%                            │   │
│  │  理由: 白酒板块估值合理，经理能力优秀...             │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  同类基金对比                                        │   │
│  │  收益率排名: 前 30%  夏普比率排名: 前 25%           │   │
│  │  对比表格                                            │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

**组件树**：
```
FundDetailPage
├── BackNavigation
├── FundHeader (基金基本信息)
│   ├── FundName (代码+名称)
│   ├── FundType (类型)
│   ├── FundScale (规模)
│   └── FundManager (经理)
├── PerformanceChart (业绩走势图)
│   ├── TimeframeSelector (1月/3月/6月/1年/3年/成立以来)
│   └── Chart (Recharts LineChart, 基金 vs 基准)
├── TwoColumnLayout
│   ├── HoldingsTable (持仓明细)
│   │   └── HoldingItem × 10
│   └── RiskAnalysis (风险分析)
│       ├── SharpeRatio
│       ├── MaxDrawdown
│       ├── Volatility
│       └── RiskLevel
├── StyleDrift (风格漂移检测)
│   ├── CurrentStyle (当前风格)
│   ├── DriftLevel (漂移程度)
│   └── RadarChart (风格雷达图)
├── AIFundRecommendation (AI 基金推荐)
│   ├── Recommendation (买入/持有/赎回)
│   ├── ConfidenceScore
│   └── AnalysisReason
└── PeerComparison (同类基金对比)
    ├── RankingSummary (排名摘要)
    └── ComparisonTable (对比表格)
```

**Props 设计**：
```typescript
interface FundDetailPageProps {
  code: string;
}

interface FundDetail {
  basic: FundBasicInfo;
  performance: FundPerformance;
  holdings: FundHolding[];
  risk: FundRisk;
  style: FundStyle;
  ai_recommendation: AIFundRecommendation;
  peers: FundPeer[];
}

interface FundBasicInfo {
  code: string;
  name: string;
  type: string;
  manager: string;
  scale: number;
  establish_date: string;
  rating: number;
  benchmark: string;
}

interface FundPerformance {
  nav: number;
  nav_change: number;
  cumulative_return: { period: string; value: number }[];
  benchmark_return: { period: string; value: number }[];
  daily_nav: { date: string; nav: number; benchmark: number }[];
}

interface FundHolding {
  stock_code: string;
  stock_name: string;
  percentage: number;
  change: number;
}

interface FundRisk {
  sharpe_ratio: number;
  max_drawdown: number;
  volatility: number;
  risk_level: 'low' | 'medium' | 'high';
  sortino_ratio: number;
  information_ratio: number;
}

interface FundStyle {
  current_style: string;
  drift_level: 'low' | 'medium' | 'high';
  style_scores: {
    large_cap: number;
    mid_cap: number;
    small_cap: number;
    value: number;
    growth: number;
    blend: number;
  };
}

interface AIFundRecommendation {
  recommendation: 'buy' | 'hold' | 'redeem';
  confidence: number;
  reason: string;
  risk_factors: string[];
}

interface FundPeer {
  code: string;
  name: string;
  return_1y: number;
  sharpe_ratio: number;
  max_drawdown: number;
  rank_return: number;
  rank_sharpe: number;
}
```

**数据流**：
```
GET /api/v1/fund/detail/:code
→ FundDetail
→ 缓存 600 秒
→ 业绩走势独立请求: GET /api/v1/fund/performance/:code?period=1y
→ AI 推荐独立请求: GET /api/v1/fund/ai-recommendation/:code
```

---

## 九、Tab 6: AI 分析（AI Analysis）

### 9.1 AIPage 主页面

**文件**：`features/finance/pages/AIPage.tsx`

**页面布局**：
```
┌─────────────────────────────────────────────────────────────┐
│  二级导航: [市场预测] [个股分析] [组合优化] [风险评估]        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  MarketPrediction (市场预测)                         │   │
│  │  三场景预测: 牛市 / 基准 / 熊市                      │   │
│  │  概率分布 + 关键因素分析                              │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  ┌─────────────────────────────┬─────────────────────────┐ │
│  │  StockAnalysis              │  PortfolioOptimization   │ │
│  │  (个股分析)                 │  (组合优化)               │ │
│  └─────────────────────────────┴─────────────────────────┘ │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  RiskAssessment (风险评估)                           │   │
│  │  持仓风险评估 + 压力测试 + 相关性分析                │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### 9.2 MarketPrediction 市场预测

**文件**：`features/finance/components/ai/MarketPredictionCard.tsx`

**Props 设计**：
```typescript
interface MarketPredictionProps {
  prediction: MarketPrediction | null;
  loading: boolean;
  error: string | null;
  horizon: '1w' | '1m';
  onHorizonChange: (horizon: '1w' | '1m') => void;
}

interface MarketPrediction {
  summary: string;
  confidence: number;
  generated_at: string;
  scenarios: Scenario[];
  key_factors: KeyFactor[];
}

interface Scenario {
  name: 'bullish' | 'base' | 'bearish';
  probability: number;
  description: string;
  target_change: number;
  triggers: string[];
}

interface KeyFactor {
  name: string;
  impact: 'positive' | 'negative' | 'neutral';
  description: string;
  probability: number;
}
```

**数据流**：
```
GET /api/v1/ai/market-prediction?horizon=1w
→ MarketPrediction
→ 缓存 600 秒（AI 预测更新频率低）
```

**布局实现**：
- Tab 切换：1 周 / 1 月
- 顶部：预测摘要文字 + 置信度
- 中部：三个场景卡片（横向排列）
  - 牛市=绿色边框，基准=蓝色边框，熊市=红色边框
  - 每个卡片：概率% + 描述 + 目标涨幅 + 触发条件
- 底部：关键因素列表，每个因素显示影响方向和概率

### 9.3 StockAnalysis 个股分析

**文件**：`features/finance/components/ai/StockAnalysisCard.tsx`

**Props 设计**：
```typescript
interface StockAnalysisProps {
  result: AIStockAnalysisResult | null;
  loading: boolean;
  error: string | null;
  onAnalyze: (code: string) => void;
}

interface AIStockAnalysisResult {
  code: string;
  name: string;
  recommendation: 'buy' | 'hold' | 'sell';
  confidence: number;
  summary: string;
  technical_analysis: TechnicalAnalysis;
  fundamental_analysis: FundamentalAnalysis;
  sentiment_analysis: SentimentAnalysis;
  risk_factors: string[];
  catalysts: string[];
  price_target: {
    current: number;
    target: number;
    stop_loss: number;
    upside: number;
  };
  generated_at: string;
}

interface TechnicalAnalysis {
  trend: 'upward' | 'downward' | 'sideways';
  support_levels: number[];
  resistance_levels: number[];
  indicators: {
    rsi: number;
    macd: { signal: string; value: number };
    ma_cross: string;
    volume_trend: string;
  };
}

interface FundamentalAnalysis {
  valuation: 'undervalued' | 'fair' | 'overvalued';
  pe_vs_industry: string;
  growth_potential: string;
  financial_health: string;
}

interface SentimentAnalysis {
  overall: 'positive' | 'negative' | 'neutral';
  news_sentiment: number;
  analyst_consensus: string;
  insider_activity: string;
}
```

**数据流**：
```
POST /api/v1/ai/stock-analysis
Body: { code: "600036", market: "cn_stock" }
→ AIStockAnalysisResult
→ 无缓存（每次分析都是实时生成）
→ 预计响应时间 3-5 秒
```

**布局实现**：
- 输入框：股票代码输入 + 分析按钮
- 分析结果：
  - 顶部：推荐 + 置信度 + 目标价
  - 三列：技术分析 / 基本面分析 / 情绪分析
  - 底部：风险因素 + 催化剂
- 加载状态：骨架屏 + "AI 正在分析..." 提示

### 9.4 PortfolioOptimization 组合优化

**文件**：`features/finance/components/ai/PortfolioOptimizationCard.tsx`

**Props 设计**：
```typescript
interface PortfolioOptimizationProps {
  result: OptimizationResult | null;
  loading: boolean;
  error: string | null;
  onOptimize: (holdings: HoldingInput[]) => void;
}

interface HoldingInput {
  code: string;
  name: string;
  weight: number;
}

interface OptimizationResult {
  current_portfolio: PortfolioMetrics;
  optimized_portfolio: PortfolioMetrics;
  suggestions: RebalanceSuggestion[];
  efficient_frontier: EfficientFrontierPoint[];
  risk_return_analysis: RiskReturnAnalysis;
}

interface PortfolioMetrics {
  expected_return: number;
  volatility: number;
  sharpe_ratio: number;
  max_drawdown: number;
  diversification_score: number;
}

interface RebalanceSuggestion {
  code: string;
  name: string;
  current_weight: number;
  suggested_weight: number;
  action: 'increase' | 'decrease' | 'hold' | 'remove';
  reason: string;
}

interface EfficientFrontierPoint {
  return: number;
  risk: number;
  is_optimal: boolean;
}

interface RiskReturnAnalysis {
  correlation_matrix: { stock1: string; stock2: string; correlation: number }[];
  concentration_risk: number;
  sector_exposure: { sector: string; weight: number }[];
}
```

**数据流**：
```
POST /api/v1/ai/portfolio-optimization
Body: { holdings: HoldingInput[] }
→ OptimizationResult
→ 无缓存（每次优化都是实时计算）
→ 预计响应时间 5-8 秒
```

**布局实现**：
- 输入区：持仓列表（代码 + 权重）
- 优化结果：
  - 对比卡片：当前组合 vs 优化后组合
  - 建议列表：每个持仓的调整建议
  - 有效前沿图：Recharts ScatterChart
  - 风险收益分析：相关性矩阵 + 行业暴露

### 9.5 RiskAssessment 风险评估

**文件**：`features/finance/components/ai/RiskAssessmentCard.tsx`

**Props 设计**：
```typescript
interface RiskAssessmentProps {
  result: RiskAssessmentResult | null;
  loading: boolean;
  error: string | null;
  onAssess: (holdings: HoldingInput[]) => void;
}

interface RiskAssessmentResult {
  overall_risk: 'low' | 'medium' | 'high';
  risk_score: number;
  stress_test: StressTestResult;
  correlation_analysis: CorrelationAnalysis;
  var_analysis: VaRAnalysis;
  scenario_analysis: ScenarioAnalysis[];
}

interface StressTestResult {
  scenarios: StressScenario[];
  worst_case_loss: number;
  recovery_time: string;
}

interface StressScenario {
  name: string;
  description: string;
  portfolio_impact: number;
  probability: number;
}

interface CorrelationAnalysis {
  average_correlation: number;
  max_correlation: { stock1: string; stock2: string; value: number };
  diversification_benefit: number;
}

interface VaRAnalysis {
  var_95: number;
  var_99: number;
  cvar_95: number;
  time_horizon: string;
}

interface ScenarioAnalysis {
  name: string;
  description: string;
  impact: number;
  probability: number;
}
```

**数据流**：
```
POST /api/v1/ai/risk-assessment
Body: { holdings: HoldingInput[] }
→ RiskAssessmentResult
→ 无缓存（每次评估都是实时计算）
→ 预计响应时间 3-5 秒
```

**布局实现**：
- 输入区：持仓列表
- 风险概览：总体风险等级 + 风险分数
- 压力测试：场景列表 + 最大损失
- 相关性分析：平均相关性 + 分散化收益
- VaR 分析：95%/99% VaR + CVaR
- 场景分析：多个宏观场景的影响

---

## 十、全局组件设计

### 10.1 TabNavigation Tab 导航栏

**文件**：`features/finance/components/layout/TabNavigation.tsx`

**Props 设计**：
```typescript
interface TabNavigationProps {
  activeTab: TabKey;
  onTabChange: (tab: TabKey) => void;
}

type TabKey = 'dashboard' | 'domestic' | 'us' | 'korea' | 'funds' | 'ai';

interface TabConfig {
  key: TabKey;
  label: string;
  icon: LucideIcon;
  path: string;
}
```

**实现**：
- 固定在顶部，全宽
- 6 个 Tab 按钮横向排列
- 激活态：`pill-tab-active` 样式（黑底白字）
- 非激活态：`pill-tab` 样式（白底灰字）
- 响应式：移动端改为下拉菜单

**Tab 配置**：
```typescript
const TABS: TabConfig[] = [
  { key: 'dashboard', label: 'Dashboard', icon: LayoutDashboard, path: '/' },
  { key: 'domestic', label: 'A股/港股', icon: TrendingUp, path: '/domestic' },
  { key: 'us', label: '美股', icon: Globe, path: '/us' },
  { key: 'korea', label: '韩国股票', icon: Flag, path: '/korea' },
  { key: 'funds', label: '基金', icon: Wallet, path: '/funds' },
  { key: 'ai', label: 'AI 分析', icon: Brain, path: '/ai' },
];
```

### 10.2 SubPageNavigation 二级页面导航

**文件**：`features/finance/components/layout/SubPageNavigation.tsx`

**Props 设计**：
```typescript
interface SubPageNavigationProps {
  pages: SubPageConfig[];
  activePage: string;
  onPageChange: (path: string) => void;
}

interface SubPageConfig {
  key: string;
  label: string;
  path: string;
  icon?: LucideIcon;
}
```

**实现**：
- 水平滚动的 pill 按钮组
- 激活态：`pill-tab-active`
- 非激活态：`pill-tab`
- 移动端：横向滚动

### 10.3 BackNavigation 返回导航

**文件**：`features/finance/components/layout/BackNavigation.tsx`

**Props 设计**：
```typescript
interface BackNavigationProps {
  title: string;
  subtitle?: string;
  onBack?: () => void;
}
```

**实现**：
- 左侧返回箭头 + 标题 + 副标题
- 点击返回：`useNavigate(-1)`
- 样式：`text-lg font-semibold`

### 10.4 StockCard 股票卡片

**文件**：`features/finance/components/shared/StockCard.tsx`

**Props 设计**：
```typescript
interface StockCardProps {
  code: string;
  name: string;
  market: MarketKey;
  current_price: number;
  change: number;
  change_percent: number;
  signal?: SignalType;
  onClick?: () => void;
  onAddWatchlist?: () => void;
  compact?: boolean;
}
```

**实现**：
- 卡片样式：`card-base`
- 代码 + 名称 + 市场标签
- 现价 + 涨跌 + 涨跌幅
- 信号标签（可选）
- hover：放大 + 阴影
- 点击：跳转到详情页

### 10.5 IndexCard 指数卡片

**文件**：`features/finance/components/shared/IndexCard.tsx`

**Props 设计**：
```typescript
interface IndexCardProps {
  name: string;
  symbol: string;
  price: number;
  change_percent: number;
  onClick?: () => void;
  compact?: boolean;
}
```

**实现**：
- 小卡片样式，适合网格布局
- 名称 + 价格 + 涨跌幅
- 背景色随涨跌幅变化（涨=浅绿，跌=浅红）

### 10.6 RankingTable 排行表格

**文件**：`features/finance/components/shared/RankingTable.tsx`

**Props 设计**：
```typescript
interface RankingTableProps<T> {
  data: T[];
  columns: ColumnConfig<T>[];
  loading: boolean;
  onRowClick?: (item: T) => void;
  sortBy?: string;
  sortOrder?: 'asc' | 'desc';
  onSort?: (field: string) => void;
}

interface ColumnConfig<T> {
  key: string;
  label: string;
  width?: string;
  align?: 'left' | 'center' | 'right';
  render?: (value: unknown, item: T) => ReactNode;
  sortable?: boolean;
}
```

**实现**：
- 通用表格组件，支持自定义列
- 表头可点击排序
- 行点击跳转详情
- 加载态：骨架屏
- 空态：友好提示

### 10.7 FilterBar 筛选栏

**文件**：`features/finance/components/shared/FilterBar.tsx`

**Props 设计**：
```typescript
interface FilterBarProps {
  filters: FilterConfig[];
  values: Record<string, unknown>;
  onChange: (key: string, value: unknown) => void;
}

interface FilterConfig {
  key: string;
  label: string;
  type: 'select' | 'range' | 'toggle';
  options?: { label: string; value: unknown }[];
  min?: number;
  max?: number;
  step?: number;
}
```

**实现**：
- 水平排列的筛选控件
- 下拉选择：`filter-dropdown` 样式
- 范围滑块：双滑块
- 切换按钮：`pill-tab` 样式

### 10.8 ChartContainer 图表容器

**文件**：`features/finance/components/shared/ChartContainer.tsx`

**Props 设计**：
```typescript
interface ChartContainerProps {
  title: string;
  actions?: ReactNode;
  loading?: boolean;
  error?: string | null;
  height?: number;
  children: ReactNode;
}
```

**实现**：
- 标题 + 操作按钮区
- 图表内容区（固定高度）
- 加载态：骨架屏
- 错误态：错误提示

---

## 十一、全局状态管理

### 11.1 状态管理方案

采用 React Context + useReducer 方案，不引入外部状态管理库：

```typescript
// 全局状态结构
interface FinanceState {
  // 主题
  theme: 'light' | 'dark';

  // 导航
  activeTab: TabKey;
  navigationHistory: string[];

  // 市场数据缓存
  marketData: {
    indices: MarketIndex[];
    sentiment: MarketSentiment | null;
    lastFetched: number;
  };

  // 用户持仓
  portfolio: {
    summary: PortfolioData | null;
    lastFetched: number;
  };

  // 自选股
  watchlist: {
    items: WatchlistItem[];
    lastFetched: number;
  };

  // 设置
  settings: {
    refreshInterval: 30 | 60 | 300;
    defaultMarket: MarketKey;
    compactMode: boolean;
  };
}
```

### 11.2 Context Provider

```typescript
// FinanceContext.tsx
interface FinanceContextType {
  state: FinanceState;
  dispatch: React.Dispatch<FinanceAction>;
  // 便捷方法
  refreshMarketData: () => Promise<void>;
  refreshPortfolio: () => Promise<void>;
  refreshWatchlist: () => Promise<void>;
  toggleTheme: () => void;
}
```

### 11.3 数据缓存策略

```typescript
// 缓存配置
const CACHE_CONFIG = {
  marketIndices: { ttl: 60_000, key: 'market-indices' },
  sentiment: { ttl: 300_000, key: 'market-sentiment' },
  portfolio: { ttl: 30_000, key: 'portfolio-summary' },
  watchlist: { ttl: 30_000, key: 'watchlist' },
  news: { ttl: 120_000, key: 'news-feed' },
  aiInsights: { ttl: 300_000, key: 'ai-insights' },
  sectors: { ttl: 120_000, key: 'sectors' },
  fundCategories: { ttl: 600_000, key: 'fund-categories' },
} as const;

// 缓存 Hook
function useCachedData<T>(
  key: string,
  fetcher: () => Promise<T>,
  ttl: number
): { data: T | null; loading: boolean; refresh: () => void } {
  const [data, setData] = useState<T | null>(null);
  const [loading, setLoading] = useState(true);
  const [lastFetched, setLastFetched] = useState(0);

  const refresh = useCallback(async () => {
    setLoading(true);
    try {
      const result = await fetcher();
      setData(result);
      setLastFetched(Date.now());
    } finally {
      setLoading(false);
    }
  }, []);

  useEffect(() => {
    if (Date.now() - lastFetched > ttl || !data) {
      void refresh();
    }
  }, [ttl, lastFetched, data, refresh]);

  return { data, loading, refresh };
}
```

### 11.4 自动刷新机制

```typescript
// AutoRefresh.tsx
function useAutoRefresh(interval: number, callback: () => void) {
  useEffect(() => {
    const timer = setInterval(callback, interval);
    return () => clearInterval(timer);
  }, [interval, callback]);
}

// 使用示例
useAutoRefresh(settings.refreshInterval * 1000, () => {
  void refreshMarketData();
  void refreshWatchlist();
});
```

---

## 十二、API 接口清单

### 12.1 市场数据接口

| 方法 | 端点 | 说明 | 缓存 |
|------|------|------|------|
| GET | `/api/v1/market/overview` | 全球指数概览 | 60s |
| GET | `/api/v1/market/sectors?type={type}` | 板块列表 | 120s |
| GET | `/api/v1/market/sectors/movers?type={type}` | 板块动向 | 60s |
| GET | `/api/v1/market/sentiment` | 市场情绪 | 300s |

### 12.2 股票接口

| 方法 | 端点 | 说明 | 缓存 |
|------|------|------|------|
| GET | `/api/v1/stock/search?q={query}&market={market}` | 股票搜索 | 无 |
| GET | `/api/v1/stock/ranking?type={type}` | 个股排行 | 30s |
| GET | `/api/v1/stock/detail/:code?market={market}` | 股票详情 | 30s |
| GET | `/api/v1/stock/kline/:code?period={period}` | K 线数据 | 30s |
| GET | `/api/v1/stock/financials/:code` | 财务数据 | 600s |
| GET | `/api/v1/stock/fund-flow/:code` | 资金流向 | 60s |
| GET | `/api/v1/stock/research/:code` | 研报列表 | 600s |
| GET | `/api/v1/stock/news/:code` | 相关新闻 | 120s |

### 12.3 美股接口

| 方法 | 端点 | 说明 | 缓存 |
|------|------|------|------|
| GET | `/api/v1/us/market/overview` | 美股概览 | 60s |
| GET | `/api/v1/us/stock/ranking?type={type}` | 美股排行 | 30s |

### 12.4 韩国股票接口

| 方法 | 端点 | 说明 | 缓存 |
|------|------|------|------|
| GET | `/api/v1/korea/market/overview` | KOSPI 概览 | 60s |
| GET | `/api/v1/korea/stock/ranking?type={type}` | 韩股排行 | 30s |

### 12.5 基金接口

| 方法 | 端点 | 说明 | 缓存 |
|------|------|------|------|
| GET | `/api/v1/fund/categories` | 基金分类 | 600s |
| GET | `/api/v1/fund/ranking?type={type}&category={cat}` | 基金排名 | 300s |
| GET | `/api/v1/fund/search?q={query}` | 基金搜索 | 无 |
| GET | `/api/v1/fund/detail/:code` | 基金详情 | 600s |
| GET | `/api/v1/fund/performance/:code?period={period}` | 业绩走势 | 600s |
| GET | `/api/v1/fund/holdings/:code` | 持仓明细 | 600s |

### 12.6 AI 分析接口

| 方法 | 端点 | 说明 | 缓存 |
|------|------|------|------|
| GET | `/api/v1/ai/insights` | AI 洞察 | 300s |
| GET | `/api/v1/ai/market-prediction?horizon={horizon}` | 市场预测 | 600s |
| POST | `/api/v1/ai/stock-analysis` | 个股分析 | 无 |
| POST | `/api/v1/ai/portfolio-optimization` | 组合优化 | 无 |
| POST | `/api/v1/ai/risk-assessment` | 风险评估 | 无 |
| GET | `/api/v1/news?importance={importance}&limit={limit}` | 新闻列表 | 120s |

### 12.7 BroZoomOut 接口（已有）

| 方法 | 端点 | 说明 | 缓存 |
|------|------|------|------|
| GET | `/api/brozoomout/summary?scope={scope}` | 总览摘要 | - |
| GET | `/api/brozoomout/dashboard` | Dashboard 数据 | - |
| GET | `/api/brozoomout/markets/{market}` | 市场详情 | - |
| GET | `/api/brozoomout/watchlist` | 自选股列表 | 30s |
| POST | `/api/brozoomout/watchlist` | 添加自选 | - |
| DELETE | `/api/brozoomout/watchlist/:id` | 删除自选 | - |
| POST | `/api/brozoomout/strategy/preview` | 策略预览 | - |
| POST | `/api/brozoomout/snapshots/refresh` | 刷新快照 | - |

### 12.8 请求/响应格式

**统一请求格式**：
```typescript
// GET 请求
GET /api/v1/stock/search?q=招商&market=cn_stock&limit=20

// POST 请求
POST /api/v1/ai/stock-analysis
Content-Type: application/json
Authorization: Bearer {token}

{
  "code": "600036",
  "market": "cn_stock"
}
```

**统一响应格式**：
```typescript
interface ApiResponse<T> {
  ok: boolean;
  data: T;
  meta?: {
    request_id: string;
    timestamp: string;
    cache_hit: boolean;
  };
}

interface ApiError {
  ok: false;
  error: {
    code: string;
    message: string;
    details?: Record<string, unknown>;
  };
}
```

**错误处理**：
```typescript
// BroZoomOutApiError 已有，扩展为通用
class FinanceApiError extends Error {
  code: string;
  status: number;
  retryable: boolean;

  constructor(message: string, status: number, code: string, retryable = false) {
    super(message);
    this.name = 'FinanceApiError';
    this.status = status;
    this.code = code;
    this.retryable = retryable;
  }
}

// 重试逻辑
async function requestWithRetry<T>(
  fetcher: () => Promise<T>,
  maxRetries = 2,
  delay = 1000
): Promise<T> {
  for (let i = 0; i <= maxRetries; i++) {
    try {
      return await fetcher();
    } catch (err) {
      if (i === maxRetries || !(err instanceof FinanceApiError) || !err.retryable) {
        throw err;
      }
      await new Promise(r => setTimeout(r, delay * (i + 1)));
    }
  }
  throw new Error('unreachable');
}
```

---

## 十三、设计系统与样式规范

### 13.1 颜色系统

基于 `design-tokens.css` 已有变量，扩展金融专用颜色：

```css
/* 金融专用颜色 */
:root {
  /* 涨跌颜色 */
  --zl-up: 142 70% 45%;         /* 涨 - 绿色 */
  --zl-up-light: 142 70% 92%;   /* 涨 - 浅绿背景 */
  --zl-down: 0 70% 55%;         /* 跌 - 红色 */
  --zl-down-light: 0 70% 92%;   /* 跌 - 浅红背景 */
  --zl-flat: 220 14% 60%;       /* 平 - 灰色 */

  /* 信号颜色 */
  --zl-signal-buy: 142 70% 45%;
  --zl-signal-sell: 0 70% 55%;
  --zl-signal-hold: 220 14% 60%;
  --zl-signal-watch: 45 100% 59%;

  /* 风险等级 */
  --zl-risk-low: 142 70% 45%;
  --zl-risk-medium: 45 100% 59%;
  --zl-risk-high: 0 70% 55%;

  /* 情感标签 */
  --zl-sentiment-positive: 142 70% 45%;
  --zl-sentiment-negative: 0 70% 55%;
  --zl-sentiment-neutral: 220 14% 60%;

  /* 场景颜色 */
  --zl-scenario-bullish: 142 70% 45%;
  --zl-scenario-base: 230 100% 63%;
  --zl-scenario-bearish: 0 70% 55%;
}

.dark {
  --zl-up: 142 70% 55%;
  --zl-up-light: 142 30% 15%;
  --zl-down: 0 70% 65%;
  --zl-down-light: 0 30% 15%;
  --zl-flat: 230 10% 55%;
}
```

### 13.2 Tailwind CSS 扩展

```javascript
// tailwind.config.js 扩展
module.exports = {
  theme: {
    extend: {
      colors: {
        'zl-up': 'hsl(var(--zl-up))',
        'zl-up-light': 'hsl(var(--zl-up-light))',
        'zl-down': 'hsl(var(--zl-down))',
        'zl-down-light': 'hsl(var(--zl-down-light))',
        'zl-flat': 'hsl(var(--zl-flat))',
      },
      fontFamily: {
        'display': ['Roobert PRO', 'Noto Sans', '-apple-system', 'sans-serif'],
      },
    },
  },
};
```

### 13.3 组件样式规范

**卡片样式**：
```css
/* 标准卡片 */
.card-base {
  background: hsl(var(--zl-canvas));
  border-radius: var(--zl-radius-xl);
  padding: var(--zl-space-xl);
  border: 1px solid hsl(var(--zl-hairline-soft));
}

/* 金融数据卡片 */
.card-finance {
  background: hsl(var(--zl-canvas));
  border-radius: var(--zl-radius-lg);
  padding: var(--zl-space-md);
  border: 1px solid hsl(var(--zl-hairline-soft));
  transition: box-shadow var(--zl-motion-base) var(--zl-motion-ease);
}

.card-finance:hover {
  box-shadow: var(--zl-shadow-hover);
}
```

**数字样式**：
```css
/* 等宽数字 */
.tabular-nums {
  font-variant-numeric: tabular-nums;
}

/* 涨跌数字 */
.price-up {
  color: hsl(var(--zl-up));
  font-weight: 600;
  font-variant-numeric: tabular-nums;
}

.price-down {
  color: hsl(var(--zl-down));
  font-weight: 600;
  font-variant-numeric: tabular-nums;
}

.price-flat {
  color: hsl(var(--zl-flat));
  font-variant-numeric: tabular-nums;
}
```

**标签样式**：
```css
/* 信号标签 */
.badge-signal {
  padding: 2px 8px;
  border-radius: var(--zl-radius-full);
  font-size: 11px;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.5px;
}

.badge-buy {
  background: hsl(var(--zl-up-light));
  color: hsl(var(--zl-up));
}

.badge-sell {
  background: hsl(var(--zl-down-light));
  color: hsl(var(--zl-down));
}

.badge-hold {
  background: hsl(var(--zl-surface));
  color: hsl(var(--zl-steel));
}
```

### 13.4 字体规范

| 用途 | 样式类 | 字号 | 字重 |
|------|--------|------|------|
| 页面标题 | `text-2xl font-semibold` | 24px | 600 |
| 模块标题 | `text-lg font-medium` | 18px | 500 |
| 卡片标题 | `text-base font-medium` | 16px | 500 |
| 正文 | `text-sm` | 14px | 400 |
| 标签 | `text-xs` | 12px | 400 |
| 微标签 | `text-[11px] font-semibold uppercase` | 11px | 600 |
| 大数字 | `text-3xl font-bold tabular-nums` | 30px | 700 |
| 中数字 | `text-xl font-semibold tabular-nums` | 20px | 600 |
| 小数字 | `text-sm font-medium tabular-nums` | 14px | 500 |

---

## 十四、响应式设计规范

### 14.1 断点定义

| 断点 | 宽度 | 布局 |
|------|------|------|
| Mobile S | < 480px | 单列，紧凑布局 |
| Mobile L | 480-767px | 单列，标准布局 |
| Tablet | 768-1023px | 双列 |
| Desktop | 1024-1279px | 三列 |
| Wide Desktop | ≥ 1280px | 完整布局 |

### 14.2 组件响应式行为

**MarketTicker**：
- 桌面：完整显示 7 个指数
- 平板：显示 5 个指数，横向滚动
- 移动端：显示 3 个指数，横向滚动

**MarketHeatmap**：
- 桌面：完整热力图，3×3 网格
- 平板：2×3 网格
- 移动端：改为列表视图

**RankingTable**：
- 桌面：完整表格，所有列
- 平板：隐藏部分列（市值、PE）
- 移动端：改为卡片列表

**StockDetail**：
- 桌面：多列布局（K 线图 + 财务数据并排）
- 平板：两列布局
- 移动端：单列堆叠

**NewsFeed**：
- 桌面：两列（新闻列表 + 侧边栏）
- 平板：单列
- 移动端：单列，卡片堆叠

### 14.3 响应式工具类

```css
/* 响应式容器 */
.container-finance {
  width: 100%;
  max-width: 1280px;
  margin: 0 auto;
  padding: 0 var(--zl-space-md);
}

@media (min-width: 768px) {
  .container-finance {
    padding: 0 var(--zl-space-xl);
  }
}

@media (min-width: 1280px) {
  .container-finance {
    padding: 0 var(--zl-space-xxl);
  }
}

/* 响应式网格 */
.grid-finance {
  display: grid;
  gap: var(--zl-space-md);
  grid-template-columns: 1fr;
}

@media (min-width: 768px) {
  .grid-finance {
    grid-template-columns: repeat(2, 1fr);
  }
}

@media (min-width: 1280px) {
  .grid-finance {
    grid-template-columns: repeat(3, 1fr);
  }
}
```

### 14.4 移动端特殊处理

| 组件 | 移动端处理 |
|------|-----------|
| TabNavigation | 改为底部 Tab 栏 |
| SubPageNavigation | 改为横向滚动 |
| MarketHeatmap | 改为列表视图 |
| RankingTable | 改为卡片列表 |
| KLineChart | 全宽，支持手势缩放 |
| FilterBar | 改为全屏筛选弹窗 |
| StockDetail | 单列堆叠，K 线图全宽 |

---

## 十五、日间/夜间模式规范

### 15.1 主题切换机制

```typescript
// ThemeContext.tsx
interface ThemeContextType {
  theme: 'light' | 'dark';
  toggleTheme: () => void;
  setTheme: (theme: 'light' | 'dark') => void;
}

// 切换逻辑
function toggleTheme() {
  const next = theme === 'light' ? 'dark' : 'light';
  setTheme(next);
  document.documentElement.classList.toggle('dark', next === 'dark');
  localStorage.setItem('zl-theme', next);
}

// 初始化
useEffect(() => {
  const saved = localStorage.getItem('zl-theme');
  const systemPrefersDark = window.matchMedia('(prefers-color-scheme: dark)').matches;
  const initial = saved || (systemPrefersDark ? 'dark' : 'light');
  setTheme(initial);
  document.documentElement.classList.toggle('dark', initial === 'dark');
}, []);
```

### 15.2 颜色映射表

| 语义 | 日间模式 | 夜间模式 |
|------|----------|----------|
| 页面背景 | `bg-white` | `dark:bg-zinc-950` |
| 卡片背景 | `bg-gray-50` | `dark:bg-zinc-900` |
| 文字主色 | `text-gray-900` | `dark:text-white` |
| 文字次色 | `text-gray-500` | `dark:text-gray-400` |
| 边框 | `border-gray-200` | `dark:border-zinc-800` |
| 涨（正） | `text-emerald-600` | `dark:text-emerald-400` |
| 跌（负） | `text-red-600` | `dark:text-red-400` |
| 利好 | `text-emerald-600` | `dark:text-emerald-400` |
| 利空 | `text-red-600` | `dark:text-red-400` |
| 骨架屏 | `bg-gray-200` | `dark:bg-zinc-800` |
| 输入框 | `bg-white border-gray-200` | `dark:bg-zinc-900 dark:border-zinc-700` |

### 15.3 过渡动画

```css
/* 主题切换过渡 */
* {
  transition: background-color 200ms ease, color 200ms ease, border-color 200ms ease;
}

/* 禁用过渡（性能优化） */
.no-transition * {
  transition: none !important;
}
```

### 15.4 图表主题适配

```typescript
// Recharts 主题配置
const chartTheme = {
  light: {
    background: '#ffffff',
    textColor: '#1c1c1e',
    gridColor: '#e0e2e8',
    tooltipBg: '#ffffff',
    tooltipBorder: '#e0e2e8',
  },
  dark: {
    background: '#0f0f12',
    textColor: '#e0e2e8',
    gridColor: '#27272a',
    tooltipBg: '#18181b',
    tooltipBorder: '#27272a',
  },
};
```

---

## 十六、交互与动画规范

### 16.1 页面切换

```typescript
// Tab 切换动画
.tab-transition {
  transition: opacity 200ms ease, transform 200ms ease;
}

.tab-enter {
  opacity: 0;
  transform: translateY(8px);
}

.tab-enter-active {
  opacity: 1;
  transform: translateY(0);
}

.tab-exit {
  opacity: 1;
  transform: translateY(0);
}

.tab-exit-active {
  opacity: 0;
  transform: translateY(-8px);
}
```

### 16.2 数据刷新

```typescript
// 刷新动画
.refresh-spin {
  animation: spin 1s linear infinite;
}

@keyframes spin {
  from { transform: rotate(0deg); }
  to { transform: rotate(360deg); }
}

// 骨架屏脉冲
.skeleton-pulse {
  animation: pulse 2s cubic-bezier(0.4, 0, 0.6, 1) infinite;
}

@keyframes pulse {
  0%, 100% { opacity: 1; }
  50% { opacity: 0.5; }
}
```

### 16.3 交互反馈

```typescript
// 按钮点击反馈
.button-press {
  transition: transform 100ms ease, box-shadow 100ms ease;
}

.button-press:active {
  transform: scale(0.98);
}

// 卡片悬停
.card-hover {
  transition: box-shadow 200ms ease, transform 200ms ease;
}

.card-hover:hover {
  box-shadow: var(--zl-shadow-hover);
  transform: translateY(-2px);
}

// 数字变化动画
.number-change {
  transition: color 300ms ease;
}
```

### 16.4 空状态引导

```typescript
// 空状态组件
interface EmptyStateProps {
  icon: LucideIcon;
  title: string;
  description: string;
  action?: {
    label: string;
    onClick: () => void;
  };
}

// 使用示例
<EmptyState
  icon={StarOff}
  title="暂无自选股"
  description="添加关注的股票，获取实时行情和 AI 分析"
  action={{ label: "添加股票", onClick: () => setShowAdd(true) }}
/>
```

---

## 十七、组件清单与层级关系

### 17.1 组件总览

| 类别 | 组件数 | 说明 |
|------|--------|------|
| 布局组件 | 5 | TabNavigation, SubPageNavigation, BackNavigation, TwoColumnLayout, PageContainer |
| 共享组件 | 8 | StockCard, IndexCard, RankingTable, FilterBar, ChartContainer, EmptyState, LoadingState, ErrorState |
| 市场组件 | 6 | MarketOverviewCard, MarketHeatmap, SectorMovers, SentimentGauge, IndexGrid, SectorBar |
| 搜索组件 | 4 | StockSearch, USStockSearch, KoreanStockSearch, FundSearch |
| 排行组件 | 4 | StockRanking, USStockRanking, KoreanStockRanking, FundRanking |
| 新闻组件 | 3 | NewsFeedCard, NewsItem, NewsDetailModal |
| 持仓组件 | 3 | PortfolioSummaryCard, PositionList, SectorPieChart |
| AI 组件 | 5 | AIInsightsCard, MarketPredictionCard, StockAnalysisCard, PortfolioOptimizationCard, RiskAssessmentCard |
| 基金组件 | 4 | FundCategories, FundRanking, FundDetail, PeerComparison |
| K 线组件 | 3 | KLineChart, TimeframeSelector, IndicatorSelector |
| 财务组件 | 4 | FinancialData, IncomeStatement, BalanceSheet, CashFlowStatement |
| 详情组件 | 4 | StockDetailPage, USStockDetailPage, KoreanStockDetailPage, FundDetailPage |
| **合计** | **56** | |

### 17.2 组件复用说明

**高复用组件**（被 3+ 页面使用）：
- `StockCard` — 所有股票列表页面
- `IndexCard` — 所有指数展示页面
- `RankingTable` — 所有排行页面
- `FilterBar` — 所有筛选场景
- `ChartContainer` — 所有图表容器
- `EmptyState` — 所有空状态
- `LoadingState` — 所有加载状态
- `ErrorState` — 所有错误状态

**中复用组件**（被 2 页面使用）：
- `StockSearch` — A 股搜索 + 全局搜索
- `NewsFeedCard` — Dashboard 新闻 + 详情页新闻
- `BackNavigation` — 所有三级页面

**低复用组件**（仅 1 页面使用）：
- `MarketHeatmap` — 仅 A 股页面
- `SentimentGauge` — 仅 Dashboard
- `SectorPieChart` — 仅持仓概览

### 17.3 组件层级关系图

```
PortalShell
├── Header
│   ├── BrandMark
│   ├── TabNavigation (6 Tab)
│   ├── GlobalSearch
│   ├── ThemeToggle
│   └── UserMenu
├── TabContent
│   ├── DashboardPage
│   │   ├── MarketTicker
│   │   ├── MarketOverviewCard
│   │   │   ├── IndexCard × 7
│   │   │   └── SentimentGauge
│   │   ├── PortfolioSummaryCard
│   │   │   └── SectorPieChart
│   │   ├── NewsFeedCard
│   │   │   └── NewsItem × N
│   │   ├── AIInsightsCard
│   │   └── WatchlistCard
│   │       └── WatchlistItem × N
│   ├── DomesticPage
│   │   ├── SubPageNavigation
│   │   ├── MarketHeatmap
│   │   ├── SectorMovers
│   │   ├── StockSearch
│   │   ├── StockRanking
│   │   │   └── RankingTable
│   │   └── StockDetailPage (三级)
│   │       ├── BackNavigation
│   │       ├── StockHeader
│   │       ├── KLineChart
│   │       ├── FinancialData
│   │       ├── ValuationAnalysis
│   │       ├── FundFlowAnalysis
│   │       ├── ResearchReports
│   │       └── AIStockAnalysis
│   ├── USPage
│   │   ├── SubPageNavigation
│   │   ├── USMarketOverview
│   │   ├── USStockSearch
│   │   ├── USStockRanking
│   │   └── USStockDetailPage (三级)
│   ├── KoreaPage
│   │   ├── SubPageNavigation
│   │   ├── KOSPIOverview
│   │   ├── KoreanStockSearch
│   │   ├── KoreanStockRanking
│   │   └── KoreanStockDetailPage (三级)
│   ├── FundsPage
│   │   ├── SubPageNavigation
│   │   ├── FundCategories
│   │   ├── FundRanking
│   │   ├── FundSearch
│   │   └── FundDetailPage (三级)
│   │       ├── BackNavigation
│   │       ├── FundHeader
│   │       ├── PerformanceChart
│   │       ├── HoldingsTable
│   │       ├── RiskAnalysis
│   │       ├── StyleDrift
│   │       ├── AIFundRecommendation
│   │       └── PeerComparison
│   └── AIPage
│       ├── SubPageNavigation
│       ├── MarketPredictionCard
│       ├── StockAnalysisCard
│       ├── PortfolioOptimizationCard
│       └── RiskAssessmentCard
└── Footer
```

---

## 十八、验收标准

### 18.1 功能验收标准

**Tab 1: Dashboard**
- [ ] MarketTicker 正确显示 7 个全球指数，60 秒自动刷新
- [ ] MarketOverviewCard 显示 7 个指数卡片 + 恐惧贪婪仪表盘
- [ ] PortfolioSummaryCard 显示总资产、日收益、行业分布饼图
- [ ] NewsFeedCard 显示新闻列表，支持筛选（全部/重大/利好/利空）
- [ ] AIInsightsCard 显示预测摘要、风险提示、机会信号
- [ ] WatchlistCard 显示自选股列表，支持添加/删除

**Tab 2: A股/港股**
- [ ] MarketHeatmap 显示板块热力图，红涨绿跌，点击可交互
- [ ] SectorMovers 显示板块涨跌排行，对称条形图
- [ ] StockSearch 支持代码/名称/拼音搜索，防抖 300ms
- [ ] StockRanking 支持涨幅/跌幅/换手率/成交额排行
- [ ] StockDetail 显示完整股票详情（K 线/财务/估值/资金流向/研报/AI 分析）

**Tab 3: 美股**
- [ ] USMarketOverview 显示三大指数 + 热门板块
- [ ] USStockSearch 支持美股搜索
- [ ] USStockRanking 支持美股排行
- [ ] USStockDetail 显示完整美股详情

**Tab 4: 韩国股票**
- [ ] KOSPIOverview 显示 KOSPI + KOSDAQ 指数
- [ ] KoreanStockSearch 支持韩国股票搜索
- [ ] KoreanStockRanking 支持韩国股票排行
- [ ] KoreanStockDetail 显示完整韩国股票详情

**Tab 5: 基金**
- [ ] FundCategories 显示 5 类基金卡片
- [ ] FundRanking 支持收益率/夏普比率/最大回撤/规模排行
- [ ] FundSearch 支持基金搜索
- [ ] FundDetail 显示完整基金详情（业绩/持仓/风险/风格/AI 推荐/同类对比）

**Tab 6: AI 分析**
- [ ] MarketPrediction 显示三场景预测 + 概率分布
- [ ] StockAnalysis 支持输入股票代码生成 AI 分析报告
- [ ] PortfolioOptimization 支持持仓输入生成优化建议
- [ ] RiskAssessment 支持持仓输入生成风险评估

### 18.2 性能验收标准

| 指标 | 目标 | 说明 |
|------|------|------|
| 首屏加载 | < 2s | 使用骨架屏 + 懒加载 |
| Tab 切换 | < 300ms | 无刷新切换，数据缓存 |
| 数据刷新 | < 1s | 缓存 + 增量更新 |
| 搜索响应 | < 500ms | 防抖 300ms + 缓存 |
| AI 分析 | < 5s | 流式响应 + 加载提示 |
| 包大小 | < 500KB | 代码分割 + 懒加载 |
| Lighthouse | > 90 | 性能评分 |

### 18.3 设计验收标准

| 检查项 | 标准 |
|--------|------|
| 颜色一致性 | 所有颜色使用 design-tokens.css 变量 |
| 字体一致性 | 所有字体使用 Roobert PRO |
| 间距一致性 | 所有间距使用 4px 基准网格 |
| 圆角一致性 | 所有圆角使用 rounded 变量 |
| 日夜模式 | 所有组件支持日间/夜间模式 |
| 响应式 | 桌面/平板/移动端布局正确 |
| 交互反馈 | 所有可交互元素有 hover/active 状态 |
| 加载状态 | 所有异步组件有骨架屏 |
| 空状态 | 所有列表有空状态提示 |
| 错误状态 | 所有数据获取有错误处理 |

---

## 十九、实施计划与里程碑

### 19.1 阶段划分

| 阶段 | 内容 | 预计工期 | 依赖 |
|------|------|----------|------|
| **Phase 0** | 基础设施（路由/状态管理/主题/布局） | 3 天 | 无 |
| **Phase 1** | Dashboard（5 个二级页面） | 5 天 | Phase 0 |
| **Phase 2** | A股/港股（4 个二级 + 1 个三级） | 5 天 | Phase 0 |
| **Phase 3** | 美股（3 个二级 + 1 个三级） | 4 天 | Phase 0 |
| **Phase 4** | 韩国股票（3 个二级 + 1 个三级） | 4 天 | Phase 0 |
| **Phase 5** | 基金（3 个二级 + 1 个三级） | 4 天 | Phase 0 |
| **Phase 6** | AI 分析（4 个二级页面） | 4 天 | Phase 0 |
| **Phase 7** | 全局搜索 + 交互增强 | 3 天 | Phase 1-6 |
| **Phase 8** | 响应式适配 + 性能优化 | 3 天 | Phase 1-6 |
| **Phase 9** | 测试 + Bug 修复 + 文档 | 3 天 | Phase 1-8 |
| **合计** | | **38 天** | |

### 19.2 Phase 0 详细任务

```
Phase 0: 基础设施（3 天）
├── Day 1: 路由系统
│   ├── 修改 App.tsx 路由结构
│   ├── 创建 Tab 导航组件
│   ├── 创建二级页面导航组件
│   └── 实现三级页面返回逻辑
├── Day 2: 状态管理
│   ├── 创建 FinanceContext
│   ├── 实现主题切换
│   ├── 实现数据缓存 Hook
│   └── 实现自动刷新机制
└── Day 3: 布局系统
    ├── 修改 PortalShell 支持 Tab 导航
    ├── 创建 TwoColumnLayout
    ├── 创建 PageContainer
    └── 扩展 design-tokens.css
```

### 19.3 文件结构规划

```
frontend/src/
├── features/
│   └── finance/
│       ├── context/
│       │   ├── FinanceContext.tsx
│       │   ├── ThemeContext.tsx
│       │   └── NavigationContext.tsx
│       ├── hooks/
│       │   ├── useCachedData.ts
│       │   ├── useAutoRefresh.ts
│       │   ├── useMarketData.ts
│       │   ├── usePortfolio.ts
│       │   └── useWatchlist.ts
│       ├── api/
│       │   ├── marketClient.ts
│       │   ├── stockClient.ts
│       │   ├── usClient.ts
│       │   ├── koreaClient.ts
│       │   ├── fundClient.ts
│       │   └── aiClient.ts
│       ├── types/
│       │   ├── market.ts
│       │   ├── stock.ts
│       │   ├── us.ts
│       │   ├── korea.ts
│       │   ├── fund.ts
│       │   └── ai.ts
│       ├── components/
│       │   ├── layout/
│       │   │   ├── TabNavigation.tsx
│       │   │   ├── SubPageNavigation.tsx
│       │   │   ├── BackNavigation.tsx
│       │   │   ├── TwoColumnLayout.tsx
│       │   │   └── PageContainer.tsx
│       │   ├── shared/
│       │   │   ├── StockCard.tsx
│       │   │   ├── IndexCard.tsx
│       │   │   ├── RankingTable.tsx
│       │   │   ├── FilterBar.tsx
│       │   │   ├── ChartContainer.tsx
│       │   │   ├── EmptyState.tsx
│       │   │   ├── LoadingState.tsx
│       │   │   └── ErrorState.tsx
│       │   ├── market/
│       │   │   ├── MarketOverviewCard.tsx
│       │   │   ├── MarketHeatmap.tsx
│       │   │   ├── SectorMovers.tsx
│       │   │   ├── SentimentGauge.tsx
│       │   │   ├── IndexGrid.tsx
│       │   │   └── SectorBar.tsx
│       │   ├── search/
│       │   │   ├── StockSearch.tsx
│       │   │   ├── USStockSearch.tsx
│       │   │   ├── KoreanStockSearch.tsx
│       │   │   └── FundSearch.tsx
│       │   ├── ranking/
│       │   │   ├── StockRanking.tsx
│       │   │   ├── USStockRanking.tsx
│       │   │   ├── KoreanStockRanking.tsx
│       │   │   └── FundRanking.tsx
│       │   ├── news/
│       │   │   ├── NewsFeedCard.tsx
│       │   │   ├── NewsItem.tsx
│       │   │   └── NewsDetailModal.tsx
│       │   ├── portfolio/
│       │   │   ├── PortfolioSummaryCard.tsx
│       │   │   ├── PositionList.tsx
│       │   │   └── SectorPieChart.tsx
│       │   ├── ai/
│       │   │   ├── AIInsightsCard.tsx
│       │   │   ├── MarketPredictionCard.tsx
│       │   │   ├── StockAnalysisCard.tsx
│       │   │   ├── PortfolioOptimizationCard.tsx
│       │   │   └── RiskAssessmentCard.tsx
│       │   ├── funds/
│       │   │   ├── FundCategories.tsx
│       │   │   ├── FundRanking.tsx
│       │   │   ├── FundDetail.tsx
│       │   │   └── PeerComparison.tsx
│       │   ├── kline/
│       │   │   ├── KLineChart.tsx
│       │   │   ├── TimeframeSelector.tsx
│       │   │   └── IndicatorSelector.tsx
│       │   └── detail/
│       │       ├── StockHeader.tsx
│       │       ├── FinancialData.tsx
│       │       ├── ValuationAnalysis.tsx
│       │       ├── FundFlowAnalysis.tsx
│       │       ├── ResearchReports.tsx
│       │       └── AIStockAnalysis.tsx
│       └── pages/
│           ├── DashboardPage.tsx
│           ├── DomesticPage.tsx
│           ├── USPage.tsx
│           ├── KoreaPage.tsx
│           ├── FundsPage.tsx
│           ├── AIPage.tsx
│           ├── StockDetailPage.tsx
│           ├── USStockDetailPage.tsx
│           ├── KoreanStockDetailPage.tsx
│           └── FundDetailPage.tsx
```

### 19.4 依赖库清单

| 库 | 版本 | 用途 |
|----|------|------|
| react-router-dom | ^6.x | 路由管理 |
| recharts | ^2.x | 图表（K 线/饼图/折线图/柱状图） |
| lucide-react | latest | 图标 |
| tailwindcss | ^3.x | 样式 |
| date-fns | ^3.x | 日期格式化 |
| clsx | ^2.x | 条件类名 |
| tailwind-merge | latest | Tailwind 类名合并 |

### 19.5 风险与缓解

| 风险 | 影响 | 缓解措施 |
|------|------|----------|
| API 接口未就绪 | 阻塞开发 | 使用 Mock 数据，定义好接口契约 |
| K 线图性能问题 | 卡顿 | 使用 Canvas 渲染，虚拟滚动 |
| 移动端适配工作量大 | 延期 | 优先桌面端，移动端后续迭代 |
| AI 分析响应慢 | 用户体验差 | 流式响应 + 加载提示 + 超时处理 |
| 数据量大导致渲染慢 | 卡顿 | 虚拟滚动 + 分页 + 懒加载 |

---

## 附录 A：TypeScript 类型定义汇总

### A.1 市场类型

```typescript
type MarketKey = 'cn_stock' | 'hk_stock' | 'us_stock' | 'kr_stock' | 'fund' | 'fx';
type MarketRegion = 'domestic' | 'us' | 'korea';

interface MarketIndex {
  market: MarketRegion;
  name: string;
  symbol: string;
  price: number;
  change: number;
  change_percent: number;
  volume: number;
  updated_at: string;
}

interface MarketSentiment {
  fear_greed_index: number;
  label: '极度恐惧' | '恐惧' | '中性' | '贪婪' | '极度贪婪';
  volume_trend: '放量' | '缩量' | '持平';
  northbound_flow: number;
}
```

### A.2 股票类型

```typescript
interface StockBasicInfo {
  code: string;
  name: string;
  market: MarketKey;
  current_price: number;
  change: number;
  change_percent: number;
  open: number;
  high: number;
  low: number;
  prev_close: number;
  volume: number;
  turnover: number;
  turnover_rate: number;
  market_cap: number;
  pe_ratio: number;
  pb_ratio: number;
  industry: string;
  list_date: string;
}

interface KLineData {
  date: string;
  open: number;
  high: number;
  low: number;
  close: number;
  volume: number;
  ma5?: number;
  ma10?: number;
  ma20?: number;
  ma60?: number;
}

interface RankingStock {
  rank: number;
  code: string;
  name: string;
  current_price: number;
  change: number;
  change_percent: number;
  volume: number;
  turnover: number;
  market_cap: number;
  pe_ratio: number;
  industry: string;
}
```

### A.3 基金类型

```typescript
interface FundBasicInfo {
  code: string;
  name: string;
  type: string;
  manager: string;
  scale: number;
  establish_date: string;
  rating: number;
  benchmark: string;
}

interface RankedFund {
  rank: number;
  code: string;
  name: string;
  type: string;
  manager: string;
  scale: number;
  return_1m: number;
  return_3m: number;
  return_6m: number;
  return_1y: number;
  return_3y: number;
  sharpe_ratio: number;
  max_drawdown: number;
  volatility: number;
}
```

### A.4 AI 类型

```typescript
interface AIStockAnalysisResult {
  code: string;
  name: string;
  recommendation: 'buy' | 'hold' | 'sell';
  confidence: number;
  summary: string;
  technical_analysis: TechnicalAnalysis;
  fundamental_analysis: FundamentalAnalysis;
  sentiment_analysis: SentimentAnalysis;
  risk_factors: string[];
  catalysts: string[];
  price_target: {
    current: number;
    target: number;
    stop_loss: number;
    upside: number;
  };
  generated_at: string;
}

interface MarketPrediction {
  summary: string;
  confidence: number;
  generated_at: string;
  scenarios: Scenario[];
  key_factors: KeyFactor[];
}

interface OptimizationResult {
  current_portfolio: PortfolioMetrics;
  optimized_portfolio: PortfolioMetrics;
  suggestions: RebalanceSuggestion[];
  efficient_frontier: EfficientFrontierPoint[];
  risk_return_analysis: RiskReturnAnalysis;
}

interface RiskAssessmentResult {
  overall_risk: 'low' | 'medium' | 'high';
  risk_score: number;
  stress_test: StressTestResult;
  correlation_analysis: CorrelationAnalysis;
  var_analysis: VaRAnalysis;
  scenario_analysis: ScenarioAnalysis[];
}
```

---

## 附录 B：API 接口详细定义

### B.1 市场概览

```typescript
// GET /api/v1/market/overview
interface MarketOverviewResponse {
  ok: boolean;
  data: {
    indices: MarketIndex[];
    sentiment: MarketSentiment;
    updated_at: string;
  };
}
```

### B.2 股票详情

```typescript
// GET /api/v1/stock/detail/:code?market=cn_stock
interface StockDetailResponse {
  ok: boolean;
  data: {
    basic: StockBasicInfo;
    kline: KLineData[];
    financials: FinancialData;
    valuation: ValuationData;
    fund_flow: FundFlowData;
    research: ResearchReport[];
    news: NewsItem[];
    ai_analysis: AIStockAnalysisResult;
  };
}
```

### B.3 AI 分析

```typescript
// POST /api/v1/ai/stock-analysis
interface StockAnalysisRequest {
  code: string;
  market: MarketKey;
}

interface StockAnalysisResponse {
  ok: boolean;
  data: AIStockAnalysisResult;
}

// POST /api/v1/ai/portfolio-optimization
interface PortfolioOptimizationRequest {
  holdings: { code: string; name: string; weight: number }[];
}

interface PortfolioOptimizationResponse {
  ok: boolean;
  data: OptimizationResult;
}
```

---

> 文档结束 | 共计约 3200 行 | 版本 v4.0
