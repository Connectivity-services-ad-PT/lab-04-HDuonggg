# FIT4110_lab04_docker_packaging

**Học phần:** FIT4110 – Dịch vụ kết nối và Công nghệ nền tảng  
**Buổi 4:** Đóng gói service với Docker & tư duy công nghệ nền tảng  
**Case study:** Smart Campus Operations Platform  
**Repo nền:** `FIT4110_lab03_postman_mock_testing`

> Lab 03 đã có OpenAPI contract, Postman Collection, Mock Server và Newman report.  
> Lab 04 dùng lại logic đó để kiểm tra một điều mới: **service có chạy ổn khi được đóng gói thành Docker container không?**

---

## 1. Ý tưởng nối tiếp từ Lab 03 sang Lab 04

Ở Lab 03, luồng làm việc là:

```text
OpenAPI Contract → Mock Server → Postman Test → Newman Report → CI Evidence
```

Ở Lab 04, luồng đó được mở rộng thành:

```text
OpenAPI Contract
→ Service thật
→ Dockerfile
→ Docker Image
→ Docker Container
→ Postman/Newman chạy lại trên container
→ Evidence
```

Lab 04 hiện đã đồng bộ lại với contract IoT của Lab 03 theo payload:

```json
{
  "device_id": "ESP32-LAB-A01",
  "metric": "temperature",
  "value": 31.5,
  "unit": "celsius",
  "timestamp": "2026-05-13T08:30:00+07:00"
}
```

Boundary dùng trong bài:

```text
temperature: -40 đến 80
```

Thông điệp chính của buổi học:

> Một API pass Postman trên máy cá nhân chưa đủ.  
> Service cần được đóng gói thành container để người khác có thể chạy lại nhất quán.

---

## 2. Mục tiêu sau buổi lab

Sau khi hoàn thành Lab 04, mỗi nhóm cần làm được:

- Viết được `Dockerfile` cho service của nhóm.
- Dùng `.dockerignore` để giảm context build.
- Tách cấu hình runtime qua `.env.example`.
- Không commit secret thật vào repo.
- Chạy app bằng user non-root trong container.
- Có `HEALTHCHECK` gọi `GET /health`.
- Build được Docker image.
- Run được container từ image.
- Chạy lại Postman Collection của Lab 03 trên container.
- Kiểm tra được functional, auth, negative, boundary và schema lỗi `ProblemDetails`.
- Xuất Newman report làm bằng chứng.
- Viết được `RUN_LOCAL.md` hướng dẫn người khác chạy lại trong 3–5 bước.

---

## 3. Cấu trúc repo

```text
FIT4110_lab04_docker_packaging/
├── README.md
├── RUN_LOCAL.md
├── Dockerfile
├── .dockerignore
├── .env.example
├── .gitignore
├── Makefile
├── package.json
├── requirements.txt
├── src/
│   └── iot_app/
│       ├── __init__.py
│       └── main.py
├── contracts/
│   └── iot-ingestion.openapi.yaml
├── postman/
│   ├── collections/
│   │   └── FIT4110_lab04_iot_docker.postman_collection.json
│   └── environments/
│       ├── FIT4110_lab04_mock.postman_environment.json
│       └── FIT4110_lab04_local.postman_environment.json
├── mock-data/
├── scripts/
├── docs/
├── checklists/
├── templates/
├── reports/
└── .github/
    └── workflows/
        └── docker-newman.yml
```

---

## 4. Chuẩn bị môi trường

Cần cài trước:

- Git
- Docker Desktop hoặc Docker Engine
- Node.js 20.x LTS
- npm
- Postman Desktop hoặc Postman Web

Cài dependencies phục vụ Prism, Spectral, Newman:

```bash
npm install
```

Kiểm tra:

```bash
docker --version
docker info
node --version
npx newman --version
npx prism --version
```

---

## 5. Chạy service local không dùng Docker

Cài Python dependencies:

```bash
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

Chạy API:

```bash
uvicorn iot_app.main:app --app-dir src --host 0.0.0.0 --port 8000
```

Kiểm tra:

```bash
curl http://localhost:8000/health
```

---

## 6. Build và chạy bằng Docker

Build image:

```bash
docker build -t fit4110/iot-ingestion:lab04 .
```

Run container:

```bash
docker run --rm \
  --name fit4110-iot-lab04 \
  -p 8000:8000 \
  --env-file .env.example \
  fit4110/iot-ingestion:lab04
