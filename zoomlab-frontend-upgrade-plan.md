# ZoomLab 金融模块前端详细升级计划

> 版本: v5.0 | 日期: 2026-05-28 | 状态: 规划中
> 范围: 6 大 Tab 页、30+ 二级页面、6 个三级详情页、Dashboard 9 大模块

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
| Dashboard 9 模块 | MarketOverview / PortfolioSummary / NewsFeed / AIDailyPrediction / AIRiskAlerts / AIOpportunitySignals / AIStockRecommendations / AITradingDecisions / Watchlist |
| 专业级 UI | 参照 Bloomberg Terminal / TradingView 的信息密度 |
| 双模式 | 日间模式（白底）+ 夜间模式（深色），平滑切换 |
| 响应式 | 5 断点：≥1280 / 1024-1279 / 768-1023 / 480-767 / <480 |
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
│           │   ├── NewsFeed (近24小时重要信息)
│           │   ├── AIDailyPrediction (AI 今日预测)
│           │   ├── AIRiskAlerts (AI 风险提示)
│           │   ├── AIOpportunitySignals (AI 机会信号)
│           │   ├── AIStockRecommendations (AI 推荐股票)
│           │   ├── AITradingDecisions (AI 买卖决策)
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
| Dashboard | 9 (模块) | 0 | 9 |
| A股/港股 | 4 | 1 (StockDetail) | 5 |
| 美股 | 3 | 1 (USStockDetail) | 4 |
| 韩国股票 | 3 | 1 (KoreanStockDetail) | 4 |
| 基金 | 3 | 1 (FundDetail) | 4 |
| AI 分析 | 4 | 0 | 4 |
| **合计** | **26** | **4** | **30** |

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
  dashboard_ai_prediction: '/dashboard/ai-prediction',
  dashboard_ai_risk: '/dashboard/ai-risk',
  dashboard_ai_opportunity: '/dashboard/ai-opportunity',
  dashboard_ai_recommendations: '/dashboard/ai-recommendations',
  dashboard_ai_decisions: '/dashboard/ai-decisions',
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
      <Route path="ai-prediction" element={<AIDailyPrediction />} />
      <Route path="ai-risk" element={<AIRiskAlerts />} />
      <Route path="ai-opportunity" element={<AIOpportunitySignals />} />
      <Route path="ai-recommendations" element={<AIStockRecommendations />} />
      <Route path="ai-decisions" element={<AITradingDecisions />} />
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

**页面布局**（9 模块）：
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
│  │  (持仓概览)                 │  (近24小时重要信息)      │ │
│  │  ┌───────────────────┐     │  分类筛选 Tab            │ │
│  │  │ 总资产 ¥1,234,567 │     │  ┌─────────────────┐   │ │
│  │  │ 日收益 +¥12,345   │     │  │ Top 3 重要卡片   │   │ │
│  │  │ 收益率 +1.01%     │     │  └─────────────────┘   │ │
│  │  └───────────────────┘     │  时间线新闻列表         │ │
│  │  行业分布饼图               │                         │ │
│  └─────────────────────────────┴─────────────────────────┘ │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  AIDailyPrediction (AI 今日预测)                     │   │
│  │  预测摘要文字 + 置信度徽章                            │   │
│  │  ┌──────┐ ┌──────┐ ┌──────┐                        │   │
│  │  │ 牛市 │ │ 基准 │ │ 熊市 │ 三场景概率分布条形图    │   │
│  │  │ 30%  │ │ 50%  │ │ 20%  │                        │   │
│  │  └──────┘ └──────┘ └──────┘                        │   │
│  │  关键影响因素列表                                    │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  ┌──────────────────────┬──────────────────────────────┐   │
│  │  AIRiskAlerts        │  AIOpportunitySignals        │   │
│  │  (AI 风险提示)       │  (AI 机会信号)                │   │
│  │  风险等级徽章         │  机会等级徽章                  │   │
│  │  风险趋势折线图       │  机会分类筛选                  │   │
│  │  风险因素卡片列表     │  机会卡片列表                  │   │
│  └──────────────────────┴──────────────────────────────┘   │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  AIStockRecommendations (AI 推荐股票)               │   │
│  │  市场筛选 Tab [全部/A股/港股/美股/韩国]              │   │
│  │  股票卡片网格 (推荐强度排序)                          │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  AITradingDecisions (AI 买卖决策)                   │   │
│  │  操作筛选 Tab [全部/买入/卖出/持有]                  │   │
│  │  决策卡片列表 + 免责声明                             │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  Watchlist (自选股)                                  │   │
│  │  600036 招商银行 +1.2%  |  300750 宁德时代 -0.5%    │   │
│  │  005930 三星电子 +2.1%  |  ...                       │   │
│  └─────────────────────────────────────────────────────┘   │
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
│   │   ├── AIDailyPredictionCard
│   │   │   ├── PredictionSummary (预测摘要 + 置信度)
│   │   │   ├── ThreeScenarioChart (三场景概率条形图)
│   │   │   └── KeyFactorsList (关键因素列表)
│   │   ├── AIStockRecommendationsCard
│   │   │   ├── MarketFilterTabs (市场筛选)
│   │   │   └── StockCardGrid (推荐股票网格)
│   │   └── AITradingDecisionsCard
│   │       ├── ActionFilterTabs (操作筛选)
│   │       ├── DecisionCardList (决策卡片列表)
│   │       └── Disclaimer (免责声明)
│   └── RightColumn
│       ├── NewsFeedCard (近24小时重要信息)
│       │   ├── CategoryFilterTabs (分类筛选)
│       │   ├── TopHighlightCards (Top 3 重要卡片)
│       │   └── TimelineList (时间线列表)
│       ├── AIRiskAlertsCard
│       │   ├── RiskLevelBadge (风险等级徽章)
│       │   ├── RiskTrendChart (风险趋势图)
│       │   └── RiskFactorCards (风险因素卡片)
│       ├── AIOpportunitySignalsCard
│       │   ├── OpportunityBadge (机会等级徽章)
│       │   ├── TypeFilterTabs (类型筛选)
│       │   └── OpportunityCards (机会卡片列表)
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

**状态管理**：
```typescript
// 组件内部状态
interface MarketOverviewState {
  indices: MarketIndex[];
  sentiment: MarketSentiment | null;
  loading: boolean;
  error: string | null;
  lastRefreshed: number;
}

// 使用 useReducer 管理
type MarketOverviewAction =
  | { type: 'FETCH_START' }
  | { type: 'FETCH_SUCCESS'; payload: { indices: MarketIndex[]; sentiment: MarketSentiment } }
  | { type: 'FETCH_ERROR'; payload: string }
  | { type: 'INDEX_UPDATE'; payload: Partial<MarketIndex> };
```

**数据流**：
```
GET /api/v1/market/overview
→ { indices: MarketIndex[], sentiment: MarketSentiment }
→ 缓存 60 秒
→ MarketTicker 和 MarketOverviewCard 共享数据
→ 通过 FinanceContext 分发
```

**样式规范**：
- 桌面：7 个 IndexCard 横向排列（4+3 或 7 列）
- 平板：IndexCard 两行排列（4+3）
- 移动端：IndexCard 横向滚动
- 每个 IndexCard：`card-base` 样式，`rounded-lg`，`p-md`
- 价格数字：`tabular-nums`（等宽数字），`font-bold`
- 涨跌颜色：正=`text-emerald-500`，负=`text-red-500`，零=`text-gray-400`
- 恐惧贪婪仪表盘：半圆弧形量表，0-100 渐变色

**响应式设计**：

| 断点 | 宽度 | 布局 |
|------|------|------|
| Wide Desktop | >=1280px | 7 列 IndexCard + 恐惧贪婪仪表盘右侧 |
| Desktop | 1024-1279px | 7 列 IndexCard，仪表盘下方 |
| Tablet | 768-1023px | 两行 4+3 排列 |
| Mobile L | 480-767px | 横向滚动，显示 3 个 |
| Mobile S | <480px | 横向滚动，显示 2 个 |

**交互逻辑**：
- IndexCard hover：`scale(1.02)` + `shadow-md`，持续 200ms
- IndexCard click：跳转到对应市场详情页
- 恐惧贪婪仪表盘：hover 显示数值 tooltip
- 自动刷新：每 60 秒静默刷新，数字变化时闪烁动画

**空状态设计**：
- 图标：`TrendingDown`
- 标题：「暂无市场数据」
- 描述：「市场数据加载中，请稍候」
- 操作：自动重试按钮

**加载状态设计**：
- 7 个 IndexCard 位置显示 `Skeleton` 矩形，`animate-pulse`
- 恐惧贪婪仪表盘显示半圆弧形骨架

**错误处理**：
- 错误信息横幅：红色背景 + 重试按钮
- 部分数据失败：显示已有数据，失败位置显示「--」
- 网络超时：5 秒后显示超时提示 + 手动刷新按钮

**API 调用清单**：
| 项目 | 规格 |
|------|------|
| 请求方法 | GET |
| 端点 | `/api/v1/market/overview` |
| 请求参数 | 无 |
| 响应格式 | `{ ok: boolean, data: { indices: MarketIndex[], sentiment: MarketSentiment }, meta: { request_id, timestamp, cache_hit } }` |
| 缓存 TTL | 60 秒 |
| 失效条件 | 用户手动刷新、页面重新激活（visibilitychange） |
| 降级策略 | API 失败时显示上次缓存数据 + "数据可能过期" 提示 |
| 限流策略 | 前端最少间隔 30 秒请求一次 |

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

**状态管理**：
```typescript
interface PortfolioSummaryState {
  portfolio: PortfolioData | null;
  loading: boolean;
  error: string | null;
  expanded: boolean;  // 是否展开完整持仓列表
}

type PortfolioSummaryAction =
  | { type: 'FETCH_START' }
  | { type: 'FETCH_SUCCESS'; payload: PortfolioData }
  | { type: 'FETCH_ERROR'; payload: string }
  | { type: 'TOGGLE_EXPAND' };
```

**数据流**：
```
GET /api/v1/portfolio/summary
→ PortfolioData
→ 缓存 30 秒（持仓数据需要实时性）
→ 未登录时显示引导提示
→ 通过 FinanceContext.refreshPortfolio() 触发刷新
```

**样式规范**：
- 总资产大数字：`text-3xl font-bold tabular-nums`
- 日收益：正=`text-emerald-500`，负=`text-red-500`，前缀 +/- 符号
- 行业分布饼图：使用 Recharts PieChart，颜色映射 8 个行业
- 持仓列表：最多显示 Top 5，点击"查看更多"展开

**响应式设计**：

| 断点 | 宽度 | 布局 |
|------|------|------|
| Wide Desktop | >=1280px | 左侧资产摘要 + 右侧饼图 |
| Desktop | 1024-1279px | 同上，饼图略小 |
| Tablet | 768-1023px | 上下排列，饼图全宽 |
| Mobile L | 480-767px | 紧凑模式，隐藏饼图，仅数字 |
| Mobile S | <480px | 极简模式，仅总资产 + 日收益 |

**交互逻辑**：
- 卡片 hover：整体 `shadow-md` 效果
- "查看更多" 按钮：展开/收起完整持仓列表，动画 300ms
- 饼图 sector hover：高亮对应扇区 + tooltip 显示占比
- 点击持仓项：跳转到对应股票详情页

**空状态设计**：
- 图标：`Wallet`
- 标题：「暂无持仓」
- 描述：「登录后可查看持仓概览和收益分析」
- 操作：「立即登录」按钮

**加载状态设计**：
- 资产数字位置显示 Skeleton 文本
- 饼图位置显示圆形骨架
- 持仓列表显示 3 行 Skeleton 行

**错误处理**：
- 未登录：显示登录引导（非错误态）
- 网络错误：显示错误横幅 + 重试按钮
- 数据异常：数字显示「--」，饼图显示空态

**API 调用清单**：
| 项目 | 规格 |
|------|------|
| 请求方法 | GET |
| 端点 | `/api/v1/portfolio/summary` |
| 请求头 | `Authorization: Bearer {token}` |
| 响应格式 | `{ ok: boolean, data: PortfolioData }` |
| 缓存 TTL | 30 秒 |
| 失效条件 | 用户交易后、手动刷新 |
| 降级策略 | 未登录显示引导，网络错误显示缓存 |
| 限流策略 | 最少间隔 15 秒 |

### 4.4 NewsFeed 近24小时重要信息

**文件**：`features/finance/components/news/NewsFeedCard.tsx`

> **重大变更**：从简单新闻列表升级为「近24小时重要信息」模块，增加时间范围限制、重要性排序、多维分类筛选。

**Props 设计**：
```typescript
interface NewsFeedCardProps {
  items: NewsItem[];
  loading: boolean;
  error: string | null;
  activeCategory: NewsCategory;
  onCategoryChange: (category: NewsCategory) => void;
  onItemClick: (item: NewsItem) => void;
  onLoadMore?: () => void;
  hasMore: boolean;
  lastUpdated: string;
}

type NewsCategory = 'all' | 'macro' | 'industry' | 'company' | 'anomaly';

interface NewsItem {
  id: string;
  title: string;
  summary: string;
  source: string;
  published_at: string;
  time_ago: string;           // 相对时间，如 "2小时前"
  importance: 'major' | 'notable' | 'general';
  sentiment: 'positive' | 'negative' | 'neutral';
  sentiment_score: number;    // -1 到 1
  category: NewsCategory;
  category_label: string;     // 宏观政策 / 行业动态 / 公司公告 / 市场异动
  related_stocks: RelatedStock[];
  impact_sectors: string[];
  url: string;
  ai_summary: string;
}

interface RelatedStock {
  code: string;
  name: string;
  change_percent?: number;
}
```

**状态管理**：
```typescript
interface NewsFeedState {
  items: NewsItem[];
  topHighlights: NewsItem[];    // Top 3 重要新闻
  timelineItems: NewsItem[];    // 其余按时间排序
  activeCategory: NewsCategory;
  loading: boolean;
  error: string | null;
  hasMore: boolean;
  offset: number;
  lastUpdated: string;
}

type NewsFeedAction =
  | { type: 'FETCH_START' }
  | { type: 'FETCH_SUCCESS'; payload: { items: NewsItem[]; total: number; has_more: boolean } }
  | { type: 'FETCH_ERROR'; payload: string }
  | { type: 'CHANGE_CATEGORY'; payload: NewsCategory }
  | { type: 'LOAD_MORE'; payload: NewsItem[] }
  | { type: 'SET_OFFSET'; payload: number };

// 数据处理逻辑
function processNewsItems(items: NewsItem[]): { topHighlights: NewsItem[]; timelineItems: NewsItem[] } {
  // 按重要性排序：major > notable > general
  const sorted = [...items].sort((a, b) => {
    const importanceOrder = { major: 0, notable: 1, general: 2 };
    return importanceOrder[a.importance] - importanceOrder[b.importance];
  });
  // Top 3 为重要性最高的，其余按时间倒序
  const topHighlights = sorted.slice(0, 3);
  const timelineItems = items
    .filter(item => !topHighlights.find(th => th.id === item.id))
    .sort((a, b) => new Date(b.published_at).getTime() - new Date(a.published_at).getTime());
  return { topHighlights, timelineItems };
}
```

**数据流**：
```
GET /api/v1/news?category={category}&hours=24&limit=20&offset=0
→ { items: NewsItem[], total: number, has_more: boolean }
→ 前端排序：Top 3 按重要性，其余按时间倒序
→ 缓存 120 秒
→ 支持分页加载（滚动到底自动加载）
```

**布局实现**：
```
┌─────────────────────────────────────────────┐
│  近24小时重要信息              更新于 14:32  │
├─────────────────────────────────────────────┤
│  [全部] [宏观] [行业] [公司] [异动]          │
├─────────────────────────────────────────────┤
│  ┌────────────────────────────────────────┐ │
│  │ ⭐ 央行宣布降准 50 个基点              │ │
│  │ 宏观政策 | 重大 | 利好                  │ │
│  │ 为支持实体经济发展，中国人民银行决定...  │ │
│  │ 影响: 银行 保险 房地产                   │ │
│  │ 相关: 600036 招商银行 +2.1%            │ │
│  └────────────────────────────────────────┘ │
│  ┌────────────────────────────────────────┐ │
│  │ ⭐ 美联储会议纪要释放鹰派信号           │ │
│  │ 宏观政策 | 重大 | 利空                  │ │
│  └────────────────────────────────────────┘ │
│  ┌────────────────────────────────────────┐ │
│  │ ⭐ 宁德时代发布新一代麒麟电池            │ │
│  │ 公司公告 | 重大 | 利好                  │ │
│  └────────────────────────────────────────┘ │
├─────────────────────────────────────────────┤
│  ──── 14:00 ────                            │
│  · A股三大指数集体收涨                       │
│  · 半导体板块资金净流入居前                  │
│  ──── 12:00 ────                            │
│  · 午间休市快讯汇总                          │
│  ──── 10:00 ────                            │
│  · 北向资金早盘净流入 35 亿                  │
│  · 比亚迪 3 月销量创新高                     │
│  ──── 08:00 ────                            │
│  · 隔夜美股三大指数涨跌不一                  │
│  · 亚太市场早盘前瞻                          │
└─────────────────────────────────────────────┘
```

