# Home Assistant + Mitsubishi Electric: external room-temperature control

Home Assistant examples for Mitsubishi Electric air conditioners that use **external room-temperature sensors as the real comfort reference** instead of relying only on the sensor inside the indoor unit.

This repository now documents two primary architectures:

1. **Local-primary control with MELCloud Home as fallback** — recommended when a compatible local adapter/integration is available.
2. **MELCloud Home primary control** — cloud-based variant.

Legacy MELCloud examples are kept for older installations.

> Not affiliated with Mitsubishi Electric, Home Assistant, or the authors of the local custom integration.

> Back up your Home Assistant configuration before changing integrations, entity IDs or automations.

---

## New: local Mitsubishi control with MELCloud Home as fallback

Step-by-step guide:

`LOCAL_CONTROL_WITH_MELCLOUD_FALLBACK.md`

Tested concept:

```text
PRIMARY
Home Assistant -> local LAN/WLAN -> Mitsubishi Wi-Fi adapter -> indoor unit

FALLBACK
MELCloud Home -> Mitsubishi cloud -> same indoor unit
```

The tested local custom integration is:

[pymitsubishi/homeassistant-mitsubishi](https://github.com/pymitsubishi/homeassistant-mitsubishi)

It supports Mitsubishi units with the MAC-577IF-2E / MAC-577IF2-E Wi-Fi adapter and exposes climate control, target temperature, HVAC mode, fan control and vane control locally.

### Why there is no duplicated 1,100-line controller file

The tested local climate entity exposes the same relevant HVAC states used by the current MELCloud Home controller (`auto`, `heat`, `cool`, `off`) and the same standard Home Assistant climate services.

Therefore the **control logic is shared** with:

`melcloud_home_external_temperature_control_public.yaml`

For local-primary operation:

- point the main-room and second-room climate placeholders at the **local Mitsubishi climate entities**;
- keep MELCloud Home configured only as a fallback/manual path;
- in the optional horizontal-vane automation use local `center` instead of MELCloud Home `centre`.

This avoids maintaining two almost identical large controller files that could drift apart over time.

### Why this architecture is useful

- A MELCloud API outage does not stop the Home Assistant controller.
- An Internet/DSL outage does not stop local Home Assistant climate control.
- If Home Assistant or its server is unavailable but Internet/MELCloud still works, MELCloud Home remains a separate fallback path.
- MELCloud Home can remain available on phones/tablets for manual control.
- Manual MELCloud target changes are seen again by the local adapter state and can be copied into the Home Assistant desired-temperature helper.
- Changes made with the original IR remote were verified to propagate back to the local Home Assistant entity and to MELCloud Home.

The complete local controller and the complete MELCloud controller should **not** both be active against the same devices at the same time. Choose one primary controller.

---

## MELCloud Home primary variant

Current complete public example:

`melcloud_home_external_temperature_control_public.yaml`

This variant uses Home Assistant's MELCloud Home climate entities as the primary control path.

---

## Core control strategy

The current complete controller keeps the user-selected Mitsubishi HVAC mode unchanged:

- `AUTO` stays `AUTO`
- `HEAT` stays `HEAT`
- `COOL` stays `COOL`

Home Assistant changes only the target temperature.

The real room error is:

```text
error = external room temperature - desired room temperature
```

### AUTO mode

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

### Fixed HEAT / COOL

HEAT and COOL use the same `±0.25 °C` neutral zone. Outside that zone the correction is linear 1:1:

```text
correction = external - desired
target = internal - correction
```

The calculated target is limited to the climate entity's supported `min_temp` / `max_temp` and rounded to the nearest `0.5 °C`.

---

## Manual target synchronization and feedback-loop protection

The controller stores the last target that Home Assistant wrote automatically.

A device target change is treated as a genuine user request only when it differs from both:

1. the last automatically written target, and
2. the current desired-temperature helper.

That value is then copied into the desired-temperature helper.

This prevents the controller's own compensation target from being interpreted as a new user request.

In the **local-primary architecture**, a manual target change from MELCloud Home reaches the physical unit first. The local integration polls the adapter and reports the changed device target back to Home Assistant, where the same feedback-loop protection applies.

The same path was verified for the original IR remote: the indoor unit updates its state, the MAC-577IF2-E receives that state, Home Assistant picks up the new target locally, and MELCloud Home is updated as well.

---

## Local-primary migration pattern

A convenient migration from an existing MELCloud Home setup is:

```text
old MELCloud entity:
climate.room -> climate.room_melcloud

new local entity:
climate.generated_local_name -> climate.room
```

This lets existing automations continue using the old primary entity ID while the underlying control path becomes local.

Alternatively, keep explicit names such as:

```text
climate.hauptraum_ac_local
climate.nebenraum_ac_local
```

and replace the placeholders in the public controller YAML accordingly.

---

## Horizontal vane values

The two integrations expose a different spelling for the horizontal center position in the tested setup:

- MELCloud Home: `centre`
- local `pymitsubishi/homeassistant-mitsubishi`: `center`

Both use `swing` for horizontal swing in the tested setup.

Always check the values exposed by your own climate entity before enabling optional vane automation.

---

## Optional auxiliary-heating example

The complete public YAML also contains generalized optional logic for:

- two auxiliary thermostat/climate entities
- 25-minute heating-demand release
- outdoor-temperature checks
- day/night targets
- away-mode shutdown
- a separate auxiliary heater
- automatic hysteresis
- a manual 4-hour runtime limit
- a 23:00 safety shutdown event

These temperatures, schedules and entity IDs are examples. Review them before use.

If you only need the Mitsubishi external-temperature controller, remove the optional auxiliary-heating and away-mode sections.

---

## Installation summary

1. Back up Home Assistant.
2. Choose **one** primary architecture: local or MELCloud Home.
3. Create/choose external room-temperature sensors.
4. Create the desired-temperature and last-automatic-target Number helpers.
5. Replace the neutral placeholder entity IDs in `melcloud_home_external_temperature_control_public.yaml`.
6. For local-primary operation, point the climate placeholders to the local Mitsubishi entities and change optional horizontal center from `centre` to `center`.
7. Check supported HVAC modes, `current_temperature`, target-temperature range and vane values.
8. Add the automations to Home Assistant.
9. Reload automations or restart Home Assistant.
10. Test one room and one control path at a time.

For the local-primary architecture, follow `LOCAL_CONTROL_WITH_MELCLOUD_FALLBACK.md` first.

---

## Original Mitsubishi infrared remote

The original Mitsubishi IR remote was tested with the local-primary setup.

Verified signal path:

```text
IR remote -> indoor-unit control board -> internal interface/CN105 -> MAC-577IF2-E
```

The changed target was then reflected in the local Home Assistant climate entity, the desired-temperature helper through the existing manual-target synchronization automation, and MELCloud Home.

In the tested setup, the local Home Assistant value followed the IR change within seconds.

The IR remote therefore remains a direct control path that does not depend on the Home Assistant server or Internet access.

---

## Files

| File | Purpose |
| --- | --- |
| `LOCAL_CONTROL_WITH_MELCLOUD_FALLBACK.md` | Local-primary architecture, migration and test guide |
| `melcloud_home_external_temperature_control_public.yaml` | Complete current controller; use local or MELCloud climate entities as described above |
| `helpers_example.yaml` | Generic helper examples |
| `living_room_multi_sensor.yaml` | Legacy MELCloud multi-sensor example |
| `bedroom_single_sensor.yaml` | Legacy MELCloud bedroom example |
| `melcloud_refresh_5min.yaml` | Optional forced refresh for legacy MELCloud only |
| `optional_horizontal_swing.yaml` | Legacy/device-dependent horizontal swing example |
| `FORUM_POST_LOCAL_EN.md` | Ready-to-adapt English forum post for the local-primary architecture |
| `SECURITY_PRIVACY.md` | Security and privacy notes |
| `VALIDATION.txt` | Validation notes |

---

## Security and privacy

The public examples intentionally contain no passwords, cloud credentials, Home Assistant tokens, API keys, webhook IDs, MAC addresses, e-mail addresses, personal names, local IP addresses or private hostnames.

Entity IDs are neutral placeholders.

Before publishing your own adapted configuration, scan it again for credentials and unique network/device identifiers.

---

## Feedback

Useful feedback includes:

- Home Assistant version
- local integration version or MELCloud/MELCloud Home variant
- Mitsubishi indoor-unit model
- Wi-Fi adapter model
- external sensor type
- AUTO / HEAT / COOL behavior
- MELCloud-to-local synchronization delay
- horizontal-vane behavior
- behavior during Internet or Home Assistant outages
- IR-remote synchronization behavior

Please do not post credentials, access tokens, e-mail addresses or private network information.

Issues, test results and improvements are welcome.
