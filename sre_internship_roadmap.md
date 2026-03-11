# 🚀 运维实习准备路线图（3月 → 7/8月）

> **起点**：大三下，基础入门阶段，有4台VPS，已完成 Python 运维脚本教程、Docker 部署经验（Nginx Proxy Manager / Uptime Kuma / Portainer / gh-proxy）

---

## 📅 时间线总览

| 阶段 | 时间 | 目标 |
|------|------|------|
| **第一阶段：补齐核心基础** | 3月中 → 4月中 | Linux + 网络 + Shell 扎实 |
| **第二阶段：工具链实战** | 4月中 → 5月中 | Docker/CI-CD/监控/自动化 |
| **第三阶段：项目实战 + 简历** | 5月中 → 6月中 | 打造2-3个可展示的项目 |
| **第四阶段：投递 + 面试** | 6月中 → 7月 | 刷面经、投递、模拟面试 |

---

## 🔧 第一阶段：补齐核心基础（3月中 → 4月中）

### 1. Linux 系统管理 ⭐ 最重要

> [!IMPORTANT]
> Linux 是运维的根基，面试必考。建议每天 1-2 小时在 VPS 上实操。

**必须掌握**（在你的 VPS 上练）：
- 用户/权限管理：`useradd`, `chmod`, `chown`, `sudo`, `/etc/sudoers`
- 进程管理：`ps aux`, `top`/`htop`, `kill`, `systemctl`, `journalctl`
- 磁盘/文件系统：`df`, `du`, `fdisk`, `mount`, `lsblk`, `ln`
- 网络排查：`ss`, `netstat`, `ip addr`, `iptables`/`nftables`, `tcpdump`, `traceroute`, `dig`, `curl`
- 包管理：`apt`/`yum`/`dnf`, 配置国内源
- 文本处理三剑客：`grep`, `sed`, `awk`（面试高频）
- 日志排查：`tail -f`, `less`, `journalctl -u xxx`

