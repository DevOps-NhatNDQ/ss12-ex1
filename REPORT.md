# BÁO CÁO KỸ THUẬT BÀI 1: XÂY DỰNG QUY TRÌNH CI ĐA PHIÊN BẢN JDK (MATRIX STRATEGY) CHO JAVA SPRING BOOT GRADLE

- **Khóa học:** DevOps Chuyên Sâu
- **Mã bài tập:** SS12 - Exercise 1
- **Công nghệ áp dụng:** GitHub Actions, Matrix Strategy, Gradle Dependency Cache, Java 17 & Java 21 (Eclipse Temurin), Spring Boot 3.2.5

---

## 1. Mục tiêu và Ý nghĩa Kỹ thuật

### 1.1. Mục tiêu
- Xây dựng ma trận kiểm thử (**Matrix Strategy**) tự động hóa trong quy trình Continuous Integration (CI).
- Kiểm tra tính tương thích chéo của mã nguồn trên các phiên bản Java Runtime Environment (LTS) phổ biến hiện nay: **Java 17** và **Java 21**.
- Tối ưu hóa thời gian thực thi (Build Time) của các runner thông qua cơ chế lưu bộ đệm các gói phụ thuộc (**Gradle Dependency Caching**).

### 1.2. Tại sao cần Matrix Strategy?
Trong môi trường doanh nghiệp hiện đại, một ứng dụng có thể được triển khai trên các hạ tầng khác nhau hoặc các phiên bản JVM khác nhau. Việc chỉ kiểm thử trên một phiên bản Java duy nhất tiềm ẩn nguy cơ phát sinh lỗi tương thích (incompatible bytecode, deprecated APIs, JVM runtime flags). Matrix Strategy cho phép:
1. **Chạy song song (Parallel Execution):** Rút ngắn tổng thời gian kiểm thử thay vì chạy tuần tự.
2. **Bảo đảm chất lượng toàn diện:** Phát hiện sớm các lỗi cú pháp, tính năng hoặc thư viện bên thứ ba không hỗ trợ trên JDK cao hơn/thấp hơn.
3. **Tiết kiệm tài nguyên bảo trì:** Cùng một file cấu hình duy nhất, chỉ cần khai báo mảng phiên bản cần test.

---

## 2. Thiết kế Kịch bản CI (`.github/workflows/ci-matrix.yml`)

```yaml
name: Java Spring Boot CI Matrix

on:
  push:
    branches: [ "main" ]

jobs:
  test-matrix:
    name: Test on Java ${{ matrix.java-version }}
    runs-on: ubuntu-latest
    strategy:
      matrix:
        java-version: ['17', '21']
    steps:
      - name: Checkout repository
        uses: actions/checkout@v5

      - name: Setup Java JDK ${{ matrix.java-version }}
        uses: actions/setup-java@v5
        with:
          java-version: ${{ matrix.java-version }}
          distribution: 'temurin'
          cache: 'gradle'

      - name: Grant execute permission for gradlew
        run: chmod +x gradlew

      - name: Run Unit Tests with Gradle
        run: ./gradlew test
```

### Phân tích cấu hình chi tiết:
- `strategy.matrix.java-version: ['17', '21']`: GitHub Actions tạo ra 2 tiến trình độc lập (matrix sub-jobs):
  - Tiến trình 1: `matrix.java-version = '17'`
  - Tiến trình 2: `matrix.java-version = '21'`
- `actions/setup-java@v5`:
  - `distribution: 'temurin'`: Sử dụng bản phân phối OpenJDK Eclipse Temurin chuẩn hóa cao.
  - `cache: 'gradle'`: Tự động nhận diện cấu hình Gradle, tạo hash key dựa trên các file `build.gradle` và `gradle-wrapper.properties`. Khi chạy các commit tiếp theo hoặc sub-job thứ hai, các dependencies đã tải về sẽ được phục hồi ngay từ cache của GitHub Actions, giảm 50-70% thời gian tải thư viện.
- `chmod +x gradlew`: Đảm bảo Gradle Wrapper script có quyền thực thi (`x`) trên runner Linux `ubuntu-latest`.

---

## 3. Mô phỏng Kết quả Thực thi trên GitHub Actions

### 3.1. Phân nhánh Sub-jobs trong Tab Actions
Khi commit được push lên nhánh `main`, workflow `Java Spring Boot CI Matrix` tự động kích hoạt:
```text
Workflow: Java Spring Boot CI Matrix (#1)
Commit: feat: setup matrix CI with JDK 17 and 21

Jobs:
├── test-matrix (17) [Test on Java 17] - Success (34s)
└── test-matrix (21) [Test on Java 21] - Success (31s)
```

### 3.2. Trích xuất nhật ký thực thi của `Test on Java 17`
```text
Run actions/setup-java@v5
  Configuring Java 17 (temurin)
  Resolved Java version: 17.0.10
  Extracting Java archive...
  Setting up gradle cache...
  Cache restored successfully from key: Linux-gradle-e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855

Run chmod +x gradlew

Run ./gradlew test
> Task :compileJava
> Task :processResources
> Task :classes
> Task :compileTestJava
> Task :processTestResources
> Task :testClasses
> Task :test
  com.example.demo.DemoApplicationTests > contextLoads() PASSED
  com.example.demo.HelloControllerTest > testHelloEndpointReturnsOkAndSuccessStatus() PASSED

BUILD SUCCESSFUL in 14s
4 actionable tasks: 4 executed
```

### 3.3. Trích xuất nhật ký thực thi của `Test on Java 21`
```text
Run actions/setup-java@v5
  Configuring Java 21 (temurin)
  Resolved Java version: 21.0.2
  Extracting Java archive...
  Setting up gradle cache...
  Cache restored successfully from key: Linux-gradle-e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855

Run chmod +x gradlew

Run ./gradlew test
> Task :compileJava
> Task :processResources
> Task :classes
> Task :compileTestJava
> Task :processTestResources
> Task :testClasses
> Task :test
  com.example.demo.DemoApplicationTests > contextLoads() PASSED
  com.example.demo.HelloControllerTest > testHelloEndpointReturnsOkAndSuccessStatus() PASSED

BUILD SUCCESSFUL in 12s
4 actionable tasks: 4 executed
```

---

## 4. Hướng dẫn minh chứng hình ảnh nộp bài

Để hoàn thiện sản phẩm nộp bài theo yêu cầu đề bài:
1. Đẩy mã nguồn bài tập lên GitHub Repository của bạn.
2. Điều hướng vào menu **Actions** trên GitHub.
3. Chọn workflow run vừa thực thi.
4. Chụp ảnh màn hình bao gồm:
   - Tên workflow: **Java Spring Boot CI Matrix**.
   - Khung đồ thị hiển thị 2 sub-job chạy song song: `Test on Java 17` và `Test on Java 21` kèm biểu tượng tích xanh (Success).
   - Lưu ảnh vào thư mục hoặc gửi kèm file báo cáo.
