# ESPHome für den Nous A1T

ESPHome-Firmware für die WLAN-Steckdose **Nous A1T** (ESP8285, BL0937-Leistungsmessung). Der Stecker wird **ohne Öffnen** umgestellt: Die fertige Firmware wird einfach über die Weboberfläche des vorinstallierten Tasmota hochgeladen. Kein Schraubendreher, kein USB-Seriell-Adapter, etwa 10 Minuten.

## Dateien

| Datei | Inhalt |
| --- | --- |
| `nous_a1t.bin.gz` | Fertig kompilierte, gzip-komprimierte Firmware zum direkten Hochladen über Tasmota |
| `nous_a1t_plug_V2.yaml` | ESPHome-Konfiguration, aus der die Firmware gebaut ist |

## Funktionen

- Relais schaltbar aus Home Assistant und über den Taster am Stecker
- Messung von Spannung, Strom, Leistung und Tagesenergie (kWh), kalibriert
- Überlastabschaltung ab 2500 W mit Benachrichtigung in Home Assistant
- Relaiszustand und Energiezähler überstehen einen Stromausfall
- Weboberfläche unter `http://<IP>/`
- Buttons für Neustart, Safe Mode und Werksreset

**LED**

| LED | Bedeutung |
| --- | --- |
| blinkt schnell | kein WLAN (Hotspot aktiv) |
| „atmet“ langsam | verbunden, Relais aus |
| dauerhaft an | verbunden, Relais an |

**Taster**

| Bedienung | Wirkung |
| --- | --- |
| kurz drücken | Relais umschalten |
| 10 s halten | LED blinkt 5× schnell, dann Werksreset |

## Hardware

| Funktion | GPIO |
| --- | --- |
| Relais | GPIO14 |
| Taster | GPIO0 |
| LED (invertiert, PWM) | GPIO13 |
| BL0937 SEL (invertiert) | GPIO12 |
| BL0937 CF | GPIO4 |
| BL0937 CF1 | GPIO5 |

Chip: ESP8285 mit 1 MB Flash.

## Installation

### Was du brauchst

- Den Nous A1T mit werkseitigem Tasmota, in der Steckdose, ohne Last
- Einen Browser im selben WLAN
- Die Datei `nous_a1t.bin.gz` aus diesem Repository (Datei anklicken → **Download raw file**)

> **Hinweis:** Ein Stromausfall während des Schreibens kann den Stecker unbrauchbar machen; dann hilft nur noch Öffnen und seriell Flashen. Das Zeitfenster ist kurz, den Stecker währenddessen trotzdem nicht ausstecken.

### 1. Tasmota aufrufen

Ist der Stecker neu, öffnet er einen Hotspot `tasmota-xxxx`. Damit verbinden, `192.168.4.1` aufrufen und dein WLAN eintragen. Danach die IP des Steckers (steht im Router) im Browser öffnen.

![Tasmota-Startseite](images/01_tasmota_start.png)

In der Fußzeile muss `tasmota` stehen (nicht `tasmota32`). Dann auf **Firmware Upgrade** klicken.

### 2. Firmware hochladen

Unter **Upgrade by file upload** die Datei `nous_a1t.bin.gz` wählen und **Start upgrade** klicken.

![Firmware-Upload in Tasmota](images/02_tasmota_upload.png)

Die Datei muss die Endung `.gz` behalten: Ungepackt passt die Firmware nicht neben das laufende Tasmota in den 1-MB-Flash.

![Upload Successful](images/03_upload_ok.png)

### 3. WLAN einrichten

Nach dem Neustart blinkt die LED schnell: ESPHome läuft, kennt aber noch kein WLAN.

1. Mit Handy oder PC den Hotspot **`NOUS-Smart-Wifi-Socket`** verbinden.
2. Das Captive Portal öffnet sich von selbst, sonst `http://192.168.4.1` aufrufen.
3. Dein WLAN wählen, Passwort eingeben, speichern.

Sobald der Stecker verbunden ist, wechselt die LED zum Atmen.

### 4. In Home Assistant einbinden

Home Assistant findet den Stecker automatisch als neues ESPHome-Gerät (**Einstellungen → Geräte & Dienste**). Auf **Konfigurieren** klicken, fertig. Der Name enthält die letzten Stellen der MAC-Adresse (`nous-a1t-xxxxxx`), so lassen sich mehrere Stecker auseinanderhalten.

## Selbst bauen

Wer die Konfiguration ändern will, z. B. WLAN-Daten fest eintragen oder `max_power` anpassen:

```bash
esphome compile nous_a1t_plug_V2.yaml
```

Die Firmware liegt danach unter `.esphome/build/nous-a1t/.pioenvs/nous-a1t/firmware.bin`.

- **Stecker läuft schon mit ESPHome:** Einfach per OTA aktualisieren, ohne gzip:
  `esphome run nous_a1t_plug_V2.yaml --device <IP>`
