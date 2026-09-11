# Mitsubishi + Home Assistant: external room temperature

> **Update 2026-09-11:** For the local `pymitsubishi/homeassistant-mitsubishi` integration, **Remote Temperature is now the recommended path**. If it works on your adapter/unit, the older target-offset / step-compensation logic is no longer needed for the main Mitsubishi room-temperature control.

## Recommended current setup

Use the local integration and configure:

1. **Experimental Features** = enabled
2. **External Temperature Sensor** = your real room sensor or HA room-average sensor
3. **Temperature Source** = `Remote`
4. Mitsubishi target temperature = your real desired room temperature
5. Disable the old main-unit offset / step-compensation automation

Detailed current guide:

**[`REMOTE_TEMPERATURE_RECOMMENDED.md`](REMOTE_TEMPERATURE_RECOMMENDED.md)**

Conceptually:

```text
external room sensor -> Mitsubishi Remote Temperature
user desired temp     -> Mitsubishi target temperature
```

No artificial +0.5 / +1.0 / +1.5 °C target correction is required when Remote Temperature works correctly.

---

## Other solution paths

### 1. Legacy / classic MELCloud

Older external-sensor and simulated/compensated control remains available as a fallback for installations that cannot use the local Remote Temperature path.

### 2. MELCloud Home as primary control

Cloud-primary users can still use the target-offset automation:

`melcloud_home_external_temperature_control_public.yaml`

### 3. Local Remote Temperature — recommended

Current preferred architecture:

```text
Home Assistant -> local LAN/WLAN -> MAC-577IF2-E -> Mitsubishi indoor unit
```

MELCloud Home may remain configured only as a manual/fallback path.

---

## What remains useful from the older work?

Remote Temperature replaces only the **main AC room-temperature compensation**. The following are still useful:

- local MAC-577 control
- `hvac_action`
- auxiliary-heating logic
- winter reserve
- away/vacation safety
- external-sensor averaging
- vane control
- restart/fallback logic

Old local offset/step documents are kept only as **archive/legacy references**. If you followed an older link, use the current guide above instead.

---

## Important

Do not run two complete main controllers against the same physical Mitsubishi unit at the same time.

Back up Home Assistant before changing integrations, entity IDs or automations.
