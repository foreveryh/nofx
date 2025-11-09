# 生产环境VPS部署与安全配置指南

**适用场景**: 24小时运行 + 单用户 + VPS部署 + 安全优先

---

## 📋 核心配置原则

### ✅ 保持 `admin_mode: true` 的理由

对于您的使用场景，这是最合适的配置：

```json
{
  "admin_mode": true
}
```

**优势**:
```
✅ 无需注册流程（减少攻击面）
✅ 快速登录（直接进入系统）
✅ 减少日志和用户表开销（数据库更轻量）
✅ 无需用户管理（维护更简单）
```

**劣势及解决方案**:
```
❌ 没有用户认证隔离？
   ✅ 通过API密钥和IP白名单保护
   ✅ 使用Nginx Basic Auth
   ✅ 配置防火墙限制访问
```

---

## 🛡️ 安全配置方案

### 方案1: API密钥保护（推荐）

在 `config.json` 中添加自定义API密钥中间件

**修改 `api/server.go`**:

```go
// 添加API密钥中间件
func (s *Server) apiKeyMiddleware() gin.HandlerFunc {
	return func(c *gin.Context) {
		// 放行健康检查和静态资源
		if c.Request.URL.Path == "/api/health" ||
		   strings.HasPrefix(c.Request.URL.Path, "/api/public/") {
			c.Next()
			return
		}

		// 检查API密钥
		apiKey := c.GetHeader("X-API-Key")
		if apiKey == "" {
			apiKey = c.Query("api_key")
		}

		expectedKey := os.Getenv("NOFX_API_KEY")
		if expectedKey != "" && apiKey != expectedKey {
			c.JSON(http.StatusUnauthorized, gin.H{"error": "无效的API密钥"})
			c.Abort()
			return
		}

		c.Next()
	}
}

// 在setupRoutes中注册
func (s *Server) setupRoutes() {
	// ... 现有路由

	// 添加API密钥保护（在auth中间件之前）
	s.router.Use(s.apiKeyMiddleware())  // ← 添加这一行
	s.router.Use(s.authMiddleware())

	// ... 后续路由
}
```

**配置环境变量 `.env`**:

```bash
# API密钥（32位以上随机字符串）
NOFX_API_KEY=your-secret-api-key-here-32-chars-minimum

# 其他配置
NOFX_BACKEND_PORT=8080
NOFX_FRONTEND_PORT=3000
NOFX_TIMEZONE=Asia/Shanghai
```

**前端调用示例**:

```javascript
// 在请求头中添加API密钥
const response = await fetch('/api/traders', {
  headers: {
    'X-API-Key': 'your-secret-api-key-here-32-chars-minimum'
  }
});
```

---

### 方案2: Nginx Basic Auth（快速实施）

在Frontend容器前添加Nginx反向代理

**创建 `.htpasswd` 文件**:

```bash
# 安装htpasswd工具
# Ubuntu/Debian: sudo apt install apache2-utils
# CentOS: sudo yum install httpd-tools

# 创建密码文件
htpasswd -c .htpasswd your_username
# 输入密码两次确认
```

**修改 `docker-compose.yml`**:

```yaml
services:
  # 添加Nginx代理层
  nginx:
    image: nginx:alpine
    container_name: nofx-nginx
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      - ./.htpasswd:/etc/nginx/.htpasswd:ro
      - ./ssl:/etc/nginx/ssl:ro  # SSL证书
    depends_on:
      - nofx
      - nofx-frontend
    networks:
      - nofx-network

  # 原有backend和frontend服务保持不变
  nofx:
    # ... backend配置
    ports:
      - "127.0.0.1:8080:8080"  # 改为仅本地访问

  nofx-frontend:
    # ... frontend配置
    ports:
      - "127.0.0.1:3000:3000"  # 改为仅本地访问
```

**创建 `nginx.conf`**:

```nginx
events {
    worker_connections 1024;
}

http {
    # 压缩配置
    gzip on;
    gzip_types text/plain text/css application/json application/javascript;

    # Backend代理
    server {
        listen 80;
        server_name api.yourdomain.com;

        # API密钥验证（可选）
        location /api/ {
            # Basic Auth
            auth_basic "Restricted Access";
            auth_basic_user_file /etc/nginx/.htpasswd;

            proxy_pass http://nofx:8080;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        }
    }

    # Frontend代理
    server {
        listen 80;
        server_name app.yourdomain.com;

        # Basic Auth
        auth_basic "Restricted Access";
        auth_basic_user_file /etc/nginx/.htpasswd;

        location / {
            proxy_pass http://nofx-frontend:80;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        }
    }
}
```

