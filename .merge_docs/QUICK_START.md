# NOFX AI交易系统 - 快速启动指南

**⚡ 5分钟快速部署，包含动态止盈止损功能**

---

## 🚀 一键启动

```bash
# 1. 克隆项目
git clone https://github.com/NoFxAiOS/nofx.git
cd nofx

# 2. 配置必要文件
cp .env.example .env
cp config.json.example config.json

# 3. 修改JWT密钥（必须！）
# 编辑 config.json，修改 jwt_secret 字段
nano config.json

# 4. 启动系统
./start.sh start --build
```

**访问地址**:
- 🌐 Web界面: http://localhost:3000
- 🔌 API端点: http://localhost:8080

---

## 🔑 必需配置

### 1. 修改JWT密钥 (Critical)

在 `config.json` 中找到并修改：

```json
{
  "jwt_secret": "Qk0kAa+d0iIEzXVHXbNbm+UaN3RNabmWtH8rDWZ5OPf+4GX8pBflAHodfpbipVMyrw1fsDanHsNBjhgbDeK9Jg=="
}
```

**生成新密钥**:
```bash
openssl rand -base64 64
```

### 2. 配置交易所API

通过Web界面 (http://localhost:3000) 配置：

1. **注册管理员账户**
2. **进入 交易设置 → 交易所管理**
3. **添加交易所API**

**Binance Futures示例**:
- API Key: `sk_xxxxxxxxxxxx`
- Secret Key: `xxxxxxxxxxxxxxxx`
- 权限: 仅期货交易（关闭提现！）
- IP白名单: 添加服务器IP

### 3. 配置AI模型

**国内用户** - 阿里云DashScope:
```bash
# 设置环境变量或通过Web界面配置
DASHSCOPE_API_KEY=your_dashscope_api_key
```

**海外用户** - OpenAI:
```bash
OPENAI_API_KEY=your_openai_api_key
```

---

## 🎯 新功能亮点

### ✨ 动态止盈止损 (v2.0新增)

系统现在支持三种新的交易动作：

1. **`update_stop_loss`** - 动态调整止损价格
2. **`update_take_profit`** - 动态调整止盈价格
3. **`partial_close`** - 部分平仓（0-100%）

### 📋 策略模板

1. **Default** - 保守策略，适合新手
2. **Adaptive** - 自适应双策略，包含动态TP/SL
3. **Nof1** - 单一高频策略

---

## 🧪 快速测试

### 1. 检查服务状态

```bash
./start.sh status
```

### 2. 查看日志

```bash
./start.sh logs
```

### 3. 健康检查

```bash
curl http://localhost:8080/api/health
```

---

## 🔧 常用命令

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

1. **🔐 安全第一**
   - 必须修改默认JWT密钥
   - 不要在代码中硬编码API Key
   - 交易所API只授予交易权限

2. **💰 资金安全**
   - 从小额资金开始测试
   - 建议先使用测试网
   - 设置合理的风险控制

3. **📊 监控系统**
   - 定期检查决策日志
   - 监控API响应时间
   - 设置告警机制

---

## 📚 详细文档

- 📖 [完整部署指南](./DOCKER_DEPLOYMENT_GUIDE.md)
- 📋 [PR合并报告](./MERGE_REPORT.md)
- 🔍 [功能变更说明](./.merge_docs/trading/TRADING_CHANGES.md)
- 📝 [后续合并分析](./.merge_docs/POST_220_MERGE_ANALYSIS.md)

---

## 🆘 遇到问题？

1. **查看日志**: `./start.sh logs`
2. **检查端口**: 确保8080和3000端口未被占用
3. **验证配置**: 检查config.json和.env文件
4. **提交Issue**: [GitHub Issues](https://github.com/NoFxAiOS/nofx/issues)

---

**版本**: v2.0 (包含动态TP/SL功能)
**更新日期**: 2025-11-02

⭐ **开始您的AI交易之旅！**