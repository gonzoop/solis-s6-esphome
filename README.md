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
  solis-lilygo.yaml           # Main ESPHome device config (WiFi, time, modbus master)
  solis-binary-sensors.yaml   # Operation status & mode flag bits
  solis-sensors.yaml          # All read-only measurements (power, energy, temp, etc.)
  solis-numbers.yaml          # Writable settings (SOC limits, charge currents, etc.)
  solis-selects.yaml          # Mode selectors (Work Mode, RC Force Charge/Discharge)
  solis-text-sensors.yaml     # Status strings (Inverter Status, Work Mode, serial, RTC sync)

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

1. Copy all files from `esphome/` into your ESPHome config directory (`/config/esphome/`)
2. Edit `solis-lilygo.yaml`:
   - Set your WiFi credentials
   - Set your inverter's Modbus slave address if different from `1`
3. Flash to your device via ESPHome dashboard or CLI:
   ```
   esphome run esphome/solis-lilygo.yaml
   ```

The main file uses ESPHome `packages:` to include the five fragment files — they must all be in the same directory.

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
