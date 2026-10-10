# NESDR Mini 2 + SDRangel + Qwen3.8-27B/Hermes: Setup Plan (Windows 11)

Goal: get the NooElec NESDR Mini 2 receiving in SDRangel, then give Hermes Agent (running Qwen3.8-27B on your R9700s) tools to tune the radio, read the decoder output, and explain what you're hearing.

```
 Antenna ─MCX─► NESDR Mini 2 ─USB─► WinUSB driver (Zadig)
                                       │
                                   SDRangel 7.27.2  ──REST API :8091──► sdrangel-mcp (Python, stdio)
                                   (demods/decoders)                        │
                                       │ CSV logs / screenshots            ▼
                                       └──────────────────────────►  Hermes Agent ◄──► llama-server :8080
                                                                     (+ sdr-analyst skill)    Qwen3.8-27B (+ mmproj vision)
```

---

## 0. Where you are now (checked on this PC, 2026-10-09)

| Item | Status |
|---|---|
| Dongle | **Plugged in.** USB `VID_0BDA&PID_2838` is detected, but both `Bulk-In, Interface` entries (MI_00, MI_01) show **Error**, meaning no driver yet. Step 2 fixes this. |
| LLM | `llama-server` (lemonade b1336, ROCm, `--device ROCm0,ROCm1`) on `:8080`, serving `Qwen3.8-27B-UD-Q4_K_XL` as `qwen38`, 262K context. Started with `--no-mmproj`, so **vision is off**. |
| Vision projector | `mmproj-BF16.gguf` is **already downloaded** in the same snapshot folder. |
| Hermes Agent | v0.21.5 installed natively (`%LOCALAPPDATA%\hermes`). It already points at `http://127.0.0.1:8080/v1` / `qwen38`, with `supports_vision: false`. One MCP server (blender) is configured. An update is available (`hermes update`). |
| Python | 3.14.7 on PATH |
| SDR software | None installed yet |

---

## 1. Know the hardware

| Spec | NESDR Mini 2 |
|---|---|
| Chips | RTL2832U (USB/ADC) + Rafael Micro R820T2 tuner |
| Tuning range | ~25 MHz – 1750 MHz (varies slightly per unit) |
| Usable bandwidth | ~2.4 MS/s stable (8-bit ADC) |
| Clock | **No TCXO.** Expect a few to tens of ppm frequency error, so calibrate (step 3.3). |
| Antenna port | **MCX female** on the dongle; telescopic whip included |
| Bias-tee / direct sampling | None. **No HF (<25 MHz) without an upconverter** (e.g. Ham It Up). |
| Transmit | Receive-only. This also makes LLM control low-risk. |

**Antenna tips:** Height and placement matter more than anything else, so put it at a window or outside. A quarter-wave whip is about `71.3 / f(MHz)` metres long: ~75 cm for FM broadcast, ~49 cm for 2 m/airband, ~44 cm for AIS/APRS, ~6.5 cm for ADS-B at 1090 MHz. Plug the dongle directly into a USB port or a powered hub, not an unpowered hub.

---

## 2. Install the WinUSB driver (Zadig)

Windows loads no usable driver for SDR use (or a DVB-T TV driver). Every RTL-SDR app on Windows needs **WinUSB on Interface 0**. NooElec's "NESDR Driver Installer for Windows" is just a Zadig wrapper, so use current Zadig directly.

1. Optional: create a System Restore point.
2. Download Zadig from <https://zadig.akeo.ie> and run it as Administrator.
3. Menu **Options → List All Devices** (on Win 11 you may also need to **uncheck "Ignore Hubs or Composite Parents"**).
4. In the dropdown select **`Bulk-In, Interface (Interface 0)`**. On some systems it's labelled `RTL2838UHIDIR` or `RTL2832U`.
   - Confirm USB ID shows **`0BDA 2838 00`**. The trailing `00` is Interface 0.
   - **Do not** pick Interface 1. WinUSB on Interface 1 causes `usb_open error -12`.
   - **Do not** pick a keyboard/mouse/other device. Whatever you select gets its driver replaced.
5. Target driver: **WinUSB** → **Replace Driver / Install Driver**. Wait for "successfully installed".
6. Verify in PowerShell:
   ```powershell
   Get-PnpDevice -PresentOnly | ? InstanceId -match 'VID_0BDA&PID_2838' | ft Status,Class,FriendlyName,InstanceId
   ```
   `MI_00` should now be **OK** with class `USBDevice`. `MI_01` staying in Error is normal and harmless.

