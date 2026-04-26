# Evena — Production Deployment Plan

> **Date:** 2026-04-27
> **Author:** Evena Team
> **Status:** Ready for staging → production

---

## 1. Tổng quan kiến trúc

```
┌─────────────────────────────────────────────────────────┐
│                     CLIENT TIER                          │
│                                                          │
│  Frontend (web)              Mobile App (Android/iOS)    │
│  Vercel                      EAS Build → APK/IPA         │
│  evena.vercel.app            Play Store / App Store       │
└──────────────────┬────────────────────┬──────────────────┘
                   │                    │
                   ▼                    ▼
┌──────────────────────────────────────────────────────────┐
│                    CLOUDFLARE LAYER                       │
│                                                          │
│   api.evena.com  ←→  Proxy (Full SSL)                    │
│   sse.evena.com  ←→  DNS-only (bypass proxy timeout)     │
└─────────────────────────┬────────────────────────────────┘
                          │  HTTPS
                          ▼
┌──────────────────────────────────────────────────────────┐
│              GCE VM — asia-southeast1 (Singapore)         │
│              e2-medium · 2vCPU · 4GB RAM · 50GB SSD      │
│                                                          │
│  ┌──────────────────────────────────────────────────┐   │
│  │             Gateway (Nginx)                       │   │
│  │  :80  → redirect HTTPS                           │   │
│  │  :443 → api.evena.com → /api/* → backend:8080    │   │
│  │                       → /storage/* → minio:9000  │   │
│  │  :443 → sse.evena.com → /sse/*   → sse:8000      │   │
│  └──────────────┬───────────────────┬───────────────┘   │
│                 │                   │                    │
│         ┌───────▼────────┐  ┌───────▼──────┐            │
│         │ Backend        │  │ SSE Service  │            │
│         │ Spring Boot    │  │ FastAPI      │            │
│         │ :8080          │  │ :8000        │            │
│         └───┬────────────┘  └──────────────┘            │
│             │                      │                    │
│    ┌────────┼──────────────────────┤                    │
│    │        │                      │                    │
│  ┌─▼──────┐ │  ┌─────────┐  ┌─────▼────┐              │
│  │Postgres│ │  │  MinIO  │  │  Redis   │              │
│  │  :5432 │ │  │  :9000  │  │  :6379   │              │
│  └────────┘ │  └─────────┘  └──────────┘              │
│             └──────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────┘
```

---

## 2. Quyết định công nghệ

| Component | Platform | Lý do |
|-----------|----------|-------|
| Frontend (web) | **Vercel** | Zero-config Next.js deploy, CDN tự động, free tier |
| Mobile app | **EAS (Expo)** | Build APK/AAB/IPA không cần Mac, CI/CD tích hợp |
| Backend (Spring Boot) | **GCE VM + Docker** | Cần persistent disk, long-running process |
| SSE Service (FastAPI) | **GCE VM + Docker** | Long-lived connections (giờ), không phù hợp Cloud Run |
| Gateway (Nginx) | **GCE VM + Docker** | Cùng VM với backend, giảm latency |
| PostgreSQL | **GCE VM + Docker** | Thesis project, đủ cho scale hiện tại |
| Redis | **GCE VM + Docker** | Session/SSE state, cùng VM |
| MinIO | **GCE VM + Docker** | S3-compatible, self-hosted |

### Tại sao không dùng Cloud Run cho SSE?

Cloud Run có **max request timeout 3600s** nhưng Cloudflare proxy timeout là **100 giây** — connection sẽ bị kill sau 100s. Ngoài ra, Cloud Run là stateless; SSE service dùng `asyncio.Queue` in-memory, nếu scale thành 2 instance thì event từ instance A không tới client ở instance B (dù đã có Redis, việc broadcast cross-instance cần implement thêm logic). GCE VM đơn giải quyết cả 2 vấn đề này.

### Tại sao không dùng Cloud SQL / Memorystore?

Thesis project. Cloud SQL bắt đầu từ ~$7/tháng, Memorystore từ ~$16/tháng. Docker Postgres + Redis trên VM tốn $0 thêm, đủ cho quy mô hiện tại.

