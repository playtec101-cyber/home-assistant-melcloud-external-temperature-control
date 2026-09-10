# Home Assistant + Mitsubishi Electric: external room-temperature control

Home Assistant examples for Mitsubishi Electric air conditioners that use **external room-temperature sensors as the real comfort reference** instead of relying only on the sensor inside the indoor unit.

This repository documents **three different solution paths**. They represent different stages and architectures and are intentionally kept side by side so users can choose the approach that best fits their installation.

> Not affiliated with Mitsubishi Electric, Home Assistant, AVM, or the authors of the local custom integration.

> Back up your Home Assistant configuration before changing integrations, entity IDs or automations.

---

# The three solution paths

## 1. Legacy / simulated AUTO with Home Assistant deciding HEAT and COOL

This is the older approach for installations where native Mitsubishi AUTO behavior was not satisfactory.

Home Assistant uses external room-temperature sensor(s) as the real reference and can decide whether the unit should run in HEAT or COOL instead of relying entirely on Mitsubishi's native AUTO decision.

Relevant legacy files include:

- `living_room_multi_sensor.yaml`
- `bedroom_single_sensor.yaml`
- `melcloud_refresh_5min.yaml`
- `optional_horizontal_swing.yaml`

These files are mainly for the classic/legacy MELCloud integration and should not be copied blindly into MELCloud Home or the local integration because state names, polling behavior and vane values can differ.

---

## 2. MELCloud Home primary control with real Mitsubishi AUTO

This is the cloud-based real-AUTO controller.

The user-selected Mitsubishi mode stays unchanged:

- `AUTO` stays `AUTO`
- `HEAT` stays `HEAT`
- `COOL` stays `COOL`

Home Assistant only adjusts the Mitsubishi target temperature from the error between the external room temperature and the desired room temperature.

Current complete cloud-primary example:

`melcloud_home_external_temperature_control_public.yaml`

### AUTO correction

AUTO uses fixed 0.5 °C correction steps:

| Absolute room error | Correction |
| --- | --- |
| `<= 0.25 °C` | `0.0 °C` |
| `> 0.25 to 0.75 °C` | `0.5 °C` |
| `> 0.75 to 1.25 °C` | `1.0 °C` |
| `> 1.25 to 1.75 °C` | `1.5 °C` |
| `> 1.75 to 2.25 °C` | `2.0 °C` |
| `> 2.25 to 2.75 °C` | `2.5 °C` |
| `> 2.75 °C` | `3.0 °C` |

The sign follows the direction of the room error.

HEAT and COOL use the same `±0.25 °C` neutral zone and, outside it, a linear 1:1 correction.

This cloud-primary example still documents the older optional target back-synchronization pattern. The newer local-primary controller below deliberately uses a different, stricter desired-temperature model.

---

## 3. Local-primary Mitsubishi control with MELCloud Home as fallback

This is the current preferred architecture in this repository.

Primary path:

```text
Home Assistant -> local LAN/WLAN -> MAC-577IF2-E -> indoor unit
```

Fallback/manual path:

```text
MELCloud Home -> Mitsubishi cloud -> indoor unit
```

Tested local custom integration:

