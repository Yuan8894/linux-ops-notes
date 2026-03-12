# 第一阶段：Linux 系统管理核心教程

> **适用环境**：Rocky Linux 9.5（上海/雅加达）、EulerOS 2.0（广州）、Debian（北京）
> **学习周期**：3月中 → 4月中（约 4 周）
> **前置条件**：已完成 VPS 初始化安全加固（见 `vps-init-setup.md`）

---

## 目录

1. [用户与权限管理](#1-用户与权限管理)
2. [进程管理](#2-进程管理)
3. [磁盘与文件系统](#3-磁盘与文件系统)
4. [网络排查](#4-网络排查)
5. [包管理](#5-包管理)
6. [文本处理三剑客](#6-文本处理三剑客)
7. [日志排查](#7-日志排查)
8. [综合练习项目](#8-综合练习项目)

---

## 1. 用户与权限管理

### 1.1 用户管理

#### 创建与删除用户

```bash
# 创建用户（RHEL 系 / Debian 通用）
useradd alice                         # 创建用户，不自动创建家目录（RHEL 行为）
useradd -m -s /bin/bash alice         # 创建用户 + 家目录 + 指定 shell（推荐）
adduser alice                         # Debian 系推荐，交互式创建，自动建家目录

# 设置密码
passwd alice

# 修改用户信息
usermod -s /bin/zsh alice             # 修改默认 shell
usermod -aG wheel alice              # RHEL 系：加入 wheel 组（获得 sudo）
usermod -aG sudo alice               # Debian 系：加入 sudo 组（获得 sudo）
usermod -L alice                     # 锁定账户（禁止登录）
usermod -U alice                     # 解锁账户

# 删除用户
userdel alice                        # 删除用户，保留家目录
userdel -r alice                     # 删除用户 + 家目录 + 邮件目录
```

| 参数 | 含义 |
|------|------|
| `-m` | 创建家目录 `/home/<用户名>` |
| `-s` | 指定登录 shell |
| `-aG` | append + Group，追加到指定组 |
| `-L` / `-U` | Lock / Unlock |

#### 查看用户信息

```bash
id alice                             # 查看 uid、gid 及所属组
whoami                               # 查看当前用户名
who                                  # 查看当前登录用户列表
w                                    # 查看登录用户 + 正在执行的命令
last                                 # 查看登录历史
cat /etc/passwd                      # 用户账户数据库（格式：用户名:x:uid:gid:备注:家目录:shell）
cat /etc/group                       # 组数据库
getent passwd alice                  # 查询指定用户（比 cat 更规范）
```

#### /etc/passwd 字段说明

```
alice:x:1001:1001:Alice:/home/alice:/bin/bash
  |   |  |    |     |        |          |
用户名 密码占位 uid  gid    备注      家目录      shell
```

---

### 1.2 权限管理

#### 文件权限数字速查

```
r = 4（读）    w = 2（写）    x = 1（执行）
                            所有者  同组  其他人
chmod 755 file.sh    →       rwx    r-x   r-x
chmod 644 file.txt   →       rw-    r--   r--
chmod 700 ~/.ssh/    →       rwx    ---   ---
chmod 600 ~/.ssh/authorized_keys  →  rw-  ---  ---
```

#### chmod 常用操作

```bash
# 数字方式
chmod 755 script.sh              # 所有者 rwx，其他人 r-x
chmod -R 755 /var/www/html       # 递归设置目录

# 符号方式（更直观）
chmod u+x script.sh              # 给所有者加执行权限
chmod o-w file.txt               # 去掉其他人的写权限
chmod g=r config.conf            # 组权限精确设为只读
chmod a+r README.md              # 所有人加读权限（a = all）
```

#### chown 修改所有者

```bash
chown alice file.txt             # 改变文件所有者
chown alice:devops file.txt      # 改变所有者 + 组
chown -R alice:alice /home/alice # 递归改变目录
chown :devops file.txt           # 只改变组（所有者不变）
```

#### 特殊权限

```bash
# SUID（Set User ID）：执行时以文件所有者身份运行
chmod u+s /usr/bin/passwd        # passwd 命令用此原理让普通用户修改密码
ls -l /usr/bin/passwd            # 看到 -rwsr-xr-x，s 表示 SUID

# Sticky Bit：只有文件所有者才能删除目录内的文件
chmod +t /tmp                    # /tmp 目录默认启用
ls -ld /tmp                      # 看到 drwxrwxrwt，末尾 t 表示 sticky bit
```

---

### 1.3 sudo 与 /etc/sudoers

```bash
# 查看当前用户 sudo 权限
sudo -l

# 以其他用户身份执行命令
sudo -u alice ls /home/alice

# 编辑 sudoers（必须用 visudo，会检查语法）
sudo visudo
```

#### /etc/sudoers 常用配置

```
# 格式：用户 主机=(运行身份) 命令
alice  ALL=(ALL)  ALL              # alice 可以 sudo 任何命令
alice  ALL=(ALL)  NOPASSWD: ALL    # sudo 不需要输密码
%wheel ALL=(ALL)  ALL              # wheel 组所有成员可 sudo（RHEL 系默认）
%sudo  ALL=(ALL)  ALL              # sudo 组所有成员可 sudo（Debian 系默认）

# 限制只能执行特定命令
alice  ALL=(root)  /usr/bin/systemctl restart nginx
```

> **安全提醒**：生产环境不要用 NOPASSWD，测试机可以用以省事。

---

### 练习 1：用户与权限

```bash
# 在广州 EulerOS 上练习：
# 1. 创建两个用户 devuser 和 webuser
# 2. 为 devuser 配置 sudo 权限
# 3. 创建 /data/webapp 目录，让 webuser 拥有写权限，devuser 只读
# 4. 创建一个脚本文件，设置只有所有者能执行
# 5. 验证权限设置是否生效

# 参考命令：
useradd -m devuser
useradd -m webuser
usermod -aG wheel devuser
mkdir -p /data/webapp
chown webuser:webuser /data/webapp
chmod 755 /data/webapp            # devuser 能进入但不能写
# 如果需要 devuser 也能写，考虑用组或 ACL
```

---

## 2. 进程管理

### 2.1 查看进程

#### ps 命令

```bash
ps aux                           # 查看所有进程（最常用）
ps aux | grep nginx              # 过滤查看 nginx 进程
ps -ef                           # 另一种格式（显示 PPID 父进程 ID）
ps -p 1234                       # 查看指定 PID 的进程
pstree                           # 树形显示进程关系
```

**ps aux 字段说明**：
```
USER   PID  %CPU  %MEM    VSZ   RSS  TTY  STAT  START    TIME  COMMAND
root     1   0.0   0.1  171484  5332  ?    Ss   10:00   0:01  /usr/lib/systemd/systemd
  |       |    |     |     |     |    |     |     |       |        |
用户  进程ID  CPU% 内存%  虚拟内存 物理内存 终端 状态 启动时间 CPU时间  命令
```

**STAT 进程状态**：
| 状态 | 含义 |
|------|------|
| `R` | Running，正在运行 |
| `S` | Sleeping，可中断休眠（最常见） |
| `D` | Uninterruptible sleep，等待 IO（不可被杀死） |
| `Z` | Zombie，僵尸进程（已结束但父进程未回收） |
| `T` | Stopped，暂停 |
| `s` | Session leader（小写） |
| `+` | 前台进程 |

#### top / htop

```bash
top                              # 实时进程监控
# top 内快捷键：
# P → 按 CPU 排序
# M → 按内存排序
# k → 输入 PID 杀进程
# q → 退出
# 1 → 展开显示每个 CPU 核心

htop                             # 彩色交互式版本（需安装：dnf install htop）
```

**top 界面解读**：
```
top - 10:30:01 up 5 days,  2:10,  1 user,  load average: 0.12, 0.08, 0.05
Tasks: 145 total,   1 running, 144 sleeping,   0 stopped,   0 zombie
%Cpu(s):  2.3 us,  0.5 sy,  0.0 ni, 96.8 id,  0.3 wa,  0.0 hi,  0.1 si
MiB Mem :   3733.9 total,   1200.3 free,    800.5 used,   1733.1 buff/cache
MiB Swap:   2048.0 total,   2048.0 free,      0.0 used.   2500.0 avail Mem
```

| 字段 | 含义 |
|------|------|
| `load average` | 1分钟/5分钟/15分钟平均负载（数值 ≤ CPU核数为正常） |
| `us` | user，用户态 CPU 占用 |
| `sy` | system，内核态 CPU 占用 |
| `wa` | wait，等待 IO 的时间（高 = IO 瓶颈） |
| `id` | idle，空闲 CPU |
| `buff/cache` | 被用作缓冲/缓存的内存（可被回收） |

---

### 2.2 信号与 kill

```bash
kill -l                          # 列出所有信号
kill 1234                        # 默认发 SIGTERM(15)，优雅终止
kill -15 1234                    # 同上，程序有机会做清理工作
kill -9 1234                     # SIGKILL，强制杀死（无法被捕获/忽略）
kill -HUP 1234                   # SIGHUP(1)，通常用于让程序重载配置
killall nginx                    # 按进程名杀所有 nginx 进程
pkill -f "python script.py"      # 按完整命令行匹配杀进程
```

> **面试考点**：`kill -9` vs `kill -15`
> - `-15` (SIGTERM)：发送终止信号，程序可以捕获、执行清理（关闭连接、写完日志）后退出。**优先使用**。
> - `-9` (SIGKILL)：内核直接强制终止，程序无法拦截。可能导致数据损坏。只在 `-15` 无效时使用。

---

### 2.3 systemctl 服务管理

```bash
# 服务状态操作
systemctl start nginx            # 启动
systemctl stop nginx             # 停止
systemctl restart nginx          # 重启（先 stop 再 start）
systemctl reload nginx           # 重载配置（不重启进程，Nginx 支持）
systemctl status nginx           # 查看状态 + 最近日志

# 开机自启
systemctl enable nginx           # 设置开机自启
systemctl disable nginx          # 取消开机自启
systemctl enable --now nginx     # 设置开机自启 + 立即启动（常用）
systemctl is-enabled nginx       # 查询是否开机自启

# 查看服务列表
systemctl list-units --type=service          # 所有运行中的 service
systemctl list-units --type=service --all    # 包括未运行的
systemctl list-unit-files --type=service     # 查看所有 service 文件及启用状态
```

---

### 2.4 journalctl 日志查看

```bash
journalctl -u nginx                          # 查看 nginx 服务日志
journalctl -u nginx --since "1 hour ago"     # 最近 1 小时
journalctl -u nginx -n 50                    # 最新 50 行
journalctl -u nginx -f                       # 实时跟踪（tail -f 效果）
journalctl -p err                            # 只看错误级别日志（err/warning/info/debug）
journalctl --since "2026-03-01" --until "2026-03-11"  # 时间范围
journalctl -b                                # 本次启动以来的日志
journalctl -b -1                             # 上次启动的日志
journalctl --disk-usage                      # 查看日志占用磁盘
```

---

### 练习 2：进程管理

```bash
# 在上海 Rocky Linux 上练习：

# 1. 安装并启动 nginx，查看其进程信息
dnf install nginx -y
systemctl enable --now nginx
ps aux | grep nginx
systemctl status nginx

# 2. 故意制造高负载，观察 top
# 安装 stress 工具
dnf install stress -y
stress --cpu 2 --timeout 30 &     # 后台运行 30 秒 CPU 压力测试
top                                # 观察 load average 变化

# 3. 练习 kill
# 先找到 nginx worker 进程 PID
ps aux | grep "nginx: worker"
# 发送 SIGHUP 让 nginx 重载配置
sudo kill -HUP $(cat /run/nginx.pid)

# 4. 写一个永远不退出的脚本，然后用不同方式结束它
cat > /tmp/test_loop.sh << 'EOF'
#!/bin/bash
trap "echo 'Received SIGTERM, cleaning up...'; exit 0" SIGTERM
while true; do
    echo "Running... PID=$$"
    sleep 2
done
EOF
chmod +x /tmp/test_loop.sh
bash /tmp/test_loop.sh &          # 后台运行
echo "PID: $!"
kill -15 $!                       # 优雅终止，观察 trap 输出
```

---

## 3. 磁盘与文件系统

### 3.1 查看磁盘使用情况

```bash
df -h                             # 查看文件系统挂载点和使用率（-h 人类可读）
df -hT                            # 同上 + 显示文件系统类型
df -i                             # 查看 inode 使用情况（文件数量限制）

du -sh /var/log                   # 查看 /var/log 目录总大小
du -sh /*                         # 查看根目录各子目录大小
du -sh /var/log/* | sort -rh      # 按大小排序，找最大的
du -sh /var/log/* | sort -rh | head -10  # 只看前 10 个
```

> **排查磁盘满了（面试场景题）**：
> ```bash
> df -h                           # 第一步：找哪个分区满了
> du -sh /* 2>/dev/null | sort -rh | head    # 第二步：找根目录下最大的目录
> du -sh /var/* | sort -rh | head            # 第三步：进入大目录继续缩小范围
> # 找到后：删除旧日志、清理 Docker 镜像等
> ```

---

### 3.2 磁盘分区与挂载

```bash
lsblk                             # 列出块设备（磁盘/分区/挂载点），树形显示
lsblk -f                          # 同上 + 文件系统类型 + UUID
fdisk -l                          # 列出所有磁盘分区详情（需要 root）
blkid                             # 查看设备 UUID 和文件系统类型

# 分区操作（在有新磁盘时用）
fdisk /dev/sdb                    # 交互式分区工具
# 交互命令：n=新建分区 d=删除 p=查看 w=写入保存 q=不保存退出

# 格式化
mkfs.ext4 /dev/sdb1               # 格式化为 ext4
mkfs.xfs /dev/sdb1                # 格式化为 xfs（RHEL 系默认）

# 挂载
mount /dev/sdb1 /mnt/data         # 临时挂载（重启失效）
umount /mnt/data                  # 卸载

# 永久挂载（写入 /etc/fstab）
echo "UUID=xxxx-xxxx  /mnt/data  xfs  defaults  0  0" >> /etc/fstab
mount -a                          # 重新挂载所有 fstab 条目（验证配置）
```

**/etc/fstab 字段说明**：
```
设备(UUID)          挂载点      文件系统  选项      dump  fsck顺序
UUID=abc123...    /mnt/data    xfs     defaults    0      0
```

---

### 3.3 软链接与硬链接

```bash
ln -s /etc/nginx/nginx.conf ~/nginx.conf.link   # 创建软链接（符号链接）
ln /etc/nginx/nginx.conf ~/nginx.conf.hard      # 创建硬链接

ls -li /etc/nginx/nginx.conf ~/nginx.conf.hard  # -i 显示 inode 编号（硬链接相同）
ls -la ~/nginx.conf.link                        # 软链接显示 -> 指向的路径
```

> **面试考点**：软链接 vs 硬链接
> | 对比 | 软链接（Symlink） | 硬链接（Hard Link） |
> |------|-------------------|---------------------|
> | 本质 | 存储目标路径的特殊文件 | 指向同一 inode 的目录项 |
> | 跨分区 | ✅ 可以 | ❌ 不能（同一文件系统） |
> | 目录 | ✅ 可以链接目录 | ❌ 通常不允许 |
> | 源文件删除 | 成为悬空链接（dangling） | 文件仍可访问（引用计数 -1） |
> | inode | 不同 | 相同 |

---

### 练习 3：磁盘管理

```bash
# 在广州 EulerOS 上练习（有 40G 系统盘）：

# 1. 全面查看磁盘状态
df -hT
lsblk -f
fdisk -l

# 2. 模拟"磁盘满了"排查流程
# 先创建一些大文件
dd if=/dev/zero of=/tmp/bigfile1 bs=1M count=500
dd if=/dev/zero of=/tmp/bigfile2 bs=1M count=300
# 然后用 du 找出来
du -sh /tmp/* | sort -rh | head

# 3. 软链接实战（运维常用场景）
# Nginx 配置文件管理：用软链接启用/禁用站点配置
mkdir -p /etc/nginx/sites-available /etc/nginx/sites-enabled
echo "# test site config" > /etc/nginx/sites-available/mysite.conf
ln -s /etc/nginx/sites-available/mysite.conf /etc/nginx/sites-enabled/mysite.conf
ls -la /etc/nginx/sites-enabled/
# 禁用站点只需删除软链接，不删原文件
rm /etc/nginx/sites-enabled/mysite.conf

# 4. 查看 inode 使用情况
df -i
# 找到 inode 使用最多的目录（inode 耗尽会导致"No space left"但 df -h 显示有空间）
find / -xdev -printf '%h\n' 2>/dev/null | sort | uniq -c | sort -rn | head -10
```

---

## 4. 网络排查

### 4.1 查看网络配置

```bash
ip addr                          # 查看所有网卡 IP（推荐，替代 ifconfig）
ip addr show eth0                # 查看指定网卡
ip route                         # 查看路由表
ip route show default            # 查看默认网关

# 网卡操作
ip link set eth0 up              # 启用网卡
ip link set eth0 down            # 禁用网卡
```

---

### 4.2 端口与连接查看

```bash
ss -tlnp                         # 查看所有监听的 TCP 端口（推荐，替代 netstat）
ss -tlnp | grep :80              # 查看 80 端口
ss -anp                          # 查看所有连接

netstat -tlnp                    # 功能同 ss（老系统兼容用）
# 安装：dnf install net-tools    # netstat 不再预装

# 字段说明：
# -t: TCP  -u: UDP  -l: listening  -n: 数字端口（不解析）  -p: 显示进程
```

**ss 输出解读**：
```
State   Recv-Q  Send-Q  Local Address:Port  Peer Address:Port  Process
LISTEN  0       511     0.0.0.0:80         0.0.0.0:*          users:(("nginx",pid=1234))
```

---

### 4.3 网络连通性排查

```bash
# 第一步：ping（ICMP 连通性）
ping -c 4 8.8.8.8                # 发 4 个包，测试连通性
ping -c 4 google.com             # 测试 DNS + 连通性

# 第二步：traceroute / tracepath（路由追踪）
traceroute 8.8.8.8               # 追踪到目标的路由路径
tracepath 8.8.8.8                # 不需要 root，功能类似

# 第三步：telnet / curl（端口连通性）
telnet 192.168.1.1 80            # 测试 TCP 端口是否可达（需安装）
curl -v http://example.com       # 测试 HTTP，-v 显示详细请求/响应头
curl -I https://example.com      # 只看响应头（HEAD 请求）
curl -o /dev/null -w "%{http_code}" https://example.com  # 只看状态码

# 第四步：dig / nslookup（DNS 查询）
dig google.com                   # 完整 DNS 查询
dig google.com A                 # 只查 A 记录（IPv4）
dig google.com MX                # 查邮件交换记录
dig @8.8.8.8 google.com          # 指定 DNS 服务器查询
nslookup google.com              # 简单 DNS 查询
```

> **面试场景题：如何排查网络不通？**
> ```
> 1. ping 目标 IP       → 判断 IP 层连通性
> 2. ping 网关           → 判断本地网络是否正常
> 3. traceroute          → 找到在哪一跳断掉
> 4. telnet IP 端口      → 判断目标端口是否开放
> 5. curl -v URL         → 判断应用层是否正常
> 6. dig 域名            → 排查 DNS 解析问题
> 7. ss -tlnp           → 确认本机服务是否在监听
> 8. iptables -L / firewall-cmd → 检查防火墙规则
> ```

---

### 4.4 防火墙管理

#### firewalld（RHEL 系推荐）

```bash
# 服务管理
systemctl start firewalld
systemctl enable firewalld
systemctl status firewalld

# 查看规则
firewall-cmd --list-all                        # 查看当前 zone 所有规则
firewall-cmd --get-active-zones                # 查看活跃 zone

# 开放端口/服务
firewall-cmd --permanent --add-port=8080/tcp   # 开放 TCP 8080
firewall-cmd --permanent --add-service=http    # 开放 HTTP 服务
firewall-cmd --permanent --add-service=https   # 开放 HTTPS 服务

# 关闭端口
firewall-cmd --permanent --remove-port=8080/tcp

# 生效（加了 --permanent 后必须 reload）
firewall-cmd --reload

# 临时规则（不加 --permanent，重启后失效）
firewall-cmd --add-port=9090/tcp
```

#### iptables（通用，面试必知）

```bash
iptables -L -n -v                # 查看所有规则（-n 数字 -v 详细）
iptables -L INPUT -n --line-numbers  # 查看 INPUT 链带行号

# 添加规则
iptables -A INPUT -p tcp --dport 80 -j ACCEPT    # 允许 80 端口入站
iptables -A INPUT -p tcp --dport 22 -j ACCEPT    # 允许 SSH
iptables -A INPUT -j DROP                        # 默认拒绝其他入站

# 删除规则
iptables -D INPUT 3              # 删除 INPUT 链第 3 条规则

# 保存规则（RHEL 系）
dnf install iptables-services -y
iptables-save > /etc/sysconfig/iptables
systemctl enable iptables
```

---

### 4.5 tcpdump 抓包

```bash
tcpdump -i eth0                  # 抓取 eth0 网卡所有流量
tcpdump -i eth0 port 80          # 只抓 80 端口
tcpdump -i eth0 host 1.2.3.4     # 只抓指定 IP 的流量
tcpdump -i eth0 -n -w /tmp/capture.pcap   # 写入文件（用 Wireshark 分析）
tcpdump -i eth0 tcp and port 443 -nn      # 抓 HTTPS，-nn 不解析主机名和端口
```

---

### 练习 4：网络排查

```bash
# 跨 VPS 网络排查练习（利用你的 4 台服务器）：

# 1. 基本连通性测试
ping -c 4 <上海VPS IP>
ping -c 4 <广州VPS IP>
ping -c 4 <雅加达VPS IP>

# 2. 路由追踪（观察跨地域延迟）
traceroute <雅加达VPS IP>        # 从上海到雅加达，会经过多少跳？

# 3. 确认你的服务都在监听正确端口
ssh 上海VPS
ss -tlnp | grep -E "22|80|443|9090|3000"

# 4. 用 curl 测试你部署的服务
curl -I https://你的域名              # 测试 HTTPS
curl -o /dev/null -sw "%{http_code} %{time_total}s\n" https://你的域名

# 5. 在广州机器上练习 firewalld
sudo systemctl start firewalld
sudo firewall-cmd --list-all
sudo firewall-cmd --permanent --add-port=8888/tcp
sudo firewall-cmd --reload
sudo firewall-cmd --list-ports
# 验证：在其他机器 telnet 广州IP 8888

# 6. tcpdump 抓包练习
sudo tcpdump -i eth0 -n port 22 -c 20   # 抓 20 个 SSH 包后停止
# 观察握手包（SYN/SYN-ACK/ACK）
```

---

## 5. 包管理

### 5.1 dnf / yum（RHEL 系 - Rocky Linux / EulerOS）

```bash
# 搜索与查询
dnf search nginx                 # 搜索包名
dnf info nginx                   # 查看包详情（版本、依赖等）
dnf list installed               # 查看已安装的包
dnf list installed | grep nginx  # 查询某个包是否已安装
dnf provides /usr/bin/python3    # 查找哪个包提供了某个文件

# 安装与卸载
dnf install nginx -y
dnf install nginx httpd php -y   # 同时安装多个
dnf remove nginx -y              # 卸载
dnf autoremove -y                # 卸载不再需要的依赖

# 更新
dnf update -y                    # 更新所有包
dnf update nginx -y              # 只更新指定包
dnf check-update                 # 查看可用更新（不执行）

# 缓存管理
dnf clean all                    # 清除缓存
dnf makecache                    # 重建缓存

# 历史记录
dnf history                      # 查看操作历史
dnf history undo 3               # 撤销第 3 次操作
```

#### 配置国内源（Rocky Linux 9）

```bash
# Rocky Linux 9 换阿里云镜像源
# 备份原始源
cp /etc/yum.repos.d/rocky.repo /etc/yum.repos.d/rocky.repo.bak

# 替换为阿里云源
sed -i 's|dl.rockylinux.org/$contentdir|mirrors.aliyun.com/rockylinux|g' \
    /etc/yum.repos.d/rocky*.repo

dnf clean all && dnf makecache
```

#### 配置 EPEL 源（Extra Packages for Enterprise Linux）

```bash
dnf install epel-release -y     # 启用 EPEL（htop 等工具在这里）
dnf install htop iotop ncdu -y  # 安装实用工具
```

---

### 5.2 apt（Debian 系 - 北京服务器）

```bash
# 更新包列表（必须先 update 再 install）
apt update

# 搜索与查询
apt search nginx
apt show nginx
dpkg -l | grep nginx             # 查看已安装的包
dpkg -L nginx                    # 查看包安装了哪些文件

# 安装与卸载
apt install nginx -y
apt remove nginx -y              # 卸载（保留配置文件）
apt purge nginx -y               # 彻底卸载（删除配置文件）
apt autoremove -y                # 清除无用依赖

# 更新
apt upgrade -y                   # 更新所有包
apt full-upgrade -y              # 升级（可能删除旧包）

# 清理
apt clean                        # 清除下载的 .deb 缓存
```

#### Debian 换国内源

```bash
# 查看系统版本代号
cat /etc/os-release | grep VERSION_CODENAME   # 如：bookworm

# 编辑源配置
nano /etc/apt/sources.list
```

```
# 替换为清华源（将 deb.debian.org 改为 mirrors.tuna.tsinghua.edu.cn）
deb https://mirrors.tuna.tsinghua.edu.cn/debian/ bookworm main contrib non-free non-free-firmware
deb https://mirrors.tuna.tsinghua.edu.cn/debian/ bookworm-updates main contrib non-free non-free-firmware
deb https://mirrors.tuna.tsinghua.edu.cn/debian-security bookworm-security main contrib non-free non-free-firmware
```

```bash
apt update
```

---

### 练习 5：包管理

```bash
# 在上海 Rocky Linux 上：
# 1. 安装实用运维工具
dnf install -y epel-release
dnf install -y htop iotop ncdu tree wget curl jq net-tools bind-utils

# 2. 查询某个命令属于哪个包
dnf provides netstat
dnf provides dig

# 3. 查看已安装包的文件列表
rpm -ql nginx                    # rpm 查询（RHEL 系）
rpm -qi nginx                    # 查看包详细信息

# 在北京 Debian 上：
# 4. 换清华源后验证速度
time apt update                  # 测量更新耗时

# 5. 用 ncdu 可视化磁盘使用
ncdu /var                        # 交互式查找大文件
```

---

## 6. 文本处理三剑客

> **面试高频！** `grep`/`sed`/`awk` 的组合使用是运维日常核心技能。

### 6.1 grep

```bash
# 基本用法
grep "error" /var/log/nginx/error.log        # 搜索包含 error 的行
grep -i "error" /var/log/syslog              # 忽略大小写
grep -n "error" app.log                      # 显示行号
grep -v "DEBUG" app.log                      # 反向匹配（不含 DEBUG 的行）
grep -c "error" app.log                      # 只显示匹配行数
grep -r "password" /etc/                     # 递归搜索目录
grep -l "nginx" /etc/systemd/system/*.service # 只显示匹配的文件名

# 上下文
grep -A 3 "error" app.log                    # 显示匹配行 + 后 3 行
grep -B 3 "error" app.log                    # 显示匹配行 + 前 3 行
grep -C 3 "error" app.log                    # 前后各 3 行

# 正则表达式
grep -E "error|warning" app.log              # 匹配 error 或 warning（扩展正则）
grep "^2026" access.log                      # 以 2026 开头
grep "\.log$" /etc/logrotate.d/*             # 以 .log 结尾
grep -E "[0-9]{1,3}\.[0-9]{1,3}" access.log  # 匹配 IP 地址格式
```

---

### 6.2 sed（流编辑器）

```bash
# 替换（最常用）
sed 's/old/new/' file.txt                    # 替换每行第一个匹配
sed 's/old/new/g' file.txt                   # 替换每行所有匹配（global）
sed 's/old/new/gi' file.txt                  # 忽略大小写替换
sed -i 's/old/new/g' file.txt                # 直接修改文件（-i in-place）
sed -i.bak 's/old/new/g' file.txt            # 修改前自动备份为 .bak

# 删除行
sed '3d' file.txt                            # 删除第 3 行
sed '/^#/d' config.conf                      # 删除注释行
sed '/^$/d' file.txt                         # 删除空行
sed '3,10d' file.txt                         # 删除第 3-10 行

# 打印指定行
sed -n '5p' file.txt                         # 打印第 5 行
sed -n '5,10p' file.txt                      # 打印第 5-10 行
sed -n '/start/,/end/p' file.txt             # 打印 start 到 end 之间的内容

# 插入/追加
sed '3a\新增的一行' file.txt                  # 在第 3 行后追加
sed '3i\插入到第3行前' file.txt               # 在第 3 行前插入
```

**实战案例**：
```bash
# 批量修改 Nginx 配置中的端口
sed -i 's/listen 80/listen 8080/g' /etc/nginx/conf.d/*.conf

# 删除配置文件中的所有注释行和空行
sed -i '/^[[:space:]]*#/d; /^[[:space:]]*$/d' config.conf

# 提取 IP 地址列（Nginx 访问日志第一列）
sed 's/ .*//' /var/log/nginx/access.log | sort | uniq -c | sort -rn | head
```

---

### 6.3 awk

> awk 是按列处理文本的强大工具，把每行按分隔符分成 `$1, $2, $3...`

```bash
# 基本用法
awk '{print $1}' file.txt                    # 打印每行第一列（默认空格分隔）
awk '{print $1, $3}' file.txt                # 打印第 1 和第 3 列
awk -F: '{print $1}' /etc/passwd             # 以 : 为分隔符，打印用户名
awk -F: '{print $1, $3}' /etc/passwd         # 打印用户名和 UID

# 条件过滤
awk '$3 > 1000' /etc/passwd                  # 打印 UID > 1000 的行（普通用户）
awk -F: '$3 > 1000 {print $1, $3}' /etc/passwd  # 过滤 + 格式化输出
awk '/nginx/' /var/log/syslog                # 相当于 grep nginx

# 内置变量
awk '{print NR, $0}' file.txt                # NR=行号，$0=整行
awk 'END {print NR}' file.txt                # 统计总行数（等同 wc -l）
awk -F: '{print NF}' /etc/passwd             # NF=每行的字段数

# 求和计算
awk '{sum += $1} END {print sum}' numbers.txt
df -h | awk 'NR>1 {print $5, $6}'            # 跳过标题行，打印使用率和挂载点
```

**实战案例（面试最爱考）**：

```bash
# 分析 Nginx 访问日志 - Top 10 IP
awk '{print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -rn | head -10

# 统计各 HTTP 状态码数量
awk '{print $9}' /var/log/nginx/access.log | sort | uniq -c | sort -rn

# 找出访问量最大的 URL
awk '{print $7}' /var/log/nginx/access.log | sort | uniq -c | sort -rn | head -10

# 计算平均响应时间（假设第 10 列是响应时间秒数）
awk '{sum+=$10; count++} END {printf "平均响应时间: %.3f秒\n", sum/count}' access.log

# 从 /etc/passwd 提取普通用户（UID >= 1000）
awk -F: '$3 >= 1000 && $3 != 65534 {print $1}' /etc/passwd
```

---

### 练习 6：三剑客实战

```bash
# 准备练习数据
cat > /tmp/access.log << 'EOF'
192.168.1.1 - - [11/Mar/2026:10:00:01 +0800] "GET /index.html HTTP/1.1" 200 1234
192.168.1.2 - - [11/Mar/2026:10:00:02 +0800] "POST /api/login HTTP/1.1" 401 56
192.168.1.1 - - [11/Mar/2026:10:00:03 +0800] "GET /about.html HTTP/1.1" 200 5678
10.0.0.5    - - [11/Mar/2026:10:00:04 +0800] "GET /admin HTTP/1.1" 403 100
192.168.1.3 - - [11/Mar/2026:10:00:05 +0800] "GET /index.html HTTP/1.1" 200 1234
192.168.1.1 - - [11/Mar/2026:10:00:06 +0800] "GET /image.png HTTP/1.1" 304 0
10.0.0.5    - - [11/Mar/2026:10:00:07 +0800] "POST /api/upload HTTP/1.1" 500 200
192.168.1.2 - - [11/Mar/2026:10:00:08 +0800] "GET /index.html HTTP/1.1" 200 1234
EOF

# Task 1：用 grep 找出所有 4xx 和 5xx 错误
grep -E '" [45][0-9]{2} ' /tmp/access.log

# Task 2：用 awk 统计各 IP 访问次数
awk '{print $1}' /tmp/access.log | sort | uniq -c | sort -rn

# Task 3：用 awk 统计各状态码数量
awk '{print $9}' /tmp/access.log | sort | uniq -c | sort -rn

# Task 4：用 sed 将日志中所有 192.168.1. 开头的 IP 替换为 INTERNAL
sed 's/192\.168\.1\.[0-9]*/INTERNAL/g' /tmp/access.log

# Task 5：组合使用 - 找到访问 500 错误的请求，提取 IP 和 URL
grep '" 500 ' /tmp/access.log | awk '{print $1, $7}'

# Task 6：用 awk 提取所有 200 响应的字节数并求总和
awk '$9 == 200 {sum += $10} END {print "总传输字节:", sum}' /tmp/access.log
```

---

## 7. 日志排查

### 7.1 日志文件分布

| 日志文件 | 内容 |
|----------|------|
| `/var/log/syslog` | 系统综合日志（Debian） |
| `/var/log/messages` | 系统综合日志（RHEL 系） |
| `/var/log/auth.log` | 认证/登录日志（Debian） |
| `/var/log/secure` | 认证/登录日志（RHEL 系） |
| `/var/log/nginx/access.log` | Nginx 访问日志 |
| `/var/log/nginx/error.log` | Nginx 错误日志 |
| `/var/log/dmesg` | 内核启动消息 |
| `/var/log/cron` | Cron 任务日志（RHEL 系） |
| `journalctl` | systemd 统一日志（推荐） |

---

### 7.2 日志查看命令

```bash
# tail - 实时跟踪日志（最常用）
tail -f /var/log/nginx/access.log             # 实时追加显示
tail -n 100 /var/log/nginx/error.log          # 查看最后 100 行
tail -f /var/log/nginx/error.log | grep -i "error"   # 实时过滤

# less - 交互式查看大文件
less /var/log/messages
# less 内快捷键：
# G → 跳到文件末尾     g → 跳到开头
# / → 向前搜索        ? → 向后搜索
# n → 下一个匹配      N → 上一个匹配
# q → 退出

# 多文件同时监控
tail -f /var/log/nginx/access.log /var/log/nginx/error.log

# 按时间范围查日志（journalctl 最强）
journalctl -u nginx --since "2026-03-11 10:00" --until "2026-03-11 12:00"
journalctl -u nginx --since "30 min ago"
journalctl -p err -u nginx                   # 只看 error 级别
```

---

### 7.3 logrotate 日志轮转

```bash
cat /etc/logrotate.conf          # 全局配置
ls /etc/logrotate.d/             # 各应用的日志轮转配置
cat /etc/logrotate.d/nginx       # Nginx 日志轮转配置

# 手动触发轮转（测试配置）
logrotate -f /etc/logrotate.d/nginx

# 查看上次轮转时间
cat /var/lib/logrotate/logrotate.status
```

**logrotate 配置示例**：
```
/var/log/nginx/*.log {
    daily           # 每天轮转
    rotate 14       # 保留 14 份
    compress        # 压缩旧文件（.gz）
    delaycompress   # 延迟压缩（最近一份不压缩）
    missingok       # 日志文件不存在不报错
    notifempty      # 空文件不轮转
    sharedscripts
    postrotate      # 轮转后重载 nginx
        /bin/kill -USR1 $(cat /run/nginx.pid 2>/dev/null) 2>/dev/null || true
    endscript
}
```

---

### 练习 7：日志排查

```bash
# 在上海 VPS 上练习（安装 nginx 后）：

# 1. 实时监控 Nginx 访问日志
sudo tail -f /var/log/nginx/access.log &
# 在另一个终端发几个请求
curl http://localhost/
curl http://localhost/nonexistent
curl http://localhost/admin

# 2. 排查 SSH 暴力破解
sudo grep "Failed password" /var/log/secure | tail -20
sudo grep "Failed password" /var/log/secure | awk '{print $11}' | sort | uniq -c | sort -rn | head
# 查看你的 VPS 被扫描的频率（IP 可以拉黑）

# 3. 用 journalctl 排查服务问题
# 故意制造错误：修改 nginx.conf 加入语法错误，然后 reload
sudo bash -c 'echo "invalid_directive;" >> /etc/nginx/conf.d/default.conf'
sudo systemctl reload nginx        # 应该失败
sudo journalctl -u nginx -n 20     # 查看错误原因
# 恢复
sudo sed -i '/invalid_directive/d' /etc/nginx/conf.d/default.conf
sudo systemctl reload nginx

# 4. 查看系统历史告警
sudo journalctl -p warning --since "24 hours ago"

# 5. 分析 auth.log 找出今日登录记录
sudo journalctl _SYSTEMD_UNIT=sshd.service --since today
```

---

## 8. 综合练习项目

### 项目一：手动部署 Web 服务器（不用 Docker）

在**广州 EulerOS** 上从零手动部署 Nginx：

```bash
# Step 1: 安装 Nginx
dnf install nginx -y

# Step 2: 配置防火墙
systemctl enable --now firewalld
firewall-cmd --permanent --add-service=http
firewall-cmd --permanent --add-service=https
firewall-cmd --reload

# Step 3: 创建网站目录和内容
mkdir -p /var/www/mysite
cat > /var/www/mysite/index.html << 'EOF'
<!DOCTYPE html>
<html>
<head><title>My VPS</title></head>
<body>
<h1>Welcome to my VPS!</h1>
<p>Hostname: guangzhou-euleros</p>
</body>
</html>
EOF
chown -R nginx:nginx /var/www/mysite
chmod -R 755 /var/www/mysite

# Step 4: 配置 Nginx 虚拟主机
cat > /etc/nginx/conf.d/mysite.conf << 'EOF'
server {
    listen 80;
    server_name _;          # 接受所有域名/IP 请求
    root /var/www/mysite;
    index index.html;

    access_log /var/log/nginx/mysite_access.log;
    error_log  /var/log/nginx/mysite_error.log;

    location / {
        try_files $uri $uri/ =404;
    }
}
EOF

# Step 5: 验证配置并启动
nginx -t                             # 检查配置语法
systemctl enable --now nginx
systemctl status nginx
curl http://localhost                 # 本地验证

# Step 6: 配置定时日志清理（crontab）
crontab -e
# 添加：每天凌晨 2 点删除 30 天前的日志
# 0 2 * * * find /var/log/nginx -name "*.log.*" -mtime +30 -delete
```

---

### 项目二：Systemd Service 管理自定义脚本

```bash
# Step 1: 创建一个需要持续运行的脚本
cat > /opt/health-monitor.sh << 'EOF'
#!/bin/bash
# 简单的服务健康监控脚本
LOG="/var/log/health-monitor.log"

while true; do
    TIMESTAMP=$(date '+%Y-%m-%d %H:%M:%S')
    CPU=$(top -bn1 | grep "Cpu(s)" | awk '{print $2}' | cut -d. -f1)
    MEM=$(free | grep Mem | awk '{printf "%.1f", $3/$2 * 100}')
    DISK=$(df / | tail -1 | awk '{print $5}' | tr -d '%')

    echo "$TIMESTAMP CPU:${CPU}% MEM:${MEM}% DISK:${DISK}%" >> "$LOG"

    # 告警阈值检查
    if [ "$DISK" -gt 85 ]; then
        echo "$TIMESTAMP ALERT: Disk usage ${DISK}% exceeded 85%!" >> "$LOG"
    fi

    sleep 60
done
EOF
chmod +x /opt/health-monitor.sh

# Step 2: 创建 systemd service 文件
cat > /etc/systemd/system/health-monitor.service << 'EOF'
[Unit]
Description=Server Health Monitor
After=network.target

[Service]
Type=simple
ExecStart=/opt/health-monitor.sh
Restart=on-failure
RestartSec=5s
User=root
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
EOF

# Step 3: 启用并启动服务
systemctl daemon-reload              # 重新加载 systemd 配置
systemctl enable --now health-monitor
systemctl status health-monitor

# Step 4: 验证
journalctl -u health-monitor -f      # 实时查看输出
tail -f /var/log/health-monitor.log  # 查看日志文件
```

---

### 项目三：Shell 脚本综合练习

#### 自动备份脚本

```bash
cat > /opt/backup.sh << 'EOF'
#!/bin/bash
# 自动备份脚本：备份指定目录，保留最近 7 天

set -euo pipefail   # 严格模式：错误立即退出，未定义变量报错

# 配置
BACKUP_SOURCE="/etc/nginx"
BACKUP_DEST="/backup/nginx"
KEEP_DAYS=7
DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_FILE="${BACKUP_DEST}/nginx_${DATE}.tar.gz"

# 创建备份目录
mkdir -p "$BACKUP_DEST"

# 执行备份
echo "[$(date)] Starting backup of $BACKUP_SOURCE..."
tar -czf "$BACKUP_FILE" "$BACKUP_SOURCE" 2>&1
echo "[$(date)] Backup saved to: $BACKUP_FILE"
echo "[$(date)] Backup size: $(du -sh "$BACKUP_FILE" | cut -f1)"

# 删除超过 7 天的备份
DELETED=$(find "$BACKUP_DEST" -name "*.tar.gz" -mtime +${KEEP_DAYS} -print -delete | wc -l)
echo "[$(date)] Deleted $DELETED old backup(s) older than ${KEEP_DAYS} days"

echo "[$(date)] Backup completed. Current backups:"
ls -lh "$BACKUP_DEST"
EOF
chmod +x /opt/backup.sh

# 添加 crontab 定时任务：每天凌晨 3 点执行
(crontab -l 2>/dev/null; echo "0 3 * * * /opt/backup.sh >> /var/log/backup.log 2>&1") | crontab -
crontab -l   # 验证
```

---

## 知识检测：面试模拟问答

> 合上教程，尝试回答这些问题（在 VPS 上演示给自己看）：

**用户权限**
- [ ] 新建用户 `testops`，加入 `wheel` 组，设置密码，用 `sudo -l` 验证权限
- [ ] 创建一个目录，让 `testops` 可读可写，其他用户只读
- [ ] `chmod 4755` 是什么意思？和 `755` 有什么区别？

**进程管理**
- [ ] 用 `ps aux` 找到 nginx 主进程 PID，再用 `kill -HUP` 重载配置
- [ ] `kill -9` 和 `kill -15` 什么时候分别用？
- [ ] 如何查看一个进程打开了哪些文件？（提示：`lsof -p <PID>`）
- [ ] 系统负载（load average）超过多少才算高？

**磁盘**
- [ ] 磁盘 100% 时如何快速找到占用最多的目录和文件？
- [ ] inode 耗尽是什么现象？如何排查？
- [ ] 软链接和硬链接在 `ls -li` 中有什么区别？

**网络**
- [ ] 如何查看哪个进程在用 8080 端口？
- [ ] `ss -tlnp` 中 `LISTEN` 的 `0.0.0.0:80` 和 `127.0.0.1:6379` 有什么区别？
- [ ] ping 通但 curl 失败，可能是什么原因？

**三剑客**
- [ ] 用 `awk` 统计 `/var/log/nginx/access.log` 中 Top 5 IP 地址
- [ ] 用 `sed` 将配置文件中所有 `http://` 替换为 `https://`，并备份原文件
- [ ] 用 `grep` 找到 `/var/log/secure` 中今天失败的 SSH 登录尝试

---

## 学习进度检查清单

### 第 1 周（用户权限 + 进程管理）
- [ ] 能独立完成用户/组创建和权限配置
- [ ] 理解 chmod 数字和符号两种方式
- [ ] 能用 ps/top 找出异常进程并处理
- [ ] 掌握 systemctl 的常用操作
- [ ] 完成练习 1 + 练习 2

### 第 2 周（磁盘 + 网络）
- [ ] 能快速排查磁盘满了的问题
- [ ] 理解软链接和硬链接的区别和应用场景
- [ ] 能用 ss/ip/firewall-cmd 排查网络问题
- [ ] 完成跨 VPS 网络排查练习
- [ ] 完成练习 3 + 练习 4

### 第 3 周（包管理 + 三剑客）
- [ ] 掌握 dnf 和 apt 的常用命令
- [ ] 完成换国内源配置
- [ ] grep/sed/awk 组合分析 Nginx 日志
- [ ] 完成练习 5 + 练习 6

### 第 4 周（综合项目）
- [ ] 完成广州 VPS 手动部署 Nginx 网站
- [ ] 编写并运行 Systemd 托管的监控脚本
- [ ] 完成自动备份脚本 + crontab 配置
- [ ] 能回答「知识检测」里所有问题
- [ ] 完成综合练习项目

---

> **下一步**：完成本阶段后，进入 [第二阶段：工具链实战](./sre_internship_roadmap.md#-第二阶段工具链实战4月中--5月中)
> 重点：Docker 进阶 + Prometheus/Grafana 监控栈部署
