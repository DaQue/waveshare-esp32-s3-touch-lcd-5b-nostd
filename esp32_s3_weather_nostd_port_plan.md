# ESP32-S3 Weather App `no_std` Rust Port Plan

Target board: Waveshare ESP32-S3-Touch-LCD-5B (SKU 28151) — 5" IPS RGB565 1024×600, parallel RGB panel  
Reference (`std`) project: `/data/rust/waveshare-esp32-s3-touch-lcd-5b`  
New (`no_std`) project: `/data/rust/waveshare-esp32-s3-touch-lcd-5b-nostd`  
Machine: G760 — `/home/davidq` on nvme0n1p2 (488 GB, label G760), `/data` on nvme0n1p4 (1.3 TB, label DATA)  
Shell: fish

This file is written for CLI-based AI agents such as Codex, Claude Code, Gemini CLI, or Cursor agent. Follow it as an implementation plan, not as a loose brainstorming document.

---

## 0. Current conclusion

Porting is now worth attempting.

The old blocker was that the Rust ESP `no_std` ecosystem was not mature enough for the project's needs, especially Wi-Fi/networking. That has changed enough that a staged proof-of-life port is reasonable now.

Current ecosystem assumptions:

- Use `esp-hal 1.0.0` for bare-metal `no_std` ESP32-S3 support.
- Use `esp-radio` (the successor to `esp-wifi`) for Wi-Fi; check current examples before assuming the crate name.
- Use Embassy async for cooperative tasks via `esp-rtos` (wraps Embassy executor + timer setup).
- Use `embassy-net` for IP networking.
- Keep the existing `std` app at `/data/rust/waveshare-esp32-s3-touch-lcd-5b` as the reference implementation.
- Do not rewrite the app blindly.

Important caveat: even when the Rust app is `no_std`, ESP Wi-Fi still depends on vendor radio firmware/blobs internally. The practical goal is `no_std` application firmware, not "pure Rust all the way down."

---

## 1. Board specifics — what differs from generic ESP32-S3 templates

### 1.1 Hardware summary

- MCU: ESP32-S3-WROOM-1-N16R8 — 240 MHz, 16 MB flash, 8 MB octal PSRAM
- Display: 5" IPS RGB565, 1024×600, 16-bit parallel RGB interface via ESP32-S3 LCD peripheral
- Touch: GT911 5-point capacitive, I2C @ 0x5D (SDA=GPIO8, SCL=GPIO9, IRQ=GPIO4)
- IO expander: CH422G @ I2C 0x24 — controls LCD_BL (EXIO2), LCD_RST (EXIO3), TP_RST (EXIO1)
- RTC: PCF85063 @ I2C 0x51 (same I2C bus)
- Sensor: BME280 on same I2C bus (add-on, not factory fitted — verify address 0x76 or 0x77)
- SD card: SPI (MOSI=GPIO11, SCK=GPIO12, MISO=GPIO13, CS=CH422G EXIO4)
- USB: native USB-CDC on GPIO19/20 (VID:PID 303a:1001, `/dev/ttyACM0`) — **console goes here**
- RS485: GPIO43/44 — do not use for logs
- No IMU (the 3.5B had QMI8658; the 5B does not)

### 1.2 Parallel RGB display — not SPI/QSPI

The 5B display uses a 16-bit parallel RGB interface driven by the ESP32-S3's built-in LCD controller, not SPI. This is fundamentally different from the 3.5B's QSPI AXS15231B panel.

esp-hal exposes this as the `lcd_cam::lcd` peripheral. The driver must:

- Configure 16 data GPIOs (R3–R7, G2–G7, B3–B7)
- Configure PCLK (GPIO7), HSYNC (GPIO46), VSYNC (GPIO3), DE (GPIO5)
- Set PCLK to 20 MHz (20 MHz is the only verified-stable clock — 24/30/40 MHz cause pixel scrambling under WiFi load in the std reference)
- Use double framebuffer in PSRAM
- Configure bounce buffer (10 × H_RES pixels)

Pin list for `data_gpio_nums[]` (R3→R7, G2→G7, B3→B7):

```text
[1, 2, 42, 41, 40, 39, 0, 45, 48, 47, 21, 14, 38, 18, 17, 10]
```

GPIO0 (G3) is a strapping pin — must be high at boot. Confirm it is not driven low.  
GPIO46 (HSYNC) and GPIO3 (VSYNC) are strapping pins — confirm no boot conflicts.

