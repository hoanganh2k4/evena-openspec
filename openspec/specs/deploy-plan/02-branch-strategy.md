# 02 — Chiến lược nhánh Git

---

## Quy tắc nhánh

| Nhánh | Mục đích | Trigger deploy |
|-------|----------|---------------|
| `develop` | Phát triển tính năng mới | Không |
| `staging` | Nhánh production deploy | **Có** — push → GitHub Actions |

> **Lưu ý:** Tên nhánh `main` đã đổi thành `develop`. Nhánh `staging` là nhánh deploy duy nhất.

---

## Cấu trúc nhánh cho từng repo

### gateway (Event-management-bkproject/gateway)
```
develop   ← phát triển, thêm nginx rules mới
staging   ← production deploy, GitHub Actions build + push GHCR + SSH vào VM
```

### backend (Event-management-bkproject/backend)
```
develop   ← phát triển Spring Boot
staging   ← production deploy, GitHub Actions build + push GHCR + SSH vào VM
```

### frontend (Event-management-bkproject/frontend)
```
develop   ← phát triển Next.js
staging   ← production deploy, GitHub Actions → Vercel production
```

### sse-service (hoanganh2k4/sse-service)
```
develop   ← phát triển FastAPI
staging   ← production deploy, GitHub Actions build + push GHCR + SSH vào VM
```

---

## Quy trình phát triển

```
1. Làm việc trên nhánh develop (hoặc feature branch)
2. Commit + push lên develop
3. Khi muốn deploy production:
   git checkout staging
   git merge develop
   git push origin staging
4. GitHub Actions tự động chạy
```

---

## Cách đổi tên nhánh main → develop (đã làm)

```bash
# Trên local
git branch -m main develop
git push origin develop
git push origin --delete main

# Đặt develop làm default branch
# GitHub → Settings → Branches → Default branch → develop
```

---

## Visibility repo

Tất cả repo trong `Event-management-bkproject` đặt **Public** để dùng Vercel Hobby (free tier).  
Nếu muốn private: cần Vercel Pro ($20/tháng) hoặc dùng Vercel CLI deploy thủ công.
