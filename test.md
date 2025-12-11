# My Personal Cloud Server Operations
# 个人云服务器运维项目

**Owner:** Ops Engineer (You)  
**Server OS:** Debian 12 (Bookworm)  
**Specs:** 2C / 2G / 1Mbps

---

## 📂 Project Structure / 项目结构

```text
.
├── docs/                   # Documentation logs
├── services/
│   ├── npm/                # Nginx Proxy Manager (Docker Compose)
│   └── my-website/         # Static Website (HTML)
└── README.md               # This file
```

---

## 🛠 Deployed Services / 已部署服务

| Service | Port (Internal) | Port (External) | URL |
|---------|----------------|-----------------|-----|
| Nginx Proxy Manager | 81 | 81 | http://&lt;IP&gt;:81 |
| Uptime Kuma | 3001 | 3001 | http://&lt;IP&gt;:3001 |
| Portainer | 9000 | 9000 | http://&lt;IP&gt;:9000 |
| My Website | 80 | 8080 (via Proxy) | http://&lt;IP&gt; |

---

## 🚀 Quick Start / 快速恢复指南

### 1. Nginx Proxy Manager
```bash
cd services/npm
docker compose up -d
```

### 2. Static Website
```bash
docker run -d \
  --name my-website \
  -p 8080:80 \
  -v $(pwd)/services/my-website/html:/usr/share/nginx/html \
  nginx
```

---

## 📝 Change Log / 变更日志

- Initialized server security (UFW, SSH Hardening).
- Installed Docker & Docker Compose.
- Deployed Uptime Kuma for monitoring.
- Deployed Portainer for management.
- Migrated to Nginx Proxy Manager for gateway management.
