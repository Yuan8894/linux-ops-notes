# RedHat系与Debian系防火墙运维学习笔记 (包含 UFW)

## 1. 核心差异概述
- **RedHat 系**（CentOS, RHEL, Fedora, AlmaLinux, Rocky Linux 等）：
  - 默认使用的是 **firewalld**。
  - 采用 **Zone（区域）** 概念进行网络隔离管理，适合处理多网卡或复杂的网络环境。
  - 支持动态刷新（在不丢失现有连接的情况下更新规则）。
- **Debian 系**（Debian, Ubuntu, Linux Mint 等）：
  - 默认使用 **UFW (Uncomplicated Firewall)**（在 Ubuntu 尤为常见，Debian 也可快速安装）或原生的 **iptables/nftables**。
  - 设计初衷是为相对复杂的 iptables 提供一个极度易于理解的命令行界面（CLI）。
- **底层技术**：
  - 两者的底层往往都在向 **nftables** 演进（过去是 iptables）。无论是 firewalld 还是 ufw，其实本质上都只是包过滤内核子系统（Netfilter）的“配置前端”。

---

## 2. RedHat 系：Firewalld 运维指南

### 2.1 基础概念
- **Zone（区域）**：定义网络连接的信任级别（如 `public`, `internal`, `trusted`, `drop` 等）。默认的流量通常都归在 `public` 区域内。
- **Service（服务）**：可以通过常见服务名称（如 `http`, `ssh`, `https`）直接开放，而无需记住它所用的具体端口。

### 2.2 常用命令 (firewall-cmd)

**状态与基础管理：**
```bash
systemctl status firewalld   # 查看防火墙系统服务运行状态
systemctl start firewalld    # 启动防火墙服务
systemctl enable firewalld   # 设置防火墙开机自启

firewall-cmd --state         # 查看 firewalld 自身的状态（返回 running 或 not running）
firewall-cmd --reload        # 重新加载配置（动态应用规则，且不中断现有连接。改动后必须执行！）
```

**查看当前生效规则：**
```bash
firewall-cmd --get-active-zones    # 查看当前正在激活动作的区域
firewall-cmd --list-all            # 查看默认区域（通常是 public）的所有规则
firewall-cmd --list-ports          # 查看当前开放的端口
firewall-cmd --list-services       # 查看当前开放的服务
```

**放行端口与服务（⚠️关键点：必须加 `--permanent` 才能永久生效）：**
```bash
# 放行 TCP 80 端口（永久生效）
firewall-cmd --zone=public --add-port=80/tcp --permanent

# 移除已开放的 TCP 80 端口
firewall-cmd --zone=public --remove-port=80/tcp --permanent

# 放行 http 服务（默认对应端口 80）
firewall-cmd --zone=public --add-service=http --permanent

# 移除 http 服务
firewall-cmd --zone=public --remove-service=http --permanent

# ⚠️ 重载生效（只要使用了 --permanent 添加或删除规则，必须执行此命令才能立即体现在当前运行中）
firewall-cmd --reload
```

**富规则（Rich Rules，精细化控制）：**
```bash
# 允许特定 IP (192.168.1.100) 访问本机特定的端口 (3306)
firewall-cmd --permanent --add-rich-rule="rule family='ipv4' source address='192.168.1.100' port protocol='tcp' port='3306' accept"

# 拒绝特定 IP 访问本机的任何信息
firewall-cmd --permanent --add-rich-rule="rule family='ipv4' source address='10.0.0.5' reject"

firewall-cmd --reload
```

---

## 3. Debian 系：UFW (Uncomplicated Firewall) 运维指南

### 3.1 基础概念
UFW (Uncomplicated Firewall) 旨在使防火墙配置变得简单，非常适合绝大多数常规的单机防护需求，它是 Ubuntu 系统预装并推荐使用的主流前端工具。默认的初始策略是：**拒绝所有传入连接（incoming），允许所有传出连接（outgoing）**。

### 3.2 常用命令 (ufw)

