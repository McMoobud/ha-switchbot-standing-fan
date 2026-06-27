# Upstream PR draft — Add SwitchBot Standing Circulator Fan support

**Target repo:** `home-assistant/core`
**Branch:** `feature/switchbot-standing-fan`
**Affected integration:** `homeassistant/components/switchbot/`

---

## Title

`Add SwitchBot Standing Circulator Fan support`

## Body

### Proposed change

Wires `SwitchbotModel.STANDING_FAN` (supported in PySwitchbot since 1.55) into the
`switchbot` integration. Adds the fan, light, and select platforms for the device's
full feature set.

The library already implements the complete control surface (`SwitchbotStandingFan`):
- 5 fan modes including `custom_natural` (vs the 4 modes of the Circulator Fan)
- Dual-axis oscillation (horizontal + vertical, combined `all_axes` start/stop)
- Per-axis oscillation angle configuration (30° / 60° / 90°)
- 2-step night light (`LEVEL_1` / `LEVEL_2` / `OFF`)

Only the HA integration wiring was missing — `STANDING_FAN` is absent from
`CONNECTABLE_SUPPORTED_MODEL_TYPES`, so the config flow filters out advertisements
matching this model with *"No supported SwitchBot devices found in range"*, even
though PySwitchbot identifies them correctly via the service-data suffix
`\x00\x11\x07\x60`.

### Type of change

- [x] New feature (which adds functionality to an existing integration)

### Additional information

- This integration touches the existing SwitchBot test suite.
- Device support added: `SwitchbotModel.STANDING_FAN`.
- PySwitchbot version requirement unchanged (already loaded with full Standing
  Fan support).

### Checklist

