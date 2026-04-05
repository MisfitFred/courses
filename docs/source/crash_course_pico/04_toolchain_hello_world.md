# Kapitel 4: Toolchain & Hello World auf dem Pico

**Zeitaufwand:** ca. 2 Stunden  
**Voraussetzung:** Kapitel 1–3 abgeschlossen  
**Ziel:** Du hast die Zephyr-Entwicklungsumgebung eingerichtet, das Kursprojekt gebaut, auf den Pico W geflasht und die USB-Shell gesehen.

---

## Der große Moment

Bisher hast du Linux, Git und C gelernt – alles auf deinem PC. Jetzt wird es ernst: Du baust ein echtes Embedded-Programm und bringst es auf einen Mikrocontroller zum Laufen. Wenn am Ende dieses Kapitels die USB-Shell auf deinem Pico antwortet, hast du etwas geschafft, was viele Informatik-Studenten erst im 3. Semester erleben.

---

## 4.1 — Was ist Zephyr?

### Lies zuerst

- **Zephyr Introduction (offizielle Doku)**  
  <https://docs.zephyrproject.org/latest/introduction/index.html>  
  Lies nur die erste Seite (ca. 10 Min). Du musst nicht alles verstehen – es geht darum, ein Gefühl dafür zu bekommen, was Zephyr ist.

### Kurzfassung

**Zephyr** ist ein Echtzeit-Betriebssystem (RTOS) für Mikrocontroller. Es ist:
- **Winzig:** Läuft auf Geräten mit nur 8 KB RAM
- **Portabel:** Gleicher Code läuft auf verschiedenen Chips (ARM, RISC-V, x86...)
- **Professionell:** Wird von der Linux Foundation betreut, eingesetzt von Google, Intel, Nordic und vielen anderen
- **Open Source:** Komplett kostenlos und quelloffen

Zephyr bringt Dinge mit, die man von einem „großen" Betriebssystem kennt: Threads, Scheduling, Treiber, Shell, Logging – aber alles extrem schlank und für Embedded optimiert.

### Wichtige Begriffe

| Begriff | Bedeutung |
|---------|-----------|
| **west** | Zephyrs Meta-Tool – verwaltet Repos, baut Projekte, flasht Firmware |
| **Manifest** | Eine YAML-Datei, die beschreibt, welche Repos und Versionen ein Projekt braucht |
| **Board** | Die Zephyr-Konfiguration für ein bestimmtes Hardware-Board (z.B. `rpi_pico/rp2040/w`) |
| **prj.conf** | Die Projektkonfiguration – hier schaltet man Zephyr-Features an/aus |
| **Overlay** | Eine Datei, die den Devicetree für ein bestimmtes Board anpasst |
| **Snippet** | Ein vorgefertigtes Konfigurationspaket (z.B. USB-Serial aktivieren) |

---

## 4.2 — Systemabhängigkeiten installieren

Öffne dein Ubuntu-Terminal (WSL2) und installiere alles, was Zephyr braucht:

```bash
sudo apt update
sudo apt install -y git cmake ninja-build gperf ccache dfu-util \
  device-tree-compiler wget python3-dev python3-pip python3-venv \
  python3-setuptools python3-tk python3-wheel xz-utils file \
  make gcc gcc-multilib g++-multilib libsdl2-dev libmagic1 \
  libhidapi-hidraw0 clang-format
```

Das sind die Compiler, Build-Tools und Bibliotheken, die Zephyr zum Bauen braucht. `libhidapi-hidraw0` ist speziell für den Debug Probe nötig.

---

## 4.3 — Zephyr-Workspace einrichten

Jetzt richtest du den Zephyr-Workspace ein. Unser Kursprojekt ist das sogenannte **West-Manifest-Repo** – es sagt West, welche Version von Zephyr und welche Module heruntergeladen werden sollen.

### Schritt für Schritt

1. **Projektordner und Python-Umgebung erstellen:**
   ```bash
   mkdir -p ~/zephyrproject
   cd ~/zephyrproject
   python3 -m venv .venv
   source .venv/bin/activate
   ```
   Dein Prompt zeigt jetzt `(.venv)` am Anfang – das bedeutet, die Python-Umgebung ist aktiv.

2. **West installieren:**
   ```bash
   pip install west
   ```

3. **Workspace mit dem Kursprojekt initialisieren:**
   ```bash
   west init -m https://github.com/MisfitFred/rpico-room-sensor ~/zephyrproject
   ```
   Das sagt West: „Verwende dieses Repo als Manifest – es bestimmt, was heruntergeladen wird."

