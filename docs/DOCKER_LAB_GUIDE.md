# DOCKER_LAB_GUIDE.md – Hướng dẫn Lab 04 Docker

---

## 1. Tổng quan Docker

### 1.1 Image vs Container

```
Image     = bản đóng gói bất biến (như .iso file)
Container = tiến trình đang chạy từ image (như VM instance)
```

### 1.2 Vì sao cần Docker?

| Vấn đề | Docker giải quyết |
|--------|-------------------|
| "Works on my machine" | Cô lập env: OS + runtime + app + config |
| Deploy khác dev | Image bao gồm tất cả dependencies |
| Team collaboration | Ai cũng chạy image giống nhau |
| CI/CD automation | Registry → Pull → Run → Test → Deploy |

---

## 2. Kiến trúc Dockerfile (Lab 04)

### 2.1 Multi-stage build

```dockerfile
# Stage 1: Builder
FROM python:3.11-slim AS builder
RUN python -m venv /opt/venv
COPY requirements.txt .
RUN /opt/venv/bin/pip install --no-cache-dir -r requirements.txt

# Stage 2: Runtime
FROM python:3.11-slim AS runtime
COPY --from=builder /opt/venv /opt/venv
COPY src/ ./src/
USER appuser
CMD ["uvicorn", "iot_app.main:app"]
```

**Lợi ích:**
- Final image nhỏ 50% (loại build tools)
- Faster layer cache: code changes không rebuild deps
- Security: no compiler, no pip cache

### 2.2 Non-root user (Security)

```dockerfile
RUN addgroup --system appgroup
RUN adduser --system --ingroup appgroup appuser
USER appuser
```

**Lý do:**
- Ngăn privilege escalation
- Best practice: quyền thấp nhất cần thiết

### 2.3 HEALTHCHECK (Production-ready)

```dockerfile
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
  CMD python -c "import urllib.request; urllib.request.urlopen('http://127.0.0.1:8000/health')" || exit 1
```

**Hoạt động:**
```
Docker daemon mỗi 30s gọi /health:
├─ 200 OK  →  Status: healthy
├─ Error 3 lần  →  Status: unhealthy
└─ CI/K8s restart container
```

---

## 3. Environment Configuration

### 3.1 .env.example (Public template)

```bash
APP_HOST=0.0.0.0
APP_PORT=8000
AUTH_TOKEN=local-dev-token
SERVICE_NAME=iot-ingestion
SERVICE_VERSION=0.4.0
```

✅ Commit vào repo  
❌ Không chứa production secrets

### 3.2 .env (Private runtime config)

Tạo khi deploy:
```bash
AUTH_TOKEN=super-secret-prod-token
SERVICE_VERSION=1.0.0-prod
```

❌ Không commit vào repo  
✅ Deployed bởi CI/CD hoặc DevOps

### 3.3 Truyền config vào container

**Option 1: Via environment file**
```bash
docker run --env-file .env.example fit4110/iot:lab04
```

**Option 2: Via CLI flags**
```bash
docker run -e AUTH_TOKEN=xyz -e APP_PORT=8000 fit4110/iot:lab04
```

**Option 3: Trong code**
```python
import os
AUTH_TOKEN = os.getenv("AUTH_TOKEN", "default-token")
SERVICE_VERSION = os.getenv("SERVICE_VERSION", "0.1.0")
```

---

## 4. Build & Run Workflow

### 4.1 Build image

```bash
docker build -t fit4110/iot-ingestion:lab04 .
```

**Cách hoạt động:**
```
1. Đọc Dockerfile
2. Fetch base image (python:3.11-slim)
3. Execute mỗi instruction (RUN, COPY, ADD)
4. Build layer cache: reuse nếu không thay đổi
5. Tag final image
```

**Layer caching:**
```dockerfile
FROM python:3.11-slim       # Layer 1: cached (base image)
WORKDIR /app                # Layer 2: cached
COPY requirements.txt .     # Layer 3: cached (deps stable)
RUN pip install -r ...      # Layer 4: cached

# Nếu thay đổi code:
COPY src/ ./src/            # Layer 5: INVALIDATED (rebuild)
```

### 4.2 Run container

```bash
docker run --rm \
  --name fit4110-iot-lab04 \
  -p 8000:8000 \
  --env-file .env.example \
  fit4110/iot-ingestion:lab04
```

**Flags:**
| Flag | Mục đích |
|------|---------|
| `--rm` | Auto cleanup khi exit |
| `--name` | Container name (reference) |
| `-p 8000:8000` | Port mapping (host:container) |
| `-e`, `--env-file` | Environment variables |
| `-d` | Detached (background) |
| `-it` | Interactive terminal |

---

## 5. Testing Strategy

### 5.1 5 layers of test

