# NOFX AI交易系统 - Dokploy部署指南

**域名**: nofx.deeptoai.com
**认证方式**: Cloudflare Zero Trust
**环境变量文件**: .env.dokploy.complete

---

## 📝 部署概述

Dokploy是一款轻量级PaaS平台，可以简化Docker应用的部署和管理。本指南将指导您如何通过Dokploy部署NOFX AI交易系统。

**架构**:
```
互联网
  ↓
Cloudflare (DNS + Zero Trust认证)
  ↓
Dokploy服务器
  ↓
Docker容器
  ├─ backend (Go应用, 端口8080)
  └─ frontend (Nginx, 端口80/443)
```

---

## 📋 前置条件

### 1. 服务器准备
- [ ] VPS服务器 (推荐: 4核4GB内存, 100GB SSD)
- [ ] 操作系统: Ubuntu 22.04 LTS 或 Debian 12
- [ ] 已安装Docker和Docker Compose
- [ ] 服务器已配置SSH密钥登录

### 2. Dokploy安装
```bash
# Dokploy官方安装脚本
curl -sSL https://dokploy.com/install.sh | sh

# 查看安装日志
tail -f /var/log/dokploy/install.log

# 访问Dokploy面板
# http://your-server-ip:3000
# 默认用户名: admin
# 默认密码: admin123

# 首次登录后请立即修改密码!
```

### 3. Cloudflare准备
- [ ] Cloudflare账号
- [ ] 域名: `deeptoai.com` 已接入Cloudflare
- [ ] DNS记录:
  - Type: A
  - Name: nofx
  - Content: [您的VPS IP]
  - Proxy: ✅ (橙色云)
  - TTL: Auto

---

## 🔐 Cloudflare Zero Trust配置

### 阶段1: 基础配置

**步骤1: 启用Zero Trust**

1. 登录Cloudflare Dashboard
2. 左侧菜单 → Zero Trust
3. 点击"Get started"
4. 选择免费计划 (50个用户以内)
5. 添加支付方式 (不会扣费，仅验证)

**步骤2: 配置身份源**

1. **Settings** → **Authentication** → **Login methods**
2. 添加登录方式:
   - **One-time PIN**: 通过邮箱验证码登录 (推荐初次配置)
   - 或 **GitHub** / **Google OAuth**: 如果你已有OAuth应用
3. 记下**Team Domain**: `https://your-team.cloudflareaccess.com`

**步骤3: 创建策略**

1. **Access** → **Applications**
2. 点击 **"Add an application"**
3. 选择 **"Self-hosted"**

### 阶段2: 应用配置

**应用基本信息**:
```yaml
Application Name: NOFX Trading System
Application Domain: nofx.deeptoai.com
Session Duration: 24h  # 登录后24小时内免验证
```

**策略配置**:

**策略1**: 仅允许你的邮箱
```yaml
Policy Name: Admin Access Only
Action: Allow

Rules:
  - Rule Type: Emails
    Value: your-email@example.com  # 替换成你的邮箱

# 可选: 要求MFA
Require:
  - Presence of device certificate
```

**策略2**: 允许你的IP段 (可选)
```yaml
Policy Name: Home IP Access
Action: Allow

Rules:
  - Rule Type: IP Range
    Value: x.x.x.x/32  # 你的公网IP
```

**策略3**: 阻止所有其他访问
```yaml
Policy Name: Block Others
Action: Block

Rules:
  - Rule Type: Everyone
```

**应用设置** (高级):
- [x] Enable automatic cloudflared authentication
- [x] Enable binding cookie
- [x] Enable CORS headers
- [x] Enable automatic HTTP->HTTPS redirect

### 阶段3: 验证配置

访问 `https://nofx.deeptoai.com`
- 应该会跳转到Cloudflare登录页面
- 输入你的邮箱
- 收取PIN码
- 验证通过后会自动跳转回应用

**调试工具**:
- Cloudflare Dashboard → **Access** → **Audit Logs** (查看认证日志)

