# DYMO LabelWriter 450 → ESP32-S3 Network Print Server

Turn a **USB-only DYMO LabelWriter 450** into a network printer using an
**ESP32-S3** as a transparent **raw TCP-9100 (JetDirect / AppSocket) → USB
Printer Class** bridge.

Once flashed, the label printer appears on the network as `hostname:9100`, and it
advertises itself over mDNS as a **“DYMO LabelWriter 450”** raw printer
(`_pdl-datastream._tcp`, port 9100), so Bonjour-aware clients can discover it. It
answers on the same `socket://<host>:9100` endpoint DYMO's own print server uses.

```
  DYMO software / CUPS  --TCP 9100-->  ESP32-S3  --USB bulk OUT-->  LabelWriter 450
```

This is a dedicated repo for the LabelWriter 450. The HP LaserJet 1010 lives in
its own repo: [`cosmicboy6/esp32-s3-hp1010-printserver`](https://github.com/cosmicboy6/esp32-s3-hp1010-printserver).

## Install

### Prebuilt firmware (browser, no build required)

A ready-to-flash **factory image** (`dymo-firmware.factory.bin`) is committed to
the repo root.

Connect a blank ESP32-S3 over USB and open the installer page:

> **https://cosmicboy6.github.io/esp32-s3-dymo450-printserver/**

It uses [ESP Web Tools](https://esphome.github.io/esp-web-tools/) (Web Serial), so
the browser flashes the board directly — nothing is uploaded. The installer is
served straight from this repo's root via GitHub Pages; an always-available
mirror is `https://raw.githack.com/cosmicboy6/esp32-s3-dymo450-printserver/main/index.html`.

**Or flash it from the command line** — the merged image goes at offset `0x0`:

```bash
esptool.py --chip esp32s3 write_flash 0x0 dymo-firmware.factory.bin
```

> **No secrets inside.** The prebuilt image uses placeholder WiFi, so the device
> boots into its fallback AP (`DYMO Print Fallback` / `dymosetup`); join it and set
> your real WiFi through the captive portal at `http://192.168.4.1`. It is compiled
> for the exact target board (**ESP32-S3 N16R8**). The baked-in API encryption key
> is `9GXBCDmqW1uiTOdRJC60k8qvZ9W/yYW0jiE3xV/obGc=` — use it when adding the device
> to Home Assistant, or build from source to use your own.

### From source

See [Build & flash](#build--flash) below.

## Why this works

The LabelWriter 450 is a **standard USB Printer Class device** (class `0x07`,
`0922:0020`): the client renders the label into DYMO's ESC raster stream and sends
those bytes to TCP port 9100, and this component forwards them verbatim to the
printer's USB bulk OUT endpoint. It does not interpret the raster data.

The component never injects anything into the stream — in particular **no PJL**.
The LabelWriter has no PJL interpreter, so a page-count query would be treated as
label data and corrupt the output.

## Configuration

```yaml
dymo_printserver:
  id: prn
  port: 9100
  vid: 0x0922      # USB vendor id to match
  pid: 0x0020      # USB product id to match
  status_polling: true # USB Printer Class GET_PORT_STATUS poll
```

| Option | Default | Meaning |
|---|---|---|
| `port` | `9100` | TCP port the raw print server listens on |
| `vid` / `pid` | `0x0922` / `0x0020` | USB device to bind to |
| `status_polling` | `true` | Poll `GET_PORT_STATUS` for the IEEE-1284 status byte |

### Advertising the DYMO identity (mDNS)

The config adds an ESPHome `mdns: services:` block that publishes a
`_pdl-datastream._tcp` (Port 9100) service with the CUPS/Bonjour TXT records that
make the queue show up as a DYMO:

```yaml
mdns:
  services:
    - service: _pdl-datastream
      protocol: _tcp
      port: 9100
      txt:
        ty: "DYMO LabelWriter 450"
        usb_MFG: "DYMO"
        usb_MDL: "LabelWriter 450"
        product: "(DYMO LabelWriter 450)"
```

## Web label form (`/print`)

The device's ESPHome web server also serves a small label form at
**`http://<device>/print`**: a single text field — one line, drawn large (up to
20 characters) — and a **Print label** button. The field starts empty, and an
empty form is rejected with a hint instead of printing a blank label.

The firmware renders the text into a DYMO ESC/P raster label: the same
672 × 240 bitmap layout, bold **DejaVu Sans Bold** glyphs, and raster framing
(`ESC @`, `ESC D 84`, `SYN`-prefixed scan lines, `ESC E`) as a label sent by the
CUPS queue. The finished job is handed to the local raw-9100 pipeline, so it
takes exactly the same USB path as a network job. A line wider than the label is
scaled down to fit, and Latin-1 accented characters (e.g. German umlauts) are
supported.

The glyph tables are pre-rendered from DejaVuSans-Bold and committed as
`external_components/dymo_printserver/dymo_label_font.h`; regenerate them with:

```bash
python3 tools/gen_label_font.py
```

## Key implementation notes

- **4 KiB chunks, one transfer at a time (`MAX_INFLIGHT = 1`).** The ESP-IDF USB
  host controller rejects a second concurrent URB on the same non-periodic
  (bulk) endpoint with `ESP_ERR_INVALID_STATE`, which corrupts the stream.
- **`start_output_()` is serialized with a FreeRTOS mutex.** The TCP task and the
  USB completion callback both feed the pipe; without the lock they can both
  pass the in-flight check and submit out of order.
- **The reverse (bulk IN) channel is drained** so the printer can report status
  and keeps accepting data.
- **Adaptive interface selection.** The component walks the configuration
  descriptor, prefers a bidirectional printer-class interface with a bulk OUT
  endpoint, and issues `SET_INTERFACE` only when the interface actually offers
  more than one alternate setting. The LabelWriter has a single alternate
  setting, so it is used as-is.
- The printer's USB Printer Class `GET_PORT_STATUS` is polled every 15 s (when
  `status_polling` is enabled). The status byte drives the *Paper Out*, *Error*
  and *Online* binary sensors plus the *Raw Status* diagnostic sensor.

## Hardware

- **ESP32-S3-DevKitC-1 N16R8** (16 MB flash, 8 MB octal PSRAM) — or any
  ESP32-S3 board with a free USB-OTG port.
- **DYMO LabelWriter 450** (USB ID `0922:0020`).
- USB-OTG port in **host** mode connected to the printer.

> The ESP32-S3 has a single USB-OTG peripheral: it is a USB **host** to the
> printer. It cannot also act as a USB *device* at the same time.

## Repository layout

```
dymo-printserver.yaml                                ESPHome device config
secrets.yaml.example                                 copy to secrets.yaml
index.html                                           browser installer (ESP Web Tools)
manifest.json                                        installer manifest
dymo-firmware.factory.bin                            prebuilt merged image (no secrets)
external_components/dymo_printserver/__init__.py     component registration
external_components/dymo_printserver/dymo_printserver.h
external_components/dymo_printserver/dymo_printserver.cpp
external_components/dymo_printserver/dymo_label_font.h  generated 1-bit DejaVu Sans Bold glyphs
tools/gen_label_font.py                              regenerates dymo_label_font.h
```

## Build & flash

```bash
# 1. Fill in your secrets
cp secrets.yaml.example secrets.yaml
$EDITOR secrets.yaml

# 2. Compile
esphome compile dymo-printserver.yaml

# 3. First flash over USB (substitute your serial port)
esphome upload dymo-printserver.yaml --device /dev/ttyUSB0

# ... or over the air afterwards
esphome upload dymo-printserver.yaml --device dymo-labelwriter-450.local
```

Live logs:

```bash
esphome logs dymo-printserver.yaml --device dymo-labelwriter-450.local
```

## Client setup

Point the DYMO software (or CUPS) at the raw 9100 endpoint — the same connection
DYMO's own print server uses (**AppSocket / HP JetDirect**, port 9100):

```bash
# Linux (dymo-cups-drivers)
lpadmin -p DYMO450 -E -v socket://dymo-labelwriter-450.local:9100 \
        -m /usr/share/cups/model/lw450.ppd
```

On Windows use the DYMO driver on a **Standard TCP/IP Port** (RAW, port 9100);
on macOS the Bonjour entry *DYMO LabelWriter 450* should appear automatically.

## Home Assistant entities

The component exposes the printer's IEEE-1284 status byte (polled every 15 s) plus
a set of ESP32 diagnostics. ESPHome prefixes every entity with the device name
(`dymo_labelwriter_450_…`):

| Entity | Meaning |
|---|---|
| `binary_sensor.…_printer_ready` | Printer enumerated, claimed and usable |
| `binary_sensor.…_paper_out` | No labels / tray empty (status bit 5) |
| `binary_sensor.…_error` | Error / needs attention (status bit 3 clear) |
| `binary_sensor.…_online` | Selected / online (status bit 4) |
| `sensor.…_raw_status` | Raw IEEE-1284 status byte (diagnostic) |
| `sensor.…_uptime` / `…_wifi_signal` | Device diagnostics |
| `sensor.…_free_heap` / `…_max_free_block` / `…_heap_fragmentation` / `…_loop_time` | Device diagnostics |
| `sensor.…_firmware_version` | Running firmware version |
| `sensor.…_wifi_ssid` / `…_wifi_bssid` / `…_mac_address` / `…_ip_address` | Device network identity |

## Limitations

- The USB Printer Class status byte reflects the **engine/paper** state, not the
  formatter state. It can report **no labels**, but may **not** catch a jam; a jam
  typically shows up as the printer dropping off USB (*Printer Ready* goes off).
- mDNS advertising makes the DYMO **discoverable**; if your DYMO software only
  supports manual network setup, add it by hostname/IP on port 9100.

## Credits

- Inspired by [`mckinlk/esp32-zp450-print-server`](https://github.com/mckinlk/esp32-zp450-print-server)
  (4 KiB transfers, one at a time).
- Firmware and documentation built with **[OpenCode](https://opencode.ai)**,
  running in the [`sprisa/opencode`](https://hub.docker.com/r/sprisa/opencode)
  Docker image (source: [`sprisa/opencode-server`](https://github.com/sprisa/opencode-server)):

  ```sh
  docker pull sprisa/opencode:latest
  ```

- Built by **Big Pickle GO** 🥒
