# BroZoomOut 仪表盘渐进式改造 Prompt

> 适用于 Trae Solo Coding 或类似在线编码 Agent，分阶段逐步改善 BroZoomOut 页面。

---

## 项目背景

你正在改造一个名为 **BroZoomOut** 的个人金融分析仪表盘。它是 **zoomlab-web** 项目的一部分。

### 项目位置

- 主项目：`/Volumes/ExSSD/Projects/zoomlab-web/`
- 前端：`/Volumes/ExSSD/Projects/zoomlab-web/frontend/`（React 18 + TypeScript + Vite + Tailwind CSS + shadcn/ui）
- 后端：`/Volumes/ExSSD/Projects/zoomlab-web/backend/`（Go + Gin + SQLite）
- 外部服务：`/Volumes/ExSSD/Projects/Bro,Zoom Out/finance-intelligence-service/`（Python + FastAPI，提供市场认知数据）
- 设计系统文档：`/Volumes/ExSSD/Projects/zoomlab-web/DESIGN.md`

### 技术栈

- **前端**：React 18, TypeScript, Vite, Tailwind CSS, shadcn/ui, Radix UI, Lucide Icons
- **后端**：Go 1.23, Gin, SQLite, JWT 认证
- **外部服务**：Python FastAPI, SQLAlchemy, OpenAI-compatible LLM API
- **部署**：Docker Compose, Cloudflare Tunnel

### 当前状态

BroZoomOut 页面目前是一个信息聚合仪表盘，从 finance-intelligence-service 获取数据并展示。主要问题：

1. **数据空洞**：Actions（行动建议）和 Watchlist（观察清单）返回空数组
2. **图表抽象**：信号数量、分类分布等图表使用合成数据，非真实市场数据
3. **内容泛化**：Insights（洞察）部分包含硬编码的通用文本，缺乏个人化分析
4. **缺乏金融实体**：没有股票、ETF、基金等具体金融产品的跟踪
5. **缺少交互**：没有收藏、笔记、提醒等用户交互功能
6. **行动建议空洞**：`buildActions()` 和 `buildWatchlist()` 函数直接返回空数组

### 核心数据流

```
finance-intelligence-service (Python)
  ↓ HTTP API
zoomlab-web/backend (Go)
  ↓ /api/brozoomout/today
zoomlab-web/frontend (React)
  ↓ BroZoomOutPage.tsx
  ↓ 各子组件渲染
```

### 现有 API 端点（finance-intelligence-service）

| 端点 | 用途 |
|------|------|
| `/api/v1/experience/morning-brief` | 每日晨间简报 |
| `/api/v1/experience/feed` | 认知信息流 |
| `/api/v1/experience/timeline` | 认知时间线 |
| `/api/v1/market/reports/timeline` | 风险评分时间线 |
| `/api/v1/market/overview` | 市场概览 |
| `/api/v1/market/daily-report` | 每日报告 |

---

## 改造目标

将 BroZoomOut 从一个抽象的信息聚合页面，改造为一个**实用的个人金融分析仪表盘**，具备：

1. **真实市场数据展示**：股票、ETF、指数的实时/历史数据
2. **个人化分析**：基于用户持仓和关注标的的定制化洞察
3. **可操作的建议**：具体的买入/卖出/持有建议，附带理由
4. **风险管理**：仓位监控、止损提醒、相关性分析
5. **历史回溯**：市场认知的时间线演进

---

## 分阶段实施计划

### Phase 1：数据层修复与基础 UI 优化（预计 2-3 小时）

**目标**：修复空数据问题，优化现有组件的视觉表现

#### 1.1 后端：修复空数据函数

**文件**：`/Volumes/ExSSD/Projects/zoomlab-web/backend/internal/brozoomout/handler.go`

修改 `buildActions()` 函数，从 morning brief 和 feed 数据中提取可操作的建议：

