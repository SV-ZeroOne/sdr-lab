# NESDR Mini 2 + SDRangel + Hermes: As-Built Setup Record

Set up 2026-10-09/10 on the Windows 11 PC (hostname: this machine, user `Neo`). This records **what was actually done and what worked**, so it can be reproduced or debugged later. The step-by-step guide is in [NESDR_Mini2_SDRangel_Hermes_Plan.md](NESDR_Mini2_SDRangel_Hermes_Plan.md).

```
 whip ─MCX─► NESDR Mini 2 ─USB (rear port)─► WinUSB (Zadig, Interface 0)
                                                │
                                   SDRangel 7.27.2 ──REST :8091──► sdrangel-mcp.exe (stdio, 25 tools)
                                                                            │
                                                                   Hermes Agent v0.21.5 ◄──► llama-server :8080
                                                                   + sdr-analyst skill        Qwen3.8-27B Q4_K_XL
```

---

## 1. Hardware facts

| Item | Value |
|---|---|
| Dongle | NooElec NESDR Mini 2, RTL2832U + Rafael Micro R820T2, serial `00000001` |
| USB ID | `VID_0BDA&PID_2838` (shows as "RTL2838UHIDIR" in rtl_test) |
| Tuning range | ~24–1766 MHz (R820T2). No HF without an upconverter. |
| Sample rate used | 2,400,000 S/s (2,048,000 also fine) |
| Gain steps (dB) | 0.0 0.9 1.4 2.7 3.7 7.7 8.7 12.5 14.4 15.7 16.6 19.7 20.7 22.9 25.4 28.0 29.7 32.8 33.8 36.4 37.2 38.6 40.2 42.1 43.4 43.9 44.5 48.0 49.6 |
| Crystal error | **+19 ppm** (rtl_test -p, 20+ min, cumulative held 17–20). Entered as LO ppm = 19. No TCXO, so re-check after warm-up. |
| USB quality | 1 sample per million lost (rtl_test summary). Rear motherboard port. |
| Antenna | Stock telescopic whip, MCX, indoors. ~75 cm for FM; collapse to shortest (~6.5 cm) for 1090 MHz. |
| Location | UK (RDS PI codes start with C; clock GMT+1) |

**rtl_test quirks on Windows** (not faults):
- `-t` ends with `No E4000 tuner found, aborting.` That's the E4000 test; normal on an R820T.
- `-p` prints repeating `real sample rate: 0 … -1000000` / `10 … -999995` pairs. Reporting glitch; the cumulative ppm staying steady proves no data was lost.
- Occasional `lost at least N bytes` (tens–hundreds of bytes out of ~4 MB/s) is negligible.

---

## 2. Driver (Zadig)

What went wrong first: Zadig was run on **Interface 1** → `MI_01` OK, `MI_00` Error, and `rtl_test` failed with `usb_claim_interface error -12`.

Fix that worked:
1. Zadig as Administrator → Options → **List All Devices**.
2. Select **`Bulk-In, Interface (Interface 0)`**, USB ID **`0BDA 2838 00`** (last box `00`).
3. Target **WinUSB** → Install/Replace Driver.

Verify:
```powershell
Get-PnpDevice -PresentOnly | ? InstanceId -match 'VID_0BDA&PID_2838' | ft Status,Class,FriendlyName,InstanceId
# MI_00 must be OK / USBDevice. MI_01 being OK as well is harmless.
C:\sdr\rtl-sdr\rtl_test.exe -t      # "Found Rafael Micro R820T tuner"
```
Re-run Zadig if the dongle moves to a different USB port or after a Windows feature update.

---

## 3. Command-line tools