**样式规范**：
- 时间线布局：左侧竖线 + 圆点标记，时间分组标签
- 重大新闻卡片：左侧黄色竖线 + 加粗标题 + 展开的摘要
- 利好新闻：左侧绿色边框 + 绿色圆点
- 利空新闻：左侧红色边框 + 红色圆点
- 中性新闻：左侧灰色边框 + 灰色圆点
- Top 3 卡片：`card-finance` 样式，`border-l-4`（颜色随情感）
- 时间线分组：灰色小字时间标签 + 细线分隔
- 顶部筛选：pill-tab 样式按钮组
- 关联股票标签：小型 pill badge，显示代码 + 涨跌幅

**响应式设计**：

| 断点 | 宽度 | 布局 |
|------|------|------|
| Wide Desktop | >=1280px | Top 3 卡片横排，时间线列表右侧 |
| Desktop | 1024-1279px | Top 3 卡片纵排，时间线列表下方 |
| Tablet | 768-1023px | Top 3 纵排，时间线列表全宽 |
| Mobile L | 480-767px | 仅时间线列表，Top 3 混入时间线 |
| Mobile S | <480px | 紧凑列表，隐藏摘要，仅标题 + 来源 + 时间 |

**交互逻辑**：
- 分类 Tab 切换：pill-tab 动画，切换时淡入淡出 200ms
- 新闻卡片 hover：`shadow-md` + 微上移 2px
- 新闻卡片 click：展开完整摘要（如果是列表态）或跳转详情
- 关联股票 pill click：跳转对应股票详情页
- 滚动到底：自动触发 `onLoadMore`，底部显示 loading spinner
- 下拉刷新：移动端支持下拉刷新（`pull-to-refresh`）
- 重要性徽章：major=红色 + 脉冲动画，notable=橙色，general=灰色

**空状态设计**：
- 图标：`Newspaper`
- 标题：「暂无近期重要信息」
- 描述：「近 24 小时暂无匹配的重要信息」
- 操作：切换分类筛选或手动刷新

**加载状态设计**：
- Top 3 区域：3 个 `Skeleton` 卡片，`animate-pulse`
- 时间线区域：5 行 `Skeleton` 行，左侧圆点 + 右侧文本行
- 首次加载显示全屏骨架屏，分页加载显示底部 spinner

**错误处理**：
- 网络错误：红色错误横幅 + 重试按钮
- 部分加载失败：保留已加载数据，底部显示「加载失败，点击重试」
- 数据为空：根据当前分类显示对应空状态

**API 调用清单**：
| 项目 | 规格 |
|------|------|
| 请求方法 | GET |
| 端点 | `/api/v1/news` |
| 请求参数 | `category?: NewsCategory`, `hours: 24`, `limit: 20`, `offset: 0` |
| 响应格式 | `{ ok: boolean, data: { items: NewsItem[], total: number, has_more: boolean } }` |
| 缓存 TTL | 120 秒 |
| 失效条件 | 分类切换、手动刷新、超过 TTL |
| 降级策略 | API 失败时显示缓存数据 + "信息可能不是最新" 提示 |
| 限流策略 | 分页加载间隔最少 2 秒，全量刷新间隔最少 60 秒 |


### 4.5 AIDailyPrediction AI 今日预测

**文件**：`features/finance/components/ai/AIDailyPredictionCard.tsx`

**Props 设计**：
```typescript
interface AIDailyPredictionCardProps {
  prediction: AIDailyPrediction | null;
  loading: boolean;
  error: string | null;
  onRefresh?: () => void;
}

interface AIDailyPrediction {
  id: string;
  summary: string;                    // 预测摘要段落（2-3 句话）
  confidence: number;                 // 置信度 0-100
  generated_at: string;               // 生成时间
  model_version: string;              // 模型版本号
  scenarios: ScenarioDistribution;    // 三场景概率分布
  key_factors: KeyFactor[];           // 关键影响因素（3-5 个）
  prediction_basis: PredictionBasis;  // 预测依据
  historical_accuracy: HistoricalAccuracy;  // 历史准确率
}

interface ScenarioDistribution {
  bull: ScenarioCard;   // 牛市场景
  base: ScenarioCard;   // 基准场景
  bear: ScenarioCard;   // 熊市场景
  total_probability: number;  // 应为 100
}

interface ScenarioCard {
  label: string;            // "牛市" / "基准" / "熊市"
  probability: number;      // 百分比 0-100
  description: string;      // 场景描述（1-2 句话）
  expected_change: number;  // 预期涨跌幅 %
  color: string;            // 场景颜色标识
}

interface KeyFactor {
  id: string;
  name: string;             // 因素名称，如 "央行货币政策"
  impact: 'positive' | 'negative' | 'neutral';
  weight: number;           // 权重 0-1
  description: string;      // 影响说明
}

interface PredictionBasis {
  data_sources: string[];   // 数据来源列表
  analysis_period: string;  // 分析周期
  model_type: string;       // 模型类型
}

interface HistoricalAccuracy {
  total_predictions: number;       // 总预测次数
  correct_predictions: number;     // 正确次数
  accuracy_rate: number;           // 准确率 0-100
  last_30_days_accuracy: number;   // 近 30 天准确率
  accuracy_trend: 'improving' | 'stable' | 'declining';
}
```

**状态管理**：
```typescript
interface AIDailyPredictionState {
  prediction: AIDailyPrediction | null;
  loading: boolean;
  error: string | null;
  showDetails: boolean;    // 是否展开详细信息
}

type AIDailyPredictionAction =
  | { type: 'FETCH_START' }
  | { type: 'FETCH_SUCCESS'; payload: AIDailyPrediction }
  | { type: 'FETCH_ERROR'; payload: string }
  | { type: 'TOGGLE_DETAILS' };
```

**数据流**：
```
GET /api/v1/ai/daily-prediction
→ AIDailyPrediction
→ 缓存 600 秒（AI 预测更新频率低）
→ 生成时间显示在卡片底部
→ 历史准确率独立请求: GET /api/v1/ai/prediction/accuracy
```

**布局实现**：
```
┌─────────────────────────────────────────────────────┐
│  AI 今日预测                     置信度: 72% [中高] │
│  ─────────────────────────────────────────────────  │
│  市场短期偏向震荡整理，受美联储鹰派预期和国内      │
│  降准利好博弈影响，预计沪指在 3200-3300 区间运行。  │
│                                                     │
│  ┌──────────┬──────────┬──────────┐                 │
│  │  🟢 牛市  │  🔵 基准  │  🔴 熊市  │                 │
│  │   30%    │   50%    │   20%    │                 │
│  │ +2.5%    │  ±0.5%   │ -1.8%    │                 │
│  │ 站上3300 │ 区间震荡  │ 跌破3150 │                 │
│  └──────────┴──────────┴──────────┘                 │
│                                                     │
│  关键因素:                                           │
│  ● 央行降准50bp (正面) ────────────── 权重 35%      │
│  ● 美联储鹰派信号 (负面) ──────────── 权重 25%      │
│  ● 北向资金持续流入 (正面) ────────── 权重 20%      │
│  ● A股成交量萎缩 (负面) ──────────── 权重 15%      │
│  ● 地缘政治风险 (中性) ────────────── 权重 5%       │
│                                                     │
│  ▼ 展开详情 (预测依据 / 历史准确率)                  │
│                                                     │
│  生成时间: 2026-05-28 08:00 | 模型: ZL-Predict v3.2│
└─────────────────────────────────────────────────────┘
```

**样式规范**：
- 置信度徽章：pill 样式，颜色映射（>=70%=绿，40-69%=黄，<40%=红）
- 三场景卡片：横向排列，各自边框颜色（牛=绿，基准=蓝，熊=红）
- 概率数字：`text-2xl font-bold tabular-nums`
- 关键因素列表：圆点 + 名称 + 影响方向色标 + 权重条形图
- 权重条形图：进度条样式，宽度 = 权重百分比
- 生成时间/模型版本：底部小字灰色 `text-xs text-gray-400`

**响应式设计**：

| 断点 | 宽度 | 布局 |
|------|------|------|
| Wide Desktop | >=1280px | 三场景卡片横排，因素列表单列 |
| Desktop | 1024-1279px | 三场景卡片横排 |
| Tablet | 768-1023px | 三场景卡片纵向堆叠 |
| Mobile L | 480-767px | 三场景卡片纵向堆叠，隐藏权重条形图 |
| Mobile S | <480px | 仅显示摘要 + 总概率条形图 |

**交互逻辑**：
- 置信度徽章 hover：tooltip 显示准确率详情
- 场景卡片 hover：放大 `scale(1.02)` + 阴影增强
- 场景卡片 click：展开该场景的详细触发条件
- "展开详情" 折叠按钮：300ms 高度动画展开/收起
- 刷新按钮：手动触发重新预测（需等待 3-5 秒）
- 因素列表 hover：高亮对应因素描述

**空状态设计**：
- 图标：`Brain`
- 标题：「今日预测尚未生成」
- 描述：「AI 预测通常在每日 08:00 前生成」
- 操作：「手动触发预测」按钮（限频 1 次/小时）

**加载状态设计**：
- 摘要区域：2 行 Skeleton 文本
- 三场景卡片：3 个矩形 Skeleton，等宽排列
- 因素列表：5 行 Skeleton 行
- 加载提示：「AI 正在分析市场数据...」

**错误处理**：
- 预测生成失败：显示「预测服务暂时不可用」+ 预计恢复时间
- 模型超时（>10s）：显示超时提示 + 自动重试
- 部分字段缺失：已有字段正常显示，缺失字段显示「--」

**API 调用清单**：
| 项目 | 规格 |
|------|------|
| 请求方法 | GET |
| 端点 | `/api/v1/ai/daily-prediction` |
| 请求参数 | 无 |
| 响应格式 | `{ ok: boolean, data: AIDailyPrediction }` |
| 缓存 TTL | 600 秒 |
| 失效条件 | 超过 TTL、手动刷新 |
| 降级策略 | 显示上一次预测结果 + "数据可能过期" 标记 |
| 限流策略 | 手动触发最少间隔 3600 秒（1 小时） |
| 超时设置 | 15 秒（AI 生成可能较慢） |

### 4.6 AIRiskAlerts AI 风险提示

**文件**：`features/finance/components/ai/AIRiskAlertsCard.tsx`

**Props 设计**：
```typescript
interface AIRiskAlertsCardProps {
  data: AIRiskAlerts | null;
  loading: boolean;
  error: string | null;
  onRefresh?: () => void;
}

interface AIRiskAlerts {
  id: string;
  risk_level: RiskLevel;                    // 整体风险等级
  risk_score: number;                       // 风险分数 0-100
  generated_at: string;
  risk_factors: RiskFactor[];               // 风险因素列表（5-10 个）
  historical_comparison: HistoricalRiskComparison;  // 历史风险事件对比
  risk_trend: RiskTrendData;                // 30 天风险趋势
}

type RiskLevel = 'low' | 'medium' | 'high' | 'extreme';

interface RiskFactor {
  id: string;
  name: string;
  description: string;
  impact_level: 'critical' | 'high' | 'medium' | 'low';
  affected_sectors: string[];
  probability: number;                      // 发生概率 0-100
  potential_impact: string;                 // 潜在影响描述
  mitigation: string;                       // 应对建议
}

interface HistoricalRiskComparison {
  similar_events: HistoricalEvent[];
  avg_recovery_days: number;
  worst_case_loss: number;
}

interface HistoricalEvent {
  date: string;
  event_name: string;
  risk_level: RiskLevel;
  market_impact: number;                    // 当时市场涨跌幅 %
  recovery_days: number;
}

interface RiskTrendData {
  daily_risk_scores: { date: string; score: number; level: RiskLevel }[];
  trend_direction: 'rising' | 'stable' | 'falling';
  current_vs_30d_avg: number;              // 当前分数与 30 天均值的差值
}
```

**状态管理**：
```typescript
interface AIRiskAlertsState {
  data: AIRiskAlerts | null;
  loading: boolean;
  error: string | null;
  expandedFactorId: string | null;   // 当前展开的风险因素 ID
  showHistory: boolean;              // 是否显示历史对比
}

type AIRiskAlertsAction =
  | { type: 'FETCH_START' }
  | { type: 'FETCH_SUCCESS'; payload: AIRiskAlerts }
  | { type: 'FETCH_ERROR'; payload: string }
  | { type: 'EXPAND_FACTOR'; payload: string | null }
  | { type: 'TOGGLE_HISTORY' };
```

**数据流**：
```
GET /api/v1/ai/risk-alerts
→ AIRiskAlerts
→ 缓存 300 秒
→ 趋势数据独立请求: GET /api/v1/ai/risk-trend?days=30
→ 历史对比独立请求: GET /api/v1/ai/risk-history
```

**布局实现**：
```
┌─────────────────────────────────────────────────────┐
│  AI 风险提示                                       │
│  ┌──────────┐                                       │
│  │ ⚠️ 中等  │  风险分数: 62/100    趋势: ↑ 上升    │
│  │  风险    │  较30天均值 +8分                      │
│  └──────────┘                                       │
│                                                     │
│  ┌─────────────────────────────────────────────┐   │
│  │  30 天风险趋势图                             │   │
│  │  ▁▂▃▃▄▅▅▆▆▇▇█▇▇▆▆▅▅▄▃▃▂▂▁▁                 │   │
│  │  低 ─────────── 中 ─────────── 高           │   │
│  └─────────────────────────────────────────────┘   │
│                                                     │
│  风险因素:                                           │
│  ┌─────────────────────────────────────────────┐   │
│  │ 🔴 美联储加息预期升温       严重 | 概率 75% │   │
│  │ 影响板块: 银行 地产 科技                      │   │
│  │ ▼ 展开详情                                   │   │
│  └─────────────────────────────────────────────┘   │
│  ┌─────────────────────────────────────────────┐   │
│  │ 🟡 人民币汇率波动加大       高   | 概率 60% │   │
│  │ 影响板块: 出口 外贸 航运                      │   │
│  └─────────────────────────────────────────────┘   │
│  ┌─────────────────────────────────────────────┐   │
│  │ 🟡 地缘政治不确定性         中   | 概率 45% │   │
│  │ 影响板块: 军工 能源 黄金                      │   │
│  └─────────────────────────────────────────────┘   │
│  ... (更多风险因素)                                 │
│                                                     │
│  ▶ 历史风险事件对比                                  │
└─────────────────────────────────────────────────────┘
```

**样式规范**：
- 风险等级徽章：大号 pill 样式
  - `low`: 绿色背景 + "低风险" 文字
  - `medium`: 黄色背景 + "中等风险" 文字
  - `high`: 橙色背景 + "高风险" 文字
  - `extreme`: 红色背景 + 脉冲动画 + "极高风险" 文字
- 风险趋势图：Recharts AreaChart，颜色随风险等级变化
- 风险因素卡片：左侧彩色竖线（严重=红，高=橙，中=黄，低=绿）
- 影响概率：进度条样式，宽度 = 概率百分比
- 影响板块标签：小型 pill badge，灰色背景

**响应式设计**：

| 断点 | 宽度 | 布局 |
|------|------|------|
| Wide Desktop | >=1280px | 左侧风险等级 + 趋势图，右侧因素列表 |
| Desktop | 1024-1279px | 上下排列，趋势图全宽 |
| Tablet | 768-1023px | 趋势图全宽，因素卡片纵向排列 |
| Mobile L | 480-767px | 紧凑模式，隐藏趋势图 |
| Mobile S | <480px | 仅风险等级 + 前 3 个因素 |

**交互逻辑**：
- 风险等级徽章 hover：tooltip 显示风险分数详情
- 风险因素卡片 click：展开/收起详细描述和应对建议
- 趋势图 hover：显示当日风险分数 tooltip
- "历史对比" 折叠面板：展开显示历史相似事件列表
- 刷新按钮：重新获取风险评估

**空状态设计**：
- 图标：`ShieldCheck`
- 标题：「暂无风险提示」
- 描述：「当前市场风险较低，暂无重大风险因素」

**加载状态设计**：
- 风险等级区域：矩形 Skeleton
- 趋势图：波浪形 Skeleton
- 因素列表：5 行 Skeleton 卡片

