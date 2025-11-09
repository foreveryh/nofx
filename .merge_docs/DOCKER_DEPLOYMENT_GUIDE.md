# NOFX AI交易系统 Docker部署指南

**版本**: v2.0 (包含动态止盈止损功能)
**更新日期**: 2025-11-02

---

## 📋 目录

1. [系统要求](#系统要求)
2. [快速部署](#快速部署)
3. [配置详解](#配置详解)
4. [必要KEY配置](#必要key配置)
5. [交易所配置](#交易所配置)
6. [AI模型配置](#ai模型配置)
7. [部署验证](#部署验证)
8. [常见问题](#常见问题)
9. [监控与日志](#监控与日志)

---

## 🔧 系统要求

### 最低要求
- **操作系统**: Linux, macOS, Windows (支持Docker)
- **Docker**: >= 20.10.0
- **Docker Compose**: >= 2.0.0
- **内存**: >= 2GB RAM
- **存储**: >= 5GB 可用空间
- **网络**: 稳定的互联网连接

### 推荐配置
- **内存**: >= 4GB RAM
- **CPU**: >= 2核
- **存储**: >= 10GB SSD
- **网络**: 低延迟（特别是交易API）

---

## 🚀 快速部署

### 1. 克隆项目

```bash
git clone https://github.com/NoFxAiOS/nofx.git
cd nofx
```

### 2. 配置环境变量

```bash
# 复制环境变量模板
cp .env.example .env

# 复制配置文件模板
cp config.json.example config.json

# 编辑配置文件（详细配置见下文）
nano .env
nano config.json
```

### 3. 一键部署

```bash
# 使用提供的脚本启动
./start.sh start --build

# 或者使用docker-compose
docker-compose up -d --build
```

### 4. 访问应用

- **前端界面**: http://localhost:3000
- **后端API**: http://localhost:8080
- **健康检查**: http://localhost:8080/api/health

---

## ⚙️ 配置详解

### 1. 环境变量 (.env)

```bash
# 端口配置
NOFX_BACKEND_PORT=8080          # 后端API端口
NOFX_FRONTEND_PORT=3000         # 前端访问端口

# 时区设置
NOFX_TIMEZONE=Asia/Shanghai     # 系统时区
```

### 2. 主配置文件 (config.json)

```json
{
  "admin_mode": true,                    // 是否启用管理员模式
  "leverage": {
    "btc_eth_leverage": 5,              // BTC/ETH杠杆倍数
    "altcoin_leverage": 5               // 其他币种杠杆倍数
  },
  "use_default_coins": true,            // 是否使用默认币种列表
  "default_coins": [                    // 默认交易币种
    "BTCUSDT", "ETHUSDT", "SOLUSDT",
    "BNBUSDT", "XRPUSDT", "DOGEUSDT",
    "ADAUSDT", "HYPEUSDT"
  ],
  "coin_pool_api_url": "",              // 币种池API（可选）
  "oi_top_api_url": "",                 // 持仓量API（可选）
  "api_server_port": 8080,              // API服务器端口
  "max_daily_loss": 10.0,               // 最大日亏损百分比
  "max_drawdown": 20.0,                 // 最大回撤百分比
  "stop_trading_minutes": 60,           // 止损后停止交易分钟数
  "jwt_secret": "your-jwt-secret-key"   // JWT密钥（必须修改！）
}
```

---

## 🔑 必要KEY配置

### 1. JWT Secret (必须配置)

**位置**: `config.json` 中的 `jwt_secret`

**重要性**: 🔴 **Critical**
- 用于用户认证和数据加密
- **必须修改默认值！**

**生成方法**:
```bash
# 方法1: 使用openssl生成
openssl rand -base64 64

# 方法2: 使用Python生成
python3 -c "import secrets; print(secrets.token_urlsafe(64))"

# 方法3: 使用Node.js生成
node -e "console.log(require('crypto').randomBytes(64).toString('base64'))"
```

**示例**:
```json
{
  "jwt_secret": "Qk0kAa+d0iIEzXVHXbNbm+UaN3RNabmWtH8rDWZ5OPf+4GX8pBflAHodfpbipVMyrw1fsDanHsNBjhgbDeK9Jg=="
}
```

### 2. 交易所API Keys

需要配置至少一个交易所的API。**请务必只授予交易权限，不要提币权限！**

#### Binance Futures配置

**1. 创建API Key**
- 登录 [Binance](https://www.binance.com)
- 进入 API管理 → 创建API
- **权限设置**:
  - ✅ 现货与杠杆交易
  - ✅ 合约交易
  - ❌ 提现（不要开启！）
  - ❌ 内部转账（不要开启！）

**2. 环境变量** (可选，也可通过Web界面配置)
```bash
# 可选：通过环境变量设置Binance Keys
BINANCE_API_KEY=your_binance_api_key
BINANCE_SECRET_KEY=your_binance_secret_key
```

**3. 限制设置** (强烈建议)
- **IP白名单**: 添加您的服务器IP
- **交易权限**: 仅启用期货交易
- **提现权限**: 必须关闭

#### Hyperliquid配置

**1. 获取Private Key**
- 访问 [Hyperliquid](https://hyperliquid.xyz/)
- 备份您的钱包私钥
- **建议**: 创建专门的交易账户

**2. 环境变量** (可选)
```bash
# 可选：通过环境变量设置Hyperliquid Private Key
HYPERLIQUID_PRIVATE_KEY=your_hyperliquid_private_key
```

#### Aster DEX配置

**1. 准备钱包**
- 安装MetaMask或类似钱包
- 准备Aster链上的私钥
- **建议**: 使用专门的交易钱包

**2. 环境变量** (可选)
```bash
# 可选：通过环境变量设置Aster Private Key
ASTER_PRIVATE_KEY=your_aster_private_key
```

---

## 🤖 AI模型配置

### 1. DashScope (阿里云)

**适用**: 国内用户

```bash
# 环境变量配置
DASHSCOPE_API_KEY=your_dashscope_api_key

# 支持的模型
- qwen-plus (推荐)
- qwen-turbo
- qwen-max
```

**获取API Key**:
1. 访问 [阿里云DashScope控制台](https://dashscope.console.aliyun.com/)
2. 开通服务并创建API Key
3. 配置到环境变量或Web界面

### 2. OpenAI

**适用**: 海外用户

```bash
# 环境变量配置
OPENAI_API_KEY=your_openai_api_key
OPENAI_BASE_URL=https://api.openai.com/v1  # 可选：自定义代理

# 支持的模型
- gpt-4 (推荐)
- gpt-4-turbo
- gpt-3.5-turbo
```

### 3. 自定义模型

可通过Web界面配置自定义的AI模型：
- 支持OpenAI兼容的API
- 可调整temperature、max_tokens等参数
- 支持自定义system prompt

---

## 📊 Web界面配置

### 1. 首次访问

1. 打开 http://localhost:3000
2. 注册管理员账户
3. 登录后进入系统设置

### 2. 交易所配置

通过Web界面配置交易所API：

```
交易设置 → 交易所管理 → 添加交易所

示例配置：
- 交易所: Binance Futures
- API Key: sk_xxxxxxxxxxxx
- Secret Key: xxxxxxxxxxxx
- 测试模式: 建议先开启
- 环境: 选择Production（生产）或Testnet（测试）
```

### 3. 交易员配置

```
交易设置 → 交易员管理 → 创建交易员

示例配置：
- 名称: AI交易员001
- 初始资金: 1000 USDT
- 杠杆: 5x
- AI模型: qwen-plus
- 交易币种: BTCUSDT, ETHUSDT
- 风险控制: 启用
```

### 4. 策略模板选择

系统提供多种策略模板：

1. **Default (默认)**
   - 保守策略，适合新手
   - 严格的风险控制
   - 推荐初始使用

2. **Adaptive (自适应)**
   - 双策略系统（震荡+趋势）
   - 动态调整
   - 包含动态TP/SL功能

3. **Nof1 (单一策略)**
   - 专注单一策略
   - 高频交易
   - 适合经验丰富的用户

---

## 🔍 部署验证

### 1. 检查服务状态

```bash
# 检查容器状态
docker-compose ps

# 预期输出：
# NAME              COMMAND                  SERVICE             STATUS              PORTS
# nofx-trading      "./nofx"                 nofx                running             0.0.0.0:8080->8080/tcp
# nofx-frontend     "nginx -g 'daemon of…"   nofx-frontend       running             0.0.0.0:3000->80/tcp
```

### 2. 检查健康状态

```bash
# 检查后端健康
curl http://localhost:8080/api/health

# 预期输出：
# {
#   "status": "healthy",
#   "timestamp": "2025-11-02T10:00:00Z",
#   "uptime": "5m30s",
#   "version": "v2.0"
# }
```

### 3. 检查日志

```bash
# 查看后端日志
docker-compose logs -f nofx

# 查看前端日志
docker-compose logs -f nofx-frontend
```

### 4. 功能测试

1. **Web界面测试**
   - 访问 http://localhost:3000
   - 检查页面是否正常加载
   - 测试登录功能

2. **API测试**
   ```bash
   # 测试API端点
   curl http://localhost:8080/api/v1/health
   curl http://localhost:8080/api/v1/status
   ```

3. **交易功能测试**
   - 在测试网环境中创建交易员
   - 执行一次小额测试交易
   - 验证TP/SL功能是否正常

---

## ❓ 常见问题

### Q1: 容器启动失败

**问题**: Docker容器无法启动

**解决方案**:
```bash
# 1. 检查Docker和docker-compose版本
docker --version
docker-compose --version

# 2. 检查端口是否被占用
netstat -tulpn | grep :8080
netstat -tulpn | grep :3000

# 3. 查看详细错误日志
docker-compose logs nofx

# 4. 重新构建
docker-compose down
docker-compose up -d --build
```

### Q2: API Key配置错误

**问题**: 交易所连接失败

**解决方案**:
1. **检查API Key权限**
   - 确保启用了期货交易权限
   - 确认IP白名单设置

2. **验证Key有效性**
   ```bash
   # 使用Binance CLI测试
   curl -X GET "https://fapi.binance.com/fapi/v1/ping"
   ```

3. **检查网络连接**
   ```bash
   # 测试API连通性
   curl -X GET "https://fapi.binance.com/fapi/v1/time"
   ```

### Q3: AI模型调用失败

**问题**: AI决策失败

**解决方案**:
1. **检查API Key**
   ```bash
   # 测试DashScope API
   curl -X POST "https://dashscope.aliyuncs.com/api/v1/services/aigc/text-generation/generation" \
     -H "Authorization: Bearer YOUR_API_KEY" \
     -H "Content-Type: application/json" \
     -d '{"model":"qwen-plus","input":{"messages":[{"role":"user","content":"Hello"}]}}'
   ```

2. **检查模型配置**
   - 确认模型名称正确
   - 检查API调用限额

3. **查看详细错误**
   ```bash
   # 查看决策日志
   tail -f decision_logs/*.json
   ```

### Q4: 数据库初始化失败

**问题**: 首次启动时数据库错误

**解决方案**:
```bash
# 1. 确保数据库文件存在
touch config.db

# 2. 检查文件权限
chmod 666 config.db

# 3. 重新初始化
docker-compose down
docker-compose up -d
```

### Q5: 前端页面无法访问

**问题**: 404错误或页面加载失败

**解决方案**:
```bash
# 1. 检查前端容器状态
docker-compose ps nofx-frontend

# 2. 重建前端镜像
docker-compose build nofx-frontend

# 3. 检查Nginx配置
docker exec nofx-frontend nginx -t
```

---

## 📊 监控与日志

### 1. 系统监控

```bash
# 实时查看容器资源使用
docker stats

# 查看容器健康状态
docker-compose ps
```

### 2. 日志管理

```bash
# 查看实时日志
docker-compose logs -f

# 查看特定时间段日志
docker-compose logs --since="2025-11-02T10:00:00" --until="2025-11-02T11:00:00"

# 日志轮转配置
# 在docker-compose.yml中添加：
logging:
  driver: "json-file"
  options:
    max-size: "10m"
    max-file: "3"
```

### 3. 性能监控

```bash
# 查看API响应时间
curl -w "@curl-format.txt" -s -o /dev/null http://localhost:8080/api/health

# curl-format.txt内容：
#      time_namelookup:  %{time_namelookup}\n
#         time_connect:  %{time_connect}\n
#      time_appconnect:  %{time_appconnect}\n
#     time_pretransfer:  %{time_pretransfer}\n
#        time_redirect:  %{time_redirect}\n
#   time_starttransfer:  %{time_starttransfer}\n
#                      ----------\n
#           time_total:  %{time_total}\n
```

### 4. 数据备份

```bash
# 备份重要数据
tar -czf backup-$(date +%Y%m%d).tar.gz \
  config.db \
  decision_logs/ \
  config.json \
  .env

# 恢复数据
tar -xzf backup-20251102.tar.gz
```

---

## 🔄 更新与维护

### 1. 更新应用

```bash
# 1. 备份数据
./start.sh backup

# 2. 拉取最新代码
git pull origin main

# 3. 重新构建和部署
./start.sh start --build
```

### 2. 定期维护

```bash
# 清理Docker资源
docker system prune -f

# 更新Docker镜像
docker-compose pull

# 重启服务
docker-compose restart
```

### 3. 安全检查

```bash
# 检查API Key安全性
# 1. 确保不在代码中硬编码API Key
# 2. 定期轮换API Key
# 3. 监控API使用情况

# 检查端口暴露
# 只暴露必要的端口（8080, 3000）
```

---

## 📞 技术支持

### 问题反馈

1. **GitHub Issues**: [提交Issue](https://github.com/NoFxAiOS/nofx/issues)
2. **社区讨论**: [Discussions](https://github.com/NoFxAiOS/nofx/discussions)

### 常用命令参考

```bash
# 启动服务
./start.sh start

# 停止服务
./start.sh stop

# 重启服务
./start.sh restart

# 查看状态
./start.sh status

# 查看日志
./start.sh logs

# 清理数据
./start.sh clean

# 更新应用
./start.sh update
```

---

## ⚠️ 重要提醒

### 安全建议

1. **🔐 API Key安全**
   - 绝不要提交API Key到代码仓库
   - 使用环境变量或安全的密钥管理
   - 定期轮换API Key

2. **🛡️ 防火墙设置**
   - 只开放必要的端口（80, 443, 8080）
   - 使用反向代理（如Nginx）
   - 启用HTTPS

3. **💾 数据备份**
   - 定期备份config.db
   - 备份决策日志
   - 备份配置文件

### 风险控制

1. **💰 资金管理**
   - 从小额开始测试
   - 设置合理的止损
   - 不要投入全部资金

2. **📈 策略测试**
   - 先在测试网验证
   - 监控决策质量
   - 及时调整参数

3. **⚡ 系统监控**
   - 监控API延迟
   - 监控系统资源
   - 设置告警机制

---

**文档版本**: v2.0
**最后更新**: 2025-11-02
**适用版本**: NOFX v2.0+ (包含动态TP/SL功能)

**相关文档**:
- [Docker Compose配置](./docker-compose.yml)
- [配置文件模板](./config.json.example)
- [环境变量模板](./.env.example)
- [快速启动脚本](./start.sh)