- rtl-sdr Windows build (from Downloads\Release\x64) copied to **`C:\sdr\rtl-sdr\`**: `rtl_test`, `rtl_fm`, `rtl_power`, `rtl_tcp`, `rtl_adsb`, `rtl_sdr`, `rtl_eeprom`, `rtl_biast`.
- Log folder for decoders: **`C:\sdr\logs\`**.
- Only **one program** can own the dongle at a time (close SDRangel before rtl_* and vice versa).

Useful one-liners:
```powershell
C:\sdr\rtl-sdr\rtl_test.exe -p                                   # ppm estimate (10+ min)
C:\sdr\rtl-sdr\rtl_power.exe -f 118M:137M:12.5k -g 40 -i 10 -e 15m C:\sdr\logs\airband_survey.csv
```

---

## 4. SDRangel 7.27.2

- Installer: `sdrangel-7.27.2-win64.exe` (release 2026-08-19). Qt 6.9.3, OpenGL 4.6 — renders fine on this multi-GPU box.
- Started from the shipped **RTL SDR → ADS-B** example configuration, then changed to FM.
- Rolled-up subwindows: use the **Tile** toolbar button or double-click the title bars.
- Device start button: **green square = running**, triangle = stopped, magenta = error.

### 4.1 Working FM-broadcast settings
| Panel | Setting | Value |
|---|---|---|
| RTL-SDR | Centre | 100,000 kHz |
| | Sample rate | 2,400,000 S/s, Dec 1, RF BW 2500 kHz |
| | LO ppm | 19 |
| | Gain | **29.7 dB manual, AGC off** (23–33 dB range is right for FM) |
| | Bias-T / DS / Ofs | off |
| BFM Demod | Δf | +600,000 Hz → 100.6 MHz (`-200,000` → 99.8) |
| | RF BW | **200 kHz** (180 clipped RDS; 250 max) |
| | AF BW | 15 kHz (10–12 kHz = less hiss) |
| | De-emphasis | **50 µs** (UK/EU; 75 µs is US) |
| | Stereo | on for RDS pilot indicator; **mono = much less hiss** on a ~30 dB signal |
| | RDS | on |

Results: stations at 99.8 / 100.0 / 100.6 MHz ~30 dB above the floor; weaker at 99.2, 99.6, 100.5, 100.9, 101.1. **100.6 MHz decodes RDS: PI `C2A1`, PTY "Light Classics M", coverage National** (almost certainly Classic FM). A narrow wobbling carrier at ~99.6 MHz is local interference (switching PSU / monitor), not a station.

### 4.2 RDS lessons
- Gain 42–44 dB **overloaded** the dongle: audio sounded fine, RDS Dem 5 %. Dropping to ~23–30 dB fixed it.
- Dem % = clean RDS bits; Dec % = groups passing CRC. Need Dem ≳ 80 % before PSN/radiotext fill in.
- RDS is far more multipath-sensitive than audio: antenna position/orientation moves Dem % within seconds.
- "Light Classics M" is the **PTY**, not the station name; PSN needs several G0 groups.

### 4.3 ADS-B (1090 MHz)
- Preset settings: 1,090,000 kHz, 2.4 MS/s, gain 42–49.6, ADS-B Demod SR 4, threshold 8.2.
- Result at 23:30 with the whip indoors: **no decodes**, empty waterfall. FM test proved the RF path works, so it's a signal problem. Retry in daylight, whip collapsed, at a window.
- Map shows **"API KEY REQUIRED carto.com/basemaps/apikey"**: CARTO started requiring keys on 2026-09-25; 7.27.2 has **no field** for the key (maintainer fix pending, [issue #2968](https://github.com/f4exb/sdrangel/issues/2968)). Workarounds: Display Settings → Map → **Street** type, or a free **MapTiler** key + Satellite. Set **Preferences → My Position** for correct decoding.

### 4.4 Saving
- **Preset** (per station): star ★ in the *device window's* title bar → Group/Description → save-as-new.
- **Configuration** (whole layout): Preferences → Configurations → Save.

---

## 5. REST API (verified 2026-10-10)

Base: `http://127.0.0.1:8091/sdrangel` (no auth — keep on localhost). Swagger UI at `http://127.0.0.1:8091`.

```powershell
Invoke-RestMethod http://127.0.0.1:8091/sdrangel
Invoke-RestMethod http://127.0.0.1:8091/sdrangel/deviceset/0/device/settings
Invoke-RestMethod http://127.0.0.1:8091/sdrangel/deviceset/0/channel/0/settings
Invoke-RestMethod http://127.0.0.1:8091/sdrangel/deviceset/0/channel/0/report
```

Field names as returned by this build:
- Device: `deviceHwType: "RTLSDR"`, key **`rtlSdrSettings`** → `centerFrequency` (Hz), **`gain` in tenths of dB** (297 = 29.7), `devSampleRate`, `loPpmCorrection`, `agc`, `rfBandwidth`, `biasTee`, `dcBlock`.
- Channel: `channelType: "BFMDemod"`, key **`BFMDemodSettings`** → `inputFrequencyOffset` (Hz from centre), `rfBandwidth`, `afBandwidth`, `deEmphasis` (0 = 50 µs), `audioStereo`, `rdsActive`, `squelch`, `volume`.
- Channel report: `BFMDemodReport` → `channelPowerDB`, `pilotLocked`, `pilotPowerDB`, `rdsReport.{pid, progServiceName, radioText, piType, piCoverage, demodStatus, decodStatus, rdsDemodFrequency}`.
- PATCH to `/deviceset/0/device/settings` or `/channel/N/settings` with the same key wrapper changes only the listed fields.

