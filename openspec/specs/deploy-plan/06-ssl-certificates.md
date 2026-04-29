# 06 — SSL Certificates

---

## Tổng quan

Evena dùng 2 loại cert cho 2 subdomain khác nhau:

| Subdomain | Cert type | Lý do |
|-----------|-----------|-------|
| `api.evena.id.vn` | Cloudflare Origin CA | Subdomain qua CF Proxy — chỉ CF cần tin tưởng cert này |
| `sse.evena.id.vn` | Let's Encrypt | Subdomain DNS only — browser kết nối trực tiếp, cần cert public CA |

---

## Phần A — Cloudflare Origin Certificate (api subdomain)

### Tạo cert

1. Cloudflare → domain `evena.id.vn` → **SSL/TLS** → **Origin Server**
2. Click **Create Certificate**
3. Cấu hình:
   - Private key type: **RSA (2048)**
   - Hostnames: `api.evena.id.vn` (hoặc `*.evena.id.vn`)
   - Certificate Validity: **15 years**
4. Click **Create**
5. Trang hiện ra 2 text box:
   - **Origin Certificate** → copy, lưu thành `cf.crt`
   - **Private Key** → copy, lưu thành `cf.key`

> **Quan trọng:** Private Key chỉ hiển thị MỘT LẦN. Lưu ngay.

### Copy lên VM

```bash
# Từ máy local, copy 2 file lên VM
scp cf.crt nguyenhuyhoanganh900@34.2.18.161:/home/deploy/evena/gateway/ssl/cf.crt
scp cf.key nguyenhuyhoanganh900@34.2.18.161:/home/deploy/evena/gateway/ssl/cf.key

# Trên VM, set permissions
chmod 600 /home/deploy/evena/gateway/ssl/cf.key
chmod 644 /home/deploy/evena/gateway/ssl/cf.crt
```

Hoặc tạo trực tiếp trên VM:
```bash
# Trên VM
mkdir -p /home/deploy/evena/gateway/ssl
nano /home/deploy/evena/gateway/ssl/cf.crt
# Paste nội dung Origin Certificate

nano /home/deploy/evena/gateway/ssl/cf.key
# Paste nội dung Private Key

chmod 600 /home/deploy/evena/gateway/ssl/cf.key
```

### Nginx sử dụng cert này

```nginx
# nginx.prod.conf — server block cho api.evena.id.vn
ssl_certificate     /etc/nginx/ssl/cf.crt;
ssl_certificate_key /etc/nginx/ssl/cf.key;
```

Trong `docker-compose.prod.yml`, thư mục `ssl` được mount:
```yaml
volumes:
  - ./ssl:/etc/nginx/ssl:ro
```

---

## Phần B — Let's Encrypt Certificate (sse subdomain)

### Yêu cầu
- Port 80 phải đang mở về VM (firewall GCE đã mở)
- nginx chưa chạy khi chạy certbot standalone (hoặc dùng webroot)

### Cài certbot và lấy cert

```bash
# Trên VM
sudo apt-get update
sudo apt-get install -y certbot

# Lấy cert (port 80 phải free — dừng nginx nếu đang chạy)
sudo certbot certonly --standalone \
  -d sse.evena.id.vn \
  --agree-tos \
  --email nguyenhuyhoanganh900@gmail.com \
  --non-interactive
```

Cert được lưu tại:
```
/etc/letsencrypt/live/sse.evena.id.vn/fullchain.pem
/etc/letsencrypt/live/sse.evena.id.vn/privkey.pem
```

### Nginx sử dụng cert này

```nginx
# nginx.prod.conf — server block cho sse.evena.id.vn
ssl_certificate     /etc/letsencrypt/live/sse.evena.id.vn/fullchain.pem;
ssl_certificate_key /etc/letsencrypt/live/sse.evena.id.vn/privkey.pem;
```

Trong `docker-compose.prod.yml`, thư mục letsencrypt được mount:
```yaml
volumes:
  - /etc/letsencrypt:/etc/letsencrypt:ro
```

### Tự động renew cert

Let's Encrypt cert hết hạn sau **90 ngày**. Cài cron để tự động renew:

```bash
# Thêm vào crontab
(crontab -l 2>/dev/null; echo "0 3 * * * certbot renew --quiet && docker exec evena-gateway nginx -s reload") | crontab -

# Kiểm tra crontab
crontab -l
```

Cron này:
1. Chạy lúc 3:00 AM mỗi ngày
2. `certbot renew --quiet` — renew nếu cert sắp hết hạn (< 30 ngày)
3. `nginx -s reload` — reload nginx để dùng cert mới

### Renew thủ công

```bash
sudo certbot renew
docker exec evena-gateway nginx -s reload
```

---

## Kiểm tra cert

```bash
# Kiểm tra cert api (qua CF proxy, hiện CF cert cho browser)
openssl s_client -connect api.evena.id.vn:443 -servername api.evena.id.vn 2>/dev/null | openssl x509 -noout -dates

# Kiểm tra cert sse (kết nối thẳng VM, hiện Let's Encrypt cert)
openssl s_client -connect sse.evena.id.vn:443 -servername sse.evena.id.vn 2>/dev/null | openssl x509 -noout -dates -issuer
# Issuer phải là: Let's Encrypt
```
