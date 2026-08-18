# immurok Hardware

## Component Selection

### MCU: WCH CH592F

| Spec | Value |
|------|-------|
| Core | RISC-V (QingKe V4C), 60 MHz |
| Flash | 448 KB |
| SRAM | 26 KB |
| Wireless | BLE 5.4 |
| Package | QFN28 (4x4 mm) |
| Supply voltage | 1.8–3.6 V |
| Sleep current | < 1 uA (shutdown), ~2.5 uA (idle with RTC) |

**Why CH592F:**

- Native BLE 5.4 with HID keyboard profile support — critical for macOS auto-reconnect. macOS maintains persistent connections to HID peripherals, so by appearing as a keyboard the device stays connected without application-level keep-alive.
- Large flash (448 KB) accommodates the OTA dual-image scheme (two 216 KB slots) plus bootloaders.
- Built-in hardware AES and RNG accelerators used for OTA decryption and ECDH key generation.
- Ultra-low sleep current enables battery-powered designs in future revisions.
- Low cost (~$0.60 USD in quantity).

**Trade-offs:**

- No hardware ECC accelerator — ECDH P-256 key generation takes ~2 seconds in software (via `uECC`). Mitigated by deferring computation to TMOS cooperative scheduler events so BLE stays responsive.
- Closed-source BLE stack (WCH `libCH59xBLE.a`). The application-layer code is fully open source; the link layer is provided as a binary library.

### Fingerprint Sensor: R559S

| Spec | Value |
|------|-------|
| Type | Capacitive area sensor |
| Resolution | 508 DPI |
| Sensor area | 8.0 x 8.0 mm |
| Template capacity | 29 fingerprints |
| False acceptance rate (FAR) | < 0.001% |
| False rejection rate (FRR) | < 1% |
| Match time | < 500 ms |
| Interface | UART, 57600 baud, 8N2 |
| Operating voltage | 3.3 V |
| Touch detection | Capacitive, interrupt output (active high) |

**Why R559S:**

- Self-contained fingerprint processor — template enrollment, storage, and matching all happen on-chip. The host MCU only receives match/no-match results and template IDs, never raw biometric data.
- Built-in capacitive touch detection with interrupt output, eliminating the need for a separate touch controller.
- UART interface keeps wiring simple (4 wires: TX, RX, power, touch interrupt).
- 29-template capacity is sufficient for single-user multi-finger enrollment.

**Security properties:**

- Templates stored in R559S internal flash, not accessible from the host MCU.
- Module password protection (derived from device MAC address) prevents unauthorized access.
- No API to extract raw template data — only match results are returned.

### Connection: Bluetooth Low Energy

| Spec | Value |
|------|-------|
| Standard | BLE 5.4 |
| Profiles | HID Keyboard (0x1812) + Custom GATT |
| Custom service UUID | `12340010-0000-1000-8000-00805f9b34fb` |
| Bonding | Yes, No Input No Output |
| Connection interval | 30–50 ms |
| Slave latency | 29 intervals |
| Supervision timeout | 6 s |
| Effective idle interval | ~1.5 s |

**Why BLE HID Keyboard:**

- macOS and Linux maintain automatic persistent connections to bonded HID peripherals. A custom GATT-only device would be disconnected when no application is actively scanning.
- HID keyboard capability is also used functionally: the device sends a Ctrl keypress to pre-trigger authentication dialogs before the fingerprint match notification arrives.
- Actual command/response communication uses the custom GATT service, not HID reports.

### Power: LiPo Battery + USB-C Charging

| Spec | Value |
|------|-------|
| Battery | LiPo cell, 110 mAh |
| Charger | TI BQ21040 with NTC thermistor — hardware JEITA thermal cutoff (IEC 62368-1) |
| Main regulator | XC6206 3.3 V LDO |
| Sensor rail | Load switch gated by SENSOR_EN — the R559S is fully powered off between touches |
| USB-C | Charging only — 6-pin power-only receptacle, no data lines |
| Power switch | Slide switch between battery and the regulator input |

---

## GPIO Pinout (production hardware, VER=6)

| Pin | Function | Notes |
|-----|----------|-------|
| PA5 | UART3 TX — debug log output | 115200 8N1, debug builds only; **DEBUG** pad on the PCB |
| PA8 | UART1 RX — R559S data / serial ISP | 57600 8N2 (sensor); ROM-bootloader flashing; **RXD1** pad |
| PA9 | UART1 TX — R559S data / serial ISP | **TXD1** pad |
| PA10/PA11 | 32.768 kHz crystal | |
| PA14 | Battery voltage (AIN4) | 1/4 divider (3 MΩ / 1 MΩ) |
| PB4 | Button | Active low, hardware RC debounce |
| PB7 | LED red | |
| PB10 | Tamper switch (ANTI_OPEN) | High = case opened; floating input |
| PB12 | Fingerprint sensor power (SENSOR_EN) | Active high, gates the sensor's load switch |
| PB13 | Touch interrupt (DETECT) | Active high |
| PB14/PB15 | WCH 2-wire debug (SWDIO/SWCLK) | Not exposed on the production PCB |
| PB22 | LED blue / **BOOT** | Held low at power-on → ROM bootloader (**BOOT** pad) |
| PB23 | LED green | RST alternate function kept disabled |

Unused pins are configured as pull-up inputs in `main()` to minimize leakage current.

When the fingerprint module is powered off, the UART1 pins are switched to pull-down inputs to prevent current leaking through ESD protection diodes, and the UART1 clock is gated.

### Debug & Flashing Pads

The back of the production PCB exposes **VCC5 / TXD1 / RXD1 / DEBUG / GND** pads and a separate **BOOT** pad (with its own GND):

- **TXD1 / RXD1 + BOOT**: serial ISP firmware flashing via the CH592F ROM bootloader — see [firmware/README.md → Flashing](https://github.com/immurok/firmware#flashing)
- **DEBUG**: UART3 log output (115200 8N1) from `debug` / `release-debug` firmware builds

---

## Wiring Diagram

```
 USB-C (5V, power only)          LiPo 110 mAh
        │                            │
        ▼            charge          │
   ┌─────────┐    ┌──────────┐       │
   │ BQ21040 ├───►│ battery  ├── [power switch]
   └─────────┘    └──────────┘       │
                                     ▼
                              ┌────────────┐ 3.3V
                              │ XC6206 LDO ├──────┬──────────────┐
                              └────────────┘      │              │
                                             ┌────▼────┐   ┌─────▼─────┐
                                             │ CH592F  │   │load switch│◄─ PB12 SENSOR_EN
                                             │         │   └─────┬─────┘
                                             │    PA9 ─┼───►┌────▼────┐
                                             │    PA8 ─┼───◄│  R559S  │
                                             │   PB13 ─┼───◄│  TOUCH  │
                                             │         │    └─────────┘
                                             │    PB4 ─┼──[BTN]
                                             │   PB10 ─┼──[tamper switch]
                                             │ PB7/22/23┼──[RGB LED]
                                             └─────────┘
```

## Schematic & PCB

- [`schematic/Schematic1.6.5.pdf`](schematic/Schematic1.6.5.pdf) — complete schematic, current production revision (HW v1.6.5)
- [`pcb/`](pcb/) — PCB layout renders (both sides); layout source files will be released after the Kickstarter campaign
