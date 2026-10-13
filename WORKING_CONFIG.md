# Camera on the custom ESP32-P4 board — working config and findings

Board: custom 2-layer PCB (`ESP32_P4_final files`) + Waveshare ESP32-P4-MODULE-32MB
(silicon v3.2) + RPi Camera (B) OV5647, WiFi via onboard ESP32-C6 (esp-hosted over SDIO).
Last updated: 2026-10-12.

**Status:** real images stream to the browser at ~0.2–0.4 fps with horizontal banding.
The two limits left are both hardware: CSI signal integrity (2-layer routing) and
power (board resets while streaming from a PC USB port).

---

## 1. Everything that had to change (in the order it was found)

### 1.1 C6 co-processor firmware: 0.0.0 → 3.0.9
The module shipped with ancient esp-hosted slave firmware (boot log:
`major version mismatch — OTA coprocessor from host`), so the SDIO link ran in slow
compat mode. Updated over SDIO from the P4 with
`esp_hosted/examples/ota/coprocessor_ota` (C6 fw built at `C:\c6cp`, P4 updater at
`C:\c6host`, LittleFS method). Now: `host=3.0.9 coprocessor=3.0.9 (match)`,
SDIO streaming mode.

### 1.2 Lower MIPI lane rate (sensor patch)
At stock 400 Mbps/lane this board produces no usable frames. Sensor now runs
**160 Mbps/lane** (`0x3035 = 0xA1`, sysclk div 4→10; `0x3036 = 0x80` fixed).
- div 8 (200 M): works but noisier. div 15 (107 M): sensor/receiver go silent.

### 1.3 IDF hsfreqrange table bug (workaround)
ESP32-P4 TRM Table 35.5-2 (D-PHY HS frequency ranges) vs IDF v6.1
`components/esp_hal_cam/esp32p4/mipi_csi_periph.c`: IDF's table is **shifted one
row** (it starts `{90, 99, 0x00}`; TRM says 80–89 → 0x00, 90–99 → 0x10). So IDF
programs the HS-settle range for one band *slower* than the real lane rate for
every rate, e.g. 160 Mbps → IDF writes 0x21 (TRM: 140–149), correct is 0x02;
stock 400 Mbps → IDF writes 0x25 (TRM: 360–399), correct is 0x06.
Settle mismatch is a classic cause of `ErrSotSyncHS`.
Workaround without touching ESP-IDF: the OV5647 driver *declares* 175 Mbps
(`OV5647_MIPI_CSI_LINE_RATE_800x800_50FPS = 175000000`) so IDF picks its 0x02 row,
which is TRM-correct for the real 160 Mbps. The sensor-side timing (`0x4837`) still
uses the real IDI clock (40 MHz). Measured effect on this board: no clear change
(error rate is dominated by the board itself), but keep it — it's the correct value.
**Worth reporting to Espressif**; it may hurt marginal links on other boards too.

### 1.4 Stream server hang (firmware fix)
esp_http_server runs handlers in ONE task. The stock stream handler blocked in
`VIDIOC_DQBUF` forever once its client left → every later request, including the
browser, hung "pending". Fix in `main/simple_video_server_example.c`:
- `VIDIOC_S_DQBUF_TIMEOUT` = 250 ms at device init
- stream loop: DQBUF timeout → `continue`; top of loop
  `recv(MSG_PEEK | MSG_DONTWAIT)` → returns on FIN (0) or RST (<0, not EAGAIN)
- one-shot capture endpoint retries DQBUF up to 15 s

### 1.5 Capture-stall watchdog (firmware fix)
Measured with per-2s DQBUF counters: after 1–3 good frames the CSI receiver
desyncs and DQBUF never returns another buffer (not-done = 0, only timeouts).
- Kconfig per-error restart (`..._ERROR_VIDEO_RESTART=y`): thrashes — stream-on
  itself raises SoT errors → ~700 restarts in 16 s, zero frames. **Keep it =n.**
- Receiver-only `VIDIOC_RESTART`: never recovers the OV5647.
- **Working:** if no good frame for 300 ms → `VIDIOC_RESTART{restart_sensor=true}`
  (measured 2–7 ms, fully synchronous), then drop the first frame after the
  restart (dark — exposure latches a frame late).

### 1.6 Exposure: sensor auto-AE with raised target
- ISP pipeline controller OFF (`CONFIG_ESP_VIDEO_ENABLE_ISP_PIPELINE_CONTROLLER=n`).
- Sensor auto AEC/AGC (`0x3503 = 0x00`). An earlier manual-exposure workaround
  (8× gain) is gone — auto gives more detail and correct color.