---

## 🚀 Dokploy部署步骤

### 步骤1: 创建项目

1. 登录Dokploy面板
2. **Projects** → **Create Project**
3. 项目名称: `nofx-trading`
4. 描述: `AI Powered Trading System`

### 步骤2: 添加后端服务 (Backend)

**服务类型**: **Application**

**基本信息**:
```yaml
Name: backend
Repository: https://github.com/foreveryh/nofx
Branch: main
Build Path: /
Dockerfile Path: ./docker/Dockerfile.backend
```

**环境变量**:
1. 点击 **"Import from .env file"**
2. 上传 `.env.dokploy.complete` 文件
3. 或者手动添加所有变量（推荐导入）

**必需变量** (必须修改):
```bash
# JWT密钥 (生成新的)
JWT_SECRET=generate_new_key_here  # openssl rand -base64 64

# 交易所API (至少配置一个)
BINANCE_API_KEY=your_binance_key
BINANCE_SECRET_KEY=your_binance_secret

# AI模型 (至少配置一个)
DEEPSEEK_API_KEY=your_deepseek_key
# 或 QWEN_API_KEY=your_qwen_key
```

**部署配置**:
```yaml
Port: 8080
Restart Policy: unless-stopped
Auto Deploy: true  # 推送到Git时自动部署
Replicas: 1  # 单实例
```

**资源限制**:
```yaml
Memory: 2G
CPU: 1.0
```

**Volumes** (Dokploy会自动创建):
```yaml
Mount Path: /app/data
Type: Volume
Name: backend_data

Mount Path: /app/logs
Type: Volume
Name: backend_logs

Mount Path: /app/config
Type: Volume
Name: backend_config

Mount Path: /app/decision_logs
Type: Volume
Name: backend_decision_logs

Mount Path: /app/prompts
Type: Bind  # 绑定到宿主机的prompts目录
Path: /home/user/nofx/prompts  # 确保宿主机的这个目录存在
```

**Health Check**:
```yaml
Test: CMD wget -q --spider http://localhost:8080/api/health
Interval: 30s
Timeout: 10s
Retries: 3
Start Period: 120s
```

**Domains**:
- Backend通常不直接暴露，通过Frontend代理

点击 **Deploy** 部署

### 步骤3: 添加前端服务 (Frontend)

**服务类型**: **Application**

**基本信息**:
```yaml
Name: frontend
Repository: https://github.com/foreveryh/nofx
Branch: main
Build Path: /
Dockerfile Path: ./docker/Dockerfile.frontend
```

**环境变量**:
```bash
NEXT_PUBLIC_API_URL=https://nofx.deeptoai.com/api  # 通过Nginx代理到backend
NEXT_PUBLIC_WS_URL=wss://nofx.deeptoai.com/ws
NEXT_PUBLIC_DOMAIN=nofx.deeptoai.com
```

**部署配置**:
```yaml
Port: 80
Restart Policy: unless-stopped
Auto Deploy: true
```

**Domains**:
```yaml
Domain: nofx.deeptoai.com
HTTPS: true  # Dokploy会自动申请Let's Encrypt证书
Certificate: Auto-generated (Let's Encrypt)
```

**Nginx配置** (Dokploy会自动配置反向代理):
```nginx
# Dokploy会自动添加 (无需手动配置)
location /api {
    proxy_pass http://backend:8080;
}

location /ws {
    proxy_pass http://backend:8080;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
}
```

点击 **Deploy** 部署

---

## 🔧 Dokploy高级配置

### 1. 环境变量管理

可以在Dokploy中创建 **Secrets** 管理敏感信息:

```yaml
# 在Dokploy Dashboard → Secrets

Name: binance_api_key
Value: <your-key>

# 在服务中使用
Environment: BINANCE_API_KEY=${binance_api_key}
```

优点:
- ✅ 密钥不会明文存储在.env文件
- ✅ 可以在多个服务间共享
- ✅ 修改后自动重新部署

