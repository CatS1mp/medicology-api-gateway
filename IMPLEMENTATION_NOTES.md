# Medicology API Gateway — Tóm tắt các bước triển khai

Tài liệu này mô tả ngắn gọn những thay đổi đã thực hiện để đưa **`medicology-api-gateway`** vào hệ thống. Mục tiêu: mọi request FE → BE đi qua gateway, gateway **stateless** (không lưu gì, không parse JWT), FE giữ token, BE tự verify.

---

## 1. Tạo project gateway

Thư mục mới: `c:\code\Medicology\medicology-api-gateway`

| File                                                                      | Vai trò                                                |
| ------------------------------------------------------------------------- | ------------------------------------------------------ |
| `pom.xml`                                                                 | Spring Boot 3.3.0 + Spring Cloud Gateway 2023.0.3      |
| `src/main/java/com/medicology/gateway/GatewayApplication.java`            | Entrypoint `@SpringBootApplication`                    |
| `src/main/resources/application.yml`                                      | Cổng `8090` + định nghĩa route                          |
| `mvnw`, `mvnw.cmd`, `.mvn/wrapper/maven-wrapper.properties`               | Maven wrapper (copy từ assessment-service)             |
| `.gitignore`, `README.md`                                                 | Cấu hình IDE/Git + hướng dẫn chạy                       |

### Dependencies (chỉ 3 món)

- `spring-cloud-starter-gateway`
- `spring-boot-starter-actuator` (health check)
- `spring-boot-starter-test` (scope test)

**Cố tình không có**: `spring-security`, `spring-data-jpa`, JWT library, datasource. Gateway thuần forward.

---

## 2. Cấu hình route (`application.yml`)

Toàn bộ logic định tuyến nằm ở 4 route khai báo dạng YAML, dùng filter `RewritePath` của Spring Cloud Gateway:

| Predicate (path FE/BFF gọi)                                                                                          | Filter rewrite                            | URI upstream                                            |
| -------------------------------------------------------------------------------------------------------------------- | ----------------------------------------- | ------------------------------------------------------- |
| `/api/assessment/**`                                                                                                 | `/api/assessment/(.*) → /api/v1/$1`        | `ASSESSMENT_SERVICE_URL` (mặc định `localhost:8083`)    |
| `/api/learning/**`                                                                                                   | `/api/(.*) → /api/v1/$1`                   | `LEARNING_SERVICE_URL` (mặc định `localhost:8081`)      |
| `/api/dictionary/**`                                                                                                 | — (giữ nguyên)                            | `DICTIONARY_SERVICE_URL` (mặc định `localhost:8082`)    |
| `/api/auth/**,/api/admin/**,/api/oauth/**,/api/sessions/**,/api/profiles/**,/api/settings/**,/api/users/**`           | `/api/(.*) → /api/v1/$1`                   | `AUTH_SERVICE_URL` (mặc định `localhost:8080`)          |

Default filter: `PreserveHostHeader`. Không có filter ghi/đọc cookie, không có auth filter.

---

## 3. Quy tắc bảo mật / phi-state

- Gateway **không** thêm/đọc cookie, không quản lý session.
- Header `Authorization` (cùng các header khác) được **chuyển nguyên trạng** từ FE → BE.
- BE tự xác thực JWT trên header như trước.
- FE tự giữ token (HttpOnly cookie do Next.js BFF set, hoặc localStorage tuỳ flow).
- Không có config inter-service (token nội bộ, base-url khác service…) — gateway chỉ là proxy.

---

## 4. Tích hợp Frontend (`medicology-website`)

### 4.1. Helper proxy chung — `src/app/api/_proxy.ts`

Thêm:

```ts
export const API_GATEWAY_URL = (process.env.API_GATEWAY_URL ?? '').trim();

export function proxyThroughGateway(req, params, config: {
    gatewayBasePath: string;          // ví dụ '/api/assessment'
    legacy: { backendUrl, upstreamBasePath };
}) {
    if (API_GATEWAY_URL) {
        return proxyToBackend(req, params, {
            backendUrl: API_GATEWAY_URL,
            upstreamBasePath: config.gatewayBasePath,
        });
    }
    return proxyToBackend(req, params, config.legacy);
}
```

Khi `API_GATEWAY_URL` được đặt: FE đi qua gateway. Khi trống: fallback gọi thẳng service (giữ tương thích cũ).

