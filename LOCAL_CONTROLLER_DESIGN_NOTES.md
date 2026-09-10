# Local-primary controller design notes (2026-09-10)

This document explains the reasoning behind the current **local-primary Mitsubishi controller** and why it now differs from the older MELCloud Home example.

The matching public YAML is:

`local_primary_hvac_action_external_temperature_control_public.yaml`

The file is intentionally neutralized. It contains only generic Home Assistant entity IDs and no local IP addresses, MAC addresses, accounts, tokens, e-mail addresses, private hostnames or personal names.

## 1. Architecture: local control first, MELCloud Home only as fallback

Primary path:

```text
Home Assistant -> local LAN/WLAN -> MAC-577IF2-E -> Mitsubishi indoor unit
```

Fallback/manual path:

```text
MELCloud Home -> Mitsubishi cloud -> Mitsubishi indoor unit
```

The local integration used for this design is:

`pymitsubishi/homeassistant-mitsubishi`

The goal is simple: normal control should keep working without depending on the Mitsubishi cloud. MELCloud Home may remain configured as a manual fallback, but a second full automation controller must not run against the same unit at the same time.

## 2. Native AUTO is kept native

The current controller does **not** simulate AUTO by switching Home Assistant between HEAT and COOL.

The user-selected mode remains unchanged:

- `auto` stays `auto`
- `heat` stays `heat`
- `cool` stays `cool`

Home Assistant only compensates the Mitsubishi target temperature from the difference between the external room reference and the desired room temperature.

This preserves Mitsubishi's own inverter, compressor, fan and AUTO decision logic.

## 3. Why `hvac_action` matters

A climate entity state such as `auto` only tells us the selected HVAC mode. It does not, by itself, say whether the unit is currently heating, cooling or waiting.

The local integration exposes the current operating action through the standard Home Assistant `hvac_action` attribute. Its implementation maps the Mitsubishi AUTO state to Home Assistant actions such as:

- `heating`
- `cooling`
- `idle`

That distinction is used only for **auxiliary heating permission**.

The rule is deliberately strict:

```text
HEAT                  -> auxiliary heating may be allowed
AUTO + heating        -> auxiliary heating may be allowed
AUTO + cooling        -> auxiliary heating blocked
AUTO + idle           -> auxiliary heating blocked
AUTO + unknown/None   -> auxiliary heating blocked (fail-safe)
COOL                  -> auxiliary heating blocked
OFF                   -> auxiliary heating blocked, except the separate winter reserve
```

This prevents an auxiliary radiator or electric heater from fighting the air conditioner while AUTO has internally chosen cooling.

Long-term reliability of `hvac_action` should still be observed on each real installation over multiple AUTO heating/cooling/idle cycles.

## 4. Desired temperature is authoritative

An earlier design copied target-temperature changes reported by the climate device back into the desired-temperature helper.

That approach was removed from the local-primary controller.

Reason: asynchronous device echoes and automatic compensation writes can race each other. Even with a helper storing the last automatic target, an intermediate device target can be misclassified as a user request and create a rapid target bounce or feedback loop.

The safer rule is therefore:

> The desired-temperature helper is authoritative.

The controller may change the Mitsubishi **device target**, but a device target change never changes the desired-temperature helper.

If a user wants to change the real desired room temperature, the UI, dashboard, voice command or service should change the desired-temperature helper directly.

The `last_automatic_target_*` helpers remain useful, but only to detect the controller's own writes, avoid unnecessary repeats and support the periodic safety retry. They are no longer used as permission for device-target-to-helper back-synchronization.

## 5. Temperature-compensation algorithm

The room error is:

```text
error = external_room_temperature - desired_room_temperature
```

### AUTO

AUTO uses fixed 0.5 °C correction steps, matching the 0.5 °C target resolution of the tested local Mitsubishi climate entity:

| Absolute room error | Correction magnitude |
| --- | ---: |
| `<= 0.25 °C` | `0.0 °C` |
| `> 0.25 to 0.75 °C` | `0.5 °C` |
| `> 0.75 to 1.25 °C` | `1.0 °C` |
| `> 1.25 to 1.75 °C` | `1.5 °C` |
| `> 1.75 to 2.25 °C` | `2.0 °C` |
| `> 2.25 to 2.75 °C` | `2.5 °C` |
| `> 2.75 °C` | `3.0 °C` |

The sign follows the direction of the room error.

### HEAT and COOL

HEAT and COOL use the same `±0.25 °C` neutral zone. Outside it, correction is linear 1:1:

```text
correction = external - desired
target = internal_sensor_temperature - correction
```

The result is clamped to the climate entity's supported range and rounded to 0.5 °C.

### Invalid values

If desired, external or internal temperatures are invalid, the controller does not calculate a new target. This is intentional fail-safe behavior.

All numeric template conversions use explicit `float(...)` defaults to avoid template errors during startup or temporary `unknown`/`unavailable` states.

## 6. Auxiliary radiator thermostats

The public example includes two optional auxiliary radiator thermostat entities.

They are intentionally subordinate to the Mitsubishi system.

### Start permission

A release is possible only when:

- away mode is off;
- Mitsubishi is in `heat`, or in `auto` with `hvac_action == heating`;
- the room reference is at least `0.5 °C` below the desired temperature;
- outdoor temperature is `<= 20 °C`;
- outdoor temperature is not warmer than the indoor room reference;
- the condition remains true for 25 minutes.

The 25-minute delay prevents short Mitsubishi recovery periods from immediately calling for radiator assistance.

