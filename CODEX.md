# CODEX.md — QMX Panadapter

This is the development guide for Codex, VS Code, and human contributors working on `qmx-panadapter`.

## Project target

ESP-IDF firmware for the M5Stack Tab5 (ESP32-P4) implementing a standalone real-time panadapter for the QRP Labs QMX/QMX+ transceiver.

The intended USB topology is:

- Tab5 acts as the USB host.
- QMX/QMX+ is the attached composite USB device.
- CAT control is USB CDC-ACM.
- I/Q audio is USB Audio Class (UAC).
- Display output is local LVGL spectrum/waterfall plus a browser view over Wi-Fi.

Do not treat the Android/Windows behavior of a cable as proof that the Tab5 port/host/VBUS path is correct. Those systems are known-good hosts; this firmware still needs to prove host-mode enumeration on Tab5.

## Current blocker: QMX+ does not enumerate/connect on Tab5

Prioritize USB physical/host enumeration before DSP, CAT parsing, waterfall, or UI polish.

Observed user facts:

- The same QMX+ and data cables work with qFT8 on Android and WSJT-X on Windows.
- The Tab5 firmware does not connect through either USB-C to USB-C or USB-C to USB-A cabling.
- The code does not currently log enough information to distinguish: wrong Tab5 connector, missing VBUS, no host-mode enumeration, VID/PID mismatch, CDC interface mismatch, or UAC alternate-setting mismatch.

### What the code currently does

`main/main.c` calls:

```c
ESP_ERROR_CHECK(bsp_usb_host_start(BSP_USB_HOST_POWER_MODE_USB_DEV, true));
```

The local BSP implementation of `bsp_usb_host_start()` only installs the ESP-IDF USB Host library and starts the USB host event task. It does not choose between physical USB-A and USB-C connectors. The BSP exposes `bsp_usb_c_detect()`, `bsp_usb_a_detect()`, and `bsp_set_usb_5v_en()`, but this application does not currently use those functions for diagnostics or port selection.

The PI4IO expander setup currently appears to assert `USB5V_EN` during display bring-up, before the USB host is started. That should be verified with hardware logs and, if possible, a meter.

### First USB diagnostic changes to make

Add a small `main/usb_diag/` module or equivalent early boot logging before CAT/UAC driver open:

1. Log Tab5 connector-detect state:
   - `bsp_usb_c_detect()`
   - `bsp_usb_a_detect()`
2. Explicitly call and log `bsp_set_usb_5v_en(true)` before installing USB host.
3. Register a minimal USB host client that logs root-port events and every newly enumerated device descriptor:
   - address
   - VID/PID
   - manufacturer/product strings if available
   - full configuration/interface/endpoint summary
4. Do not assume device address 1.
5. Do not require CDC to open before dumping descriptors.
6. Leave the diagnostic path behind a build-time flag, for example `CONFIG_QMX_USB_DIAG`.

Expected triage:

- No `NEW_DEV` event: physical port, VBUS, OTG role, or cable orientation/CC issue.
- Device appears but VID/PID is not `0483:A34C`: update or broaden QMX identity detection.
- VID/PID appears but CDC does not open: descriptor/interface-selection issue.
- CDC opens but UAC does not stream: UAC interface, alternate setting, or sample-format issue.
- UAC streams but spectrum is wrong: audio sample decode or I/Q handling issue.

### Known fragile USB assumptions

`main/cat/cat.c` hardcodes:

```c
#define QMX_VID 0x0483
#define QMX_PID 0xA34C
cdc_acm_host_open(QMX_VID, QMX_PID, 0, &cfg, &s_cdc_dev);
```

The third argument is currently fixed at `0`. If the QMX+ composite descriptor exposes CDC on a different interface number, CAT will never open. Replace this with descriptor-based CDC discovery or a retry loop over plausible CDC interface numbers.

`main/audio/audio.c` opens UAC using the address/interface reported by the UAC driver, but then assumes alternate setting `1`:

```c
uac_host_get_device_alt_param(s_uac_dev, 1, &alt);
```

Replace this with discovery that selects a receive alt-setting matching:

- 2 channels
- 24-bit samples
- 48000 Hz, or the closest expected QMX rate
- isochronous IN endpoint

The decode path currently assumes packed little-endian 24-bit stereo, six bytes per I/Q pair. Keep that assumption only after descriptor logs confirm it.

## Build environment

Pinned target:

- ESP-IDF: v5.4.4
- Target: `esp32p4`
- ESP32-P4 v1.3 / ECO2 constraints remain important:
  - `CONFIG_ESP32P4_REV_MIN_0=y`
  - `CONFIG_ESP_DEFAULT_CPU_FREQ_MHZ=360`