**错误处理**：
- 服务不可用：显示「风险评估服务暂时不可用」
- 数据过期：显示缓存数据 + 「数据可能不是最新」标记
- 部分加载失败：已有数据正常显示

**API 调用清单**：
| 项目 | 规格 |
|------|------|
| 请求方法 | GET |
| 端点 | `/api/v1/ai/risk-alerts` |
| 请求参数 | 无 |
| 响应格式 | `{ ok: boolean, data: AIRiskAlerts }` |
| 缓存 TTL | 300 秒 |
| 失效条件 | 超过 TTL、重大市场事件触发 |
| 降级策略 | 显示缓存数据 + 过期标记 |
| 限流策略 | 最少间隔 120 秒 |

### 4.7 AIOpportunitySignals AI 机会信号

**文件**：`features/finance/components/ai/AIOpportunitySignalsCard.tsx`

**Props 设计**：
```typescript
interface AIOpportunitySignalsCardProps {
  data: AIOpportunitySignals | null;
  loading: boolean;
  error: string | null;
  activeType: OpportunityType;
  onTypeChange: (type: OpportunityType) => void;
  onRefresh?: () => void;
}

type OpportunityType = 'all' | 'sector' | 'stock' | 'rotation' | 'event';

interface AIOpportunitySignals {
  id: string;
  opportunity_level: OpportunityLevel;
  score: number;                              // 机会分数 0-100
  generated_at: string;
  signals: OpportunitySignal[];               // 机会信号列表（5-10 个）
  historical_performance: HistoricalPerformance;
}

type OpportunityLevel = 'low' | 'medium' | 'high' | 'very_high';

interface OpportunitySignal {
  id: string;
  name: string;
  description: string;
  type: OpportunityType;
  type_label: string;                         // "行业" / "个股" / "板块轮动" / "事件驱动"
  expected_return: number;                    // 预期收益率 %
  risk_level: 'low' | 'medium' | 'high';
  confidence: number;                         // 置信度 0-100
  related_stocks: RelatedStock[];
  time_horizon: 'short' | 'medium' | 'long'; // 持有周期
  catalyst: string;                           // 催化剂描述
  entry_point?: string;                       // 建议入场点
}

interface HistoricalPerformance {
  total_signals: number;
  successful_signals: number;
  avg_return: number;                         // 平均收益率 %
  win_rate: number;                           // 胜率 %
  best_signal: { name: string; return: number; date: string };
  recent_signals: RecentSignal[];
}

interface RecentSignal {
  name: string;
  type: OpportunityType;
  expected_return: number;
  actual_return?: number;                     // 如果已到期
  status: 'active' | 'expired' | 'hit_target';
}
```

**状态管理**：
```typescript
interface AIOpportunitySignalsState {
  data: AIOpportunitySignals | null;
  loading: boolean;
  error: string | null;
  activeType: OpportunityType;
  expandedSignalId: string | null;
  showPerformance: boolean;
}

type AIOpportunitySignalsAction =
  | { type: 'FETCH_START' }
  | { type: 'FETCH_SUCCESS'; payload: AIOpportunitySignals }
  | { type: 'FETCH_ERROR'; payload: string }
  | { type: 'CHANGE_TYPE'; payload: OpportunityType }
  | { type: 'EXPAND_SIGNAL'; payload: string | null }
  | { type: 'TOGGLE_PERFORMANCE' };
```

**数据流**：
```
GET /api/v1/ai/opportunity-signals
→ AIOpportunitySignals
→ 缓存 300 秒
→ 历史表现独立请求: GET /api/v1/ai/opportunity-performance
→ 前端按 type 过滤信号列表
```

**布局实现**：
```
┌─────────────────────────────────────────────────────┐
│  AI 机会信号                   机会等级: ★★★★ 高   │
│  ─────────────────────────────────────────────────  │
│  [全部] [行业] [个股] [轮动] [事件]                  │
│                                                     │
│  ┌─────────────────────────────────────────────┐   │
│  │ 🟢 半导体板块超跌反弹     行业 | 预期+8%   │   │
│  │ 置信度: 78% | 风险: 中 | 周期: 中期         │   │
│  │ 催化剂: 国产替代政策加码 + 库存周期见底       │   │
│  │ 相关: 688981 中芯国际  002371 北方华创       │   │
│  └─────────────────────────────────────────────┘   │
│  ┌─────────────────────────────────────────────┐   │
│  │ 🟢 新能源车销量超预期     个股 | 预期+12%  │   │
│  │ 置信度: 72% | 风险: 中高 | 周期: 短期       │   │
│  └─────────────────────────────────────────────┘   │
│  ┌─────────────────────────────────────────────┐   │
│  │ 🟡 大盘蓝筹轮动机会       轮动 | 预期+5%   │   │
│  │ 置信度: 65% | 风险: 低 | 周期: 长期         │   │
│  └─────────────────────────────────────────────┘   │
│  ... (更多信号)                                     │
│                                                     │
│  ▶ 历史信号表现 (胜率 72%, 平均收益 +4.5%)         │
└─────────────────────────────────────────────────────┘
```

**样式规范**：
- 机会等级徽章：大号 pill 样式，颜色映射（low=灰，medium=蓝，high=绿，very_high=金黄 + 脉冲）
- 类型筛选 Tab：pill-tab 样式，横排
- 机会信号卡片：左侧绿色渐变边框，内部显示名称 + 类型 + 预期收益
- 预期收益数字：`text-emerald-500 font-bold`
- 风险等级标签：pill 样式（低=绿，中=黄，高=红）
- 置信度：进度条 + 百分比数字
- 相关股票：pill badge，点击可跳转

**响应式设计**：

| 断点 | 宽度 | 布局 |
|------|------|------|
| Wide Desktop | >=1280px | 机会卡片网格 2 列 |
| Desktop | 1024-1279px | 机会卡片单列 |
| Tablet | 768-1023px | 机会卡片单列，隐藏催化剂 |
| Mobile L | 480-767px | 紧凑卡片，仅名称 + 类型 + 收益 |
| Mobile S | <480px | 列表视图，仅一行显示 |

**交互逻辑**：
- 类型 Tab 切换：前端过滤，无网络请求
- 信号卡片 hover：`shadow-md` + 微上移
- 信号卡片 click：展开详细信息（催化剂 + 入场建议）
- 相关股票 pill click：跳转股票详情页
- 历史表现展开：折叠面板动画

**空状态设计**：
- 图标：`Target`
- 标题：「暂无机会信号」
- 描述：「当前市场暂无明显投资机会，建议观望」
- 操作：「刷新信号」按钮

**加载状态设计**：
- 机会等级区域：矩形 Skeleton
- 信号卡片：3 个 Skeleton 卡片
- 类型 Tab：4 个圆形 Skeleton

**错误处理**：
- 服务不可用：显示「机会分析服务暂时不可用」
- 部分字段缺失：显示已有内容，缺失项显示「--」
- 历史数据不可用：隐藏历史表现区域

**API 调用清单**：
| 项目 | 规格 |
|------|------|
| 请求方法 | GET |
| 端点 | `/api/v1/ai/opportunity-signals` |
| 请求参数 | 无 |
| 响应格式 | `{ ok: boolean, data: AIOpportunitySignals }` |
| 缓存 TTL | 300 秒 |
| 失效条件 | 超过 TTL、手动刷新 |
| 降级策略 | 显示缓存数据 + 过期标记 |
| 限流策略 | 最少间隔 120 秒 |


### 4.8 AIStockRecommendations AI 推荐股票

**文件**：`features/finance/components/ai/AIStockRecommendationsCard.tsx`

**Props 设计**：
```typescript
interface AIStockRecommendationsCardProps {
  stocks: AIRecommendedStock[];
  loading: boolean;
  error: string | null;
  activeMarket: MarketFilter;
  onMarketChange: (market: MarketFilter) => void;
  onStockClick: (stock: AIRecommendedStock) => void;
  onLoadMore?: () => void;
  hasMore: boolean;
}

type MarketFilter = 'all' | 'cn' | 'hk' | 'us' | 'kr';

interface AIRecommendedStock {
  code: string;
  name: string;
  market: MarketKey;
  market_label: string;               // "A股" / "港股" / "美股" / "韩国"
  current_price: number;
  change_percent: number;
  recommendation: RecommendationLevel;
  recommendation_label: string;       // "强推" / "推荐" / "关注"
  reason: string;                     // 推荐理由（1-2 句话）
  target_price: number;
  expected_return: number;            // 预期收益率 %
  confidence: number;                 // 置信度 0-100
  related_news_count: number;         // 关联新闻数量
  sector: string;
  market_cap: number;
}

type RecommendationLevel = 'strong_buy' | 'buy' | 'watch';
```

**状态管理**：
```typescript
interface AIStockRecommendationsState {
  stocks: AIRecommendedStock[];
  filteredStocks: AIRecommendedStock[];
  activeMarket: MarketFilter;
  loading: boolean;
  error: string | null;
  hasMore: boolean;
  offset: number;
  sortBy: 'recommendation' | 'return' | 'confidence';
}

type AIStockRecommendationsAction =
  | { type: 'FETCH_START' }
  | { type: 'FETCH_SUCCESS'; payload: { stocks: AIRecommendedStock[]; has_more: boolean } }
  | { type: 'FETCH_ERROR'; payload: string }
  | { type: 'CHANGE_MARKET'; payload: MarketFilter }
  | { type: 'CHANGE_SORT'; payload: 'recommendation' | 'return' | 'confidence' }
  | { type: 'LOAD_MORE'; payload: AIRecommendedStock[] }
  | { type: 'SET_OFFSET'; payload: number };

// 过滤和排序逻辑
function filterAndSort(
  stocks: AIRecommendedStock[],
  market: MarketFilter,
  sortBy: string
): AIRecommendedStock[] {
  let filtered = market === 'all' ? stocks : stocks.filter(s => s.market === market);
  const sortMap = {
    recommendation: (a: AIRecommendedStock, b: AIRecommendedStock) => {
      const order = { strong_buy: 0, buy: 1, watch: 2 };
      return order[a.recommendation] - order[b.recommendation];
    },
    return: (a: AIRecommendedStock, b: AIRecommendedStock) => b.expected_return - a.expected_return,
    confidence: (a: AIRecommendedStock, b: AIRecommendedStock) => b.confidence - a.confidence,
  };
  return filtered.sort(sortMap[sortBy as keyof typeof sortMap] || sortMap.recommendation);
}
```

**数据流**：
```
GET /api/v1/ai/stock-recommendations?market={market}&limit=20&offset=0
→ { stocks: AIRecommendedStock[], has_more: boolean }
→ 缓存 300 秒
→ 市场切换时前端过滤（如果数据已加载）
→ 分页加载（滚动到底触发）
```

**布局实现**：
```
┌─────────────────────────────────────────────────────┐
│  AI 推荐股票                                       │
│  [全部] [A股] [港股] [美股] [韩国]                  │
│                                                     │
│  ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐              │
│  │ 强推 │ │ 强推 │ │ 推荐 │ │ 推荐 │              │
│  │600036│ │688981│ │300750│ │005930│              │
│  │招商银行│ │中芯国际│ │宁德时代│ │三星电子│              │
│  │+15%  │ │+12%  │ │+8%   │ │+10%  │              │
│  │置信85%│ │置信78%│ │置信72%│ │置信68%│              │
│  └──────┘ └──────┘ └──────┘ └──────┘              │
│  ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐              │
│  │ 推荐 │ │ 关注 │ │ 关注 │ │ ...  │              │
│  │AAPL  │ │TSLA  │ │MSFT  │ │      │              │
│  └──────┘ └──────┘ └──────┘ └──────┘              │
│                                                     │
│  [加载更多]                                          │
└─────────────────────────────────────────────────────┘
```

**样式规范**：
- 市场筛选 Tab：pill-tab 样式
- 股票卡片网格：CSS Grid，桌面 4 列，平板 2 列，移动端 1 列
- 卡片内部：
  - 推荐等级徽章：`strong_buy`=金色，`buy`=绿色，`watch`=蓝色
  - 代码：`font-mono text-sm`
  - 名称：`text-base font-medium`
  - 涨跌幅：正=绿，负=红
  - 预期收益：`text-emerald-500 font-bold`
  - 置信度：进度条 + 数字
  - 市场标签：小型 pill，颜色区分市场
- "加载更多" 按钮：居中 pill 按钮

**响应式设计**：

| 断点 | 宽度 | 布局 |
|------|------|------|
| Wide Desktop | >=1280px | 4 列网格 |
| Desktop | 1024-1279px | 3 列网格 |
| Tablet | 768-1023px | 2 列网格 |
| Mobile L | 480-767px | 1 列网格 |
| Mobile S | <480px | 列表视图 |

**交互逻辑**：
- 市场 Tab 切换：前端过滤 + 动画过渡 200ms
- 股票卡片 hover：`shadow-md` + 微上移
- 股票卡片 click：跳转到对应市场详情页
- 推荐等级徽章 hover：tooltip 显示推荐详情
- "加载更多" click：触发分页加载

**空状态设计**：
- 图标：`Sparkles`
- 标题：「暂无推荐股票」
- 描述：「当前市场暂无符合条件的推荐」
- 操作：切换市场筛选

**加载状态设计**：
- 8 个 Skeleton 卡片网格
- 每个卡片：矩形骨架 + 行文本骨架

**错误处理**：
- 网络错误：显示错误横幅 + 重试
- 部分加载：保留已加载数据

**API 调用清单**：
| 项目 | 规格 |
|------|------|
| 请求方法 | GET |
| 端点 | `/api/v1/ai/stock-recommendations` |
| 请求参数 | `market?: MarketFilter`, `limit: 20`, `offset: 0` |
| 响应格式 | `{ ok: boolean, data: { stocks: AIRecommendedStock[], has_more: boolean } }` |
| 缓存 TTL | 300 秒 |
| 失效条件 | 超过 TTL、手动刷新 |
| 降级策略 | 显示缓存 + 过期标记 |
| 限流策略 | 分页加载间隔最少 2 秒 |

### 4.9 AITradingDecisions AI 买卖决策

**文件**：`features/finance/components/ai/AITradingDecisionsCard.tsx`

**Props 设计**：
```typescript
interface AITradingDecisionsCardProps {
  decisions: AITradingDecision[];
  loading: boolean;
  error: string | null;
  activeAction: ActionFilter;
  onActionChange: (action: ActionFilter) => void;
  onStockClick: (decision: AITradingDecision) => void;
  onLoadMore?: () => void;
  hasMore: boolean;
}

type ActionFilter = 'all' | 'buy' | 'sell' | 'hold';

interface AITradingDecision {
  id: string;
  code: string;
  name: string;
  market: MarketKey;
  market_label: string;
  current_price: number;
  action: TradingAction;
  action_label: string;              // "买入" / "卖出" / "持有"
  target_price: number;
  stop_loss: number;
  expected_return: number;           // 预期收益率 %
  confidence: number;                // 置信度 0-100
  reason: string;                    // 决策理由
  risk_reward_ratio: number;         // 风险收益比
  suggested_position: string;        // 建议仓位，如 "轻仓" / "半仓" / "重仓"
  time_horizon: string;              // 持有周期
  generated_at: string;
}

type TradingAction = 'buy' | 'sell' | 'hold';
```

**状态管理**：
```typescript
interface AITradingDecisionsState {
  decisions: AITradingDecision[];
  filteredDecisions: AITradingDecision[];
  activeAction: ActionFilter;
  loading: boolean;
  error: string | null;
  hasMore: boolean;
  offset: number;
}

type AITradingDecisionsAction =
  | { type: 'FETCH_START' }
  | { type: 'FETCH_SUCCESS'; payload: { decisions: AITradingDecision[]; has_more: boolean } }
  | { type: 'FETCH_ERROR'; payload: string }
  | { type: 'CHANGE_ACTION'; payload: ActionFilter }
  | { type: 'LOAD_MORE'; payload: AITradingDecision[] }
  | { type: 'SET_OFFSET'; payload: number };
```

**数据流**：
```
GET /api/v1/ai/trading-decisions?action={action}&limit=20&offset=0
→ { decisions: AITradingDecision[], has_more: boolean }
→ 缓存 300 秒
→ 前端按 action 过滤
→ 分页加载
```

