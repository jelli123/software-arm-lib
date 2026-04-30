# software-arm-lib
Repository for the ARM library.

## RP2350 Port (Work in Progress)

Die sblib wurde auf den RP2350 (Raspberry Pi Pico 2, Cortex-M33) portiert.
Der bestehende LPC11xx-Code bleibt vollständig funktionsfähig — alle plattformspezifischen
Abschnitte sind mit `#if !defined(__SBLIB_TARGET_RP2350__)` geschützt.

### Architektur

| Schicht | Beschreibung |
|---------|-------------|
| **HAL** | `sblib/src/hal/rp2350/` — Plattform-Abstraktion (GPIO, Timer, Flash, Serial, I2C, SPI, ADC) |
| **Bus-Interface** | `KnxBusPio` (PIO-basiert) und `KnxBusTpuart` (TPUART-IC) ersetzen die alte LPC `Bus`-Klasse |
| **KNX-Stack** | Plattformunabhängiger Code (`bcu_base`, `com_objects`, `knx_tlayer4` etc.) unverändert nutzbar |
| **CMake** | `sblib/sblib_rp2350.cmake` + `examples/rp2350_example_common.cmake` |

### Durchgeführte Anpassungen

- **BcuBase Refactoring** — `KnxBusInterface` / `KnxBusCallback` Abstraktion eingeführt
- **PIO State Machine** — KNX TP1 TX über PIO-Hardware (`knx_tp1.pio`), RX über GPIO-Interrupt mit µs-Timestamps
- **Schmitt-Trigger** — Explizit aktiviert am RX-Pin (`gpio_set_input_hysteresis_enabled`) für saubere Flankenerkennung
- **Alarm Timer** — `hardware_timer`-basiert (Pico SDK `add_alarm_in_us`)
- **MemMapper** — Optimiert (256-Byte `allocTable` aus RAM entfernt, on-the-fly berechnet)
- **HAL-Implementierungen:**
  - `digital_pin_rp2350.cpp` — `pinMode`, `digitalWrite`, `digitalRead`
  - `timer_rp2350.cpp` — Timer-Klasse via Pico SDK `repeating_timer`
  - `serial_rp2350.cpp` — UART über Pico SDK `uart_*`
  - `i2c_rp2350.cpp` — I2C über Pico SDK `i2c_*`
  - `spi_rp2350.cpp` — SPI über Pico SDK `spi_*`
  - `analog_pin_rp2350.cpp` — ADC über Pico SDK `adc_*`
  - `iap_rp2350.cpp` — Flash-Zugriff via `hardware_flash`
  - `utils_rp2350.cpp` — `fatalError`, `hashUID` etc.
  - `platform_hal_rp2350.cpp`, `gpio_hal_rp2350.cpp`, `flash_hal_rp2350.cpp`
- **Guards in bestehenden Dateien:**
  - `serial.cpp`, `serial0.cpp`, `i2c.cpp`, `spi.cpp`, `analog_pin.cpp`, `timer.cpp`, `utils.cpp` — komplett per `#if !defined(...)` ausgeblendet
  - `serial.h` — RP2350-Default-Pins (GP0/GP1) statt LPC PIO1_7/PIO1_6
  - `digital_pin.h`, `interrupt.h`, `timer.h`, `platform.h` — Inline-Funktionen/Typen per Guard getrennt
  - `bcu_base.cpp` — `watchdog_reboot()` statt `NVIC_SystemReset()` auf RP2350
  - `main.cpp` — `stdio_init_all()` statt `SysTick_Config()` auf RP2350
- **Nicht im RP2350-Build** (bewusst ausgeschlossen): `bus.cpp`, `bus_debug.cpp`, `ioports.cpp`, `digital_pin_shift.cpp`, `lpc11xx/*`, `lcd/*`

### Beispielprojekte (RP2350)

| Beispiel | Beschreibung |
|----------|-------------|
| `example-rp2350-blink` | GPIO-Blink mit `digitalWrite` |
| `example-rp2350-serial` | UART-Ausgabe |
| `example-rp2350-timer-int` | Timer-Interrupt (repeating) |
| `example-rp2350-analog` | ADC-Messung |
| `example-rp2350-spi` | SPI-Kommunikation |
| `example-rp2350-pwm` | PWM-Ausgabe |
| `example-rp2350-i2c-sht4x` | I2C Sensor (SHT4x) |
| `example-rp2350-i2c-bh1750` | I2C Sensor (BH1750) |
| `example-rp2350-ds18x20` | 1-Wire Temperatursensor |
| `example-rp2350-knx-tpuart` | KNX via TPUART-IC |

### Build (RP2350)

```bash
mkdir build && cd build
cmake -DPICO_SDK_PATH=/path/to/pico-sdk -DPICO_BOARD=pico2 ..
make
```

---
