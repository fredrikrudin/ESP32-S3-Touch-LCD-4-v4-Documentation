# ⚠️ Findings, Errata & Open Questions

Companion to [README.md](README.md). The README documents **how the board is wired**.
This file documents **what is easy to get wrong**, where the official sources are
misleading, and which questions are still open.

Every item is marked with how well it is established:

| Mark | Meaning |
| :--- | :--- |
| ✅ **Verified** | Observed directly on this board, via `ESP32-S3-Touch-LCD-4_diags.ino` or a working build |
| 📄 **Documented** | From the Waveshare wiki or library sources; not independently confirmed here |
| ❓ **Open** | Sources disagree, or reconstructed from indirect evidence — do not trust blindly |

---

## 🔴 1. Open Questions — sources disagree

These three are the most important entries in this file. Each one has two
plausible answers in circulation, and picking the wrong one costs an evening.

### 1.1 Which IO expander is actually fitted? ❓

| Source | Claim |
| :--- | :--- |
| This repo's README, `ESP32-S3-Touch-LCD-4_diags.ino` | **TCA9554PWR** at `0x20`, driven over plain `Wire` |
| `esp32-S3-ws4-boat` README | **CH32V003** IO expander, driven via the `WS_CH32_IO` library |
| Waveshare's own repo README for "V4.0" | **CH32V003** at I2C address `0x24` |

**What the hardware says:** the boot scan on this board finds `0x20` and `0x3C`
on bus 0 and nothing at `0x24`. So on *this* unit the expander answering is at
`0x20`.

**Still to resolve:** whether `WS_CH32_IO` in the boat firmware is in fact
talking to `0x20` (in which case the "CH32V003" naming there is inherited text
and should be corrected), or whether Waveshare shipped more than one board under
the "V4" label. The second possibility is real — Waveshare has reused revision
labels before.

> **Test:** add `Serial.printf` of the I2C address `WS_CH32_IO` uses, or scan bus 0
> from the boat firmware and compare with the diags output. Five minutes, settles it.

### 1.2 Is the backlight dimmable? ❓

| Source | Claim |
| :--- | :--- |
| This repo's README | `EXIO1` = **Backlight Enable** — a digital on/off line |
| `esp32-S3-ws4-boat` README | **PWM from the CH32V003, inverted** (`0` = full brightness, `255` = off) |

These cannot both describe the same circuit. A TCA9554 output pin has no PWM
capability, so if the expander is a TCA9554 the backlight is on/off only.

> **Test:** set brightness to 50 % in the boat firmware and look at the screen.
> If nothing changes, `EXIO1` is an enable line and any brightness slider in the
> UI is decorative. Follows directly from 1.1.

### 1.3 Micro SD — SPI or SDMMC? ❓

| Source | Claim |
| :--- | :--- |
| This repo's README | Hardware **SPI**: `MOSI 1`, `MISO 4`, `SCK 2`, CS via `EXIO3` |
| `esp32-S3-ws4-boat` README | **SDMMC 1-bit**: `CLK 2`, `CMD 1`, `DATA0 4` |

Here both are probably correct. The same three pins carry both, because an SD
slot wired for SDMMC 1-bit can also be driven in SPI mode — that is designed into
the SD spec, not a hack.

The practical differences:

* **SDMMC 1-bit** is faster and has **no chip-select line**, so the `EXIO3` note
  does not apply in that mode. This is the mode proven on hardware in the boat
  firmware.
* **SPI mode** needs `EXIO3` pulled low by the expander, which means the expander
  must be alive before the card will mount. This is what the diags sketch uses.

> Document whichever mode a given project uses. Mixing the two mental models is
> how "card not found" bugs happen.

---

## 📐 2. Wiki Errata

### 2.1 The pin table implies GPIO 8/9 are shared with the RGB bus — they are not ✅

The Waveshare wiki pin table lists:

```
GPIO8  |  R3  |  Expander_SDA
GPIO9  |  G5  |  Expander_SCL
```

Read literally, this says the expander's I2C bus shares two pins with the LCD's
red and green data lines — which would mean the expander has to be configured
*before* the RGB panel starts and becomes unreachable afterwards.

