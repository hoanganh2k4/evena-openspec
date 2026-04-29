# 05 — Cloudflare Setup

---

## Tổng quan

Cloudflare đóng vai trò:
1. **DNS** — trỏ domain về GCE VM
2. **CDN + DDoS protection** — cho `api.evena.id.vn` (Proxied)
3. **SSL termination** — Full SSL mode với Origin Certificate
4. **DNS only** — cho `sse.evena.id.vn` để tránh 100s timeout

---

## Bước 1 — Thêm domain vào Cloudflare

1. Mở trình duyệt, vào [dash.cloudflare.com](https://dash.cloudflare.com)
2. Đăng nhập hoặc tạo tài khoản
3. Trang chủ dashboard → click nút **Add a site** (hoặc **Add a domain**)
4. Nhập domain: `evena.id.vn` → click **Continue**
5. Chọn plan **Free** → click **Continue**
6. Cloudflare tự scan DNS records hiện tại (mất 10–30 giây)
7. Xem lại records đã scan → click **Continue**
8. Cloudflare cấp 2 nameservers (ví dụ: `kai.ns.cloudflare.com`, `lily.ns.cloudflare.com`)
9. **Ghi lại 2 nameservers này** — cần điền vào trang quản lý domain

### Cập nhật Nameserver tại nhà đăng ký domain

> Domain `evena.id.vn` mua ở đâu thì vào đó để đổi nameserver.

**Ví dụ với Inet.vn:**
1. Đăng nhập [inet.vn](https://inet.vn)
2. **Quản lý domain** → chọn `evena.id.vn`
3. Tab **Nameserver** → chọn **Nameserver tùy chỉnh**
4. Điền 2 nameserver của Cloudflare vào ô NS1, NS2
5. **Lưu thay đổi**

> **Thời gian propagate:** thường 5–30 phút, tối đa 48 giờ.  
> Kiểm tra trạng thái tại dashboard Cloudflare — khi đã kích hoạt sẽ hiện **"Active"**.

---

## Bước 2 — Cấu hình DNS Records

1. Vào [dash.cloudflare.com](https://dash.cloudflare.com)
2. Click vào domain `evena.id.vn`
3. Sidebar trái → **DNS** → **Records**
4. Click **Add record** (nút màu xanh)

### Record 1 — api (Proxied)

| Field | Giá trị |
|-------|---------|
| Type | **A** |
| Name | `api` |
| IPv4 address | `34.2.18.161` |
| Proxy status | **Proxied** (bật icon cloud màu cam 🟠) |
| TTL | Auto |

Click **Save**

### Record 2 — sse (DNS only)

| Field | Giá trị |
|-------|---------|
| Type | **A** |
| Name | `sse` |
| IPv4 address | `34.2.18.161` |
| Proxy status | **DNS only** (tắt icon cloud, chuyển sang màu xám ⚫) |
| TTL | Auto |

Click **Save**

### Cách bật/tắt Proxy status

Khi tạo/sửa record, có nút toggle hình cloud:
- Cloud **cam** (🟠 Proxied) = traffic đi qua Cloudflare → áp dụng CDN, DDoS protection
- Cloud **xám** (⚫ DNS only) = Cloudflare chỉ làm DNS, traffic đến thẳng VM

> **Tại sao `sse` phải DNS only?**  
> Cloudflare proxy timeout = **100 giây**. SSE cần kết nối mở hàng giờ.  
> DNS only = browser kết nối thẳng đến IP VM, bỏ qua Cloudflare proxy.

---

## Bước 3 — Cấu hình SSL/TLS Mode

1. Vào [dash.cloudflare.com](https://dash.cloudflare.com) → click domain `evena.id.vn`
2. Sidebar trái → **SSL/TLS** → **Overview**
3. Nhìn phần **"Your SSL/TLS encryption mode"**
4. Click **Configure** (nút bên phải)
5. Chọn **Full (strict)**
6. Click **Save**

| Mode | Ý nghĩa |
|------|---------|
| Off | Không SSL |
| Flexible | CF↔Browser HTTPS, CF↔VM HTTP |
| Full | CF↔Browser HTTPS, CF↔VM HTTPS (cert không cần valid) |
| **Full (strict)** | CF↔Browser HTTPS, CF↔VM HTTPS (cần cert hợp lệ — dùng Origin CA) |

Dùng **Full (strict)** kết hợp với Cloudflare Origin Certificate (xem [06-ssl-certificates.md](06-ssl-certificates.md)).

---

## Bước 4 — Tạo Cloudflare Origin Certificate (cho api.evena.id.vn)

Origin Certificate là cert do Cloudflare cấp, chỉ valid với Cloudflare — dùng cho mode Full strict.

1. Sidebar trái → **SSL/TLS** → **Origin Server**
2. Click **Create Certificate**
3. Cấu hình:
   - Private key type: **RSA (2048)**
   - Hostnames: `evena.id.vn`, `*.evena.id.vn` (wildcard — cover tất cả subdomains)
   - Certificate Validity: **15 years**
4. Click **Create**
5. Trang tiếp theo hiện **Origin Certificate** (PEM) và **Private Key** (PEM)
6. **Copy và lưu ngay** — Private Key chỉ hiện 1 lần, sau không xem lại được
7. Lưu vào 2 file:
   - `cf.crt` — nội dung Origin Certificate
   - `cf.key` — nội dung Private Key

### Upload cert lên VM

```bash
# Copy 2 file lên VM
scp cf.crt nguyenhuyhoanganh900@34.2.18.161:/home/deploy/evena/gateway/ssl/
scp cf.key nguyenhuyhoanganh900@34.2.18.161:/home/deploy/evena/gateway/ssl/

# Trên VM, kiểm tra
ls -la /home/deploy/evena/gateway/ssl/
# cf.crt  cf.key
```

---

## Bước 5 — Kiểm tra

```bash
# api subdomain (qua Cloudflare proxy)
curl https://api.evena.id.vn/health
# Expected: {"status":"healthy","service":"gateway"}

# sse subdomain (DNS only, kết nối thẳng VM)
curl https://sse.evena.id.vn/health
# Expected: {"status":"healthy"}
```

Kiểm tra `sse` đang DNS only (phải trả về IP VM, không phải IP Cloudflare):
```bash
nslookup sse.evena.id.vn
# → 34.2.18.161  ✓

nslookup api.evena.id.vn
# → 104.x.x.x (Cloudflare IP) ✓
```

---

## Lỗi thường gặp

### SSE connection bị ngắt sau ~100s
**Nguyên nhân:** `sse.evena.id.vn` đang ở chế độ Proxied (orange cloud).  
**Sửa:** DNS → record `sse` → click icon cloud → chuyển sang DNS only (grey).

### ERR_SSL_PROTOCOL_ERROR trên api
**Nguyên nhân:** SSL mode đang là "Flexible" trong khi nginx dùng HTTPS.  
**Sửa:** SSL/TLS → Overview → Full (strict).

### 525 SSL Handshake Failed
**Nguyên nhân:** Cert trên VM không hợp lệ với Full strict mode.  
**Sửa:** Kiểm tra `cf.crt` và `cf.key` đã copy đúng vào `/home/deploy/evena/gateway/ssl/` chưa.

### Domain vẫn chưa "Active" sau nhiều giờ
**Nguyên nhân:** Nameserver chưa được cập nhật tại nhà đăng ký domain.  
**Sửa:** Kiểm tra lại nameserver đã đổi đúng chưa — phải là nameserver của Cloudflare, không phải nameserver cũ.