```

Kiểm tra health:

```bash
curl http://localhost:8000/health
```

---

## 7. Chạy lại Postman Collection trên container

Chạy Newman với local environment:

```bash
npm run test:local
```

Hoặc dùng script:

```bash
bash scripts/run-newman.sh local
```

Report được sinh trong:

```text
reports/
```

---

## 8. Các lệnh nhanh bằng Makefile

```bash
make install
make lint
make mock
make test-mock
make build
make run
make test-docker
make stop
```

---

## 9. Bài làm của từng nhóm

Mỗi nhóm dùng repo này làm mẫu, sau đó thay phần IoT bằng service của mình.

| Nhóm | Cần thay đổi |
|---|---|
| `team-iot` | Có thể dùng mẫu này trực tiếp, mở rộng thêm endpoint từ Lab 03 |
| `team-camera` | Thay `src/` bằng Camera Stream service, thêm OpenCV headless |
| `team-gate` | Thay bằng Access Gate service, lưu ý biến môi trường DB |
| `team-vision` | Thay bằng AI Vision service, chuẩn bị model YOLOv8n hoặc mock model |
| `team-analytics` | Thay bằng Analytics service, chưa bắt buộc TimescaleDB trong Lab 04 |
| `team-core` | Thay bằng Core Business policy engine |
| `team-notify` | Thay bằng Notification service, không commit token thật |

---

## 10. Điều kiện hoàn thành Lab 04

Một nhóm được xem là hoàn thành khi:

- `Dockerfile` build được image.
- Image chạy được container.
- Container có `GET /health` trả `200`.
- Service chạy bằng non-root user.
- Có `.dockerignore`.
- Có `.env.example`.
- Có `RUN_LOCAL.md`.
- Chạy lại Postman/Newman pass trên container.
- Có test cho functional, auth, negative, boundary.
- Error response trả đúng dạng `ProblemDetails`.
- Có report trong `reports/`.
- Có bằng chứng image tag đúng quy ước.

Tag gợi ý:

```text
v0.1.0-<team>
```

Ví dụ:

```bash
docker tag fit4110/iot-ingestion:lab04 ghcr.io/<owner>/team-iot:v0.1.0-team-iot
```

---

## 11. Artefact cần nộp

```text
Dockerfile
.dockerignore
.env.example
RUN_LOCAL.md
contracts/<team>.openapi.yaml
postman/collections/<team>.postman_collection.json
postman/environments/<team>_local.postman_environment.json
reports/newman-lab04-local.xml
reports/newman-lab04-local.html
ảnh chụp /health hoặc log container
tag image đã push lên registry
```

---

## 12. Rubric gợi ý

| Tiêu chí | Điểm |
|---|---:|
| Dockerfile đúng, build được | 2.0 |
| Container chạy được và `/health` pass | 2.0 |
| Non-root, `.dockerignore`, `.env.example` tốt | 2.0 |
| Newman/Postman test pass trên container | 2.0 |
| RUN_LOCAL.md rõ ràng, người khác chạy lại được | 1.0 |
| Evidence đầy đủ: log/report/image tag | 1.0 |
| **Tổng** | **10.0** |

---

## 13. Tinh thần của buổi học

Sau Buổi 3, nhóm đã chứng minh:

```text
API đúng contract khi kiểm thử bằng Postman/Newman.
```

Sau Buổi 4, nhóm cần chứng minh thêm:

```text
API đó có thể được đóng gói, chạy lại và kiểm thử trong container.
```

Đây là bước đệm trực tiếp cho Buổi 5:

```text
Docker container đơn lẻ → Docker Compose nhiều service → Plug-a-thon.
```

---

## 14. Tài liệu Hỗ Trợ

### 14.1 Hướng dẫn nhanh

- 🚀 **[RUN_LOCAL.md](RUN_LOCAL.md)** – Hướng dẫn chạy trong 5 bước + troubleshooting
- 📚 **[docs/DOCKER_LAB_GUIDE.md](docs/DOCKER_LAB_GUIDE.md)** – Docker concepts, best practices, debugging
- 🔧 **[docs/TROUBLESHOOTING.md](docs/TROUBLESHOOTING.md)** – Lỗi thường gặp & giải pháp
- ✅ **[checklists/docker_readiness_checklist.md](checklists/docker_readiness_checklist.md)** – Pre-submission checklist
- ✅ **[checklists/submission_checklist.md](checklists/submission_checklist.md)** – Bằng chứng cần nộp

### 14.2 Tham khảo ngoài

- [Docker Official Docs](https://docs.docker.com/)
- [Dockerfile Reference](https://docs.docker.com/engine/reference/builder/)
- [FastAPI + Docker](https://fastapi.tiangolo.com/deployment/docker/)
- [OpenAPI 3.1.0 Spec](https://spec.openapis.org/oas/v3.1.0)
- [RFC 7807 Problem Details](https://tools.ietf.org/html/rfc7807)

---

## 15. Cheat Sheet – Lệnh thường dùng

```bash
# Setup
npm install
docker --version

# Build
docker build -t fit4110/iot:lab04 .
docker build --no-cache -t fit4110/iot:lab04 .  # Force rebuild

