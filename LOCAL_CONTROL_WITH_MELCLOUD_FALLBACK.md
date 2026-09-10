# Local Mitsubishi control with MELCloud Home as fallback

This guide describes the current **local-primary** architecture and the matching public controller:

`local_primary_hvac_action_aux_heating_example.yaml`

Detailed design reasoning is in:

`LOCAL_CONTROLLER_DESIGN_NOTES.md`

## Goal

Normal control should not depend on the Mitsubishi cloud:

```text
Home Assistant -> local LAN/WLAN -> Mitsubishi Wi-Fi adapter -> indoor unit
```

MELCloud Home can stay configured as a separate fallback/manual path:

```text
MELCloud Home -> Mitsubishi cloud -> indoor unit
```

Do not keep two complete automation controllers active against the same physical unit.

## Tested local integration

The local integration used for this architecture is:

[pymitsubishi/homeassistant-mitsubishi](https://github.com/pymitsubishi/homeassistant-mitsubishi)

Home Assistant name: **Mitsubishi Air Conditioner**.

The project supports Mitsubishi units using the MAC-577IF-2E / MAC-577IF2-E Wi-Fi adapter and exposes standard Home Assistant climate control including target temperature, HVAC mode, fan/vane controls and `hvac_action`.

## 1. Back up first

Create a full Home Assistant backup before installing, renaming or replacing any climate entities or automations.

## 2. Give the Mitsubishi adapters stable local IP addresses

Use DHCP reservations in your router so each Mitsubishi Wi-Fi adapter keeps the same local address.

Do not publish those private addresses.

## 3. Install HACS and the local integration

Install HACS if needed, then install **Mitsubishi Air Conditioner** from HACS.

Repository:

```text
https://github.com/pymitsubishi/homeassistant-mitsubishi
```

Restart Home Assistant after installation.

## 4. Add and test each local Mitsubishi device

In Home Assistant:

```text
Settings -> Devices & services -> Add integration -> Mitsubishi Air Conditioner
```

Before migrating production automations, test:

- power on/off;
- target temperature;
- AUTO;
- HEAT;
- COOL;
- fan mode;
- horizontal vane values, if used.

For the tested local integration, horizontal center is `center`, not MELCloud Home `centre`.

## 5. Verify `hvac_action`

This is now important for auxiliary-heating safety.

In Developer Tools, inspect the climate entity while the selected HVAC mode is AUTO.

Observe the `hvac_action` attribute through several real cycles. The controller expects standard Home Assistant values such as:

```text
heating
cooling
idle
```

Auxiliary heating is permitted only for:

```text
heat
or
auto + hvac_action == heating
```

AUTO cooling, AUTO idle, unclear AUTO action, COOL and OFF do not give normal auxiliary-heating permission.

Do not assume long-term reliability from one state transition; observe several cycles on the actual hardware/firmware.

## 6. Keep native AUTO native

The local controller does not simulate AUTO by switching modes.

```text
AUTO stays AUTO
HEAT stays HEAT
COOL stays COOL
```

Home Assistant only compensates the target temperature from the external room-temperature error.

## 7. Desired-temperature helper is authoritative

The local-primary controller intentionally does not copy climate-device target changes back into the desired-temperature helper.

This differs from the older MELCloud Home example.

Reason: asynchronous device echoes and automatic target compensation can race each other. An intermediate target can be misclassified as user intent and create a target feedback loop.

Therefore:

```text
desired-temperature helper = user intent / source of truth
climate target              = compensated device target
```

Change the helper directly from dashboards, scripts, services or voice-control paths when you want to change the real desired room temperature.

The `last_automatic_target_*` helpers remain only for self-write detection, avoiding redundant commands and the periodic safety retry.

## 8. Create the matching helpers

Use:

`local_primary_helpers_example.yaml`

or create equivalent helpers in the Home Assistant UI.

The public controller expects a desired-temperature Number helper, a last-automatic-target Number helper, an auxiliary-heating release Boolean, a winter-reserve Boolean and an away-mode Boolean.

Additional optional helpers can be added when adapting the separate switch-controlled heater logic from the design notes.

## 9. Replace all neutral placeholders

Start from:

`local_primary_hvac_action_aux_heating_example.yaml`

Replace all placeholder climate, sensor, switch, helper and weather entities with entities from your installation.

Do not copy the example thresholds blindly. Review every number and schedule.

## 10. Auxiliary radiator logic

The example uses two optional radiator thermostat entities.

Normal auxiliary release requires:

- real heating permission (`heat` or `auto + heating`);
- room at least 0.5 °C below desired;
- outdoor <= 20 °C;
- outdoor not warmer than room;
- 25 minutes of stable demand.

After release, the example target is:

```text
desired - 1.0 °C
```

The 25-minute `for:` duration intentionally restarts after a Home Assistant restart or automation reload. For this design that is fail-safe because it can only delay auxiliary heat.

## 11. AC-OFF winter reserve

The public controller separates AC-OFF radiator behavior from the normal AC-active radiator logic.

That separation avoids duplicate commands and race-like overlap.

Example behavior while the main AC is OFF:

- 23:00–08:00: 18 °C night reserve when outdoor temperature is valid and <=20 °C;
- daytime: normally off;
- below 16 °C: daytime winter reserve starts and remains active until 18 °C.

The reserve state is stored in a Boolean helper so the 16 -> 18 °C hysteresis survives ordinary sensor updates and Home Assistant restarts.

## 12. Separate switch-controlled auxiliary heater

The production design can additionally use a switch-controlled auxiliary heater with its own reference sensor.

Automatic operation is permitted only with real heating permission. The design can also check outdoor temperature and a dedicated heater-reference sensor.

Manual starts can be limited to four hours. The absolute deadline can be stored with:

```yaml
timestamp: "{{ as_timestamp(now()) + 14400 }}"
```

Using an absolute UNIX timestamp is intentional; it survives midnight and restarts because the deadline lives in an `input_datetime` helper.

A fixed time such as 23:00 can remain a hard one-time shutdown event.

## 13. Numeric template safety

All numeric state/attribute conversions in the current public local controller use explicit `float(...)` defaults.

Invalid sensor values do not create heating permission. Fail-safe behavior is preferred over guessing a temperature.

## 14. Away mode

The neutral example uses an away-mode Boolean as a hard permission condition for auxiliary heating.

A full production installation may additionally shut down Mitsubishi units, radiator thermostats, auxiliary heaters and release/reserve markers when away mode is activated.

Use `continue_on_error` where appropriate so one unavailable device does not prevent remaining safety actions.

## 15. Local/MELCloud coexistence

MELCloud Home may remain configured as a manual fallback.

A useful entity-ID migration pattern is:

```text
old MELCloud entity:
climate.room -> climate.room_melcloud

new local entity:
climate.generated_local_name -> climate.room
```

The automation can then keep using the stable primary entity ID while the underlying control path becomes local.

Do not run the old cloud-primary controller and the new local-primary controller simultaneously against the same unit.

## 16. IR remote

The original Mitsubishi infrared remote directly controls the indoor unit and should remain usable independently of Home Assistant and Internet access.

The return synchronization path after an IR change should be tested per installation. Do not assume that an IR target change should become a new desired-temperature helper value: the current local controller intentionally keeps the helper authoritative.

## 17. Failure behavior

### Internet/DSL outage

Local Home Assistant control continues as long as Home Assistant, LAN/WLAN and the Mitsubishi adapter are available.

### MELCloud API outage

Local Home Assistant control continues.

### Home Assistant/server outage

The local automation is unavailable. MELCloud Home and the original IR remote may still provide manual control if their respective paths are available.

## 18. Final checks before production

1. Validate YAML.
2. Confirm all placeholder entity IDs are replaced.
3. Confirm the desired helper changes the real desired temperature.
4. Confirm no device-target-to-helper back-sync automation remains.
5. Observe AUTO `heating`, `cooling` and `idle` states.
6. Confirm auxiliary radiators are blocked during AUTO cooling/idle.
7. Confirm AC-OFF winter reserve works only in its own scope.
8. Review logs for unavailable entities or template warnings.
9. Keep a rollback backup.

## Security

Do not publish local IP addresses, MAC addresses, passwords, MELCloud credentials, Home Assistant tokens, API keys, e-mail addresses or private hostnames.