**布局实现**：
```
┌─────────────────────────────────────────────────────┐
│  AI 买卖决策                                       │
│  [全部] [买入 (5)] [卖出 (2)] [持有 (8)]            │
│                                                     │
│  ┌─────────────────────────────────────────────┐   │
│  │ 🟢 买入 600036 招商银行     置信度: 85%    │   │
│  │ 现价 ¥38.50 → 目标 ¥42.00 (+9.1%)         │   │
│  │ 止损 ¥36.50 | 风险收益比 1:2.5             │   │
│  │ 仓位建议: 半仓 | 周期: 中期 (1-3月)        │   │
│  │ 理由: 银行板块估值修复 + 降准利好...         │   │
│  └─────────────────────────────────────────────┘   │
│  ┌─────────────────────────────────────────────┐   │
│  │ 🔴 卖出 002475 立讯精密     置信度: 72%    │   │
│  │ 现价 ¥32.10 → 目标 ¥29.00 (-9.7%)         │   │
│  │ 止损 ¥33.50 | 风险收益比 1:2.0             │   │
│  └─────────────────────────────────────────────┘   │
│  ┌─────────────────────────────────────────────┐   │
│  │ ⚪ 持有 601318 中国平安     置信度: 68%    │   │
│  │ 现价 ¥48.20 → 目标 ¥50.00 (+3.7%)         │   │
│  └─────────────────────────────────────────────┘   │
│  ... (更多决策)                                     │
│                                                     │
│  ⚠️ 免责声明: AI 买卖决策仅供参考，不构成投资建议。  │
│  投资有风险，入市需谨慎。                            │
└─────────────────────────────────────────────────────┘
```

**样式规范**：
- 操作筛选 Tab：pill-tab 样式，括号内显示数量
  - 买入 Tab：绿色文字
  - 卖出 Tab：红色文字
  - 持有 Tab：灰色文字
- 决策卡片：
  - 买入：左侧绿色竖线 + 绿色操作徽章
  - 卖出：左侧红色竖线 + 红色操作徽章
  - 持有：左侧灰色竖线 + 灰色操作徽章
- 目标价/止损价：`tabular-nums font-mono`
- 预期收益：正=绿，负=红
- 风险收益比：pill 样式标签
- 免责声明：底部黄色警告条，`text-xs text-amber-600`

**响应式设计**：

| 断点 | 宽度 | 布局 |
|------|------|------|
| Wide Desktop | >=1280px | 卡片网格 2 列 |
| Desktop | 1024-1279px | 卡片单列 |
| Tablet | 768-1023px | 卡片单列，隐藏理由 |
| Mobile L | 480-767px | 紧凑卡片，仅操作 + 代码 + 目标价 |
| Mobile S | <480px | 列表视图 |

**交互逻辑**：
- 操作 Tab 切换：前端过滤 + 计数徽章更新
- 决策卡片 hover：`shadow-md`
- 决策卡片 click：展开完整理由和分析
- 股票代码 click：跳转详情页
- 置信度进度条 hover：tooltip 显示分析详情

**空状态设计**：
- 图标：`BarChart3`
- 标题：「暂无买卖决策」
- 描述：「AI 暂未生成针对当前市场的交易决策」
- 操作：切换操作筛选

**加载状态设计**：
- 5 个 Skeleton 决策卡片
- 每个卡片：矩形骨架 + 多行文本骨架

**错误处理**：
- 服务不可用：显示「决策服务暂时不可用」
- 数据过期：缓存数据 + 过期标记

**API 调用清单**：
| 项目 | 规格 |
|------|------|
| 请求方法 | GET |
| 端点 | `/api/v1/ai/trading-decisions` |
| 请求参数 | `action?: ActionFilter`, `limit: 20`, `offset: 0` |
| 响应格式 | `{ ok: boolean, data: { decisions: AITradingDecision[], has_more: boolean } }` |
| 缓存 TTL | 300 秒 |
| 失效条件 | 超过 TTL、手动刷新 |
| 降级策略 | 显示缓存 + 过期标记 |
| 限流策略 | 分页加载间隔最少 2 秒 |

### 4.10 Watchlist 自选股

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

**状态管理**：
```typescript
interface WatchlistState {
  items: WatchlistItem[];
  loading: boolean;
  error: string | null;
  showAddDialog: boolean;
  searchQuery: string;
}

type WatchlistAction =
  | { type: 'FETCH_START' }
  | { type: 'FETCH_SUCCESS'; payload: WatchlistItem[] }
  | { type: 'FETCH_ERROR'; payload: string }
  | { type: 'ADD_ITEM'; payload: WatchlistItem }
  | { type: 'REMOVE_ITEM'; payload: string }
  | { type: 'TOGGLE_ADD_DIALOG' }
  | { type: 'SET_SEARCH'; payload: string };
```

**数据流**：
```
GET /api/brozoomout/watchlist
→ { total: number, items: WatchlistItem[] }
→ 缓存 30 秒
→ 支持 POST 添加、DELETE 删除
→ 删除后立即更新本地状态（乐观更新）
```

**样式规范**：
- 卡片列表，每只股票一行
- 信号标签颜色：BUY=绿，SELL=红，HOLD=灰，WATCH=黄
- 涨跌颜色：正=绿，负=红
- 添加按钮：底部 pill 按钮
- 空状态：引导提示 + 添加按钮
- 排序：按 signal_strength 排序（强信号在前）

**响应式设计**：

| 断点 | 宽度 | 布局 |
|------|------|------|
| Wide Desktop | >=1280px | 完整列表，所有字段 |
| Desktop | 1024-1279px | 完整列表 |
| Tablet | 768-1023px | 隐藏 added_at 列 |
| Mobile L | 480-767px | 紧凑列表，隐藏信号强度 |
| Mobile S | <480px | 仅代码 + 名称 + 涨跌幅 |

**交互逻辑**：
- 股票行 hover：`bg-gray-50` 背景
- 股票行 click：跳转详情页
- 添加按钮 click：弹出搜索对话框
- 删除按钮：hover 显示，click 弹出确认对话框（AlertDialog）
- 删除确认后：乐观更新（立即移除），失败时回滚
- 下拉刷新（移动端）：重新获取自选股列表

**空状态设计**：
- 图标：`StarOff`
- 标题：「暂无自选股」
- 描述：「添加关注的股票，获取实时行情和 AI 分析」
- 操作：「添加股票」按钮

**加载状态设计**：
- 5 行 Skeleton 列表行
- 每行：代码 + 名称 + 价格 + 涨跌幅

**错误处理**：
- 网络错误：显示缓存数据 + 错误横幅
- 删除失败：回滚删除操作 + toast 提示
- 添加失败：toast 提示错误原因

**API 调用清单**：
| 项目 | 规格 |
|------|------|
| GET | `/api/brozoomout/watchlist` — 获取列表，缓存 30s |
| POST | `/api/brozoomout/watchlist` — 添加自选，Body: `{ code, market }` |
| DELETE | `/api/brozoomout/watchlist/:id` — 删除自选 |
| 失效条件 | 添加/删除后立即刷新 |
| 限流策略 | 添加操作最少间隔 1 秒 |

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
  volume_change: number;               // 成交量变化 %
  fund_flow: number;                   // 板块资金净流入（亿）
  leading_stock: string;
  leading_stock_name: string;
  leading_stock_change: number;
  stock_count: number;
  up_count: number;
  down_count: number;
  market: 'cn_stock' | 'hk_stock';
  historical_return: {                 // 历史表现对比
    '1d': number;
    '5d': number;
    '1m': number;
    '3m': number;
  };
}

interface HeatmapColorConfig {
  positiveColor: string;
  negativeColor: string;
  neutralColor: string;
  maxPercent: number;
}
```

**新增功能**：
- **板块资金流向**：每个方块内显示资金净流入/流出（绿色=流入，红色=流出）
- **成交量柱状图**：点击方块后展开，显示近 10 日成交量柱状图
- **领涨股预览**：hover 时 tooltip 显示领涨股详情（代码、名称、涨跌幅）
- **历史表现对比**：点击方块后弹出面板，显示 1 日/5 日/1 月/3 月涨跌幅

**数据流**：
```
GET /api/v1/market/sectors?type=industry&market=cn_stock
→ { sectors: HeatmapSector[] }
→ 缓存 120 秒
→ 切换 industry/concept 时重新请求
→ 资金流向独立请求: GET /api/v1/market/sectors/fund-flow
```

**颜色映射算法**：
```typescript
function getHeatmapColor(pct: number, config: HeatmapColorConfig): string {
  const clamped = Math.max(-config.maxPercent, Math.min(config.maxPercent, pct));
  const ratio = clamped / config.maxPercent;
  if (ratio > 0) {
    const lightness = 85 - ratio * 40;
    return `hsl(142, 70%, ${lightness}%)`;
  } else if (ratio < 0) {
    const lightness = 85 + ratio * 40;
    return `hsl(0, 70%, ${lightness}%)`;
  }
  return 'hsl(220, 14%, 97%)';
}
```

**响应式设计**：

| 断点 | 宽度 | 布局 |
|------|------|------|
| Wide Desktop | >=1280px | 完整热力图，方块可点击 |
| Desktop | 1024-1279px | 热力图略缩 |
| Tablet | 768-1023px | 方块缩小，tooltip 改为点击弹出 |
| Mobile L | 480-767px | 列表视图，按涨跌幅排序 |
| Mobile S | <480px | 紧凑列表，仅板块名 + 涨跌幅 |

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
  fund_flow: number;                   // 板块资金净流入（亿）
  fund_flow_trend: FundFlowTrendData;  // 资金流向趋势
  leading_stock: string;
  leading_stock_change: number;
  up_count: number;
  down_count: number;
  flat_count: number;
  top_stocks: SectorStock[];           // 板块内个股统计
  news_count: number;                  // 关联新闻数量
  valuation_level: 'low' | 'medium' | 'high';  // 板块估值水平
}

interface SectorStock {
  code: string;
  name: string;
  change_percent: number;
  volume: number;
}

interface FundFlowTrendData {
  daily_flow: { date: string; flow: number }[];  // 近 10 日资金流向
  total_5d: number;   // 5 日累计净流入
  total_10d: number;  // 10 日累计净流入
}
```

**新增功能**：
- **板块内个股统计**：点击板块展开，显示板块内涨跌家数饼图
- **资金流向趋势图**：点击板块展开，显示近 10 日资金流向折线图
- **关联新闻**：点击板块展开，显示关联新闻列表
- **板块估值水平**：每个板块显示估值等级标签（低/中/高）

**布局实现**：
- 两个 Tab：行业板块 / 概念板块
- 对称条形图：中轴线居中，涨向右（绿），跌向左（红）
- 每条：板块名 + 涨跌幅 + 涨跌家数 + 领涨股 + 资金流向 + 估值标签
- 排序：涨幅降序
- 展开面板：个股统计 + 资金趋势 + 新闻

**响应式设计**：
- 桌面：完整条形图 + 展开面板
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
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  ┌──────────────────────┬──────────────────────────────┐   │
│  │  财务数据             │  估值分析                     │   │
│  └──────────────────────┴──────────────────────────────┘   │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  资金流向分析                                        │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  股东信息                                            │   │
│  │  十大股东 | 机构持仓 | 股权变化                       │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  ┌──────────────────────┬──────────────────────────────┐   │
│  │  研报列表             │  相关新闻                     │   │
│  └──────────────────────┴──────────────────────────────┘   │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  分红历史 | 限售解禁 | 龙虎榜 | 融资融券 | 大宗交易  │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  AI 个股分析                                        │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  投资者问答 (Q&A)                                   │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

**新增数据模块**（相比原版扩展）：

```typescript
// 扩展 StockDetail 接口
interface StockDetailExtended {
  basic: StockBasicInfo;
  kline: KLineData[];
  financials: FinancialData;
  valuation: ValuationData;
  fund_flow: FundFlowData;
  shareholders: ShareholderData;       // 新增：股东信息
  dividend_history: DividendData[];    // 新增：分红历史
  lock_up_expiry: LockUpData[];        // 新增：限售解禁
  dragon_tiger: DragonTigerData[];     // 新增：龙虎榜
  margin_trading: MarginData;          // 新增：融资融券
  block_trades: BlockTradeData[];      // 新增：大宗交易
  qa_list: QAItem[];                   // 新增：投资者问答
  research: ResearchReport[];
  news: NewsItem[];
  ai_analysis: AIStockAnalysis;
}

interface ShareholderData {
  top_10: Shareholder[];               // 十大股东
  institutional: InstitutionalHolder[];// 机构持仓
  change_summary: string;              // 股权变化摘要
}

interface Shareholder {
  rank: number;
  name: string;
  type: 'natural' | 'institutional' | 'state' | 'foreign';
  shares: number;
  percentage: number;
  change: number;                      // 持股变化
}

interface InstitutionalHolder {
  name: string;
  shares: number;
  percentage: number;
  market_value: number;
  quarter: string;
}

interface DividendData {
  year: string;
  dividend_per_share: number;
  ex_date: string;
  pay_date: string;
  yield: number;
}

interface LockUpData {
  date: string;
  shares: number;
  holder: string;
  type: 'initial' | 'custom';
}

interface DragonTigerData {
  date: string;
  reason: string;
  net_buy: number;
  top_buy_seats: string[];
  top_sell_seats: string[];
}

interface MarginData {
  margin_balance: number;              // 融资余额
  short_balance: number;               // 融券余额
  total_balance: number;
  change: number;
  margin_buy: number;                  // 融资买入
  short_sell: number;                  // 融券卖出
  daily_trend: { date: string; margin: number; short: number }[];
}

interface BlockTradeData {
  date: string;
  price: number;
  volume: number;
  amount: number;
  buyer: string;
  seller: string;
  premium_discount: number;            // 溢折价率
}

interface QAItem {
  id: string;
  question: string;
  answer: string;
  asker: string;
  date: string;
  likes: number;
}
```

**数据流**：
```
GET /api/v1/stock/detail/:code?market=cn_stock
→ StockDetailExtended
→ 缓存 30 秒
→ K 线数据: GET /api/v1/stock/kline/:code?period=daily&limit=120
→ 财务数据: GET /api/v1/stock/financials/:code
→ 股东信息: GET /api/v1/stock/shareholders/:code
→ 分红历史: GET /api/v1/stock/dividends/:code
→ 融资融券: GET /api/v1/stock/margin/:code
→ 龙虎榜:   GET /api/v1/stock/dragon-tiger/:code
→ 大宗交易: GET /api/v1/stock/block-trades/:code
→ AI 分析:  GET /api/v1/stock/ai-analysis/:code
```

**交互逻辑**：
- 返回按钮：`useNavigate(-1)`
- K 线图切换：日K/周K/月K 重新请求
- 指标切换：MA/MACD/RSI/KDJ 在图表上叠加
- 股东信息 Tab：切换十大股东/机构持仓/股权变化
- 分红历史表格：按年份排序
- 限售解禁日历：日历视图 + 列表视图切换
- 融资融券趋势图：Recharts AreaChart
- 投资者问答：支持折叠/展开回答
- 添加自选按钮：右上角

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

### 6.5 USStockDetail 美股详情（三级页面）

**文件**：`features/finance/pages/USStockDetailPage.tsx`

**扩展数据模块**：

```typescript
interface USStockDetailExtended {
  basic: StockBasicInfo;
  kline: KLineData[];
  financials: FinancialData;       // US GAAP 格式
  valuation: ValuationData;
  fund_flow: FundFlowData;
  institutional_holdings: InstitutionalHoldings13F;  // 新增：机构持仓 (13F)
  analyst_ratings: AnalystRatingsSummary;             // 新增：分析师评级汇总
  options_chain: OptionsChainData;                    // 新增：期权链数据
  pre_post_market: PrePostMarketData;                // 新增：盘前盘后数据
  related_etfs: RelatedETF[];                         // 新增：相关 ETF
  research: ResearchReport[];
  news: NewsItem[];
  ai_analysis: AIStockAnalysis;
}

interface InstitutionalHoldings13F {
  total_institutional: number;           // 机构持股总数
  total_shares: number;
  total_value: number;
  quarterly_change: number;              // 季度变化 %
  top_holders: Institution13F[];
  filing_date: string;                   // 最新 13F 提交日期
}

interface Institution13F {
  name: string;                          // 机构名称 (Berkshire, etc.)
  shares: number;
  percentage: number;
  value: number;
  change_shares: number;                 // 持股变化
  change_percent: number;                // 变化百分比
}

interface AnalystRatingsSummary {
  total_analysts: number;
  ratings: {
    strong_buy: number;
    buy: number;
    hold: number;
    sell: number;
    strong_sell: number;
  };
  consensus: 'strong_buy' | 'buy' | 'hold' | 'sell' | 'strong_sell';
  avg_target_price: number;
  high_target: number;
  low_target: number;
  avg_upside: number;
  recent_ratings: AnalystRating[];
}

interface AnalystRating {
  firm: string;
  analyst: string;
  rating: string;
  target_price: number;
  date: string;
}

interface OptionsChainData {
  expiration_dates: string[];
  calls: OptionData[];
  puts: OptionData[];
  implied_volatility: number;
  put_call_ratio: number;
}

interface OptionData {
  strike: number;
  expiry: string;
  type: 'call' | 'put';
  bid: number;
  ask: number;
  last: number;
  volume: number;
  open_interest: number;
  implied_volatility: number;
}

interface PrePostMarketData {
  pre_market_price: number;
  pre_market_change: number;
  pre_market_change_percent: number;
  pre_market_volume: number;
  post_market_price: number;
  post_market_change: number;
  post_market_change_percent: number;
  post_market_volume: number;
}

interface RelatedETF {
  code: string;
  name: string;
  weight: number;                       // 该股票在 ETF 中的权重
  ytd_return: number;
}
```

