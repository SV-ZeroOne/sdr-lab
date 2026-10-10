# SDR Station: Current Setup and Lessons Learnt

Living reference. Update this when the setup or understanding changes.
Last updated: **2026-10-10 16:50 (BST)**.

Companion files:
- [NESDR_Mini2_SDRangel_Hermes_Plan.md](NESDR_Mini2_SDRangel_Hermes_Plan.md): the original step-by-step guide and missions list.
- [NESDR_SDRangel_Hermes_Setup_Record.md](NESDR_SDRangel_Hermes_Setup_Record.md): as-built record of the first install (driver, paths, Hermes config verbatim).

```
 stock whip ─MCX─► NESDR Mini 2 (RTL-SDR) ──┐                 R0: FM broadcast 100.6 MHz (muted)
                                            ├─ SDRangel 7.27.2 ─ R1: HackRF airband 118–121 MHz (4 × AM, Director 120.400 live)
 ANT500 (sill) ─SMA─► HackRF One ───────────┘        │ REST :8091
                                                     ▼
                                    sdrangel-mcp.exe (25 tools) ◄── Hermes Agent ◄── Qwen3.8-27B @ :8080
```

**Location:** London area, under the Heathrow approach (ADS-B positions 51.3–51.6 N, 0.25 W–0.2 E; aircraft at 2,000–6,000 ft overhead).

---

## 1. Current state

### Receivers
| | NESDR Mini 2 (RTL-SDR) | HackRF One |
|---|---|---|
| SDRangel device set | R0 | R1 |
| USB ID / driver | 0BDA:2838, WinUSB on Interface 0 (Zadig) | 1D50:6089, WinUSB already present (no Zadig needed) |
| Range | ~24–1766 MHz | 1 MHz–6 GHz (HF possible) |
| ADC | 8-bit, ≤2.4 MS/s | 8-bit, 2–20 MS/s |
| Crystal error | **+19 ppm** (measured) | not yet measured (0 entered) |
| Transmit | no | **yes: never add Tx device sets/channels** (UK licence required) |
| Antenna | stock telescopic whip (MCX), on the desk | **ANT500** (SMA, telescopic 20–88 cm, 75 MHz–1 GHz), HackRF itself on the **window sill** |

### ANT500 lengths (quarter-wave)
| Band | Length |
|---|---|
| FM broadcast 100 MHz | ~71 cm |
| Airband 118–137 MHz | ~57–60 cm |
| 2 m amateur 145 MHz | ~49 cm |
| Marine / AIS 156–162 MHz | ~45 cm |
| 70 cm / PMR446 / 433 ISM | ~16 cm → use fully collapsed (20 cm) |
| ADS-B 1090 MHz | ~6.9 cm → ANT500 is too long; use antenna #6 instead |

### Other antennas owned (AliExpress kit)
| # | What it really is | Best use |
|---|---|---|
| 1 | Wi-Fi paddle 2.4/5 GHz | none here |
| 2 | Mag-mount LTE whip (~800–2100 MHz) | spare for ADS-B |
| 4 | Telescopic, same class as the ANT500 | spare; fits the NESDR with an SMA→MCX adapter |
| 5 | Mag-mount 40–860 MHz whip, ~30 cm, 3 m cable | relocation; electrically short on FM (-7 dB vs ANT500) |
| 6 | Mag-mount LTE whip with coil, 700–2700 MHz, 3 m cable | **ADS-B, best 1090 MHz antenna owned (not yet tried)** |

