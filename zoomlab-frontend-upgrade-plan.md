# ZoomLab Web 前端升级技术方案

> 版本: v1.0 | 日期: 2026-05-28 | 状态: 规划中

## 一、项目定位

ZoomLab Web 是个人金融工作台的前端应用，基于 React + TypeScript + Tailwind CSS + Vite 构建。核心模块 BroZoomOut 提供 AI 驱动的金融 intelligence 仪表盘。

**目标**：对标同花顺 70% 的面板功能，同时保持"AI 综合研判"的差异化定位。

---

## 二、现状分析

### 2.1 技术栈

| 层 | 技术 | 版本 |
|----|------|------|
| 框架 | React | 18.x |
| 语言 | TypeScript | 5.x |
| 样式 | Tailwind CSS | 3.x |
| 构建 | Vite | 5.x |
| 图表 | Recharts | 2.x |
| 路由 | React Router | 6.x |
| 状态 | React hooks (useState/useEffect) | - |

### 2.2 BroZoomOut 现有组件

| 组件 | 功能 | 状态 |
|------|------|------|
| `DashboardHero.tsx` | 总览卡片（风险模式、置信度、波动率） | ✅ 已修复主题 |
| `MarketAnalysis.tsx` | 市场分段 Tab（A股/港股/美股等） | ✅ 已修复主题 |
| `StrategyPanel.tsx` | 策略配比（现金/权益/防御） | ✅ 已修复主题 |
| `RiskHistoryPanel.tsx` | 风险因素 + 历史快照 | ✅ 已修复主题 |
| `WatchlistPanel.tsx` | 特别关注列表 | ✅ 已修复主题 |
| `MarketTicker.tsx` | 指数滚动条（新增） | ✅ P0 已完成 |

### 2.3 已知问题

| 问题 | 严重度 | 状态 |
|------|--------|------|
| 日间模式内容区黑色 | P0 | ✅ 已修复 |
| 无实时市场数据 | P0 | ✅ MarketTicker 已添加 |
| 置信度数字不突出 | P0 | ✅ 彩色药丸已添加 |
| 无板块热力图 | P1 | 待开发 |
| 无新闻流 | P1 | 待开发 |
| 无个股推荐 | P1 | 待开发 |
| 无研报摘要 | P1 | 待开发 |
| 无 AI 预测面板 | P2 | 待开发 |
| 无基金推荐 | P2 | 待开发 |
| 无响应式移动端适配 | P2 | 待开发 |

---

## 三、升级方案

### Phase 0 — 已完成 ✅

| 改动 | 组件 | 状态 |
|------|------|------|
| 主题修复（dark: 前缀） | 全部 6 个组件 | ✅ |
| 指数滚动条 | MarketTicker.tsx | ✅ |
| 置信度药丸 | DashboardHero.tsx | ✅ |

### Phase 1 — 核心信息面板（P1）

#### 3.1 市场热力图 `MarketHeatmap.tsx`

**功能**：6 个市场的方块热力图，一眼看全局

**设计**：
- Treemap 布局，方块大小 = 市场权重（可配置）
- 颜色 = 涨跌幅（红绿渐变，-3% 到 +3% 映射）
- 每个方块显示：市场名 + 涨跌幅百分比
- 点击方块 = 切换到该市场的 MarketAnalysis Tab
- 响应式：移动端纵向堆叠

**数据源**：
```typescript
// 从 /api/v1/market/overview 获取
interface MarketOverviewResponse {
  indices: Array<{
    market: string;
    name: string;
    price: number;
    change_percent: number;
  }>;
}
```

**实现**：
- 使用 CSS Grid 实现 Treemap（无需 D3.js）
- 颜色映射：`change_pct` → HSL 色相（120° 绿 → 0° 红）
- 动画：初始加载时方块从 0 透明度渐入

#### 3.2 板块动向 `SectorMovers.tsx`

**功能**：行业/概念板块涨跌排行

**设计**：
- 两个 Tab：行业板块 / 概念板块
- 每个 Tab 显示 Top 10 涨 + Top 10 跌
- 条形图样式：涨=绿色向右，跌=红色向左
- 每条显示：板块名 + 涨跌幅 + 成交量
- 点击板块 = 展开详情（领涨股、关联新闻）

