# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project overview

`no_std` Rust port of a weather app targeting the Waveshare ESP32-S3-Touch-LCD-5B (SKU 28151).

- **Board**: ESP32-S3-WROOM-1-N16R8 — 16 MB flash, 8 MB octal PSRAM
- **Display**: 5" IPS RGB565, 1024×600, 16-bit parallel RGB (not SPI/QSPI)
- **Touch**: GT911 5-pt capacitive, I2C @ 0x5D (SDA=GPIO8, SCL=GPIO9, IRQ=GPIO4)
- **IO expander**: CH422G @ I2C 0x24 — controls LCD_BL (EXIO2), LCD_RST (EXIO3), TP_RST (EXIO1)
- **RTC**: PCF85063 @ I2C 0x51; **BME280** sensor @ 0x76 or 0x77 (add-on, same bus)
- **Console**: USB-CDC on GPIO19/20 → `/dev/ttyACM0`. RS485 on GPIO43/44 — **no logs there**.
- **Reference** (`std`) app: `/data/rust/waveshare-esp32-s3-touch-lcd-5b` — use as behavioral spec, not copy source
- **BSP crate** (if needed): `/data/rust/waveshare-esp32-s3-touch-lcd-5b-nostd-bsp` — sibling directory for a `no_std` BSP (pin constants, timing, board init). Create it there if the firmware crate needs a shared BSP.
- **Implementation plan**: `esp32_s3_weather_nostd_port_plan.md` in the project root

## Shell / CLI rules

Default shell is **fish**. All commands must be fish-compatible:

- No `[[ ]]`, no `(( ))`, no `export VAR=value`, no `VAR=value cmd` prefix
- Fish equivalents: `set VAR value`, `set -x VAR value`, `env VAR=value cmd`
- Avoid `&&` / `||` chaining — use separate lines or a script
- Wrap commands in fenced code blocks, no line numbers, no placeholders

## Workspace layout (target)

```
/data/rust/waveshare-esp32-s3-touch-lcd-5b-nostd/
  Cargo.toml             workspace manifest
  rust-toolchain.toml    pins esp toolchain
  crates/
    weather-core/        #![no_std] app logic — data model, history, stats, JSON
    weather-desktop/     std simulator/test harness
    weather-s3/          ESP32-S3 firmware (esp-hal + Embassy)
      src/board/         hardware glue: ch422g.rs, display.rs, pins.rs, touch.rs
      src/tasks/         Embassy tasks: display.rs, sensor.rs, web.rs, wifi.rs
```

## Build and test

Generate firmware crate (once, from project root):

```bash
mkdir -p crates
cd crates
esp-generate --chip=esp32s3 weather-s3
```

Build firmware:

```bash
cd crates/weather-s3
cargo +esp build -Zbuild-std=core,alloc
```

Test `weather-core` on desktop (no hardware needed):

```bash
cargo test -p weather-core
```

Flash (confirm `/dev/ttyACM0` is the device, ask user before flashing):

```bash
espflash flash --monitor target/xtensa-esp32s3-none-elf/debug/weather-s3
```

Check for stale processes before flashing:

```bash
fuser /dev/ttyACM0
```

## Pinned dependency baseline

From esp-generate 2026-04-11 — verify against current template output before changing:

```toml
esp-hal = { version = "=1.0.0", features = ["esp32s3", "unstable"] }
esp-rtos = { version = "0.2.0", features = ["embassy", "esp-alloc", "esp32s3"] }
esp-bootloader-esp-idf = { version = "0.4.0", features = ["esp32s3"] }
esp-backtrace = { version = "0.18.1", features = ["esp32s3", "panic-handler", "println"] }
esp-println = { version = "0.16.1", features = ["esp32s3"] }
esp-alloc = "0.9.0"
embassy-executor = "0.9.1"
embassy-time = "0.5.0"
```

Radio builds need:

```toml
[profile.dev.package.esp-radio]
opt-level = 3
```

Do not upgrade crates while debugging. Pin versions once a build works.

## Key hardware constraints

