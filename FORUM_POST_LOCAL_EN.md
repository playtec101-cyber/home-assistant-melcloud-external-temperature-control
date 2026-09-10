# Local Mitsubishi control in Home Assistant with MELCloud Home kept as fallback

I moved my Mitsubishi Home Assistant control from a cloud-primary design to a local-primary architecture:

```text
PRIMARY:
Home Assistant -> local LAN/WLAN -> MAC-577IF2-E -> indoor unit

FALLBACK:
MELCloud Home -> Mitsubishi cloud -> indoor unit
```

The local integration is:

https://github.com/pymitsubishi/homeassistant-mitsubishi

The current neutral public controller is:

`local_primary_hvac_action_aux_heating_example.yaml`

Detailed design notes are in:

`LOCAL_CONTROLLER_DESIGN_NOTES.md`

## Why local-primary

Normal room-temperature control no longer depends on the Mitsubishi cloud. If the Internet or MELCloud API is unavailable, the local Home Assistant path can continue as long as Home Assistant, LAN/WLAN and the Mitsubishi adapter are available.

MELCloud Home may remain configured as a manual fallback, but I do not run two complete automation controllers against the same unit.

## Native AUTO stays native

Home Assistant does not simulate AUTO by switching between HEAT and COOL.

AUTO stays AUTO. Home Assistant only compensates the device target from the difference between the external room reference and the desired room temperature.

## `hvac_action` now gates auxiliary heating

The selected HVAC mode alone is not enough to decide whether auxiliary heat is safe while the Mitsubishi is in AUTO.

The local integration exposes the current operating action via Home Assistant `hvac_action`.

The auxiliary-heating permission is therefore:

```text
HEAT              -> allowed after normal conditions
AUTO + heating    -> allowed after normal conditions
AUTO + cooling    -> blocked
AUTO + idle       -> blocked
AUTO + unclear    -> blocked
COOL              -> blocked
OFF               -> blocked, except the separate low-temperature winter reserve
```

This prevents radiator/electric auxiliary heating from fighting the air conditioner while AUTO has internally chosen cooling.

I still recommend watching `hvac_action` over several real cycles on each installation before treating it as fully proven for that hardware/firmware combination.

## Desired temperature is now authoritative

The local controller deliberately removed the older device-target-to-helper synchronization.

Automatic target compensation and asynchronous device echoes can race each other. An intermediate device target can be misread as a new user request and create target bouncing.

The local design therefore uses:

```text
desired-temperature helper = user intent / source of truth
climate target              = compensated device target
```

Dashboards, scripts, services or voice-control paths should change the desired helper directly.

The last-automatic-target helper is still used to recognize the controller's own writes and avoid redundant target commands, but it no longer authorizes back-synchronization.

## Auxiliary radiator logic

The public example includes two optional auxiliary thermostat entities.

A release can happen only after 25 minutes of stable heating demand, with the room at least 0.5 °C below desired and with outdoor-temperature checks satisfied.

The example auxiliary target is `desired - 1.0 °C`.

The 25-minute `for:` intentionally restarts after a Home Assistant restart. In this use case that is fail-safe because it can only delay auxiliary heat.

## AC-OFF winter reserve

When the main AC is OFF, a separate and mutually exclusive winter-reserve automation handles the auxiliary thermostats:

- 23:00–08:00: example 18 °C night reserve;
- daytime: normally off;
- below 16 °C: reserve starts and stays active until 18 °C.

Separating the AC-OFF and AC-active logic avoids duplicate commands from overlapping automations.

## Separate auxiliary heater

The full production design can additionally use an optional switch-controlled heater with its own reference sensor.

It can be restricted to HEAT or AUTO+heating. Manual starts can be limited to four hours using an absolute `input_datetime` deadline, and a fixed time such as 23:00 can remain a hard shutdown.

## Template hardening

All numeric conversions in the current local public controller use explicit `float(...)` defaults. Invalid/unavailable values cannot accidentally create heating permission.

## Horizontal vane spelling

The local integration uses:

```text
center
```

where MELCloud Home may use:

```text
centre
```

The repository is:

https://github.com/playtec101/home-assistant-melcloud-external-temperature-control

Feedback from other MAC-577IF2-E users is welcome, especially around `hvac_action` behavior in AUTO, polling delay, vane behavior and auxiliary-heating interaction.
