# ZoomLab Web 前端 — 剩余工作技术方案

> 版本: v2.0 | 日期: 2026-05-28 | 状态: 规划中
> P0 已完成：主题修复 + MarketTicker + 置信度药丸

---

## 一、已完成 vs 待做

| 阶段 | 内容 | 状态 |
|------|------|------|
| P0 | 主题修复（dark: 前缀，6 个组件） | ✅ commit `fcb372a` |
| P0 | MarketTicker 指数滚动条 | ✅ commit `fcb372a` |
| P0 | DashboardHero 置信度药丸 | ✅ commit `fcb372a` |
| **P1** | **MarketHeatmap 市场热力图** | 📋 待做 |
| **P1** | **SectorMovers 板块动向** | 📋 待做 |
| **P1** | **NewsFeed 新闻流** | 📋 待做 |
| **P1** | **StockPicks 个股推荐** | 📋 待做 |
| **P1** | **ResearchDigest 研报精选** | 📋 待做 |
| **P2** | **AIForecast AI 预测面板** | 📋 待做 |
| **P2** | **FundPicks 基金推荐** | 📋 待做 |
| **P2** | **RiskMatrix 风险矩阵** | 📋 待做 |
| **P3** | **HistoryTrend 历史趋势图** | 📋 待做 |
| **P3** | **响应式移动端适配** | 📋 待做 |
| **P3** | **交互增强（联动/动画/空状态）** | 📋 待做 |

---

## 二、现有组件清单（已实现）

| 组件 | 文件 | P0 改动 |
|------|------|---------|
| DashboardHero | `components/DashboardHero.tsx` | ✅ 主题 + 置信度药丸 |
| MarketAnalysis | `components/MarketAnalysis.tsx` | ✅ 主题修复 |
| StrategyPanel | `components/StrategyPanel.tsx` | ✅ 主题修复 |
| RiskHistoryPanel | `components/RiskHistoryPanel.tsx` | ✅ 主题修复 |
| WatchlistPanel | `components/WatchlistPanel.tsx` | ✅ 主题修复 |
| MarketTicker | `components/MarketTicker.tsx` | 🆕 新增 |
| BroZoomOutPage | `pages/BroZoomOutPage.tsx` | ✅ 集成 MarketTicker |

---

## 三、待做组件详细设计

### P1-1：MarketHeatmap 市场热力图

**文件**：`components/MarketHeatmap.tsx`

**功能**：6 个市场的方块热力图，一眼看全局涨跌

**数据源**：
```typescript
GET /api/v1/market/overview
→ data.indices: Array<{market, name, price, change_percent}>
```

**设计**：
```
┌──────────────────────────────────────────────┐
│  A 股          │  港股          │  美股      │
│  上证 -1.25%   │  恒生 -1.06%   │  S&P -0.1% │
│  ████████░░    │  ██████░░░░    │  ░░░░░░░░  │
│                │                │            │
│  深证 -0.88%   │  恒科 -0.80%   │  NASDAQ    │
│  ███████░░░    │  ██████░░░░    │  -0.20%    │
│                │                │  ░░░░░░░░  │
├────────────────┼────────────────┼────────────┤
│  创业板 +0.07% │  韩股          │            │
│  ░░░░░░░░░░    │  KOSPI +2.25%  │            │
│                │  ██████████    │            │
└────────────────┴────────────────┴────────────┘
```

**实现**：
- CSS Grid 布局，3 列 × 2 行
- 方块大小 = 固定比例（非 treemap）
- 颜色映射：`change_pct` → HSL 色相（120° 绿 → 0° 红）
- -3% 到 +3% 线性映射
- 每个方块显示：市场名 + 指数名 + 涨跌幅
- hover 效果：放大 + 显示详情 tooltip
- 点击方块 → 切换 MarketAnalysis Tab

