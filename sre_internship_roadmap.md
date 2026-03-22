# 运维实习准备路线图（3月 → 7月）

> **起点**：大三下，有 4 台 VPS（百度云北京 / 火山引擎上海 / 华为云广州 / 火山引擎雅加达），
> 已完成 Python 运维脚本教程、Docker 基础使用（NPM / Uptime Kuma / Portainer / gh-proxy）

---

## 时间线总览

| 阶段 | 时间 | 核心目标 | 状态 |
|------|------|---------|------|
| **第一阶段：Linux 核心基础** | 3月中 → 4月中 | Linux + 网络 + Shell 扎实 | 已完成 |
| **第二阶段：工具链实战** | 4月中 → 5月底 | Docker / CI-CD / 监控 / Nginx | 进行中 |
| **第三阶段：K8s + 项目实战** | 6月 → 7月初 | K8s 基础 + 2个可展示项目 | 待开始 |
| **第四阶段：投递 + 面试** | 7月 | 刷面经、投递、模拟面试 | 待开始 |

> **为什么第二阶段从 1 个月延长到 6 周？**
> 原计划 1 个月内覆盖 Docker / CI-CD / Prometheus / Nginx，
> 实际上每项都需要 1-2 周才能真正操作熟练。压缩时间只会造成「看懂了但做不了」。

---

## 学习原则（先读这里）

在看任何阶段的内容之前，先理解这几条原则。它们决定你最终能不能真正学到手。

### 原则一：「能讲出来」才算掌握

每学完一个技术点，问自己：**不看笔记，能流畅讲 2 分钟吗？**

如果不能，说明还没掌握，需要再做一遍或重新整理笔记。

### 原则二：故意破坏，主动排查

每部署一个服务，不要满足于「能跑起来」，要主动制造故障：

```
部署完 Nginx 后：
  → 故意写错配置文件，观察 nginx -t 的报错
  → 故意把后端端口填错，触发 502，看日志排查
  → 故意删掉 SSL 证书文件，观察 HTTPS 怎么报错
  → 修复它，把排查过程写进笔记
```

**会排查故障，比会部署服务更有价值。**

### 原则三：写「为什么」，不只写「怎么做」

```
弱笔记：firewall-cmd --reload  # 重载规则
强笔记：为什么要先 --permanent 再 --reload？
        firewalld 有两套配置：内存中的运行时配置 + 磁盘上的持久配置。
        --permanent 只写磁盘，不影响运行中的规则。
        --reload 把磁盘配置加载到内存，规则才真正生效。
        对比 ufw：直接修改运行时并持久化，所以不需要 reload。
```

### 原则四：最小可用，每天有完成感

不要计划「这周搭好整套监控系统」，要拆成每天能完成的最小任务：

```
第 1 天：让 Prometheus 成功抓到本机 node_exporter 的数据
第 2 天：在 Grafana 上看到一条 CPU 折线图
第 3 天：写一条告警规则并触发它
第 4 天：排查为什么某个远程 target 是 down 状态
第 5 天：整理成笔记，能口头讲述整套监控的工作原理
```

### 原则五：现在就开始投简历

不要等「准备好了再投」。你永远不会觉得准备好了。

**在第二阶段开始时就投中小公司**，面试被问到不会的问题，就是最准确的学习方向指引。

---

## 第一阶段：Linux 核心基础（3月中 → 4月中）已完成

### 已掌握内容

- 用户与权限管理：`useradd` / `chmod` / `chown` / `sudo` / `/etc/sudoers`
- 进程管理：`ps aux` / `top` / `htop` / `kill` / `systemctl` / `journalctl`
- 磁盘与文件系统：`df` / `du` / `fdisk` / `mount` / `lsblk` / 软硬链接
- 网络排查：`ss` / `ip addr` / `tcpdump` / `traceroute` / `dig` / `curl`
- 包管理：`dnf` / `apt` / 配置国内源
- 文本处理三剑客：`grep` / `sed` / `awk`
- 日志排查：`tail -f` / `less` / `journalctl`
- 防火墙：`firewalld` / `ufw` / `iptables` 基础
- Shell 脚本：备份脚本 / 健康检查 / Systemd service

### 验收标准