---

## 3. Vấn đề SSE + Cloudflare (Critical)

**Vấn đề:** Cloudflare free/pro proxy timeout = **100 giây**. SSE cần kết nối sống hàng giờ.

**Giải pháp:** Dùng 2 subdomain với chế độ Cloudflare khác nhau:

| Subdomain | Cloudflare mode | SSL cert | Timeout |
|-----------|----------------|----------|---------|
| `api.evena.com` | Orange cloud (Proxy) | CF Origin CA | 100s (chỉ REST API, không ảnh hưởng) |
| `sse.evena.com` | Grey cloud (DNS only) | Let's Encrypt | Không giới hạn (kết nối trực tiếp VM) |

Frontend/mobile kết nối SSE qua `sse.evena.com` — không đi qua Cloudflare proxy.

---

## 4. Cấu trúc file trong các repo

### Repo: `gateway` (nhánh `staging`)

```
gateway/
├── Dockerfile                 # Nginx image (không đổi)
├── nginx.conf                 # Dev config (giữ nguyên)
├── nginx.prod.conf            # ← MỚI: Production config
│                              #   - HTTPS cho api.evena.com + sse.evena.com
│                              #   - Xóa frontend upstream (frontend → Vercel)
│                              #   - Thêm /storage/ proxy → MinIO
├── docker-compose.yml         # Dev standalone (giữ nguyên)
├── docker-compose.prod.yml    # ← MỚI: Production compose
│                              #   - Gộp: gateway + backend + sse + postgres + redis + minio
│                              #   - restart: always
│                              #   - named volumes, health checks
│                              #   - build context: ../backend, ../sse-service
├── .env.prod.example          # ← MỚI: Template env vars (commit được)
├── .env.prod                  # KHÔNG commit — điền real secrets trên GCE
├── .gitignore                 # ← MỚI: exclude .env.prod
└── deploy.sh                  # ← MỚI: git pull + docker compose up
```

### Repo: `sse-service` (nhánh `staging`)

```
sse-service/
└── main.py   # ← SỬA: CORS đọc ALLOWED_ORIGINS env var
```

### Repo: `backend` (nhánh `staging`)

Không cần sửa code. Các env vars được inject qua `docker-compose.prod.yml`:

| Env var | Giá trị production |
|---------|-------------------|
| `SPRING_DATASOURCE_URL` | `jdbc:postgresql://postgres:5432/${DB_NAME}` |
| `APP_STORAGE_ENDPOINT` | `http://minio:9000` |
| `APP_STORAGE_PUBLIC_URL` | `https://api.evena.com/storage` |
| `APP_SSE_URL` | `http://sse:8000` |
| `SECURE_COOKIE` | `true` |
| `SPRING_JPA_SHOW_SQL` | `false` |

### Repo: `frontend` (nhánh `staging` → deploy Vercel)

Cần set trong Vercel dashboard → Settings → Environment Variables:

| Key | Value |
|-----|-------|
| `NEXT_PUBLIC_API_URL` | `https://api.evena.com` |
| `NEXT_PUBLIC_SSE_URL` | `https://sse.evena.com` |

---

## 5. Các bước triển khai đầy đủ

### Bước 1 — Tạo GCE VM

```bash
# Tạo VM qua gcloud CLI
gcloud compute instances create evena-prod \
  --zone=asia-southeast1-b \
  --machine-type=e2-medium \
  --image-family=debian-12 \
  --image-project=debian-cloud \
  --boot-disk-size=50GB \
  --boot-disk-type=pd-ssd \
  --tags=http-server,https-server

# Mở firewall
gcloud compute firewall-rules create allow-http-https \
  --allow=tcp:80,tcp:443 \
  --target-tags=http-server,https-server
```

> **GCP Free Tier note:** `e2-micro` miễn phí nhưng không đủ RAM cho Spring Boot (~512MB). Dùng `e2-medium` (~$27/tháng) tại Singapore.

### Bước 2 — Cài Docker trên VM

