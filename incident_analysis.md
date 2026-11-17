# BÁO CÁO PHÂN TÍCH SỰ CỐ PIPELINE CI (`notification-service`)

**Người thực hiện:** DevOps / Technical Lead  
**Dự án:** QuickBite — Microservices Platform  
**Service gặp sự cố:** `notification-service`  
**Trạng thái:** Đã xác định nguyên nhân gốc rễ & Đề xuất giải pháp khắc phục  

---

## 1. Tóm tắt thông báo lỗi

Khi lập trình viên đẩy mã nguồn mới lên repository của `notification-service`, job build trên GitHub Actions đã thất bại với ngoại lệ (exception) sau:

```text
java.lang.UnsupportedClassVersionError: class file has been compiled by a more recent version of the Java Runtime (class file version 65.0), this version of the Java Runtime only recognizes class file versions up to 61.0
```

---

## 2. Phân tích nguyên nhân gốc rễ

### 2.1. Bối cảnh xảy ra lỗi trong Pipeline CI
Lỗi `UnsupportedClassVersionError` xảy ra khi **Java Virtual Machine (JVM) đang thực thi trên môi trường CI cố gắng nạp (load) hoặc thực thi một file bytecode (`.class`) được biên dịch bởi một phiên bản trình biên dịch Java (`javac`) mới hơn phiên bản JVM hiện tại**. 

Trong quy trình CI của một ứng dụng Spring Boot/Gradle, lỗi này thường bùng phát ở hai thời điểm:
1. **Trong quá trình biên dịch/đóng gói:** Một thư viện phụ thuộc (dependency), plugin của Gradle/Maven, hoặc một module nội bộ đã được biên dịch sẵn bằng JDK mới hơn đang được nạp vào classpath của tiến trình build chạy trên JDK cũ hơn.
2. **Trong quá trình khởi chạy ứng dụng / chạy Unit Test:** Mã nguồn được biên dịch với target bytecode của phiên bản mới nhưng lại được chạy bởi một môi trường JRE/JDK phiên bản cũ hơn.

### 2.2. Vai trò của số hiệu "Class File Version" và cơ sở tra cứu
Mỗi file `.class` trong Java đều chứa một cặp số nguyên ở phần header (major.minor version) dùng để định danh phiên bản định dạng bytecode của file đó. JVM sử dụng con số này để kiểm tra tính tương thích ngược trước khi thực thi: **JVM phiên bản $X$ chỉ có thể chạy được các class file có major version $\le X$**.

Dựa theo **Đặc tả Máy ảo Java chính thức của Oracle (Java Virtual Machine Specification — JVMS, Chương 4: *The `class` File Format*)** và danh sách mã định danh của OpenJDK, ta có quy tắc ánh xạ tuyến tính theo công thức:
$$\text{Major Version} = 44 + \text{Phiên bản Java SE}$$

Bảng tra cứu đối chiếu các phiên bản Java phổ biến:

| Phiên bản Java SE | Class File Major Version | Ghi chú |
| :--- | :--- | :--- |
| Java 8 | `52.0` | LTS |
| Java 11 | `55.0` | LTS |
| **Java 17** | **`61.0`** | **LTS (Giới hạn tối đa của Runner CI hiện tại)** |
| **Java 21** | **`65.0`** | **LTS (Phiên bản phát triển của `notification-service`)** |

**Kết luận từ thông số lỗi:**
* **`class file version 65.0`**: Mã nguồn hoặc thư viện của `notification-service` yêu cầu/được biên dịch bằng **Java 21**.
* **`only recognizes class file versions up to 61.0`**: Môi trường Java Runtime đang chạy trên GitHub Actions runner chỉ là **Java 17**.

