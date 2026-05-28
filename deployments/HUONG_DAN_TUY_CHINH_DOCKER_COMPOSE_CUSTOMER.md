# Hướng dẫn tùy chỉnh docker-compose.yml cho triển khai khách hàng

Tài liệu này chỉ ra các thay đổi cần thực hiện trên file `docker-compose.yml` (bản chuẩn ở thư mục gốc) để tạo ra `deployments/docker-compose.customer.yml` — bản tùy chỉnh dùng biến môi trường `CUSTOMER_NAME` để tách biệt dữ liệu, container name, image tag cho từng khách hàng trên cùng một host.

---

## 1. Xoá dòng `version`

Bản chuẩn có dòng đầu:

```yaml
version: "3.8"
```

Xoá dòng này. Docker Compose bản mới đã bỏ version tag.

---

## 2. Sửa `build.context`

File customer nằm trong thư mục `deployments/`, nên context phải trỏ lên một cấp:

- **Gốc:**
  ```yaml
      context: .
  ```
- **Chỉnh lại thành:**
  ```yaml
      context: ..
  ```

---

## 3. Thêm `image` cho service `app`

Thêm image tag vào service `app`, dưới `dockerfile:`:

```yaml
    image: 'zalocrm_${CUSTOMER_NAME:?err}_app:latest'
```

Dùng `${CUSTOMER_NAME}` để tạo image riêng cho mỗi khách hàng, phục vụ Jenkins pipeline (giữ bản `:previous` cho rollback).

---

## 4. Tham số hoá container name

Tất cả container name đều thêm hậu tố `-${CUSTOMER_NAME}` để tránh xung đột khi chạy nhiều khách hàng trên cùng host:

| Service | Gốc | Chỉnh lại thành |
|-|-|-|
| app | `zalo-crm-app` | `zalo-crm-${CUSTOMER_NAME:?err}-app` |
| db | `zalo-crm-db` | `zalo-crm-${CUSTOMER_NAME:?err}-db` |
| redis | `zalo-crm-redis` | `zalo-crm-${CUSTOMER_NAME:?err}-redis` |
| minio | `zalo-crm-minio` | `zalo-crm-${CUSTOMER_NAME:?err}-minio` |
| minio-init | `zalo-crm-minio-init` | `zalo-crm-${CUSTOMER_NAME:?err}-minio-init` |
| backup | `zalo-crm-backup` | `zalo-crm-${CUSTOMER_NAME:?err}-backup` |

---

## 5. Tham số hoá cổng host

Các cổng trong service `app`, `db`, `redis`, `minio` đổi từ giá trị cứng sang biến môi trường có fallback:

- **app:** `"3080:3000"` → `'${APP_PORT:-3080}:3000'`
- **db:** `"127.0.0.1:5433:5432"` → `'127.0.0.1:${DB_PORT:-5432}:5432'`
- **redis:** `"127.0.0.1:6379:6379"` → `'127.0.0.1:${REDIS_PORT:-6379}:6379'`
- **minio:** `"9000:9000"` → `'${MINIO_API_PORT:-9000}:9000'`, `"127.0.0.1:9001:9001"` → `'127.0.0.1:${MINIO_CONSOLE_PORT:-9001}:9001'`

---

## 6. Đổi `env_file` sang đường dẫn tuyệt đối trên host

- **Gốc:**
  ```yaml
  env_file:
    - .env
  ```
- **Chỉnh lại thành:**
  ```yaml
  env_file:
    - /home/ZaloCRM_${CUSTOMER_NAME:?err}/.env
  ```

Lý do: workspace của Jenkins có thể bị xoá giữa các build. Trỏ thẳng vào file trên host đảm bảo container luôn lấy đúng secret, không lệ thuộc thư mục chạy `docker compose`.

---

## 7. Chuyển volumes từ **named volumes** sang **bind-mounts tuyệt đối**

Thay vì dùng named volumes do Docker quản lý, mount thẳng ra thư mục `/home/ZaloCRM_${CUSTOMER_NAME}/...` để dễ backup, kiểm tra và di chuyển.

### Service `app`

- **Gốc:**
  ```yaml
  volumes:
    - file_storage:/var/lib/zalo-crm/files
  ```
- **Chỉnh lại thành:**
  ```yaml
  volumes:
    - '/home/ZaloCRM_${CUSTOMER_NAME:?err}/files:/var/lib/zalo-crm/files'
  ```

### Service `db`

- **Gốc:**
  ```yaml
  volumes:
    - pg_data:/var/lib/postgresql/data
  ```
- **Chỉnh lại thành:**
  ```yaml
  volumes:
    - '/home/ZaloCRM_${CUSTOMER_NAME:?err}/pg_data:/var/lib/postgresql/data'
  ```

### Service `redis`

- **Gốc:**
  ```yaml
  volumes:
    - redis_data:/data
  ```
- **Chỉnh lại thành:**
  ```yaml
  volumes:
    - '/home/ZaloCRM_${CUSTOMER_NAME:?err}/redis_data:/data'
  ```

### Service `minio`

- **Gốc:**
  ```yaml
  volumes:
    - minio_data:/data
  ```
- **Chỉnh lại thành:**
  ```yaml
  volumes:
    - '/home/ZaloCRM_${CUSTOMER_NAME:?err}/minio_data:/data'
  ```

### Service `backup`

