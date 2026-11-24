# AGENTS.md — DYMO LabelWriter 450 ESP32-S3 Print Server

Diese Datei richtet sich an KI-Agents, die in diesem Repository arbeiten. Sie
beschreibt die Installation und — wichtiger — die Grenzen, die vor jeder Aktion
gelten.

## 0. Regeln, die immer gelten (zuerst lesen)

1. **WLAN, Access Point und physisches Gerät nie ohne ausdrückliche, aktuelle
   Anweisung des Benutzers anfassen.** Keine Scans, kein Flashen, kein OTA, kein
   Speichern von WLAN-Daten, kein "Ich helfe mal schnell". Vor **jeder** Aktion
   neu fragen.
2. Ein früheres "Ja" ist **keine** Dauererlaubnis. Nach einem "Nein" oder bei
   Themenwechsel gilt die Regel unverändert weiter.
3. **Keine Secrets committen.** `secrets.yaml`, echte WLAN-Zugangsdaten,
   API-Keys und gerätespezifische Firmware-Images gehören nie ins Repo. Die
   vorgefertigte Binärdatei bleibt bei Platzhalter-WLAN.
4. **Kein PJL in dieses Projekt.** Die LabelWriter 450 hat keinen PJL-Interpreter;
   eine PJL-Anfrage würde als Label-Daten interpretiert und das Etikett
   zerstören. PJL gehört ausschließlich zum HP1010-Repo.
5. **DYMO und HP1010 strikt getrennt.** Eigene Repos, eigene Komponenten,
   eigene Firmware. Keine HP-Artefakte hierher kopieren und umgekehrt.
6. Nicht ankündigen, was nicht bestellt wurde. Fakten nennen, keine
   Rechtfertigungen erfinden, nichts als "schon erledigt" hinstellen.
7. Bei Unklarheit fragen — nicht raten, nicht ausprobieren.

## 1. Was das Repository ist

Ein ESP32-S3 fungiert als **USB-Host-Brücke** für eine USB-only DYMO
LabelWriter 450. Der Client rendert das Etikett in DYMOs ESC-Raster-Stream und
sendet die Bytes an TCP 9100; der ESP leitet sie 1:1 an den USB-Bulk-OUT-Endpunkt
weiter. Es wird **nichts** interpretiert und **nichts** eingefügt.

```
DYMO-Software / CUPS --TCP 9100--> ESP32-S3 --USB bulk OUT--> LabelWriter 450
```

- ESPHome-Device-Name: `dymo-labelwriter-450`
- mDNS: `dymo-labelwriter-450.local`, Port `9100`, Service `_pdl-datastream._tcp`
  mit TXT `ty: "DYMO LabelWriter 450"`
- Zielboard: **ESP32-S3-DevKitC-1 N16R8** (16 MB Flash, 8 MB Octal PSRAM),
  USB-OTG im Host-Modus
- Drucker-USB-ID: `0922:0020`

## 2. Voraussetzungen

- ESP32-S3 mit freiem USB-OTG-Port (Host-Modus) und USB-Kabel zum Drucker
- DYMO LabelWriter 450
- Für Pfad A: Browser mit WebSerial (Chrome/Edge) oder `esptool.py`
- Für Pfad B: `esphome` (empfohlen: aktuelle Version)

## 3. Installation — Pfad A: vorgefertigtes Image (kein Build)

Die Datei `dymo-firmware.factory.bin` liegt im Repo-Root.

**Browser-Installer:**

```
https://cosmicboy6.github.io/esp32-s3-dymo450-printserver/
```

ESP Web Tools flasht das Board direkt per WebSerial; es wird nichts hochgeladen.
Spiegel: `https://raw.githack.com/cosmicboy6/esp32-s3-dymo450-printserver/main/index.html`

**Oder per Kommandozeile** (gemergtes Image ab Offset `0x0`):

```bash
esptool.py --chip esp32s3 write_flash 0x0 dymo-firmware.factory.bin
```

Das Image enthält **keine echten Zugangsdaten**. Es bootet in den Fallback-AP
(siehe Abschnitt 5). Eingebackener API-Key:
`9GXBCDmqW1uiTOdRJC60k8qvZ9W/yYW0jiE3xV/obGc=`.

## 4. Installation — Pfad B: aus dem Quellcode

```bash
cp secrets.yaml.example secrets.yaml
$EDITOR secrets.yaml          # SSID, Passwort, AP-Name/Passwort, API-Key
esphome compile dymo-printserver.yaml

# Erstes Flashen über USB:
esphome upload dymo-printserver.yaml --device /dev/ttyUSB0

# Danach per OTA:
esphome upload dymo-printserver.yaml --device dymo-labelwriter-450.local
```

API-Key erzeugen mit `openssl rand -base64 32`. `secrets.yaml` ist gitignored.

**Feste IP:** Nur lokal im `wifi:`-Block ergänzen:

```yaml
wifi:
  manual_ip:
    static_ip: 192.168.0.240
    gateway: 192.168.0.1
    subnet: 255.255.255.0
    dns1: 192.168.0.1
```

Weder echte WLAN-Daten noch `manual_ip` in das Repo committen. Ein
gerätespezifisches Image (mit echten Daten) gehört nach `.tmp/`, nie in git.

**Wichtig:** Nach dem Flashen des generischen Repo-Images fallen `manual_ip` und
die echten WLAN-Daten weg. Für ein bereits auf statischer IP laufendes Gerät
nicht einfach `esphome upload` aus dem Repo ausführen.

Logs:

```bash
esphome logs dymo-printserver.yaml --device dymo-labelwriter-450.local
```

## 5. WLAN-Einrichtung über den Fallback-AP

Wenn das Gerät das konfigurierte WLAN nicht erreicht, öffnet es seinen eigenen
AP:

- SSID: `DYMO Print Fallback`
- Passwort: `dymosetup` (nur im vorgefertigten Image; bei Source-Builds der Wert
  aus `secrets.yaml`)

Vorgehen:

1. Mit dem AP verbinden.
2. `http://192.168.4.1` öffnen und SSID/Passwort des echten WLANs eintragen.
3. **Nach dem Speichern verlässt das Gerät den AP und geht ins WLAN — der AP
   verschwindet. Das ist der Erfolgsfall, kein Fehler.**

Known pitfall: Bei ESPHome `web_server` v3 belegt die Weboberfläche den Pfad
`/`. Bleibt die Seite leer, das Portal unter **`/portal`** öffnen; die
Konfigurations-JSON liegt unter **`/config.json`**.

## 6. Installation verifizieren (nur lesen)

- Device erreichbar: `dymo-labelwriter-450.local`, Port `9100`.
- Home Assistant: `binary_sensor.dymo_labelwriter_450_printer_ready` = **on**,
  `sensor.dymo_labelwriter_450_ip_address` zeigt die IP.
- ESPHome-Logs: USB-Gerät `0922:0020` erkannt; **kein** `SET_INTERFACE` nötig
  (die LabelWriter hat nur einen Alternate Setting).
- `http://<device>/print` zeigt das Ein-Zeilen-Formular.
- Testdruck über die CUPS-Queue; danach Druckerstatus `idle`.

**Für Agents:** Auch diese Prüfungen nur ausführen, wenn der Benutzer sie
ausdrücklich verlangt. Es handelt sich um Netzwerkzugriff auf das Gerät.

## 7. Web-Label-Formular (`/print`)

Die Firmware liefert unter `http://<device>/print` ein kleines Formular:
**eine** Textzeile, groß gezeichnet (maximal 20 Zeichen), und ein
**Print label**-Button. Das Feld startet leer; eine leere Eingabe wird mit einem
Hinweis abgewiesen statt ein leeres Etikett zu drucken.

Gerendert wird eine DYMO-ESC-Rasterseite (672 × 240 Punkte, fetter
DejaVu-Sans-Bold-Schriftsatz, `ESC @`, `ESC D 84`, `SYN`-Rasterzeilen, `ESC E`).
Der Job läuft durch dieselbe lokale 9100-Pipeline wie ein Netzwerkjob. Die
Glyphtabelle ist committed; neu erzeugen mit:

```bash
python3 tools/gen_label_font.py
```

## 8. Client-Setup (CUPS)

Roh-Queue auf Port 9100 (AppSocket / JetDirect):

```bash
lpadmin -p DYMO450 -E -v socket://dymo-labelwriter-450.local:9100 \
        -m /usr/share/cups/model/lw450.ppd
```

Bei fester IP lieber explizit die Adresse angeben:

```bash
lpadmin -p DYMO450 -E -v socket://192.168.0.240:9100 -m /usr/share/cups/model/lw450.ppd
```

**Hinweis:** Eine veraltete `dnssd://…`-Device-URI passt oft nicht zur
tatsächlichen mDNS-Instanz (`dymo-labelwriter-450`) — dann bleibt die Queue
stumm, obwohl das Gerät erreichbar ist. Im Zweifel `socket://<IP>:9100`
verwenden.

Windows: DYMO-Treiber auf einem **Standard TCP/IP Port** (RAW, 9100).
macOS: Bonjour-Eintrag *DYMO LabelWriter 450* sollte automatisch erscheinen.

## 9. Home Assistant

ESP über die ESPHome-Integration mit dem Key aus Abschnitt 3 hinzufügen.
Entitäten tragen das Präfix `dymo_labelwriter_450_`, u. a. `printer_ready`,
`paper_out`, `error`, `online`, `raw_status`, `uptime`, `wifi_signal`,
`firmware_version`, `ip_address`.

## 10. Repository-Konventionen

- Öffentlich unter `cosmicboy6/esp32-s3-dymo450-printserver`.
- GitHub Pages aus dem Repo-Root (`.nojekyll` vorhanden).
- `dymo-firmware.factory.bin` bleibt generisch (Platzhalter-WLAN). Gerätebilder
  mit echten Zugangsdaten werden **nie** committet.
- Keine PJL-Logik, keine HP1010-Artefakte.
- Credits: OpenCode, Docker-Image `sprisa/opencode:latest`, Big Pickle GO 🥒.

## 11. Häufige Stolpersteine

| Symptom | Ursache / Prüfung |
|---|---|
| Etikett ist mit Klartext/Kommandos vollgemüllt | PJL oder fremde Steuerung im Stream — strikt verboten |
| Job bricht mit `ESP_ERR_INVALID_STATE` ab | Zweiter paralleler USB-Transfer; nur 4 KiB-Chunks, `MAX_INFLIGHT = 1` |
| Drucker wird nicht gefunden | mDNS-Instanz heißt `dymo-labelwriter-450`; CUPS lieber `socket://<IP>:9100` |
| Fallback-AP sichtbar, kein WLAN | Falsche Zugangsdaten in `secrets.yaml` oder im Portal |
| `/` im Portal leer | `web_server` v3 belegt `/` → `/portal` verwenden |
| Nach OTA keine statische IP mehr | Geflashtes generisches Repo-Image entfernt `manual_ip`; lokales Geräte-Image nutzen |
| CUPS: "Drucker existiert nicht oder ist zurzeit nicht verfügbar" | Gerät aus / nicht erreichbar; Strom und Port 9100 prüfen — **nur auf Ansage** |
| Wartungshinweis | `esphome` → `platformio_options` → `build_flags` ist veraltet (Entfernung in 2026.12.0); bei einem Build ggf. umstellen |
