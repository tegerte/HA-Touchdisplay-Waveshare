# HA Touch Display – Waveshare ESP32-S3-Touch-LCD-7

ESPHome-Konfiguration für das **Waveshare ESP32-S3-Touch-LCD-7** (800×480), integriert in Home Assistant.

Portiert vom MAtouch-Schwesterprojekt unter `../ha_7zoll_disp/` (1024×600). Sensorlogik, EVCC-Skripte und der Online-Image-Graph wurden 1:1 übernommen; das LVGL-Layout ist auf 800×480 neu skaliert.

## Hardware

- **MCU:** ESP32-S3, 8 MB Flash, Octal PSRAM @ 80 MHz, 240 MHz CPU
- **Display:** 800×480 RGB-Parallel (mipi_rgb)
- **Touch:** GT911 kapazitiv (I²C, INT auf GPIO4)
- **I/O-Expander:** CH422G (steuert Backlight, Display-Reset, Touch-Reset)
- **Backlight:** binär via CH422G (kein PWM-Dimming auf diesem Board ohne Hardware-Mod)

Das gesamte Hardware-Setup wird via [inytar/waveshare-esp32-s3-touch-lcd-7-esphome](https://github.com/inytar/waveshare-esp32-s3-touch-lcd-7-esphome) als ESPHome-Package eingebunden — erfordert ESPHome ≥ 2025.8.0. PSRAM wird per Override auf **80 MHz** zurückgesetzt (das vom Package gesetzte 120 MHz octal verursacht auf diesem Board eine Boot-Loop).

## Gehäuse

3D-druckbares Gehäuse: `esp32-s3-touch-lcd-7-case-5.stl` (im Repo). Quelle: [Waveshare ESP32-S3 7inch Capacitive Touch Display Case auf Printables](https://www.printables.com/model/1425850-waveshare-esp32-s3-7inch-capacitive-touch-display) — passt exakt auf das Waveshare ESP32-S3-Touch-LCD-7-Board.

## UI-Seiten

| Seite       | Inhalt                                                                        |
| ----------- | ----------------------------------------------------------------------------- |
| `main_page` | Header mit KW/Datum/Uhrzeit, Thermometer, 4 vertikale Status-Bars, Solar-Tabelle, Graph |
| `Licht`     | Bürolicht-Toggle                                                              |
| `Charge`    | EV-Lademodus (4 Buttons) + Plan/PV-Start-Buttons + EV-Info-Tabelle (9 Zeilen) |

**Header der Hauptseite:** Zeigt Kalenderwoche, Datum (`dd.mm.yy`) und Uhrzeit (`hh:mm`) im Format `KW18    -    04.05.26    -    23:01`. Update jede volle Minute via `time.on_time` und einmalig sofort nach dem ersten HA-Time-Sync via `on_time_sync`. Schrift `montserrat_22`, Header-Höhe 34 px (auf der `main_page` per Override; andere Pages bleiben bei den Default-30 px).

Navigation via Buttonmatrix am unteren Rand (`Automationen` / Home-Icon / `Laden`).

## Vertikale Status-Balken (linke Seite, main_page)

Vier vertikale Bars, je w=40 / h=350 px:

| Bar  | Farbe        | Sensor                                  | Skala  |
| ---- | ------------ | --------------------------------------- | ------ |
| 1    | Blau         | EV Reichweite (`mercedes_range`)        | 400 km |
| 2    | Orange       | PV-Leistung (`evcc_pv_power`)           | 10 kW  |
| 3    | Grün         | Hausbatterie SoC (`evcc_battery_soc`)   | 100 %  |
| 4    | Dunkelgrün   | EV-Ladeleistung (`evcc_e_auto_laden_charge_power`) | 11 kW  |

Bar 4 trägt eine **gelbe Markierungslinie bei 3.6 kW** (1-phasige Lade-Schwelle). Werte werden via Lambda als Fill-Höhe berechnet (Faktor 3.5 bei %, sonst proportional).

## Thermometer

LVGL-Meter-Widget, 190×200 px (asymmetrisch — die Panel-Pixel sind breiter als hoch, daher kompensiert). Zwei überlagerte Skalen: eine treibt die Nadel (Skala −100..400 = °C × 10), die andere zeichnet die −10..40 °C Tick-Beschriftung. Außenrahmen entfernt für sauberen Look.

## Charge-Page

- **4 Modus-Buttons** (PV / Min+PV / Schnell / Aus) — rufen `script.evcc_lademodus` mit `data: modus:` auf, aktiver Modus wird grün via `select.evcc_e_auto_laden_mode` markiert
- **Ladeplan-Button** "60% bis 7:00" — togelt zwischen `script.evcc_ladeplan_morgen` / `script.evcc_ladeplan_loeschen` je nach `input_boolean.evcc_ladeplan_aktiv`
- **PV-Start-Button** — aktiviert MinPV via `script.evcc_minpv_aktivieren`
- Beide großen Buttons zeigen **horizontale Fill-Bars**: Ladeleistung (oben, dunkelgrün, 3.6-kW-Marker bei x=139) und Reichweite (unten, orange)
- **EV-Info-Tabelle** rechts mit 9 Zeilen — kompaktere Schrift (`montserrat_22` / `_22_ext` für €) als die Solar-Tabelle (`montserrat_28`), weil mehr Zeilen auf weniger Höhe

## Solar-Energie-Graph

Identisch zum MAtouch-Setup:

```
HA Automation (browser_mod) → Python-Server :8765 → ESP32 (online_image)
```

JPEG wird alle 2 min gepullt und in einem `image`-Widget (580×185 px) angezeigt. Server-Setup, browser_mod und apexcharts-Abhängigkeiten sind unter `../ha_7zoll_disp/README.md` dokumentiert.

## Home Assistant Integration

Sensoren werden 1:1 vom MAtouch-Projekt verwendet (selbe Entity-IDs, dieselben evcc/Mercedes/PV-Sensoren). **API-Key und OTA-Passwort sind beim Port frisch generiert worden** — das Waveshare läuft daher in HA als eigenständiges neues ESPHome-Gerät parallel zum MAtouch.

Zusätzlich vom inytar-Package bereitgestellt:

- `light.lcdbacklight` — Backlight an/aus (binär)
- Antiburn-Switch (Schutz vor Einbrennen)

## Dateien

- `ha_7zoll_waveshare_esp32-s3-touch.yaml` — komplette ESPHome-Konfiguration
- `secrets.yaml` — WiFi-Credentials (nicht im Repo)
- `fonts/materialdesignicons-webfont.ttf` — Material Design Icons

## Build & Flash

```bash
# Validieren
esphome config ha_7zoll_waveshare_esp32-s3-touch.yaml

# Compile only
esphome compile ha_7zoll_waveshare_esp32-s3-touch.yaml

# OTA-Upload (Standardweg — Device im WiFi)
esphome run ha_7zoll_waveshare_esp32-s3-touch.yaml --device waveshare-esp32s3-7.local

# USB-Upload (nur beim Erstflash oder wenn OTA fehlschlägt)
esphome run ha_7zoll_waveshare_esp32-s3-touch.yaml --device /dev/cu.usbmodem<...>

# Live-Logs
esphome logs ha_7zoll_waveshare_esp32-s3-touch.yaml --device waveshare-esp32s3-7.local
```

Aktuelle Geräte-IP: `192.168.1.99` (DHCP).

### USB-Flash-Quirks

Das Board hat einen **UART-Switch auf der Platine**, der die Verbindung zum CH343-USB-UART umlegt. Wenn der Erstflash mit `Failed to connect to ESP32-S3: No serial data received` scheitert:

1. Switch in die andere Stellung schieben
2. Erneut versuchen — Auto-Reset funktioniert dann via DTR/RTS
3. Falls weiterhin nicht: `BOOT` halten + `RST` kurz tippen + `BOOT` loslassen, sofort Upload starten

App-Logs landen *nicht* zuverlässig auf der CH343-UART (vermutlich routet die Switch-Konfiguration sie auf USB-Serial-JTAG). Nur die ROM-Boot-Codes (`rst:0x1 POWERON` = sauberer Boot, `rst:0xc RTC_SW_CPU_RST` = Software-Crash) sind sichtbar — fürs Debugging eher Display, HA-Logs oder OTA-Logger nutzen.

### macOS mDNS-Hinweis

`esphome run` zeigt auf macOS regelmäßig `Address in use when binding to (..., 5353)` — der mDNSResponder des Systems belegt den Port. Die Warnung ist harmlos, esphome resolved den Hostname trotzdem direkt.

## Resources

- [Waveshare ESP32-S3-Touch-LCD-7 Wiki](https://www.waveshare.com/wiki/ESP32-S3-Touch-LCD-7)
- [inytar ESPHome-Package](https://github.com/inytar/waveshare-esp32-s3-touch-lcd-7-esphome)
- [ESPHome LVGL Cookbook](https://esphome.io/cookbook/lvgl/)
- [Material Design Icons](https://materialdesignicons.com/)