**TypeScript 接口**：
```typescript
interface HeatmapItem {
  market: string;
  name: string;
  price: number;
  change_pct: number;
  gridArea: string;  // CSS grid-area
}

function getHeatmapColor(pct: number): string {
  // -3% → red (0°), 0% → gray (210°), +3% → green (120°)
  const clamped = Math.max(-3, Math.min(3, pct));
  const hue = clamped > 0
    ? 120 - (clamped / 3) * 90   // 120° → 30°
    : 210 + (clamped / 3) * 30;  // 210° → 240°
  return `hsl(${hue}, 70%, 45%)`;
}
```

---

### P1-2：SectorMovers 板块动向

**文件**：`components/SectorMovers.tsx`

**功能**：行业/概念板块涨跌排行

**数据源**：
```typescript
GET /api/brozoomout/sectors?type=industry&limit=10
→ data.sectors: Array<{name, change_pct, volume, leading_stock}>
```

**设计**：
```
┌─────────────────────────────────────┐
│  [行业板块]  [概念板块]              │
├─────────────────────────────────────┤
│  🔺 银行          +2.15%  ████████ │
│  🔺 房地产        +1.80%  ██████   │
│  🔺 医药          +0.95%  ███      │
│  ─  化工          +0.10%  ░        │
│  ─  电子          -0.05%  ░        │
│  🔻 半导体        -2.30%  ████████ │
│  🔻 AI            -3.10%  ██████████│
└─────────────────────────────────────┘
```

**实现**：
- 两个 Tab：行业板块 / 概念板块
- 涨 = 绿色向右条形图，跌 = 红色向左条形图
- 中轴线居中，涨跌对称显示
- 每条：板块名 + 涨跌幅 + 领涨股
- 排序：涨幅降序（涨在上，跌在下）

---

### P1-3：NewsFeed 新闻流

**文件**：`components/NewsFeed.tsx`

**功能**：重大新闻实时流，利好🟢/利空🔴标签

**数据源**：
```typescript
GET /api/brozoomout/news?importance=all&limit=20
→ data.items: Array<{id, title, source, importance, sentiment_events, ai_summary}>
```

**设计**：
```
┌─────────────────────────────────────┐
│  📰 新闻动态           [全部|重大|利好|利空] │
├─────────────────────────────────────┤
│  🟢 央行降准 50bp           10:00  │
│     释放约 1 万亿流动性，利好金融    │
│     影响板块：银行、地产             │
│  ─────────────────────────────────  │
│  🔴 美联储鹰派发言          08:30  │
│     市场担忧加息持续               │
│     影响板块：科技、成长             │
│  ─────────────────────────────────  │
│  ⚪ A 股缩量震荡           昨日    │
│     成交量创 3 个月新低            │
└─────────────────────────────────────┘
```

**实现**：
- 竖向时间线布局
- 每条新闻卡片：情感标签 + 标题 + 时间 + AI 摘要 + 影响板块
- 利好 = 绿色左边框 + 🟢
- 利空 = 红色左边框 + 🔴
- 中性 = 灰色左边框 + ⚪
- 重大新闻 = 左侧黄色竖线标记
- 顶部筛选按钮：全部 / 重大 / 利好 / 利空
- 最多 20 条，滚动加载

---

### P1-4：StockPicks 个股推荐

**文件**：`components/StockPicks.tsx`

**功能**：AI 筛选的个股买卖推荐

**数据源**：
```typescript
GET /api/brozoomout/stock-picks?signal=all&limit=10
→ data.picks: Array<{code, name, signal, current_price, target_price, reason, metrics}>
```

**设计**：
```
┌─────────────────────────────────────┐
│  📊 个股推荐                        │
├─────────────────────────────────────┤
│  ┌─────────────────────────────┐    │
│  │  BUY  招商银行 600036       │    │
│  │  ¥38.50 → ¥42.00 (+9.1%)  │    │
│  │  止损 ¥36.00               │    │
│  │  RSI 32 | MACD 金叉 | 量比 2.1│   │
│  │  "银行板块领涨，超卖反弹..."  │    │
│  └─────────────────────────────┘    │
│  ┌─────────────────────────────┐    │
│  │  WATCH 宁德时代 300750      │    │
│  │  ¥180.00 → ¥200.00 (+11%) │    │
│  │  ...                      │    │
│  └─────────────────────────────┘    │
└─────────────────────────────────────┘
```

