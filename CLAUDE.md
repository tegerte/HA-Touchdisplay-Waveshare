# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

ESPHome configuration for a **Waveshare ESP32-S3-Touch-LCD-7** (800×480) integrated with Home Assistant. Portiert vom Schwesterprojekt unter `../ha_7zoll_disp/` (Makerfabs MAtouch ESP32-S3 7", 1024×600). Sensorlogik und HA-Integration sind 1:1 übernommen; UI-Layout neu skaliert.

## Key Commands

```bash
# Validate
esphome config ha_7zoll_waveshare_esp32-s3-touch.yaml

# Compile only
esphome compile ha_7zoll_waveshare_esp32-s3-touch.yaml

# OTA upload (Standardweg)
esphome run ha_7zoll_waveshare_esp32-s3-touch.yaml --device waveshare-esp32s3-7.local

# USB upload (nur Erstflash oder bei OTA-Problemen)
esphome run ha_7zoll_waveshare_esp32-s3-touch.yaml --device /dev/cu.usbmodem<...>

# Logs (App-Logs kommen meist nicht durch — siehe Notes)
esphome logs ha_7zoll_waveshare_esp32-s3-touch.yaml --device waveshare-esp32s3-7.local
```

Aktuelle Geräte-IP: `192.168.1.99`. mDNS-Port-5353-Warnung auf macOS ist harmlos.

## Architecture

Alles in **`ha_7zoll_waveshare_esp32-s3-touch.yaml`**.

**Hardware** wird über das ESPHome-Package [inytar/waveshare-esp32-s3-touch-lcd-7-esphome](https://github.com/inytar/waveshare-esp32-s3-touch-lcd-7-esphome) (`packages:`) abstrahiert — liefert: PSRAM, ESP32-S3, I²C (GPIO8/9), CH422G-Expander, mipi_rgb-Display, GT911-Touch (INT=GPIO4, RESET via CH422G), binäres Backlight (Light-Entity `lcdbacklight`), Antiburn-Schutz. Erfordert ESPHome ≥ 2025.8.0.

**PSRAM-Override:** Das Package setzt 120 MHz octal (experimentell) — das verursacht eine Boot-Loop. Daher per Top-Level-`psram:`-Block auf 80 MHz reduziert.

**UI-Struktur (LVGL):**

- Globales Theme + `header_footer`-Style (analog MAtouch)
- Bottom-Navigation als `buttonmatrix` im `top_layer` (Automationen / Home / Laden)
- Drei Pages: `main_page`, `Licht`, `Charge`

**main_page Layout (800×480, y=30..450 nutzbar):**

- 4 vertikale Bars links: w=40, h=350, x=10/60/110/160
  - Container y=35..385, Value-Label (mont_14) bei y=388, Icon-Label (icons_30) bei y=415
  - Lambdas: Faktor 3.5 für % (max 350 px), `/400.0f * 350.0f` für Mercedes-km, `/11.0f * 350.0f` für EV-kW
  - Bar 4 (EV) hat gelben 1-Phasen-Marker bei y=271 (= 3.6 kW / 11 kW × 350 = 114 px vom Boden)
- Thermometer: x=215, y=40, **width=190 / height=200** (Pixel-Aspekt-Korrektur — Panel-Pixel sind breiter als hoch), kein Border
- Solar-Tabelle: x=420, y=35, w=370, h=210 — montserrat_28 Werte, montserrat_16 Labels
- Graph: x=210, y=255, w=580, h=185 (JPEG vom Homeserver, gleiche Pipeline wie MAtouch)

**Charge Layout:**

- 4 Mode-Buttons w=95 h=60 mit gap=5: x=10/110/210/310, y=45 (PV/Min+PV/Schnell/Aus)
- Mode/SoC-Labels: y=115
- Plan-Button: x=10, y=150, w=395, h=70 ("60% bis 7:00")
- PV-Start-Button: x=10, y=240, w=395, h=70
- Horizontale Fill-Bars: 11 kW → 395 px, 1-Phasen-Marker bei x=139 (= 3.6 kW × 395 / 11 + x_offset 10)
- EV-Tabelle: x=420, y=35, w=370, h=370 — **montserrat_22 / _22_ext** Werte, montserrat_14 Labels (kompakter als Solar-Tabelle wegen 9 Zeilen)

**HA-Integration:** Identisch zum MAtouch — selbe Entity-IDs (`sensor.evcc_*`, `select.evcc_e_auto_laden_mode`, `light.remote_light`, `input_boolean.evcc_ladeplan_aktiv`, alle `cw_mt_891_e_*`-Mercedes-Sensoren). API-Encryption-Key und OTA-Passwort sind beim Port frisch generiert worden — Waveshare läuft als eigenständiges HA-Gerät parallel zum MAtouch.

**Fonts:**

- `montserrat_14/16/18/20/22/28`: LVGL built-in
- `montserrat_22_ext` / `montserrat_28_ext`: gfonts mit €-Glyph (U+20AC) — für Kostenfelder
- `icons_20` (Tabellen), `icons_30` (Bar-Labels & Charge-Buttons), `icons_100` (Lichtbutton): MDI-Codepoints aus `fonts/materialdesignicons-webfont.ttf` — beim Hinzufügen neuer Icons Codepoint zur `glyphs`-Liste ergänzen

## Hardware-Quirks

1. **UART-Switch auf der Platine** schaltet zwischen den beiden Board-UARTs. USB-Upload funktioniert nur in der einen Stellung — falls `esptool` "No serial data received" meldet, Switch umlegen. Bei Bedarf zusätzlich BOOT halten + RST kurz tippen + BOOT loslassen, dann sofort `esphome run --device /dev/cu.usbmodem...`.

2. **App-Logs landen nicht zuverlässig auf der CH343-UART** — vermutlich routet die Switch-Konfiguration sie auf USB-Serial-JTAG. Nur ROM-Boot-Codes (`rst:0x1 POWERON` / `rst:0xc RTC_SW_CPU_RST`) sind über die normale serielle Schnittstelle sichtbar. Fürs Debugging stattdessen Display, HA-Logs oder OTA-Logger nutzen.

3. **Panel-Pixel sind nicht quadratisch** — sie sind ~5–10 % breiter als hoch. Bei runden LVGL-Widgets (Meter) Width entsprechend kleiner als Height setzen, sonst wird's breit-oval.

## Important Notes

- OTA-Flash ist der Standardweg (USB nur für Erstflash); explizit `--device waveshare-esp32s3-7.local` angeben, sonst will esphome interaktiv USB/OTA wählen und scheitert in non-interactive Shells mit `EOFError`
- Backlight ist nur binär (`light.lcdbacklight`) — kein PWM-Dimming auf diesem Board ohne Hardware-Mod
- Bei UI-Änderungen die Lambdas in den Sensoren mit den Container-Höhen synchron halten: Wenn `height: 350` der Bars geändert wird, müssen alle 4 Lambdas (Faktor + Maximum + Top-Position) und der Yellow-Marker (`y: 271`) mit angepasst werden
- Build-Artefakte unter `.esphome/` (gitignored)
- `secrets.yaml` enthält WiFi-Credentials und ist gitignored
