# Docker Environment & Service Deployment Log
# Docker 环境安装与服务部署记录

**Environment / 环境信息:**
- **OS:** Debian 12 (Bookworm)
- **User:** ops (sudoer)
- **Network Strategy:** Aliyun Mirrors (for apt) + Registry Mirrors (for docker pull)
- **Deployed Services:** Nginx (Web Server), Uptime Kuma (Monitoring), Portainer (GUI), NPM (Reverse Proxy)

---

## 1. Docker Installation (Aliyun Mirror Source)
Due to network restrictions, use Aliyun mirrors instead of the official Docker hub.  
由于网络原因，使用阿里云镜像源替代官方源进行安装。

### 1.1 Prerequisite / 准备工作
```bash
# Remove conflicting packages (Clean install)
sudo apt-get remove docker.io docker-doc docker-compose podman-docker containerd runc

# Install required tools
sudo apt-get update
sudo apt-get install ca-certificates curl gnupg -y
```

### 1.2 Add GPG Key & Repository / 添加密钥与软件源
```bash
# Create key directory
sudo install -m 0755 -d /etc/apt/keyrings

# Download Docker GPG key (Official)
curl -fsSL https://download.docker.com/linux/debian/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# Add Aliyun Repository (Crucial Step for Speed)
# 使用阿里云的 Docker-CE 源
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://mirrors.aliyun.com/docker-ce/linux/debian \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

### 1.3 Install Docker Engine / 正式安装
```bash
# Update package index again
sudo apt-get update

# Install Docker and plugins
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y

# Verify installation
sudo docker --version
```

---

## 2. Docker Registry Mirror Configuration
Configure image accelerators to fix dial tcp i/o timeout errors when pulling images.  
配置镜像加速器，解决拉取镜像超时的问题。

```bash
# 1. Edit daemon config
sudo nano /etc/docker/daemon.json

# 2. Add the following content:
{
  "registry-mirrors": [
    "https://docker.m.daocloud.io",
    "https://huecker.io",
    "https://dockerhub.timeweb.cloud",
    "https://noohub.ru"
  ]
}

# 3. Apply changes
sudo systemctl daemon-reload
sudo systemctl restart docker

# 4. Test connectivity
sudo docker run hello-world
```

---

## 3. Nginx Deployment (Static Web Server)
Deploy Nginx with volume mapping for easy content updates.  
部署 Nginx 并挂载本地目录，实现网页热更新。

### 3.1 Prepare Content / 准备网页文件
```bash
# Create directory structure
mkdir -p ~/my-website/html

# Create a UTF-8 encoded HTML file (Fixes encoding issues)
nano ~/my-website/html/index.html
# (Content omitted, ensure <meta charset="UTF-8"> is included)
```

### 3.2 Run Container / 启动容器
```bash
sudo docker run -d \
  --name my-nginx \
  -p 80:80 \
  -v ~/my-website/html:/usr/share/nginx/html \
  nginx
```

**Parameters / 参数详解:**
- `-d`: Run in background (Detached mode). / 后台运行。
- `--name`: Container name. / 容器名称。
- `-p 80:80`: Map host port 80 to container port 80. / 端口映射。
- `-v ~/path:/path`: Map host directory to container directory. / 挂载卷，实现"所见即所得"修改。

---

## 4. Uptime Kuma Deployment (Monitoring)
Self-hosted monitoring tool for tracking server uptime.  
部署自托管监控工具。

### 4.1 Firewall Config / 防火墙配置
```bash
# Allow Kuma port
sudo ufw allow 3001/tcp
```

### 4.2 Run Container / 启动容器
```bash
# 1. Create a persistent volume for database
# 创建独立数据卷，防止数据丢失
sudo docker volume create uptime-kuma

# 2. Start Uptime Kuma
sudo docker run -d --restart=always \
  -p 3001:3001 \
  -v uptime-kuma:/app/data \
  --name uptime-kuma \
  louislam/uptime-kuma:1
