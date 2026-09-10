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

## 3. What `hvac_action` actually tells us

The selected climate mode and the current operating action are different things.

The local integration exposes `hvac_action`. Its implementation first checks whether the compressor is operating. If it is not operating, the integration returns `idle`. Only while the compressor is operating does AUTO resolve to `heating` or `cooling` from Mitsubishi's internal AUTO state.

That means:

> `AUTO + idle` does **not** mean "the room no longer needs heat" and it does not tell us whether the previous active phase was heating or cooling. It means only that the compressor is currently not operating.

This matters for auxiliary heat. Treating `idle` as a hard heating block can switch auxiliary heat off exactly while an external room sensor still shows genuine heating demand.

The current permission model is therefore:

```text
HEAT                  -> auxiliary heat may be allowed
AUTO + heating        -> auxiliary heat may be allowed
AUTO + idle           -> neutral: auxiliary heat may be allowed if external demand conditions say so
AUTO + cooling        -> hard block
AUTO + unknown/None   -> hard block (fail-safe)
COOL / DRY / FAN      -> hard block
OFF                   -> normal auxiliary heat blocked; separate winter reserve may apply
```

`idle` is **not** interpreted as heating. It is merely no longer used as a blanket block. The external room-temperature, outdoor-temperature, timing and hysteresis rules still decide whether auxiliary heat actually starts.

## 4. Why `AUTO + cooling` remains a hard block

This state is unambiguous: the Mitsubishi is actively cooling.

Radiator or electric auxiliary heating must not run against it, regardless of the external heating thresholds. A transition from `idle`/`heating` to `cooling` therefore immediately invalidates auxiliary-heating permission.

An unclear AUTO action (`None`, switching, unavailable) is also fail-safe blocked because we cannot prove that heating assistance is safe.

## 5. Desired temperature is authoritative

An earlier design copied target-temperature changes reported by the climate device back into the desired-temperature helper.

That approach was removed from the local-primary controller.

Reason: asynchronous device echoes and automatic compensation writes can race each other. An intermediate device target can then be misclassified as a user request and create rapid target bouncing or a feedback loop.

The safer rule is:

> The desired-temperature helper is authoritative.

The controller may change the Mitsubishi **device target**, but a device target change never changes the desired-temperature helper.

If a user wants to change the real desired room temperature, a dashboard, script, voice command or service should change the desired-temperature helper directly.

The `last_automatic_target` helper remains only to mark the controller's own writes, avoid unnecessary repeats and support the periodic target-delivery retry. It no longer authorizes device-target-to-helper back-synchronization.

## 6. Temperature-compensation algorithm

The room error is:

```text
error = external_room_temperature - desired_room_temperature
```

### AUTO

AUTO currently keeps the established fixed 0.5 °C correction steps:

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

### Why the AUTO curve was not steepened after the latest test

During one test the external room reference was clearly above the desired temperature while the Mitsubishi remained `AUTO + idle` for a period. This initially looked like a stuck AUTO decision.

Without Home Assistant changing HVAC mode, the same unit later transitioned from `idle` to `cooling` while remaining in native AUTO. That observation is important: it shows that an `idle` period can simply be part of Mitsubishi's own AUTO/inverter timing rather than proof that the current offset curve is too weak.

Therefore the controller does **not** yet increase the AUTO correction curve and does not add an AUTO->COOL/HEAT override. More observations over longer periods are preferable before changing a curve that otherwise behaves correctly.

A useful diagnostic is to display:

```jinja2
{{ state_attr('climate.main_room_ac', 'hvac_action') }}
```

on a temporary Home Assistant dashboard Markdown card and compare it with external room temperature, desired temperature, internal Mitsubishi temperature and the compensated device target.

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

## 7. Auxiliary radiator thermostats

The focused public example includes two optional auxiliary radiator thermostat entities. They are subordinate to the Mitsubishi system.

A release is possible only when:

- away mode is off;
- Mitsubishi is in `heat`, or in `auto` with `hvac_action` equal to `heating` **or** `idle`;
- the room reference is at least `0.5 °C` below desired;
- outdoor temperature is valid and `<= 20 °C`;
- outdoor temperature is not warmer than the room reference;
- the condition remains true for 25 minutes.

The 25-minute delay prevents short recovery periods from immediately calling for radiator assistance.

When released, the example auxiliary thermostat target is:

```text
desired room temperature - 1.0 °C
```

