# UPS2MQTT User Guide

Everything you need to set up, configure, and operate UPS2MQTT.

---

## Contents

1. [What You Need](#what-you-need)
2. [First Boot](#first-boot)
3. [Provisioning via Captive Portal](#provisioning-via-captive-portal)
4. [Configuration Reference](#configuration-reference)
5. [MQTT Integration](#mqtt-integration)
6. [Home Assistant Setup](#home-assistant-setup)
7. [OTA Firmware Updates](#ota-firmware-updates)
8. [Factory Reset](#factory-reset)
9. [Status API](#status-api)
10. [Debug Dashboard](#debug-dashboard)
11. [Diagnostics & Safe Mode](#diagnostics--safe-mode)
12. [Troubleshooting](#troubleshooting)

---

## What You Need

| Item | Notes |
|------|-------|
| **UPS2MQTT board** | ESP32-S3-MINI-1-N8 custom PCB dongle |
| **USB Type-A to Type-B cable** | Standard printer-style cable — Type-A end into the UPS2MQTT board, Type-B end into the River 3 Plus Comm port |
| **USB 5V power supply** | Powers the UPS2MQTT board |
| **EcoFlow River 3 Plus** | Firmware tested against this model specifically |

Connect the USB Type-A to Type-B cable between the UPS2MQTT board's USB-A socket and the EcoFlow River 3 Plus **Comm** port. Power the board separately from a USB 5V supply — a convenient option is one of the USB-A ports on the front of the River 3 Plus itself.

---

## First Boot

On first boot — or after a factory reset — the device has no saved configuration. It will:

1. Attempt to connect to a WiFi network (fails because no SSID is saved)
2. Start a SoftAP named **`UPS2MQTT-XXXXXX`** where `XXXXXX` is derived from the board's WiFi MAC address (this is the **device ID** used throughout)
3. The status LED will pulse to indicate provisioning mode

Connect to the SoftAP and configure the device as described in the next section.

---

## Provisioning via Captive Portal

**On your phone or laptop:**

1. Open WiFi settings and connect to **`UPS2MQTT-XXXXXX`**
2. A "Sign in to network" or captive portal notification should appear — tap it.
   If it does not appear automatically, open a browser and navigate to `http://192.168.4.1`
3. The UPS2MQTT configuration form will load

**Fill in the form:**

- **WiFi SSID and Password** — your home network credentials
- **MQTT Broker Host** — IP address or hostname of your MQTT broker (e.g. `192.168.1.10`)
- **MQTT Port** — typically `1883` (plain TCP); TLS not yet supported
- **Username / Password** — if your broker requires authentication
- **Topic Prefix** — the MQTT topic root; default is `ups2mqtt`
- **Poll Interval** — how often to publish, in milliseconds; default is `5000` (5 seconds)
- **Syslog Host** — optional; leave blank to disable syslog forwarding
- **NTP Time Sync** — enabled by default; disable on air-gapped networks that block NTP (port 123/UDP)

Tap **Save & Restart**. The device saves the configuration to flash and reboots.
After about 5 seconds it will connect to your WiFi network.

**Verifying the connection**: check your MQTT broker's client list — the device will appear within a few seconds of booting.

---

## Configuration Reference

All settings are persisted to NVS flash and survive power cycles. They can be changed at any time by visiting `http://<device-ip>/` while the device is on your network.

| Field | NVS Key | Type | Default | Description |
|-------|---------|------|---------|-------------|
| WiFi SSID | `wifi_ssid` | string | — | 2.4 GHz network name |
| WiFi password | `wifi_pass` | string | — | Passphrase |
| MQTT host | `mqtt_host` | string | — | Broker IP or hostname; leave blank to disable MQTT |
| MQTT port | `mqtt_port` | uint16 | 1883 | Broker TCP port |
| MQTT username | `mqtt_user` | string | — | Optional credential |
| MQTT password | `mqtt_pass` | string | — | Optional credential |
| Topic prefix | `mqtt_prefix` | string | `ups2mqtt` | MQTT topic root |
| Poll interval | `poll_ms` | uint32 | 5000 | Publish period in ms |
| Syslog host | `syslog_host` | string | — | Leave blank to disable syslog |
| Syslog port | `syslog_port` | uint16 | 514 | UDP port |
| Syslog format | `syslog_fmt` | uint8 | 0 | 0 = RFC 3164, 1 = CEF |
| NTP time sync | `ntp_enabled` | uint8 | 1 | 0 = disabled, 1 = enabled |
| NUT server | `nut_enabled` | uint8 | 0 | 0 = disabled, 1 = enabled |
| NUT port | `nut_port` | uint16 | 3493 | TCP port for NUT clients |
| NUT UPS name | `nut_name` | string | `ups` | Name clients query as `<name>@<device-ip>` |

**Blank password fields on the config form preserve the currently saved value.** You do not need to re-enter your WiFi or MQTT password when changing other settings.

---

## MQTT Integration

The device uses a **device ID** derived from its WiFi MAC address in the form `UPS2MQTT-XXXXXX`. This ID is stable across reboots and uniquely identifies the device on the broker. You can find it on the `/debug` page under System > Device ID.

### Published: `<prefix>/<device-id>/ups/telemetry`

A retained JSON object published every `mqtt_interval_s` seconds (default 60):

```json
{
  "serial":          "R3P2024001234",
  "total_load_w":    258.77,
  "total_draw_w":    0.00,
  "ac_draw_w":       0.00,
  "ac_load_w":       258.77,
  "solar_dc_w":      0.00,
  "dc_load_w":       0.00,
  "usba_w":          12.50,
  "usbc_w":          45.00,
  "charge_pct":      83,
  "batt_time_min":   52,
  "charge_time_min": -1,
  "temp_sys":        28,
  "temp_batt":       24,
  "flags":           4,
  "cap_mah":         12800,
  "line_freq_hz":    50.0,
  "batt_v":          25.60,
  "batt_nom_v":      25.60,
  "data_age_s":      3,
  "data_stale":      false
}
```

| Field | Unit | Notes |
|-------|------|-------|
| `serial` | — | EcoFlow device serial number |
| `total_load_w` | W | Sum of all output loads |
| `total_draw_w` | W | Sum of all input sources |
| `ac_draw_w` | W | AC mains input |
| `ac_load_w` | W | AC inverter output |
| `solar_dc_w` | W | Solar / DC input |
| `dc_load_w` | W | DC barrel output |
| `usba_w` | W | USB-A output (both ports combined) |
| `usbc_w` | W | USB-C output |
| `charge_pct` | % | State of charge (0–100) |
| `batt_time_min` | min | Estimated runtime; −1 = not discharging |
| `charge_time_min` | min | Estimated time to full; −1 = not charging |
| `temp_sys` | °C | System temperature |
| `temp_batt` | °C | Battery temperature |
| `flags` | — | Bitmask of device status flags |
| `cap_mah` | mAh | Design capacity (12,800 mAh for River 3 Plus) |
| `line_freq_hz` | Hz | AC line frequency |
| `batt_v` | V | Current battery voltage |
| `batt_nom_v` | V | Nominal battery voltage |
| `data_age_s` | s | Age of the UPS readings when this message was published; −1 = no data received since boot |
| `data_stale` | — | `true` when `data_age_s` exceeds max(60 s, 3 × `poll_ms`) or is −1. The other fields are the last known values and are still published — treat them as frozen |

### Published: `<prefix>/<device-id>/system/telemetry`

A retained JSON object published every `mqtt_interval_s` seconds (default 60) with device health information:

```json
{
  "device_id":  "UPS2MQTT-A1B2C3",
  "version":    "1.0.0",
  "heap_free":  198432,
  "uptime_s":   3600,
  "wifi_rssi":  -62,
  "wifi_ssid":  "HomeNetwork",
  "ip":         "192.168.1.105"
}
```

### Subscribed: `<prefix>/<device-id>/ups/ota`

Publish a plain-HTTP URL to this topic to trigger an OTA firmware update:

```
mosquitto_pub -h 192.168.1.10 -t ups2mqtt/UPS2MQTT-A1B2C3/ups/ota \
              -m "http://192.168.1.10:8080/firmware.bin"
```

---

## NUT (Network UPS Tools) Server

UPS2MQTT can present itself as a read-only [NUT](https://networkupstools.org)
server, so any tool that already speaks the NUT protocol — `upsc`, Home
Assistant's NUT integration, Synology and TrueNAS UPS pages — can read the
telemetry without going through MQTT.

Enable it under **NUT Server** on the config page. It is off by default.
The defaults (port 3493, UPS name `ups`) suit almost every client.

Query it with the standard client tools:

```bash
upsc ups@<device-ip>                 # all variables
upsc ups@<device-ip> battery.charge  # one variable
upsc -l <device-ip>                  # list UPS names
```

Exposed variables:

| Variable | Meaning |
|----------|---------|
| `ups.status` | `OL` (on line), `OB` (on battery), plus `CHRG`, `DISCHRG`, `LB`, `OVER`, `RB` modifiers |
| `battery.charge` | Charge percentage |
| `battery.runtime` | Runtime remaining in seconds (absent while not discharging) |
| `battery.voltage`, `battery.voltage.nominal` | Battery voltage in V |
| `ups.realpower` | Total load in W |
| `ups.temperature` | System temperature in °C |
| `input.frequency` | AC line frequency in Hz |
| `device.mfr`, `device.model`, `device.serial`, `device.type` | Device identity |
| `ups.mfr`, `ups.model`, `ups.serial` | Same identity under the `ups.*` names some clients prefer |
| `driver.name`, `driver.version` | Driver name and firmware version |

### Limitations

This is a **read-only status server**, not a full NUT UPS driver:

- **No authentication.** Anyone on your network can read the status, the
  same as the existing `/status` HTTP endpoint. Do not expose port 3493 to
  the internet.
- **No shutdown support.** `upsmon` can query the device but cannot
  register as a monitor or trigger a coordinated shutdown — `LOGIN`, `FSD`
  and instant commands are not implemented. Use MQTT plus a Home Assistant
  automation if you need shutdown actions.
- **Three concurrent clients maximum.** A fourth connection is refused.
- Variables are read-only; `LIST RW` and `LIST CMD` are always empty.

If the UPS has not been read yet (for example immediately after boot), or
the last successful read is older than the staleness window (USB unplugged,
or the UPS stopped responding), variable queries answer `ERR DATA-STALE`
rather than reporting stale or invented values. That window is at least
**60 seconds**, but scales up with your configured poll interval so that a
slower poll rate does not cause false staleness reports — it is always
enough to tolerate a couple of missed polls. `upsc` reports this as
`Error: Data stale`. Reconnecting USB clears it on the next successful
poll.

The UPS name you enter on the config page is restricted to letters,
digits, `.`, `_` and `-`. Anything else is stripped, and a blank name
becomes `ups` — the NUT protocol separates fields with spaces, so a name
containing a space or quote could never be queried by a client.

---

## Home Assistant Setup

### Auto-Discovery (Recommended)

When the device first connects to the broker it automatically publishes Home Assistant MQTT discovery payloads for **16 sensors** and **2 binary sensors**, all grouped under a single device entry.

No YAML configuration is needed. After the device connects, navigate to **Settings → Devices & Services → MQTT** and the device will appear automatically.

| Entity | Class | Unit |
|--------|-------|------|
| WiFi RSSI | signal_strength | dBm |
| Uptime | duration | s |
| Heap Free | data_size | B |
| Battery | battery | % |
| Load | power | W |
| Draw | power | W |
| AC Draw | power | W |
| AC Load | power | W |
| Solar/DC | power | W |
| USB-A | power | W |
| USB-C | power | W |
| Runtime | duration | min |
| Charge Time | duration | min |
| Voltage | voltage | V |
| UPS Data Age | duration | s |
| IP Address | — (diagnostic) | — |
| Connectivity | connectivity | — |
| UPS Data Stale | problem | — |

**UPS Data Stale** turns on when the dongle is still online but the River 3 Plus has stopped
answering over USB (cable unplugged, unit off, protocol failure). **UPS Data Age** and
**UPS Data Stale** both have `expire_after` set to 3 × `mqtt_interval_s`, so if the dongle itself
stops publishing they go `unavailable` rather than showing the last retained value forever.
A useful automation is to alert on *UPS Data Stale* being `on` **or** `unavailable`.

### Manual YAML (Alternative)

Replace `UPS2MQTT-A1B2C3` with your actual device ID:

```yaml
mqtt:
  sensor:
    - name: "River 3 Plus Charge"
      state_topic: "ups2mqtt/UPS2MQTT-A1B2C3/ups/telemetry"
      value_template: "{{ value_json.charge_pct }}"
      unit_of_measurement: "%"
      device_class: battery
      state_class: measurement

    - name: "River 3 Plus Load"
      state_topic: "ups2mqtt/UPS2MQTT-A1B2C3/ups/telemetry"
      value_template: "{{ value_json.total_load_w }}"
      unit_of_measurement: "W"
      device_class: power
      state_class: measurement

    - name: "River 3 Plus Battery Time"
      state_topic: "ups2mqtt/UPS2MQTT-A1B2C3/ups/telemetry"
      value_template: "{{ value_json.batt_time_min }}"
      unit_of_measurement: "min"
      icon: mdi:timer-outline
```

Your device ID is visible on the `/debug` page under System > Device ID.

---

## OTA Firmware Updates

Firmware updates are delivered over plain HTTP. You need an HTTP server accessible from the device's network.

Download the **OTA app image** (`UPS2MQTT-production-vX.Y.Z.bin`, not the `-factory`
image) from this repository's [Releases](../../releases) page, then either:

- **Upload it in the browser:** open `http://<device-ip>/ota` and upload the file, or
- **Trigger it over MQTT:** serve the file from any machine on your network and publish
  its URL to the device's OTA topic:

```bash
# Serve the downloaded file
python3 -m http.server 8080

# Trigger the update
mosquitto_pub -h <broker-ip> \
  -t "ups2mqtt/<device-id>/ups/ota" \
  -m "http://<your-pc-ip>:8080/UPS2MQTT-production-vX.Y.Z.bin"
```

Check the file against `SHA256SUMS.txt` from the same release before flashing.

The device downloads the binary, flashes it to the inactive OTA partition, and reboots. If the download fails, the device stays on the current firmware — it does not brick.

---

## Factory Reset

Hold the **BOOT button** for **5 seconds**.

The status LED will flash while the button is held. After 5 seconds the device erases its configuration and reboots into provisioning mode (SoftAP).

**When to use factory reset:**
- You have moved to a new WiFi network and cannot reach the config page
- You want to hand the device to someone else

Releasing the button before 5 seconds cancels the reset.

---

## Status API

```
GET http://<device-ip>/status
```

Returns a JSON snapshot of all subsystems:

```json
{
  "system": {
    "device_id":          "UPS2MQTT-A1B2C3",
    "version":            "1.0.0",
    "heap_free":          198432,
    "uptime_ms":          3600000,
    "cpu_temp":           43.2,
    "ntp_synced":         true
  },
  "debug": { "msg": "OK" },
  "wifi": {
    "state": "connected",
    "ssid":  "HomeNetwork",
    "rssi":  -62,
    "ip":    "192.168.1.105"
  },
  "mqtt":   { "connected": true },
  "ota":    { "state": "idle" },
  "ups": {
    "charge_pct":      83,
    "total_load_w":    258.77,
    "batt_time_min":   52
  }
}
```

The `debug.msg` field is useful for diagnosing USB issues without a serial monitor — it shows the current USB subsystem state (e.g. `"OK"`, `"CDC Open Fail"`, `"Response timeout"`).

---

## Debug Dashboard

```
GET http://<device-ip>/debug
```

Auto-refreshes every 2 seconds. Shows WiFi, MQTT, OTA, and UPS telemetry in colour-coded cards, including:

- Device ID and firmware version
- CPU temperature
- RSSI bar with signal strength indicator
- UPS `debug.msg` status
- Seconds since last UPS data received

---

## Diagnostics & Safe Mode

### Safe Mode

If the device panics and reboots three or more times in a row, it enters **safe mode** on the next boot:

- The USB host is **not started** — breaking the crash loop
- WiFi, MQTT, and the HTTP server remain up
- The `/debug` page shows `SAFE MODE`

To exit safe mode after resolving the underlying issue (typically via OTA):

```
POST http://<device-ip>/reset_panics
```

This clears the panic counter. The device will start the USB host on the next reboot.

A factory reset (hold BOOT 5 s) also exits safe mode, but erases all configuration.

---

## Troubleshooting

### The device does not create a SoftAP

Unplug and replug the device. If it still does not appear, hold the BOOT button for 5 seconds to factory-reset.

### The captive portal page does not open automatically

Navigate manually to `http://192.168.4.1`. Some browsers suppress captive portal dialogs on networks without internet access.

### MQTT is not receiving messages

1. Visit `http://<device-ip>/debug` — confirm MQTT shows `connected`
2. If not connected, verify the broker IP and port in the config form
3. Confirm the broker allows the configured username/password (or unauthenticated connections)
4. Use `mosquitto_sub -h <broker> -t "ups2mqtt/#" -v` to watch all traffic

### UPS data is missing from `/debug` after several minutes

Check `debug.msg` on the debug page:

| `debug.msg` | Likely cause |
|-------------|--------------|
| `Connecting to UPS...` | CDC open in progress — wait a few seconds |
| `CDC Open Fail (N): ...` | Interface not found — check USB cable and EcoFlow comm port |
| `Response timeout` | Cable connected but no data — try replugging |
| `Bad response frame` | Corrupt or incomplete data — likely a cable issue |
| `Write failed` | Device disconnected mid-poll |

### OTA update fails with "URL too long"

The OTA URL must be under 256 characters. Use a shorter hostname or path.

### Home Assistant entities show "unavailable"

- Confirm the device is connected to MQTT (`/debug`)
- Check the broker for `homeassistant/sensor/UPS2MQTT-XXXXXX/...` retained messages
- Reload the MQTT integration in HA: Settings → Devices & Services → MQTT → Reload
- If only **UPS Data Age** / **UPS Data Stale** are unavailable, the dongle has stopped
  publishing for over 3 × `mqtt_interval_s` — check its power and WiFi