### 1.3 CH422G IO expander — required before display or touch works

The CH422G must be initialized over I2C (0x24) before the display backlight, LCD reset, or touch reset can be controlled. Without this the panel stays dark and touch stays in reset.

Sequence:

1. Initialize I2C (GPIO8=SDA, GPIO9=SCL)
2. Write CH422G OE register to enable outputs
3. Assert LCD_RST low (EXIO3), hold, then deassert high
4. Assert TP_RST low (EXIO1), hold, then deassert high
5. Enable LCD_BL (EXIO2)

There is no existing `no_std` Rust crate for CH422G as of this writing. A small driver (20–30 lines) needs to be written inline or in a local module. The reference project (`/data/rust/waveshare-esp32-s3-touch-lcd-5b`) has a working C-level sequence via `components/` — use it as the behavioral reference.

### 1.4 GT911 touch

- I2C address: 0x5D
- IRQ: GPIO4
- Reset: CH422G EXIO1 (shared with TP_RST)
- No existing `no_std` GT911 Rust crate is verified — may need a small hand-rolled driver or adaptation

### 1.5 No NTFS, no target-dir redirect needed

The std reference project had an NTFS workaround: source on `/mnt/Shared` (NTFS via ntfs3 driver) forced the target dir off-tree to `/home/david/.cache/ws-5b-target`. On the G760 machine, all project source lives on native Linux filesystems:

- Source: `/data/rust/waveshare-esp32-s3-touch-lcd-5b-nostd` (DATA partition, ext4/btrfs)
- Home: `/home/davidq` (G760 partition)

No target-dir redirect is needed. Do not copy the NTFS workaround from the old CLAUDE.md.

---

## 2. Design principle

Do not port the whole app at once.

Split the project into three layers:

```text
weather-core/       no_std-compatible app logic
weather-desktop/    std simulator/test harness using existing app behavior
weather-s3/         ESP32-S3 hardware firmware using esp-hal + Embassy
```

The existing `std` program at `/data/rust/waveshare-esp32-s3-touch-lcd-5b` is the best reference for design decisions, data shape, screen behavior, JSON format, sampling cadence, and chart logic.

It should not be copied wholesale into the firmware.

Use the existing program as:

- the behavioral spec,
- the desktop simulator,
- the comparison target,
- and the source of already-made design choices.

---

## 3. What should be preserved from the existing `std` app

Preserve these parts as much as possible:

- Weather sample data model.
- Temperature, humidity, and pressure fields.
- Unit conversion rules.
- Sensor sampling cadence.
- History retention design.
- Ring buffer behavior.
- Min/max/average calculations.
- JSON/API response shape.
- Chart data format.
- Display page/state decisions.
- Error/status concepts such as Wi-Fi connected, sensor missing, stale data, etc.
- Any naming conventions already used by the local web dashboard.

These are app decisions, not platform decisions.

---

## 4. What must be rewritten for `no_std`

Expect to rewrite or heavily adapt:

- Wi-Fi setup.
- HTTP server/client code.
- Threading model.
- Sleep/timer logic.
- `std::time` usage.
- `std::sync` usage.
- File I/O.
- Heap-heavy strings and vectors.
- Display driver glue (parallel RGB, not SPI/QSPI).
- CH422G IO expander driver.
- Sensor driver async/blocking interface.
- Logging setup.
- Panic handler and boot/runtime setup.

Embassy should mostly change the outer control flow, not the weather logic.

Current style:

```text
read sensor -> update history -> generate JSON -> update display
```

Embassy style:

```text
sensor_task  -> periodically reads sensor and updates shared state
web_task     -> serves current/history JSON from shared state
display_task -> redraws from shared state
wifi_task    -> connects/reconnects Wi-Fi
```

---

## 5. Target architecture

### 5.1 Workspace layout

All work lives inside the project root on DATA:

```text
/data/rust/waveshare-esp32-s3-touch-lcd-5b-nostd/
  Cargo.toml             workspace manifest
  rust-toolchain.toml    pins esp toolchain
  crates/
    weather-core/        no_std app logic
      Cargo.toml
      src/lib.rs
      src/sample.rs
      src/history.rs
      src/stats.rs
      src/json.rs
      src/state.rs
    weather-desktop/     std simulator
      Cargo.toml
      src/main.rs
    weather-s3/          ESP32-S3 firmware
      Cargo.toml
      src/main.rs
      src/board/
        mod.rs
        ch422g.rs        CH422G IO expander driver
        display.rs       parallel RGB panel init
        pins.rs          GPIO constants
        touch.rs         GT911 driver
      src/tasks/
        mod.rs
        display.rs
        sensor.rs
        web.rs
        wifi.rs
```