4. **Alle Abhängigkeiten herunterladen:**
   ```bash
   cd ~/zephyrproject
   west update
   ```
   ⚠️ **Das dauert einige Minuten!** West lädt den gesamten Zephyr-Quellcode und alle Module herunter (mehrere GB).

5. **Zephyr exportieren und Python-Abhängigkeiten installieren:**
   ```bash
   west zephyr-export
   pip install -r zephyr/scripts/requirements.txt
   ```

6. **Zephyr SDK installieren:**
   ```bash
   west sdk install
   ```
   Das SDK enthält die Cross-Compiler (die erzeugen ARM-Code auf deinem x86-PC).

### Die Verzeichnisstruktur verstehen

Nach der Installation sieht dein Workspace so aus:

```
~/zephyrproject/
├── .venv/                          ← Python-Umgebung
├── .west/                          ← West-Konfiguration
├── zephyr/                         ← Zephyr-Quellcode (RIESIG)
├── modules/                        ← Externe Module (HALs etc.)
└── rpico-room-sensor/              ← UNSER Projekt (das Manifest-Repo)
    ├── CMakeLists.txt
    ├── prj.conf
    ├── west.yml
    ├── components/
    │   ├── main/
    │   ├── component_a/
    │   └── component_b/
    └── ...
```

**Wichtig:** `zephyr/` enthält das gesamte Zephyr-RTOS. `rpico-room-sensor/` ist das Projekt, an dem wir arbeiten.

---

## 4.4 — Das Projekt bauen

### Wichtig: Virtuelle Umgebung aktivieren!

Jedes Mal, wenn du ein neues Terminal öffnest, musst du die Python-Umgebung erst aktivieren:

```bash
cd ~/zephyrproject
source .venv/bin/activate
```

### Build starten

```bash
west build -p auto -b rpi_pico/rp2040/w \
  -d rpico-room-sensor/build \
  rpico-room-sensor \
  -S usb_serial_port
```

Was bedeutet das?
- `-p auto` = „Räume alte Build-Dateien auf, wenn nötig" (pristine)
- `-b rpi_pico/rp2040/w` = „Baue für den Raspberry Pi Pico W"
- `-d rpico-room-sensor/build` = „Speichere Build-Ergebnisse in diesem Ordner"
- `rpico-room-sensor` = „Das ist das Projekt, das gebaut werden soll"
- `-S usb_serial_port` = „Aktiviere USB-Serial (damit wir die Shell über USB nutzen können)"

Das Ergebnis liegt in `rpico-room-sensor/build/zephyr/`:
- `zephyr.uf2` — Firmware zum manuellen Kopieren
- `zephyr.elf` — Firmware zum Flashen über Debug Probe

> **Fehler beim Bauen?** Lies die Fehlermeldung genau. Häufige Ursache: Virtuelle Umgebung nicht aktiviert. Prüfe, ob `(.venv)` in deinem Prompt steht.

---

## 4.5 — Firmware flashen (UF2-Methode)

Zum ersten Mal verwenden wir die einfachste Methode – ohne Debug Probe.

### Schritt für Schritt

1. **Trenne** den Pico W vom USB, falls er angeschlossen ist.
2. **Halte** die **BOOTSEL-Taste** auf dem Pico W gedrückt.
3. **Schließe** den Pico W **mit gedrückter Taste** per USB an den PC an.
4. **Lasse** die Taste los. Im Windows-Explorer erscheint ein neues Laufwerk (z.B. `RPI-RP2`).
5. **Kopiere** die Datei `zephyr.uf2` auf dieses Laufwerk.

Aus WSL2 heraus kannst du auf Windows-Laufwerke zugreifen:
```bash
cp rpico-room-sensor/build/zephyr/zephyr.uf2 /mnt/d/
```
(Passe den Laufwerksbuchstaben an – prüfe im Windows-Explorer, wo das Laufwerk gemountet ist.)

Dann im **Windows-Explorer** die Datei vom Laufwerk `D:` auf das `RPI-RP2`-Laufwerk kopieren.

Der Pico startet automatisch neu und führt dein Programm aus.

---

## 4.6 — Die USB-Shell testen

Nach dem Flashen erscheint der Pico als serieller Port. So verbindest du dich:

### In WSL2

1. Installiere einen seriellen Monitor:
   ```bash
   sudo apt install -y picocom
   ```

2. Finde den seriellen Port:
   ```bash
   ls /dev/ttyACM*
   ```
   Normalerweise ist es `/dev/ttyACM0`.

3. Verbinde dich:
   ```bash
   picocom /dev/ttyACM0 -b 115200
   ```

4. Drücke **Enter** – du solltest den Zephyr-Shell-Prompt sehen:
   ```
   uart:~$
   ```

