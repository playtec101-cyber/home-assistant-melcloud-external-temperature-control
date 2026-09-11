# Archived local-control migration guide

> **Archived on 2026-09-11.**
>
> This guide described the older local-primary controller that compensated the Mitsubishi target temperature from an external room sensor.
>
> If your local `pymitsubishi/homeassistant-mitsubishi` installation supports **Remote Temperature**, use the new guide instead:

**[`REMOTE_TEMPERATURE_RECOMMENDED.md`](REMOTE_TEMPERATURE_RECOMMENDED.md)**

Recommended setup now:

1. Enable **Experimental Features**.
2. Select the **External Temperature Sensor**.
3. Set **Temperature Source** to `Remote`.
4. Use the real desired temperature directly as the Mitsubishi target.
5. Disable the old main-unit target-offset / step-compensation automation.

MELCloud Home can still remain configured as a manual/fallback path.

The old migration/offset method remains relevant only when Remote Temperature cannot be used reliably.