**实现**：
- 卡片列表，每只股票一张
- 信号颜色：BUY=绿色，SELL=红色，WATCH=黄色
- 显示：股票名 + 代码 + 当前价 → 目标价 + 涨幅 + 止损价
- 技术指标行：RSI | MACD | 量比
- 推荐理由：AI 生成的一句话
- 按信号强度排序（high > medium > low）

---

### P1-5：ResearchDigest 研报精选

**文件**：`components/ResearchDigest.tsx`

**功能**：今日券商研报 Top N + AI 一句话总结

**数据源**：
```typescript
GET /api/brozoomout/research?days=1&limit=5
→ data.reports: Array<{title, broker, rating, target_price, ai_summary}>
```

**设计**：
```
┌─────────────────────────────────────┐
│  📄 研报精选                        │
├─────────────────────────────────────┤
│  中金公司 | 推荐 | 目标 ¥45.00     │
│  招商银行深度：零售之王再出发       │
│  "中金看好招行零售转型，12x PE"     │
│  ─────────────────────────────────  │
│  华泰证券 | 买入 | 目标 ¥220.00    │
│  宁德时代：全球电池龙头地位稳固     │
│  "维持买入，海外产能扩张超预期"     │
└─────────────────────────────────────┘
```

**实现**：
- 竖向列表，每条一行
- 评级标签颜色：推荐/买入=绿色，中性=灰色，回避=红色
- 显示：券商 + 评级 + 目标价 + 标题 + AI 摘要
- 点击展开完整摘要（折叠式）

---

### P2-1：AIForecast AI 预测面板

**文件**：`components/AIForecast.tsx`

**功能**：AI 未来推演（1 周/1 月），三场景分析

**数据源**：
```typescript
GET /api/brozoomout/forecast?horizon=1w
→ data: {prediction, confidence, key_factors, scenarios}
```

**设计**：
```
┌─────────────────────────────────────┐
│  🔮 AI 预测            [1 周] [1 月]│
├─────────────────────────────────────┤
│  "下周市场大概率维持震荡，          │
│   关注美联储议息会议结果"           │
│  置信度: 65%                        │
│                                     │
│  ┌──────┐ ┌──────┐ ┌──────┐       │
│  │ 🟢   │ │ 🔵   │ │ 🔴   │       │
│  │ 牛市  │ │ 基准  │ │ 熊市  │       │
│  │ 25%  │ │ 50%  │ │ 25%  │       │
│  │鸽派+ │ │维持  │ │鹰派+ │       │
│  │北向流│ │震荡  │ │地缘  │       │
│  └──────┘ └──────┘ └──────┘       │
│                                     │
│  关键因素：                          │
│  • 美联储议息会议                    │
│  • A 股量能萎缩                      │
│  • 北向资金流向                      │
└─────────────────────────────────────┘
```

**实现**：
- Tab 切换：1 周 / 1 月
- 顶部：预测摘要文字 + 置信度
- 中部：三个场景卡片（横向排列）
  - 牛市=绿色边框，基准=蓝色边框，熊市=红色边框
  - 每个卡片：概率% + 描述 + 影响
- 底部：关键因素列表

---

### P2-2：FundPicks 基金推荐

**文件**：`components/FundPicks.tsx`

**功能**：同类基金 Top N 推荐

**数据源**：
```typescript
GET /api/brozoomout/fund-picks?type=all&limit=10
→ data.funds: Array<{code, name, type, return_1m, return_3m, return_6m, risk_level}>
```

