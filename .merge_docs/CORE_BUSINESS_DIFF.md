# 核心业务逻辑差异对比

> 本文档对比合并前后的核心业务逻辑变化

---

## 📋 目录

1. [架构层面变化](#架构层面变化)
2. [用户系统](#用户系统)
3. [交易员管理](#交易员管理)
4. [AI决策系统](#ai决策系统)
5. [提示词系统](#提示词系统)
6. [数据存储](#数据存储)
7. [API接口](#api接口)

---

## 🏗️ 架构层面变化

### Before (main分支)

```
单用户模式
└── 交易员配置(config.json)
    └── 直接启动交易员
        └── AI决策
            └── 执行交易
```

**特点**:
- 所有配置在`config.json`
- 单一用户，无认证
- 交易员配置硬编码
- 无多租户支持

### After (合并后)

```
多用户模式
└── 用户注册/登录 (JWT认证)
    └── 用户管理交易员
        └── 交易员配置(数据库)
            └── AI决策(支持自定义提示词)
                └── 执行交易(支持动态TP/SL)
```

**特点**:
- 数据库驱动配置
- 多用户系统
- 每个用户独立管理交易员
- 灵活的提示词系统

---

## 👥 用户系统

### Before: 无用户概念

```go
// main.go - 直接启动
func main() {
    // 读取config.json
    // 创建交易员
    // 启动交易
}
```

### After: 完整用户系统

**新增文件**: `auth/auth.go`

**核心功能**:

1. **用户注册**
```go
POST /api/register
{
    "email": "user@example.com",
    "password": "secure_password"
}

Response:
{
    "message": "请检查邮箱验证码",
    "otp_required": true
}
```

2. **OTP验证**
```go
POST /api/verify-otp
{
    "email": "user@example.com",
    "otp": "123456"
}
```

3. **登录认证**
```go
POST /api/login
{
    "email": "user@example.com",
    "password": "password"
}

Response:
{
    "token": "eyJhbGciOiJIUzI1NiIs...",
    "user_id": "uuid-xxxx"
}
```

4. **JWT Token验证**
```go
// 所有API请求需要携带
Authorization: Bearer <token>
```

**影响**:
- ✅ 支持多用户
- ✅ 数据隔离
- ✅ 安全性提升
- ⚠️ 增加了复杂度

---

## 🤖 交易员管理

### Before: 单一交易员配置

**配置文件**: `config.json`

```json
{
  "exchange": "binance",
  "ai_model": "deepseek",
  "deepseek_key": "sk-xxx",
  "binance_key": "xxx",
  "binance_secret": "xxx"
}
```

**启动方式**:
```bash
# 只能运行一个交易员
./nofx
```

### After: 多交易员动态管理

**存储位置**: 数据库 `traders` 表

**数据结构** (`config/database.go:194-215`):
```go
type Trader struct {
    ID                   string
    UserID               string
    Name                 string
    AIModelID            string
    ExchangeID           string
    InitialBalance       float64
    BTCETHLeverage       int
    AltcoinLeverage      int
    TradingSymbols       string  // JSON数组
    CustomPrompt         string
    OverrideBasePrompt   bool
    SystemPromptTemplate string  // "default", "adaptive", "nof1"
    IsCrossMargin        bool
    UseCoinPool          bool
    UseOITop             bool
    IsRunning            bool
    CreatedAt            time.Time
    UpdatedAt            time.Time
}
```

**API管理**:
```go
// 创建交易员
POST /api/traders
{
    "name": "BTC策略1号",
    "ai_model_id": "admin_deepseek",
    "exchange_id": "binance",
    "initial_balance": 1000.0,
    "btc_eth_leverage": 5,
    "altcoin_leverage": 3,
    "trading_symbols": "BTCUSDT,ETHUSDT",
    "system_prompt_template": "adaptive",
    "is_cross_margin": true
}

// 启动/停止交易员
POST /api/traders/:id/start
POST /api/traders/:id/stop

// 更新配置
PUT /api/traders/:id

// 删除交易员
DELETE /api/traders/:id
```

**启动方式**:
```bash
# 可以同时运行多个交易员
./nofx
# 通过Web UI或API控制启动/停止
```

**影响**:
- ✅ 支持多策略并行
- ✅ 动态启停
- ✅ 配置热更新
- ⚠️ 需要通过Web UI管理

---

## 🧠 AI决策系统

### 决策结构变化

#### Before
```go
type Decision struct {
    Symbol          string
    Action          string  // "open_long", "open_short", "close_long", "close_short", "hold"
    Leverage        int
    PositionSizeUSD float64
    StopLoss        float64
    TakeProfit      float64
    Confidence      int
    Reasoning       string
}
```

#### After
```go
type Decision struct {
    Symbol          string
    Action          string  // 🆕 新增 "update_stop_loss", "update_take_profit", "partial_close"

    // 开仓参数
    Leverage        int
    PositionSizeUSD float64
    StopLoss        float64
    TakeProfit      float64

    // 🆕 调整参数
    NewStopLoss     float64  // 新增：用于动态调整止损
    NewTakeProfit   float64  // 新增：用于动态调整止盈
    ClosePercentage float64  // 新增：用于部分平仓 (0-100)

    Confidence      int
    RiskUSD         float64
    Reasoning       string
}
```

### 决策获取流程变化

#### Before
```go
// decision/engine.go
func GetFullDecision(ctx *Context, mcpClient *mcp.Client) (*FullDecision, error) {
    // 使用硬编码的system prompt
    systemPrompt := buildSystemPrompt(...)
    userPrompt := buildUserPrompt(ctx)

    // 调用AI
    response := mcpClient.CallAI(systemPrompt, userPrompt)
    return parseResponse(response)
}
```

**固定提示词**, 所有交易员使用相同策略

#### After
```go
// decision/engine.go
func GetFullDecisionWithCustomPrompt(
    ctx *Context,
    mcpClient *mcp.Client,
    customPrompt string,          // 🆕 自定义提示词
    overrideBase bool,            // 🆕 是否覆盖基础提示词
    templateName string           // 🆕 模板名称
) (*FullDecision, error) {
    // 🆕 从模板文件加载system prompt
    systemPrompt := buildSystemPromptWithCustom(
        ctx.Account.TotalEquity,
        ctx.BTCETHLeverage,
        ctx.AltcoinLeverage,
        customPrompt,
        overrideBase,
        templateName
    )

    // 构建用户prompt（包含市场数据）
    userPrompt := buildUserPrompt(ctx)

    // 调用AI
    response := mcpClient.CallAI(systemPrompt, userPrompt)

    // 🆕 记录完整prompt到日志
    decision.SystemPrompt = systemPrompt
    decision.UserPrompt = userPrompt

    return decision, nil
}
```

**灵活提示词**, 每个交易员可以自定义策略

---

## 📝 提示词系统

### Before: 硬编码提示词

**位置**: `decision/engine.go` 内嵌字符串

```go
func buildSystemPrompt(...) string {
    return `
你是专业的加密货币交易AI...
# 核心目标
最大化夏普比率...
...（约500行硬编码文本）
`
}
```

**缺点**:
- ❌ 无法自定义
- ❌ 修改需要重新编译
- ❌ 所有交易员策略相同
- ❌ 难以A/B测试

### After: 模板化提示词系统

**新增文件**:
- `prompts/default.txt` - 默认保守策略
- `prompts/adaptive.txt` - 自适应双策略（震荡+趋势）
- `prompts/nof1.txt` - 社区贡献策略

**新增模块**: `decision/prompt_manager.go`

```go
// 提示词模板管理
type PromptTemplate struct {
    Name        string
    Description string
    Content     string
    CreatedAt   time.Time
}

// 加载提示词模板
func GetPromptTemplate(name string) (*PromptTemplate, error) {
    // 从 prompts/ 目录读取
    // 支持热加载，无需重启
}

// 列出所有模板
func ListPromptTemplates() ([]*PromptTemplate, error) {
    // 返回所有可用模板
}
```

**使用方式**:

1. **使用默认模板**
```go
trader := NewAutoTrader(AutoTraderConfig{
    SystemPromptTemplate: "default",  // 使用默认策略
})
```

2. **使用adaptive模板**
```go
trader := NewAutoTrader(AutoTraderConfig{
    SystemPromptTemplate: "adaptive",  // 使用自适应策略
})
```

3. **完全自定义**
```go
trader := NewAutoTrader(AutoTraderConfig{
    SystemPromptTemplate: "default",
    CustomPrompt: `
        # 我的个性化策略
        - 只做BTC和ETH
        - 只做多不做空
        - 最大回撤3%
    `,
    OverrideBasePrompt: true,  // 完全覆盖基础模板
})
```

4. **增强默认模板**
```go
trader := NewAutoTrader(AutoTraderConfig{
    SystemPromptTemplate: "adaptive",
    CustomPrompt: `
        # 额外风控规则
        - 单笔最大亏损2%
        - 避开周末交易
    `,
    OverrideBasePrompt: false,  // 在adaptive基础上增强
})
```

**优势**:
- ✅ 灵活配置
- ✅ 支持A/B测试
- ✅ 社区可贡献策略
- ✅ 无需重新编译

**adaptive.txt核心特性**:
- 震荡市场策略（支撑阻力位交易）
- 趋势市场策略（趋势跟踪）
- 自适应切换
- **包含动态TP/SL说明** (447行)

---

## 💾 数据存储

### Before: 文件存储

**配置**: `config.json`
**日志**: `decision_logs/` 目录
**数据库**: 无

### After: 混合存储

**SQLite数据库**: `config.db`

**表结构**:
```sql
-- 用户表
CREATE TABLE users (
    id TEXT PRIMARY KEY,
    email TEXT UNIQUE,
    password_hash TEXT,
    created_at TIMESTAMP,
    updated_at TIMESTAMP
);

-- 交易员表
CREATE TABLE traders (
    id TEXT PRIMARY KEY,
    user_id TEXT,
    name TEXT,
    ai_model_id TEXT,
    exchange_id TEXT,
    initial_balance REAL,
    btc_eth_leverage INTEGER,
    altcoin_leverage INTEGER,
    trading_symbols TEXT,
    custom_prompt TEXT,
    override_base_prompt INTEGER,
    system_prompt_template TEXT,
    is_cross_margin INTEGER,
    use_coin_pool INTEGER,
    use_oi_top INTEGER,
    is_running INTEGER,
    created_at TIMESTAMP,
    updated_at TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users(id)
);

-- AI模型配置表
CREATE TABLE ai_models (
    id TEXT PRIMARY KEY,
    user_id TEXT,
    provider TEXT,
    enabled INTEGER,
    api_key TEXT,
    custom_api_url TEXT,
    custom_model_name TEXT,
    created_at TIMESTAMP,
    updated_at TIMESTAMP
);

-- 交易所配置表
CREATE TABLE exchanges (
    id TEXT PRIMARY KEY,
    user_id TEXT,
    name TEXT,
    type TEXT,
    enabled INTEGER,
    api_key TEXT,
    secret_key TEXT,
    testnet INTEGER,
    hyperliquid_wallet_addr TEXT,
    aster_user TEXT,
    aster_signer TEXT,
    aster_private_key TEXT,
    created_at TIMESTAMP,
    updated_at TIMESTAMP
);

-- 系统配置表
CREATE TABLE system_config (
    key TEXT PRIMARY KEY,
    value TEXT,
    updated_at TIMESTAMP
);
```

**文件存储**:
- 提示词模板: `prompts/`
- 决策日志: `decision_logs/<trader_id>/`

**配置迁移**:
```go
// 启动时从config.json同步到数据库
func syncConfigToDatabase(database *Database) error {
    // 读取config.json
    // 写入system_config表
}
```

**影响**:
- ✅ 支持复杂查询
- ✅ 数据关联性强
- ✅ 支持事务
- ⚠️ 需要数据库迁移

---

## 🔌 API接口

### Before: 简单REST API

**端点**:
```
GET  /api/health
GET  /api/traders
GET  /api/status?trader_id=xxx
GET  /api/account?trader_id=xxx
GET  /api/positions?trader_id=xxx
GET  /api/decisions?trader_id=xxx
```

**无认证**, 所有接口公开

### After: 完整REST API + 认证

**公开端点** (无需认证):
```
POST /api/register               # 用户注册
POST /api/login                  # 用户登录
POST /api/verify-otp             # OTP验证
POST /api/complete-registration  # 完成注册
GET  /api/supported-models       # 获取支持的AI模型列表
GET  /api/supported-exchanges    # 获取支持的交易所列表
GET  /api/config                 # 获取系统配置
GET  /api/prompt-templates       # 获取提示词模板列表
GET  /api/prompt-templates/:name # 获取指定提示词模板
```

**认证端点** (需要JWT Token):
```
# 交易员管理
GET    /api/traders                   # 获取用户的交易员列表
GET    /api/traders/:id/config        # 获取交易员配置
POST   /api/traders                   # 创建新交易员
PUT    /api/traders/:id               # 更新交易员配置
DELETE /api/traders/:id               # 删除交易员
POST   /api/traders/:id/start         # 启动交易员
POST   /api/traders/:id/stop          # 停止交易员
PUT    /api/traders/:id/prompt        # 更新交易员提示词

# AI模型配置
GET    /api/models                    # 获取用户的AI模型配置
PUT    /api/models                    # 更新AI模型配置

# 交易所配置
GET    /api/exchanges                 # 获取用户的交易所配置
PUT    /api/exchanges                 # 更新交易所配置

# 用户信号源配置
GET    /api/user/signal-sources       # 获取用户信号源配置
POST   /api/user/signal-sources       # 保存用户信号源配置

# 竞赛总览（保留原有功能）
GET    /api/competition               # 对比所有trader

# 交易员数据（需要trader_id参数）
GET    /api/status?trader_id=xxx
GET    /api/account?trader_id=xxx
GET    /api/positions?trader_id=xxx
GET    /api/decisions?trader_id=xxx
GET    /api/decisions/latest?trader_id=xxx
GET    /api/statistics?trader_id=xxx
GET    /api/equity-history?trader_id=xxx
GET    /api/performance?trader_id=xxx
```

**认证流程**:
```
1. POST /api/login → 获取JWT token
2. 后续请求携带: Authorization: Bearer <token>
3. 服务端验证token → 提取user_id → 数据隔离
```

**API版本**: 无版本控制（建议未来添加 /api/v1/）

---

## 📊 核心流程对比

### 交易员启动流程

#### Before
```
1. 读取config.json
2. 初始化单个交易员
3. 启动交易循环
```

#### After
```
1. 读取config.json（向后兼容）
2. 同步配置到数据库
3. 用户通过Web UI登录
4. 用户创建/配置交易员
5. 用户点击"启动"按钮
6. 后台加载交易员配置
7. 启动对应交易循环
```

### AI决策流程

#### Before
```
market_data → build_prompt → call_AI → parse → execute
```

#### After
```
market_data → load_template → merge_custom_prompt →
build_prompt → call_AI → parse → sort_by_priority →
validate → execute → record_full_context
```

---

## 🔄 向后兼容性

### 保留的功能

1. **config.json支持**
   - 仍然可以使用config.json配置
   - 启动时自动同步到数据库

2. **单交易员模式**
   - 如果数据库为空，使用config.json创建默认交易员

3. **原有API端点**
   - `/api/status`, `/api/account` 等仍然可用
   - 默认返回第一个交易员数据（如果无user_id）

### 不兼容变更

1. **需要数据库**
   - 首次启动会创建`config.db`
   - 如果删除数据库，需要重新配置

2. **API认证**
   - 大部分API需要JWT token
   - 公开API减少

3. **前端UI完全重写**
   - 旧版UI不可用
   - 必须使用新版登录界面

---

## 🎯 关键决策点

### 1. 为什么选择SQLite而不是MySQL/PostgreSQL？

**优势**:
- ✅ 零配置，开箱即用
- ✅ 单文件，易于备份
- ✅ 适合中小规模（<100用户）
- ✅ 事务支持

**劣势**:
- ⚠️ 并发写入性能有限
- ⚠️ 不适合大规模多租户
- ⚠️ 缺少网络访问能力

### 2. 为什么使用JWT而不是Session？

**优势**:
- ✅ 无状态，易于扩展
- ✅ 支持分布式部署
- ✅ 前后端分离友好

**劣势**:
- ⚠️ Token无法主动失效（除非设置黑名单）
- ⚠️ Token体积较大

### 3. 为什么提示词使用文件而不是数据库？

**优势**:
- ✅ 易于版本控制（Git）
- ✅ 易于分享和贡献
- ✅ 支持长文本和格式化

**劣势**:
- ⚠️ 动态修改需要文件操作
- ⚠️ 无法通过Web UI编辑

---

## 📈 性能影响

### WebSocket优化

**Before**: 每次决策都实时调用Binance API获取K线
```
决策周期: 3分钟
API调用: 8个币种 × 3次请求 = 24次/周期
延迟: ~500ms per API call
```

**After**: 使用WebSocket缓存
```
决策周期: 3分钟
WebSocket订阅: 1次（启动时）
API调用: 仅缓存未命中时
延迟: ~10ms from cache
```

**性能提升**: ~50倍

### 数据库查询

**新增开销**:
- 每次决策前需要查询trader配置（~1ms）
- 每次API请求需要验证JWT（~0.5ms）

**影响**: 可忽略不计

---

## 🔒 安全性提升

### Before
- ❌ 无用户认证
- ❌ API密钥明文存储
- ❌ 无访问控制
- ❌ 无操作审计

### After
- ✅ JWT认证
- ✅ 密码哈希存储（bcrypt）
- ✅ API密钥加密存储（可选）
- ✅ 基于用户的访问控制
- ✅ 操作日志记录

---

## 📌 迁移建议

### 从旧版本升级

1. **备份数据**
```bash
cp config.json config.json.backup
cp -r decision_logs decision_logs.backup
```

2. **启动新版本**
```bash
# 会自动创建config.db并迁移配置
./nofx
```

3. **注册账号**
- 访问 http://localhost:3000
- 注册新账号
- 邮箱验证（如果启用）

4. **创建交易员**
- 使用原config.json的配置
- 通过Web UI创建交易员

5. **验证功能**
- 启动交易员
- 检查决策日志
- 验证交易执行

### 配置文件对照表

| config.json | 数据库位置 | Web UI位置 |
|-------------|-----------|-----------|
| `exchange` | `traders.exchange_id` | 交易员设置→交易所 |
| `ai_model` | `traders.ai_model_id` | 交易员设置→AI模型 |
| `deepseek_key` | `ai_models.api_key` | 设置→AI配置 |
| `binance_key` | `exchanges.api_key` | 设置→交易所配置 |
| `default_coins` | `system_config.default_coins` | 系统设置→默认币种 |

---

## 🎓 学习曲线

### 开发者需要了解

1. **数据库操作** (`config/database.go`)
2. **JWT认证流程** (`auth/auth.go`)
3. **多用户架构**
4. **提示词模板系统** (`decision/prompt_manager.go`)
5. **WebSocket数据流** (`market/`)

### 用户需要了解

1. **注册/登录流程**
2. **创建和管理交易员**
3. **配置AI模型和交易所**
4. **选择提示词模板**
5. **监控交易状态**

---

## 📞 后续改进建议

### 短期 (1-2周)

- [ ] 补全未实现的交易执行函数
- [ ] 添加单元测试
- [ ] 完善错误处理
- [ ] 添加API文档（Swagger）

### 中期 (1-2月)

- [ ] 支持数据库迁移工具
- [ ] 添加性能监控
- [ ] 实现WebSocket推送（前端实时更新）
- [ ] 支持多语言

### 长期 (3-6月)

- [ ] 支持PostgreSQL/MySQL
- [ ] 添加回测功能
- [ ] 支持策略市场（分享提示词）
- [ ] 移动端App

---

**相关文档**:
- [交易功能变更](./trading/TRADING_CHANGES.md)
- [合并总结](./MERGE_SUMMARY.md)
