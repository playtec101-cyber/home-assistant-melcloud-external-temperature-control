# Home Assistant + Mitsubishi MELCloud / MELCloud Home: external room-temperature control

A Home Assistant approach for Mitsubishi Electric air conditioners that uses external room-temperature sensors as the real comfort reference instead of relying only on the temperature sensor inside the indoor unit.

The repository contains examples for both:

- the **legacy `MELCloud` integration**
- the newer **`MELCloud Home` integration** available in Home Assistant 2026.7+

The current MELCloud Home example uses a **target-temperature offset**. Home Assistant does not overwrite the Mitsubishi room sensor. Instead it shifts the Mitsubishi target temperature so the indoor unit reacts to the temperature error measured by the external room sensor.

> **Not affiliated with Mitsubishi Electric or the Home Assistant project.**
>
> Back up your Home Assistant configuration before changing automations.

---

## Current MELCloud Home strategy

The current public MELCloud Home example is:

`melcloud_home_external_temperature_control_public.yaml`

The main design rules are:

- **AUTO stays AUTO**
- **HEAT stays HEAT**
- **COOL stays COOL**
- Home Assistant changes only the Mitsubishi **target temperature**
- one external room sensor or a room-sensor average is used as the real comfort reference
- AUTO uses a stronger external-temperature offset with factor **1.5**
- fixed HEAT and COOL use factor **1.0**
- correction is limited to **±3.0 °C**
- calculated targets are rounded to **0.5 °C**
- inside a **±0.25 °C neutral zone**, the artificial offset is removed
- duplicate cloud writes are suppressed
- a 5-minute automation check is retained as a retry/safety check
- no separate 5-minute forced MELCloud Home refresh is used

The Mitsubishi controller still performs the actual compressor and fan control.

---

## Why this exists

The indoor unit measures temperature at or near the indoor unit itself. Depending on mounting position and airflow, that value can differ noticeably from the temperature where people actually sit, sleep or live.

For example:

```text
external room sensor = 23.8 °C
desired room temperature = 22.0 °C
Mitsubishi internal temperature = 22.0 °C
```

The real room is still about 1.8 °C too warm, even though the Mitsubishi's own sensor is already close to the desired temperature.

The purpose of the automation is to make the Mitsubishi react to that real room error without taking over the compressor or fan logic itself.

---

## MELCloud Home: target-temperature offset

Home Assistant cannot write a fake `current_temperature` value into MELCloud Home.

Instead, it creates an equivalent control effect by shifting the Mitsubishi target temperature.

Let:

```text
desired  = desired real room temperature
external = external room temperature
internal = Mitsubishi current_temperature
```

The real room error is:

```text
error = external - desired
```

### AUTO mode

In native Mitsubishi AUTO mode, the current public example uses:

```text
virtual_error = error * 1.5
target = internal - virtual_error
```

The factor **1.5** makes the virtual temperature error stronger in AUTO. This helps prevent the unit from reducing output too early while the real room is still noticeably away from the desired temperature.

Example:

```text
external = 23.8 °C
desired  = 22.0 °C
internal = 22.0 °C

error         = 1.8 °C
virtual_error = 1.8 * 1.5 = 2.7 °C
target        = 22.0 - 2.7 = 19.3 °C
rounded       = 19.5 °C
```

The Mitsubishi remains in **AUTO**. Home Assistant does not force it into COOL or HEAT.

### Fixed HEAT / COOL

In explicit HEAT or COOL mode, the factor remains **1.0**:

```text
target = internal - (external - desired)
```

So the user-selected HVAC mode remains untouched.

---

## Neutral zone

When the external room temperature is already very close to the desired room temperature, the automation removes the artificial offset.

Current neutral zone:

```text
desired - 0.25 °C
to
desired + 0.25 °C
```

Inside that zone:

```text
correction = 0
target = Mitsubishi internal temperature
```

This allows native Mitsubishi AUTO to reduce output naturally near the real room target.

---

## Limits and rounding

To avoid extreme target shifts:

```text
maximum correction = ±3.0 °C
```

The final target is also clamped to the climate entity's supported minimum and maximum values and rounded to the nearest:

```text
0.5 °C
```

---

## AUTO remains AUTO