**访问方式**:
```
访问 http://app.yourdomain.com
弹出窗口输入: your_username / your_password
```

---

### 方案3: IP白名单 + 防火墙（最严格）

**修改 `docker-compose.yml`**:

```yaml
services:
  nofx:
    ports:
      - "127.0.0.1:8080:8080"  # 仅本地访问

  nofx-frontend:
    ports:
      - "127.0.0.1:3000:3000"  # 仅本地访问
```

**配置服务器防火墙（UFW示例）**:

```bash
# Ubuntu/Debian防火墙配置

# 1. 安装UFW
sudo apt update
sudo apt install ufw

# 2. 默认拒绝所有访问
sudo ufw default deny incoming
sudo ufw default allow outgoing

# 3. 允许SSH（必须！）
sudo ufw allow 22/tcp

# 4. 允许特定IP访问（替换 YOUR_HOME_IP）
sudo ufw allow from YOUR_HOME_IP to any port 80
sudo ufw allow from YOUR_HOME_IP to any port 443

# 5. 启用防火墙
sudo ufw enable

# 6. 查看状态
sudo ufw status verbose
```

**配置服务器防火墙（firewalld示例）**:

```bash
# CentOS/RHEL防火墙配置

# 1. 允许SSH
sudo firewall-cmd --permanent --add-service=ssh

# 2. 允许特定IP
sudo firewall-cmd --permanent --add-rich-rule="rule family='ipv4' source address='YOUR_HOME_IP' port protocol='tcp' port='80' accept"
sudo firewall-cmd --permanent --add-rich-rule="rule family='ipv4' source address='YOUR_HOME_IP' port protocol='tcp' port='443' accept"

# 3. 重载配置
sudo firewall-cmd --reload

# 4. 查看规则
sudo firewall-cmd --list-all
```

---

### 方案4: Cloudflare Zero Trust（现代化方案）

如果您的域名使用Cloudflare，可以配置Zero Trust免费认证

**步骤**:

1. **登录Cloudflare Dashboard**
2. **进入 Zero Trust → Access → Applications**
3. **添加自托管应用**:
   - 应用名: nofx-trading
   - 域名: app.yourdomain.com
   - 会话时长: 24h
4. **配置策略**:
   - 添加邮箱策略（只允许您的邮箱）
   - 或添加IP策略（只允许家庭IP）
5. **启用**

**优势**:
- ✅ 无需自己维护认证系统
- ✅ 免费（50个用户以内）
- ✅ 支持MFA
- ✅ 自动SSL证书

---

## 📝 VPS部署完整清单

### 环境准备

```bash
# 1. 更新系统
sudo apt update && sudo apt upgrade -y

# 2. 安装Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
sudo usermod -aG docker $USER

# 3. 安装Docker Compose
sudo apt install docker-compose-plugin -y

# 4. 创建项目目录
mkdir -p ~/nofx-trading
cd ~/nofx-trading

# 5. 复制项目文件（从本地或Git）
git clone https://github.com/foreveryh/nofx.git .

# 6. 创建配置文件
cp config.json.example config.json
cp .env.example .env

# 7. 编辑配置
nano config.json  # 配置交易所API、AI模型
nano .env         # 配置端口和时区
```

### Docker配置

**修改 `docker-compose.yml` 为生产环境优化**:

```yaml
services:
  nofx:
    build:
      context: .
      dockerfile: ./docker/Dockerfile.backend
    container_name: nofx-trading
    restart: unless-stopped  # 生产环境重启策略
    mem_limit: 2g            # 内存限制（防止OOM）
    cpus: 1.0                # CPU限制
    ports:
      - "127.0.0.1:8080:8080"  # 仅本地访问
    environment:
      - TZ=Asia/Shanghai
      # JVM选项（如果是Java应用，但这里不需要）
    volumes:
      - ./config.json:/app/config.json:ro
      - ./config.db:/app/config.db
      - ./beta_codes.txt:/app/beta_codes.txt:ro
      - ./decision_logs:/app/decision_logs
      - ./prompts:/app/prompts
      - /etc/localtime:/etc/localtime:ro
    networks:
      - nofx-network
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/api/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 120s  # 延长启动时间（加载K线数据）
    logging:  # 日志配置
      driver: "json-file"
      options:
        max-size: "100m"
        max-file: "3"

  nofx-frontend:
    build:
      context: .
      dockerfile: ./docker/Dockerfile.frontend
    container_name: nofx-frontend
    restart: unless-stopped
    mem_limit: 512m
    ports:
      - "127.0.0.1:3000:3000"  # 仅本地访问
    networks:
      - nofx-network
    depends_on:
      - nofx
    healthcheck:
      test: ["CMD", "wget", "--no-verbose", "--tries=1", "--spider", "http://localhost:3000"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 30s
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"

networks:
  nofx-network:
    driver: bridge
    ipam:
      config:
        - subnet: 172.20.0.0/16  # 固定子网
```

### 构建与启动

```bash
# 构建镜像（首次）
docker compose build --no-cache

# 启动服务（后台运行）
docker compose up -d

# 查看日志
docker compose logs -f --tail=50

# 等待服务就绪（约30秒）
sleep 30

# 验证状态
docker compose ps
curl http://localhost:8080/api/health
```

### 自动重启配置

**创建 systemd 服务（Ubuntu/Debian）**:

```bash
# 创建服务文件
sudo nano /etc/systemd/system/nofx-trading.service

# 内容如下
[Unit]
Description=NOFX Trading System
Requires=docker.service
After=docker.service

[Service]
Type=oneshot
RemainAfterExit=yes
WorkingDirectory=/home/your_user/nofx-trading
ExecStart=/usr/bin/docker compose up -d
ExecStop=/usr/bin/docker compose down

# 自动重启配置
Restart=on-failure
RestartSec=30

# 环境变量
Environment="NOFX_API_KEY=your-secret-key"

[Install]
WantedBy=multi-user.target

# 启用服务
sudo systemctl daemon-reload
sudo systemctl enable nofx-trading
sudo systemctl start nofx-trading

# 验证
sudo systemctl status nofx-trading

# 设置开机自启（Docker会自动重启容器）
sudo systemctl enable docker
```

### 监控与告警

**创建监控脚本 `monitor.sh`:

```bash
#!/bin/bash
# NOFX监控系统状态

WEBHOOK_URL="https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=YOUR_KEY"  # 企业微信/钉钉/Slack
CONTAINER_NAME="nofx-trading"

# 检查容器状态
if ! docker ps --format "table {{.Names}}" | grep -q "$CONTAINER_NAME"; then
  echo "[$(date)] 容器 $CONTAINER_NAME 未运行!"

  # 发送告警
  curl "$WEBHOOK_URL" \
    -H 'Content-Type: application/json' \
    -d "{\"msgtype\":\"text\",\"text\":{\"content\":\"🚨 NOFX交易系统异常！容器已停止\n时间: $(date '+%Y-%m-%d %H:%M:%S')\"}}"

  # 尝试重启
  cd /home/your_user/nofx-trading
  docker compose restart
  exit 1
fi

# 检查API健康
STATUS=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:8080/api/health)
if [ "$STATUS" != "200" ]; then
  echo "[$(date)] API健康检查失败: $STATUS"

  # 发送告警
  curl "$WEBHOOK_URL" \
    -H 'Content-Type: application/json' \
    -d "{\"msgtype\":\"text\",\"text\":{\"content\":\"🚨 NOFX交易系统API异常！\n状态码: $STATUS\n时间: $(date '+%Y-%m-%d %H:%M:%S')\"}}"

  exit 1
fi

echo "[$(date)] 系统正常"
```

**添加到crontab（每5分钟检查一次）**:

```bash
crontab -e

# 添加以下行
*/5 * * * * /home/your_user/nofx-trading/monitor.sh >> /home/your_user/nofx-trading/monitor.log 2>&1
```

---

## 🔐 安全加固清单

### 系统层面

- [ ] **更新系统补丁**
  ```bash
  sudo apt update && sudo apt upgrade -y
  sudo reboot
  ```