```
┌─────────────────────────────────────┐
│    05: Error Schema (ProblemDetails)│  Standardized error format
├─────────────────────────────────────┤
│    04: Boundary (-40, 0, 80)        │  Edge cases
├─────────────────────────────────────┤
│    03: Negative (invalid input)     │  Bad requests
├─────────────────────────────────────┤
│    02: Authentication (401, 403)    │  Authorization
├─────────────────────────────────────┤
│    01: Functional (happy path)      │  Main use case
└─────────────────────────────────────┘
```

### 5.2 Test cases (Newman Collection)

**Group 1: Functional**
- `GET /health` → 200 {status: ok, service, version}
- `POST /readings` (valid) → 201 {reading_id, accepted: true}
- `GET /readings/latest` → 200 {items: []}
- `GET /readings/{id}` → 200 {reading: {...}}

**Group 2: Authentication**
- Missing `Authorization` → 401 ProblemDetails
- Invalid token → 401 ProblemDetails

**Group 3: Negative**
- Empty `device_id` → 422 validation error
- Missing `metric` → 422
- Bad `timestamp` → 422

**Group 4: Boundary**
- `value: -40` (min) → 201 ✓
- `value: 80` (max) → 201 ✓
- `value: 81` (over) → 422 ✗

**Group 5: Error Schema**
- 422 response có: `type`, `title`, `status`, `detail`, `instance`
- Media type: `application/problem+json`

---

## 6. Debugging

### 6.1 View image info

```bash
docker images                                    # List images
docker history fit4110/iot-ingestion:lab04       # Layer history
docker inspect fit4110/iot-ingestion:lab04       # Image config
```

### 6.2 View container logs

```bash
docker logs fit4110-iot-lab04                    # Show logs
docker logs -f fit4110-iot-lab04                 # Follow (real-time)
docker logs --tail 50 fit4110-iot-lab04          # Last 50 lines
```

### 6.3 Run commands in container

```bash
docker exec fit4110-iot-lab04 curl http://localhost:8000/health
docker exec -it fit4110-iot-lab04 /bin/bash      # Interactive shell
```

### 6.4 Inspect running container

```bash
docker ps                                        # Running containers
docker ps -a                                     # All containers
docker stats fit4110-iot-lab04                   # CPU/Memory usage
docker inspect fit4110-iot-lab04                 # Full config
```

---

## 7. Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| `docker: command not found` | Docker not installed | Install Docker Desktop or Engine |
| `permission denied` | User not in docker group | `sudo usermod -aG docker $USER` |
| `port 8000 already in use` | Port occupied | `docker run -p 9000:8000 ...` |
| `image not found` | Image not built | `docker build -t fit4110/iot:lab04 .` |
| `container exited` | Service crashed | `docker logs <container>` |
| `HEALTHCHECK failed` | /health down | Check service running |
| `ProblemDetails wrong` | Exception handler missing | Check FastAPI setup |

---

## 8. Best Practices

✅ **DO:**
- Use specific base image version (not `latest`)
- Multi-stage build for optimization
- Non-root user for security
- HEALTHCHECK for monitoring
- .env.example for template config
- NO hardcoded secrets in code

❌ **DON'T:**
- Hardcode passwords or tokens
- Use `latest` tag
- Run as root
- Copy unnecessary files (use .dockerignore)
- Large base images (use `-slim` or `-alpine`)

---

## 9. Evidence Checklist

Before submitting:

```
✅ Dockerfile (multi-stage, non-root, HEALTHCHECK)
✅ .dockerignore (clean context)
✅ .env.example (no secrets)
✅ RUN_LOCAL.md (3-5 steps)
✅ Screenshot: docker build
✅ Screenshot: docker run
✅ Screenshot: curl /health
✅ Newman report (XML + HTML)
✅ Image tag: ghcr.io/<org>/team-iot:v0.1.0-lab04
```

---

## 10. Next: Lab 05 (Docker Compose)

Lab 04 lập nền cho Lab 05:

```
Lab 04: Single container ✓
Lab 05: Multi-container (docker-compose.yml)

docker-compose.yml:
  services:
    iot-service:
      build: .
      ports: [8000:8000]
    
    analytics-service:
      build: ./analytics
      ports: [8001:8000]
    
    postgres:
      image: postgres:16
      environment:
        POSTGRES_PASSWORD: local
```

---

**Chúc bạn thành công Lab 04!** 🐳

- multi-stage build
- non-root user
- healthcheck
- environment variables
- `.dockerignore`

---

## 4. Healthcheck

Container không chỉ cần "đang chạy", mà cần "service bên trong sẵn sàng".

```Dockerfile
HEALTHCHECK CMD python -c "import urllib.request; urllib.request.urlopen('http://127.0.0.1:8000/health').read()" || exit 1
```

Kiểm tra:

```bash
docker ps
docker inspect fit4110-iot-lab04
```

---

## 5. Không đưa secret vào image

Không viết trực tiếp token vào code hoặc Dockerfile.

Sai:

```Dockerfile
ENV TELEGRAM_TOKEN=123456
```

Đúng:

```Dockerfile
ENV TELEGRAM_TOKEN=
```

và truyền khi chạy:

```bash
docker run --env-file .env ...
```
