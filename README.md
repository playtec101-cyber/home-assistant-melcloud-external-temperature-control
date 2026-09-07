# Home Assistant + Mitsubishi MELCloud / MELCloud Home: external room-temperature control

A Home Assistant example for Mitsubishi Electric air conditioners that uses an external room-temperature sensor as the real comfort reference instead of relying only on the sensor inside the indoor unit.

The repository contains examples for both:

- the legacy `MELCloud` integration
- the newer `MELCloud Home` integration

The current MELCloud Home example is:

`melcloud_home_external_temperature_control_public.yaml`

> Not affiliated with Mitsubishi Electric or the Home Assistant project.

> Back up your Home Assistant configuration before changing automations.

---

## Current MELCloud Home strategy

The current public example keeps the user-selected Mitsubishi HVAC mode unchanged:

- `AUTO` stays `AUTO`
- `HEAT` stays `HEAT`
- `COOL` stays `COOL`

Home Assistant changes only the Mitsubishi target temperature.

The real room error is:

```text
error = external room temperature - desired room temperature
```

The Mitsubishi target is then calculated from the unit's own `current_temperature` and the correction derived from the external sensor.

### AUTO mode

AUTO intentionally uses fixed 0.5 °C correction steps:

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

Example:

```text
external = 23.5 °C
desired  = 23.0 °C
internal = 23.5 °C

error      = +0.5 °C
correction = +0.5 °C
target     = 23.5 - 0.5 = 23.0 °C
```

The Mitsubishi remains in native AUTO mode.

### Fixed HEAT / COOL

HEAT and COOL use the same `±0.25 °C` neutral zone.

Outside that neutral zone the correction is linear 1:1:

```text
correction = external - desired
target = internal - correction
```

### Limits and rounding

The calculated target is:

- limited to the climate entity's supported `min_temp` / `max_temp`
- rounded to the nearest `0.5 °C`
- limited to a maximum AUTO correction of `±3.0 °C`

---

## MELCloud / app / voice-control synchronization

Manual temperature changes made through MELCloud, MELCloud Home, a supported app or a connected voice-control path are treated as real user requests.

The public example watches the climate entity's `temperature` attribute.

If the new device target differs from both:

1. the last target automatically written by Home Assistant, and
2. the current desired-temperature helper,

the new value is copied into the desired-temperature helper.

This preserves external control while preventing a feedback loop.

The controller stores its own automatic target **before** sending the cloud command, so the following device update can be recognized as an automatic correction rather than a new user request.

External changes are applied when the MELCloud integration reports the changed setpoint back to Home Assistant. Cloud update latency can therefore still be visible.

---

## Main example entities

The public YAML uses neutral placeholder entity IDs. Replace them with the IDs from your own installation.

### Main room

```text
climate.hauptraum_ac
sensor.hauptraum_referenztemperatur
input_number.wunschtemperatur_hauptraum
input_number.letzte_automatische_zieltemperatur_hauptraum
```

### Second room

```text
climate.nebenraum_ac
sensor.nebenraum_referenztemperatur
input_number.wunschtemperatur_nebenraum
input_number.letzte_automatische_zieltemperatur_nebenraum
```

### General helper / weather entity

```text
input_boolean.abwesenheit
weather.home
```

---

## Optional auxiliary-heating example

The same public YAML also contains generalized examples for optional auxiliary heating.

Placeholder entities:

```text
climate.zusatzheizung_1
climate.zusatzheizung_2
input_boolean.zusatzheizung_freigabe_hauptraum

switch.zusatzheizer_strom
sensor.zusatzheizer_referenztemperatur
input_boolean.zusatzheizer_automatisch
input_datetime.zusatzheizer_manuell_bis
```

The included example logic contains:

- 25-minute auxiliary-heating release after sustained heating demand
- outside-temperature checks
- summer lockout
- day/night targets when the main AC is off
- away-mode safety shutdown
- separate auxiliary-heater control
- a manual 4-hour runtime limit

The temperature limits and schedules in this part are example values. Review them before use.

If you only want the MELCloud external-temperature controller, remove the auxiliary-heating / away sections and keep the MELCloud room-control sections.

---

## Optional horizontal swing

The public YAML includes an optional horizontal-vane automation for the main room.

Example behavior:

```text
absolute room error >= 0.7 °C for 1 minute
-> horizontal swing

absolute room error <= 0.3 °C for 1 minute
-> centre
```

The tested MELCloud Home horizontal center value is:

```text
centre
```

Remove this automation if you do not want Home Assistant to control the horizontal vane.

Check the actual swing values exposed by your own climate entity before enabling it.

---

## Required helpers

Create Number helpers for the desired room temperatures and the last automatically written targets.

Example placeholders:

```text
input_number.wunschtemperatur_hauptraum
input_number.wunschtemperatur_nebenraum
input_number.letzte_automatische_zieltemperatur_hauptraum
input_number.letzte_automatische_zieltemperatur_nebenraum
```

Use ranges that cover the supported target-temperature range of your climate units.

For a room with several physical sensors, use a mean/average sensor as the external reference.

---

## Installation

1. Back up your Home Assistant configuration.
2. Create the required helpers.
3. Create or choose the external room-temperature sensor(s).
4. Replace the placeholder entity IDs in the public YAML.
5. Remove optional sections you do not use.
6. Check that your climate entity exposes:
   - `current_temperature`
   - `temperature`
   - `min_temp`
   - `max_temp`
   - the HVAC modes you intend to use
7. Add the automations to Home Assistant.
8. Reload automations or restart Home Assistant.
9. Set the desired-temperature helpers to your normal comfort targets.
10. Test one room and one control path at a time.

---

## MELCloud Home polling

The current MELCloud Home example does not force `homeassistant.update_entity`.

The `/5` time-pattern trigger inside the controller is only a regulation/retry safety check.

It is not a forced cloud refresh.

---

## Duplicate-write protection

Cloud updates can arrive with some delay.

To reduce unnecessary repeated writes, the controller stores the last target automatically sent by Home Assistant.

A new target is sent when:

- the calculated target has changed
- the HVAC mode has changed
- or the 5-minute retry check sees that the device still reports a different target

---

## Failure behavior

If the external room temperature or the Mitsubishi `current_temperature` is invalid or unavailable, no new compensated target is calculated.

The automation does not invent a replacement room temperature.

---

## Original Mitsubishi infrared remote

The original Mitsubishi infrared remote was not tested in the source installation because it is not normally used there.

The tested control paths are:

- Home Assistant
- MELCloud / MELCloud Home
- supported app controls
- voice control

No claim is made about how quickly or reliably changes made with the original IR remote are synchronized back through the indoor unit, MELCloud Home and Home Assistant.

Feedback from users who regularly use the original remote is welcome.

---

## Legacy MELCloud files

Older repository files remain available for users of the legacy MELCloud integration.

These include:

```text
living_room_multi_sensor.yaml
bedroom_single_sensor.yaml
melcloud_refresh_5min.yaml
optional_horizontal_swing.yaml
```

Do not copy legacy examples blindly into MELCloud Home because mode names and vane values can differ.

---

## Files

| File | Purpose |
| --- | --- |
| `melcloud_home_external_temperature_control_public.yaml` | Current MELCloud Home example with fixed AUTO correction steps, manual MELCloud/app/voice synchronization, auxiliary-heating examples and optional horizontal swing |
| `helpers_example.yaml` | Generic helper examples |
| `living_room_multi_sensor.yaml` | Legacy MELCloud multi-sensor example |
| `bedroom_single_sensor.yaml` | Legacy MELCloud bedroom example |
| `melcloud_refresh_5min.yaml` | Optional forced refresh for legacy MELCloud only |
| `optional_horizontal_swing.yaml` | Legacy/device-dependent horizontal swing example |
| `SECURITY_PRIVACY.md` | Security and privacy notes |
| `VALIDATION.txt` | Validation notes |

---

## Security and privacy

The public examples intentionally contain no:

- passwords
- MELCloud credentials
- MELCloud Home credentials
- Home Assistant access tokens
- API keys
- webhook IDs
- MAC addresses
- e-mail addresses
- personal names
- local IP addresses
- private hostnames

Entity IDs are generic placeholders.

Before publishing your own adapted configuration, search it again for credentials and unique device or network identifiers.

---

## Feedback

Useful feedback includes:

- Home Assistant version
- legacy MELCloud or MELCloud Home
- Mitsubishi indoor-unit model
- external sensor type
- single sensor or room average
- AUTO behavior
- fixed HEAT/COOL behavior
- MELCloud/app/voice synchronization behavior
- horizontal-vane behavior
- approximate cloud/state update delay
- IR-remote behavior, if tested

Please do not post credentials, access tokens, API keys, e-mail addresses, public endpoints or private network information.

Issues, test results and improvements are welcome.
