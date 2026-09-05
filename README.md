# Home Assistant + Mitsubishi MELCloud: external room-temperature control

A tested Home Assistant approach for Mitsubishi Electric air conditioners using the **legacy MELCloud integration**, with external room-temperature sensors as the real comfort reference.

The project covers:

- one external sensor (bedroom example)
- multiple external sensors combined to a mean value (living-room example)
- compensated target temperatures instead of trusting only the indoor-unit sensor
- manual target changes from MELCloud/App/voice control
- feedback-loop protection
- two practical approaches to poor AUTO-mode behavior
- an optional 5-minute forced MELCloud refresh
- optional horizontal swing control

> **Not affiliated with Mitsubishi Electric or the Home Assistant project.**
> Back up your Home Assistant configuration before changing automations.

## Why this exists

In the tested installation, fixed **HEAT** and **COOL** modes were generally usable, but Mitsubishi's native **AUTO** behavior was much less satisfactory.

The core problem is the temperature reference. The indoor unit measures temperature at the unit itself, often high on a wall and in its own airflow. That can differ noticeably from the temperature where people actually sit, sleep, or live.

This matters especially in AUTO mode because the controller also has to decide **whether to heat or cool**.

The solution is to let Home Assistant use one or more external room sensors as the comfort reference and continuously compensate the Mitsubishi target.

## Important: legacy MELCloud vs MELCloud Home

This repository was tested with Home Assistant's **legacy `MELCloud` integration**.

Home Assistant also has a newer **`MELCloud Home` integration**. As of Home Assistant 2026.7+, its documented update behavior is different, so the 5-minute polling workaround in this repository is mainly relevant to legacy MELCloud.

Official docs:

- https://www.home-assistant.io/integrations/melcloud/
- https://www.home-assistant.io/integrations/melcloud_home/

## How the compensation works

### Fixed HEAT / COOL

Let:

- `desired` = the temperature you actually want in the room
- `external` = temperature from the external sensor or sensor average

The external error is:

```text
error = external - desired
```

The Mitsubishi target is then compensated:

```text
target = desired - error
```

The correction is limited to **±3 °C** and rounded to the nearest **0.5 °C**.

### Living room: native Mitsubishi AUTO

For the multi-sensor living-room setup, the tested configuration uses a different calculation while the climate entity is in `heat_cool`:

```text
Mitsubishi target =
Mitsubishi internal temperature - (external average - desired temperature)
```

This makes the Mitsubishi controller "see" approximately the same error that the external room average sees.

The source configuration also avoids continuously forcing HEAT/Cool in normal operation.

## AUTO-mode strategies used here

### Living room: multiple sensors + corrected native AUTO

When switched from OFF to `heat_cool`:

- external average >= desired + 0.5 °C -> start in COOL
- external average <= desired - 0.5 °C -> start in HEAT
- within ±0.5 °C -> stay in AUTO

If HEAT or COOL is selected, it remains active for **15 minutes**, then the unit returns to native Mitsubishi AUTO (`heat_cool`), where the dynamic target correction continues.

### Bedroom: one sensor + Home Assistant HEAT/COOL switching

The single-sensor example uses a more conservative replacement for native AUTO:

- if currently HEAT and the room stays >= desired + 1.0 °C for 15 min -> switch to COOL
- if currently COOL and the room stays <= desired - 1.0 °C for 15 min -> switch to HEAT

That provides a **2 °C total deadband** and a **15-minute stability delay**, preventing rapid mode changes.

## Files

| File | Purpose |
| --- | --- |
| `helpers_example.yaml` | Example desired-temperature helpers, feedback-loop helpers, and multi-sensor mean |
| `living_room_multi_sensor.yaml` | Multi-sensor living-room control + manual sync + improved AUTO start |
| `bedroom_single_sensor.yaml` | Single-sensor bedroom control + manual sync + HA-controlled HEAT/COOL switching |
| `melcloud_refresh_5min.yaml` | Optional forced legacy-MELCloud refresh every 5 minutes |
| `optional_horizontal_swing.yaml` | Optional device-dependent horizontal swing automation |

## 1. Create the helpers

The easiest method is the Home Assistant UI.

For the desired-temperature and last-automatic-target values create **Number helpers**.

For a room with several temperature sensors, create:

**Settings -> Devices & services -> Helpers -> Create helper -> Min/Max**

Choose:

- input entities: your room-temperature sensors
- type: **Mean**

Home Assistant's Min/Max helper can calculate the mean of multiple sensors.

