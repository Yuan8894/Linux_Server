# Debian 12 Server Initialization & Security Hardening
# Debian 12 服务器初始化与安全加固记录

**Environment / 环境信息:**
- **OS:** Debian 12 (Bookworm)
- **Specs:** 2 Core / 2GB RAM / 1Mbps Bandwidth
- **Initial User:** Root
- **Target:** Production-ready secure environment (Docker/Nginx baseline)

---

## 1. System Update / 系统更新
First, ensure all repositories and installed packages are up to date.  
首先确保软件源和已安装的包是最新的。

```bash
# Update package list and upgrade packages
# -y: Automatically answer "yes" to prompts
apt update && apt upgrade -y
```

---

## 2. User Management / 用户管理
Avoid using the root user for daily operations. Create a sudo-privileged user.  
避免日常使用 root 用户。创建一个具有 sudo 权限的普通用户。

```bash
# 1. Install sudo (Debian minimal install may miss this)
apt install sudo -y

# 2. Create new user (Follow prompts, press Enter for defaults)
# Replace <username> with your actual username (e.g., ops)
adduser <username>

# 3. Grant sudo privileges
# -a: Append (add to group)
# -G: Group name (sudo)
usermod -aG sudo <username>
```

---

## 3. SSH Key Migration / 密钥迁移
Transfer existing root SSH keys to the new user to enable passwordless login.  
将 Root 的 SSH 公钥迁移给新用户,实现免密登录。

```bash
# 1. Create .ssh directory for the new user
mkdir -p /home/<username>/.ssh

# 2. Copy authorized_keys from root
cp /root/.ssh/authorized_keys /home/<username>/.ssh/

# 3. Change ownership (Crucial Step!)
# -R: Recursive
# Syntax: user:group
chown -R <username>:<username> /home/<username>/.ssh

# 4. Set strict permissions (Security Requirement)
# 700: rwx------ (Owner only read/write/execute directory)
# 600: rw------- (Owner only read/write file)
chmod 700 /home/<username>/.ssh
chmod 600 /home/<username>/.ssh/authorized_keys
```

---

## 4. Firewall Configuration (UFW) / 防火墙配置
Configure firewall rules before changing SSH config to avoid lockout.  
在修改 SSH 配置前先配置防火墙,防止被锁在外面。

```bash
# 1. Install UFW
apt install ufw -y

# 2. Allow SSH Port (Custom Port)
# Replace <port> with your custom SSH port (e.g., 20240)
ufw allow <port>/tcp

# 3. Allow Web Ports (HTTP/HTTPS)
ufw allow 80/tcp
ufw allow 443/tcp

# 4. Enable Firewall
# Warning: This may disrupt existing connections if port is not allowed.
ufw enable

# 5. Check Status
ufw status
```

---

## 5. SSH Hardening / SSH 加固
Modify `/etc/ssh/sshd_config` to secure remote access.  
修改 SSH 配置文件以增强安全性。

**Configuration Changes / 配置变更点:**

```conf
# Change default port to avoid automated scanners
Port <your_custom_port>

# Disable root login for security
PermitRootLogin no

# Enable Public Key Authentication
PubkeyAuthentication yes

# Disable Password Authentication (Force Key-only)
PasswordAuthentication no
```

**Apply Changes / 应用变更:**

```bash
# Restart SSH service to apply changes
systemctl restart ssh
```

---

## 6. Validation / 验证
**Do not close the current session.** Open a new terminal to test.  
**切勿关闭当前会话。** 打开一个新的终端窗口进行测试。

```bash
# Test login with new user and port
ssh -p <your_custom_port> <username>@<server_ip>
```
