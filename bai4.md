# Non-Functional Requirements (NFR) - Guai-api (Shop AI Platform)

Tài liệu này đặc tả các yêu cầu phi chức năng (Non-Functional Requirements - NFR) cho hệ thống **Guai-api** (Spring Boot + Java) thuộc nền tảng thương mại điện tử **Shop AI**, tập trung vào khả năng chịu tải cao, tối ưu cơ sở dữ liệu MySQL và bảo mật hệ thống thông qua JWT trong các sự kiện lưu lượng truy cập đột biến (Flash Sale, 11/11, Black Friday).

---

## 1. Bảng Đặc tả Yêu cầu Phi chức năng (NFR Table)

| Ma NFR | Khía cạnh | Mô tả yêu cầu | Chỉ tiêu kỹ thuật | Phương pháp đo lường |
| :--- | :--- | :--- | :--- | :--- |
| **NFR-RT-01** | **Thời gian phản hồi**<br>(Response Time)<br>Tra cứu chi tiết sản phẩm | Thời gian phản hồi của endpoint lấy thông tin chi tiết sản phẩm (`GET /products/{id}`) phục vụ khách hàng xem sản phẩm trong sự kiện Flash Sale. | - **Bình thường (Normal Load):** p95 < 100ms, p99 < 200ms.<br>- **Tải cao (Peak Load):** p95 < 250ms, p99 < 400ms.<br>- **Ngưỡng cảnh báo (Warning):** p95 > 300ms kéo dài trong 1 phút.<br>- **Ngưỡng lỗi (Error/Timeout):** > 800ms (HTTP 504 Gateway Timeout hoặc kích hoạt Circuit Breaker). | Giả lập tải bằng **k6** hoặc **JMeter** (tăng dần đến peak CCU dự kiến). Giám sát thời gian phản hồi thực tế qua APM (**Prometheus + Grafana** hoặc **Datadog**). |
| **NFR-RT-02** | **Thời gian phản hồi**<br>(Response Time)<br>Tìm kiếm & Lọc danh mục | Thời gian phản hồi của endpoint tìm kiếm và lọc danh sách sản phẩm có phân trang (`GET /products?category=...&page=...`). | - **Bình thường (Normal Load):** p95 < 250ms, p99 < 400ms.<br>- **Tải cao (Peak Load):** p95 < 600ms, p99 < 1000ms.<br>- **Ngưỡng cảnh báo (Warning):** p95 > 800ms.<br>- **Ngưỡng lỗi (Error/Timeout):** > 1500ms. | Đo lường độ trễ từ API Gateway (Spring Cloud Gateway) đến microservice xử lý sản phẩm thông qua Distributed Tracing (**Spring Cloud Sleuth/Micrometer + Zipkin/Jaeger**). |
| **NFR-DB-01** | **Hiệu suất MySQL**<br>Indexing bảng Sản phẩm (`products`) | Chiến lược lập chỉ mục (Indexing) tối ưu cho bảng sản phẩm để hỗ trợ hiển thị nhanh và giảm tải Disk I/O. | - **Composite Index:** `idx_category_status` trên `(category_id, status)` giúp truy vấn danh sách sản phẩm theo danh mục nhanh chóng.<br>- **Covering Index:** `idx_prod_listing` trên `(status, price, id, name)` phục vụ sắp xếp theo giá không cần truy cập bảng gốc (Index-Only Scan).<br>- **Single Index:** `idx_sku` (Unique) trên `sku` phục vụ tra cứu chính xác.<br>- **Chỉ tiêu thực thi (Execution Time):** < 10ms (Normal), < 30ms (Peak). | Chạy phân tích truy vấn bằng cú pháp `EXPLAIN` hoặc `EXPLAIN ANALYZE` trong MySQL để đảm bảo không bị table scan. Cấu hình **Slow Query Log** với ngưỡng `long_query_time = 0.05` (50ms). |
| **NFR-DB-02** | **Hiệu suất MySQL**<br>Indexing bảng Giỏ hàng (`cart_items`) | Tối ưu hóa chỉ mục cho các thao tác thêm sản phẩm vào giỏ hàng và đọc thông tin giỏ hàng của người dùng khi checkout. | - **Composite Index:** `idx_user_product` trên `(user_id, product_id)` để kiểm tra nhanh sự tồn tại của sản phẩm trong giỏ.<br>- **Covering Index:** `idx_user_cart` trên `(user_id, product_id, quantity)` để lấy nhanh toàn bộ giỏ hàng của user.<br>- **Chỉ tiêu thực thi (Execution Time):** SELECT < 5ms, INSERT/UPDATE < 15ms. | Sử dụng công cụ giám sát hiệu năng database (**Percona Monitoring and Management - PMM** hoặc **MySQL Enterprise Monitor**) và trace SQL thông qua Spring Boot JPA logging. |
| **NFR-DB-03** | **Hiệu suất MySQL**<br>Indexing bảng Đơn hàng (`orders`, `order_items`) | Bảo đảm tốc độ ghi nhận đơn hàng lớn trong Flash Sale và tra cứu lịch sử mua hàng của khách hàng mà không gây lock bảng. | - **Composite Index:** `idx_user_created` trên `(user_id, created_at DESC)` hỗ trợ hiển thị lịch sử đơn hàng mới nhất.<br>- **Composite Index:** `idx_status_created` trên `(status, created_at)` phục vụ hệ thống hậu cần quét đơn hàng cần xử lý.<br>- **Chỉ tiêu thực thi (Execution Time):** SELECT < 15ms, INSERT/UPDATE < 30ms (sử dụng Queue/Kafka để giảm tải ghi trực tiếp nếu cần). | Giám sát chỉ số connection pool (**HikariCP** metrics: active connections, pending connections, connection acquire time) và các lock wait timeout trong MySQL. |
| **NFR-SEC-01** | **Bảo mật và độ trễ JWT**<br>Thời gian cấp phát Token | Độ trễ tối đa của luồng sinh mã JWT sau khi người dùng xác thực thành công (Login hoặc Refresh Token). | - **Thời gian cấp phát tối đa:** < 20ms (Normal Load), < 50ms (Peak Load) cho toàn bộ logic tạo và ký JWT.<br>- **Mức độ tiêu thụ tài nguyên:** CPU cho tác vụ mã hóa/ký số không được chiếm quá 15% tổng năng lực CPU của API node. | Đo lường bằng các bài test benchmark hiệu năng sử dụng **JMH (Java Microbenchmark Harness)** cho hàm sinh token. Giám sát JVM CPU usage qua **Micrometer/Prometheus**. |
| **NFR-SEC-02** | **Bảo mật và độ trễ JWT**<br>Thời hạn mã xác thực (Lifespan) | Thiết lập vòng đời hợp lý cho Access Token và Refresh Token để giảm thiểu thiệt hại nếu token bị rò rỉ. | - **Access Token:** Thời hạn hiệu lực tối đa là **15 phút** (Short-lived).<br>- **Refresh Token:** Thời hạn hiệu lực tối đa là **7 ngày** (Long-lived). | Rà soát và kiểm thử cấu hình biến môi trường trong file cấu hình Spring `application.yml` (hoặc Spring Cloud Config Server). |
| **NFR-SEC-03** | **Bảo mật và độ trễ JWT**<br>Yêu cầu kỹ thuật ký mã | Sử dụng thuật toán mã hóa mạnh mẽ và cấu hình an toàn, phù hợp với kiến trúc chịu tải cao và xác thực phân tán. | - **Thuật toán:** Sử dụng thuật toán ký bất đối xứng **RS256 (RSA với SHA-256)** với độ dài khóa tối thiểu **2048-bit** (hoặc **ES256 - ECDSA**).<br>- **Kiến trúc Verify:** Private Key được lưu trữ bảo mật tại Identity Provider (Auth Service). Các API Gateway hoặc Business Services sử dụng **JWKS (JSON Web Key Set)** endpoint để lấy Public Key verify token locally (không cần gọi API Auth Service), đảm bảo latency verify < 2ms. | Sử dụng thư viện **Nimbus-JWT** hoặc **jjwt** trong Spring Boot để triển khai. Kiểm tra thông qua các công cụ scan mã nguồn tĩnh (SAST như **SonarQube**, **Veracode**). |
| **NFR-SEC-04** | **Bảo mật và độ trễ JWT**<br>Thu hồi token bị đánh cắp | Chiến lược thu hồi token (Revocation) nhanh chóng trước khi hết hạn để ngăn chặn truy cập trái phép. | - **Access Token Revocation:** Sử dụng **Redis Blacklist** (In-memory storage). Lưu cặp `(jti: JWT ID, expiration_time)` vào Redis khi logout hoặc phát hiện nghi ngờ. Độ trễ kiểm tra blacklist tại Gateway/Service < 2ms.<br>- **Refresh Token Rotation (RTR):** Mỗi lần refresh token được sử dụng, hệ thống sẽ thu hồi nó và cấp một token mới. Nếu phát hiện một Refresh Token cũ (đã dùng) được gửi lên lần 2, lập tức vô hiệu hóa toàn bộ Token Family của user đó. | Viết Integration Test giả lập phát lại (replay attack) token cũ và token đã thu hồi. Kiểm tra latency truy vấn Redis Blacklist dưới tải cao bằng **k6**. |