### 2.3. Sự khác biệt cốt lõi giữa máy cá nhân (Local) và GitHub Actions
Lỗi này không xảy ra trên máy của lập trình viên nhưng lại xuất hiện trên GitHub Actions do **sự mất đồng bộ (environment drift) giữa môi trường phát triển cục bộ và môi trường tích hợp liên tục**:
* **Trên máy cá nhân (Local):** Lập trình viên đã cài đặt và trỏ biến môi trường `JAVA_HOME` tới **JDK 21**, đồng thời cấu hình trong `build.gradle` (ví dụ `sourceCompatibility` hoặc `toolchain`) cũng hướng tới Java 21. Mọi thao tác biên dịch và chạy test đều diễn ra đồng nhất trên nền tảng JDK 21.
* **Trên GitHub Actions (CI):** Runner trên đám mây là một máy ảo sạch (ephemeral environment), hoàn toàn độc lập với máy cá nhân của lập trình viên. Phiên bản Java mà runner sử dụng phụ thuộc 100% vào khai báo trong file cấu hình `.github/workflows/ci.yml` (hoặc phiên bản Java mặc định được cài sẵn trên image `ubuntu-latest`). Hiện tại, file cấu hình CI của `notification-service` đang thiết lập **Java 17** (phiên bản cũ từ các thiết lập mẫu trước đó) thay vì nâng cấp lên **Java 21**.

---

## 3. Đề xuất giải pháp khắc phục

### 3.1. Vị trí cần điều chỉnh trong file `.github/workflows/ci.yml`
Cần kiểm tra bước thiết lập môi trường Java sử dụng action `actions/setup-java` trong file workflow và cập nhật tham số `java-version` từ `'17'` lên `'21'` để khớp với phiên bản phát triển của dự án.

**Cấu hình minh họa phần cần điều chỉnh:**

```yaml
name: Notification Service CI

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout source code
        uses: actions/checkout@v4

      # [VỊ TRÍ CẦN ĐIỀU CHỈNH]
      # Cập nhật java-version từ '17' lên '21' để hỗ trợ class file version 65.0
      - name: Setup Java JDK 21
        uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'

      - name: Build and Test with Gradle
        run: |
          chmod +x ./gradlew
          ./gradlew clean build
```

**Giải thích lý do khắc phục được lỗi:**  
Khi thay đổi `java-version: '21'`, action `actions/setup-java@v4` sẽ tải về, cài đặt JDK 21 (Eclipse Temurin) và cập nhật biến môi trường `JAVA_HOME` cùng `PATH` trên runner. Nhờ đó, JVM trên môi trường CI có khả năng nhận diện và xử lý các file bytecode lên tới version `65.0`, loại bỏ hoàn toàn xung đột phiên bản với mã nguồn dự án.

### 3.2. Nguyên tắc phòng ngừa lỗi tái diễn trong tương lai
Để tránh tình trạng *"chạy được trên máy em nhưng lỗi trên CI"* do lệch phiên bản công cụ, đội ngũ kỹ thuật nên áp dụng các nguyên tắc chuẩn hóa sau:

1. **Quản lý phiên bản ngôn ngữ bằng file cấu hình trong Repository (Single Source of Truth):**
   * Tạo file `.java-version` (hoặc `.sdkmanrc`, `.tool-versions`) đặt ngay tại thư mục gốc của repository (chứa nội dung `21`).
   * Cấu hình `actions/setup-java@v4` đọc trực tiếp từ file này thông qua thuộc tính `java-version-file: '.java-version'` thay vì hardcode số phiên bản trong YAML. Khi dự án nâng cấp Java, lập trình viên chỉ cần sửa file này trong cùng một commit với code.
2. **Sử dụng Gradle Java Toolchains:**
   * Khai báo tường minh phiên bản Java yêu cầu ngay trong file `build.gradle`:
     ```groovy
     java {
         toolchain {
             languageVersion = JavaLanguageVersion.of(21)
         }
     }
     ```
   * Cơ chế này buộc Gradle tự động tìm kiếm hoặc tải về đúng JDK 21 để biên dịch, đảm bảo tính nhất quán tuyệt đối bất kể hệ điều hành hay phiên bản Java mặc định của máy chạy là gì.
3. **Đồng bộ quy trình cập nhật hạ tầng (Infrastructure as Code):**
   * Mọi thay đổi liên quan đến phiên bản ngôn ngữ lập trình, framework hoặc thư viện cốt lõi phải được ghi nhận trong tài liệu kỹ thuật của service và kiểm tra thông qua Pull Request trên nhánh phụ trước khi merge vào nhánh chính (`main`).