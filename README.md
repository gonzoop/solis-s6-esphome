# Solis S6 EH3P — ESPHome + Home Assistant Configuration

ESPHome configuration and Home Assistant dashboard for the **Solis S6 EH3P three-phase hybrid inverter**, using a LilyGo T-RSC3 (ESP32 with RS485) to poll the inverter directly over Modbus RTU.

> **This project is built on top of the excellent [solis_modbus](https://github.com/Pho3niX90/solis_modbus) Home Assistant integration by [@Pho3niX90](https://github.com/Pho3niX90).** Register maps, scaling factors, and entity structure are derived from that project. All credit for the underlying Modbus research goes to the original authors.

---

## Hardware

| Component | Details |
|---|---|
| Inverter | Solis S6-EH3P (3-phase hybrid) |
| Bridge | LilyGo T-RSC3 (ESP32-S3 + RS485) |
| Connection | RS485 to inverter COM port |
| Firmware | ESPHome 2026.2+ |

---

## What's included

```
esphome/
  solis-lilygo.yaml           # Main config — native HA API variant
  solis-lilygo-mqtt.yaml      # Main config — MQTT variant (standalone / any MQTT broker)
  solis-binary-sensors.yaml   # Operation status & mode flag bits       (shared by both)
  solis-sensors.yaml          # All read-only measurements               (shared by both)
  solis-numbers.yaml          # Writable settings                        (shared by both)
  solis-selects.yaml          # Mode selectors                           (shared by both)
  solis-text-sensors.yaml     # Status strings                           (shared by both)
  secrets.yaml.example        # Template — copy to secrets.yaml and fill in

solis-dashboard.yaml          # Lovelace dashboard (4-view: Overview, Solar, Battery, Grid)
solis-sunsynk-card.yaml       # Sunsynk Power Flow Card config (animated energy flow diagram)
```

---

## Features

- **Full register coverage** — 115 sensors, 22 writable numbers, 2 selects, 22 binary sensors, 11 text sensors
- **Human-readable Inverter Status** — maps all 58 status/fault codes to plain English strings
- **Inverter RTC sync** — automatically corrects inverter clock if it drifts >42 seconds from NTP
- **Computed sensors** — PV Power 1–4 (V×I), Grid Power Net (signed import/export), Battery Power (signed charge/discharge)
- **4-view Lovelace dashboard** — Overview, Solar detail, Battery detail, Grid & AC detail
- **Sunsynk Power Flow Card** — animated real-time energy flow diagram with solar, battery, grid and load nodes including backup (EPS) port

---

## ESPHome setup

All files in `esphome/` must be in the same directory. The main config includes the five fragment files via ESPHome `packages:`.

### Option A — Native Home Assistant API (default)

1. Copy all files from `esphome/` into your ESPHome config directory (`/config/esphome/`)
2. Copy `secrets.yaml.example` → `secrets.yaml` and set your WiFi credentials
3. Edit `solis-lilygo.yaml` if your inverter's Modbus slave address differs from `1`
4. Flash:
   ```
   esphome run esphome/solis-lilygo.yaml
   ```
5. Accept the integration in Home Assistant — all entities appear automatically

### Option B — MQTT (standalone / no direct HA integration needed)

Use this variant when you want the ESP32 to publish all readings to an MQTT broker directly — works with any MQTT client (Node-RED, Telegraf, Grafana, plain subscribers) and still supports Home Assistant via MQTT discovery.

1. Copy all files from `esphome/` into your ESPHome config directory
2. Copy `secrets.yaml.example` → `secrets.yaml` and fill in WiFi **and** MQTT credentials:
   ```yaml
   wifi_ssid: "YourNetwork"
   wifi_password: "YourPassword"
   mqtt_broker: 192.168.x.x      # your broker IP or hostname
   mqtt_username: mqtt_user
   mqtt_password: mqtt_password
   ```
3. Flash:
   ```
   esphome run esphome/solis-lilygo-mqtt.yaml
   ```

**MQTT topic structure** (published automatically by ESPHome):

| Entity type | State topic | Command topic |
|---|---|---|
| Sensors | `solis-lilygo/sensor/{name}/state` | — |
| Binary sensors | `solis-lilygo/binary_sensor/{name}/state` | — |
| Numbers | `solis-lilygo/number/{name}/state` | `solis-lilygo/number/{name}/command` |
| Selects | `solis-lilygo/select/{name}/state` | `solis-lilygo/select/{name}/command` |
| Device status | `solis-lilygo/status` | — (`online` / `offline`) |

With `discovery: true` the device self-registers in Home Assistant via the MQTT integration — no manual entity configuration needed.

---

## Dashboard setup

### Main dashboard
1. In Home Assistant go to **Settings → Dashboards → Add Dashboard**
2. Create a new dashboard, then edit it in raw YAML mode
3. Paste the contents of `solis-dashboard.yaml`

The dashboard uses these HACS frontend cards — install them first via **HACS → Frontend**:

| Card | Repository |
|---|---|
| `custom:multiple-entity-row` | [benct/lovelace-multiple-entity-row](https://github.com/benct/lovelace-multiple-entity-row) |
| `custom:sunsynk-power-flow-card` | [slipx06/sunsynk-power-flow-card](https://github.com/slipx06/sunsynk-power-flow-card) |

### Sunsynk Power Flow Card
The file `solis-sunsynk-card.yaml` can be added as a **manual card** anywhere in your dashboard. Adjust these values for your system:

```yaml
battery:
  energy: 14400        # Your battery capacity in Wh
  shutdown_soc: 10     # Your overdischarge SOC %

solar:
  max_power: 15000     # Your total PV peak watts
  pv1_name: String 1   # Rename to your string locations
```

---

## Customisations vs the original register map

The following changes were made relative to the base register definitions in [solis_modbus](https://github.com/Pho3niX90/solis_modbus):

| Change | Detail |
|---|---|
| Inverter Status (33095) | Split into internal raw sensor + text_sensor with 58-case lambda mapping codes to plain English |
| Grid Power Net | Negated meter reading so positive = importing, negative = exporting |
| Lead-acid Battery Temperature | NaN filter for disconnected sensor (`< -100°C → unavailable`) |
| Meter Grid Frequency | NaN filter for zero readings when meter is offline |
| PV Power 1–4 | Computed template sensors (Voltage × Current) since register 33057 is total only |
| Inverter RTC Sync | Template sensor checks inverter clock vs NTP and writes correction if drift > 42s |
| Fragment split | Single 2300-line file split into 6 files by entity type to prevent duplicate section key errors |

---

## Credits

- **[Pho3niX90/solis_modbus](https://github.com/Pho3niX90/solis_modbus)** — Home Assistant Solis Modbus integration; register maps, scaling factors, and entity definitions that this ESPHome config is based on
- **[fboundy/ha_solis_modbus](https://github.com/fboundy/ha_solis_modbus)** — the original project that inspired solis_modbus
- **[slipx06/sunsynk-power-flow-card](https://github.com/slipx06/sunsynk-power-flow-card)** — animated power flow card
- **[benct/lovelace-multiple-entity-row](https://github.com/benct/lovelace-multiple-entity-row)** — multi-value entity rows used in the detail views

---

## License

MIT — free to use, modify and share. If you improve it, consider opening a PR or filing an issue on the original [solis_modbus](https://github.com/Pho3niX90/solis_modbus) repo.
