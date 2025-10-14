# Barkasse Hub (MQTT over Ethernet/WiFi)

A tiny, easy-to-extend sensor hub for the Barkasse project.

## Core Ideas
- Sensor nodes (ESP32/Arduino) publish JSON over MQTT via Ethernet/PoE or WiFi.
- A Raspberry Pi acts as a central hub with Mosquitto and a FastAPI + WebSocket UI.
- The UI auto-discovers new sensors and displays them live on a 10-inch touch panel.

## MQTT Topics & JSON Schema
- Topic: `barkasse/<node>/<cluster>/<sensor>`
- Payload:
  ```json
  {
  "node": "esp32p4-01",
  "cluster": "weather",
  "sensor": "temperature",
  "value": 22.4,
  "unit": "°C",
  "ts": "2025-10-10T12:00:00Z"
  }
  ```
## Directory Overview

| Folder                  | Description                                            |
| ----------------------- | ------------------------------------------------------ |
| `example-sensor-implementations/esp32p4-weatherstation-mock`  | ESP32-P4 (Ethernet) demo publishing weather data |
| `example-sensor-implementations/esp32-wroom-waterstation-mock`| ESP32 WROOM (WiFi) demo publishing water data    |
| `pi/docker-compose.yml` | Mosquitto MQTT broker                                  |
| `pi/app/`               | FastAPI backend with WebSocket + static touchscreen UI |
| `systemd/`              | Auto-update, backend, and kiosk startup services       |


## Setup on Raspberry Pi

### 1. Setup Mosquitto
```bash
cd pi
docker compose up -d
```

### 2. Setup Python environment
```bash
cd app
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

### 3. Enable services
```bash
sudo cp systemd/*.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now auto-git-update.service barkasse-ui.service chromium-kiosk.service
```

### Kiosk Mode

- Chromium launches automatically in fullscreen Wayland mode.  
- The UI is served at http://localhost:8080.  

## WiFi Access Point (no existing WiFi)

If there is no boat WiFi, run the Pi as an AP so ESP32 nodes can connect.

Network plan:
- Interface `wlan0`, static IP `192.168.50.1/24`
- SSID `barkasse-hub`, WPA2 passphrase (example) `barkasse1234`
- DHCP range `192.168.50.10–192.168.50.200`

Config (do not commit secrets; adjust to your OS):

`/etc/dhcpcd.conf`:
```
interface wlan0
static ip_address=192.168.50.1/24
nohook wpa_supplicant
```

`/etc/hostapd/hostapd.conf`:
```
interface=wlan0
ssid=barkasse-hub
hw_mode=g
channel=6
wmm_enabled=1
auth_algs=1
wpa=2
wpa_passphrase=barkasse1234
wpa_key_mgmt=WPA-PSK
rsn_pairwise=CCMP
```

`/etc/dnsmasq.d/barkasse-ap.conf`:
```
interface=wlan0
dhcp-range=192.168.50.10,192.168.50.200,255.255.255.0,12h
```

Mosquitto already listens on `0.0.0.0:1883`, reachable at `192.168.50.1:1883`.

## ESP32 Examples

### ESP32-P4 Weather (Ethernet)
See `example-sensor-implementations/esp32p4-weatherstation-mock`.

### ESP32 WROOM Water (WiFi)
Folder: `example-sensor-implementations/esp32-wroom-waterstation-mock`

1) Copy and edit credentials:
```
cp includes/secrets_example.h includes/secrets.h
// Set WIFI_SSID/WIFI_PASS to your AP (default shown above)
// Set MQTT_HOST=192.168.50.1, MQTT_USER/PASS to match Mosquitto
```
2) Build and flash with Arduino IDE (Board: ESP32). The node publishes:
- `barkasse/esp32wifi-01/status` (retained online/offline)
- `barkasse/esp32wifi-01/water/<sensor>` per sensor
- `barkasse/esp32wifi-01/water/state` cluster summary

The hub UI auto-discovers and renders without code changes.

## Extend
Add any new sensor/cluster by publishing to the topic contract. No UI edits required.

## Maintenance

- Check backend: systemctl status barkasse-ui.service
- Check kiosk: systemctl status chromium-kiosk.service
- Update manually: git pull && sudo systemctl restart barkasse-ui chromium-kiosk