---

## 6. sdrangel-mcp (the bridge Hermes uses)

- Project: <https://github.com/thereisnotime/sdrangel-mcp> — Go, MIT, **v0.1.0 (2026-09-20)**, tested upstream against SDRangel 7.27.2.
- Installed to **`C:\sdr\sdrangel-mcp\sdrangel-mcp.exe`** from `sdrangel-mcp_0.1.0_windows_amd64.zip`.
  SHA-256 verified: `1ac26c97bababa135fad9fbeb2ad52db06494288eb7a5828fd26e0bd5c2373b4`.
- Config via env: `SDRANGEL_BASE_URL` (default `http://localhost:8091`), `SDRANGEL_TIMEOUT`, `SDRANGEL_LOG_LEVEL`.
- Exposes **93 tools**. Manual test:
  ```powershell
  $env:SDRANGEL_BASE_URL = "http://127.0.0.1:8091"
  C:\sdr\sdrangel-mcp\sdrangel-mcp.exe tools
  C:\sdr\sdrangel-mcp\sdrangel-mcp.exe call get_instance_summary
  ```
- Tool-call convention: `get_*_settings` first, then `patch_*_settings` echoing `settingsKey` (`rtlSdrSettings`, `BFMDemodSettings`, …) with only the changed fields.

---

## 7. Hermes Agent wiring

- Hermes v0.21.5 (native Windows install), data dir `%LOCALAPPDATA%\hermes\`, config **`%LOCALAPPDATA%\hermes\config.yaml`**.
- Model: provider `'100'` "Llama Server" → `http://127.0.0.1:8080/v1`, model `qwen38` (Qwen3.8-27B-UD-Q4_K_XL, 262K ctx, on ROCm0+ROCm1 = both R9700s). `supports_vision: false` (see §9).
- A stale `sdrangel:` entry (`url: http://127.0.0.1:8092/mcp`, nothing listening) was **replaced** on 2026-10-10. Backup: `%LOCALAPPDATA%\hermes\config.yaml.bak-2026-10-10-sdrangel`.

Current entry (verbatim):
```yaml
  sdrangel:
    # thereisnotime/sdrangel-mcp v0.1.0 (stdio) over the SDRangel REST API on :8091
    command: C:/sdr/sdrangel-mcp/sdrangel-mcp.exe
    args: ["serve"]
    env:
      SDRANGEL_BASE_URL: "http://127.0.0.1:8091"
    connect_timeout: 30
    timeout: 60
    enabled: true
    sampling:
      enabled: false
    tools:
      include:
        - get_instance_summary
        - list_device_sets
        - get_device_settings
        - patch_device_settings
        - get_device_run_status
        - start_device
        - stop_device
        - list_channel_plugins
        - add_channel
        - delete_channel
        - get_channel_settings
        - patch_channel_settings
        - get_channel_report
        - get_channels_report
        - list_presets
        - load_preset
        - create_preset
        - list_configurations
        - load_configuration
        - list_features
        - add_feature
        - get_feature_report
        - patch_feature_settings
        - start_feature
        - stop_feature
      prompts: false
      resources: false
```
`include` is a whitelist, so `stop_instance`, `set_location`, `import_*`/`export_*`, `delete_configuration/preset`, `remove_device_set` and all `set_*` replace-all tools are **never registered**. Tools appear in Hermes as `mcp_sdrangel_<name>`.

Verification commands:
```powershell
hermes mcp list            # sdrangel ... 25 selected  ✓ enabled
hermes mcp test sdrangel   # Connected, Tools discovered: 93 (raw; filter applies in-session)
hermes config check        # Config version 49 ✓
```
In a running session: `/reload-mcp` after editing the config. **SDRangel must be running before Hermes starts**, or the server fails to connect.

Skill: **`%LOCALAPPDATA%\hermes\skills\sdr-analyst\SKILL.md`** — hardware facts, API conventions, gain guidance, identification method, receive-only rules, results so far.

First test prompt for Hermes:
> Using the sdrangel tools, tell me what the radio is tuned to right now and how strong the signal is. Then retune the FM channel to 99.8 MHz by changing only the channel's input frequency offset, wait, and tell me whether RDS is decoding there.

