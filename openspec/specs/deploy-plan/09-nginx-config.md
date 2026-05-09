# 09 — Nginx Production Config

File: `gateway/nginx.prod.conf`

---

## Quyết định thiết kế quan trọng

### 1. KHÔNG có CORS headers trong nginx

Spring Security và FastAPI CORSMiddleware tự handle CORS.  
Nếu nginx cũng thêm `Access-Control-Allow-Origin` → **duplicate header** → browser reject toàn bộ request.

```nginx
# SAI — gây duplicate CORS header, browser báo lỗi
location /api/ {
    add_header Access-Control-Allow-Origin $http_origin always;  # ← XÓA
    proxy_pass http://backend_servers;
}

# ĐÚNG — để backend tự trả CORS header
location /api/ {
    proxy_pass http://backend_servers;
}
```

### 2. KHÔNG rewrite path /sse/

SSE service (FastAPI) có endpoint native: `GET /sse/stream/{channel}`  
Nếu nginx rewrite `^/sse/(.*)$ /$1 break` → strip mất prefix `/sse/` → FastAPI nhận `/stream/{channel}` → 404

```nginx
# SAI — strip /sse/ prefix, FastAPI không tìm thấy route
location /sse/ {
    rewrite ^/sse/(.*)$ /$1 break;  # ← XÓA
    proxy_pass http://sse_servers;
}

# ĐÚNG — giữ nguyên path, FastAPI nhận /sse/stream/{channel}
location /sse/ {
    proxy_pass http://sse_servers;
}
```

### 3. SSE cần tắt buffering

```nginx
location /sse/ {
    proxy_buffering off;     # không buffer response
    proxy_cache off;         # không cache
    chunked_transfer_encoding off;
    proxy_set_header Connection '';   # HTTP/1.1 keep-alive
    proxy_read_timeout 3600s;         # giữ connection 1 giờ
}
```

### 4. Swagger UI cần cả 2 paths

Backend dùng `springdoc-openapi`. Cần expose cả 2:

```nginx
location /swagger-ui/ {
    proxy_pass http://backend_servers/swagger-ui/;
}

location /v3/api-docs {
    proxy_pass http://backend_servers;
}

location /api-docs {
    proxy_pass http://backend_servers;
}
```

> `springdoc.api-docs.path=/api-docs` trong `application.properties` (không phải `/v3/api-docs` mặc định).

---

### 5. Cloudflare Real IP — `set_real_ip_from`

Cloudflare proxy che giấu IP thực của client. Nginx nhận `$remote_addr` là IP của Cloudflare server, không phải client.

Cấu hình `set_real_ip_from` + `real_ip_header CF-Connecting-IP` để nginx lấy đúng IP client từ header Cloudflare:

```nginx
set_real_ip_from 173.245.48.0/20;
set_real_ip_from 103.21.244.0/22;
# ... (đầy đủ danh sách Cloudflare IPv4 + IPv6 trong nginx.prod.conf)
real_ip_header CF-Connecting-IP;
real_ip_recursive on;
```

> `sse.evena.id.vn` dùng DNS only (không qua Cloudflare proxy) → `$remote_addr` vẫn là IP thực của client.

---

### 6. Rate Limiting — phân vùng theo loại traffic

Mỗi loại endpoint có zone và limit riêng:

| Zone | Rate | Burst | Áp dụng cho |
|------|------|-------|-------------|
| `api_limit` | 10r/s | — | `/api/*` |
| `storage_limit` | 30r/s | 60 nodelay | `/storage/*` |
| `docs_limit` | 2r/s | 5 nodelay | Swagger, api-docs, `/channels`, `/stats`, `/metrics`, `/docs` |
| `sse_subscribe_limit` | 5r/s | 20 nodelay | `/sse/*`, `/api/events/subscribe` |
| `sse_conn_limit` | 20 conn/IP | — | SSE subscribe endpoints |

```nginx
limit_req_zone $rate_limit_key zone=api_limit:10m rate=10r/s;
limit_req_zone $rate_limit_key zone=storage_limit:10m rate=30r/s;
limit_req_zone $rate_limit_key zone=docs_limit:10m rate=2r/s;
limit_req_zone $rate_limit_key zone=sse_subscribe_limit:10m rate=5r/s;
limit_conn_zone $conn_limit_key zone=sse_conn_limit:10m;
```

Khi vượt limit: HTTP 429 (bởi `limit_req_status 429`).

---

### 7. Geo Whitelist — bypass rate limit cho performance test

IP trong whitelist sẽ dùng `$rate_limit_key = ""` → không vào limit zone:

```nginx
geo $rate_limit_bypass {
    default 0;
    116.108.115.97/32 1;   # ← cập nhật khi public IP thay đổi
}

map $rate_limit_bypass $rate_limit_key {
    0 $binary_remote_addr;
    1 "";   # empty key = không bị rate limit
}
```

> Để thêm/sửa IP whitelist: cập nhật `nginx.prod.conf` (geo block) + `RATE_LIMIT_WHITELIST_IPS` trong `.env.prod` → `./deploy.sh`

---

## Upstreams

```nginx
upstream backend_servers {
    least_conn;
    server backend:8080 weight=1 max_fails=3 fail_timeout=30s;
    keepalive 32;
}

upstream sse_servers {
    server sse:8000 max_fails=3 fail_timeout=30s;
    keepalive 100;   # SSE cần nhiều keepalive connections
}

upstream minio_servers {
    server minio:9000;
    keepalive 16;
}
```

---

## SSL config

```nginx
# api.evena.id.vn — Cloudflare Origin CA cert
ssl_certificate     /etc/nginx/ssl/cf.crt;
ssl_certificate_key /etc/nginx/ssl/cf.key;

# sse.evena.id.vn — Let's Encrypt cert
ssl_certificate     /etc/letsencrypt/live/sse.evena.id.vn/fullchain.pem;
ssl_certificate_key /etc/letsencrypt/live/sse.evena.id.vn/privkey.pem;
```