### 2. 自动部署Webhook

配置GitHub自动部署:

```bash
# 在GitHub仓库 Settings → Webhooks

Payload URL: https://your-dokploy-domain/api/deploy?projectId=xxx&applicationId=xxx
Content type: application/json
Secret: 留空或设置
Events: push

# 获取Webhook URL (在Dokploy服务页面)
# Applications → backend → Settings → Webhook URL
```

### 3. 日志监控

在Dokploy中查看日志:
```
Applications → backend → Logs
- Real-time logs
- Search functionality
- Download logs
```

### 4. 资源监控

```
Monitoring → Resources
- CPU usage
- Memory usage
- Network I/O
- Disk usage
```

### 5. 数据库备份 (SQLite)

由于使用SQLite，需要手动配置备份:

在Dokploy中添加 **Scheduled Jobs**:

```yaml
Name: Backup Database
Cron: 0 3 * * *  # 每天凌晨3点
Command: |
  cd /home/user/nofx-trading
  docker compose exec backend sqlite3 /app/data/config.db ".backup /app/data/backup_$(date +%Y%m%d).db"

  # 上传到S3 (可选)
  aws s3 cp /app/data/backup_$(date +%Y%m%d).db s3://your-bucket/
```

---

## 🔐 安全加固 (Dokploy环境)

### 1. 网络隔离

Dokploy会自动创建隔离网络，但可以额外配置:

```bash
# 在服务器上执行
# 仅允许Cloudflare IP访问 (可选)

# 获取Cloudflare IP列表: https://www.cloudflare.com/ips-v4

sudo ufw reset
sudo ufw default deny incoming
sudo ufw default allow outgoing

# 允许SSH
sudo ufw allow 22/tcp

# 允许Cloudflare IPs
for ip in $(curl https://www.cloudflare.com/ips-v4); do
  sudo ufw allow from $ip to any port 80,443
  echo "Allowed: $ip"
done

sudo ufw enable
```

### 2. API密钥保护 (Backend)

在Dokploy中配置 **Middleware**:

1. **Applications → backend → Settings → Middleware**
2. **自定义Nginx配置**:

```nginx
# 添加API密钥验证
location /api {
    # Cloudflare传递真实IP
    real_ip_header CF-Connecting-IP;
    set_real_ip_from 172.20.0.0/16;

    # API密钥验证
    set $api_key $http_x_api_key;
    if ($api_key = "") {
        set $api_key $arg_api_key;
    }

    # 检查API密钥 (从环境变量读取)
    set $expected_key "${BINANCE_API_KEY}";  # 或使用独立的环境变量
    if ($api_key != $expected_key) {
        return 401 'Invalid API Key';
    }
}
```

**更好的方式**: 修改应用代码 (见下文)

### 3. 应用层API保护

**修改 `api/server.go`** (在部署前):

```go
func (s *Server) apiKeyMiddleware() gin.HandlerFunc {
	return func(c *gin.Context) {
		// Cloudflare传递的请求头
		cfIp := c.GetHeader("CF-Connecting-IP")
		if cfIp != "" {
			log.Printf("Request from Cloudflare IP: %s", cfIp)
		}

		// API密钥验证
		apiKey := c.GetHeader("X-API-Key")
		if apiKey == "" {
			apiKey = c.Query("api_key")
		}

		expectedKey := os.Getenv("NOFX_API_KEY")
		if expectedKey != "" && apiKey != expectedKey {
			c.JSON(http.StatusUnauthorized, gin.H{"error": "Invalid API Key"})
			c.Abort()
			return
		}

		c.Next()
	}
}

// 在setupRoutes中
func (s *Server) setupRoutes() {
    // ...
    s.router.Use(s.apiKeyMiddleware())  // 在auth中间件之前
    s.router.Use(s.authMiddleware())
    // ...
}
```

**添加环境变量**:
```bash
NOFX_API_KEY=your-secret-api-key  # 在Dokploy中添加
```

---

