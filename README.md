# NodeRED QuickWin Application — ESP32 MQTT Course

> **Online Course by THAITECHZONE**
> เรียนรู้การเชื่อมต่อ ESP32 กับ Node-RED ผ่าน MQTT พร้อม Dashboard แบบ Real-time

---

## ภาพรวมโปรเจกต์

Firmware นี้ทำงานบน **ESP32 Devkit V2** โดยใช้ **PlatformIO + Arduino Framework** ทำหน้าที่:

- เชื่อมต่อ WiFi และ sync เวลาจาก NTP (GMT+7 Bangkok)
- ส่ง Telemetry (อุณหภูมิ, ความชื้น, AQI, สถานะ Relay) ผ่าน **MQTT** ทุก 10 วินาที
- รับคำสั่งควบคุม **Relay 3 ช่อง** จาก Node-RED Dashboard
- ดึงข้อมูล **สภาพอากาศและคุณภาพอากาศ** จาก OpenWeatherMap ทุก 10 นาที
- แสดงสถานะทั้งหมดบนจอ **OLED SSD1306 128×64**

---

## Stack

| ส่วน | รายละเอียด |
|---|---|
| MCU | ESP32 Devkit V2 |
| Framework | Arduino บน PlatformIO |
| Broker | Mosquitto (Local) / HiveMQ Public |
| Dashboard | Node-RED + node-red-dashboard |
| API | OpenWeatherMap (Weather + Air Pollution) |
| Display | OLED SSD1306 I2C 128×64 |

---

## Hardware — Pin Map

| กลุ่ม | GPIO | ชนิด | หมายเหตุ |
|---|:---:|---|---|
| Relay 1 | 17 | Digital Output | Active Low |
| Relay 2 | 16 | Digital Output | Active Low |
| Relay 3 | 4 | Digital Output | Active Low |
| AUX 1 | 12 | TTL 3.3V Output | — |
| AUX 2 | 13 | TTL 3.3V Output | — |
| AUX 3 | 14 | TTL 3.3V Output | — |
| AUX 4 | 15 | TTL 3.3V Output | — |
| SW1 | 34 | Digital Input | Active Low |
| SW2 | 35 | Digital Input | Active Low |
| SW3 | 32 | Digital Input | Active Low |
| ISO IN 1 | 33 | Isolated Input | Active Low |
| ISO IN 2 | 27 | Isolated Input | Active Low |
| Onboard LED | 2 | Digital Output | Status LED |
| OLED SDA | 21 | I2C | — |
| OLED SCL | 22 | I2C | — |

> OLED เป็นอุปกรณ์เสริม — ถ้าไม่ได้ต่อจอ โปรแกรมยังทำงานปกติ และดูสถานะผ่าน Serial Monitor ได้

---

## การตั้งค่า (Configuration)

แก้ค่าหลักใน `src/main.cpp`:

```cpp
// WiFi
const char* WIFI_SSID     = "your-wifi-ssid";
const char* WIFI_PASSWORD = "your-wifi-password";

// MQTT Broker
const char* MQTT_BROKER   = "192.168.1.x";       // Local Mosquitto
// const char* MQTT_BROKER = "broker.hivemq.com"; // หรือ Public Broker
const int   MQTT_PORT     = 1883;
const char* MQTT_BOARD_ID = "esp32-devkit-01";

// OpenWeatherMap
const char* OPENWEATHERMAP_API_KEY = "your-api-key";
const char* WEATHER_CITY           = "Tha Sala,Nakhon Si Thammarat,TH";
```

> `WEATHER_CITY` ใส่ชื่อเมืองแบบปกติได้เลย — โปรแกรมจะ encode ช่องว่างเป็น `%20` ให้เอง
> พิกัดจาก Weather API จะถูกส่งต่อให้ Air Pollution API โดยอัตโนมัติ

ค่า fallback พิกัด (ใช้เมื่อดึง Weather ไม่สำเร็จ):

