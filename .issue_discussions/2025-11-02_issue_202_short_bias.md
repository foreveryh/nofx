# Issue #202 分析报告：AI 做空偏好问题

**日期**: 2025-11-02  
**Issue**: https://github.com/NoFxAiOS/nofx/issues/202  
**状态**: 🔍 已分析  
**优先级**: 🔴 高  

---

## 问题概述

**用户反馈**：
> "基础策略有很严重的空头喜好倾向，需要修改一部分代码和参数。改完以后判断就正常了。目前盈利"

**用户解决方案**：
> "在engine.go里面修改prompt，初始prompt引导ai做空偏多"

---

## 核心业务逻辑

### AI 决策流程架构

```
📊 获取市场数据 
   ↓
💭 构建 System Prompt（固定规则）
   ↓
📝 构建 User Prompt（动态数据）
   ↓
🤖 调用 AI API
   ↓
🔍 解析 AI 决策
   ↓
✅ 验证决策合理性
   ↓
⚡ 执行交易
```

**核心文件**: `decision/engine.go`

#### 主要函数链：
1. `GetFullDecision()` - 第93行 - 主入口
2. `fetchMarketDataForContext()` - 第121行 - 获取市场数据
3. `buildSystemPrompt()` - 第203行 - 构建固定规则提示词 🔴 **问题源头**
4. `buildUserPrompt()` - 第319行 - 构建动态数据提示词
5. `mcpClient.CallWithMessages()` - 调用 AI API
6. `parseFullDecisionResponse()` - 第420行 - 解析AI响应
7. `validateDecision()` - 第534行 - 验证决策 🔴 **第二问题源**

---

## 问题根源分析

### 问题 1️⃣：System Prompt 措辞偏向做空

**位置**: `decision/engine.go` 第 229-235 行

```go
// === 做空激励 ===  ⚠️ 标题本身就强调"激励"
sb.WriteString("# 📉 做多做空平衡\n\n")
sb.WriteString("**重要**: 下跌趋势做空的利润 = 上涨趋势做多的利润\n\n")
sb.WriteString("- 上涨趋势 → 做多\n")
sb.WriteString("- 下跌趋势 → 做空\n")
sb.WriteString("- 震荡市场 → 观望\n\n")
sb.WriteString("**不要有做多偏见！做空是你的核心工具之一**\n\n")
```

**问题**：
- 标题"**做空激励**"强调程度过高
- 最后一句"**不要有做多偏见！**"逻辑反向，可能暗示做空应该优先考虑
- 措辞不对称，做多被动提及，做空主动强调

---

### 问题 2️⃣：JSON 示例顺序偏好

**位置**: `decision/engine.go` 第 298-301 行

```go
// 示例：做空优先于平多
fmt.Sprintf("  {\\\"symbol\\\": \\\"BTCUSDT\\\", \\\"action\\\": \\\"open_short\\\", \\\"leverage\\\": %d, ..."),
// 然后才是平多
sb.WriteString("  {\\\"symbol\\\": \\\"ETHUSDT\\\", \\\"action\\\": \\\"close_long\\\", ...\")")
```

**问题**：
- `open_short` 优先出现在示例中
- `close_long` 出现在后面
- 可能强化 AI 对做空的优先级认知

---

### 问题 3️⃣：风险回报比计算不对称 🔴 **最关键**

**位置**: `decision/engine.go` 第 534-622 行

#### 入场价计算差异

```go
if d.Action == "open_long" {
    // 做多：入场价在止损和止盈之间的20%位置
    entryPrice = d.StopLoss + (d.TakeProfit - d.StopLoss) * 0.2
} else {
    // 做空：入场价在止损和止盈之间的20%位置（反向）
    entryPrice = d.StopLoss - (d.StopLoss - d.TakeProfit) * 0.2
}
```

#### 风险回报比计算（做多 vs 做空）

**做多时**（第601-606行）：
```go
riskPercent = (entryPrice - d.StopLoss) / entryPrice * 100
rewardPercent = (d.TakeProfit - entryPrice) / entryPrice * 100
if riskPercent > 0 {
    riskRewardRatio = rewardPercent / riskPercent
}
```

**做空时**（第608-613行）：
```go
riskPercent = (d.StopLoss - entryPrice) / entryPrice * 100
rewardPercent = (entryPrice - d.TakeProfit) / entryPrice * 100
if riskPercent > 0 {
    riskRewardRatio = rewardPercent / riskPercent
}
```

#### 具体验证示例

假设参数：
- 做多: `StopLoss = 80, TakeProfit = 100`
- 做空: `StopLoss = 100, TakeProfit = 80`（对称场景）

**做多情况计算**：
```
entryPrice = 80 + (100 - 80) × 0.2 = 84
riskPercent = (84 - 80) / 84 × 100 = 4.76%
rewardPercent = (100 - 84) / 84 × 100 = 19.05%
riskRewardRatio = 19.05 / 4.76 = 4.0 ✅ 通过（需 ≥ 3.0）
```

**做空情况计算**：
```
entryPrice = 100 - (100 - 80) × 0.2 = 96
riskPercent = (100 - 96) / 96 × 100 = 4.17%
rewardPercent = (96 - 80) / 96 × 100 = 16.67%
riskRewardRatio = 16.67 / 4.17 = 4.0 ✅ 通过（需 ≥ 3.0）
```

