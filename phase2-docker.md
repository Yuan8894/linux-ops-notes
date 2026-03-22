# 第二阶段 · Docker 深度教程

> **适用环境**：Rocky Linux 9.5（上海/雅加达 VPS）
> **学习周期**：第二阶段第 1-2 周（共约 10 天）
> **前置条件**：已完成第一阶段 Linux 核心，有 Docker 基础使用经验

---

## 目录

1. [底层原理：Docker 为什么能跑起来](#1-底层原理docker-为什么能跑起来)
2. [安装与全局配置](#2-安装与全局配置)
3. [镜像：从拉取到构建](#3-镜像从拉取到构建)
4. [容器生命周期与管理](#4-容器生命周期与管理)
5. [Dockerfile 深度解析](#5-dockerfile-深度解析)
6. [Docker 网络](#6-docker-网络)
7. [数据持久化](#7-数据持久化)
8. [Docker Compose](#8-docker-compose)
9. [资源限制与性能调优](#9-资源限制与性能调优)
10. [日志管理](#10-日志管理)
11. [安全最佳实践](#11-安全最佳实践)
12. [常见故障排查](#12-常见故障排查)
13. [综合实战项目](#13-综合实战项目)
14. [面试题精选](#14-面试题精选)

---

## 1. 底层原理：Docker 为什么能跑起来

> 理解原理，才能理解为什么这样配置，才能排查奇怪的问题。

### 1.1 三大技术支柱

Docker 本质上是 Linux 内核三项技术的组合封装：

```
Namespace   → 隔离：让容器「看不到」宿主机和其他容器的资源
Cgroup      → 限制：控制容器能用多少 CPU、内存、磁盘 IO
Union FS    → 分层：让镜像可以共享层、增量存储
```

#### Namespace（命名空间）—— 实现隔离

Linux 有 6 种 Namespace，Docker 全部使用：

| Namespace | 隔离内容 | 容器的效果 |
|-----------|---------|-----------|
| `pid` | 进程 ID | 容器内看不到宿主机进程，PID 从 1 开始 |
| `net` | 网络栈 | 容器有独立网卡、IP、路由表 |
| `mnt` | 文件系统挂载点 | 容器有独立的文件系统视图 |
| `uts` | 主机名和域名 | 容器可设置独立 hostname |
| `ipc` | 进程间通信 | 独立的信号量、消息队列、共享内存 |
| `user` | 用户 ID | 容器内的 root ≠ 宿主机的 root（user ns） |

```bash
# 验证：查看容器的进程在宿主机上的样子
docker run -d --name test nginx:alpine

# 容器内认为自己是 PID 1 的 nginx
docker exec test ps aux
# PID   USER     COMMAND
#   1   root     nginx: master process nginx -g daemon off;
#  31   nginx    nginx: worker process

# 宿主机上看到的是真实 PID（完全不同）
ps aux | grep nginx
# hw_yyy  18423  ...  nginx: master process nginx -g daemon off;
# hw_yyy  18456  ...  nginx: worker process
```

#### Cgroup（Control Groups）—— 实现限制

Cgroup 负责限制和统计容器的资源使用：

```bash
# 运行一个内存限制为 256MB 的容器
docker run -d --name mem-test --memory 256m nginx:alpine

# 在 cgroup 文件系统中看到这个限制
cat /sys/fs/cgroup/memory/docker/<container_id>/memory.limit_in_bytes
# 268435456  ← 256 * 1024 * 1024 = 256MB

# 查看容器实时资源使用
docker stats mem-test --no-stream
# CONTAINER   CPU %   MEM USAGE / LIMIT   MEM %   NET I/O   BLOCK I/O
# mem-test    0.00%   7.2MiB / 256MiB     2.81%   ...       ...
```

#### Union FS（联合文件系统）—— 实现分层

镜像是只读的分层结构，容器在最上层加一个可写层：

```
容器可写层（Container Layer）  ← 容器内的所有修改都在这里
─────────────────────────────
镜像层 4：nginx 配置文件
镜像层 3：nginx 二进制
镜像层 2：依赖库
镜像层 1：base OS（Alpine Linux）
─────────────────────────────
所有层共享，节省磁盘空间
```

```bash
# 查看镜像的分层结构
docker history nginx:alpine
# IMAGE          CREATED       CREATED BY                   SIZE
# 3f8a4339aadd   2 weeks ago   /bin/sh -c #(nop) CMD [...]  0B
# <missing>      2 weeks ago   /bin/sh -c #(nop) EXPOSE 80  0B
# <missing>      2 weeks ago   /bin/sh -c nginx 配置...     1.62kB
# <missing>      2 weeks ago   /bin/sh -c apk add nginx...  15.2MB
# <missing>      4 weeks ago   /bin/sh -c #(nop) FROM alp   0B

# 查看容器使用的存储驱动
docker info | grep "Storage Driver"
# Storage Driver: overlay2   ← Rocky Linux 默认，性能好
```

**写时复制（Copy-on-Write）**：

容器修改只读层中的文件时，不会改变原镜像层，而是：
1. 将文件从镜像层复制到容器可写层
2. 在可写层中修改这份副本
3. 原镜像层的文件不变

这就是为什么：多个容器共享同一镜像，互不影响；容器删除后，所有修改也消失。

---

### 1.2 Docker 架构全景

```
┌─────────────────────────────────────────────┐
│                 docker CLI                   │
│         docker build / run / ps ...         │
└─────────────────┬───────────────────────────┘
                  │ REST API（Unix Socket）
                  ▼
┌─────────────────────────────────────────────┐
│              Docker Daemon (dockerd)         │
│  - 管理镜像、容器、网络、数据卷              │
│  - 接收 CLI 命令并执行                       │
└─────────────────┬───────────────────────────┘
                  │ gRPC
                  ▼
┌─────────────────────────────────────────────┐
│            containerd                        │
│  - 容器生命周期管理（创建/启动/停止/删除）    │
│  - 镜像拉取和存储                             │
└─────────────────┬───────────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────────┐
│         runc / crun                          │
│  - 真正调用 Linux 内核接口                   │
│  - 创建 Namespace + Cgroup + 启动进程        │
└─────────────────────────────────────────────┘
```

---

## 2. 安装与全局配置

### 2.1 安装 Docker（Rocky Linux 9）

```bash
# 移除旧版本
dnf remove docker docker-client docker-common docker-engine -y 2>/dev/null

# 添加 Docker 官方仓库
dnf config-manager --add-repo https://download.docker.com/linux/rhel/docker-ce.repo

# 安装 Docker Engine + Compose 插件
dnf install docker-ce docker-ce-cli containerd.io docker-compose-plugin -y

# 启动并设置开机自启
systemctl enable --now docker

# 验证
docker version
docker compose version
```

### 2.2 daemon.json 全局配置

`/etc/docker/daemon.json` 是 Docker Daemon 的全局配置文件，所有容器都受影响：

```json
{
    "data-root": "/var/lib/docker",
    "log-driver": "json-file",
    "log-opts": {
        "max-size": "10m",
        "max-file": "3"
    },
    "registry-mirrors": [
        "https://mirror.ccs.tencentyun.com",
        "https://hub-mirror.c.163.com"
    ],
    "default-address-pools": [
        {"base": "172.20.0.0/16", "size": 24}
    ],
    "live-restore": true,
    "max-concurrent-downloads": 5,
    "storage-driver": "overlay2"
}
```

| 配置项 | 说明 | 建议值 |
|--------|------|-------|
| `log-driver` | 日志驱动类型 | `json-file`（默认） |
| `log-opts.max-size` | 单个日志文件最大大小 | `10m`（防止日志把磁盘撑满） |
| `log-opts.max-file` | 保留日志文件数量 | `3` |
| `registry-mirrors` | 镜像加速地址 | 国内服务器必配 |
| `live-restore` | Docker Daemon 重启不影响运行中的容器 | `true` |
| `storage-driver` | 存储驱动 | `overlay2`（Rocky Linux 9 默认，无需改） |

```bash
# 修改后重启 Daemon（不影响运行中的容器，因为 live-restore: true）
systemctl reload docker
# 或
systemctl restart docker

# 验证配置生效
docker info | grep -E "Storage Driver|Logging Driver|Registry Mirrors" -A 2
```

### 2.3 非 root 用户使用 Docker

```bash
# 将当前用户加入 docker 组（避免每次都要 sudo）
usermod -aG docker $USER

# 重新登录后生效（或运行以下命令立即生效）
newgrp docker

# 验证
docker ps   # 不再需要 sudo
```

> **安全提醒**：`docker` 组成员等同于拥有 root 权限（可以挂载宿主机目录、启动特权容器）。
> 生产环境中要严格控制谁能使用 Docker。

---

## 3. 镜像：从拉取到构建

### 3.1 镜像命名规范

```
registry/namespace/repository:tag

示例：
  nginx:1.24                          → Docker Hub 官方镜像，tag 1.24
  nginx:latest                        → Docker Hub 官方镜像，最新版
  library/nginx:1.24                  → 同上（library/ 是官方命名空间的完整写法）
  myuser/myapp:v2.0                   → Docker Hub 个人镜像
  registry.cn-hangzhou.aliyuncs.com/myns/myapp:v1  → 阿里云 ACR 私有镜像
  ghcr.io/owner/repo:sha-abc123       → GitHub Container Registry
```

**Tag 选择建议**：

| 场景 | 推荐 Tag | 原因 |
|------|---------|------|
| 生产环境 | `nginx:1.24.0` | 固定版本，不会因镜像更新而行为改变 |
| 开发/测试 | `nginx:1.24` | 小版本跟随更新，获取 bug fix |
| 永远不要 | `nginx:latest` | 不可预期的变化，CI 构建不可重现 |
| 基础镜像 | `python:3.11-slim` | slim 去掉了不必要的包，比完整版小很多 |

### 3.2 镜像操作命令全集

```bash
# ===== 拉取 =====
docker pull nginx:1.24                  # 拉取指定 tag
docker pull nginx@sha256:abc123...      # 按摘要拉取（最稳定，不可篡改）

# ===== 查看 =====
docker images                           # 列出所有本地镜像
docker images nginx                     # 只列出 nginx 相关镜像
docker images --format "table {{.Repository}}\t{{.Tag}}\t{{.Size}}\t{{.CreatedSince}}"
docker image ls -f dangling=true        # 列出悬空镜像（tag 为 <none> 的）

# ===== 检查 =====
docker inspect nginx:1.24               # 完整元数据（JSON 格式）
docker inspect --format '{{.Config.Env}}' nginx:1.24   # 只看环境变量
docker inspect --format '{{.Config.ExposedPorts}}' nginx:1.24  # 只看暴露端口
docker history nginx:1.24               # 镜像构建层历史

# ===== 标记 =====
docker tag nginx:1.24 my-nginx:prod     # 为镜像打新标签（不复制，共享 ID）
docker tag nginx:1.24 registry.cn-hangzhou.aliyuncs.com/ns/nginx:1.24

# ===== 推送 =====
docker login registry.cn-hangzhou.aliyuncs.com  # 登录私有仓库
docker push registry.cn-hangzhou.aliyuncs.com/ns/nginx:1.24

# ===== 删除 =====
docker rmi nginx:1.24                   # 删除指定镜像
docker rmi $(docker images -q)          # 删除所有本地镜像（危险！）
docker image prune                      # 删除悬空镜像（<none>:<none>）
docker image prune -a                   # 删除所有未被容器使用的镜像

# ===== 导出/导入（离线传输）=====
docker save nginx:1.24 | gzip > nginx-1.24.tar.gz    # 导出并压缩
docker load < nginx-1.24.tar.gz         # 从压缩文件导入

# 在服务器之间传输镜像（无法访问 Docker Hub 时）
docker save myapp:v1 | ssh user@remote-server docker load
```

### 3.3 理解镜像摘要（Digest）

```bash
# 查看镜像的 SHA256 摘要
docker images --digests
# REPOSITORY   TAG     DIGEST                                           IMAGE ID
# nginx        1.24    sha256:3c4c1f42a89e343c7b050517...               abc123def456

# 按摘要拉取（CI/CD 中推荐，100% 确保镜像一致）
docker pull nginx@sha256:3c4c1f42a89e343c7b050517...
```

---

## 4. 容器生命周期与管理

### 4.1 容器状态机

```
            docker create
                 ↓
            [Created]
                 ↓ docker start
            [Running] ←──────────────────┐
           ↓         ↓                   │
    docker pause  docker stop / kill     │
           ↓         ↓                   │ docker restart
        [Paused]  [Exited] ──────────────┘
           ↓
    docker unpause
```

| 状态 | 说明 |
|------|------|
| `Created` | 容器已创建，但未启动 |
| `Running` | 正在运行 |
| `Paused` | 进程被暂停（SIGSTOP），不消耗 CPU |
| `Exited` | 已退出，保留文件系统 |
| `Dead` | 删除失败，处于异常状态 |
| `Restarting` | 正在重启 |

### 4.2 docker run 参数详解

```bash
docker run \
    -d \                               # 后台运行（detached）
    -it \                              # 交互式终端（-i 保持stdin，-t 分配伪终端）
    --name web \                       # 容器名（不指定则随机生成）
    -p 8080:80 \                       # 端口映射：宿主机:容器
    -p 127.0.0.1:8080:80 \            # 只绑定本地（不对外暴露）
    -P \                               # 随机映射所有 EXPOSE 端口
    -v /data:/app/data \              # 绑定挂载
    -v myvolume:/app/data \           # 命名卷挂载
    --mount type=tmpfs,dst=/tmp \     # tmpfs（内存文件系统）
    -e APP_ENV=production \            # 环境变量
    --env-file .env \                  # 从文件批量读取环境变量
    --network mynet \                  # 指定网络
    --network-alias myapp \           # 网络别名（其他容器可用此名访问）
    --restart unless-stopped \        # 重启策略
    --memory 512m \                    # 内存硬限制
    --memory-swap 1g \                 # 内存 + swap 总限制
    --cpus 1.5 \                       # CPU 核心数限制（可以是小数）
    --cpu-shares 512 \                 # CPU 相对权重（默认 1024）
    --user 1000:1000 \                 # 以指定用户:组运行
    --workdir /app \                   # 工作目录
    --hostname mycontainer \          # 容器内的 hostname
    --add-host db:192.168.1.10 \      # 在 /etc/hosts 中添加记录
    --tmpfs /tmp \                     # 挂载临时内存文件系统
    --read-only \                      # 只读文件系统（安全加固）
    --security-opt no-new-privileges \ # 禁止容器内进程提权
    --cap-drop ALL \                   # 移除所有 Linux capabilities
    --cap-add NET_BIND_SERVICE \      # 按需添加单个 capability
    --pid host \                       # 共享宿主机 PID 命名空间（node_exporter 需要）
    --rm \                             # 容器退出后自动删除（适合一次性任务）
    nginx:1.24 \                       # 镜像
    nginx -g "daemon off;"             # 覆盖默认 CMD（可选）
```

### 4.3 容器管理命令全集

```bash
# ===== 查看 =====
docker ps                              # 运行中的容器
docker ps -a                           # 所有容器（含已停止）
docker ps -q                           # 只输出容器 ID（便于批量操作）
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
docker inspect <container>             # 完整元数据
docker inspect --format '{{.State.Status}}' <container>   # 只看状态
docker inspect --format '{{.NetworkSettings.IPAddress}}' <container>  # 看 IP

# ===== 生命周期 =====
docker stop <container>                # 发 SIGTERM，等待 10s 后 SIGKILL
docker stop -t 30 <container>          # 等待 30s 再强杀（默认 10s）
docker kill <container>                # 直接发 SIGKILL
docker kill -s HUP <container>         # 发 SIGHUP（nginx reload 用这个）
docker start <container>
docker restart <container>
docker pause <container>               # 暂停（SIGSTOP）
docker unpause <container>

# ===== 清理 =====
docker rm <container>                  # 删除已停止的容器
docker rm -f <container>               # 强制删除（运行中的也删）
docker rm $(docker ps -aq)             # 删除所有已停止的容器
docker container prune                 # 删除所有已停止的容器（交互确认）
docker container prune -f              # 不需要确认

# ===== 交互 =====
docker exec -it <container> bash       # 进入容器 bash
docker exec -it <container> sh         # 没有 bash 时用 sh（alpine 常见）
docker exec <container> <cmd>          # 在容器内执行命令，不进入
docker exec -it -e DEBUG=true <container> bash  # 带环境变量进入

# ===== 文件操作 =====
docker cp <container>:/etc/nginx/nginx.conf ./   # 容器 → 宿主机
docker cp ./index.html <container>:/usr/share/nginx/html/  # 宿主机 → 容器

# ===== 监控 =====
docker stats                           # 实时资源监控（CPU/内存/网络/IO）
docker stats --no-stream               # 只刷新一次
docker stats --format "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}"
docker top <container>                 # 容器内进程列表（在宿主机上看）

# ===== 日志 =====
docker logs <container>
docker logs -f <container>             # 实时追踪
docker logs --tail 100 <container>     # 最后 100 行
docker logs --since 2h <container>     # 最近 2 小时
docker logs --since "2026-04-01T10:00:00" <container>
```

### 4.4 重启策略详解

```bash
# 创建容器时指定
docker run --restart <policy> ...
```

| 策略 | 行为 |
|------|------|
| `no`（默认） | 不自动重启 |
| `on-failure` | 退出码非 0 时重启，可指定最大次数 `on-failure:5` |
| `unless-stopped` | 除非手动 stop，否则总是重启（推荐生产使用） |
| `always` | 总是重启，Docker Daemon 重启后也会恢复 |

```bash
# unless-stopped vs always 的区别：
# always：手动 docker stop 后，重启 Docker Daemon，容器会自动启动
# unless-stopped：手动 docker stop 后，重启 Docker Daemon，容器不会自动启动

# 修改已有容器的重启策略（不需要重新创建）
docker update --restart unless-stopped web
```

---

## 5. Dockerfile 深度解析

### 5.1 所有指令详解

#### FROM — 基础镜像

```dockerfile
FROM ubuntu:22.04                      # 从 ubuntu 22.04 开始构建
FROM scratch                           # 空白起点，用于构建最小镜像（Go 静态二进制）
FROM ubuntu:22.04 AS builder           # 多阶段构建，给阶段命名
FROM --platform=linux/amd64 node:18    # 指定平台（在 M1 Mac 上构建 AMD64 镜像）
```

#### RUN — 构建时执行命令

```dockerfile
# Shell 格式（通过 /bin/sh -c 执行）
RUN apt-get update && apt-get install -y nginx

# Exec 格式（直接执行，不通过 shell，不支持管道/重定向/环境变量展开）
RUN ["apt-get", "install", "-y", "nginx"]

# 最佳实践：合并 RUN 命令，同一层内清理
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
        nginx \
        curl \
        ca-certificates && \
    rm -rf /var/lib/apt/lists/*         # 清理 apt 缓存，减小层大小
```

#### COPY vs ADD

```dockerfile
# COPY：简单复制（推荐，语义清晰）
COPY requirements.txt .
COPY ./app /app
COPY --from=builder /app/server .      # 从其他构建阶段复制
COPY --chown=appuser:appuser . .       # 复制并设置所有权

# ADD：COPY 的超集（不推荐，除非需要以下特性）
ADD app.tar.gz /app                    # 自动解压 tar.gz（ADD 独有）
ADD https://example.com/file.txt /tmp/ # 从 URL 下载（不推荐，用 RUN curl 代替）

# 为什么推荐 COPY 而不是 ADD？
# 1. 语义明确，不会有「它会不会自动解压」的疑惑
# 2. ADD 的 URL 功能没有缓存，每次构建都重新下载
```

#### CMD vs ENTRYPOINT

```dockerfile
# CMD：容器默认命令，可被 docker run 后面的命令覆盖
CMD ["nginx", "-g", "daemon off;"]     # Exec 格式（推荐）
CMD nginx -g "daemon off;"             # Shell 格式（不推荐，PID 1 是 /bin/sh）

# ENTRYPOINT：容器入口，不易被覆盖，docker run 的参数会追加
ENTRYPOINT ["python"]
CMD ["app.py"]                         # 默认参数（可被覆盖）

# 组合使用的典型模式：
ENTRYPOINT ["/docker-entrypoint.sh"]   # 初始化脚本（不可覆盖）
CMD ["nginx", "-g", "daemon off;"]     # 默认命令（可覆盖）

# 运行示例：
# docker run myapp              → 执行 /docker-entrypoint.sh nginx -g "daemon off;"
# docker run myapp nginx -V     → 执行 /docker-entrypoint.sh nginx -V
# docker run --entrypoint bash myapp  → 强制覆盖 ENTRYPOINT
```

**PID 1 问题**（面试常考）：

容器内 PID 1 的进程承担特殊职责：接收并转发信号给子进程、回收僵尸进程。

```dockerfile
# 问题：Shell 格式的 CMD，PID 1 是 sh，不会转发信号
CMD python app.py                       # PID 1 = /bin/sh，app.py 是子进程
                                        # docker stop 时，app.py 收不到 SIGTERM

# 解决方案一：使用 Exec 格式
CMD ["python", "app.py"]               # PID 1 = python，直接收到 SIGTERM

# 解决方案二：使用 tini 作为 init 进程（处理僵尸进程）
RUN apt-get install -y tini
ENTRYPOINT ["/usr/bin/tini", "--"]
CMD ["python", "app.py"]

# 解决方案三：exec 替换 shell 进程
# 在 entrypoint.sh 脚本末尾用 exec：
exec "$@"    # 用后续命令替换当前 shell 进程（PID 不变）
```

#### WORKDIR、ENV、ARG

```dockerfile
# WORKDIR：设置工作目录（不存在会自动创建）
WORKDIR /app                           # 后续 RUN/COPY/CMD 都在 /app 下执行
WORKDIR /app/src                       # 可以多次设置，支持相对路径

# ENV：运行时环境变量（构建和运行阶段都有效）
ENV NODE_ENV=production
ENV DB_HOST=localhost DB_PORT=5432     # 一行设置多个
# 引用：$NODE_ENV 或 ${NODE_ENV}

# ARG：仅构建时有效的变量（docker build --build-arg 传入）
ARG VERSION=1.0
ARG REGISTRY=docker.io
RUN echo "Building version $VERSION"

# ARG vs ENV：
# ARG 只在构建时可用，不会保留在镜像中（安全，不会泄露密码）
# ENV 保留在镜像中，容器运行时也可以访问
# 错误做法：ARG DB_PASSWORD=secret（构建缓存中会保留，不安全）
# 正确做法：ENV DB_PASSWORD 通过 -e 在运行时传入
```

#### EXPOSE、VOLUME、USER

```dockerfile
# EXPOSE：声明容器监听的端口（仅文档作用，不实际暴露）
EXPOSE 80
EXPOSE 80/tcp 443/tcp                  # 指定协议
# -P 参数会将所有 EXPOSE 的端口随机映射到宿主机

# VOLUME：声明数据卷挂载点
VOLUME ["/data", "/var/log/app"]       # 容器启动时自动创建匿名卷
# 注意：Dockerfile 中的 VOLUME 在 COPY 之后才有效
# 在 VOLUME 指令之后的 RUN 不会修改该目录

# USER：指定后续 RUN/CMD/ENTRYPOINT 的运行用户
RUN useradd -m -u 1000 appuser         # 先创建用户
USER appuser                            # 切换到该用户
USER 1000:1000                         # 也可以直接用 UID:GID
```

#### HEALTHCHECK

```dockerfile
# 容器健康检查（k8s、docker compose 可以感知健康状态）
HEALTHCHECK --interval=30s \           # 检查间隔
            --timeout=5s \             # 单次超时
            --start-period=10s \       # 启动等待期（期间失败不计）
            --retries=3 \              # 连续失败多少次才标记为 unhealthy
    CMD curl -f http://localhost:5000/health || exit 1

# 查看健康状态
docker inspect --format '{{.State.Health.Status}}' <container>
# 可能值：starting / healthy / unhealthy
```

#### LABEL、ONBUILD、SHELL

```dockerfile
# LABEL：添加元数据标签
LABEL maintainer="your@email.com"
LABEL version="1.0" \
      description="My application" \
      org.opencontainers.image.source="https://github.com/..."

# 查看标签
docker inspect --format '{{json .Config.Labels}}' myimage | python3 -m json.tool

# ONBUILD：当此镜像被作为 FROM 基础镜像时触发的指令
ONBUILD COPY . /app                    # 子镜像 FROM 此镜像时，自动复制代码
ONBUILD RUN pip install -r requirements.txt

# SHELL：修改 RUN 指令使用的 shell
SHELL ["/bin/bash", "-c"]              # 默认是 ["/bin/sh", "-c"]
RUN source /etc/profile && some-command  # bash 特有语法
```

---

### 5.2 .dockerignore

`.dockerignore` 的作用是告诉 Docker build context 中哪些文件不要发送给 Daemon。
**减少 build context 大小，加快构建速度，也防止敏感文件进入镜像。**

```
# .dockerignore

# 版本控制
.git
.gitignore
.gitattributes

# Python 缓存
__pycache__
*.pyc
*.pyo
*.pyd
.Python
*.egg-info/

# 虚拟环境（有自己的依赖，不应该复制进去）
venv/
env/
.env
.venv/

# 测试和文档
tests/
*.test.js
*.spec.js
docs/
*.md
!README.md                            # 排除所有 md，但保留 README.md

# 构建产物
dist/
build/
*.o
*.a

# 敏感文件（最重要！）
.env
.env.local
*.pem
*.key
secrets/

# Node.js
node_modules/
npm-debug.log

# IDE
.vscode/
.idea/
*.swp
```

```bash
# 查看 build context 大小（修改 .dockerignore 前后对比）
du -sh .                               # 总大小
docker build --no-cache . 2>&1 | grep "Sending build context"
# Sending build context to Docker daemon  2.048kB  ← 越小越好
```

---

### 5.3 多阶段构建深度解析

#### Go 语言示例

```dockerfile
# ===== 阶段一：编译 =====
FROM golang:1.21-alpine AS builder

WORKDIR /build
# 先复制依赖文件，利用缓存
COPY go.mod go.sum ./
RUN go mod download

# 再复制代码
COPY . .

# 静态编译（去掉 CGO 依赖，生成纯静态二进制）
RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 \
    go build -ldflags="-w -s" -o server .
#            |   |
#            |   -s: 去掉符号表
#            -w: 去掉 DWARF 调试信息
# 最终二进制文件约 5-10MB

# ===== 阶段二：运行（最小镜像）=====
FROM scratch                           # 完全空白的镜像

# scratch 没有任何工具，需要手动添加
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
COPY --from=builder /build/server /server

EXPOSE 8080
ENTRYPOINT ["/server"]

# 最终镜像大小：约 10-15MB（vs golang:1.21 的 600MB）
```

#### Python 应用示例（带依赖编译）

```dockerfile
# ===== 阶段一：编译 Python 依赖 =====
FROM python:3.11 AS builder

WORKDIR /build

# 安装编译工具
RUN apt-get update && apt-get install -y --no-install-recommends \
    gcc libpq-dev && \
    rm -rf /var/lib/apt/lists/*

COPY requirements.txt .

# 将依赖安装到指定目录（便于下一阶段复制）
RUN pip install --no-cache-dir \
    --target /build/deps \
    -r requirements.txt

# ===== 阶段二：运行 =====
FROM python:3.11-slim

WORKDIR /app

# 只复制编译好的依赖（不复制 gcc 等编译工具）
COPY --from=builder /build/deps /usr/local/lib/python3.11/site-packages/
COPY . .

RUN useradd -m -u 1000 appuser
USER appuser

EXPOSE 5000
CMD ["python", "app.py"]
```

#### 指定复制特定阶段

```dockerfile
FROM alpine AS dev
RUN apk add curl vim
...

FROM alpine AS prod
COPY . /app
...

# 构建时指定目标阶段
# docker build --target dev .      → 只构建 dev 阶段
# docker build --target prod .     → 只构建 prod 阶段（默认最后一个）
# docker build .                   → 构建最后一个阶段（prod）
```

---

### 5.4 镜像体积优化全攻略

```bash
# 测量镜像大小
docker images --format "{{.Repository}}:{{.Tag}}\t{{.Size}}" | sort -k2 -h
```

**优化手段对比**：

| 手段 | 效果 | 示例 |
|------|------|------|
| 使用 slim/alpine 基础镜像 | 节省 50-80% | `python:3.11-slim` vs `python:3.11` |
| 多阶段构建 | 节省 80-95% | Go 应用从 600MB → 15MB |
| 合并 RUN 指令 | 节省 10-30% | 安装和清理在同一层 |
| `.dockerignore` 排除不必要文件 | 加快构建速度 | 排除 node_modules |
| `--no-install-recommends` | 节省 20-40% | apt 只装必要包 |
| `--no-cache-dir`（pip） | 节省 10-20% | pip 不缓存安装包 |
| 清理包管理器缓存 | 节省 50-200MB | `rm -rf /var/lib/apt/lists/*` |

**具体操作**：

```dockerfile
# apt 最佳实践
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
        libpq5 \
        curl && \
    rm -rf /var/lib/apt/lists/*        # 必须在同一个 RUN 层内清理！

# apk (Alpine) 最佳实践
RUN apk add --no-cache nginx curl      # --no-cache 不写缓存文件

# pip 最佳实践
RUN pip install --no-cache-dir \
    --no-compile \                     # 不编译 .pyc 文件（运行时编译）
    -r requirements.txt

# dnf/yum 最佳实践
RUN dnf install -y nginx && \
    dnf clean all && \
    rm -rf /var/cache/dnf
```

---

### 5.5 利用构建缓存加速构建

Docker 按层缓存。**缓存失效时，该层及之后所有层都要重新构建。**

```dockerfile
# 低效写法：代码变化 → 缓存失效 → 重新安装所有依赖
COPY . .                               # 代码变了 → 缓存失效
RUN pip install -r requirements.txt    # 每次都重新安装

# 高效写法：依赖文件不变 → 依赖层命中缓存
COPY requirements.txt .                # 只有 requirements 变才失效
RUN pip install -r requirements.txt    # 缓存命中，跳过
COPY . .                               # 代码变化只影响这一层之后
```

**缓存失效的触发条件**：

| 指令 | 触发条件 |
|------|---------|
| `FROM` | 基础镜像变化 |
| `RUN` | 命令字符串变化 |
| `COPY/ADD` | 文件内容或权限变化 |
| `ENV/ARG` | 值变化（影响后续所有层） |

```bash
# 强制不使用缓存（排查缓存问题时）
docker build --no-cache -t myapp .

# 查看哪些层命中了缓存
docker build -t myapp . 2>&1 | grep -E "CACHED|Step"
# Step 1/8 : FROM python:3.11-slim
#  ---> CACHED          ← 命中缓存
# Step 2/8 : WORKDIR /app
#  ---> CACHED
# Step 3/8 : COPY requirements.txt .
#  ---> abc123def456    ← 未命中，重新构建
```

---

## 6. Docker 网络

### 6.1 网络模式全解析

```bash
# 查看所有网络
docker network ls
# NETWORK ID   NAME      DRIVER    SCOPE
# abc123       bridge    bridge    local   ← 默认网络
# def456       host      host      local
# ghi789       none      null      local
```

#### bridge 模式（默认）

```
宿主机
 ├── eth0: 1.2.3.4（公网IP）
 ├── docker0: 172.17.0.1（虚拟网桥）
 │    ├── veth0 ──→ 容器A: 172.17.0.2
 │    └── veth1 ──→ 容器B: 172.17.0.3
 └── iptables NAT 规则（容器访问外网时 MASQUERADE）
```

```bash
# 验证 bridge 网络
docker run -d --name a nginx:alpine
docker run -d --name b nginx:alpine

docker inspect a --format '{{.NetworkSettings.IPAddress}}'
# 172.17.0.2
docker inspect b --format '{{.NetworkSettings.IPAddress}}'
# 172.17.0.3

# 默认 bridge 中，容器间只能通过 IP 通信
docker exec a ping 172.17.0.3          # OK（可以通信）
docker exec a ping b                   # 失败！（默认 bridge 无 DNS）
```

#### host 模式

```bash
docker run -d --name web --network host nginx:alpine
# 容器直接使用宿主机的 80 端口，不需要 -p

# 查看：宿主机的 ss 可以直接看到 nginx 监听
ss -tlnp | grep :80
# LISTEN  0  511  0.0.0.0:80   ...  users:(("nginx",...))

# 使用场景：
# - node_exporter（需要看到宿主机的所有网络信息）
# - 性能极致场景（省去 NAT 转换开销）
# - 容器需要监听大量动态端口
```

#### none 模式

```bash
docker run -d --name isolated --network none busybox sleep 3600
docker exec isolated ip addr
# 只有 lo（loopback），完全无网络
# 使用场景：安全沙箱、只做数据处理不需要网络的任务
```

#### 自定义 bridge 网络（生产推荐）

```bash
# 创建自定义网络
docker network create \
    --driver bridge \
    --subnet 172.20.0.0/24 \           # 自定义 IP 段
    --gateway 172.20.0.1 \
    myapp-net

# 在自定义网络中，容器间可通过容器名和网络别名通信（内置 DNS）
docker run -d --name db --network myapp-net mysql:8.0
docker run -d --name web --network myapp-net nginx:alpine

docker exec web ping db                # OK！（通过容器名解析）
docker exec web curl http://db:3306    # OK！

# 加入多个网络（一个容器可以同时属于多个网络）
docker network connect frontend-net web  # web 容器加入 frontend-net
docker network disconnect myapp-net web  # 移除
```

**默认 bridge vs 自定义 bridge 对比**：

| 对比 | 默认 bridge（docker0） | 自定义 bridge |
|------|----------------------|--------------|
| 容器间 DNS 解析 | 不支持（只能用 IP） | 支持（容器名 / 别名） |
| 网络隔离 | 所有容器共享同一网络 | 每个应用独立网络 |
| 运行时加入 | 不支持 | 支持 `docker network connect` |
| 推荐使用 | 不推荐 | 推荐 |

### 6.2 端口映射原理

```bash
# 查看端口映射
docker port web
# 80/tcp -> 0.0.0.0:8080

# Docker 通过 iptables NAT 实现端口映射
iptables -t nat -L DOCKER -n
# Chain DOCKER
# DNAT  tcp  0.0.0.0/0 → 0.0.0.0:8080  to:172.17.0.2:80
```

**Docker 与宿主机防火墙的冲突**（重要！）：

```bash
# 问题：即使 ufw/firewalld 禁止了 8080 端口，Docker 的 iptables 规则也绕过了防火墙
# 例如：ufw deny 8080   → 但 Docker 容器映射的 8080 依然可以从外网访问！

# 解决方案一：只绑定本地（推荐）
docker run -p 127.0.0.1:8080:80 nginx  # 只有本机能访问

# 解决方案二：用 Nginx 反向代理，让 Nginx 处理外网请求
# 容器只绑定本地，Nginx 做反向代理

# 解决方案三：修改 Docker daemon 不操作 iptables（副作用：容器访问外网受影响）
# /etc/docker/daemon.json: {"iptables": false}  ← 慎用
```

### 6.3 容器 DNS 解析

```bash
# 查看容器内的 DNS 配置
docker exec web cat /etc/resolv.conf
# nameserver 127.0.0.11     ← Docker 内置 DNS 服务器（自定义网络）
# options ndots:0

# Docker 内置 DNS 解析顺序：
# 1. 容器名解析（myapp-net 网络中的所有容器）
# 2. 用户定义的 extra_hosts（--add-host）
# 3. 宿主机的 /etc/resolv.conf 中的上游 DNS

# 自定义 DNS 服务器
docker run --dns 8.8.8.8 --dns 8.8.4.4 nginx:alpine
docker run --dns-search example.com nginx:alpine   # 添加 DNS 搜索域
```

---

## 7. 数据持久化

### 7.1 三种挂载类型

```
宿主机文件系统
     │
     ├── Bind Mount（绑定挂载）：直接挂载宿主机的指定路径
     │      → 适合：配置文件、开发环境代码热更新
     │
     ├── Volume（命名卷）：Docker 管理的存储区域
     │      → 适合：数据库数据、需要跨容器共享的数据
     │
     └── tmpfs（内存文件系统）：数据仅在内存中
            → 适合：敏感临时数据、高性能临时存储
```

### 7.2 Volume（命名卷）

```bash
# 创建和管理
docker volume create mydata
docker volume ls
docker volume inspect mydata
# [{
#     "Name": "mydata",
#     "Driver": "local",
#     "Mountpoint": "/var/lib/docker/volumes/mydata/_data",
#     ...
# }]

# 使用命名卷
docker run -d \
    --name db \
    -v mydata:/var/lib/mysql \         # 命名卷挂载
    mysql:8.0

# 匿名卷（不推荐，难以管理）
docker run -d -v /var/lib/mysql mysql:8.0   # 自动创建随机名称的卷

# 清理
docker volume rm mydata                # 删除指定卷
docker volume prune                    # 删除所有未使用的卷

# 数据备份（重要！）
docker run --rm \
    -v mydata:/source:ro \             # 挂载要备份的卷（只读）
    -v $(pwd):/backup \                # 挂载备份目标目录
    busybox tar czf /backup/mydata-$(date +%Y%m%d).tar.gz -C /source .
```

### 7.3 Bind Mount（绑定挂载）

```bash
# 长语法（更清晰，推荐）
docker run -d \
    --mount type=bind,source=/etc/nginx,target=/etc/nginx,readonly \
    nginx:alpine

# 短语法
docker run -d \
    -v /etc/nginx:/etc/nginx:ro \      # :ro = 只读
    nginx:alpine

# 开发场景：代码热更新
docker run -d \
    --name dev \
    -v $(pwd):/app \                   # 宿主机当前目录 → 容器 /app
    -p 5000:5000 \
    python:3.11-slim \
    python app.py

# 注意：bind mount 的权限问题
# 容器内 root 修改的文件，宿主机上看是 root 所有
# 容器内用 USER 1000，宿主机目录如果是 root，容器可能无法写入
```

### 7.4 tmpfs（内存文件系统）

```bash
# tmpfs 中的数据只在内存，重启后消失
docker run -d \
    --mount type=tmpfs,destination=/tmp,tmpfs-size=100m \
    nginx:alpine

# 使用场景：
# - 存放运行时生成的敏感临时文件（密码、token）
# - 高频读写的临时数据（缓存文件）
# - 测试时不想污染磁盘
```

### 7.5 Volume vs Bind Mount 选择指南

| 场景 | 推荐方式 | 原因 |
|------|---------|------|
| 数据库数据 | Volume | Docker 管理，不依赖宿主机目录结构 |
| 配置文件 | Bind Mount | 直接编辑宿主机文件即可 |
| 开发代码 | Bind Mount | 实时同步代码改动 |
| 日志文件 | Bind Mount 或 Volume | 需要在宿主机上直接查看 |
| 跨容器共享 | Volume | 多个容器可挂载同一个 Volume |
| 备份传输 | Volume | 独立于容器，易于备份 |

---

## 8. Docker Compose

### 8.1 docker-compose.yml 完整语法

```yaml
version: "3.8"

# ===== Services（服务定义）=====
services:

  # ── 基础服务示例 ──
  web:
    # 镜像来源（二选一）
    image: nginx:1.24                  # 使用现有镜像
    build:                             # 从 Dockerfile 构建
      context: ./app                   # Dockerfile 所在目录
      dockerfile: Dockerfile.prod      # 指定 Dockerfile 文件名（默认 Dockerfile）
      args:                            # 构建参数（ARG）
        APP_VERSION: "2.0"
      target: production               # 多阶段构建，指定目标阶段

    # 容器配置
    container_name: my-web             # 指定容器名（不指定则 projectname_service_n）
    hostname: web-server               # 容器内 hostname

    # 端口映射
    ports:
      - "80:80"                        # 宿主机:容器
      - "127.0.0.1:443:443"            # 只绑定本地
      - target: 8080                   # 长语法
        published: 80
        protocol: tcp

    # 环境变量
    environment:
      - NODE_ENV=production             # 列表格式
      DB_HOST: db                       # 映射格式
    env_file:
      - .env                           # 从文件读取
      - .env.local                     # 多个文件

    # 数据卷
    volumes:
      - ./html:/usr/share/nginx/html:ro     # 绑定挂载
      - app-data:/var/lib/app               # 命名卷
      - type: bind                          # 长语法
        source: ./config
        target: /etc/app/config
        read_only: true

    # 网络
    networks:
      - frontend
      - backend
    network_mode: "service:proxy"      # 共享另一个服务的网络（sidecar 模式）

    # 依赖关系
    depends_on:
      db:                              # 简单版（只控制启动顺序）
        condition: service_healthy     # 高级版（等待健康检查通过）
      redis:
        condition: service_started

    # 重启策略
    restart: unless-stopped

    # 健康检查
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost/health"]
      interval: 30s
      timeout: 5s
      retries: 3
      start_period: 10s

    # 资源限制（compose v3，需要 docker swarm 或指定 deploy）
    deploy:
      resources:
        limits:
          cpus: '1.0'
          memory: 512M
        reservations:
          memory: 128M

    # 命令覆盖
    command: ["nginx", "-g", "daemon off;"]   # 覆盖 CMD
    entrypoint: ["/docker-entrypoint.sh"]     # 覆盖 ENTRYPOINT

    # 日志
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"

    # 其他
    user: "1000:1000"
    working_dir: /app
    extra_hosts:
      - "db-legacy:192.168.1.100"      # 添加 /etc/hosts 记录
    labels:
      - "traefik.enable=true"
    stdin_open: true                   # 保持 stdin（-i）
    tty: true                          # 分配 TTY（-t）

  # ── 数据库服务 ──
  db:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: ${DB_ROOT_PASSWORD}   # 从 .env 文件读取
      MYSQL_DATABASE: myapp
      MYSQL_USER: appuser
      MYSQL_PASSWORD: ${DB_PASSWORD}
    volumes:
      - db-data:/var/lib/mysql
      - ./mysql/init.sql:/docker-entrypoint-initdb.d/init.sql:ro  # 初始化 SQL
    ports:
      - "127.0.0.1:3306:3306"          # 只本地访问，不对外暴露
    networks:
      - backend
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "-u", "root", "-p${DB_ROOT_PASSWORD}"]
      interval: 10s
      timeout: 5s
      retries: 5

# ===== Volumes（命名卷定义）=====
volumes:
  app-data:                            # 使用 Docker 管理的本地卷
  db-data:
    driver: local
    driver_opts:                       # 高级：挂载 NFS 共享
      type: nfs
      o: addr=192.168.1.100,rw
      device: ":/path/on/nfs"

# ===== Networks（网络定义）=====
networks:
  frontend:
    driver: bridge
  backend:
    driver: bridge
    internal: true                     # 内部网络，不能访问外网

  # 使用已存在的外部网络（不由 Compose 管理）
  existing-net:
    external: true
    name: my-existing-network
```

### 8.2 Compose 命令全集

```bash
# ===== 构建与启动 =====
docker compose up                      # 前台启动（Ctrl+C 停止）
docker compose up -d                   # 后台启动
docker compose up -d --build           # 强制重新构建镜像后启动
docker compose up -d --force-recreate  # 强制重新创建容器（配置未变也重建）
docker compose up -d --no-deps web     # 只启动 web，不启动依赖服务
docker compose up -d web db            # 只启动指定服务

# ===== 查看状态 =====
docker compose ps                      # 所有服务状态
docker compose ps -a                   # 包含已停止的
docker compose top                     # 各服务进程
docker compose port web 80             # 查看 web 服务 80 端口映射到宿主机哪个端口

# ===== 日志 =====
docker compose logs                    # 所有服务日志
docker compose logs -f                 # 实时跟踪
docker compose logs -f --tail 50 web   # 只看 web 服务最后 50 行
docker compose logs --since 1h        # 最近 1 小时

# ===== 生命周期 =====
docker compose stop                    # 停止（保留容器）
docker compose start                   # 启动已停止的服务
docker compose restart                 # 重启所有服务
docker compose restart web             # 只重启 web
docker compose pause web               # 暂停
docker compose unpause web

# 彻底清理
docker compose down                    # 停止并删除容器 + 网络
docker compose down --rmi all          # 同上 + 删除镜像
docker compose down -v                 # 同上 + 删除数据卷（危险！生产慎用）

# ===== 执行与调试 =====
docker compose exec web bash           # 进入 web 容器
docker compose exec -T db mysql -uroot -p  # -T 禁用 TTY（适合脚本）
docker compose run --rm web python manage.py migrate  # 一次性运行（用完删容器）
docker compose run --rm --no-deps web env  # 不启动依赖，查看环境变量

# ===== 构建 =====
docker compose build                   # 构建所有有 build 配置的服务
docker compose build --no-cache web    # 不使用缓存重建 web 镜像
docker compose push                    # 推送所有镜像到仓库
docker compose pull                    # 拉取最新镜像

# ===== 扩缩容 =====
docker compose up -d --scale web=3    # 将 web 扩展为 3 个实例

# ===== 配置校验 =====
docker compose config                  # 合并 + 验证配置，输出最终配置
docker compose config --services       # 列出所有服务名
docker compose config --volumes        # 列出所有卷
```

### 8.3 环境变量与多环境配置

```bash
# .env 文件（默认自动加载，不要提交到 git！）
DB_ROOT_PASSWORD=supersecret
DB_PASSWORD=apppassword
APP_PORT=8080

# 在 docker-compose.yml 中引用
ports:
  - "${APP_PORT}:5000"
```

**多环境配置（开发/测试/生产）**：

```bash
# docker-compose.yml（基础配置）
# docker-compose.override.yml（开发环境，自动合并）
# docker-compose.prod.yml（生产环境，手动指定）

# 开发环境（默认，自动合并 docker-compose.yml + docker-compose.override.yml）
docker compose up -d

# 生产环境（只用 docker-compose.yml + prod 配置）
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d
```

```yaml
# docker-compose.override.yml（开发环境特有配置）
services:
  app:
    build:
      target: development              # 开发阶段
    volumes:
      - .:/app                         # 代码热更新
    environment:
      - DEBUG=true
    command: python -m flask run --reload --host=0.0.0.0
```

```yaml
# docker-compose.prod.yml（生产环境特有配置）
services:
  app:
    image: registry.example.com/myapp:${VERSION}   # 从仓库拉取固定版本
    restart: unless-stopped
    deploy:
      resources:
        limits:
          memory: 512m
```

---

## 9. 资源限制与性能调优

### 9.1 内存限制

```bash
# --memory：容器使用的最大内存（超过则触发 OOM Killer）
# --memory-swap：memory + swap 的总限制
# --memory-reservation：软限制（内存紧张时优先回收超过此值的容器）
# --oom-kill-disable：禁止 OOM Kill（慎用，可能导致宿主机 OOM）

docker run -d \
    --name web \
    --memory 512m \                    # 最多 512MB 内存
    --memory-swap 1g \                 # 最多 512MB 内存 + 512MB swap
    --memory-reservation 256m \       # 软限制 256MB
    nginx:alpine

# 验证 OOM Kill：
docker run --name oom-test --memory 100m python:3.11-slim python -c "
a = []
while True:
    a.append(' ' * 1024 * 1024)       # 每次分配 1MB，直到 OOM
"
# 观察容器被 OOM Kill
docker inspect oom-test --format '{{.State.OOMKilled}}'
# true

# 查看宿主机 OOM 日志
dmesg | grep -i "out of memory"
```

### 9.2 CPU 限制

```bash
# --cpus：最多使用几个 CPU 核（可以是小数）
docker run -d --name web --cpus 1.5 nginx:alpine   # 最多使用 1.5 个核

# --cpu-shares：相对权重（默认 1024）
# 两个容器，权重 512:1024，CPU 争用时 512 权重的容器只能拿到 1/3 的 CPU
docker run -d --name low  --cpu-shares 512  nginx:alpine
docker run -d --name high --cpu-shares 1024 nginx:alpine

# --cpuset-cpus：限制使用哪几个 CPU 核（NUMA 优化）
docker run -d --name app --cpuset-cpus "0,2" nginx:alpine  # 只用第 0 和 2 核

# 验证 CPU 限制
docker run --rm --cpus 0.5 \
    python:3.11-slim \
    python -c "
import time
start = time.time()
while time.time() - start < 10:
    pass  # 空循环，消耗 CPU
"
# docker stats 观察，CPU 使用率不超过 50%
```

### 9.3 IO 限制

```bash
# --device-read-bps：限制读取速度
# --device-write-bps：限制写入速度
docker run -d \
    --device-read-bps /dev/sda:10mb \   # 读取限速 10MB/s
    --device-write-bps /dev/sda:5mb \   # 写入限速 5MB/s
    nginx:alpine

# 查看宿主机块设备
lsblk

# 测试磁盘 IO
docker run --rm \
    --device-write-bps /dev/sda:1mb \
    busybox dd if=/dev/zero of=/tmp/test bs=1M count=100
```

---

## 10. 日志管理

### 10.1 日志驱动

Docker 支持多种日志驱动，通过 `daemon.json` 全局设置或 `docker run --log-driver` 单容器设置：

| 驱动 | 存储位置 | 适用场景 |
|------|---------|---------|
| `json-file`（默认） | 本地 JSON 文件 | 小规模，配合 `max-size` |
| `journald` | systemd journal | 与系统日志集成 |
| `syslog` | syslog 服务 | 传统日志基础设施 |
| `fluentd` | Fluentd 服务 | 大规模日志收集 |
| `gelf` | Graylog | ELK 生态 |
| `none` | 不记录 | 高性能、无需日志 |

```bash
# 查看容器使用的日志驱动
docker inspect --format '{{.HostConfig.LogConfig.Type}}' <container>

# 设置单个容器的日志驱动
docker run -d \
    --log-driver json-file \
    --log-opt max-size=10m \
    --log-opt max-file=3 \
    nginx:alpine

# 日志文件实际位置
ls /var/lib/docker/containers/<container_id>/
# <container_id>-json.log   ← 实际日志文件
```

### 10.2 日志查看技巧

```bash
# 基本查看
docker logs <container>
docker logs -f <container>             # 实时追踪
docker logs --tail 100 <container>
docker logs --since "2026-04-15T10:00:00" <container>
docker logs --until "2026-04-15T11:00:00" <container>
docker logs --since 2h --until 1h <container>   # 1-2小时前的日志

# 搜索日志
docker logs <container> 2>&1 | grep "ERROR"
docker logs <container> 2>&1 | grep -E "ERROR|WARN"

# 注意：很多应用把日志输出到 stderr，不是 stdout
# 2>&1 将 stderr 合并到 stdout，才能被 grep 到
```

### 10.3 容器不输出日志的排查

```bash
# 问题：docker logs 看不到任何内容
# 可能原因：
# 1. 应用把日志写到文件，不是 stdout/stderr
docker exec <container> ls /var/log/app/   # 检查日志文件

# 2. 应用有缓冲，还没 flush
docker exec <container> env | grep PYTHONUNBUFFERED  # Python 需要此变量

# 3. 日志被丢弃（log-driver: none）
docker inspect --format '{{.HostConfig.LogConfig.Type}}' <container>

# 最佳实践：应用日志直接输出到 stdout/stderr，由 Docker 统一管理
# Python：print() 直接输出到 stdout
# Nginx：/dev/stdout 和 /dev/stderr（Docker 官方镜像已配置）
```

---

## 11. 安全最佳实践

### 11.1 不要以 root 运行容器

```dockerfile
# 创建非 root 用户
RUN groupadd -r appgroup && \
    useradd -r -g appgroup -u 1000 appuser

# 设置目录权限
RUN chown -R appuser:appgroup /app

# 切换用户
USER appuser
```

```bash
# 验证容器内运行用户
docker exec <container> whoami        # 应该是 appuser 而非 root
docker exec <container> id
```

### 11.2 限制 Linux Capabilities

```bash
# Linux Capabilities 将 root 权限拆分为细粒度权限
# 默认情况下 Docker 给容器一组能力，大多数应用不需要全部

# 最安全的做法：移除所有能力，按需添加
docker run -d \
    --cap-drop ALL \                   # 移除所有能力
    --cap-add NET_BIND_SERVICE \       # 只添加绑定 <1024 端口的能力
    --security-opt no-new-privileges \ # 禁止 setuid 提权
    nginx:alpine

# 常用能力说明
# NET_BIND_SERVICE  绑定 1024 以下端口（Web 服务需要）
# SYS_TIME          修改系统时间
# SYS_ADMIN         各种管理操作（非常危险，相当于 root）
# NET_ADMIN         配置网络（iptables 等）
# CHOWN             修改文件所有权
```

### 11.3 只读文件系统

```bash
# 让容器文件系统只读，阻止攻击者写入后门
docker run -d \
    --read-only \                      # 根文件系统只读
    --tmpfs /tmp \                     # 允许写入 /tmp（内存）
    --tmpfs /var/run \                 # 允许写入 PID 文件等
    -v app-logs:/var/log/app \         # 允许写入日志（数据卷）
    myapp:v1
```

### 11.4 镜像安全扫描

```bash
# 使用 Trivy 扫描镜像漏洞（常用免费工具）
# 安装
curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh

# 扫描镜像
trivy image nginx:1.24
trivy image --severity HIGH,CRITICAL myapp:v1  # 只显示高危和严重漏洞

# 在 CI/CD 中集成（GitHub Actions）
- name: Scan image for vulnerabilities
  uses: aquasecurity/trivy-action@master
  with:
    image-ref: 'myapp:${{ github.sha }}'
    severity: 'CRITICAL,HIGH'
    exit-code: '1'                     # 发现漏洞时 CI 失败
```

### 11.5 私有镜像仓库

```bash
# 阿里云容器镜像服务（ACR）- 国内推荐
# 免费的个人版

# 登录
docker login --username=<阿里云账号> registry.cn-hangzhou.aliyuncs.com

# 推送
docker tag myapp:v1 registry.cn-hangzhou.aliyuncs.com/<namespace>/myapp:v1
docker push registry.cn-hangzhou.aliyuncs.com/<namespace>/myapp:v1

# 在 CI 中使用（GitHub Actions）
- name: Login to Aliyun Registry
  run: docker login -u ${{ secrets.ALIYUN_REGISTRY_USER }} \
       -p ${{ secrets.ALIYUN_REGISTRY_PASSWORD }} \
       registry.cn-hangzhou.aliyuncs.com
```

---

## 12. 常见故障排查

### 12.1 故障速查表

| 现象 | 第一步 | 常见原因 |
|------|-------|---------|
| 容器启动即退出 | `docker logs <c>` | CMD 执行失败、程序 crash |
| `CrashLoopBackOff` | `docker logs <c> --previous` | 启动依赖未就绪（DB 还没起来） |
| `OOMKilled` | `docker inspect <c> | grep OOM` | 内存限制太小 |
| `ImagePullBackOff` | `docker events` | 网络问题、镜像名错误、仓库需要登录 |
| 端口无法访问 | `ss -tlnp` + `iptables -L` | 防火墙、Docker 只绑了 127.0.0.1 |
| 容器间无法通信 | `docker network inspect` | 不在同一网络 |
| 磁盘空间不足 | `docker system df` | 镜像/容器/卷积累太多 |
| 容器内时区错误 | `date` 命令确认 | 未挂载 `/etc/localtime` |

### 12.2 容器启动失败排查流程

```bash
# Step 1：查看容器状态
docker ps -a --filter name=myapp
# STATUS: Exited (1) 30 seconds ago  ← 退出码 1 表示错误

# Step 2：查看日志
docker logs myapp
# Error: Cannot connect to database at db:3306

# Step 3：查看详细事件
docker events --since 5m --filter container=myapp

# Step 4：检查配置
docker inspect myapp

# Step 5：手动执行命令调试（覆盖 CMD 进入容器）
docker run -it --rm myapp bash
# 在容器内手动运行应用，观察报错

# Step 6：检查环境变量是否正确传入
docker exec myapp env | grep DB_
```

### 12.3 网络连通性排查

```bash
# 两容器无法通信
docker exec container-a ping container-b
# ping: bad address 'container-b'   ← 没有 DNS 解析 → 不在同一网络

# 查看容器所在网络
docker inspect container-a --format '{{json .NetworkSettings.Networks}}'
docker inspect container-b --format '{{json .NetworkSettings.Networks}}'

# 将两个容器加入同一网络
docker network create mynet
docker network connect mynet container-a
docker network connect mynet container-b

# 端口无法从外部访问
ss -tlnp | grep <port>              # 确认是否在监听
docker port <container>              # 确认端口映射
iptables -t nat -L DOCKER -n       # 确认 iptables 规则是否存在
curl -v http://127.0.0.1:<port>    # 本地测试
```

### 12.4 性能问题排查

```bash
# 查看哪个容器消耗资源最多
docker stats --no-stream --format "table {{.Name}}\t{{.CPUPerc}}\t{{.MemPerc}}\t{{.MemUsage}}" | \
    sort -k2 -rh | head

# 容器内进程级别排查
docker exec -it <container> top
docker exec -it <container> ps aux

# 查看容器的 IO 情况
docker exec -it <container> sh -c "cat /proc/1/io"

# 内存泄漏排查
docker stats <container> --format "{{.MemUsage}}"
# 观察内存是否持续增长
```

### 12.5 磁盘空间清理

```bash
# 查看 Docker 占用的磁盘空间
docker system df
# TYPE            TOTAL     ACTIVE    SIZE      RECLAIMABLE
# Images          15        3         8.5GB     6.2GB (73%)
# Containers      5         2         245MB     0B (0%)
# Local Volumes   8         2         15.5GB    12.5GB (80%)
# Build Cache     142       0         4.2GB     4.2GB

# 精确查看
docker system df -v

# 清理（从安全到危险排序）
docker container prune -f             # 删除已停止的容器
docker image prune -f                 # 删除悬空镜像
docker volume prune -f                # 删除未使用的卷
docker network prune -f               # 删除未使用的网络
docker builder prune -f               # 删除构建缓存
docker system prune -f                # 以上全部（不含卷）
docker system prune -f --volumes      # 以上全部（含卷，危险！）
```

---

## 13. 综合实战项目

### 项目：从零构建完整 Web 应用栈

在**上海 VPS** 上完成以下任务，主动制造并排查每种故障。

#### 任务一：构建优化的 Flask 镜像

```bash
mkdir -p ~/docker-practice/flask && cd ~/docker-practice/flask

# 应用代码
cat > app.py << 'EOF'
from flask import Flask, jsonify
import os, platform, datetime

app = Flask(__name__)

@app.route('/')
def index():
    return jsonify({
        "status": "ok",
        "hostname": platform.node(),
        "env": os.getenv("APP_ENV", "unknown"),
        "time": datetime.datetime.now().isoformat()
    })

@app.route('/health')
def health():
    return jsonify({"status": "healthy"}), 200

@app.route('/error')
def error():
    raise RuntimeError("Intentional error for testing")

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000, debug=os.getenv("DEBUG") == "true")
EOF

cat > requirements.txt << 'EOF'
flask==3.0.0
gunicorn==21.2.0
EOF

# 编写优化后的 Dockerfile
cat > Dockerfile << 'EOF'
FROM python:3.11-slim

WORKDIR /app

# 先安装依赖（利用缓存）
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# 再复制代码
COPY . .

# 非 root 用户运行
RUN useradd -m -u 1000 appuser && chown -R appuser:appuser /app
USER appuser

EXPOSE 5000

# 生产环境用 gunicorn
CMD ["gunicorn", "--bind", "0.0.0.0:5000", "--workers", "2", "app:app"]
EOF

cat > .dockerignore << 'EOF'
__pycache__
*.pyc
.git
.env
EOF

# 构建并记录大小
docker build -t flask-app:v1 .
docker images flask-app:v1

# 运行
docker run -d \
    --name flask-app \
    -p 5000:5000 \
    -e APP_ENV=production \
    --restart unless-stopped \
    --memory 256m \
    flask-app:v1

# 验证
curl http://localhost:5000
curl http://localhost:5000/health
docker logs flask-app
docker stats flask-app --no-stream
```

**故障练习一：OOM Kill**

```bash
# 给容器设置极小内存，触发 OOM
docker run -d --name oom-test --memory 30m flask-app:v1
sleep 5
docker inspect oom-test --format '{{.State.OOMKilled}}'
# 预期：true
docker logs oom-test
# 预期：Killed
```

#### 任务二：Docker Compose 编排多容器栈

```bash
mkdir -p ~/docker-practice/stack && cd ~/docker-practice/stack
mkdir -p nginx

cat > docker-compose.yml << 'EOF'
version: "3.8"

services:
  nginx:
    image: nginx:1.24
    ports:
      - "80:80"
    volumes:
      - ./nginx/default.conf:/etc/nginx/conf.d/default.conf:ro
    depends_on:
      app:
        condition: service_healthy
    restart: unless-stopped
    networks:
      - frontend

  app:
    build: ../flask
    environment:
      - APP_ENV=production
    deploy:
      resources:
        limits:
          memory: 256m
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:5000/health"]
      interval: 10s
      timeout: 5s
      retries: 3
      start_period: 5s
    restart: unless-stopped
    networks:
      - frontend
      - backend

  redis:
    image: redis:7-alpine
    volumes:
      - redis-data:/data
    command: redis-server --appendonly yes
    restart: unless-stopped
    networks:
      - backend

volumes:
  redis-data:

networks:
  frontend:
  backend:
    internal: true
EOF

cat > nginx/default.conf << 'EOF'
upstream app {
    server app:5000;
}

server {
    listen 80;
    server_name _;

    location / {
        proxy_pass http://app;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    location /health {
        proxy_pass http://app/health;
        access_log off;
    }
}
EOF

# 启动
docker compose up -d

# 验证
docker compose ps
curl http://localhost
```

**故障练习二：depends_on 的局限性**

```bash
# 故意让 app 启动变慢（模拟 DB 初始化）
# 修改 app.py，在启动时 sleep 20 秒
# 不用 condition: service_healthy 时，nginx 先启动，但 app 还没好
# 观察 nginx 返回 502

# 加上 healthcheck + condition: service_healthy 后：
# nginx 等待 app health 检查通过后才启动，502 消失
docker compose logs nginx
```

**故障练习三：数据卷丢失**

```bash
# 执行以下命令，观察 redis 数据是否丢失
docker compose down          # 停止但不删除卷
docker compose up -d         # 重启，数据应该还在

docker compose down -v       # 删除卷！数据丢失！
docker compose up -d         # 重启，数据已丢失

# 养成习惯：down 命令永远不加 -v，除非你明确知道要删除数据
```

---

## 14. 面试题精选

### 基础概念

**Q：容器和虚拟机的区别？**

```
虚拟机：通过 Hypervisor 模拟完整硬件，每个 VM 有独立内核和完整 OS，隔离性强但开销大（GB级）。
容器：共享宿主机内核，通过 Namespace 隔离进程视图，通过 Cgroup 限制资源，启动快（秒级）、占用少（MB级）。
不是替代关系：容器通常跑在 VM 里（VM 提供基础隔离，容器提供应用隔离）。
```

**Q：Docker 的底层技术是什么？**

```
Namespace：实现隔离（PID/Net/Mnt/IPC/UTS/User 六种）
Cgroup：实现资源限制（CPU/内存/IO/网络带宽）
Union FS（overlay2）：实现镜像分层，写时复制，多容器共享镜像层
```

**Q：镜像和容器的关系？**

```
镜像是只读的分层模板，容器是镜像的运行实例。
容器 = 镜像的所有只读层 + 一个可写的容器层。
一个镜像可以同时运行多个容器，它们共享镜像层但有各自独立的可写层。
```

### Dockerfile

**Q：CMD 和 ENTRYPOINT 的区别？**

```
CMD：容器默认执行的命令，可以被 docker run 后面的参数完全覆盖。
ENTRYPOINT：容器的固定入口，docker run 后面的参数会追加到 ENTRYPOINT 后。
组合使用：ENTRYPOINT 定义不变的入口，CMD 定义可覆盖的默认参数。
```

**Q：COPY 和 ADD 的区别，应该用哪个？**

```
COPY：纯粹复制文件，语义清晰。
ADD：额外支持自动解压 tar 文件和从 URL 下载。
优先用 COPY，只有需要自动解压 tar 时才用 ADD。
不要用 ADD 从 URL 下载（没有缓存，每次都下载；用 RUN curl 代替）。
```

**Q：如何减小 Docker 镜像体积？**

```
1. 使用 slim/alpine 基础镜像（python:3.11-slim 比 python:3.11 小 70%）
2. 多阶段构建（构建阶段的工具链不进入最终镜像）
3. 合并 RUN 指令（减少层数，且在同一层内清理缓存）
4. 使用 .dockerignore 排除不必要的文件
5. apt 加 --no-install-recommends，pip 加 --no-cache-dir
6. 安装依赖后立即清理包管理器缓存（在同一 RUN 层内！）
```

### 网络

**Q：Docker bridge 网络和自定义 bridge 网络有什么区别？**

```
默认 bridge（docker0）：
  - 容器间只能通过 IP 通信，没有 DNS 解析
  - 所有容器共享同一网络，隔离性差

自定义 bridge：
  - 内置 DNS，容器间可通过容器名互相访问
  - 每个应用独立网络，天然隔离
  - 支持运行时动态加入/退出
  - 生产环境推荐使用
```

**Q：Docker 为什么会绕过 ufw/firewalld？**

```
Docker 直接操作 iptables 的 DOCKER 链，插入的 NAT 规则优先级高于 ufw/firewalld 管理的规则。
即使 ufw 禁止了某端口，Docker 映射的端口依然可以从外网访问。
解决方案：将容器端口只绑定到 127.0.0.1（-p 127.0.0.1:8080:80），再用 Nginx 反向代理出去。
```

### 故障排查

**Q：容器启动后立即退出，如何排查？**

```
1. docker logs <container>         # 看退出原因
2. docker ps -a                    # 看退出码（exit 0 正常，其他异常）
3. docker inspect <container>      # 看 State.ExitCode 和 State.Error
4. docker run -it --rm <image> bash  # 覆盖 CMD，手动进入排查
```

**Q：如何让容器平滑处理停止信号（优雅退出）？**

```
问题：Shell 格式的 CMD（CMD python app.py）中，PID 1 是 sh，
     docker stop 发送的 SIGTERM 不会转发给 python 进程，等 10 秒后被强杀。

解决：使用 Exec 格式（CMD ["python", "app.py"]），python 直接是 PID 1，
     能收到 SIGTERM 并做清理（关闭连接、flush 日志）后优雅退出。
```

**Q：docker system df 显示镜像占用了 50GB，如何安全清理？**

```bash
docker images                          # 查看所有镜像，标记哪些可以删
docker ps -a                           # 检查是否有已停止的容器用到这些镜像
docker image prune -f                  # 先清理悬空镜像（最安全）
docker container prune -f              # 清理已停止容器
docker image prune -a                  # 清理所有未使用镜像（确认没有重要镜像后再做）
```

---

> **下一节**：[CI/CD 基础](./phase2-toolchain.md#2-cicd-基础)
> 将 Docker 构建集成到 GitHub Actions，实现代码提交自动构建和部署
