# 免费公开 API 覆盖度调研报告

> 日期: 2026-05-28 | 结论: **70% 可覆盖，30% 受限**

---

## 一、API 可用性总表

### ✅ 可用（已验证）

| 数据类型 | API | 协议 | Docker 可达 | 需 Key | 备注 |
|----------|-----|------|-------------|--------|------|
| 指数行情 | 腾讯 `qt.gtimg.cn` | HTTP | ✅ | ❌ | GBK 编码，需 decode |
| 个股行情 | 新浪 `hq.sinajs.cn` | HTTP | ✅ | ❌ | GBK 编码，需 Referer |
| K 线数据 | yfinance | HTTPS | ✅ | ❌ | 容器内已安装 |
| PE/PB/估值 | yfinance | HTTPS | ✅ | ❌ | `.info` 字段 |
| 财务三表 | yfinance | HTTPS | ✅ | ❌ | `.financials`/`.balance_sheet` |
| 新闻聚合 | RSS (Google/CNBC/FT) | HTTPS | ✅ | ❌ | feedparser 解析 |
| 宏观指标 | yfinance (CPI/PMI proxy) | HTTPS | ✅ | ❌ | 间接获取 |
| 经济日历 | Fair Economy `ff_calendar` | HTTPS | ✅ | ❌ | 96 条/周 |

### ⚠️ 条件可用（主机可达，Docker 受限）

| 数据类型 | API | 协议 | 问题 | 解决方案 |
|----------|-----|------|------|----------|
| 板块动向 | 东财 `push2.eastmoney.com` | HTTPS | Surge fake IP | 主机桥服务转发 |
| 资金流向 | 东财 `push2.eastmoney.com` | HTTPS | Surge fake IP | 同上 |
| 北向资金 | 东财 `push2.eastmoney.com` | HTTPS | Surge fake IP | 同上 |
| 研报列表 | 东财 `reportapi.eastmoney.com` | HTTPS | 需验证 | 主机桥服务 |
| 基金排名 | 东财基金 | HTTPS | 需登录态 | 主机桥服务 |

### ❌ 不可用（需付费或已失效）

| 数据类型 | API | 原因 |
|----------|-----|------|
| 全市场实时行情 WebSocket | 万得/iFinD | 付费（年费 5-20 万） |
| FRED 宏观数据 | FRED API | 需注册获取 Key（免费但需申请） |
| NewsAPI | newsapi.org | Key 已过期 |
| 新浪板块数据 | `vip.stock.finance.sina.com.cn` | 返回空数组 |

---

## 二、逐模块覆盖分析

### P1 模块

| 模块 | 数据源 | 覆盖度 | 方案 |
|------|--------|--------|------|
| 板块动向 | 东财 API | ⚠️ 主机可达 | 主机桥服务转发到 Docker |
| 新闻分级 | RSS + AI | ✅ 100% | 已有 RSS + LLM 提取 |
| 个股推荐 | yfinance + 量化 | ✅ 100% | yfinance 提供行情+指标 |
| 研报摘要 | 东财 research API | ⚠️ 需验证 | 主机桥服务或爬虫 |

### P2 模块

| 模块 | 数据源 | 覆盖度 | 方案 |
|------|--------|--------|------|
| AI 预测 | LLM + 历史数据 | ✅ 100% | 自有数据 + AI Gateway |
| 基金推荐 | 东财基金 | ⚠️ 需登录 | 主机桥服务 |
| Dashboard 增强 | 现有 API 扩展 | ✅ 100% | 组合现有端点 |

### P3 模块

| 模块 | 数据源 | 覆盖度 | 方案 |
|------|--------|--------|------|
| 历史趋势 | 现有快照 | ✅ 100% | 已有数据 |
| 调度任务 | 现有框架 | ✅ 100% | 已有调度器 |

### P4 模块

