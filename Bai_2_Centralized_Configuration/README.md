# FoodX Centralized Configuration Repository

Kho lưu trữ cấu hình tập trung (Centralized Configuration Repository) cho toàn bộ hệ sinh thái Microservices của nền tảng **FoodX**, được phục vụ thông qua **Spring Cloud Config Server**.

---

## 1. Cấu trúc thư mục (Directory Structure)

```text
foodx-config-repo/
├── README.md                          # Tài liệu hướng dẫn quản trị và quy chuẩn cấu hình
├── application.yml                    # Cấu hình chung cho toàn bộ các microservices (global fallback)
├── application-dev.yml                # Cấu hình chung áp dụng cho môi trường Development
├── application-prod.yml               # Cấu hình chung áp dụng cho môi trường Production
│
├── restaurant-service.yml             # Cấu hình mặc định cho restaurant-service
├── restaurant-service-dev.yml         # Cấu hình môi trường dev cho restaurant-service
├── restaurant-service-prod.yml        # Cấu hình môi trường production cho restaurant-service
│
├── order-service.yml                  # Cấu hình mặc định cho order-service
├── order-service-dev.yml              # Cấu hình môi trường dev cho order-service
├── order-service-prod.yml             # Cấu hình môi trường production cho order-service
│
└── delivery-service.yml               # Cấu hình mặc định cho delivery-service
    ├── delivery-service-dev.yml       # Cấu hình môi trường dev cho delivery-service
    └── delivery-service-prod.yml      # Cấu hình môi trường production cho delivery-service
```

---

## 2. Quy ước đặt tên file cấu hình (Naming Conventions)

Spring Cloud Config Server định tuyến và phục vụ file cấu hình dựa theo HTTP Resource Path:

$$\text{URL pattern}: \quad \texttt{/\{application\}/\{profile\}[/\{label\}]}$$

- **`{application}`**: Khớp chính xác với thuộc tính `spring.application.name` được khai báo trong file `bootstrap.yml` hoặc `application.yml` của service client.
- **`{profile}`**: Khớp với `spring.profiles.active` của client (ví dụ: `default`, `dev`, `prod`, `staging`).
- **`{label}`**: Tên nhánh (branch), thẻ (tag) hoặc commit ID trên Git (mặc định là `main` hoặc `master`).

### Bảng đối chiếu ánh xạ:

| Service | `spring.application.name` | Profile Active | Tên file Git tương ứng |
| :--- | :--- | :--- | :--- |
| **Restaurant Service** | `restaurant-service` | `default` | `restaurant-service.yml` |
| **Restaurant Service** | `restaurant-service` | `dev` | `restaurant-service-dev.yml` |
| **Restaurant Service** | `restaurant-service` | `prod` | `restaurant-service-prod.yml` |
| **Order Service** | `order-service` | `default` | `order-service.yml` |
| **Delivery Service** | `delivery-service` | `default` | `delivery-service.yml` |
| **Cấu hình dùng chung** | *mọi service* | *tương ứng* | `application.yml` / `application-{profile}.yml` |

> [!IMPORTANT]
> - **Tuyệt đối không đặt tên file tự do** (như `config.yml`, `my-config.yml`). Nếu đặt `config.yml`, Config Server sẽ coi đây là cấu hình của một service có tên là `config`, khiến `restaurant-service` không thể nạp được thông số kết nối Database, dẫn tới lỗi khởi động.
> - Sử dụng thống nhất định dạng **kebab-case** (chữ thường nối bằng dấu gạch ngang) cho tất cả tên file và tên service.

---

## 3. Cơ chế mã hóa và bảo mật thông tin nhạy cảm (`{cipher}`)

### 3.1. Rủi ro của Plaintext Credentials
Lưu mật khẩu trực tiếp (`RestaurantPass123`) trong Git repository gây ra các rủi ro nghiêm trọng:
1. **Lộ lọt thông tin trong nội bộ & công cộng:** Bất kỳ ai có quyền truy cập Git repository (developers, CI/CD runners, auditors) đều thấy được thông tin đăng nhập database sản xuất.
2. **Lưu vết lịch sử vĩnh viễn (Git History):** Dù sau này có sửa đổi hoặc xóa đi thì credential vẫn tồn tại trong git commits trước đó.
3. **Vi phạm tiêu chuẩn tuân thủ an toàn thông tin:** Vi phạm các chuẩn PCI-DSS, ISO 27001, OWASP Top 10 và nguyên tắc DevSecOps.

### 3.2. Cú pháp và Quy trình mã hóa `{cipher}`
Spring Cloud Config hỗ trợ mã hóa đối xứng (Symmetric Key) hoặc bất đối xứng (Asymmetric RSA Key Pair).

#### Bước 1: Mã hóa mật khẩu thông qua Config Server
Thực hiện gọi API `POST /encrypt` trên Spring Cloud Config Server:
```bash
curl -X POST http://config-server:8888/encrypt -d "RestaurantPass123"
```
Kết quả trả về một chuỗi mã hóa (ví dụ: `AQA7b92KxLfNm84QeZ1uP0oRstVwXyZ3...`).

#### Bước 2: Lưu vào file cấu hình với tiền tố `{cipher}`
Trong file YAML, đặt chuỗi mã hóa sau tiền tố `{cipher}` và **bắt buộc bọc trong dấu nháy đơn** (`'...'`) để YAML parser không xem dấu ngoặc nhọn `{}` là inline mapping:

```yaml
spring:
  datasource:
    username: restaurant_service_user
    password: '{cipher}AQA7b92KxLfNm84QeZ1uP0oRstVwXyZ3AbCdEfGhIjKlMnOpQrStUvWxYz1234567890=='
```

#### Bước 3: Tự động giải mã tại Config Server
Khi client gửi yêu cầu lấy cấu hình, Config Server sẽ tự động giải mã chuỗi `{cipher}...` bằng master key đã được cấu hình tại server và trả về plain text qua kênh truyền nội bộ an toàn (HTTPS/mTLS), giúp ứng dụng client khởi động bình thường mà không cần tự xử lý thuật toán giải mã.

---

## 4. Hướng dẫn Service Client kết nối Config Server

Trong `bootstrap.yml` (hoặc `application.yml` với Spring Boot 2.4+ / Spring Cloud 2020+):

```yaml
spring:
  application:
    name: restaurant-service
  profiles:
    active: dev
  config:
    import: "optional:configserver:http://foodx-config-server:8888"
  cloud:
    config:
      fail-fast: true # Dừng khởi động ngay nếu không kết nối được Config Server
      retry:
        max-attempts: 5
```

---

## 5. Quy trình quản trị Git Repository (GitOps & Governance)

1. **Branch Protection:** Khóa nhánh `main`/`production`, yêu cầu Code Review (Pull Request) có phê duyệt từ Tech Lead / SecOps trước khi merge.
2. **Pre-commit Hooks:** Cài đặt tool tự động phát hiện secret lọt vào commit (như `git-secrets`, `TruffleHog`, `Gitleaks`).
3. **Phân quyền truy cập (RBAC):** Chỉ cấp quyền đọc (Read-only) cho các Deploy Keys của Config Server trên từng môi trường tương ứng.
