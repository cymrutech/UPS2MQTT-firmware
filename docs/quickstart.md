# UPS2MQTT — Quick Start Guide

**UPS2MQTT** is a small Wi-Fi dongle that reads live data from your **EcoFlow River 3 Plus** and sends it to your MQTT broker. Once set up, it publishes battery level, load, temperatures, and more every 5 seconds — with automatic Home Assistant discovery included.

---

## What You Need

- UPS2MQTT board (the dongle)
- USB Type-A to Type-B cable (standard "printer" cable)
- A USB charger or power bank to power the dongle (5V, any USB-A port — including one on the River 3 Plus itself)
- Your Wi-Fi network name and password
- An MQTT broker on your local network (e.g. Mosquitto)

---

## Step 1 — Connect the hardware

1. Plug the **Type-B end** of the cable into the **Comm port** on the back of the River 3 Plus.
2. Plug the **Type-A end** into the UPS2MQTT board's USB socket.
3. Power the UPS2MQTT board from a USB charger (a spare USB-A port on the front of the River 3 Plus works fine).

The status LED will light up as soon as the board has power.

---

## Step 2 — Connect to the setup Wi-Fi

On first boot the dongle creates its own temporary Wi-Fi network called **`UPS2MQTT-XXXXXX`** (where XXXXXX is unique to your board).

1. On your phone or laptop, open Wi-Fi settings and connect to **`UPS2MQTT-XXXXXX`**.
2. A setup page should pop up automatically. If it doesn't, open a browser and go to **`http://192.168.4.1`**.

---

## Step 3 — Enter your details

Fill in the form:

| Field | What to enter |
|-------|--------------|
| **Wi-Fi Network** | Your home Wi-Fi name (2.4 GHz only) |
| **Wi-Fi Password** | Your Wi-Fi password |
| **MQTT Broker** | IP address of your MQTT broker, e.g. `192.168.1.10` |
| **MQTT Port** | Leave as `1883` unless you've changed it |
| **MQTT Username / Password** | Only if your broker requires login |
| **Topic Prefix** | Leave as `ups2mqtt` unless you want to change it |

Leave everything else as default.

Tap **Save & Restart**. The board saves your settings and reboots — this takes about 5 seconds.

---

## Step 4 — Verify it's working

### Home Assistant
Navigate to **Settings → Devices & Services → MQTT**. The device will appear automatically as **UPS2MQTT-XXXXXX** with 14+ sensors ready to use. No YAML needed.

### MQTT (without Home Assistant)
Subscribe to `ups2mqtt/#` on your broker. You should see a JSON message arrive within a few seconds containing battery level, load, temperatures, and more.

---

## Changing settings later

Once the dongle is on your network, find its IP address from your router's device list (it will show up as **UPS2MQTT-XXXXXX**), then open `http://<ip-address>/` in a browser to update any settings.

---

## Factory reset

If you need to start over — for example, if you've moved to a new Wi-Fi network:

Hold the small **BOOT button** on the board for **5 seconds**.

The LED will flash while the button is held. After 5 seconds the board erases its settings and restarts in setup mode, showing the `UPS2MQTT-XXXXXX` Wi-Fi network again.

---

## LED status guide

| LED colour | What it means |
|------------|--------------|
| Pulsing blue | Setup mode — waiting for configuration |
| Solid green | Connected and publishing data |
| Flashing amber | Connecting to Wi-Fi or MQTT |
| Flashing red | USB connection to the River 3 Plus failed — check the cable |

---

## Need help?

Visit `http://<device-ip>/debug` in your browser for a live diagnostics page showing Wi-Fi signal, MQTT status, and the latest data from the River 3 Plus.