```go
func buildActions(mb *morningBriefResponse, feed *feedResponse) []BroZoomOutAction {
    var actions []BroZoomOutAction
    
    // 从 morning brief 的 key observations 提取行动建议
    if mb != nil && len(mb.KeyObservations) > 0 {
        for i, obs := range mb.KeyObservations {
            if i >= 3 { break } // 最多3条
            actions = append(actions, BroZoomOutAction{
                ID:            fmt.Sprintf("action-%03d", i+1),
                Title:         obs,
                Description:   "基于今日关键观察的行动建议",
                Difficulty:    "medium",
                EstimatedTime: "15-30分钟",
                Tier:          "collect",
            })
        }
    }
    
    // 从 feed 中高重要性条目提取行动建议
    if feed != nil {
        for _, item := range feed.Feed {
            if utf8.RuneCountInString(item.Content) > 150 {
                actions = append(actions, BroZoomOutAction{
                    ID:            fmt.Sprintf("action-feed-%s", item.Type),
                    Title:         fmt.Sprintf("关注：%s", item.Title),
                    Description:   item.CalmReflection,
                    Difficulty:    "easy",
                    EstimatedTime: "5-10分钟",
                    Tier:          "quick",
                })
            }
        }
    }
    
    if len(actions) == 0 {
        return []BroZoomOutAction{}
    }
    return actions[:min(len(actions), 5)] // 最多返回5条
}
```

同样修改 `buildWatchlist()` 函数：

```go
func buildWatchlist(mb *morningBriefResponse, feed *feedResponse) []BroZoomOutWatchlistItem {
    var watchlist []BroZoomOutWatchlistItem
    
    if mb != nil {
        for i, signal := range mb.KeyObservations {
            if i >= 5 { break }
            watchlist = append(watchlist, BroZoomOutWatchlistItem{
                ID:       fmt.Sprintf("watch-%03d", i+1),
                Title:    signal,
                Category: "关键信号",
                AddedAt:  mb.Date,
            })
        }
    }
    
    return watchlist
}
```

**验证**：调用 `/api/brozoomout/today` 确认 actions 和 watchlist 不再为空。

#### 1.2 前端：优化 Hero 区域

**文件**：`/Volumes/ExSSD/Projects/zoomlab-web/frontend/src/features/brozoomout/components/BroZoomOutHero.tsx`

改进 regime 状态的视觉表现，添加更明确的状态指示器：

- 为每种 regime（risk-on, risk-off, transition, high_volatility）添加对应的图标
- 优化置信度进度条的动画效果
- 添加 "最后更新时间" 的相对时间显示（如 "2小时前"）

#### 1.3 前端：优化 SignalFeed 组件

**文件**：`/Volumes/ExSSD/Projects/zoomlab-web/frontend/src/features/brozoomout/components/SignalFeed.tsx`

- 添加信号来源的可视化标识（不同来源用不同颜色/图标）
- 优化展开/收起动画
- 添加 "标记为已读" 功能（本地状态）

**验收标准**：
- [ ] Actions 列表显示至少 3 条行动建议
- [ ] Watchlist 显示至少 3 条观察项
- [ ] Hero 区域 regime 状态有对应图标
- [ ] SignalFeed 展开/收起动画流畅

---

### Phase 2：引入真实市场数据（预计 4-6 小时）

**目标**：集成股票、ETF、指数的真实行情数据

#### 2.1 后端：添加市场数据 API

**新文件**：`/Volumes/ExSSD/Projects/zoomlab-web/backend/internal/market/handler.go`

创建市场数据模块，提供以下端点：

```go
// GET /api/market/quotes?symbols=AAPL,MSFT,000001.SZ
// GET /api/market/indices
// GET /api/market/etf/:symbol/history
```

数据源优先级：
1. AKShare（A股、港股）
2. Yahoo Finance（美股、全球）
3. Alpha Vantage（备用）

**文件**：`/Volumes/ExSSD/Projects/zoomlab-web/backend/internal/market/client.go`

```go
type MarketClient struct {
    akshareBaseURL string
    yahooBaseURL   string
    http           *http.Client
}

func (c *MarketClient) GetQuote(ctx context.Context, symbol string) (*Quote, error) {
    // 根据 symbol 后缀选择数据源
    // .SZ/.SH -> AKShare
    // others -> Yahoo Finance
}

func (c *MarketClient) GetIndexQuotes(ctx context.Context) ([]IndexQuote, error) {
    // 返回主要指数：上证、深证、恒生、标普、纳斯达克
}
```

#### 2.2 后端：扩展 BroZoomOut 数据结构

**文件**：`/Volumes/ExSSD/Projects/zoomlab-web/backend/internal/brozoomout/handler.go`

添加市场数据到 BroZoomOutData：