- [ ] **配置SSH安全**
  ```bash
  # 修改SSH端口（非22）
  sudo nano /etc/ssh/sshd_config
  # Port 2222

  # 禁用root登录
  # PermitRootLogin no

  # 重启SSH
  sudo systemctl restart sshd
  ```

- [ ] **配置防火墙**
  ```bash
  # UFW示例
  sudo ufw default deny incoming
  sudo ufw allow 22/tcp
  sudo ufw allow from YOUR_IP to any port 80
  sudo ufw enable
  ```

- [ ] **安装Fail2ban**
  ```bash
  sudo apt install fail2ban -y
  sudo systemctl enable fail2ban
  ```

- [ ] **配置自动安全更新**
  ```bash
  sudo apt install unattended-upgrades -y
  sudo dpkg-reconfigure -plow unattended-upgrades
  ```

### Docker层面

- [ ] **配置Docker守护进程安全**
  ```bash
  sudo nano /etc/docker/daemon.json

  {
    "live-restore": true,
    "no-new-privileges": true,
    "userland-proxy": false
  }
  ```

- [ ] **定期清理旧镜像**
  ```bash
  # 添加crontab
  0 3 * * * docker system prune -af
  ```

### 应用层面

- [ ] **配置API密钥** (见方案1)
- [ ] **配置Nginx Basic Auth** (见方案2)
- [ ] **配置IP白名单** (见方案3)
- [ ] **启用SSL/HTTPS** (见下文SSL配置)
- [ ] **配置日志轮转**
- [ ] **备份数据库** (见下文备份策略)

---

## 🔒 SSL/HTTPS配置

### 方式1: Let's Encrypt (推荐)

```bash
# 1. 安装Certbot
sudo apt install certbot python3-certbot-nginx -y

# 2. 获取证书
sudo certbot --nginx -d app.yourdomain.com

# 3. 自动续期
sudo systemctl enable certbot.timer

# 4. 测试续期
sudo certbot renew --dry-run
```

### 方式2: Cloudflare Origin证书

**在Cloudflare Dashboard**:
1. SSL/TLS → Origin Server → 创建证书
2. 保存证书和私钥

**创建证书文件**:

```bash
# 保存证书
sudo nano /etc/nginx/ssl/cert.pem
# 粘贴证书内容

# 保存私钥
sudo nano /etc/nginx/ssl/key.pem
# 粘贴私钥内容，设置权限 600

sudo chmod 600 /etc/nginx/ssl/key.pem
```

**Nginx配置**:

```nginx
server {
    listen 443 ssl http2;
    server_name app.yourdomain.com;

    ssl_certificate /etc/nginx/ssl/cert.pem;
    ssl_certificate_key /etc/nginx/ssl/key.pem;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;

    # 其他配置...
}
```

---

## 💾 备份策略

### 数据库备份脚本 `backup.sh`:

```bash
#!/bin/bash
# NOFX数据库备份脚本

BACKUP_DIR="/home/your_user/backups"
DATE=$(date +%Y%m%d_%H%M%S)
RETAIN_DAYS=7

# 创建备份目录
mkdir -p "$BACKUP_DIR"

# 停止容器（确保数据一致性）
cd /home/your_user/nofx-trading
docker compose stop nofx

# 备份数据库
cp config.db "$BACKUP_DIR/config.db.$DATE"

# 压缩备份
cd "$BACKUP_DIR"
tar -czf "nofx_backup_$DATE.tar.gz" config.db.$DATE config.json beta_codes.txt

# 删除原文件
rm config.db.$DATE

# 删除旧备份（保留7天）
find "$BACKUP_DIR" -name "nofx_backup_*.tar.gz" -mtime +$RETAIN_DAYS -delete

# 重启服务
cd /home/your_user/nofx-trading
docker compose start nofx

# 可选：上传到S3
# aws s3 cp "$BACKUP_DIR/nofx_backup_$DATE.tar.gz" s3://your-bucket/backups/

echo "备份完成: $BACKUP_DIR/nofx_backup_$DATE.tar.gz"
```

**添加到crontab（每天凌晨3点备份）**:

```bash
crontab -e

# 添加
0 3 * * * /home/your_user/nofx-trading/backup.sh >> /home/your_user/backups/backup.log 2>&1
```

---

## 📊 性能优化

