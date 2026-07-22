# TODO / Roadmap

## ACTIVE TASKS 🔄

### FUTURE PROTOCOLS & FEATURES 🔵

#### Web Interface Improvements

- [ ] **Show connected clients info in AP mode**
  - Number of connected stations
  - List of client IP addresses
  - IP of current web interface user

- [x] **Terminal protocol optimization** (v2.19.0)
  - [x] `terminal_parser` — raw with 5ms flush timeout, taps data to ring buffer (PSRAM 32KB / internal 4KB)
  - [x] WebSocket `/ws/terminal` with history replay on connect
  - [x] Web UI: collapsible terminal window, save/copy/clear buttons, fullscreen mode
  - [x] ANSI rendering via xterm.js + fit addon (dynamic toggle, lazy load)
  - [x] Web input: keyboard toggle button → xterm onData → WebSocket → UART TX
  - [ ] Phase 3: dynamic UDP connect button near terminal window (non-persistent, applies on the fly)
  - [ ] Phase 4: configurable pattern matching → status badges (e.g. `login:` → Ready)
  - [x] Phase 5: **Hex view mode** (v2.20.1) — third toggle (`0x` button) alongside ANSI on/off, hexdump-style output for diagnosing unknown/misparsed protocols

- [ ] **Frontend JS libraries upgrade** — deferred until current backend stability is confirmed (post-lib-upgrade v2.20.2+)
  - **Alpine.js + plugins**: `3.15.8 → 3.15.12` (all three: `alpinejs`, `@alpinejs/persist`, `@alpinejs/collapse` — must upgrade together, plugins have to match core version)
    - Only patch fixes, no breaking changes: `x-for` crashes on `Object.groupBy()` results, `@teleport` memory leaks (we don't use), `x-model` bugs, CSP compat, x-anchor cleanups
    - Update `src/webui_src/lib/versions.json` alongside the .min.js files
    - Low risk — patch bump only
  - **xterm.js + fit addon**: `5.5.0 → 6.x` — **wait for 6.0.1 release** before upgrading (see below)
    - **6.0.0 released 2025-12-22**; 7+ months of activity in master (~30 commits) but no 6.0.1 tag yet
    - Important fixes already in master, NOT in 6.0.0:
      - **UTF-8 continuation drop** (#6003/#6004) — codepoint lost when `write(Uint8Array)` splits a UTF-8 sequence at 0x80 boundary. Relevant to us: we push WS binary → `TextDecoder('utf-8', {fatal:false})` → potential silent byte loss
      - **WriteBuffer setTimeout on dispose** (#5936) — memory leak fix. Relevant: we dispose/reinit xterm on every ANSI toggle
      - Selection drag-scroll clamp (#5993), clear at cursor home (#5992)
    - Fixes NOT relevant to us (WebGL renderer, clipboard addon, image addon) — we use default DOM renderer, none of these addons
    - Open bugs mostly non-relevant (Safari-specific, WebGL, image addon)
    - **Wins after 6.0.1**: memory improvements at dispose (main reason to upgrade), synchronized output DEC 2026, UTF-8 correctness
    - **Breaking changes to review** (already present in 6.0.0):
      - Removed alt-to-ctrl+arrow mapping (may affect users relying on Alt+arrow in shell)
      - Viewport/scrollbar reorg — visually retest scroll, fullscreen, fit resize
      - Requires simultaneous upgrade of `xterm-addon-fit` 0.10.0 → matching version
    - **Retest checklist** on xterm upgrade:
      - ANSI escape rendering (colors, cursor positioning, CSI sequences)
      - Input mode keyboard: printable, arrows, Ctrl+letters, Alt+letters, F-keys, Escape, Tab
      - `attachCustomKeyEventHandler` still fires with same event shape
      - `_xterm.selectAll()` / `getSelection()` / `write(text)` API unchanged
      - fit addon still resizes on fullscreen toggle
  - **Order**: Alpine patch bump can go early — as soon as current backend upgrade soaks for a few hours (safe patches only, no breaking changes). xterm major bump waits for longer soak after Alpine lands. Do NOT bundle with backend library upgrade — if something breaks, blame stays clear

- [ ] **Binary recording (sniffer mode)** — record raw UART stream into a `.bin` file for offline analysis (unknown protocols, reverse engineering, regression samples). Complement to hex view.
  - **Available only in Terminal protocol optimization mode** — that's the only mode where `/ws/terminal` is active. In other modes (Raw / MAVLink / SBUS / CRSF) parsers don't tap into TerminalBuffer, WS is silent or absent. Record button must be shown/enabled only when Terminal mode is selected AND WS is connected.
  - **Future extension** (out of scope for v1): tap the `raw_parser` into TerminalBuffer too, so users can pick Raw mode for generic sniffing of any protocol. Requires ESP-side change (PSRAM buffer allocation + parser tap).
  - **Architecture: client-side only.** No changes on ESP. Piggyback on the existing `/ws/terminal` binary stream.
  - **Event-driven, not timer-driven** — `ws.onmessage` handler pushes `Uint8Array` chunks into an array. WebSocket message events are NOT subject to background-tab throttling (unlike `setInterval`), so a hidden/minimised tab keeps recording without data loss.
  - **Flow**:
    1. Record button toggles `recording` flag
    2. While `recording`: every incoming WS binary message → `recChunks.push(new Uint8Array(await evt.data.arrayBuffer()))`
    3. Stop button → `new Blob(recChunks, {type: 'application/octet-stream'})` → `URL.createObjectURL` → invisible `<a download>` click → `setTimeout(1500, revokeObjectURL)` for cleanup
    4. Filename: `terminal_${new Date().toISOString().replace(/[:.]/g, '-')}.bin`
  - **UI**: single button next to hex view (`● REC` / `■ STOP`), small time display `MM:SS` next to it. Button disabled when WS not connected. Timer `setInterval(1s)` for the display IS throttled in background — that's fine, it's cosmetic, data collection continues via WS events.
  - **Format**: pure raw bytes, 1:1 with UART. No timestamps, no PCAP. User picks their own analysis tool (hex editor, Python, Wireshark with custom dissector).
  - **Auto-stop safety limits** — user may forget the recording is on; browser RAM fills up over time. Whichever limit trips first stops recording AND auto-saves the file (never silently drop data):
    - **Size cap: 50 MB hard stop** (soft warning at 25 MB — badge turns orange). UART throughput varies (9600 → 1 KB/s, 921600 → 100 KB/s), so size is the meaningful bound
    - **Time cap: 1 hour hard stop** — trickle-rate protection for slow UART where 50 MB is never reached
    - On auto-stop: show notification "Recording auto-stopped at <limit>", file saved automatically with same filename convention
  - **Edge cases to handle**:
    - WS reconnect mid-recording — history replay from ESP ring buffer will re-inject last 32KB, causing duplicated bytes in the output. Either accept as known limitation or set a `justReconnected` flag to skip first replay burst
    - Clear button pressed while recording — ask "stop recording first?" (Clear doesn't affect recording state, but user may expect it to)
    - Tab close mid-recording — data lost. Add `beforeunload` handler when recording is active to warn user
  - **Works on all boards** (no PSRAM requirement — all client-side).
  - Estimate: ~1 hour, ~30-50 lines of JS in `app.js` + one button + one span in `index.html`

- [ ] **pioarduino platform upgrade** — deferred, unpin only after both boards test clean (separate step, AFTER Alpine + xterm are stable)
  - Currently frozen at `55.03.35` (Arduino 3.3.5 / ESP-IDF 5.5.1) — see "Known Issues & Warnings" section for the original BLE BTDM regression that caused the pin
  - Target: `55.03.38-1` (Arduino 3.3.8 / ESP-IDF 5.5.4) — ESP-IDF 5.5.3 shipped fixes matching our symptoms (`Fixed crash in btdm_controller_task on ESP32`, `Fixed scan HCI command timeout issue on ESP32`), 5.5.4 closes a NimBLE host regression from 5.5.3
  - **Migration approach**: use conditional compilation on `ESP_ARDUINO_VERSION` so the code compiles cleanly on BOTH the old (3.3.5) and the new (3.3.8) platform. Lets us flip `platformio.ini` between versions for A/B testing without editing source. Remove the `#ifdef` guards after 2-3 months of stable operation on 3.3.8 (e.g. v2.20.2 → v2.20.5).
    ```cpp
    #include "esp_arduino_version.h"
    #if ESP_ARDUINO_VERSION >= ESP_ARDUINO_VERSION_VAL(3, 3, 7)
      // new API path
    #else
      // old API path (3.3.5/3.3.6)
    #endif
    ```
  - **Migration steps** (all work happens on a dedicated branch, e.g. `pioarduino-3.3.8`, never on `main`):
    1. Branch off `main`, bump platform URL to `55.03.38-1`
    2. Wrap `esp32-hal-bt-mem.h` include in `bluetooth_ble.cpp:11-15` in an `#if ESP_ARDUINO_VERSION >= ESP_ARDUINO_VERSION_VAL(3,3,7)` block (S3 only). MiniKit already handled via existing `btInUse()` override, no change there
    3. **Try full clean build** — the compiler will surface any other API breaking changes between Arduino 3.3.5 → 3.3.8 or ESP-IDF 5.5.1 → 5.5.4. Wrap each in `ESP_ARDUINO_VERSION` guards too. Don't speculate ahead — react only to what actually breaks
    4. Delete cached `sdkconfig.{env}` files if sdkconfig keys changed (see MEMORY note about sdkconfig cache gotcha)
    5. **Quick sanity on both boards on the branch**:
       - MiniKit: BLE NUS pairing + data transfer (the original regression that caused the pin)
       - Zero (S3): regression check — BLE, WiFi, USB Host, UART2 still working
    6. If sanity passes → cut a **beta release from the branch** (`v2.X.Y-beta1`, tag from the branch not main) so a few users can soak it in real conditions. Beta scope in CHANGELOG must be limited to the pioarduino bump — no other unrelated changes tag along
    7. Only after beta soaks clean for the required window → merge branch into `main`, promote to stable release, remove the Known Issue note, drop the pin. Guards stay in code for 2-3 months of soak, then get removed in a separate cleanup commit
  - **Do this LAST** in the sequence: backend libs (done) → Alpine → xterm → pioarduino. Each is a separate release cycle with soak time — never bundle

#### Network Improvements

- [ ] **DNS hostname support in UDP target**
  - Currently only IP address accepted, add `WiFi.hostByName()` resolve
  - Cache resolved IP with TTL to avoid per-packet DNS lookup

- [ ] **WiFi network scan button in Client mode** — needs careful implementation
  - Goal: add "Scan" button next to SSID fields, `esp_wifi_scan_start()` → list of visible networks with RSSI/security → click to auto-fill an SSID slot. Avoids silent auth-failures from typos.
  - **First attempt in v2.20.1 was reverted**: blocking `esp_wifi_scan_start(&cfg, true)` from an HTTP handler (running in AsyncTCP task context) caused heap corruption that surfaced as a `LoadProhibited` crash in the mDNS service task on the next outgoing mDNS packet (`mdns_priv_if_write → pbuf_alloc → malloc → tlsf block_next → invalid pointer`). Concurrent WiFi scan + AP+STA + active mDNS is known-fragile on ESP-IDF.
  - Proper implementation needs:
    - Non-blocking `esp_wifi_scan_start(&cfg, false)` + event handler for `WIFI_EVENT_SCAN_DONE`, results cached in a static buffer with a mutex
    - REST split: `POST /api/wifi/scan` starts, `GET /api/wifi/scan` returns `{status, results}` with client-side polling
    - Coordination with existing scan loop in `wifi_manager` (it runs its own scan in Client mode — must not step on each other)
    - Consider pausing mDNS (`mdns_free()` + `mdns_init()` around scan) or scheduling scan on core 1 via `xTaskCreatePinnedToCore` to reduce cross-subsystem races

#### Advanced Protocol Management

- [ ] **R2D2 RC text format support** — wire-compatibility with EdgeTX LUA scripts (Apachi Team)

  **Goal**: ESP can pretend to be an EdgeTX radio running the R2D2 LUA script, so any R2D2-aware receiver (RC-Connector, MP plugin, future tools) accepts the ESP stream byte-for-byte.

  **Current state**: ESP only emits ESP-Bridge format `RC val,val,...\n` (16ch, µs 1000-2000) in two places:
  - `sbus_text.h::sbusFrameToText` — SBUS → text path
  - `crsf_parser.h::formatRcChannels` — CRSF → text path

  Used by 8 text-output roles (D2/D3/D4 SBUS_OUT TEXT, D2_USB_SBUS_TEXT, D5_BT_SBUS_TEXT, D2_USB_CRSF_TEXT, D4_CRSF_TEXT, D5_BT_CRSF_TEXT). Input parsing for "RC ..." doesn't exist — text format is output-only.

  **R2D2 wire format** (from `r2d2-usb-vcp.lua`): `$val,val,...,val,\r\n` — 24 fields, raw -1024..+1024, **trailing comma after last value**, conversion `raw = (us - 1500) * 2`. We send all 24 fields (first 16 = real data, last 8 = 0) to match LUA byte-for-byte — guards against any strict R2D2 parser. Overhead is ~16 bytes/frame, negligible.

  **Architecture** — single global setting (no per-device mix needed):
  - New `Config.rcTextFormat: 0=ESP_BRIDGE, 1=R2D2` (extensible enum, not bool)
  - Existing per-device `sbusOutputFormat` (BINARY/TEXT/MAVLINK) stays untouched — orthogonal axis: per-device picks WHAT to emit, global picks HOW the text is shaped
  - 5 hardcoded text-only roles (USB/BT) need no config changes — they just read the global flag

  **Implementation outline**:
  - [ ] New `rc_text_format.h` with two formatters over `us[16]`:
    - `formatEspBridge(us[16], buf)` → "RC ..." (current behavior)
    - `formatR2D2(us[16], buf)` → "$..." (24 fields, last 8 = 0, trailing comma)
  - [ ] Switch in `sbus_text.h::sbusFrameToText` and `crsf_parser.h::formatRcChannels` based on global flag
  - [ ] `Config.rcTextFormat` field + load/save/migration in `config.cpp` (default ESP_BRIDGE)
  - [ ] Web UI: single dropdown "RC Text Format: ESP-Bridge | R2D2" near Protocol Optimization
  - [ ] `help.html` — describe both formats and when to pick each

  **Buffer sizing**: R2D2 line is longer than ESP-Bridge (`$` + 24 fields × up to 6 chars + 24 commas + `\r\n` ≈ 170-180 bytes). `CRSF_TEXT_BUFFER_SIZE = 200` already fits; `SBUS_TEXT_BUFFER_SIZE = 101` (in `sbus_text.h:12`) needs to grow to ~200 (or pick max of both formats).

  **Rate limiting**: existing `outRate` / `udpSendRate` / `btSendRate` (10-70 Hz, decimation) applies to R2D2 unchanged.

  **Synergy with MP plugin v2.3.0**: plugin already auto-detects `$` prefix on input — after this task ESP can talk to MP plugin in either format. Same for RC-Connector (auto-detects via `RcParser.cs`).

- [ ] **Protocol Conversion Features**

  - [ ] **SBUS↔UART Bridge (ESP↔ESP transport)**

    **Use case**: Two ESP devices create a wired/wireless bridge for SBUS transport
    ```
    [SBUS Receiver] → ESP1 (SBUS IN → UART/UDP OUT) ~~~~ ESP2 (UART/UDP IN → SBUS OUT) → [FC]
    ```

    **UDP path — already implemented:**
    - ESP1: D1_SBUS_IN → D4_SBUS_UDP_TX (binary 25 bytes over UDP)
    - ESP2: D4_SBUS_UDP_RX → D3_SBUS_OUT (binary 25 bytes to FC)

    **UART path — implementation needed:**

    - [ ] **Output side (SBUS → UART)** — 80% ready
      - Infrastructure exists: UART 115200 8N1 init (from sbusTextFormat)
      - Need: "Raw binary" mode — passthrough 25 bytes without text conversion
      - Change: Add `sbusRawFormat` flag or modify sbusTextFormat logic
      - In sendDirect(): if raw mode, skip sbusFrameToText(), send binary directly

    - [ ] **Input side (UART → SBUS)** — needs new code
      - Need: Parse SBUS frames from standard UART stream (115200 8N1)
      - Reuse: `SbusParser` already exists for frame detection
      - New role: D2_UART_SBUS_IN or D3_UART_SBUS_IN
      - Flow: UART RX → SbusParser → validate → SbusRouter::feedFrame()
      - Challenge: No inter-frame gaps on standard UART, rely on 0x0F/0x00 markers

    **Implementation steps:**
    1. Add raw binary output mode (skip text conversion, keep 115200 8N1)
    2. Create new roles: D2_UART_SBUS_IN, D3_UART_SBUS_IN
    3. Connect SbusParser to UART input stream
    4. Register parsed frames with SbusRouter as new source
    5. Add web UI options for UART-SBUS bridge configuration

    **Existing code to reuse:**
    - `SbusParser` (protocols/sbus_parser.h) — frame detection
    - `SbusRouter` — source registration and routing
    - `initDevice*SBUS()` — UART configuration with 115200 mode
    - `sendDirect()` — output path in senders

    **ESP as hardware converter replacement:**
    ```
    Scenario A: ESP replaces SBUS→UART converter
    [SBUS Receiver] → ESP (SBUS IN 100000 8E2 INV) → UART OUT (115200 8N1) → [device]
    Role: D1_SBUS_IN + D2_UART_SBUS_OUT (raw binary, no text conversion)

    Scenario B: ESP replaces UART→SBUS converter
    [device] → UART (115200 8N1) → ESP (UART_SBUS_IN) → SBUS OUT (100000 8E2 INV) → [FC]
    Role: D2_UART_SBUS_IN + D3_SBUS_OUT

    Scenario C: ESP paired with external hardware converter
    [SBUS Receiver] → [HW SBUS→UART] → ESP (UART IN 115200) → SBUS OUT (100000 8E2) → [FC]

    Scenario D: Full ESP↔ESP wireless bridge
    [SBUS RX] → ESP1 (SBUS IN → UDP TX) ~~~WiFi~~~ ESP2 (UDP RX → SBUS OUT) → [FC]

    Scenario E: ESP↔ESP wired bridge (long cable)
    [SBUS RX] → ESP1 (SBUS IN → UART OUT 115200) ~~~wire~~~ ESP2 (UART IN → SBUS OUT) → [FC]
    ```
    - ESP can fully replace both SBUS→UART and UART→SBUS hardware converters
    - Compatible with existing hardware converters (direct wiring)
    - socat and ser2net for network transport
    - Any device that can send/receive 25-byte SBUS frames over standard UART

  - [ ] Configurable send rate for binary SBUS output (D2_SBUS_OUT, D3_SBUS_OUT, D4_SBUS_UDP_TX binary)
    - Currently binary SBUS outputs at native 70 Hz (14ms frame timing)
    - Some FC or receivers may benefit from throttled rate (50 Hz typical)
    - D4 binary (format=0) also needs rate control — currently only text format (format=1) has it
    - Reuse existing rate selector UI (10-70 Hz)

### Pre-release Testing

- [ ] **MAVLink RC Monitor** — test RC channel bars with real MAVLink traffic (msg 65 RC_CHANNELS, msg 70 RC_CHANNELS_OVERRIDE)
- [ ] **Terminal protocol** — test WebSocket terminal with UART device (e.g. RPi console)

### Future Considerations (Low Priority)

#### Platform Testing (waiting for hardware)

**Super Mini** — implemented, basic tested (WiFi, web, LEDs, UDP). Needs: UART devices, SBUS inverter, BLE testing.

**XIAO ESP32-S3** — implemented, basic tested (D1 UART + D2 USB with FC/MP). Needs: D3, RTS/CTS, SBUS, external antenna range, BLE testing.

- [ ] **Test CRSF input from TX16S JR bay** — ESP in TX module case, D1_CRSF_IN reads channels from radio, text output via BLE/UDP. Radio sends CRSF regardless of receiver link status.

#### BLE Remaining Tasks

  - [ ] **LOW**: Check encryption before sending notifications (`desc.sec_state.encrypted`)

  **Reference**: BLE GATT SBUS (NUS + Just Works pairing + bond persistence + WiFi coexistence) — implemented.
  Heap: S3 ~60KB free (min ~38KB), MiniKit ~42KB free (min ~16KB). WiFi + BLE + UDP + WebServer work simultaneously.

#### MP Plugin Improvements

  - [ ] **MEDIUM**: MAVLink telemetry via BLE NUS (MP plugin)

    **Use case**: ESP connected to drone via UART (MAVLink), sends telemetry via BLE to laptop.
    Laptop keeps WiFi for internet (maps, remote access), BLE works in parallel.
    Alternative: DroneBridge and similar use external BLE→vCOM bridge — plugin is cleaner.
    ```
    [Drone] →UART→ [ESP32-S3 BLE] →BLE NUS→ [Laptop: MP plugin]
                                               ↕ WiFi (internet, maps)
    ```

    **Approach**: WinRT `ICommsSerial` in plugin, integration into standard MP dropdown

    **MP API (verified, public, accessible from plugin):**
    - `MissionPlanner.Comms.SerialPort.GetCustomPorts` — `public static Func<List<string>>`
      - Plugin adds BLE devices to port list (appear as "BLE_ESP-Bridge_XXXX")
    - `Host.MainForm.CustomPortList` — `public Dictionary<Regex, Func<string, string, ICommsSerial>>`
      - Plugin registers factory: regex `BLE_.*` → creates `BleNusSerial`
    - User selects BLE port in standard dropdown, clicks Connect — MP works as usual

    **Implementation:**
    - [ ] Class `BleNusSerial : Stream, ICommsSerial` (~200-300 lines)
      - WinRT BLE API (Windows.Devices.Bluetooth) — no external DLLs
      - MemoryStream buffer for incoming data (notification callback)
      - Read() with timeout (deadline pattern, like CommsBLE/UdpSerial)
      - Write() via NUS RX characteristic
      - Open()/Close() — BLE connect/disconnect
      - Thread-safe buffer (lock, notification callback vs MP read thread)
    - [ ] Registration in Init(): GetCustomPorts += GetBleNusPorts, CustomPortList.Add(...)
    - [ ] Device discovery: reuse GetPairedNusDevicesAsync() from current plugin
    - [ ] Handle disconnect (IsOpen=false, exception from Read)

    **Throughput:** BLE NUS MTU 256 → ~15-20 KB/s. MAVLink 57600 baud ≈ 5.7 KB/s — 3x headroom.

    **ESP side:** no changes needed — BLE NUS already implemented, MAVLink data transparent for NUS.

    **Reference**: MP has experimental CommsBLE (since 2020, SimpleBLE, not in releases, BUSL-1.1 license).
    Our approach — WinRT, no license restrictions, no external DLLs.
    MP API: MainV2.cs:522 (CustomPortList), CommsSerialPort.cs:312 (GetCustomPorts).

  - [ ] **LOW**: Auto-reset ESP from bootloader mode (if manual RST checkbox isn't enough)
    - When COM connected but no data for >3s, try RTS toggle (up to 3 attempts with 3s intervals)
    - Reset attempt counter on first valid data line
    - Only when RST checkbox is enabled
    - Complements existing manual RST-on-connect feature (v2.2.5+)

#### UART Improvements 🔵 LOW PRIORITY

- [ ] **Hardware Flow Control CTS/RTS detection**
  - Currently: if flow control enabled in config but CTS/RTS not physically connected, TX blocks waiting for CTS signal
  - Other devices often auto-detect and fall back to no flow control
  - Idea: check CTS pin state before enabling hardware flow control
    - Configure CTS as input with pull-up
    - If CTS stays HIGH (pull-up wins) → likely not connected → disable flow control
    - If CTS is LOW → assume device connected and holding CTS
  - Not 100% reliable (CTS LOW could mean "not connected but grounded" or "connected, wait")
  - Low priority: UI checkbox now works correctly, users can disable manually

#### Memory Optimization (MiniKit) 🔵 LOW PRIORITY

  Current situation: ~35KB free heap with WiFi+BT+WebServer enabled

  - [ ] **Scenario A: WiFi only (D5_NONE)**
    - Release BT memory: `esp_bt_controller_mem_release(ESP_BT_MODE_CLASSIC_BT)` — frees ~30KB
    - Expected heap: ~65KB — sufficient for stable WiFi+UDP+WebServer
    - Use case: network telemetry without Bluetooth

  - [ ] **Scenario B: BT only (no network)**
    - Release WiFi memory: `esp_wifi_deinit()` + release — frees ~40KB
    - Expected heap: ~75KB — sufficient for stable BT operation
    - Use case: standalone BT serial adapter (SBUS text to phone)
    - WebServer only on quick reset for configuration

  - [ ] **Scenario C: Headless WiFi+UDP+BT**
    - Disable AsyncWebServer in normal operation — saves ~15-30KB runtime heap
    - Keep WiFi for UDP bridge, enable BT simultaneously
    - Expected heap: ~50-65KB
    - WebServer only on quick reset (for configuration)
    - mDNS still needed for quick reset discovery
    - Use case: full bridge functionality (UART↔UDP + UART↔BT)

  Implementation notes:
  - Memory release is one-way (cannot re-enable without reboot) — fits "save = reboot" model
  - Prefer runtime selection based on config (D5 role, D4 role) — avoid separate builds
  - Separate build configs only as last resort
  - Map file analysis: ESPAsyncWebServer ~109KB flash, web_api+web_interface ~97KB flash (static RAM minimal ~65 bytes)

  **Heap reference (MiniKit v2.18.12):**
  - WiFi+BT+WebServer: ~35KB free (BT stack takes ~40KB static)
  - BT only (no WiFi): ~70KB free
  - WiFi only (no BT in sdkconfig): ~76KB free

#### New Protocol Support 🔵 LOW PRIORITY

- [~] **CRSF Protocol** - RC channels and telemetry for ELRS/Crossfire systems
  - Phase 1-3: ✅ DONE (text/binary/bidirectional outputs via USB, UART3, UDP, BLE)
  - [ ] Test bidirectional CRSF with flight controller (Phase 3, telemetry back to ELRS RX)
  - [ ] Per-device filter UI (Phase 4, backend ready — UI currently sets all devices at once)
  - [ ] Telemetry extraction (Phase 4): structured data for web UI (battery, GPS, attitude)
  - [ ] VTX control via MSP-over-CRSF (Phase 5, requires bidirectional wiring)
  - Alternative: direct VTX protocols (SmartAudio by TBS, IRC Tramp by ImmersionRC) — low priority
  - **Reference**: [ExpressLRS crsf_protocol.h](https://github.com/ExpressLRS/ExpressLRS/blob/master/src/lib/CrsfProtocol/crsf_protocol.h), [CRSF wiki](https://github.com/crsf-wg/crsf/wiki), [TBS spec](https://github.com/tbs-fpv/tbs-crsf-spec)
- [ ] **MSP Protocol** (Betaflight/iNav) 🟡 MEDIUM PRIORITY
  - MSPv1 (`$M`) + MSPv2 (`$X`) frame parsing, CRC validation
  - Passive telemetry extraction from FC responses (no polling needed when GCS is connected)
  - Use cases beyond raw bridge:
    - RC Channel Monitor in web UI (MSP_RC msg 105, same as SBUS/CRSF/MAVLink)
    - Telemetry text output to USB/UDP/BLE (battery, GPS, attitude — like CRSF Text)
    - Arming state detection for LED indication
    - FC status in web UI (flight mode, CPU load, sensor flags)
    - VTX control (band/channel/power) — also available via MSP-over-CRSF (Phase 5)
  - Request-response protocol — FC doesn't push data, needs GCS polling or own polling timer
  - Implementation scope (MAVLink parity analysis):
    1. `MspParser` — MSPv1 + MSPv2 from scratch (two frame formats, two CRC algorithms)
    2. Protocol optimization mode — `MSP = 4` in enum, pipeline integration, web UI
    3. RC channels — MSP_RC (msg 105) → shared `RcChannelData`
    4. Statistics — valid/invalid frames, CRC errors, last activity in web UI
    5. No routing needed — MSP has no addressing (unlike MAVLink sysid/compid)
    6. Text telemetry output — optional, lower priority
  - Reference: [Betaflight MSP](https://betaflight.com/docs/development/API/MSP), [iNav MSP](https://github.com/iNavFlight/inav/wiki/MSP-V2)

#### System Reliability & Memory Management 🔵 LOW PRIORITY

- [ ] **Periodic Reboot System**
  - Configurable interval (12h, 24h, 48h, never)
  - Graceful shutdown with connection cleanup
  - Pre-reboot statistics logging

- [ ] **Emergency Memory Protection**
  - Critical threshold: 20KB (reboot after sustained low memory)
  - Warning threshold: 50KB (log warnings)
  - Heap fragmentation tracking

## Known Issues & Warnings

ℹ️ **`zero_ble_debug` does not fit on 4MB Zero (ESP32-S3FH4R2)**
- BLE debug build is ~1645 KB, app slot in `partitions_custom_4mb.csv` is 1.5 MB (1572 KB) — exceeds by ~73 KB, link fails
- BLE production (~1469 KB) and non-BLE debug (~1416 KB) both fit; only BLE+debug combo overflows
- On N8R8 (8MB Zero) no issue — app slots in `partitions_custom_8mb.csv` are 2 MB
- Possible fix on 4MB: shrink LittleFS partition (384 KB → ~192 KB) and coredump (64 KB → 32 KB) to free ~224 KB total (~112 KB per app slot). Config fits in 192 KB easily — the real cost is **breaking change**: existing devices upgrading OTA would lose stored config and crashlogs because LittleFS/coredump partition offsets shift. Would require an export-then-import flow or in-firmware migration logic.
- Decision: accept the limitation. BLE debug on 4MB Zero is dev-only and rarely needed. Revisit if real use case appears.



⚠️ **pioarduino pinned to 3.3.5 (ESP-IDF 5.5.1)** — DO NOT upgrade until BLE fix confirmed
- pioarduino 3.3.6+ (ESP-IDF 5.5.2) breaks BLE on MiniKit (ESP32 WROOM)
- Symptom: BLE pairing succeeds but NUS characteristics inaccessible, no data transfer
- ESP32-S3 boards not affected — only WROOM BTDM controller
- ESP-IDF 5.5.2 changelog suspects: BTDM scheduling priority changes, NimBLE handle duplication fix, HCI status fix
- When upgrading: uncomment `esp32-hal-bt-mem.h` include in bluetooth_ble.cpp (needed for 3.3.7+), delete cached sdkconfig files, test BLE on MiniKit before release

### Likely fixed upstream — needs testing before unpinning

ESP-IDF 5.5.3 / 5.5.4 ship fixes that match our symptoms exactly:
- IDF 5.5.3 controller: "Fixed crash in btdm_controller_task on ESP32", "Fixed scan HCI command timeout issue on ESP32" — both target WROOM BTDM, the same area we suspected
- IDF 5.5.4: "Fixed potential NimBLE host connection loss in ESP-IDF v5.5.3 on ESP32 / ESP32C3 / ESP32S3" — closes regression introduced by 5.5.3

Available upgrade target: **pioarduino 55.03.38-1** (Arduino 3.3.8 + ESP-IDF 5.5.4, released 2026-04-13).

Current state works stably — no rush to upgrade. When time permits:
1. Bump platform URL to `55.03.38-1`
2. Uncomment `esp32-hal-bt-mem.h` include in `bluetooth_ble.cpp` (BLE memory mgmt became automatic in Arduino 3.3.7)
3. **Test on both boards before committing**:
   - **MiniKit**: BLE NUS pairing + data transfer (the original regression)
   - **Zero (S3)**: regression check — BLE, WiFi, USB Host, UART2 all still working
4. Only after both boards pass — commit, remove this Known Issue note, drop the pin

## Build & CI/CD

### Debug Toolchain Paths

```
# ESP32 (MiniKit) - xtensa-esp32
TOOLCHAIN: ~/.platformio/packages/toolchain-xtensa-esp-elf/bin/
addr2line: xtensa-esp32-elf-addr2line.exe -pfiaC -e .pio/build/minikit_debug/firmware.elf <addresses>
size:      xtensa-esp32-elf-size.exe .pio/build/minikit_production/firmware.elf
nm:        xtensa-esp32-elf-nm.exe .pio/build/minikit_production/firmware.elf

# ESP32-S3 (Zero, SuperMini, XIAO) - xtensa-esp32s3
addr2line: xtensa-esp32s3-elf-addr2line.exe -pfiaC -e .pio/build/zero_debug/firmware.elf <addresses>
size:      xtensa-esp32s3-elf-size.exe .pio/build/zero_production/firmware.elf
```

## Libraries and Dependencies

### Current Dependencies
```ini
lib_deps =
    bblanchon/ArduinoJson@^7.4.2      # JSON parsing and generation
    fastled/FastLED@^3.10.3           # WS2812 LED control
    arkhipenko/TaskScheduler@^4.0.3   # Task scheduling
    ESP32Async/ESPAsyncWebServer@3.10.0 # Async web server (6x faster since 3.9.0)
    ESP32Async/AsyncTCP@3.4.10         # TCP support (deferred close fix)

# BLE builds use ESP-IDF NimBLE component (configured in sdkconfig), no external lib
```