**This is wrong, or at least not true of this board.** The boot scan finds the
expander and the charger on bus 0 both before and after `ESP_Panel` brings the
display up, and the SW6106 keep-alive keeps working for the lifetime of the
sketch. GPIO 8 and 9 are dedicated I2C.

Do not design an elaborate "configure then release the bus" sequence around this.
It is unnecessary complexity solving a problem that does not exist.

### 2.2 The wiki contradicts itself on which bus is which 📄

In one paragraph the wiki calls GPIO 8/9 "I2C1" and GPIO 15/7 "I2C0"; elsewhere
it swaps them. The numbering is cosmetic — what matters is the pin pair and what
answers on it. Trust the pin table and the scan, not the prose.

### 2.3 `ESP32_Display_Panel` does not list this board 📄

The library's supported-board list covers the ESP32-S3-Touch-LCD-4.3 and
4.3B, but **not** the 4 (480×480). A custom board configuration is therefore
mandatory, not optional. See §4.

---

## ⚡ 3. The SW6106 Will Switch Your Board Off ✅

The single most confusing failure mode on this board.

The SW6106 power-management chip powers the board down when it detects a
**light load**. An idle display showing a clock or a status page is exactly that.
Running on the LiPo battery, the board appears to switch itself off for no reason,
and it looks like a firmware crash.

**Fix:** write `0x0A` to register `0x38` of device `0x3C` periodically. A 5-second
interval is comfortable.

```cpp
// Cancels the light-load auto-shutdown timer. Call every ~5 s.
Wire.beginTransmission(0x3C);
Wire.write(0x38);
Wire.write(0x0A);
Wire.endTransmission();
```

Not needed on USB-C power, which is why the problem only shows up in the field.

### Battery registers ❓

These two are **reconstructed** from the diags log format, not from a datasheet.
Verify before relying on them:

| Register | Reported meaning |
| :--- | :--- |
| `0x31` | State of charge, 0–100 % |
| `0x32` | Status bits — charging indicated by bit 6 / bit 7 |

Readings of `0`, `0xFF` or anything above 100 should be treated as "no battery".

---

## 🧩 4. The `ESP_Panel` Configuration Trap ✅

**Symptom:**

```
error: 'ESP_Panel' does not name a type; did you mean 'ESP_PanelLcd'?
```

**Cause:** the whole `ESP_Panel` class is wrapped in `#if ESP_PANEL_USE_BOARD`.
With no configuration found, the library falls back to defaults where both board
flags are `0`, that block compiles to nothing, and only the low-level classes
survive — hence the compiler helpfully suggesting `ESP_PanelLcd`, which does exist.

**Lookup order** for `ESP_Panel_Conf.h`:

1. the sketch folder (next to the `.ino`) — highest priority, always wins
2. `Arduino/libraries/`
3. the `ESP32_Display_Panel` library's own root

A config living in one project's sketch folder is invisible to every other
project. That is the usual reason a second sketch fails to build while the first
one still works.

**Required contents:**

```c
#define ESP_PANEL_USE_CUSTOM_BOARD      1
#define ESP_PANEL_USE_SUPPORTED_BOARD   0
```

plus `ESP_Panel_Board_Custom.h` beside it with the panel geometry and timings.

**Fail loudly instead of cryptically** — put this above your panel code:

```cpp
#if !defined(ESP_PANEL_USE_BOARD) || !ESP_PANEL_USE_BOARD
#error "ESP_Panel board configuration not found. Copy ESP_Panel_Conf.h and \
ESP_Panel_Board_Custom.h into this sketch folder and set ESP_PANEL_USE_CUSTOM_BOARD to 1."
#endif
```

### 🔺 Reproducibility gap

`ESP_Panel_Conf.h` and `ESP_Panel_Board_Custom.h` currently live **outside every
repository**, inside the Arduino libraries folder. A fresh Arduino install
cannot reproduce any working build without them, and they contain the RGB timings
that are hardest to recover by guesswork.

**Committing a copy of both files to this repository is the single highest-value
addition it could receive.**

---

## 🔧 5. Toolchain Caveats