Alternatively use the example in `helpers_example.yaml`.

## 2. Replace the example entity IDs

### Living room

Replace:

```text
climate.living_room
sensor.living_room_temperature_average
input_number.desired_temperature_living_room
input_number.last_automatic_target_living_room
```

### Bedroom

Replace:

```text
climate.bedroom
sensor.bedroom_temperature
input_number.desired_temperature_bedroom
input_number.last_automatic_target_bedroom
```

No IP address, password, API key, access token, MAC address, webhook ID, e-mail address or other credential is required in these examples.

## 3. Install the automations

You can either:

1. create automations in the Home Assistant UI and use **Edit in YAML**, or
2. merge the list entries into your YAML automation configuration.

Home Assistant's current automation YAML format uses `triggers`, `conditions`, and `actions`.

After saving/reloading the automations, a full Home Assistant restart is normally not required.

## 4. Initialize the helper values

Before the first test:

- set `desired_temperature_*` to your real comfort target
- set `last_automatic_target_*` to a valid temperature inside the unit's normal range

After the controller has run once, it maintains the `last_automatic_target_*` helper automatically.

## 5. Manual changes from MELCloud/App/voice control

The manual-sync automations watch the climate entity's `temperature` attribute.

If the new value differs from both:

1. the last target written automatically by Home Assistant, and
2. the current desired-temperature helper,

the change is treated as a genuine manual request and copied into the desired-temperature helper.

That is the feedback-loop protection.

## 6. Reduce legacy MELCloud update latency

The legacy MELCloud integration can feel slow when a setpoint is changed outside Home Assistant.

The tested workaround is `homeassistant.update_entity` on the MELCloud climate entity every five minutes:

```yaml
triggers:
  - trigger: time_pattern
    minutes: "/5"

actions:
  - action: homeassistant.update_entity
    target:
      entity_id: climate.living_room
```

`/5` is **clock aligned**: `:00`, `:05`, `:10`, `:15`, etc. It does not wait exactly five minutes after your change.

### Test result

In the tested setup, a MELCloud target change to **22.5 °C** was visible in Home Assistant at **19:10:02**, the next five-minute polling boundary. The previous logged value had been **21.5 °C at 19:05:09**.

The same `update_entity` action also worked immediately when the automation was manually executed.

The tested setup **left MELCloud's normal polling enabled** and simply added an extra 5-minute refresh.

Home Assistant also documents `homeassistant.update_entity` as the standard way to define custom polling behavior:

https://www.home-assistant.io/common-tasks/general/#defining-a-custom-polling-interval

Do not poll aggressively. Five minutes is the interval tested here. If you see rate-limit or API errors, increase it.

## 7. Failure behavior

### Living room

- In native AUTO, no new correction is sent if either the external average or Mitsubishi internal temperature is invalid.
- In fixed HEAT/COOL, if the external average is invalid, the desired temperature is used as a safe fallback.

### Bedroom

- If the external sensor is invalid but Mitsubishi's internal temperature is still valid, the desired temperature is sent without external compensation.
- If both are invalid, no new target is sent.

## 8. Optional swing automation

`optional_horizontal_swing.yaml` is included because it was part of the tested larger setup.

It enables horizontal swing when the room-average error is at least **0.7 °C** for one minute and returns to the center position once the error is at most **0.3 °C** for one minute.

Horizontal swing modes are device-dependent. Check the values supported by your own climate entity before enabling this automation.

## Security and privacy

The public examples intentionally contain:

- no private or public IP addresses
- no passwords
- no MELCloud credentials
- no Home Assistant long-lived tokens
- no API keys
- no webhook IDs
- no MAC addresses
- no e-mail addresses
- no personal names

Entity IDs are generic placeholders.

Before posting your own configuration publicly, search it for credentials and unique device/network identifiers.

## Tested source configuration

The public examples were distilled from a working Home Assistant configuration dated **2026-09-05**. House-specific logic unrelated to the Mitsubishi temperature-control method (extra heaters, holiday mode, FRITZ!DECT scheduling, etc.) was intentionally removed so the examples are easier to reuse.

## Feedback

If you test this on another Mitsubishi/MELCloud setup, please open an issue or discussion with:

- Home Assistant version
- legacy MELCloud or MELCloud Home
- Mitsubishi indoor-unit model
- external sensor type
- whether fixed HEAT/COOL and AUTO behave as expected

Please do **not** post passwords, tokens, e-mail addresses, public endpoints or other secrets.