- AE target raised (`0x3a0f/10/1b/1e/11/1f`: `58/50/58/50/60/28 → 78/70/78/70/90/30`).
  The bright speckle on corrupted lines inflates metered brightness, so at the
  stock target AE drove the picture dark over about a minute. Verified stable 90 s.

---

## 2. Remaining problems — both hardware

### 2.1 CSI signal integrity → banding + low fps
Error timing on serial: **~80 MIPI errors/s, continuous** (gaps 0–20 ms, avg 13 ms;
no clean windows). Only frames that happen to fall between errors survive, so
throughput is capped no matter how the watchdog is tuned. Banding = corrupted
lines inside frames that still complete DMA; no firmware filter can detect that.
- Reseating the ribbon: no change (≈82/s). Cleaning flux with IPA: no change (≈82/s).
- Root cause (from the KiCad files): 2-layer board, no solid ground plane under the
  CSI pairs, intra-pair skew D0 1.165 mm / D1 0.663 mm (limit ~0.13 mm), 23 mm
  lane-to-lane mismatch, clock pair changes layers through 2 vias.
- Fix: 4-layer respin — see `ESP32_P4_final files/CHANGES_AND_TODO.md`.
- **Resolution now 800×640** (`CONFIG_CAMERA_OV5647_MIPI_DEFAULT_FMT_RAW8_800X640_50FPS`,
  same sensor patches ported to its register table + settings.h): 20% fewer bits per
  frame at the same lane rate. Measured 40 s each: 800×800 ≈ 0.34 fps / ~80 err/s →
  **800×640 ≈ 0.72 fps / ~65 err/s** (~2× frames). 800×640 is the smallest stock mode;
  going lower needs a custom sensor window (0x3808–0x380b) in a new register table.

### 2.2 Power → board resets while streaming
Captured with a port-reopening serial logger: uptime jumps 54114 ms → 282 ms, no
panic text, and `esp_reset_reason()` after the drop = **1 (POWERON)** — the rail fully
collapsed (a firmware crash would be 4/5/6; a sag caught by the brownout detector
would be 9). Happens only while streaming: P4 + PSRAM + C6 WiFi TX bursts + camera
exceed a PC USB port (500 mA USB2 / 900 mA USB3), the port cuts power, the COM port
disappears and the board cold-boots. Earlier "board disconnected / COM port moved /
WiFi dropped" events were very likely this.
- Fix now: power from a 5 V 2 A+ USB-C charger or a powered USB hub.
- Respin: add bulk capacitance (100–470 µF) on 5 V and 3.3 V near the module and C6.
- Suspicion to test once powered properly: rail noise may contribute to the MIPI
  errors — re-measure errors/s on a strong supply.
- Software option (not applied): cap WiFi TX power to shrink current spikes.

---

## 3. Where the changes live

Firmware project: `C:\Users\mvadr\Downloads\p4Webserver-main\p4Webserver-main`
- `main/simple_video_server_example.c` — 1.4, 1.5, reset-reason log at boot,
  DQBUF debug counters (ESP_LOGD).
- `main/idf_component.yml` — absolute `override_path` to the local
  esp-video-components clone.
- `sdkconfig.defaults.esp32p4` — OV5647, ISP off, CSI event on, restart-on-error
  off, WiFi creds, 16 MB flash, JPEG q60, 3 buffers.

Local component clone (**re-cloning loses these**): `D:\aihardware\esp-video-components`
- `esp_cam_sensor/sensors/ov5647/private_include/ov5647_settings.h`
  — IDI 40 MHz, declared lane rate 175 Mbps (1.2, 1.3).
- `.../ov5647_mipi_2lane_24Minput_800x800_raw8_50fps.h`
  — `0x3035/0x3036`, auto-AE, raised AE targets (1.2, 1.6).

The factory Waveshare POE board runs the stock config: rebuild without the sensor
edits before flashing that board.

---

## 4. Using it

- Open **http://192.168.4.46** in ONE tab (stream on :81 is single-client). The
  first frame takes a few seconds. The IP has come up as .47 after replugging.
- If WiFi shows `rssi=-128` / `reason=201`: the antenna is unplugged.
- If the COM port vanishes or the board reboots mid-stream: power (section 2.2).
- Healthy signal check on serial: count `MIPI-CSI` lines per second.

Build/flash:
```
$env:IDF_COMPONENT_CACHE_PATH = "C:\cc"   # avoids Windows MAX_PATH on esp_hosted
. C:\esp\v6.1\esp-idf\export.ps1
cd C:\Users\mvadr\Downloads\p4Webserver-main\p4Webserver-main
idf.py build
idf.py -p COM5 flash
```
WiFi: uac / andromeda123. The ESP32-C6 is 2.4 GHz only — 5 GHz / WPA3-only APs and
iPhone hotspots without "Maximize Compatibility" won't be found.
