# BroZoomOut 市场分析模块 QA 报告

**测试日期**: 2026-05-18
**测试环境**: STG (localhost:5174)
**测试方式**: 浏览器实时测试 + 源码静态分析
**测试设备**: PC (Chrome CDP) + 源码分析

---

## 测试结果总结

| 级别 | 数量 |
|------|------|
| **Bug 总数** | **9** |
| **Critical** | 2 |
| **Major** | 2 |
| **Minor** | 3 |
| **Cosmetic** | 2 |

---

## Bug 详情

### 🔴 BZ-001: STG 环境未部署 BroZoomOut — 运行的是旧版 ZoomLab

- **设备**: Both (PC + Mobile)
- **严重性**: **Critical**
- **类型**: Deploy / Build

**浏览器实测确认**：
- http://localhost:5174/ → 显示"ZoomLab"登录页，仅"使用 Google 账号登录"按钮
- 所有路径 `/market`、`/brozoomout`、`/markets`、`/analysis`、`/market-analysis` 均返回相同的登录页面
- Console 无错误（SPA fallback 正常，但内容不对）

**根因**：`Dockerfile.frontend` 从 `frontend/` 目录构建旧版 ZoomLab。BroZoomOut v2 源码在仓库根目录 `src/`（STG 分支 commit `43b5ba0`），但 Docker 构建从未引用。

**修复方向**：
- 方案 A：修改 `Dockerfile.frontend` 指向根目录的 `src/` + `index.html`
- 方案 B：新建 `Dockerfile.brozoomout` 并在 STG compose override 中切换
- 修复后必须 `docker compose -f docker-compose.yml -f docker-compose.override.stg.yml up -d --build frontend`

---

### 🔴 BZ-002: SignalFeed 分类标签文字几乎不可见

- **设备**: Both
- **严重性**: **Critical** (核心 UI 元素不可用)
- **类型**: Layout / CSS
- **文件**: `src/features/brozoomout/components/SignalFeed.jsx:77`

```jsx
// 问题代码
style={{
  backgroundColor: data.categories.find(c => c.name === item.category)?.color,
  opacity: 0.2,  // ← 整个 span 20% 透明度，文字也跟着透明
  color: ...
}}
```

**预期**: 标签背景半透明，文字清晰可读
**实际**: `opacity: 0.2` 作用于整个 `<span>` 元素，文字只有 20% 不透明度

**修复**：
```jsx
style={{
  backgroundColor: color + "33",  // hex alpha ~20%
  color: color
}}
// 移除 opacity: 0.2
```

---

### 🟠 BZ-003: TrendChart 缺少"国内(China)"数据系列

- **设备**: Both
- **严重性**: **Major**
- **类型**: Logic / Data
- **文件**: `src/features/brozoomout/components/TrendChart.jsx:24-34`

**数据**包含 `total`、`high`、`ai`、`china` 四个字段，图表只渲染前三个。图例缺少"国内(China)"系列。

**修复方向**：增加第 4 组柱体 + 图例项，或从 mock 数据中移除 `china` 字段。

---

### 🟠 BZ-004: TrendChart 柱体重叠而非分组/堆叠

- **设备**: Both
- **严重性**: **Major**
- **类型**: Layout
- **文件**: `src/features/brozoomout/components/TrendChart.jsx`

三个 `<rect>` 的 `x` 和 `width` 完全相同（30px），视觉上完全重叠。仅最后绘制的系列可见。

**修复方向**：分组柱状图（width=10px, x 偏移 0/10/20）或堆叠柱状图。

---

### 🟡 BZ-005: 行动看板"开始"按钮无响应

- **设备**: Both
- **严重性**: Minor
- **类型**: UX
- **文件**: `src/features/brozoomout/components/ActionBoard.jsx:35`

`<button>开始</button>` 没有任何 `onClick` 处理器，纯占位符。

---

### 🟡 BZ-006: SignalFeed 重要性筛选按钮显示英文

- **设备**: Both
- **严重性**: Minor
- **类型**: UX
- **文件**: `src/features/brozoomout/components/SignalFeed.jsx:53-62`

按钮文字为 `"high" / "medium" / "low"`，应改为中文 `高/中/低`。

---

### 🟢 BZ-007: OverviewCards 归档数量硬编码 +1

- **设备**: Both
- **严重性**: Cosmetic
- **类型**: Logic
- **文件**: `src/features/brozoomout/components/OverviewCards.jsx:28`

```jsx
value: data.archives.length + 1  // ← 为什么+1？
```

空数组时显示 1，可能重复计数。

---

### 🟢 BZ-008: Hero AI 占比硬编码 30%

- **设备**: Both
- **严重性**: Cosmetic
- **类型**: Logic
- **文件**: `src/features/brozoomout/components/BroZoomOutHero.jsx:18`

徽章显示固定值 `30%`，应动态计算 `Math.round(data.items.filter(i => i.category === "AI").length / data.items.length * 100) + "%"`。

---

## 修复优先级建议

| 优先级 | Bug | 原因 |
|--------|-----|------|
| **P0** | BZ-001 | 模块完全不可访问，必须先修复才能测试其他 |
| **P0** | BZ-002 | 核心 UI 元素在视觉上不可用 |
| **P1** | BZ-003 | 数据与展示不一致，误导用户 |
| **P1** | BZ-004 | 图表视觉语义错误 |
| **P2** | BZ-005 | 交互中断，降低完成度 |
| **P2** | BZ-006 | 国际化不一致 |
| **P3** | BZ-007 | 计数偏差，影响不大 |
| **P3** | BZ-008 | 静态 mock 下不显著 |

---

## 建议的线上 Agent 修复指令

### 修复顺序
1. **先修 BZ-001**：改 `Dockerfile.frontend` 或加新的 Dockerfile，确保 BroZoomOut 能构建部署
2. **再修 BZ-002**：SignalFeed 标签透明度
3. **再修 BZ-003 + BZ-004**：TrendChart 图表问题
4. **最后修 BZ-005 ~ BZ-008**：抛光性问题

### 文件修改汇总

| 文件 | 改动 |
|------|------|
| `Dockerfile.frontend` | 构建源切换到根目录 `src/` + `index.html` |
| `src/features/brozoomout/components/SignalFeed.jsx:77` | 移除 `opacity:0.2`，用 hex alpha |
| `src/features/brozoomout/components/SignalFeed.jsx:61` | `importances` 改为中文 |
| `src/features/brozoomout/components/TrendChart.jsx` | 增加 china 系列，修复 rect 重叠 |
| `src/features/brozoomout/components/ActionBoard.jsx:35` | 加 `onClick` 回调 |
| `src/features/brozoomout/components/OverviewCards.jsx:28` | 移除 `+ 1` |
| `src/features/brozoomout/components/BroZoomOutHero.jsx:18` | 动态计算 AI 占比 |
