---
name: sdr-analyst
description: "Use when operating the SDR radio rig (tune, scan, decode, diagnose via SDRangel MCP/REST). Two Rx dongles: NESDR Mini 2 RTL-SDR + PortaPack H2 HackRF One, London under the Heathrow approach. Rx only."
version: 0.2.0
author: Neo
license: MIT
platforms: [windows]
---
# SDR analyst

## The setup (ground truth)
- SDRangel 7.27.2 (Windows 11). REST `http://127.0.0.1:8091/sdrangel` (no auth, localhost only),
  MCP `C:\sdr\sdrangel-mcp\sdrangel-mcp.exe` registered in Hermes as `sdrangel` (25-tool whitelist).
- **Two receivers, two device sets** (indices can shift — see rule below):
  - **NESDR Mini 2** (RTL-SDR, R820T2): ~24–1766 MHz, ≤2.4 MS/s stable, +19 ppm (`loPpmCorrection: 19`),
    stock MCX whip on the desk. No transmit.
  - **HackRF One** (inside the PortaPack H2; Mayhem/ESP32 coexists, radio is a normal HackRF):
    1 MHz–6 GHz, 2–20 MS/s, ANT500 on the window sill, ppm not yet measured (`LOppmTenths` 0).
    **It can transmit — Rx only, always `direction: 0`** (UK licence required to Tx).
- Location: London area, under the Heathrow approach (aircraft 2,000–6,000 ft overhead).
- Full living doc (current state, antenna table, open items, quick commands):
  `C:\Users\Neo\Desktop\Claude_Projects\SDR_Current_Setup_and_Lessons.md` — read it when unsure.

## API conventions (measured, do not guess)
- **Device-set indices shift** when a configuration is loaded/removed. Look the index up by
  `hwType` from `list_device_sets` / `GET /devicesets` every session; never hard-code R0/R1.
- Settings keys are plugin-specific and must be echoed in the PATCH:
  `rtlSdrSettings`, `hackRFInputSettings`; channel keys `<type>Settings`
  (`BFMDemodSettings`, `NFMDemodSettings`, `AMDemodSettings`, `ADSBDemodSettings`).
  Reports: `<type>Report` with `channelPowerDB` + decoder fields.
- Units: RTL `gain` in **tenths of dB** (297 = 29.7). HackRF `lnaGain` 0–40 step 8, `vgaGain`
  0–62 step 2, `lnaExt` = RF amp, `LOppmTenths`. Frequencies in Hz; `inputFrequencyOffset` is
  relative to device centre; keep every channel within ±(devSampleRate/2) of centre.
- **A PATCH can return HTTP 200 and not apply.** Always GET the settings back after a change.
- `add_channel` → HTTP **202** (async); wait ~2 s before patching the new channel.
- Device run control: `POST /deviceset/N/device/run` starts, `DELETE` stops; GET → `state`
  (idle/running). A loaded configuration can leave a device **idle** — check before measuring.
- **Never send a malformed device run/action body**: a bad body hangs the REST server and can
  take SDRangel down (user restarts the app to recover). Use the MCP `start/stop_device` tools
  or the exact `POST/DELETE …/device/run` shape.
- **Rx only.** Never add a device set or channel with `direction: 1`.

## Gain & signal recipes (measured on this rig)
- RTL FM broadcast: gain **23–33 dB**; 42–44 dB overloads and kills RDS.
- HackRF FM (sill): **LNA 8 / VGA 32**; LNA 16 clips. RF amp off.
- HackRF weak signals (2 m / 70 cm / airband / ADS-B): **LNA 40 / VGA 32–40** — receiver
  noise-limited, high gain is right (floor rises dB-for-dB with gain). Opposite of FM broadcast.
- ADS-B: RTL gain 42–49.6; HackRF LNA 40 / VGA 32, RF amp **off** (+14 dB gave no improvement).
- RDS: RF BW **≥ 200 kHz** (180 kHz default clips it), AF BW 10–12 kHz, **mono** (stereo costs
  ~20 dB SNR). Stereo on/off does **not** gate RDS — the pilot PLL runs whenever RDS is enabled;
  the pilot reading only shows when stereo is on. Dem% = clean bits, Dec% = groups passing CRC;
  need Dem ≳ 80% before anything decodes. PSN needs 4 G0 groups; PI `C2A1` @ 100.6 = Classic FM.
  De-emphasis 50 µs (UK/EU). RDS lives at 57 kHz ~25 dB under the audio: audio sounding fine ≠
  RDS working. On this site RDS is daytime-marginal (Dem 2–9% with pilot solid) — one decode at
  00:29; night-test pending before buying anything.
- ADS-B `channelPowerDB` −29…−37 dB is normal (bursty); **judge by aircraft count**, not power.
  The report's `aircraftState[]` (icao, callsign, altitude, groundSpeed, lat/lon) = live traffic
  snapshot in one call. Daylight: 11–12 aircraft (RTL) / ~5 (HackRF+ANT500); nothing at 23:30.
- Airband AM (working recipe): HackRF centre 120.0, 4 MS/s, LNA 40 / VGA 32, amp off; AMDemod
  RF BW 6 kHz, AF BW 3 kHz, squelch −38 (floor −46, voice −18…−22). Overhead **aircraft are the
  strong signals**, not ground stations; ATIS 128.075 / VOLMET 128.600 not receivable here.
  **UK charts use 8.33 kHz names**: "118.505" = 118.500 MHz, "120.405" = 120.400, "128.080" =
  128.075 — tune the real frequency. Heathrow: Tower 118.500/118.700, Director 119.725/120.400,
  Ground 121.900, Delivery 121.975. One 4 MS/s window at 120.0 covers 118.25–121.75.