**数据流**：
```
GET /api/v1/us/stock/detail/:code
→ USStockDetailExtended
→ 13F 持仓: GET /api/v1/us/stock/institutional/:code
→ 分析师评级: GET /api/v1/us/stock/analyst-ratings/:code
→ 期权链: GET /api/v1/us/stock/options/:code
→ 盘前盘后: GET /api/v1/us/stock/pre-post/:code
→ 相关 ETF: GET /api/v1/us/stock/related-etfs/:code
```

**响应式设计**：

| 断点 | 宽度 | 布局 |
|------|------|------|
| Wide Desktop | >=1280px | 多列并排，所有模块完整显示 |
| Desktop | 1024-1279px | 两列布局 |
| Tablet | 768-1023px | 两列布局，期权链折叠 |
| Mobile L | 480-767px | 单列堆叠，隐藏盘前盘后 |
| Mobile S | <480px | 极简模式，仅基本信息 + K 线 |

---

## 七、Tab 4: 韩国股票（Korean Stocks）

### 7.1 KoreaPage 主页面

**文件**：`features/finance/pages/KoreaPage.tsx`

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

### 7.5 KoreanStockDetail 韩国详情（三级页面）

**扩展数据模块**：

```typescript
interface KoreanStockDetailExtended {
  basic: StockBasicInfo;
  kline: KLineData[];
  financials: FinancialData;
  valuation: ValuationData;
  fund_flow: FundFlowData;
  foreign_holding: ForeignHoldingData;    // 新增：外国人持股比例
  short_selling: ShortSellingData;        // 新增：卖空数据
  related_derivatives: DerivativeData[];  // 新增：相关衍生品
  research: ResearchReport[];
  news: NewsItem[];
  ai_analysis: AIStockAnalysis;
}

interface ForeignHoldingData {
  foreign_ratio: number;                  // 外国人持股比例 %
  foreign_shares: number;
  daily_change: number;                   // 日变化
  monthly_trend: { date: string; ratio: number }[];
  top_foreign_holders: ForeignHolder[];
}

interface ForeignHolder {
  name: string;
  nationality: string;
  shares: number;
  percentage: number;
}

interface ShortSellingData {
  short_balance: number;                  // 卖空余额
  short_volume: number;                   // 卖空量
  short_ratio: number;                    // 卖空比率
  daily_trend: { date: string; short: number; volume: number }[];
}

interface DerivativeData {
  type: 'call' | 'put' | 'futures' | 'warrant';
  code: string;
  name: string;
  price: number;
  change_percent: number;
  volume: number;
  underlying: string;                     // 标的代码
  conversion_ratio: number;               // 换股比率
}
```

**数据流**：
```
GET /api/v1/korea/stock/detail/:code
→ KoreanStockDetailExtended
→ 外国人持股: GET /api/v1/korea/stock/foreign-holding/:code
→ 卖空数据: GET /api/v1/korea/stock/short-selling/:code
→ 相关衍生品: GET /api/v1/korea/stock/derivatives/:code
```

---

## 八、Tab 5: 基金（Funds）

### 8.1 FundsPage 主页面

**文件**：`features/finance/pages/FundsPage.tsx`

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

### 8.3 FundRanking 基金排名

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

### 8.5 FundDetail 基金详情（三级页面）

**文件**：`features/finance/pages/FundDetailPage.tsx`

**扩展数据模块**：

```typescript
interface FundDetailExtended {
  basic: FundBasicInfo;
  performance: FundPerformance;
  holdings: FundHolding[];
  risk: FundRisk;
  style: FundStyle;
  ai_recommendation: AIFundRecommendation;
  peers: FundPeer[];
  manager_career: ManagerCareer;              // 新增：基金经理履历
  company_background: FundCompanyBackground;  // 新增：基金公司背景
  fund_ratings: FundRatings;                  // 新增：基金评级
  scale_trend: ScaleTrendData;                // 新增：基金规模趋势
  institutional_holding: InstitutionalHolding; // 新增：机构持有比例
  large_alerts: LargeAlertData;               // 新增：大额申购/赎回提醒
}

interface ManagerCareer {
  name: string;
  title: string;
  appointment_date: string;           // 任职日期
  tenure_days: number;                // 任职天数
  career_history: CareerEntry[];      // 履历
  managed_funds: ManagedFundSummary[];
  total_aum: number;                  // 管理总规模
}

interface CareerEntry {
  period: string;
  company: string;
  position: string;
  description: string;
}

interface ManagedFundSummary {
  code: string;
  name: string;
  return_during_tenure: number;       // 任职期间收益率
  rank_in_category: number;           // 同类排名
}

interface FundCompanyBackground {
  name: string;
  established: string;
  total_aum: number;                  // 公司总管理规模
  fund_count: number;                 // 管理基金数量
  rating: string;                     // 公司评级
  key_info: string;                   // 公司简介
}

interface FundRatings {
  morningstar: number;                // 晨星评级 (1-5)
  galaxy: number;                     // 银河评级 (1-5)
  zhaoshang: number;                  // 招商评级 (1-5)
  rating_date: string;
}

interface ScaleTrendData {
  quarterly_scale: { date: string; scale: number }[];
  latest_scale: number;
  scale_change_1y: number;            // 近一年规模变化
}

interface InstitutionalHolding {
  ratio: number;                      // 机构持有比例 %
  shares: number;
  holders_count: number;
  top_institutional: {
    name: string;
    shares: number;
    percentage: number;
  }[];
}

interface LargeAlertData {
  has_alert: boolean;
  alerts: {
    date: string;
    type: 'large_purchase' | 'large_redemption';
    amount: number;
    description: string;
  }[];
}
```

**布局实现（扩展后）**：
```
┌─────────────────────────────────────────────────────────────┐
│  ← 返回  招商中证白酒 161725                                │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────────────────────────────────────────────────┐   │
│  │  基本信息卡片                                        │   │
│  │  类型: 股票型  规模: 150亿  经理: 侯昊              │   │
│  │  成立日期: 2015-05-27  评级: ★★★★☆                 │   │
│  │  晨星: ★★★★  银河: ★★★★  招商: ★★★★☆            │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  业绩走势图                                          │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  ┌──────────────────────┬──────────────────────────────┐   │
│  │  持仓明细             │  风险分析                     │   │
│  └──────────────────────┴──────────────────────────────┘   │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  基金经理履历                                        │   │
│  │  侯昊 | 任职 2856 天 | 管理规模 1,200 亿            │   │
│  │  履历: 博时基金 → 招商基金                           │   │
│  │  任职期间表现: +85.2%, 同类排名前 15%               │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  基金公司背景                                        │   │
│  │  招商基金管理有限公司 | 成立 2002 年                 │   │
│  │  总规模: 8,500 亿 | 基金数量: 280+                  │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  规模趋势                                            │   │
│  │  近 4 季度规模变化折线图                              │   │
│  │  最新规模: 150 亿 | 近 1 年变化: +12.5%             │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  ┌──────────────────────┬──────────────────────────────┐   │
│  │  机构持有比例         │  大额申赎提醒                 │   │
│  │  机构占比: 35.2%     │  ⚠️ 2026-05-15 大额赎回     │   │
│  │  持有人数: 1,250     │     金额: 5.2 亿              │   │
│  └──────────────────────┴──────────────────────────────┘   │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  风格漂移检测                                        │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  AI 基金推荐                                        │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  同类基金对比                                        │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

**数据流**：
```
GET /api/v1/fund/detail/:code
→ FundDetailExtended
→ 缓存 600 秒
→ 基金经理: GET /api/v1/fund/manager/:code
→ 基金公司: GET /api/v1/fund/company/:code
→ 评级数据: GET /api/v1/fund/ratings/:code
→ 规模趋势: GET /api/v1/fund/scale-trend/:code
→ 机构持有: GET /api/v1/fund/institutional/:code
→ 大额申赎: GET /api/v1/fund/large-alerts/:code
→ 业绩走势: GET /api/v1/fund/performance/:code?period=1y
→ AI 推荐: GET /api/v1/fund/ai-recommendation/:code
```


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
│  │  多时间框架: 1日 / 1周 / 1月 / 3月                   │   │
│  │  三场景预测 + 历史准确率 + 因子权重                  │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  ┌─────────────────────────────┬─────────────────────────┐ │
│  │  StockAnalysis              │  PortfolioOptimization   │ │
│  │  (个股分析)                 │  (组合优化)               │ │
│  │  综合评分 0-100             │  当前 vs 优化后对比      │ │
│  │  技术/基本面/资金流/新闻    │  有效前沿图              │ │
│  └─────────────────────────────┴─────────────────────────┘ │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  RiskAssessment (风险评估)                           │   │
│  │  持仓风险评估 + 压力测试 + 相关性分析                │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### 9.2 MarketPrediction 市场预测（扩展）

**文件**：`features/finance/components/ai/MarketPredictionCard.tsx`

**Props 设计**：
```typescript
interface MarketPredictionProps {
  prediction: MarketPrediction | null;
  loading: boolean;
  error: string | null;
  horizon: PredictionHorizon;
  onHorizonChange: (horizon: PredictionHorizon) => void;
}

type PredictionHorizon = '1d' | '1w' | '1m' | '3m';

interface MarketPrediction {
  summary: string;
  confidence: number;
  generated_at: string;
  horizon: PredictionHorizon;
  scenarios: Scenario[];
  key_factors: KeyFactor[];
  historical_accuracy: PredictionAccuracy;    // 新增：历史准确率追踪
  factor_weights: FactorWeight[];             // 新增：因子权重
  model_explanation: ModelExplanation;         // 新增：模型解释
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

interface PredictionAccuracy {
  total_predictions: number;
  correct_predictions: number;
  accuracy_by_horizon: {
    '1d': number;
    '1w': number;
    '1m': number;
    '3m': number;
  };
  recent_trend: 'improving' | 'stable' | 'declining';
  monthly_accuracy: { month: string; accuracy: number }[];
}

interface FactorWeight {
  factor: string;
  weight: number;                   // 权重 0-1
  trend: 'increasing' | 'stable' | 'decreasing';
  description: string;
}

interface ModelExplanation {
  model_name: string;
  model_version: string;
  training_data: string;
  last_retrained: string;
  feature_importance: { feature: string; importance: number }[];
}
```

**新增功能**：
- **多时间框架预测**：1 日 / 1 周 / 1 月 / 3 月 四个时间维度
- **历史准确率追踪**：按时间框架统计准确率，月度趋势图
- **模型解释**：显示模型名称、版本、训练数据、特征重要性
- **因子权重**：显示各因子的权重和变化趋势

**布局实现**：
```
┌─────────────────────────────────────────────────────┐
│  市场预测                     [1日] [1周] [1月] [3月]│
│  ─────────────────────────────────────────────────  │
│  预测摘要文字...                       置信度: 72%   │
│                                                     │
│  ┌──────────┬──────────┬──────────┐                 │
│  │  🟢 牛市  │  🔵 基准  │  🔴 熊市  │                 │
│  │   30%    │   50%    │   20%    │                 │
│  └──────────┴──────────┴──────────┘                 │
│                                                     │
│  关键因素 + 因子权重:                                │
│  ● 央行降准 (正面) ──────── 35% ████████████        │
│  ● 美联储鹰派 (负面) ────── 25% ████████           │
│  ● 北向资金 (正面) ──────── 20% ██████             │
│  ● 成交量 (负面) ────────── 15% █████              │
│  ● 地缘政治 (中性) ──────── 5%  ██                │
│                                                     │
│  ┌─────────────────────────────────────────────┐   │
│  │  历史准确率                                 │   │
│  │  总预测: 150 次 | 正确: 108 次 | 准确率: 72%│   │
│  │  1日: 68% | 1周: 72% | 1月: 75% | 3月: 70% │   │
│  │  ▁▂▃▃▄▅▅▆▆▇▇█▇▇▆▆▅▅▄▃▃▂▂ (月度趋势图)     │   │
│  └─────────────────────────────────────────────┘   │
│                                                     │
│  ▼ 模型信息: ZL-Predict v3.2 | 训练数据: 2020-2026 │
└─────────────────────────────────────────────────────┘
```

**数据流**：
```
GET /api/v1/ai/market-prediction?horizon=1w
→ MarketPrediction
→ 缓存 600 秒
→ 历史准确率: GET /api/v1/ai/prediction/accuracy
→ 因子权重: GET /api/v1/ai/prediction/factor-weights
```

**交互逻辑**：
- 时间框架 Tab 切换：重新请求不同 horizon 的预测
- 场景卡片 hover：tooltip 显示触发条件列表
- 因子权重条形图 hover：tooltip 显示因子详细说明
- 历史准确率区域：点击展开月度趋势图
- 模型信息：点击展开特征重要性列表

**响应式设计**：

| 断点 | 宽度 | 布局 |
|------|------|------|
| Wide Desktop | >=1280px | 三场景卡片横排，因素列表单列 |
| Desktop | 1024-1279px | 三场景卡片横排 |
| Tablet | 768-1023px | 三场景卡片纵向堆叠 |
| Mobile L | 480-767px | 简化显示，隐藏因子权重图 |
| Mobile S | <480px | 仅摘要 + 总概率 |

**空状态设计**：
- 图标：`BrainCircuit`
- 标题：「预测数据暂不可用」
- 描述：「AI 模型正在更新中，请稍后重试」
- 操作：「切换时间框架」按钮

**加载状态设计**：
- 摘要：2 行 Skeleton
- 场景卡片：3 个矩形 Skeleton
- 因素列表：5 行 Skeleton
- 准确率区域：矩形 Skeleton

**错误处理**：
- 模型超时：显示「预测生成超时」+ 自动重试
- 部分数据缺失：已有字段正常显示

**API 调用清单**：
| 项目 | 规格 |
|------|------|
| 请求方法 | GET |
| 端点 | `/api/v1/ai/market-prediction` |
| 请求参数 | `horizon: '1d' | '1w' | '1m' | '3m'` |
| 响应格式 | `{ ok: boolean, data: MarketPrediction }` |
| 缓存 TTL | 600 秒 |
| 超时设置 | 15 秒 |
| 限流策略 | 最少间隔 300 秒 |

### 9.3 StockAnalysis 个股分析（扩展）

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
  composite_score: number;              // 新增：综合评分 0-100
  summary: string;
  technical_analysis: TechnicalAnalysis;
  fundamental_analysis: FundamentalAnalysis;
  fund_flow_analysis: FundFlowAnalysisResult;  // 新增：资金流分析
  news_analysis: NewsAnalysisResult;           // 新增：新闻分析
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
  score: number;                        // 技术面评分 0-100
}

interface FundamentalAnalysis {
  valuation: 'undervalued' | 'fair' | 'overvalued';
  pe_vs_industry: string;
  growth_potential: string;
  financial_health: string;
  score: number;                        // 基本面评分 0-100
}

interface FundFlowAnalysisResult {
  main_flow_trend: 'inflow' | 'outflow' | 'balanced';
  northbound_flow: number;
  institutional_flow: number;
  retail_sentiment: string;
  score: number;                        // 资金流评分 0-100
}

interface NewsAnalysisResult {
  overall_sentiment: 'positive' | 'negative' | 'neutral';
  recent_news_count: number;
  positive_ratio: number;
  key_headlines: string[];
  score: number;                        // 新闻面评分 0-100
}

interface SentimentAnalysis {
  overall: 'positive' | 'negative' | 'neutral';
  news_sentiment: number;
  analyst_consensus: string;
  insider_activity: string;
}
```

**新增功能**：
- **综合评分（0-100）**：技术面 + 基本面 + 资金流 + 新闻面加权评分
- **资金流分析**：主力资金趋势、北向资金、机构资金、散户情绪
- **新闻分析**：近期新闻情感、正面比例、关键标题
- **四维评分雷达图**：技术面/基本面/资金流/新闻面的雷达图对比

**布局实现**：
```
┌─────────────────────────────────────────────────────┐
│  AI 个股分析                                         │
│  输入股票代码: [600036      ] [分析]                  │
│                                                     │
│  ┌─────────────────────────────────────────────┐   │
│  │  综合评分: 78/100     推荐: 买入  置信度: 75%│   │
│  │  ┌─────────────────────────────────────────┐ │   │
│  │  │          雷达图                          │ │   │
│  │  │     技术面 82                           │ │   │
│  │  │    /      \                             │ │   │
│  │  │ 基本面75  资金流70                       │ │   │
│  │  │    \      /                             │ │   │
│  │  │     新闻面 72                            │ │   │
│  │  └─────────────────────────────────────────┘ │   │
│  └─────────────────────────────────────────────┘   │
│                                                     │
│  四列分析:                                           │
│  ┌────────┬────────┬────────┬────────┐             │
│  │ 技术面  │ 基本面  │ 资金流  │ 新闻面  │             │
│  │ 趋势↑  │ 低估   │ 流入   │ 正面   │             │
│  │ RSI:45 │ PE:6.8 │ 北向+  │ 80%正面│             │
│  │ MA金叉 │ 行业前3│ 1.5亿  │        │             │
│  └────────┴────────┴────────┴────────┘             │
│                                                     │
│  目标价: ¥42.00  止损: ¥36.00  上涨空间: +9.1%    │
│  风险因素: ... | 催化剂: ...                        │
└─────────────────────────────────────────────────────┘
```

**数据流**：
```
POST /api/v1/ai/stock-analysis
Body: { code: "600036", market: "cn_stock" }
→ AIStockAnalysisResult
→ 无缓存（每次分析都是实时生成）
→ 预计响应时间 3-5 秒
```

**交互逻辑**：
- 输入框：支持代码搜索，防抖 300ms
- 分析按钮：触发 POST 请求，显示加载动画
- 雷达图 hover：tooltip 显示详细评分
- 四列分析：点击展开详细内容
- 目标价区域：价格标签 + 上涨空间可视化
- 历史分析：保存最近 10 次分析记录

**空状态设计**：
- 图标：`Search`
- 标题：「输入股票代码开始分析」
- 描述：「AI 将从技术面、基本面、资金流、新闻面四个维度进行综合分析」

**加载状态设计**：
- 分析中：全屏加载动画 + 「AI 正在分析 600036 招商银行...」
- 预计时间：显示进度条（3-5 秒）
- 流式响应：逐步显示各维度分析结果

**错误处理**：
- 分析超时：显示「分析超时，请重试」
- 股票不存在：显示「未找到该股票」
- 模型错误：显示「分析服务暂时不可用」

**API 调用清单**：
| 项目 | 规格 |
|------|------|
| 请求方法 | POST |
| 端点 | `/api/v1/ai/stock-analysis` |
| Body | `{ code: string, market: MarketKey }` |
| 响应格式 | `{ ok: boolean, data: AIStockAnalysisResult }` |
| 缓存 TTL | 无缓存（实时生成） |
| 超时设置 | 15 秒 |
| 限流策略 | 同一股票最少间隔 60 秒，全局最少间隔 10 秒 |

### 9.4 PortfolioOptimization 组合优化（扩展）

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
  current_portfolio: PortfolioMetrics;       // 当前组合风险收益分析
  optimized_portfolio: PortfolioMetrics;     // 优化后组合
  suggestions: RebalanceSuggestion[];
  efficient_frontier: EfficientFrontierPoint[];
  risk_return_analysis: RiskReturnAnalysis;
  stress_test: StressTestResult;            // 新增：压力测试结果
  comparison_chart: ComparisonDataPoint[];  // 新增：优化前后对比图
}

