# NOFX AI交易系统 - Dokploy操作指南

**可视化界面操作，无需命令行**

---

## 📋 目录

1. [Web界面操作](#web界面操作)
2. [环境变量配置](#环境变量配置)
3. [域名和SSL](#域名和ssl)
4. [监控和日志](#监控和日志)
5. [更新和维护](#更新和维护)

---

## 🖥️ Web界面操作

### 1. 创建应用

1. **登录Dokploy**: http://your-server-ip:3000
2. **点击"New Application"**
3. **选择部署方式**: Git
4. **填写基本信息**:
   ```
   Name: NOFX Trading
   Repository: https://github.com/yourusername/nofx.git
   Branch: main
   Build Path: .
   ```
5. **点击"Create Application"**

### 2. 构建设置

在应用设置页面：

**General Settings**:
- **Dockerfile Path**: `./docker/Dockerfile.backend`
- **Build Context**: `.`
- **Restart Policy**: `always`

**Resources** (可选):
- **Memory Limit**: 1GB
- **CPU Limit**: 0.5 cores

### 3. 环境变量配置

**步骤**:
1. 点击"Environment Variables"标签
2. 点击"Add Variable"
3. 复制 `.env.dokploy.template` 中的配置
4. **必须修改**以下配置：
   - `JWT_SECRET` - JWT密钥
   - `BINANCE_API_KEY` - Binance API密钥
   - `BINANCE_SECRET_KEY` - Binance Secret密钥
   - `DASHSCOPE_API_KEY` - AI模型API密钥

**重要配置示例**:
```
NODE_ENV=production
JWT_SECRET=your-super-secret-jwt-key-here
BINANCE_API_KEY=your_binance_api_key
BINANCE_SECRET_KEY=your_binance_secret_key
DASHSCOPE_API_KEY=your_dashscope_api_key
```

---

## 🌐 域名和SSL配置

### 1. 添加域名

1. 点击"Domains"标签
2. 点击"Add Domain"
3. **填写域名**: `yourdomain.com`
4. **Protocol**: HTTPS (自动)
5. **SSL Certificate**: Auto-generate (Let's Encrypt)
6. 点击"Add Domain"

### 2. DNS配置

在您的域名提供商处设置：

```
Type: A
Name: @
Value: your-server-ip
TTL: 300

Type: A
Name: www
Value: your-server-ip
TTL: 300
```

### 3. SSL证书状态

Dokploy会自动：
- 申请Let's Encrypt证书
- 自动续期
- 配置HTTPS重定向

**状态检查**:
- 🟢 Active - 证书正常
- 🟡 Pending - 申请中
- 🔴 Failed - 需要检查DNS

---

## 📊 监控和日志

### 1. 实时监控

在Dokploy控制台查看：

**Overview页面**:
- 📈 CPU使用率
- 💾 内存使用率
- 📊 网络流量
- 💿 磁盘使用
- 🔄 容器状态

**健康检查**:
- 自动检测服务可用性
- 失败时自动重启
- 状态指示器 (绿色=正常, 红色=异常)

### 2. 日志查看

**实时日志**:
1. 点击应用名称
2. 点击"Logs"标签
3. 选择日志级别: All, Error, Warning, Info
4. 实时查看最新日志

**历史日志**:
- 设置时间范围查看历史日志
- 搜索特定关键词
- 下载日志文件

### 3. 部署历史

**Deployments页面**:
- 📋 部署记录
- 🕐 部署时间
- ✅ 部署状态
- 🔍 查看部署日志
- 🔄 回滚到之前版本

### 4. 资源使用

**Statistics页面**:
- CPU使用图表
- 内存使用图表
- 网络流量图表
- 磁盘使用图表

---

## 🔄 更新和维护

### 1. 自动部署

**Git Webhook** (推荐):
1. 在Git仓库设置Webhook
2. URL: `https://your-dokploy-domain.com/webhook/github`
3. 每次推送代码自动部署

**手动部署**:
1. 点击应用名称
2. 点击"Deploy Now"
3. 选择分支
4. 等待部署完成

### 2. 回滚操作

如果部署出现问题：

1. **点击"Deployments"**
2. **找到之前的成功部署**
3. **点击右侧的"Rollback"**
4. **确认回滚**

### 3. 重启服务

1. **点击应用名称**
2. **点击"Actions"按钮**
3. **选择"Restart"**
4. **确认重启**

### 4. 更新环境变量

1. **点击"Environment Variables"**
2. **修改需要的变量**
3. **点击"Save"**
4. **重启应用以应用更改**

---

## ⚙️ 高级操作

### 1. 数据备份

**自动备份**:
1. 点击"Backups"标签
2. 设置备份计划
3. 配置存储位置 (S3, Google Drive等)

**手动备份**:
1. 点击"Create Backup"
2. 等待备份完成
3. 下载备份文件

### 2. 多环境部署

**创建环境**:
1. 创建多个应用 (production, staging, development)
2. 使用不同分支
3. 配置不同的环境变量

**环境切换**:
- 通过域名区分不同环境
- 使用不同的API密钥
- 独立的数据库

### 3. 性能调优

**资源限制**:
1. 点击"Settings"
2. 调整Memory和CPU限制
3. 根据实际使用情况优化

**健康检查**:
1. 调整检查间隔
2. 设置超时时间
3. 配置重试次数

---

## 🚨 故障排除

### 1. 部署失败

**检查步骤**:
1. 查看"Deployments"页面的错误日志
2. 检查环境变量是否正确
3. 验证Dockerfile配置
4. 确认Git仓库可访问

**常见问题**:
- **构建超时**: 增加构建超时时间
- **依赖安装失败**: 检查package.json或go.mod
- **权限错误**: 检查文件权限设置

### 2. 服务无法访问

**检查清单**:
- [ ] 容器是否运行 (Overview页面)
- [ ] 健康检查是否通过
- [ ] 域名DNS是否正确解析
- [ ] SSL证书是否有效
- [ ] 防火墙是否开放80/443端口

### 3. 性能问题

**优化建议**:
- 增加内存限制
- 检查数据库查询
- 启用缓存
- 优化应用代码

### 4. API错误

**调试步骤**:
1. 查看应用日志
2. 检查环境变量配置
3. 验证API密钥有效性
4. 测试外部服务连通性

---

## 📞 获取帮助

### Dokploy支持
- **官方文档**: [docs.dokploy.com](https://docs.dokploy.com)
- **GitHub Issues**: [github.com/dokploy/dokploy/issues](https://github.com/dokploy/dokploy/issues)
- **Discord社区**: [discord.gg/dokploy](https://discord.gg/dokploy)

### NOFX项目支持
- **项目文档**: [README.md](./README.md)
- **部署指南**: [DOCKER_DEPLOYMENT_GUIDE.md](./DOCKER_DEPLOYMENT_GUIDE.md)
- **GitHub Issues**: [github.com/NoFxAiOS/nofx/issues](https://github.com/NoFxAiOS/nofx/issues)

---

## 🎯 最佳实践

### 1. 安全配置
- 定期轮换API密钥
- 使用强密码的JWT密钥
- 启用防火墙和访问控制
- 定期备份数据

### 2. 监控告警
- 设置资源使用告警
- 配置邮件/Slack通知
- 监控API响应时间
- 跟踪错误率

### 3. 运维建议
- 定期更新Docker镜像
- 监控磁盘空间使用
- 清理未使用的镜像
- 定期测试备份恢复

---

**记住**: Dokploy是可视化工具，所有操作都通过Web界面完成，无需复杂的命令行操作！