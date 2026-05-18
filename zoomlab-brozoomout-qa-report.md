# ZoomLab STG 环境全量 QA 报告

**测试日期**: 2026-05-18
**测试环境**: STG (localhost:5174, localhost:8081)
**测试方式**: 源码静态分析 + Docker 日志分析 + API 端点测试 + 浏览器快照
**测试分支**: `origin/STG` (cf877ea)

> ⚠️ 当前 STG 环境需要 Google OAuth 登录才能访问前端页面，无法进行交互式浏览器测试。所有发现基于源码分析、Docker 日志和 API 测试。

---

## 测试结果总结

| 级别 | BroZoomOut 模块 | 其他模块 | 基础设施 |
|------|----------------|----------|---------|
| **Critical** | 1 | 0 | 0 |
| **Major** | 5 | 3 | 0 |
| **Minor** | 6 | 8 | 0 |
| **Cosmetic** | 1 | 1 | 0 |

**Bug 总数**: 25

---

## 🔴 Critical

### BZ-009: RiskOpportunityMatrix 静默丢弃 60%+ 信号数据

- **设备**: Both (PC + Mobile)
- **类型**: Logic / Data
- **文件**: `frontend/src/features/brozoomout/components/RiskOpportunityMatrix.tsx:11-25`
- **文件**: `frontend/src/features/brozoomout/utils/scoring.ts:23-33`

**根因**: `matrix` 对象只定义了 4 个 key (`high-high`, `high-low`, `low-high`, `low-low`)，但 `getRiskLevel()` 和 `getOpportunityLevel()` 返回值包含 `"medium"`。当 risk 或 opportunity 为 `"medium"` 时，`matrix[key]` 为 `undefined`，第 22 行 `if (matrix[key])` 条件为 false，该信号被**静默丢弃**。

**影响**: mock 数据 23 条信号中 14 条（60.9%）被丢弃。所有 `importance="medium"` 的信号均受影响。

**修复**:
- 方案 A：将 `matrix` 扩展为 9 种组合（3×3 网格）
- 方案 B：将 scoring 函数改为只返回 `"high" | "low"`

---

## 🟠 Major

### BZ-010: Watchlist 删除按钮无操作

- **设备**: Both
- **类型**: UX
- **文件**: `frontend/src/features/brozoomout/components/Watchlist.tsx:41-43`

删除按钮 (`<X />` 图标) hover 时可见，但**没有任何 onClick 处理函数**，完全是占位符。

---

### BZ-011: ArchiveTimeline 行点击无导航

- **设备**: Both
- **类型**: UX
- **文件**: `frontend/src/features/brozoomout/components/ArchiveTimeline.tsx:24`

每行 div 设置了 `cursor-pointer` 和 `hover:bg-[...]`，右侧有 `ChevronRight` 图标，视觉上完全像可点击。但无 `onClick`，API `fetchBroZoomOutArchive` 已实现但从未被调用。

---

### BZ-012: ActionBoard tierConfig 颜色解析脆弱

- **设备**: Both
- **类型**: CSS / Maintainability
- **文件**: `frontend/src/features/brozoomout/components/ActionBoard.tsx:51`

`config.color.split(" ")[0]` 取第一个 class 作为背景色。若 tailwind 配置或类名顺序变化（如添加 `dark:` 变体），可能取到错误的值，背景色完全丢失。

**修复**: 使用独立 `bgClass` 字段或 Tailwind `bg-${tier}-50` 模式。

---

### BZ-013: SignalCard 使用未确认的 CSS 动画类

- **设备**: Both
- **类型**: CSS
- **文件**: `frontend/src/features/brozoomout/components/SignalCard.tsx:53`

`animate-in` 和 `fade-in` 来自 `tailwindcss-animate` 插件，未确认该插件在项目 Tailwind 配置中是否启用。若无，展开动画静默失效。

---

### BZ-014: TrendChart 忽略 china/ai 趋势数据

- **设备**: Both
- **类型**: Data / UI
- **文件**: `frontend/src/features/brozoomout/components/TrendChart.tsx`

`BroZoomOutTrend` 类型定义了 `china` 和 `ai` 字段，mock 数据也提供了这些值，但 TrendChart **只渲染 `total` 和 `high` 两条折线**。

---

### BZ-001 (重构): BroZoomOut 页面因登录墙不可访问

- **设备**: Both
- **类型**: Deploy / Auth
- **严重性**: **Major** (降级，因这是 STG 环境预期行为而非 Bug)

STG 前端已具备 `?test=1` 测试登录按钮和 `VITE_STG_MODE` mock 数据路径，但**后端没有 `/api/test/login` 端点**。该端点在之前被删除的本地 commit 中，重置后丢失。

**影响**: 无法通过浏览器直接访问任何页面内容（所有路由均需 AuthGuard）。

**修复**: 在 Go 后端添加 `/api/test/login` 端点返回模拟 session，或在 STG 构建时注入 `VITE_STG_MODE=true` 使 BroZoomOut 使用 mock 数据。

---

## 🟡 Minor

### BZ-015: AiRatioMonitor 阈值硬编码

- **文件**: `AiRatioMonitor.tsx:10`
- `const isHealthy = ratio <= 30;` — 30% 硬编码。建议通过 props 传递。