不看笔记，能在 VPS 上独立完成以下操作：

- [ ] 创建用户，配置 sudo 权限，验证生效
- [ ] 排查一个进程为何占用高 CPU（用 top + ps + lsof）
- [ ] 磁盘 100% 时，找到占用最大的目录和文件
- [ ] 用 awk 统计 Nginx 日志 Top 10 IP
- [ ] 用 sed 批量替换配置文件中的某个值并备份原文件
- [ ] 写一个带 `set -euo pipefail` 的 Shell 脚本，有完整错误处理

详细教程：[phase1-linux-sysadmin.md](./phase1-linux-sysadmin.md)

---

## 第二阶段：工具链实战（4月中 → 5月底，共 6 周）

详细教程：[phase2-toolchain.md](./phase2-toolchain.md)

### 第 1-2 周：Docker 进阶

**掌握目标**：

- Dockerfile 编写：多阶段构建、镜像优化（体积减半）
- Docker Compose：多容器编排、服务依赖、数据卷管理
- Docker 网络：理解 bridge / host / 自定义网络的区别
- 日志管理：限制容器日志大小，`docker logs` 熟练使用

**VPS 实操任务**：

```
1. 手写 Dockerfile 打包 Flask 应用，镜像体积控制在 150MB 以内
2. 用 Docker Compose 部署 Nginx + WordPress + MySQL
3. 故意触发以下故障并排查：
   - 容器内存超限被 OOM kill（docker stats 观察，dmesg 确认）
   - 数据卷权限错误导致服务启动失败
   - 容器间通信失败（默认 bridge vs 自定义网络的 DNS 区别）
```

**掌握验收**：不看文档，能解释 CMD vs ENTRYPOINT 的区别，
并说出至少 4 种减小镜像体积的方法。

---

### 第 3 周：CI/CD 基础

**掌握目标**：

- Git 分支策略：feature branch、merge vs rebase 的区别和使用场景
- GitHub Actions：编写 CI Pipeline（lint + test）
- 自动部署：push 到 main → SSH 到 VPS → 自动拉取更新并重启服务

**VPS 实操任务**：

```
1. 为你的某个 Python 项目配置 GitHub Actions CI（flake8 + pytest）
2. 配置自动部署 Pipeline：push → SSH → docker compose up -d --build
3. 故意让 CI 失败（加入一行语法错误），观察 GitHub Actions 的报错信息
4. 在 GitHub 仓库 Settings > Secrets 中管理 VPS 的 SSH 密钥
```

**掌握验收**：能描述一次完整的「代码提交 → 自动测试 → 自动部署」流程，
说出每一步发生了什么。

---

### 第 4-5 周：监控与告警（重点）

**掌握目标**：

- Prometheus + Grafana + Alertmanager + node_exporter 完整监控栈
- 在 4 台 VPS 上建立跨地域监控（pull 模式原理）
- PromQL 基础查询：CPU / 内存 / 磁盘 / 网络流量
- 配置告警规则：实例宕机 / CPU 超 80% / 磁盘超 80%
- Grafana Dashboard：导入 Node Exporter Full（ID: 1860）+ 自定义面板

**VPS 实操任务**：

```
主力机（上海）部署：Prometheus + Grafana + Alertmanager（Docker Compose）
其余 3 台部署：node_exporter（systemd 二进制方式，轻量）

故意触发以下故障并排查：
1. 关闭某台 VPS 的 node_exporter，观察 Prometheus 中 target 变为 down
   → 排查：云安全组是否放行 9100？防火墙？服务是否在运行？
2. 在某台 VPS 上运行 stress，触发 CPU 告警
   → 验证 Alertmanager 是否收到告警
3. 修改 Prometheus 配置文件加入语法错误，用 --web.enable-lifecycle 热加载
   → 观察加载失败的错误日志
```

**掌握验收**：能在白板上画出这套监控系统的数据流向，
并解释 Prometheus 为什么用 pull 模式而不是 push 模式。

---

### 第 6 周：Nginx 深入

**掌握目标**：

- 反向代理：`proxy_pass` + 正确传递 `X-Real-IP` 等 Header
- 负载均衡：upstream 配置，5 种策略的区别和适用场景
- HTTPS：用 acme.sh 申请 Let's Encrypt 证书，配置自动续期
- 安全加固：限制请求速率、禁止访问敏感文件