## Scanning & monitoring techniques
- Wide band: step the device centre in sample-rate-sized windows, read `channelPowerDB` per step.
- Confirm any broadband hit with a **narrow** demod (NFM 12.5 kHz / AM 6 kHz) — leakage shows up
  in the wide filter only (400–440 MHz sweep: BFM saw bumps, only 423.75 survived 12.5 kHz).
- Hits on the **exact first/last step** of a window or at device centre = HackRF **filter-edge /
  DC artefacts**, not signals. Keep channels **off the device centre** (e.g. centre 99.4 + Δf
  600 kHz for 100.0 MHz). UK FM sits on 100 kHz steps — type Δf, don't drag the marker.
- **Bursty voice needs continuous monitoring, not a sweep.** Best technique = multi-channel
  activity monitor: several muted demods in one device set, poll `GET /deviceset/N/channels/report`
  every 0.3 s for 60–90 s, compare peak vs median power and % samples above median+6 dB per
  channel, unmute the busiest. `AMDemodReport.squelch` = 1 while a transmission is open.
- A continuous narrowband carrier (solid 40 s+, 12.5 kHz) = data/telemetry/beacon signature,
  not voice (voice = on/off PTT bursts). 423.75 MHz found this way (Ofcom licensed land-mobile data band).
- Frequency Scanner channel exists in 7.27.2 but its **frequency list is GUI-only**; REST exposes
  only `threshold`, `channelBandwidth`, `channelFrequencyOffset`, `channelShift`.
- Two demods playing at once mix audio — mute what you're not listening to (`audioMute=1` works
  via API). The 3D spectrum button hides the 19/57 kHz structure; keep off when diagnosing.

## Antenna & placement (measured)
- **Location beats antenna beats gain.** Desk → window sill = +2.5 dB at equal gain and the best
  RDS of the day; antenna/gain swaps moved things far less.
- **The body is part of the antenna at VHF** (holding a mag-mount swung power 10 dB).
- **Never foil under a mag-mount**: −15…−22 dB (foil touching whip/feedpoint); a ground plane
  must not touch the radiator.
- ANT500 (SMA, telescopic 20–88 cm, HackRF on sill) quarter-wave lengths: FM 100 MHz ~71 cm,
  airband 57–60, 2 m 49, AIS 45, 70 cm/PMR446/433 fully collapsed (20), too long for 1090 MHz.
  Antenna #6 (mag-mount LTE whip w/ coil, 700–2700 MHz, 3 m cable) = best 1090 candidate (untried).
  Short whip (#5, ~30 cm) is −7 dB on FM vs ANT500; gain buys level back, not SNR.
- The HackRF **radiates its own spurs** (even vertical lines in waterfall; carriers on exact whole
  MHz) — keep it away from the other antenna. Not yet proven to be what limits the RTL's RDS.
- Wobbling narrow carrier near 99.6 MHz = local interference (switching PSU/monitor), not a station.
- **External LNA owned** (OpenSourceSDRLab 50 MHz–6 GHz, +20 dB, 5 V micro-USB, not bias-tee
  powered, max input unpublished). You cannot see whether it's connected: **ask the user** before
  tuning. If it's in the chain: HackRF `lnaExt=0` (RF amp off), `biasT=0`, start HackRF `lnaGain`
  ~16 dB lower than the recipes above (ADS-B 24 not 40), RTL ~10–15 dB lower. Expect help on
  HackRF / UHF / 1090 MHz with long coax, **overload on FM broadcast**, and wideband FM leakage
  elsewhere (rising floor, ghosts → suggest an 88–108 MHz band-stop filter). Never transmit through it.

## Legal & safety
- **Receive-only. Never add Tx device sets/channels** (HackRF can transmit; UK licence required).
- UK emergency services = Airwave (encrypted TETRA): nothing to find, **never attempt to decrypt**.
- Legal-to-listen voice: amateur 144–146 / 430–440, PMR446 446.0–446.2, CB 27 MHz. Airband/marine
  = grey area, universally practised; **WT Act 2006 s.48: disclosing or acting on what you hear is
  an offence**. Business radio 450–470 (often DMR): identify, don't eavesdrop.
- For private/unknown communications: describe the signal type; don't repeat or summarise content.

## Workflow
1. `get_instance_summary` + `list_device_sets` (index lookup by hwType) + `get_device_settings` /
   `get_device_run_status` for the target before changing anything.
2. State the planned change in one line (freq, gain, channel type, offset) before `patch_*`.
3. PATCH with the settings key echoed; **GET back to confirm it applied**; wait a few seconds.
4. `get_channel_report` / `get_channels_report`; sample noisy counters (RDS, aircraft) 5–8× over
   20–30 s and report the trend, not a single sample.
5. When identifying a signal: give the UK/Ofcom band allocation, likely service, modulation,
   bandwidth, which SDRangel plugin decodes it, and your confidence. A guess is never a fact.
6. SDRangel 7.27.2 ADS-B map: CARTO now needs an API key (no field in this build, issue #2968) —
   use the Street map type; set Preferences → My Position.
7. UI gotchas: loading a **configuration replaces the whole workspace** (that's how the RTL
   device set vanished once); per-station presets = ★ in the device window title bar; device
   start button green square = running.