```go
type BroZoomOutData struct {
    // ... 现有字段
    MarketOverview  *MarketOverview  `json:"marketOverview"`
    WatchlistQuotes []Quote          `json:"watchlistQuotes"`
}

type MarketOverview struct {
    Indices    []IndexQuote `json:"indices"`
    HotSectors []string     `json:"hotSectors"`
    Sentiment  string       `json:"sentiment"`
}
```

#### 2.3 前端：添加市场概览组件

**新文件**：`/Volumes/ExSSD/Projects/zoomlab-web/frontend/src/features/brozoomout/components/MarketOverview.tsx`

```tsx
export function MarketOverview({ data }: { data: MarketOverview }) {
  return (
    <section className="mb-8">
      <h3 className="text-lg font-semibold text-white mb-4">市场概览</h3>
      <div className="grid grid-cols-2 md:grid-cols-4 gap-4">
        {data.indices.map(index => (
          <IndexCard key={index.symbol} index={index} />
        ))}
      </div>
      <div className="mt-4 flex gap-2">
        {data.hotSectors.map(sector => (
          <Badge key={sector} variant="secondary">{sector}</Badge>
        ))}
      </div>
    </section>
  );
}
```

#### 2.4 前端：添加持仓跟踪组件

**新文件**：`/Volumes/ExSSD/Projects/zoomlab-web/frontend/src/features/brozoomout/components/PortfolioTracker.tsx`

显示用户关注的股票/ETF 实时行情，支持：
- 添加/删除关注标的
- 显示当前价格、涨跌幅
- 显示技术指标（MA5, MA20, RSI）

**验收标准**：
- [ ] 市场概览显示主要指数行情
- [ ] 可添加最多 10 个关注标的
- [ ] 关注标的显示实时价格和涨跌幅
- [ ] 数据刷新间隔 ≤ 5 分钟

---

### Phase 3：个人化分析与建议（预计 6-8 小时）

**目标**：基于用户持仓和市场数据，生成个性化分析

#### 3.1 后端：持仓管理 API

**新文件**：`/Volumes/ExSSD/Projects/zoomlab-web/backend/internal/portfolio/handler.go`

```go
// POST /api/portfolio/positions - 添加持仓
// GET /api/portfolio/positions - 获取持仓列表
// DELETE /api/portfolio/positions/:id - 删除持仓
// GET /api/portfolio/summary - 持仓汇总（总市值、盈亏、行业分布）
```

**新文件**：`/Volumes/ExSSD/Projects/zoomlab-web/backend/internal/portfolio/models.go`

```go
type Position struct {
    ID        string    `json:"id"`
    UserID    string    `json:"userId"`
    Symbol    string    `json:"symbol"`
    Name      string    `json:"name"`
    Quantity  float64   `json:"quantity"`
    AvgCost   float64   `json:"avgCost"`
    CreatedAt time.Time `json:"createdAt"`
}

type PortfolioSummary struct {
    TotalValue   float64            `json:"totalValue"`
    TotalCost    float64            `json:"totalCost"`
    TotalPnL     float64            `json:"totalPnL"`
    PnLPercent   float64            `json:"pnlPercent"`
    Positions    []PositionDetail    `json:"positions"`
    SectorAlloc  map[string]float64  `json:"sectorAlloc"`
}
```

#### 3.2 后端：AI 分析服务

**新文件**：`/Volumes/ExSSD/Projects/zoomlab-web/backend/internal/analysis/handler.go`

```go
// POST /api/analysis/position/:id - 单标的分析
// POST /api/analysis/portfolio - 组合分析
// GET /api/analysis/daily - 每日分析摘要
```

调用 AI Gateway 生成分析：

```go
type AnalysisRequest struct {
    Symbol    string   `json:"symbol"`
    Position  *Position `json:"position,omitempty"`
    MarketData *MarketData `json:"marketData"`
}

type AnalysisResponse struct {
    Summary     string   `json:"summary"`
    Risks       []string `json:"risks"`
    Opportunities []string `json:"opportunities"`
    Suggestion  string   `json:"suggestion"` // buy/sell/hold
    Confidence  float64  `json:"confidence"`
    Reasoning   string   `json:"reasoning"`
}
```

#### 3.3 前端：分析面板组件

**新文件**：`/Volumes/ExSSD/Projects/zoomlab-web/frontend/src/features/brozoomout/components/AnalysisPanel.tsx`

