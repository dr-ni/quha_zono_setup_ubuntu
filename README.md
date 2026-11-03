# Quha Zono unter Linux

Der Quha Zono (Kopf-/Handmaus mit USB-Empfänger) wird unter Linux als normale
Maus erkannt. Die Einstellungen (Geschwindigkeit, Vibrationsdämpfung,
Doppelklick, Autoklick) und die Kalibrierung stecken aber in Windows-Software.
Dieses Repo bietet zwei Wege:

1. **`quha-settings`**: Kommandozeilenwerkzeug, das Einstellungen und
   Kalibrierung direkt über `/dev/hidraw*` setzt, ohne Wine.
2. **Original-Software unter Wine** (`Quha Zono Settings.exe`, `Qcal.exe`):
   optional, falls du die grafische Oberfläche brauchst.

Getestet mit Quha Zono Setup v1.51, USB-Empfänger `04d8:f8c4`
(Microchip Technology, Inc. Quha USB Receiver), Ubuntu.

## Inhalt

| Pfad | Zweck |
| --- | --- |
| `quha-settings` | CLI (Python 3, keine Abhängigkeiten) |
| `udev/70-quha.rules` | Gerätezugriff für den angemeldeten Benutzer |
| `udev/99-quha-apply.rules` | optional: Einstellungen beim Einstecken/Booten setzen |
| `systemd/quha-settings.service` | optional: der Dienst dazu |

## 1. Gerätezugriff (udev)

Der Empfänger meldet sich mit zwei HID-Interfaces. Die Einstellungen laufen
über `/dev/hidraw*`, das standardmäßig nur root gehört. Ohne diese Regel
zeigt die Windows-Software unter Wine "USB-Empfänger Status unbekannt".

```bash
sudo install -m 644 udev/70-quha.rules /etc/udev/rules.d/
sudo udevadm control --reload-rules
sudo udevadm trigger --subsystem-match=hidraw --action=add
```

Danach den Empfänger einmal abziehen und wieder einstecken. Ein `+` hinter den
Rechten (`ls -l /dev/hidraw0` zeigt `crw-rw----+`) heißt: `uaccess` greift.
Das funktioniert nur in einer lokalen Desktop-Sitzung. Per SSH stattdessen
`TAG+="uaccess"` durch `MODE="0666"` ersetzen.

Der Dateiname ist wichtig: Regeln mit `uaccess` müssen vor
`73-seat-late.rules` laufen, daher `70-` und nicht `99-`. Unter einem `99-`
Namen bleibt der Tag wirkungslos, und der Zugriff scheitert mit "Keine
Berechtigung auf /dev/hidraw*". Eine `MODE`-Regel ist davon nicht betroffen.

## 2. Einstellungen per CLI

```bash
sudo install -m 755 quha-settings /usr/local/bin/quha-settings   # optional

quha-settings get                    # aktuelle Werte lesen
quha-settings speed 40               # horizontal und vertikal
quha-settings speed 80 60            # horizontal 80, vertikal 60
quha-settings filter 3 2             # Vibrationsdämpfung 3, Stufe 2
quha-settings filter off             # off | 1 | 2 | 3 | joystick, Stufe 0..3
quha-settings doubleclick 500        # Doppelklick-Assistent in ms, oder off
quha-settings autoclick 100 1        # Verzögerung (Einheit 10 ms), Empfindlichkeit 0..2, oder off
```

Wichtig: Bei der Geschwindigkeit bedeutet ein **niedrigerer Wert schneller**
(21 = schnellste Stufe, 255 = langsamste). Die Regler der Windows-Software
sind entsprechend gespiegelt.

Mit `--dry-run` zeigt das Tool nur die Bytes, die es senden würde. Mit
`--device /dev/hidrawN` lässt sich ein bestimmtes Interface erzwingen. Sonst
probiert das Tool beide Interfaces des Empfängers durch und nimmt das, das
antwortet.

### Kalibrierung

```bash
quha-settings calibrate kopf-rechts --yes    # feste Anbringungsstelle laden
quha-settings calibrate free --yes           # freie Stelle: Messung im Empfänger
quha-settings get --calibration              # zusätzlich die Register 0x10..0x15
quha-settings reset --yes                    # nur Werksreset
```

Stellen: `kopf-rechts`, `kopf-links`, `hand-rechts`, `hand-links`,
`fuss-rechts`, `fuss-links`. Das entspricht den sechs Bildern im
Kalibrierungsassistenten.

