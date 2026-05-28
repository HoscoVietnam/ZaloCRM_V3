# Hướng dẫn deploy nhiều khách hàng trên cùng server

Tài liệu mô tả cách chạy nhiều instance ZaloCRM độc lập (mỗi khách hàng một instance) trên cùng một máy chủ Linux, sử dụng `docker-compose.customer.yml` và `Jenkinsfile.customer`.

---

## Tổng quan kiến trúc

Mỗi khách hàng được tách hoàn toàn:

```
Server
├── /home/ZaloCRM_develop/              ← dữ liệu customer "develop"
│   ├── .env                            ← secrets + config
│   ├── pg_data/  minio_data/  files/  backups/
│   └── redis_data/
│
├── /home/ZaloCRM_customera/            ← dữ liệu customer "customera"
│   └── ...
```

| Thành phần | Isolation |
|------------|-----------|
| **Docker network** | Mỗi customer có network riêng (tên compose project `zalocrm_<customer>_default`) |
| **Docker image** | `zalocrm_<customer>_app:latest` riêng → rollback A không ảnh hưởng B |
| **Container name** | đặt theo pattern `zalo-crm-<customer>-app`, `zalo-crm-<customer>-db`, ... |
| **Volume/dữ liệu** | Bind-mount vào `/home/ZaloCRM_<customer>/` |
| **Port** | Mỗi customer dùng bộ port riêng biệt (Jenkins params) |
| **Jenkins job** | Dùng chung `Jenkinsfile.customer` — tham số `CUSTOMER_NAME` phân biệt instance |

---

## Bảng phân bổ port (gợi ý)

| # | `CUSTOMER_NAME` | `APP_PORT` | `DB_PORT` | `MINIO_API_PORT` | `MINIO_CONSOLE_PORT` | `REDIS_PORT` |
|---|----------------|-----------|-----------|-----------------|---------------------|-------------|
| 0 | *(default)* | 3080      | 5432      | 9000            | 9001                | 6379        |
| 1 | customera      | **3081**  | **5433**  | **9002**        | **9003**            | **6380**    |
| 2 | customerb      | 3082      | 5434      | 9004            | 9005                | 6381        |
| 3 | customerc      | 3083      | 5435      | 9006            | 9007                | 6382        |
| 4 | customerd      | 3084      | 5436      | 9008            | 9009                | 6383        |

> `APP_PORT` là port user/reverse-proxy truy cập trực tiếp.  
> `DB_PORT`, `MINIO_CONSOLE_PORT`, `REDIS_PORT` chỉ bind `127.0.0.1` — không expose ra ngoài.  
> `MINIO_API_PORT` cần browser/Zalo CDN truy cập file đính kèm — cần mở firewall nếu không dùng reverse proxy.

Trước khi gán port, kiểm tra port chưa bị dùng:

```bash
ss -tlnp | grep -E '3081|5433|9002|9003'
```

---

## Thiết lập customer mới (lần đầu)

### Bước 1 — Tạo thư mục và file `.env` trên server

Chọn `CUSTOMER_NAME`, **viết thường, không dấu cách** (VD: `hosco`, `nhasachxyz`, `develop`).

```bash
CUSTOMER=hosco

# Tạo thư mục dữ liệu (cần sudo vì /home/ thuộc root)
sudo mkdir -p /home/ZaloCRM_${CUSTOMER}/{pg_data,minio_data,redis_data,files,backups}
sudo chown -R jenkins:jenkins /home/ZaloCRM_${CUSTOMER}

# Copy .env từ instance HOSCO làm base, sau đó sửa
sudo cp /home/ZaloCRM/.env /home/ZaloCRM_${CUSTOMER}/.env
sudo nano /home/ZaloCRM_${CUSTOMER}/.env
```

> **Lưu ý quyền**: Jenkins user phải có quyền ghi vào thư mục này. 
> Nếu không thể `chown`, có thể đổi `HOST_DIR` trong Jenkinsfile sang đường dẫn khác (VD `/opt/zalocrm/${CUSTOMER_NAME}`).

### Bước 2 — Nội dung `.env` cho customer

File `.env` giống `.env.example`. **Không** cần thêm biến `CUSTOMER_NAME` hay port vào `.env` — các biến đó được truyền qua Jenkins params và env vars của Docker Compose.

```env
# === SECRETS bắt buộc (phải thay thế toàn bộ) ===
JWT_SECRET=<openssl rand -hex 32>
ENCRYPTION_KEY=<openssl rand -hex 32>
DB_PASSWORD=<mật khẩu Postgres riêng>
MINIO_ROOT_PASSWORD=<mật khẩu MinIO riêng>

# === CÁC BIẾN KHÁC ===
NODE_ENV=production
APP_URL=https://<domain-của-customer>

DB_USER=crmuser
DB_NAME=zalocrm
DATABASE_URL=postgresql://crmuser:<DB_PASSWORD>@db:5432/zalocrm

UPLOAD_DIR=/var/lib/zalo-crm/files
S3_ENDPOINT=http://minio:9000
S3_PUBLIC_URL=http://<server-ip>:<MINIO_API_PORT>   # port application
S3_BUCKET=zalocrm-attachments
S3_REGION=us-east-1
S3_ACCESS_KEY=minioadmin
S3_SECRET_KEY=<MINIO_ROOT_PASSWORD>

MINIO_ROOT_USER=minioadmin
MINIO_ROOT_PASSWORD=<mật khẩu MinIO riêng>

# Bật Redis nếu cần
# REDIS_URL=redis://redis:6379
```

