# Security / privacy review

The public example files in this package were intentionally generalized.

## Removed / not included

- IP addresses
- hostnames
- passwords
- MELCloud account credentials
- Home Assistant long-lived access tokens
- API keys
- webhook IDs
- e-mail addresses
- MAC addresses
- serial numbers
- personal names
- house-specific heater / holiday / FRITZ!DECT logic

## Generic entity IDs used

Examples:

- `climate.living_room`
- `climate.bedroom`
- `sensor.living_room_temperature_average`
- `sensor.bedroom_temperature`
- `input_number.desired_temperature_living_room`
- `input_number.last_automatic_target_living_room`

These names do not expose a reachable network endpoint.

## Before publishing future edits

Search for:

- `http://` and `https://`
- `192.168.` / `10.` / `172.16-31.`
- `password`
- `token`
- `api_key`
- `secret`
- `Authorization`
- `Bearer`
- `webhook`
- e-mail addresses
- MAC addresses

Do not rely on masking only part of a password or token. Remove the entire secret.