```bash
# SSH vào VM
gcloud compute ssh evena-prod --zone=asia-southeast1-b

# Cài Docker Engine
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER
newgrp docker

# Kiểm tra
docker --version          # Docker 27.x
docker compose version    # Docker Compose v2.x
```

### Bước 3 — Clone code lên VM

```bash
mkdir -p /home/deploy/evena
cd /home/deploy/evena

# Clone từng repo, nhánh staging
git clone -b staging https://github.com/Event-management-bkproject/gateway.git gateway
git clone -b staging https://github.com/Event-management-bkproject/backend.git backend
git clone -b staging https://github.com/YOUR_USERNAME/sse-service.git sse-service
```

**Cấu trúc thư mục trên VM sau khi clone:**

```
/home/deploy/evena/
├── gateway/          ← chứa docker-compose.prod.yml, nginx.prod.conf, deploy.sh
├── backend/          ← Spring Boot source
└── sse-service/      ← FastAPI source
```

### Bước 4 — Cấu hình DNS trên Cloudflare

1. Đăng nhập Cloudflare → chọn domain → DNS → Add records:

| Type | Name | IPv4 address | Proxy status |
|------|------|--------------|-------------|
| A | `api` | `GCE_VM_EXTERNAL_IP` | **Proxied** (Orange ☁️) |
| A | `sse` | `GCE_VM_EXTERNAL_IP` | **DNS only** (Grey ⛅) |

2. SSL/TLS → Overview → chọn **Full (strict)**

### Bước 5 — Tạo Cloudflare Origin Certificate

1. Cloudflare dashboard → SSL/TLS → **Origin Server** → Create Certificate
2. Chọn RSA 2048, validity **15 years**
3. Hostnames: `api.evena.com`
4. Download `certificate.pem` (→ `cf.crt`) và `private-key.pem` (→ `cf.key`)
5. Copy lên VM:

```bash
mkdir -p /home/deploy/evena/gateway/ssl

# Từ máy local, copy lên VM:
scp cf.crt USERNAME@GCE_IP:/home/deploy/evena/gateway/ssl/cf.crt
scp cf.key USERNAME@GCE_IP:/home/deploy/evena/gateway/ssl/cf.key

# Trên VM, set permissions:
chmod 600 /home/deploy/evena/gateway/ssl/cf.key
```

### Bước 6 — Cài Let's Encrypt cho sse.evena.com

```bash
# Cài certbot trên VM (ngoài Docker)
sudo apt-get install -y certbot

# Obtain cert (cần port 80 đang mở về VM, chưa chạy Docker gateway)
sudo certbot certonly --standalone -d sse.evena.com \
  --agree-tos --email YOUR_EMAIL --non-interactive

# Cert sẽ ở: /etc/letsencrypt/live/sse.evena.com/fullchain.pem
# docker-compose.prod.yml đã mount: /etc/letsencrypt → /etc/letsencrypt:ro

# Cron tự động renew cert + reload nginx
(crontab -l 2>/dev/null; echo "0 3 * * * certbot renew --quiet && docker exec evena-gateway nginx -s reload") | crontab -
```

### Bước 7 — Tạo .env.prod

```bash
cd /home/deploy/evena/gateway

cp .env.prod.example .env.prod
nano .env.prod   # điền tất cả giá trị thực

# Sinh JWT_SECRET (≥ 64 chars):
openssl rand -base64 64

# Sinh QR_SECRET (≥ 32 chars):
openssl rand -base64 32
```

**Danh sách secrets cần điền:**

| Biến | Ví dụ / Nguồn |
|------|--------------|
| `DB_PASSWORD` | Strong password, ví dụ: `openssl rand -hex 16` |
| `REDIS_PASSWORD` | Strong password |
| `MINIO_PASSWORD` | Strong password |
| `JWT_SECRET` | `openssl rand -base64 64` |
| `QR_SECRET` | `openssl rand -base64 32` |
| `ADMIN_PASS` | Password admin đăng nhập web |
| `MAIL_PASSWORD` | Gmail App Password (bật 2FA → App Passwords) |
| `PROD_MOMO_*` | Keys từ MoMo Business portal |
| `PROD_VNPAY_*` | Keys từ VNPay merchant portal |
| `ALLOWED_ORIGINS` | `https://evena.vercel.app,https://evena.com` |
| `FRONTEND_URL` | `https://evena.vercel.app` |