**Achtung:** `calibrate` und `reset` setzen den Empfänger zuerst auf
Werkseinstellungen zurück, genau wie die Original-Software. Geschwindigkeit,
Vibrationsdämpfung und Doppelklick sind danach zurückgesetzt, und die
Kalibrierung wird überschrieben. Deshalb verlangen diese Befehle `--yes`. Die
eigenen Werte danach neu setzen (oder den systemd-Dienst unten laufen lassen).
Die Anbringungsstellen laden nur feste Registerwerte. Bei `free` misst der
Empfänger selbst: Nach der Meldung den Zeiger zuerst nach **rechts**, dann
nach **unten** bewegen. Pro Richtung bleiben bis zu 60 s Zeit, wie im Original.

### Einstellungen automatisch setzen

Falls die Werte beim Einstecken oder Booten wieder gelten sollen:

```bash
sudo install -m 755 quha-settings /usr/local/bin/quha-settings
sudo install -m 644 systemd/quha-settings.service /etc/systemd/system/
sudo install -m 644 udev/99-quha-apply.rules /etc/udev/rules.d/
sudo systemctl daemon-reload
sudo udevadm control --reload-rules
sudo udevadm trigger --subsystem-match=hidraw --action=add
```

Die gewünschten Werte stehen in den `ExecStart`-Zeilen des Dienstes. Der Dienst
läuft als root und startet einmal je HID-Interface, also zweimal
hintereinander, was harmlos ist. Das Ergebnis zeigt
`journalctl -u quha-settings -b`.

## 3. Original-Software unter Wine (optional)

```bash
sudo dpkg --add-architecture i386
sudo apt update
sudo apt install wine wine32 wine64

unzip Quha_Zono_Setup_v1.51.exe.zip -d quha_zono
cd quha_zono
wine Quha_Zono_Setup_v1.51.exe
```

