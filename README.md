# SwitchBot Standing Fan — Home Assistant custom integration

> ⚠️ **Disclaimer:** This integration was built end-to-end using **[Claude Code](https://claude.com/claude-code)** (Anthropic) — including the BLE diagnostics that identified the device, the patches against `home-assistant/core`, the entity wiring, on-device testing, and this README. Treat it as community-quality AI-assisted work: it works on the maintainer's hardware (tested on a SwitchBot Standing Circulator Fan with RGB Lights, MAC) and the code is plain-Python and reviewable, but it has not been through HA's core review process. PRs and issues welcome.


A drop-in replacement for Home Assistant's bundled `switchbot` integration that adds **full local Bluetooth control** for the **SwitchBot Standing Circulator Fan** (and any other Standing Fan model that identifies via service-data suffix `0x00 11 07 60`).

> **Why?** PySwitchbot recognises the Standing Fan and exposes the full control surface, but `homeassistant/core` itself does not list `STANDING_FAN` in `CONNECTABLE_SUPPORTED_MODEL_TYPES`. The config flow filters it out with *"No supported SwitchBot devices found in range"* — even though the library identifies the device perfectly. This integration fills that gap until upstream catches up.

## What it adds

When a Standing Fan is discovered, the following entities are created:

| Entity | Purpose |
|---|---|
| `fan.<name>` | Power, 1–100% speed, 4 preset modes (Normal / Natural / Sleep / Baby) |
| `switch.<name>_horizontal_oscillation` | Horizontal sweep on/off |
| `switch.<name>_vertical_oscillation` | Vertical sweep on/off |
| `select.<name>_night_light` | Off / Soft / Bright |
| `sensor.<name>_battery` | Battery % |

All commands are sent **locally over BLE** via an active-scanning Bluetooth proxy — no cloud, no SwitchBot account, no SwitchBot Hub required.

### State sync

The fan pushes live state (on/off, speed, mode, battery) to HA through its BLE advertisements — but only when an **active-scanning** proxy is in range to solicit them. So that changes made from the fan's own buttons, the remote, or the SwitchBot app still show up on passive-only setups, the integration also **actively polls the fan** as a fallback. With active scanning, updates are effectively instant and the poll is just a safety net.

The poll interval is configurable: **Settings → Devices & Services → SwitchBot → (your fan) → Configure → "Fan status poll interval (seconds)"**. It defaults to **30 s**; raise it (or set **0** to disable) if the fan runs on battery and you want to minimise standby drain. Note the fan's BLE radio stays awake while the fan is off, so polling consumes a little battery even in off mode.

The **night-light level** is the one exception — it's only ever present in active-scan advertisements, never in the polled connection response. The selector is therefore optimistic (it shows what you last set and survives restarts) and only mirrors changes made *outside* HA when an active-scanning proxy is in range.

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

This integration **shadows** the bundled `switchbot` integration (same `domain: switchbot`). Existing SwitchBot devices already configured in HA (Bots, Curtains, Roller Shades, Locks, Plug Minis, …) continue to work — all 26 files from core 2026.6.4 are copied verbatim, only the handful of files needed for Standing Fan support are patched.

If/when upstream adds STANDING_FAN, simply delete `/config/custom_components/switchbot/` and restart — core takes over.

## Known limitations

- **Oscillation angle setters (30° / 60° / 90°)** are **not exposed**. PySwitchbot's per-axis angle commands don't appear to land on this firmware variant — the device ignores them. Removed from the entity list to avoid confusing UX. Open to PRs that find a reliable command format.
- **9-hour sleep timer** — not exposed by PySwitchbot.
- **RGB colour for the night light** — not exposed by PySwitchbot (only Off / Soft / Bright). The product is marketed as "RGB Lights" but the BLE command surface does not currently support hue/saturation control. Open to a PySwitchbot PR.
  - Note: the **Off** option sends night-light brightness byte `0x00`. PySwitchbot's `NightLightState.OFF` encodes off as byte `0x03`, which this firmware ignores (the light stays on), so the integration sends the raw `0x00` command instead. See [`docs/upstream-pr.md`](docs/upstream-pr.md) for the upstream report.

## Patched files (vs core 2026.6.4)

```
custom_components/switchbot/
├── const.py            # +SupportedModels.STANDING_FAN, +CONNECTABLE_SUPPORTED_MODEL_TYPES entry, +fan poll-interval consts
├── __init__.py         # +PLATFORMS_BY_TYPE, +CLASS_BY_DEVICE, +configurable fan poll interval
├── config_flow.py      # +Standing Fan poll-interval option
├── fan.py              # +SwitchBotStandingFanEntity (1–100% speed, 4 modes, no combined oscillate)
├── select.py           # +SwitchBotStandingFanNightLightSelect (raw 0x00 off command, optimistic state)
├── switch.py           # +SwitchBotStandingFan{H,V}OscillationSwitch
├── coordinator.py      # +configurable active-poll interval (state-sync fallback)
├── manifest.json       # version bump
└── translations/en.json
```

All other files are byte-identical to `home-assistant/core@2026.6.4`.

## Upstream PR

A PR draft to fold this into core is in [`docs/upstream-pr.md`](docs/upstream-pr.md). Contributions / co-signers welcome.

## License

Apache 2.0 (matches Home Assistant core, since the bulk of the code is copied verbatim from `home-assistant/core`).