**隐藏问题**：
- 虽然比例相同，但**现实中价格向下波动时通常幅度更大**
- AI 可能倾向于设置更大的做空止盈范围，更容易满足风险回报比要求
- 市场心理：熊市恐慌性下跌 > 牛市上涨幅度，导致做空机会更"容易"满足条件

---

## 硬约束条件

**位置**: `decision/engine.go` 第 222-227 行

```go
// 1. 风险回报比: 必须 ≥ 1:3（冒1%风险，赚3%+收益）
// 2. 最多持仓: 3个币种（质量>数量）
// 3. 单币仓位: 山寨%.0f-%.0f U(%dx杠杆) | BTC/ETH %.0f-%.0f U(%dx杠杆)
// 4. 保证金: 总使用率 ≤ 90%
```

**硬约束对做空的影响**：
- 做空的止损止盈设置往往更容易满足 ≥ 1:3 比例
- BTC/ETH 可以用更高杠杆（最多 10 倍），做空时风险更容易被夸大

---

## 三层问题总结

| 问题层级 | 问题类型 | 位置 | 对做空的影响 | 优先级 |
|---------|---------|------|------------|--------|
| **第 1 层** | Prompt 措辞 | L229-235, L298-301 | 心理暗示 | 🟠 中 |
| **第 2 层** | 数学不对称 | L591-614 | 验证难度低 | 🟡 中-高 |
| **第 3 层** | 市场现实 | N/A | 实际做空机会多 | 🔴 高 |

---

## 修复方案

### 方案 A：修改 System Prompt（推荐）

**文件**: `decision/engine.go` 第 203-316 行

#### 修改 1：调整措辞（L229-235）

**原始**：
```
# 📉 做多做空平衡
**重要**: 下跌趋势做空的利润 = 上涨趋势做多的利润
- 上涨趋势 → 做多
- 下跌趋势 → 做空
- 震荡市场 → 观望
**不要有做多偏见！做空是你的核心工具之一**
```

**修改**：
```
# ⚖️ 做多做空平衡策略
**原则**: 
- 上涨趋势 → 做多（强信号）
- 下跌趋势 → 做空（强信号）
- 震荡市场 → 观望

**重要提示**: 做多和做空价值等同，都是盈利的重要工具。
选择做多或做空应基于**市场趋势**，而非偏好。
```

#### 修改 2：调整示例顺序（L298-301）

**原始**：
```json
{
  "symbol": "BTCUSDT",
  "action": "open_short",  // 做空示例先出现
  "leverage": 10,
  ...
},
{
  "symbol": "ETHUSDT",
  "action": "close_long",  // 平多示例后出现
  ...
}
```

**修改**：
```json
{
  "symbol": "ETHUSDT",
  "action": "open_long",   // 做多示例先出现
  "leverage": 5,
  ...
},
{
  "symbol": "BTCUSDT",
  "action": "open_short",  // 做空示例后出现
  "leverage": 10,
  ...
}
```

---

### 方案 B：添加做空额外限制（保守）

**文件**: `decision/engine.go` 第 534-622 行

在 `validateDecision` 中添加：

```go
// 对做空交易额外要求更严格的信心度
if d.Action == "open_short" {
    minConfidence := 80  // 做空需要 ≥ 80 信心度
    if d.Confidence < minConfidence {
        return fmt.Errorf("做空交易需要更高的信心度(≥%d)，当前: %d", minConfidence, d.Confidence)
    }
}
```

---

### 方案 C：优化风险计算（高级）

**文件**: `decision/engine.go` 第 534-622 行

使用**更对称的入场点假设**：

```go
if d.Action == "open_long" {
    // 做多：假设在中点入场（50%位置）
    entryPrice = d.StopLoss + (d.TakeProfit - d.StopLoss) * 0.5
} else {
    // 做空：假设在中点入场（50%位置）
    entryPrice = d.TakeProfit + (d.StopLoss - d.TakeProfit) * 0.5
}
```

这会让做多和做空的风险回报比计算更对称。

---

## 监控指标

修复后需要监控：

1. **做多/做空交易比例**
   - 修复前：可能 1:3 或更极端
   - 修复后：应接近 1:1

2. **胜率对比**
   - 做多胜率 vs 做空胜率
   - 应该大致相等

3. **平均盈亏比**
   - 做多 avg profit:loss ratio vs 做空
   - 应该大致相等

4. **夏普比率**
   - 修复前后的夏普比率变化
   - 不应该显著下降

---

## 时间线

- **发现时间**: 2025-11-02
- **问题确认**: 用户已实现修复并获利
- **建议**: 立即在主分支实施 方案 A + B

---

## 参考信息

- 前端竞赛页面：`screenshots/competition-page.png`
- 交易详情页：`screenshots/details-page.png`
- 配置文件：`config.json.example`
- AI 模型：Qwen vs DeepSeek

---

## 相关代码文件

- `decision/engine.go` - AI 决策引擎 ⚠️ 主要修改文件
- `trader/auto_trader.go` - 自动交易执行
- `decision/decision.go` - 决策结构定义
- `mcp/client.go` - AI API 通信