`wine-gecko` und `wine-mono` gibt es oft nicht als apt-Paket. Die Settings-App
ist eine .NET-Anwendung und braucht Wine Mono. Meldet Wine
"Wine Mono is not installed", manuell nachinstallieren (aktuelle Version unter
<https://dl.winehq.org/wine/wine-mono/> prüfen):

```bash
cd /tmp
wget https://dl.winehq.org/wine/wine-mono/9.3.0/wine-mono-9.3.0-x86.msi
wine msiexec /i wine-mono-9.3.0-x86.msi
```

Die Programme liegen danach unter
`~/.wine/drive_c/Program Files (x86)/Quha/Zono/`:

```bash
# Kalibrierung / Einrichtung
cd ~/.wine/drive_c/"Program Files (x86)"/Quha/Zono/Initialization
wine Qcal.exe

# Einstellungen
wine "C:\Program Files (x86)\Quha\Zono\Settings\Quha Zono Settings.exe"
```

Zeigt die Statuszeile "USB-Empfänger angeschlossen", hat der Zugriff geklappt.
Die Meldungen `Unable to install a function driver for device WINEBUS\VID_845E…`
beim Installieren betreffen nicht den Quha-Empfänger (der hat `04d8:f8c4`) und
waren unkritisch.

Mit `MODE="0666"` auf den hidraw-Geräten liefen sowohl das CLI als auch die
Wine-Programme. `uaccess` (Abschnitt 1) ist die restriktivere Variante. Sie
funktioniert nur unter dem Dateinamen `70-quha.rules`, nicht unter `99-`.

### Screenshots

Kalibrierungsassistent (Auswahl der Anbringungsstelle):

![Anbringung wählen](images/quha_installation_anbringung.png)

Einstellungen, Geschwindigkeit:

![Geschwindigkeit](images/quha_settings_geschwindigkeit.png)

Einstellungen, Funktionen zur Steigerung der Nutzerfreundlichkeit:

![Funktionen](images/quha_settings_funktionen.png)

## 4. Protokoll

Das Protokoll wurde aus `Quha Zono Settings.exe` und `Qcal.exe` (v1.51) und der
darin verwendeten `usbGenericHidCommunications.dll` abgelesen. Die DLL ist nur
ein generischer HID-Wrapper (`WriteFile` / `ReadFile`).

Alle Reports haben 7 Bytes, Byte 0 ist die Report-ID 0. Unter Linux liefert
`hidraw` Antworten meist ohne diese ID (6 Bytes), `quha-settings` normalisiert
beides.

| Aktion | Senden (Byte 1 ff.) | Antwort |
| --- | --- | --- |
| Register schreiben | `F2 ADR LO HI` | Bestätigung: Byte 1 = `01` |
| Register lesen | `F0 ADR` | Bestätigung (`01`), danach Byte 1 = `F1`, Wert: Byte 3 = LO, Byte 4 = HI |
| Filter setzen | `32 FILTER STÄRKE` | Bestätigung: Byte 1 = `01` |
| Filter lesen | `30` | Bestätigung (`01`), danach Byte 1 = `31`, Byte 2 = FILTER, Byte 3 = STÄRKE |
| Register schreiben (32 Bit) | `F2 ADR B0 B1 B2 B3` | Bestätigung: Byte 1 = `01` (für Kalibrierwerte) |
| Werksreset | `FF 52 65 73 65 74` (`FF` + ASCII "Reset") | Byte 1 ungleich `00` |
| Messung starten | `20 CODE`, CODE `22` = rechts, `12` = unten | Bestätigung (`01`), danach Byte 1 = `21` nach erfolgreicher Bewegung (bis 60 s) |

Register (16-Bit-Wert, LO/HI):

| Adresse | Bedeutung | Bereich |
| --- | --- | --- |
| `0x30` | Geschwindigkeit horizontal | 21..255, niedriger = schneller |
| `0x31` | Geschwindigkeit vertikal | 21..255, niedriger = schneller |
| `0x40` | Doppelklick-Assistent, Verzögerung in ms | 1..2048, `0` = aus |
| `0x41` | Autoklick, Verzögerung in 10-ms-Einheiten | 1..500, `0` = aus |
| `0x42` | Autoklick, Empfindlichkeit | 0..2 |

Filter (Vibrationsdämpfung), Stufe 0..3:

| FILTER | Bedeutung | STÄRKE-Byte |
| --- | --- | --- |
| `0` | aus | `0` |
| `1` | Vibrationsdämpfung 1 | Stufe × 85 |
| `2` | Vibrationsdämpfung 2 | Stufe × 85 |
| `4` | Vibrationsdämpfung 3 | (Stufe + 1) × 20 |
| `8` | Joystick-Zustand | Stufe × 85 |

### Kalibrierung

Die Original-Software (`Qcal.exe`) setzt vor jeder Kalibrierung den Empfänger
zurück: Werksreset, Filter aus (`32 00 00`), Doppelklick aus (Register `0x40`
= 0). Danach lädt sie je nach Anbringungsstelle feste Registerwerte
(Finnisch im Code: *paa* = Kopf, *kasi* = Hand, *jalka* = Fuß, *oikea* = rechts,
*vasen* = links):

| Stelle | `0x10` | `0x11` | `0x12` | `0x13` | `0x14` | `0x15` | `0x30` | `0x31` |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| Kopf rechts | `0x807F` | 0 | 0 | 0 | 0 | `0x807F` | `0x90` | `0x60` |
| Kopf links | `0x807F` | 0 | 0 | 0 | 0 | `0x7F` | `0x90` | `0x60` |
| Hand rechts / links | `0x7F` | 0 | 0 | 0 | `0x7F` | 0 | `0xB0` | `0xB0` |
| Fuß rechts / links | 0 | 0 | `0x807F` | 0 | `0x807F` | 0 | `0x80` | `0x80` |

Hand und Fuß haben für links und rechts identische Werte. So steht es im
Original, es ist kein Tippfehler dieses Repos. `0x30` und `0x31` sind die
Geschwindigkeiten aus der Registertabelle oben.

Die "freie Stelle" setzt nach dem Reset keine Werte, sondern lässt den
Empfänger messen: erst `20 22` (Zeiger nach rechts bewegen), dann `20 12`
(nach unten). Die Berechnung passiert im Empfänger, die Software prüft nur
die Antwort `21`.

## Teststand

Auf echter Hardware geprüft: Lesen aller Werte, Schreiben der Geschwindigkeit
mit Zurücklesen sowie der Dienst-Lauf. Die Schreibzugriffe für Filter,
Doppelklick und Autoklick wurden vom Empfänger bestätigt, ihre Wirkung wurde
aber nicht einzeln durch Zurücklesen eines anderen Werts geprüft. Ob der
Empfänger Einstellungen beim Abziehen behält, ist nicht geklärt. Deshalb gibt
es den optionalen Dienst.

Die Kalibrierung (`calibrate`, `reset`, `get --calibration`) wurde nur gegen
einen simulierten Empfänger getestet, nicht an echter Hardware. Die Presets und
Kommandos stammen aus der Analyse von `Qcal.exe`. Weil der Werksreset
Einstellungen überschreibt, verlangt das Tool dafür `--yes`. Wie das Gerät auf
eine fehlgeschlagene Messung antwortet, ist unbekannt: Das Original prüft nur
auf `21`, das CLI zeigt bei jeder anderen Antwort die rohen Bytes an.
