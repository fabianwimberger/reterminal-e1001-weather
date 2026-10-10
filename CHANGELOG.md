# Changelog

All notable changes to this project are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [v1.0.1] - 2026-10-02

Adds full firmware compilation coverage for the reTerminal E1001 weather display configuration.

### Testing

- Compile the ESP32-S3 firmware, including the patched e-paper driver, on pull requests and pushes to main and develop.
- Retain configuration validation as a separate check.
- Validate and compile locally with ESPHome 2026.9.1 using the example configuration.

### Documentation & Links

- [README](https://github.com/fabianwimberger/reterminal-e1001-weather#readme)
- [Full changelog](https://github.com/fabianwimberger/reterminal-e1001-weather/compare/v1.0.0...v1.0.1)

## [v1.0.0] - 2026-09-19

Initial public release of the battery-powered reTerminal E1001 weather display: an ESPHome configuration that keeps a minute clock ticking while the ESP32-S3 spends almost all of its time in deep sleep, fed by Home Assistant over retained MQTT.

### Features

- Minute clock without Wi-Fi: the on-board PCF8563 RTC seeds the system clock on every wake, SNTP corrects it every 6 hours.
- Network only every 10 minutes: sync wakes read retained MQTT payloads; the rest draw from values persisted in flash.
- Tiered e-paper refresh: partial refresh on every wake, fast full refresh hourly, ghost-clearing full refresh every 6 hours.
- OTA-friendly deep sleep: uploads block sleep via a Home Assistant toggle.
- Diagnostics: room temperature/humidity (SHT40), battery, Wi-Fi signal and wake duration published to MQTT.
- Home Assistant package with sensor-group source entities, plus a step-by-step setup guide mirroring the tested broker/ACL configuration.

### Testing

- `esphome config` validated locally against ESPHome 2026.9.0 and enforced by CI on every push.
- Running on hardware: partial refresh across deep-sleep wakes, hourly fast full and 6-hourly ghost-clearing full refresh verified over multi-day operation.

### Documentation & Links

- [README](https://github.com/fabianwimberger/reterminal-e1001-weather#readme)
- [Setup guide](https://github.com/fabianwimberger/reterminal-e1001-weather/blob/main/docs/SETUP.md)