- **Gốc:**
  ```yaml
  volumes:
    - ./backups:/backups
  ```
- **Chỉnh lại thành:**
  ```yaml
  volumes:
    - '/home/ZaloCRM_${CUSTOMER_NAME:?err}/backups:/backups'
  ```

---

## 8. Bỏ khai báo `volumes:` ở cuối file

Bản gốc có 4 dòng cuối:

```yaml
volumes:
  pg_data:
  file_storage:
  redis_data:
  minio_data:
```

Khi đã bind-mount ra thư mục host, bỏ hẳn khối này.

---

## 9. Các thay đổi nhỏ khác

### MinIO environment — fallback mềm

Bản gốc dùng `${MINIO_ROOT_USER:?err}` (hard-fail nếu thiếu). Bản customer dùng fallback mềm:

```yaml
MINIO_ROOT_USER: '${MINIO_ROOT_USER:-minioadmin}'
MINIO_ROOT_PASSWORD: '${MINIO_ROOT_PASSWORD:-minioadmin}'
```

Giúp Jenkins pipeline không fail cứng nếu thiếu biến trong `.env` (vẫn khuyến cáo đặt giá trị thật).

### Thêm `security_opt` quoting

Bản gốc:
```yaml
    security_opt:
      - no-new-privileges:true
```
Bản customer:
```yaml
    security_opt:
      - 'no-new-privileges:true'
```

(Chỉ khác về quoting YAML — tương đương về mặt chức năng.)

---

## Tóm tắt (docker-compose)

| Thành phần | Gốc (`docker-compose.yml`) | Chỉnh lại thành (`docker-compose.customer.yml`) |
|-|-|-|
| `version:` đầu file | `"3.8"` | Không cần nữa |
| `build.context` | `.` | `..` |
| `image:` trong service `app` | Không có | `zalocrm_${CUSTOMER_NAME}_app:latest` |
| Container names | `zalo-crm-{svc}` | `zalo-crm-${CUSTOMER_NAME}-{svc}` |
| Host ports | Giá trị cứng (`3080`, `5433`, ...) | Biến môi trường (`${APP_PORT:-3080}`, ...) |
| `env_file` của `app` | `.env` | `/home/ZaloCRM_${CUSTOMER_NAME}/.env` |
| App file storage | `file_storage:/var/lib/...` | `/home/ZaloCRM_${CUSTOMER_NAME}/files:/var/lib/...` |
| PostgreSQL data | `pg_data:/var/lib/postgresql/...` | `/home/ZaloCRM_${CUSTOMER_NAME}/pg_data:/var/lib/...` |
| Redis data | `redis_data:/data` | `/home/ZaloCRM_${CUSTOMER_NAME}/redis_data:/data` |
| MinIO data | `minio_data:/data` | `/home/ZaloCRM_${CUSTOMER_NAME}/minio_data:/data` |
| Backup output | `./backups:/backups` | `/home/ZaloCRM_${CUSTOMER_NAME}/backups:/backups` |
| Khối `volumes:` cuối file | Có (4 named volumes) | Không cần nữa |
| MinIO env fallback | `:?err` (hard-fail) | `:-minioadmin` (soft fallback) |

---

## Chuẩn bị host trước khi chạy

Lần đầu tiên triển khai cho một khách hàng, tạo thư mục gốc và file `.env` trên host:

```bash
CUSTOMER=HOSCO
sudo mkdir -p /home/ZaloCRM_${CUSTOMER}/{pg_data,minio_data,redis_data,files,backups}
sudo cp .env.example /home/ZaloCRM_${CUSTOMER}/.env
sudo nano /home/ZaloCRM_${CUSTOMER}/.env   # điền JWT_SECRET, ENCRYPTION_KEY, DB_PASSWORD, MINIO_ROOT_PASSWORD
```

Bốn secret bắt buộc:

- `JWT_SECRET` (≥32 ký tự)
- `ENCRYPTION_KEY` (≥32 ký tự)
- `DB_PASSWORD`
- `MINIO_ROOT_PASSWORD`

Nếu thiếu, Fastify sẽ throw khi boot ở `NODE_ENV=production` (xem `backend/src/config/index.ts`).

> **Lưu ý:** `CUSTOMER_NAME` phải là chữ thường, không dấu cách (Docker không chấp nhận chữ hoa trong tên image). Jenkins pipeline đã có bước validate tự động.

---

## Chạy app

Sau khi sửa biến trong `/home/ZaloCRM_${CUSTOMER}/.env`:

```bash
# Không có Redis
CUSTOMER_NAME=hosco docker compose \
  --env-file /home/ZaloCRM_hosco/.env \
  -f deployments/docker-compose.customer.yml up -d

# Có Redis
CUSTOMER_NAME=hosco docker compose \
  --env-file /home/ZaloCRM_hosco/.env \
  -f deployments/docker-compose.customer.yml \
  --profile redis up -d
```

> **Quan trọng:** Phải có `--env-file /home/ZaloCRM_hosco/.env`. Directive `env_file:` trong service chỉ inject biến vào container đang chạy, **không** dùng cho việc thay thế `${DB_PASSWORD}` ngay trong file compose. Thiếu cờ này thì Postgres sẽ boot fail với lỗi `superuser password is not specified`.

Trên môi trường production thật, nên dùng Jenkins pipeline thay vì chạy tay — xem `Jenkinsfile.customer`.
