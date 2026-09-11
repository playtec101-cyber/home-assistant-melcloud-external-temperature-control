# Update 2026-09-11: local Mitsubishi Remote Temperature is now the preferred path

If you use the local `pymitsubishi/homeassistant-mitsubishi` integration and its **Remote Temperature** feature works with your adapter/unit, the older Home Assistant target-offset / step-compensation logic is no longer needed for the main room-temperature control.

The recommended local setup is now:

```text
Home Assistant external room sensor
        -> Mitsubishi Remote Temperature

User desired temperature
        -> Mitsubishi target temperature
```

No extra ±0.5 / ±1.0 / ±1.5 °C target correction is required.

## Three solution paths

### 1. Legacy / classic MELCloud

Keep the older Home Assistant offset/simulated-AUTO approach if that is the architecture you still use.

### 2. MELCloud Home primary

The target-correction automation remains useful if you intentionally keep cloud control as the primary path.

### 3. Recommended: local Mitsubishi integration + Remote Temperature

Local path:

```text
Home Assistant -> local LAN/WLAN -> MAC-577IF2-E -> Mitsubishi indoor unit
```

MELCloud Home may remain configured as a manual fallback.

## Setup

1. Open the Mitsubishi Air Conditioner integration entry.
2. Reconfigure it.
3. Enable **Experimental Features**.
4. Select the desired **External Temperature Sensor**.
5. Reload the integration if necessary.
6. Open the new **Temperature Source** entity.
7. Change `Internal` to `Remote`.
8. Verify that `Remote` survives a Home Assistant restart.
9. Set the real desired room temperature directly as the climate target.
10. Disable the old main-unit offset/step automation for that unit.

## Multiple external sensors

A Home Assistant combination/statistics helper using the **arithmetic mean** can be used as the Remote Temperature source.

Independent room sensors are preferable to radiator-thermostat temperature values because thermostat placement near a radiator can bias the measurement.

## What remains useful from the older automation work?

A lot:

- local MAC-577 control
- MELCloud Home fallback
- `hvac_action`
- auxiliary radiator/heater logic
- winter reserve
- away/vacation shutdowns
- desired-temperature helpers
- sensor averaging
- vane control
- restart-safe timers and safety checks

Only the old **main Mitsubishi target compensation** becomes unnecessary when Remote Temperature works correctly.

## `hvac_action`

The local integration still exposes the real operating state:

```text
heating  -> active heating
cooling  -> active cooling
idle     -> compressor currently not operating
```

This remains useful for auxiliary-heating logic. `idle` should not be interpreted as proof that the room is already at the desired temperature.

## Important fallback note

If the external sensor becomes invalid while Home Assistant is running, the integration can fall back to the internal sensor.

If Home Assistant itself or network connectivity to the AC is lost while Remote mode is active, the unit may continue using the last received Remote Temperature until communication returns.

That is why reliable sensors, a stable LAN/WLAN and a known-good fallback/backup are recommended.

Full updated documentation:

https://github.com/playtec101/home-assistant-melcloud-external-temperature-control

Recommended local guide:

`REMOTE_TEMPERATURE_RECOMMENDED.md`

The older offset YAML files are intentionally kept as Legacy/Fallback material for users who cannot or do not want to use Remote Temperature.