**设计**：
```
┌─────────────────────────────────────┐
│  💰 基金推荐                        │
│  [股票型] [债券型] [混合型] [指数型] │
├─────────────────────────────────────┤
│  招商中证白酒 161725               │
│  近 1 月 +5.2% | 3 月 +12.8% | 6 月 -3.5%│
│  风险: 中等 | 规模: 150 亿         │
│  ─────────────────────────────────  │
│  易方达蓝筹精选 005827             │
│  近 1 月 +3.1% | 3 月 +8.5% | 6 月 +2.1% │
│  风险: 中高 | 规模: 320 亿         │
└─────────────────────────────────────┘
```

**实现**：
- Tab 切换：股票型 / 债券型 / 混合型 / 指数型
- 每只基金一行
- 收益率颜色：正=绿色，负=红色
- 风险等级标签：低=绿色，中=黄色，高=红色
- 按近 3 月收益排序

---

### P2-3：RiskMatrix 风险矩阵

**文件**：`components/RiskMatrix.tsx`

**功能**：2×2 风险矩阵（影响 × 概率）

**数据源**：
```typescript
// 从 /api/brozoomout/dashboard 获取
data.risk: {
  top_risk_factors: string[],
  potential_black_swan: string[],
  trigger_conditions: string[]
}
```

**设计**：
```
┌─────────────────────────────────────┐
│  ⚠️ 风险矩阵                        │
├─────────────────────────────────────┤
│          │  低概率        │  高概率    │
│  ────────┼───────────────┼──────────│
│  高影响  │  🟠 黑天鹅     │  🔴 关键  │
│          │  地缘政治事件   │  信号分化  │
│          │  政策意外转向   │  波动上升  │
│  ────────┼───────────────┼──────────│
│  低影响  │  🟢 忽略       │  🟡 关注  │
│          │  技术性调整     │  量能萎缩  │
└─────────────────────────────────────┘
```

**实现**：
- CSS Grid 2×2 布局
- 四象限颜色：红（高高）、橙（高低）、黄（低高）、绿（低低）
- 每个象限内列出对应风险因素
- 黑天鹅固定在右上象限

---

### P3-1：HistoryTrend 历史趋势图

**文件**：`components/HistoryTrend.tsx`

**功能**：30 天 confidence 折线图 + risk_mode 标注

**数据源**：
```typescript
GET /api/brozoomout/archive?days=30
→ data.items: Array<{snapshot_id, as_of, risk_mode, confidence_score}>
```

**实现**：
- 使用 Recharts AreaChart
- X 轴：日期，Y 轴：confidence_score（0-100）
- 填充颜色随 risk_mode 变化：
  - risk_on = 绿色区域
  - neutral = 黄色区域
  - risk_off = 红色区域
- hover 显示当日详情 tooltip
- risk_mode 变化点用竖线标注

---

### P3-2：响应式适配

| 断点 | 布局 |
|------|------|
| < 768px | 单列，指数条横向滚动，Tab 改为下拉菜单 |
| 768-1024px | 两列（主面板 + 侧边栏） |
| > 1024px | 三列（主面板 + 信息流 + 侧边栏） |

**移动端特殊处理**：
- MarketHeatmap：改为 2×3 网格
- SectorMovers：改为横向滑动卡片
- NewsFeed：全宽，卡片堆叠
- StockPicks：全宽，卡片堆叠
- AIForecast：场景卡片纵向堆叠

---

### P3-3：交互增强

| 功能 | 说明 |
|------|------|
| 市场 Tab 联动 | 点 A 股 → 热力图高亮 + 新闻筛选 A 股相关 |
| 刷新动画 | "刷新快照" 按钮加 loading skeleton |
| 空状态引导 | "特别关注" 为空时显示引导卡片 |
| 骨架屏 | 所有异步组件加载时显示 skeleton |
| 数据刷新 | 切换 Tab 时重新请求对应市场数据 |

---

## 四、目标布局

