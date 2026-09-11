# Archived local offset/step design notes

> **Archived on 2026-09-11.**
>
> This document previously described the local target-offset / fixed-step compensation controller.
>
> For the local `pymitsubishi/homeassistant-mitsubishi` integration, **Remote Temperature is now the recommended main room-control path** when it works with the adapter/unit.

Use the current guide instead:

**[`REMOTE_TEMPERATURE_RECOMMENDED.md`](REMOTE_TEMPERATURE_RECOMMENDED.md)**

Current principle:

```text
external room sensor -> Mitsubishi Remote Temperature
user desired temp     -> Mitsubishi target temperature
```

The old local offset/step controller is retained only as historical/legacy fallback material for installations where Remote Temperature cannot be used reliably.

Auxiliary-heating ideas such as `hvac_action`, winter reserve, restart-safe timers and safety interlocks remain useful, but the old main-unit temperature-compensation algorithm should not be copied into a working Remote Temperature setup.