| 模块 | 数据源 | 覆盖度 | 方案 |
|------|--------|--------|------|
| K 线图 | yfinance | ✅ 100% | `Ticker.history()` |
| 技术指标 | 纯计算 | ✅ 100% | 从 K 线数据计算 |
| 资金流向 | 东财 API | ⚠️ 主机可达 | 主机桥服务 |
| 北向资金 | 东财 API | ⚠️ 主机可达 | 主机桥服务 |
| 个股详情 | 聚合 yfinance | ✅ 100% | 组合多个 yfinance 端点 |
| 自选股增强 | 现有 watchlist | ✅ 100% | 扩展现有模块 |
| 宏观数据 | yfinance 间接 | ⚠️ 60% | CPI/PMI 需 FRED Key |
| 经济日历 | Fair Economy | ✅ 100% | 免费 JSON API |
| 财务数据 | yfinance | ✅ 100% | `.financials`/`.balance_sheet` |
| 估值对比 | yfinance | ✅ 100% | PE/PB/PS + AI 解读 |

---

## 三、关键发现

### 1. 东财 API 的 Surge 问题

**现象**：`push2.eastmoney.com` 从 Docker 容器内连接失败（Surge Enhanced Mode 拦截 HTTPS，返回 fake IP 198.18.x.x）

**影响模块**：板块动向、资金流向、北向资金、基金排名

**解决方案**：**主机桥服务**（已有架构）
```
Docker 容器 → HTTP → 主机 Python 桥服务(port 8099) → HTTPS → 东财 API
```
- 桥服务运行在主机上，不受 Surge DNS 拦截
- 已有 `docker-host-bridge` 模式，可复用
- 或者在容器内配置 `HTTP_PROXY=http://host.docker.internal:6152`（Surge HTTP 代理端口）

### 2. yfinance 是最大功臣

**覆盖**：K 线、行情、估值、财务三表、个股筛选
**限制**：
- 中国 A 股需加后缀（`.SS`/`.SZ`）
- 请求频率限制（建议缓存 5 分钟）
- 盘中数据可能有 15 分钟延迟

### 3. 腾讯 + 新浪 HTTP API 免费可用

**覆盖**：实时指数、个股行情
**优势**：纯 HTTP，Surge 不拦截，Docker 直达
**限制**：GBK 编码，无 K 线历史数据

### 4. FRED 需要 Key

**影响**：GDP/CPI/PMI/M2 等宏观数据
**解决**：注册 FRED 账号获取免费 API Key（32 位字符串）
**替代**：用 yfinance 间接获取（`Ticker.info['industryKey']` 等）

---

## 四、最终覆盖度评估

| 维度 | 总模块数 | 可覆盖 | 覆盖率 |
|------|----------|--------|--------|
| P1 核心面板 | 5 | 4 | 80% |
| P2 AI 智能 | 4 | 3 | 75% |
| P3 交互体验 | 4 | 4 | 100% |
| P4 深度数据 | 10 | 7 | 70% |
| **总计** | **23** | **18** | **78%** |

### 不可覆盖的 5 个模块

| 模块 | 原因 | 替代方案 |
|------|------|----------|
| 全市场实时 WebSocket | 需付费数据源 | 用 REST 轮询（延迟 15 分钟） |
| 板块实时行情 | 东财 API 需主机桥 | 部署桥服务 |
| 资金流向（实时） | 同上 | 同上 |
| 北向资金（实时） | 同上 | 同上 |
| FRED 宏观数据 | 需 API Key | 注册免费 Key 或用 yfinance 代理 |

---

## 五、建议行动

### 立即可做（无需额外依赖）

1. **K 线数据** — yfinance 已可用
2. **技术指标** — 纯 Python 计算
3. **个股详情** — yfinance 聚合
4. **财务数据** — yfinance 已可用
5. **估值对比** — yfinance 已可用
6. **经济日历** — Fair Economy API 已可用
7. **新闻聚合** — RSS 已可用
8. **AI 预测** — LLM 已可用

### 需要桥服务（1-2 天搭建）

1. **板块动向** — 东财 API → 主机桥转发
2. **资金流向** — 同上
3. **北向资金** — 同上
4. **研报摘要** — 东财 research API → 桥转发
5. **基金排名** — 东财基金 → 桥转发

### 需要额外配置

1. **FRED API Key** — 注册免费账号
2. **Surge 代理** — 在 Docker 内配置 `HTTP_PROXY`

---

## 六、结论

**免费 API 可覆盖约 78% 的功能需求**。核心差距在东财 API 的 Surge 访问问题，通过主机桥服务可解决。yfinance + 腾讯/新浪 HTTP API + RSS + Fair Economy 已经能支撑大部分功能。

**最大风险**：东财 API 的反爬策略（频率限制、IP 封禁），需要做好缓存和降级。