### 5.2 Core crate rule

`weather-core` must be usable with:

```rust
#![no_std]
```

Avoid these in `weather-core`:

```rust
std::thread
std::time
std::sync
std::fs
std::net
Vec unless behind alloc and justified
String unless behind alloc and justified
HashMap
println!
```

Prefer:

```rust
core
heapless
array-backed ring buffers
fixed-capacity strings
small enums
explicit error types
```

### 5.3 Shared app state

Define a central state object in `weather-core`:

```rust
pub struct WeatherState<const N: usize> {
    pub latest: Option<WeatherSample>,
    pub history: HistoryBuffer<N>,
    pub status: SystemStatus,
}
```

Keep state mutation simple:

```rust
impl<const N: usize> WeatherState<N> {
    pub fn ingest_sample(&mut self, sample: WeatherSample) {
        self.latest = Some(sample);
        self.history.push(sample);
    }
}
```

Do not put Wi-Fi handles, display objects, timers, or Embassy types into `weather-core`.

---

## 6. Embassy task model

Use Embassy in `weather-s3` only.

Entry point pattern (current esp-hal 1.0.0 / esp-rtos style):

```rust
#[esp_rtos::main]
async fn main(spawner: Spawner) -> ! { ... }
```

Proposed tasks:

```text
main/init
  - initialize HAL and clocks
  - initialize I2C (GPIO8/9)
  - initialize CH422G (reset LCD + touch)
  - initialize Embassy via esp-rtos
  - create shared state
  - spawn tasks

wifi_task
  - connect to configured SSID
  - monitor link state
  - reconnect on failure
  - signal network readiness

sensor_task
  - initialize BME280 over I2C
  - read every configured interval
  - update shared WeatherState
  - signal display/web tasks if needed

web_task
  - wait for network readiness
  - serve /api/latest
  - serve /api/history
  - optionally serve /api/status

display_task
  - initialize CH422G outputs (already done in main)
  - initialize parallel RGB LCD via esp-hal lcd_cam
  - allocate framebuffer in PSRAM
  - draw boot/status screen
  - periodically redraw latest values from WeatherState

watchdog_task
  - optional later phase
  - track stale sensor readings and network failures
```

---

## 7. Shared state in Embassy

Use Embassy synchronization primitives in the firmware crate only.

Likely choices:

```rust
embassy_sync::mutex::Mutex
embassy_sync::blocking_mutex::raw::CriticalSectionRawMutex
embassy_sync::signal::Signal
embassy_sync::channel::Channel
```

Pattern:

```rust
static WEATHER_STATE: Mutex<CriticalSectionRawMutex, WeatherState<1440>> = ...;
static SAMPLE_SIGNAL: Signal<CriticalSectionRawMutex, ()> = Signal::new();
```

Use a small fixed history size at first.

Possible history sizes:

```text
120 samples   = 2 hours at 1/minute
720 samples   = 12 hours at 1/minute
1440 samples  = 24 hours at 1/minute
```

Start with 120 or 240 while proving the port. Increase later.

---

## 8. Networking strategy

### 8.1 First network goal

Do not start with the full dashboard.

First goal:

```text
ESP32-S3 connects to Wi-Fi and serves one endpoint:
GET /api/latest
```

Expected test response:

```json
{
  "ok": true,
  "temperature_f": 72.4,
  "humidity_percent": 45.2,
  "pressure_hpa": 1012.8
}
```

### 8.2 Avoid HTTPS on-device initially

Do not implement TLS/HTTPS on the ESP32-S3 at first.

Use local HTTP on the device and handle remote secure access externally:

```text
ESP32-S3 local HTTP
  -> LAN
  -> optional Nginx reverse proxy
  -> optional Tailscale Serve/TLS
```

Reason:

- TLS increases RAM pressure.
- Certificates complicate embedded builds.
- The project's local dashboard design does not require the device itself to terminate HTTPS.

### 8.3 Endpoints to implement

Phase 1:

```text
GET /api/latest
GET /api/status
```

Phase 2:

```text
GET /api/history?window=2h
GET /api/history?window=24h
```

