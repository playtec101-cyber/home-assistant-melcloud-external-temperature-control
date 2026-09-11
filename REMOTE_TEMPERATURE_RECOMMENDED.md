# Recommended local setup: Remote Temperature (11.09.2026)

> **Recommended path for `pymitsubishi/homeassistant-mitsubishi` when Remote Temperature works.**
>
> The older offset/step-compensation YAML is no longer required for the main Mitsubishi room-temperature control in this setup.

## Why this replaces the old local offset logic

The old controller had to compensate for the indoor unit's own temperature sensor by calculating a corrected target:

```text
external room temperature
        -> error to desired temperature
        -> offset / fixed 0.5 °C step
        -> artificial Mitsubishi target
```

With Remote Temperature, the external Home Assistant sensor is sent directly to the Mitsubishi as the room-temperature reference:

```text
external room sensor -> Mitsubishi Remote Temperature
user desired temp     -> Mitsubishi target temp
```

The unit can then use its own native inverter and AUTO logic with the external room value.

## Tested architecture

```text
PRIMARY
Home Assistant -> local LAN/WLAN -> MAC-577IF2-E -> Mitsubishi indoor unit

OPTIONAL FALLBACK
MELCloud Home -> Mitsubishi cloud -> indoor unit
```

Tested with the custom integration:

- `pymitsubishi/homeassistant-mitsubishi`
- Remote Temperature / experimental feature support

## Setup steps

1. Open the Mitsubishi Air Conditioner integration entry.
2. Reconfigure it.
3. Enable **Experimental Features**.
4. Select the desired **External Temperature Sensor**.
5. Reload the integration if the new entity does not appear immediately.
6. Find the new **Temperature Source** select entity.
7. Change it from `Internal` to `Remote`.
8. Verify that it remains `Remote` after a Home Assistant restart.
9. Use the real desired room temperature directly as the climate target.
10. Disable/remove any old target-offset automation for the same unit.

## Multiple sensors

If several independent room sensors should be used, create a Home Assistant combination/statistics helper using the **arithmetic mean** and select that helper as the external temperature sensor.

Prefer independent room sensors over radiator-thermostat temperature readings. Radiator thermostats can be biased by their physical location near the radiator.

## Expected behavior

Example:

```text
external room average: 24.3 °C
desired temperature:   23.0 °C
Mitsubishi target:     23.0 °C
Temperature Source:    Remote
hvac_action:            idle
```

No extra +0.5 / +1.0 / +1.5 °C target correction is needed.

## Fallback behavior

If the configured external sensor becomes unavailable or invalid while Home Assistant is running, the integration can fall back to the internal sensor.

Important limitation: if Home Assistant itself or network connectivity to the AC disappears while Remote mode is active, the AC may continue using the last received remote temperature until communication is restored.

For that reason:

- use reliable room sensors;
- monitor battery-powered sensors;
- keep the local network stable;
- test restart behavior;
- keep a known-good fallback configuration/backup.

## `hvac_action` still matters

Remote Temperature replaces only the main room-temperature compensation logic.

`hvac_action` is still useful for auxiliary-heating rules:

```text
heating  -> active heating
cooling  -> active cooling
idle     -> compressor currently not operating
```

Do not interpret `idle` as proof that the room is at the desired temperature. External demand checks can still be relevant for optional auxiliary heating.

## What from the old work is still useful?

The following parts remain valuable:

- local MAC-577 control
- MELCloud Home fallback
- `hvac_action` logic
- auxiliary heaters / radiator thermostats
- winter reserve
- away / vacation shutdowns
- desired-temperature helpers
- external-sensor averaging
- vane control
- restart-safe timers and safety checks

Only the old **main-unit offset / step compensation** becomes unnecessary when Remote Temperature is working correctly.

## Legacy / fallback paths

The older YAML files are intentionally kept in the repository for users who:

- stay on classic MELCloud;
- stay on MELCloud Home as the primary path;
- cannot use the local integration;
- cannot use Remote Temperature reliably on their adapter/unit;
- want a tested fallback.

See the main README for the three solution paths.
