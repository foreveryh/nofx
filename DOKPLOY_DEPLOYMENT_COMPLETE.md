# NOFX - Dokploy 完整部署方案

## 🎯 架构概述

```
┌─────────────────────────────────────────────────────────┐
│  Dokploy + Traefik                                      │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  Frontend: app.yourdomain.com (Nginx - Static only)    │
│            ↓                                           │
│  Traefik: Port 80/443 → Frontend Container:80         │
│                                                         │
│  Backend: api.yourdomain.com (Go API)                 │
│            ↓                                           │
│  Traefik: Port 80/443 → Backend Container:8080        │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

**关键特点**:
- ✅ Frontend 直接调用 Backend API（不同域名）
- ✅ 无 Nginx proxy_pass，减少一层代理
- ✅ Traefik 在边缘路由，最佳实践
- ✅ Init 服务自动修复 config.json 目录问题

---

## 📋 必需文件

### 1. `docker-compose.dokploy.yml`

**文件路径**: `docker-compose.dokploy.yml`（已创建）

**特点**:
- 包含 `init` 服务，自动修复 config.json 目录问题
- Backend 使用 `:8080` 端口
- Frontend 使用 `:80` 端口（Nginx 只服务静态文件）
- 连接到 Dokploy 网络
- Traefik 标签自动配置域名和 SSL

### 2. `nginx/nginx.static.conf`

**文件路径**: `nginx/nginx.static.conf`（已创建）

**特点**:
- 不含 `proxy_pass` 到 Backend
- 只服务静态文件
- Frontend 通过 `NEXT_PUBLIC_API_URL` 直接调用 Backend API

### 3. `.env.example`

**文件路径**: `.env.example`（已创建）

包含所有必需的环境变量模板。

---

## 🔧 Dokploy 中部署步骤

### 步骤 1：创建项目

**登录 Dokploy → Projects → Create Project**

```yaml
Name: nofx-trading
Description: AI Multi-Model Trading System
Type: Docker Compose
```

### 步骤 2：配置环境变量

在 Dokploy 项目设置中，添加全局环境变量：

```bash
# ============================================================
# 1. 域名配置
# ============================================================
FRONTEND_DOMAIN=app.yourdomain.com
BACKEND_DOMAIN=api.yourdomain.com

# ============================================================
# 2. 基础配置
# ============================================================
NOFX_TIMEZONE=Asia/Shanghai
NODE_ENV=production

# ============================================================
# 3. AI 模型配置 (必需)
# 从 https://openrouter.ai/keys 获取
# ============================================================
CUSTOM_AI_ENDPOINT=https://openrouter.ai/api/v1#
CUSTOM_AI_API_KEY=sk-openrouter-your-api-key-here
CUSTOM_AI_MODEL=anthropic/claude-3.5-sonnet
AI_MAX_TOKENS=4000

# 备选模型：
# anthropic/claude-3.5-sonnet (推荐)
# openai/gpt-4o
# deepseek/deepseek-chat
# alibaba/qwen-max
# mistralai/mixtral-8x7b-instruct

# ============================================================
# 4. 币安交易所配置 (必需)
# 从 https://www.binance.com/en/my/settings/api-management
# 创建合约交易 API
# ============================================================
BINANCE_API_KEY=your-binance-api-key-here
BINANCE_SECRET_KEY=your-binance-secret-key-here
BINANCE_TESTNET=false
BINANCE_FUTURES=true
BTC_ETH_LEVERAGE=5
ALTCOIN_LEVERAGE=5

# ============================================================
# 5. 安全与加密 (自动生成更安全)
# ============================================================
# 数据加密密钥 (32字符)
# 生成命令: openssl rand -base64 32 | tr -d '=' | cut -c1-32
DATA_ENCRYPTION_KEY=aFnd63HmWtpgdcPar89ep8sIxGZIKke

# JWT 密钥 (base64格式)
# 生成命令: openssl rand -base64 64
JWT_SECRET=your-base64-jwt-secret-here

