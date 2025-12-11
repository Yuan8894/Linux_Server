Markdown

# Docker Environment & Service Deployment Log
# Docker 环境安装与服务部署记录

**Environment / 环境信息:**
- **OS:** Debian 12 (Bookworm)
- **User:** ops (sudoer)
- **Network Strategy:** Aliyun Mirrors (for apt) + Registry Mirrors (for docker pull)
- **Deployed Services:** Nginx (Web Server), Uptime Kuma (Monitoring)

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
1.2 Add GPG Key & Repository / 添加密钥与软件源
Bash

# Create key directory
sudo install -m 0755 -d /etc/apt/keyrings

# Download Docker GPG key (Official)
curl -fsSL [https://download.docker.com/linux/debian/gpg](https://download.docker.com/linux/debian/gpg) | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# Add Aliyun Repository (Crucial Step for Speed)
# 使用阿里云的 Docker-CE 源
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] [https://mirrors.aliyun.com/docker-ce/linux/debian](https://mirrors.aliyun.com/docker-ce/linux/debian) \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
1.3 Install Docker Engine / 正式安装
Bash

# Update package index again
sudo apt-get update

# Install Docker and plugins
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y

# Verify installation
sudo docker --version
2. Docker Registry Mirror Configuration
Configure image accelerators to fix dial tcp i/o timeout errors when pulling images. 配置镜像加速器，解决拉取镜像超时的问题。

Bash

# 1. Edit daemon config
sudo nano /etc/docker/daemon.json

# 2. Add the following content:
{
  "registry-mirrors": [
    "[https://docker.m.daocloud.io](https://docker.m.daocloud.io)",
    "[https://huecker.io](https://huecker.io)",
    "[https://dockerhub.timeweb.cloud](https://dockerhub.timeweb.cloud)",
    "[https://noohub.ru](https://noohub.ru)"
  ]
}

# 3. Apply changes
sudo systemctl daemon-reload
sudo systemctl restart docker

# 4. Test connectivity
sudo docker run hello-world
3. Nginx Deployment (Static Web Server)
Deploy Nginx with volume mapping for easy content updates. 部署 Nginx 并挂载本地目录，实现网页热更新。

3.1 Prepare Content / 准备网页文件
Bash

# Create directory structure
mkdir -p ~/my-website/html

# Create a UTF-8 encoded HTML file (Fixes encoding issues)
nano ~/my-website/html/index.html
# (Content omitted, ensure <meta charset="UTF-8"> is included)
3.2 Run Container / 启动容器
Bash

sudo docker run -d \
  --name my-nginx \
  -p 80:80 \
  -v ~/my-website/html:/usr/share/nginx/html \
  nginx
Parameters / 参数详解:

-d: Run in background (Detached mode). / 后台运行。

--name: Container name. / 容器名称。

-p 80:80: Map host port 80 to container port 80. / 端口映射。

-v ~/path:/path: Map host directory to container directory. / 挂载卷，实现“所见即所得”修改。

4. Uptime Kuma Deployment (Monitoring)
Self-hosted monitoring tool for tracking server uptime. 部署自托管监控工具。

4.1 Firewall Config / 防火墙配置
Bash

# Allow Kuma port
sudo ufw allow 3001/tcp
4.2 Run Container / 启动容器
Bash

# 1. Create a persistent volume for database
# 创建独立数据卷，防止数据丢失
sudo docker volume create uptime-kuma

# 2. Start Uptime Kuma
sudo docker run -d --restart=always \
  -p 3001:3001 \
  -v uptime-kuma:/app/data \
  --name uptime-kuma \
  louislam/uptime-kuma:1
Parameters / 参数详解:

--restart=always: Auto-restart container if it crashes or server reboots. / 守护进程模式，开机自启，挂掉重启。

-v uptime-kuma:/app/data: Maps a named Docker volume to the app data folder. / 使用命名卷持久化存储数据。

5. Usage / 使用说明
Web Access: http://<Server-IP>

Monitoring Panel: http://<Server-IP>:3001

Update Website: Edit ~/my-website/html/index.html directly on the server.


### 💡 提交建议

1.  **文件结构**：建议你在本地 Git 仓库里建一个文件夹叫 `docs`，把这两个 md 文件放进去。
2.  **代码提交**：
    * `git add docs/Server_Init.md docs/Docker_Services_Setup.md`
    * `git commit -m "docs: add server initialization and docker service deployment logs"`
    * `git push`

这样你的 GitHub 上就会有非常漂亮的绿色格子，而且面试官看到你这么规范的文档，好感度会直接拉满。