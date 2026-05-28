# 任务：生成金融系统升级文档

## 目标
为 BroZoomOut 和 ZoomLab 两个项目生成 3 份详尽的技术文档，目标是达到同花顺/东方财富 70-80% 的功能覆盖度。

## 步骤

### 第一步：读取项目代码理解现状

**BroZoomOut 项目** (`/Volumes/ExSSD/Projects/Bro,Zoom Out/finance-intelligence-service/`):
- 读取 `app/` 目录结构
- 读取 `app/api/v1/` 下所有 API 端点
- 读取 `app/services/` 下所有服务
- 读取 `app/data/providers/` 下所有数据源
- 读取 `app/jobs/` 下所有调度任务
- 读取 `requirements.txt` 了解依赖
- 读取 `Dockerfile` 了解部署方式

**ZoomLab 项目** (`/Volumes/ExSSD/Projects/zoomlab-web/`):
- 读取 `frontend/src/` 目录结构
- 读取 `frontend/src/features/brozoomout/components/` 下所有组件
- 读取 `frontend/src/pages/` 下所有页面
- 读取 `backend/internal/` 下所有模块
- 读取 `DESIGN.md` 了解设计规范
- 读取 `frontend/package.json` 了解前端依赖
- 读取 `docker-compose.yml` 了解部署架构

### 第二步：生成 3 份文档

#### 文档 1: `shared-api-spec.md` — 共同 API 约定

定义两个项目共用的所有 API 端点规范，包括：

1. **市场数据 API**
   - 指数行情（腾讯 `qt.gtimg.cn`）
   - 个股行情（新浪 `hq.sinajs.cn`）
   - K 线数据（yfinance）
   - 板块动向（新浪 `newFLJK`，163 个概念板块）

2. **资金流向 API**
   - 个股资金流向（东财 `push2his`，需缓存）
   - 北向资金（东财 `kamt.rtmin`，241 数据点）
   - 主力资金流向

3. **研报与新闻 API**
   - 研报摘要（东财 `reportapi`）
   - 新闻聚合（RSS: Google/CNBC/FT）
   - 经济日历（Fair Economy）

4. **AI 分析 API**
   - AI 预测（LLM + 历史数据）
   - AI 基金推荐（yfinance 净值 + LLM 策略分析）
   - AI 个股分析

5. **用户数据 API**
   - 自选股管理
   - 持仓管理
   - 收藏与笔记

每个 API 需要包含：
- 请求/响应格式（JSON Schema）
- 数据源说明
- 缓存策略
- 错误处理
- 限流策略

#### 文档 2: `brozoomout-upgrade-plan.md` — BroZoomOut 升级计划

**目标**: 将 BroZoomOut 从信息聚合页面升级为专业金融分析仪表盘

**功能模块**（按优先级）:

**P1 核心面板**:
- 板块热力图（163 个概念板块，涨跌幅颜色编码）
- 新闻分级（AI 分级：major/notable/general，利好利空标注）
- 个股推荐（量化筛选 + AI 评估）
- 研报摘要（AI 一句话总结）

**P2 AI 智能**:
- AI 预测（三场景预测：牛市/基准/熊市）
- AI 基金推荐（夏普比率、最大回撤、风格漂移检测）
- Dashboard 增强（置信度趋势、风险模式变化）

**P3 交互体验**:
- 历史趋势数据扩充（30 天快照）
- 调度任务扩展（板块扫描、新闻聚合、研报摘要）

**P4 深度数据**:
- K 线图（日/周/月 K，支持 MA/MACD/RSI）
- 技术指标计算（纯 Python）
- 资金流向分析（主力/散户/北向）
- 个股详情页（行情+财务+估值+研报）
- 宏观数据（CPI/PMI/GDP）

每个模块需要包含：
- 功能描述
- API 端点设计
- 数据流图
- 前端组件设计
- 后端服务设计
- 调度任务设计
- 验收标准

#### 文档 3: `zoomlab-upgrade-plan.md` — ZoomLab 升级计划

**目标**: 将 ZoomLab 前端升级为专业金融终端界面

**组件模块**:

**P1 核心组件**:
- MarketHeatmap（市场热力图，行业/概念板块）
- SectorMovers（板块动向，涨跌排行）
- NewsFeed（新闻流，AI 分级标注）
- StockPicks（个股推荐卡片）
- ResearchDigest（研报精选）

**P2 AI 组件**:
- AIForecast（AI 预测面板，三场景可视化）
- FundPicks（基金推荐，AI 策略分析）
- RiskMatrix（风险矩阵，热力图）

**P3 交互组件**:
- HistoryTrend（历史趋势图，30 天快照）
- 响应式移动端适配
- 交互增强（联动/动画/空状态）

**设计规范**:
- 必须遵循 `DESIGN.md` 的设计系统
- 支持日间/夜间模式切换
- 颜色编码：涨红跌绿（中国标准）
- 字体：中文使用系统字体，英文使用 Inter
- 间距：遵循 4px 基准网格

每个组件需要包含：
- 组件 props 设计
- 状态管理
- 样式规范（日间/夜间）
- 交互逻辑
- 响应式断点
- 验收标准

### 第三步：上传到 GitHub

使用 `gh` CLI 上传到 `zoomc/prompt-hub` 仓库：
```bash
cd /Volumes/ExSSD/Projects/prompt-hub
# 创建文件
# git add -A
# git commit -m "docs: 生成金融系统升级文档 v1.0"
# git push origin main
```

## 要求

1. **详尽性**: 每个模块都要有详细的技术实现方案，不要笼统描述
2. **专业性**: 对标同花顺/东方财富 70-80% 功能覆盖度
3. **可执行性**: 计划要可直接指导开发
4. **设计一致性**: ZoomLab 必须遵循 DESIGN.md
5. **AI 整合**: 所有模块都要考虑 AI 分析/推荐能力
6. **双模式**: 支持日间/夜间模式
7. **性能**: 考虑缓存、限流、降级策略

## 输出

3 个 Markdown 文件：
1. `shared-api-spec.md`
2. `brozoomout-upgrade-plan.md`
3. `zoomlab-upgrade-plan.md`
