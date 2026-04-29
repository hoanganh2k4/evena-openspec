# 03 — GitHub Actions CI/CD

---

## Tổng quan luồng CI/CD

```
push → staging
       ├── gateway repo  → build nginx image  → push GHCR → SSH VM → docker pull + up
       ├── backend repo  → build Spring Boot  → push GHCR → SSH VM → docker pull + up
       ├── sse-service   → build FastAPI       → push GHCR → SSH VM → docker pull + up
       └── frontend repo → Vercel deploy (không qua GHCR)
```

**Chiến lược: Cách B — Build trên CI, VM chỉ pull image**
- GitHub Actions build Docker image, push lên GHCR
- VM SSH vào, chạy `docker compose pull` + `docker compose up`
- VM không cần RAM/CPU để build (tiết kiệm tài nguyên e2-medium)

---

## Image Registry — GHCR

| Service | Image |
|---------|-------|
| gateway | `ghcr.io/event-management-bkproject/gateway:staging` |
| backend | `ghcr.io/event-management-bkproject/backend:staging` |
| sse-service | `ghcr.io/hoanganh2k4/sse-service:staging` |

> SSE service nằm ở org khác (`hoanganh2k4`) vì repo cá nhân. VM login GHCR bằng Personal Access Token của `hoanganh2k4` có quyền đọc cả 2 org.

---

## Bước 1 — Tạo nhánh staging

Làm một lần cho mỗi repo (gateway, backend, sse-service, frontend).

### Cách A — Qua GitHub UI

1. Vào repo trên github.com (ví dụ `Event-management-bkproject/backend`)
2. Click dropdown **main** (góc trên bên trái, gần tên repo)
3. Gõ `staging` vào ô search
4. Click **"Create branch: staging from 'main'"**
5. Làm tương tự cho `develop`

### Cách B — Qua Terminal (khuyến nghị)

```bash
# Clone repo về (nếu chưa có)
git clone https://github.com/Event-management-bkproject/backend.git
cd backend

# Tạo và push nhánh staging
git checkout -b staging
git push origin staging

# Tạo và push nhánh develop
git checkout -b develop
git push origin develop
```

---

## Bước 2 — Tạo file workflow trong từng repo

### 2.1 gateway repo

Tạo file `.github/workflows/deploy.yml` trong repo gateway:

```yaml
name: Build & Deploy Gateway → GCE
on:
  push:
    branches: [staging]

env:
  IMAGE_NAME: ghcr.io/event-management-bkproject/gateway

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-buildx-action@v3
      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.IMAGE_NAME }}
          tags: |
            type=ref,event=branch
            type=sha,prefix=,suffix=,format=short
      - uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  deploy:
    runs-on: ubuntu-latest
    needs: build-and-push
    steps:
      - uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.GCE_HOST }}
          username: ${{ secrets.GCE_USER }}
          key: ${{ secrets.GCE_SSH_KEY }}
          script: |
            set -e
            cd /home/deploy/evena/gateway
            git pull origin staging
            docker compose -f docker-compose.prod.yml --env-file .env.prod pull
            docker compose -f docker-compose.prod.yml --env-file .env.prod up -d --remove-orphans
            docker compose -f docker-compose.prod.yml --env-file .env.prod ps
```

### 2.2 backend repo

```yaml
name: Build & Deploy Backend → GCE
on:
  push:
    branches: [staging]

env:
  IMAGE_NAME: ghcr.io/event-management-bkproject/backend

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-buildx-action@v3
      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.IMAGE_NAME }}
          tags: |
            type=ref,event=branch
            type=sha,prefix=,suffix=,format=short
      - uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  deploy:
    runs-on: ubuntu-latest
    needs: build-and-push
    steps:
      - uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.GCE_HOST }}
          username: ${{ secrets.GCE_USER }}
          key: ${{ secrets.GCE_SSH_KEY }}
          script: |
            set -e
            cd /home/deploy/evena/gateway
            docker compose -f docker-compose.prod.yml --env-file .env.prod pull backend
            docker compose -f docker-compose.prod.yml --env-file .env.prod up -d --no-deps backend
```

`--no-deps` = giữ nguyên postgres/redis/minio đang chạy, chỉ restart backend.

### 2.3 sse-service repo

```yaml
name: Build & Deploy SSE Service → GCE
on:
  push:
    branches: [staging]

env:
  IMAGE_NAME: ghcr.io/hoanganh2k4/sse-service

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-buildx-action@v3
      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.IMAGE_NAME }}
          tags: |
            type=ref,event=branch
            type=sha,prefix=,suffix=,format=short
      - uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  deploy:
    runs-on: ubuntu-latest
    needs: build-and-push
    steps:
      - uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.GCE_HOST }}
          username: ${{ secrets.GCE_USER }}
          key: ${{ secrets.GCE_SSH_KEY }}
          script: |
            set -e
            cd /home/deploy/evena/gateway
            docker compose -f docker-compose.prod.yml --env-file .env.prod pull sse
            docker compose -f docker-compose.prod.yml --env-file .env.prod up -d --no-deps sse
```

### 2.4 frontend repo

