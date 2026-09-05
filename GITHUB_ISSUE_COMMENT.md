# Suggested comment for home-assistant/core issue #143121

I tested a workaround for the legacy MELCloud polling delay that may be useful to others.

Calling `homeassistant.update_entity` on the MELCloud climate entity triggers an additional refresh in my setup. I created a time-pattern automation with `minutes: "/5"`.

Test result:
- previous HA value: 21.5 °C at 19:05:09
- setpoint changed in MELCloud to 22.5 °C
- HA received 22.5 °C at 19:10:02, at the next 5-minute boundary
- manually executing the same refresh automation also pulled the new value immediately

Example:

```yaml
triggers:
  - trigger: time_pattern
    minutes: "/5"

actions:
  - action: homeassistant.update_entity
    target:
      entity_id: climate.living_room
```

I left the integration's normal polling enabled and used this only as an additional refresh.

I documented the workaround together with a larger external-temperature-control setup here:

https://github.com/playtec101-cyber/home-assistant-melcloud-external-temperature-control

The repository is intentionally free of credentials/IP addresses and uses generic entity IDs.
