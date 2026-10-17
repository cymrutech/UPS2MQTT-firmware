# UPS2MQTT

**Local, cloud-free telemetry for the EcoFlow River 3 Plus.**

UPS2MQTT is a small ESP32-S3 USB dongle that reads live telemetry from an **EcoFlow
River 3 Plus** and publishes it to your own **MQTT** broker over WiFi — with **Home
Assistant auto-discovery**. Everything stays on your local network: **no cloud, no app,
no account, no subscription.**

It is **read-only** — it sends only telemetry poll/query requests and cannot change any
settings, limits, or firmware on your EcoFlow.

![platform](https://img.shields.io/badge/platform-ESP32--S3-blue)
![framework](https://img.shields.io/badge/framework-ESP--IDF%205.x-orange)
![home assistant](https://img.shields.io/badge/Home%20Assistant-auto--discovery-41BDF5)
![license](https://img.shields.io/badge/firmware-proprietary-lightgrey)

> Every other EcoFlow → Home Assistant option talks to EcoFlow's cloud. UPS2MQTT talks
> to the unit **directly over USB**, so your data never leaves your network.

---

## How it works

Plug the board into the River 3 Plus USB **Comm** port. It reads telemetry over CDC/ACM
serial and HID feature reports, and publishes a JSON payload to your MQTT broker. In Home
Assistant the device appears automatically under MQTT — no YAML.

---

## Features

- **Real-time telemetry** — load, draw, charge %, runtime, temperatures, voltages, flags
- **Dual USB interface** — CDC/ACM for the full telemetry stream + HID for charge %, runtime, status flags
- **MQTT publisher** — UPS telemetry to `<prefix>/<device-id>/ups/telemetry`, system health to `<prefix>/<device-id>/system/telemetry`
- **Home Assistant auto-discovery** — 16 sensors (including the device's own IP address and UPS data age) + connectivity and UPS-data-stale binary sensors
- **Stale-data detection** — every payload carries `data_age_s` and `data_stale`, so a UPS that stops answering is flagged rather than shown as healthy
- **NUT server** — read-only Network UPS Tools server on port 3493, advertised over mDNS
- **Read-only & local** — no cloud, no telemetry, no commands that change your EcoFlow
- **Zero-touch provisioning** — SoftAP captive portal; no app required
- **OTA firmware updates** — publish a firmware URL to `<prefix>/<device-id>/ups/ota`
- **Syslog forwarding** — optional RFC 3164 or CEF over UDP
- **RGB status LED** — setup / connecting / connected / error at a glance
- **Factory reset** — hold BOOT for 5 seconds
- **Safe mode** — after repeated panics, USB host is skipped so WiFi/MQTT/HTTP stay up for diagnostics
- **Web debug dashboard** — live auto-refreshing diagnostics at `/debug`

---

## Hardware

A custom PCB dongle based on the **ESP32-S3-MINI-1-N8** module.

| Component | Detail |
|-----------|--------|
| MCU | ESP32-S3-MINI-1-N8 (8 MB flash, built-in antenna) |
| Connector | USB Type-A socket → Type-A-to-Type-B cable → River 3 Plus Comm port |
| ESD protection | USBLC6 on USB D+/D− |
| Power | USB 5V (the USB-A port on the front of the River 3 Plus works well) |
| Status LED | WS2812B addressable RGB (GPIO21) |
| Factory reset | BOOT button (GPIO0), hold 5 s |

---

## Quick start

1. Connect a USB Type-A-to-Type-B cable between the UPS2MQTT board and the River 3 Plus **Comm** port.
2. Power the board from a 5V USB supply (the River 3 Plus USB-A port works well).
3. On your phone or laptop, join the WiFi network **`UPS2MQTT-XXXXXX`**.
4. The captive portal opens (or browse to `http://192.168.4.1`) — enter your **2.4 GHz WiFi** and **MQTT broker** details.
5. Tap **Save & Restart**. The LED turns solid green and the device starts publishing.

See the **[Quick Start](docs/quickstart.md)** and the full **[User Guide](docs/user-guide.md)**
for provisioning, configuration, Home Assistant, OTA, and troubleshooting.

---

## MQTT topics

| Topic | Direction | Description |
|-------|-----------|-------------|
| `<prefix>/<device-id>/ups/telemetry`    | Published  | UPS JSON telemetry, every `mqtt_interval_s` (default 60 s) |
| `<prefix>/<device-id>/system/telemetry` | Published  | Device health (RSSI, uptime, heap, IP), every `mqtt_interval_s` |
| `<prefix>/<device-id>/ups/ota`          | Subscribed | Publish a firmware URL to trigger OTA |

Default prefix: `ups2mqtt`. Device ID: `UPS2MQTT-XXXXXX`, derived from the ESP32-S3 STA MAC.

### Example UPS telemetry payload

```json
{
  "serial":          "R6XXXXXXXXXXXXXX",
  "total_load_w":    16.56,
  "ac_draw_w":       16.56,
  "ac_load_w":       16.56,
  "solar_dc_w":      0.00,
  "dc_load_w":       0.00,
  "usba_w":          0.00,
  "usbc_w":          0.00,
  "charge_pct":      79,
  "batt_time_min":   1805,
  "charge_time_min": -1,
  "temp_sys":        30,
  "temp_batt":       31,
  "flags":           12,
  "cap_mah":         12800,
  "line_freq_hz":    50.0,
  "batt_v":          25.60,
  "batt_nom_v":      25.60,
  "data_age_s":      3,
  "data_stale":      false
}
```

---

## HTTP API

| Method | Path | Description |
|--------|------|-------------|
| GET  | `/`             | Config form |
| POST | `/config`       | Save settings and reboot |
| GET  | `/status`       | JSON snapshot of all subsystems |
| GET  | `/debug`        | Live auto-refreshing diagnostics dashboard |
| GET  | `/ota`          | OTA status page |
| POST | `/ota`          | Upload firmware binary |
| POST | `/reset_panics` | Clear the panic counter to exit safe mode |
| GET  | `/about`        | Device info, NUT setup and changelog |

---

## Firmware downloads

Compiled firmware is published on this repository's **[Releases](../../releases)** page.
Each release has:

- `UPS2MQTT-production-vX.Y.Z.bin` — **OTA app image**, for updating a running dongle
  (upload at `http://<device-ip>/ota`, or see the [User Guide](docs/user-guide.md#ota-firmware-updates)).
- `UPS2MQTT-production-vX.Y.Z-factory.bin` — **factory image**, a full flash at offset `0x0`.
- `SHA256SUMS.txt` — checksums. Verify before flashing.

This repository contains documentation and compiled firmware only; the firmware source
is not published.

---

## Compatibility

Supported and tested: **EcoFlow River 3 Plus**. Other EcoFlow models are **not** supported.

---

## License

- **Firmware binaries** (release assets): **proprietary, All Rights Reserved** —
  licensed for use on UPS2MQTT hardware only. See [LICENSE](LICENSE).
- **Documentation** (`README.md`, `docs/`): **[CC BY 4.0](LICENSE-CC-BY-4.0)**.

UPS2MQTT and Cymru Tech are independent and are **not affiliated with, authorised by, or
endorsed by EcoFlow**. "EcoFlow" and "River 3 Plus" are used only to describe
compatibility. Provided "as is", without warranty; used at your own risk.

More at **[cymrutech.uk](https://cymrutech.uk)**.
