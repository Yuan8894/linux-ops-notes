# 第二阶段：工具链实战

> **适用环境**：Rocky Linux 9.5（上海/雅加达）、EulerOS 2.0（广州）、Debian（北京）
> **学习周期**：4月中 → 5月中（约 4 周）
> **前置条件**：已完成第一阶段 Linux 系统管理核心（见 `phase1-linux-sysadmin.md`）

---

## 目录

1. [Docker 进阶](#1-docker-进阶)
2. [CI/CD 基础](#2-cicd-基础)
3. [监控与告警](#3-监控与告警)
4. [Nginx 深入](#4-nginx-深入)
5. [综合练习项目](#5-综合练习项目)

---

## 1. Docker 进阶

> 你已经有 Docker 基础（Nginx Proxy Manager / Uptime Kuma / Portainer / gh-proxy），本阶段重点是理解原理 + 手写配置。

### 1.1 Docker 核心概念回顾

```
镜像（Image）    → 只读模板，类似操作系统 ISO
容器（Container） → 镜像的运行实例，类似一台虚拟机
仓库（Registry）  → 存放镜像的地方（Docker Hub、阿里云 ACR 等）
```

**容器 vs 虚拟机（面试必考）**：

| 对比 | 容器（Docker） | 虚拟机（VM） |
|------|---------------|-------------|
| 隔离级别 | 进程级（共享宿主机内核） | 硬件级（独立内核） |
| 启动速度 | 秒级 | 分钟级 |
| 资源占用 | MB 级（只打包应用和依赖） | GB 级（完整 OS） |
| 性能 | 接近原生 | 有虚拟化开销 |
| 安全性 | 较弱（共享内核，逃逸风险） | 较强（完全隔离） |
| 典型场景 | 微服务部署、CI/CD | 多租户隔离、不同 OS |

**Docker 架构**：
```
客户端（docker CLI）  →  Docker Daemon（dockerd）  →  容器运行时（containerd + runc）
                              ↕
                       镜像仓库（Registry）
```

### 1.2 Docker 常用命令

#### 镜像操作

```bash
# 拉取镜像
docker pull nginx:1.24               # 拉取指定版本
docker pull nginx:latest              # 拉取最新版（生产不推荐用 latest）

# 查看本地镜像
docker images                         # 列出所有镜像
docker images --format "table {{.Repository}}\t{{.Tag}}\t{{.Size}}"  # 格式化输出

# 删除镜像
docker rmi nginx:1.24                 # 删除指定镜像
docker image prune                    # 删除所有悬空镜像（无标签的）
docker image prune -a                 # 删除所有未使用的镜像（慎用）

# 查看镜像详情
docker inspect nginx:1.24             # 查看镜像完整元信息（JSON）
docker history nginx:1.24             # 查看镜像构建层历史
```

#### 容器操作

```bash
# 运行容器
docker run -d --name my-nginx -p 80:80 nginx:1.24
#          |     |            |           |
#        后台运行  容器名      端口映射     镜像

# 常用参数
docker run -d \
    --name web \
    -p 8080:80 \                      # 宿主机8080 → 容器80
    -v /data/html:/usr/share/nginx/html \  # 目录挂载
    -e NGINX_HOST=example.com \       # 环境变量
    --restart unless-stopped \        # 重启策略
    --memory 512m \                   # 内存限制
    --cpus 1.0 \                      # CPU 限制
    nginx:1.24

# 容器管理
docker ps                             # 查看运行中的容器
docker ps -a                          # 查看所有容器（含已停止的）
docker stop my-nginx                  # 优雅停止
docker start my-nginx                 # 启动
docker restart my-nginx               # 重启
docker rm my-nginx                    # 删除已停止的容器
docker rm -f my-nginx                 # 强制删除运行中的容器

# 进入容器
docker exec -it my-nginx /bin/bash    # 进入容器 shell（交互式）
docker exec my-nginx cat /etc/nginx/nginx.conf  # 在容器内执行命令

# 查看日志
docker logs my-nginx                  # 查看全部日志
docker logs -f my-nginx               # 实时跟踪日志（tail -f 效果）
docker logs --tail 50 my-nginx        # 最后 50 行
docker logs --since "2h" my-nginx     # 最近 2 小时

# 查看容器资源使用
docker stats                          # 实时资源监控（CPU/内存/网络/IO）
docker stats --no-stream              # 只显示一次
docker top my-nginx                   # 查看容器内进程

# 复制文件
docker cp my-nginx:/etc/nginx/nginx.conf ./nginx.conf   # 从容器复制到宿主机
docker cp ./index.html my-nginx:/usr/share/nginx/html/  # 从宿主机复制到容器
```

#### 重启策略

| 策略 | 说明 |
|------|------|
| `no` | 默认，不自动重启 |
| `on-failure` | 非正常退出时重启（exit code ≠ 0） |
| `unless-stopped` | 除非手动 stop，否则总是重启（**推荐**） |
| `always` | 总是重启（包括手动 stop 后 Docker 重启时） |

---

### 1.3 Dockerfile 编写

> Dockerfile 是构建 Docker 镜像的脚本，面试常考各指令的区别。

#### 基本示例

```dockerfile
# 基础镜像（尽量选 slim/alpine 减小体积）
FROM python:3.11-slim

# 元数据标签
LABEL maintainer="your-email@example.com"
LABEL description="Python ops tool"

# 设置工作目录（不存在会自动创建）
WORKDIR /app

# 先复制依赖文件（利用 Docker 层缓存）
COPY requirements.txt .

# 安装依赖
RUN pip install --no-cache-dir -r requirements.txt

# 复制应用代码（放在安装依赖之后，代码变动不会重新安装依赖）
COPY . .

# 暴露端口（仅声明，不实际映射）
EXPOSE 5000

# 创建非 root 用户运行应用（安全最佳实践）
RUN useradd -m appuser
USER appuser

# 启动命令
CMD ["python", "app.py"]
```

#### 常用指令详解

| 指令 | 说明 | 示例 |
|------|------|------|
| `FROM` | 基础镜像，必须是第一条 | `FROM ubuntu:22.04` |
| `RUN` | 构建时执行命令（每条 RUN 增加一层） | `RUN apt update && apt install -y curl` |
| `COPY` | 复制本地文件到镜像 | `COPY ./app /app` |
| `ADD` | 类似 COPY，但支持自动解压 tar 和远程 URL | `ADD app.tar.gz /app` |
| `WORKDIR` | 设置工作目录 | `WORKDIR /app` |
| `ENV` | 设置环境变量 | `ENV NODE_ENV=production` |
| `EXPOSE` | 声明端口（文档作用） | `EXPOSE 8080` |
| `CMD` | 容器启动时默认命令（可被 docker run 覆盖） | `CMD ["nginx", "-g", "daemon off;"]` |
| `ENTRYPOINT` | 容器启动入口（不易被覆盖） | `ENTRYPOINT ["python"]` |
| `USER` | 指定运行用户 | `USER appuser` |
| `VOLUME` | 声明挂载点 | `VOLUME ["/data"]` |
| `ARG` | 构建时变量（仅构建阶段有效） | `ARG VERSION=1.0` |

**CMD vs ENTRYPOINT（面试高频）**：

```dockerfile
# CMD：可被 docker run 的命令覆盖
CMD ["python", "app.py"]
# docker run myapp                    → 执行 python app.py
# docker run myapp python test.py     → 执行 python test.py（CMD 被覆盖）

# ENTRYPOINT：固定入口，docker run 的参数会追加
ENTRYPOINT ["python"]
CMD ["app.py"]                        # 默认参数
# docker run myapp                    → 执行 python app.py
# docker run myapp test.py            → 执行 python test.py（追加参数）
```

**COPY vs ADD**：
- 优先使用 `COPY`（语义清晰）
- 只在需要自动解压 tar 时用 `ADD`
- 不要用 `ADD` 下载远程文件（用 `RUN curl` 代替，可以在同一层清理）

#### 多阶段构建（Multi-stage Build）

> 用于减小最终镜像体积：编译阶段用完整工具链，运行阶段只保留产物。

```dockerfile
# ===== 第一阶段：编译 =====
FROM golang:1.21 AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -o server .

# ===== 第二阶段：运行 =====
FROM alpine:3.18
RUN apk --no-cache add ca-certificates
WORKDIR /app
# 从 builder 阶段复制编译好的二进制文件
COPY --from=builder /app/server .
EXPOSE 8080
CMD ["./server"]
```

| 对比 | 不用多阶段 | 用多阶段 |
|------|-----------|---------|
| 最终镜像大小 | ~1GB（包含 Go 工具链） | ~15MB（只有二进制 + Alpine） |
| 安全性 | 包含编译工具（攻击面大） | 最小化（攻击面小） |

#### 镜像优化技巧

```dockerfile
# ❌ 反面教材：每个 RUN 一层，且不清理缓存
RUN apt update
RUN apt install -y curl
RUN apt install -y wget
RUN rm -rf /var/lib/apt/lists/*

# ✅ 正确做法：合并 RUN，同一层内清理
RUN apt update && \
    apt install -y --no-install-recommends curl wget && \
    rm -rf /var/lib/apt/lists/*
```

**镜像体积优化清单**：
- 使用 slim/alpine 基础镜像
- 合并 RUN 指令减少层数
- 使用多阶段构建
- 安装依赖时加 `--no-install-recommends`（apt）或 `--no-cache-dir`（pip）
- 在同一 RUN 层清理缓存和临时文件
- 使用 `.dockerignore` 排除不需要的文件

```
# .dockerignore 示例
.git
.gitignore
README.md
__pycache__
*.pyc
.env
node_modules
```

#### 构建与推送镜像

```bash
# 构建镜像
docker build -t myapp:v1.0 .                     # 从当前目录的 Dockerfile 构建
docker build -t myapp:v1.0 -f Dockerfile.prod .   # 指定 Dockerfile
docker build --no-cache -t myapp:v1.0 .            # 不使用缓存

# 标记镜像（用于推送到仓库）
docker tag myapp:v1.0 registry.example.com/myapp:v1.0

# 推送镜像
docker push registry.example.com/myapp:v1.0

# 导出/导入（离线传输镜像）
docker save myapp:v1.0 -o myapp.tar               # 导出为 tar
docker load -i myapp.tar                           # 从 tar 导入
```

---

### 练习 1：Dockerfile

```bash
# 在上海 VPS 上练习：

# 任务：为一个 Python Flask 应用编写 Dockerfile 并构建

# 1. 创建项目目录
mkdir -p ~/docker-lab/flask-app && cd ~/docker-lab/flask-app

# 2. 创建应用文件
cat > app.py << 'EOF'
from flask import Flask, jsonify
import platform
import datetime

app = Flask(__name__)

@app.route('/')
def index():
    return jsonify({
        "message": "Hello from Docker!",
        "hostname": platform.node(),
        "time": datetime.datetime.now().isoformat()
    })

@app.route('/health')
def health():
    return jsonify({"status": "healthy"})

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
EOF

cat > requirements.txt << 'EOF'
flask==3.0.0
EOF

# 3. 编写 Dockerfile
cat > Dockerfile << 'EOF'
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
RUN useradd -m appuser
USER appuser
EXPOSE 5000
CMD ["python", "app.py"]
EOF

# 4. 创建 .dockerignore
cat > .dockerignore << 'EOF'
__pycache__
*.pyc
.git
EOF

# 5. 构建并运行
docker build -t flask-app:v1.0 .
docker run -d --name flask-test -p 5000:5000 flask-app:v1.0
curl http://localhost:5000
curl http://localhost:5000/health

# 6. 查看镜像大小
docker images flask-app

# 7. 清理
docker rm -f flask-test
```

---

### 1.4 Docker 网络

#### 网络模式

| 模式 | 说明 | 适用场景 |
|------|------|---------|
| `bridge` | 默认模式，容器通过虚拟网桥通信 | 大多数场景 |
| `host` | 容器直接使用宿主机网络（无端口映射） | 性能敏感场景 |
| `none` | 无网络 | 安全隔离 |
| `overlay` | 跨主机容器通信（Docker Swarm） | 集群 |

```bash
# 查看网络
docker network ls
docker network inspect bridge         # 查看 bridge 网络详情

# 创建自定义网络（推荐：容器间可通过容器名通信）
docker network create mynet
docker run -d --name web --network mynet nginx
docker run -d --name app --network mynet python:3.11-slim sleep 3600
# app 容器内可以通过 `ping web` 访问 nginx（自动 DNS 解析）

# host 模式
docker run -d --name web-host --network host nginx
# 直接监听宿主机 80 端口，不需要 -p

# 容器间通信测试
docker exec app ping -c 3 web         # 自定义网络中可通过容器名通信
```

> **重要**：默认 bridge 网络中，容器间只能通过 IP 通信；自定义 bridge 网络中，容器间可通过容器名通信（内置 DNS）。**生产环境推荐使用自定义网络**。

---

### 1.5 数据卷管理

```bash
# 创建数据卷
docker volume create mydata
docker volume ls                       # 列出所有卷
docker volume inspect mydata           # 查看卷详情

# 使用数据卷
docker run -d --name db \
    -v mydata:/var/lib/mysql \         # 命名卷挂载
    -e MYSQL_ROOT_PASSWORD=secret \
    mysql:8.0

# 使用绑定挂载（bind mount）
docker run -d --name web \
    -v /data/html:/usr/share/nginx/html:ro \   # :ro = 只读挂载
    nginx

# 清理
docker volume rm mydata                # 删除指定卷
docker volume prune                    # 删除所有未使用的卷
```

**数据卷 vs 绑定挂载**：

| 对比 | 数据卷（Volume） | 绑定挂载（Bind Mount） |
|------|-----------------|----------------------|
| 管理方式 | Docker 管理（`docker volume`） | 直接操作宿主机目录 |
| 存储位置 | `/var/lib/docker/volumes/` | 宿主机任意路径 |
| 可移植性 | 好（不依赖宿主机目录结构） | 差（依赖宿主机路径） |
| 推荐场景 | 数据库数据、持久化数据 | 配置文件、开发环境代码 |

---

### 1.6 Docker Compose

> Docker Compose 用于定义和管理多容器应用，用一个 YAML 文件描述所有服务。

#### 安装

```bash
# Rocky Linux 9 安装 docker-compose-plugin（V2，推荐）
dnf install docker-compose-plugin -y
docker compose version                # V2 命令格式：docker compose（无横杠）

# 如果用的是独立安装的 docker-compose（V1）
# docker-compose version              # V1 命令格式：docker-compose（有横杠）
```

#### docker-compose.yml 基本结构

```yaml
# docker-compose.yml
version: "3.8"                         # Compose 文件格式版本

services:                              # 定义各个服务
  web:                                 # 服务名
    image: nginx:1.24                  # 使用的镜像
    ports:
      - "80:80"                        # 端口映射
    volumes:
      - ./html:/usr/share/nginx/html   # 绑定挂载
    depends_on:
      - app                            # 依赖关系（app 先启动）
    restart: unless-stopped

  app:
    build: ./app                       # 从 Dockerfile 构建
    ports:
      - "5000:5000"
    environment:                       # 环境变量
      - DB_HOST=db
      - DB_PORT=3306
    depends_on:
      - db

  db:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: secret
      MYSQL_DATABASE: myapp
    volumes:
      - db-data:/var/lib/mysql         # 命名卷
    restart: unless-stopped

volumes:                               # 声明命名卷
  db-data:

networks:                              # 自定义网络（可选，Compose 自动创建默认网络）
  default:
    driver: bridge
```

#### 常用命令

```bash
# 启动
docker compose up -d                   # 后台启动所有服务
docker compose up -d --build           # 重新构建镜像后启动
docker compose up -d web               # 只启动指定服务

# 查看
docker compose ps                      # 查看服务状态
docker compose logs                    # 查看所有服务日志
docker compose logs -f web             # 实时跟踪 web 服务日志
docker compose top                     # 查看各服务进程

# 管理
docker compose stop                    # 停止所有服务
docker compose start                   # 启动已停止的服务
docker compose restart web             # 重启指定服务
docker compose down                    # 停止并删除容器 + 网络
docker compose down -v                 # 同上 + 删除数据卷（慎用）

# 扩容
docker compose up -d --scale web=3     # 将 web 服务扩展到 3 个实例

# 执行命令
docker compose exec web bash           # 进入 web 容器
docker compose exec db mysql -uroot -p # 连接数据库
```

#### docker-compose.yml 字段速查

| 字段 | 说明 |
|------|------|
| `image` | 使用的镜像 |
| `build` | 构建上下文（含 Dockerfile 的目录） |
| `ports` | 端口映射 `"宿主机:容器"` |
| `volumes` | 挂载卷 `"宿主机路径:容器路径"` |
| `environment` | 环境变量 |
| `env_file` | 从文件加载环境变量 |
| `depends_on` | 服务依赖（控制启动顺序） |
| `restart` | 重启策略 |
| `networks` | 加入的网络 |
| `command` | 覆盖默认启动命令 |
| `healthcheck` | 健康检查配置 |
| `deploy.resources` | 资源限制（CPU/内存） |

---

### 练习 2：Docker Compose

```bash
# 在上海 VPS 上练习：搭建 Nginx + WordPress + MySQL

mkdir -p ~/docker-lab/wordpress && cd ~/docker-lab/wordpress

cat > docker-compose.yml << 'EOF'
version: "3.8"

services:
  nginx:
    image: nginx:1.24
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/conf.d/default.conf:ro
    depends_on:
      - wordpress
    restart: unless-stopped

  wordpress:
    image: wordpress:6-fpm
    environment:
      WORDPRESS_DB_HOST: db
      WORDPRESS_DB_USER: wordpress
      WORDPRESS_DB_PASSWORD: wp_password
      WORDPRESS_DB_NAME: wordpress
    volumes:
      - wp-data:/var/www/html
    depends_on:
      - db
    restart: unless-stopped

  db:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: root_password
      MYSQL_DATABASE: wordpress
      MYSQL_USER: wordpress
      MYSQL_PASSWORD: wp_password
    volumes:
      - db-data:/var/lib/mysql
    restart: unless-stopped

volumes:
  wp-data:
  db-data:
EOF

# Nginx 反向代理配置
cat > nginx.conf << 'EOF'
server {
    listen 80;
    server_name _;

    root /var/www/html;
    index index.php index.html;

    location / {
        try_files $uri $uri/ /index.php?$args;
    }

    location ~ \.php$ {
        fastcgi_pass wordpress:9000;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
    }
}
EOF

# 启动
docker compose up -d

# 验证
docker compose ps
curl -I http://localhost

# 查看日志
docker compose logs -f

# 清理（完成练习后）
docker compose down -v
```

---

### 1.7 Docker 日志与清理

```bash
# 查看 Docker 磁盘使用
docker system df                       # 概览（镜像/容器/卷/构建缓存）
docker system df -v                    # 详细

# 一键清理（删除所有停止的容器、悬空镜像、未使用的网络）
docker system prune                    # 基本清理
docker system prune -a --volumes       # 深度清理（慎用，会删除所有未使用镜像和卷）

# 限制容器日志大小（daemon.json 全局配置）
cat > /etc/docker/daemon.json << 'EOF'
{
    "log-driver": "json-file",
    "log-opts": {
        "max-size": "10m",
        "max-file": "3"
    },
    "registry-mirrors": [
        "https://mirror.ccs.tencentyun.com"
    ]
}
EOF
systemctl restart docker
```

---

## 2. CI/CD 基础

### 2.1 Git 工作流

#### Git 分支策略

```
main（主分支）
 ├── develop（开发分支）
 │    ├── feature/add-login      # 功能分支
 │    ├── feature/add-dashboard  # 功能分支
 │    └── bugfix/fix-auth        # 修复分支
 └── release/v1.0               # 发布分支
```

#### 常用 Git 操作

```bash
# 分支操作
git branch                             # 查看本地分支
git branch -a                          # 查看所有分支（含远程）
git checkout -b feature/add-api        # 创建并切换到新分支
git switch -c feature/add-api          # 同上（Git 2.23+ 新语法，推荐）

# 合并
git checkout main
git merge feature/add-api              # 合并分支到 main（保留合并记录）
git merge --no-ff feature/add-api      # 强制创建合并 commit（推荐）

# Rebase（变基）
git checkout feature/add-api
git rebase main                        # 将 feature 分支变基到 main 最新
# 注意：不要对已推送到远程的公共分支 rebase

# 解决冲突
# 冲突时 Git 会标记冲突文件，手动编辑后：
git add <冲突文件>
git rebase --continue                  # 如果在 rebase 中
# 或
git merge --continue                   # 如果在 merge 中

# 暂存（stash）
git stash                              # 暂存当前修改
git stash list                         # 查看暂存列表
git stash pop                          # 恢复最近的暂存

# 查看历史
git log --oneline --graph              # 简洁图形化显示
git log --oneline -10                  # 最近 10 条
git diff main..feature/add-api         # 比较两个分支的差异
```

**Merge vs Rebase（面试考点）**：

| 对比 | Merge | Rebase |
|------|-------|--------|
| 历史记录 | 保留完整分支历史（有合并 commit） | 线性历史（更干净） |
| 适用场景 | 合并公共分支、团队协作 | 整理个人 feature 分支 |
| 安全性 | 不改变已有 commit | 会重写 commit 历史 |
| 黄金法则 | — | 不要 rebase 已推送的公共分支 |

---

### 2.2 GitHub Actions

> GitHub Actions 是 GitHub 内置的 CI/CD 工具，通过 YAML 文件定义工作流。

#### 核心概念

```
Workflow（工作流）  → 一个 .yml 文件，定义自动化流程
  └── Job（作业）   → 一组步骤，运行在同一个 Runner 上
       └── Step（步骤） → 单个任务（运行命令或使用 Action）

Event（事件）       → 触发工作流的条件（push、PR、定时等）
Runner（运行器）    → 执行 Job 的服务器（GitHub 提供免费的）
Action（动作）      → 可复用的步骤模块（GitHub Marketplace 有大量现成的）
```

#### 基本工作流示例

```yaml
# .github/workflows/ci.yml
name: CI Pipeline                      # 工作流名称

on:                                    # 触发条件
  push:
    branches: [main, develop]          # push 到 main 或 develop 时触发
  pull_request:
    branches: [main]                   # 向 main 提 PR 时触发

jobs:
  lint-and-test:                       # Job 名称
    runs-on: ubuntu-latest             # 运行环境

    steps:
      - name: Checkout code            # 步骤1：拉取代码
        uses: actions/checkout@v4

      - name: Set up Python            # 步骤2：安装 Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'

      - name: Install dependencies     # 步骤3：安装依赖
        run: |
          pip install -r requirements.txt
          pip install flake8 pytest

      - name: Lint with flake8         # 步骤4：代码检查
        run: flake8 . --count --max-line-length=120 --statistics

      - name: Run tests                # 步骤5：运行测试
        run: pytest tests/ -v
```

#### 自动部署到 VPS 示例

```yaml
# .github/workflows/deploy.yml
name: Deploy to VPS

on:
  push:
    branches: [main]                   # 只在 push 到 main 时部署

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Deploy via SSH
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.VPS_HOST }}           # 在 GitHub 仓库 Settings > Secrets 中配置
          username: ${{ secrets.VPS_USER }}
          key: ${{ secrets.VPS_SSH_KEY }}
          script: |
            cd /opt/myapp
            git pull origin main
            docker compose down
            docker compose up -d --build
            docker compose ps
```

**配置 GitHub Secrets**：
1. 进入 GitHub 仓库 → Settings → Secrets and variables → Actions
2. 添加以下 Secrets：
   - `VPS_HOST`：VPS IP 地址
   - `VPS_USER`：SSH 用户名
   - `VPS_SSH_KEY`：SSH 私钥内容

#### 其他常用触发方式

```yaml
on:
  # 定时触发（cron 语法）
  schedule:
    - cron: '0 2 * * *'               # 每天 UTC 02:00

  # 手动触发
  workflow_dispatch:
    inputs:
      environment:
        description: 'Deploy environment'
        required: true
        default: 'staging'

  # 标签触发（发布版本时）
  push:
    tags:
      - 'v*'                           # 匹配 v1.0、v2.0 等标签
```

---

### 练习 3：GitHub Actions

```bash
# 为你的某个 GitHub 项目添加 CI 流程

# 1. 在项目根目录创建 workflow
mkdir -p .github/workflows

# 2. 创建 CI 配置
cat > .github/workflows/ci.yml << 'EOF'
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'
      - name: Install dependencies
        run: pip install -r requirements.txt
      - name: Run linting
        run: |
          pip install flake8
          flake8 . --max-line-length=120
EOF

# 3. 提交并推送，然后在 GitHub 仓库的 Actions 标签页查看运行结果
git add .github/
git commit -m "ci: add GitHub Actions CI pipeline"
git push origin main
```

---

## 3. 监控与告警

> 这是第二阶段的重点项目。用 Prometheus + Grafana 构建监控体系，监控你的 4 台 VPS。

### 3.1 监控体系架构

```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│  北京 VPS    │    │  上海 VPS    │    │  广州 VPS    │    │ 雅加达 VPS   │
│ node_exporter│    │ node_exporter│    │ node_exporter│    │ node_exporter│
│  :9100       │    │  :9100       │    │  :9100       │    │  :9100       │
└──────┬───────┘    └──────┬───────┘    └──────┬───────┘    └──────┬───────┘
       │                   │                   │                   │
       └──────────┬────────┴──────┬────────────┘                   │
                  ▼               │                                │
           ┌──────────────┐       │                                │
           │  Prometheus   │◄─────┴────────────────────────────────┘
           │  (上海 VPS)   │       定期抓取指标（pull 模式）
           │  :9090        │
           └──────┬───────┘
                  │
           ┌──────▼───────┐
           │   Grafana     │
           │  (上海 VPS)   │
           │  :3000        │
           └──────┬───────┘
                  │
           ┌──────▼───────┐
           │ Alertmanager  │
           │  (上海 VPS)   │
           │  :9093        │
           └──────────────┘
                  │
           通知 → 邮件 / 钉钉 / 微信
```

**各组件职责**：

| 组件 | 作用 | 端口 |
|------|------|------|
| **node_exporter** | 采集服务器硬件/OS 指标（CPU/内存/磁盘/网络） | 9100 |
| **Prometheus** | 时序数据库 + 指标收集器 + 告警规则引擎 | 9090 |
| **Grafana** | 数据可视化面板（漂亮的 Dashboard） | 3000 |
| **Alertmanager** | 告警通知管理（去重、分组、路由、通知） | 9093 |

---

### 3.2 在被监控节点部署 node_exporter

> 在北京/广州/雅加达 VPS 上分别执行（上海也要装，监控自己）。

#### 方式一：二进制部署（推荐用于被监控节点，轻量）

```bash
# 下载（在每台被监控 VPS 上执行）
cd /tmp
curl -LO https://github.com/prometheus/node_exporter/releases/download/v1.7.0/node_exporter-1.7.0.linux-amd64.tar.gz
tar xzf node_exporter-1.7.0.linux-amd64.tar.gz
mv node_exporter-1.7.0.linux-amd64/node_exporter /usr/local/bin/

# 创建 systemd service
cat > /etc/systemd/system/node_exporter.service << 'EOF'
[Unit]
Description=Prometheus Node Exporter
After=network.target

[Service]
Type=simple
User=nobody
ExecStart=/usr/local/bin/node_exporter
Restart=on-failure
RestartSec=5s

[Install]
WantedBy=multi-user.target
EOF

# 启动
systemctl daemon-reload
systemctl enable --now node_exporter

# 验证
curl http://localhost:9100/metrics | head -20
ss -tlnp | grep 9100
```

#### 方式二：Docker 部署（如果节点已有 Docker）

```bash
docker run -d \
    --name node_exporter \
    --net host \
    --pid host \
    -v /:/host:ro,rslave \
    --restart unless-stopped \
    prom/node-exporter:v1.7.0 \
    --path.rootfs=/host
```

> **安全提醒**：node_exporter 暴露了大量系统信息，确保 9100 端口**不对公网开放**。通过云安全组/防火墙限制只允许 Prometheus 所在 IP 访问。

---

### 3.3 部署 Prometheus + Grafana + Alertmanager

> 在**上海 VPS**（主力机）上用 Docker Compose 部署整套监控栈。

```bash
mkdir -p ~/monitoring && cd ~/monitoring
mkdir -p prometheus alertmanager grafana
```

#### Prometheus 配置

```bash
cat > prometheus/prometheus.yml << 'EOF'
global:
  scrape_interval: 15s                 # 全局采集间隔
  evaluation_interval: 15s             # 告警规则评估间隔

# 告警规则文件
rule_files:
  - "alert_rules.yml"

# Alertmanager 配置
alerting:
  alertmanagers:
    - static_configs:
        - targets:
            - alertmanager:9093

# 采集目标配置
scrape_configs:
  # 监控 Prometheus 自身
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]

  # 监控上海 VPS（本机）
  - job_name: "shanghai"
    static_configs:
      - targets: ["node_exporter:9100"]      # Docker 网络中用容器名
        labels:
          region: "shanghai"
          cloud: "volcengine"

  # 监控北京 VPS
  - job_name: "beijing"
    static_configs:
      - targets: ["<北京VPS_IP>:9100"]
        labels:
          region: "beijing"
          cloud: "baidu"

  # 监控广州 VPS
  - job_name: "guangzhou"
    static_configs:
      - targets: ["<广州VPS_IP>:9100"]
        labels:
          region: "guangzhou"
          cloud: "huawei"

  # 监控雅加达 VPS
  - job_name: "jakarta"
    static_configs:
      - targets: ["<雅加达VPS_IP>:9100"]
        labels:
          region: "jakarta"
          cloud: "volcengine"
EOF
```

#### 告警规则

```bash
cat > prometheus/alert_rules.yml << 'EOF'
groups:
  - name: node_alerts
    rules:
      # 实例宕机
      - alert: InstanceDown
        expr: up == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "实例 {{ $labels.instance }} 宕机"
          description: "{{ $labels.job }} 的 {{ $labels.instance }} 已停止响应超过 1 分钟"

      # CPU 使用率过高
      - alert: HighCpuUsage
        expr: 100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 80
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "{{ $labels.instance }} CPU 使用率过高"
          description: "CPU 使用率超过 80%，当前值: {{ $value | printf \"%.1f\" }}%"

      # 内存使用率过高
      - alert: HighMemoryUsage
        expr: (1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100 > 85
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "{{ $labels.instance }} 内存使用率过高"
          description: "内存使用率超过 85%，当前值: {{ $value | printf \"%.1f\" }}%"

      # 磁盘使用率过高
      - alert: HighDiskUsage
        expr: (1 - node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"}) * 100 > 80
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "{{ $labels.instance }} 磁盘使用率过高"
          description: "根分区使用率超过 80%，当前值: {{ $value | printf \"%.1f\" }}%"

      # 磁盘即将满
      - alert: DiskAlmostFull
        expr: (1 - node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"}) * 100 > 90
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "{{ $labels.instance }} 磁盘即将满"
          description: "根分区使用率超过 90%，当前值: {{ $value | printf \"%.1f\" }}%"
EOF
```

#### Alertmanager 配置

```bash
cat > alertmanager/alertmanager.yml << 'EOF'
global:
  resolve_timeout: 5m

route:
  group_by: ['alertname', 'instance']
  group_wait: 30s                      # 等待 30 秒收集同组告警
  group_interval: 5m                   # 同组告警发送间隔
  repeat_interval: 4h                  # 重复告警间隔
  receiver: 'default'

  routes:
    - match:
        severity: critical
      receiver: 'critical'

receivers:
  - name: 'default'
    # 配置邮件通知（按需配置）
    # email_configs:
    #   - to: 'your-email@example.com'
    #     from: 'alertmanager@example.com'
    #     smarthost: 'smtp.example.com:587'
    #     auth_username: 'alertmanager@example.com'
    #     auth_password: 'your-password'

  - name: 'critical'
    # 严重告警的通知渠道
    # webhook_configs:
    #   - url: 'https://your-webhook-url'   # 钉钉/微信机器人 webhook
EOF
```

#### Docker Compose 编排

```bash
cat > docker-compose.yml << 'EOF'
version: "3.8"

services:
  node_exporter:
    image: prom/node-exporter:v1.7.0
    container_name: node_exporter
    pid: host
    volumes:
      - /:/host:ro,rslave
    command:
      - '--path.rootfs=/host'
    ports:
      - "127.0.0.1:9100:9100"         # 只监听本地，不对外暴露
    restart: unless-stopped

  prometheus:
    image: prom/prometheus:v2.48.0
    container_name: prometheus
    volumes:
      - ./prometheus:/etc/prometheus
      - prometheus-data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--storage.tsdb.retention.time=30d'    # 数据保留 30 天
      - '--web.enable-lifecycle'               # 支持热加载配置
    ports:
      - "127.0.0.1:9090:9090"
    restart: unless-stopped

  grafana:
    image: grafana/grafana:10.2.0
    container_name: grafana
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=your_secure_password   # 修改为强密码！
      - GF_USERS_ALLOW_SIGN_UP=false
    volumes:
      - grafana-data:/var/lib/grafana
    ports:
      - "3000:3000"                    # Grafana 需要对外访问
    depends_on:
      - prometheus
    restart: unless-stopped

  alertmanager:
    image: prom/alertmanager:v0.26.0
    container_name: alertmanager
    volumes:
      - ./alertmanager:/etc/alertmanager
    command:
      - '--config.file=/etc/alertmanager/alertmanager.yml'
    ports:
      - "127.0.0.1:9093:9093"
    restart: unless-stopped

volumes:
  prometheus-data:
  grafana-data:
EOF
```

#### 启动并验证

```bash
# 启动
docker compose up -d

# 检查状态
docker compose ps

# 验证各组件
curl http://localhost:9090/-/healthy        # Prometheus
curl http://localhost:9100/metrics | head   # node_exporter
curl http://localhost:9093/-/healthy        # Alertmanager
# Grafana 在浏览器访问：http://<上海VPS_IP>:3000

# 检查 Prometheus 目标状态
curl http://localhost:9090/api/v1/targets | python3 -m json.tool
# 所有 target 的 state 应该是 "up"

# 热加载 Prometheus 配置（修改配置后无需重启）
curl -X POST http://localhost:9090/-/reload
```

---

### 3.4 配置 Grafana Dashboard

#### 初始配置步骤

1. 浏览器访问 `http://<上海VPS_IP>:3000`
2. 用 admin / your_secure_password 登录
3. 添加数据源：
   - 左侧菜单 → Connections → Data sources → Add data source
   - 选择 Prometheus
   - URL 填 `http://prometheus:9090`（Docker 网络内用容器名）
   - 点击 Save & test

#### 导入现成 Dashboard

Grafana 社区有大量现成的 Dashboard 模板：

1. 左侧菜单 → Dashboards → Import
2. 输入 Dashboard ID，常用：
   - **1860** — Node Exporter Full（最全面的服务器监控面板）
   - **13978** — Node Exporter（简洁版）
3. 选择 Prometheus 数据源
4. 点击 Import

#### 常用 PromQL 查询

```promql
# CPU 使用率
100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# 内存使用率
(1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100

# 磁盘使用率
(1 - node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"}) * 100

# 网络流量（接收，每秒字节数）
rate(node_network_receive_bytes_total{device="eth0"}[5m])

# 网络流量（发送）
rate(node_network_transmit_bytes_total{device="eth0"}[5m])

# 系统负载（1 分钟平均）
node_load1

# 开机时间
time() - node_boot_time_seconds

# TCP 连接数
node_netstat_Tcp_CurrEstab

# 磁盘 IO（读字节/秒）
rate(node_disk_read_bytes_total[5m])
```

---

### 练习 4：监控部署

```bash
# 完整部署流程：

# 1. 在上海 VPS 部署 Prometheus + Grafana + Alertmanager（Docker Compose）
# 2. 在其他 3 台 VPS 分别安装 node_exporter（二进制方式）
# 3. 配置安全组/防火墙：允许上海 VPS IP 访问其他机器的 9100 端口
# 4. 在 Prometheus 配置中添加所有 target
# 5. 导入 Grafana Dashboard 1860
# 6. 自定义一个 Dashboard 面板：
#    - 4 台 VPS 的 CPU 使用率对比（单图多线）
#    - 内存使用率（仪表盘样式）
#    - 磁盘使用率（百分比条形图）
#    - 跨地域网络延迟（如果配置了 blackbox_exporter）

# 验证告警规则：
# 在某台 VPS 上故意制造高负载
stress --cpu 2 --timeout 120 &
# 观察 Prometheus Alerts 页面 → Alertmanager 是否触发
```

---

## 4. Nginx 深入

### 4.1 Nginx 架构与进程模型

```
Nginx 进程模型：
  master process（主进程）
    ├── worker process 1    ← 处理实际请求
    ├── worker process 2
    ├── worker process 3
    └── worker process N    ← worker_processes 配置控制数量（建议 = CPU 核心数）
```

```bash
# 查看 Nginx 进程
ps aux | grep nginx
# root     12345  ...  nginx: master process /usr/sbin/nginx
# nginx    12346  ...  nginx: worker process
# nginx    12347  ...  nginx: worker process

# 查看编译参数和版本
nginx -V

# 配置文件检查（重要！修改配置后必须先检查再 reload）
nginx -t

# 重载配置（不中断服务）
systemctl reload nginx
# 等效于：nginx -s reload
```

---

### 4.2 核心配置结构

```nginx
# /etc/nginx/nginx.conf — 主配置文件

# ===== 全局块 =====
user nginx;                            # 运行用户
worker_processes auto;                 # worker 进程数（auto = CPU 核心数）
error_log /var/log/nginx/error.log warn;
pid /run/nginx.pid;

# ===== events 块 =====
events {
    worker_connections 1024;           # 每个 worker 最大连接数
    use epoll;                         # Linux 高性能事件模型
}

# ===== http 块 =====
http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    # 日志格式
    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" "$http_x_forwarded_for"';
    access_log /var/log/nginx/access.log main;

    # 性能优化
    sendfile on;                       # 高效文件传输
    tcp_nopush on;                     # 减少网络报文段数量
    tcp_nodelay on;                    # 禁用 Nagle 算法
    keepalive_timeout 65;              # 长连接超时
    gzip on;                           # 开启 gzip 压缩
    gzip_types text/plain text/css application/json application/javascript;

    # 加载站点配置
    include /etc/nginx/conf.d/*.conf;
}
```

---

### 4.3 反向代理

> 反向代理是 Nginx 最常用的功能：客户端 → Nginx → 后端应用。

```nginx
# /etc/nginx/conf.d/app-proxy.conf

server {
    listen 80;
    server_name app.example.com;

    location / {
        proxy_pass http://127.0.0.1:5000;      # 代理到后端应用
        proxy_set_header Host $host;             # 传递原始 Host
        proxy_set_header X-Real-IP $remote_addr; # 传递客户端真实 IP
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # 超时设置
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
    }
}
```

**为什么要传递这些 Header**：
- `Host`：后端应用需要知道请求的原始域名
- `X-Real-IP`：后端应用需要知道客户端真实 IP（否则看到的是 Nginx 的 127.0.0.1）
- `X-Forwarded-For`：完整的代理链 IP 列表
- `X-Forwarded-Proto`：原始协议（http/https）

---

### 4.4 负载均衡

```nginx
# /etc/nginx/conf.d/loadbalancer.conf

# 定义后端服务器组
upstream backend {
    # 负载均衡策略（默认轮询）
    # least_conn;                      # 最少连接
    # ip_hash;                         # 按客户端 IP 哈希（会话保持）
    # hash $request_uri consistent;    # 按 URL 哈希

    server 192.168.1.10:8080 weight=3;  # 权重（weight 越大，分配越多）
    server 192.168.1.11:8080 weight=2;
    server 192.168.1.12:8080 backup;    # 备用服务器（前面都挂了才启用）
    # server 192.168.1.13:8080 down;   # 标记为下线
}

server {
    listen 80;
    server_name www.example.com;

    location / {
        proxy_pass http://backend;              # 代理到 upstream 组
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

**负载均衡策略对比**：

| 策略 | 说明 | 适用场景 |
|------|------|---------|
| 轮询（默认） | 按顺序依次分配 | 后端性能一致 |
| `weight` | 按权重分配 | 后端性能不一致 |
| `least_conn` | 分配给活跃连接最少的 | 请求处理时间差异大 |
| `ip_hash` | 同一 IP 固定到同一后端 | 需要会话保持 |
| `hash` | 按指定变量哈希 | 缓存场景 |

> **面试考点**：四层负载 vs 七层负载
> - **四层**（传输层）：基于 IP+端口转发（如 LVS、Nginx stream），不解析 HTTP，性能高
> - **七层**（应用层）：基于 HTTP 内容转发（如 Nginx http、HAProxy），可按 URL/Header 路由

---

### 4.5 HTTPS / SSL 配置

#### 使用 acme.sh 申请 Let's Encrypt 证书

```bash
# 安装 acme.sh
curl https://get.acme.sh | sh
source ~/.bashrc

# 申请证书（HTTP 验证方式，需要 80 端口可访问）
acme.sh --issue -d example.com -d www.example.com --webroot /var/www/html

# 安装证书到指定目录
acme.sh --install-cert -d example.com \
    --key-file /etc/nginx/ssl/example.com.key \
    --fullchain-file /etc/nginx/ssl/example.com.crt \
    --reloadcmd "systemctl reload nginx"

# acme.sh 会自动配置 crontab 定时续期
crontab -l | grep acme
```

#### Nginx HTTPS 配置

```nginx
# /etc/nginx/conf.d/https.conf

# HTTP → HTTPS 重定向
server {
    listen 80;
    server_name example.com www.example.com;
    return 301 https://$host$request_uri;
}

# HTTPS 配置
server {
    listen 443 ssl http2;
    server_name example.com www.example.com;

    # SSL 证书
    ssl_certificate /etc/nginx/ssl/example.com.crt;
    ssl_certificate_key /etc/nginx/ssl/example.com.key;

    # SSL 安全配置
    ssl_protocols TLSv1.2 TLSv1.3;                     # 只允许安全协议
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256;
    ssl_prefer_server_ciphers on;

    # SSL 性能优化
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 1d;
    ssl_session_tickets off;

    # HSTS（严格传输安全，强制浏览器使用 HTTPS）
    add_header Strict-Transport-Security "max-age=63072000" always;

    root /var/www/html;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

---

### 4.6 常见 Nginx 配置场景

#### 静态资源缓存

```nginx
location ~* \.(jpg|jpeg|png|gif|ico|css|js|woff2)$ {
    expires 30d;                       # 浏览器缓存 30 天
    add_header Cache-Control "public, no-transform";
    access_log off;                    # 不记录静态资源日志
}
```

#### 自定义错误页面

```nginx
error_page 404 /404.html;
error_page 500 502 503 504 /50x.html;

location = /404.html {
    root /var/www/error;
    internal;                          # 只能内部重定向访问
}
```

#### 限制请求速率（防 DDoS/CC）

```nginx
# 在 http 块中定义限速区
http {
    limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;
    #              |                        |          |
    #         按客户端IP限制           共享内存区名    每秒10个请求

    server {
        location /api/ {
            limit_req zone=api burst=20 nodelay;
            #                   |         |
            #              允许突发20个    不延迟处理突发请求
            proxy_pass http://backend;
        }
    }
}
```

#### 禁止访问敏感文件

```nginx
# 禁止访问隐藏文件（.git/.env 等）
location ~ /\. {
    deny all;
    access_log off;
    log_not_found off;
}

# 禁止访问 .sql/.bak 等备份文件
location ~* \.(sql|bak|log|old|orig|swp)$ {
    deny all;
}
```

#### 自定义日志格式

```nginx
# JSON 格式日志（便于 ELK/Loki 等日志系统解析）
log_format json_log escape=json
    '{'
        '"time":"$time_iso8601",'
        '"remote_addr":"$remote_addr",'
        '"request":"$request",'
        '"status":$status,'
        '"body_bytes_sent":$body_bytes_sent,'
        '"request_time":$request_time,'
        '"upstream_response_time":"$upstream_response_time",'
        '"http_user_agent":"$http_user_agent"'
    '}';

access_log /var/log/nginx/access.json json_log;
```

---

### 练习 5：Nginx 实战

```bash
# 在广州 VPS 上练习：

# 任务 1：配置反向代理
# 在广州 VPS 上运行一个 Python HTTP 服务，用 Nginx 做反向代理
python3 -m http.server 8080 &
# 配置 Nginx 将 80 端口的请求代理到 8080

# 任务 2：配置负载均衡（模拟多后端）
# 启动 3 个不同端口的后端
python3 -m http.server 8081 &
python3 -m http.server 8082 &
python3 -m http.server 8083 &
# 配置 Nginx upstream 实现负载均衡

# 任务 3：HTTPS 配置
# 在雅加达 VPS 上（绑定未备案域名）：
# 用 acme.sh 申请证书，配置 HTTPS
# 配置 HTTP 自动跳转 HTTPS
# 用 curl 和浏览器验证

# 任务 4：安全加固
# 配置以下安全项：
# - 禁止访问 .git / .env
# - 限制请求速率
# - 添加安全 Header
# - 自定义 404 错误页面
```

---

## 5. 综合练习项目

### 项目一：Docker 化部署完整 Web 应用栈

> 在上海 VPS 上用 Docker Compose 一键部署。

```bash
mkdir -p ~/project-webapp && cd ~/project-webapp

cat > docker-compose.yml << 'EOF'
version: "3.8"

services:
  nginx:
    image: nginx:1.24
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/conf.d:/etc/nginx/conf.d:ro
      - ./nginx/ssl:/etc/nginx/ssl:ro
    depends_on:
      - app
    restart: unless-stopped

  app:
    build: ./app
    environment:
      - DATABASE_URL=mysql://root:secret@db:3306/webapp
      - REDIS_URL=redis://redis:6379
    depends_on:
      - db
      - redis
    restart: unless-stopped

  db:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: secret
      MYSQL_DATABASE: webapp
    volumes:
      - mysql-data:/var/lib/mysql
    restart: unless-stopped

  redis:
    image: redis:7-alpine
    volumes:
      - redis-data:/data
    restart: unless-stopped

  # 监控
  node_exporter:
    image: prom/node-exporter:v1.7.0
    pid: host
    volumes:
      - /:/host:ro,rslave
    command: ['--path.rootfs=/host']
    ports:
      - "127.0.0.1:9100:9100"
    restart: unless-stopped

volumes:
  mysql-data:
  redis-data:
EOF
```

### 项目二：CI/CD 自动部署流水线

> 实现 push 到 GitHub → 自动测试 → 自动部署到 VPS。

```yaml
# .github/workflows/deploy.yml
name: CI/CD Pipeline

on:
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'
      - name: Install & test
        run: |
          pip install -r requirements.txt
          pip install pytest flake8
          flake8 .
          pytest tests/ -v

  deploy:
    needs: test                        # 测试通过后才部署
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'

    steps:
      - uses: actions/checkout@v4

      - name: Deploy to VPS
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.VPS_HOST }}
          username: ${{ secrets.VPS_USER }}
          key: ${{ secrets.VPS_SSH_KEY }}
          script: |
            cd /opt/myapp
            git pull origin main
            docker compose down
            docker compose up -d --build
            echo "Deploy completed at $(date)"
```

---

## 知识检测：面试模拟问答

> 合上教程，尝试回答这些问题：

**Docker**
- [ ] 容器和虚拟机有什么区别？各自适用什么场景？
- [ ] Dockerfile 中 CMD 和 ENTRYPOINT 的区别？
- [ ] COPY 和 ADD 的区别？优先用哪个？
- [ ] 如何减小 Docker 镜像体积？（至少说 4 种方法）
- [ ] Docker 网络模式有哪些？默认 bridge 和自定义 bridge 有什么区别？
- [ ] `docker run -p 8080:80` 是什么意思？
- [ ] Docker Compose 的 `depends_on` 能保证服务真正就绪吗？（不能，只保证启动顺序）

**CI/CD**
- [ ] 什么是 CI？什么是 CD？有什么区别？
- [ ] Git merge 和 rebase 的区别？什么时候用哪个？
- [ ] GitHub Actions 的工作流是怎么触发的？

**监控**
- [ ] Prometheus 的数据采集模式是 push 还是 pull？（pull）
- [ ] PromQL 中 `rate()` 和 `increase()` 的区别？
- [ ] 告警规则中的 `for` 字段是什么意思？（持续时间，避免瞬时抖动触发告警）
- [ ] Grafana 的数据源可以是哪些？（Prometheus、InfluxDB、MySQL、Elasticsearch 等）

**Nginx**
- [ ] Nginx 的 master-worker 进程模型是怎样的？worker_processes 建议设为多少？
- [ ] 反向代理和正向代理的区别？
- [ ] 四层负载均衡和七层负载均衡的区别？
- [ ] Nginx 中 `location` 的匹配优先级是怎样的？
- [ ] 如何配置 Nginx HTTPS？证书自动续期怎么做？
- [ ] `proxy_set_header X-Real-IP` 有什么用？

---

## 学习进度检查清单

### 第 1 周（Docker 进阶）
- [ ] 掌握 Dockerfile 编写（多阶段构建、镜像优化）
- [ ] 掌握 Docker 网络模式和数据卷
- [ ] 能用 Docker Compose 编排多容器应用
- [ ] 完成 WordPress + MySQL + Nginx 部署
- [ ] 完成练习 1 + 练习 2

### 第 2 周（CI/CD + Git）
- [ ] 掌握 Git 分支策略（feature branch、merge、rebase）
- [ ] 能编写 GitHub Actions 工作流
- [ ] 为项目配置 CI Pipeline（lint + test）
- [ ] 配置自动部署到 VPS
- [ ] 完成练习 3

### 第 3 周（监控与告警）
- [ ] 在 4 台 VPS 上部署 node_exporter
- [ ] 用 Docker Compose 部署 Prometheus + Grafana + Alertmanager
- [ ] 导入并自定义 Grafana Dashboard
- [ ] 配置告警规则并测试触发
- [ ] 完成练习 4

### 第 4 周（Nginx 深入）
- [ ] 掌握反向代理配置
- [ ] 掌握负载均衡配置和策略
- [ ] 配置 HTTPS + 证书自动续期
- [ ] 完成安全加固配置
- [ ] 完成练习 5 + 综合项目

---

> **下一步**：完成本阶段后，进入 [第三阶段：项目实战 + 简历](./sre_internship_roadmap.md#️-第三阶段项目实战--简历5月中--6月中)
> 重点：打造 2-3 个可写进简历的完整项目