Standard command-line flow:

```powershell
idf.py set-target esp32p4
idf.py build
idf.py -p COMx flash monitor
```

Exit monitor with `Ctrl+T`, then `Ctrl+X`.

The repository contains machine-specific VS Code settings. Treat `.vscode/settings.json` as a local starting point only. In particular, do not assume:

- `COM3` is the correct port.
- `C:\esp\v5.4.4\esp-idf` is the correct IDF path.
- JTAG flashing is correct for every developer machine.

For VS Code:

1. Install the Espressif IDF extension.
2. Use `ESP-IDF: Configure ESP-IDF extension`.
3. Select the existing ESP-IDF v5.4.4 installation.
4. Set target to `esp32p4`.
5. Select the actual Tab5 serial/JTAG port shown by Windows Device Manager.
6. Run `ESP-IDF: Build your project` before flashing.

Recommended repository improvements:

- Move machine-specific VS Code settings to `.vscode/settings.example.json`.
- Add `.vscode/extensions.json` recommending the Espressif extension.
- Add `.vscode/tasks.json` for build, flash, monitor, and fullclean.
- Keep `sdkconfig` committed because this project depends on non-default P4, USB, display, watchdog, PSRAM, Wi-Fi remote, and PPA settings.

## Module map

```text
main/
  main.c                    app_main, task launch, orchestration
  display/display.c         Tab5 BSP bring-up, PI4IO reset sequencing, LVGL rotation
  ui/ui.c                   LVGL widgets, touch tuning, drawer, canvases
  ui/wifi_config.c          On-device Wi-Fi credential modal
  cat/cat.c                 USB CDC-ACM CAT, FA/MD/FW polling, tune writes
  audio/audio.c             USB UAC host, ring-buffer producer, I/Q sample decode
  dsp/dsp.c                 FFT consumer, spectrum state, DC blocker
  dsp/iq_balance.c          Blind adaptive Gram-Schmidt I/Q correction
  render/render.c           Spectrum rendering, smoothing, flat/absolute mode
  render/render_waterfall.c Waterfall rendering and double-height scroll buffer
  screenshot/screenshot.c   Hidden UART screenshot capture
  storage/settings.c        NVS persistence for settings, Wi-Fi, last VFO
  wifi/wifi.c               ESP-Hosted Wi-Fi STA, SNTP, web server launch
  net/webserver.c           HTTP root and `/api/status`
  net/webserver_ws.c        Binary spectrum WebSocket
components/
  m5stack_tab5/             Local Tab5 BSP
  espressif__usb_host_uac/  Patched/local USB UAC component
```

Data flow:

```text
QMX UAC -> audio ring buffer -> DSP FFT -> spectrum mutex -> render -> LVGL
QMX CDC -> CAT parser -> UI frequency/mode/passband state -> touch-to-tune CAT writes
DSP/render state -> WebSocket -> browser spectrum/waterfall
```

## Critical project rules

### Do not enable LVGL PPA rotation

`CONFIG_LVGL_PORT_ENABLE_PPA` must remain disabled. Prior work found that PPA and the USB host stack conflict over DMA resources on this hardware. The failure mode looks like QMX disconnection: both UAC and CDC stop working.

### Keep the local UAC component patch

The project intentionally carries `components/espressif__usb_host_uac/`. Do not replace it blindly with the registry component. The local component is expected to preserve the background-task behavior needed for UAC and CDC-ACM coexistence.

### Keep the polling audio architecture unless replacing it deliberately

`audio_task` polls `uac_host_device_read()` on core 0 and drains the driver buffer in a loop. An earlier event-driven design caused periodic starvation/truncation and visible noise-floor pumping. Do not revert to event-driven reads without reproducing and fixing that failure mode.

### Initialize PI4IO before display and USB diagnostics

`display_init()` initializes the Tab5 PI4IO expander before display bring-up. This releases LCD/touch reset and also configures board power rails. USB diagnostics should verify the resulting USB5V state rather than assuming it.

### Do not send `AI1;` to the QMX CAT port

Prior notes indicate that `AI1;` can partially enable auto-info mode despite returning an error, breaking FA polling until QMX power-cycle. Avoid it.

## Wi-Fi persistence and upgrade policy

Wi-Fi credentials are already stored in NVS under the `qmx` namespace. They should survive ordinary `idf.py app-flash` and normal firmware upgrades as long as the NVS partition is not erased.

Current risks to fix:

