# Home Assistant + Mitsubishi Electric: external room-temperature control

> **Update 2026-09-11:** If you use the local `pymitsubishi/homeassistant-mitsubishi` integration and **Remote Temperature** works with your adapter/unit, the older target-offset / step-compensation logic is no longer required for the main Mitsubishi room-temperature control. The external Home Assistant sensor can be sent directly to the unit as its room-temperature reference.

This repository keeps three solution paths side by side. The older files remain intentionally available as **Legacy/Fallback** material for users who stay on MELCloud/MELCloud Home or cannot use Remote Temperature reliably.

> Not affiliated with Mitsubishi Electric, Home Assistant, AVM, or the authors of the local custom integration.

> Back up Home Assistant before changing integrations, entity IDs or automations.

---

# Which solution should I use?

## 1. Legacy / classic MELCloud

Use the older external-sensor / simulated-AUTO / target-compensation approach if that is the architecture your installation still requires.

Typical legacy files:

- `living_room_multi_sensor.yaml`
- `bedroom_single_sensor.yaml`
- `melcloud_refresh_5min.yaml`
- `optional_horizontal_swing.yaml`

**Status:** still usable, but not the preferred path when the local Mitsubishi integration can use Remote Temperature.

---

## 2. MELCloud Home primary control

If you intentionally keep cloud control as the primary path, the target-offset automation is still useful.

Native mode stays unchanged:

- `AUTO` stays `AUTO`
- `HEAT` stays `HEAT`
- `COOL` stays `COOL`

Home Assistant compensates the Mitsubishi target from the difference between external room temperature and desired room temperature.

Main file:

`melcloud_home_external_temperature_control_public.yaml`

**Status:** still relevant for cloud-primary setups.

---

## 3. Recommended: local Mitsubishi control with Remote Temperature

Tested local integration:

[pymitsubishi/homeassistant-mitsubishi](https://github.com/pymitsubishi/homeassistant-mitsubishi)

Primary path:

```text
Home Assistant -> local LAN/WLAN -> MAC-577IF2-E -> indoor unit
```

Optional fallback/manual path:

```text
MELCloud Home -> Mitsubishi cloud -> indoor unit
```

### Setup

1. Reconfigure the Mitsubishi Air Conditioner integration entry.
2. Enable **Experimental Features**.
3. Select the desired **External Temperature Sensor**.
4. Reload the integration if the new entity does not appear immediately.
5. Open the new **Temperature Source** select entity.
6. Change it from `Internal` to `Remote`.
7. Verify that `Remote` remains selected after a Home Assistant restart.
8. Use the real desired room temperature directly as the climate target.
9. Disable the old main-unit target-offset / step-compensation automation for that unit.

Detailed guide:

`REMOTE_TEMPERATURE_RECOMMENDED.md`

### What changes conceptually?

Old local compensation path:

```text
external room temp
 -> calculate error
 -> calculate offset / 0.5 °C step
 -> write artificial Mitsubishi target
```

Recommended Remote Temperature path:

```text
external room sensor -> Mitsubishi Remote Temperature
user desired temp     -> Mitsubishi target temperature
```

Example:

```text
external room average: 24.3 °C
desired temperature:   23.0 °C
Mitsubishi target:     23.0 °C
Temperature Source:    Remote
```

No extra ±0.5 / ±1.0 / ±1.5 °C target correction is required.

### External sensors

For Remote Temperature, prefer independent room sensors over radiator-thermostat temperature readings. Radiator thermostats can be biased by placement close to the radiator.

If several sensors should be combined, a Home Assistant combination/statistics helper using the **arithmetic mean** can be selected as the Remote Temperature source.

### Fallback behavior

If the configured external sensor becomes unavailable or invalid while Home Assistant is running, the integration can fall back to the internal sensor.

Important limitation: if Home Assistant itself or network connectivity to the AC disappears while Remote mode is active, the unit may continue using the last received remote temperature until communication is restored.

Use reliable sensors, monitor battery devices, keep the LAN/WLAN stable and keep a known-good backup/fallback configuration.

---

# What remains useful from the older automation work?

A lot. Remote Temperature replaces only the **main-unit room-temperature compensation engine**.

The following remain valuable:

- local MAC-577 control
- MELCloud Home fallback
- `hvac_action`
- auxiliary radiator / heater logic
- winter reserve
- away / vacation shutdowns
- desired-temperature helpers
- external-sensor averaging
- vane control
- restart-safe timers and safety checks

The older offset/step files are therefore kept as Legacy/Fallback material instead of being deleted.

---

# `hvac_action` still matters

The local integration exposes the real operating state:

```text
heating  -> active heating
cooling  -> active cooling
idle     -> unit is on, compressor is currently not operating
```

This remains important for auxiliary-heating logic.

`idle` does **not** prove that the external room has reached the desired temperature. External room-temperature, outdoor-temperature, timing and hysteresis checks may still be needed before allowing auxiliary heat.

`AUTO + cooling` is unambiguous and should remain a hard block for auxiliary heating.

---

# Desired temperature in the Remote Temperature path

Recommended model:

```text
desired-temperature helper = user intent / source of truth
climate target              = same desired temperature (subject to device resolution)
external room sensor        = real Mitsubishi room-temperature reference
```

The older compensated-target model is no longer needed for the main AC when Remote Temperature works correctly.

---

# Files

| File | Purpose / status |
| --- | --- |
| `REMOTE_TEMPERATURE_RECOMMENDED.md` | **Recommended local guide** |
| `melcloud_home_external_temperature_control_public.yaml` | Still relevant for MELCloud Home primary control |
| `living_room_multi_sensor.yaml` | Legacy/Fallback |
| `bedroom_single_sensor.yaml` | Legacy/Fallback |
| `melcloud_refresh_5min.yaml` | Legacy MELCloud only |
| `optional_horizontal_swing.yaml` | Device-dependent legacy/optional example |
| `local_primary_hvac_action_aux_heating_example.yaml` | Auxiliary/safety logic still useful; its old main-target compensation is not recommended when Remote Temperature is active |
| `local_primary_helpers_example.yaml` | Helper examples |
| `LOCAL_CONTROLLER_DESIGN_NOTES.md` | Technical history and design rationale |
| `LOCAL_CONTROL_WITH_MELCLOUD_FALLBACK.md` | Migration/fallback background; Remote Temperature now takes precedence for main room control |
| `FORUM_POST_LOCAL_EN.md` | Updated ready-to-adapt forum post for the Remote Temperature path |
| `SECURITY_PRIVACY.md` | Security and privacy notes |

---

# No competing controllers

Do **not** run two complete main controllers against the same physical Mitsubishi unit at the same time.

MELCloud Home may remain configured as a manual/fallback path, but only one main automation strategy should regulate the unit.

---

# Security and privacy

Public examples intentionally contain no passwords, cloud credentials, Home Assistant tokens, API keys, webhook IDs, MAC addresses, e-mail addresses, personal names, local IP addresses or private hostnames.

Before publishing your own adapted configuration, scan it again for credentials and unique network/device identifiers.

---

# Feedback

Useful feedback includes:

- Home Assistant version
- selected solution path
- local integration version or MELCloud/MELCloud Home variant
- Mitsubishi indoor-unit model
- Wi-Fi adapter model
- external sensor type
- Remote Temperature behavior
- `hvac_action` transitions
- behavior during Home Assistant or network outages
- fallback to internal sensor
- vane behavior
- auxiliary-heating interaction

Issues, test results and improvements are welcome.