**VPS 实操任务**：

```
1. 在广州 VPS 上配置 Nginx 反向代理到 Flask 应用
2. 配置 upstream 对 3 个后端实例做轮询负载均衡
3. 在雅加达 VPS 上（绑未备案域名）用 acme.sh 申请证书，配置 HTTPS
4. 故意触发以下故障并排查：
   - 后端应用挂掉，Nginx 返回 502 → 从日志定位原因
   - SSL 证书过期（用测试证书模拟）→ 观察浏览器和 curl 的报错
   - 配置文件语法错误后 reload → nginx -t 检测 + journalctl 查错误
```

**掌握验收**：能解释 `proxy_set_header X-Real-IP $remote_addr` 的作用，
以及不加这行后端应用会拿到什么 IP。

---

## 第三阶段：Kubernetes 基础 + 项目实战（6月 → 7月初，共 5 周）

### 第 1-2 周：Kubernetes 基础

> 不需要深入，能用、能讲、会排查基础问题就够应付实习面试。
> 用 **k3s**（轻量 K8s，单节点资源需求低）部署在上海 VPS（2vCPU 4G 够用）。

**掌握目标**：

- 核心对象：Pod / Deployment / Service / Ingress / ConfigMap / Secret
- `kubectl` 基本操作：get / describe / logs / exec / apply / delete
- 理解 K8s 的工作原理：调度器 / etcd / kubelet / kube-proxy
- Helm 基础：用 Helm 安装一个应用

**安装 k3s**：

```bash
# 上海 VPS 安装 k3s（单节点，约 2 分钟）
curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | \
    INSTALL_K3S_MIRROR=cn sh -

# 验证
kubectl get nodes
kubectl get pods -A
```

**实操任务**：

```
1. 用 K8s 部署第二阶段写的 Flask 应用（Deployment + Service）
2. 配置 Ingress 通过域名访问
3. 用 ConfigMap 管理应用配置，用 Secret 管理数据库密码
4. 故意触发以下故障并排查：
   - 镜像名写错，Pod 一直 ImagePullBackOff
     → kubectl describe pod <name> 看事件
   - 应用 crash，Pod 一直 CrashLoopBackOff
     → kubectl logs <pod> --previous 看上次崩溃日志
   - Service 配置错误，无法访问
     → kubectl get endpoints 排查是否关联到 Pod
```

**掌握验收**：能解释 Pod / Deployment / Service 三者的关系，
以及 Service 的 ClusterIP / NodePort 的区别。

---

### 第 3-5 周：两个项目实战

> 这是最重要的阶段。项目的价值不在于「搭了什么」，而在于「遇到了什么问题、怎么解决的」。
> 每个项目都要主动制造故障，把排查过程写进 README。

---

#### 项目一：多节点可观测性平台

**这个项目能展示**：监控 + 日志 + 多地域 + Docker Compose 运维能力

```
技术栈：Prometheus / Grafana / Alertmanager / node_exporter / Loki / Promtail
部署位置：上海 VPS（监控中心），其余 3 台被监控

核心功能：
  - 4 台 VPS 的指标监控（CPU/内存/磁盘/网络）
  - 日志收集（Loki + Promtail）接入同一个 Grafana
  - 告警通知（钉钉机器人 Webhook 或邮件）
  - 自定义 Grafana Dashboard（多地域延迟对比）
```

**Loki 日志收集**（接在 Prometheus 监控之后，补全可观测性）：

```yaml
# 在 docker-compose.yml 中加入 Loki + Promtail
services:
  loki:
    image: grafana/loki:2.9.0
    ports:
      - "127.0.0.1:3100:3100"
    volumes:
      - loki-data:/loki
    restart: unless-stopped

  promtail:
    image: grafana/promtail:2.9.0
    volumes:
      - /var/log:/var/log:ro
      - ./promtail:/etc/promtail
    command: -config.file=/etc/promtail/config.yml
    restart: unless-stopped
```

**必须经历的故障场景（写进 README）**：

