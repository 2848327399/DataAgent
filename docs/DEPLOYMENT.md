# 生产服务器部署

本文档使用 Docker Compose 在一台 Linux 服务器上部署 DataAgent 的前端、后端和管理数据库。
浏览器只访问前端 Nginx，Java 后端与 MySQL 均不直接暴露到公网。

## 1. 前置条件

- 一台 Linux 服务器，建议至少 4 核 CPU、8 GB 内存和 30 GB 可用磁盘
- 已安装 Git、Docker Engine 与 Docker Compose v2
- 安全组或防火墙开放业务端口（默认 `80`）
- 若使用域名和 HTTPS，提前将域名解析到服务器

Python 分析会创建临时沙盒容器，因此后端容器会挂载宿主机的
`/var/run/docker.sock`。这相当于授予后端很高的宿主机权限，只应部署可信代码，并应在
专用服务器或隔离的虚拟机上运行。若只使用 SQL 分析，也建议结合实际需求移除该挂载并禁用
Python 工作流。

## 2. 准备配置

在服务器克隆仓库并进入项目根目录：

```bash
git clone <your-repository-url> data-agent
cd data-agent
cp .env.example .env
```

编辑 `.env`，至少替换下面两个密码：

```dotenv
MYSQL_PASSWORD=生成的长随机密码
MYSQL_ROOT_PASSWORD=另一个长随机密码
```

可以使用 `openssl rand -hex 32` 分别生成密码。`.env` 已被 Git 忽略，不要把真实密钥提交到
仓库。大模型提供商、模型名和 API Key 在应用启动后通过“模型配置”页面录入。

如果宿主机已有 HTTPS 反向代理，把下面两项改为：

```dotenv
APP_BIND_ADDRESS=127.0.0.1
APP_PORT=3000
```

## 3. 构建并启动

以下命令均在项目根目录运行：

```bash
docker compose --env-file .env -f docker-file/docker-compose.yml config
docker compose --env-file .env -f docker-file/docker-compose.yml build --pull
docker compose --env-file .env -f docker-file/docker-compose.yml up -d
```

首次构建需要下载 Maven、pnpm、Java 和 Node.js 依赖，耗时取决于服务器网络。首次启动时，
MySQL 会自动执行 `schema.sql` 和 `data.sql`。初始化脚本仅在数据库卷为空时执行。

查看运行状态与日志：

```bash
docker compose --env-file .env -f docker-file/docker-compose.yml ps
docker compose --env-file .env -f docker-file/docker-compose.yml logs -f --tail=200 backend frontend
```

三个服务均显示为 `healthy` 后，访问 `http://服务器IP/`（或 `.env` 中配置的端口）。健康检查：

```bash
curl -fsS http://127.0.0.1:${APP_PORT:-80}/healthz
```

返回 `ok` 表示前端入口可用。之后进入“模型配置”添加模型，再添加并初始化需要分析的数据源。

## 4. 域名与 HTTPS

项目内置的 Nginx 负责静态页面和 API/SSE 转发，不负责签发 TLS 证书。生产环境建议在宿主机
或负载均衡器上终止 HTTPS，再代理到 `127.0.0.1:3000`。外层代理必须：

- 保留 `Host`、`X-Forwarded-For` 和 `X-Forwarded-Proto` 请求头
- 关闭 `/api/` 的响应缓冲，以保证 SSE 实时输出
- 将读取超时设为足够长（项目内层配置为 24 小时）
- 将请求体上限设为至少 10 MB

当前管理 API 除已发布智能体的流式调用外没有后台登录保护，不应直接暴露给不受信任的公网。
正式上线前应通过 VPN、零信任网关或带身份认证的外层反向代理限制管理页面访问。

## 5. 更新与回滚

更新代码后重新构建并滚动替换：

```bash
git pull --ff-only
docker compose --env-file .env -f docker-file/docker-compose.yml build
docker compose --env-file .env -f docker-file/docker-compose.yml up -d
docker image prune -f
```

需要回滚时，切回已经验证的 Git 提交，设置一个明确的 `IMAGE_TAG`，再执行相同的构建和启动
命令。不要使用 `docker compose down -v`，它会删除数据库、上传文件和本地向量数据。

## 6. 备份

备份管理数据库：

```bash
docker compose --env-file .env -f docker-file/docker-compose.yml exec -T mysql \
  sh -c 'exec mysqldump -u"$MYSQL_USER" -p"$MYSQL_PASSWORD" "$MYSQL_DATABASE"' \
  > "data-agent-$(date +%F-%H%M%S).sql"
```

同时应定期备份 Compose 卷 `backend_uploads` 和 `backend_vectorstore`。数据库备份与上传文件应
使用同一个时间点的快照，并存放到服务器之外。

## 7. 常用排障命令

```bash
# 后端启动日志
docker compose --env-file .env -f docker-file/docker-compose.yml logs --tail=300 backend

# 数据库健康状态
docker compose --env-file .env -f docker-file/docker-compose.yml exec mysql \
  mysqladmin ping -h 127.0.0.1 -u root -p

# 查看 Python 任务沙盒（任务结束后通常不应残留）
docker ps --format '{{.Names}}' | grep '^dataagent-sandbox-'
```

若后端无法启动，优先检查 `.env` 密码是否完整、MySQL 是否健康、服务器能否访问 Maven/Node
依赖源和模型服务。若 Python 步骤失败，再检查 Docker socket 权限、沙盒镜像拉取状态和 PyPI
网络连通性。