**状态与基础管理：**
```bash
ufw status          # 查看状态（激活会显示 active 并列出规则，未激活显示 inactive）
ufw status verbose  # 查看详细状态（包括日志级别、默认策略等）
ufw status numbered # 按编号列出所有规则（这在需要精确删除某条规则时非常有用）

ufw enable          # 启动防火墙并设置开机自启（执行时可能会提示会阻断现有 SSH，输入 y 确认）
ufw disable         # 停止防火墙并禁用开机自启
ufw reload          # 重新加载配置文件（通常配置后立刻生效，手动更改配置文件时才需此命令）
```

**放行端口与服务：**
```bash
# 允许特定端口（如果不指定协议，默认 TCP 和 UDP 都允许）
ufw allow 80

# 明确指定仅允许 TCP 或 UDP 端口
ufw allow 22/tcp
ufw allow 53/udp

# 允许某个服务名称（如 ssh, http，依据 /etc/services ）
ufw allow ssh

# 拒绝（Deny）特定端口的访问
ufw deny 23/tcp
```

**删除规则：**
```bash
# 方法一：写出与添加时完全一样的规则结构前加 delete
ufw delete allow 80/tcp

# 方法二（推荐）：按编号删除
# 1. 先查看编号：ufw status numbered
# 2. 删除对应编号的规则：
ufw delete 1
```

**精细化控制（限制 IP 源和防爆破）：**
```bash
# 允许从特定 IP 访问本机的任意端口
ufw allow from 192.168.1.100

# 允许特定 IP 或者 IP 段 访问本机的指定端口 (如 MySQL 的 3306)
ufw allow from 192.168.1.0/24 to any port 3306

# 拒绝特定 IP 的所有访问
ufw deny from 10.0.0.5

# 防护暴力破解（速率限制：如果在一个IP上发生6次连接失败，则会在30秒内被系统拒绝，常用于防 SSH 爆破）
ufw limit ssh
# 或者
ufw limit 22/tcp
```

---

## 4. 两者的对比总结与实战避坑建议

| 对比维度 | RedHat 系 (`firewalld`) | Debian 系 (`UFW`) |
| :--- | :--- | :--- |
| **设计理念** | 强大、动态重载、基于区域 (Zone) 隔离 | 极简、易上手、作为 iptables 的平替前端 |
| **生效方式** | `--permanent` 写入配置，需配合 `--reload` 生效 | 输入配置命令后实时生效，自动存入配置，极少需要重载 |
| **重载命令** | `firewall-cmd --reload` (非常频繁) | `ufw reload` (很少需要手动执行) |
| **查看规则** | `firewall-cmd --list-all` | `ufw status numbered` |
| **适用场景** | 多网卡、复杂网络环境组网划分、企业级服务器 | 单网卡、个人VPS建站、快速配置基础 Web 应用环境 |

### 🔥 实战避坑建议：
1. **防止自己被关在门外（SSH 斩首）**：开启任何防火墙工具前（即执行 `ufw enable` 或启动 `firewalld` 服务**之前**），请**务必确保已放行 SSH 端口**（通常是 `22/tcp`，如果你修改了 SSH 端口，请放行对应自定义的那个端口），否则敲下回车的瞬间你就会与 VPS 永久失联！
2. **Docker 与防火墙的“相爱相杀”！注意！**：
   - Docker 服务进程默认会直接操纵底层的 `iptables` 来配置容器的端口映射，**这种行为会直接绕过 UFW 和 firewalld 设定的拦截规则**！
   - 问题案例：即使你用 UFW 设置了 `ufw deny 8080`，但你的 Docker 容器刚好发布到了 `0.0.0.0:8080` 暴露给网络，此时外网依然可以直接访问防不住！
   - **解决思路**：在复杂场景下，不要指望主机防火墙能有效掐段 Docker 的向外映射。建议在运行 Docker 时将端口限制死在本机内网IP（例如 `docker run -p 127.0.0.1:8080:80...`），然后再使用装在宿主机的 Nginx 代理出去。也可以修改 Docker daemon 配置文件使其不操作 iptables，但这往往会导致容器内连不上外网等衍生问题，建议慎重。
3. **混合使用的禁忌**：永远不要在同一个 Linux 服务器上同时运行 `ufw` 和 `firewalld` 等多个前端工具，这会产生严重的规则冲突和不可预知的网络行为。遵循发行版“入乡随俗”，用系统自带推荐的那个即可。