```cpp
const float DEFAULT_WEATHER_LAT = 8.4304;
const float DEFAULT_WEATHER_LON = 99.9631;
```

---

## MQTT Topics

```text
device/{boardID}/telemetry           <- Publish: ข้อมูล sensor ทุก 10 วินาที
device/{boardID}/status              <- Publish: สถานะ online/offline
device/{boardID}/control/relay/1    <- Subscribe: สั่ง Relay 1
device/{boardID}/control/relay/2    <- Subscribe: สั่ง Relay 2
device/{boardID}/control/relay/3    <- Subscribe: สั่ง Relay 3
device/{boardID}/control/relay/all  <- Subscribe: สั่ง Relay ทั้ง 3 พร้อมกัน
device/{boardID}/relay/+/status     <- Publish: Feedback สถานะ Relay
```

**Payload สำหรับควบคุม Relay** รองรับทั้ง JSON และข้อความสั้น:

```json
{"state": true}
```

```text
true  / false
on    / off
1     / 0
```

---

## Telemetry Payload

```json
{
  "aqi": 2,
  "pm2_5": 12.4,
  "pm10": 20.1,
  "quality": "Fair",
  "temperature": 29,
  "humidity": 74,
  "weather_lat": 8.667,
  "weather_lon": 99.917,
  "relay1": false,
  "relay2": true,
  "relay3": false,
  "timestamp": "2026-05-07 21:30:00",
  "rssi": -54,
  "uptime_seconds": 3600
}
```

---

## OLED Display Layout

```text
+-------------------------+
| ESP32 MQTT      21:30   |
| WiFi:-54dBm    MQ:OK    |
| RLY:010  Topic:/relay   |
| T:29C H:74%  AQI:2      |
| PM2.5:12                |
| PUB:10s                 |
+-------------------------+
```

Relay state แสดงเป็น `1` = ON, `0` = OFF

---

## Libraries (platformio.ini)

```ini
[env:esp32dev]
platform      = espressif32
board         = esp32dev
framework     = arduino
monitor_speed = 115200

lib_deps =
  adafruit/Adafruit GFX Library
  adafruit/Adafruit SSD1306
  bblanchon/ArduinoJson@^6.19.4
  knolleary/PubSubClient@^2.8
```

---

## Build & Upload

```bash
# Build
pio run

# Upload ไปยัง ESP32
pio run --target upload

# เปิด Serial Monitor
pio device monitor
```

> บน Windows ถ้า `pio` ไม่อยู่ใน PATH:
> ```powershell
> & $env:USERPROFILE\.platformio\penv\Scripts\platformio.exe run
> ```

---

## เอกสารประกอบ

| ไฟล์ | รายละเอียด |
|---|---|
| [BluePrint.md](BluePrint.md) | สถาปัตยกรรม firmware และ flow การทำงาน |
| [MQTT_TESTING_GUIDE.md](MQTT_TESTING_GUIDE.md) | ทดสอบ MQTT ด้วย MQTT Explorer และ mosquitto CLI |

---

## หลักสูตร — NodeRED QuickWin Application

คอร์สนี้ครอบคลุม:

1. **ESP32 + PlatformIO** — ตั้งค่าโปรเจกต์และเขียน Firmware เบื้องต้น
2. **MQTT Protocol** — หลักการทำงาน Topics, Publish/Subscribe
3. **Node-RED** — ติดตั้งและสร้าง Flow รับ-ส่งข้อมูลกับ ESP32
4. **Node-RED Dashboard** — สร้าง UI แสดงผล Telemetry และควบคุม Relay แบบ Real-time
5. **OpenWeatherMap Integration** — ดึงข้อมูลสภาพอากาศและ AQI มาแสดงบน Dashboard
6. **OLED Display** — แสดงสถานะระบบบนจอ ESP32

> สอบถามและติดตามคอร์สได้ที่ **THAITECHZONE**