5. Probiere ein paar Shell-Befehle:
   ```
   uart:~$ help
   uart:~$ kernel version
   uart:~$ kernel uptime
   ```

6. Zum Beenden: **Strg+A**, dann **Strg+X**.

> **Kein `/dev/ttyACM0`?** In WSL2 sind USB-Geräte nicht automatisch verfügbar. Siehe nächsten Abschnitt für die usbipd-Methode, oder verwende alternativ den **Serial Monitor** direkt in Windows (z.B. PuTTY oder den Serial Monitor in VS Code).

### Alternative: Serial Monitor in Windows

1. Öffne den **Geräte-Manager** in Windows und finde unter „Anschlüsse (COM & LPT)" den neuen COM-Port (z.B. COM3).
2. Verwende **PuTTY** (<https://www.putty.org/>) oder den VS Code Serial Monitor:
   - Verbindungstyp: Serial
   - Port: COM3 (oder was bei dir steht)
   - Baud rate: 115200
3. Drücke Enter – die Zephyr-Shell erscheint.

---

## 4.7 — Debug Probe einrichten (für spätere Kapitel)

Der Debug Probe ermöglicht es, Firmware direkt zu flashen (ohne BOOTSEL-Taste) und Programme Schritt für Schritt zu debuggen. Das richten wir jetzt schon ein, auch wenn wir es erst ab Kapitel 5 richtig nutzen.

### Hardware-Verkabelung

Verbinde den Debug Probe mit dem Pico W:

| Debug Probe | Pico W |
|-------------|--------|
| GND | GND |
| GP2 (SWCLK) | SWCLK (Debug-Port) |
| GP3 (SWDIO) | SWDIO (Debug-Port) |

> **Tipp:** Der Pico W hat drei kleine Debug-Pads am Rand der Platine (beschriftet mit DEBUG). Der Debug Probe wird dort angeschlossen. Schau dir das Diagramm in der Raspberry Pi Dokumentation an: <https://www.raspberrypi.com/documentation/microcontrollers/debug-probe.html>

### USB-Weiterleitung mit usbipd (WSL2)

In WSL2 sind USB-Geräte nicht direkt verfügbar. Du musst sie per `usbipd` weiterleiten.

1. **Auf Windows:** Installiere usbipd-win: <https://github.com/dorssel/usbipd-win/releases>

2. **PowerShell als Admin** – Liste USB-Geräte auf:
   ```powershell
   usbipd list
   ```
   Suche den Eintrag mit `CMSIS-DAP` (das ist der Debug Probe).

3. **Binde und leite weiter** (BUSID anpassen!):
   ```powershell
   usbipd bind --busid 3-9
   usbipd attach --wsl --busid 3-9
   ```

4. **In WSL2** – Setze Berechtigungen:
   ```bash
   sudo chmod a+rw $(lsusb | grep "2e8a" | sed "s/Bus \([0-9]*\) Device \([0-9]*\).*/\/dev\/bus\/usb\/\1\/\2/")
   ```

5. **Teste den Flash-Vorgang über den Debug Probe:**
   ```bash
   cd ~/zephyrproject
   source .venv/bin/activate
   west flash -d rpico-room-sensor/build
   ```

Wenn alles funktioniert, wird die Firmware direkt geflasht – ohne BOOTSEL-Taste.

---

## Selbsttest – Kapitel 4

- [ ] Ist mein Zephyr-Workspace eingerichtet? (~/zephyrproject mit zephyr/, rpico-room-sensor/)
- [ ] Kann ich das Projekt fehlerfrei bauen? (`west build ...`)
- [ ] Habe ich die Firmware auf den Pico W geflasht (UF2 oder Debug Probe)?
- [ ] Sehe ich die Zephyr-Shell über USB? (`uart:~$`)
- [ ] Habe ich `help` und `kernel version` in der Shell ausprobiert?

> **Alle Haken gesetzt?** Glückwunsch – du hast ein RTOS auf einem Mikrocontroller zum Laufen gebracht! Weiter zu Kapitel 5: Threads & Scheduling.  
> **Probleme?** Schreib eine Mail an [LEHRER-EMAIL] mit der genauen Fehlermeldung und einem Screenshot.

---

## Zusammenfassung: Wichtige Befehle

```bash
# Immer zuerst:
cd ~/zephyrproject
source .venv/bin/activate

# Projekt bauen:
west build -p auto -b rpi_pico/rp2040/w \
  -d rpico-room-sensor/build \
  rpico-room-sensor \
  -S usb_serial_port

# Firmware flashen (Debug Probe):
west flash -d rpico-room-sensor/build

# Shell verbinden:
picocom /dev/ttyACM0 -b 115200
```
