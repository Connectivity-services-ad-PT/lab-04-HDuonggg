# TROUBLESHOOTING.md – Lỗi thường gặp Lab 04

---

## Quick Reference Table

| Lỗi | Nguyên nhân | Cách xử lý |
|---|---|---|
| `Cannot connect to Docker daemon` | Docker Desktop chưa chạy | Mở Docker Desktop, chờ Docker ready |
| `permission denied` | User không trong docker group | `sudo usermod -aG docker $USER` |
| `port is already allocated` | Port 8000 đang bị service khác dùng | Dừng service cũ hoặc đổi `-p 9000:8000` |
| `ModuleNotFoundError` | Thiếu dependency hoặc sai `--app-dir` | Kiểm tra `requirements.txt` và lệnh CMD |
| `/health` không phản hồi | App chưa start hoặc sai port | `docker logs fit4110-iot-lab04` |
| `ECONNREFUSED` trong Newman | Container chưa chạy hoặc baseUrl sai | Check `docker ps` và postman environment |
| Image quá nặng (1GB+) | Copy thừa `.git`, `.venv`, node_modules | Sửa `.dockerignore` |
| Container chạy bằng root | Dockerfile chưa tạo user | Thêm `USER appuser` trước CMD |
| CI fail ở bước Newman | Service chưa ready khi test | Dùng `wait-for-health.sh` |
| 401/403 trong test | Auth token sai hoặc mismatch | Verify `.env.example` = postman env |

---

## Detailed Troubleshooting

### 🔴 Docker Installation

#### `docker: command not found`
```bash
# Check installation
docker --version

# If not found:
# Windows/Mac: https://www.docker.com/products/docker-desktop
# Linux: sudo apt-get install docker.io

# Verify
docker run hello-world
```

#### `permission denied while trying to connect to Docker daemon`
```bash
# Linux: add user to docker group
sudo usermod -aG docker $USER
newgrp docker

# Verify
groups  # should include 'docker'
```

---

### 🔴 Build Issues

#### `Step X/Y : ERROR`
```bash
# Common: requirements.txt not found
ls -la requirements.txt

# Check Docker context
cat .dockerignore  # if too restrictive

# Rebuild with verbose
docker build -t fit4110/iot:lab04 . --progress=plain
```

#### `dockerfile: syntax error`
```dockerfile
# ✓ Correct
FROM python:3.11-slim
RUN pip install -r requirements.txt

# ✗ Wrong (typo, lowercase)
from python:3.11-slim
run pip install -r requirements.txt
```

#### Build hangs/timeout
```bash
# Check network
ping pypi.org

# Skip cache (force rebuild)
docker build --no-cache -t fit4110/iot:lab04 .

# Check pip speed
pip install fastapi -v
```

---

### 🔴 Runtime Issues

#### Port 8000 already in use

**Windows:**
```powershell
netstat -ano | findstr :8000
taskkill /PID <PID> /F
```

**macOS/Linux:**
```bash
lsof -i :8000
kill -9 <PID>
```

**Or use different port:**
```bash
docker run -p 9000:8000 fit4110/iot:lab04
# Access: http://localhost:9000/health
```

#### Container exits immediately
```bash
# View logs
docker run fit4110/iot:lab04  # shows error
docker logs fit4110-iot-lab04  # if container named

# Common causes:
# 1. Missing requirements
ls -la requirements.txt

# 2. Wrong source path
ls -la src/iot_app/main.py

# 3. Bad CMD
# Check Dockerfile last line is correct
```

#### Can't reach /health endpoint

```bash
# Check if container running
docker ps

# If not, start it
docker run -d --name fit4110-iot-lab04 \
  -p 8000:8000 \
  --env-file .env.example \
  fit4110/iot:lab04

# Test from inside container
docker exec fit4110-iot-lab04 \
  curl http://localhost:8000/health

# Check actual port mapping
docker port fit4110-iot-lab04
# Output: 8000/tcp -> 0.0.0.0:8000
```

---

### 🔴 Testing Issues

#### Newman tests fail

```bash
# Step 1: Verify prerequisites
docker ps  # container running?
curl http://localhost:8000/health  # endpoint works?
npm list newman  # newman installed?

# Step 2: Check Postman environment
cat postman/environments/FIT4110_lab04_local.postman_environment.json

# Step 3: Run with verbose
npx newman run postman/collections/FIT4110_lab04_iot_docker.postman_collection.json \
  -e postman/environments/FIT4110_lab04_local.postman_environment.json \
  --verbose
```

#### Authentication test fails (401)