### Auxiliary target offset

When released, the auxiliary thermostat target is:

```text
desired room temperature - 1.0 °C
```

Why 1.0 °C? The auxiliary emitters should help the primary system, not become the main thermostat. A larger 1.5 °C separation can be too passive after the 25-minute delay and 0.5 °C demand threshold are already applied. A 1.0 °C separation keeps Mitsubishi clearly primary while allowing useful assistance.

This is still an example tuning value; different buildings may prefer 0.5, 1.0 or 1.5 °C.

## 7. Why the 25-minute `for:` is intentionally not restart-persistent

Home Assistant resets a trigger `for:` duration when automations are reloaded or Home Assistant restarts.

For this particular use case that behavior is accepted deliberately.

After a restart, the auxiliary radiator release must prove the heating-demand condition for 25 minutes again. The only consequence is delayed auxiliary heat. It cannot cause auxiliary heating to start too early.

A restart-persistent implementation would require another timestamp/helper and more state handling. The current design chooses the simpler fail-safe behavior.

## 8. Strict separation between AC-on and AC-off radiator logic

Two automations control different, mutually exclusive scopes:

```text
main AC != off  -> normal auxiliary-radiator logic
main AC == off  -> winter-reserve logic
```

This separation is intentional. Earlier versions allowed overlapping automations to evaluate the same cooling state and both send `off`/frost-protection commands to the radiator thermostats.

The current design removes that overlap at the architecture level instead of relying on action ordering.

## 9. Night reserve when the main AC is OFF

When the main Mitsubishi is OFF, the radiator thermostats may provide a low-level winter reserve.

Night window:

```text
23:00 -> 08:00
```

If outdoor temperature is valid and `<= 20 °C`, the example sets both auxiliary radiator thermostats to:

```text
18.0 °C
```

This avoids a very cold room overnight without requiring the Mitsubishi to stay on.

The Home Assistant automation is the source of truth for this target. Vendor comfort/eco schedules should not be required for the logic.

## 10. Daytime winter reserve when the main AC is OFF

During the day, the radiator thermostats normally remain off when the Mitsubishi is off.

A separate reserve starts only if the room falls below:

```text
16.0 °C
```

Once started, it remains active until the room reaches:

```text
18.0 °C
```

A dedicated Boolean helper stores that reserve state. This gives a wide 16 -> 18 °C hysteresis and avoids rapid on/off cycling around the start threshold.

The separate electric auxiliary heater is **not** included in this AC-off reserve.

## 11. Separate electric auxiliary heater

The example also contains an optional switch-controlled heater with its own room-reference sensor.

Automatic operation requires:

- away mode off;
- `heat`, or `auto + heating`;
- a valid heater-reference temperature;
- outdoor temperature `<= 15 °C`;
- outdoor temperature not warmer than the heater-reference temperature.

It starts immediately below the desired temperature and stops at desired `+0.5 °C`, or when any permission condition disappears.

### Manual runtime limit

A manual/voice-started run is allowed only while real heating permission exists and is limited to four hours.

The end time is stored as an absolute UNIX timestamp:

```yaml
timestamp: "{{ as_timestamp(now()) + 14400 }}"
```

This is intentional. It survives midnight and Home Assistant restarts because the absolute deadline is stored in an `input_datetime` helper.

A 23:00 event remains a hard shutdown event. It is a one-time safety shutdown, not a permanent night lock.

## 12. Away-mode shutdown

Away mode shuts down:

- both Mitsubishi units;
- both auxiliary radiator thermostats;
- the separate auxiliary heater;
- auxiliary release/reserve markers.

The shutdown is retried periodically and uses `continue_on_error` where appropriate so that one unavailable device does not prevent the remaining shutdown actions.

The example also allows away mode to end automatically if a Mitsubishi unit is deliberately switched from OFF into an active mode. A transition from `unknown`/`unavailable` is not treated as a deliberate return.

## 13. Horizontal vane spelling

The tested local integration uses:

```text
center
```

for horizontal center.

MELCloud Home may use:

```text
centre
```

This small spelling difference is integration-specific and is a common migration trap.

## 14. Safety retry

The controller checks periodically whether the climate entity actually reports the calculated target. If it differs, the target is sent again.

This is a target-delivery safety retry. It is not the old forced MELCloud `update_entity` workaround, which is unnecessary for the local-primary path.

## 15. Required neutral helper entities

See `local_primary_helpers_example.yaml` for a matching example.

The local controller expects:

```text
input_number.wunschtemperatur_hauptraum
input_number.wunschtemperatur_nebenraum
input_number.letzte_automatische_zieltemperatur_hauptraum
input_number.letzte_automatische_zieltemperatur_nebenraum
input_boolean.zusatzheizung_freigabe_hauptraum
input_boolean.zusatzheizung_winterreserve_hauptraum
input_boolean.zusatzheizer_automatisch
input_boolean.abwesenheit
input_datetime.zusatzheizer_manuell_bis
```

The temperature sensors, climate entities, heater switch and weather entity must be replaced with entities from the target installation.

## 16. Validation status

For the public 2026-09-10 local-primary example:

- YAML syntax validated;
- 12 automations;
- 12 unique automation IDs;
- no bare `| float` conversions without defaults;
- generic entity IDs only;
- no private IP addresses, MAC addresses, e-mail addresses, account names, API tokens or private hostnames included.

This is still an example configuration, not a universal thermostat design. Review every threshold, schedule and entity ID before deployment.
