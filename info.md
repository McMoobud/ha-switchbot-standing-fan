# SwitchBot Standing Fan for Home Assistant

Local Bluetooth control for the SwitchBot Standing Circulator Fan.

Fills the gap where `homeassistant/core`'s `switchbot` integration recognises the device via PySwitchbot but does not expose it through the config flow.

**Adds:** fan entity (9-step speed, 4 preset modes), per-axis oscillation switches, night-light select, battery sensor — all over local BLE via your existing active-scanning proxies.

**Replaces** the bundled `switchbot` integration as a superset; existing SwitchBot devices keep working.

See the [README](https://github.com/McMoobud/ha-switchbot-standing-fan) for full documentation.
