# SwitchBot Standing Fan — Home Assistant custom integration

> ⚠️ **Disclaimer:** This integration was built end-to-end using **[Claude Code](https://claude.com/claude-code)** (Anthropic) — including the BLE diagnostics that identified the device, the patches against `home-assistant/core`, the entity wiring, on-device testing, and this README. Treat it as community-quality AI-assisted work: it works on the maintainer's hardware (tested on a SwitchBot Standing Circulator Fan with RGB Lights, MAC `B0:E9:FE:CD:7B:CF`) and the code is plain-Python and reviewable, but it has not been through HA's core review process. PRs and issues welcome.


A drop-in replacement for Home Assistant's bundled `switchbot` integration that adds **full local Bluetooth control** for the **SwitchBot Standing Circulator Fan** (and any other Standing Fan model that identifies via service-data suffix `0x00 11 07 60`).

> **Why?** PySwitchbot recognises the Standing Fan and exposes the full control surface, but `homeassistant/core` itself does not list `STANDING_FAN` in `CONNECTABLE_SUPPORTED_MODEL_TYPES`. The config flow filters it out with *"No supported SwitchBot devices found in range"* — even though the library identifies the device perfectly. This integration fills that gap until upstream catches up.

## What it adds

When a Standing Fan is discovered, the following entities are created:

| Entity | Purpose |
|---|---|
| `fan.<name>` | Power, 9-step speed, 4 preset modes (Normal / Natural / Sleep / Baby) |
| `switch.<name>_horizontal_oscillation` | Horizontal sweep on/off |
| `switch.<name>_vertical_oscillation` | Vertical sweep on/off |
| `select.<name>_night_light` | Off / Soft / Bright |
| `sensor.<name>_battery` | Battery % |

All commands are sent **locally over BLE** via an active-scanning Bluetooth proxy — no cloud, no SwitchBot account, no SwitchBot Hub required.

## Requirements

- Home Assistant 2026.6+ (tested on 2026.6.4)
- An **active-scanning** BLE proxy within range of the fan. ESPHome BT proxies in `bluetooth_scanning_mode: active` work great. Passive-only adapters (e.g. Shelly Gen4) will **see** the fan but can't connect to control it
- PySwitchbot ≥ 2.2.0 (ships with HA core)

## Install via HACS (recommended)

1. HACS → ⋮ → **Custom repositories**
2. Add `https://github.com/<your-username>/ha-switchbot-standing-fan` as **Integration**
3. Search "SwitchBot Standing Fan" in HACS and install
4. **Restart Home Assistant**
5. Your fan should appear on the Devices & Services page as a "Discovered" card. Click Configure → done.

## Install manually

1. Copy `custom_components/switchbot/` to `/config/custom_components/switchbot/` on your HA instance
2. Restart Home Assistant
3. **Settings → Devices & Services** → look for the "Discovered SwitchBot" card

## How it replaces core

This integration **shadows** the bundled `switchbot` integration (same `domain: switchbot`). Existing SwitchBot devices already configured in HA (Bots, Curtains, Roller Shades, Locks, Plug Minis, …) continue to work — all 26 files from core 2026.6.4 are copied verbatim, only the four files needed for Standing Fan support are patched.

If/when upstream adds STANDING_FAN, simply delete `/config/custom_components/switchbot/` and restart — core takes over.

## Known limitations

- **Oscillation angle setters (30° / 60° / 90°)** are **not exposed**. PySwitchbot's per-axis angle commands don't appear to land on this firmware variant — the device ignores them. Removed from the entity list to avoid confusing UX. Open to PRs that find a reliable command format.
- **9-hour sleep timer** — not exposed by PySwitchbot.
- **RGB colour for the night light** — not exposed by PySwitchbot (only LEVEL_1 / LEVEL_2 / OFF). The product is marketed as "RGB Lights" but the BLE command surface does not currently support hue/saturation control. Open to a PySwitchbot PR.

## Patched files (vs core 2026.6.4)

```
custom_components/switchbot/
├── const.py            # +SupportedModels.STANDING_FAN, +CONNECTABLE_SUPPORTED_MODEL_TYPES entry
├── __init__.py         # +PLATFORMS_BY_TYPE, +CLASS_BY_DEVICE
├── fan.py              # +SwitchBotStandingFanEntity (9 speeds, 4 modes, no combined oscillate)
├── select.py           # +SwitchBotStandingFanNightLightSelect
├── switch.py           # +SwitchBotStandingFan{H,V}OscillationSwitch
├── manifest.json       # version bump
└── translations/en.json
```

All other files are byte-identical to `home-assistant/core@2026.6.4`.

## Upstream PR

A PR draft to fold this into core is in [`docs/upstream-pr.md`](docs/upstream-pr.md). Contributions / co-signers welcome.

## License

Apache 2.0 (matches Home Assistant core, since the bulk of the code is copied verbatim from `home-assistant/core`).