| 故障 | 排查工具 | 可讲的经历 |
|------|---------|-----------|
| 跨地域 target 一直 down | 安全组 / 防火墙 / `curl target:9100/metrics` | 网络排查实战 |
| Grafana 无数据 | 数据源配置 / Prometheus 查询 / 时区问题 | 多组件联调 |
| 磁盘写满导致 Prometheus 停止写入 | `df -h` / `docker system df` / 调整 retention | 容量规划意识 |
| Alertmanager 收到告警但没有通知 | Alertmanager 日志 / Webhook 连通性 | 告警链路排查 |

**GitHub 仓库应包含**：

```
monitoring-platform/
├── docker-compose.yml
├── prometheus/
│   ├── prometheus.yml
│   └── alert_rules.yml
├── alertmanager/
│   └── alertmanager.yml
├── grafana/
│   └── dashboards/         # 导出的 Dashboard JSON
├── scripts/
│   └── install-node-exporter.sh  # 一键安装脚本
└── README.md               # 包含架构图 + 部署步骤 + 故障排查记录
```

---

#### 项目二：自动化部署流水线

**这个项目能展示**：CI/CD + Docker + Nginx + K8s 的完整 DevOps 链路

```
技术栈：GitHub Actions / Docker / Nginx / k3s（可选 K8s 方向）
场景：一个 Flask Web 应用，从代码提交到线上更新全自动化

流水线流程：
  git push → GitHub Actions 触发
    → flake8 代码检查
    → pytest 单元测试
    → docker build 构建镜像
    → docker push 推送到仓库（Docker Hub 或阿里云 ACR）
    → SSH 到 VPS，pull 新镜像，docker compose up -d
    → 部署完成通知
```

**必须经历的故障场景（写进 README）**：

| 故障 | 排查工具 | 可讲的经历 |
|------|---------|-----------|
| CI 测试失败 | GitHub Actions 日志 | 发现代码逻辑 bug |
| 镜像推送超时 | 切换为阿里云 ACR | 解决国内网络问题 |
| 部署后应用 502 | `docker logs` / `ss -tlnp` | 新版本启动失败排查 |
| 健康检查失败导致流量未切换 | Nginx upstream 健康检测配置 | 零停机部署思路 |

**GitHub 仓库应包含**：

```
auto-deploy-demo/
├── app/
│   ├── app.py
│   ├── tests/
│   └── requirements.txt
├── Dockerfile
├── docker-compose.yml
├── nginx/
│   └── app.conf
├── .github/
│   └── workflows/
│       └── deploy.yml
└── README.md               # 包含流水线架构图 + 每个步骤的说明
```

---

### 简历如何写这两个项目

用 STAR 法则（情景 → 任务 → 行动 → 结果）：

```
项目一示例写法：
  基于 Prometheus + Grafana + Loki 构建多节点可观测性平台，
  覆盖 4 台分布于北京/上海/广州/雅加达的 VPS，
  实现指标监控、日志收集和告警通知一体化，
  解决了跨云平台、跨地域的统一运维视图问题。

项目二示例写法：
  为 Python Web 应用设计并实现 GitHub Actions CI/CD 流水线，
  覆盖代码检查、自动测试、镜像构建、自动部署全流程，
  将手动部署时间从 15 分钟缩短至代码合并后 3 分钟内自动完成。
```

---

## 第四阶段：投递与面试（7月）

### 面试题准备方向

按照「能在 VPS 上演示」的标准准备，不要背答案。

**Linux 高频题**：

```
必须能实操演示：
- 排查服务器负载高：top → 看 load average → ps aux 找进程 → lsof/strace 定位
- 磁盘满了：df -h → du -sh /* → du -sh /var/* → 找大文件删除
- 服务不通：ss -tlnp → 防火墙 → curl 测端口 → 日志
- 被入侵排查：last → /var/log/secure → ps aux 找可疑进程 → netstat 找外联
```

**Docker 高频题**：

```
必须能清楚解释：
- 容器 vs 虚拟机（内核共享 vs 完全隔离，启动速度，资源占用）
- CMD vs ENTRYPOINT
- 减小镜像体积（多阶段构建、合并 RUN、slim/alpine、.dockerignore）
- Docker 网络模式（bridge / host / 自定义 bridge 的 DNS 区别）
- 数据卷 vs 绑定挂载
```

