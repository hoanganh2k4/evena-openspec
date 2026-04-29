# Evena — Production Deployment

> Tài liệu này đã được tổ chức lại thành các file riêng biệt.  
> Xem [README.md](README.md) để biết cấu trúc đầy đủ.

---

## Các file tài liệu

| File | Nội dung |
|------|----------|
| [README.md](README.md) | Index tổng quan, thông tin VM, danh sách repos |
| [01-architecture.md](01-architecture.md) | Sơ đồ kiến trúc, lý do chọn công nghệ, chi phí |
| [02-branch-strategy.md](02-branch-strategy.md) | Chiến lược nhánh git (develop / staging) |
| [03-github-actions.md](03-github-actions.md) | CI/CD workflows, GHCR, GitHub Secrets |
| [04-gce-vm-setup.md](04-gce-vm-setup.md) | Khởi tạo VM, Docker, clone code, cấu trúc thư mục |
| [05-cloudflare.md](05-cloudflare.md) | DNS records, SSL mode, kiểm tra |
| [06-ssl-certificates.md](06-ssl-certificates.md) | Cloudflare Origin CA (api) + Let's Encrypt (sse) |
| [07-env-config.md](07-env-config.md) | Toàn bộ biến môi trường .env.prod |
| [08-vercel.md](08-vercel.md) | Deploy frontend, custom domain, env vars, CI/CD |
| [09-nginx-config.md](09-nginx-config.md) | nginx.prod.conf — quyết định thiết kế quan trọng |
| [10-operations.md](10-operations.md) | Deploy hàng ngày, backup, troubleshooting |
