# ESP32-S3 WROOM CAM (N16R8) - Stalkbot IoT Firmware

Kho mã nguồn firmware Camera Web Server cho vi điều khiển **ESP32-S3 WROOM CAM (16MB Flash, 8MB PSRAM, Dual Type-C)** thuộc dự án **Stalkbot**.

Tài liệu này đúc kết toàn bộ quy trình cấu hình, nạp code, tối ưu hóa bộ nhớ và các kinh nghiệm thực chiến xử lý sự cố (troubleshooting) đã được kiểm chứng thành công 100%.

---

## 1. Thông số phần cứng bo mạch

| Thành phần | Cấu hình chi tiết |
| :--- | :--- |
| **Vi điều khiển (MCU)** | ESP32-S3 Dual-Core Xtensa® 32-bit LX7 @ **240 MHz** (Tích hợp Vector AI) |
| **Bộ nhớ Flash** | **16 MB** SPI Flash (`N16`) |
| **Bộ nhớ PSRAM** | **8 MB Octal PSRAM** (`R8` / OPI PSRAM) - Chuyên dụng cho bộ đệm hình ảnh |
| **Cảm biến Camera** | Socket FPC 24-pin DVP (Hỗ trợ OV2640 / OV3660 / OV5640) |
| **Sơ đồ chân (Pinout)** | Chuẩn **`CAMERA_MODEL_ESP32S3_EYE`** trong `camera_pins.h` |
| **Cổng kết nối kép (Dual Type-C)** | • **Cổng COM / UART**: Giao tiếp qua chip nạp WCH CH343/CH340.<br>• **Cổng Native USB / OTG**: Kết nối trực tiếp vào phần cứng USB-Serial/JTAG của ESP32-S3. |

---

## 2. Cấu trúc thư mục mã nguồn

Để nạp code bằng **Arduino IDE**, bắt buộc tên thư mục sketch phải trùng với tên file `.ino` chính:

```text
iot/
├── CameraWebServer/             <-- Thư mục sketch chính
│   ├── CameraWebServer.ino      <-- File chạy chính (setup, loop, Wi-Fi, camera_init)
│   ├── board_config.h           <-- Chọn model mạch (CAMERA_MODEL_ESP32S3_EYE)
│   ├── camera_pins.h            <-- Định nghĩa GPIO 24-pin của Camera
│   ├── app_httpd.cpp            <-- Xử lý Web Server & luồng stream video MJPEG
│   ├── camera_index.h           <-- Giao diện Web điều khiển (HTML/CSS/JS nén gzip)
│   ├── wifi_credentials.h       <-- Cấu hình tên & mật khẩu Wi-Fi
│   └── partitions.csv           <-- Bảng phân vùng bộ nhớ Flash 16MB
└── README.md                    <-- Tài liệu hướng dẫn
```

---

## 3. Hướng dẫn thiết lập từng bước (Setup Guide)

### Bước 1: Cấu hình Wi-Fi
Mở file `CameraWebServer/wifi_credentials.h`, điền thông tin Wi-Fi nhà bạn (băng tần **2.4 GHz**):

```cpp
#define WIFI_SSID     "Ten_Wifi_Cua_Ban"
#define WIFI_PASSWORD "Mat_Khau_Wifi"
```

### Bước 2: Kiểm tra cấu hình Model Camera
Trong file `CameraWebServer/board_config.h`, đảm bảo dòng sau đã được bật:
```cpp
#define CAMERA_MODEL_ESP32S3_EYE // Khớp 100% chân FPC 24-pin của bo mạch
```

### Bước 3: Cấu hình trên Arduino IDE
Vào menu **Tools** của Arduino IDE và thiết lập chính xác các thông số:

* **Board:** `ESP32S3 Dev Module`
* **Flash Size:** `16MB (128Mb)`
* **Partition Scheme:** `16M Flash (3MB APP/9.9MB FATFS)` hoặc `Huge APP (3MB No OTA)`
* **PSRAM:** `OPI PSRAM` *(BẮT BUỘC - Để nhận đủ 8MB RAM đệm ảnh độ nét cao)*
* **Upload Speed:** `921600` (hoặc `460800` / `115200`)
* **Port:** Chọn đúng cổng COM của thiết bị.

---

## 4. Kinh nghiệm thực chiến: Cổng Type-C & `USB CDC On Boot`

Mạch sở hữu 2 cổng Type-C với cơ chế hoạt động khác nhau:

### Trường hợp A: Cắm cổng Native USB / OTG (Khuyên dùng - Nạp cực mượt)
* Máy tính nhận dạng cổng: **`USB-Serial/JTAG`** (ví dụ `COM4`).
* Cài đặt trong menu Tools: **`USB CDC On Boot: Enabled`**.
* **Đặc điểm:** Nạp cực nhanh, không bao giờ bị kẹt lỗi bootloader.
* ⚠️ **Lưu ý sau khi nạp:** Mạch sẽ báo `Hard resetting via RTS pin...` nhưng không tự khởi động lại. Bạn **phải bấm nút `RST` (hoặc `EN`) trên thân mạch 1 lần**, sau đó mở Serial Monitor (115200 baud) để xem IP.

### Trường hợp B: Cắm cổng UART / COM (Qua chip CH343)
* Máy tính nhận dạng cổng: **`USB-Enhanced-SERIAL CH343`** (ví dụ `COM3`).
* Cài đặt trong menu Tools: **`USB CDC On Boot: Disabled`**.
* ⚠️ **Xử lý lỗi `Wrong boot mode detected (0x8)`:** Mạch nạp tự động của CH343 có thể chưa kịp kéo chân GPIO 0 xuống đất.
  * **Cách xử lý:** Bấm nút **Upload** trên IDE, ngay khi thấy xuất hiện dòng `Connecting................`, lấy ngón tay **bấm và giữ nút `BOOT`** trên mạch cho đến khi thấy chữ `Writing...` mới thả tay ra.

---

## 5. Khởi chạy & Xem Video Stream

1. Mở **Serial Monitor** trong Arduino IDE (chọn tốc độ **`115200 baud`**).
2. Khi khởi động thành công, màn hình sẽ in ra thông báo:
   ```text
   --- ESP32-S3 Camera Booting ---
   Connecting to Wi-Fi SSID: Khanh Hoa ...
   Wi-Fi connected successfully!
   ESP32-S3 IP Address: 192.168.100.104
   Camera Web Server Ready! Use 'http://192.168.100.104' to connect
   ```
3. Mở trình duyệt web (Chrome / Edge / Safari...), truy cập địa chỉ IP nhận được:
   ```text
   http://192.168.100.104
   ```
4. 💡 **LƯU Ý QUAN TRỌNG:** Khi giao diện web hiện lên, màn hình camera ban đầu sẽ màu đen vì chế độ stream chưa bật. Bạn hãy **cuộn chuột xuống dưới cùng của menu điều khiển bên trái**, bấm vào nút **`Start Stream`** để video bắt đầu phát.

---

## 6. Các đường dẫn API tích hợp

* **Luồng video trực tiếp (MJPEG Stream):**
  `http://<ESP32_IP>:81/stream` (Dùng để nhúng trực tiếp vào thẻ `<img src="...">` của Web hoặc đọc qua OpenCV Python).
* **Chụp 1 tấm ảnh tĩnh (Snapshot):**
  `http://<ESP32_IP>/capture`
* **Điều chỉnh thông số Camera từ xa:**
  `http://<ESP32_IP>/control?var=framesize&val=7` (SVGA)