> Re-run Zadig if you later plug the dongle into a **different USB port** and it stops being found. Windows can also revert the driver after a major feature update.

---

## 3. Smoke-test with the RTL-SDR command-line tools

These tools are useful later too: `rtl_power` gives the LLM spectrum surveys as CSV.

1. Download the Windows release zip from **rtl-sdr-blog** (<https://github.com/rtlsdrblog/rtl-sdr-blog/releases>). It works with generic R820T2 dongles. Extract to `C:\sdr\rtl-sdr\`.
2. Test the tuner:
   ```powershell
   C:\sdr\rtl-sdr\rtl_test.exe -t
   ```
   Expect `Found Rafael Micro R820T tuner` and a list of 29 supported gains. The run then ends with `No E4000 tuner found, aborting.` That's **normal**: `-t` is a test for a different tuner chip (E4000), and the "direct sampling" line is harmless too.
3. **Rough ppm measurement:** `rtl_test.exe -p` and let it run 5–10 min. It reports a cumulative PPM estimate. Write it down; you'll enter it in SDRangel. You'll refine it in step 4.5.

> **One app at a time.** Only one program can own the dongle. Close SDRangel before running `rtl_*`/`rtl_433`, and vice versa (see 5.3 for running both).

---

## 4. Install and configure SDRangel

### 4.1 Install
- Download **`sdrangel-7.27.2-win64.exe`** (released 2026-08-19) from <https://github.com/f4exb/sdrangel/releases/latest> and install it.
- SDRangel needs **OpenGL 3.0+**. If the spectrum/waterfall is black on this multi-GPU box, go to **Settings → System → Display → Graphics**, add SDRangel, and pick the GPU that drives your monitor.

### 4.2 First run: add the RTL-SDR source
1. Start SDRangel. In the device set, click the **device selector** and choose **RTL-SDR [0] …** (the NESDR serial is usually `00000001`).
2. Suggested starting settings in the RTL-SDR source window:
   - **Sample rate:** 2,048,000 S/s (go up to 2.4 MS/s if it doesn't drop samples)
   - **LO ppm:** the value from step 3.3 (**measured 2026-10-09: +19**, cumulative PPM held at 17–20 over a 20+ minute run; the repeating `sample rate: 0 / -1000000` lines were a Windows reporting glitch, not data loss)
   - **Gain:** start manual ~30–40 dB. Raise it until the noise floor lifts a few dB, then stop. Too much gain overloads with FM broadcast "ghosts" everywhere.
   - **AGC:** off for now (manual is easier to learn with)
3. Press the **play/start** button. You should see a live spectrum and waterfall.
4. **Save a configuration** (Preferences → Configurations) so you can restore it later.

### 4.3 First listen: FM broadcast
- Tune the centre to a local FM station (88–108 MHz).
- Add channel **Broadcast FM Demodulator** (BFM; it shows RDS station name/text). Set audio output to your speakers.

### 4.4 First decoder: ADS-B aircraft (best "wow" test)
- Centre 1090 MHz, sample rate 2.0–2.4 MS/s, gain high (~40+ dB), use a short (~6.5 cm) antenna near a window.
- Add channel **ADS-B Demodulator**. Aircraft appear in the table and on the map (add the **Map** feature).

### 4.5 Fine-tune ppm
Tune to a strong, known-exact carrier: a local NOAA Weather Radio/broadcast carrier, or the centre of an FM station. Zoom in and adjust **LO ppm** until the signal sits exactly on its published frequency. The value drifts a little as the dongle warms up, so recalibrate after ~10 minutes of running.

### 4.6 Enable logging for the LLM
Most SDRangel decoders (ADS-B, AIS, Packet/APRS, Pager/POCSAG, Radiosonde, FT8 …) have a **log-to-file (CSV)** button and/or UDP message forwarding in their channel window. Create `C:\sdr\logs\` and point every decoder's log there. That folder becomes the LLM's "eyes" on decoded data (step 7).

### 4.7 REST API: check it's reachable
The GUI starts the REST API by default on **`http://127.0.0.1:8091`** (run `sdrangel -h` for the address/port flags).
```powershell
Invoke-RestMethod http://127.0.0.1:8091/sdrangel | ConvertTo-Json -Depth 4
Invoke-RestMethod http://127.0.0.1:8091/sdrangel/deviceset/0/device/settings | ConvertTo-Json -Depth 6
Invoke-RestMethod "http://127.0.0.1:8091/sdrangel/channels?direction=0" | ConvertTo-Json -Depth 4
```
- Open <http://127.0.0.1:8091> in a browser for the **built-in docs and Swagger UI** (live, try-it-out).
- The second call shows the **exact field names** for your RTL-SDR (e.g. `centerFrequency`, `gain` in tenths of dB, `devSampleRate`, `loPpmCorrection`). Compare them against the MCP server below.
- The third call lists the exact Rx **channel type IDs** (`NFMDemod`, `ADSBDemod` …).
- Leave the API on `127.0.0.1`. It has no authentication.

---

## 5. Optional companion tools (very LLM-friendly)

### 5.1 `rtl_433`: ISM-band sensors (weather stations, TPMS, doorbells, meters)
- Windows `win-x64` zip from <https://github.com/merbanan/rtl_433/releases> → `C:\sdr\rtl_433\`.
- It outputs **structured JSON**, which is ideal for an LLM:
  ```powershell
  C:\sdr\rtl_433\rtl_433.exe -f 433.92M -M time:iso -M level -F json:C:\sdr\logs\rtl433.jsonl
  ```
  (US: also try `-f 915M`. EU: `-f 868M`.)

### 5.2 `rtl_power`: band surveys for the LLM to interpret
```powershell
C:\sdr\rtl-sdr\rtl_power.exe -f 118M:137M:12.5k -g 40 -i 10 -e 15m C:\sdr\logs\airband_survey.csv
```
Hand the CSV to Hermes: *"Which frequencies in this survey show activity, how often, and what services are allocated there?"*

### 5.3 Running SDRangel and rtl_433 at the same time (advanced, verify on your build)
SDRangel has a **Remote TCP Sink** channel that speaks the `rtl_tcp` protocol. Add it in SDRangel (centre 433.92 MHz, 250 kS/s channel rate), then:
`rtl_433.exe -d rtl_tcp:127.0.0.1:1234 …`

---

## 6. LLM side: Qwen3.8-27B + vision

You're already serving Qwen3.8-27B. For radio work, **vision matters**: the model can read waterfall screenshots (signal shape, bandwidth, burst pattern) to help identify unknown signals.

### 6.1 Relaunch llama-server with the vision projector
In your current launch command, replace `--no-mmproj` with:
```
--mmproj C:\llama-cache\models--unsloth--Qwen3.8-27B-GGUF\snapshots\4ca720788d1e01f1bff70c033e0d0028fd02e502\mmproj-BF16.gguf
```
- Check afterwards: `Invoke-RestMethod http://127.0.0.1:8080/props | % modalities` should show `vision: True`.
- Your command also uses `--spec-type draft-mtp`. If the server refuses to start or speculative decoding stops working once mmproj is loaded, drop the spec flags for a vision session.
- 0.87 GB extra VRAM is no problem on 2× R9700.
- Reminder from your notes: don't run R9700 benchmarks while extra vision servers are up.

### 6.2 Tell Hermes the model can see
In `%LOCALAPPDATA%\hermes\config.yaml`:
```yaml
providers:
  '100':
    models:
      qwen38:
        supports_vision: true    # was false
```

### 6.3 Qwen3.8 behaviour notes
- It's known to **over-think by default**. For quick back-and-forth tool calls ("tune to X, read the report"), keep prompts specific. Use long reasoning for identification/analysis questions.
- Your 262K context is plenty. Decoder logs can be large, so have tools return the **last N lines**, never whole files.

---

## 7. Bridge: an SDRangel MCP server for Hermes

SDRangel has **no MCP built in**, only the REST API. Two ways to bridge it:

| | **7A. `sdrangel-mcp` (recommended)** | **7B. Own Python server (fallback)** |
|---|---|---|
| What | Community Go binary, <https://github.com/thereisnotime/sdrangel-mcp>, MIT | ~100 lines of Python in 7.4–7.6 below |
| Status | v0.1.0 (2026-09-20), tested against SDRangel **7.27.2**, Ollama/local-model guide included | Syntax-checked only, field names unverified |
| Tools | 93 (whole API) → **filter to ~15 in Hermes** | 11, hand-picked |
| Risk | Includes destructive tools (`stop_instance`, `delete_*`, `set_location`, import/export) → **exclude them** | Receive-only by design |

Of the options in the gist, *SDRangel + Ollama + Node-RED* is heavier than you need, *gr-mcp* needs a GNU Radio install (painful on Windows), and *AetherLink* is a hosted API service (data leaves your PC). Both routes here are local and offline.

### 7A. Install `sdrangel-mcp`

1. Download `sdrangel-mcp_0.1.0_windows_amd64.zip` from <https://github.com/thereisnotime/sdrangel-mcp/releases/latest> and extract to `C:\sdr\sdrangel-mcp\`.
   SHA-256 of the v0.1.0 zip: `1ac26c97bababa135fad9fbeb2ad52db06494288eb7a5828fd26e0bd5c2373b4` (check with `Get-FileHash`).
2. With SDRangel running, list the tools and try one:
   ```powershell
   cd C:\sdr\sdrangel-mcp
   $env:SDRANGEL_BASE_URL = "http://127.0.0.1:8091"
   .\sdrangel-mcp.exe tools          # prints all 93 tool names - pick your include list from this
   .\sdrangel-mcp.exe call <a get-instance/summary tool name from the list>
   ```
3. Register it in Hermes (`%LOCALAPPDATA%\hermes\config.yaml`, under `mcp_servers:`). Adjust the `include` names to what step 2 printed:
   ```yaml
     sdrangel:
       command: C:/sdr/sdrangel-mcp/sdrangel-mcp.exe
       args: ["serve"]
       env:
         SDRANGEL_BASE_URL: "http://127.0.0.1:8091"
       connect_timeout: 30.0
       timeout: 60
       enabled: true
       tools:
         # keep the model's tool list short: devices, channels, reports, presets, spectrum
         include: ["*instance*summary*", "*deviceset*list*", "*device*settings*", "*device*start*", "*device*stop*",
                   "*channel*add*", "*channel*settings*", "*channel*report*", "*channel*delete*",
                   "*preset*load*", "*preset*save*", "*spectrum*"]
         exclude: ["stop_instance", "set_location", "*import*", "*export*", "*workspace*", "*audio*", "*logging*"]
         prompts: false
         resources: false
   ```
   `include` wins when a name matches both lists, so keep `stop_instance`/`set_location` out of the include patterns.
4. `hermes mcp test sdrangel` → should list only the included tools. In a session, `/reload-mcp` if Hermes was already open. Tools appear as `mcp_sdrangel_<tool>`.
5. Decoder logs: `sdrangel-mcp` doesn't read files. Hermes's own file tools can read `C:\sdr\logs\`, or add the two `read_decoder_log` tools from 7B as a second tiny server.

### 7B. Fallback: your own Python server

### 7.1 Create the environment (mirrors your blender-mcp layout)
```powershell
mkdir $env:LOCALAPPDATA\sdrangel-mcp; cd $env:LOCALAPPDATA\sdrangel-mcp
python -m venv .venv
.\.venv\Scripts\pip install "mcp[cli]" httpx
mkdir C:\sdr\logs -Force
```

### 7.2 `server.py`
```python
"""Thin MCP bridge from Hermes to the SDRangel REST API (receive-only)."""
from pathlib import Path

import httpx
from mcp.server.fastmcp import FastMCP

API = "http://127.0.0.1:8091/sdrangel"
LOG_DIR = Path(r"C:\sdr\logs").resolve()
MIN_HZ, MAX_HZ = 24_000_000, 1_766_000_000  # R820T2 tuning range

http = httpx.Client(base_url=API, timeout=10)
mcp = FastMCP("sdrangel")


def _call(method: str, path: str, json: dict | None = None):
    r = http.request(method, path, json=json)
    r.raise_for_status()
    return r.json() if r.content else {"ok": True}


@mcp.tool()
def status() -> dict:
    """SDRangel instance summary and open device sets (call this first)."""
    return {"instance": _call("GET", ""), "devicesets": _call("GET", "/devicesets")}


@mcp.tool()
def get_device_settings(deviceset: int = 0) -> dict:
    """Current RTL-SDR settings: centre frequency (Hz), gain (tenths of dB), sample rate, ppm."""
    return _call("GET", f"/deviceset/{deviceset}/device/settings")


@mcp.tool()
def tune(center_hz: int, gain_tenths_db: int | None = None, sample_rate: int | None = None,
         ppm: int | None = None, deviceset: int = 0) -> dict:
    """Retune the RTL-SDR. center_hz must be within 24-1766 MHz. Gain is in tenths of dB (e.g. 402 = 40.2 dB)."""
    if not MIN_HZ <= center_hz <= MAX_HZ:
        raise ValueError(f"{center_hz} Hz is outside the R820T2 range")
    s = {"centerFrequency": center_hz}
    if gain_tenths_db is not None:
        s["gain"] = gain_tenths_db
    if sample_rate is not None:
        s["devSampleRate"] = sample_rate
    if ppm is not None:
        s["loPpmCorrection"] = ppm
    body = {"deviceHwType": "RTLSDR", "direction": 0, "rtlSdrSettings": s}
    return _call("PATCH", f"/deviceset/{deviceset}/device/settings", body)


@mcp.tool()
def run_device(start: bool = True, deviceset: int = 0) -> dict:
    """Start (True) or stop (False) streaming from the dongle."""
    return _call("POST" if start else "DELETE", f"/deviceset/{deviceset}/device/run")


@mcp.tool()
def list_rx_channel_types() -> dict:
    """Receive channel plugins available (demodulators/decoders) and their type IDs."""
    r = http.get("/channels", params={"direction": 0})
    r.raise_for_status()
    return r.json()


@mcp.tool()
def add_channel(channel_type: str, deviceset: int = 0) -> dict:
    """Add a receive channel, e.g. 'NFMDemod', 'AMDemod', 'ADSBDemod', 'AISDemod'."""
    return _call("POST", f"/deviceset/{deviceset}/channel", {"channelType": channel_type, "direction": 0})


@mcp.tool()
def set_channel_offset(channel_index: int, channel_type: str, offset_hz: int, deviceset: int = 0) -> dict:
    """Move a channel within the passband: offset_hz is relative to the device centre frequency."""
    body = {"channelType": channel_type, "direction": 0,
            f"{channel_type}Settings": {"inputFrequencyOffset": offset_hz}}
    return _call("PATCH", f"/deviceset/{deviceset}/channel/{channel_index}/settings", body)


@mcp.tool()
def channel_reports(deviceset: int = 0) -> dict:
    """Live reports for all channels in the set (channel power dB, squelch, decoder stats...)."""
    return _call("GET", f"/deviceset/{deviceset}/channels/report")


@mcp.tool()
def list_decoder_logs() -> list[str]:
    """Decoder output files available in C:\\sdr\\logs."""
    return sorted(p.name for p in LOG_DIR.iterdir() if p.is_file())


@mcp.tool()
def read_decoder_log(filename: str, last_lines: int = 50) -> str:
    """Tail of a decoder log (CSV/JSONL) from C:\\sdr\\logs. Max 500 lines."""
    p = (LOG_DIR / filename).resolve()
    if p.parent != LOG_DIR or not p.is_file():
        raise ValueError("file must be inside C:\\sdr\\logs")
    lines = p.read_text(errors="replace").splitlines()
    return "\n".join(lines[-min(last_lines, 500):])


if __name__ == "__main__":
    mcp.run()
```
> Before relying on `tune`/`set_channel_offset`, compare the field names against what step 4.7's `GET …/device/settings` and the Swagger UI return for your build. Adjust them if anything differs. Channels are always added with `direction: 0` (Rx), so the agent can't add transmit modulators.
### 7.3 Register it in Hermes
Add to `mcp_servers:` in `%LOCALAPPDATA%\hermes\config.yaml`, next to `blender`:
```yaml
  sdrangel:
    command: C:/Users/Neo/AppData/Local/sdrangel-mcp/.venv/Scripts/python.exe
    args: ["C:/Users/Neo/AppData/Local/sdrangel-mcp/server.py"]
    connect_timeout: 60.0
    timeout: 60
    enabled: true
```
Then, with SDRangel running:
```powershell
hermes mcp test sdrangel      # should list the tools
hermes                         # in a session: /reload-mcp if it was already open
```
Tools appear in Hermes as `mcp_sdrangel_<tool>` (e.g. `mcp_sdrangel_tune`).

---

## 8. Hermes skill: `sdr-analyst`

Create `%LOCALAPPDATA%\hermes\skills\sdr-analyst\SKILL.md`:

```markdown
---
name: sdr-analyst
description: "Operate SDRangel via MCP and explain/identify radio signals received by a NESDR Mini 2 (24–1766 MHz)."
version: 0.1.0
author: Neo
platforms: [windows]
---
# SDR analyst

Hardware: NooElec NESDR Mini 2 (RTL2832U + R820T2), receive-only, 24–1766 MHz, ~2.4 MS/s max,
no TCXO (expect ppm drift), no HF without an upconverter.

## Workflow
1. Always call `mcp_sdrangel_status` and `get_device_settings` before changing anything.
2. State the planned change (frequency, gain, channel) in one line before calling `tune`/`add_channel`.
3. After tuning, wait a few seconds, then read `channel_reports` and/or `read_decoder_log`.
4. When identifying a signal: give the band allocation, likely service, modulation, typical
   bandwidth, and which SDRangel demodulator decodes it. Cite sigidwiki.com where possible.
   Say how confident you are; never present a guess as fact.
5. For screenshots of the waterfall: describe bandwidth, duty cycle, burst structure,
   and centre offset before guessing what the signal is.

## Rules
- Never attempt to decrypt encrypted traffic. Don't repeat or summarise the content of private
  communications (phone calls, private dispatch); describe the signal type instead.
- Keep log reads to the last 50–200 lines.
```

Add a `references/` folder with your own notes as you learn: local frequencies from RadioReference, your calibrated ppm, and what you've already identified. The skill gets smarter over time.

---

## 9. Learning missions (in order of difficulty)

Region matters: you'll want both the US and EU values until you know which apply.

| # | Target | Frequency | SDRangel channel | Ask Hermes |
|---|---|---|---|---|
| 1 | FM broadcast + RDS | 88–108 MHz | Broadcast FM Demod | "What's RDS and what is this station sending?" |
| 2 | Aircraft ADS-B | 1090 MHz | ADS-B Demod (+ Map) | "Summarise the last 200 aircraft log lines: altitudes, airlines, routes." |
| 3 | Airband voice | 118–137 MHz (AM) | AM Demod | "Survey this rtl_power CSV; which are tower/approach/ATIS?" |
| 4 | NOAA Weather Radio (US) | 162.400–162.550 MHz | NFM Demod | Calibrate ppm against it |
| 5 | Ships AIS | 161.975 / 162.025 MHz | AIS Demod | "Explain these NMEA sentences." |
| 6 | APRS packet | 144.390 (US) / 144.800 (EU) MHz | Packet Demod | "Decode the APRS paths and positions." |
| 7 | ISM sensors | 433.92 / 315 / 868 / 915 MHz | rtl_433 (JSON) | "What devices are around me, and how strong?" |
| 8 | Weather balloons | 400–406 MHz | Radiosonde Demod | "Plot/explain the sonde's ascent data." |
| 9 | Pagers | e.g. 929–932 MHz (US) | Pager Demod (POCSAG) | Signal structure only (see rules) |
| 10 | Digital voice ID | various | DSD Demod (DMR/D-STAR/YSF/dPMR) | Screenshot → "What mode is this?" |
| 11 | FT8 on 6 m / 2 m | 50.313 / 144.174 MHz (USB) | FT8 Demod | "Decode these calls/grids, how far away?" |
| 12 | DAB radio (EU/UK/AU) | Band III 174–240 MHz | DAB Demod | "What's in this multiplex?" |
| 13 | Unknown signal hunt | anything | Waterfall screenshot → vision | "Identify this; compare with sigidwiki." |

> NOAA-15/18/19 APT weather satellites were retired in 2025. For satellite imagery, look at Meteor-M LRPT (137 MHz) with **SatDump** instead.

---

## 10. Troubleshooting

| Symptom | Fix |
|---|---|
| SDRangel shows no RTL-SDR | Zadig WinUSB on **Interface 0**. Another app has the dongle (close rtl_433/SDR#). Try another port and re-run Zadig. |
| `usb_open error -12` | WinUSB went onto Interface 1. Re-run Zadig on Interface 0. |
| `PLL not locked` / tuning errors near band edges | Normal near the R820T2 limits. Stay within ~25–1750 MHz. |
| Strong signals repeated across the band | Gain too high, or FM broadcast overload. Lower the gain, add an FM-notch filter. |
| Signal is offset/drifts | Recalibrate ppm after warm-up (step 4.5). |
| Dropped samples / stutter | Lower the sample rate to 2.048 or 1.024 MS/s. Use a rear USB port. Avoid hubs. |
| Black waterfall | OpenGL/GPU selection (step 4.1). |
| Hermes can't see tools | SDRangel must be running *before* the tool call. Check `hermes mcp test sdrangel`, then `/reload-mcp`. |
| Hermes ignores images | `supports_vision: true` plus llama-server started with `--mmproj`. Check `/props` → `modalities.vision`. |

---

## 11. Legal and etiquette

Receiving is broadly legal in many countries but **not everywhere, and not for everything**. For example, in the US, intercepting cellular calls is illegal and you may not divulge or use the contents of private communications. The UK and some EU countries restrict listening to non-broadcast traffic at all. Never attempt to decrypt encrypted signals. Check your local rules. The `sdr-analyst` skill above already steers the agent toward describing signals rather than repeating private content.

---

## Checklist

- [x] Zadig: WinUSB on `Bulk-In, Interface (Interface 0)` (0BDA 2838 00) — done 2026-10-09
- [x] `rtl_test -t` finds the R820T; ppm measured **+19** (`rtl_test -p`)
- [x] SDRangel 7.27.2 installed, RTL-SDR streaming
- [x] FM working with RDS (100.6 MHz, PI C2A1). ADS-B: no decodes at night — retry in daylight
- [ ] Decoder logs → `C:\sdr\logs` (folder exists; enable log-to-file in each decoder)
- [x] REST API answers on `127.0.0.1:8091`; field names verified (`rtlSdrSettings.centerFrequency`, `gain` ×10, `inputFrequencyOffset`)
- [ ] llama-server relaunched with `--mmproj`, and `supports_vision: true` is set
- [x] `sdrangel-mcp` v0.1.0 in `C:\sdr\sdrangel-mcp`, registered in Hermes (25 tools selected), `hermes mcp list` shows ✓ enabled — done 2026-10-10
- [x] `sdr-analyst` skill created at `%LOCALAPPDATA%\hermes\skills\sdr-analyst\SKILL.md`
- [ ] Missions 1–13 (mission 1 done)

---

## Sources
- NESDR Mini 2 product page: <https://www.nooelec.com/store/sdr/sdr-receivers/nesdr-mini-2.html>
- NooElec NESDR support article (returned 403 to automated fetch, so the driver steps come from the sources below): <https://support.nooelec.com/hc/en-us/articles/360005805834-NESDR-Series>
- RTL-SDR on Windows 10/11 / Zadig interface notes: <https://www.rtl-sdr.com/getting-the-rtl-sdr-to-work-on-windows-10/>, <https://learn.adafruit.com/getting-started-with-rtl-sdr-and-sdr-sharp/driver-fix>
- SDRangel: <https://www.sdrangel.org>, <https://github.com/f4exb/sdrangel>, releases <https://github.com/f4exb/sdrangel/releases>
- SDRangel REST API: <https://github.com/f4exb/sdrangel/wiki/Web-(http)-REST-API>, swagger spec `swagger/sdrangel/api/swagger/swagger.yaml`
- SDR ↔ MCP comparison gist: <https://gist.github.com/danbri/b0c04702ed7a6a303d67c911cde4497a>
- Hermes Agent docs: install <https://hermes-agent.nousresearch.com/docs/getting-started/installation>, MCP <https://hermes-agent.nousresearch.com/docs/user-guide/features/mcp>, local models <https://hermes-agent.nousresearch.com/docs/user-guide/local-models>
- Qwen3.8-27B (released 2026-08-14, dense VLM, 262K ctx): <https://artificialanalysis.ai/models/qwen3-8-27b>, <https://simonwillison.net/2026/Aug/16/qwen-38-27b/>
- rtl-sdr-blog drivers/tools: <https://github.com/rtlsdrblog/rtl-sdr-blog/releases>; rtl_433: <https://github.com/merbanan/rtl_433>
- Signal Identification Wiki: <https://www.sigidwiki.com>