The current MELCloud Home example intentionally does **not** automatically change HVAC mode.

If the user selects:

```text
AUTO
```

the unit remains:

```text
AUTO
```

If the user selects:

```text
HEAT
```

the unit remains:

```text
HEAT
```

If the user selects:

```text
COOL
```

the unit remains:

```text
COOL
```

This is important because Mitsubishi's native AUTO mode can use its own compressor and fan modulation behavior.

---

## MELCloud Home vs legacy MELCloud

This repository originally documented Home Assistant's legacy `MELCloud` integration.

The newer MELCloud Home integration differs in several important details.

### Native AUTO mode

Legacy MELCloud may expose native Mitsubishi AUTO as:

```text
heat_cool
```

MELCloud Home exposes it as:

```text
auto
```

### Horizontal vane center

In the tested MELCloud Home entity, horizontal vane positions included:

```text
auto
swing
left
left_centre
centre
right_centre
right
```

The horizontal center position is therefore:

```text
centre
```

instead of a legacy numeric value such as `3`.

---

## MELCloud Home polling

MELCloud Home already performs regular cloud polling through the Home Assistant integration.

The current MELCloud Home example therefore does **not** use the old optional:

```yaml
action: homeassistant.update_entity
```

five-minute forced-refresh workaround.

The automation can still contain a normal:

```yaml
trigger:
  - trigger: time_pattern
    minutes: "/5"
```

This is only a regulation/retry check. It is **not** a forced cloud refresh.

---

## Duplicate-write protection

Cloud APIs can report changes with some delay.

To avoid repeatedly sending the same target temperature while MELCloud Home is still updating, the public example uses a helper that stores the last target automatically sent by Home Assistant.

Example helpers:

```text
input_number.living_room_last_automatic_target
input_number.bedroom_last_automatic_target
```

A new target is sent when:

- the calculated target has actually changed
- the user has changed HVAC mode
- or the 5-minute retry check sees that the device still reports a different target

This reduces unnecessary cloud commands without preventing a later retry.

---

## Required example entities

The public file uses generic placeholders.

### Living room

```text
climate.living_room_heat_pump
sensor.living_room_external_temperature
input_number.living_room_desired_temperature
input_number.living_room_last_automatic_target
```

### Bedroom

```text
climate.bedroom_heat_pump
sensor.bedroom_external_temperature
input_number.bedroom_desired_temperature
input_number.bedroom_last_automatic_target
```

Replace these with the entity IDs from your own installation.

No password, API key, access token, MAC address, e-mail address or local IP address is required in these automation examples.

---

## Creating the desired-temperature helpers

Create Number helpers in Home Assistant for the real desired room temperatures.

For example:

```text
input_number.living_room_desired_temperature
input_number.bedroom_desired_temperature
```

Also create helpers for the last automatic target:

```text
input_number.living_room_last_automatic_target
input_number.bedroom_last_automatic_target
```

Use ranges that cover the supported target-temperature range of your climate units.

---

## Using multiple room sensors

For a room with several sensors, create an average/mean sensor and use that as the external room-temperature reference.

For example:

```text
sensor.living_room_external_temperature
```

can represent the average of several physical room sensors.

The important point is that the sensor used by the automation should represent the temperature in the actual occupied room area, not the temperature close to the indoor unit.

---

## Optional horizontal swing automation

The public MELCloud Home YAML includes an optional horizontal vane automation.

Current example behavior:

```text
absolute room error >= 0.7 °C for 1 minute
-> horizontal swing

absolute room error <= 0.3 °C for 1 minute
-> centre
```

The standard Home Assistant action is:

```yaml
action: climate.set_swing_horizontal_mode
target:
  entity_id: climate.living_room_heat_pump
data:
  swing_horizontal_mode: centre
```

Delete the optional swing automation if you do not want Home Assistant to control horizontal vanes.

Before enabling it, check the actual values exposed by your own climate entity.

---

## Original Mitsubishi infrared remote

The original Mitsubishi infrared remote was **not tested** in the source installation because it is not normally used there.

The tested control paths are:

- Home Assistant
- MELCloud / MELCloud Home
- supported app controls
- voice control

No claim is made about how quickly or reliably changes made with the original IR remote are synchronized back through:

