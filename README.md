# Lab 3 – Đề 5: Mạch phát hiện té ngã với ESP32, FreeRTOS và MQTT

**Sinh viên:** Nguyễn Văn Hoàng Phi – **MSSV:** 231432312

## 1. Giới thiệu
Hệ thống phát hiện té ngã dùng **ESP32** đọc gia tốc và góc nghiêng từ cảm biến **MPU6050**, hiển thị dữ liệu qua Serial, gửi lên **MQTT Broker** qua WiFi và hiển thị/điều khiển trên **Web Dashboard**. Chương trình được tổ chức bằng **FreeRTOS** với 3 Task chạy trên hai Core.

## 2. Chức năng chính
- Đọc gia tốc 3 trục, tính gia tốc tổng hợp, góc Roll và Pitch (20 mẫu/giây).
- Phát hiện té ngã theo 2 giai đoạn: va chạm (≥ 2.2 g) rồi nghiêng mạnh (≥ 55°) trong 2 giây.
- Cảnh báo bằng LED và còi trong 5 giây.
- Web hiển thị số liệu, 2 biểu đồ thời gian thực, banner cảnh báo và 2 nút điều khiển LED/còi (manual).

## 3. Phần cứng (mô phỏng trên Wokwi)
| Thiết bị | Chân thiết bị | Chân ESP32 |
|---|---|---|
| MPU6050 | VCC | 3V3 |
| MPU6050 | GND | GND |
| MPU6050 | SDA | GPIO21 |
| MPU6050 | SCL | GPIO22 |
| LED đỏ (qua điện trở) | Anode | GPIO18 |
| LED đỏ | Cathode | GND |
| Buzzer | Cực (+) | GPIO16 |
| Buzzer | Cực (−) | GND |

![Sơ đồ mạch](images/so-do-mach.png)

## 4. Kiến trúc FreeRTOS
| Task | Core | Ưu tiên | Stack | Chức năng |
|---|---|---|---|---|
| `Task_ReadData` | 1 | 2 | 4096 | Đọc MPU6050 mỗi 50 ms, in Serial, ghi vào `dataQueue` và `dataAlert` |
| `Task_SendData` | 0 | 1 | 6144 | Kết nối WiFi/MQTT, lấy dữ liệu từ `dataQueue`, gửi lên broker, nhận lệnh điều khiển |
| `Task_Alert` | 1 | 3 | 4096 | Đọc `dataAlert`, phát hiện té ngã, điều khiển LED/còi |

**Hàng đợi:** `dataQueue` (độ dài 10, bỏ mẫu cũ khi đầy) và `dataAlert` (độ dài 1, ghi đè mẫu mới nhất).

![Sơ đồ khối](images/so-do-khoi.png)

## 5. MQTT
- Broker: `broker.emqx.io` (ESP32: TCP 1883, Web: WebSocket Secure 8084)
- Tiền tố topic: `fall_demo_esp32_741852`

| Topic | Hướng | Nội dung |
|---|---|---|
| `.../sensor` | ESP32 → Web | JSON: ax, ay, az, accMagnitude, roll, pitch, fall, uptime |
| `.../status` | ESP32 → Web | `ONLINE` |
| `.../control/led` | Web → ESP32 | `ON` / `OFF` |
| `.../control/buzzer` | Web → ESP32 | `ON` / `OFF` |

## 6. Kết quả mô phỏng
**Serial Monitor**

![Serial Monitor](images/serial.png)

**Dashboard – trạng thái bình thường**

![Dashboard bình thường](images/dashboard-binh-thuong.png)

**Phát hiện té ngã (LED sáng, còi kêu, Web hiện cảnh báo)**

![Cảnh báo té ngã](images/te-nga.png)

Video demo: [dán link video nếu có]

## 7. Cấu trúc thư mục
```
├── baocao/    Báo cáo Lab 3 (PDF)
├── images/    Hình ảnh sơ đồ và kết quả
├── web/       Web Dashboard (HTML)
├── wokwi/     Code ESP32, diagram.json, libraries.txt
└── README.md
```

## 8. Hướng dẫn chạy
1. Mở dự án Wokwi theo link bên dưới (hoặc tạo dự án ESP32 mới, dán `diagram.json`, `libraries.txt` và file `.ino` trong thư mục `wokwi/`).
2. Bấm **Start** để chạy mô phỏng, mở Serial Monitor kiểm tra WiFi/MQTT đã kết nối.
3. Mở file trong thư mục `web/` bằng trình duyệt (hoặc dùng link Web bên dưới). Dashboard hiện chấm xanh khi kết nối MQTT thành công.
4. Kéo thanh trượt gia tốc của MPU6050 trong Wokwi để mô phỏng té ngã: gia tốc ≥ 2.2 g, sau đó đổi góc ≥ 55°.

## 9. Liên kết
- Mô phỏng Wokwi: [dán link Wokwi]
- Web Dashboard: [dán link GitHub Pages]
- Báo cáo: [Báo cáo Lab 3](baocao/)
