# 07 — Cấu hình Environment Variables

---

## File .env.prod trên VM

Vị trí: `/home/deploy/evena/gateway/.env.prod`

File này **KHÔNG được commit** lên git. Template công khai ở `.env.prod.example`.

```bash
# Tạo từ template
cd /home/deploy/evena/gateway
cp .env.prod.example .env.prod
nano .env.prod
```

---

## Danh sách biến đầy đủ

### Domain

```env
API_DOMAIN=api.evena.id.vn
SSE_DOMAIN=sse.evena.id.vn
```

### Database (Cloud PostgreSQL)

> **Quan trọng:** PostgreSQL không còn chạy trong Docker container. DB đã được chuyển sang Cloud managed.
> Container `postgres` trong `docker-compose.prod.yml` đã bị comment out.

```env
# Full JDBC URL đến Cloud PostgreSQL — giá trị này ghi đè spring.datasource.url
SPRING_DATASOURCE_URL=jdbc:postgresql://<cloud-host>:<port>/<db_name>

# Các biến này vẫn cần nếu docker-compose truyền riêng lẻ (dùng cho healthcheck hoặc scripts)
DB_NAME=eventdb
DB_USERNAME=evena_user
DB_PASSWORD=<strong_password>
```

> `SPRING_DATASOURCE_URL` phải chứa full connection string dạng:
> `jdbc:postgresql://HOST:5432/DBNAME?sslmode=require`
>
> Sinh password mạnh: `openssl rand -hex 16`

### Redis

```env
REDIS_PASSWORD=<strong_password>
```

### MinIO (Object Storage)

```env
MINIO_USER=evena_admin
MINIO_PASSWORD=<strong_password>
```

### JWT / Security

```env
JWT_SECRET=<base64_64chars>     # openssl rand -base64 64
QR_SECRET=<base64_32chars>      # openssl rand -base64 32
ADMIN_PASS=<admin_password>     # Password đăng nhập tài khoản admin
```

> `JWT_SECRET` phải ≥ 64 ký tự.  
> `QR_SECRET` phải ≥ 32 ký tự.

### Rate Limiting — Whitelist

```env
# IP hoặc CIDR được bypass rate limit (dùng cho JMeter/performance test).
# Dạng: IP/32 hoặc CIDR, nhiều giá trị cách nhau dấu phẩy.
RATE_LIMIT_WHITELIST_IPS=116.108.115.97/32
```

> Cập nhật giá trị này khi public IP của máy test thay đổi.
> Được inject vào container backend và dùng bởi Spring Boot rate limit filter.

### Frontend URL

```env
FRONTEND_URL=https://evena.id.vn
```

> Dùng bởi Spring Security để set CORS policy. Phải khớp chính xác URL của Vercel frontend (kể cả `https://`, không có `/` cuối).

### SSE CORS

```env
ALLOWED_ORIGINS=https://evena.id.vn,https://evena.vercel.app
```

> Dùng bởi FastAPI CORSMiddleware. Thêm các origin cần cho phép, cách nhau bởi dấu phẩy.

### Mail (SMTP Gmail)

```env
MAIL_HOST=smtp.gmail.com
MAIL_PORT=587
MAIL_USERNAME=tqt12092004@gmail.com
MAIL_PASSWORD=<gmail_app_password>
MAIL_FROM=noreply@evena.id.vn
```

> **Gmail App Password:** Google Account → Security → 2-Step Verification → App passwords → Tạo password 16 ký tự.

### Payment — MoMo

```env
# Sandbox (dev)
DEV_MOMO_ENDPOINT=https://test-payment.momo.vn
MOMO_SANDBOX=true
DEV_PARTNER_CODE=MOMOLRJZ20181206
DEV_ACCESS_KEY=mTCKt9W3eU1m39TW
DEV_SECRET_KEY=SetA5RDnLHvt51AULf51DyauxUo3kDU6

# Production (lấy từ MoMo Business portal)
# PROD_MOMO_ENDPOINT=https://payment.momo.vn/v2/gateway/api
# PROD_PARTNER_CODE=YOUR_CODE
# PROD_ACCESS_KEY=YOUR_KEY
# PROD_SECRET_KEY=YOUR_SECRET
```

### Payment — VNPay

```env
# Sandbox (dev)
DEV_VNPAY_TMN_CODE=DX0PAO58
DEV_VNPAY_HASH_SECRET=OSC4D2DMUDJYYYJAT2854Y4T91TGIA20
DEV_VNPAY_PAYMENT_URL=https://sandbox.vnpayment.vn/paymentv2/vpcpay.html
DEV_VNPAY_QUERY_URL=https://sandbox.vnpayment.vn/merchant_webapi/api/transaction
DEV_VNPAY_REFUND_URL=https://sandbox.vnpayment.vn/merchant_webapi/api/transaction

# Production (lấy từ VNPay merchant portal)
# PROD_VNPAY_TMN_CODE=YOUR_CODE
# PROD_VNPAY_HASH_SECRET=YOUR_SECRET
```

### App Base URL

```env
APP_BASE_URL=https://api.evena.id.vn
```

> Dùng cho MoMo/VNPay callback URL: `https://api.evena.id.vn/api/payment/callback`

---

## Giá trị thực tế đang dùng (production)

| Biến | Ghi chú |
|------|---------|
| `DB_PASSWORD` | evena_redis_2026 (redis), evena_minio_2026 (minio) |
| `FRONTEND_URL` | https://evena.id.vn |
| `ALLOWED_ORIGINS` | https://evena.id.vn |
| `APP_BASE_URL` | https://api.evena.id.vn |

> Passwords thực tế lưu riêng, không ghi vào tài liệu này.

---

## Biến được inject tự động bởi docker-compose.prod.yml

Các biến sau KHÔNG cần trong `.env.prod` vì `docker-compose.prod.yml` đã hard-code:

| Biến | Giá trị cố định |
|------|----------------|
| `SERVER_PORT` | 8080 |
| `SPRING_JPA_SHOW_SQL` | false |
| `SPRING_JPA_HIBERNATE_DDL_AUTO` | update |
| `APP_STORAGE_ENDPOINT` | `http://minio:9000` |
| `APP_STORAGE_PUBLIC_URL` | `https://${API_DOMAIN}/storage` |
| `APP_SSE_URL` | `http://sse:8000` |
| `APP_SSE_ENABLED` | true |
| `SECURE_COOKIE` | true |
| `USE_REDIS` | true |
| `REDIS_URL` | `redis://redis:6379` |

> **Lưu ý:** `SPRING_DATASOURCE_URL` **không còn** được hard-code trong docker-compose.
> Phải đặt giá trị đầy đủ trong `.env.prod` (Cloud PostgreSQL connection string).