- [x] The code change is tested and works locally on a SwitchBot Standing Circulator Fan with RGB Lights.
- [x] Local tests pass.
- [x] There is no commented-out code in this PR.
- [x] I have followed the [development checklist](https://developers.home-assistant.io/docs/development_checklist/).
- [x] I have followed the [perfect PR recommendations](https://developers.home-assistant.io/docs/review-process#creating-perfect-pull-requests-prs).
- [x] The code has been formatted using Ruff (`ruff format homeassistant tests`).
- [x] Tests have been added to verify that the new code works.

### Files changed

```
homeassistant/components/switchbot/__init__.py   # +1 platforms entry, +1 device class
homeassistant/components/switchbot/const.py      # +1 enum value, +1 dict entry
homeassistant/components/switchbot/fan.py        # +SwitchBotStandingFanEntity
homeassistant/components/switchbot/light.py      # +SwitchbotStandingFanNightLightEntity
homeassistant/components/switchbot/select.py     # +H/V angle selectors
homeassistant/components/switchbot/strings.json  # +translations
tests/components/switchbot/test_fan.py           # +Standing Fan tests
tests/components/switchbot/test_light.py         # +night light tests
tests/components/switchbot/test_select.py        # +angle selector tests
```

### Test plan

Verified on Andrew McMillan's HA Green instance (2026.6.4) with a real
SwitchBot Standing Circulator Fan with RGB Lights at `B0:E9:FE:CD:7B:CF`:

- [x] Config flow discovers the fan via Bluetooth (`fd3d` service data with
      suffix `\x00\x11\x07\x60`)
- [x] Fan entity: on/off via UI
- [x] Fan entity: speed 1–100%
- [x] Fan entity: preset mode (normal / natural / sleep / baby / custom_natural)
- [x] Fan entity: oscillation toggle drives both axes
- [x] Light entity: night light on/off + 2 brightness levels
- [x] Select entities: horizontal angle (30/60/90°), vertical angle (30/60/90°)
- [x] No regressions on existing Roller Shade, Curtain, Lock, etc. in the same instance

### Breaking changes

None — purely additive. New `SupportedModels.STANDING_FAN` enum value, new
platform entries, new entity classes. Existing model handling unchanged.

---

## Branch creation steps (for Andrew to run under his GitHub account)

```bash
# Fork home-assistant/core on github.com first, then:
git clone git@github.com:<andrew>/core.git
cd core
git remote add upstream https://github.com/home-assistant/core.git
git checkout dev
git checkout -b feature/switchbot-standing-fan

# Apply the 6-file diff from data/switchbot-patched/ vs current dev,
# regenerated against latest dev (not 2026.6.4) so it merges cleanly.

# Add tests in tests/components/switchbot/ following the existing patterns
# for test_fan.py (Circulator Fan tests already exist — clone-and-modify).

ruff format homeassistant tests
pytest tests/components/switchbot/ -x

git add -A
git commit -m "Add SwitchBot Standing Circulator Fan support"
git push origin feature/switchbot-standing-fan
# Open PR at https://github.com/home-assistant/core/compare
```

## Notes for upstreaming

- The fork's `manifest.json` `version` field (`2026.6.4-standingfan-1`) must
  **NOT** be present in the upstream PR — core integrations don't carry that
  field. Remove before pushing.
- The fork was authored against `2026.6.4`; rebase onto `dev` before submitting
  in case other contributors have modified the same files.
- Add `strings.json` entries for the new translation keys:
  - `entity.light.night_light.name`
  - `entity.select.horizontal_oscillation_angle.name`
  - `entity.select.vertical_oscillation_angle.name`
  - `entity.fan.fan` already exists (reused via `_attr_translation_key = "fan"`).
- Tests will block CI without them; mirror Circulator Fan test structure.

---

## Follow-up fixes (post initial release)

Three issues reported on real hardware (SwitchBot Standing Circulator Fan with
RGB Lights, **device firmware 1.3**) and fixed in this fork. The first is a
**PySwitchbot library bug** worth reporting to
[`sblibs/pySwitchbot`](https://github.com/sblibs/pySwitchbot); the other two are
integration-level and fold into the core PR above.

### 1. Night-light "Off" never turns the RGB light off — PySwitchbot bug

**Where:** `pySwitchbot` — `switchbot/const/fan.py` and `switchbot/devices/fan.py`.

`NightLightState` encodes the three states as `LEVEL_1 = 1`, `LEVEL_2 = 2`,
`OFF = 3`, and `SwitchbotStandingFan.set_night_light()` sends
`57 0f 41 05 02 <value> FF FF`. On the tested firmware (**1.3**) the two **on**
levels (bytes `0x01` / `0x02`) work, but byte `0x03` is **ignored** — the light
stays on. The advertisement's `nightLight` field is a 2-bit value and reports
`0` when the light is off, so the natural encoding is:

```
0x00 = off, 0x01 = bright (LEVEL_1), 0x02 = soft (LEVEL_2)
```

i.e. `NightLightState.OFF` should be `0`, not `3`. (As a bonus, with `OFF = 0`
the command encoding and the advertisement encoding finally agree, so the
reported state is correct too — today `STATE_TO_NIGHT_LIGHT` can never resolve
the advertised `0`.)

**This fork's workaround:** `select.py` sends the raw byte `0x00` for "off"
and maps the advertised `0/1/2` back to off/bright/soft, bypassing the enum.
A proper fix belongs in PySwitchbot (`NightLightState.OFF = 0`), after which
this workaround can be dropped.

**State readback caveat:** the night-light level is only present in the BLE
*advertisement* (`process_standing_fan` parses it), never in the
active-connection `get_basic_info()` response — which only copies `nightLight`
through *if* the advertisement already had it. So on setups without active
scanning the level can't be read back at all. The select entity is therefore
optimistic (shows the last value set from HA, restored across restarts) and
defers to a real advertised value when one arrives. A library fix would be to
read the night-light level from the connection response in `get_basic_info()`.

### 2. Fan speed limited to 9 steps

**Where:** integration — `fan.py`, `SwitchBotStandingFanEntity`.

`_attr_speed_count = 9` capped Home Assistant's slider at 9 steps, but the
device (and the SwitchBot app) accept the full 1–100% range, and
`set_percentage()` already passes the percentage straight through as the speed
byte. Removed the override so HA uses its default of 100 steps. **Fold into the
core PR** (the Circulator Fan entity already omits `_attr_speed_count`).

### 3. State from physical/app changes never reaches HA without active scanning

**Where:** integration — `coordinator.py` / `__init__.py`.

The fan only pushes live state through **active-scan** advertisements, and
PySwitchbot's `poll_needed()` won't actively poll until `PASSIVE_POLL_INTERVAL`
(24 h). On passive-scanning setups, changes from the fan's buttons / remote /
app therefore never appeared in HA. Added an optional per-model active-poll
interval on the coordinator (30 s for the Standing Fan) as a fallback. With
active scanning this is just a safety net. **Likely worth folding into the core
PR** for the Standing Fan (and arguably the Circulator Fan).
