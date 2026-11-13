# LAB 3 – ĐỀ 5: MẠCH PHÁT HIỆN TÉ NGÃ VỚI ESP32, MPU6050, FREERTOS VÀ MQTT

**Sinh viên:** Nguyễn Văn Hoàng Phi  
**MSSV:** 231432312  
**Đề tài:** Thiết kế hệ thống phát hiện té ngã sử dụng ESP32, cảm biến MPU6050, hệ điều hành thời gian thực FreeRTOS và giao thức MQTT.

**Link mô phỏng Wokwi:**  
https://wokwi.com/projects/480585417449540609

---

## 1. Giới thiệu đề tài

Hệ thống phát hiện té ngã được xây dựng dựa trên vi điều khiển ESP32 kết hợp với cảm biến gia tốc và góc nghiêng MPU6050.

Hệ thống có nhiệm vụ thu thập dữ liệu gia tốc theo 3 trục X, Y, Z, tính gia tốc tổng hợp và góc nghiêng Roll, Pitch để nhận biết dấu hiệu té ngã.

Dữ liệu sau khi xử lý được hiển thị trên Serial Monitor và truyền lên MQTT Broker thông qua WiFi. Đồng thời, giao diện Web Dashboard cho phép người dùng theo dõi thông số theo thời gian thực, nhận cảnh báo và điều khiển LED, còi bằng chế độ thủ công.

Chương trình được tổ chức bằng FreeRTOS với 3 Task chạy trên 2 Core của ESP32 nhằm đảm bảo các chức năng đọc cảm biến, truyền dữ liệu và cảnh báo hoạt động độc lập.

## 2. Chức năng chính

- Đọc dữ liệu gia tốc 3 trục X, Y, Z từ cảm biến MPU6050.
- Tính gia tốc tổng hợp (Acceleration Magnitude).
- Tính góc nghiêng Roll và Pitch.
- Lấy mẫu dữ liệu với chu kỳ 50 ms (20 mẫu/giây).
- Phát hiện té ngã dựa trên hai điều kiện:
  - Gia tốc tổng hợp ≥ 2.2 g.
  - Góc nghiêng Roll hoặc Pitch ≥ 55° trong vòng 2 giây sau va chạm.
- Bật LED và còi cảnh báo trong 5 giây khi phát hiện dấu hiệu té ngã.
- Gửi dữ liệu cảm biến lên MQTT Broker qua WiFi.
- Hiển thị dữ liệu trên Web Dashboard theo thời gian thực.
- Hiển thị 2 biểu đồ: gia tốc tổng hợp và góc nghiêng Roll/Pitch.
- Điều khiển bật/tắt LED và còi từ giao diện Web bằng chế độ Manual.
- Hiển thị trạng thái kết nối MQTT và cảnh báo té ngã trên Web.

## 3. Phần cứng sử dụng

| STT | Thiết bị | Chức năng |
|---|---|---|
| 1 | ESP32 DevKit | Vi điều khiển trung tâm |
| 2 | MPU6050 | Đo gia tốc và góc nghiêng |
| 3 | LED đỏ | Hiển thị cảnh báo |
| 4 | Buzzer | Phát âm thanh cảnh báo |
| 5 | Điện trở | Hạn dòng cho LED |

### 3.1. Sơ đồ kết nối chân

| Thiết bị | Chân thiết bị | Chân ESP32 |
|---|---|---|
| MPU6050 | VCC | 3V3 |
| MPU6050 | GND | GND |
| MPU6050 | SDA | GPIO21 |
| MPU6050 | SCL | GPIO22 |
| LED đỏ | Anode (qua điện trở) | GPIO18 |
| LED đỏ | Cathode | GND |
| Buzzer | Cực (+) | GPIO16 |
| Buzzer | Cực (-) | GND |

### 3.2. Sơ đồ mạch

Sơ đồ kết nối các linh kiện được thiết kế và mô phỏng trên nền tảng Wokwi.

<img width="1129" height="595" alt="so-do-wokwi" src="https://github.com/user-attachments/assets/f7db806e-b7be-412a-80da-783c80e1e5b3" />


## 4. Kiến trúc hệ thống FreeRTOS

Hệ thống sử dụng 3 Task độc lập, phân bổ trên 2 Core của ESP32.

### 4.1. Các Task trong hệ thống

| Task | Core | Độ ưu tiên | Stack | Chức năng |
|---|---|---|---|---|
| Task_ReadData | 1 | 2 | 4096 | Đọc MPU6050, tính toán thông số và gửi dữ liệu vào Queue |
| Task_SendData | 0 | 1 | 6144 | Kết nối WiFi, MQTT và gửi dữ liệu lên Broker |
| Task_Alert | 1 | 3 | 4096 | Xử lý phát hiện té ngã và điều khiển LED, Buzzer |

