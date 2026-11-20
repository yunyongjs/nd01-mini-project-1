# ND01 Mini-Project #1 — สรุปเนื้อหา Week 01–05

> ESP32-S3 · ESP-IDF · FreeRTOS

| | |
|---|---|
| **ชื่อ** | ยรรยง จันทร์สวย |
| **รหัส / Module** | ND69-1 (ND01 — Embedded & Hardware) |
| **แหล่งเนื้อหาหลัก** | [drsanti/kmutt-nd69 › nd1/docs](https://github.com/drsanti/kmutt-nd69/tree/main/nd1/docs) |

สรุปสิ่งที่เรียนและฝึกปฏิบัติตลอด 5 สัปดาห์แรก ตั้งแต่ภาพรวม Edge AI/IoT → ฮาร์ดแวร์บอร์ด ES3C28P → หน่วยความจำ → ESP-IDF → FreeRTOS

---

## สารบัญ

- [Week 01 — Introduction: Edge AI & IoT](#week-01--introduction-edge-ai--iot)
- [Week 02 — ES3C28P Hardware & Embedded Fundamentals](#week-02--es3c28p-hardware--embedded-fundamentals)
  - [1. ชิป ESP32-S3R8](#1-ชิป-esp32-s3r8)
  - [2. โครงสร้างหน่วยความจำ](#2-โครงสร้างหน่วยความจำ-memory)
  - [3. อุปกรณ์บนบอร์ดและแผนที่ขา](#3-อุปกรณ์บนบอร์ดและแผนที่ขา-gpio)
  - [4. Embedded C + FreeRTOS](#4-embedded-c--freertos-review)
- [Week 03 — ESP-IDF Setup & Hello World](#week-03--esp-idf-setup--esp32-s3-hello-world)
  - [FreeRTOS บน PC Simulator](#freertos-บน-pc-simulator--digital-twin)
- [Week 04 — Onsite Workshop: ESP-IDF + RTOS Demo](#week-04--onsite-workshop-esp-idf--rtos-demo-บนบอร์ด-es3c28p)
- [Week 05](#week-05)
- [สิ่งที่ได้เรียนรู้](#สิ่งที่ได้เรียนรู้)
- [อ้างอิง](#อ้างอิง)

---

## Week 01 — Introduction: Edge AI & IoT

ภาพรวมระบบ Edge AI และ IoT ในยุค AI และทิศทางของงาน embedded systems

- **แนวคิดหลักของหลักสูตร:** เมื่อ AI เขียนโค้ดได้แล้ว ความต่างอยู่ที่ **ความเข้าใจโลกกายภาพ** (แรงดัน, timing, memory, sensor) และการสั่งให้ฮาร์ดแวร์ตอบสนองต่อวัตถุจริง
- **โครงสร้างหลักสูตร 3 track:** ND01 Embedded & Hardware (กลุ่มนี้), ND02 Application Developers, ND03 Automation & Robotics
- **บอร์ดที่ใช้ตลอดคอร์ส:** ES3C28P (ESP32-S3 + จอ 2.8" touch + audio)

---

## Week 02 — ES3C28P Hardware & Embedded Fundamentals

### 1. ชิป ESP32-S3R8

| รายการ | ค่าบนบอร์ด ES3C28P |
|---|---|
| CPU | Xtensa LX7 dual-core 32-bit สูงสุด 240 MHz |
| SRAM / ROM / RTC SRAM | 512 KB / 384 KB / 16 KB |
| PSRAM | 8 MB octal (OPI) อยู่ใน package เดียวกับชิป (ตัว "R8") |
| Flash | 16 MB QSPI ภายนอก |
| Wireless | WiFi 2.4 GHz b/g/n + BLE 5.0 (ไม่มี BT Classic) |
| USB | Native USB OTG + USB-Serial/JTAG (ไม่มีชิป USB-UART) |
| แรงดัน I/O | **3.3 V เท่านั้น — ไม่ทน 5 V** |

- **ES3C28P** คือชื่อ SKU ของบอร์ด ส่วน **ESP32-S3R8** คือ part number ของชิป
- อ่านชื่อรุ่น: `R8` = PSRAM 8 MB ในแพ็กเกจ, module `N16R8` = Flash 16 MB + PSRAM 8 MB
- ตระกูล ESP32: **S** = Xtensa, **C** = RISC-V ราคาถูก, **H** = 802.15.4 (Thread/Zigbee), **P** = performance ไม่มี radio
- ESP32-S3 ถูกออกแบบมาสำหรับงาน HMI (จอ + touch + เสียง) และมี vector instruction สำหรับ DSP/AI เบา ๆ

### 2. โครงสร้างหน่วยความจำ (Memory)

แยกด้วย 2 แกน: **ในชิป/นอกชิป** และ **ดับไฟแล้วหาย/ไม่หาย**

| | Volatile | Non-volatile |
|---|---|---|
| **Internal** | SRAM 512 KB (เร็วสุด, DMA ได้), RTC memory 16 KB (รอดจาก deep sleep) | ROM 384 KB (bootloader), eFuse (MAC, ADC calibration, key) |
| **External** | PSRAM 8 MB (ผ่าน cache, ช้ากว่า) | Flash 16 MB (code + data, XIP) |

**ประเด็นสำคัญ:**

- **XIP (eXecute In Place)** — CPU รันโค้ดจาก Flash ผ่าน cache ได้โดยตรง แต่ระหว่างเขียน/ลบ Flash cache จะถูกปิด → ISR ต้องอยู่ใน IRAM (`IRAM_ATTR`)
- **DMA buffer** ต้องอยู่ใน internal SRAM เท่านั้น ใช้ `heap_caps_malloc(n, MALLOC_CAP_DMA)`
- `CONFIG_SPIRAM_MALLOC_ALWAYSINTERNAL=16384` → ของเล็ก ≤16 KB ลง SRAM เสมอ ของใหญ่ไป PSRAM
- **PSRAM ไม่ใช่ของแถม:** WiFi + lwIP กิน SRAM หนัก ถ้าไม่เปิด PSRAM heap เหลือไม่กี่ KB จน network stack ล่ม

**มุมมองฝั่ง firmware:**

| Region | เก็บอะไร |
|---|---|
| IROM / DROM | โค้ดและ const ส่วนใหญ่ (รันจาก Flash) |
| IRAM | ISR, hot path |
| DRAM | ตัวแปร global/static, heap |
| RTC | ข้อมูลที่ต้องข้าม deep sleep |

- **Partition table** แบ่ง Flash เป็น bootloader, app (`ota_0`/`ota_1` สำหรับ OTA), NVS (key-value เช่น WiFi credential), filesystem
- Flash ต้อง erase ก่อนเขียน (sector 4 KB) และมีอายุ ~100K รอบ → ห้ามใช้แทน RAM หรือ log ถี่ ๆ

### 3. อุปกรณ์บนบอร์ดและแผนที่ขา (GPIO)

| อุปกรณ์ | Interface | GPIO หลัก | ข้อควรรู้ |
|---|---|---|---|
| จอ ILI9341V 240×320 | SPI 4-line | SCK 12, MOSI 11, CS 10, DC 46, BL 45 | RGB565 ต้อง byte-swap, ต้องส่ง `INVON`, ใช้ dirty-rect + DMA |
| Touch FT6336G | I2C `0x38` | SDA 16, SCL 15, INT 17, RST 18 | I2C bus ใช้ร่วมกับ codec |
| Audio ES8311 + amp FM8002E + MEMS mic | I2C (control) + I2S (data) | MCLK 4, BCLK 5, WS 7, DOUT 8, DIN 6, PA_EN 1 (active-low) | ลำโพงเงียบ = ลืมดึง GPIO1 ลง low |
| RGB LED WS2812 | RMT | 42 | ต้องใช้ RMT ไม่ใช่ bit-bang (timing เพี้ยนเมื่อมี DMA/interrupt), ลำดับสี GRB |
| MicroSD | SDIO 4-bit | 38–41, 47, 48 | FAT32, ไม่มีขา card-detect |
| แบตเตอรี่ | TP4054 charger + ME6217 LDO | ADC 9 (divider 1/2) | TP4054 ไม่ใช่ fuel gauge ต้องวัดเองผ่าน ADC |
| USB | USB-Serial/JTAG | 19/20 | เปิด port = ชิป reset → รอ ~2.5 s |
| UART0 header | UART | TX 43 / RX 44 | serial ที่ไม่ auto-reset |

ขาที่ว่างจริงมีแค่ 4 ขา: **GPIO 2, 3, 14, 21** (expansion header) — ใช้ทำ UART1, SPI ชุดที่สอง, I2C ใหม่ หรือ ADC ได้

> [!WARNING]
> **GPIO 26–37 ห้ามใช้เด็ดขาด** (bus ของ Flash + PSRAM), GPIO 0/45/46 เป็น strapping pin, GPIO 19/20 คือ USB

### 4. Embedded C + FreeRTOS review

- `volatile` สำหรับตัวแปรที่ ISR แก้ (แต่ไม่ได้ทำให้ atomic)
- Bit manipulation, ใช้ type ขนาดชัดเจน (`uint8_t`, `uint32_t`), ระวัง endianness
- ใช้ static allocation เป็นค่าเริ่มต้น, ไม่ประกาศ array ใหญ่ใน stack ของ task
- **ISR rules:** ห้าม block, ห้าม `malloc`, ห้าม `printf` หนัก → ส่งงานต่อให้ task
- FreeRTOS: preemptive + priority, ใช้ `vTaskDelay()` แทน busy-wait

| เครื่องมือ IPC | ใช้เมื่อ |
|---|---|
| Queue | ส่งข้อมูลระหว่าง task หรือ ISR → task |
| Binary semaphore | ISR ส่งสัญญาณบอก task ว่ามีงาน |
| Mutex | ป้องกัน shared resource เช่น I2C bus (มี priority inheritance) |
| Task notification | signal เบา ๆ ตรงถึง task เดียว |
| Event group | รอหลาย flag พร้อมกัน |

**Pattern ที่ใช้จริง:** ISR → queue/notification → worker task, แยก task ตามหน้าที่, fixed pool แทน `malloc`

---

## Week 03 — ESP-IDF Setup & ESP32-S3 Hello World

**ESP-IDF** คือ framework ทางการของ Espressif (บน FreeRTOS) เหนือกว่า Arduino ตรงที่ควบคุม memory, power, security, OTA และ debug ได้เต็มที่

| เครื่องมือ | หน้าที่ |
|---|---|
| `idf.py` | คำสั่งกลาง: build, flash, monitor, menuconfig, size |
| CMake + Ninja | build system (component = folder ที่มี `CMakeLists.txt`) |
| `menuconfig` → `sdkconfig` | ตั้งค่าทั้งระบบ |
| `esptool` | คุยกับ ROM bootloader เพื่อ flash/erase |
| OpenOCD + GDB | debug ผ่าน JTAG |

### ขั้นตอนที่ทำ

1. ติดตั้ง Git, Python และ ESP-IDF v6.1 + tools สำหรับ ESP32-S3 (หรือใช้ VS Code extension "ESP-IDF")
2. ตรวจว่า `idf.py --version` ทำงาน
3. เปิด terminal ที่ activate แล้วในโฟลเดอร์โปรเจกต์ `esp32-s3`
4. Build → Flash → Monitor

```bash
idf.py set-target esp32s3
idf.py build
idf.py -p /dev/cu.usbmodemXXXX flash monitor   # Windows: idf.py -p COM5 flash monitor
```

ตัวอย่างโค้ด `app_main()` (Hello World):

```c
#include <stdio.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "esp_chip_info.h"

void app_main(void)
{
    printf("Hello world!\n");

    esp_chip_info_t chip_info;
    esp_chip_info(&chip_info);
    printf("%s, %d CPU core(s)\n", CONFIG_IDF_TARGET, chip_info.cores);

    for (int i = 10; i >= 0; i--) {
        printf("Restarting in %d seconds...\n", i);
        vTaskDelay(1000 / portTICK_PERIOD_MS);
    }
    esp_restart();
}
```

### ผลการทดลอง

- _(แปะผลที่ได้ เช่น chip ESP32-S3, 2 cores, flash 16 MB, minimum free heap ...)_

### ข้อควรระวัง

- Windows ใช้ **PowerShell** เท่านั้น (Git Bash ทำให้ toolchain พัง)
- ไม่ commit `build/`, `sdkconfig.old` ขึ้น Git (เก็บ `sdkconfig.defaults` ไว้)
- เข้า download mode ด้วยมือ: กด **BOOT** ค้าง → กด **RESET** → ปล่อย **BOOT**

### FreeRTOS บน PC Simulator / Digital Twin

ฝึก FreeRTOS บน PC ก่อนลงบอร์ดจริง ด้วย simulator และ digital twin แบบฝึกหัด: [drsanti/freertos-pc-workshops](https://github.com/drsanti/freertos-pc-workshops) — รัน FreeRTOS kernel จริงบน PC (`./run ch00`) พร้อมแอป board-twin จำลองบอร์ด

| Phase | บทที่ | เนื้อหา |
|---|---|---|
| Fundamentals | ch00–ch05 | Why RTOS, tasks & delay, task states, scheduler, priority & preemption, time slicing & starvation |
| Features | ch06–ch13 | Queue, binary semaphore, mutex, critical section, software timer, event group, task notification, capstone |
| Board | ch14–ch17 | GPIO output/input, UART console, ADC & PWM บน board-twin |

**สิ่งที่ฝึกทำ**

- _(บทที่ทำ เช่น ch01 สร้าง 2 task กระพริบคนละจังหวะ)_
- _(ผลที่สังเกต เช่น task priority สูงแย่ง CPU เมื่อไม่มี `vTaskDelay`)_

---

## Week 04 — Onsite Workshop: ESP-IDF + RTOS Demo บนบอร์ด ES3C28P

> Onsite **31 ต.ค. 2026** — ลงมือติดตั้ง ESP-IDF บนเครื่องตัวเอง และรันโค้ด RTOS demo บนบอร์ด ESP32-S3 Touch Screen (ES3C28P) ของจริง

### ขั้นตอนที่ทำ

1. ติดตั้ง ESP-IDF v6.1 + toolchain ESP32-S3 (ผ่าน VS Code extension หรือ course script) แล้วตรวจด้วย `idf.py --version`
2. ต่อบอร์ดผ่าน USB-C (ต้องเป็นสายที่มีเส้น data) และหา port ของบอร์ด
3. Build โค้ด RTOS demo → Flash ลงบอร์ด → เปิด Monitor ดูผล
4. สังเกตการทำงานของหลาย task บนบอร์ดจริง (จอ / touch / LED / serial log)

```bash
idf.py set-target esp32s3
idf.py build
idf.py -p /dev/cu.usbmodemXXXX flash monitor   # Windows: idf.py -p COM5 flash monitor
```

### รายละเอียด demo ที่รัน

- _(ชื่อ demo / โปรเจกต์ที่ได้รับ)_
- _(มีกี่ task แต่ละ task ทำอะไร เช่น task อ่าน touch, task วาดจอ, task กระพริบ LED)_
- _(ใช้ IPC อะไร เช่น queue / semaphore / mutex)_

### ผลการทดลอง

- _(สิ่งที่เห็นบนจอและใน monitor)_

### ปัญหาที่เจอและวิธีแก้

| อาการ | สาเหตุ / วิธีแก้ |
|---|---|
| เปิด monitor แล้วบอร์ด restart | ปกติของ USB-Serial/JTAG (auto-reset) — รอ ~2.5 s |
| Flash ไม่เข้า / หา port ไม่เจอ | เปลี่ยนสายที่มี data, หรือเข้า download mode: กด BOOT ค้าง → กด RESET → ปล่อย BOOT |
| port busy | ปิด monitor ตัวอื่นที่เปิดค้าง |
| _(ปัญหาที่เจอจริง)_ | _(วิธีแก้)_ |

---

## Week 05

**หัวข้อ:** _(หัวข้อสัปดาห์นี้)_

- _(สรุปสิ่งที่เรียน / ทำ)_

---

## สิ่งที่ได้เรียนรู้

- ต้องรู้ **ข้อจำกัดของฮาร์ดแวร์** ก่อนเขียนโค้ด: ขาไหนว่าง, memory ไหนใช้กับ DMA ได้, แรงดัน 3.3 V
- **Memory คือเรื่องใหญ่ของ embedded** — SRAM น้อย, PSRAM ช้าแต่จำเป็น, Flash มีอายุการเขียน
- ลองบน **simulator** ก่อน แล้วค่อยลง **บอร์ดจริง** ช่วยแยกปัญหาโค้ดออกจากปัญหาฮาร์ดแวร์ (สาย, port, ไฟ)
- ESP-IDF + FreeRTOS ให้การควบคุมระดับ production (multitask, OTA, security, debug)
- แม้ AI จะช่วยเขียนโค้ดได้ แต่ต้องเข้าใจพื้นฐานเองเพื่อ review โค้ดและ debug อาการบนฮาร์ดแวร์จริง
- _(เพิ่มข้อคิดของตัวเอง)_

---

## อ้างอิง

- [KMUTT ND69 — ND01 documents](https://github.com/drsanti/kmutt-nd69/tree/main/nd1/docs)
- Week 02 lessons (ES3C28P hardware)
- Week 03 lessons (ESP-IDF setup)
- [FreeRTOS PC Workshops](https://github.com/drsanti/freertos-pc-workshops)
- GitHub Hands-On: A Practical Guide for Beginners
