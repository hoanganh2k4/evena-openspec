# 10 — Operations

---

## Deploy hàng ngày

### Tự động (qua GitHub Actions)
Push lên nhánh `staging` → CI/CD tự chạy → VM tự pull image mới + restart service.

### Thủ công (từ VM)
```bash
cd /home/deploy/evena/gateway
./deploy.sh
```

### Restart một service không downtime
```bash
cd /home/deploy/evena/gateway

# Chỉ restart backend
docker compose -f docker-compose.prod.yml --env-file .env.prod up -d --no-deps backend

# Chỉ restart sse
docker compose -f docker-compose.prod.yml --env-file .env.prod up -d --no-deps sse

# Chỉ restart gateway (nginx)
docker compose -f docker-compose.prod.yml --env-file .env.prod up -d --no-deps gateway

# Restart toàn bộ
docker compose -f docker-compose.prod.yml --env-file .env.prod up -d --remove-orphans
```

---

## Xem logs

```bash
cd /home/deploy/evena/gateway

# Xem log tất cả services (live)
docker compose -f docker-compose.prod.yml --env-file .env.prod logs -f

# Chỉ xem log backend
docker compose -f docker-compose.prod.yml --env-file .env.prod logs -f backend

# Chỉ xem log gateway (nginx)
docker compose -f docker-compose.prod.yml --env-file .env.prod logs -f gateway

# 100 dòng gần nhất
docker compose -f docker-compose.prod.yml --env-file .env.prod logs --tail=100 backend
```

---

## Kiểm tra health

```bash
# Gateway
curl https://api.evena.id.vn/health

# Backend Spring Boot
curl https://api.evena.id.vn/api/actuator/health

# SSE Service
curl https://sse.evena.id.vn/health

# Test SSE stream (giữ kết nối, nhận heartbeat mỗi 30s)
curl -N "https://sse.evena.id.vn/sse/stream/public"
```

---

## Backup PostgreSQL

### Tự động (cron hàng ngày lúc 2:00 AM)
```bash
mkdir -p /home/deploy/backups

(crontab -l 2>/dev/null; echo "0 2 * * * docker exec evena-postgres pg_dump -U postgres eventdb | gzip > /home/deploy/backups/db_\$(date +\%Y\%m\%d).sql.gz && find /home/deploy/backups -name 'db_*.sql.gz' -mtime +7 -delete") | crontab -
```

### Backup thủ công
```bash
docker exec evena-postgres pg_dump -U postgres eventdb | gzip > /home/deploy/backups/db_manual.sql.gz
```

### Restore
```bash
gunzip -c /home/deploy/backups/db_20260429.sql.gz | \
  docker exec -i evena-postgres psql -U postgres eventdb
```

---

## MinIO Admin Console

Port 9001 không expose public. Truy cập qua SSH tunnel:

```bash
# Từ máy local
ssh -L 9001:localhost:9001 nguyenhuyhoanganh900@34.2.18.161

# Mở browser: http://localhost:9001
# Login: MINIO_USER / MINIO_PASSWORD từ .env.prod
```

---

## Cập nhật .env.prod

```bash
# SSH vào VM
ssh nguyenhuyhoanganh900@34.2.18.161

# Sửa file
nano /home/deploy/evena/gateway/.env.prod

# Restart service bị ảnh hưởng
cd /home/deploy/evena/gateway
docker compose -f docker-compose.prod.yml --env-file .env.prod up -d --no-deps backend
```

---

## Renew Let's Encrypt cert

```bash
# Thủ công
sudo certbot renew
docker exec evena-gateway nginx -s reload

# Kiểm tra cron tự động đã setup chưa
crontab -l | grep certbot
```

---

## Troubleshooting thường gặp

### Gateway Restarting liên tục
```bash
docker logs evena-gateway --tail=50
```
Nguyên nhân phổ biến:
- `cf.crt` hoặc `cf.key` thiếu/sai
- Let's Encrypt cert chưa tồn tại
- `nginx.prod.conf` syntax error

### Backend không start (OOM)
```bash
docker stats evena-backend
```
Nếu RAM > 3.5GB → thêm JVM limit vào `backend/Dockerfile`:
```dockerfile
ENTRYPOINT ["java", "-Xmx1g", "-Xms512m", "-jar", "app.jar"]
```

### 500 errors từ backend
```bash
docker compose -f docker-compose.prod.yml --env-file .env.prod logs backend | grep ERROR
```

### SSE connection bị ngắt sớm
- Kiểm tra Cloudflare: `sse.evena.id.vn` phải là DNS only (Grey)
- Kiểm tra nginx timeout: `proxy_read_timeout 3600s` trong `/sse/` location

### CORS error trên frontend
- Kiểm tra `FRONTEND_URL` trong `.env.prod` = `https://evena.id.vn`
- Không được có `/` ở cuối
- Restart backend sau khi sửa

### Cookie không gửi lên API
- Frontend phải dùng domain `evena.id.vn` (không phải `evena.vercel.app`)
- Cả frontend và API phải cùng registrable domain (`evena.id.vn`)