---

## 2. Phân tích Chi tiết & Hướng dẫn Triển khai Kỹ thuật

### 2.1. Tối ưu hóa Response Time cho Endpoint Sản phẩm
Để đạt được các chỉ tiêu thời gian phản hồi cực thấp trong điều kiện tải cao (Peak Load):
- **Cơ chế Caching**: Áp dụng mô hình **Multi-level Caching**.
  - **L1 Cache (In-Memory / Caffeine Cache)**: Lưu trữ các sản phẩm hot/bán chạy ngay tại bộ nhớ RAM của ứng dụng Spring Boot để đạt thời gian phản hồi < 5ms.
  - **L2 Cache (Distributed / Redis)**: Lưu trữ danh mục sản phẩm và chi tiết sản phẩm chung. Thời gian phản hồi từ Redis đạt khoảng < 15ms.
  - **Cơ chế ghi đè Cache (Cache Eviction)**: Sử dụng mô hình **Write-through** hoặc **Cache-aside** kết hợp với **CDC (Change Data Capture)** từ MySQL qua Debezium để cập nhật cache tức thời khi có thay đổi về tồn kho hoặc giá sản phẩm, tránh hiện tượng stale data.
- **Circuit Breaker & Rate Limiting**: Tích hợp **Resilience4j** tại Spring Boot. Khi thời gian phản hồi của database vượt quá ngưỡng lỗi (800ms) liên tục, Circuit Breaker chuyển sang trạng thái *Open* để trả về dữ liệu cache cũ hoặc thông báo lỗi thân thiện thay vì làm nghẽn toàn bộ luồng xử lý của luồng xử lý Tomcat/Undertow.