# ============================================================
# 6. 风险控制 (推荐设置)
# ============================================================
MAX_DAILY_LOSS=10.0          # 每日最大亏损 10%
MAX_DRAWDOWN=20.0           # 最大回撤 20%
STOP_TRADING_MINUTES=60     # 触发风控后暂停 60 分钟

# ============================================================
# 7. 默认交易币种
# ============================================================
USE_DEFAULT_COINS=true
DEFAULT_COINS=BTCUSDT,ETHUSDT,SOLUSDT,BNBUSDT,XRPUSDT,DOGEUSDT,ADAUSDT,HYPEUSDT

# ============================================================
# 8. 通知配置 (可选)
# 从 https://api.slack.com/messaging/webhooks 获取
# ============================================================
SLACK_WEBHOOK_URL=https://hooks.slack.com/services/YOUR/WEBHOOK/URL

# ============================================================
# 9. 系统配置 (通常不需要修改)
# ============================================================
ADMIN_MODE=true
LOG_LEVEL=info
HEALTH_CHECK_INTERVAL=30
HEALTH_CHECK_TIMEOUT=10
API_TIMEOUT=30s
WEBSOCKET_TIMEOUT=60s
```

**生成安全密钥（必须）**:
```bash
# 在服务器上执行:
openssl rand -base64 32 | tr -d '=' | cut -c1-32
# 输出示例: aFnd63HmWtpgdcPar89ep8sIxGZIKke

openssl rand -base64 64
# 输出示例: W3JzLWp3dC1zZWNyZXQta2V5LWZvci1zZWN1cml0eS1hbmQtYXV0aGVudGljYXRpb24=
```

### 步骤 3：创建 Backend 服务

**Services → Create → Application**

**基础信息**:
```yaml
Name: nofx-backend
Service Type: Docker Compose
Docker Compose Path: ./docker-compose.dokploy.yml
Service Name (in compose): nofx
```

**构建配置**:
```yaml
Repository: https://github.com/foreveryh/nofx
Branch: main
Build Path: /
Dockerfile Path: ./docker/Dockerfile.backend
```

**环境变量**: 使用步骤 2 中配置的全局变量

**部署配置**:
```yaml
Port: 8080
Restart Policy: unless-stopped
Auto Deploy: true
Replicas: 1
"Memory Reservation": 512M
"Memory Limit": 2G
"CPU Limit": 1.0
```

**卷挂载**:
```yaml
- 源路径: ./config.json
   目标路径: /app/config.json
   类型: Bind
   只读: true

- 源路径: ./config.db
   目标路径: /app/config.db
   类型: Bind
   只读: false

- 源路径: ./beta_codes.txt
   目标路径: /app/beta_codes.txt
   类型: Bind
   只读: true

- 源路径: ./decision_logs
   目标路径: /app/decision_logs
   类型: Bind
   只读: false

- 源路径: ./prompts
   目标路径: /app/prompts
   类型: Bind
   只读: true

- 源路径: ./secrets
   目标路径: /app/secrets
   类型: Bind
   只读: false

- 源路径: /etc/localtime
   目标路径: /etc/localtime
   类型: Bind
   只读: true
