# 交易功能核心变更文档

> 本文档详细记录所有与交易逻辑相关的核心变更

---

## 📋 目录

1. [新增交易动作](#新增交易动作)
2. [决策引擎变更](#决策引擎变更)
3. [交易执行流程](#交易执行流程)
4. [风险控制变更](#风险控制变更)
5. [交易接口扩展](#交易接口扩展)
6. [市场数据获取](#市场数据获取)
7. [未实现功能清单](#未实现功能清单)

---

## 🎯 新增交易动作

### 1. 动态止损调整 (`update_stop_loss`)

**功能**: 允许AI动态调整现有持仓的止损价格

**触发条件**:
- 已有持仓
- 市场走势发生变化
- AI判断需要移动止损位

**AI决策示例**:
```json
{
  "symbol": "BTCUSDT",
  "action": "update_stop_loss",
  "new_stop_loss": 42000.0,
  "reasoning": "价格突破关键阻力位，移动止损到盈亏平衡点保护利润"
}
```

**执行逻辑** (trader/auto_trader.go:597):
```go
case "update_stop_loss":
    return at.executeUpdateStopLossWithRecord(decision, actionRecord)
```

⚠️ **警告**: `executeUpdateStopLossWithRecord()` 函数当前**未实现**，需要补充！

---

### 2. 动态止盈调整 (`update_take_profit`)

**功能**: 允许AI动态调整现有持仓的止盈价格

**触发条件**:
- 已有持仓
- 盈利扩大，预期继续上涨
- AI判断可以追踪更高目标

**AI决策示例**:
```json
{
  "symbol": "ETHUSDT",
  "action": "update_take_profit",
  "new_take_profit": 2500.0,
  "reasoning": "多头趋势强劲，上调止盈目标以获取更多利润"
}
```

**执行逻辑** (trader/auto_trader.go:599):
```go
case "update_take_profit":
    return at.executeUpdateTakeProfitWithRecord(decision, actionRecord)
```

⚠️ **警告**: `executeUpdateTakeProfitWithRecord()` 函数当前**未实现**，需要补充！

---

### 3. 部分平仓 (`partial_close`)

**功能**: 部分平仓以锁定利润或减少损失

**触发条件**:
- 已有持仓
- 盈利达到一定程度，想锁定部分利润
- 或损失扩大，想减少风险敞口

**AI决策示例**:
```json
{
  "symbol": "SOLUSDT",
  "action": "partial_close",
  "close_percentage": 50.0,
  "reasoning": "盈利已达8%，先平仓50%锁定利润，剩余仓位继续持有等待更高目标"
}
```

**执行逻辑** (trader/auto_trader.go:601):
```go
case "partial_close":
    return at.executePartialCloseWithRecord(decision, actionRecord)
```

⚠️ **警告**: `executePartialCloseWithRecord()` 函数当前**未实现**，需要补充！

---

## 🧠 决策引擎变更

### Decision结构扩展

**文件**: `decision/engine.go:73-87`

**新增字段**:
```go
type Decision struct {
    Symbol          string  `json:"symbol"`
    Action          string  `json:"action"` // 新增: "update_stop_loss", "update_take_profit", "partial_close"

    // 开仓参数（原有）
    Leverage        int     `json:"leverage,omitempty"`
    PositionSizeUSD float64 `json:"position_size_usd,omitempty"`
    StopLoss        float64 `json:"stop_loss,omitempty"`
    TakeProfit      float64 `json:"take_profit,omitempty"`

    // 🆕 调整参数（新增）
    NewStopLoss     float64 `json:"new_stop_loss,omitempty"`     // 用于 update_stop_loss
    NewTakeProfit   float64 `json:"new_take_profit,omitempty"`   // 用于 update_take_profit
    ClosePercentage float64 `json:"close_percentage,omitempty"`  // 用于 partial_close (0-100)

    // 通用参数（原有）
    Confidence      int     `json:"confidence,omitempty"`
    RiskUSD         float64 `json:"risk_usd,omitempty"`
    Reasoning       string  `json:"reasoning"`
}
```

### 执行优先级调整

**文件**: `trader/auto_trader.go:936-948`

**新优先级规则**:
```go
func sortDecisionsByPriority(decisions []decision.Decision) []decision.Decision {
    getActionPriority := func(action string) int {
        switch action {
        case "close_long", "close_short", "partial_close":
            return 1 // 🔴 最高优先级：平仓（包括部分平仓）
        case "update_stop_loss", "update_take_profit":
            return 2 // 🟡 调整止盈止损
        case "open_long", "open_short":
            return 3 // 🟢 次优先级：开仓
        case "hold", "wait":
            return 4 // ⚪ 最低优先级：观望
        default:
            return 999
        }
    }
    // ... 排序逻辑
}
```

**为什么这样排序**:
1. **先平仓** - 释放保证金，避免仓位叠加超限
2. **再调整TP/SL** - 保护现有持仓
3. **后开仓** - 确保有足够保证金
4. **最后观望** - 无需执行

---

## ⚙️ 交易执行流程

### 完整执行链路

```
AI决策 → 排序 → 验证 → 执行 → 记录 → 日志
```

### 执行函数映射

| 动作 | 函数名 | 状态 | 位置 |
|------|--------|------|------|
| `open_long` | `executeOpenLongWithRecord()` | ✅ 已实现 | trader/auto_trader.go:618 |
| `open_short` | `executeOpenShortWithRecord()` | ✅ 已实现 | trader/auto_trader.go:677 |
| `close_long` | `executeCloseLongWithRecord()` | ✅ 已实现 | trader/auto_trader.go:... |
| `close_short` | `executeCloseShortWithRecord()` | ✅ 已实现 | trader/auto_trader.go:... |
| `update_stop_loss` | `executeUpdateStopLossWithRecord()` | ❌ **未实现** | - |
| `update_take_profit` | `executeUpdateTakeProfitWithRecord()` | ❌ **未实现** | - |
| `partial_close` | `executePartialCloseWithRecord()` | ❌ **未实现** | - |

---

## 🛡️ 风险控制变更

### 仓位模式控制

**文件**: `trader/auto_trader.go:636-640`

**新增全仓/逐仓模式支持**:
```go
// 设置仓位模式
if err := at.trader.SetMarginMode(decision.Symbol, at.config.IsCrossMargin); err != nil {
    log.Printf("  ⚠️ 设置仓位模式失败: %v", err)
    // 继续执行，不影响交易
}
```

**配置项** (trader/auto_trader.go:66):
```go
type AutoTraderConfig struct {
    // ... 其他配置
    IsCrossMargin bool // true=全仓模式, false=逐仓模式
}
```

### 幽灵持仓过滤

**文件**: `trader/auto_trader.go:487-490`

**修复**: 过滤quantity=0的持仓，防止传递给AI
```go
// 跳过已平仓的持仓（quantity = 0），防止"幽灵持仓"传递给AI
if quantity == 0 {
    continue
}
```

**影响**: 避免AI基于已平仓的持仓做出错误决策

---

## 🔌 交易接口扩展

### Trader接口新增方法

**文件**: `trader/interface.go`

**新增接口**:
```go
type Trader interface {
    // ... 原有方法

    // 🆕 动态止盈止损接口
    UpdateStopLoss(symbol string, newStopLoss float64) error
    UpdateTakeProfit(symbol string, newTakeProfit float64) error
    PartialClose(symbol string, closePercentage float64) error

    // 🆕 仓位模式设置
    SetMarginMode(symbol string, isCrossMargin bool) error
}
```

### 交易所实现状态

| 交易所 | UpdateStopLoss | UpdateTakeProfit | PartialClose | SetMarginMode |
|--------|----------------|------------------|--------------|---------------|
| **Binance Futures** | ❌ 未实现 | ❌ 未实现 | ❌ 未实现 | ⚠️ 部分实现 |
| **Hyperliquid** | ❌ 未实现 | ❌ 未实现 | ❌ 未实现 | ⚠️ 部分实现 |
| **Aster** | ❌ 未实现 | ❌ 未实现 | ❌ 未实现 | ❌ 未实现 |

⚠️ **关键问题**: 所有交易所都缺少核心方法实现！

---

## 📊 市场数据获取优化

### WebSocket实时数据流

**新增文件**:
- `market/websocket_client.go` - WebSocket客户端
- `market/combined_streams.go` - 组合数据流
- `market/monitor.go` - 数据监控器
- `market/api_client.go` - API客户端
- `market/types.go` - 数据类型定义

### 核心改进

**1. 实时K线数据缓存**

**文件**: `market/monitor.go`

**工作流程**:
```
启动时 → 订阅交易员币种 → 建立WebSocket连接 → 缓存数据
决策时 → 优先读缓存 → 缓存未命中 → 实时API获取 → 添加订阅
```

**优势**:
- ✅ 减少API调用次数
- ✅ 降低延迟
- ✅ 支持自动重连
- ✅ 支持币种动态添加

**2. 启动WebSocket监控**

**文件**: `main.go:275-276`

**代码**:
```go
// 启动流行情数据 - 默认使用所有交易员设置的币种
go market.NewWSMonitor(150).Start(database.GetCustomCoins())
```

**参数说明**:
- `150` - 最大订阅币种数
- `database.GetCustomCoins()` - 从数据库获取交易员配置的币种

---

## ⚠️ 未实现功能清单

### Critical Priority (影响核心功能)

#### 1. 缺少执行函数实现

**位置**: `trader/auto_trader.go`

**需要实现的函数**:

```go
// ❌ 未实现
func (at *AutoTrader) executeUpdateStopLossWithRecord(
    decision *decision.Decision,
    actionRecord *logger.DecisionAction,
) error {
    // TODO: 实现逻辑
    // 1. 验证持仓存在
    // 2. 验证新止损价格合理性
    // 3. 调用交易所API更新止损
    // 4. 记录执行结果
    return fmt.Errorf("未实现")
}

// ❌ 未实现
func (at *AutoTrader) executeUpdateTakeProfitWithRecord(
    decision *decision.Decision,
    actionRecord *logger.DecisionAction,
) error {
    // TODO: 实现逻辑
    return fmt.Errorf("未实现")
}

// ❌ 未实现
func (at *AutoTrader) executePartialCloseWithRecord(
    decision *decision.Decision,
    actionRecord *logger.DecisionAction,
) error {
    // TODO: 实现逻辑
    // 1. 验证持仓存在
    // 2. 计算平仓数量 = quantity * (closePercentage / 100)
    // 3. 调用交易所API部分平仓
    // 4. 记录执行结果和盈亏
    return fmt.Errorf("未实现")
}
```

#### 2. 缺少交易所接口实现

**位置**: `trader/binance_futures.go`, `trader/hyperliquid_trader.go`

**Binance Futures需要实现**:
```go
func (b *BinanceFuturesTrader) UpdateStopLoss(symbol string, newSL float64) error {
    // TODO: 调用Binance API修改止损单
    // API: POST /fapi/v1/order (修改止损单)
}

func (b *BinanceFuturesTrader) UpdateTakeProfit(symbol string, newTP float64) error {
    // TODO: 调用Binance API修改止盈单
}

func (b *BinanceFuturesTrader) PartialClose(symbol string, percentage float64) error {
    // TODO:
    // 1. 获取当前持仓数量
    // 2. 计算平仓数量
    // 3. 下平仓市价单
}
```

**Hyperliquid需要实现**:
```go
func (h *HyperliquidTrader) UpdateStopLoss(symbol string, newSL float64) error {
    // TODO: 调用Hyperliquid API
}

func (h *HyperliquidTrader) UpdateTakeProfit(symbol string, newTP float64) error {
    // TODO: 调用Hyperliquid API
}

func (h *HyperliquidTrader) PartialClose(symbol string, percentage float64) error {
    // TODO: 实现逻辑
}
```

### High Priority (影响用户体验)

#### 3. 缺少错误处理

**问题**: 当前如果调用未实现的函数，会直接panic
**建议**: 添加graceful降级

```go
case "update_stop_loss":
    if err := at.executeUpdateStopLossWithRecord(decision, actionRecord); err != nil {
        log.Printf("⚠️ 动态调整止损失败: %v，跳过该操作", err)
        actionRecord.Success = false
        actionRecord.ErrorMessage = err.Error()
        return nil // 不要中断整个决策流程
    }
```

#### 4. 缺少参数验证

**建议**: 在执行前验证决策参数

```go
// 验证部分平仓百分比
if decision.Action == "partial_close" {
    if decision.ClosePercentage <= 0 || decision.ClosePercentage > 100 {
        return fmt.Errorf("无效的平仓百分比: %.2f，必须在(0, 100]范围内", decision.ClosePercentage)
    }
}

// 验证止损价格合理性
if decision.Action == "update_stop_loss" {
    currentPrice := marketData.CurrentPrice
    if decision.NewStopLoss >= currentPrice {
        return fmt.Errorf("多头止损价格不能高于当前价: SL=%.2f, Price=%.2f",
            decision.NewStopLoss, currentPrice)
    }
}
```

---

## 🧪 测试建议

### 单元测试

**需要覆盖的场景**:

```go
// TestExecutePartialClose
func TestExecutePartialClose(t *testing.T) {
    // 测试用例
    cases := []struct{
        name string
        percentage float64
        expectError bool
    }{
        {"正常平仓50%", 50.0, false},
        {"平仓100%", 100.0, false},
        {"无效百分比-负数", -10.0, true},
        {"无效百分比-超过100", 150.0, true},
        {"无效百分比-0", 0.0, true},
    }
    // ... 测试逻辑
}
```

### 集成测试

**测试流程**:
1. 创建测试交易员
2. 开仓 → 等待持仓确认
3. 执行`partial_close` 50% → 验证仓位减半
4. 执行`update_stop_loss` → 验证止损单更新
5. 执行`update_take_profit` → 验证止盈单更新
6. 全部平仓 → 验证盈亏计算

---

## 📋 实现清单

### 开发者TODO

- [ ] **Critical**: 实现`executeUpdateStopLossWithRecord()`
- [ ] **Critical**: 实现`executeUpdateTakeProfitWithRecord()`
- [ ] **Critical**: 实现`executePartialCloseWithRecord()`
- [ ] **Critical**: Binance Futures实现新接口
- [ ] **Critical**: Hyperliquid实现新接口
- [ ] **High**: 添加参数验证逻辑
- [ ] **High**: 添加错误降级处理
- [ ] **Medium**: 编写单元测试
- [ ] **Medium**: 编写集成测试
- [ ] **Low**: 添加详细日志记录

---

## 🔍 代码示例

### 推荐的实现模板

```go
func (at *AutoTrader) executePartialCloseWithRecord(
    decision *decision.Decision,
    actionRecord *logger.DecisionAction,
) error {
    log.Printf("📉 执行部分平仓: %s (%.1f%%)", decision.Symbol, decision.ClosePercentage)

    // 1. 参数验证
    if decision.ClosePercentage <= 0 || decision.ClosePercentage > 100 {
        return fmt.Errorf("无效的平仓百分比: %.2f", decision.ClosePercentage)
    }

    // 2. 获取当前持仓
    positions, err := at.trader.GetPositions()
    if err != nil {
        return fmt.Errorf("获取持仓失败: %w", err)
    }

    var currentPosition map[string]interface{}
    for _, pos := range positions {
        if pos["symbol"] == decision.Symbol {
            currentPosition = pos
            break
        }
    }

    if currentPosition == nil {
        return fmt.Errorf("未找到持仓: %s", decision.Symbol)
    }

    // 3. 计算平仓数量
    quantity := currentPosition["quantity"].(float64)
    closeQuantity := quantity * (decision.ClosePercentage / 100.0)

    // 4. 执行平仓
    side := currentPosition["side"].(string)
    var order map[string]interface{}

    if side == "long" {
        order, err = at.trader.CloseLong(decision.Symbol, closeQuantity)
    } else {
        order, err = at.trader.CloseShort(decision.Symbol, closeQuantity)
    }

    if err != nil {
        actionRecord.Success = false
        actionRecord.ErrorMessage = fmt.Sprintf("平仓失败: %v", err)
        return err
    }

    // 5. 记录结果
    actionRecord.Success = true
    actionRecord.Quantity = closeQuantity
    actionRecord.Price = order["avgPrice"].(float64)
    actionRecord.OrderID = int64(order["orderId"].(float64))

    log.Printf("  ✓ 部分平仓成功: %.4f %s @ %.2f",
        closeQuantity, decision.Symbol, actionRecord.Price)

    return nil
}
```

---

## 📞 问题反馈

如发现本文档有误或需要补充，请提交Issue或PR。

**相关文档**:
- [核心业务差异](../CORE_BUSINESS_DIFF.md)
- [合并总结](../MERGE_SUMMARY.md)