```

**Parameters / 参数详解:**
- `--restart=always`: Auto-restart container if it crashes or server reboots. / 守护进程模式，开机自启，挂掉重启。
- `-v uptime-kuma:/app/data`: Maps a named Docker volume to the app data folder. / 使用命名卷持久化存储数据。

---

## 5. Portainer Deployment (GUI Management)
Deploy a visual management panel for Docker containers.  
部署 Portainer 可视化面板，用于图形化管理 Docker。

### 5.1 Firewall Config / 防火墙配置
```bash
# Allow Portainer Web UI port
sudo ufw allow 9000/tcp
```

### 5.2 Run Container / 启动容器
```bash
# 1. Create volume for Portainer data
sudo docker volume create portainer_data

# 2. Start Portainer CE (Community Edition)
# Critical: Mounting /var/run/docker.sock grants control over the host Docker daemon.
# 关键点：挂载 docker.sock 赋予了容器管理宿主机 Docker 的权限。
sudo docker run -d \
  -p 9000:9000 \
  --name portainer \
  --restart=always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest
```

### 5.3 Usage / 使用说明
- **Access:** http://&lt;Server-IP&gt;:9000
- **Security Note:** Ensure a strong password is used as this port is public.

---

## 6. Nginx Proxy Manager (Reverse Proxy Gateway)
Deploy an advanced reverse proxy to manage all incoming traffic on port 80/443.  
部署反向代理网关，接管 80/443 端口的所有流量。

### 6.1 Architecture Change / 架构变更
- **Old:** External → Nginx (Port 80)
- **New:** External → NPM (Port 80) → Forward to Internal Services (e.g., Nginx @ 8080)
- **Action:** Stop and remove the old Nginx container occupying port 80.
  ```bash
  sudo docker rm -f my-nginx
  ```

### 6.2 Firewall Config / 防火墙配置
```bash
# Allow NPM Admin Panel port
sudo ufw allow 81/tcp

# Note: Port 80/443 are already allowed in previous steps.
# 注意：80/443 端口在之前的步骤中已放行。
```

### 6.3 Deployment via Docker Compose / 使用 Compose 部署
NPM requires a database (SQLite by default) and complex port mapping, so Docker Compose is preferred.  
NPM 需要数据库支持且端口映射较多，使用 Docker Compose 管理更方便。

```bash
# 1. Create workspace
mkdir -p ~/npm
cd ~/npm

# 2. Create config file
nano docker-compose.yml
```

**docker-compose.yml content:**
```yaml
version: '3.8'
services:
  app:
    image: 'jc21/nginx-proxy-manager:latest'
    restart: unless-stopped
    ports:
      - '80:80'      # HTTP Traffic
      - '81:81'      # Admin Panel
      - '443:443'    # HTTPS Traffic
    volumes:
      - ./data:/data
      - ./letsencrypt:/etc/letsencrypt
```

```bash
# 3. Start the service
sudo docker compose up -d
```

### 6.4 Service Adjustments / 服务调整
Re-deploy the static website on a non-standard port (e.g., 8080) so NPM can proxy it.  
将静态网站重新部署到非标准端口（如 8080），以便 NPM 进行代理。

```bash
sudo docker run -d \
  --name my-website \
  -p 8080:80 \
  -v ~/my-website/html:/usr/share/nginx/html \
  nginx
```

### 6.5 Proxy Configuration / 代理配置
- **Access:** http://&lt;Server-IP&gt;:81
- **Default Login:** admin@example.com / changeme
- **Rule:** Forward &lt;Server-IP&gt; (Port 80) → 172.17.0.1 (Port 8080)

---

## 7. Usage Summary / 使用说明

| Service | URL | Note |
|---------|-----|------|
| Web Access | http://&lt;Server-IP&gt; | Via NPM Proxy |
| NPM Admin | http://&lt;Server-IP&gt;:81 | Default: admin@example.com |
| Monitoring Panel | http://&lt;Server-IP&gt;:3001 | Uptime Kuma |
| Docker GUI | http://&lt;Server-IP&gt;:9000 | Portainer |

**Update Website:** Edit `~/my-website/html/index.html` directly on the server.