interface PortfolioMetrics {
  expected_return: number;
  volatility: number;
  sharpe_ratio: number;
  max_drawdown: number;
  diversification_score: number;
  concentration_risk: number;               // 集中度风险
}

interface RebalanceSuggestion {
  code: string;
  name: string;
  current_weight: number;
  suggested_weight: number;
  action: 'increase' | 'decrease' | 'hold' | 'remove' | 'add';
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

interface StressTestResult {
  scenarios: StressScenario[];
  worst_case_loss: number;
  recovery_estimate: string;
}

interface StressScenario {
  name: string;
  description: string;
  portfolio_impact: number;
  probability: number;
}

interface ComparisonDataPoint {
  metric: string;
  current: number;
  optimized: number;
  improvement: number;
}
```

**新增功能**：
- **当前组合风险收益分析**：在优化前先展示当前组合的完整风险收益指标
- **优化建议**：每个持仓的调整建议（增加/减少/持有/移除/新增）
- **优化前后对比**：并排对比卡片 + 雷达图
- **压力测试结果**：多个宏观场景下的组合影响
- **有效前沿图**：Recharts ScatterChart，标注当前点和最优点

**布局实现**：
```
┌─────────────────────────────────────────────────────┐
│  组合优化                                           │
│  ┌─────────────────────────────────────────────┐   │
│  │  当前组合输入                                 │   │
│  │  600036 招商银行 30% | 300750 宁德时代 25%   │   │
│  │  005930 三星电子 25% | AAPL 苹果 20%         │   │
│  │                              [开始优化]       │   │
│  └─────────────────────────────────────────────┘   │
│                                                     │
│  ┌──────────────────┬──────────────────────┐       │
│  │  当前组合          │  优化后组合            │       │
│  │  预期收益: 12.5%  │  预期收益: 14.2%     │       │
│  │  波动率: 18.3%    │  波动率: 15.8%       │       │
│  │  夏普比率: 0.68   │  夏普比率: 0.90      │       │
│  │  最大回撤: -15.2% │  最大回撤: -12.1%    │       │
│  │  分散度: 65       │  分散度: 82          │       │
│  └──────────────────┴──────────────────────┘       │
│                                                     │
│  调整建议:                                           │
│  ┌─────────────────────────────────────────────┐   │
│  │ 招商银行 30% → 25%  ▼ 减持  银行占比过高     │   │
│  │ 宁德时代 25% → 20%  ▼ 减持  新能源波动大     │   │
│  │ 三星电子 25% → 30%  ▲ 增持  估值合理          │   │
│  │ AAPL 20% → 15%      ▼ 减持  美股占比调整     │   │
│  │ + 腾讯 0700 10%     ➕ 新增  增加港股配置     │   │
│  └─────────────────────────────────────────────┘   │
│                                                     │
│  ┌─────────────────────────────────────────────┐   │
│  │  有效前沿图 (ScatterChart)                    │   │
│  │  ● 当前组合位置  ◆ 最优组合位置              │   │
│  └─────────────────────────────────────────────┘   │
│                                                     │
│  ┌─────────────────────────────────────────────┐   │
│  │  压力测试结果                                 │   │
│  │  2008 金融危机: -25.3%                       │   │
│  │  2020 新冠暴跌: -18.7%                       │   │
│  │  利率上升 200bp: -8.5%                       │   │
│  │  最大单日损失: -5.2% (预估)                  │   │
│  └─────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────┘
```

**数据流**：
```
POST /api/v1/ai/portfolio-optimization
Body: { holdings: HoldingInput[] }
→ OptimizationResult
→ 无缓存（每次优化都是实时计算）
→ 预计响应时间 5-8 秒
→ 压力测试: POST /api/v1/ai/stress-test
Body: { holdings: HoldingInput[] }
```

**交互逻辑**：
- 持仓输入：支持代码搜索 + 权重调整滑块
- "开始优化" 按钮：触发优化请求，显示加载动画
- 对比卡片：hover 显示改善百分比
- 调整建议列表：点击展开详细理由
- 有效前沿图：hover 显示风险收益坐标
- 压力测试：场景卡片 hover 显示详细影响

**空状态设计**：
- 图标：`PieChart`
- 标题：「输入持仓开始优化」
- 描述：「添加您的持仓信息，AI 将为您生成最优组合方案」
- 操作：「从持仓导入」按钮（如果有持仓数据）

**加载状态设计**：
- 全屏加载动画 + 「正在优化组合...」
- 预计时间：5-8 秒
- 流式显示：先显示对比数据，再显示建议，最后显示图表

**错误处理**：
- 持仓不足：显示「请至少添加 2 只股票」
- 优化超时：显示超时提示 + 重试
- 数据异常：显示具体错误原因

**API 调用清单**：
| 项目 | 规格 |
|------|------|
| 请求方法 | POST |
| 端点 | `/api/v1/ai/portfolio-optimization` |
| Body | `{ holdings: HoldingInput[] }` |
| 响应格式 | `{ ok: boolean, data: OptimizationResult }` |
| 缓存 TTL | 无缓存（实时计算） |
| 超时设置 | 20 秒 |
| 限流策略 | 最少间隔 30 秒 |

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

**布局实现**：
- 输入区：持仓列表
- 风险概览：总体风险等级 + 风险分数
- 压力测试：场景列表 + 最大损失
- 相关性分析：平均相关性 + 分散化收益
- VaR 分析：95%/99% VaR + CVaR
- 场景分析：多个宏观场景的影响

**数据流**：
```
POST /api/v1/ai/risk-assessment
Body: { holdings: HoldingInput[] }
→ RiskAssessmentResult
→ 无缓存（每次评估都是实时计算）
→ 预计响应时间 3-5 秒
```


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
- 响应式：移动端改为底部 Tab 栏

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
  badge?: number;          // 新增：徽章数字（如通知数）
}
```

**实现**：
- 水平滚动的 pill 按钮组
- 激活态：`pill-tab-active`
- 非激活态：`pill-tab`
- 移动端：横向滚动
- 支持 badge 数字显示

### 10.3 BackNavigation 返回导航

**文件**：`features/finance/components/layout/BackNavigation.tsx`

**Props 设计**：
```typescript
interface BackNavigationProps {
  title: string;
  subtitle?: string;
  onBack?: () => void;
  actions?: ReactNode;       // 新增：右侧操作按钮区
}
```

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
  recommendation?: RecommendationLevel;  // 新增：AI 推荐等级
  onClick?: () => void;
  onAddWatchlist?: () => void;
  compact?: boolean;
}
```

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
  emptyMessage?: string;
}

interface ColumnConfig<T> {
  key: string;
  label: string;
  width?: string;
  align?: 'left' | 'center' | 'right';
  render?: (value: unknown, item: T) => ReactNode;
  sortable?: boolean;
  hideOnMobile?: boolean;
}
```

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
  type: 'select' | 'range' | 'toggle' | 'tabs';
  options?: { label: string; value: unknown; count?: number }[];
  min?: number;
  max?: number;
  step?: number;
}
```

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
  onRetry?: () => void;
}
```

### 10.9 ScoreGauge 评分仪表盘（新增）

**文件**：`features/finance/components/shared/ScoreGauge.tsx`

**Props 设计**：
```typescript
interface ScoreGaugeProps {
  score: number;           // 0-100
  label: string;
  size?: 'sm' | 'md' | 'lg';
  showLabel?: boolean;
}

// 颜色映射: 0-30=红, 30-60=黄, 60-80=绿, 80-100=深绿
```

**实现**：
- 半圆弧形仪表盘
- 数字居中显示
- 底部标签
- 三种尺寸：sm(64px), md(96px), lg(128px)

### 10.10 ConfidenceBadge 置信度徽章（新增）

**文件**：`features/finance/components/shared/ConfidenceBadge.tsx`

**Props 设计**：
```typescript
interface ConfidenceBadgeProps {
  value: number;           // 0-100
  showLabel?: boolean;
  size?: 'sm' | 'md';
}

// 颜色映射: >=70%=绿, 40-69%=黄, <40%=红
// 标签: >=80%="高", 60-79%="中高", 40-59%="中", <40%="低"
```

### 10.11 RiskLevelBadge 风险等级徽章（新增）

**文件**：`features/finance/components/shared/RiskLevelBadge.tsx`

**Props 设计**：
```typescript
interface RiskLevelBadgeProps {
  level: 'low' | 'medium' | 'high' | 'extreme';
  score?: number;
  showScore?: boolean;
  size?: 'sm' | 'md' | 'lg';
}

// 颜色: low=绿, medium=黄, high=橙, extreme=红+脉冲
```

### 10.12 ScenarioCard 场景卡片（新增）

**文件**：`features/finance/components/shared/ScenarioCard.tsx`

**Props 设计**：
```typescript
interface ScenarioCardProps {
  scenario: 'bull' | 'base' | 'bear';
  probability: number;
  expected_change: number;
  description: string;
  expanded?: boolean;
  onClick?: () => void;
}

// 颜色: bull=绿, base=蓝, bear=红
// 样式: 边框颜色 + 背景渐变
```

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

  // AI 模块数据缓存
  aiModules: {
    dailyPrediction: AIDailyPrediction | null;
    riskAlerts: AIRiskAlerts | null;
    opportunitySignals: AIOpportunitySignals | null;
    stockRecommendations: AIRecommendedStock[];
    tradingDecisions: AITradingDecision[];
    lastFetched: Record<string, number>;
  };

  // 设置
  settings: {
    refreshInterval: 30 | 60 | 300;
    defaultMarket: MarketKey;
    compactMode: boolean;
    newsCategory: NewsCategory;
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
  refreshAIModules: () => Promise<void>;
  toggleTheme: () => void;
}
```

### 11.3 数据缓存策略

```typescript
const CACHE_CONFIG = {
  marketIndices: { ttl: 60_000, key: 'market-indices' },
  sentiment: { ttl: 300_000, key: 'market-sentiment' },
  portfolio: { ttl: 30_000, key: 'portfolio-summary' },
  watchlist: { ttl: 30_000, key: 'watchlist' },
  news: { ttl: 120_000, key: 'news-feed' },
  aiDailyPrediction: { ttl: 600_000, key: 'ai-daily-prediction' },
  aiRiskAlerts: { ttl: 300_000, key: 'ai-risk-alerts' },
  aiOpportunitySignals: { ttl: 300_000, key: 'ai-opportunity-signals' },
  aiStockRecommendations: { ttl: 300_000, key: 'ai-stock-recommendations' },
  aiTradingDecisions: { ttl: 300_000, key: 'ai-trading-decisions' },
  sectors: { ttl: 120_000, key: 'sectors' },
  fundCategories: { ttl: 600_000, key: 'fund-categories' },
} as const;

function useCachedData<T>(
  key: string,
  fetcher: () => Promise<T>,
  ttl: number
): { data: T | null; loading: boolean; refresh: () => void; error: string | null } {
  const [data, setData] = useState<T | null>(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);
  const [lastFetched, setLastFetched] = useState(0);

  const refresh = useCallback(async () => {
    setLoading(true);
    setError(null);
    try {
      const result = await fetcher();
      setData(result);
      setLastFetched(Date.now());
    } catch (err) {
      setError(err instanceof Error ? err.message : 'Unknown error');
    } finally {
      setLoading(false);
    }
  }, [fetcher]);

  useEffect(() => {
    if (Date.now() - lastFetched > ttl || !data) {
      void refresh();
    }
  }, [ttl, lastFetched, data, refresh]);

  return { data, loading, refresh, error };
}
```

### 11.4 自动刷新机制

```typescript
function useAutoRefresh(interval: number, callback: () => void) {
  useEffect(() => {
    const timer = setInterval(callback, interval);
    return () => clearInterval(timer);
  }, [interval, callback]);
}

