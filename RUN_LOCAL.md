# RUN_LOCAL.md – Hướng dẫn chạy Lab 04

Tài liệu này giúp người khác clone repo sạch và chạy lại FIT4110 Lab 04 — Docker Packaging service trong Docker container.

---

## Yêu cầu tiên quyết

Trước khi bắt đầu, đảm bảo đã cài:

- **Git**: Để clone repo
- **Docker**: `docker --version` ≥ 24.0
- **Node.js**: `node --version` ≥ 20.x LTS
- **npm**: `npm --version` ≥ 9.x
- **curl hoặc Postman**: Để test endpoint

---

## 5 bước chạy service

### 1️⃣ Clone repo và cài dependencies

```bash
git clone <repo-url>
cd FIT4110_lab04_docker_packaging

# Cài npm packages (Newman, Prism, Spectral)
npm install
```

---

### 2️⃣ Build Docker image

```bash
docker build -t fit4110/iot-ingestion:lab04 .
```

**Kết quả mong đợi:**

```
...
Successfully built <image-id>
Successfully tagged fit4110/iot-ingestion:lab04
```

**Troubleshooting:**

- Nếu lỗi `permission denied`: Thêm `sudo` hoặc add user vào docker group:
  ```bash
  sudo usermod -aG docker $USER
  newgrp docker
  ```
- Nếu lỗi `python requirements not found`: Đảm bảo `requirements.txt` nằm ở root folder

---

### 3️⃣ Run container

Mở **terminal 1** và chạy:

```bash
docker run --rm \
  --name fit4110-iot-lab04 \
  -p 8000:8000 \
  --env-file .env.example \
  fit4110/iot-ingestion:lab04
```

**Kết quả mong đợi:**

```
INFO:     Uvicorn running on http://0.0.0.0:8000 (Press CTRL+C to quit)
```

**Troubleshooting:**

- Nếu port 8000 đã bị dùng: `docker run -p 9000:8000 ...` rồi dùng `http://localhost:9000`
- Nếu container exit ngay lập tức: Kiểm tra logs:
  ```bash
  docker logs fit4110-iot-lab04
  ```

---

### 4️⃣ Kiểm tra health (terminal 2)

```bash
curl http://localhost:8000/health
```

**Kết quả mong đợi:**

```json
{
  "status": "ok",
  "service": "iot-ingestion",
  "version": "0.4.0"
}
```

**Tips:**

- Dùng flag `-i` để xem headers:
  ```bash
  curl -i http://localhost:8000/health
  ```

---

### 5️⃣ Chạy Postman/Newman tests

Tại **terminal 2**, chạy:

```bash
npm run test:local
```

**Kết quả mong đợi:**

```
│ newman │
│ ✓  01_Functional    │ 4 / 4  │
│ ✓  02_Authentication│ 2 / 2  │
│ ✓  03_Negative      │ 3 / 3  │
│ ✓  04_Boundary      │ 2 / 2  │
│ ✓  05_Error Schema  │ 1 / 1  │
...
Pass ✓  │ 12 / 12
```

**Report sinh tại:**

```
reports/newman-lab04-local.xml
reports/newman-lab04-local.html
```

---

## Lệnh nhanh (dùng Makefile)

Thay vì gõ lệnh dài, dùng:

```bash
make build          # Build image
make run            # Chạy container (foreground)
make run-detached   # Chạy container (background)
make health         # Test /health endpoint
make test-docker    # Chạy Newman test
make stop           # Dừng container
make clean-reports  # Xóa reports cũ
```

**Ví dụ:**

```bash
make build && make run-detached && sleep 5 && make health && make test-docker
```

---

## Dừng container

**Option 1:** Nếu chạy với `--rm` (tự xóa):

```bash
# Terminal 1: Ctrl+C
# Container tự xóa
```

**Option 2:** Nếu chạy background:

```bash
docker stop fit4110-iot-lab04
```

**Option 3:** Xóa tất cả containers và images:

```bash
docker stop $(docker ps -q)
docker rmi fit4110/iot-ingestion:lab04
```

---

## Các test case được kiểm tra

Newman collection bao gồm:

| Nhóm | Test | Mục đích |
|------|------|---------|
| **01_Functional** | GET /health | Health check pass |
| | POST /readings | Create reading thành công |
| | GET /readings/latest | Lấy danh sách readings |
| | GET /readings/{id} | Lấy chi tiết reading |
| **02_Authentication** | Missing token | Trả 401 + ProblemDetails |
| | Invalid token | Trả 401 + ProblemDetails |
| **03_Negative** | Empty device_id | Trả 422 validation error |
| | Missing metric | Trả 422 validation error |
| | Invalid timestamp | Trả 422 validation error |
| **04_Boundary** | Temp = -40 | Pass (min boundary) |
| | Temp = 80 | Pass (max boundary) |
| **05_Error Schema** | 422 error | Có `type`, `title`, `status`, `detail` (ProblemDetails) |

---

## Cấu hình tuỳ chỉnh (Environment Variables)

Edit `.env.example` hoặc tạo `.env`:

```bash
APP_HOST=0.0.0.0
APP_PORT=8000
AUTH_TOKEN=local-dev-token
SERVICE_NAME=iot-ingestion
SERVICE_VERSION=0.4.0
ENV=local
```

**Chạy với cấu hình khác:**

   curl -v http://localhost:8000/health
   ```

3. Xem logs:
   ```bash
   docker logs -f fit4110-iot-lab04
   ```

### Permission denied (Linux)

```bash
sudo usermod -aG docker $USER
newgrp docker
```

---

## Evidence (Bằng chứng nộp)

Sau khi chạy thành công, lưu:

- [ ] Screenshot `docker build` output
- [ ] Screenshot `docker run` output
- [ ] Screenshot `curl /health`
- [ ] `reports/newman-lab04-local.html` (Newman report)
- [ ] `reports/newman-lab04-local.xml` (XML report)
- [ ] Image tag:
  ```bash
  docker tag fit4110/iot-ingestion:lab04 ghcr.io/<your-org>/team-iot:v0.1.0-lab04
  docker push ghcr.io/<your-org>/team-iot:v0.1.0-lab04
  ```

---

## Lưu ý

- ✅ `.env.example` được commit, `.env` không
- ✅ Service chạy bằng non-root user `appuser`
- ✅ HEALTHCHECK tích hợp, Docker tự restart nếu fail
- ✅ Multi-stage build: optimize size
- ✅ `.dockerignore`: giảm context size
- ✅ Tất cả biến config từ environment

---

## Hỗ trợ

Nếu gặp vấn đề:

1. Kiểm tra [troubleshooting docs](docs/TROUBLESHOOTING.md)
2. Kiểm tra [docker lab guide](docs/DOCKER_LAB_GUIDE.md)
3. Chạy diagnostic:
   ```bash
   docker version
   docker info
   docker images fit4110
   docker ps -a
   ```

---

**Chúc bạn thành công với Lab 04!** 🎉
