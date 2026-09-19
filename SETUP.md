# Setup Guide

This is the exact setup the display was built and tested against. Follow it top
to bottom and you end up with the same system: a Mosquitto broker, Home
Assistant publishing every value the screen shows as retained MQTT messages,
and a mean-group-based entity setup that keeps working when individual room
sensors drop out.

The [README](README.md) covers what the device does and why; this guide is the
step-by-step. Everything here lives on the Home Assistant side — the device
firmware is flashed last, in step 6.

## 1. MQTT broker

Any MQTT 3.1.1/5 broker with retained-message support works. The tested setup
is Mosquitto in Docker, with authentication (no anonymous access) and
per-user ACLs:

```yaml
# docker-compose.yml
services:
  mqtt:
    image: eclipse-mosquitto:latest
    restart: unless-stopped
    ports:
      - "1883:1883"
    volumes:
      - ./mosquitto/config:/mosquitto/config
      - ./mosquitto/data:/mosquitto/data
```

`mosquitto/config/mosquitto.conf`:

```conf
allow_anonymous false
listener 1883
persistence true
persistence_location /mosquitto/data/
password_file /mosquitto/config/pwfile
acl_file /mosquitto/config/acl.conf
```

Create two users (then `docker compose restart`):

```bash
mosquitto_passwd -c mosquitto/config/pwfile homeassistant
mosquitto_passwd  mosquitto/config/pwfile reterminal-e1001
```

`mosquitto/config/acl.conf` — Home Assistant may publish anywhere (it also
hosts the MQTT integration and possibly other devices); the display user gets
only what it needs:

```conf
user homeassistant
topic readwrite #

user reterminal-e1001
topic read home/reterminal/#
topic write home/reterminal/status/#
# availability + MQTT discovery for the display's own sensors
topic write reterminal-e1001/status
topic write homeassistant/sensor/reterminal-e1001/#
```

The last two lines use the ESPHome node name (`name:` substitution in
`reterminal-e1001.yaml`, default `reterminal-e1001`). If you rename the node,
change these lines to match.

## 2. Home Assistant MQTT integration

In Home Assistant: **Settings → Devices & Services → Add Integration → MQTT**.
Point it at the broker host/port and the `homeassistant` credentials from
step 1.

## 3. Weather integration

Any weather integration that supports the `weather.get_forecasts` action with
both `daily` and `hourly` forecast types works. The tested one is
**OpenWeatherMap** (free tier is enough): install the integration, enter an
API key, pick your location. Note the resulting entity ID — by default
`weather.openweathermap` — you need it in step 5.

## 4. Source entities

The automations read from four entities. If you only have single sensors
instead of groups, skip the groups and put your entity IDs directly into the
automation (step 5).

| Package entity | Meaning | Tested source |
|---|---|---|
| `sensor.outdoor_temperature` | mean of all outdoor temp sensors, provider included so the display matches its own forecast | sensor group over OpenWeatherMap temp + heat pump outdoor sensor + wall sensors |
| `sensor.outdoor_humidity` | mean of all outdoor humidity sensors | sensor group, same sources |
| `sensor.house_upper_floor` | mean indoor temperature, upper floor | sensor group over room sensors |
| `sensor.house_lower_floor` | mean indoor temperature, lower floor | sensor group over room sensors |

Create the groups once via YAML (or as UI helpers with the same settings —
`type: mean`, `ignore_non_numeric` on). The group definitions are already part
of the package in step 5; the only thing you edit there is the member list.

Two details worth copying from the tested setup:

- **The weather provider is part of the outdoor averages.** The temperature
  shown next to the forecast then comes from the same source as the forecast,
  which makes the display internally consistent.
- **The display's own room sensor joins the lower-floor group.** After the
  first sync the device publishes `sensor.reterminal_room_temperature` via
  MQTT discovery; adding it to the group means the room the display hangs in
  contributes to its own indoor reading.

Only one indoor reading available? Publish the same sensor to both
`indoor_upper` and `indoor_lower` in the automation — the display shows their
average, which then equals that one sensor.

## 5. Home Assistant package

Enable packages in `configuration.yaml`:

```yaml
homeassistant:
  packages: !include_dir_named packages
```

Copy the package file in and edit it:

```bash
cp homeassistant/reterminal.yaml /config/packages/reterminal.yaml
```

Replace every line marked `# CHANGE`:

- the four `sensor:` group member lists (step 4) — if you don't want a group,
  delete that group block and put a single entity ID in the matching
  automation condition/payload
- `weather.openweathermap` → your weather entity (3×)
- the `home/reterminal` topic prefix — must match the `topic` substitution in
  `reterminal-e1001.yaml` (default matches, so usually no edit)

Restart Home Assistant. Then check:

- **Developer Tools → States**: `sensor.reterminal_forecast` exists, its
  `daily`/`hourly` attributes contain JSON, and `sensor.outdoor_temperature`,
  `sensor.outdoor_humidity`, `sensor.house_upper_floor`,
  `sensor.house_lower_floor` have numeric states.
- **The automations run**: within a minute, "reTerminal E1001 - Publish State
  to MQTT" and the prevent-sleep automation should have `last_triggered` set.

## 6. Verify end to end

Before flashing, watch the topics from any machine with MQTT access:

```bash
mosquitto_sub -h <broker-host> -u homeassistant -P '<password>' \
  -t 'home/reterminal/#' -v
```

Within a minute you should see every topic in the automation with `retain`
set. Retained messages persist on the broker — that's the whole mechanism:
the device reads them the moment it subscribes, on a wake that lasts a few
seconds.

## 7. Flash the device

Follow the README Quick Start: `cp secrets.yaml.example secrets.yaml`, fill in
Wi-Fi, MQTT (the `reterminal-e1001` credentials from step 1), OTA password and
static IP, then `esphome run reterminal-e1001.yaml` over USB. Later updates go
over the air (see README).

The date label (`Thursday, September 18`) is produced by the automation's
`strftime('%A, %B %-d')` — it renders in the locale of the machine running
Home Assistant, which is what the tested setup uses.

## 8. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Display shows stale values forever | topic prefix mismatch between ESPHome `topic` substitution and the package | make both `home/reterminal` |
| `--:--` clock, everything else fine | RTC never seeded, or SNTP unreachable | first sync wake needs Wi-Fi + broker reachable; check `sensor.reterminal_*` discovery entities appeared |
| Forecast area blank | `sensor.reterminal_forecast` unavailable | weather entity wrong or provider doesn't support `weather.get_forecasts` |
| Outdoor fields blank | outdoor group has no numeric members | fix group members; check the automation's `last_triggered` |
| Battery % jumps oddly | calibration table in the YAML is for the tested cell | edit the `calibrate_linear` rows for your pack |
| Date in wrong language | `strftime` uses HA host locale | change the `%A`/`%B` payload or pass a locale to HA |
