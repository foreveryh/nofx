# Dokploy 环境变量快速参考

## 快速开始（3步）

### 步骤 1：生成密钥
```bash
# 数据加密密钥 (32字符)
openssl rand -base64 32 | tr -d '=' | cut -c1-32
# 示例: aFnd63HmWtpgdcPar89ep8sIxGZIKke

# JWT 密钥 (64字符 base64)
openssl rand -base64 64
# 示例: W3JzLWp3dC1zZWNyZXQta2V5LWZvci1zZWN1cml0eS1hbmQtYXV0aGVudGljYXRpb24=
```

### 步骤 2：配置必需变量

**必需（无默认值）**:
- `CUSTOM_AI_API_KEY`: OpenRouter API 密钥
- `BINANCE_API_KEY`: 币安 API 密钥
- `BINANCE_SECRET_KEY`: 币安密钥
- `DATA_ENCRYPTION_KEY`: 32字符加密密钥
- `JWT_SECRET`: Base64 JWT 密钥

**推荐设置**:
- `FRONTEND_DOMAIN`: app.yourdomain.com
- `BACKEND_DOMAIN`: api.yourdomain.com
- `MAX_DAILY_LOSS`: 10.0
- `MAX_DRAWDOWN`: 20.0

### 步骤 3：在 Dokploy 中部署

使用 `docker-compose.dokploy.yml`，配置全局环境变量后部署。

---

## 环境变量完整列表

### 1. 域名配置 (必需)
```bash
FRONTEND_DOMAIN=app.yourdomain.com      # 前端域名
BACKEND_DOMAIN=api.yourdomain.com        # 后端 API 域名
```

### 2. AI 模型配置 (必需)
```bash
CUSTOM_AI_ENDPOINT=https://openrouter.ai/api/v1#
CUSTOM_AI_API_KEY=sk-openrouter-xxxxxxxxxx
CUSTOM_AI_MODEL=anthropic/claude-3.5-sonnet
AI_MAX_TOKENS=4000
```

**推荐模型**:
- `anthropic/claude-3.5-sonnet` (最佳性能)
- `openai/gpt-4o` (快速响应)
- `deepseek/deepseek-chat` (性价比高)
- `alibaba/qwen-max` (中文优化)

### 3. 币安交易所 (必需)
```bash
BINANCE_API_KEY=your_binance_api_key_here
BINANCE_SECRET_KEY=your_binance_secret_key_here
BINANCE_TESTNET=false                # true=测试网, false=主网
BINANCE_FUTURES=true                 # 合约交易
BTC_ETH_LEVERAGE=5                   # BTC/ETH 杠杆
ALTCOIN_LEVERAGE=5                   # 山寨币杠杆
```

### 4. 安全与加密 (必需)
```bash
DATA_ENCRYPTION_KEY=32_char_key_here    # openssl rand -base64 32 | tr -d '=' | cut -c1-32
JWT_SECRET=base64_jwt_secret_here       # openssl rand -base64 64
ADMIN_MODE=true                        # 管理员模式（无需登录）
```

### 5. 风控设置 (推荐)
```bash
MAX_DAILY_LOSS=10.0         # 每日最大亏损百分比
MAX_DRAWDOWN=20.0           # 最大回撤百分比
STOP_TRADING_MINUTES=60     # 触发风控后暂停时间
```

### 6. 默认交易币种
```bash
USE_DEFAULT_COINS=true
DEFAULT_COINS=BTCUSDT,ETHUSDT,SOLUSDT,BNBUSDT,XRPUSDT,DOGEUSDT,ADAUSDT,HYPEUSDT
```

### 7. 通知配置 (可选)
```bash
SLACK_WEBHOOK_URL=https://hooks.slack.com/services/YOUR/WEBHOOK/URL
```

### 8. 系统配置 (通常不改)
```bash
TZ=Asia/Shanghai
AI_MAX_TOKENS=4000
LOG_LEVEL=info
HEALTH_CHECK_INTERVAL=30
HEALTH_CHECK_TIMEOUT=10
API_TIMEOUT=30s
WEBSOCKET_TIMEOUT=60s
NODE_ENV=production
```

### 9. 前端构建变量
```bash
NEXT_PUBLIC_API_URL=https://api.yourdomain.com/api
NEXT_PUBLIC_WS_URL=wss://api.yourdomain.com/ws
NEXT_PUBLIC_DOMAIN=app.yourdomain.com
SKIP_ENV_VALIDATION=true
```

---

## Dokploy 服务配置

### Backend 服务 (nofx-backend)

**基础**:
```yaml
Service Type: Docker Compose
Docker Compose Path: ./docker-compose.dokploy.yml
Service Name: nofx
```

**端口**: 8080

**域名**: api.yourdomain.com

