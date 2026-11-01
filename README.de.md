# RTSP-Firmware für die GNCC GC3

**[English version](README.md)**

Mach aus deiner GNCC GC3 eine ganz normale Netzwerkkamera: RTSP-Stream, keine Cloud, keine App,
kein Konto, keine Verbindung ins Internet. Du brauchst nur eine microSD-Karte. Du musst die Kamera
nicht öffnen und nichts löten, und du brauchst keinen Linux-PC.

Paket auf eine Karte kopieren, WLAN eintragen, Karte in die Kamera, Strom an, fünf Minuten warten.

Das ist die GC3-Ausgabe von [gncc-gt1-pro-rtsp](https://github.com/LarzVader/gncc-gt1-pro-rtsp):
derselbe Installer, dieselbe Firmware-Grundlage. Was an der GC3 anders ist und wie das herausgefunden
wurde, steht unter [Was herausgefunden wurde](#was-herausgefunden-wurde).

## Was du bekommst

- **H.264 über RTSP**: `rtsp://<kamera>:8554/main` (1920×1080) und `/sub` (640×360), 15 Bilder pro
  Sekunde, für VLC, ffmpeg, Frigate, Home Assistant, Scrypted, Blue Iris und andere
- **ONVIF**-Geräteerkennung
- **Nachtsicht**: IR-LEDs und IR-Filter schalten von selbst um, das Nachtbild ist schwarz-weiß
- **Kein Weg nach draußen**: Die Kamera ignoriert Gateway und DNS-Server deines Routers und kommt
  deshalb nicht ins Internet, auch ohne Sperrregel im Router
- **Eigene Passwörter** auf jeder Kamera, zufällig erzeugt, wenn du keine vorgibst. Ein Standardpasswort
  gibt es nicht.
- **SSH-Zugang** als root, denn es ist deine Kamera
- **Mikrofon aus** in der Grundeinstellung
- **Eine vollständige Sicherung** des Flash auf deiner Karte, bevor sich irgendetwas ändert, und ein
  Weg zurück zur Originalfirmware

## Passt das zu deiner Kamera?

| | |
| --- | --- |
| Kamera | **GNCC GC3**, die Innenkamera mit festem Objektiv und USB-C-Stromanschluss. Auf der Platine steht `K301A0_V1.1`. |
| Getestet mit | Originalfirmware 7.3.77, an zwei Kameras |
| Karte | microSD, **FAT32**, mindestens 256 MB |
| WLAN | 2,4 GHz mit WPA2. WPA2/WPA3 gemischt geht, nur WPA3 nicht. |

Der Installer prüft die Kamera zuerst. Ist es ein anderes Modell oder eine Firmware, die er nicht kennt,
hält er an, sagt dir das und ändert nichts. Du kannst auch nur diese Prüfung laufen lassen: dafür in
`settings.txt` `CHECK_ONLY=yes` setzen.

## Installation

**1. Herunterladen.** Lade die Datei `gc3-rtsp-sdcard-*.zip` von den
[Releases](https://github.com/LarzVader/gncc-gc3-rtsp/releases) und entpacke sie direkt auf die leere
Karte, sodass die Ordner `wifitest` und `rtsp-setup` ganz oben auf der Karte liegen.

**2. WLAN eintragen** in der Datei `settings.txt` auf der Karte:

```
WIFI_SSID=Name deines Netzes
WIFI_PASSWORD=dein Passwort
```

**3. Installieren.** Kamera ausschalten. **Erst die Karte hinein, dann einschalten.** Die Originalfirmware
schaut nur einmal nach der Karte, ganz am Anfang: Eine Karte, die später hineinkommt, übersieht sie.
Dann die Kamera etwa 5 Minuten in Ruhe lassen:

| LEDs | Bedeutung |
| --- | --- |
| Rot blitzt kurz auf, etwa alle 4 Sekunden | Prüfung, Sicherung und Bauen, etwa 1 Minute |
| Rot blinkt schnell | Firmware wird geschrieben, etwa anderthalb Minuten. **Nicht ausstecken.** |
| Rot dauerhaft, Weiß blinkt | die neue Firmware läuft und verbindet sich mit deinem WLAN |
| **Rot dauerhaft, Weiß dauerhaft** | **fertig**, die Kamera ist in deinem Netz |
| Rot und Weiß blinken gemeinsam | angehalten. Warum, steht in `RESULT.txt` auf der Karte. |
| Nur Weiß dauerhaft, von Anfang an, sonst nichts | der Installer lief nicht, die Originalfirmware ist normal gestartet. Ausschalten und noch einmal, siehe [Fehlersuche](docs/troubleshooting.md). |

Mit `CHECK_ONLY=yes` sieht das Ende anders aus: Rot dauerhaft, Weiß 4 Sekunden an und 1 Sekunde aus.
Dann wurde nichts installiert.

Die rote LED der GC3 leuchtet ständig, sobald diese Firmware läuft. Das ist eine Eigenheit der Platine,
siehe [Was herausgefunden wurde](#was-herausgefunden-wurde).

**4. Ergebnis lesen.** Kamera ausschalten, Karte herausnehmen und in den Computer stecken:

- `CAMERA-ACCESS-<Seriennummer>.txt`: Stream-Adressen und Passwörter
- `camera-ip.txt`: die Adresse der Kamera. Du findest sie auch in der Geräteliste deines Routers unter dem
  Namen `gc3`.

**5. Kopiere den Ordner `backup-<Seriennummer>` auf deinen Computer.** Er ist dein Weg zurück zur
Originalfirmware. Er enthält private Daten deiner Kamera, etwa dein altes WLAN-Passwort und die
Cloud-Zertifikate: gib ihn nie weiter.

Die Karte kann ab jetzt draußen bleiben, die Kamera braucht sie nicht. Auf der Karte stehen dein
WLAN-Passwort und die Passwörter der Kamera. Bewahre sie also gut auf oder lösche `settings.txt` und
`CAMERA-ACCESS-*.txt`. Gelöschte Dateien lassen sich auf einer FAT-Karte oft wiederherstellen, also
formatiere die Karte, bevor sie das Haus verlässt.

Noch mehr Kameras? Nimm dieselbe Karte noch einmal, eine Kamera nach der anderen. Jede bekommt eigene
Passwörter. Im Router heißen sie alle `gc3`, unterscheide sie also an der Adresse.

## Den Stream ansehen

```bash
ffplay -rtsp_transport tcp "rtsp://camera:PASSWORT@192.168.1.50:8554/main"
```

In VLC: *Medien → Netzwerkstream öffnen*, mit derselben Adresse.

Unter `http://<kamera>/` gibt es außerdem eine kleine Webseite, mit demselben Benutzernamen und Passwort.

## Gut zu wissen

- **Gib der Kamera eine feste Adresse** im Router, damit ihre Stream-Adresse gleich bleibt.
- **Häng die Kamera normal herum auf.** Der Sensor der GC3 startet gedreht, und die Firmware stellt
  die Ausrichtung bei jedem Start der Aufnahme ein, danach jede Minute neu.
- **Nachts wird das Bild alle 10 Minuten für etwa 5 Sekunden dunkel.** Die Firmware nutzt keinen
  Lichtsensor und schaltet deshalb ab und zu ihr IR-Licht aus, um zu prüfen, ob es schon Tag ist.
  `ir_peek_s` in `ipcd.conf` stellt den Abstand in Sekunden ein.
- **RTSP läuft nur über TCP.** Eine Anfrage nach UDP bekommt `461 Unsupported Transport`, und ffmpeg, VLC und
  die anderen wechseln dann von selbst auf TCP. Über WLAN würde UDP nur Pakete verlieren. `rtsp_udp = 1` in
  `ipcd.conf` erlaubt UDP trotzdem.
- **Die Uhr geht falsch**, weil die Kamera nicht ins Internet kommt und die Firmware keinen NTP-Client
  hat. Auf das Video hat das keinen Einfluss.
- **Drück die Reset-Taste nicht im laufenden Betrieb.** Sie ist nur für die Rettung beim Einschalten gedacht.
- Es gibt kein Einrichtungsportal: Diese Firmware öffnet nie einen eigenen WLAN-Zugangspunkt. Dein WLAN änderst
  du mit einer Datei auf der Karte, siehe [Fehlersuche](docs/troubleshooting.md#the-white-led-keeps-blinking-the-camera-does-not-get-onto-the-wi-fi)
  (englisch).

## Wenn etwas schiefgeht

Lies [Troubleshooting and recovery](docs/troubleshooting.md) (englisch). Dort steht auch, wie du zur
Originalfirmware zurückkommst.

Der Installer schreibt nie auf U-Boot, den Teil der Kamera, der alles startet. U-Boot hat einen eigenen
Updater, der mit der Reset-Taste funktioniert, egal was sonst im Flash steht. **An der GC3 wurde dieser
Updater noch nie gebraucht und darum nie ausprobiert.** Getestet ist er an der GT1 Pro, deren U-Boot sich
bis auf 14 Bytes nicht vom U-Boot der GC3 unterscheidet, und die GC3 hat dieselben Befehle darin. Solange
ihn niemand gebraucht hat, gilt der Rückweg als ungetestet.

## Was herausgefunden wurde

Die Einzelheiten mit allen Zahlen stehen in [docs/gc3-findings.md](docs/gc3-findings.md) (englisch).
Kurz:

- Die GC3 ist **dieselbe Plattform wie GT1 Pro und GK2**: Anyka AK3918EV330, 8 MB Flash mit derselben
  Partitionsaufteilung, derselbe AltoBeam-WLAN-Treiber und ein Kernel, der Byte für Byte der aus der
  Originalfirmware der GT1 Pro ist.
- Der **Sensor ist ein SmartSens SC2331** mit 1920×1080 statt des 3-Megapixel-JX-Q03 der GT1 Pro. Die
  eingestellte Bildrate kommt auch an: 15 Bilder pro Sekunde.
- Die Originalfirmware hat denselben **SD-Karten-Einstieg** wie die der GT1 Pro, darum funktioniert der
  Installer ohne Öffnen der Kamera. Die Platine hat keine beschrifteten Lötpunkte für eine Konsole, und
  es brauchte auch keine.
- Der **Gerätebaum ist anders**: Der WLAN-Chip hängt an einem Schalt-GPIO, den die Firmware einschalten
  muss, der IR-Filter sitzt an anderen Pins mit umgekehrter Polarität, die LEDs an anderen Pins, und die
  rote LED leuchtet, wenn ihr GPIO „aus“ meldet. Die Firmware liest die Pins des IR-Filters jetzt aus dem
  Gerätebaum, das gilt damit für jede Platine dieser Familie.
- Der **Sensor startet um 180° gedreht**, in seinem eigenen Register; ein Registerbefehl stellt ihn aufrecht.
- Das Nachtbild ist **schwarz-weiß** über den ISP, genauso wie bei der GT1 Pro.

## Wieso das erlaubt ist

Eine Firmware für diese Kamera braucht Dateien, die nicht frei sind, etwa die Medienbibliotheken des
Chipherstellers und den Treiber für den Sensor. Die darf niemand veröffentlichen, ein fertiges Firmware-Abbild
kann es also nicht geben. Aber sie liegen schon auf deiner Kamera. Deshalb **enthält dieses Paket keine
Herstellerdateien**: Es enthält die freien Teile, und deine Kamera baut sich die Firmware selbst zusammen, aus
ihren eigenen Dateien. Das Ergebnis wird Bit für Bit mit der getesteten Firmware verglichen.

Wie das funktioniert und was sonst anders ist als bei der Originalfirmware: [How it works](docs/how-it-works.md).
Selbst bauen: [Building](docs/building.md).

## Risiken

Du ersetzt die Firmware eines Geräts, auf eigene Gefahr und nur bei einer Kamera, die dir gehört.

- **Du verlierst die Garantie** und alles, was App und Cloud gemacht haben: Fernzugriff über die App,
  Cloud-Aufnahmen, Bewegungsmeldungen, Gegensprechen.
- **Getestet an zwei Kameras**, beide mit Originalfirmware 7.3.77 und derselben Platine. Das ist nicht viel.
  Der Installer lief auf beiden mit Bauständen kurz vor 1.0, nicht mit den Abbildern dieses Release. Das `ipcd`
  dieses Release lief aus dem RAM auf der zweiten Kamera, und `docs/gc3-findings.md` sagt, was wo lief. Eine Kamera mit
  abweichenden Dateien lehnt der Installer ab, außer du erzwingst es.
- **Der Rückweg ist an der GC3 nicht ausprobiert.** `restore-stock.txt` hat die Originalfirmware einmal auf einer
  GT1 Pro geschrieben und geprüft, mit einem Backup ohne WLAN- und Cloud-Daten. Die Rettung per Reset-Taste wurde dort
  mit Firmware-Abbildern benutzt, nicht mit einem Backup. Heb den Backup-Ordner auf.
- **Ein Stromausfall, während Rot schnell blinkt**, lässt den Flash halb beschrieben zurück. Der Ausweg ist die
  Rettung per Reset-Taste, und die ist an der GC3 ungetestet, siehe oben. Im schlimmsten Fall braucht die
  Kamera einen Flash-Programmer.
- **Der Kernel ist ein Linux 4.4 von 2019** mit den Binärtreibern des Herstellers und bekommt nie wieder
  Sicherheitsupdates. Deshalb hat die Firmware keine Route ins Internet. Öffne von außen keinen Port zu ihr;
  wenn es sein muss, geh über ein VPN.
- **Keine Verschlüsselung im Netz**: RTSP und Webseite nutzen Digest-Anmeldung über reines TCP, nur SSH ist
  verschlüsselt. Wer in deinem Netz ist, kann den Videoverkehr mitlesen.
- **Wer an die Kamera kommt, hat sie.** Die Originalfirmware führt aus, was eine Karte namens `wifitest` ihr
  sagt, und auch diese Firmware übernimmt Einstellungen und Updates von einer Karte. Genau so kommt dieses
  Paket hinein, und so bleibt es.
- **Karte und Backup enthalten Geheimnisse**: dein WLAN-Passwort, die Passwörter der Kamera und im Backup das
  alte WLAN-Passwort und die Cloud-Zertifikate der Kamera.
- **Menschen aufzunehmen ist fast überall geregelt.** Das Mikrofon ist in der Grundeinstellung aus; was du wo
  und von wem aufnimmst, verantwortest du.

Hat es bei dir geklappt oder nicht? Sag bitte in einem [Issue](https://github.com/LarzVader/gncc-gc3-rtsp/issues)
Bescheid, mit deiner Firmware-Version und der `install.log` von der Karte. Sie enthält keine Passwörter, keine
Seriennummer und keinen WLAN-Namen. Häng `dmesg.txt` und `boot.log` nicht an: Darin steht die MAC-Adresse
deiner Kamera.

## Haftungsausschluss

Das ist ein privates Projekt. Es steht in keiner Verbindung zu GNCC, apemans, Osaio, Anyka, SmartSens,
AltoBeam oder irgendwem sonst, dessen Name hier auftaucht, und wird von niemandem davon unterstützt. Diese
Namen sind Marken ihrer Inhaber und werden nur gebraucht, um zu sagen, um welche Hardware es geht. Die
Software wird ohne jede Gewährleistung bereitgestellt, „wie sie ist“, siehe [LICENSE](LICENSE). Nichts hier
ist eine Beratung; prüf die Gesetze, die für dich gelten, bevor du irgendetwas aufnimmst.

## Danke

Dieses Projekt baut auf der Arbeit anderer auf:

- **[gncc-gt1-pro-rtsp](https://github.com/LarzVader/gncc-gt1-pro-rtsp)**: der Installer, der die Firmware
  auf der Kamera baut, und die ipcd-Patches. Dieses Repository ist seine GC3-Ausgabe.
- **[akost-firmware](https://github.com/michelostojski/akost-firmware)** von Michel Ostojski (MIT): die freie
  Firmware für diesen Chip. Das Schwierige wurde dort gemacht.
- **[ipcd](https://github.com/medevil84/ipcd)** von medevil (MIT): das Programm, das die eigentliche Kameraarbeit
  macht, mit RTSP, ONVIF und der Webseite.
- **[thingino](https://github.com/themactep/thingino-firmware)**: das Vorbild dafür, wie sich eine Kamera-Firmware
  selbst aktualisieren sollte.
- **[TECKIN-TC100 Anyka hacks](https://github.com/ThatUsernameAlreadyExist/TECKIN-TC100-Anyka-AK3918-camera-hacks)**:
  der SD-Karten-Einstieg, der ohne Werkzeug die Tür öffnet.
- Die GPL-Quellen im [Tuya BSP](https://github.com/tuya/tuya-bsp-gpl-public-components): Kernel und Werkzeugkette.
- Wesentliche Teile dieses Projekts, vom Installer über die Untersuchung der GC3 bis zu dieser Anleitung, sind
  zusammen mit einer KI entstanden.

## Lizenz

Die Arbeit in diesem Projekt steht unter der [MIT-Lizenz](LICENSE). Das Paket enthält außerdem Software anderer
unter deren eigenen Lizenzen. [THIRD-PARTY.md](THIRD-PARTY.md) listet sie auf und sagt, was im Quellpaket jedes Release liegt.
