# Local-primary controller design notes (2026-09-10)

This document explains the reasoning behind the current **local-primary Mitsubishi controller** and why it differs from the older MELCloud Home example.

Matching focused public YAML:

`local_primary_hvac_action_aux_heating_example.yaml`

Matching helper example:

`local_primary_helpers_example.yaml`

The public files are intentionally neutralized. They contain generic Home Assistant entity IDs and no local IP addresses, MAC addresses, accounts, tokens, e-mail addresses, private hostnames or personal names.

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

The goal is that normal control no longer depends on the Mitsubishi cloud. MELCloud Home may remain configured as a manual fallback, but a second full automation controller must not run against the same unit at the same time.

## 2. Native AUTO is kept native

The current controller does **not** simulate AUTO by switching Home Assistant between HEAT and COOL.

The selected mode remains unchanged:

- `auto` stays `auto`
- `heat` stays `heat`
- `cool` stays `cool`

Home Assistant only compensates the Mitsubishi target temperature from the difference between the external room reference and the desired room temperature.

This preserves Mitsubishi's own inverter, compressor, fan and AUTO decision logic.

## 3. Why `hvac_action` matters

A climate entity state such as `auto` only tells us the selected HVAC mode. It does not say whether the unit is currently heating, cooling or waiting.

The local integration exposes the current operating action through Home Assistant `hvac_action`. The auxiliary-heating rule is deliberately strict:

```text
HEAT                  -> auxiliary heating may be allowed
AUTO + heating        -> auxiliary heating may be allowed
AUTO + cooling        -> auxiliary heating blocked
AUTO + idle           -> auxiliary heating blocked
AUTO + unknown/None   -> auxiliary heating blocked (fail-safe)
COOL                  -> auxiliary heating blocked
OFF                   -> auxiliary heating blocked, except the separate winter reserve
```

This prevents an auxiliary radiator from fighting the air conditioner while AUTO has internally chosen cooling.

Long-term reliability of `hvac_action` should still be observed on each real installation over multiple AUTO heating/cooling/idle cycles.

## 4. Desired temperature is authoritative

An earlier design copied target-temperature changes reported by the climate device back into the desired-temperature helper.

That approach was removed from the local-primary controller.

Reason: asynchronous device echoes and automatic compensation writes can race each other. An intermediate device target can then be misclassified as a user request and create rapid target bouncing or a feedback loop.

The safer rule is:

> The desired-temperature helper is authoritative.

The controller may change the Mitsubishi **device target**, but a device target change never changes the desired-temperature helper.

If a user wants to change the real desired room temperature, a dashboard, script, voice command or service should change the desired-temperature helper directly.

The `last_automatic_target` helper remains only to mark the controller's own writes, avoid unnecessary repeats and support the periodic target-delivery retry. It no longer authorizes device-target-to-helper back-synchronization.

## 5. Temperature-compensation algorithm

The room error is:

```text
error = external_room_temperature - desired_room_temperature
```

### AUTO

AUTO uses fixed 0.5 °C correction steps:

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

The focused public example includes two optional auxiliary radiator thermostat entities. They are subordinate to the Mitsubishi system.

A release is possible only when:

- away mode is off;
- Mitsubishi is in `heat`, or in `auto` with `hvac_action == heating`;
- the room reference is at least `0.5 °C` below desired;
- outdoor temperature is `<= 20 °C`;
- outdoor temperature is not warmer than the room reference;
- the condition remains true for 25 minutes.

The 25-minute delay prevents short recovery periods from immediately calling for radiator assistance.

When released, the example auxiliary thermostat target is:

```text
desired room temperature - 1.0 °C
```

A larger 1.5 °C separation can be too passive after the 25-minute delay and 0.5 °C demand threshold are already applied. A 1.0 °C separation keeps Mitsubishi clearly primary while allowing useful assistance. This is still a tuning value, not a universal rule.

## 7. Why the 25-minute `for:` is intentionally not restart-persistent

Home Assistant resets a trigger `for:` duration when automations are reloaded or Home Assistant restarts.

For this use case that behavior is accepted deliberately. After a restart, the auxiliary radiator release must prove the heating-demand condition for 25 minutes again. The only consequence is delayed auxiliary heat; it cannot cause auxiliary heating to start too early.

A restart-persistent implementation would require another timestamp/helper and more state handling. The current design chooses the simpler fail-safe behavior.

## 8. Strict separation between AC-active and AC-off radiator logic

Two automations control mutually exclusive scopes:

```text
main AC != off  -> normal auxiliary-radiator logic
main AC == off  -> winter-reserve logic
```

This separation is intentional. Earlier versions could let overlapping automations evaluate the same state and both send off/frost-protection commands to the radiator thermostats.

The current design removes that overlap at the architecture level rather than relying on action ordering.

## 9. Night reserve when the main AC is OFF

When the main Mitsubishi is OFF, the radiator thermostats may provide a low-level winter reserve.

Example night window:

```text
23:00 -> 08:00
```

If outdoor temperature is valid and `<= 20 °C`, the example sets the auxiliary radiator thermostats to:

```text
18.0 °C
```

Home Assistant is the source of truth for this target. Vendor comfort/eco schedules are not required for the logic.

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

## 11. Separate switch-controlled auxiliary heater in the fuller production design

The fuller production design can additionally contain a switch-controlled heater with its own room-reference sensor. The focused public YAML intentionally does **not** include that heater block.

A suitable production rule is:

- away mode off;
- `heat`, or `auto + heating`;
- valid heater-reference temperature;
- optional outdoor-temperature limits;
- automatic stop when permission disappears.

A manual/voice-started run can be limited to four hours with an absolute deadline stored in `input_datetime`, for example:

```yaml
timestamp: "{{ as_timestamp(now()) + 14400 }}"
```

Using an absolute UNIX timestamp is intentional because the stored deadline survives midnight and Home Assistant restarts. A fixed time such as 23:00 can remain a one-time hard shutdown event.

## 12. Away mode

The focused public YAML uses away mode as a hard permission condition for auxiliary heating.

A fuller production configuration may additionally perform explicit shutdown/retry actions for Mitsubishi units, radiator thermostats, auxiliary heaters and release/reserve markers. `continue_on_error` is useful so one unavailable device does not prevent the remaining safety actions.

## 13. Horizontal vane spelling

The tested local integration uses:

```text
center
```

for horizontal center. MELCloud Home may use:

```text
centre
```

This spelling difference is integration-specific and is a common migration trap.

## 14. Safety retry

The main target controller periodically checks whether the climate entity actually reports the calculated target. If it differs, the target is sent again.

This is a target-delivery safety retry, not the older forced MELCloud `update_entity` workaround.

## 15. Required neutral helper entities

See `local_primary_helpers_example.yaml`.

The focused public controller expects:

```text
input_number.desired_temperature_main_room
input_number.last_automatic_target_main_room
input_boolean.aux_radiator_release
input_boolean.aux_winter_reserve
input_boolean.away_mode
```

The two auxiliary radiator climate entities, main-room climate entity, external room-temperature sensor and weather entity are neutral placeholders that must also be replaced.

## 16. Validation status

For the public 2026-09-10 focused local-primary example:

- YAML syntax validated;
- 6 automations;
- 6 unique automation IDs;
- no bare `| float` conversions without defaults;
- generic entity IDs only;
- no private IP addresses, MAC addresses, e-mail addresses, account names, API tokens or private hostnames included.

This is still an example configuration, not a universal thermostat design. Review every threshold, schedule and entity ID before deployment.
