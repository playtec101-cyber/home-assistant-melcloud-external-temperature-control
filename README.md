# Home Assistant + Mitsubishi Electric: external room-temperature control

Home Assistant examples for Mitsubishi Electric air conditioners that use **external room-temperature sensors as the real comfort reference** instead of relying only on the sensor inside the indoor unit.

This repository documents **three different solution paths**. They represent different stages/architectures and are intentionally kept side by side so users can choose the approach that best fits their installation.

> Not affiliated with Mitsubishi Electric, Home Assistant, or the authors of the local custom integration.

> Back up your Home Assistant configuration before changing integrations, entity IDs or automations.

---

# The three solution paths

## 1. Legacy / simulated AUTO with Home Assistant deciding HEAT and COOL

This is the older approach for installations where native Mitsubishi AUTO behavior was not satisfactory.

Home Assistant uses the external room-temperature sensor(s) as the real reference and can decide whether the unit should run in:

```text
HEAT
or
COOL
```

instead of relying entirely on Mitsubishi's native AUTO decision.

The legacy examples include two related strategies:

- a direct Home-Assistant-controlled HEAT/COOL switching strategy with hysteresis/stability delay;
- an initialization strategy where Home Assistant can start in HEAT or COOL based on the external room error and later return to native AUTO.

Relevant legacy files include:

- `living_room_multi_sensor.yaml`
- `bedroom_single_sensor.yaml`
- `melcloud_refresh_5min.yaml`
- `optional_horizontal_swing.yaml`

These files are mainly for the **classic/legacy MELCloud integration** and should not be copied blindly into MELCloud Home because state names, polling behavior and vane values can differ.

### When this solution is useful

- You want Home Assistant to take more control over the heating/cooling decision.
- Native Mitsubishi AUTO behaves poorly in your room.
- You are still using the classic MELCloud integration or want to study the older logic.

---

## 2. MELCloud Home primary control with real Mitsubishi AUTO

This is the newer cloud-based controller.

The important difference from solution 1 is that Home Assistant **does not simulate AUTO by switching between HEAT and COOL**.

The user-selected Mitsubishi mode stays unchanged:

- `AUTO` stays `AUTO`
- `HEAT` stays `HEAT`
- `COOL` stays `COOL`

Home Assistant only adjusts the Mitsubishi target temperature from the error between the external room temperature and the desired room temperature.

Current complete public example:

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

### Fixed HEAT / COOL

HEAT and COOL use the same `±0.25 °C` neutral zone. Outside that neutral zone the correction is linear 1:1:

```text
correction = external - desired
target = internal - correction
```

The calculated target is limited to the climate entity's supported `min_temp` / `max_temp` and rounded to the nearest `0.5 °C`.

### Manual target synchronization

Manual target changes reported through MELCloud Home, app or supported voice-control paths can be copied into the desired-temperature helper.

A helper stores the **last automatic target** written by Home Assistant so the controller's own compensated target is not mistaken for a new user request. This is the feedback-loop protection.

### Limitation

This solution depends on the Mitsubishi cloud/API for the Home Assistant climate entity. If MELCloud Home or the Internet connection is unavailable, the Home Assistant control path is affected.

---

## 3. Local-primary Mitsubishi control with MELCloud Home as fallback

This is the current local-primary architecture.

Primary path:

```text
Home Assistant -> local LAN/WLAN -> MAC-577IF2-E -> indoor unit
```

Fallback/manual path:

```text
MELCloud Home -> Mitsubishi cloud -> indoor unit
```

The tested local custom integration is:

[pymitsubishi/homeassistant-mitsubishi](https://github.com/pymitsubishi/homeassistant-mitsubishi)

Step-by-step migration guide:

`LOCAL_CONTROL_WITH_MELCLOUD_FALLBACK.md`

### Why this differs from solution 2

The **regulation logic is essentially the same as solution 2**: native Mitsubishi AUTO stays AUTO, and Home Assistant adjusts the target temperature from the external room-temperature error.

The difference is the communication path. Normal automation commands no longer need the Mitsubishi cloud because Home Assistant talks directly to the MAC-577IF2-E over the local network.

MELCloud Home remains configured as a separate fallback/manual control path.

### Failure behavior

- Internet/DSL outage: local Home Assistant control continues as long as Home Assistant, LAN/WLAN and the adapter are available.
- MELCloud API outage: local Home Assistant control continues.
- Home Assistant/server outage with Internet OK: MELCloud Home remains available as a manual fallback.
- MELCloud Home can still be used from phones/tablets when the cloud path is available.

### Bidirectional synchronization tested

The following two directions were tested:

```text
Home Assistant local -> indoor unit -> MELCloud Home
```

and

```text
MELCloud Home -> indoor unit -> local Home Assistant climate entity
```

This allows a manual MELCloud target change to be seen by the local entity and then handled by the same desired-temperature / feedback-loop logic.

### Horizontal vane difference

In the tested setup:

```text
MELCloud Home center: centre
local integration center: center
```

Both used `swing` for horizontal swing.

### Entity-ID migration option

A convenient migration pattern is:

```text
old MELCloud entity:
climate.room -> climate.room_melcloud

new local entity:
climate.generated_local_name -> climate.room
```

Existing automations can then keep using `climate.room`, while the underlying control path becomes local.

Do **not** run two complete automation controllers against the same physical unit at the same time. MELCloud Home may remain configured as fallback, but only one main automation controller should be active.

---

# Original Mitsubishi infrared remote

The original Mitsubishi IR remote should continue to control the indoor unit directly in all three architectures because it talks to the indoor unit independently of Home Assistant.

However, this project has **not yet tested the return synchronization path from the IR remote**.

In particular, we do not yet claim as verified how quickly or reliably an IR-made change is reflected back into:

- the legacy MELCloud Home Assistant entity in solution 1;
- the MELCloud Home entity in solution 2;
- the local MAC-577IF2-E Home Assistant entity in solution 3;
- the desired-temperature helper;
- MELCloud Home after an IR change.

For solution 3, it is technically plausible that the changed indoor-unit state is reported through the MAC-577IF2-E and then read locally by Home Assistant, but this should be treated as **expected/possible, not tested**, until verified on the real installation.

A useful test is:

1. Change the target by 1 °C with the original IR remote.
2. Watch the active Home Assistant climate entity's `temperature` attribute.
3. Watch the desired-temperature helper.
4. Check whether MELCloud Home reaches the same value.
5. Record the approximate delay.

Feedback from users who test this path is welcome.

---

# Optional auxiliary-heating example

The complete current controller also contains generalized optional logic for:

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

---

# Installation summary

1. Back up Home Assistant.
2. Choose **one** of the three solution paths above.
3. Create or choose the external room-temperature sensor(s).
4. Create the desired-temperature and last-automatic-target Number helpers required by the selected controller.
5. Replace the neutral placeholder entity IDs in the example YAML.
6. Check supported HVAC states, `current_temperature`, target-temperature range and vane values.
7. For local-primary operation, point the climate placeholders to the local Mitsubishi entities and use `center` instead of MELCloud Home `centre` for optional horizontal center.
8. Add/reload the automations or restart Home Assistant.
9. Test one room and one control path at a time.

---

# Files

| File | Purpose |
| --- | --- |
| `melcloud_home_external_temperature_control_public.yaml` | Complete current real-AUTO controller used for MELCloud Home primary or adapted to local-primary control |
| `LOCAL_CONTROL_WITH_MELCLOUD_FALLBACK.md` | Local-primary architecture, migration and test guide |
| `living_room_multi_sensor.yaml` | Legacy MELCloud external-sensor / AUTO example |
| `bedroom_single_sensor.yaml` | Legacy MELCloud Home-Assistant-controlled HEAT/COOL example |
| `melcloud_refresh_5min.yaml` | Optional forced refresh for legacy MELCloud only |
| `optional_horizontal_swing.yaml` | Legacy/device-dependent horizontal swing example |
| `helpers_example.yaml` | Generic helper examples |
| `FORUM_POST_LOCAL_EN.md` | Ready-to-adapt English forum post for the local-primary architecture |
| `SECURITY_PRIVACY.md` | Security and privacy notes |
| `VALIDATION.txt` | Validation notes |

---

# Security and privacy

The public examples intentionally contain no passwords, cloud credentials, Home Assistant tokens, API keys, webhook IDs, MAC addresses, e-mail addresses, personal names, local IP addresses or private hostnames.

Entity IDs are neutral placeholders.

Before publishing your own adapted configuration, scan it again for credentials and unique network/device identifiers.

---

# Feedback

Useful feedback includes:

- Home Assistant version
- selected solution path (1, 2 or 3)
- local integration version or MELCloud/MELCloud Home variant
- Mitsubishi indoor-unit model
- Wi-Fi adapter model
- external sensor type
- AUTO / HEAT / COOL behavior
- MELCloud-to-local synchronization delay
- horizontal-vane behavior
- behavior during Internet or Home Assistant outages
- IR-remote synchronization behavior, if tested

Please do not post credentials, access tokens, e-mail addresses or private network information.

Issues, test results and improvements are welcome.