- **Erstinstallation über Tasmota:** Die `firmware.bin` vorher mit gzip packen.

  Linux/macOS:
  ```bash
  gzip -9 -k firmware.bin
  ```
  Windows mit 7-Zip: Rechtsklick auf `firmware.bin` → **7-Zip** → **Zu einem Archiv hinzufügen…** → **Archivformat: gzip** (Voreinstellung ist `7z`!) → **Kompressionsstärke: Ultra**. Oder in der Kommandozeile:
  ```
  "C:\Program Files\7-Zip\7z.exe" a -tgzip -mx=9 firmware.bin.gz firmware.bin
  ```
  Nur `.gz` funktioniert; `.zip`, `.7z` oder `.tar.gz` lehnt Tasmota ab.

## Kalibrierung

Die Werte in der YAML wurden an einem Exemplar mit Fluke 179 und Leistungsmesser ermittelt (231 V Netzspannung, Fön 1792 W):

```yaml
current_res: "0.000991"
voltage_div: "1551"
```

Andere Exemplare können um einige Prozent abweichen. Zum Nachkalibrieren eine ohmsche Last (Fön, Heizlüfter, Wasserkocher) und ein Messgerät verwenden, erst die Spannung, dann die Leistung korrigieren:

```
voltage_div_neu = voltage_div × U_gemessen / U_angezeigt
current_res_neu = current_res × P_angezeigt / P_gemessen × voltage_div_neu / voltage_div
```

> **Achtung:** Die A1T-Vorlage auf devices.esphome.io (`775` / `0.00280`) rechnet ohne `model:`, also mit dem HLW8012-Modell. Mit `model: BL0937` zeigen diese Werte nur etwa die halbe Spannung an.

## Fehlersuche

| Symptom | Ursache | Lösung |
| --- | --- | --- |
| „Not enough space“ oder „Upload buffer miscompare“ | Datei ungepackt, oder zu wenig freier Flash | Prüfen, ob es die `.bin.gz` ist; sonst Umweg über tasmota-minimal (siehe unten) |
| Tasmota (alte Version) lehnt die Firmware ab | Image-Prüfung in Tasmota 7.2+ | In der Tasmota-Konsole `SetOption78 1`, Neustart, erneut hochladen |
| LED blinkt dauerhaft schnell | Kein oder falsches WLAN | Mit Hotspot `NOUS-Smart-Wifi-Socket` verbinden, WLAN neu eintragen |
| `nous-a1t.local` nicht auffindbar | MAC-Suffix im Namen oder mDNS unzuverlässig | Mit der IP arbeiten (`--device <IP>`) |
| Spannung zeigt ca. 115 V | HLW8012-Kalibrierwerte mit `model: BL0937` | Werte aus diesem Repository verwenden |
| Spätere OTA-Updates scheitern am Platz | Eigene Konfiguration zu groß für 1 MB | `web_server` entfernen |

### Falls der Platz nicht reicht

Meldet Tasmota trotz `.gz`-Datei „Not enough space“, ist nichts kaputt, Tasmota läuft weiter. Dann zuerst [`tasmota-minimal.bin.gz`](http://ota.tasmota.com/tasmota/release/tasmota-minimal.bin.gz) auf demselben Weg hochladen. Tasmota startet danach mit einer Minimal-Oberfläche; dort über **Firmware Upgrade** die `nous_a1t.bin.gz` nachschieben.

![Tasmota minimal](images/04_tasmota_minimal.png)

Wenn ESPHome einmal läuft, nie wieder ein tasmota-minimal-Image darüberspielen; das kann den Stecker unbrauchbar machen.

### Notfall: Stecker startet nicht mehr

Dann führt kein Weg am Öffnen vorbei: seriell über UART (TX, RX, GND, 3,3 V, GPIO0 beim Einschalten auf GND) und `esptool`. Den USB-Seriell-Adapter **niemals** anschließen, solange der Stecker am Netz hängt.

## Hinweise

- Die Screenshots sind Nachbildungen der Tasmota-Oberfläche; Versionsnummern weichen ab.
- Die Konfiguration nutzt keinen API-Schlüssel und kein OTA-Passwort, damit die fertige Firmware für jeden Stecker passt. Wer das absichern will, ergänzt `api: encryption: key:` und `ota: password:` und baut selbst.
- Umbau auf eigene Gefahr. Arbeiten an Netzspannung nur durch Fachkundige.

## Quellen

- [Nous A1T – devices.esphome.io](https://devices.esphome.io/devices/nous-a1t/)
- [Migrating from Tasmota – ESPHome-Dokumentation](https://esphome.io/guides/migrate_sonoff_tasmota/)
- [Nous A1T Tasmota → ESPHome – Home Assistant Community](https://community.home-assistant.io/t/nous-a1t-tasmota-esphome/522530)