### Docker资源限制

**在docker-compose.yml中配置**:

```yaml
services:
  nofx:
    deploy:
      resources:
        limits:
          cpus: '1'      # 限制为1个CPU
          memory: 2G     # 限制为2GB内存
        reservations:
          cpus: '0.5'
          memory: 1G
```

### SQLite性能优化

**在程序启动时执行**:

```sql
-- 启用WAL模式（提高并发性能）
PRAGMA journal_mode=WAL;

-- 设置缓存大小（根据内存调整）
PRAGMA cache_size=-10000;  -- 约10MB

-- 启用外键
PRAGMA foreign_keys=ON;
```

### 日志级别

**配置日志轮转**: 已经在docker-compose.yml中配置

**配置程序日志级别**:

在 `config.json` 中添加:

```json
{
  "log_level": "info",  // debug, info, warn, error
  "log_file": "decision_logs/trading.log",
  "log_max_size": "100MB"
}
```

---

## 🚨 故障恢复流程

### 场景1: 容器崩溃

```bash
# 查看日志
docker compose logs --tail=100 nofx

# 重启
docker compose restart nofx

# 如果持续崩溃，进入容器调试
docker compose exec nofx /bin/sh
# 在容器内手动运行
./nofx
```

### 场景2: 数据库损坏

```bash
# 停止服务
docker compose down

# 从备份恢复
# 找到最近的备份
ls -lah backups/*.tar.gz

# 解压覆盖
cd /home/your_user/nofx-trading
tar -xzf backups/nofx_backup_20251109_030000.tar.gz
mv config.db config.db.corrupted
mv config.db.20251109_030000 config.db

# 重启
docker compose up -d
```

### 场景3: 配置错误

```bash
# 回滚config.json
cp config.json.backup config.json
docker compose restart nofx
```

---

## 📞 支持联系方式

如果遇到问题，检查以下资源：

1. **容器日志**: `docker compose logs -f --tail=100`
2. **数据库**: `sqlite3 config.db "SELECT * FROM system_config;"`
3. **API测试**: `curl http://localhost:8080/api/health`
4. **文件权限**: `ls -la config.json config.db`
5. **端口占用**: `netstat -tln | grep 8080`

---

## 🎯 部署任务清单

### 初次部署前

- [ ] 购买VPS（推荐: 4核4GB内存）
- [ ] 配置DNS解析（可选）
- [ ] 生成API密钥（随机32位以上）
- [ ] 配置交易所API密钥（Binance/Hyperliquid）
- [ ] 配置AI模型API密钥（DeepSeek/Qwen）
- [ ] 准备交易资金（测试用小资金）
- [ ] 准备监控告警（企业微信/钉钉Webhook）

### 部署阶段

- [ ] 更新系统 `apt upgrade`
- [ ] 安装Docker和Docker Compose
- [ ] 复制项目文件
- [ ] 配置config.json
- [ ] 配置.env文件
- [ ] 配置API密钥（方案1）或Nginx Basic Auth（方案2）
- [ ] 修改docker-compose.yml（生产优化）
- [ ] 构建镜像 `docker compose build`
- [ ] 启动服务 `docker compose up -d`
- [ ] 验证API `curl http://localhost:8080/api/health`
- [ ] 检查日志无错误

### 安全加固

- [ ] 配置SSH安全（修改端口，密钥登录）
- [ ] 配置防火墙（UFW/firewalld）
- [ ] 配置fail2ban防暴力破解
- [ ] 配置SSL证书（Let's Encrypt）
- [ ] 配置自动备份脚本
- [ ] 配置监控告警
- [ ] 配置日志轮转
- [ ] 配置自动安全更新

### 上线前测试

- [ ] API响应正常
- [ ] WebSocket连接正常
- [ ] 数据库读写正常
- [ ] K线数据加载正常
- [ ] 交易所API连接正常
- [ ] AI决策生成正常
- [ ] 资金/仓位查询正常
- [ ] 强制平仓逻辑正常（重要！）

### 正式运行

- [ ] 设置systemd服务自动启动
- [ ] 配置开机自启
- [ ] 启动监控告警
- [ ] 配置备份任务
- [ ] 准备应急预案

---

**部署完成！系统已准备就绪，可以24小时稳定运行！** 🚀