[pymitsubishi/homeassistant-mitsubishi](https://github.com/pymitsubishi/homeassistant-mitsubishi)

Current neutral public controller:

`local_primary_hvac_action_aux_heating_example.yaml`

Matching helper example:

`local_primary_helpers_example.yaml`

Detailed design rationale and change notes:

`LOCAL_CONTROLLER_DESIGN_NOTES.md`

Step-by-step migration guide:

`LOCAL_CONTROL_WITH_MELCLOUD_FALLBACK.md`

### What changed in the local-primary controller

The core room-temperature compensation remains the same: native Mitsubishi AUTO stays native AUTO and Home Assistant adjusts only the target temperature.

The local controller now adds several important safeguards and behaviors:

- `hvac_action` is used to distinguish `AUTO + heating` from `AUTO + cooling` and `AUTO + idle`.
- Auxiliary heating is allowed in HEAT, and in AUTO only while `hvac_action == heating`.
- Auxiliary heating is blocked during AUTO cooling/idle/unknown and during COOL.
- The desired-temperature helper is authoritative; device target changes are **not** written back into it.
- The older target back-synchronization automations were removed to eliminate a race/feedback-loop class.
- The auxiliary radiator target is `desired - 1.0 °C` after a 25-minute demand delay.
- When the main AC is OFF, a separate winter reserve can hold 18 °C at night and can start a 16 -> 18 °C daytime reserve.
- The AC-ON and AC-OFF auxiliary-radiator automations are mutually exclusive to avoid duplicate commands.
- Numeric template conversions use explicit defaults.
- A separate switch-controlled heater can use the same real-heating permission and a restart-safe absolute 4-hour manual deadline.

See `LOCAL_CONTROLLER_DESIGN_NOTES.md` for the full reasoning behind every decision.

### Important status note about `hvac_action`

The local integration implementation exposes the current HVAC action and maps Mitsubishi AUTO heating/cooling states into Home Assistant `heating` / `cooling` actions, with `idle` when the compressor is not operating.

That makes it suitable for auxiliary-heating gating. However, every installation should still observe `hvac_action` over several real AUTO heating/cooling/idle cycles before treating the behavior as fully proven for its exact hardware/firmware combination.

### Horizontal vane difference

In the tested local integration:

```text
local integration center: center
```

MELCloud Home may use:

```text
MELCloud Home center: centre
```

Both may use `swing` for horizontal swing.

### No competing controllers

Do **not** run two complete automation controllers against the same physical Mitsubishi unit at the same time.

MELCloud Home may remain configured as a manual/fallback path, but only one main automation controller should actively regulate the unit.

---

# Why the desired-temperature helper is authoritative in solution 3

The local controller intentionally does not copy device target changes back into the desired-temperature helper.

Automatic compensation writes and asynchronous device echoes can otherwise race each other. An intermediate target can be mistaken for a user request and create rapid target bouncing.

Therefore:

```text
desired-temperature helper = real user intent / source of truth
climate entity target       = compensated device target
```

If the user wants to change the real desired temperature, dashboards, scripts, voice control or services should change the desired-temperature helper directly.

The `last_automatic_target_*` helpers remain only for detecting the controller's own writes, avoiding unnecessary repeats and supporting the periodic target-delivery retry.

---

# Optional auxiliary-heating behavior in solution 3

The local public controller includes generalized optional logic for two radiator thermostat entities and one switch-controlled heater.

The example defaults are intentionally conservative and must be reviewed for each building:

- radiator release after 25 minutes of stable demand;
- demand threshold: room at least 0.5 °C below desired;
- outdoor limit for radiator assistance: 20 °C;
- radiator target after release: desired minus 1.0 °C;
- AC OFF night reserve: 18 °C from 23:00 to 08:00 when outdoor temperature is valid and <= 20 °C;
- AC OFF daytime reserve: start below 16 °C, hold until 18 °C;
- switch-controlled auxiliary heater outdoor limit: 15 °C;
- switch-controlled heater automatic stop: desired + 0.5 °C;
- manual heater runtime limit: 4 hours;
- 23:00 heater shutdown event.

The 25-minute trigger duration intentionally starts again after Home Assistant restarts. This is fail-safe for this use case: it can delay auxiliary heating, but cannot make it start early.

---

# Installation summary for local-primary control

1. Back up Home Assistant.
2. Install and test the local Mitsubishi integration.
3. Keep MELCloud Home only as fallback/manual control if desired.
4. Verify local power, target temperature, AUTO, HEAT, COOL and optional vane control.
5. Verify the `hvac_action` attribute in Developer Tools while AUTO is heating, cooling and idle.
6. Create the helpers from `local_primary_helpers_example.yaml` or in the Home Assistant UI.
7. Replace every placeholder entity ID in `local_primary_hvac_action_aux_heating_example.yaml`.
8. Review every temperature threshold, schedule and auxiliary-heating rule.
9. Ensure no second full controller is active against the same unit.
10. Reload automations or restart Home Assistant.
11. Test one function at a time and review logs.

---

# Files

| File | Purpose |
| --- | --- |
| `local_primary_hvac_action_aux_heating_example.yaml` | Current neutral local-primary controller with native AUTO, `hvac_action` gating, authoritative desired temperature, auxiliary heating and winter reserve |
| `local_primary_helpers_example.yaml` | Matching helpers for the local-primary controller |
| `LOCAL_CONTROLLER_DESIGN_NOTES.md` | Detailed explanation of the architecture and why each change was made |
| `LOCAL_CONTROL_WITH_MELCLOUD_FALLBACK.md` | Local-primary architecture, migration and test guide |
| `melcloud_home_external_temperature_control_public.yaml` | Cloud-primary real-AUTO controller / older MELCloud Home design |
| `living_room_multi_sensor.yaml` | Legacy external-sensor / simulated-AUTO example |
| `bedroom_single_sensor.yaml` | Legacy single-sensor example |
| `melcloud_refresh_5min.yaml` | Optional forced refresh for legacy MELCloud only |
| `optional_horizontal_swing.yaml` | Legacy/device-dependent horizontal swing example |
| `helpers_example.yaml` | Generic/older helper examples |
| `FORUM_POST_LOCAL_EN.md` | Ready-to-adapt English forum post for the local-primary architecture |
| `SECURITY_PRIVACY.md` | Security and privacy notes |
| `VALIDATION.txt` | Validation notes |

---

# Security and privacy

Public examples intentionally contain no passwords, cloud credentials, Home Assistant tokens, API keys, webhook IDs, MAC addresses, e-mail addresses, personal names, local IP addresses or private hostnames.

Entity IDs are neutral placeholders.

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
- observed `hvac_action` during AUTO heating/cooling/idle
- target-temperature behavior
- auxiliary-heating behavior
- horizontal-vane behavior
- behavior during Internet or Home Assistant outages
- IR-remote synchronization behavior, if tested

Please do not post credentials, access tokens, e-mail addresses or private network information.

Issues, test results and improvements are welcome.
