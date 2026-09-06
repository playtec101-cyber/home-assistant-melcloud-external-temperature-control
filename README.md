# Home Assistant + Mitsubishi MELCloud / MELCloud Home: external room-temperature control

A tested Home Assistant approach for Mitsubishi Electric air conditioners using external room-temperature sensors as the real comfort reference instead of relying only on the temperature sensor inside the indoor unit.

The project now contains examples for both:

- the **legacy `MELCloud` integration**
- the newer **`MELCloud Home` integration** available in Home Assistant 2026.7+

The project covers:

- one external room sensor
- multiple external sensors combined into a mean value
- compensated Mitsubishi target temperatures
- improved behavior in native Mitsubishi AUTO mode
- manual target changes from MELCloud / MELCloud Home / app / voice control
- feedback-loop protection
- weather-dependent AUTO neutral-zone adjustment
- optional horizontal swing control
- an optional 5-minute forced refresh for **legacy MELCloud only**

> **Not affiliated with Mitsubishi Electric or the Home Assistant project.**
>
> Back up your Home Assistant configuration before changing automations.

---

## Why this exists

In the tested installation, fixed **HEAT** and **COOL** modes were generally usable, but Mitsubishi's native **AUTO** behavior was much less satisfactory.

The main problem is the temperature reference.

The indoor unit measures temperature at the unit itself, often high on a wall and inside or close to its own airflow. That temperature can differ noticeably from the temperature where people actually sit, sleep, or live.

This matters especially in AUTO mode because the Mitsubishi controller must decide both:

- what target temperature to regulate toward
- whether heating or cooling is required

The approach used here lets Home Assistant treat one or more external room-temperature sensors as the comfort reference and compensate the target sent to the Mitsubishi unit.

The Mitsubishi controller still performs the actual compressor and fan control.

---

## Legacy MELCloud vs MELCloud Home

This repository originally documented Home Assistant's **legacy `MELCloud` integration**.

A tested **MELCloud Home** example is now also included:

`melcloud_home_external_temperature_control_public.yaml`

### Important differences

With the legacy MELCloud integration, native Mitsubishi AUTO is represented as:

```text
heat_cool
```

With MELCloud Home, native Mitsubishi AUTO is represented as:

```text
auto
```

In the tested MELCloud Home climate entity, the following HVAC modes were exposed:

```text
off
heat
cool
auto
dry
fan_only
```

MELCloud Home also exposes the values required by this compensation approach, including:

- target temperature
- current room temperature
- supported HVAC modes
- fan-speed modes
- vertical vane modes
- horizontal vane modes, where supported by the physical unit

In the tested unit, horizontal vane positions were exposed as:

```text
auto
swing
left
left_centre
centre
right_centre
right
```

The MELCloud Home example therefore uses:

```text
centre
```

for the horizontal center position instead of a legacy numeric position such as `3`.

---

## MELCloud Home polling behavior

Home Assistant documents MELCloud Home as polling the cloud service approximately every **60 seconds**.

Because of that, the separate 5-minute `homeassistant.update_entity` workaround previously used with legacy MELCloud is **not included in the MELCloud Home example**.

The MELCloud Home example can still contain 5-minute automation triggers for regulation or safety checks.

Those are normal Home Assistant control checks and are **not forced MELCloud refreshes**.

Official Home Assistant documentation:

- https://www.home-assistant.io/integrations/melcloud/
- https://www.home-assistant.io/integrations/melcloud_home/

---

## How the external-temperature compensation works

### Fixed HEAT / COOL

Let:

- `desired` = the room temperature you actually want
- `external` = the external room sensor or room-sensor average

The room error is:

```text
error = external - desired
```

The Mitsubishi target is then compensated:

```text
target = desired - error
```

Example:

```text
desired = 23.0 °C
external = 22.0 °C

error = -1.0 °C
target = 24.0 °C
```

Home Assistant therefore asks the Mitsubishi unit for a higher target until the external room sensor reaches the real desired temperature.

The correction is limited to **±3 °C** and rounded to the nearest **0.5 °C**.

---

## Native Mitsubishi AUTO compensation

Native AUTO needs slightly different handling because the Mitsubishi unit itself decides whether to heat or cool.

The tested AUTO calculation is:

```text
Mitsubishi target =
Mitsubishi internal temperature - (external room temperature - desired temperature)
```

For a room using several external sensors:

```text
Mitsubishi target =
Mitsubishi internal temperature - (external room average - desired temperature)
```

This makes the Mitsubishi controller approximately "see" the same temperature error that the external room sensor sees.

The automation does **not** continuously force HEAT and COOL during normal AUTO operation.

---

## AUTO neutral zone

