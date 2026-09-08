# Local Mitsubishi control in Home Assistant with MELCloud Home kept as fallback

I originally used MELCloud Home as the primary Home Assistant control path for my Mitsubishi Electric air conditioners and used external room-temperature sensors to correct the target temperature.

After seeing occasional MELCloud Home API communication errors, I tested a different architecture:

```text
PRIMARY:
Home Assistant -> local LAN/WLAN -> MAC-577IF2-E -> indoor unit

FALLBACK:
MELCloud Home -> Mitsubishi cloud -> indoor unit
```

The local Home Assistant integration I used is:

https://github.com/pymitsubishi/homeassistant-mitsubishi

It exposes the Mitsubishi unit locally as a normal Home Assistant climate entity.

## Why I changed it

With MELCloud Home as the primary control path, Home Assistant still depends on the Mitsubishi cloud. If the API has a problem, the automations cannot reliably control the unit even though Home Assistant itself is running.

With local-primary control, the normal room-temperature controller continues to work if:

- the Internet/DSL connection is down,
- MELCloud Home is unavailable,
- or the Mitsubishi cloud API has a temporary problem.

MELCloud Home stays configured and can still be used as a fallback if Home Assistant/server is unavailable but Internet/MELCloud still works.

## Both directions tested

I tested a target-temperature change from local Home Assistant. The indoor unit reacted immediately and MELCloud Home later showed the same target.

I then changed the target in MELCloud Home. The unit reacted and the local Home Assistant entity picked up the new value again.

That is important because the existing manual-target synchronization logic still works: a manual MELCloud change can be copied into the desired-temperature helper without creating a feedback loop.

## IR remote: direct control expected, return synchronization not yet tested

The original Mitsubishi IR remote should continue to control the indoor unit directly because that path does not depend on Home Assistant or Internet access.

What I have **not yet tested** is whether and how quickly a target change made with the IR remote is then reflected back into:

- the local Home Assistant climate entity,
- the desired-temperature helper,
- and MELCloud Home.

For the local architecture it is technically plausible that the indoor unit updates the MAC-577IF2-E state and that Home Assistant reads the changed value locally, but I am treating that as expected/possible rather than verified until I test it on the real installation.

A simple test is to change the target by 1 °C with the IR remote and watch the Home Assistant `temperature` attribute, the desired-temperature helper and MELCloud Home.

## No second competing controller

I did not run two complete automation controllers against the same unit.

The main controller now points to the **local** Mitsubishi climate entities. MELCloud Home remains configured only as a manual/fallback path.

The feedback-loop protection continues to use a helper that stores the last target automatically sent by Home Assistant. A new device target is treated as a genuine user request only when it differs from both:

1. the last automatic target, and
2. the current desired-temperature helper.

## Small integration difference: horizontal vane center

The optional horizontal-vane automation needed one integration-specific spelling change:

```text
MELCloud Home: centre
local integration: center
```

`Swing` was available in both tested paths.

## Migration trick

To avoid rewriting every automation, I renamed the old MELCloud entity, for example:

```text
climate.room -> climate.room_melcloud
```

and then renamed the new local entity to the old primary entity ID:

```text
climate.generated_local_name -> climate.room
```

The existing YAML can then keep referring to `climate.room`, but the underlying control path is local.

## Failure behavior

- Internet/DSL down: local Home Assistant control still works.
- MELCloud API down: local Home Assistant control still works.
- Home Assistant/server down, Internet OK: MELCloud Home can be used as fallback.
- Both HA and Internet down: the original IR remote should still directly control the indoor unit; return synchronization has not yet been tested.

The public examples, all three solution paths and the local-primary migration guide are in this repository:

https://github.com/playtec101-cyber/home-assistant-melcloud-external-temperature-control

The local-primary guide is:

`LOCAL_CONTROL_WITH_MELCLOUD_FALLBACK.md`

Feedback from other MAC-577IF2-E users is welcome, especially around model compatibility, polling delay, vane behavior and IR synchronization.
