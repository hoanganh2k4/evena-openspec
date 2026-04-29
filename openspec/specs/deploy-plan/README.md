# Evena — Production Deployment Documentation

> **Cập nhật lần cuối:** 2026-04-29  
> **Trạng thái:** Production đang chạy

---

## Tổng quan

Hệ thống Evena được triển khai trên 2 nền tảng:

| Tầng | Nền tảng | URL |
|------|----------|-----|
| Frontend (Next.js) | Vercel | https://evena.id.vn |
| Backend API | GCE VM + Docker | https://api.evena.id.vn |
| SSE Service | GCE VM + Docker | https://sse.evena.id.vn |
| Object Storage | MinIO trên GCE | https://api.evena.id.vn/storage/ |

---

## Cấu trúc tài liệu

| File | Nội dung |
|------|----------|
| [01-architecture.md](01-architecture.md) | Sơ đồ kiến trúc, lý do chọn công nghệ |
| [02-branch-strategy.md](02-branch-strategy.md) | Chiến lược nhánh git cho tất cả repo |
| [03-github-actions.md](03-github-actions.md) | CI/CD workflows — build image, push GHCR, deploy GCE |
| [04-gce-vm-setup.md](04-gce-vm-setup.md) | Khởi tạo VM, cài Docker, clone code, cấu trúc thư mục |
| [05-cloudflare.md](05-cloudflare.md) | Thêm domain, cấu hình DNS, SSL mode |
| [06-ssl-certificates.md](06-ssl-certificates.md) | Cloudflare Origin CA (api) + Let's Encrypt (sse) |
| [07-env-config.md](07-env-config.md) | Toàn bộ biến môi trường .env.prod trên VM |
| [08-vercel.md](08-vercel.md) | Deploy frontend, custom domain, env vars Vercel |
| [09-nginx-config.md](09-nginx-config.md) | nginx.prod.conf — các quyết định thiết kế quan trọng |
| [10-operations.md](10-operations.md) | Deploy hàng ngày, backup, troubleshooting |

---

## GitHub Repositories

| Repo | Org | Deploy branch |
|------|-----|--------------|
| `gateway` | `Event-management-bkproject` | `staging` |
| `backend` | `Event-management-bkproject` | `staging` |
| `frontend` | `Event-management-bkproject` | `staging` |
| `sse-service` | `hoanganh2k4` | `staging` |

---

## Thông tin VM

| Thông số | Giá trị |
|---------|---------|
| Provider | Google Cloud Compute Engine |
| Zone | asia-southeast1-b (Singapore) |
| Machine type | e2-medium (2 vCPU, 4 GB RAM) |
| Disk | 50 GB SSD |
| External IP | 34.2.18.161 |
| OS user | nguyenhuyhoanganh900 |
| Deploy path | /home/deploy/evena/gateway |