// Dashboard 模块刷新策略
const REFRESH_STRATEGY = {
  marketOverview: 60_000,         // 市场概览: 60s
  portfolio: 30_000,              // 持仓概览: 30s
  newsFeed: 120_000,              // 新闻: 120s
  aiDailyPrediction: 600_000,     // AI 预测: 10min
  aiRiskAlerts: 300_000,          // AI 风险: 5min
  aiOpportunitySignals: 300_000,  // AI 机会: 5min
  aiStockRecommendations: 300_000,// AI 推荐: 5min
  aiTradingDecisions: 300_000,    // AI 决策: 5min
  watchlist: 30_000,              // 自选股: 30s
} as const;
```

---

## 十二、API 接口清单

### 12.1 市场数据接口

| 方法 | 端点 | 说明 | 缓存 |
|------|------|------|------|
| GET | `/api/v1/market/overview` | 全球指数概览 | 60s |
| GET | `/api/v1/market/sectors?type={type}` | 板块列表 | 120s |
| GET | `/api/v1/market/sectors/movers?type={type}` | 板块动向 | 60s |
| GET | `/api/v1/market/sectors/fund-flow` | 板块资金流向 | 60s |
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
| GET | `/api/v1/stock/shareholders/:code` | 股东信息 | 600s |
| GET | `/api/v1/stock/dividends/:code` | 分红历史 | 600s |
| GET | `/api/v1/stock/margin/:code` | 融资融券 | 60s |
| GET | `/api/v1/stock/dragon-tiger/:code` | 龙虎榜 | 120s |
| GET | `/api/v1/stock/block-trades/:code` | 大宗交易 | 120s |

### 12.3 美股接口

| 方法 | 端点 | 说明 | 缓存 |
|------|------|------|------|
| GET | `/api/v1/us/market/overview` | 美股概览 | 60s |
| GET | `/api/v1/us/stock/ranking?type={type}` | 美股排行 | 30s |
| GET | `/api/v1/us/stock/detail/:code` | 美股详情 | 30s |
| GET | `/api/v1/us/stock/institutional/:code` | 13F 机构持仓 | 600s |
| GET | `/api/v1/us/stock/analyst-ratings/:code` | 分析师评级 | 600s |
| GET | `/api/v1/us/stock/options/:code` | 期权链数据 | 60s |
| GET | `/api/v1/us/stock/pre-post/:code` | 盘前盘后数据 | 30s |
| GET | `/api/v1/us/stock/related-etfs/:code` | 相关 ETF | 600s |

### 12.4 韩国股票接口

| 方法 | 端点 | 说明 | 缓存 |
|------|------|------|------|
| GET | `/api/v1/korea/market/overview` | KOSPI 概览 | 60s |
| GET | `/api/v1/korea/stock/ranking?type={type}` | 韩股排行 | 30s |
| GET | `/api/v1/korea/stock/detail/:code` | 韩股详情 | 30s |
| GET | `/api/v1/korea/stock/foreign-holding/:code` | 外国人持股 | 120s |
| GET | `/api/v1/korea/stock/short-selling/:code` | 卖空数据 | 60s |
| GET | `/api/v1/korea/stock/derivatives/:code` | 相关衍生品 | 120s |

### 12.5 基金接口

| 方法 | 端点 | 说明 | 缓存 |
|------|------|------|------|
| GET | `/api/v1/fund/categories` | 基金分类 | 600s |
| GET | `/api/v1/fund/ranking?type={type}&category={cat}` | 基金排名 | 300s |
| GET | `/api/v1/fund/search?q={query}` | 基金搜索 | 无 |
| GET | `/api/v1/fund/detail/:code` | 基金详情 | 600s |
| GET | `/api/v1/fund/performance/:code?period={period}` | 业绩走势 | 600s |
| GET | `/api/v1/fund/holdings/:code` | 持仓明细 | 600s |
| GET | `/api/v1/fund/manager/:code` | 基金经理履历 | 600s |
| GET | `/api/v1/fund/company/:code` | 基金公司背景 | 600s |
| GET | `/api/v1/fund/ratings/:code` | 基金评级 | 600s |
| GET | `/api/v1/fund/scale-trend/:code` | 规模趋势 | 600s |
| GET | `/api/v1/fund/institutional/:code` | 机构持有比例 | 600s |
| GET | `/api/v1/fund/large-alerts/:code` | 大额申赎提醒 | 300s |

### 12.6 AI 分析接口

| 方法 | 端点 | 说明 | 缓存 |
|------|------|------|------|
| GET | `/api/v1/ai/daily-prediction` | AI 今日预测 | 600s |
| GET | `/api/v1/ai/prediction/accuracy` | 预测准确率 | 3600s |
| GET | `/api/v1/ai/prediction/factor-weights` | 因子权重 | 600s |
| GET | `/api/v1/ai/risk-alerts` | AI 风险提示 | 300s |
| GET | `/api/v1/ai/risk-trend?days=30` | 风险趋势 | 300s |
| GET | `/api/v1/ai/risk-history` | 历史风险事件 | 3600s |
| GET | `/api/v1/ai/opportunity-signals` | AI 机会信号 | 300s |
| GET | `/api/v1/ai/opportunity-performance` | 机会历史表现 | 3600s |
| GET | `/api/v1/ai/stock-recommendations?market={market}` | AI 推荐股票 | 300s |
| GET | `/api/v1/ai/trading-decisions?action={action}` | AI 买卖决策 | 300s |
| GET | `/api/v1/ai/market-prediction?horizon={horizon}` | 市场预测 | 600s |
| POST | `/api/v1/ai/stock-analysis` | 个股分析 | 无 |
| POST | `/api/v1/ai/portfolio-optimization` | 组合优化 | 无 |
| POST | `/api/v1/ai/stress-test` | 压力测试 | 无 |
| POST | `/api/v1/ai/risk-assessment` | 风险评估 | 无 |
| GET | `/api/v1/news?category={cat}&hours=24` | 近24小时新闻 | 120s |

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
:root {
  /* 涨跌颜色 */
  --zl-up: 142 70% 45%;
  --zl-up-light: 142 70% 92%;
  --zl-down: 0 70% 55%;
  --zl-down-light: 0 70% 92%;
  --zl-flat: 220 14% 60%;

  /* 信号颜色 */
  --zl-signal-buy: 142 70% 45%;
  --zl-signal-sell: 0 70% 55%;
  --zl-signal-hold: 220 14% 60%;
  --zl-signal-watch: 45 100% 59%;

  /* 风险等级 */
  --zl-risk-low: 142 70% 45%;
  --zl-risk-medium: 45 100% 59%;
  --zl-risk-high: 0 70% 55%;
  --zl-risk-extreme: 0 84% 60%;

  /* 情感标签 */
  --zl-sentiment-positive: 142 70% 45%;
  --zl-sentiment-negative: 0 70% 55%;
  --zl-sentiment-neutral: 220 14% 60%;

  /* 场景颜色 */
  --zl-scenario-bullish: 142 70% 45%;
  --zl-scenario-base: 230 100% 63%;
  --zl-scenario-bearish: 0 70% 55%;

  /* 机会等级 */
  --zl-opp-low: 220 14% 60%;
  --zl-opp-medium: 230 100% 63%;
  --zl-opp-high: 142 70% 45%;
  --zl-opp-very-high: 45 100% 59%;

  /* 新闻分类 */
  --zl-news-macro: 230 100% 63%;
  --zl-news-industry: 142 70% 45%;
  --zl-news-company: 45 100% 59%;
  --zl-news-anomaly: 0 70% 55%;
}

.dark {
  --zl-up: 142 70% 55%;
  --zl-up-light: 142 30% 15%;
  --zl-down: 0 70% 65%;
  --zl-down-light: 0 30% 15%;
  --zl-flat: 230 10% 55%;
  --zl-risk-extreme: 0 84% 65%;
}
```

### 13.2 Tailwind CSS 扩展

```javascript
module.exports = {
  theme: {
    extend: {
      colors: {
        'zl-up': 'hsl(var(--zl-up))',
        'zl-up-light': 'hsl(var(--zl-up-light))',
        'zl-down': 'hsl(var(--zl-down))',
        'zl-down-light': 'hsl(var(--zl-down-light))',
        'zl-flat': 'hsl(var(--zl-flat))',
        'zl-risk-extreme': 'hsl(var(--zl-risk-extreme))',
        'zl-opp-very-high': 'hsl(var(--zl-opp-very-high))',
      },
      fontFamily: {
        'display': ['Roobert PRO', 'Noto Sans', '-apple-system', 'sans-serif'],
      },
      animation: {
        'pulse-slow': 'pulse 3s cubic-bezier(0.4, 0, 0.6, 1) infinite',
      },
    },
  },
};
```

### 13.3 组件样式规范

**卡片样式**：
```css
.card-base {
  background: hsl(var(--zl-canvas));
  border-radius: var(--zl-radius-xl);
  padding: var(--zl-space-xl);
  border: 1px solid hsl(var(--zl-hairline-soft));
}

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
.tabular-nums {
  font-variant-numeric: tabular-nums;
}

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
```

**标签样式**：
```css
.badge-signal {
  padding: 2px 8px;
  border-radius: var(--zl-radius-full);
  font-size: 11px;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.5px;
}

.badge-buy { background: hsl(var(--zl-up-light)); color: hsl(var(--zl-up)); }
.badge-sell { background: hsl(var(--zl-down-light)); color: hsl(var(--zl-down)); }
.badge-hold { background: hsl(var(--zl-surface)); color: hsl(var(--zl-steel)); }

/* 风险等级徽章 */
.badge-risk-low { background: hsl(142 70% 92%); color: hsl(142 70% 45%); }
.badge-risk-medium { background: hsl(45 100% 92%); color: hsl(45 100% 45%); }
.badge-risk-high { background: hsl(0 70% 92%); color: hsl(0 70% 55%); }
.badge-risk-extreme { background: hsl(0 84% 92%); color: hsl(0 84% 60%); animation: pulse 2s infinite; }

/* 机会等级徽章 */
.badge-opp-low { background: hsl(220 14% 92%); color: hsl(220 14% 60%); }
.badge-opp-medium { background: hsl(230 100% 92%); color: hsl(230 100% 63%); }
.badge-opp-high { background: hsl(142 70% 92%); color: hsl(142 70% 45%); }
.badge-opp-very-high { background: hsl(45 100% 88%); color: hsl(45 100% 45%); animation: pulse 2s infinite; }

/* 推荐等级徽章 */
.badge-rec-strong { background: hsl(45 100% 88%); color: hsl(45 100% 45%); }
.badge-rec-buy { background: hsl(142 70% 92%); color: hsl(142 70% 45%); }
.badge-rec-watch { background: hsl(230 100% 92%); color: hsl(230 100% 63%); }
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
| Wide Desktop | >= 1280px | 完整布局 |

### 14.2 组件响应式行为

**MarketTicker**：
- 桌面：完整显示 7 个指数
- 平板：显示 5 个指数，横向滚动
- 移动端：显示 3 个指数，横向滚动

**MarketHeatmap**：
- 桌面：完整热力图，3x3 网格
- 平板：2x3 网格
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
- 桌面：Top 3 卡片 + 时间线列表
- 平板：Top 3 纵排 + 时间线
- 移动端：仅时间线列表

**AI 模块卡片**：
- 桌面：完整布局，所有信息展示
- 平板：适当精简，隐藏次要信息
- 移动端：极简模式，仅核心数据

### 14.3 响应式工具类

```css
.container-finance {
  width: 100%;
  max-width: 1280px;
  margin: 0 auto;
  padding: 0 var(--zl-space-md);
}

@media (min-width: 768px) {
  .container-finance { padding: 0 var(--zl-space-xl); }
}

@media (min-width: 1280px) {
  .container-finance { padding: 0 var(--zl-space-xxl); }
}

.grid-finance {
  display: grid;
  gap: var(--zl-space-md);
  grid-template-columns: 1fr;
}

@media (min-width: 768px) {
  .grid-finance { grid-template-columns: repeat(2, 1fr); }
}