### Bước 8 — Deploy lần đầu

```bash
cd /home/deploy/evena/gateway

# Start tất cả services
docker compose -f docker-compose.prod.yml --env-file .env.prod up -d --build

# Xem tiến trình build (5-10 phút cho Spring Boot)
docker compose -f docker-compose.prod.yml logs -f

# Kiểm tra health
docker compose -f docker-compose.prod.yml ps
```

### Bước 9 — Kiểm tra sau deploy

```bash
# Gateway health
curl https://api.evena.com/health

# Backend Spring Boot health
curl https://api.evena.com/api/actuator/health

# SSE service health
curl https://sse.evena.com/health

# Test SSE streaming (giữ kết nối, nhận heartbeat mỗi 30s)
curl -N "https://sse.evena.com/sse/api/events/subscribe?channel=public"
```

### Bước 10 — Cấu hình Vercel

1. Vào [vercel.com](https://vercel.com) → Import project → chọn repo `frontend`
2. Settings → Environment Variables → chọn **Production**:
   - `NEXT_PUBLIC_API_URL` = `https://api.evena.com`
   - `NEXT_PUBLIC_SSE_URL` = `https://sse.evena.com`
3. Deploy → kiểm tra build log

### Bước 11 — EAS Mobile Build

```bash
cd /path/to/mobile_app
npm install -g @expo/eas-cli
eas login

# Cấu hình EAS
eas build:configure
```

Thêm vào `eas.json`:

```json
{
  "build": {
    "production": {
      "env": {
        "EXPO_PUBLIC_API_URL": "https://api.evena.com",
        "EXPO_PUBLIC_SSE_URL": "https://sse.evena.com"
      }
    },
    "preview": {
      "distribution": "internal",
      "env": {
        "EXPO_PUBLIC_API_URL": "https://api.evena.com",
        "EXPO_PUBLIC_SSE_URL": "https://sse.evena.com"
      }
    }
  }
}
```

```bash
# Build APK (Android internal testing)
eas build --platform android --profile preview

# Build AAB (Google Play Store)
eas build --platform android --profile production

# Build IPA (App Store) — cần Apple Developer account
eas build --platform ios --profile production
```

---

## 6. Script deploy hàng ngày

Sau lần đầu setup, mọi lần deploy tiếp theo:

```bash
cd /home/deploy/evena/gateway
./deploy.sh
```

Script tự động:
1. `git pull origin staging` cho gateway, backend, sse-service
2. `docker compose up -d --build --remove-orphans`
3. Hiển thị trạng thái tất cả containers

---

## 7. MinIO — public file access

Backend sinh URL dạng: `https://api.evena.com/storage/event-images/photo.jpg`

Nginx proxy `location /storage/` → `http://minio:9000/` (internal Docker network).
Buckets đã set `anonymous download` (trong minio-init) nên không cần auth khi đọc.

**Quan trọng:** `.env.prod` phải có `APP_STORAGE_PUBLIC_URL=https://api.evena.com/storage`

MinIO admin console (port 9001) **không** expose public. Truy cập qua SSH tunnel:

```bash
# Từ máy local
ssh -L 9001:localhost:9001 USERNAME@GCE_VM_IP
# Mở browser: http://localhost:9001
```

---

## 8. Backup và bảo trì

### Backup PostgreSQL (cron hàng ngày)

```bash
mkdir -p /home/deploy/backups

# Thêm vào crontab
(crontab -l 2>/dev/null; echo "0 2 * * * docker exec evena-postgres pg_dump -U evena_user eventdb | gzip > /home/deploy/backups/db_\$(date +\%Y\%m\%d).sql.gz && find /home/deploy/backups -name 'db_*.sql.gz' -mtime +7 -delete") | crontab -
```

### Restore PostgreSQL

```bash
gunzip -c /home/deploy/backups/db_20260427.sql.gz | \
  docker exec -i evena-postgres psql -U evena_user eventdb
```

### Restart một service không downtime

```bash
docker compose -f docker-compose.prod.yml restart backend
```

---

## 9. Chi phí ước tính

| Dịch vụ | Spec | Chi phí/tháng |
|---------|------|--------------|
| GCE e2-medium | 2vCPU, 4GB RAM, Singapore | ~$27 |
| Persistent SSD 50GB | Boot disk | ~$8.5 |
| Network egress | ~10GB/tháng | ~$1 |
| **Tổng GCP** | | **~$36.5** |
| Vercel (frontend) | Free tier | $0 |
| EAS Build | Free (30 builds/tháng) | $0 |
| Cloudflare | Free tier | $0 |
| **Tổng tất cả** | | **~$36.5/tháng** |

---

## 10. Checklist go-live

### Infrastructure
- [ ] GCE VM (`e2-medium`, `asia-southeast1`) đã tạo
- [ ] Docker + Docker Compose installed trên VM
- [ ] Firewall rules: port 80, 443 mở

### DNS / SSL
- [ ] Cloudflare: `api.evena.com` → Proxied (Orange ☁️)
- [ ] Cloudflare: `sse.evena.com` → DNS only (Grey ⛅)
- [ ] Cloudflare SSL mode: **Full (strict)**
- [ ] Cloudflare Origin Certificate: `gateway/ssl/cf.crt` + `cf.key`
- [ ] Let's Encrypt cert cho `sse.evena.com` obtained
- [ ] Certbot auto-renew cron configured

### Code & Config
- [ ] Code cloned (gateway, backend, sse-service) — nhánh `staging`
- [ ] `.env.prod` tạo và điền đầy đủ secrets
- [ ] `docker compose up -d --build` thành công
- [ ] Tất cả containers trạng thái `healthy`

### Functional Tests
- [ ] `curl https://api.evena.com/health` → `{"status":"healthy"}`
- [ ] `curl https://api.evena.com/api/actuator/health` → `{"status":"UP"}`
- [ ] `curl https://sse.evena.com/health` → `{"status":"healthy"}`
- [ ] SSE streaming test giữ kết nối > 2 phút không bị ngắt
- [ ] Login/Register qua frontend Vercel
- [ ] Upload ảnh → URL trả về `https://api.evena.com/storage/...` load được

### Mobile
- [ ] EAS build APK thành công
- [ ] App kết nối được `api.evena.com`
- [ ] SSE push notification hoạt động

### Operations
- [ ] Backup cron hoạt động
- [ ] MinIO admin accessible qua SSH tunnel
- [ ] Log monitoring: `docker compose logs -f`

---

## 11. Câu hỏi thường gặp

**Q: Frontend Vercel gọi API bị CORS error?**

Kiểm tra `FRONTEND_URL` trong `.env.prod` — phải đúng Vercel domain (kể cả `https://`). Spring Security dùng `FRONTEND_URL` để set CORS policy.

**Q: SSE connection bị ngắt sau ~100s?**

Cloudflare đang proxy `sse.evena.com`. Vào Cloudflare DNS → `sse` record → bỏ proxy (Grey cloud). Kết nối phải đi thẳng đến GCE VM.

**Q: MinIO file URL không load được?**

Kiểm tra `APP_STORAGE_PUBLIC_URL=https://api.evena.com/storage` trong `.env.prod`. Nếu đúng rồi, kiểm tra nginx `location /storage/` trong `nginx.prod.conf`.

**Q: Spring Boot OOM / Crash?**

Thêm JVM memory limit vào `backend/Dockerfile`:
```dockerfile
ENTRYPOINT ["java", "-Xmx1g", "-Xms512m", "-jar", "app.jar"]
```

**Q: Let's Encrypt certificate expired?**

```bash
sudo certbot renew
docker exec evena-gateway nginx -s reload
```