Expected call sequence: `get_device_settings` → `get_channel_settings` → `patch_channel_settings` (`inputFrequencyOffset: -200000`, key `BFMDemodSettings`) → `get_channel_report`.

---

## 8. Quick troubleshooting

| Symptom | Fix |
|---|---|
| `usb_claim_interface error -12` / SDRangel can't open RTL-SDR | Zadig WinUSB on **Interface 0**; close any other program using the dongle |
| Dongle vanished after moving ports / Windows update | Re-run Zadig |
| Audio fine but RDS/decoder dead | Gain too high → 23–33 dB for FM; RF BW ≥ 200 kHz |
| Hiss on FM | Mono, gain +3–6 dB, better antenna position, AF BW 10–12 kHz |
| Empty ADS-B table | Daylight, whip collapsed at a window, gain 45–49.6; else a real 1090 MHz antenna |
| Map "API KEY REQUIRED" | Known 7.27.2 limitation; Street map type or MapTiler key |
| Hermes: no `mcp_sdrangel_*` tools | Start SDRangel first; `hermes mcp list`; `/reload-mcp` |
| Hermes tool error "settingsKey" | Model must `get_*_settings` first and echo the key in `patch_*` |

---

## 9. Still open (as of 2026-10-10)

1. **Vision**: llama-server runs with `--no-mmproj`. Relaunch with
   `--mmproj C:\llama-cache\models--unsloth--Qwen3.8-27B-GGUF\snapshots\4ca720788d1e01f1bff70c033e0d0028fd02e502\mmproj-BF16.gguf`
   and set `providers.'100'.models.qwen38.supports_vision: true` in Hermes. Check `/props` → `modalities.vision: true`. If `--spec-type draft-mtp` conflicts with mmproj, drop the spec flags for vision sessions.
2. ADS-B retry in daylight; consider a quarter-wave ground-plane antenna (6.9 cm + 4 radials).
3. Turn on log-to-file in each decoder → `C:\sdr\logs\`.
4. Upstream SDRangel release with a Carto key field (watch issue #2968). `hermes update` is also pending.
5. Missions 2–13 in the plan (airband AM, AIS, APRS 144.800, rtl_433 at 433.92/868 MHz, radiosondes, DAB Band III, FT8, unknown-signal ID).

---

## 10. File and path index

| What | Where |
|---|---|
| Step-by-step plan | `Desktop\Claude_Projects\NESDR_Mini2_SDRangel_Hermes_Plan.md` |
| This record | `Desktop\Claude_Projects\NESDR_SDRangel_Hermes_Setup_Record.md` |
| rtl-sdr tools | `C:\sdr\rtl-sdr\` |
| sdrangel-mcp | `C:\sdr\sdrangel-mcp\sdrangel-mcp.exe` (+ README.md, LICENSE, zip) |
| Decoder logs | `C:\sdr\logs\` |
| Hermes config | `%LOCALAPPDATA%\hermes\config.yaml` (backup `.bak-2026-10-10-sdrangel`) |
| Hermes skill | `%LOCALAPPDATA%\hermes\skills\sdr-analyst\SKILL.md` |
| Qwen3.8 GGUFs | `C:\llama-cache\models--unsloth--Qwen3.8-27B-GGUF\snapshots\4ca7207…\` (Q4_K_XL 16.4 GB, IQ4_XS 13.3 GB, mmproj-BF16 0.87 GB) |
| llama-server | `C:\llama\lemonade-b1336-rocm-gfx120X-x64\llama-server.exe`, port 8080 |

## Sources
- NESDR Mini 2: <https://www.nooelec.com/store/sdr/sdr-receivers/nesdr-mini-2.html>
- Zadig / RTL-SDR on Windows: <https://www.rtl-sdr.com/getting-the-rtl-sdr-to-work-on-windows-10/>
- SDRangel: <https://github.com/f4exb/sdrangel> · REST API wiki: <https://github.com/f4exb/sdrangel/wiki/Web-(http)-REST-API> · ADS-B readme: <https://github.com/f4exb/sdrangel/blob/master/plugins/channelrx/demodadsb/readme.md> · map key issue: <https://github.com/f4exb/sdrangel/issues/2968>
- sdrangel-mcp: <https://github.com/thereisnotime/sdrangel-mcp>
- Hermes docs: <https://hermes-agent.nousresearch.com/docs/user-guide/features/mcp>
- CARTO basemap key change: <https://docs.carto.com/faqs/carto-basemaps>
- SDR↔MCP comparison gist: <https://gist.github.com/danbri/b0c04702ed7a6a303d67c911cde4497a>