```tsx
export function AnalysisPanel({ analysis }: { analysis: AnalysisResponse }) {
  return (
    <Card className="border-cyan-500/20">
      <CardHeader>
        <CardTitle className="flex items-center gap-2">
          <Brain className="h-5 w-5 text-cyan-400" />
          AI 分析
        </CardTitle>
      </CardHeader>
      <CardContent className="space-y-4">
        <div className="flex items-center gap-3">
          <SuggestionBadge suggestion={analysis.suggestion} />
          <span className="text-sm text-slate-400">
            置信度 {Math.round(analysis.confidence * 100)}%
          </span>
        </div>
        <p className="text-slate-300">{analysis.summary}</p>
        <div className="grid grid-cols-2 gap-4">
          <div>
            <p className="text-sm font-medium text-red-400 mb-2">风险因素</p>
            <ul className="space-y-1">
              {analysis.risks.map((risk, i) => (
                <li key={i} className="text-sm text-slate-400 flex items-start gap-2">
                  <AlertTriangle className="h-4 w-4 text-red-400 mt-0.5 flex-shrink-0" />
                  {risk}
                </li>
              ))}
            </ul>
          </div>
          <div>
            <p className="text-sm font-medium text-green-400 mb-2">机会信号</p>
            <ul className="space-y-1">
              {analysis.opportunities.map((opp, i) => (
                <li key={i} className="text-sm text-slate-400 flex items-start gap-2">
                  <TrendingUp className="h-4 w-4 text-green-400 mt-0.5 flex-shrink-0" />
                  {opp}
                </li>
              ))}
            </ul>
          </div>
        </div>
        <div className="pt-4 border-t border-white/10">
          <p className="text-sm text-slate-400 italic">{analysis.reasoning}</p>
        </div>
      </CardContent>
    </Card>
  );
}
```

#### 3.4 前端：集成到 BroZoomOutPage

**文件**：`/Volumes/ExSSD/Projects/zoomlab-web/frontend/src/pages/BroZoomOutPage.tsx`

在现有组件之间插入新组件：

```tsx
return (
  <div className="rounded-3xl bg-white text-slate-900 dark:bg-slate-950 dark:text-slate-50">
    <div className="mx-auto max-w-6xl px-4 py-8 sm:px-6 md:py-12 md:px-10 lg:px-12">
      <BroZoomOutHero summary={data.summary} />
      <MarketOverview data={data.marketOverview} />  {/* 新增 */}
      <PortfolioTracker />  {/* 新增 */}
      <OverviewCards data={data} />
      <AiRatioMonitor ratio={data.summary.aiRatio} threshold={30} />
      <TrendChart trend={data.trend} />
      <CategoryChart categories={data.categories} />
      <RiskOpportunityMatrix items={data.items} />
      <SignalFeed items={data.items} categories={data.categories} />
      <AnalysisPanel analysis={data.analysis} />  {/* 新增 */}
      <InsightsSection data={data} />
      <ActionBoard data={data} />
      <CognitionTimeline items={data.cognitionTimeline || []} />
      <ReplayPanel timeline={data.cognitionTimeline || []} />
      <Watchlist watchlist={data.watchlist} />
      <ArchiveTimeline archives={data.archives} />
    </div>
  </div>
);
```

**验收标准**：
- [ ] 可添加/删除持仓标的
- [ ] 持仓汇总显示总市值和盈亏
- [ ] 单标的分析显示 AI 生成的买入/卖出建议
- [ ] 分析包含风险和机会因素
- [ ] 组合分析显示行业分布

---

### Phase 4：交互功能与数据持久化（预计 4-6 小时）

**目标**：添加收藏、笔记、提醒等交互功能

#### 4.1 后端：用户交互 API

**新文件**：`/Volumes/ExSSD/Projects/zoomlab-web/backend/internal/userdata/handler.go`

```go
// POST /api/userdata/favorites - 添加收藏
// GET /api/userdata/favorites - 获取收藏列表
// DELETE /api/userdata/favorites/:id - 删除收藏

// POST /api/userdata/notes - 添加笔记
// GET /api/userdata/notes?signalId=xxx - 获取信号笔记
// PUT /api/userdata/notes/:id - 更新笔记

// POST /api/userdata/reminders - 添加提醒
// GET /api/userdata/reminders - 获取提醒列表
// DELETE /api/userdata/reminders/:id - 删除提醒
```