### 2.2. Chiến lược Indexing trên MySQL để Tránh Bottleneck
MySQL (thường dùng InnoDB engine) rất dễ gặp hiện tượng nghẽn cổ chai I/O khi lượng truy vấn đồng thời tăng cao. Thiết lập index đúng là điều kiện tiên quyết:
- **Composite Index (Chỉ mục tổ hợp)**: 
  - Đặt các cột có độ chọn lọc cao (High Cardinality) lên trước. Ví dụ, trong `idx_category_status`, `category_id` nên đứng trước `status`.
  - Quy tắc Leftmost Prefix phải được tuân thủ nghiêm ngặt trong các câu truy vấn SQL của ứng dụng JPA.
- **Covering Index (Chỉ mục bao phủ)**:
  - Khi thực hiện truy vấn như `SELECT id, name, price FROM products WHERE status = 'ACTIVE' ORDER BY price ASC`, việc tạo index `(status, price, id, name)` giúp MySQL lấy toàn bộ dữ liệu cần thiết ngay từ index tree (B+ Tree) mà không cần thực hiện bước "Bookmark Lookup" (truy cập vào data page để lấy `name` hay `price`). Điều này giúp giảm tới 80% số lượng Disk Page Read.
- **Tránh Lock Bảng (Table Lock) & Lock Dòng (Row Lock) kéo dài**:
  - Đối với bảng `cart_items` và `orders`, hạn chế tối đa các truy vấn cập nhật diện rộng (Bulk Update). Sử dụng truy vấn dựa trên Primary Key hoặc Index duy nhất để InnoDB chỉ lock dòng cần thiết (Row-level Lock).

### 2.3. Thiết kế Cơ chế JWT chịu tải cao và Bảo mật
- **JWKS (JSON Web Key Set)**: 
  - Thay vì Gateway hoặc các microservices phải gọi REST API sang Auth Service để verify chữ ký của JWT (gây nghẽn mạng và tăng latency thêm 20-50ms), cấu hình Spring Security sử dụng JWKS. 
  - Gateway sẽ download Public Key của Auth Service về cache nội bộ và tự thực hiện verify chữ ký bằng CPU local. Latency verify giảm xuống < 1ms.
- **Redis Blacklist cho Access Token**:
  - Access Token là vô trạng thái (stateless), do đó để thu hồi nó khi người dùng logout hoặc khi phát hiện token bị lộ, ta lưu `jti` (mã định danh duy nhất của token) vào Redis với TTL = `remaining_time_to_expire`.
  - Mỗi khi request đi qua Gateway, Gateway sử dụng lệnh `EXISTS blacklist:<jti>` trên Redis. Redis được cấu hình dạng Cluster/Replicated để đảm bảo tốc độ phản hồi cực nhanh (< 2ms) và không trở thành điểm nghẽn đơn lẻ (SPOF).
- **Refresh Token Rotation (RTR) - Bảo vệ chuỗi Token**:
  - Lưu trữ Refresh Token Family trong database hoặc Redis dưới dạng cấu trúc cây/chuỗi liên kết. 
  - Khi client gửi Refresh Token $RT_1$ để lấy Access Token mới $AT_2$ và Refresh Token mới $RT_2$:
    - Hệ thống đánh dấu $RT_1$ đã sử dụng.
    - Nếu kẻ tấn công đánh cắp được $RT_1$ trước đó và gửi lên hệ thống, hệ thống sẽ phát hiện $RT_1$ đã được dùng. 
    - Lập tức kích hoạt cơ chế tự vệ: Vô hiệu hóa toàn bộ các Refresh Token tiếp theo trong nhánh ($RT_2$), ép buộc người dùng thực sự phải đăng nhập lại (re-authenticate) để sinh khóa mới, ngăn chặn triệt để hacker lợi dụng token trộm được.