A small neutral zone is used around the desired room temperature.

Without a neutral zone, small sensor differences and 0.5 °C target rounding can cause the calculated Mitsubishi target to move enough to provoke unnecessary heating or cooling even though the real room temperature is already close to the desired value.

### Default neutral zone

When the outside temperature is close to the desired indoor temperature, or when no valid outside value is available:

```text
desired - 0.25 °C
to
desired + 0.25 °C
```

Inside this range, the automation uses the Mitsubishi internal temperature as the Mitsubishi target instead of deliberately creating a heating or cooling offset.

This keeps native Mitsubishi AUTO active while avoiding an unnecessary correction impulse near the real room target.

---

## Weather-dependent AUTO neutral zone

The MELCloud Home example can use an outside-temperature entity to shift the AUTO thresholds.

The outside temperature does **not** directly decide whether the Mitsubishi unit heats or cools.

The external **indoor room temperature** remains the primary control value.

### Outside clearly warmer than desired indoor temperature

If:

```text
outside >= desired + 0.5 °C
```

the neutral range becomes:

```text
desired - 0.5 °C
to
desired + 0.25 °C
```

This delays unnecessary heating while still allowing cooling somewhat earlier.

### Outside clearly colder than desired indoor temperature

If:

```text
outside <= desired - 0.5 °C
```

the neutral range becomes:

```text
desired - 0.25 °C
to
desired + 0.5 °C
```

This allows heating somewhat earlier while delaying unnecessary cooling.

### Important

The weather value only shifts the neutral thresholds.

It never replaces the external indoor room sensor and never acts as the room temperature.

---

## AUTO start behavior

When native Mitsubishi AUTO is started from OFF, the tested larger configuration can temporarily choose a clear HEAT or COOL direction based on the external room temperature.

Typical behavior:

```text
external >= desired + 0.5 °C -> start in COOL
external <= desired - 0.5 °C -> start in HEAT
within the start band         -> remain in native AUTO
```

If HEAT or COOL is selected, it can remain active for a short initialization period before returning to native Mitsubishi AUTO.

In the tested source configuration, this initialization period was **15 minutes**.

The purpose is to give the unit an unambiguous initial direction without permanently replacing Mitsubishi's native AUTO logic.

---

## Files

| File | Purpose |
| --- | --- |
| `melcloud_home_external_temperature_control_public.yaml` | Consolidated MELCloud Home example with external-sensor compensation, native AUTO, weather-dependent neutral zone, manual-target sync and optional horizontal swing |
| `helpers_example.yaml` | Example desired-temperature helpers, feedback-loop helpers and multi-sensor mean |
| `living_room_multi_sensor.yaml` | Legacy MELCloud multi-sensor living-room example |
| `bedroom_single_sensor.yaml` | Legacy MELCloud single-sensor bedroom example |
| `melcloud_refresh_5min.yaml` | Optional forced refresh for **legacy MELCloud only** |
| `optional_horizontal_swing.yaml` | Legacy/device-dependent optional horizontal swing example |
| `SECURITY_PRIVACY.md` | Security and privacy notes |
| `VALIDATION.txt` | Validation notes for the public example files |

The older files remain intentionally available for users who still use Home Assistant's legacy MELCloud integration.

---

## 1. Create the helpers

The easiest method is the Home Assistant UI.

For the desired-temperature and last-automatic-target values, create **Number helpers**.

Typical generic entity IDs used by the examples are:

```text
input_number.desired_temperature_living_room
input_number.last_automatic_target_living_room

input_number.desired_temperature_bedroom
input_number.last_automatic_target_bedroom
```

For a room with several temperature sensors, create a mean/average sensor.

For example:

**Settings -> Devices & services -> Helpers -> Create helper -> Min/Max**

Choose:

- your room-temperature sensors as input entities
- **Mean** as the calculation type

A generic example entity is:

```text
sensor.living_room_temperature_average
```

Alternatively, adapt the example in:

`helpers_example.yaml`

---

## 2. Replace the example entity IDs

All public files use generic placeholders.

You must replace them with the entity IDs from your own Home Assistant installation.

### Living room

Typical placeholders are:

```text
climate.living_room
sensor.living_room_temperature_average
input_number.desired_temperature_living_room
input_number.last_automatic_target_living_room
```

### Bedroom

Typical placeholders are:

```text
climate.bedroom
sensor.bedroom_temperature
input_number.desired_temperature_bedroom
input_number.last_automatic_target_bedroom
```

### Weather entity

The MELCloud Home public example also uses:

```text
weather.home
```

Replace that with your own weather entity if you want to use the weather-dependent AUTO neutral zone.