### BZ-016: SignalFeed 分类 Tab 硬编码

- **文件**: `SignalFeed.tsx:12`
- 分类列表完全硬编码。建议从 `data.categories` 动态生成。

### BZ-017: ArchiveTimeline 日期排序依赖 mock 顺序

- **文件**: `ArchiveTimeline.tsx`
- 生产 API 可能返回任意顺序。需 `[...archives].sort((a,b) => b.date.localeCompare(a.date))`。

### BZ-018: scoring.ts 中两个函数为死代码

- **文件**: `utils/scoring.ts:3-21`
- `calculateImpactScore` 和 `calculateActionValue` export 但未被任何组件使用。

### BZ-019: mockAdapter.ts 浅拷贝共享引用

- **文件**: `api/mockAdapter.ts:13-19`
- `{...mockBroZoomOutData}` 浅拷贝，数组指向同一引用。建议 `JSON.parse(JSON.stringify(...))`。

### BZ-020: SignalFeed 搜索输入框缺无障碍标签

- **文件**: `SignalFeed.tsx:44`
- `<input>` 缺少 `aria-label`。添加 `aria-label="搜索信号"`。

### B1: timeAgo 函数缺少日期无效保护

- **文件**: `NewsPage.tsx:41-48`, `NewsDetailPage.tsx:22-29`
- `new Date(isoString).getTime()` 在 isoString 无效时返回 NaN，显示 `"NaN天前"`。
- `RightSidebar.tsx:9-17` 已有正确保护，需统一覆盖。

### B2: NewsPage 桌面端多列布局缺少 break-inside-avoid

- **文件**: `NewsPage.tsx:~476`
- `columns-2 xl:columns-3` 布局中 HeadlineCard 未设置 `break-inside-avoid`，卡片可能被跨列截断。

### B3: 昨日模式 + 视口切换内容空白

- **文件**: `NewsPage.tsx:702-708`
- 移动端切"昨日"后切换到桌面视图，桌面切换钮保持 `today`，导致无内容显示。
- **修复**: `handleDateToggle` 中同步 `activeTab`。

### B4: NewsDetailModal displaySummary 重复引用

- **文件**: `NewsDetailModal.tsx:~L89`
- `detail?.summary_zh || detail?.summary_zh || news.summary || ""` — `summary_zh` 重复检查两次，`news.summary` 无法作为后备。

### B5: SettingsPage 未使用的 import

- **文件**: `SettingsPage.tsx:~L11`
- `import { useState as useToggleState }` 从未使用。

### B6: NewsDetailPage 付费墙警告缺深色模式

- **文件**: `NewsDetailPage.tsx:121-125`
- 固定 `bg-amber-50 text-amber-800`，无 `dark:` 变体。`NewsDetailModal.tsx` 已正确处理。

### B7: LabPage "favorites" Tab 触发无效 fetch

- **文件**: `LabPage.tsx:264-266`
- `fetchData` switch 未处理 `'favorites'` 分支，`loading` 设为 true 后永不重置。

### B8: Notes/全局类型定义重复

- **文件**: `types.ts:23-44` vs `features/notes/types.ts:4-19,33-45`
- 两套独立的 `Note`/`Todo` 接口，字段不同，混用可能产生类型错误。

---

## 🟢 Cosmetic

### BZ-021: 加载骨架屏与实际布局差异大

- **文件**: `BroZoomOutPage.tsx:52-63`
- 骨架屏只有 hero + 4 卡片 + 1 长条，真实页面有 10+ 区块。跳变感强。

### B9: Frontend JS bundle 过大

- **文件**: 构建产物
- `index-DpybTD88.js` = 797.84KB（超过 500KB 建议 code-splitting）

---

## 基础设施 / 日志发现

| 问题 | 影响 | 说明 |
|------|------|------|
| Twitter 头像下载失败 (exit status 8) | 38 个账户 | 外部 API 限制，非关键 |
| Twitter 时间线 JSON-LD 解析失败 | 大量 | X 页面结构变更，需更新 scraper |
| AI 新闻 JSON 解析偶尔失败 | 科技/要闻 | 使用 raw fallback 降级，内容仍展示 |
| RSS 部分源超时 (虎嗅, Guardian EPL) | 该源缺失 | 其他源正常 |
| GIN 运行在 debug 模式 | 建议修复 | 生产环境推荐设为 release |
| 后端无 `/api/test/login` | 无法登录测试 | 需在 STG 分支添加 |
| 前端无 `VITE_STG_MODE=true` 构建 | 无法用 mock 数据 | 需在 STG 构建流程配置 |

---

## 修复优先级建议

| 优先级 | Bug | 原因 |
|--------|-----|------|
| **P0** | BZ-009 | 60%+ 信号数据被丢弃，产品数据失真 |
| **P1** | BZ-010/011 | 交互中断，用户操作无反馈 |
| **P1** | BZ-014 | 数据不完整，误导分析 |
| **P1** | BZ-001 (auth) | 当前无法通过浏览器完整测试 |
| **P2** | BZ-012/013 | 视觉可用性隐患 |
| **P2** | B1/B2/B3 | 影响新闻页核心使用体验 |
| **P3** | 其余 Minor/Cosmetic | 抛光性问题 |
