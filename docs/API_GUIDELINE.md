# API Gateway Guideline — Medicology

## 1. Mục đích tài liệu
Hướng dẫn cách Gateway tiếp nhận, xác thực, định tuyến và trả lỗi cho client khi gọi vào hệ thống Medicology.

## 2. Base URL và tài liệu kỹ thuật
- Local: `http://localhost:8080`
- Staging: `Cập nhật theo môi trường của dự án`
- Production: `Cập nhật theo môi trường của dự án`

Swagger / OpenAPI (nếu có):
- `GET /swagger-ui/index.html`
- `GET /v3/api-docs`

## 3. Authentication và quy ước chung
### 3.1 Authentication
- Tất cả request (trừ public routes) phải gửi `Authorization: Bearer <JWT>`.
- Gateway xác minh chữ ký JWT và các claims cơ bản (`exp`, `iss`, `aud`, `roles/scopes`).

Header chuẩn và khuyến nghị:
```http
Authorization: Bearer <jwt-token>
Content-Type: application/json
X-Correlation-Id: <uuid-để-theo-vết-end-to-end>
```

### 3.2 Kiểu lỗi (chuẩn hoá phản hồi)
Ví dụ format lỗi chuẩn hoá tại Gateway:
```json
{
  "status": 401,
  "error": "Unauthorized",
  "message": "Invalid or expired token",
  "correlationId": "...",
  "timestamp": "2026-05-14T12:00:00Z"
}
```
- Mapping gợi ý: 400 (Bad Request), 401 (Unauthorized), 403 (Forbidden), 404 (Not Found), 409 (Conflict), 422 (Unprocessable), 429 (Too Many Requests), 5xx (Server Error).
- Không rò rỉ thông tin nội bộ hay PHI vào thông điệp lỗi.

### 3.3 Quy ước response
- Pass-through JSON từ service bên dưới; có thể bọc theo wrapper chung tuỳ chính sách.
- Pagination theo chuẩn `page`, `size`, `sort`; hoặc `cursor` nếu có.

## 4. Tóm tắt mapping theo use-case
| Use-case | Route Gateway | Backend |
| --- | --- | --- |
| Tạo đánh giá | `POST /assessments` | Assessment Service |
| Tìm thuật ngữ | `GET /dictionary/...` | Dictionary Service |
| Học/tiến độ | `GET /learning/...` | Learning Service |
| Đăng nhập | `POST /auth/login` | Auth Service |

## 5. Nhóm chính — Định tuyến và tổng hợp
- `/*` → định tuyến theo prefix (`/assessments`, `/dictionary`, `/learning`, `/auth`, ...)
- (Tuỳ chọn) Endpoint tổng hợp báo cáo gọi 2+ service rồi hợp nhất dữ liệu trước khi trả cho client.

## 6. Webhook / callback
- Không áp dụng ở Gateway nếu chưa có yêu cầu.

## 7. Hợp đồng với service khác
- Tin cậy vào Auth Service để phát hành JWT; cấu hình `issuer-uri`/`jwk-set-uri` để xác thực token tại Gateway.
- Chuyển tiếp `X-Correlation-Id` tới mọi service để truy vết.

---

*Cập nhật lần cuối: 2026-05-14 — Platform team*