```
BroZoomOutPage.tsx
├── MarketTicker              # 顶部：指数滚动条（全宽）
├── <main>                    # 主内容区（max-w-6xl）
│   ├── DashboardHero         # AI 总览卡片
│   ├── MarketHeatmap         # 🆕 市场热力图
│   ├── <div grid>            # 两列布局
│   │   ├── <div left>        # 左列
│   │   │   ├── StrategyPanel
│   │   │   ├── StockPicks    # 🆕
│   │   │   └── FundPicks     # 🆕
│   │   └── <div right>       # 右列
│   │       ├── NewsFeed      # 🆕
│   │       ├── SectorMovers  # 🆕
│   │       └── ResearchDigest # 🆕
│   ├── AIForecast            # 🆕 AI 预测（全宽）
│   ├── RiskMatrix            # 🆕 风险矩阵
│   ├── HistoryTrend          # 🆕 历史趋势图
│   └── MarketAnalysis        # 市场详情 Tab
└── Footer
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
| 涨（正） | `text-emerald-500` | 同左 |
| 跌（负） | `text-red-500` | 同左 |
| 利好 | `text-emerald-600` | `dark:text-emerald-400` |
| 利空 | `text-red-600` | `dark:text-red-400` |

### 5.2 字体规范

| 用途 | 类 |
|------|-----|
| 页面标题 | `text-2xl font-semibold` |
| 模块标题 | `text-lg font-medium` |
| 正文 | `text-sm` |
| 标签 | `text-xs` |
| 数字（重点） | `text-2xl font-bold tabular-nums` |

---

## 六、数据获取模式

统一通过 `api/broZoomOutClient.ts`：

```typescript
export async function fetchBroZoomOut<T>(
  endpoint: string,
  params?: Record<string, string>
): Promise<T> {
  const url = new URL(endpoint, API_BASE);
  if (params) Object.entries(params).forEach(([k, v]) => url.searchParams.set(k, v));
  const res = await fetch(url.toString());
  const json = await res.json();
  if (!json.ok) throw new Error(json.detail || 'API error');
  return json.data;
}
```

---

## 七、工期估算

| 阶段 | 模块 | 工期 | 依赖后端 |
|------|------|------|----------|
| P1 | MarketHeatmap | 1 天 | /api/v1/market/overview（已有） |
| P1 | SectorMovers | 1 天 | /api/brozoomout/sectors（待开发） |
| P1 | NewsFeed | 1.5 天 | /api/brozoomout/news（待开发） |
| P1 | StockPicks | 1.5 天 | /api/brozoomout/stock-picks（待开发） |
| P1 | ResearchDigest | 1 天 | /api/brozoomout/research（待开发） |
| P2 | AIForecast | 1.5 天 | /api/brozoomout/forecast（待开发） |
| P2 | FundPicks | 1 天 | /api/brozoomout/fund-picks（待开发） |
| P2 | RiskMatrix | 0.5 天 | /api/brozoomout/dashboard（已增强） |
| P3 | HistoryTrend | 1 天 | /api/brozoomout/archive（已增强） |
| P3 | 响应式适配 | 1 天 | 无 |
| P3 | 交互增强 | 1 天 | 无 |
| **总计** | | **12 天** | |

---

## 八、与同花顺对比（P0 后）

| 功能 | 同花顺 | ZoomLab（当前） | 差距 |
|------|--------|-----------------|------|
| 实时行情 | ✅ 全市场 WebSocket | ✅ 7 指数 REST | 需接 WebSocket |
| 主题适配 | ✅ 完整 | ✅ 已修复 | 已解决 |
| 板块动向 | ✅ 实时 | ❌ 待开发 | P1 |
| 个股推荐 | ✅ AI+量化 | ❌ 待开发 | P1 |
| 新闻 | ✅ 全量+快讯 | ❌ 待开发 | P1 |
| 研报 | ✅ 全量 | ❌ 待开发 | P1 |
| AI 预测 | ❌ 无 | ❌ 待开发 | P2（差异化） |
| 基金推荐 | ✅ 全量 | ❌ 待开发 | P2 |
| 风险预警 | ⚠️ 基础 | ❌ 待开发 | P2（差异化） |
| 策略建议 | ✅ 付费 | ✅ 免费 | 差异化优势 |
