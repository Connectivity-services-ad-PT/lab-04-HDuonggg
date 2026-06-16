# Docker Evidence – Lab 04

## Team

- Team name:
- Service:
- Image tag:

## 1. Build Evidence

Command:

```bash
docker build -t <image-name>:<tag> .
```

Paste log or screenshot here.

## 2. Run Evidence

Command:

```bash
docker run --rm -p 8000:8000 --env-file .env.example <image-name>:<tag>
```

Paste log or screenshot here.

## 3. Healthcheck Evidence

Command:

```bash
curl http://localhost:8000/health
```

Result:

```json
{
  "status": "ok"
}
```

## 4. Newman Evidence

Command:

```bash
npm run test:local
```

Report path:

```text
reports/newman-lab04-local.html
reports/newman-lab04-local.xml
```

## 5. Notes

- Known limitation:
- Next step for Lab 05:
