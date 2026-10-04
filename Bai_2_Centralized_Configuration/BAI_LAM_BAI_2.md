# BÁO CÁO BÀI TẬP 2 — TỔ CHỨC CENTRALIZED CONFIGURATION CHO HỆ THỐNG FOODX

**Môn học:** Kiến trúc Hệ thống / Microservices (IT209)  
**Session 02:** Từ Monolithic đến Microservice  
**Cấp độ:** Vận dụng cơ bản  
**Đối tượng thực hiện:** Kỹ sư hệ thống FoodX  

---

## 1. PHÂN TÍCH LỖI VÀ ĐỀ XUẤT KHẮC PHỤC

### 1.1. Lỗi đặt sai tên file cấu hình (`config.yml`)

#### Phân tích nguyên nhân & cơ chế hoạt động của Spring Cloud Config
- **Cơ chế ánh xạ của Spring Cloud Config Server:**
  Spring Cloud Config Server quản lý và cung cấp cấu hình dựa trên tên định danh của ứng dụng (`spring.application.name`) và môi trường hoạt động (`spring.profiles.active`).
  Quy tắc ánh xạ tài nguyên từ HTTP REST API của Config Server sang file trong Git repository tuân thủ định dạng:
  $$\texttt{/\{application\}/\{profile\}[/\{label\}]}$$
  tương ứng với các file:
  - `{application}-{profile}.yml`
  - `{application}.yml`
  - `{application}-{profile}.properties`
  - Hoặc fallback về `application.yml` (dùng chung cho mọi service).

- **Lỗi cụ thể của thực tập sinh:**
  - `restaurant-service` khai báo định danh: `spring.application.name = "restaurant-service"`.
  - File cấu hình trên Git lại được đặt tên là `config.yml`.
  - Khi `restaurant-service` khởi động, nó sẽ gửi HTTP GET request tới Config Server theo đường dẫn:  
    `GET http://config-server:8888/restaurant-service/default`
  - Config Server sẽ chỉ tìm kiếm các file: `restaurant-service.yml`, `restaurant-service-default.yml`, `application.yml` trên Git repository. Nó **hoàn toàn bỏ qua** file `config.yml` (vì Config Server chỉ nạp `config.yml` nếu service có tên là `config`).

#### Hậu quả khi Config Server không tìm thấy file cấu hình:
1. **Lỗi khởi động dịch vụ (Startup Failure):** Nếu service cấu hình `spring.cloud.config.fail-fast=true`, service sẽ ném ngoại lệ và dừng khởi động ngay lập tức do thiếu thông số cấu hình tối thiểu.
2. **Lỗi kết nối cơ sở dữ liệu (Datasource Connection Error):** Nếu không bật `fail-fast`, service sẽ khởi động bằng cấu hình mặc định cục bộ (fallback). Khi đó, các thông tin Datasource (`url`, `username`, `password`) không được cung cấp, dẫn tới lỗi `HikariPool: Failed to initialize pool: Connection refused` hoặc `Driver not found`, làm sập tầng DAO/Repository.
3. **Xung đột cổng mạng (Port Conflict):** Cổng chạy chỉ định `server.port: 8085` trong `config.yml` không được nạp, Spring Boot sẽ tự fallback về cổng mặc định `8080`, gây xung đột cổng với các service khác hoặc reverse proxy.

#### Sửa đổi và quy ước đặt tên đúng:
- **Đổi tên file:** Từ `config.yml` thành `restaurant-service.yml`.
- **Nếu phân tách môi trường:**
  - `restaurant-service-dev.yml` (dùng cho môi trường Development).
  - `restaurant-service-prod.yml` (dùng cho môi trường Production).
  - `restaurant-service-staging.yml` (dùng cho môi trường Staging/Test).

---

### 1.2. Phân tích rủi ro của Plaintext Password và Cấu hình mã hóa `{cipher}`

#### Rủi ro khi lưu mật khẩu dạng văn bản thuần (`RestaurantPass123`):
1. **Lỗ hổng rò rỉ thông tin trong Git Repository (Data Leakage):**
   - Git repository là nơi nhiều thành viên trong dự án (Dev, QA, DevOps) cùng có quyền truy cập. Bất kỳ ai clone repo đều đọc được mật khẩu database production.
   - Khi repo được push lên các dịch vụ lưu trữ (GitHub, GitLab, Bitbucket), nếu repo bị lộ quyền public hoặc tài khoản bị xâm nhập, kẻ tấn công sẽ có ngay toàn quyền truy cập cơ sở dữ liệu `restaurants_db`.
2. **Lưu vết vĩnh viễn trong Git History:**
   - Git lưu lại toàn bộ commit log (`git log -p`). Dù sau này lập trình viên nhận ra lỗi và sửa mật khẩu bằng một commit mới, mật khẩu cũ vẫn nằm trong lịch sử commit và rất dễ bị khai thác thông qua các công cụ rà quét (Git leaks scanners).
3. **Vi phạm tiêu chuẩn bảo mật doanh nghiệp (Compliance & Regulatory Standards):**
   - Vi phạm các tiêu chuẩn an toàn bảo mật thông tin như **PCI-DSS** (bảo vệ dữ liệu thanh toán), **ISO/IEC 27001**, **GDPR**, và vi phạm tiêu chuẩn **DevSecOps** cơ bản ("No hardcoded secrets").