### 4.2. Refactor 10 route BFF Next.js

Cùng pattern cho mỗi route, ví dụ `src/app/api/assessment/[...path]/route.ts`:

```ts
const config = {
    gatewayBasePath: '/api/assessment',
    legacy: { backendUrl: process.env.ASSESSMENT_SERVICE_URL ?? '', upstreamBasePath: '/api/v1' },
};
// 5 method (GET/POST/PUT/PATCH/DELETE) gọi proxyThroughGateway(req, await params, config)
```

Áp dụng cho: `assessment`, `learning`, `dictionary`, `auth/[...]`, `admin`, `users`, `profiles`, `oauth/[...]`, `settings`, `sessions`.

### 4.3. Route auth đặc biệt (set HttpOnly cookie)

`auth/login`, `auth/refresh`, `auth/logout`, `oauth/oauth` cần bóc token từ response để set cookie phía FE. Các route này dùng helper mới `getAuthBackend()` trong `src/lib/env-backend.ts`:

```ts
export function getAuthBackend(): { base; basePath } | null {
    if (API_GATEWAY_URL) return { base: API_GATEWAY_URL, basePath: '/api/auth' };
    if (AUTH_SERVICE_URL) return { base: AUTH_SERVICE_URL, basePath: '/api/v1/auth' };
    return null;
}
```

Cả 4 route đổi `fetch(${backend}/api/v1/auth/xxx)` thành `fetch(${backend.base}${backend.basePath}/xxx)`.

### 4.4. `.env`

```dotenv
API_GATEWAY_URL=http://localhost:8090

# Chỉ dùng khi API_GATEWAY_URL trống (fallback)
AUTH_SERVICE_URL=http://localhost:8080
LEARNING_SERVICE_URL=http://localhost:8081
DICTIONARY_SERVICE_URL=http://localhost:8082
ASSESSMENT_SERVICE_URL=http://localhost:8083
```

---

## 5. Sơ đồ flow

```
Browser ──► Next.js BFF (/api/{service}/...) ──► Gateway (rewrite path) ──► Service đích
   ▲                                                                              │
   └────────────── response (status + body + headers, giữ nguyên) ────────────────┘
```

Token nằm ở cookie HttpOnly do Next.js BFF set khi login. Mỗi request kế tiếp, BFF đọc cookie và đính header `Authorization: Bearer …` → gateway chuyển tiếp → BE verify.

---

## 6. Kiểm thử đã chạy

| Bước                                                                                          | Kết quả                                          |
| --------------------------------------------------------------------------------------------- | ------------------------------------------------ |
| `./mvnw -DskipTests package` trong gateway                                                    | BUILD SUCCESS                                    |
| `./mvnw spring-boot:run`                                                                       | `Started GatewayApplication` trong ~5 s          |
| `GET http://localhost:8090/actuator/health`                                                   | `{"status":"UP"}`                                |
| `curl http://localhost:8090/api/assessment/users/me/attempts` (không kèm JWT)                 | `HTTP 401` từ assessment-service ⇒ forwarding OK |
| TypeScript `tsc --noEmit` + ESLint trên các file BFF đã sửa                                   | Sạch (chỉ còn 1 lỗi pre-existing không liên quan) |

---

## 7. Cách chạy

```bash
# Terminal 1 — gateway
cd medicology-api-gateway
./mvnw spring-boot:run     # listens on :8090

# Terminal 2..N — các BE service (giữ nguyên port cũ: 8080/8081/8082/8083)

# Terminal — FE
cd medicology-website
# đảm bảo .env có API_GATEWAY_URL=http://localhost:8090
npm run dev
```

Production: deploy gateway như một service riêng (vd Heroku/Docker), set 4 env `*_SERVICE_URL` trỏ tới URL nội bộ của các service, FE chỉ cần biết `API_GATEWAY_URL` public.

---

## 8. Việc có thể mở rộng sau (chưa làm trong scope này)

- Rate limit / circuit breaker (Resilience4j) — Spring Cloud Gateway hỗ trợ sẵn nhưng chưa bật để giữ "chỉ forward".
- Tracing (Micrometer + Zipkin/Tempo) cho observability.
- Mở public CORS ở gateway nếu sau này FE gọi gateway trực tiếp thay vì qua Next.js BFF.