**卷挂载**:
```yaml
- ./config.json:/app/config.json:ro
- ./config.db:/app/config.db
- ./beta_codes.txt:/app/beta_codes.txt:ro
- ./decision_logs:/app/decision_logs
- ./prompts:/app/prompts:ro
- ./secrets:/app/secrets
- /etc/localtime:/etc/localtime:ro
```

**健康检查**:
```yaml
Test: CMD curl -f http://localhost:8080/api/health
Interval: 30s
Timeout: 10s
Retries: 3
Start Period: 60s
```

### Frontend 服务 (nofx-frontend)

**基础**:
```yaml
Service Type: Docker Compose
Docker Compose Path: ./docker-compose.dokploy.yml
Service Name: nofx-frontend
```

**端口**: 80

**域名**: app.yourdomain.com

**卷挂载**:
```yaml
- ./nginx/nginx.static.conf:/etc/nginx/conf.d/default.conf:ro
- /etc/localtime:/etc/localtime:ro
```

**健康检查**:
```yaml
Test: CMD wget --no-verbose --tries=1 --spider http://localhost/health
Interval: 30s
Timeout: 10s
Retries: 3
Start Period: 5s
```

---

## 成本估算

### AI 模型费用 (每月)

基于每 5 分钟 1 次决策:

| 模型 | 费用/月 | 推荐度 |
|------|---------|--------|
| Claude 3.5 Sonnet | ~$45 | ⭐⭐⭐⭐⭐ |
| GPT-4o | ~$90 | ⭐⭐⭐⭐ |
| DeepSeek V2.5 | ~$12 | ⭐⭐⭐⭐ |
| Qwen-Max | ~$180 | ⭐⭐ |

**建议组合**: Claude 3.5 + DeepSeek (~$57/月)

### 服务器费用

- **VPS**: 4核8GB ~$20-40/月
- **Dokploy**: 免费
- **Traefik**: 免费 (自带)

### 交易所费用

- 币安手续费: 0.02% - 0.04%
- 假设月交易量 $100万: ~$200-400/月

### 总计

```
低配置（Claude + DeepSeek）：$60/月 + 交易所费用
中配置（5个模型）：$350/月 + 交易所费用
```

---

## 故障排查速查

### 1. config.json is a directory
```bash
# 在服务器执行:
rm -rf config.json
touch config.json
```

### 2. Frontend 404 on API calls
```bash
# 检查 Frontend 环境变量:
NEXT_PUBLIC_API_URL=https://api.yourdomain.com/api
```

### 3. Database locked
```bash
# SQLite WAL 模式支持多连接
# 会自动重试，无需手动处理
```

### 4. API key invalid
```bash
# 检查加密服务初始化
# 确保 DATA_ENCRYPTION_KEY 设置正确
```

---

## 安全建议

### 生产环境必须修改

1. **JWT_SECRET**: 使用 openssl rand -base64 64 生成
2. **DATA_ENCRYPTION_KEY**: 使用 openssl rand -base64 32 | tr -d '=' | cut -c1-32 生成
3. **Disable ADMIN_MODE**: 设置为 false，启用完整认证
4. **Enable registration**: 如果需要多用户

### 敏感文件保护

```bash
# 这些文件不应在 Git 中
.env
config.db*
beta_codes.txt
secrets/
```

### API 密钥轮换

- OpenRouter: 可随时在控制台撤销和重新生成
- Binance: 启用 IP 白名单，限制访问

---

## 性能优化

### Backend

- `AI_MAX_TOKENS=4000`: 平衡响应时间和内容长度
- `AI_MAX_TOKENS=8000`: 如果 AI 响应被截断
- Funding Rate 缓存: 已启用，减少 90% API 调用

### Frontend

- 静态资源缓存: 1年 (js/css/images)
- Gzip 压缩: 已启用
- 健康检查: 30秒间隔

### Database

- WAL 模式: 已启用，支持并发读写
- FULL 同步: 已启用，保证数据持久化

---

## 监控指标

### 关键指标

- **API 健康**: https://api.yourdomain.com/api/health
- **Frontend 健康**: https://app.yourdomain.com/health
- **决策日志**: ./decision_logs/
- **数据库**: ./config.db

### Dokploy 监控

- CPU 使用率: < 50%
- 内存使用: Backend < 2GB, Frontend < 1GB
- 重启次数: 应该很少
- 健康检查通过率: 100%

---

## 快速验证清单

部署后检查:

- [ ] Backend 健康: curl https://api.yourdomain.com/api/health
- [ ] Frontend 访问: https://app.yourdomain.com
- [ ] WebSocket 连接: wss://api.yourdomain.com/ws
- [ ] AI 模型响应: 正常
- [ ] 币安 API: 连接成功
- [ ] 决策日志: 有输出
- [ ] 持仓显示: 准确
- [ ] 余额同步: 自动更新

---

**🎉 准备开始交易！**