No IP address, password, API key, access token, MAC address, webhook ID or e-mail address is required in these automation examples.

---

## 3. Check the HVAC mode names

This is particularly important when choosing between the legacy and MELCloud Home examples.

### Legacy MELCloud

Native Mitsubishi AUTO may appear as:

```text
heat_cool
```

### MELCloud Home

Native Mitsubishi AUTO appears as:

```text
auto
```

Do not blindly replace files between the two integrations without checking the supported modes of your own climate entity.

---

## 4. Check the available climate attributes

Before adapting the examples, inspect the climate entity in Home Assistant and check which attributes your unit actually exposes.

The MELCloud Home setup used for this project exposed:

```text
hvac_modes
min_temp
max_temp
fan_modes
swing_modes
swing_horizontal_modes
current_temperature
temperature
fan_mode
swing_mode
swing_horizontal_mode
```

Not every Mitsubishi model necessarily exposes exactly the same feature set.

The external-temperature compensation requires at least:

- a usable climate entity
- target temperature
- current Mitsubishi room temperature for native AUTO compensation
- an external indoor room-temperature sensor or sensor average

Horizontal swing control is optional.

---

## 5. Install the automations

You can either:

1. create automations in the Home Assistant UI and use **Edit in YAML**, or
2. merge the automation list entries into your YAML automation configuration.

The examples use Home Assistant's current automation YAML structure with:

```text
triggers
conditions
actions
```

After saving the configuration, reload the automations or restart Home Assistant if required by your setup.

---

## 6. Initialize the helper values

Before the first test:

- set `desired_temperature_*` to your normal comfort target
- set `last_automatic_target_*` to a valid temperature within the unit's supported range

After the controller has run successfully, it maintains the `last_automatic_target_*` helper automatically.

---

## 7. Manual changes from MELCloud / MELCloud Home / app / voice control

The manual-sync automations monitor the climate entity's target `temperature`.

If a newly reported target differs from both:

1. the last target written automatically by Home Assistant, and
2. the current desired-temperature helper,

the change is treated as a genuine manual request and copied into the desired-temperature helper.

This prevents Home Assistant's own compensated target from being fed back into the desired-temperature setting.

In other words, it is a simple feedback-loop protection mechanism.

---

## 8. Original Mitsubishi infrared remote control

The original Mitsubishi **infrared remote controls were not tested as part of this project**.

They are not used in the tested installation.

The tested system is operated through Home Assistant, MELCloud / MELCloud Home, supported apps and voice control.

Because the original IR remote is not used, this project does **not** currently confirm how changes made directly with the Mitsubishi infrared remote are reflected back through:

```text
indoor unit
-> MELCloud / MELCloud Home
-> Home Assistant
```

For example, the following have not been verified with the original IR remote:

- target-temperature changes
- HVAC mode changes
- fan-speed changes
- vertical vane changes
- horizontal vane changes
- timing and reliability of the resulting Home Assistant state update

This does **not** mean that the IR remote is known to be incompatible.

It simply means that this control path has not been tested in the source installation.

Feedback from users who regularly use the original Mitsubishi IR remote together with MELCloud Home is therefore especially welcome.

---

## 9. Legacy MELCloud 5-minute refresh

This section applies only to the **legacy MELCloud integration**.

The legacy integration can sometimes feel slow when a target is changed outside Home Assistant.

The tested workaround was to call:

```yaml
action: homeassistant.update_entity
target:
  entity_id: climate.living_room
```

every five minutes:

```yaml
triggers:
  - trigger: time_pattern
    minutes: "/5"
```

`/5` is **clock aligned**:

```text
:00
:05
:10
:15
...
```

It does not mean "five minutes after the last change."

### Tested legacy result

In the original tested setup, a MELCloud target change to **22.5 °C** became visible in Home Assistant at **19:10:02**, the next five-minute polling boundary.

The previously logged value had been **21.5 °C at 19:05:09**.

The same `homeassistant.update_entity` call also worked when manually executed.

The normal legacy MELCloud polling remained enabled; the five-minute action was only an additional refresh.

Do not poll cloud APIs aggressively.

### MELCloud Home users

Do **not** copy this workaround automatically to MELCloud Home.

MELCloud Home already polls the API approximately every 60 seconds, so the separate five-minute forced-refresh automation is not part of the new MELCloud Home example.

---

## 10. Failure behavior

### Native AUTO

If either:

- the required external room temperature, or
- the Mitsubishi internal room temperature

is invalid or unavailable, no new AUTO compensation target is calculated.

The automation does not invent a replacement room value.

### Fixed HEAT / COOL

If the external room sensor is unavailable while using an explicit HEAT or COOL mode, the normal desired temperature can be used as a safe fallback.