**推荐资源**：
- 《鸟哥的Linux私房菜》基础篇（选读，当字典用）
- [Linux Journey](https://linuxjourney.com/)（英文，免费在线教程）
- B站搜"Linux运维"系统课（韩顺平、老男孩等）

**你的 VPS 练习任务**：
```
✅ 在上海/广州 VPS 上从零配置一台 Web 服务器（手动安装 Nginx，不用 Docker）
✅ 配置 SSH 密钥登录，禁用密码登录
✅ 配置 firewall/iptables 只开放 22/80/443
✅ 设置定时任务 crontab 做日志清理
✅ 写一个 Systemd Service 管理你的自定义脚本
```

### 2. 计算机网络（面试必考）

**必须掌握**：
- TCP/IP 四层模型 + OSI 七层（各层协议、功能）
- TCP 三次握手 / 四次挥手（画图能讲清楚）
- HTTP/HTTPS 原理、状态码含义、常见 Header
- DNS 解析过程
- NAT、VLAN、子网划分基础
- 负载均衡概念（四层 vs 七层）

**推荐资源**：
- 小林coding 的[图解网络](https://xiaolincoding.com/network/)（免费，面试向，中文）
- B站"计算机网络微课堂"湖科大教书匠

### 3. Shell 脚本（面试高频）

**在 VPS 上练这些**：
```bash
# 写脚本实现：
✅ 自动备份指定目录到 /backup/，按日期命名，保留最近7天
✅ 监控 CPU/内存/磁盘使用率，超阈值发告警（结合你之前的 Python 脚本）
✅ 批量添加用户脚本
✅ 日志分析脚本：统计 Nginx 访问日志 Top 10 IP、状态码分布
✅ 服务健康检查脚本（检测端口/URL 是否存活）
```

---

## 🐳 第二阶段：工具链实战（4月中 → 5月中）

### 1. Docker（你已有基础，进阶）

**进阶掌握**：
- Dockerfile 编写（多阶段构建）
- Docker Compose 编排多容器应用
- Docker 网络模式（bridge/host/none/overlay）
- 数据卷管理、日志管理
- 镜像优化、安全最佳实践

**VPS 练习任务**：
```
✅ 手写 Dockerfile 将你的 Python 运维工具打包成镜像
✅ 用 Docker Compose 搭建 Nginx + MySQL + WordPress 完整环境
✅ 用 Docker Compose 搭建 Prometheus + Grafana 监控栈
```

### 2. CI/CD 基础

**必须掌握**：
- Git 工作流（feature branch、merge、rebase）
- GitHub Actions 基础（写一个简单的 CI Pipeline）
- 了解 Jenkins 概念

**VPS 练习任务**：
```
✅ 在 GitHub 上为你的 python-ops-tutorial 项目配置 CI
   （自动跑 linting / 简单测试）
✅ 配置一个自动部署 Pipeline：push 到 main → SSH 到 VPS 自动拉取更新
```

### 3. 监控与告警

**你已经有 Uptime Kuma 了，继续深入**：
- Prometheus + Grafana（核心监控栈）
- node_exporter 采集服务器指标
- 配置告警规则 + 通知（微信/钉钉/邮件）

**VPS 练习任务**：
```
✅ 在雅加达 VPS 上部署 Prometheus + Grafana + node_exporter
✅ 监控你其他3台 VPS 的 CPU/内存/磁盘/网络
✅ 配置 Grafana Dashboard 可视化
✅ 设置告警规则（磁盘 > 80% 发通知）
```

### 4. Nginx 深入

**必须掌握**：
- 反向代理配置
- 负载均衡配置（upstream）
- HTTPS / SSL 证书配置（Let's Encrypt / acme.sh）
- 常见性能优化参数
- 日志格式自定义与分析

---

## 🏗️ 第三阶段：项目实战 + 简历（5月中 → 6月中）

> [!IMPORTANT]
> 这是最关键的阶段！你需要 2-3 个能写进简历的项目，放到 GitHub 上。

### 推荐项目方案（利用你现有的 4 台 VPS）

#### 项目一：多节点服务器监控平台
```
目标：用 Prometheus + Grafana + Alertmanager 构建一个监控你 4 台 VPS 的平台
技术栈：Docker Compose / Prometheus / Grafana / node_exporter / Alertmanager
亮点：
  - 多地域（北京/上海/广州/雅加达）节点监控
  - 自定义 Grafana Dashboard
  - 告警通知集成
  - 写一份详细的部署文档（README）
```

#### 项目二：自动化运维工具集（基于你现有的 Python 脚本，扩展升级）
```
目标：将之前的 Python 运维脚本升级为一个完整的 CLI 工具 + Web 界面
技术栈：Python / Flask (简单API) / Shell / Crontab / Systemd
亮点：
  - 系统巡检报告自动生成
  - 日志分析与告警
  - API 接口暴露监控数据
  - GitHub Actions CI/CD
```

#### 项目三：高可用 Web 服务部署
```
目标：在 2 台 VPS 上配 Nginx 负载均衡 + 后端应用 + 自动部署
技术栈：Nginx / Docker / GitHub Actions / Shell
亮点：
  - Nginx 反向代理 + 负载均衡
  - 自动化部署脚本（git push → 自动上线）
  - HTTPS 证书自动续期
  - Ansible 管理多台机器（加分项）
```

### 简历要点

```
必备信息：
✅ 技术技能：Linux / Shell / Python / Docker / Nginx / 监控 / CI-CD / Git
✅ 项目经历：2-3 个上面的项目，写清楚 "做了什么 → 解决什么问题 → 成果"
✅ GitHub 地址：保持活跃（绿点很重要）
✅ 写上你有 4 台 VPS 的实操经验，面试官会加分

简历格式建议：
- 控制在 1 页
- 用 STAR 法则写项目（情景-任务-行动-结果）
- 可以用超级简历/木及简历等在线工具
```

---

## 📝 第四阶段：投递 + 面试准备（6月中 → 7月）

### 面试高频题准备

#### Linux 方向
- 进程、线程、协程的区别？
- `kill -9` 和 `kill -15` 的区别？
- 如何排查服务器负载高的问题？（top → 看 CPU/IO → 找进程 → strace/perf）
- 如何排查网络不通？（ping → traceroute → telnet → tcpdump）
- 文件权限 755 是什么意思？
- 软链接和硬链接的区别？
- `/proc` 文件系统是什么？

#### 网络方向
- TCP 三次握手四次挥手（必会画图）
- TIME_WAIT 是什么？太多 TIME_WAIT 怎么处理？
- HTTP 和 HTTPS 的区别？HTTPS 握手过程？
- 浏览器输入 URL 到页面显示的完整过程？
- GET 和 POST 的区别？

#### Docker 方向
- 容器和虚拟机的区别？
- Dockerfile 常用指令？
- Docker 网络模型？
- 怎么减小镜像体积？

#### 场景题
- 服务器被入侵了怎么排查？
- 磁盘满了怎么排查和处理？
- 某个服务挂了怎么快速恢复？

### 投递渠道

| 渠道 | 说明 |
|------|------|
| **Boss直聘** | 运维实习最多的平台，重点投 |
| **实习僧** | 实习专门平台 |
| **牛客网** | 看内推信息 + 面经 |
| **公司官网** | 大厂暑期实习招聘页面（腾讯/阿里/字节/百度/华为等 3-4 月就开始了） |
| **微信公众号** | 关注"运维派"、"Linux中国"等获取招聘信息 |

> [!CAUTION]
> ⏰ 大厂暑期实习一般 **3-4月** 就开始招了！建议第一阶段学习的同时就开始关注和投递，边学边面，不要等到"准备好了"再投，你永远不会觉得"准备好了"。

---

## 📚 每日学习计划建议

```
工作日（约 3-4 小时）：
  上午/课余：1h 看教程/文档（网络/Linux 理论）
  晚上：2-3h VPS 实操 + 编码练习

周末（约 5-6 小时）：
  上午：2h 理论学习
  下午：3-4h 项目实战
```

---

## 🎯 你现有资源的最佳利用方案

> [!TIP]
> 你同时使用了 **百度智能云 + 华为云 + 火山引擎** 三家云平台，这本身就是简历亮点——"具备多云平台实操经验"。

| VPS | 云平台 | 配置 | 系统 | 建议用途 |
|-----|--------|------|------|---------|
| 🏠 北京 | 百度智能云 | 2H2G / 1M | Debian | 已有服务 + 已备案域名，维持现有服务，加装 node_exporter 被监控 |
| 🚀 上海 | 火山引擎 | 2vCPU 4G / 5M | Rocky Linux 9.5 | **主力机**：部署 Prometheus + Grafana 监控中心 + 主要练习场 |
| 🔧 广州 | 华为云 | 2核 2G / 3M / 系统盘40G / 流量包400G | EulerOS 2.0 | 练习机：手动部署 Nginx/MySQL、Shell 脚本测试场 |
| 🌏 雅加达 | 火山引擎 | 2vCPU 2G / 5M | Rocky Linux 9.5 | 海外节点：node_exporter + 跨地域延迟监控练习 |

**域名资源**：
- ✅ 1 个已备案域名（绑定在北京百度云服务器上）
- 📝 2~3 个未备案域名 → 可绑到海外雅加达 VPS（海外服务器不需要备案）

**多系统练习价值**：
- **Debian**（北京百度云）→ Debian 系，学 `apt`/`ufw`/`dpkg`，社区生态最广
- **Rocky Linux 9.5**（上海/雅加达）→ RHEL 系，学 `dnf`/`yum`/`firewalld`/`SELinux`
- **EulerOS 2.0**（广州华为云）→ 华为自研发行版，RHEL 系，熟悉国产化生态（面试加分）
- 同时掌握 Debian 系 + RHEL 系两大阵营，面试时体现适应能力

---

## ✅ 立即行动清单（本周就开始）

1. [ ] 在上海 VPS 上关掉不必要的服务，开始从零手动配置
2. [ ] 开始看"小林coding 图解网络" TCP 部分
3. [ ] 写第一个 Shell 脚本：自动备份 + 日志清理
4. [ ] 注册 Boss直聘 / 实习僧，搜索"运维实习"，看看 JD 里要求什么技能
5. [ ] 关注大厂暑期实习招聘信息（很多 3月就截止了！）
6. [ ] 每天在 GitHub 上有提交记录

> [!TIP]
> 别追求完美再开始投简历。先投几个中小公司练手面试，积累经验后再冲大厂。面试本身就是最好的学习驱动力！