```yaml
name: Deploy Frontend → Vercel
on:
  push:
    branches: [staging]
  pull_request:
    branches: [staging]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: amondnet/vercel-action@v25
        with:
          vercel-token: ${{ secrets.VERCEL_TOKEN }}
          vercel-org-id: ${{ secrets.VERCEL_ORG_ID }}
          vercel-project-id: ${{ secrets.VERCEL_PROJECT_ID }}
          # push staging → production deploy
          # PR → preview deploy
          vercel-args: ${{ github.ref == 'refs/heads/staging' && '--prod' || '' }}
```

---

## Bước 3 — Thêm Secrets vào GitHub (từng repo)

Phải làm cho **mỗi repo riêng biệt** (secrets không chia sẻ giữa repos).

### Thêm secret step by step

1. Vào repo trên github.com (ví dụ `Event-management-bkproject/backend`)
2. Click tab **Settings** (thanh ngang trên cùng của repo — cần quyền admin)
3. Sidebar trái → **Secrets and variables** → **Actions**
4. Click nút **New repository secret**
5. Điền **Name** và **Secret**, click **Add secret**
6. Lặp lại cho từng secret

### Secrets cho gateway, backend, sse-service (3 repo này giống nhau)

| Secret | Giá trị | Ghi chú |
|--------|---------|---------|
| `GCE_HOST` | `34.2.18.161` | External IP của VM |
| `GCE_USER` | `nguyenhuyhoanganh900` | Username login VM |
| `GCE_SSH_KEY` | `-----BEGIN OPENSSH PRIVATE KEY-----...` | Toàn bộ nội dung private key |

### Secrets cho frontend repo

| Secret | Cách lấy |
|--------|----------|
| `VERCEL_TOKEN` | Xem [08-vercel.md](08-vercel.md) → Bước 4 |
| `VERCEL_ORG_ID` | Xem [08-vercel.md](08-vercel.md) → Bước 4 |
| `VERCEL_PROJECT_ID` | Xem [08-vercel.md](08-vercel.md) → Bước 4 |

---

## Bước 4 — Tạo SSH Key cho GitHub Actions

Chạy trên VM (SSH vào VM trước):

```bash
# Tạo key riêng cho CI/CD (không dùng key cá nhân)
ssh-keygen -t ed25519 -f ~/.ssh/github_actions -C "github-actions-deploy" -N ""

# Thêm public key vào authorized_keys
cat ~/.ssh/github_actions.pub >> ~/.ssh/authorized_keys

# Xem private key để copy vào GitHub
cat ~/.ssh/github_actions
```

Copy toàn bộ output (gồm cả dòng `-----BEGIN OPENSSH PRIVATE KEY-----` và `-----END OPENSSH PRIVATE KEY-----`), dán vào secret `GCE_SSH_KEY`.

> **Lưu ý khi paste:** Không được có khoảng trắng thừa ở đầu/cuối. SSH key bị format sai → lỗi `SSH authentication failed`.

---

## Bước 5 — Cấu hình GHCR Package Visibility

Sau lần build đầu tiên, image GHCR có thể ở trạng thái **private**.  
Nếu VM gặp lỗi pull image, làm theo:

### Chuyển package sang public (không cần login)

1. GitHub → profile icon (góc trên phải) → **Your profile**
2. Tab **Packages**
3. Click vào package (ví dụ `backend`)
4. **Package settings** → **Danger Zone** → **Change visibility** → **Public**

### Hoặc: Login GHCR trên VM (giữ private)

```bash
# Tạo PAT trước: GitHub → Settings → Developer settings → Personal access tokens
# → Fine-grained tokens → Generate new token
# Scopes cần: read:packages

# Login trên VM
docker login ghcr.io -u hoanganh2k4 --password-stdin
# Nhập PAT khi được hỏi (paste rồi Enter)

# Kiểm tra
docker pull ghcr.io/event-management-bkproject/backend:staging
```

---

## Bước 6 — Xem kết quả chạy Actions

1. Vào repo trên github.com
2. Click tab **Actions** (thanh ngang trên cùng)
3. Sidebar trái hiển thị danh sách workflows
4. Click vào workflow run gần nhất để xem logs
5. Click từng step để xem chi tiết output

### Trigger thủ công (nếu cần test)

1. Actions → chọn workflow
2. Click **Run workflow** (nút bên phải)
3. Chọn branch `staging` → **Run workflow**

---

## Lỗi thường gặp

### "missing server host"
**Nguyên nhân:** Secret `GCE_HOST` hoặc `GCE_USER` không tồn tại trong repo.  
**Sửa:** Settings → Secrets and variables → Actions → thêm đủ 3 secrets.

### "SSH authentication failed"
**Nguyên nhân:** `GCE_SSH_KEY` bị format sai khi paste (thừa newline, thiếu header/footer).  
**Sửa:** Copy lại toàn bộ private key, paste cẩn thận vào GitHub secret.

### "denied: permission_denied"
**Nguyên nhân:** Image private, VM chưa login GHCR.  
**Sửa:** Chạy `docker login ghcr.io` trên VM với PAT có quyền `read:packages`.

### Build chạy nhưng không deploy
**Nguyên nhân:** Workflow file không ở đúng nhánh `staging`.  
**Sửa:** Đảm bảo file `.github/workflows/deploy.yml` đã được push lên nhánh `staging`.