1. `main/main.c` erases the default NVS partition when `nvs_flash_init()` reports `ESP_ERR_NVS_NO_FREE_PAGES` or `ESP_ERR_NVS_NEW_VERSION_FOUND`. That loses Wi-Fi, display settings, and last VFO.
2. `panadapter_wifi_reconnect()` schedules Wi-Fi credential writes through the debounced settings task, but does not force a synchronous flush before reconnecting or before the user might power-cycle.
3. The partition table has a small default NVS partition. It mixes UI settings, Wi-Fi credentials, and last VFO in one place.

Recommended policy:

- Never erase NVS automatically during normal boot.
- If NVS cannot be opened, boot with defaults and show/log a clear warning.
- Add an explicit factory-reset gesture or web endpoint for intentional erasure.
- Add a schema/version key and migrate settings forward explicitly.
- After saving Wi-Fi credentials, force a deterministic commit before reconnecting.
- Consider a dedicated, fixed-offset config NVS partition for credentials and long-lived user configuration.

A minimal immediate patch is:

- Add `settings_set_wifi_credentials(ssid, pass)` that writes both values under one mutex and commits synchronously.
- Use it from the LVGL modal and future web config endpoint.
- Stop calling `nvs_flash_erase()` automatically in `app_main()`.

## Web configuration plan

The current web UI starts only after STA Wi-Fi is connected. It serves:

- `/` — browser panadapter/status page
- `/api/status` — battery, Wi-Fi status, IP, frequency JSON
- `/ws` — binary spectrum WebSocket

For configuration through the web, implement two paths:

### 1. LAN configuration when already connected

Add endpoints:

- `GET /api/config`
  - Return non-secret settings only.
  - Return Wi-Fi SSID, RSSI, IP, and connection state.
  - Never return the Wi-Fi password.
- `POST /api/wifi`
  - Accept JSON: `{ "ssid": "...", "password": "..." }`.
  - Validate SSID length <= 32 and password length <= 64.
  - Persist synchronously before reconnect.
  - Return status before dropping the old connection if possible.
- `POST /api/settings`
  - Display range, smoothing, I/Q balance, flat mode, future flat-mode tunables.

### 2. First-boot or recovery configuration

If no SSID is configured, or if the user holds a defined boot/touch gesture, start a temporary SoftAP:

- SSID: `QMX-PANADAPTER-xxxx`, where `xxxx` is derived from MAC/device ID.
- Serve a minimal setup page and `POST /api/wifi`.
- After successful save, stop AP and switch to STA.

Do not expose CAT control or spectrum WebSocket on the provisioning AP unless deliberately enabled.

## Suggested immediate work order

1. Add USB diagnostics and connector/VBUS logging.
2. Verify which Tab5 physical connector can host the QMX+ on this BSP.
3. Replace hardcoded CDC interface `0` with descriptor-based discovery.
4. Replace hardcoded UAC alt-setting `1` with descriptor-based selection.
5. Make Wi-Fi credential commits synchronous and stop automatic NVS erase.
6. Add LAN `POST /api/wifi` and `GET /api/config`.
7. Add SoftAP first-boot provisioning.
8. Clean VS Code settings and add task/extension templates.

## Expected USB success log sequence

A healthy boot should eventually show the equivalent of:

```text
usb_diag: USB-C detect=<0/1> USB-A detect=<0/1> USB5V_EN=on
M5STACK_TAB5: Installing USB Host
usb_diag: NEW_DEV addr=<n>
usb_diag: VID=0483 PID=A34C ...
cat: QMX CDC opened
cat: QMX configured: 38400 baud, 8N1
cat: QMX IQ mode enabled (Q9 1;)
audio: Lib event: RX_CONNECTED addr=<n> iface=<m>
audio: UAC stream started: 48000 Hz, 2 ch, 24-bit
audio: RX ~48000 pairs/s peak L=... R=...
```

If a later log appears without an earlier one, debug the earliest missing stage.

## Review checklist for future changes

- Does the change preserve USB host stability with both CDC and UAC active?
- Does it keep `CONFIG_LVGL_PORT_ENABLE_PPA` disabled?
- Does it avoid automatic NVS erasure?
- Does it preserve Wi-Fi credentials across app-only upgrades?
- Does it avoid leaking Wi-Fi passwords through logs, JSON, screenshots, or browser UI?
- Does it remain compatible with ESP-IDF v5.4.4 and `esp32p4`?
- Does it keep long-running rendering/audio work off paths that can starve USB reads?
- Does it provide enough boot logging to diagnose hardware state without a debugger?