A larger 1.5 °C separation can be too passive after the 25-minute delay and 0.5 °C demand threshold are already applied. A 1.0 °C separation keeps Mitsubishi clearly primary while allowing useful assistance. This is still a tuning value, not a universal rule.

### Why the 25-minute timer survives AUTO idle

The release template stays true when AUTO changes from `heating` to `idle`, provided all external demand conditions remain true. Therefore an ordinary `heating -> idle` transition does not by itself reset the 25-minute demand period.

A transition to `cooling`, an unclear AUTO action, invalid sensors, unsuitable outdoor conditions or loss of external heating demand makes the template false and correctly cancels the pending release.

## 8. Why the 25-minute `for:` is intentionally not restart-persistent

Home Assistant resets a trigger `for:` duration when automations are reloaded or Home Assistant restarts.

For this use case that behavior is accepted deliberately. After a restart, the auxiliary radiator release must prove the heating-demand condition for 25 minutes again. The only consequence is delayed auxiliary heat; it cannot cause auxiliary heating to start too early.

A restart-persistent implementation would require another timestamp/helper and more state handling. The current design chooses the simpler fail-safe behavior.

## 9. Strict separation between AC-active and AC-off radiator logic

Two automations control mutually exclusive scopes:

```text
main AC != off  -> normal auxiliary-radiator logic
main AC == off  -> winter-reserve logic
```

This separation is intentional. Earlier versions could let overlapping automations evaluate the same state and both send off/frost-protection commands to the radiator thermostats.

The current design removes that overlap at the architecture level rather than relying on action ordering.

## 10. Night reserve when the main AC is OFF

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

## 11. Daytime winter reserve when the main AC is OFF

During the day, the radiator thermostats normally remain off when the Mitsubishi is off.

A separate reserve starts only if:

- room temperature falls below `16.0 °C`;
- outdoor temperature is valid and `<= 20 °C`;
- away mode is off.

Once started, it remains active until the room reaches `18.0 °C`, the outdoor value becomes invalid/too warm, the main AC is switched on or away mode is enabled.

A dedicated Boolean helper stores the reserve state. This gives a wide 16 -> 18 °C hysteresis and avoids rapid on/off cycling around the start threshold.

## 12. Separate switch-controlled auxiliary heater in the fuller production design

The fuller production design can additionally contain a switch-controlled heater with its own room-reference sensor. The focused public YAML intentionally does **not** include that heater block.

The same safety interpretation applies:

```text
HEAT -> may run after its own demand conditions
AUTO + heating -> may run after its own demand conditions
AUTO + idle -> may run after its own external demand conditions
AUTO + cooling / unclear -> blocked
COOL / OFF -> blocked
```

A manual/voice-started run can be limited to four hours with an absolute deadline stored in `input_datetime`, for example:

```yaml
timestamp: "{{ as_timestamp(now()) + 14400 }}"
```

Using an absolute UNIX timestamp is intentional because the stored deadline survives midnight and Home Assistant restarts. A fixed time such as 23:00 can remain a one-time hard shutdown event.

## 13. Away mode

The focused public YAML uses away mode as a hard permission condition for auxiliary heating.

A fuller production configuration may additionally perform explicit shutdown/retry actions for Mitsubishi units, radiator thermostats, auxiliary heaters and release/reserve markers. `continue_on_error` is useful so one unavailable device does not prevent the remaining safety actions.

## 14. Horizontal vane spelling

The tested local integration uses:

```text
center
```

for horizontal center. MELCloud Home may use:

```text
centre
```

This spelling difference is integration-specific and is a common migration trap.

## 15. Safety retry

The main target controller periodically checks whether the climate entity actually reports the calculated target. If it differs, the target is sent again.

This is a target-delivery safety retry, not the older forced MELCloud `update_entity` workaround.

## 16. Required neutral helper entities

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

## 17. Validation status

For the public 2026-09-10 focused local-primary example:

- 6 automations;
- 6 unique automation IDs;
- no device-target-to-desired-helper back-synchronization;
- AUTO target-correction curve unchanged;
- AUTO `idle` is neutral for auxiliary-heating permission, not a hard block;
- AUTO `cooling` and unclear AUTO action are hard blocks;
- daytime and nighttime AC-OFF winter reserve require valid outdoor temperature `<= 20 °C`;
- numeric conversions use explicit defaults;
- generic entity IDs only;
- no private IP addresses, MAC addresses, e-mail addresses, account names, API tokens or private hostnames included.

This is still an example configuration, not a universal thermostat design. Review every threshold, schedule and entity ID before deployment.
