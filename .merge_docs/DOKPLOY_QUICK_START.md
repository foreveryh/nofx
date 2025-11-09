# NOFX AI交易系统 - Dokploy快速启动

**⚡ 10分钟部署到生产环境**

---

## 🚀 快速部署流程

Dokploy是可视化UI平台，通过Web界面操作，无需命令行脚本。

---

## 📋 前置要求

### 服务器要求
- **系统**: Ubuntu 20.04+ (推荐 Ubuntu 22.04 LTS)
- **内存**: 最低4GB，推荐8GB
- **存储**: 最低50GB SSD
- **CPU**: 最低2核，推荐4核

### 必需软件
- Docker >= 20.10
- Docker Compose >= 2.0
- Git

### 快速安装Docker
```bash
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
sudo usermod -aG docker $USER
```

---

## 🎯 部署步骤

### 第一步：准备服务器

```bash
# 1. 更新系统
sudo apt update && sudo apt upgrade -y

# 2. 安装必要工具
sudo apt install -y curl wget git htop

# 3. 防火墙配置
sudo ufw allow ssh
sudo ufw allow 80
sudo ufw allow 443
sudo ufw --force enable
```

### 第二步：在Dokploy中创建应用

1. **访问Dokploy**: http://your-server-ip:3000
2. **点击"New Application"**
3. **选择"Git"部署方式**
4. **填写仓库信息**:
   - Repository: https://github.com/yourusername/nofx.git
   - Branch: main
   - Build Path: .

### 第三步：配置构建设置

在Dokploy应用设置中：

1. **构建配置**:
   - Dockerfile Path: `./docker/Dockerfile.backend`
   - Build Context: `.`
   - Build Args: (如需要可添加)

2. **运行配置**:
   - Restart Policy: `always`
   - Health Check: 启用

### 第四步：添加环境变量

在Dokploy环境变量页面添加：

```bash
# 系统配置
NODE_ENV=production
TZ=Asia/Shanghai

# JWT密钥 (必须修改)
JWT_SECRET=your-super-secret-jwt-key-here

# 数据库
DATABASE_URL=sqlite:///app/data/config.db

# 交易所API (填入您的密钥)
BINANCE_API_KEY=your_binance_api_key
BINANCE_SECRET_KEY=your_binance_secret_key

# AI模型 (选择其一)
DASHSCOPE_API_KEY=your_dashscope_api_key
# 或
OPENAI_API_KEY=your_openai_api_key
```

### 第五步：配置域名和SSL

1. **点击"Domains"标签**
2. **添加域名**: `yourdomain.com`
3. **SSL设置**: 选择"Auto-generate" (Let's Encrypt)
4. **保存配置**

### 第六步：部署应用

点击"Deploy Now"开始部署，Dokploy会：
1. 拉取最新代码
2. 构建Docker镜像
3. 启动容器
4. 配置SSL证书
5. 健康检查

---

## 🌐 访问应用

部署成功后：

- **Web界面**: https://yourdomain.com
- **API端点**: https://yourdomain.com/api/health
- **Dokploy管理**: http://your-server-ip:3000

---

## 📊 功能特性

### Dokploy优势
- 🚀 **一键部署** - Git push自动部署
- 🔒 **自动SSL** - Let's Encrypt证书
- 📈 **实时监控** - CPU、内存、网络监控
- 🔄 **自动回滚** - 部署失败自动回滚
- 📝 **日志管理** - 集中日志查看
- 🌍 **多环境** - 开发、测试、生产环境

### NOFX功能
- 🤖 **AI交易** - 支持DashScope、OpenAI
- 💹 **动态TP/SL** - 实时调整止盈止损
- 📊 **部分平仓** - 灵活的仓位管理
- 👥 **多用户** - JWT认证系统
- 🔐 **安全** - API Key安全管理

---

## 🔧 配置验证

### 部署后检查

```bash
# 检查API健康状态
curl https://yourdomain.com/api/health

# 预期响应
{
  "status": "healthy",
  "timestamp": "2025-11-02T10:00:00Z",
  "uptime": "5m30s"
}

# 检查前端
curl https://yourdomain.com

# 预期响应: NOFX Web界面HTML
```

### Dokploy监控

在Dokploy控制台查看：
- 📈 **资源使用** - CPU、内存监控
- 🔄 **部署历史** - 部署记录和状态
- 📝 **应用日志** - 实时日志查看
- 🔔 **告警设置** - 资源告警配置

---

## 🛠️ 常用命令

### Dokploy命令
```bash
# 查看应用状态
dokploy ps

# 重启应用
dokploy restart nofx

# 查看日志
dokploy logs nofx

# 重新部署
dokploy redeploy nofx
```

### Docker命令
```bash
# 查看容器状态
docker ps

# 查看资源使用
docker stats

# 进入容器
docker exec -it <container_name> sh
```

---

## ❓ 常见问题

### Q1: SSL证书申请失败？
**A**: 检查DNS记录，确保域名正确解析到服务器IP

### Q2: 内存不足？
**A**: 调整容器资源限制或升级服务器配置

### Q3: API响应慢？
**A**: 检查网络延迟，考虑使用CDN

### Q4: 数据库连接失败？
**A**: 检查数据库文件权限和路径

---

## 📞 技术支持

- **Dokploy文档**: [docs.dokploy.com](https://docs.dokploy.com)
- **NOFX项目**: [GitHub Issues](https://github.com/NoFxAiOS/nofx/issues)
- **社区Discord**: [discord.gg/dokploy](https://discord.gg/dokploy)

---

## 🎉 部署完成

恭喜！您已成功使用Dokploy部署NOFX AI交易系统。

### 下一步
1. **配置交易所API** - 在Web界面中添加您的API密钥
2. **创建交易员** - 设置初始资金和交易策略
3. **测试交易** - 建议先在测试网环境测试
4. **监控系统** - 设置告警和监控

---

**快速部署步骤**:
1. 访问已安装的Dokploy: http://your-server-ip:3000
2. 创建新应用，选择Git部署
3. 配置环境变量和域名
4. 点击Deploy Now

**部署时间**: 10-15分钟
**维护成本**: 极低 (自动更新、SSL续期、监控告警)

🌟 **开始您的AI交易之旅！**