**新文件**：`/Volumes/ExSSD/Projects/zoomlab-web/backend/internal/userdata/models.go`

```go
type Favorite struct {
    ID        string    `json:"id"`
    UserID    string    `json:"userId"`
    Type      string    `json:"type"` // signal, stock, etf
    RefID     string    `json:"refId"`
    Notes     string    `json:"notes,omitempty"`
    CreatedAt time.Time `json:"createdAt"`
}

type UserNote struct {
    ID        string    `json:"id"`
    UserID    string    `json:"userId"`
    SignalID  string    `json:"signalId"`
    Content   string    `json:"content"`
    CreatedAt time.Time `json:"createdAt"`
    UpdatedAt time.Time `json:"updatedAt"`
}

type Reminder struct {
    ID        string    `json:"id"`
    UserID    string    `json:"userId"`
    Title     string    `json:"title"`
    Symbol    string    `json:"symbol,omitempty"`
    TriggerAt time.Time `json:"triggerAt"`
    Message   string    `json:"message"`
    Read      bool      `json:"read"`
}
```

#### 4.2 前端：信号卡片交互

**文件**：`/Volumes/ExSSD/Projects/zoomlab-web/frontend/src/features/brozoomout/components/SignalFeed.tsx`

在每个信号卡片添加操作按钮：

```tsx
<div className="flex items-center gap-2 mt-3">
  <Button
    variant="ghost"
    size="sm"
    onClick={(e) => { e.stopPropagation(); toggleFavorite(item.id); }}
  >
    {isFavorite(item.id) ? (
      <Star className="h-4 w-4 text-yellow-400 fill-yellow-400" />
    ) : (
      <Star className="h-4 w-4 text-slate-400" />
    )}
  </Button>
  <Button
    variant="ghost"
    size="sm"
    onClick={(e) => { e.stopPropagation(); openNoteDialog(item.id); }}
  >
    <MessageSquare className="h-4 w-4 text-slate-400" />
  </Button>
  <Button
    variant="ghost"
    size="sm"
    onClick={(e) => { e.stopPropagation(); setReminder(item); }}
  >
    <Bell className="h-4 w-4 text-slate-400" />
  </Button>
</div>
```

#### 4.3 前端：笔记对话框

**新文件**：`/Volumes/ExSSD/Projects/zoomlab-web/frontend/src/features/brozoomout/components/NoteDialog.tsx`

```tsx
export function NoteDialog({ signalId, onClose }: NoteDialogProps) {
  const [note, setNote] = useState('');
  
  return (
    <Dialog open onOpenChange={onClose}>
      <DialogContent>
        <DialogHeader>
          <DialogTitle>添加笔记</DialogTitle>
        </DialogHeader>
        <Textarea
          placeholder="记录你对这条信号的思考..."
          value={note}
          onChange={(e) => setNote(e.target.value)}
        />
        <DialogFooter>
          <Button variant="ghost" onClick={onClose}>取消</Button>
          <Button onClick={() => saveNote(signalId, note)}>保存</Button>
        </DialogFooter>
      </DialogContent>
    </Dialog>
  );
}
```

**验收标准**：
- [ ] 可收藏/取消收藏信号
- [ ] 可为信号添加文字笔记
- [ ] 可设置提醒（本地通知）
- [ ] 收藏列表在侧边栏可查看

---

### Phase 5：高级分析与可视化（预计 6-8 小时）

**目标**：添加高级图表和分析功能

#### 5.1 前端：持仓盈亏图表

**新文件**：`/Volumes/ExSSD/Projects/zoomlab-web/frontend/src/features/brozoomout/components/PnLChart.tsx`

使用 Recharts 或 Nivo 绘制：
- 持仓盈亏饼图（按标的）
- 行业分布饼图
- 盈亏趋势折线图

```tsx
import { PieChart, Pie, Cell, ResponsiveContainer, Tooltip } from 'recharts';

export function PnLChart({ positions }: { positions: PositionDetail[] }) {
  const data = positions.map(p => ({
    name: p.symbol,
    value: Math.abs(p.currentValue),
    pnl: p.pnl,
  }));

  return (
    <ResponsiveContainer width="100%" height={300}>
      <PieChart>
        <Pie data={data} dataKey="value" nameKey="name" cx="50%" cy="50%">
          {data.map((entry, index) => (
            <Cell key={index} fill={entry.pnl >= 0 ? '#10b981' : '#ef4444'} />
          ))}
        </Pie>
        <Tooltip />
      </PieChart>
    </ResponsiveContainer>
  );
}
```

