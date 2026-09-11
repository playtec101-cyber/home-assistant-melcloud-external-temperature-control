# Mitsubishi + Home Assistant: external room temperature

> **Update 2026-09-11:** For the local `pymitsubishi/homeassistant-mitsubishi` integration, **Remote Temperature is now the recommended path**. If it works with your adapter/unit, the older target-offset / step-compensation logic is no longer needed for the main Mitsubishi room-temperature control.

## The four solution paths

### 1. Home Assistant simulates AUTO by choosing HEAT or COOL

Home Assistant decides whether the Mitsubishi should run in `HEAT` or `COOL` from one or more external room sensors.

Use this only if native Mitsubishi AUTO is unsuitable or unavailable for your setup.

Typical legacy files:

- `living_room_multi_sensor.yaml`
- `bedroom_single_sensor.yaml`

**Status:** Legacy / fallback.

---

### 2. MELCloud Home + native Mitsubishi AUTO + target offset

Mitsubishi `AUTO` stays native `AUTO`. Home Assistant does not switch HVAC modes. Instead it compensates the Mitsubishi target temperature from the difference between the external room reference and the desired room temperature.

Main file:

- `melcloud_home_external_temperature_control_public.yaml`

**Status:** Still useful if MELCloud Home is intentionally kept as the primary control path.

---

### 3. Local `pymitsubishi` + native Mitsubishi AUTO + target offset

Control is local through the Mitsubishi adapter, but Home Assistant still uses the older external-sensor compensation logic:

```text
external room temp -> error -> offset / 0.5 °C step -> artificial Mitsubishi target
```

This keeps native AUTO while avoiding cloud-primary control.

**Status:** Legacy/fallback for local installations where Remote Temperature is unavailable or unreliable.

---

### 4. Local `pymitsubishi` + Remote Temperature — recommended

Current preferred architecture:

```text
Home Assistant -> local LAN/WLAN -> MAC-577IF2-E -> Mitsubishi indoor unit
```

Configure:

1. **Experimental Features** = enabled
2. **External Temperature Sensor** = your real room sensor or HA room-average sensor
3. **Temperature Source** = `Remote`
4. Mitsubishi target temperature = your real desired room temperature
5. Disable the old main-unit offset / step-compensation automation

Conceptually:

```text
external room sensor -> Mitsubishi Remote Temperature
user desired temp     -> Mitsubishi target temperature
```

No artificial +0.5 / +1.0 / +1.5 °C target correction is required when Remote Temperature works correctly.

Detailed guide:

**[`REMOTE_TEMPERATURE_RECOMMENDED.md`](REMOTE_TEMPERATURE_RECOMMENDED.md)**

MELCloud Home may remain configured as a manual/fallback path.

---

## What remains useful from the older work?

Remote Temperature replaces only the **main AC room-temperature compensation engine**. The following remain useful:

- local MAC-577 control
- MELCloud Home fallback
- `hvac_action`
- auxiliary-heating logic
- winter reserve
- away/vacation safety
- desired-temperature helpers
- external-sensor averaging
- vane control
- restart/fallback logic

Old local offset/step documents remain available only as **archive/legacy references** and as a fallback if Remote Temperature cannot be used.

## Important

Do not run two complete main controllers against the same physical Mitsubishi unit at the same time.

Back up Home Assistant before changing integrations, entity IDs or automations.
