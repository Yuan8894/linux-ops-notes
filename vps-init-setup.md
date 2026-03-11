# 🔒 VPS 初始化安全加固手册

> 适用系统：RHEL 系（Rocky Linux / EulerOS / CentOS）  
> 最后更新：2026-03-11  
> 已在 3 台 VPS（上海火山引擎 / 广州华为云 / 雅加达火山引擎）上执行完毕

---

## 1. 系统更新

```bash
dnf update -y
```

| 参数 | 含义 |
|------|------|
| `update` | 更新所有已安装的软件包到最新版本 |
| `-y` | 自动回答 yes，跳过确认提示 |

---

## 2. 创建普通用户

```bash
adduser hw_yyy
passwd hw_yyy
```

| 命令 | 含义 |
|------|------|
| `adduser <用户名>` | 创建新用户，自动创建同名家目录 `/home/hw_yyy` |
| `passwd <用户名>` | 为指定用户设置密码（虽然后面禁用了密码登录，但 `sudo` 等场景仍需要） |

### 赋予 sudo 权限

```bash
usermod -aG wheel hw_yyy
```

| 参数 | 含义 |
|------|------|
| `-a` | append，追加到组（不移除用户已有的其他组） |
| `-G wheel` | 将用户加入 `wheel` 组，RHEL 系中 wheel 组默认拥有 sudo 权限 |

---

## 3. 配置 SSH 密钥登录（ed25519）

### 3.1 本地生成密钥对（在自己电脑上执行）

```bash
ssh-keygen -t ed25519 -C "your_email@example.com"
```

| 参数 | 含义 |
|------|------|
| `-t ed25519` | 指定密钥类型为 Ed25519（比 RSA 更安全、密钥更短、性能更好） |
| `-C "..."` | 添加注释，通常填邮箱，方便识别密钥归属 |

生成后会得到两个文件：
- `~/.ssh/id_ed25519` — 私钥（**绝对不能泄露**）
- `~/.ssh/id_ed25519.pub` — 公钥（放到服务器上）

### 3.2 在服务器上部署公钥

```bash
# 安装编辑器（EulerOS 默认可能没有 nano）
dnf install nano -y

# 先在 root 下写入公钥
nano ~/.ssh/authorized_keys
# 将本地 id_ed25519.pub 的内容粘贴进去，保存退出

# 为新用户创建 .ssh 目录
mkdir -p /home/hw_yyy/.ssh

# 复制公钥到新用户目录
cp ~/.ssh/authorized_keys /home/hw_yyy/.ssh/

# 修改目录所有权为新用户
chown -R hw_yyy:hw_yyy /home/hw_yyy/.ssh
```

| 命令/参数 | 含义 |
|-----------|------|
| `mkdir -p` | 创建目录，`-p` 表示递归创建（父目录不存在时自动创建，且目录已存在也不报错） |
| `cp` | 复制文件 |
| `chown -R hw_yyy:hw_yyy` | 递归修改所有者。格式为 `用户:组`，`-R` 表示递归处理子目录和文件 |

### 3.3 设置正确的权限

```bash
chmod 700 /home/hw_yyy/.ssh/
chmod 600 /home/hw_yyy/.ssh/authorized_keys
```

| 权限值 | 含义 | 为什么 |
|--------|------|--------|
| `700` | 所有者可读/写/执行，其他人无任何权限 | `.ssh` 目录必须只有所有者能访问，否则 SSH 会拒绝使用 |
| `600` | 所有者可读/写，其他人无任何权限 | `authorized_keys` 只能所有者读写，权限不对会导致密钥登录失败 |

> **权限数字速查**：`r=4, w=2, x=1`，三位分别代表 `所有者/同组/其他人`  
> 例：`750` = 所有者(rwx=7) + 同组(r-x=5) + 其他(---=0)

### 3.4 验证目录权限

```bash
ls -ld /home/hw_yyy/
# 预期输出：drwx------ 3 hw_yyy hw_yyy 4096 Mar 11 19:35 /home/hw_yyy/
```

| 参数 | 含义 |
|------|------|
| `-l` | 长格式显示（显示权限、所有者、大小等详细信息） |
| `-d` | 只显示目录本身的信息，而不是目录内的文件列表 |

---

## 4. 加固 SSH 配置

```bash
nano /etc/ssh/sshd_config
```

修改以下三项：

```sshd_config
PermitRootLogin no              # 禁止 root 直接 SSH 登录
PasswordAuthentication no       # 禁止密码登录，只允许密钥
PubkeyAuthentication yes        # 启用公钥认证（通常默认就是 yes）
```

| 配置项 | 说明 |
|--------|------|
| `PermitRootLogin no` | 攻击者最常暴力破解的就是 root，关闭后必须通过普通用户登录再 `sudo` 提权 |
| `PasswordAuthentication no` | 密码可以被暴力破解，密钥几乎不可能。关闭密码登录是最基本的安全措施 |
| `PubkeyAuthentication yes` | 确保公钥认证开启，配合上一项使密钥成为唯一登录方式 |

修改后重启 SSH 服务：

```bash
systemctl restart sshd
```

| 命令 | 含义 |
|------|------|
| `systemctl` | systemd 服务管理器，用于管理系统服务 |
| `restart` | 重启服务（先 stop 再 start），使配置生效 |
| `sshd` | SSH 守护进程服务名 |

> ⚠️ **重要提醒**：修改 `sshd_config` 后，先**不要关闭当前 SSH 会话**！  
> 新开一个终端窗口测试能否用密钥 + 新用户正常登录，确认没问题后再关闭旧会话。  
> 否则如果配置有误，你会把自己锁在服务器外面。

---

## 5. 防火墙策略

本次选择使用**云平台安全组**代替本机防火墙，理由：

- 云安全组在网络入口层过滤，比本机防火墙更早拦截流量
- 管理方便，在云控制台 Web 界面统一操作
- 实际生产环境通常 **安全组 + 本机防火墙** 双层防护

当前安全组开放端口（按需调整）：

| 端口 | 用途 |
|------|------|
| 22 | SSH 远程连接 |
| 80 | HTTP |
| 443 | HTTPS |

> 后续如需练习本机防火墙，可在广州 EulerOS 上用 `firewalld`：
> ```bash
> systemctl start firewalld
> firewall-cmd --permanent --add-service=ssh
> firewall-cmd --permanent --add-service=http
> firewall-cmd --reload
> firewall-cmd --list-all
> ```

---

## 📋 操作检查清单

- [x] 系统更新 `dnf update -y`
- [x] 创建普通用户
- [x] 生成 ed25519 密钥对
- [x] 部署公钥到服务器
- [x] 设置 .ssh 目录权限 700
- [x] 设置 authorized_keys 权限 600
- [x] 禁止 root SSH 登录
- [x] 禁止密码登录
- [x] 启用公钥认证
- [x] 重启 sshd 服务
- [x] 云安全组配置
- [ ] （可选）配置 fail2ban 防暴力破解
- [ ] （可选）修改 SSH 默认端口 22 → 其他端口
