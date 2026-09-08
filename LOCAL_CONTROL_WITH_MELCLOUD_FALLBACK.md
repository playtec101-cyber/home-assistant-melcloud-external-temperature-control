# Local Mitsubishi control with MELCloud Home as fallback

This guide describes the tested **local-primary** architecture while reusing the same external-temperature controller logic from `melcloud_home_external_temperature_control_public.yaml`.

## Goal

Normal control should not depend on the Mitsubishi cloud:

```text
Home Assistant -> local LAN/WLAN -> Mitsubishi Wi-Fi adapter -> indoor unit
```

MELCloud Home stays configured as a separate fallback/manual path:

```text
MELCloud Home -> Mitsubishi cloud -> indoor unit
```

This gives two control paths for common failure cases:

| Situation | Local Home Assistant | MELCloud Home |
| --- | --- | --- |
| Internet/DSL outage | Works locally | Unavailable |
| MELCloud API outage | Works locally | Unavailable/limited |
| Home Assistant/server outage, Internet OK | Unavailable | Works as fallback |
| Local network / adapter failure | Local path unavailable | May also be affected because both paths ultimately need the unit/adapter connectivity |

## Tested local integration

The tested custom integration is:

[pymitsubishi/homeassistant-mitsubishi](https://github.com/pymitsubishi/homeassistant-mitsubishi)

Home Assistant name: **Mitsubishi Air Conditioner**.

The upstream project documents support for Mitsubishi units using the MAC-577IF-2E / MAC-577IF2-E Wi-Fi adapter, climate control, temperature/mode/fan/vane entities and approximately 30-second polling.

## Why the controller YAML is shared

The tested local climate entity exposes the same relevant HVAC states used by the current MELCloud Home controller:

```text
auto
heat
cool
off
```

It also uses the same standard Home Assistant climate services for target temperature and HVAC mode.

Therefore the complete control logic in:

`melcloud_home_external_temperature_control_public.yaml`

can be used locally by pointing its climate placeholders at the local Mitsubishi entities.

The only tested integration-specific code difference in the current optional horizontal-vane automation is:

```text
MELCloud Home center: centre
local integration center: center
```

Keeping one shared controller file avoids two almost identical large YAML files drifting apart.

## 1. Back up first

Create a full Home Assistant backup before installing or renaming anything.

If migrating from a working MELCloud Home setup, keep that backup as your clean cloud-primary rollback point.

## 2. Give the Mitsubishi adapters stable local IP addresses

Use DHCP reservations in your router so each Mitsubishi Wi-Fi adapter keeps the same local IP address.

Do not publish those private addresses in a public repository.

## 3. Install HACS and the local integration

Install HACS if needed, then install **Mitsubishi Air Conditioner** from HACS.

Repository:

```text
https://github.com/pymitsubishi/homeassistant-mitsubishi
```

Restart Home Assistant after installation.

## 4. Add each local Mitsubishi device

In Home Assistant:

```text
Settings -> Devices & services -> Add integration -> Mitsubishi Air Conditioner
```

Add each air conditioner by its local IP address.

Keep MELCloud Home configured. Do not remove it.

## 5. Test local read/write before changing automations

Before migrating your production YAML, verify at least:

- Power on/off
- target temperature
- AUTO
- HEAT
- COOL
- fan mode
- horizontal vane values, if you use them

In the tested local integration the horizontal values included:

```text
auto
far left
left
center
right
far right
left and center
center and right
left, center and right
left and right
swing
```

## 6. Verify both synchronization directions

### Local -> unit -> MELCloud

1. Change the target temperature through the local Home Assistant entity.
2. Confirm the indoor unit reacts.
3. Confirm MELCloud Home later shows the same target.

### MELCloud -> unit -> local

1. Change the target in MELCloud Home.
2. Confirm the indoor unit reacts.
3. Wait for the local integration polling interval.
4. Confirm the local Home Assistant climate entity shows the new target.

That second direction allows the manual-target synchronization automation to keep working even though the primary controller is local.

### IR remote -> unit -> local + MELCloud

This path was also verified in the tested installation.

Signal flow:

```text
IR remote -> indoor-unit control board -> internal interface/CN105 -> MAC-577IF2-E
```

The adapter receives the changed state from the indoor unit. The local Home Assistant integration then reads the new value from the adapter, while the adapter also reports the updated state to MELCloud.

That means an IR-made target change can be reflected in:

- the local Home Assistant climate entity
- the desired-temperature helper via the manual-target sync automation
- MELCloud Home

In the tested setup the local Home Assistant value followed the IR change within seconds.

## 7. Entity-ID migration option

If an existing controller already uses stable entity IDs, you can preserve the YAML by swapping which integration owns those IDs.

Example:

```text
Existing MELCloud primary:
climate.room

Rename old MELCloud entity to:
climate.room_melcloud

Rename new local entity to:
climate.room
```

The automations continue referring to `climate.room`, but it is now local.

Alternatively keep explicit local IDs and replace the public placeholders directly.

## 8. Reuse the complete controller

Start from:

`melcloud_home_external_temperature_control_public.yaml`

Replace the main climate placeholders with your **local** climate entities.

For the optional horizontal-vane automation, change the tested center value from:

```yaml
swing_horizontal_mode: centre
```

to:

```yaml
swing_horizontal_mode: center
```

and make the same `centre` -> `center` change in the comparison template immediately above it.

Do **not** keep a second complete controller active against the same devices. MELCloud Home itself may remain configured; only the competing automation controller should be disabled/replaced.

## 9. Why there is no feedback loop

The controller keeps a Number helper containing the **last automatic target** it sent.

When the climate entity's target changes, the manual-sync automation compares the new target with:

1. the last automatically written target, and
2. the desired-temperature helper.

Only a genuinely different value is copied into the desired-temperature helper.

Therefore a compensated target written by the controller does not become a new user request.

## 10. Failure behavior

### Internet/DSL outage

Local Home Assistant control continues as long as Home Assistant, the local network and the Mitsubishi adapter are available.

Cloud voice assistants and MELCloud Home require Internet access and will not be available during the outage.

The original IR remote continues to control the indoor unit directly.

### Home Assistant/server outage

The local automation is unavailable, but MELCloud Home can still control the unit if Internet/Mitsubishi cloud service is working.

The original IR remote also remains available as a direct local control path.

### MELCloud outage

The local Home Assistant controller continues without depending on MELCloud.

## 11. Infrared remote

The original IR remote controls the indoor unit directly and does not depend on Home Assistant, the Home Assistant server or Internet access.

In the tested installation, changes made with the IR remote were propagated by the indoor unit to the MAC-577IF2-E adapter. The local Mitsubishi integration then picked up the changed target, and the existing manual-target synchronization automation updated the Home Assistant desired-temperature helper. MELCloud Home also reflected the changed state.

Practical test procedure:

1. Change the setpoint by 1 °C with the original IR remote.
2. Watch the local Home Assistant climate entity's `temperature` attribute.
3. Watch the desired-temperature Number helper.
4. Confirm MELCloud Home also reaches the same target.

## 12. Security

Do not publish:

- local IP addresses
- MAC addresses
- passwords
- MELCloud credentials
- Home Assistant tokens
- API keys
- private hostnames

The public controller YAML in this repository uses neutral placeholders only.