## 🌐 Cloudflare配置清单

### DNS记录

在Cloudflare Dashboard → **DNS** → **Records**:

```
Type   Name     Content          Proxy  TTL
A      nofx     [YOUR_VPS_IP]    🟧     Auto
```

### SSL/TLS配置

**SSL/TLS → Overview**:
- **Mode**: Full (strict)  # 需要服务器有SSL证书
- Or: Full  # 如果只有HTTP

**SSL/TLS → Edge Certificates**:
- 确保证书状态: Active
- 自动HTTPS: On
- 最低TLS版本: 1.2

### Zero Trust配置

**Access → Applications**:

```yaml
Application:
  - Name: NOFX Trading
  - Domain: nofx.deeptoai.com
  - Type: Self-hosted


Policies:
  - Name: Admin Only
    Action: Allow
    Include: Emails = [your-email@example.com]
    Require: None

  - Name: Block Others
    Action: Block
    Include: Everyone
```

### 防火墙规则

**Security → WAF → Custom Rules**:

```yaml
Rule Name: Block API without key
If: URI Path contains "/api"
And: NOT (HTTP Headers contains "X-API-Key")
Action: Block
```

---

## 📊 监控与告警

### 1. Dokploy监控

```yaml
# 在Dokploy中配置

Applications → backend → Settings → Monitoring
- CPU Alert: 80%
- Memory Alert: 80%
- Restart on failure: true
```

### 2. 健康检查

```bash
# 测试API健康
curl -f https://nofx.deeptoai.com/api/health

# 测试WebSocket
curl -i -N -H "Connection: Upgrade" -H "Upgrade: websocket" \
  https://nofx.deeptoai.com/ws
```

### 3. Cloudflare监控

**Analytics → Traffic**:
- 查看请求量
- 查看缓存命中率
- 查看威胁阻止

---

## 🔄 部署流程

### 1. 首次部署

```bash
# 1. 准备代码
mkdir -p /home/user/nofx-trading
cd /home/user/nofx-trading
git clone https://github.com/foreveryh/nofx.git .

# 2. 准备环境变量文件
cp .env.dokploy.complete .env.dokploy

# 3. 编辑环境变量
nano .env.dokploy
# → 修改所有 JWT_SECRET, API密钥, AI密钥

# 4. 创建数据目录
mkdir -p data logs config prompts
cp -r prompts/* config/

# 5. 在Dokploy中创建项目和服务 (见上文)

# 6. 部署
docker compose -f docker-compose.dokploy.yml up -d
```

### 2. 更新部署

#### 方式A: Git自动部署（推荐）

```bash
# 本地修改代码
git add .
git commit -m "Update config"
git push origin main

# Dokploy会自动触发部署
```

#### 方式B: 手动触发

```bash
# 在Dokploy Dashboard
Applications → backend → Deploy
Applications → frontend → Deploy
```

#### 方式C: Webhook

```bash
# 使用Dokploy提供的Webhook URL
curl -X POST "https://dokploy.yourdomain.com/api/deploy?projectId=xxx&appId=xxx"
```

---

## 🐛 常见问题排查

### 问题1: Cloudflare Zero Trust登录循环

**现象**: 反复重定向到登录页面

**解决**:
```bash
# 1. 清除Cookie
# 2. 检查Session Duration设置
# 3. 查看Cloudflare Audit Logs
# 4. 检查应用域名配置是否正确
```

### 问题2: WebSocket连接失败

**现象**: 前端无法连接WebSocket

**解决**:
```bash
# 检查Nginx配置
location /ws {
    proxy_pass http://backend:8080;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_read_timeout 86400;
}

# 检查Cloudflare设置
# Network → WebSockets: ON
```

### 问题3: API密钥验证失败

**现象**: 401 Unauthorized

**调试**:
```bash
# 测试API
curl -H "X-API-Key: your-key" https://nofx.deeptoai.com/api/traders

# 查看日志
docker compose logs backend -f
```

### 问题4: 容器无法启动