### 4.2. Cơ chế giao tiếp giữa các Task

Hệ thống sử dụng hai hàng đợi Queue:

| Tên | Loại | Độ dài | Chức năng |
|---|---|---|---|
| dataQueue | Queue | 10 | Truyền dữ liệu cảm biến từ Task_ReadData sang Task_SendData |
| dataAlert | Queue | 1 | Luôn giữ mẫu dữ liệu mới nhất cho Task_Alert |

Ngoài ra, hệ thống sử dụng các biến trạng thái:

- `manualLed`: Trạng thái điều khiển LED thủ công.
- `manualBuzzer`: Trạng thái điều khiển còi thủ công.
- `fallDetected`: Trạng thái phát hiện té ngã.

Task_Alert có độ ưu tiên cao nhất để đảm bảo phản ứng nhanh khi phát hiện dấu hiệu té ngã.

## 5. Thuật toán phát hiện té ngã

Hệ thống sử dụng thuật toán phát hiện té ngã gồm hai giai đoạn:

**Giai đoạn 1 – Phát hiện va chạm:**

- Tính gia tốc tổng hợp từ 3 trục X, Y, Z.
- Nếu gia tốc tổng hợp ≥ 2.2 g, hệ thống ghi nhận thời điểm va chạm.

Công thức:

`A = sqrt(ax² + ay² + az²)`

**Giai đoạn 2 – Xác nhận góc nghiêng:**

- Sau khi phát hiện va chạm, hệ thống kiểm tra góc nghiêng trong vòng 2 giây.
- Nếu giá trị tuyệt đối của Roll hoặc Pitch ≥ 55°, hệ thống xác định có dấu hiệu té ngã.

**Giai đoạn 3 – Cảnh báo:**

- Bật LED đỏ và còi Buzzer trong 5 giây.
- Cập nhật trạng thái `fallDetected = true`.
- Gửi trạng thái cảnh báo lên Web thông qua MQTT.
- Sau thời gian cảnh báo, hệ thống tự động tắt cảnh báo và xóa trạng thái.

Việc kết hợp điều kiện va chạm và góc nghiêng giúp hạn chế các trường hợp cảnh báo nhầm.

## 6. Giao tiếp MQTT

Hệ thống sử dụng MQTT Broker công cộng EMQX để trao đổi dữ liệu giữa ESP32 và Web Dashboard.

### 6.1. Thông tin kết nối

| Thông số | Giá trị |
|---|---|
| WiFi SSID | Wokwi-GUEST |
| MQTT Broker | broker.emqx.io |
| MQTT TCP Port | 1883 |
| WebSocket URL | wss://broker.emqx.io:8084/mqtt |
| Topic Prefix | fall_demo_esp32_741852 |

### 6.2. Các MQTT Topic sử dụng

| Topic | Hướng truyền | Chức năng |
|---|---|---|
| `fall_demo_esp32_741852/sensor` | ESP32 → Web | Gửi dữ liệu cảm biến dạng JSON |
| `fall_demo_esp32_741852/status` | ESP32 → Web | Gửi trạng thái ONLINE |
| `fall_demo_esp32_741852/control/led` | Web → ESP32 | Điều khiển LED ON/OFF |
| `fall_demo_esp32_741852/control/buzzer` | Web → ESP32 | Điều khiển Buzzer ON/OFF |

Dữ liệu cảm biến được gửi dưới dạng JSON, bao gồm:

- `ax`, `ay`, `az`: Gia tốc 3 trục.
- `accMagnitude`: Gia tốc tổng hợp.
- `roll`: Góc nghiêng Roll.
- `pitch`: Góc nghiêng Pitch.
- `fall`: Trạng thái phát hiện té ngã.
- `uptime`: Thời gian hoạt động của ESP32.

**Lưu ý:** Topic Prefix phải giống nhau giữa chương trình ESP32 và giao diện Web để đảm bảo truyền nhận dữ liệu chính xác.

## 7. Giao diện Web Dashboard

Giao diện Web được xây dựng bằng HTML, CSS và JavaScript trong file `Web_phat_hien_te_nga.html`.

Các thư viện sử dụng:

- **MQTT.js:** Kết nối và trao đổi dữ liệu với MQTT Broker thông qua WebSocket Secure.
- **Chart.js:** Hiển thị biểu đồ dữ liệu theo thời gian thực.

### 7.1. Các thông số hiển thị

- Gia tốc X, Y, Z (g).
- Gia tốc tổng hợp (g).
- Góc nghiêng Roll (°).
- Góc nghiêng Pitch (°).
- Trạng thái phát hiện té ngã.
- Thời gian hoạt động của ESP32 (giây).