**数据源**：
```typescript
// 从 /api/brozoomout/sectors 获取
interface SectorMoversResponse {
  sectors: Array<{
    name: string;
    change_pct: number;
    volume: number;
    leading_stock: string;
    leading_stock_change: number;
  }>;
}
```

#### 3.3 新闻流 `NewsFeed.tsx`

**功能**：重大新闻实时流，利好🟢/利空🔴标签

**设计**：
- 竖向时间线布局
- 每条新闻卡片：标题 + 来源 + 时间 + 情感标签
- 利好 = 绿色边框 + 🟢，利空 = 红色边框 + 🔴，中性 = 灰色
- 重大新闻 = 左侧黄色竖线标记
- 支持筛选：全部 / 重大 / 利好 / 利空
- 最多显示 20 条，滚动加载更多

**数据源**：
```typescript
// 从 /api/brozoomout/news 获取
interface NewsFeedResponse {
  items: Array<{
    id: string;
    title: string;
    source: string;
    published_at: string;
    importance: "major" | "notable" | "general";
    sentiment_events: Array<{
      type: "bullish" | "bearish" | "neutral";
      impact: "high" | "medium" | "low";
      affected_sectors: string[];
    }>;
    ai_summary: string;
  }>;
}
```

#### 3.4 个股推荐 `StockPicks.tsx`

**功能**：AI 筛选的个股买卖推荐

**设计**：
- 卡片列表，每只股票一张卡片
- 卡片内容：股票名 + 代码 + 市场 + 信号（Buy/Sell/Watch）+ 当前价 + 目标价 + 止损价
- 信号颜色：Buy=绿色，Sell=红色，Watch=黄色
- 展开显示：推荐理由 + 技术指标（RSI、MACD、量比）
- 按信号强度排序

**数据源**：
```typescript
// 从 /api/brozoomout/stock-picks 获取
interface StockPickResponse {
  picks: Array<{
    code: string;
    name: string;
    market: string;
    signal: "buy" | "sell" | "watch";
    signal_strength: "low" | "medium" | "high";
    current_price: number;
    target_price: number;
    stop_loss: number;
    reason: string;
    metrics: {
      rsi: number;
      macd_signal: string;
      volume_ratio: number;
    };
  }>;
}
```

#### 3.5 研报精选 `ResearchDigest.tsx`

**功能**：今日券商研报 Top N + AI 一句话总结

**设计**：
- 横向卡片轮播（或竖向列表）
- 每张卡片：研报标题 + 券商 + 评级 + 目标价 + AI 摘要
- 评级标签颜色：推荐=绿色，中性=灰色，回避=红色
- 点击展开完整摘要

**数据源**：
```typescript
// 从 /api/brozoomout/research 获取
interface ResearchResponse {
  reports: Array<{
    title: string;
    broker: string;
    target_price: number;
    rating: string;
    ai_summary: string;
    published_at: string;
    related_stock: string;
  }>;
}
```

### Phase 2 — AI 智能面板（P2）

#### 3.6 AI 预测 `AIForecast.tsx`

**功能**：AI 未来推演（1 周/1 月），三场景分析

**设计**：
- 顶部：预测摘要 + 置信度
- 中部：三个场景卡片（牛市/基准/熊市）
  - 每个卡片：概率百分比 + 描述 + 影响
  - 牛市=绿色，基准=蓝色，熊市=红色
- 底部：关键因素列表
- Tab 切换：1 周 / 1 月

**数据源**：
```typescript
// 从 /api/brozoomout/forecast 获取
interface ForecastResponse {
  horizon: "1w" | "1m";
  prediction: string;
  confidence: number;
  key_factors: string[];
  scenarios: Array<{
    name: string;
    probability: number;
    description: string;
    impact: string;
  }>;
}
```

#### 3.7 基金推荐 `FundPicks.tsx`

**功能**：同类基金 Top10 推荐

**设计**：
- Tab 切换：股票型 / 债券型 / 混合型 / 指数型
- 每只基金一行：基金名 + 代码 + 近 1/3/6 月收益 + 风险等级
- 收益率颜色：正=绿色，负=红色
- 按近 3 月收益排序

**数据源**：
```typescript
// 从 /api/brozoomout/fund-picks 获取
interface FundPickResponse {
  funds: Array<{
    code: string;
    name: string;
    type: string;
    return_1m: number;
    return_3m: number;
    return_6m: number;
    risk_level: "low" | "medium" | "high";
  }>;
}
```

