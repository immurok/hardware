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

### Fingerprint Sensor: ZW3021

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

**Why ZW3021:**

- Self-contained fingerprint processor — template enrollment, storage, and matching all happen on-chip. The host MCU only receives match/no-match results and template IDs, never raw biometric data.
- Built-in capacitive touch detection with interrupt output, eliminating the need for a separate touch controller.
- UART interface keeps wiring simple (4 wires: TX, RX, power, touch interrupt).
- 29-template capacity is sufficient for single-user multi-finger enrollment.

**Security properties:**

- Templates stored in ZW3021 internal flash, not accessible from the host MCU.
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

### Power: USB-C

Current design is bus-powered via USB-C. No battery. Future revisions may add a rechargeable LiPo cell for wireless operation.

---

## GPIO Pinout

```
                    CH592F (QFN28)
                   ┌──────────────┐
            PA4  ──┤ UART3 RX     │   Debug serial (115200)
            PA5  ──┤ UART3 TX     │   Debug serial (115200)
            PA8  ──┤ UART1 RX     │   ZW3021 data (57600 8N2)
            PA9  ──┤ UART1 TX     │   ZW3021 data (57600 8N2)
            PA12 ──┤ GPIO OUT     │   ZW3021 power (active high)
            PA13 ──┤ GPIO IN      │   ZW3021 touch INT (rising edge)
            PA14 ──┤ GPIO IN      │   Button (falling edge, pull-up)
                   └──────────────┘
```

| Pin | Function | Direction | Mode | Peripheral | Notes |
|-----|----------|-----------|------|------------|-------|
| PA4 | UART3 RX | Input | Pull-up | Debug serial | 115200 baud |
| PA5 | UART3 TX | Output | Push-pull 5 mA | Debug serial | 115200 baud |
| PA8 | UART1 RX | Input | Pull-up | ZW3021 | 57600 baud, 8N2 |
| PA9 | UART1 TX | Output | Push-pull 5 mA | ZW3021 | 57600 baud, 8N2 |
| PA12 | Power control | Output | Push-pull 5 mA | ZW3021 | High = on |
| PA13 | Touch interrupt | Input | Pull-down | ZW3021 | Rising edge, active high |
| PA14 | Button | Input | Pull-up | Physical button | Falling edge, active low |

Unused pins are configured as pull-up inputs in `main()` to minimize leakage current.

When the fingerprint module is powered off, PA8/PA9/PA12 are switched to pull-down inputs to prevent current leaking through ESD protection diodes. UART1 clock is also gated.

---

## Wiring Diagram

```
                USB-C
                  │
                  ▼
              ┌───────┐
              │ 3.3V  │
              │ REG   │
              └───┬───┘
                  │ 3.3V
        ┌─────────┼─────────┐
        │         │         │
   ┌────▼────┐    │    ┌────▼────┐
   │ CH592F  │    │    │ ZW3021  │
   │         │    │    │         │
   │    PA9 ─┼────┼───►│ RX      │
   │    PA8 ─┼────┼───◄│ TX      │
   │   PA12 ─┼────┼───►│ PWR_EN  │
   │   PA13 ─┼────┼───◄│ TOUCH   │
   │         │    │    │         │
   │    PA14─┼──[BTN]──┤         │
   │         │         │         │
   │  PA4/5 ─┼──[DBG]  │         │
   └─────────┘         └─────────┘
```

## Schematic & PCB

- `schematic/` — Full schematic files (open source)
- `pcb/` — PCB layout files (released after Kickstarter campaign)