The Mitsubishi unit then regulates using its own internal sensor until the external sensor becomes available again.

---

## 11. Optional horizontal swing control

Horizontal vane control depends on the integration and the physical Mitsubishi unit.

### MELCloud Home

For supported units, MELCloud Home exposes named horizontal vane modes.

In the tested unit the available modes were:

```text
auto
swing
left
left_centre
centre
right_centre
right
```

The public example uses:

```text
swing
```

while the room is sufficiently far from the desired temperature, and:

```text
centre
```

when the room is close to the desired temperature.

The example uses separate thresholds so that the vane does not constantly switch around one exact temperature boundary.

Before enabling this feature, check the horizontal swing modes actually exposed by your own climate entity.

The standard Home Assistant action used is:

```yaml
action: climate.set_swing_horizontal_mode
target:
  entity_id: climate.living_room
data:
  swing_horizontal_mode: centre
```

### Legacy MELCloud

The older `optional_horizontal_swing.yaml` file remains available because vane representation can differ between the legacy integration and individual devices.

---

## 12. What is intentionally not included

The public files focus on the reusable Mitsubishi/external-temperature concept.

Installation-specific logic has intentionally been removed, including examples such as:

- additional electric heaters
- radiator or thermostat scheduling
- FRITZ!DECT-specific control
- holiday/away logic
- house-specific sensors
- network configuration
- local IP addresses
- personal device names
- account information

This keeps the examples easier to understand and safer to publish.

---

## Security and privacy

The public examples intentionally contain:

- no private or public IP addresses
- no passwords
- no MELCloud credentials
- no MELCloud Home credentials
- no Home Assistant long-lived access tokens
- no API keys
- no webhook IDs
- no MAC addresses
- no e-mail addresses
- no personal names

Entity IDs are generic placeholders.

Before publishing your own adapted configuration, search it again for credentials and unique device or network identifiers.

Never publish your MELCloud / MELCloud Home login credentials or Home Assistant access tokens.

---

## Tested source configurations

The original public examples were distilled from a working Home Assistant configuration tested with the **legacy MELCloud integration** in September 2026.

The installation was subsequently migrated to **MELCloud Home**, and the reusable external-room-temperature concept was tested again.

The migration confirmed that the basic compensation principle remains usable because MELCloud Home continues to expose the important climate information needed by the controller, including:

- current room temperature
- target temperature
- HEAT
- COOL
- native AUTO

The tested installation also confirmed the MELCloud Home representation of native AUTO as:

```text
auto
```

instead of the legacy:

```text
heat_cool
```

The tested installation exposed named horizontal vane positions, including:

```text
centre
```

instead of the numeric center value used in the previous legacy configuration.

The separate legacy 5-minute forced refresh was removed after migration to MELCloud Home.

The public MELCloud Home example reflects these integration differences without exposing the installation-specific heating, holiday, network or device-control logic of the source system.

### Control methods tested

The installation is normally controlled using:

- Home Assistant
- MELCloud / MELCloud Home
- supported app controls
- voice control

### Control method not tested

The original Mitsubishi **infrared remote control is not used in the source installation and was therefore not tested**.

No claim is made that IR-remote changes are or are not synchronized correctly with MELCloud Home and Home Assistant.

That remains an open test case for users who use the original Mitsubishi remote.

---

## Known observation points

The current control strategy intentionally remains relatively simple.

Possible future refinements include:

- stateful hysteresis around the outside-temperature regime thresholds
- a more stateful hold strategy inside the AUTO neutral zone
- a hard minimum interval between cloud target commands

These are currently observation points rather than confirmed defects.

The examples already avoid unnecessary target commands by comparing the calculated target with both:

- the last target written automatically
- the target currently reported by the Mitsubishi climate entity

Changes should therefore be based on actual traces or observed command behavior rather than adding complexity preemptively.

---

## Feedback

If you test these examples on another Mitsubishi setup, please include:

- Home Assistant version
- **legacy MELCloud** or **MELCloud Home**
- Mitsubishi indoor-unit model
- external sensor type
- single sensor or multi-sensor average
- whether fixed HEAT/COOL behaves as expected
- whether native AUTO behaves as expected
- whether horizontal vane control works on your unit
- whether changes made with the original Mitsubishi IR remote are correctly reflected in Home Assistant
- approximate update delay between a device/app/remote change and Home Assistant, if relevant

Reports from users who use the original Mitsubishi infrared remote are particularly useful because that control path was **not tested in the source installation**.

Please do **not** post:

- passwords
- access tokens
- API keys
- e-mail addresses
- public endpoints
- private network information
- other credentials or secrets

Issues, test results and improvements are welcome.