#### 3.8 风险矩阵 `RiskMatrix.tsx`

**功能**：2×2 风险矩阵（影响 × 概率）

**设计**：
- 四象限：高影响高概率（红）/ 高影响低概率（橙）/ 低影响高概率（黄）/ 低影响低概率（绿）
- 黑天鹅事件固定在右上象限
- 触发条件标注在对应象限
- 历史类比作为注释

### Phase 3 — 交互与体验（P3）

#### 3.9 历史趋势图 `HistoryTrend.tsx`

**功能**：30 天 confidence 折线图 + risk_mode 标注

**设计**：
- 使用 Recharts AreaChart
- X 轴：日期，Y 轴：confidence_score
- 填充颜色随 risk_mode 变化（绿/黄/红）
- hover 显示当日详情

#### 3.10 响应式适配

| 断点 | 布局 |
|------|------|
| < 768px | 单列，指数条横向滚动，Tab 改为下拉 |
| 768-1024px | 两列（主面板 + 侧边栏） |
| > 1024px | 三列（主面板 + 信息流 + 侧边栏） |

#### 3.11 交互增强

| 功能 | 说明 |
|------|------|
| 市场 Tab 联动 | 点 A 股 → 热力图高亮 A 股方块 + 新闻筛选 A 股相关新闻 |
| 刷新动画 | "刷新快照" 加 loading skeleton |
| 空状态引导 | "特别关注" 为空时显示引导卡片 |
| 骨架屏 | 所有异步组件加载时显示 skeleton |

---

## 四、组件架构

### 4.1 新增组件清单

```
src/features/brozoomout/components/
├── DashboardHero.tsx          # ✅ 已有（已修复）
├── MarketAnalysis.tsx         # ✅ 已有（已修复）
├── StrategyPanel.tsx          # ✅ 已有（已修复）
├── RiskHistoryPanel.tsx       # ✅ 已有（已修复）
├── WatchlistPanel.tsx         # ✅ 已有（已修复）
├── MarketTicker.tsx           # ✅ 新增（P0）
├── MarketHeatmap.tsx          # 🆕 P1
├── SectorMovers.tsx           # 🆕 P1
├── NewsFeed.tsx               # 🆕 P1
├── StockPicks.tsx             # 🆕 P1
├── ResearchDigest.tsx         # 🆕 P1
├── AIForecast.tsx             # 🆕 P2
├── FundPicks.tsx              # 🆕 P2
├── RiskMatrix.tsx             # 🆕 P2
├── HistoryTrend.tsx           # 🆕 P3
└── ConfidenceGauge.tsx        # 🆕 P3（可选，替代药丸）
```

### 4.2 页面布局

```
BroZoomOutPage.tsx
├── MarketTicker              # 顶部：指数滚动条（全宽）
├── <main>                    # 主内容区（max-w-6xl）
│   ├── DashboardHero         # AI 总览卡片
│   ├── MarketHeatmap         # 市场热力图
│   ├── <div grid>            # 两列布局
│   │   ├── <div left>        # 左列
│   │   │   ├── StrategyPanel
│   │   │   ├── StockPicks
│   │   │   └── FundPicks
│   │   └── <div right>       # 右列
│   │       ├── NewsFeed
│   │       ├── SectorMovers
│   │       └── ResearchDigest
│   ├── AIForecast            # AI 预测（全宽）
│   ├── RiskMatrix            # 风险矩阵
│   ├── HistoryTrend          # 历史趋势图
│   └── MarketAnalysis        # 市场详情 Tab
└── Footer
```

### 4.3 数据获取模式

所有 API 调用统一通过 `api/broZoomOutClient.ts`：

```typescript
// 统一的 API 客户端
export async function fetchBroZoomOut<T>(endpoint: string, params?: Record<string, string>): Promise<T> {
  const url = new URL(endpoint, API_BASE);
  if (params) Object.entries(params).forEach(([k, v]) => url.searchParams.set(k, v));
  const res = await fetch(url.toString());
  const json = await res.json();
  if (!json.ok) throw new Error(json.detail || 'API error');
  return json.data;
}
```

### 4.4 自定义 Hooks

```typescript
// 数据获取 hook
export function useBroZoomOutData<T>(endpoint: string, params?: Record<string, string>) {
  const [data, setData] = useState<T | null>(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);
  // ... fetch logic with auto-refresh
}
```

---

## 五、设计规范