Phase 3:

```text
GET /
GET /app.js
GET /style.css
```

Optional: keep static web assets off-device at first and only serve JSON from the ESP32.

---

## 9. Display strategy

Do not make the display the first milestone. The parallel RGB init requires CH422G, PSRAM framebuffer, and correct pin config — it is the most board-specific part and should be attempted after serial and sensor are working.

Recommended order:

1. Serial boot log over USB-CDC (`/dev/ttyACM0`).
2. Sensor read over serial.
3. Wi-Fi connect.
4. HTTP JSON endpoint.
5. Basic display text.
6. Better display layout.
7. Touch support (GT911 via I2C + IRQ).
8. Graphs on webpage, not necessarily on panel.

For the panel:

- Use `embedded-graphics` draw target backed by the parallel RGB framebuffer.
- Defer LVGL/Slint unless there is a proven working path for this board.
- The browser dashboard remains the rich UI.

PCLK note: 20 MHz is the only verified-stable pixel clock from the std reference. 24/30 MHz caused scrambling during WiFi load. Start at 20 MHz and do not raise it.

---

## 10. Memory strategy

### 10.1 Avoid heap-first design

Prefer fixed-capacity structures:

```rust
heapless::Vec<T, N>
heapless::String<N>
array-backed ring buffers
```

Use `alloc` only if required and justified.

### 10.2 History storage

A compact sample:

```rust
pub struct WeatherSample {
    pub timestamp_seconds: u64,
    pub temperature_c_x100: i16,
    pub humidity_x100: u16,
    pub pressure_pa: u32,
}
```

This avoids floats in storage while still allowing display conversion.

```text
16 bytes/sample * 1440 samples = ~23 KB
```

This is reasonable.

### 10.3 PSRAM

The 5B has 8 MB octal PSRAM. The parallel RGB double framebuffer (1024×600×2 bytes×2 = ~2.5 MB) must live in PSRAM.

Do not depend on PSRAM for app logic until the display is proven. The firmware should boot and serve sensor data without needing a framebuffer.

Enable PSRAM in sdkconfig/esp-hal config once the display phase begins.

---

## 11. Build/version strategy

Use current templates and examples, but pin versions once working.

Do not let AI agents randomly upgrade every crate while debugging.

Rules:

1. Start from `esp-generate --chip=esp32s3 weather-s3`.
2. Confirm it targets ESP32-S3.
3. Confirm Wi-Fi example builds for ESP32-S3.
4. Copy the working version set into the project.
5. Pin versions in `Cargo.toml`.
6. Only upgrade deliberately.

Baseline deps (as of the 3.5B nostd migration on 2026-04-11 — verify against current esp-generate output):

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

Add for Wi-Fi builds:

```toml
[profile.dev.package.esp-radio]
opt-level = 3
```

Reason: radio debug builds may need optimization to connect reliably.

---

## 12. Proof-of-life milestones

### Milestone 1: Toolchain boots hello world

Goal:

```text
Build, flash, and see serial output from ESP32-S3 over USB-CDC (/dev/ttyACM0).
```

Definition of done:

```text
Firmware boots repeatedly.
Serial logs are readable.
No panic loop.
```

---

### Milestone 2: CH422G + I2C scan

Goal:

```text
Initialize I2C on GPIO8/9.
Write CH422G to enable outputs.
I2C scan shows 0x24 (CH422G), 0x51 (PCF85063), 0x5D (GT911), and BME280 (0x76 or 0x77).
```

Definition of done:

```text
All expected I2C devices appear on the bus.
No bus lockup.
```

This milestone gates display and touch work — do not skip it.

---

### Milestone 3: Sensor-only firmware

Goal:

```text
Read BME280 over I2C and print values over serial.
```

Definition of done:

```text
Temperature/humidity/pressure print every 5–60 seconds.
Bad sensor wiring produces a clean error, not a crash.
```

---

### Milestone 4: `weather-core` extraction

Goal:

```text
Move app data model, history buffer, stats, and JSON shaping into weather-core.
```

Definition of done:

```text
weather-core compiles with #![no_std].
weather-desktop can use weather-core.
Unit tests pass on desktop.
```

---

### Milestone 5: Embassy scheduler skeleton

Goal:

```text
Run at least two Embassy tasks:
- heartbeat/log task
- fake sensor task
```

Definition of done:

```text
Fake samples update WeatherState on a timer.
No Wi-Fi yet.
No display yet.
```

