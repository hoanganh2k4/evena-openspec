# 01 — Kiến trúc hệ thống

---

## Sơ đồ tổng thể

```
┌─────────────────────────────────────────────────────────────┐
│                        CLIENT TIER                           │
│                                                              │
│   Browser / Mobile App                                       │
│   evena.id.vn  (Vercel)                                      │
└────────────────────┬──────────────────────┬─────────────────┘
                     │ REST API              │ SSE stream
                     ▼                       ▼
┌────────────────────────────────┐  ┌────────────────────────┐
│     CLOUDFLARE (Proxied)       │  │  CLOUDFLARE (DNS only) │
│     api.evena.id.vn            │  │  sse.evena.id.vn       │
│     Orange cloud ☁️            │  │  Grey cloud ⛅          │
│     Full SSL (Origin CA cert)  │  │  Kết nối thẳng vào VM  │
└──────────────┬─────────────────┘  └──────────┬─────────────┘
               │ HTTPS :443                     │ HTTPS :443
               └──────────────┬─────────────────┘
                              ▼
┌─────────────────────────────────────────────────────────────┐
│         GCE VM — e2-medium — asia-southeast1-b               │
│         IP: 34.2.18.161                                      │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │              Nginx Gateway (:80, :443)                │   │
│  │  :80  → redirect 301 HTTPS                           │   │
│  │  api.evena.id.vn /api/*      → backend:8080          │   │
│  │  api.evena.id.vn /storage/*  → minio:9000            │   │
│  │  api.evena.id.vn /swagger-ui → backend:8080          │   │
│  │  sse.evena.id.vn /sse/*      → sse:8000              │   │
│  └───────────┬──────────────────────┬────────────────────┘   │
│              │                      │                        │
│    ┌─────────▼──────┐     ┌─────────▼──────┐                │
│    │  Backend        │     │  SSE Service   │               │
│    │  Spring Boot    │     │  FastAPI       │               │
│    │  :8080          │     │  :8000         │               │
│    └──┬──────────────┘     └────────────────┘               │
│       │                                                      │
│  ┌────┼───────────────────────────────────┐                 │
│  │    │  Internal Docker network          │                 │
│  │  ┌─▼──────┐  ┌──────────┐  ┌────────┐ │                 │
│  │  │Postgres│  │  MinIO   │  │ Redis  │ │                 │
│  │  │ :5432  │  │:9000/9001│  │ :6379  │ │                 │
│  │  └────────┘  └──────────┘  └────────┘ │                 │
│  └───────────────────────────────────────┘                 │
└─────────────────────────────────────────────────────────────┘
```

---

## Luồng request

### REST API
```
Browser → Cloudflare (proxy) → nginx :443 → /api/* → backend:8080
```

### SSE Streaming
```
Browser → Cloudflare (DNS only, không proxy) → nginx :443 → /sse/* → sse:8000
```
> **Tại sao DNS only cho SSE?** Cloudflare proxy có timeout **100 giây**. SSE cần kết nối sống hàng giờ. DNS only = browser kết nối thẳng đến VM, không qua Cloudflare proxy.

### File storage
```
Browser → Cloudflare (proxy) → nginx :443 → /storage/* → minio:9000
```
Backend sinh URL dạng `https://api.evena.id.vn/storage/event-images/photo.jpg`

---

## Quyết định công nghệ

| Component | Platform | Lý do |
|-----------|----------|-------|
| Frontend | **Vercel** | Zero-config Next.js, CDN toàn cầu, free tier |
| Backend | **GCE VM + Docker** | Persistent disk, long-running process, full control |
| SSE Service | **GCE VM + Docker** | Long-lived connections không phù hợp Cloud Run/serverless |
| PostgreSQL | **Docker trên VM** | Thesis project — Cloud SQL ~$7/tháng là không cần thiết |
| Redis | **Docker trên VM** | SSE state, session — Memorystore ~$16/tháng quá đắt |
| MinIO | **Docker trên VM** | S3-compatible self-hosted, không cần Google Cloud Storage |
| Gateway | **Nginx trên VM** | Cùng Docker network với backend, zero latency |

### Tại sao không dùng Cloud Run cho SSE?
1. **Cloudflare timeout 100s** — SSE cần kết nối sống nhiều giờ
2. **Stateless** — SSE service dùng `asyncio.Queue` in-memory. Scale 2 instance → event từ A không tới client ở B (cần thêm Redis pub/sub nếu muốn multi-instance)
3. GCE VM đơn giải quyết cả 2 vấn đề

---

## Chi phí ước tính

| Dịch vụ | Spec | Chi phí/tháng |
|---------|------|--------------|
| GCE e2-medium | 2vCPU, 4GB RAM, Singapore | ~$27 |
| Persistent SSD 50GB | Boot disk | ~$8.5 |
| Network egress | ~10GB | ~$1 |
| Vercel | Free tier | $0 |
| Cloudflare | Free tier | $0 |
| **Tổng** | | **~$36.5/tháng** |