#### 5.2 前端：风险热力图

**新文件**：`/Volumes/ExSSD/Projects/zoomlab-web/frontend/src/features/brozoomout/components/RiskHeatmap.tsx`

显示持仓的风险分布：
- 波动率热力图
- 相关性矩阵
- Beta 值分布

#### 5.3 后端：历史回测 API

**新文件**：`/Volumes/ExSSD/Projects/zoomlab-web/backend/internal/backtest/handler.go`

```go
// POST /api/backtest/run - 运行回测
// GET /api/backtest/results/:id - 获取回测结果
```

支持简单策略回测：
- 均线策略
- 动量策略
- 价值投资策略

**验收标准**：
- [ ] 持仓盈亏饼图正确显示
- [ ] 行业分布饼图正确显示
- [ ] 风险热力图显示波动率分布
- [ ] 可运行简单回测并查看结果

---

### Phase 6：性能优化与部署（预计 2-3 小时）

**目标**：优化性能，准备生产部署

#### 6.1 前端性能优化

- 实现数据懒加载（虚拟滚动）
- 添加 React.memo 优化重渲染
- 实现 Service Worker 缓存

#### 6.2 后端性能优化

- 添加 Redis 缓存（市场数据、AI 分析结果）
- 实现请求限流
- 添加健康检查端点

#### 6.3 Docker 配置

**文件**：`/Volumes/ExSSD/Projects/zoomlab-web/docker-compose.yml`

添加 Redis 服务：

```yaml
redis:
  image: redis:7-alpine
  restart: unless-stopped
  networks:
    - zoomlab-net
  volumes:
    - redis-data:/data
```

**验收标准**：
- [ ] 页面加载时间 < 2 秒
- [ ] 数据刷新无卡顿
- [ ] Docker 部署成功
- [ ] 健康检查端点正常

---

## 代码规范

### 前端规范

1. **组件结构**：每个组件一个文件，使用 React FC 类型
2. **状态管理**：优先使用 React hooks，复杂状态考虑 Zustand
3. **样式**：使用 Tailwind CSS，遵循 DESIGN.md 中的设计系统
4. **类型安全**：所有 props 和 state 必须有 TypeScript 类型
5. **错误处理**：每个 API 调用都要有 try-catch 和 loading 状态

### 后端规范

1. **包结构**：按功能模块组织（market, portfolio, analysis, userdata）
2. **错误处理**：返回结构化错误响应
3. **日志**：使用 log 包，关键操作记录日志
4. **配置**：通过环境变量配置，不硬编码

### Git 规范

1. **提交信息**：`type(scope): description`
   - feat: 新功能
   - fix: 修复
   - refactor: 重构
   - docs: 文档
   - style: 样式
2. **分支**：`feature/xxx`, `fix/xxx`
3. **PR**：每个功能一个 PR，描述清晰

---

## 注意事项

1. **不要破坏现有功能**：每次修改后确保现有功能正常
2. **渐进式实现**：按 Phase 顺序实施，每个 Phase 完成后测试
3. **数据安全**：用户持仓数据存储在 SQLite，不要泄露
4. **API 限流**：外部 API 调用要有限流，避免被封
5. **中文优先**：界面文字使用中文，代码注释使用英文

---

## 快速开始

1. 先读取项目文档：
   - `/Volumes/ExSSD/Projects/zoomlab-web/CLAUDE.md`
   - `/Volumes/ExSSD/Projects/zoomlab-web/DESIGN.md`
   - `/Volumes/ExSSD/Projects/zoomlab-web/.mini-spec-kit/project-constraints.md`

2. 从 Phase 1 开始，逐步实施

3. 每个 Phase 完成后运行测试：
   ```bash
   cd /Volumes/ExSSD/Projects/zoomlab-web/backend && go build ./...
   cd /Volumes/ExSSD/Projects/zoomlab-web/frontend && npm run build
   ```

4. 启动开发服务器测试：
   ```bash
   docker compose up -d
   ```

5. 访问 https://stg.zoomlab.cc/brozoomout 查看效果

---

*最后更新：2026-05-27*