```bash
# Verify auth token is consistent
grep AUTH_TOKEN .env.example
# Output: AUTH_TOKEN=local-dev-token

# Check Postman env has same token
cat postman/environments/FIT4110_lab04_local.postman_environment.json | grep authToken

# Test manually
curl -X POST http://localhost:8000/readings \
  -H "Authorization: Bearer local-dev-token" \
  -H "Content-Type: application/json" \
  -d '{"device_id":"ESP32-LAB-A01","metric":"temperature","value":25,"unit":"celsius","timestamp":"2026-05-13T08:30:00+07:00"}'
# Should return 201
```

#### Boundary tests fail

```bash
# Check value constraints in code
grep -A3 "value: float" src/iot_app/main.py
# Should show: ge=-40, le=80

# Test manually
curl -X POST http://localhost:8000/readings \
  -H "Authorization: Bearer local-dev-token" \
  -H "Content-Type: application/json" \
  -d '{"device_id":"ESP32-LAB-A01","metric":"temperature","value":80,"unit":"celsius","timestamp":"2026-05-13T08:30:00+07:00"}'
# Should return 201 (within range)

curl ... -d '...,"value":81,...'
# Should return 422 (out of range)
```

#### ProblemDetails format missing fields

```bash
# Send invalid request to see error format
curl -i -X POST http://localhost:8000/readings \
  -H "Authorization: Bearer local-dev-token" \
  -H "Content-Type: application/json" \
  -d '{"device_id":""}'  # too short

# Should return JSON like:
# {
#   "type": "...",
#   "title": "Validation error",
#   "status": 422,
#   "detail": "...",
#   "instance": "/readings"
# }

# Check FastAPI exception_handler
grep -A10 "@app.exception_handler(RequestValidationError)" src/iot_app/main.py
```

---

### 🔴 Configuration Issues

#### AUTH_TOKEN mismatch

```bash
# Verify all 3 places match:

# 1. .env.example
grep AUTH_TOKEN .env.example

# 2. Container environment
docker exec fit4110-iot-lab04 env | grep AUTH_TOKEN

# 3. Postman environment
cat postman/environments/FIT4110_lab04_local.postman_environment.json | grep authToken

# If mismatch, update all 3
```

#### Service version mismatch

```bash
# 1. Check OpenAPI contract
grep "version:" contracts/iot-ingestion.openapi.yaml

# 2. Check Python code
grep SERVICE_VERSION src/iot_app/main.py

# 3. Check .env.example
grep SERVICE_VERSION .env.example

# All should match (e.g., 0.4.0)
```

---

### 🔴 Performance Issues

#### Build too slow

```bash
# Use smaller base image
FROM python:3.11-slim  # ~165 MB ✓
FROM python:3.11       # ~900 MB ✗

# Check layer sizes
docker history fit4110/iot:lab04 --human

# Multi-stage build reduces final size
# Stage 1: build (includes pip cache)
# Stage 2: runtime (only copy venv)
```

#### Large image size (>500 MB)

```bash
# Check what's included
docker inspect fit4110/iot:lab04 | grep -i size

# Fix .dockerignore to exclude:
cat .dockerignore
# Should have: node_modules, .git, .venv, __pycache__, reports, *.pyc

# Rebuild
docker build --no-cache -t fit4110/iot:lab04 .
```

---

### 🔴 Cleanup

```bash
# Stop specific container
docker stop fit4110-iot-lab04

# Remove container
docker rm fit4110-iot-lab04

# Remove image
docker rmi fit4110/iot:lab04

# Clean all (⚠️ caution)
docker system prune -a

# Remove dangling images
docker image prune
```

---

## Debugging Commands

```bash
# Real-time logs
docker logs -f fit4110-iot-lab04

# Last 50 lines
docker logs --tail 50 fit4110-iot-lab04

# Execute command in running container
docker exec fit4110-iot-lab04 env

# Interactive shell
docker exec -it fit4110-iot-lab04 /bin/bash

# Check resource usage
docker stats fit4110-iot-lab04

# Full container inspection
docker inspect fit4110-iot-lab04

# Image layer history
docker history fit4110/iot:lab04
```

---

## Pre-Submission Checklist

Before submitting, verify:

```bash
✓ docker build -t fit4110/iot:lab04 . 
✓ docker run -p 8000:8000 --env-file .env.example fit4110/iot:lab04
✓ curl http://localhost:8000/health  # returns 200
✓ npm run test:local  # all tests pass
✓ reports/newman-lab04-local.html exists
✓ docker logs <container> | grep "started server process"  # non-root user
✓ .dockerignore has node_modules, .git, __pycache__
✓ .env.example has no secrets (AUTH_TOKEN is generic)
✓ RUN_LOCAL.md is readable and accurate
```

---

**Need more help? Check [DOCKER_LAB_GUIDE.md](DOCKER_LAB_GUIDE.md) for concepts and best practices!**
