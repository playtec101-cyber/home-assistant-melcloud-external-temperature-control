# Suggested Home Assistant Community / Reddit post

## Title

**Mitsubishi MELCloud: external room sensors, better AUTO mode and 5-minute refresh – working Home Assistant YAML**

## Post

I wanted to share a setup that may save other Mitsubishi/MELCloud users some trial and error.

I use Mitsubishi Electric air conditioners through Home Assistant's **legacy MELCloud integration**. In my installation, fixed HEAT and COOL modes were generally usable, but native AUTO was much less satisfactory because the indoor-unit temperature is not necessarily the same as the actual comfort temperature in the room.

I ended up building two tested approaches:

- **Bedroom:** one external temperature sensor
- **Living room:** several external temperature sensors combined as a mean

Home Assistant keeps a separate desired-temperature helper and compensates the Mitsubishi target from the external room temperature.

For the living room, native `heat_cool` is dynamically corrected with:

```text
Mitsubishi target =
Mitsubishi internal temperature - (external average - desired temperature)
```

When AUTO is first enabled from OFF, Home Assistant starts in HEAT or COOL for 15 minutes if the external average is clearly above/below the target, then returns to native AUTO.

For the bedroom I use a different strategy: Home Assistant switches between HEAT and COOL only after the external sensor has remained outside a +/-1 °C deadband for 15 minutes.

I also added feedback-loop protection so a target written automatically by Home Assistant is not mistaken for a manual MELCloud/App change.

### Faster legacy-MELCloud refresh

The other useful discovery was that `homeassistant.update_entity` can force an additional MELCloud refresh.

I run it every five minutes:

```yaml
triggers:
  - trigger: time_pattern
    minutes: "/5"

actions:
  - action: homeassistant.update_entity
    target:
      entity_id: climate.living_room
```

In my test, a MELCloud setpoint change to 22.5 °C appeared in Home Assistant at the next five-minute boundary (19:10:02). Manually executing the refresh automation also pulled the new value immediately.

I put the complete reusable examples, setup instructions and privacy-cleaned YAML here:

https://github.com/playtec101-cyber/home-assistant-melcloud-external-temperature-control

The repository includes:

- single-sensor bedroom YAML
- multi-sensor living-room YAML
- manual setpoint synchronization / loop protection
- improved AUTO-mode logic
- five-minute forced polling
- helper examples
- optional horizontal swing automation

This is for the **legacy MELCloud integration**. The newer MELCloud Home integration has different polling behavior.

Feedback from other Mitsubishi models/setups is welcome.
