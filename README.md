# Medicology API Gateway

Reverse-proxy stateless dùng Spring Cloud Gateway. Vai trò duy nhất là chuyển tiếp request từ FE đến đúng backend service và trả response ngược lại. Gateway **không** lưu state, **không** parse/issue JWT, **không** quản lý session — header `Authorization` được giữ nguyên, BE tự verify, FE tự giữ token (cookie/localStorage).

## Bản đồ định tuyến

| Đường dẫn gateway nhận            | Service đích          | URL upstream (mặc định)        |
| --------------------------------- | --------------------- | ------------------------------- |
| `/api/auth/**`                    | auth-service          | `http://localhost:8085`         |
| `/api/admin/**`                   | auth-service          | `http://localhost:8085`         |
| `/api/oauth/**`                   | auth-service          | `http://localhost:8085`         |
| `/api/sessions/**`                | auth-service          | `http://localhost:8085`         |
| `/api/profiles/**`                | auth-service          | `http://localhost:8085`         |
| `/api/settings/**`                | auth-service          | `http://localhost:8085`         |
| `/api/users/**`                   | auth-service          | `http://localhost:8085`         |
| `/api/learning/**`                | learning-service      | `http://localhost:8081`         |
| `/api/dictionary/**`              | dictionary-service    | `http://localhost:8082`         |
| `/api/assessment/**`              | assessment-service    | `http://localhost:8083`         |

Các path trên sẽ được rewrite về dạng `/api/v1/{segment}` trước khi gọi service (trừ `dictionary` đã đúng prefix `/api/dictionary`).

## Biến môi trường

| Biến                       | Ý nghĩa                                  | Mặc định                  |
| -------------------------- | ----------------------------------------- | ------------------------- |
| `GATEWAY_PORT`             | Cổng gateway lắng nghe                    | `8085`                    |
| `AUTH_SERVICE_URL`         | URL auth-service                          | `http://localhost:8080`   |
| `LEARNING_SERVICE_URL`     | URL learning-service                      | `http://localhost:8081`   |
| `DICTIONARY_SERVICE_URL`   | URL dictionary-service                    | `http://localhost:8082`   |
| `ASSESSMENT_SERVICE_URL`   | URL assessment-service                    | `http://localhost:8083`   |

## Chạy local

```bash
./mvnw spring-boot:run
```

Health: `GET http://localhost:8085/actuator/health`.

## Tích hợp FE

Trong `medicology-website`, đặt `API_GATEWAY_URL=http://localhost:8085` (không dùng `8084` — cổng đó là notification-service). Mọi BFF proxy của Next.js sẽ forward request tới gateway theo path FE nhận được (`/api/{service}/...`), gateway phụ trách rewrite + chuyển tiếp tới BE.
