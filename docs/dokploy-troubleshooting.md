# Dokploy 部署排障记录

本文记录最近一次在 Dokploy 上部署 NOFX 遇到的问题及解决方案，方便后续排障或复现。

## 1. 前端构建依赖 `process.env`
- **现象**：Dokploy 的 Docker 构建阶段，在 `npm run build` 时提示 `Cannot find name 'process'`。
- **原因**：Vite 项目在客户端不应访问 Node 的 `process.env`。
- **解决**：
  1. `web/src/lib/api.ts` 改为读取 `window.NEXT_PUBLIC_API_URL` 和 `import.meta.env`。
  2. 添加 `web/src/vite-env.d.ts` 让 TS 识别 `import.meta`.

## 2. `config.json` / `beta_codes.txt` 被挂载成目录
- **现象**：容器启动后日志提示 `read config.json: is a directory`。
- **原因**：Dokploy 挂载文件时若目标不存在，会创建目录并把同名文件放到目录内部。
- **解决**：
  - 新增 `init` 服务检测到目录时，改写为内部文件（例如 `config.json/config.json`），并创建缺失的文件。
  - 后端在运行时自动探测：优先读取手动设置的路径，若发现是目录则回退到 `path/path`。

## 3. Init 脚本写入的 JSON 语法损坏
- **现象**：即使文件存在，后端解析报 `invalid character 'n'`。
- **原因**：`printf` 使用了被 YAML 转义的 `\"`，实际文件里多出了反斜杠。
- **解决**：改用多行 Writing（或单行 `printf` 输出真实引号），确保落盘 JSON 无转义字符。

## 4. SQLite 启用 WAL 报 `out of memory (14)`
- **现象**：日志 `❌ 初始化数据库失败: unable to open database file: out of memory (14)`。
- **原因**：`config.db` 同样被挂载成目录，SQLite 无法在该路径创建文件。
- **解决**：
  - 后端在打开数据库前调用 `resolveDatabasePath`，当目标是目录时改为 `config.db/config.db`。
  - 启动前自动 `os.MkdirAll`，确保 SQLite 可写。

## 5. Init 脚本 YAML 引号问题
- **现象**：Dokploy 解析 compose 文件时报 `Implicit keys need to be on a single line` 或脚本执行 `sh: syntax error`.
- **原因**：多层引号/转义在 YAML 中容易被误解析。
- **解决**：将 `command` 写成 `["/bin/sh","-c", <<SCRIPT]` 的形式，脚本本体保持纯 ASCII，避免二次转义。

## 6. `CONFIG_JSON_PATH` 环境变量导致 “not a directory”
- **现象**：本地开发希望仍然读取 `config.json` 文件，但硬编码的 `CONFIG_JSON_PATH=/app/config.json/config.json` 会在非 Dokploy 环境报 `not a directory`。
- **解决**：去掉固定环境变量，完全交由后端 `resolveFallbackPath` 自动判断文件/目录。

## 7. 反向代理端口说明
- Dokploy 版本默认 **不暴露 3000 端口**，而是让 Traefik 通过 Docker 网络直接访问容器 80。
- 如果要手动挂代理到 3000，需要使用 `docker-compose.yml`（而非 Dokploy 专用文件），其中 `nofx-frontend` 会把宿主机 `${NOFX_FRONTEND_PORT:-3000}` 映射到容器 80。
- 推荐方案：让 Cloudflare → Traefik (80/443) → `nofx-frontend`，无需开放 3000。

---

若将来再次遇到 “文件成目录” 的问题，可复用本次的策略：初始化脚本统一落在目录内部文件、业务代码自动推断真实路径，从而兼容 Dokploy 的文件挂载行为。
