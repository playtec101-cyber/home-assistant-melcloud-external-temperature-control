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

## Important update: how `AUTO + idle` is handled

The local integration exposes `hvac_action`.

A key detail from the integration implementation is that `idle` is returned whenever the compressor is currently not operating. Therefore `AUTO + idle` does not prove that the external room is already at its desired temperature and it does not say whether the previous active AUTO phase was heating or cooling.

The current auxiliary-heating permission is therefore:

```text
HEAT              -> allowed after normal external-demand checks
AUTO + heating    -> allowed after normal external-demand checks
AUTO + idle       -> neutral; allowed only if the external-demand checks still require heat
AUTO + cooling    -> hard blocked
AUTO + unclear    -> hard blocked
COOL/DRY/FAN      -> hard blocked
OFF               -> blocked, except the separate low-temperature winter reserve
```

`idle` is **not** treated as heating. It simply no longer blocks auxiliary heating by itself. The room sensor, desired temperature, outdoor temperature, timing and hysteresis still decide whether auxiliary heat actually starts.

This fixes an important winter edge case: if the Mitsubishi pauses its compressor because its internal sensor is satisfied while a colder external room sensor still shows demand, the 25-minute auxiliary-heating demand period is no longer reset merely by the transition to `idle`.

If AUTO changes to `cooling`, auxiliary heat is still shut down immediately so the systems cannot fight each other.

## Latest AUTO observation

In one field test, the external room temperature was clearly above the desired value while `hvac_action` stayed `idle` for a while. That initially looked like a stuck AUTO decision.

Without Home Assistant forcing COOL, the Mitsubishi later transitioned by itself from `idle` to `cooling` while remaining in native AUTO.

Because of that observation, I have **not** steepened the existing AUTO target-correction curve and have not added an AUTO->COOL/HEAT override. One idle interval is not enough evidence that the curve is wrong; repeated real-world behavior should be observed first.

A simple temporary dashboard diagnostic is:

```jinja2
HVAC Action: **{{ state_attr('climate.main_room_ac', 'hvac_action') }}**
```

## Desired temperature is authoritative

The local controller deliberately removed the older device-target-to-helper synchronization.

Automatic target compensation and asynchronous device echoes can race each other. An intermediate device target can be misread as a new user request and create target bouncing.

The local design therefore uses:

```text
desired-temperature helper = user intent / source of truth
climate target              = compensated device target
```

Dashboards, scripts, services or voice-control paths should change the desired helper directly.

## Auxiliary radiator logic

The public example includes two optional auxiliary thermostat entities.

A release can happen only after 25 minutes of stable external heating demand, with the room at least 0.5 °C below desired and with outdoor-temperature checks satisfied.

The example auxiliary target is `desired - 1.0 °C`.

The 25-minute `for:` intentionally restarts after a Home Assistant restart. In this use case that is fail-safe because it can only delay auxiliary heat.

## AC-OFF winter reserve

When the main AC is OFF, a separate and mutually exclusive winter-reserve automation handles the auxiliary thermostats:

- 23:00–08:00: example 18 °C night reserve with valid outdoor temperature <=20 °C;
- daytime: normally off;
- below 16 °C: reserve starts only with valid outdoor temperature <=20 °C and stays active until 18 °C.

The reserve also ends if the outside value becomes invalid/too warm, the main AC is switched on or away mode is enabled.

Separating the AC-OFF and AC-active logic avoids duplicate commands from overlapping automations.

## Separate auxiliary heater

The full production design can additionally use an optional switch-controlled heater with its own reference sensor.

It follows the same AUTO interpretation: HEAT and AUTO+heating/idle may permit it after its own demand checks; AUTO+cooling/unclear, COOL and OFF block it.

Manual starts can be limited to four hours using an absolute `input_datetime` deadline, and a fixed time such as 23:00 can remain a hard shutdown.

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

Feedback from other MAC-577IF2-E users is welcome, especially around AUTO idle duration, `hvac_action` transitions, polling delay, vane behavior and auxiliary-heating interaction.