**现象**: 健康检查失败

**排查**:
```bash
# 查看容器日志
docker compose logs backend --tail=100

# 进入容器调试
docker compose exec backend /bin/sh
./nofx
```

### 问题5: SSL证书错误

**现象**: HTTPS访问失败

**检查**:
```bash
# 检查证书有效期
echo | openssl s_client -servername nofx.deeptoai.com -connect nofx.deeptoai.com:443 | openssl x509 -noout -dates

# Cloudflare SSL模式
# SSL/TLS → Full (strict) 或 Full
```

---

## 📦 备份与恢复

### 自动备份

在Dokploy中添加 **Job**:

```bash
Name: Backup Database
Schedule: 0 3 * * *  # 每天凌晨3点
Command: |
  #!/bin/bash
  DATE=$(date +%Y%m%d_%H%M%S)
  docker compose exec backend sqlite3 /app/data/config.db ".backup /app/data/backup_$DATE.db"

  # 压缩
  tar -czf /home/user/backups/nofx_backup_$DATE.tar.gz \
    /app/data/backup_$DATE.db \
    /app/config

  # 删除旧备份 (保留7天)
  find /home/user/backups -name "nofx_backup_*.tar.gz" -mtime +7 -delete
```

### 手动备份

```bash
# 备份数据库
docker compose exec backend sqlite3 /app/data/config.db ".backup /app/data/backup_$(date +%Y%m%d).db"

# 备份配置文件
cp .env.dokploy.complete backups/
cp config.json backups/
cp docker-compose.dokploy.yml backups/

# 打包
tar -czf nofx_full_backup_$(date +%Y%m%d).tar.gz backups/
```

### 恢复

```bash
# 停止服务
docker compose down

# 恢复数据库文件
cp backup_20251109.db /home/user/nofx-trading/data/config.db

# 恢复配置
cp backup.env.dokploy .env.dokploy

# 启动服务
docker compose up -d
```

---

## 🎯 性能优化

### Dokploy层面

```yaml
# 在docker-compose.dokploy.yml中添加

services:
  backend:
    deploy:
      resources:
        limits:
          memory: 2G
          cpus: '1'
        reservations:
          memory: 512M
          cpus: '0.5'

    # 启用性能调优
    sysctls:
      - net.core.somaxconn=65535
      - net.ipv4.tcp_congestion_control=bbr
```

### Cloudflare层面

**速度优化**:

1. **Caching**:
   ```yaml
   Caching Level: Standard
   Browser Cache TTL: 4 hours
   Always Online: On
   ```

2. **Auto Minify**:
   ```yaml
   JavaScript: ✓
   CSS: ✓
   HTML: ✓
   ```

3. **Rocket Loader**: Automatic

4. **Early Hints**: On

**安全优化**:

1. **SSL/TLS**:
   ```yaml
   Minimum TLS Version: 1.2
   Opportunistic Encryption: On
   TLS 1.3: On
   ```

2. **Security Level**: High

3. **Challenge Passage**: 30 minutes

4. **Enable Rate Limiting**:
   ```yaml
   - Threshold: 100 requests per 10 seconds
   - Action: Challenge
   - Duration: 1 hour
   ```

---

## 📊 监控指标

### 关键指标

**Dokploy监控**:
- CPU使用率: < 80%
- 内存使用率: < 80%
- 重启次数: 0 (稳定运行)
- 健康检查成功率: 100%

**Cloudflare监控**:
- Cache Hit Ratio: > 80%
- Threats Blocked: 监控趋势
- Request Volume: 正常模式
- Bandwidth Saved: 越高越好

**应用监控**:
- API响应时间: < 100ms (avg)
- WebSocket连接数: 保持稳定
- AI决策成功率: > 90%
- 交易成功率: > 95%

### 告警配置

在Dokploy中添加 **Alerts**:

```yaml
Name: Backend CPU Alert
Condition: CPU > 80%
Duration: 5 minutes
Action: Send notification

Notification Channels:
  - Email: your-email@example.com
  - Slack Webhook: https://hooks.slack.com/...
```

---

## 📝 部署任务清单

### 前置准备

- [ ] VPS服务器 (4核4GB)
- [ ] 安装Docker和Docker Compose
- [ ] 安装Dokploy
- [ ] 配置SSH密钥登录
- [ ] 域名: nofx.deeptoai.com 接入Cloudflare
- [ ] Cloudflare Zero Trust免费计划
- [ ] GitHub仓库 (可选)

### Cloudflare配置

- [ ] 添加DNS A记录
- [ ] 启用Proxy (橙色云)
- [ ] 配置Zero Trust Application
- [ ] 配置策略 (Email + IP)
- [ ] 测试认证流程

### Dokploy配置

- [ ] 登录Dokploy面板
- [ ] 修改默认密码
- [ ] 创建Project: nofx-trading
- [ ] 添加Backend服务
  - [ ] 配置环境变量 (.env.dokploy.complete)
  - [ ] 配置Volumes
  - [ ] 配置Resources
  - [ ] 部署
- [ ] 添加Frontend服务
  - [ ] 配置环境变量
  - [ ] 配置Domain: nofx.deeptoai.com
  - [ ] 启用HTTPS
  - [ ] 部署

### 应用配置

- [ ] 生成JWT_SECRET
- [ ] 配置交易所API密钥
- [ ] 配置AI模型API密钥
- [ ] 配置API密钥 (NOFX_API_KEY)
- [ ] 测试API连接
- [ ] 测试WebSocket连接
- [ ] 验证AI决策

### 安全加固

- [ ] 配置防火墙 (UFW)
- [ ] 配置fail2ban
- [ ] 配置API限流
- [ ] 配置自动备份
- [ ] 配置监控告警
- [ ] 测试备份恢复

### 上线前测试

- [ ] 完整功能测试
- [ ] Zero Trust认证测试
- [ ] SSL证书测试
- [ ] 压力测试 (可选)
- [ ] 监控告警测试
- [ ] 备份恢复测试

---

## 📚 重要文件清单

### 必需文件

```
~/nofx-trading/
├── .env.dokploy.complete      # 环境变量配置文件
├── docker-compose.dokploy.yml # Docker Compose配置
├── config.json               # 应用配置文件
├── prompts/                  # 提示词模板
│   ├── adaptive.txt
│   ├── default.txt
│   └── nof1.txt
└── deployment/
    └── setup.sh              # 部署脚本 (可选)
```

### 备份文件

```
~/backups/
├── nofx_backup_20251109.tar.gz
└── nofx_backup_20251110.tar.gz
```

### 配置文件

```
~/.env.dokploy.complete  (Dokploy导入)
config.json             (应用配置)
```

---

## 🎉 部署完成后的访问方式

**URL**: `https://nofx.deeptoai.com`

**认证方式**:
1. 访问URL
2. 跳转到Cloudflare Zero Trust登录页
3. 输入邮箱接收PIN码
4. 输入PIN码验证
5. 自动跳转回NOFX系统
6. 直接进入管理员界面 (admin_mode=true)

**优势**:
- ✅ 全球CDN加速
- ✅ DDoS保护
- ✅ Zero Trust认证
- ✅ 自动SSL证书
- ✅ 无需自维护认证系统

---

## 📞 支持联系方式

### 故障排查

1. **Cloudflare**: Dashboard → Analytics → Access
2. **Dokploy**: Dashboard → Applications → Logs
3. **Docker**: `docker compose logs`
4. **应用**: `curl https://nofx.deeptoai.com/api/health`

### 日志位置

```bash
# Backend日志
docker logs nofx-backend

# Frontend日志
docker logs nofx-frontend

# Dokploy日志
tail -f /var/log/dokploy/*.log
```

---

**部署完成！系统已准备就绪，可以通过 nofx.deeptoai.com 访问！**

🚀 **Happy Trading!**