# Run
docker run --rm -p 8000:8000 --env-file .env.example fit4110/iot:lab04
docker run -d --name fit4110-iot-lab04 ...      # Background

# Debug
docker logs fit4110-iot-lab04
docker logs -f fit4110-iot-lab04                # Follow
docker exec -it fit4110-iot-lab04 /bin/bash     # Shell
docker exec fit4110-iot-lab04 curl http://localhost:8000/health

# Test
npm run test:local
npm run test:mock
npm run mock:iot

# Inspect
docker ps
docker images
docker history fit4110/iot:lab04
docker inspect fit4110/iot:lab04

# Cleanup
docker stop fit4110-iot-lab04
docker rm fit4110-iot-lab04
docker rmi fit4110/iot:lab04
docker system prune -a

# Registry
docker tag fit4110/iot:lab04 ghcr.io/org/team-iot:v0.1.0-lab04
docker push ghcr.io/org/team-iot:v0.1.0-lab04
```

---

## 16. Makefile Quick Commands

```bash
make install          # npm install
make lint             # spectral lint
make mock             # Start mock server
make test-mock        # Test against mock
make build            # docker build
make run              # docker run (foreground)
make run-detached     # docker run -d (background)
make health           # curl /health
make test-docker      # npm run test:local
make stop             # docker stop
make clean-reports    # rm reports/*
```

---

## 17. FAQ

### Q: Làm sao biết service chạy được trong container?
**A:** Chạy `docker run ... fit4110/iot:lab04`, rồi `curl http://localhost:8000/health`. Nếu trả `{status: ok, ...}`, service chạy ổn.

### Q: Newman test fail, phải làm gì?
**A:** Kiểm tra:
1. Container đang chạy? `docker ps`
2. Health endpoint ok? `curl http://localhost:8000/health`
3. Postman environment có authToken đúng?
4. Xem logs: `docker logs fit4110-iot-lab04`

### Q: Image size quá lớn?
**A:** Kiểm tra `.dockerignore` có exclude `node_modules`, `.git`, `__pycache__` không. Hoặc dùng smaller base image (`python:3.11-slim` vs `python:3.11`).

### Q: Port 8000 bị chiếm?
**A:** `docker run -p 9000:8000 fit4110/iot:lab04`, rồi access `http://localhost:9000/health`.

### Q: Cách nộp bài?
**A:** Xem [checklists/submission_checklist.md](checklists/submission_checklist.md) để biết tất cả artifacts cần nộp.

---

## 18. Tips & Tricks

### 18.1 Tối ưu build time

```dockerfile
# ✓ Good: order từ ít thay đổi → hay thay đổi
COPY requirements.txt .      # Stable
RUN pip install -r ...       # Cache nếu requirements.txt không đổi
COPY src/ ./src/             # Changes often
CMD [...]

# ✗ Bad: copy all trước, rồi RUN
COPY .
RUN pip install -r requirements.txt
```

### 18.2 Debug layer caching

```bash
docker build -t fit4110/iot:lab04 . --progress=plain
# Xem từng layer, biết layer nào cache hit/miss
```

### 18.3 Interactive debug

```bash
# Build thêm shell, rồi debug
FROM python:3.11-slim
...
# Chạy interactive
docker run -it fit4110/iot:lab04 /bin/bash
```

### 18.4 Check image trước push

```bash
docker inspect fit4110/iot:lab04 | grep -i size
docker history fit4110/iot:lab04 --human
```

---

## 19. Next Steps (Lab 05 Preview)

Lab 05 sẽ cover:

```
┌─────────────────────────────────────────────────┐
│            Docker Compose                       │
├─────────────────────────────────────────────────┤
│ version: '3.8'                                  │
│ services:                                       │
│   iot-service:                                  │
│     build: .                                    │
│     ports: [8000:8000]                          │
│   analytics-service:                            │
│     build: ./analytics                          │
│     ports: [8001:8000]                          │
│   postgres:                                     │
│     image: postgres:16                          │
│     environment:                                │
│       POSTGRES_PASSWORD: local                  │
└─────────────────────────────────────────────────┘
      ↓
  docker-compose up
      ↓
  Multi-service testing
      ↓
  Plug-a-thon integration
```

---

## 20. Support & Feedback

Gặp vấn đề?

1. 👉 Check [RUN_LOCAL.md](RUN_LOCAL.md) – Quick start (5 bước)
2. 👉 Check [docs/DOCKER_LAB_GUIDE.md](docs/DOCKER_LAB_GUIDE.md) – Concepts
3. 👉 Check [docs/TROUBLESHOOTING.md](docs/TROUBLESHOOTING.md) – Common issues

Feedback? Mở issue hoặc PR trên GitHub.

---

**Chúc bạn hoàn thành Lab 04 thành công!** 🚀🐳

*Final reminder: Commit `.env.example` (public template), không commit `.env` (production secrets).*