```

**健康检查**:
```yaml
Test: CMD curl -f http://localhost:8080/api/health
Interval: 30s
Timeout: 10s
Retries: 3
Start Period: 60s
```

**点击 Deploy** 开始部署

### 步骤 4：配置 Backend 域名和 SSL

**项目 → Settings → Domains → Add Domain**

```yaml
Domain: api.yourdomain.com
Service: nofx-backend
Port: 8080
HTTPS: true
Certificate: Auto-generated (Let's Encrypt)
```

**Traefik 会自动添加标签**:
```yaml
traefik.enable=true
traefik.http.routers.nofx-backend.rule=Host(`api.yourdomain.com`)
traefik.http.routers.nofx-backend.entrypoints=websecure
traefik.http.routers.nofx-backend.tls.certresolver=letsencrypt
traefik.http.services.nofx-backend.loadbalancer.server.port=8080
```

### 步骤 5：创建 Frontend 服务

**Services → Create → Application**

**基础信息**:
```yaml
Name: nofx-frontend
Service Type: Docker Compose
Docker Compose Path: ./docker-compose.dokploy.yml
Service Name (in compose): nofx-frontend
```

**构建配置**:
```yaml
Repository: https://github.com/foreveryh/nofx
Branch: main
Build Path: /
Dockerfile Path: ./docker/Dockerfile.frontend
```

**环境变量**:
```bash
# API 地址（指向 Backend 域名）
NEXT_PUBLIC_API_URL=https://api.yourdomain.com/api
NEXT_PUBLIC_WS_URL=wss://api.yourdomain.com/ws
NEXT_PUBLIC_DOMAIN=app.yourdomain.com

# Build arguments
NODE_ENV=production
SKIP_ENV_VALIDATION=true
```

**部署配置**:
```yaml
Port: 80
Restart Policy: unless-stopped
Auto Deploy: true
Replicas: 1
"Memory Reservation": 256M
"Memory Limit": 1G
```

**卷挂载**:
```yaml
- 源路径: ./nginx/nginx.static.conf
   目标路径: /etc/nginx/conf.d/default.conf
   类型: Bind
   只读: true

- 源路径: /etc/localtime
   目标路径: /etc/localtime
   类型: Bind
   只读: true
```

**健康检查**:
```yaml
Test: CMD wget --no-verbose --tries=1 --spider http://localhost/health
Interval: 30s
Timeout: 10s
Retries: 3
Start Period: 5s
```

**点击 Deploy** 开始部署

### 步骤 6：配置 Frontend 域名和 SSL

**项目 → Settings → Domains → Add Domain**

```yaml
Domain: app.yourdomain.com
Service: nofx-frontend
Port: 80
HTTPS: true
Certificate: Auto-generated (Let's Encrypt)
```

### 步骤 7：验证部署

**1. 检查 Backend 健康**:
```bash
curl https://api.yourdomain.com/api/health
# 应该返回: {"status":"ok"}
```

**2. 检查 Frontend**:
```bash
curl https://app.yourdomain.com
# 应该返回 HTML 页面
```

**3. 查看 Dokploy 日志**:
- **Backend 日志**: Dokploy → Services → nofx-backend → Logs
- **Frontend 日志**: Dokploy → Services → nofx-frontend → Logs

**4. 浏览器访问**:
```
https://app.yourdomain.com
```

**首次登录**:
- 使用默认管理员登录
- Username: `admin@localhost`
- Password: 任意（admin_mode=true 会绕过验证）

---

## 🔧 故障排查

### 问题 1: config.json is a directory

**症状**:
```
❌ 读取config.json失败: read config.json: is a directory
```

**原因**: Dokploy 首次部署时，Git 中没有 config.json，Docker 创建为目录

**解决方案**:
- ✅ 已解决：docker-compose.dokploy.yml 包含 init 服务
- 自动在启动前创建正确的 config.json 文件

### 问题 2: Frontend 无法访问 Backend API

**症状**:
```
Error: Failed to fetch
CORS error
```

**原因**: Frontend 调用 API 时跨域

**解决方案**:
- ✅ 确保环境变量正确：
  - Frontend: NEXT_PUBLIC_API_URL=https://api.yourdomain.com/api
  - Backend: Dokploy 自动配置 CORS

### 问题 3: WebSocket 连接失败

**症状**:
```
WebSocket connection failed
```

**解决方案**:
```bash
# 检查 Backend 是否正在运行
docker logs dokploy-nofx-backend-xxxxx --tail=20

# 确保 NEXT_PUBLIC_WS_URL 正确
NEXT_PUBLIC_WS_URL=wss://api.yourdomain.com/ws
```

### 问题 4: 加密服务初始化失败

**症状**:
```
❌ 初始化加密服务失败: DATA_ENCRYPTION_KEY not set
```

**解决方案**:
- ✅ 确保 DATA_ENCRYPTION_KEY 已设置
- ✅ 确保 secrets 目录可写（docker-compose 已配置）

---

## 📊 完整环境变量参考

### Backend 环境变量 (Docker Compose)

```bash
# 1. 基础配置
TZ=Asia/Shanghai
NODE_ENV=production
AI_MAX_TOKENS=4000
ADMIN_MODE=true

# 2. AI 模型 (OpenRouter)
CUSTOM_AI_ENDPOINT=https://openrouter.ai/api/v1#
CUSTOM_AI_API_KEY=sk-openrouter-xxx
CUSTOM_AI_MODEL=anthropic/claude-3.5-sonnet

# 3. 币安交易所
BINANCE_API_KEY=xxx
BINANCE_SECRET_KEY=xxx
BINANCE_TESTNET=false
BINANCE_FUTURES=true
BTC_ETH_LEVERAGE=5
ALTCOIN_LEVERAGE=5

# 4. 安全
DATA_ENCRYPTION_KEY=xxx
JWT_SECRET=xxx

# 5. 风控
MAX_DAILY_LOSS=10.0
MAX_DRAWDOWN=20.0
STOP_TRADING_MINUTES=60

# 6. 币种
USE_DEFAULT_COINS=true
DEFAULT_COINS=BTCUSDT,ETHUSDT,SOLUSDT,BNBUSDT,XRPUSDT,DOGEUSDT,ADAUSDT,HYPEUSDT

# 7. 通知
SLACK_WEBHOOK_URL=https://hooks.slack.com/services/xxx
```

### Frontend 环境变量 (Docker Compose)

```bash
# API 地址（Backend 域名）
NEXT_PUBLIC_API_URL=https://api.yourdomain.com/api
NEXT_PUBLIC_WS_URL=wss://api.yourdomain.com/ws
NEXT_PUBLIC_DOMAIN=app.yourdomain.com

# Build
NODE_ENV=production
SKIP_ENV_VALIDATION=true
```

---

## 🚀 生产环境优化建议

### 1. 使用非 root 用户运行容器

修改 Dockerfile.backend:
```dockerfile
# Add user
RUN addgroup -g 1001 -S nofx && \
    adduser -S nofx -u 1001

USER nofx
```

### 2. 配置资源限制

Dokploy 资源限制:
- **Backend**: 2GB RAM, 1 CPU
- **Frontend**: 1GB RAM, 0.5 CPU

### 3. 启用自动备份

备份 config.db:
```bash
#!/bin/bash
# backup.sh
DATE=$(date +%Y%m%d_%H%M%S)
cp /home/user/nofx-trading/config.db /backup/nofx/db/config.db.$DATE.gz
gzip /backup/nofx/db/config.db.$DATE
```

### 4. 监控告警

在 Dokploy 中配置健康检查失败告警。

### 5. 使用 GitHub Actions 自动部署

```yaml
# .github/workflows/deploy.yml
name: Deploy to Dokploy
on:
  push:
    branches: [ main ]
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - name: Deploy to Dokploy
        run: |
          # Trigger Dokploy webhook
          curl -X POST ${{ secrets.DOKPLOY_WEBHOOK }}
```

---

## 📝 总结

### 已创建的文件

1. ✅ `docker-compose.dokploy.yml` - 完整 Dokploy 配置
2. ✅ `nginx/nginx.static.conf` - 静态 Nginx 配置（无 proxy）
3. ✅ `docker/Dockerfile.frontend` - 使用静态 Nginx 配置
4. ✅ `web/src/lib/api.ts` - 从环境变量读取 API 地址

### 部署步骤

1. ✅ 配置 Dokploy 全局环境变量
2. ✅ 创建 Backend 服务（domain: api.yourdomain.com）
3. ✅ 创建 Frontend 服务（domain: app.yourdomain.com）
4. ✅ 验证部署

### 架构优势

- ✅ Init 服务自动修复 config.json 目录问题
- ✅ Frontend 直接调用 Backend API（不同域名）
- ✅ 无 Nginx proxy_pass，性能更好
- ✅ 充分利用 Dokploy + Traefik
- ✅ 安全：所有密钥在环境变量中

**🎉 你的系统现在已准备好 Dokploy 生产部署！**