| Setting | Value | Why it matters |
| :--- | :--- | :--- |
| esp32 board package | **v3.0.7** | Newer cores break `ESP_Panel` 0.1.x |
| `ESP32_Display_Panel` | **0.1.8** | 1.x renames everything (`esp_panel::board::Board`) |
| `ESP32_IO_Expander` | **0.0.4** | Pinned alongside the above |
| USB CDC On Boot | **Enabled** | Disabled + an early crash can block future uploads |
| Erase All Flash Before Upload | **Disabled** | Otherwise every upload wipes saved settings |
| PSRAM | **OPI PSRAM** | Framebuffer and LVGL buffers live there |
| Flash Size | 16MB (128Mb) | |

### LVGL

* **LVGL 8.x only.** LVGL 9 removed `lv_meter`, which every gauge on this board uses.
* `LV_COLOR_DEPTH 16`, `LV_COLOR_16_SWAP 0`.
* Enable the Montserrat sizes you actually use (`14`, `20`, `28`, `32`, `48`);
  missing sizes silently fall back to a smaller font.
* Built-in Montserrat has **no å / ä / ö**. Keep UI strings ASCII unless you add
  a custom font — this bites Swedish projects immediately.
* Move LVGL's pool to PSRAM to free internal RAM:

  ```c
  #define LV_MEM_POOL_INCLUDE <esp32-hal-psram.h>
  #define LV_MEM_POOL_ALLOC   ps_malloc
  ```

### Internal RAM

With WiFi, Bluetooth and the display all running, free internal RAM is tight
enough that **HTTPS will not fit**. Plain HTTP works. This is a memory
constraint, not a configuration mistake — do not spend time on certificates.

---

## 📡 6. Peripheral Gotchas

### GT911 touch address varies ✅

The controller answers at **either `0x5D` or `0x14`**, and it differs between
panels. Scan for both; never hardcode one. Worth noting which address *your*
unit uses once you know it.

### CAN / RS485 termination

Two red DIP switches on the back of the PCB enable the built-in **120 Ω
terminators**. Enable them only if this board sits at a physical *end* of the bus.

> **NMEA 2000 specifically:** an N2K backbone is already terminated at both ends.
> Switching the board's terminator on puts a third resistor across the bus and
> degrades or kills communication. Leave it **off**.

### PCF85063 RTC

The oscillator-stopped flag is bit 7 of the seconds register (`0x04`). If it is
set, the stored time is meaningless — treat it as "never set" rather than
reading garbage into the system clock.

---

## 🩺 7. Symptom → Cause

| Symptom | Likely cause |
| :--- | :--- |
| `'ESP_Panel' does not name a type` | `ESP_Panel_Conf.h` not reachable from this sketch — §4 |
| Backlight on, screen black | RGB timings or ST7701 init in `ESP_Panel_Board_Custom.h` |
| Board powers itself off on battery, fine on USB | SW6106 light-load shutdown — §3 |
| Touch dead, display fine | GT911 on the other address (`0x14` vs `0x5D`) — §6 |
| Settings lost after every upload | *Erase All Flash Before Sketch Upload* enabled |
| Board no longer enumerates as a COM port | USB CDC crash loop — hold **BOOT**, tap **RESET**, release **BOOT** |
| SD card not found | Wrong mode assumed (SPI vs SDMMC), exFAT instead of FAT32, or expander dead in SPI mode — §1.3 |
| Time resets on every power cycle | RTC oscillator-stopped flag set, or no RTC fitted |
| Swedish characters render as blanks | Montserrat has no å/ä/ö — §5 |
| HTTPS request always fails | Not enough free internal RAM — use HTTP — §5 |

---

## 🧪 8. Suggested Next Additions

Ordered by how much future time they save:

1. **Commit `ESP_Panel_Conf.h` + `ESP_Panel_Board_Custom.h`** — closes the
   reproducibility gap in §4.
2. **Resolve §1.1 and §1.2** with the two five-minute tests described there, then
   correct whichever README is wrong.
3. **Record this unit's GT911 address** once observed.
4. **A photo of the PCB back** showing the two termination switches and the
   BOOT / RESET buttons.
5. **Merge `BOARD_NOTES.md`** from `esp32-S3-ws4-boat` into this repo, so there is
   one place to look rather than two.