> **Quan trọng**: Mỗi customer phải dùng `JWT_SECRET`, `ENCRYPTION_KEY`, `DB_PASSWORD`, `MINIO_ROOT_PASSWORD` **khác nhau** để tránh rủi ro bảo mật.

### Bước 3 — Tạo Jenkins Pipeline job

1. Jenkins → **New Item** → đặt tên `ZaloCRM-CustomerA` → chọn **Pipeline**.
2. Tab **Pipeline**:
   - **Definition**: `Pipeline script from SCM`
   - **SCM**: Git, URL repo
   - **Branch**: nhánh chứa `Jenkinsfile.customer`
   - **Script Path**: `deployments/Jenkinsfile.customer`
3. Lưu job.

### Bước 4 — Chạy BUILD lần đầu

Jenkins job `ZaloCRM-CustomerA` → **Build with Parameters** → điền:

| Parameter | Giá trị |
|-----------|---------|
| `CUSTOMER_NAME` | `customera` |
| `DEPLOY_MODE` | `BUILD` |
| `APP_PORT` | `3081` |
| `DB_PORT` | `5434` |
| `REDIS_PORT` | `6380` |
| `MINIO_API_PORT` | `9002` |
| `MINIO_CONSOLE_PORT` | `9003` |

→ Build.

Pipeline sẽ:
1. Tạo thư mục `/home/ZaloCRM_customera/` (nếu chưa có).
2. Kiểm tra `.env` có đủ 4 secret.
3. Build image `zalocrm_customera_app:latest`.
4. `docker compose up -d` với project name mặc định là `customera`.
5. Health-check `http://127.0.0.1:3081`.

---

## Cấu hình reverse proxy (nginx/Caddy)

Sau khi stack chạy, cấu hình reverse proxy để map domain → port:

### Nginx

```nginx
server {
    server_name crm-customera.example.com;

    location / {
        proxy_pass         http://127.0.0.1:3081;
        proxy_http_version 1.1;
        proxy_set_header   Upgrade $http_upgrade;
        proxy_set_header   Connection "upgrade";
        proxy_set_header   Host $host;
        proxy_set_header   X-Real-IP $remote_addr;
    }
}
```

### Caddy

```caddyfile
crm-customera.example.com {
    reverse_proxy 127.0.0.1:3081
}
```

---

## Rollback

Khi bản mới của customera có vấn đề:

1. Jenkins job `ZaloCRM-CustomerA` → **Build with Parameters**.
2. Chọn `DEPLOY_MODE=ROLLBACK` (các tham số khác giữ nguyên).
3. Pipeline sẽ tag `zalocrm_customera_app:previous` → `:latest` và restart.

Kiểm tra image đang chạy:

```bash
docker inspect zalo-crm-customera-app --format '{{.Image}}'
```

> Lưu ý: ROLLBACK chỉ quay về **1 bản** kế trước. Muốn giữ nhiều phiên bản lịch sử:
> ```bash
> docker tag zalocrm_customera_app:previous zalocrm_customera_app:archive-$(date +%Y%m%d)
> ```

---

## Kiểm tra trạng thái tất cả customer

Xem toàn bộ container của tất cả instance đang chạy:

```bash
docker ps --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}' | grep zalo-crm
```

Xem toàn bộ image theo customer:

```bash
docker images | grep zalocrm
```

---

## Backup & restore thủ công

Backup pre-deploy do pipeline tạo nằm ở `/home/ZaloCRM_customera/backups/zalocrm_<timestamp>.sql.gz`.

Restore:

```bash
gunzip -c /home/ZaloCRM_customera/backups/zalocrm_<timestamp>.sql.gz \
  | docker exec -i zalo-crm-customera-db psql -U crmuser -d zalocrm
```

Service `backup` trong compose cũng tự dump `@daily` vào cùng thư mục, giữ 7 ngày / 4 tuần / 3 tháng.

---

## Checklist khi thêm customer mới

- [ ] Chọn `CUSTOMER_NAME` chưa tồn tại trên server (`ls /home/ | grep ZaloCRM_`)
- [ ] Kiểm tra port chưa bị dùng (`ss -tlnp | grep <port>`)
- [ ] Tạo `.env` với secrets riêng (không copy nguyên xi secrets từ customer khác)
- [ ] `S3_PUBLIC_URL` trong `.env` trỏ đúng `MINIO_API_PORT` của customer này
- [ ] `DATABASE_URL` trong `.env` chứa đúng `DB_PASSWORD`
- [ ] Đã đảm bảo Jenkins user có quyền ghi vào `/home/ZaloCRM_<customer>/`
- [ ] Jenkins job tạo xong và chạy BUILD thành công
- [ ] Reverse proxy đã map domain → `APP_PORT`

---

## Liên quan

- `deployments/docker-compose.customer.yml` — template compose dùng biến `${CUSTOMER_NAME}`, `${APP_PORT}`, ...
- `deployments/Jenkinsfile.customer` — template Jenkinsfile nhận `CUSTOMER_NAME` + các port params.
