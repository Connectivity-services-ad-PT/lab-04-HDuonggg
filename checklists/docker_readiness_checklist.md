# Docker Readiness Checklist

## Dockerfile

- [X] Có base image hợp lý.
- [X] Có `WORKDIR`.
- [X] Có copy dependency trước source để tận dụng cache.
- [X] Có `EXPOSE`.
- [X] Có `CMD` hoặc `ENTRYPOINT`.
- [X] Có `HEALTHCHECK`.
- [X] Có user non-root.
- [X] Không chứa secret thật.

## Runtime

- [X] Container chạy được.
- [X] Port map đúng.
- [X] `/health` trả `200`.
- [X] Log khởi động rõ ràng.
- [X] Cấu hình qua ENV.

## Testing

- [X] Chạy lại Postman Collection từ Lab 03.
- [X] Newman report sinh ra trong `reports/`.
- [X] Functional test pass.
- [X] Auth test pass trên local/container.
- [X] Negative test pass trên local/container.
- [X] Boundary test pass hoặc có giải thích hợp đồng.

## Evidence

- [X] Có ảnh/log `docker build`.
- [X] Có ảnh/log `docker run`.
- [X] Có ảnh/log `curl /health`.
- [X] Có Newman HTML/XML report.
- [X] Có tag image đúng quy ước.