---

### Milestone 6: Wi-Fi connect

Goal:

```text
Connect ESP32-S3 to local Wi-Fi.
```

Definition of done:

```text
Serial log shows connected state and IP address.
Reconnect logic handles AP restart or temporary failure.
```

---

### Milestone 7: First HTTP JSON endpoint

Goal:

```text
Serve GET /api/latest over local LAN.
```

Definition of done:

```text
curl http://DEVICE-IP/api/latest returns JSON.
JSON comes from WeatherState.
```

---

### Milestone 8: Real sensor + HTTP

Goal:

```text
Sensor task updates WeatherState.
Web task serves latest real sensor reading.
```

Definition of done:

```text
Values update without rebooting.
Endpoint remains responsive.
```

---

### Milestone 9: History endpoint

Goal:

```text
Serve history data for charting.
```

Definition of done:

```text
GET /api/history returns compact JSON array or object.
Desktop chart code can consume it.
```

---

### Milestone 10: Basic display

Goal:

```text
Initialize CH422G outputs, reset LCD, enable backlight.
Initialize parallel RGB panel via esp-hal lcd_cam at 20 MHz PCLK.
Allocate double framebuffer in PSRAM.
Show latest reading and status on panel.
```

Definition of done:

```text
Display shows temperature/humidity/pressure.
Shows Wi-Fi status.
Shows stale sensor state if readings stop.
No pixel scrambling.
```

---

### Milestone 11: Touch support

Goal:

```text
Read GT911 touch events via I2C + GPIO4 IRQ.
Map tap to view change.
```

Definition of done:

```text
Tapping the screen changes the active view.
No touch event causes a crash.
```

---

### Milestone 12: Dashboard integration

Goal:

```text
Use existing web/dashboard design against ESP32 JSON endpoints.
```

Definition of done:

```text
2-hour and 24-hour charts load from ESP32 data.
Browser refresh shows recent history.
```

---

## 13. Suggested first commands

Run from `/data/rust/waveshare-esp32-s3-touch-lcd-5b-nostd`.

Install tooling (fish shell):

```bash
cargo install esp-generate
rustup toolchain install esp
```

Generate the firmware crate:

```bash
mkdir -p crates
cd crates
esp-generate --chip=esp32s3 weather-s3
```

Verify build:

```bash
cd weather-s3
cargo +esp build -Zbuild-std=core,alloc
```

Flash (USB-CDC, confirm device is `/dev/ttyACM0`):

```bash
espflash flash --monitor target/xtensa-esp32s3-none-elf/debug/weather-s3
```

Do not assume old command names if the template generated different instructions.

---

## 14. Agent instructions

When an AI coding agent works on this project, it must follow these rules.

### 14.1 Do not do these things

Do not:

- rewrite the whole app in one pass,
- introduce a large GUI framework before Wi-Fi works,
- start with HTTPS/TLS,
- use dynamic allocation everywhere,
- remove the desktop/std version at `/data/rust/waveshare-esp32-s3-touch-lcd-5b`,
- mix ESP-IDF `std` and `esp-hal` `no_std` approaches in the same firmware unless explicitly requested,
- randomly upgrade crate versions while debugging,
- invent board pin mappings — use the verified pin list in section 1,
- skip CH422G init and wonder why the display is dark,
- assume PCLK above 20 MHz is safe,
- claim Wi-Fi is stable until it has run for hours.

### 14.2 Required implementation style

Do:

- make small commits,
- keep hardware glue isolated in `src/board/`,
- preserve the existing app behavior from the std reference,
- keep `weather-core` platform-neutral,
- add tests for ring buffer/history/stats,
- prove each milestone before moving on,
- document exact crate versions that work,
- log memory use and stack issues if possible,
- keep commands copy-pasteable and fish-compatible.

### 14.3 Output formatting for CLI agents (fish shell)

When producing commands:

- Use fenced code blocks.
- No line numbers.
- No placeholder files unless explicitly marked TODO.
- Prefer one clean command block at a time.
- State which directory the command should be run from.
- No bash-specific syntax: no `[[ ]]`, no `(( ))`, no `export VAR=value`, no `VAR=value cmd` prefix.
- Fish equivalents: `set VAR value`, `set -x VAR value`, `env VAR=value cmd`.
- Avoid `&&` / `||` chaining — use separate lines or a script.

---

## 15. Initial `weather-core` sketch

This is a sketch, not final code.

```rust
#![no_std]

pub mod sample;
pub mod history;
pub mod stats;
pub mod state;
pub mod json;
```

Sample model:

```rust
#[derive(Clone, Copy, Debug, PartialEq, Eq)]
pub struct WeatherSample {
    pub timestamp_seconds: u64,
    pub temperature_c_x100: i16,
    pub humidity_x100: u16,
    pub pressure_pa: u32,
}

impl WeatherSample {
    pub fn temperature_f_x100(&self) -> i16 {
        ((self.temperature_c_x100 as i32 * 9 / 5) + 3200) as i16
    }
}
```

Status model:

```rust
#[derive(Clone, Copy, Debug, PartialEq, Eq)]
pub enum WifiStatus {
    Unknown,
    Connecting,
    Connected,
    Disconnected,
}

#[derive(Clone, Copy, Debug, PartialEq, Eq)]
pub enum SensorStatus {
    Unknown,
    Ok,
    Missing,
    Stale,
    Error,
}

#[derive(Clone, Copy, Debug, PartialEq, Eq)]
pub struct SystemStatus {
    pub wifi: WifiStatus,
    pub sensor: SensorStatus,
    pub uptime_seconds: u64,
}
```

State model:

```rust
pub struct WeatherState<const N: usize> {
    pub latest: Option<WeatherSample>,
    pub history: HistoryBuffer<N>,
    pub status: SystemStatus,
}
```

---

## 16. Testing plan

### 16.1 Desktop tests

Test in `weather-core`:

- ring buffer wraparound,
- sample ingest,
- min/max/average,
- empty history behavior,
- stale sample detection,
- JSON formatting if done in core.

Run from project root:

```bash
cargo test -p weather-core
```

### 16.2 Firmware smoke tests

For each firmware build, test:

```text
Boots from power-off.
Boots after reset.
Serial output starts cleanly on /dev/ttyACM0.
No panic loop.
Sensor read works.
Wi-Fi connects.
Endpoint responds from another LAN machine.
Runs for at least 30 minutes.
Then runs overnight before calling it stable.
```

### 16.3 Network tests

From any LAN machine (or the G760 host):

```bash
curl -v http://DEVICE-IP/api/status
curl -v http://DEVICE-IP/api/latest
curl -v http://DEVICE-IP/api/history
```

Optional repeated test:

```bash
while true
  date
  curl -fsS http://DEVICE-IP/api/latest; or echo FAILED
  sleep 10
end
```

---

## 17. Practical port order

Recommended order for actual coding:

```text
1. Read reference std app at /data/rust/waveshare-esp32-s3-touch-lcd-5b.
2. Identify existing data structs and JSON format.
3. Create workspace at /data/rust/waveshare-esp32-s3-touch-lcd-5b-nostd.
4. Generate weather-s3 with esp-generate.
5. Extract weather-core.
6. Add desktop tests.
7. Prove serial boot (Milestone 1).
8. Prove CH422G + I2C scan (Milestone 2).
9. Prove sensor read (Milestone 3).
10. Prove fake Embassy tasks (Milestone 5).
11. Prove Wi-Fi connect (Milestone 6).
12. Prove /api/status (Milestone 7 partial).
13. Prove /api/latest with fake data.
14. Connect real sensor to WeatherState (Milestone 8).
15. Add /api/history (Milestone 9).
16. Initialize parallel RGB display (Milestone 10).
17. Add touch (Milestone 11).
18. Integrate browser chart dashboard (Milestone 12).
19. Tune memory/performance.
20. Run overnight stability test.
```

---

## 18. Success criteria

The port is successful when:

```text
ESP32-S3 boots reliably.
Wi-Fi connects and reconnects.
Sensor readings are collected on schedule.
Latest and history endpoints work over LAN.
Display shows useful live status at 20 MHz PCLK without scrambling.
Touch changes views.
The browser dashboard can graph 2-hour and 24-hour data.
The firmware runs overnight without panic/reboot.
The existing std/desktop version still works as a simulator/test harness.
```

---

## 19. Decision gate

Start the port now, but treat the first week as proof-of-life, not a full rewrite.

The decision gate is simple:

```text
If ESP32-S3 + no_std + Embassy + Wi-Fi + one JSON endpoint works,
then continue the port.

If that fails due to ecosystem issues,
fall back to ESP-IDF std for networking while preserving weather-core.
```

This keeps the project safe either way.