#### Khắc phục bằng cú pháp mã hóa `{cipher}` của Spring Cloud Config:
- **Cơ chế:** Spring Cloud Config Server tích hợp mô-đun mã hóa sử dụng thuật toán đối xứng (Symmetric Encryption như AES-256) hoặc bất đối xứng (Asymmetric RSA với Public/Private key).
- **Quy trình:**
  1. Chuỗi mật khẩu ban đầu được gửi đến endpoint mã hóa của Config Server: `POST /encrypt`.
  2. Config Server trả về chuỗi cipher đã mã hóa.
  3. Đặt chuỗi này vào file cấu hình trên Git với tiền tố `{cipher}`.
  4. Phải bọc toàn bộ giá trị trong dấu ngoặc kép hoặc ngoặc đơn `'...'` để YAML parser hiểu là kiểu chuỗi (String), không bị lỗi cú pháp ký tự ngoặc nhọn `{}`.
  5. Khi client yêu cầu cấu hình, Config Server tự động giải mã và trả về giá trị thực qua kết nối HTTPS bảo mật.

**Cấu hình sửa đổi chuẩn:**
```yaml
spring:
  datasource:
    password: '{cipher}AQA7b92KxLfNm84QeZ1uP0oRstVwXyZ3AbCdEfGhIjKlMnOpQrStUvWxYz1234567890=='
```

---

## 2. NỘI DUNG 3 FILE CẤU HÌNH ĐÃ CHUẨN HÓA

### 2.1. File `restaurant-service.yml`
*(Đường dẫn: `config-repo/restaurant-service.yml`)*

```yaml
# ===================================================================
# Cấu hình Centralized Configuration cho restaurant-service
# ===================================================================
spring:
  application:
    name: restaurant-service
  datasource:
    url: jdbc:mysql://foodx-cluster.local:3306/restaurants_db
    username: restaurant_service_user
    # Mật khẩu đã được mã hóa theo chuẩn Spring Cloud Config Cipher
    password: '{cipher}AQA7b92KxLfNm84QeZ1uP0oRstVwXyZ3AbCdEfGhIjKlMnOpQrStUvWxYz1234567890=='
  jpa:
    hibernate:
      ddl-auto: update
    show-sql: false
    properties:
      hibernate:
        dialect: org.hibernate.dialect.MySQL8Dialect

server:
  port: 8085

# Cấu hình Actuator phục vụ giám sát và Dynamic Refresh
management:
  endpoints:
    web:
      exposure:
        include: health,info,refresh
```

---

### 2.2. File `order-service.yml`
*(Đường dẫn: `config-repo/order-service.yml`)*

```yaml
# ===================================================================
# Cấu hình Centralized Configuration cho order-service
# ===================================================================
spring:
  application:
    name: order-service
  datasource:
    url: jdbc:mysql://foodx-cluster.local:3306/orders_db
    username: order_service_user
    # Mật khẩu đã được mã hóa an toàn
    password: '{cipher}AQBlk82N90VcxPlmn48QaZopL991KaBvCx0192384756abcdefgh1234567890=='
  jpa:
    hibernate:
      ddl-auto: update
    show-sql: false
    properties:
      hibernate:
        dialect: org.hibernate.dialect.MySQL8Dialect

server:
  port: 8086

management:
  endpoints:
    web:
      exposure:
        include: health,info,refresh
```

---

### 2.3. File `delivery-service.yml`
*(Đường dẫn: `config-repo/delivery-service.yml`)*

```yaml
# ===================================================================
# Cấu hình Centralized Configuration cho delivery-service
# ===================================================================
spring:
  application:
    name: delivery-service
  datasource:
    url: jdbc:mysql://foodx-cluster.local:3306/delivery_db
    username: delivery_service_user
    # Mật khẩu đã được mã hóa an toàn
    password: '{cipher}AQCD981ZxqVbn710MlOpQrstU902VaBxYz0987654321qwertyuiop1234567890=='
  jpa:
    hibernate:
      ddl-auto: update
    show-sql: false
    properties:
      hibernate:
        dialect: org.hibernate.dialect.MySQL8Dialect

server:
  port: 8087

management:
  endpoints:
    web:
      exposure:
        include: health,info,refresh
```

---

## 3. TỔNG KẾT VÀ BÀI HỌC KINH NGHIỆM

1. **Tuân thủ quy ước đặt tên (Convention over Configuration):** Tên file cấu hình trên Git phải đồng bộ tuyệt đối với `spring.application.name` của microservice để Config Server tự động khám phá và liên kết chính xác.
2. **Bảo mật Secret trong GitOps:** Tuyệt đối không đưa mật khẩu, token, private key ở dạng thô vào Git. Luôn tận dụng cơ chế `{cipher}` của Spring Cloud Config hoặc các giải pháp Secret Management chuyên dụng như HashiCorp Vault / AWS Secrets Manager.
3. **Phân tách môi trường rõ ràng:** Tạo các profile riêng biệt (`-dev.yml`, `-prod.yml`) để kiểm soát độc lập tài nguyên giữa các môi trường triển khai.
