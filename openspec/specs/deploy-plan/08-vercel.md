# 08 — Vercel Frontend Deploy

---

## Tổng quan

| Thông số | Giá trị |
|---------|---------|
| Platform | Vercel (Hobby / free tier) |
| Framework | Next.js |
| Repo | `Event-management-bkproject/frontend` |
| Deploy branch | `staging` |
| Production URL | https://evena.id.vn |
| Fallback URL | https://evena.vercel.app |

---

## Bước 1 — Tạo tài khoản / Đăng nhập Vercel

1. Mở trình duyệt, vào [vercel.com](https://vercel.com)
2. Click **Sign Up** (nếu chưa có tài khoản) → chọn **Continue with GitHub**
3. Authorize Vercel truy cập GitHub account
4. Sau đăng nhập, vào trang **Dashboard** (trang chủ Vercel)

---

## Bước 2 — Import Project từ GitHub

1. Dashboard Vercel → click **Add New...** → chọn **Project**
2. Phần **Import Git Repository** → tìm repo `frontend`
   - Nếu không thấy → click **Adjust GitHub App Permissions** → cấp quyền cho repo/org
3. Click **Import** bên cạnh repo `Event-management-bkproject/frontend`
4. Vercel tự detect **Next.js** framework
5. **Root Directory:** để mặc định `/` (hoặc chỉ định nếu Next.js nằm trong subfolder)
6. **Chưa cần** thêm env vars ở đây — làm sau ở Settings
7. Click **Deploy**
8. Chờ build hoàn thành (~2-3 phút) — khi xong hiện confetti 🎉

> **Yêu cầu:** Repo phải **Public** để dùng Vercel Hobby (free tier). Nếu private cần Vercel Pro ($20/tháng).

### Nếu repo private — cách chuyển public

1. GitHub → repo `frontend` → **Settings**
2. Scroll xuống **Danger Zone**
3. Click **Change repository visibility** → chọn **Public**
4. Xác nhận bằng cách gõ tên repo → click **I understand...**

---

## Bước 3 — Cấu hình Environment Variables

1. Vercel → click vào project `frontend`
2. Tab **Settings** (thanh ngang trên cùng của project)
3. Sidebar trái → **Environment Variables**
4. Thêm từng biến: điền **Key**, **Value**, chọn environments (Production/Preview/Development)
5. Click **Save** sau mỗi biến

### Biến cần thêm

| Key | Value | Environments |
|-----|-------|-------------|
| `NEXT_PUBLIC_API_URL` | `https://api.evena.id.vn/api` | Production, Preview, Development |
| `NEXT_PUBLIC_SSE_URL` | `https://sse.evena.id.vn` | Production, Preview, Development |
| `API_URL` | `https://api.evena.id.vn/api` | Production, Preview, Development |

> - `NEXT_PUBLIC_*` — dùng ở client-side (browser), được bundle vào JS
> - `API_URL` — dùng ở server-side (Next.js API routes, middleware), không expose cho browser

### Redeploy để áp dụng env vars

Sau khi thêm env vars, deploy cũ không có env mới. Phải redeploy:

1. Vercel → project → tab **Deployments**
2. Tìm deployment mới nhất → click dấu **...** (3 chấm) bên phải
3. Click **Redeploy** → **Redeploy** (confirm)
4. Chờ build xong

---

## Bước 4 — Thêm Custom Domain

1. Vercel → project → **Settings** → **Domains**
2. Ô nhập domain → gõ `evena.id.vn` → click **Add**
3. Vercel hướng dẫn thêm DNS record

### Thêm CNAME vào Cloudflare cho domain chính

1. Vào [dash.cloudflare.com](https://dash.cloudflare.com) → domain `evena.id.vn`
2. **DNS** → **Records** → **Add record**

| Field | Giá trị |
|-------|---------|
| Type | **CNAME** |
| Name | `@` |
| Target | `cname.vercel-dns.com` |
| Proxy status | **Proxied** (orange cloud 🟠) |
| TTL | Auto |

Click **Save**

4. Quay lại Vercel → **Settings** → **Domains** — sau vài phút sẽ hiện ✅ **Valid Configuration**

> **Tại sao quan trọng khi thêm custom domain?**  
> Cookie `SameSite=Lax` chỉ chia sẻ giữa các subdomain cùng **registrable domain**:
> - `evena.id.vn` + `api.evena.id.vn` → cùng domain → cookie hoạt động ✓
> - `evena.vercel.app` + `api.evena.id.vn` → khác domain → cookie bị chặn → login không hoạt động ✗

---

## Bước 5 — Cấu hình Deploy Branch

Mặc định Vercel deploy nhánh **main**. Cần đổi sang `staging`.

1. Vercel → project → **Settings** → **Git**
2. Phần **Production Branch** → đổi từ `main` sang `staging`
3. Click **Save**

Từ đây:
- Push vào `staging` → **Production deploy** (https://evena.id.vn)
- Push vào nhánh khác / mở PR → **Preview deploy** (URL tạm thời)

---

## Bước 6 — Lấy Secrets cho GitHub Actions

Cần 3 giá trị để cấu hình workflow `frontend/.github/workflows/deploy.yml`.

### 6.1 VERCEL_TOKEN

1. Click avatar (góc trên phải) → **Account Settings**
2. Sidebar trái → **Tokens**
3. Click **Create Token**
4. **Token Name:** `github-actions`
5. **Scope:** `Full Account`
6. **Expiration:** `No expiration` (hoặc 1 year)
7. Click **Create Token**
8. **Copy token ngay** — chỉ hiện 1 lần

### 6.2 VERCEL_ORG_ID

**Nếu dùng Personal Account (không có team):**
1. Click avatar → **Account Settings**
2. Tab **General**
3. Phần **Your ID** — copy giá trị (dạng `xxxxxxxxxxxxxxxxxxxxxxxx`)

**Nếu dùng Team:**
1. Chọn team từ dropdown trên cùng
2. **Settings** (của team) → **General**
3. Phần **Team ID** — copy giá trị

### 6.3 VERCEL_PROJECT_ID

1. Vào project `frontend` trên Vercel
2. **Settings** → **General**
3. Scroll xuống phần **Project ID**
4. Copy giá trị (dạng `prj_xxxxxxxxxxxxxxxxxxxxxxxx`)

### 6.4 Thêm vào GitHub Secrets

1. GitHub → repo `frontend` → **Settings** → **Secrets and variables** → **Actions**
2. Click **New repository secret**

| Secret | Giá trị |
|--------|---------|
| `VERCEL_TOKEN` | Token vừa tạo |
| `VERCEL_ORG_ID` | Account ID hoặc Team ID |
| `VERCEL_PROJECT_ID` | Project ID |

---

## Luồng deploy frontend

```
Developer push code → nhánh staging
         ↓
GitHub Actions trigger (.github/workflows/deploy.yml)
         ↓
amondnet/vercel-action --prod
         ↓
Vercel build Next.js
         ↓
Deploy lên CDN Vercel
         ↓
https://evena.id.vn cập nhật (~1-2 phút)
```

---

## Theo dõi Deploy

### Xem logs build

1. Vercel → project → tab **Deployments**
2. Click vào deployment để xem chi tiết
3. Tab **Build Logs** — xem output build step by step

### Rollback về version cũ

1. Vercel → project → **Deployments**
2. Tìm deployment muốn quay lại
3. Click dấu **...** → **Promote to Production**
4. Confirm → deployment cũ được đặt làm production ngay lập tức

---

## Lỗi thường gặp

### "localhost:8080" trong network request
**Nguyên nhân:** Code có URL hardcoded thay vì dùng env var.  
**Nguyên tắc:** Mọi API call phải dùng `process.env.NEXT_PUBLIC_API_URL`, không hardcode URL.

### CORS error khi call API
**Nguyên nhân:** `FRONTEND_URL` trong `.env.prod` trên VM chưa đúng.  
**Sửa:** SSH vào VM → sửa `.env.prod` → restart backend:
```bash
docker compose -f docker-compose.prod.yml --env-file .env.prod up -d --no-deps backend
```

### Build fail: "private repo"
**Nguyên nhân:** Repo đang private, Vercel Hobby không hỗ trợ.  
**Sửa:** GitHub → repo → Settings → General → Danger Zone → Change visibility → Public.

### Cookie không được gửi lên API
**Nguyên nhân:** Frontend chưa có custom domain `evena.id.vn`, đang dùng `evena.vercel.app`.  
**Sửa:** Thêm custom domain vào Vercel (Bước 4) và thêm CNAME vào Cloudflare.

### Vercel không tìm thấy repo khi import
**Nguyên nhân:** Vercel GitHub App chưa được cấp quyền đọc org/repo.  
**Sửa:** Import Project → **Adjust GitHub App Permissions** → cấp quyền cho org `Event-management-bkproject`.

### Environment variables không áp dụng
**Nguyên nhân:** Thêm env sau lần deploy cuối, chưa redeploy.  
**Sửa:** Deployments → deployment mới nhất → **...** → **Redeploy**.