### 5.1 颜色系统

| 语义 | 亮色模式 | 暗色模式 |
|------|----------|----------|
| 背景 | `bg-white` | `dark:bg-zinc-950` |
| 卡片背景 | `bg-gray-50` | `dark:bg-zinc-900` |
| 文字主色 | `text-gray-900` | `dark:text-white` |
| 文字次色 | `text-gray-500` | `dark:text-gray-400` |
| 边框 | `border-gray-200` | `dark:border-zinc-800` |
| 涨（正） | `text-emerald-500` / `bg-emerald-500/10` | 同左 |
| 跌（负） | `text-red-500` / `bg-red-500/10` | 同左 |
| 中性 | `text-gray-400` | `dark:text-gray-500` |
| 利好 | `text-emerald-600` / `border-emerald-200` | `dark:text-emerald-400` |
| 利空 | `text-red-600` / `border-red-200` | `dark:text-red-400` |

### 5.2 间距规范

| 元素 | 间距 |
|------|------|
| 页面顶部 → 内容 | `py-8` |
| 模块之间 | `mb-8` |
| 卡片内部 | `p-6` |
| 紧凑卡片 | `p-4` |
| 列表项之间 | `space-y-3` |

### 5.3 字体规范

| 用途 | 类 |
|------|-----|
| 页面标题 | `text-2xl font-semibold` |
| 模块标题 | `text-lg font-medium` |
| 正文 | `text-sm` |
| 标签 | `text-xs` |
| 数字（重点） | `text-2xl font-bold tabular-nums` |
| 数字（次重点） | `text-lg font-semibold tabular-nums` |

---

## 六、性能优化

| 优化项 | 方案 |
|--------|------|
| 代码分割 | BroZoomOut 页面使用 `React.lazy` 按路由懒加载 |
| 图表按需加载 | Recharts 组件动态 import |
| 数据缓存 | SWR 或 React Query 简单缓存（5 分钟） |
| 图片优化 | SVG 图标替代 PNG |
| Bundle 分析 | `vite-plugin-visualizer` 监控包大小 |

---

## 七、测试策略

| 层 | 工具 | 覆盖 |
|----|------|------|
| 组件测试 | Vitest + React Testing Library | 所有新组件 |
| E2E 测试 | Playwright | 关键路径（加载→展示→交互） |
| 视觉回归 | Percy 或 Chromatic | 关键页面截图对比 |
| 手动测试 | Chrome DevTools | 移动端适配、暗色模式 |

---

## 八、工期估算

| 阶段 | 内容 | 工期 | 依赖 |
|------|------|------|------|
| P0 | 主题修复 + 指数条 + 置信度 | 1 天 | ✅ 已完成 |
| P1 | 热力图 + 板块 + 新闻 + 个股 + 研报 | 5 天 | 后端 API |
| P2 | AI 预测 + 基金 + 风险矩阵 | 3 天 | 后端 API |
| P3 | 历史趋势 + 响应式 + 交互优化 | 2 天 | P1/P2 |
| **总计** | | **11 天** | |

---

## 九、与同花顺对比

| 功能 | 同花顺 | ZoomLab | 差距分析 |
|------|--------|---------|----------|
| 实时行情 | ✅ 全市场 WebSocket | ⚠️ 11 个指数 REST | 需接行情 WebSocket |
| 板块动向 | ✅ 实时 | ✅ 30 分钟延迟 | 可接受 |
| 个股推荐 | ✅ AI+量化+回测 | ✅ AI 筛选 | 缺回测验证 |
| 研报 | ✅ 全量 | ✅ Top N 摘要 | 覆盖率 70% |
| 新闻 | ✅ 全量+快讯 | ✅ RSS+AI 分级 | 缺快讯推送 |
| 策略建议 | ✅ 付费功能 | ✅ 免费 | 差异化优势 |
| AI 预测 | ❌ 无 | ✅ 独家 | 差异化优势 |
| 风险预警 | ⚠️ 基础 | ✅ 结构化矩阵 | 差异化优势 |
| 基金推荐 | ✅ 全量 | ⚠️ Top N | 需扩充 |
| 研报解读 | ⚠️ 原文 | ✅ AI 一句话总结 | 差异化优势 |

**核心结论**：数据量追不上同花顺（它有全市场 WebSocket），但"AI 综合研判 + 策略建议 + 风险预警"是独有卖点，要持续加强。
