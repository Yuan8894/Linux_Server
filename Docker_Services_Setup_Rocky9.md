# Docker 环境安装与服务部署记录（Rocky Linux 9 适用）

## 1. Docker 安装（使用阿里云镜像源）

### 1.1 准备工作
```bash
# 移除可能存在的旧版本
sudo dnf remove docker-ce docker-ce-cli containerd.io docker-compose-plugin docker-buildx-plugin docker-compose podman-docker runc -y

# 安装依赖
sudo dnf install -y yum-utils device-mapper-persistent-data lvm2 curl ca-certificates gnupg2
```

### 1.2 添加 Docker 仓库（阿里云镜像源）
```bash
sudo yum-config-manager --add-repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
```

### 1.3 安装 Docker Engine
```bash
sudo dnf makecache
sudo dnf install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

sudo systemctl enable --now docker
sudo docker --version
```

## 2. 镜像加速器配置
```bash
sudo mkdir -p /etc/docker
sudo nano /etc/docker/daemon.json
# 添加如下内容：
{
  "registry-mirrors": [
    "https://docker.m.daocloud.io",
    "https://huecker.io",
    "https://dockerhub.timeweb.cloud",
    "https://noohub.ru"
  ]
}

sudo systemctl daemon-reload
sudo systemctl restart docker
sudo docker run hello-world
```

## 3. Nginx 静态网站部署
```bash
mkdir -p ~/my-website/html
nano ~/my-website/html/index.html
sudo docker run -d \
  --name my-nginx \
  -p 80:80 \
  -v ~/my-website/html:/usr/share/nginx/html \
  nginx
```

## 4. Uptime Kuma 部署
```bash
sudo firewall-cmd --add-port=3001/tcp --permanent
sudo firewall-cmd --reload
sudo docker volume create uptime-kuma
sudo docker run -d --restart=always \
  -p 3001:3001 \
  -v uptime-kuma:/app/data \
  --name uptime-kuma \
  louislam/uptime-kuma:1
```

## 5. Portainer 部署
```bash
sudo firewall-cmd --add-port=9000/tcp --permanent
sudo firewall-cmd --reload
sudo docker volume create portainer_data
sudo docker run -d \
  -p 9000:9000 \
  --name portainer \
  --restart=always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest
```

## 6. Nginx Proxy Manager 通过 Docker Compose 部署
```bash
sudo firewall-cmd --add-port=80/tcp --permanent
sudo firewall-cmd --add-port=81/tcp --permanent
sudo firewall-cmd --add-port=443/tcp --permanent
sudo firewall-cmd --reload
mkdir -p ~/npm
cd ~/npm
nano docker-compose.yml
```

docker-compose.yml 内容：
```yaml
version: '3.8'
services:
  app:
    image: 'jc21/nginx-proxy-manager:latest'
    restart: unless-stopped
    ports:
      - '80:80'
      - '81:81'
      - '443:443'
    volumes:
      - ./data:/data
      - ./letsencrypt:/etc/letsencrypt
```

启动服务：
```bash
sudo docker compose up -d
```

---
如有细节补充或实际需求差异，可继续拓展。
