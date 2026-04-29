# 04 — GCE VM Setup

---

## Thông tin VM

| Thông số | Giá trị |
|---------|---------|
| Provider | Google Cloud Compute Engine |
| Instance name | evena-prod |
| Zone | asia-southeast1-b (Singapore) |
| Machine type | e2-medium (2 vCPU, 4 GB RAM) |
| Disk | 50 GB SSD (pd-ssd) |
| OS | Debian 12 |
| External IP | 34.2.18.161 (static) |
| OS username | nguyenhuyhoanganh900 |

---

## Bước 1 — Tạo VM (đã thực hiện)

```bash
gcloud compute instances create evena-prod \
  --zone=asia-southeast1-b \
  --machine-type=e2-medium \
  --image-family=debian-12 \
  --image-project=debian-cloud \
  --boot-disk-size=50GB \
  --boot-disk-type=pd-ssd \
  --tags=http-server,https-server

# Mở firewall port 80 và 443
gcloud compute firewall-rules create allow-http-https \
  --allow=tcp:80,tcp:443 \
  --target-tags=http-server,https-server
```

---

## Bước 2 — Cài Docker

```bash
# SSH vào VM
gcloud compute ssh evena-prod --zone=asia-southeast1-b

# Cài Docker Engine
curl -fsSL https://get.docker.com | sh

# Thêm user vào group docker (không cần sudo)
sudo usermod -aG docker $USER
newgrp docker

# Kiểm tra
docker --version          # Docker 27.x
docker compose version    # Docker Compose v2.x
```

---

## Bước 3 — Tạo cấu trúc thư mục

```bash
sudo mkdir -p /home/deploy/evena
sudo chown -R $USER:$USER /home/deploy
cd /home/deploy/evena
```

---

## Bước 4 — Setup SSH key cho GitHub

VM cần SSH key để clone private repo từ GitHub:

```bash
# Kiểm tra key đã tồn tại chưa
ls ~/.ssh/

# Tạo key mới nếu chưa có
ssh-keygen -t ed25519 -C "evena-prod-vm"

# Xem public key
cat ~/.ssh/id_ed25519.pub
```

Thêm public key vào GitHub:
1. GitHub → Settings → SSH and GPG keys → New SSH key
2. Dán nội dung `id_ed25519.pub`
3. Title: "evena-prod-vm"

---

## Bước 5 — Clone code

```bash
cd /home/deploy/evena

# Clone gateway (chứa docker-compose.prod.yml, nginx, deploy.sh)
git clone -b staging git@github.com:Event-management-bkproject/gateway.git gateway

# Clone backend (Spring Boot source)
git clone -b staging git@github.com:Event-management-bkproject/backend.git backend

# Clone sse-service (FastAPI source)
git clone -b staging git@github.com:hoanganh2k4/sse-service.git sse-service
```

---

## Cấu trúc thư mục trên VM

```
/home/deploy/evena/
├── gateway/
│   ├── docker-compose.prod.yml    ← compose chính cho toàn bộ stack
│   ├── nginx.prod.conf            ← nginx config production
│   ├── .env.prod                  ← secrets (KHÔNG commit)
│   ├── .env.prod.example          ← template
│   ├── deploy.sh                  ← script deploy thủ công
│   ├── Dockerfile                 ← nginx image
│   └── ssl/
│       ├── cf.crt                 ← Cloudflare Origin Certificate
│       └── cf.key                 ← Cloudflare Origin Private Key
├── backend/
│   └── (Spring Boot source — dùng để build image trên CI, không build trên VM)
└── sse-service/
    └── (FastAPI source — dùng để build image trên CI, không build trên VM)
```

> **Quan trọng:** VM không build Docker image. CI/CD build và push lên GHCR. VM chỉ chạy `docker compose pull` + `docker compose up`.

---

## Bước 6 — Login GHCR trên VM

```bash
# Tạo Personal Access Token trên GitHub:
# GitHub → Settings → Developer settings → Personal access tokens → Fine-grained
# Scope cần: read:packages

# Login GHCR
docker login ghcr.io -u hoanganh2k4
# Nhập PAT khi được hỏi

# Test pull
docker pull ghcr.io/event-management-bkproject/backend:staging
```

---

## Bước 7 — Tạo thư mục SSL

```bash
mkdir -p /home/deploy/evena/gateway/ssl
chmod 700 /home/deploy/evena/gateway/ssl
```

Sau đó copy cert vào (xem [06-ssl-certificates.md](06-ssl-certificates.md)).

---

## SSH vào VM (từ máy local)

```bash
# Qua gcloud
gcloud compute ssh evena-prod --zone=asia-southeast1-b

# Hoặc SSH trực tiếp
ssh nguyenhuyhoanganh900@34.2.18.161
```

---

## Lỗi thường gặp khi setup VM

### nginx crash: "cannot load certificate"
```
nginx: [emerg] cannot load certificate "/etc/nginx/ssl/cf.crt"
```
**Nguyên nhân:** Thư mục `ssl/` hoặc file `cf.crt` chưa tồn tại.  
**Sửa:**
```bash
mkdir -p /home/deploy/evena/gateway/ssl
# Tạo cf.crt và cf.key (xem 06-ssl-certificates.md)
```

### nginx crash: "cannot load sse cert"
```
nginx: [emerg] cannot load certificate "/etc/letsencrypt/live/sse.evena.id.vn/fullchain.pem"
```
**Nguyên nhân:** Let's Encrypt cert chưa được tạo.  
**Sửa:** Chạy certbot (xem [06-ssl-certificates.md](06-ssl-certificates.md)).