### 7.2. Biểu đồ thời gian thực

Giao diện có hai biểu đồ:

1. Biểu đồ gia tốc tổng hợp: Theo dõi sự thay đổi gia tốc của thiết bị.
2. Biểu đồ góc nghiêng: Hiển thị hai giá trị Roll và Pitch.

Mỗi biểu đồ lưu tối đa 30 điểm dữ liệu gần nhất.

### 7.3. Điều khiển thiết bị

Web Dashboard cung cấp hai nút điều khiển thủ công:

- LED: Bật/tắt đèn cảnh báo.
- Buzzer: Bật/tắt còi cảnh báo.

Khi người dùng nhấn nút, Web gửi lệnh ON/OFF qua MQTT đến ESP32 để thực hiện điều khiển.

### 7.4. Cảnh báo trạng thái

- Chấm trạng thái MQTT hiển thị tình trạng kết nối.
- Khi phát hiện té ngã, Web hiển thị banner đỏ nhấp nháy với nội dung: **CẢNH BÁO: PHÁT HIỆN DẤU HIỆU TÉ NGÃ**.

## 8. Hướng dẫn chạy chương trình

### Bước 1: Mở mô phỏng Wokwi

Truy cập đường dẫn:

https://wokwi.com/projects/480585417449540609

### Bước 2: Khởi động ESP32

- Chạy mô phỏng trên Wokwi.
- ESP32 kết nối WiFi `Wokwi-GUEST`.
- Khởi tạo cảm biến MPU6050.
- Tạo và chạy 3 Task FreeRTOS.
- Kết nối MQTT Broker EMQX.

### Bước 3: Kiểm tra dữ liệu Serial Monitor

Mở Serial Monitor để theo dõi các thông số:

- Gia tốc AX, AY, AZ.
- Gia tốc tổng hợp.
- Góc Roll và Pitch.
- Trạng thái kết nối WiFi và MQTT.
- Thông báo khi phát hiện té ngã.

### Bước 4: Mở giao diện Web

Mở file `Web_phat_hien_te_nga.html` bằng trình duyệt web có kết nối Internet.

<img width="1836" height="910" alt="web-trang-thai-binh-thuong" src="https://github.com/user-attachments/assets/ce0b9110-23c3-4543-a509-2a621768421a" />


Giao diện sẽ kết nối đến MQTT Broker thông qua WebSocket Secure để nhận dữ liệu từ ESP32.

### Bước 5: Kiểm tra hệ thống

- Theo dõi các thông số cảm biến trên Web.
- Quan sát hai biểu đồ cập nhật theo thời gian thực.
- Mô phỏng tình huống va chạm và thay đổi góc nghiêng để kiểm tra chức năng phát hiện té ngã.
- Kiểm tra LED và còi cảnh báo.
- Thử hai nút điều khiển LED và Buzzer trên Web.

## 9. Kết quả thực hiện

Sau khi hoàn thành, hệ thống đạt được các kết quả:

- ESP32 kết nối WiFi và MQTT Broker thành công.
- Cảm biến MPU6050 đọc dữ liệu gia tốc và góc nghiêng.
- Ba Task FreeRTOS hoạt động độc lập trên hai Core.
- Dữ liệu được truyền giữa các Task thông qua Queue.
- Hệ thống phát hiện dấu hiệu té ngã dựa trên ngưỡng gia tốc và góc nghiêng.
- LED và còi hoạt động khi phát hiện dấu hiệu té ngã.
- Web Dashboard hiển thị thông số và biểu đồ theo thời gian thực.
- Người dùng có thể điều khiển LED và còi từ giao diện Web.

## 10. Kết luận

Thông qua bài Lab 3, hệ thống phát hiện té ngã sử dụng ESP32 và MPU6050 đã được xây dựng và mô phỏng thành công trên Wokwi.

Bài thực hiện đã vận dụng được hệ điều hành thời gian thực FreeRTOS để tổ chức chương trình thành ba Task độc lập, sử dụng Queue để trao đổi dữ liệu và phân bổ công việc trên hai Core của ESP32.

Bên cạnh đó, việc kết hợp MQTT và Web Dashboard giúp hệ thống có khả năng giám sát dữ liệu từ xa, hiển thị cảnh báo và điều khiển thiết bị thông qua Internet.

Qua bài thực hành, em hiểu rõ hơn về RTOS, cơ chế giao tiếp giữa các Task, giao thức MQTT và quy trình xây dựng một hệ thống IoT hoàn chỉnh từ cảm biến đến giao diện người dùng.


**Sinh viên thực hiện:** Nguyễn Văn Hoàng Phi  
**Mã sinh viên:** 231432312  
**Môn học:** Hệ thống Nhúng