**CH422G init is required before anything else works.** The display backlight, LCD reset, and touch reset are all behind this expander. Without it the display stays dark and touch stays in reset.

Init sequence:
1. I2C on GPIO8/9
2. Write `0x01` to address `0x24` (OEN enable)
3. GPIO output register is at address `0x38` (not `0x23`)
4. EXIO2=LCD_BL (bit 2), EXIO3=LCD_RST (bit 3), EXIO1=TP_RST (bit 1)
5. Assert LCD_RST low, hold, deassert; same for TP_RST; then enable LCD_BL

**PCLK must be 20 MHz.** 24/30 MHz cause pixel scrambling under WiFi load. Do not raise it.

**Parallel RGB pin order** for `data_gpio_nums[]` (R3→R7, G2→G7, B3→B7):
```
[1, 2, 42, 41, 40, 39, 0, 45, 48, 47, 21, 14, 38, 18, 17, 10]
```
PCLK=GPIO7, HSYNC=GPIO46, VSYNC=GPIO3, DE=GPIO5.

GPIO0 (G3), GPIO46 (HSYNC), GPIO3 (VSYNC) are strapping pins — must not be driven low at boot.

**Double framebuffer** (1024×600×2 bytes×2 ≈ 2.5 MB) must live in PSRAM. Do not enable PSRAM until the display phase — the firmware should boot and serve sensor data without it.

## `weather-core` rules

Must compile with `#![no_std]`. Avoid `std::thread`, `std::time`, `std::sync`, `std::fs`, `std::net`, `HashMap`, `println!`. Prefer `heapless`, `core`, array-backed ring buffers, fixed-capacity strings. Use `alloc` only when justified.

State object lives in `weather-core` and is shared across Embassy tasks via `embassy_sync::mutex::Mutex`. Do not put Wi-Fi handles, display objects, timers, or Embassy types into `weather-core`.

Sample representation uses integer-scaled fields to avoid floats in storage:

```rust
pub struct WeatherSample {
    pub timestamp_seconds: u64,
    pub temperature_c_x100: i16,
    pub humidity_x100: u16,
    pub pressure_pa: u32,
}
```

History size: start at 120–240 samples while proving the port; target 1440 (24h at 1/min = ~23 KB).

## Implementation order (milestones)

Follow the milestone sequence in `esp32_s3_weather_nostd_port_plan.md`. Short version:
1. Serial boot log (hello world over USB-CDC)
2. I2C scan — all four devices visible: 0x24, 0x51, 0x5D, 0x76/0x77
3. BME280 sensor read over serial
4. `weather-core` extraction with desktop tests
5. Embassy task skeleton (heartbeat + fake sensor)
6. Wi-Fi connect
7. `GET /api/latest` JSON endpoint
8. Real sensor → WeatherState → HTTP
9. `/api/history` endpoint
10. Parallel RGB display init (PCLK 20 MHz, double PSRAM framebuffer)
11. GT911 touch
12. Browser chart dashboard integration

Do not attempt display before I2C scan and sensor are working.

## Architecture: Embassy task split

```
main/init  → HAL + clocks + I2C + CH422G + spawn tasks
wifi_task  → connect, monitor, reconnect
sensor_task → read BME280, update WeatherState
web_task   → serve /api/latest, /api/history, /api/status
display_task → parallel RGB init, draw from WeatherState
```

Use HTTP only (no TLS on-device). Remote secure access goes through Nginx/Tailscale externally.

## What NOT to do

- Rewrite the whole app at once
- Use HTTPS/TLS on-device in early milestones
- Use dynamic allocation everywhere
- Mix ESP-IDF `std` and `esp-hal` `no_std` in the same firmware
- Upgrade crate versions while debugging
- Invent pin mappings — use the verified list above
- Skip CH422G init and wonder why the display is dark
- Assume PCLK above 20 MHz is safe
- Call Wi-Fi stable until it has run for hours
- Copy the NTFS `.cargo/config.toml` target-dir redirect from the std reference — not needed here (source is on native Linux ext4/btrfs)

## Discuss before coding

Do not change code while the user is still asking questions. Wait for explicit approval ("go", "yes", "do it") before editing files.
