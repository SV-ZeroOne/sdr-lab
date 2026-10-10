# sdr-lab

Software-defined radio notes and operating records for the NEO-X1 Pro rig —
NESDR Mini 2 (RTL-SDR) + HackRF One driven through SDRangel, with the
`sdrangel-mcp` bridge exposed to the Hermes agent.

| File | What it is |
|---|---|
| [SDR_Current_Setup_and_Lessons.md](SDR_Current_Setup_and_Lessons.md) | **Living doc** — the current working setup and hard-won lessons (ppm offset, antenna/noise, the sdrangel-mcp tool whitelist). Start here. |
| [NESDR_Mini2_SDRangel_Hermes_Plan.md](NESDR_Mini2_SDRangel_Hermes_Plan.md) | The original plan: wiring NESDR Mini 2 + SDRangel + the Hermes MCP bridge. |
| [NESDR_SDRangel_Hermes_Setup_Record.md](NESDR_SDRangel_Hermes_Setup_Record.md) | Dated setup record of what was actually done and verified. |
| [hermes-skills/sdr-analyst/SKILL.md](hermes-skills/sdr-analyst/SKILL.md) | Version-controlled copy of the Hermes `sdr-analyst` skill (rig facts, API conventions, gain recipes, legal rules). |

## Hermes skill sync

Hermes loads the **live** skill from `%LOCALAPPDATA%\hermes\skills\sdr-analyst\SKILL.md`; the copy
here is for version control. After editing either one, sync and commit:

```powershell
# live -> repo (after Hermes or you edit the live skill)
Copy-Item "$env:LOCALAPPDATA\hermes\skills\sdr-analyst\SKILL.md" .\hermes-skills\sdr-analyst\SKILL.md
# repo -> live (after editing the repo copy, e.g. on a fresh install)
Copy-Item .\hermes-skills\sdr-analyst\SKILL.md "$env:LOCALAPPDATA\hermes\skills\sdr-analyst\SKILL.md"
```

Split out of the `Claude_Projects` root on 2026-10-10 — a separate radio
project from the LLM inference rig, kept in its own repo.