**监控高频题**：

```
必须能清楚解释：
- Prometheus 为什么用 pull 模式？（主动发现、便于管理 target、无需在被监控端配置）
- PromQL 中 rate() 的含义？（计算每秒平均增长率，用于 counter 类型）
- 告警规则中 for 字段的作用？（持续时间，避免瞬时抖动触发告警）
- Grafana 能接哪些数据源？（Prometheus / Loki / MySQL / Elasticsearch 等）
```

**Kubernetes 高频题**（了解即可，不要求深入）：

```
- Pod / Deployment / Service 的关系？
- Service 的 ClusterIP / NodePort 区别？
- Pod 一直 CrashLoopBackOff 怎么排查？
  → kubectl logs <pod> --previous
  → kubectl describe pod <pod> 看 Events
```

**场景题**（最重要，面试必考）：

```
「线上某个服务响应变慢，你怎么排查？」
标准答题框架：
  1. 确认现象：curl 测响应时间，看监控图表，确定是什么时间开始的
  2. 排查系统层：top 看 CPU/内存，iostat 看磁盘 IO，是否资源瓶颈
  3. 排查网络层：ss 看连接数，tcpdump 抓包，是否网络拥塞
  4. 排查应用层：查看应用日志，是否有大量报错，是否某类请求慢
  5. 排查外部依赖：数据库慢查询，下游服务是否超时
  6. 临时处理：必要时重启服务或回滚版本，同时保留现场（日志/dump）
  7. 根因分析：找到根因后写 Postmortem，避免复现
```

---

### 投递渠道

| 渠道 | 优先级 | 说明 |
|------|-------|------|
| **Boss直聘** | 最高 | 运维实习最多，直接和 HR/技术负责人聊 |
| **实习僧** | 高 | 实习专门平台，信息集中 |
| **牛客网** | 高 | 看面经 + 内推信息 |
| **公司官网** | 中 | 大厂暑期实习（腾讯/阿里/字节/百度/华为） |

> 大厂暑期实习通常 3-4 月就开始投递，截止时间早。
> **建议在第二阶段开始的同时就注册 Boss 直聘**，先搜运维/SRE 岗位，
> 看 JD 里要求什么技能，对照自己的短板调整学习重点。

---

## VPS 资源分配

| VPS | 云平台 | 配置 | 系统 | 用途 |
|-----|--------|------|------|------|
| 北京 | 百度智能云 | 2H2G / 1M | Debian | 生产服务 + 已备案域名 / node_exporter 被监控 |
| 上海 | 火山引擎 | 2vCPU 4G / 5M | Rocky Linux 9.5 | **主力机**：监控中心 + k3s + CI/CD 部署目标 |
| 广州 | 华为云 | 2核 2G / 3M | EulerOS 2.0 | 练习机：Nginx / Shell / 服务部署 / node_exporter |
| 雅加达 | 火山引擎 | 2vCPU 2G / 5M | Rocky Linux 9.5 | 海外节点：HTTPS 练习 + 跨地域监控 / node_exporter |

**多系统的价值**：
- Debian（北京）→ Debian 系：`apt` / `ufw` / `dpkg`
- Rocky Linux（上海/雅加达）→ RHEL 系：`dnf` / `firewalld` / `SELinux`
- EulerOS（广州）→ 华为国产化生态：RHEL 兼容 + 华为加固，面试涉及国产化方向加分

同时熟悉两大发行版阵营，在简历上写「熟悉 Debian 系与 RHEL 系 Linux 运维」。

---

## 每周节奏参考

```
工作日（约 1.5-2 小时，在校期间不要贪多）：
  晚上 1h：VPS 实操一个具体任务（有完成感比时间长更重要）
  晚上 30min：写当天的学习笔记（整理到仓库）

周末（约 4-5 小时）：
  上午 2h：理论学习或补上工作日没完成的实操
  下午 2-3h：项目推进 + 故障排查练习

GitHub 提交频率：工作日至少每 2 天一次提交，周末必须有提交。
```

> 不要在某一天学 8 小时然后停 3 天。
> 每天 1 小时连续 30 天，远比突击 1 周然后断掉有效。
