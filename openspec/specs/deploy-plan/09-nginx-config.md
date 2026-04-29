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
