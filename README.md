# ESP32-S3 CAM IoT Firmware (`stalkbot-iot`)

[![Platform: ESP32-S3](https://img.shields.io/badge/Platform-ESP32--S3-orange.svg)](https://www.espressif.com/en/products/socs/esp32-s3)
[![Framework: Arduino](https://img.shields.io/badge/Framework-Arduino-00979D.svg)](https://github.com/espressif/arduino-esp32)
[![Hardware: N16R8](https://img.shields.io/badge/Hardware-N16R8%20Dual%20Type--C-blue.svg)](#hardware-specifications)
[![License: Apache-2.0](https://img.shields.io/badge/License-Apache_2.0-green.svg)](LICENSE)

Production-ready Camera Web Server firmware for the **ESP32-S3-WROOM-1 CAM (N16R8, Dual Type-C)** development board, serving as the vision acquisition node for the **Stalkbot** robotics and computer vision ecosystem.

---

## 1. Hardware Specifications

| Component | Specification |
| :--- | :--- |
| **SoC** | ESP32-S3 (Dual-core Xtensa® 32-bit LX7 @ **240 MHz**) with AI Vector Extensions |
| **Flash Memory** | **16 MB** SPI Flash (`N16`) |
| **PSRAM** | **8 MB** Octal SPI PSRAM (`R8` / OPI PSRAM) |
| **Camera Interface** | 24-pin DVP FPC connector (Supports OV2640 / OV3660 / OV5640) |
| **Default Pin Model** | `CAMERA_MODEL_ESP32S3_EYE` (`camera_pins.h`) |
| **Wireless** | 2.4 GHz Wi-Fi (802.11 b/g/n) + Bluetooth 5.0 LE & Mesh |
| **Dual Type-C Interfaces** | • **COM / UART Port:** WCH CH343/CH340 high-speed USB-to-UART bridge.<br>• **Native USB / OTG Port:** Direct hardware USB-Serial/JTAG (GPIO 19/20). |

---

## 2. Directory Structure

```text
stalkbot-iot/
├── CameraWebServer/             # Primary Arduino sketch package
│   ├── CameraWebServer.ino      # Application entrypoint & initialization
│   ├── board_config.h           # Camera model selector (CAMERA_MODEL_ESP32S3_EYE)
│   ├── camera_pins.h            # DVP 24-pin hardware pin definitions
│   ├── app_httpd.cpp            # HTTP server handler & MJPEG stream generator
│   ├── camera_index.h           # Gzip-compressed HTML/JS web control client
│   ├── wifi_credentials.h       # Local Wi-Fi network configuration
│   └── partitions.csv           # 16MB custom flash partition layout
├── .gitignore                   # Git artifact exclusion rules
├── ci.yml                       # Build matrix validation configuration
└── README.md                    # Technical documentation
```

> **Note on Arduino IDE Compatibility:** All companion header files (`.h`), source files (`.cpp`), and configuration tables (`.csv`) reside directly inside `CameraWebServer/` alongside `CameraWebServer.ino` to ensure seamless discovery by the Arduino toolchain.

---

## 3. Getting Started

### Prerequisites
1. Install **Arduino IDE** (v2.x recommended).
2. Add ESP32 board package URL in **Preferences -> Additional Boards Manager URLs**:
   ```text
   https://raw.githubusercontent.com/espressif/arduino-esp32/gh-pages/package_esp32_index.json
   ```
3. Install **`esp32`** by Espressif Systems (v3.0+ or v2.0.14+) via **Boards Manager**.

### Configuration Steps

#### 1. Wi-Fi Credentials
Edit `CameraWebServer/wifi_credentials.h`:
```cpp
#define WIFI_SSID     "Your_2.4GHz_WiFi_SSID"
#define WIFI_PASSWORD "Your_WiFi_Password"
```

#### 2. Verify Camera Model
Ensure `CAMERA_MODEL_ESP32S3_EYE` is active in `CameraWebServer/board_config.h`:
```cpp
#define CAMERA_MODEL_ESP32S3_EYE
```

---

## 4. Compiler & Upload Settings (Arduino IDE)

Configure the **Tools** menu strictly as follows:

| Menu Item | Required Value | Rationale |
| :--- | :--- | :--- |
| **Board** | `ESP32S3 Dev Module` | Standard core definition for ESP32-S3 |
| **Flash Size** | `16MB (128Mb)` | Matches on-board N16 Flash capacity |
| **Partition Scheme** | `16M Flash (3MB APP/9.9MB FATFS)` | Allocates sufficient flash for app binaries |
| **PSRAM** | `OPI PSRAM` | **Mandatory** for 8MB Octal PSRAM buffer allocation |
| **Upload Speed** | `921600` (or `460800`) | High-speed flashing throughput |

### Dual Type-C Port Flashing Matrix

| Connection Port | Recognized Device | `USB CDC On Boot` Setting | Post-Upload Requirement |
| :--- | :--- | :--- | :--- |
| **Native USB / OTG** *(Recommended)* | `USB-Serial/JTAG` (e.g. COM4) | **`Enabled`** | Press hardware **`RST`** button once to boot application. |
| **COM / UART (CH343)** | `USB-Enhanced-SERIAL CH343` (e.g. COM3) | **`Disabled`** | If bootloader hangs on `0x8`, hold **`BOOT`** button while `Connecting...` appears. |

---

## 5. Streaming & Web Interface

1. Launch **Serial Monitor** at **`115200 baud`**.
2. Press hardware **`RST` / `EN`** button. The device boots and acquires an IP address:
   ```text
   --- ESP32-S3 Camera Booting ---
   Connecting to Wi-Fi SSID: MyNetwork ...
   Wi-Fi connected successfully!
   ESP32-S3 IP Address: 192.168.100.104
   Camera Web Server Ready! Use 'http://192.168.100.104' to connect
   ```
3. Open a browser and navigate to `http://<ESP32_IP>` (e.g. `http://192.168.100.104`).
4. In the left navigation panel, scroll down to the bottom and click **`Start Stream`** to begin video feed acquisition.

---

## 6. HTTP API Endpoints

The onboard HTTP daemon exposes the following endpoints:

| Endpoint | Protocol | Description |
| :--- | :--- | :--- |
| `GET http://<ESP32_IP>/` | HTTP | Full camera diagnostic and configuration UI |
| `GET http://<ESP32_IP>:81/stream` | HTTP MJPEG | Raw multipart stream (`multipart/x-mixed-replace`) for web/OpenCV ingestion |
| `GET http://<ESP32_IP>/capture` | HTTP JPEG | Single frame snapshot capture |
| `GET http://<ESP32_IP>/status` | JSON | Sensor telemetry and active camera settings |
| `GET http://<ESP32_IP>/control?var={k}&val={v}` | HTTP | Runtime parameter modification (`framesize`, `quality`, `vflip`, `hmirror`) |

### Example Frame Size Control Values
* `framesize=10`: UXGA (1600x1200)
* `framesize=7`: SVGA (800x600 - Optimal for streaming)
* `framesize=6`: VGA (640x480)
* `framesize=4`: QVGA (320x240 - High FPS)

---

## 7. Troubleshooting Runbook

### Error `0x20003` (`ESP_ERR_NOT_FOUND` / Camera probe failed)
* **Cause:** Ribbon cable connection issue or incorrect pin mapping.
* **Resolution:** Reseat the 24-pin FPC ribbon cable with gold contacts facing downwards into the socket latch. Verify `board_config.h` selects `CAMERA_MODEL_ESP32S3_EYE`.

### Error `0x105` (`ESP_ERR_NO_MEM` / Out of Memory)
* **Cause:** External PSRAM unallocated or disabled.
* **Resolution:** Ensure **`PSRAM: "OPI PSRAM"`** is selected in compiler flags. Confirm `psramFound()` evaluates to `true` at runtime.

### Stalled Serial Log at `ESP-ROM:esp32s3-20210327`
* **Cause:** Output redirected to alternate port or chip left in ROM bootloader.
* **Resolution:** When using Native USB (COM4), set **`USB CDC On Boot: Enabled`** and press the **`RST`** button after flashing completes.

### `Wrong boot mode detected (0x8)`
* **Cause:** CH343 DTR/RTS timing failed to assert GPIO 0 during bootloader handshake.
* **Resolution:** Hold the physical **`BOOT`** button during upload until the write progress percentage displays.
