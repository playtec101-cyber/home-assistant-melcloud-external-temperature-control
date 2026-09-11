# Mitsubishi + Home Assistant: four external-temperature control paths — Remote Temperature now recommended

> **Update 2026-09-11:** For users of the local `pymitsubishi/homeassistant-mitsubishi` integration, **Remote Temperature** is now the simplest recommended path. If it works with your adapter/unit, the older Home Assistant offset / step-compensation logic is no longer needed for the main Mitsubishi room-temperature control.

## The four solution paths

### 1. Home Assistant simulates AUTO and chooses HEAT or COOL

Home Assistant decides from external room sensors whether the Mitsubishi should run in `HEAT` or `COOL`.

**Status:** legacy/fallback. Useful only if native Mitsubishi AUTO is not suitable for the installation.

### 2. MELCloud Home + native Mitsubishi AUTO + target offset

Mitsubishi `AUTO` stays native `AUTO`. Home Assistant does not switch HVAC modes; it only compensates the target temperature from the difference between external room temperature and desired room temperature.

**Status:** still relevant for cloud-primary MELCloud Home setups.

### 3. Local `pymitsubishi` + native AUTO + target offset

Control is local through the Mitsubishi adapter, native AUTO is preserved, but Home Assistant still applies the older offset/step method:

```text
external room temperature
-> calculate error
-> offset / fixed 0.5 °C step
-> artificial Mitsubishi target
```

**Status:** local fallback if Remote Temperature is unavailable or unreliable.

### 4. Local `pymitsubishi` + Remote Temperature — recommended

Recommended current path:

```text
external room sensor -> Mitsubishi Remote Temperature
user desired temp     -> Mitsubishi target temperature
```

The Mitsubishi receives the real external room temperature directly and can use its own inverter/native AUTO logic against the real desired target.

## Remote Temperature setup

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

## Fallback note

If the external sensor becomes invalid while Home Assistant is running, the integration can fall back to the internal sensor.

If Home Assistant itself or network connectivity to the AC is lost while Remote mode is active, the unit may continue using the last received Remote Temperature until communication returns.

## What remains useful from the older work?

Remote Temperature replaces only the main AC temperature-compensation engine. Still useful are:

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

Full documentation with all four paths:

https://github.com/playtec101/home-assistant-melcloud-external-temperature-control

Recommended Remote Temperature guide:

https://github.com/playtec101/home-assistant-melcloud-external-temperature-control/blob/main/REMOTE_TEMPERATURE_RECOMMENDED.md