```text
indoor unit
-> MELCloud Home
-> Home Assistant
```

Feedback from users who regularly use the original Mitsubishi remote is welcome.

---

## Legacy MELCloud files

Older repository files remain available for users of the legacy MELCloud integration.

These include examples such as:

```text
living_room_multi_sensor.yaml
bedroom_single_sensor.yaml
melcloud_refresh_5min.yaml
optional_horizontal_swing.yaml
```

The legacy files should not be copied blindly into MELCloud Home because mode names and vane values can differ.

---

## Files

| File | Purpose |
| --- | --- |
| `melcloud_home_external_temperature_control_public.yaml` | Current MELCloud Home example with AUTO factor 1.5, external room-temperature offset, duplicate-write protection and optional horizontal swing |
| `helpers_example.yaml` | Generic helper examples |
| `living_room_multi_sensor.yaml` | Legacy MELCloud multi-sensor example |
| `bedroom_single_sensor.yaml` | Legacy MELCloud bedroom example |
| `melcloud_refresh_5min.yaml` | Optional forced refresh for legacy MELCloud only |
| `optional_horizontal_swing.yaml` | Legacy/device-dependent horizontal swing example |
| `SECURITY_PRIVACY.md` | Security and privacy notes |
| `VALIDATION.txt` | Validation notes |

---

## Installation

1. Create the desired-temperature helpers.
2. Create the last-automatic-target helpers.
3. Create or choose the external room-temperature sensor or average sensor.
4. Replace the generic entity IDs in the public YAML with your own IDs.
5. Check that your climate entity exposes:
   - `current_temperature`
   - `temperature`
   - `min_temp`
   - `max_temp`
   - `auto`, `heat` and/or `cool` as required
6. Add the automations to Home Assistant.
7. Reload automations or restart Home Assistant.
8. Set the desired-temperature helpers to your normal comfort targets.
9. Test one room at a time.

---

## Failure behavior

If the external room temperature or Mitsubishi internal temperature is invalid or unavailable, the public controller does not invent a replacement room value.

No new compensated target is calculated until valid data is available again.

This is intentional.

---

## What is intentionally not included

The public MELCloud Home file focuses on the reusable Mitsubishi/external-temperature concept.

Installation-specific logic is intentionally omitted, including:

- FRITZ!DECT radiator control
- auxiliary electric heaters
- weather-based heating lockouts
- holiday/away logic
- house-specific schedules
- local network configuration
- personal device names
- account information

These should remain separate from the reusable public MELCloud Home example.

---

## Security and privacy

The public examples intentionally contain:

- no passwords
- no MELCloud credentials
- no MELCloud Home credentials
- no Home Assistant long-lived access tokens
- no API keys
- no webhook IDs
- no MAC addresses
- no e-mail addresses
- no personal names
- no local IP addresses

Entity IDs are generic placeholders.

Before publishing your own adapted configuration, search it again for credentials and unique device or network identifiers.

---

## Current testing status

The current MELCloud Home strategy was developed from live testing after migration from legacy MELCloud.

The main observed design goal is:

> use the external room temperature as the real comfort reference while preserving Mitsubishi native AUTO behavior.

The current AUTO offset factor of **1.5** is based on practical testing of the observed behavior. It is not claimed to be an official Mitsubishi parameter.

Different rooms, units, sensor placement and buildings may require different tuning.

If the unit remains too aggressive near the target, reduce the factor.

If the unit reduces output too early while the real room is still too far from target, a somewhat stronger factor may be useful.

Make changes gradually and observe full heating/cooling cycles before tuning further.

---

## Feedback

If you test the MELCloud Home example on another Mitsubishi setup, useful information includes:

- Home Assistant version
- legacy MELCloud or MELCloud Home
- Mitsubishi indoor-unit model
- external sensor type
- single sensor or multi-sensor average
- AUTO behavior
- fixed HEAT/COOL behavior
- whether the 1.5 AUTO factor works well in your room
- horizontal vane behavior
- whether IR-remote changes are reflected correctly in Home Assistant
- approximate cloud/state update delay

Please do **not** post:

- passwords
- access tokens
- API keys
- e-mail addresses
- public endpoints
- private network information
- other credentials or secrets

Issues, test results and improvements are welcome.
