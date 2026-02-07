# 📦 Docker 部署完整指南

本文档提供从 GitHub Workflow 构建 Docker 镜像到服务器部署的完整流程。

---

## 📋 目录

- [架构概览](#架构概览)
- [前置准备](#前置准备)
- [第一步：配置 GitHub Workflow](#第一步配置-github-workflow)
- [第二步：触发镜像构建](#第二步触发镜像构建)
- [第三步：服务器部署](#第三步服务器部署)
- [运维操作](#运维操作)
- [故障排查](#故障排查)

---

## 🏗️ 架构概览

```
GitHub Repository
    ↓
GitHub Actions Workflow
    ↓
Docker Image Build (multi-arch)
    ↓
GitHub Container Registry (ghcr.io)
    ↓
Server Pull Image
    ↓
Docker Compose Deployment
    ↓
Services Running:
    - analyzer (定时分析)
    - server (Web API)
```

---

## 🔧 前置准备

### 1. GitHub 仓库配置

确保你的 GitHub 仓库已启用 **GitHub Packages** (默认已启用)。

### 2. 服务器环境要求

- **操作系统**: Linux (Ubuntu 20.04+, Debian 11+, CentOS 8+)
- **Docker**: 20.10+
- **Docker Compose**: 2.0+
- **内存**: 至少 2GB
- **磁盘**: 至少 10GB 可用空间

### 3. 安装 Docker 和 Docker Compose

```bash
# 安装 Docker
curl -fsSL https://get.docker.com | bash

# 启动 Docker 服务
sudo systemctl start docker
sudo systemctl enable docker

# 安装 Docker Compose (v2)
sudo apt-get update
sudo apt-get install docker-compose-plugin

# 验证安装
docker --version
docker compose version
```

---

## 📝 第一步：配置 GitHub Workflow

Workflow 配置文件已创建在：`.github/workflows/docker-build-push.yml`

### 触发条件

Workflow 会在以下情况自动触发：

1. **推送到 main 分支**: 自动构建并推送 `latest` 镜像
2. **创建版本标签** (如 `v1.0.0`): 自动构建并推送语义化版本镜像
3. **手动触发**: 在 GitHub Actions 页面手动执行

### Workflow 特性

- ✅ 多架构支持 (linux/amd64, linux/arm64)
- ✅ 自动版本标签管理
- ✅ 构建缓存加速
- ✅ 推送到 GitHub Container Registry

---

## 🚀 第二步：触发镜像构建

### 方式一：推送代码触发 (推荐)

```bash
# 1. 提交代码
git add .
git commit -m "feat: add new feature"

# 2. 推送到 main 分支（自动触发构建）
git push origin main
```

### 方式二：创建版本标签触发

```bash
# 1. 创建并推送版本标签
git tag v1.0.0
git push origin v1.0.0

# 这会生成以下镜像标签:
# - ghcr.io/your-username/daily_stock_analysis:v1.0.0
# - ghcr.io/your-username/daily_stock_analysis:1.0
# - ghcr.io/your-username/daily_stock_analysis:1
# - ghcr.io/your-username/daily_stock_analysis:latest
```

### 方式三：手动触发

1. 进入 GitHub 仓库页面
2. 点击 **Actions** 标签
3. 选择 **Build and Push Docker Image** workflow
4. 点击 **Run workflow** 按钮
5. 输入镜像标签（可选，默认为 `latest`）
6. 点击 **Run workflow** 执行

### 查看构建状态

1. 在 **Actions** 页面查看 workflow 执行状态
2. 构建成功后，在 **Packages** 页面可以看到推送的镜像
3. 镜像地址格式: `ghcr.io/your-username/daily_stock_analysis:TAG`

---

## 🖥️ 第三步：服务器部署

### 1. 准备部署目录

```bash
# 创建部署目录
mkdir -p ~/stock-analysis
cd ~/stock-analysis

# 创建数据目录
mkdir -p data logs reports
```

### 2. 配置环境变量

```bash
# 下载环境变量模板
curl -o .env.example https://raw.githubusercontent.com/YOUR_USERNAME/daily_stock_analysis/main/.env.example

# 复制并编辑环境变量
cp .env.example .env
vim .env
```

**必须配置的环境变量**:

```bash
# AI 模型配置（至少配置一个）
GEMINI_API_KEY=your_gemini_key
# 或
OPENAI_API_KEY=your_openai_key
OPENAI_BASE_URL=https://api.deepseek.com/v1
OPENAI_MODEL=deepseek-chat

# 股票列表
STOCK_LIST=600519,300750,000001

# 通知渠道（至少配置一个）
WECHAT_WEBHOOK_URL=https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=xxx
# 或
FEISHU_WEBHOOK_URL=https://open.feishu.cn/open-apis/bot/v2/hook/xxx
# 或
TELEGRAM_BOT_TOKEN=xxx
TELEGRAM_CHAT_ID=xxx

# 可选：搜索服务
TAVILY_API_KEYS=your_tavily_key
```

### 3. 下载 docker-compose 配置

```bash
# 下载生产环境配置
curl -o docker-compose.yml https://raw.githubusercontent.com/YOUR_USERNAME/daily_stock_analysis/main/docker-compose.production.yml
```

### 4. 登录 GitHub Container Registry

```bash
# 创建 GitHub Personal Access Token (PAT)
# 1. 访问 https://github.com/settings/tokens
# 2. 点击 "Generate new token" -> "Generate new token (classic)"
# 3. 勾选 read:packages 权限
# 4. 复制生成的 token

# 登录 GHCR
echo "YOUR_GITHUB_PAT" | docker login ghcr.io -u YOUR_GITHUB_USERNAME --password-stdin
```

### 5. 拉取并启动服务

```bash
# 设置镜像仓库地址（替换为你的 GitHub 用户名）
export GITHUB_REPOSITORY="your-username/daily_stock_analysis"

# 拉取最新镜像
docker compose pull

# 启动所有服务（定时分析 + Web API）
docker compose up -d

# 或只启动定时分析服务
docker compose up -d analyzer

# 或只启动 Web API 服务
docker compose up -d server
```

### 6. 验证部署

```bash
# 查看服务状态
docker compose ps

# 查看日志
docker compose logs -f

# 查看特定服务日志
docker compose logs -f analyzer
docker compose logs -f server

# 检查 Web API（如果启动了 server 服务）
curl http://localhost:8000/api/health
```

访问 Web UI: `http://your-server-ip:8000`

---

## 🔄 运维操作

### 更新镜像

```bash
# 拉取最新镜像
docker compose pull

# 重启服务（会自动使用新镜像）
docker compose up -d

# 或使用单条命令
docker compose pull && docker compose up -d
```

### 查看日志

```bash
# 实时查看所有服务日志
docker compose logs -f

# 查看最近 100 行日志
docker compose logs --tail=100

# 查看特定服务的日志
docker compose logs -f analyzer
docker compose logs -f server
```

### 重启服务

```bash
# 重启所有服务
docker compose restart

# 重启特定服务
docker compose restart analyzer
docker compose restart server
```

### 停止服务

```bash
# 停止所有服务（保留数据）
docker compose down

# 停止并删除数据卷（危险操作！）
docker compose down -v
```

### 备份数据

```bash
# 备份数据目录
tar -czf stock-data-backup-$(date +%Y%m%d).tar.gz data/ logs/ reports/

# 恢复数据
tar -xzf stock-data-backup-20260207.tar.gz
```

### 清理镜像

```bash
# 删除旧镜像
docker image prune -a

# 查看镜像占用空间
docker system df
```

---

## 🐛 故障排查

### 1. 镜像拉取失败

**问题**: `Error response from daemon: pull access denied`

**解决方案**:

```bash
# 1. 确认镜像地址正确
echo $GITHUB_REPOSITORY

# 2. 重新登录 GHCR
docker logout ghcr.io
echo "YOUR_GITHUB_PAT" | docker login ghcr.io -u YOUR_GITHUB_USERNAME --password-stdin

# 3. 确认 PAT 有 read:packages 权限

# 4. 检查镜像是否存在
docker pull ghcr.io/$GITHUB_REPOSITORY:latest
```

### 2. 容器启动失败

**问题**: 容器状态显示 `Exited` 或 `Restarting`

**解决方案**:

```bash
# 查看容器日志
docker compose logs analyzer

# 常见原因:
# - 环境变量未配置: 检查 .env 文件
# - API Key 无效: 验证 GEMINI_API_KEY 或 OPENAI_API_KEY
# - 内存不足: 增加服务器内存或调整 deploy.resources 配置
```

### 3. 数据库锁定问题

**问题**: `database is locked`

**解决方案**:

```bash
# 1. 停止所有服务
docker compose down

# 2. 检查是否有残留进程
ps aux | grep python

# 3. 删除数据库锁文件（如果存在）
rm -f data/stock_analysis.db-journal

# 4. 重启服务
docker compose up -d
```

### 4. 网络连接问题

**问题**: `Connection timeout` 或 API 请求失败

**解决方案**:

```bash
# 1. 检查服务器网络
ping google.com

# 2. 如果需要代理，在 .env 中配置:
USE_PROXY=true
PROXY_HOST=your-proxy-host
PROXY_PORT=your-proxy-port

# 3. 重启服务
docker compose restart
```

### 5. Web UI 无法访问

**问题**: 无法通过浏览器访问 `http://server-ip:8000`

**解决方案**:

```bash
# 1. 检查 server 服务是否运行
docker compose ps server

# 2. 检查端口是否开放
netstat -tuln | grep 8000

# 3. 检查防火墙规则
sudo ufw status
sudo ufw allow 8000

# 4. 检查容器健康状态
docker compose exec server curl http://localhost:8000/api/health
```

---

## 📊 性能优化

### 1. 调整资源限制

编辑 `docker-compose.yml`:

```yaml
deploy:
  resources:
    limits:
      memory: 2G      # 增加内存限制
      cpus: '2.0'     # 增加 CPU 限制
```

### 2. 优化并发数

在 `.env` 中配置:

```bash
MAX_WORKERS=3              # 调整并发线程数
ANALYSIS_DELAY=10          # 增加分析间隔，避免 API 限流
```

### 3. 日志轮转

```yaml
logging:
  options:
    max-size: "50m"        # 增加单个日志文件大小
    max-file: "10"         # 增加日志文件数量
```

---

## 🔐 安全建议

### 1. 环境变量保护

```bash
# 设置 .env 文件权限
chmod 600 .env

# 不要将 .env 提交到 Git
echo ".env" >> .gitignore
```

### 2. 定期更新镜像

```bash
# 设置自动更新 cron 任务
crontab -e

# 添加以下行（每天凌晨 2 点更新）
0 2 * * * cd ~/stock-analysis && docker compose pull && docker compose up -d
```

### 3. 使用 HTTPS

建议使用 Nginx 反向代理并配置 SSL 证书:

```nginx
server {
    listen 443 ssl http2;
    server_name stock.example.com;

    ssl_certificate /path/to/cert.pem;
    ssl_certificate_key /path/to/key.pem;

    location / {
        proxy_pass http://localhost:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

---

## 📞 支持与反馈

如遇到问题，请：

1. 查看 [常见问题](../docs/FAQ.md)
2. 查看 [完整指南](../docs/full-guide.md)
3. 提交 [GitHub Issue](https://github.com/ZhuLinsen/daily_stock_analysis/issues)

---

## 📝 快速参考命令

```bash
# 部署
docker compose pull && docker compose up -d

# 查看日志
docker compose logs -f

# 重启服务
docker compose restart

# 更新服务
docker compose pull && docker compose up -d

# 停止服务
docker compose down

# 备份数据
tar -czf backup-$(date +%Y%m%d).tar.gz data/ logs/

# 清理镜像
docker image prune -a
```

---

**祝部署顺利！📈**