### Accessories: external LNA (owned, not yet used)
**OpenSourceSDRLab 50 MHz–6 GHz low-noise amplifier**, 20 dB, micro-USB powered ([product page](https://opensourcesdrlab.com/products/50m-6ghz-low-noise-rf-amplifier-ultra-wideband-gain-20db-microusb-power-supply)).

| Spec (from the listing) | Value |
|---|---|
| Range / gain | 50–6000 MHz, **20 dB** (no flatness curve published) |
| Noise figure | not published ("small") |
| Output P1dB | +20 to +21 dBm (listing contradicts itself) |
| Max input | **not published**: treat as fragile, keep away from transmitters |
| Power | **5 V micro-USB, 85 mA**, not bias-tee powered |
| Connectors | SMA, 50 Ω, one SMA cable included. IN/OUT orientation not stated on the listing: **read the board silkscreen** |
| Build | shielded enclosure, reverse-polarity protection diodes, 5 g |

How to use it on this rig:
- **Chain:** antenna → LNA **IN** … LNA **OUT** → (coax) → SDR. Put it **at the antenna end** of the 3 m mag-mount cable: an LNA's job is to overcome cable loss and the receiver's noise figure, so it must come *before* the loss.
- **Where it should help:** the **HackRF** (its noise figure is ~10+ dB with its own amp off, so an external low-NF stage beats the built-in RF amp, which gave nothing on ADS-B) and **long or thin coax at UHF/1090 MHz**. It helps the RTL-SDR less: the R820T2 is already fairly quiet, so mainly the cable loss gets recovered.
- **Where it will hurt:** **FM broadcast**. The signals are already strong (the RTL overloads above ~33 dB internal gain), and +20 dB in front guarantees overload. Because it's **wideband**, FM broadcast also passes through when tuned elsewhere (1090, 433…), so watch for a rising floor or ghost signals. If that happens, the cure is an **FM band-stop filter (88–108 MHz)** before the LNA.
- **When it's in the chain:** HackRF **RF amp off** (`lnaExt=0`) and start the HackRF LNA ~16 dB lower than the recipes in this file (e.g. ADS-B LNA 24 instead of 40), then raise until the floor lifts a few dB. RTL: start ~10–15 dB lower gain.
- **Bias-tee off** on the HackRF (`biasT=0`): it's USB-powered, and the NESDR Mini 2 has no bias-tee anyway.
- **Never transmit through it.** It's one-way, and HackRF Tx into its output can destroy it (another reason the rig is Rx only).
- **Power:** the PC's USB is fine (85 mA). If new evenly spaced spurs appear with it connected, try a USB power bank to rule out USB/ground noise.
- **Best first experiment:** antenna #6 (sill) → LNA → 3 m cable → HackRF on ADS-B, A/B against the same without the LNA, judged by **aircraft count over 60 s** (baseline: HackRF + ANT500 ~5, RTL + whip 11–12).

### Live configuration at last check (16:47)
**R0 RTL-SDR → FM broadcast listening** (tuned via API 2026-10-10 17:00): centre 100,000 kHz, **gain 25.4 dB**, AGC off, LO ppm 19, DC block on. BFM Δf +600,000 → **100.6 MHz (Classic FM)**, RF BW 200 kHz, AF BW 15 kHz, de-emph 50 µs, **stereo on**, RDS on, squelch -60, volume 4, unmuted. Pilot locked 100 % at -17.5 dB.
Station survey at gain 29.7 (pwr / stereo pilot): 100.0 -15 / locked · **100.6 -16 / locked** · 99.8 -16 / unreliable lock · 100.5 -17 / none · 99.2 -25 · 100.9 -26 · 101.1 -27 (mono only). Gain sweep on 100.6: 22.9–29.7 all 100 % stereo lock, 25.4 best RDS; 33.8 starts losing lock (88 %) = overload onset.

**R1 HackRF → Airband** (working):
| Setting | Value |
|---|---|
| Centre / SR / BBF | **120.000 MHz** / 4 MS/s / 3.5 MHz |
| LNA / VGA / RF amp | **40 / 32 / off** |
| DC block / IQ corr / bias-T | on / on / off |
| AM channels (RF BW 6 kHz, AF BW 3 kHz, squelch -38 dB) | ch0 Tower **118.500** (muted) · ch1 Tower **118.700** (muted) · ch2 Director **119.725** (muted) · ch3 Director **120.400** (**unmuted**) |
| Result | transmissions open the squelch at -18 to -22 dB over a -46 dB floor |

Recall values for the other HackRF setups:
- **FM 100.0 MHz**: centre 99,400 kHz, Δf +600,000, LNA 8 / VGA 32, BFM RF BW 200 kHz, AF BW 12 kHz, mono, RDS on.
- **FM 106.2 MHz**: centre 106,000 kHz, Δf +200,000.
- **ADS-B**: centre 1,090 MHz, 2.4 MS/s, **LNA 40 / VGA 32**, amp off.
- **70 cm listening**: centre 433,900 kHz, LNA 40 / VGA 40, NFM 12.5 kHz on 433.500 (simplex calling), squelch -30.

### Software and agent
- SDRangel 7.27.2, REST API `http://127.0.0.1:8091/sdrangel` (no auth; localhost only).
- `C:\sdr\sdrangel-mcp\sdrangel-mcp.exe` (v0.1.0) registered in Hermes as `sdrangel` with a 25-tool whitelist (`hermes mcp list` → `25 selected ✓`). Skill: `%LOCALAPPDATA%\hermes\skills\sdr-analyst\SKILL.md`.
- Qwen3.8-27B on llama-server :8080 (both R9700s); **vision still off** (`--no-mmproj`).
- Tools: `C:\sdr\rtl-sdr\` (rtl_test, rtl_power, …). Logs: `C:\sdr\logs\` (nothing logging yet).

### Mission status
| # | Mission | Status |
|---|---|---|
| 1 | FM broadcast + RDS | audio ✓ both radios; RDS decoded once (RTL, 00:29), not in daytime |
| 2 | ADS-B | ✓ RTL-SDR 11–12 aircraft (daylight); HackRF + ANT500 ~5 |
| 3 | Airband voice | ✓ **Heathrow Director 120.400 / Tower 118.500** on the HackRF |
| — | 2 m / 70 cm / PMR446 | scanned, quiet; Frequency Scanner set up for 70 cm |
| 5–13 | AIS, APRS, rtl_433, radiosondes, pagers, DAB, FT8, unknown-signal ID | not started |

---

## 2. Lessons learnt

### Hardware and drivers
1. **Zadig must target Interface 0** (`0BDA 2838 00`). Interface 1 gives `usb_claim_interface error -12`. Re-run after changing USB port or a Windows feature update.
2. `rtl_test -t` ending in "No E4000 tuner found" is normal on an R820T. `rtl_test -p` on Windows prints bogus `sample rate: 0 / -1000000` pairs; trust the **cumulative** ppm when it's steady (ours: +19).
3. "lost at least N bytes" at ~100 bytes per 4 MB is 1 sample per million: ignore. Moving to a rear motherboard port changed nothing.
4. **Only one program can own a dongle at a time.** Two different dongles can run together in SDRangel as separate device sets.
5. The HackRF enumerated with a working driver straight away on this PC. A USB cable swap made no difference to reception.

### Antennas and placement
6. **Location beats antenna beats gain.** Moving the HackRF + ANT500 from the desk to the window sill gave +2.5 dB at equal gain and the best RDS of the day; swapping antennas or gains moved things far less.
7. **Your body is part of the antenna at VHF.** Holding the mag-mount swung channel power by 10 dB.
8. **Don't put foil under a mag-mount.** It cut signal 15–22 dB (foil touching the whip or feed point). Lifting the base off restored it instantly. A proper ground plane must not touch the radiator.
9. Short whips (mag-mount #5, ~30 cm) are ~7 dB down on FM vs the ANT500 at 71 cm; one extra LNA step (+8 dB) buys it back, but SNR is not improved.
10. Collapse/extend the ANT500 per band (table above); a mismatched length explained a 15 dB deficit on FM.

### Receiving FM / RDS
11. **Audio sounding fine ≠ RDS working.** RDS lives at 57 kHz in the multiplex and is ~25 dB weaker than the audio; it fails first.
12. **Overload kills RDS.** RTL-SDR at 42–44 dB on FM broadcast → RDS Dem 5 %. 23–33 dB is the FM range. HackRF at the sill clips at LNA 16 (-10 dB channel power); best there was LNA 8 / VGA 32.
13. **RF BW must be ≥ 200 kHz** for RDS (180 kHz default clips it; 250 max). AF BW 10–12 kHz and **mono** cut hiss. Stereo costs ~20 dB of SNR; mono is right for a 30 dB signal.
14. **Stereo on/off does not affect RDS** in SDRangel: the pilot PLL runs whenever RDS is enabled (`bfmdemodsink.cpp`, confirmed live). The pilot reading only shows a value when stereo is on.
15. **Dem % = clean RDS bits, Dec % = groups passing CRC.** Need Dem ≳ 80 % before anything decodes. PSN (station name) needs 4 G0 groups; "Light Classics M" is the PTY, not the name. PI `C2A1` at 100.6 MHz = UK national = almost certainly Classic FM.
16. **De-emphasis 50 µs** in the UK/EU (75 µs is the Americas).
17. **Gain cannot fix multipath or noise.** Three gain sweeps across three antenna/position setups (24 combinations) all landed at RDS Dem 2–9 % with the pilot rock-solid at -17.5 dB. The only successful RDS decode was at night (00:29). Leading hypothesis: a **daytime noise source** or reflections at this location → re-test after dark before buying anything.
18. **The HackRF radiates its own spurs**: evenly spaced vertical lines in its waterfall; carriers on exact whole MHz (128.000, 130.000). Keep it away from other antennas. Not yet proven to be what broke the RTL's RDS.
19. HackRF: keep the channel **off the device centre** (DC spike), e.g. centre 99.4 + Δf 600 kHz for 100.0 MHz. RF amp stays off on FM.
20. UK FM stations sit on 100 kHz steps; dragging the channel marker lands on odd offsets (e.g. +219,472). Type Δf instead.
21. A wobbling narrow carrier near 99.6 MHz is local interference (switching PSU / monitor), not a station.

### ADS-B
22. **Nothing at 23:30, 11–12 aircraft at 15:18** on the RTL-SDR: same dongle, same whip. Daylight traffic volume and placement matter more than settings. RTL gain 42–49.6, whip collapsed.
23. **HackRF ADS-B**: the shipped ADS-B configuration loads **LNA 16 / VGA 16 → zero aircraft**. LNA 40 / VGA 32 → 4–5 aircraft per minute with the ANT500 (far too long for 1090 MHz). The **RF amp (+14 dB) gave no improvement** (4 vs 5 aircraft); leave it off.
24. `channelPowerDB` ~-29 to -37 dB on ADS-B is normal (bursty signal); judge by aircraft count, not power.
25. SDRangel 7.27.2's ADS-B map shows "API KEY REQUIRED" (CARTO requires keys since 2026-09-25): no key field exists yet ([issue #2968](https://github.com/f4exb/sdrangel/issues/2968)). Use the Street map type, or a MapTiler key for Satellite. Set **Preferences → My Position**.

### Airband (AM voice) — works
26. **Aircraft overhead are the strong signals, not the ground stations.** Heathrow Director 120.400 peaked +30 dB (active 9 % of a 90 s window), Tower 118.500 +10 dB (3 %). ATIS 128.075 and London VOLMET 128.600 were **not receivable** here (1 dB margin), even though they transmit continuously.
27. Working settings: HackRF LNA 40 / VGA 32, amp off; AM demod **RF BW 6 kHz, AF BW 3 kHz, squelch -38 dB** (floor ~-46 dB, voice -18 to -22 dB).
28. **UK charts use 8.33 kHz channel names**: "118.505" = 118.500 MHz, "118.705" = 118.700, "119.730" = 119.725, "120.405" = 120.400, "128.080" = 128.075 (ATIS). Tune the real frequency.
29. Heathrow frequencies used (real MHz): Tower 118.500 / 118.700, Director 119.725 / 120.400 / 134.975, Ground 121.900, Delivery 121.975, ATIS 128.075, London VOLMET South 128.600. One HackRF window (centre 120.0, 4 MS/s) covers 118.25–121.75.

### Voice bands and scanning
30. **UK emergency services (police/fire/ambulance) are on Airwave = encrypted TETRA.** Nothing to find; never attempt to decrypt.
31. Legal-to-listen voice: amateur 2 m (144–146) / 70 cm (430–440), PMR446 (446.0–446.2), CB (27 MHz). Airband and marine are a grey area that is universally practised; **WT Act 2006 s.48 makes it an offence to disclose or act on what you hear.** Business radio (450–470, often DMR): identify, don't eavesdrop.
32. One-shot power sweeps (0.35 s/step) found 2 m quiet, 70 cm only ISM chirps (433.875, 434.70–434.79 = key fobs/sensors), PMR446 silent. **Bursty voice needs continuous monitoring**, not a sweep.
33. **Hits on the exact first/last step of a 4 MHz window (144.000, 436.000, 440.000) are HackRF filter-edge artefacts**, like the DC spike at the centre.
34. **Multi-channel activity monitor** (best technique found): several demods in one device set, all muted, poll `GET /deviceset/N/channels/report` every 0.3 s for 60–90 s, compare each channel's peak vs median power and % of samples above median+6 dB, then unmute the busiest. `AMDemodReport.squelch` = 1 while a transmission is open.
35. SDRangel's **Frequency Scanner** channel exists in 7.27.2, but its frequency list is GUI-only; the REST schema (`FreqScannerSettings`) only exposes `threshold`, `channelBandwidth`, `channelFrequencyOffset`, `channelShift`.
36. Weak-signal gain on the HackRF: at 2 m / 70 cm / airband the floor rose dB-for-dB with gain (LNA 32/VGA 32 -45 dB → LNA 40/VGA 40 -30 dB), so receiver noise dominates and high LNA (40) is right. Opposite of FM broadcast, where LNA 8–16 is the limit.

### SDRangel UI
37. Example configurations: Preferences → Configurations. **Loading a configuration replaces the whole workspace** (that's how the NESDR device set vanished at 16:20). Presets (per station): star ★ in the *device window* title bar.
38. Device start button: green square = running. Rolled-up subwindows: Tile button or double-click the title bars. The Frequency Display feature must be pointed at the right channel.
39. The "3D" button in a baseband spectrum turns it into a colour-band view that hides the 19/57 kHz structure; keep it off when diagnosing.
40. Two demods playing at once mix their audio; mute the one you're not listening to.

### REST API / automation (what Hermes must know)
41. Keys are plugin-specific and must be echoed: `rtlSdrSettings`, `hackRFInputSettings`, `BFMDemodSettings`, `NFMDemodSettings`, `AMDemodSettings`, `ADSBDemodSettings`; reports `BFMDemodReport`, `NFMDemodReport`, `AMDemodReport`, `ADSBDemodReport`.
42. Units: RTL `gain` in **tenths of dB** (297 = 29.7). HackRF `lnaGain` 0–40 step 8, `vgaGain` 0–62 step 2, `lnaExt` = RF amp, `LOppmTenths`. Frequencies in Hz; `inputFrequencyOffset` is relative to the device centre.
43. **A PATCH can return HTTP 200 and not apply** (seen once on the HackRF retune). Always GET the settings back after a change.
44. **Device-set indices shift** when a configuration is loaded or a device set is removed (HackRF went from R1 to R0 and back). Look the index up by `hwType` from `GET /devicesets` before every session; never hard-code it.
45. Channel add returns **HTTP 202** (accepted, asynchronous); wait ~2 s before patching the new channel.
46. RDS counters are noisy; sample 5–8 times over 20–30 s and look at the trend. The ADS-B report contains the full aircraft list (`aircraftState[]` with icao, callsign, altitude, groundSpeed, lat/lon), so one call = a live traffic snapshot.
47. Device run control: `POST/DELETE /deviceset/N/device/run`; `GET` returns `state` (idle/running). A device can be left **idle** after loading a configuration; check before measuring.
48. PowerShell gotcha: variables are case-insensitive (`$b` overwrote `$B` and broke a polling loop). Use distinct names.

### Agent / LLM
49. Give the local model a **short whitelist** of tools (25 of 93) and an explicit "get → patch with key → read back" workflow; Qwen3.8 over-thinks, so keep tuning prompts specific.
50. The HackRF can transmit; the whitelist has no Tx-only tools but `add_channel`/`add_device_set` can target Tx. Rule in the skill: Rx only (`direction: 0`).

---

## 3. Open items (priority order)
1. **Night RDS test**: leave the HackRF/RTL on 100.6 FM after dark and poll Dem %. If it jumps → daytime noise source to hunt; if not → reflections, needs an outdoor/loft antenna.
2. **Callsign ↔ voice matching with Hermes**: RTL-SDR on ADS-B, HackRF on airband; when a callsign is heard on 120.400, ask Hermes to find it in the ADS-B report (altitude, speed, position).
3. **Antenna #6 on ADS-B** (sill, no foil): compare aircraft count against the RTL's 11–12 and the HackRF/ANT500's ~5. Then add the **external 20 dB LNA** at the antenna end (HackRF amp off, LNA 24) and repeat the 60 s count (see *Accessories*).
4. **Vision**: relaunch llama-server with `--mmproj …\mmproj-BF16.gguf` (drop `--no-mmproj`), set `supports_vision: true` for `qwen38` in Hermes. Then waterfall screenshots become askable.
5. **Prove/disprove HackRF noise** on the RTL: RTL on 100.6 FM, stop the HackRF (`DELETE /deviceset/<hackrf>/device/run`), watch RTL Dem %.
6. **Measure HackRF ppm** against a known carrier; enter as LO ppm (`LOppmTenths`).
7. **Logging**: ADS-B log-to-file → `C:\sdr\logs\`.
8. Save presets: "FM 100.6 Classic FM" (RTL), "ADS-B 1090" (RTL), "HackRF Airband Heathrow" (4 × AM), "HackRF FM 100.0"; save the layout as a configuration.
9. Update `sdr-analyst` skill with the HackRF keys/gain ranges, airband frequencies and the device-index lookup rule.
10. Next missions: AIS on the Thames (161.975/162.025), APRS 144.800, rtl_433 at 433.92/868 MHz, radiosondes 400–406 MHz, DAB Band III, HF shortwave on the HackRF (1–30 MHz, needs a long wire).
11. `hermes update` pending; watch SDRangel for a release with a Carto key field.

---

## 4. Quick reference commands

```powershell
$base = "http://127.0.0.1:8091/sdrangel"

# which device set is which (indices can shift!)
(Invoke-RestMethod "$base/devicesets").deviceSets | % { "R$($_.samplingDevice.index) $($_.samplingDevice.hwType) $($_.samplingDevice.centerFrequency) $($_.samplingDevice.state)" }

# all channel reports for a device set
(Invoke-RestMethod "$base/deviceset/1/channels/report").channels | ConvertTo-Json -Depth 5

# retune the HackRF and read back
$body = '{"deviceHwType":"HackRF","direction":0,"hackRFInputSettings":{"centerFrequency":120000000,"lnaGain":40,"vgaGain":32}}'
Invoke-RestMethod -Method Patch -Uri "$base/deviceset/1/device/settings" -ContentType application/json -Body $body | Out-Null
(Invoke-RestMethod "$base/deviceset/1/device/settings").hackRFInputSettings

# add an AM channel and put it on Heathrow Tower 118.500 (centre 120.0)
Invoke-RestMethod -Method Post -Uri "$base/deviceset/1/channel" -ContentType application/json -Body '{"channelType":"AMDemod","direction":0}'
Start-Sleep 2
$ch = '{"channelType":"AMDemod","direction":0,"AMDemodSettings":{"inputFrequencyOffset":-1500000,"rfBandwidth":6000,"afBandwidth":3000,"squelch":-38}}'
Invoke-RestMethod -Method Patch -Uri "$base/deviceset/1/channel/0/settings" -ContentType application/json -Body $ch | Out-Null

# Hermes
hermes mcp list                    # sdrangel  25 selected  ✓ enabled
hermes mcp test sdrangel
C:\sdr\sdrangel-mcp\sdrangel-mcp.exe call get_instance_summary
```

## Sources
- SDRangel BFM demod source (pilot PLL / RDS independence): <https://github.com/f4exb/sdrangel/blob/master/plugins/channelrx/demodbfm/bfmdemodsink.cpp>
- SDRangel REST API: <https://github.com/f4exb/sdrangel/wiki/Web-(http)-REST-API>
- sdrangel-mcp: <https://github.com/thereisnotime/sdrangel-mcp>
- Map key issue: <https://github.com/f4exb/sdrangel/issues/2968>