@media (min-width: 1280px) {
  .grid-finance { grid-template-columns: repeat(3, 1fr); }
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
| AI StockRecommendations | 改为列表视图 |
| AI TradingDecisions | 改为紧凑卡片 |

---

## 十五、日间/夜间模式规范

### 15.1 主题切换机制

```typescript
interface ThemeContextType {
  theme: 'light' | 'dark';
  toggleTheme: () => void;
  setTheme: (theme: 'light' | 'dark') => void;
}

function toggleTheme() {
  const next = theme === 'light' ? 'dark' : 'light';
  setTheme(next);
  document.documentElement.classList.toggle('dark', next === 'dark');
  localStorage.setItem('zl-theme', next);
}

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
| 风险极高 | `text-red-700 bg-red-50` | `dark:text-red-400 dark:bg-red-950` |
| 机会极高 | `text-amber-600 bg-amber-50` | `dark:text-amber-400 dark:bg-amber-950` |

### 15.3 过渡动画

```css
* {
  transition: background-color 200ms ease, color 200ms ease, border-color 200ms ease;
}

.no-transition * {
  transition: none !important;
}
```

### 15.4 图表主题适配

```typescript
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

```css
.tab-transition {
  transition: opacity 200ms ease, transform 200ms ease;
}

.tab-enter { opacity: 0; transform: translateY(8px); }
.tab-enter-active { opacity: 1; transform: translateY(0); }
.tab-exit { opacity: 1; transform: translateY(0); }
.tab-exit-active { opacity: 0; transform: translateY(-8px); }
```

### 16.2 数据刷新

```css
.refresh-spin { animation: spin 1s linear infinite; }

@keyframes spin {
  from { transform: rotate(0deg); }
  to { transform: rotate(360deg); }
}

.skeleton-pulse {
  animation: pulse 2s cubic-bezier(0.4, 0, 0.6, 1) infinite;
}

@keyframes pulse {
  0%, 100% { opacity: 1; }
  50% { opacity: 0.5; }
}

/* 风险极高脉冲 */
@keyframes risk-pulse {
  0%, 100% { box-shadow: 0 0 0 0 rgba(239, 68, 68, 0.4); }
  50% { box-shadow: 0 0 0 8px rgba(239, 68, 68, 0); }
}

.badge-risk-extreme { animation: risk-pulse 2s infinite; }
```

### 16.3 交互反馈

```css
.button-press { transition: transform 100ms ease, box-shadow 100ms ease; }
.button-press:active { transform: scale(0.98); }

.card-hover { transition: box-shadow 200ms ease, transform 200ms ease; }
.card-hover:hover {
  box-shadow: var(--zl-shadow-hover);
  transform: translateY(-2px);
}

.number-change { transition: color 300ms ease; }

/* 展开/收起动画 */
.expand-enter { max-height: 0; overflow: hidden; }
.expand-enter-active { max-height: 500px; transition: max-height 300ms ease; }
.expand-exit { max-height: 500px; overflow: hidden; }
.expand-exit-active { max-height: 0; transition: max-height 300ms ease; }
```

### 16.4 空状态引导

```typescript
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

### 16.5 骨架屏模式

```typescript
// 骨架屏组件
interface SkeletonProps {
  variant: 'text' | 'rect' | 'circle' | 'card';
  width?: string | number;
  height?: string | number;
  lines?: number;
  className?: string;
}

// 使用示例
<Skeleton variant="rect" width="100%" height={200} />          // 矩形骨架
<Skeleton variant="text" lines={3} />                           // 文本行骨架
<Skeleton variant="circle" width={48} height={48} />           // 圆形骨架
<Skeleton variant="card" />                                     // 卡片骨架
```

---

## 十七、组件清单与层级关系

### 17.1 组件总览

| 类别 | 组件数 | 说明 |
|------|--------|------|
| 布局组件 | 5 | TabNavigation, SubPageNavigation, BackNavigation, TwoColumnLayout, PageContainer |
| 共享组件 | 14 | StockCard, IndexCard, RankingTable, FilterBar, ChartContainer, EmptyState, LoadingState, ErrorState, ScoreGauge, ConfidenceBadge, RiskLevelBadge, ScenarioCard, Skeleton, Toast |
| 市场组件 | 6 | MarketOverviewCard, MarketHeatmap, SectorMovers, SentimentGauge, IndexGrid, SectorBar |
| 搜索组件 | 4 | StockSearch, USStockSearch, KoreanStockSearch, FundSearch |
| 排行组件 | 4 | StockRanking, USStockRanking, KoreanStockRanking, FundRanking |
| 新闻组件 | 3 | NewsFeedCard, NewsItem, NewsDetailModal |
| 持仓组件 | 3 | PortfolioSummaryCard, PositionList, SectorPieChart |
| Dashboard AI 组件 | 5 | AIDailyPredictionCard, AIRiskAlertsCard, AIOpportunitySignalsCard, AIStockRecommendationsCard, AITradingDecisionsCard |
| Tab 6 AI 组件 | 4 | MarketPredictionCard, StockAnalysisCard, PortfolioOptimizationCard, RiskAssessmentCard |
| 基金组件 | 4 | FundCategories, FundRanking, FundDetail, PeerComparison |
| K 线组件 | 3 | KLineChart, TimeframeSelector, IndicatorSelector |
| 财务组件 | 4 | FinancialData, IncomeStatement, BalanceSheet, CashFlowStatement |
| 详情组件 | 4 | StockDetailPage, USStockDetailPage, KoreanStockDetailPage, FundDetailPage |
| **合计** | **63** | |

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
- `ConfidenceBadge` — 所有 AI 模块

**中复用组件**（被 2 页面使用）：
- `StockSearch` — A 股搜索 + 全局搜索
- `NewsFeedCard` — Dashboard 新闻 + 详情页新闻
- `BackNavigation` — 所有三级页面
- `RiskLevelBadge` — 风险评估 + 风险提示
- `ScoreGauge` — 个股分析 + 组合优化

**低复用组件**（仅 1 页面使用）：
- `MarketHeatmap` — 仅 A 股页面
- `SentimentGauge` — 仅 Dashboard
- `SectorPieChart` — 仅持仓概览
- `ScenarioCard` — 仅市场预测

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
│   │   │   ├── CategoryFilterTabs
│   │   │   ├── TopHighlightCards × 3
│   │   │   └── TimelineList
│   │   ├── AIDailyPredictionCard
│   │   │   ├── PredictionSummary
│   │   │   ├── ConfidenceBadge
│   │   │   ├── ThreeScenarioChart
│   │   │   └── KeyFactorsList
│   │   ├── AIRiskAlertsCard
│   │   │   ├── RiskLevelBadge
│   │   │   ├── RiskTrendChart
│   │   │   └── RiskFactorCards
│   │   ├── AIOpportunitySignalsCard
│   │   │   ├── OpportunityBadge
│   │   │   ├── TypeFilterTabs
│   │   │   └── OpportunityCards
│   │   ├── AIStockRecommendationsCard
│   │   │   ├── MarketFilterTabs
│   │   │   └── StockCardGrid
│   │   ├── AITradingDecisionsCard
│   │   │   ├── ActionFilterTabs
│   │   │   ├── DecisionCardList
│   │   │   └── Disclaimer
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
│   │       ├── ShareholderInfo
│   │       ├── DividendHistory
│   │       ├── MarginTrading
│   │       ├── DragonTiger
│   │       ├── BlockTrades
│   │       ├── QAList
│   │       ├── ResearchReports
│   │       └── AIStockAnalysis
│   ├── USPage
│   │   ├── SubPageNavigation
│   │   ├── USMarketOverview
│   │   ├── USStockSearch
│   │   ├── USStockRanking
│   │   └── USStockDetailPage (三级)
│   │       ├── BackNavigation
│   │       ├── StockHeader
│   │       ├── KLineChart
│   │       ├── FinancialData
│   │       ├── ValuationAnalysis
│   │       ├── InstitutionalHoldings13F
│   │       ├── AnalystRatings
│   │       ├── OptionsChain
│   │       ├── PrePostMarket
│   │       ├── RelatedETFs
│   │       └── AIStockAnalysis
│   ├── KoreaPage
│   │   ├── SubPageNavigation
│   │   ├── KOSPIOverview
│   │   ├── KoreanStockSearch
│   │   ├── KoreanStockRanking
│   │   └── KoreanStockDetailPage (三级)
│   │       ├── BackNavigation
│   │       ├── StockHeader
│   │       ├── KLineChart
│   │       ├── FinancialData
│   │       ├── ForeignHolding
│   │       ├── ShortSelling
│   │       ├── RelatedDerivatives
│   │       └── AIStockAnalysis
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
│   │       ├── ManagerCareer
│   │       ├── CompanyBackground
│   │       ├── FundRatings
│   │       ├── ScaleTrend
│   │       ├── InstitutionalHolding
│   │       ├── LargeAlerts
│   │       ├── StyleDrift
│   │       ├── AIFundRecommendation
│   │       └── PeerComparison
│   └── AIPage
│       ├── SubPageNavigation
│       ├── MarketPredictionCard
│       │   ├── PredictionSummary
│       │   ├── ConfidenceBadge
│       │   ├── ThreeScenarioChart
│       │   ├── KeyFactorsList
│       │   ├── FactorWeightChart
│       │   ├── PredictionAccuracy
│       │   └── ModelExplanation
│       ├── StockAnalysisCard
│       │   ├── ScoreGauge
│       │   ├── RadarChart
│       │   ├── TechnicalAnalysis
│       │   ├── FundamentalAnalysis
│       │   ├── FundFlowAnalysis
│       │   └── NewsAnalysis
│       ├── PortfolioOptimizationCard
│       │   ├── ComparisonCards
│       │   ├── RebalanceSuggestions
│       │   ├── EfficientFrontier
│       │   └── StressTestResult
│       └── RiskAssessmentCard
│           ├── RiskOverview
│           ├── StressTestScenarios
│           ├── CorrelationMatrix
│           ├── VaRAnalysis
│           └── ScenarioAnalysis
└── Footer
```

---

## 十八、验收标准

### 18.1 功能验收标准

**Tab 1: Dashboard**
- [ ] MarketTicker 正确显示 7 个全球指数，60 秒自动刷新
- [ ] MarketOverviewCard 显示 7 个指数卡片 + 恐惧贪婪仪表盘
- [ ] PortfolioSummaryCard 显示总资产、日收益、行业分布饼图
- [ ] NewsFeedCard 显示近 24 小时重要信息，支持分类筛选（全部/宏观/行业/公司/异动）
- [ ] NewsFeedCard Top 3 重要卡片正确高亮，时间线列表按时间分组
- [ ] AIDailyPredictionCard 显示预测摘要 + 置信度 + 三场景概率分布 + 关键因素
- [ ] AIRiskAlertsCard 显示风险等级徽章 + 30 天趋势图 + 风险因素列表
- [ ] AIOpportunitySignalsCard 显示机会等级 + 类型筛选 + 机会卡片列表
- [ ] AIStockRecommendationsCard 显示市场筛选 + 推荐股票卡片网格
- [ ] AITradingDecisionsCard 显示操作筛选 + 决策卡片 + 免责声明
- [ ] WatchlistCard 显示自选股列表，支持添加/删除

**Tab 2: A股/港股**
- [ ] MarketHeatmap 显示板块热力图，红涨绿跌，含资金流向显示
- [ ] SectorMovers 显示板块涨跌排行，含资金流向趋势和估值水平
- [ ] StockSearch 支持代码/名称/拼音搜索，防抖 300ms
- [ ] StockRanking 支持涨幅/跌幅/换手率/成交额排行
- [ ] StockDetail 显示完整详情（K 线/财务/估值/资金流向/股东信息/分红/融资融券/龙虎榜/大宗交易/AI 分析/Q&A）

**Tab 3: 美股**
- [ ] USMarketOverview 显示三大指数 + 热门板块
- [ ] USStockSearch 支持美股搜索
- [ ] USStockRanking 支持美股排行
- [ ] USStockDetail 显示完整详情（含 13F 机构持仓/分析师评级/期权链/盘前盘后/相关 ETF）

**Tab 4: 韩国股票**
- [ ] KOSPIOverview 显示 KOSPI + KOSDAQ 指数
- [ ] KoreanStockSearch 支持韩国股票搜索
- [ ] KoreanStockRanking 支持韩国股票排行
- [ ] KoreanStockDetail 显示完整详情（含外国人持股/卖空数据/相关衍生品）

**Tab 5: 基金**
- [ ] FundCategories 显示 5 类基金卡片
- [ ] FundRanking 支持收益率/夏普比率/最大回撤/规模排行
- [ ] FundSearch 支持基金搜索
- [ ] FundDetail 显示完整详情（含基金经理履历/公司背景/评级/规模趋势/机构持有/大额申赎/AI 推荐/同类对比）

**Tab 6: AI 分析**
- [ ] MarketPrediction 显示多时间框架预测（1d/1w/1m/3m）+ 历史准确率 + 因子权重
- [ ] StockAnalysis 支持输入代码生成四维分析（技术/基本面/资金流/新闻）+ 综合评分
- [ ] PortfolioOptimization 支持持仓输入生成优化建议 + 压力测试 + 有效前沿
- [ ] RiskAssessment 支持持仓输入生成风险评估 + VaR + 压力测试

### 18.2 性能验收标准

| 指标 | 目标 | 说明 |
|------|------|------|
| 首屏加载 | < 2s | 使用骨架屏 + 懒加载 |
| Tab 切换 | < 300ms | 无刷新切换，数据缓存 |
| 数据刷新 | < 1s | 缓存 + 增量更新 |
| 搜索响应 | < 500ms | 防抖 300ms + 缓存 |
| AI 分析 | < 5s | 流式响应 + 加载提示 |
| Dashboard 加载 | < 3s | 9 个模块并行请求 |
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
| 响应式 | 5 个断点布局正确 |
| 交互反馈 | 所有可交互元素有 hover/active 状态 |
| 加载状态 | 所有异步组件有骨架屏 |
| 空状态 | 所有列表有空状态提示 |
| 错误状态 | 所有数据获取有错误处理 |
| AI 模块一致性 | 所有 AI 模块遵循统一的卡片布局模式 |

---

## 十九、实施计划与里程碑

### 19.1 阶段划分

| 阶段 | 内容 | 预计工期 | 依赖 |
|------|------|----------|------|
| **Phase 0** | 基础设施（路由/状态管理/主题/布局） | 3 天 | 无 |
| **Phase 1** | Dashboard（9 个模块） | 7 天 | Phase 0 |
| **Phase 2** | A股/港股（4 个二级 + 1 个三级，含扩展） | 6 天 | Phase 0 |
| **Phase 3** | 美股（3 个二级 + 1 个三级，含扩展） | 5 天 | Phase 0 |
| **Phase 4** | 韩国股票（3 个二级 + 1 个三级，含扩展） | 4 天 | Phase 0 |
| **Phase 5** | 基金（3 个二级 + 1 个三级，含扩展） | 5 天 | Phase 0 |
| **Phase 6** | AI 分析（4 个二级页面，含扩展） | 5 天 | Phase 0 |
| **Phase 7** | 全局搜索 + 交互增强 | 3 天 | Phase 1-6 |
| **Phase 8** | 响应式适配 + 性能优化 | 4 天 | Phase 1-6 |
| **Phase 9** | 测试 + Bug 修复 + 文档 | 4 天 | Phase 1-8 |
| **合计** | | **46 天** | |

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

### 19.3 Phase 1 详细任务（Dashboard 扩展）

```
Phase 1: Dashboard（7 天）
├── Day 1: MarketOverview + PortfolioSummary
│   ├── MarketOverviewCard 组件开发
│   ├── IndexCard × 7 实现
│   ├── SentimentGauge 实现
│   ├── PortfolioSummaryCard 组件开发
│   └── SectorPieChart 实现
├── Day 2: NewsFeed 重写
│   ├── NewsFeedCard "近24小时重要信息" 重构
│   ├── CategoryFilterTabs 实现
│   ├── TopHighlightCards 实现
│   ├── TimelineList 实现
│   └── 分页加载逻辑
├── Day 3: AIDailyPrediction
│   ├── AIDailyPredictionCard 组件开发
│   ├── PredictionSummary + ConfidenceBadge
│   ├── ThreeScenarioChart (Recharts BarChart)
│   ├── KeyFactorsList + 权重条形图
│   └── API 集成 + 缓存策略
├── Day 4: AIRiskAlerts
│   ├── AIRiskAlertsCard 组件开发
│   ├── RiskLevelBadge 实现
│   ├── RiskTrendChart (Recharts AreaChart)
│   ├── RiskFactorCards 列表
│   └── 历史对比折叠面板
├── Day 5: AIOpportunitySignals
│   ├── AIOpportunitySignalsCard 组件开发
│   ├── OpportunityBadge 实现
│   ├── TypeFilterTabs 实现
│   ├── OpportunityCards 列表
│   └── 历史表现折叠面板
├── Day 6: AIStockRecommendations + AITradingDecisions
│   ├── AIStockRecommendationsCard 组件开发
│   ├── MarketFilterTabs + StockCardGrid
│   ├── AITradingDecisionsCard 组件开发
│   ├── ActionFilterTabs + DecisionCardList
│   └── 免责声明组件
└── Day 7: Watchlist + 整合测试
    ├── WatchlistCard 组件开发
    ├── 添加/删除逻辑
    ├── DashboardPage 整合 9 个模块
    ├── 响应式适配
    └── 联调测试
```

### 19.4 文件结构规划

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
│       │   ├── useWatchlist.ts
│       │   └── useAIPrediction.ts
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
│       │   │   ├── ErrorState.tsx
│       │   │   ├── ScoreGauge.tsx
│       │   │   ├── ConfidenceBadge.tsx
│       │   │   ├── RiskLevelBadge.tsx
│       │   │   ├── ScenarioCard.tsx
│       │   │   └── Skeleton.tsx
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
│       │   ├── dashboard-ai/
│       │   │   ├── AIDailyPredictionCard.tsx
│       │   │   ├── AIRiskAlertsCard.tsx
│       │   │   ├── AIOpportunitySignalsCard.tsx
│       │   │   ├── AIStockRecommendationsCard.tsx
│       │   │   └── AITradingDecisionsCard.tsx
│       │   ├── ai/
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
│       │       ├── ShareholderInfo.tsx
│       │       ├── DividendHistory.tsx
│       │       ├── MarginTrading.tsx
│       │       ├── DragonTiger.tsx
│       │       ├── BlockTrades.tsx
│       │       ├── QAList.tsx
│       │       ├── InstitutionalHoldings13F.tsx
│       │       ├── AnalystRatings.tsx
│       │       ├── OptionsChain.tsx
│       │       ├── PrePostMarket.tsx
│       │       ├── RelatedETFs.tsx
│       │       ├── ForeignHolding.tsx
│       │       ├── ShortSelling.tsx
│       │       ├── RelatedDerivatives.tsx
│       │       ├── ManagerCareer.tsx
│       │       ├── CompanyBackground.tsx
│       │       ├── FundRatings.tsx
│       │       ├── ScaleTrend.tsx
│       │       ├── InstitutionalHolding.tsx
│       │       ├── LargeAlerts.tsx
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

### 19.5 依赖库清单

| 库 | 版本 | 用途 |
|----|------|------|
| react-router-dom | ^6.x | 路由管理 |
| recharts | ^2.x | 图表（K 线/饼图/折线图/柱状图/面积图/散点图/雷达图） |
| lucide-react | latest | 图标 |
| tailwindcss | ^3.x | 样式 |
| date-fns | ^3.x | 日期格式化 |
| clsx | ^2.x | 条件类名 |
| tailwind-merge | latest | Tailwind 类名合并 |
| @tanstack/react-query | ^5.x | 数据请求/缓存管理（可选） |

### 19.6 风险与缓解

| 风险 | 影响 | 缓解措施 |
|------|------|----------|
| API 接口未就绪 | 阻塞开发 | 使用 Mock 数据，定义好接口契约 |
| K 线图性能问题 | 卡顿 | 使用 Canvas 渲染，虚拟滚动 |
| 移动端适配工作量大 | 延期 | 优先桌面端，移动端后续迭代 |
| AI 分析响应慢 | 用户体验差 | 流式响应 + 加载提示 + 超时处理 |
| 数据量大导致渲染慢 | 卡顿 | 虚拟滚动 + 分页 + 懒加载 |
| Dashboard 9 模块并行请求 | 压力大 | 分批加载 + 优先加载核心模块 |
| 新增 AI 模块 API 复杂度 | 开发周期 | 分阶段交付，先核心后增强 |

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
  composite_score: number;
  summary: string;
  technical_analysis: TechnicalAnalysis;
  fundamental_analysis: FundamentalAnalysis;
  fund_flow_analysis: FundFlowAnalysisResult;
  news_analysis: NewsAnalysisResult;
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
  horizon: PredictionHorizon;
  scenarios: Scenario[];
  key_factors: KeyFactor[];
  historical_accuracy: PredictionAccuracy;
  factor_weights: FactorWeight[];
  model_explanation: ModelExplanation;
}

interface AIDailyPrediction {
  id: string;
  summary: string;
  confidence: number;
  generated_at: string;
  model_version: string;
  scenarios: ScenarioDistribution;
  key_factors: KeyFactor[];
  prediction_basis: PredictionBasis;
  historical_accuracy: HistoricalAccuracy;
}

interface AIRiskAlerts {
  id: string;
  risk_level: RiskLevel;
  risk_score: number;
  generated_at: string;
  risk_factors: RiskFactor[];
  historical_comparison: HistoricalRiskComparison;
  risk_trend: RiskTrendData;
}

interface AIOpportunitySignals {
  id: string;
  opportunity_level: OpportunityLevel;
  score: number;
  generated_at: string;
  signals: OpportunitySignal[];
  historical_performance: HistoricalPerformance;
}

interface OptimizationResult {
  current_portfolio: PortfolioMetrics;
  optimized_portfolio: PortfolioMetrics;
  suggestions: RebalanceSuggestion[];
  efficient_frontier: EfficientFrontierPoint[];
  risk_return_analysis: RiskReturnAnalysis;
  stress_test: StressTestResult;
  comparison_chart: ComparisonDataPoint[];
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
    shareholders: ShareholderData;
    dividend_history: DividendData[];
    lock_up_expiry: LockUpData[];
    dragon_tiger: DragonTigerData[];
    margin_trading: MarginData;
    block_trades: BlockTradeData[];
    qa_list: QAItem[];
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

// GET /api/v1/ai/daily-prediction
interface DailyPredictionResponse {
  ok: boolean;
  data: AIDailyPrediction;
}

// GET /api/v1/ai/risk-alerts
interface RiskAlertsResponse {
  ok: boolean;
  data: AIRiskAlerts;
}

// GET /api/v1/ai/opportunity-signals
interface OpportunitySignalsResponse {
  ok: boolean;
  data: AIOpportunitySignals;
}
```

### B.4 新闻接口

```typescript
// GET /api/v1/news?category={cat}&hours=24&limit=20&offset=0
interface NewsResponse {
  ok: boolean;
  data: {
    items: NewsItem[];
    total: number;
    has_more: boolean;
  };
}
```

---

> 文档结束 | 共计约 5000 行 | 版本 v5.0

