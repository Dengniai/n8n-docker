# N8N Docker 从零开始部署教程

**创建时间：** 2025年  
**适用系统：** macOS / Linux / Windows  
**难度：** 零基础友好

---

## 目录

1. [环境准备](#一环境准备)
2. [Docker 安装](#二docker-安装)
3. [快速部署（最简单方式）](#三快速部署最简单方式)
4. [Docker Compose 部署（推荐）](#四docker-compose-部署推荐)
5. [配置说明](#五配置说明)
6. [访问和使用](#六访问和使用)
7. [数据持久化](#七数据持久化)
8. [常见问题](#八常见问题)
9. [进阶配置](#九进阶配置)

---

## 一、环境准备

### 系统要求

- **操作系统：** macOS / Linux / Windows
- **Docker：** 版本 ≥ 20.10
- **Docker Compose：** 版本 ≥ v2.x（或使用 `docker compose` 命令）
- **硬件资源：**
  - CPU：至少 2 核
  - 内存：至少 2 GB RAM
  - 硬盘：建议 ≥ 10 GB（随工作流增多而增长）

### 检查系统

在终端运行以下命令检查系统信息：

```bash
# 检查操作系统
uname -a  # Linux/Mac
# 或
systeminfo  # Windows

# 检查是否已安装 Docker
docker --version
```

---

## 二、Docker 安装

### macOS 安装 Docker

1. 访问 Docker 官网：https://www.docker.com/products/docker-desktop
2. 下载 Docker Desktop for Mac
3. 双击安装包，拖拽到 Applications 文件夹
4. 启动 Docker Desktop
5. 等待 Docker 启动完成（菜单栏出现 Docker 图标）

**验证安装：**
```bash
docker --version
docker compose version
```

### Linux 安装 Docker

**Ubuntu/Debian：**
```bash
# 更新软件包索引
sudo apt-get update

# 安装必要的依赖
sudo apt-get install ca-certificates curl gnupg lsb-release

# 添加 Docker 官方 GPG 密钥
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

# 设置仓库
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# 安装 Docker Engine
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-compose-plugin

# 验证安装
docker --version
docker compose version
```

### Windows 安装 Docker

1. 访问 Docker 官网下载 Docker Desktop for Windows
2. 运行安装程序
3. 按照向导完成安装
4. 重启电脑
5. 启动 Docker Desktop

**验证安装：**
```bash
docker --version
docker compose version
```

---

## 三、快速部署（最简单方式）

适合本地测试和快速体验。

### 步骤 1：创建数据卷

```bash
docker volume create n8n_data
```

### 步骤 2：运行 N8N 容器

```bash
docker run -d \
  --name n8n \
  -p 5678:5678 \
  -e GENERIC_TIMEZONE="Asia/Shanghai" \
  -e TZ="Asia/Shanghai" \
  -v n8n_data:/home/node/.n8n \
  docker.n8n.io/n8nio/n8n:latest
```

**参数说明：**
- `-d`：后台运行
- `--name n8n`：容器名称
- `-p 5678:5678`：端口映射（宿主机:容器）
- `-e GENERIC_TIMEZONE`：设置时区
- `-v n8n_data:/home/node/.n8n`：数据持久化

### 步骤 3：访问 N8N

打开浏览器访问：`http://localhost:5678`

首次访问会进入设置页面，创建管理员账号。

### 常用命令

```bash
# 查看容器状态
docker ps

# 查看日志
docker logs n8n

# 停止容器
docker stop n8n

# 启动容器
docker start n8n

# 删除容器（数据不会丢失，因为使用了数据卷）
docker rm n8n
```

---

## 四、Docker Compose 部署（推荐）

适合生产环境和需要更多配置的场景。

### 步骤 1：创建项目目录

```bash
mkdir n8n-docker
cd n8n-docker
```

### 步骤 2：创建 `.env` 文件

```bash
# 创建 .env 文件
cat > .env << EOF
# 域名配置（如果只是本地使用，可以留空或使用 localhost）
DOMAIN_NAME=localhost
SUBDOMAIN=n8n

# 时区设置
GENERIC_TIMEZONE=Asia/Shanghai

# SSL 邮箱（如果需要 HTTPS）
SSL_EMAIL=your-email@example.com

# 基本认证（可选，但推荐生产环境使用）
N8N_BASIC_AUTH_USER=admin
N8N_BASIC_AUTH_PASSWORD=your_secure_password_here
EOF
```

### 步骤 3：创建 `docker-compose.yml` 文件

**基础版本（本地使用）：**

```yaml
services:
  n8n:
    image: docker.n8n.io/n8nio/n8n:latest
    container_name: n8n
    restart: unless-stopped
    ports:
      - "5678:5678"
    environment:
      - N8N_ENFORCE_SETTINGS_FILE_PERMISSIONS=true
      - GENERIC_TIMEZONE=${GENERIC_TIMEZONE}
      - TZ=${GENERIC_TIMEZONE}
      # 基本认证（可选）
      - N8N_BASIC_AUTH_ACTIVE=true
      - N8N_BASIC_AUTH_USER=${N8N_BASIC_AUTH_USER}
      - N8N_BASIC_AUTH_PASSWORD=${N8N_BASIC_AUTH_PASSWORD}
    volumes:
      - n8n_data:/home/node/.n8n
      - ./local-files:/files
    healthcheck:
      test: ["CMD", "wget", "--spider", "-q", "http://localhost:5678/healthz"]
      interval: 30s
      timeout: 10s
      retries: 3

volumes:
  n8n_data:
    driver: local
```

**完整版本（生产环境，带 PostgreSQL）：**

```yaml
services:
  postgres:
    image: postgres:15
    container_name: n8n_postgres
    restart: unless-stopped
    environment:
      - POSTGRES_DB=n8n
      - POSTGRES_USER=n8n
      - POSTGRES_PASSWORD=n8n_password
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U n8n"]
      interval: 10s
      timeout: 5s
      retries: 5

  n8n:
    image: docker.n8n.io/n8nio/n8n:latest
    container_name: n8n
    restart: unless-stopped
    ports:
      - "5678:5678"
    environment:
      - N8N_ENFORCE_SETTINGS_FILE_PERMISSIONS=true
      - DB_TYPE=postgresdb
      - DB_POSTGRESDB_HOST=postgres
      - DB_POSTGRESDB_PORT=5432
      - DB_POSTGRESDB_DATABASE=n8n
      - DB_POSTGRESDB_USER=n8n
      - DB_POSTGRESDB_PASSWORD=n8n_password
      - GENERIC_TIMEZONE=${GENERIC_TIMEZONE}
      - TZ=${GENERIC_TIMEZONE}
      - N8N_BASIC_AUTH_ACTIVE=true
      - N8N_BASIC_AUTH_USER=${N8N_BASIC_AUTH_USER}
      - N8N_BASIC_AUTH_PASSWORD=${N8N_BASIC_AUTH_PASSWORD}
      # 加密密钥（重要：生产环境必须设置）
      - N8N_ENCRYPTION_KEY=your-32-character-encryption-key-here
    volumes:
      - n8n_data:/home/node/.n8n
      - ./local-files:/files
    depends_on:
      postgres:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "wget", "--spider", "-q", "http://localhost:5678/healthz"]
      interval: 30s
      timeout: 10s
      retries: 3

volumes:
  n8n_data:
    driver: local
  postgres_data:
    driver: local
```

**注意：** Docker Compose v2.x 不再需要 `version` 字段，如果看到警告可以忽略或删除该行。

### 步骤 4：创建本地文件目录

```bash
mkdir local-files
```

### 步骤 5：启动服务

```bash
# 启动服务（后台运行）
docker compose up -d

# 查看日志
docker compose logs -f

# 查看服务状态
docker compose ps
```

### 步骤 6：访问 N8N

打开浏览器访问：`http://localhost:5678`

---

## 五、配置说明

### 重要环境变量

| 变量名 | 说明 | 示例 |
|--------|------|------|
| `GENERIC_TIMEZONE` | 时区设置 | `Asia/Shanghai` |
| `TZ` | 系统时区 | `Asia/Shanghai` |
| `N8N_BASIC_AUTH_ACTIVE` | 启用基本认证 | `true` |
| `N8N_BASIC_AUTH_USER` | 基本认证用户名 | `admin` |
| `N8N_BASIC_AUTH_PASSWORD` | 基本认证密码 | `your_password` |
| `N8N_ENCRYPTION_KEY` | 加密密钥（32字符） | `your-32-char-key-here` |
| `DB_TYPE` | 数据库类型 | `postgresdb` |
| `N8N_HOST` | N8N 主机地址 | `n8n.example.com` |
| `N8N_PORT` | N8N 端口 | `5678` |
| `N8N_PROTOCOL` | 协议 | `https` |
| `WEBHOOK_URL` | Webhook URL | `https://n8n.example.com/` |

### 生成加密密钥

```bash
# 方法1：使用 openssl
openssl rand -base64 24

# 方法2：使用 Python
python3 -c "import secrets; print(secrets.token_urlsafe(24))"

# 方法3：使用 Node.js
node -e "console.log(require('crypto').randomBytes(24).toString('base64'))"
```

---

## 六、访问和使用

### 首次设置

1. 访问 `http://localhost:5678`
2. 创建管理员账号（邮箱和密码）
3. 完成初始设置

### 基本操作

1. **创建工作流：** 点击 "New Workflow"
2. **添加节点：** 从左侧节点库拖拽
3. **连接节点：** 拖拽连接线
4. **配置节点：** 点击节点进行配置
5. **测试工作流：** 点击 "Execute Workflow"
6. **激活工作流：** 点击右上角开关

---

## 七、数据持久化

### 数据存储位置

- **工作流数据：** 存储在 `/home/node/.n8n` 目录
- **数据库：** SQLite（默认）或 PostgreSQL
- **文件：** 可通过 `/files` 目录访问宿主机文件

### 备份数据

```bash
# 备份数据卷
docker run --rm -v n8n_data:/data -v $(pwd):/backup alpine tar czf /backup/n8n-backup-$(date +%Y%m%d).tar.gz /data

# 恢复数据
docker run --rm -v n8n_data:/data -v $(pwd):/backup alpine tar xzf /backup/n8n-backup-20250101.tar.gz -C /
```

---

## 八、常见问题

### 1. 容器无法启动

**检查日志：**
```bash
docker logs n8n
```

**常见原因：**
- 端口被占用：修改 `-p 5678:5678` 为其他端口
- 权限问题：检查数据卷权限

### 2. 数据丢失

**原因：** 没有使用数据卷持久化

**解决：** 确保 `docker-compose.yml` 中有：
```yaml
volumes:
  - n8n_data:/home/node/.n8n
```

### 3. 无法访问 Web 界面

**检查：**
```bash
# 检查容器是否运行
docker ps

# 检查端口是否监听
netstat -an | grep 5678  # Linux/Mac
netstat -an | findstr 5678  # Windows
```

### 4. 时区不正确

**解决：** 在环境变量中设置：
```yaml
environment:
  - GENERIC_TIMEZONE=Asia/Shanghai
  - TZ=Asia/Shanghai
```

### 5. 凭证无法解密

**原因：** 加密密钥改变

**解决：** 使用固定的 `N8N_ENCRYPTION_KEY` 环境变量

### 6. 警告信息

**"No services to build" 警告：**
- 这是信息性警告，表示所有服务都使用预构建镜像
- 可以安全忽略，不影响功能

**"version is obsolete" 警告：**
- Docker Compose v2.x 不再需要 `version` 字段
- 可以从 `docker-compose.yml` 中删除 `version: '3.8'` 这一行

---

## 九、进阶配置

### 使用 HTTPS（通过 Traefik）

创建 `docker-compose.yml`：

```yaml
services:
  traefik:
    image: traefik:latest
    container_name: traefik
    command:
      - "--api.insecure=true"
      - "--providers.docker=true"
      - "--providers.docker.exposedbydefault=false"
      - "--entrypoints.web.address=:80"
      - "--entrypoints.websecure.address=:443"
      - "--certificatesresolvers.le.acme.tlschallenge=true"
      - "--certificatesresolvers.le.acme.email=${SSL_EMAIL}"
      - "--certificatesresolvers.le.acme.storage=/letsencrypt/acme.json"
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - traefik_data:/letsencrypt
      - /var/run/docker.sock:/var/run/docker.sock

  n8n:
    image: docker.n8n.io/n8nio/n8n:latest
    container_name: n8n
    restart: unless-stopped
    environment:
      - N8N_HOST=${SUBDOMAIN}.${DOMAIN_NAME}
      - N8N_PORT=5678
      - N8N_PROTOCOL=https
      - WEBHOOK_URL=https://${SUBDOMAIN}.${DOMAIN_NAME}/
      - GENERIC_TIMEZONE=${GENERIC_TIMEZONE}
      - TZ=${GENERIC_TIMEZONE}
    volumes:
      - n8n_data:/home/node/.n8n
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.n8n.rule=Host(`${SUBDOMAIN}.${DOMAIN_NAME}`)"
      - "traefik.http.routers.n8n.entrypoints=websecure"
      - "traefik.http.routers.n8n.tls.certresolver=le"

volumes:
  n8n_data:
  traefik_data:
```

### 更新 N8N

```bash
# 拉取最新镜像
docker compose pull

# 停止并删除旧容器
docker compose down

# 启动新容器
docker compose up -d
```

### 查看资源使用

```bash
# 查看容器资源使用
docker stats n8n

# 查看数据卷大小
docker system df -v
```

---

## 十、下一步学习

1. **创建第一个工作流：** 从模板开始
2. **学习节点使用：** HTTP Request、IF、Code 等
3. **配置凭据：** 连接第三方服务
4. **使用表达式：** 动态数据处理
5. **错误处理：** 配置重试和错误处理

### 推荐学习资源

- **官方文档：** https://docs.n8n.io
- **中文文档：** https://n8ndocs.com
- **视频教程：** B站搜索 "n8n 教程"
- **社区：** https://community.n8n.io

---

## 快速参考命令

```bash
# 启动服务
docker compose up -d

# 停止服务
docker compose stop

# 重启服务
docker compose restart

# 查看日志
docker compose logs -f n8n

# 进入容器
docker exec -it n8n sh

# 备份数据
docker run --rm -v n8n_data:/data -v $(pwd):/backup alpine tar czf /backup/n8n-backup.tar.gz /data

# 更新镜像
docker compose pull && docker compose up -d
```

---

**祝你使用愉快！如有问题，请查看官方文档或社区论坛。** 🚀
