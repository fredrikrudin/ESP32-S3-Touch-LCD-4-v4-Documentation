# 🛠️ Waveshare ESP32-S3-Touch-LCD-4 (v4) All-in-One Diagnostics & Troubleshooter

![GitHub top language](https://shields.io)
![Platform](https://shields.io)
![Framework](https://shields.io)

This repository contains a comprehensive diagnostics and troubleshooting application developed for the **Waveshare ESP32-S3-Touch-LCD-4 version v4**. Upon boot, the program performs a complete hardware inventory, emits an audible ready-beep, renders the results directly onto the built-in LCD screen, and saves a permanent troubleshooting log to an installed Micro SD card.

---

## 🏗️ 1. Hardware Architecture & I2C Mapping

This module utilizes an advanced architecture where several critical peripheral components are placed behind an I2C IO-expander chip (`TCA9554PWR`) to conserve physical GPIO pins on the ESP32-S3 processor.

### 📡 I2C Bus 0 (Internal System Bus)
*   **Pins:** `SDA = GPIO 8`, `SCL = GPIO 9`
*   **`0x20` – TCA9554PWR (IO-Expander):** Controls secondary hardware functions:
    *   `EXIO1`: Backlight Enable (`BL_EN`)
    *   `EXIO3`: SD Card Chip Select (`SD_CS`)
    *   `EXIO5`: Built-in Buzzer Enable (`BEE_EN`)
*   **`0x3C` – SW6106 (Smart Power Management):** Manages LiPo battery charging/discharging and battery level readings. *The software automatically prevents the chip from entering "Light Load Auto-Shutdown" via periodic register writes.*

### 🖥️ I2C Bus 1 (Touch & Real-Time Clock)
*   **Pins:** `SDA = GPIO 15`, `SCL = GPIO 7`
*   **`0x5D` / `0x14` – GT911:** Capacitive touch controller supporting up to 5 simultaneous touch points.
*   **`0x51` – PCF85063:** Hardware-based Real-Time Clock (RTC) for time tracking.

---

## 🔌 2. Bus & Peripheral Configuration

| Interface | Control IC | ESP32-S3 Pins | Mode / Specification |
| :--- | :--- | :--- | :--- |
| **LCD Screen** | ST7701S | Dedicated RGB Interface | 480 × 480 px, 16-bit RGB (65K colors) |
| **CAN-Bus (TWAI)** | TJA1051T | `TX = GPIO 6`, `RX = GPIO 0` | Listen-Only Mode, 500 kbps |
| **RS485-Bus** | SP3485 | `TX = GPIO 44`, `RX = GPIO 43` | 115200 baud, 8N1 (Automatic direction) |
| **Micro SD Card** | Built-in Slot | Hardware SPI (`MOSI:1`, `MISO:4`, `SCK:2`) | FAT32 format, CS controlled via Expander Pin 3 |

---

## 💾 3. Log File Specification (`ESP32-S3-Touch-LCD-4.txt`)

Upon a successful boot, a text file named `ESP32-S3-Touch-LCD-4.txt` is created on the root directory of the Micro SD card. The log file uses the following structure and can be used for offline analysis:

```text
=======================================================
   WAVESHARE ESP32-S3-TOUCH-LCD-4 v4 TOTAL DIAGNOSTICS   
=======================================================
[1/5] WIRELESS INTERFACES:
  Base MAC: 7C:DF:A1:XX:XX:XX   <- Unique hardware ID (eFuse)
  BLE MAC:  7C:DF:A1:XX:XX:XY   <- Integrated Bluetooth 5.0 address

[2/5] SCANNING I2C BUS 0:
  Found 0x20 -> TCA9554 Expander
  Found 0x3C -> SW6106 Charger

[2/5] SCANNING I2C BUS 1:
  Found 0x51 -> PCF85063 RTC
  Found 0x5D -> GT911 Touch     <- Confirms physical contact with the glass

[3/5] POWER & BATTERY STATUS:
  Battery connected: 84%        <- Shows capacity if a LiPo battery is used
  Status: Charging...           <- Indicates if external voltage is fed via USB-C

[4/5] CAN-BUS (TWAI) STATUS:
  Active (Listen-Only, 500kbps)

[5/5] RS485-BUS STATUS:
  Active (115200 baud, 8N1)

[SD CARD] STARTING WRITE:
  -> SD card mounted OK!
  -> Log file saved successfully!
```

---

## 🚨 4. Troubleshooting Guide

> [!IMPORTANT]
> **Critical Software Setting in Arduino IDE:**
> Before flashing the code, navigate to `Tools` -> `USB CDC On Boot` and set it to **Enabled**. If left disabled, early runtime crashes can cause the ESP32-S3 to permanently shut down its USB peripheral, blocking future sketch uploads.

### Black Screen or Frozen System?
Observe the log output in the Serial Monitor (configured to 115200 baud). Because the boot sequence is strictly divided into blocks `[1/5]` through `[5/5]`, you can instantly pinpoint where the processor hung:
*   **Stops at `[2/5]`:** Indicates a hardware failure or a short circuit on the I2C buses.
*   **Stops at `[SD CARD]`:** Verify that your card is formatted to **FAT32** (exFAT is not supported) and ensure the expander chip at `0x20` is responsive (as it is responsible for pulling `SD_CS` low).

## ⚙️ 5. Arduino IDE Environment Setup & Configuration
1. Install the Espressif board package via Additional Boards Manager URLs and install version **v3.0.7**.
2. Install required libraries: `ESP32_Display_Panel` (v0.1.8) and `ESP32_IO_Expander` (v0.0.4).
3. Recommended Tools Menu settings:
   * **Board:** `ESP32S3 Dev Module`
   * **USB CDC On Boot:** `Enabled`
   * **Flash Size:** `16MB (128Mb)`
   * **PSRAM:** `OPI PSRAM`

### CAN or RS485 Communication Fails?
There are two small red dual-switches located on the back of the PCB labeled **CAN** and **485**. These toggle the built-in **120-Ohm termination resistors**. If this board sits at the physical end of your bus wiring, these switches must be flipped to `ON` to prevent signal reflections.

### Recovery from a Blocked USB Port (Bricked Board)
If the board ends up in a crash loop and is no longer detected as a valid COM port by your computer:
1. Press and hold the **BOOT** button on the back of the board.
2. Press and release the **RESET** button quickly.
3. Release the **BOOT** button.
4. The board is now forced into *Download Mode* and will re-appear in the Arduino IDE, allowing you to flash a clean, working sketch.

### WaveShare Wiki ###
https://www.waveshare.com/wiki/ESP32-S3-Touch-LCD-4?srsltid=AfmBOorQe24zfo9qZ0bkl-78hbrpERO4lrStSTrBJWj-46-DTtiBlW1h
