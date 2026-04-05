# Kapitel 9: Abschlussprojekt

**Zeitaufwand:** ca. 3 Stunden  
**Voraussetzung:** Alle vorherigen Kapitel abgeschlossen  
**Ziel:** Du baust eine eigene Sensor-Komponente in der Projektstruktur, die den BME280 zyklisch ausliest und die Daten über einen eigenen Shell-Befehl ausgibt.

---

## Die Aufgabe

Du hast in den vorherigen Kapiteln alles Nötige gelernt. Jetzt wendest du es an, indem du eine **eigene Komponente** im Kursprojekt erstellst – so wie es in echten Embedded-Projekten gemacht wird.

Deine Komponente soll:
1. Den BME280-Sensor in einem eigenen Thread zyklisch auslesen
2. Die letzten Messwerte intern speichern
3. Einen eigenen Shell-Befehl `sensor read` bereitstellen, der die aktuellen Werte anzeigt

---

## 9.1 — Die Projektstruktur verstehen

Schau dir die bestehende Struktur des Kursprojekts an:

```bash
ls ~/zephyrproject/rpico-room-sensor/components/
```

Du siehst `main/`, `component_a/`, `component_b/`. Jede Komponente hat ihre eigene `CMakeLists.txt` und eigenen Code. Das Prinzip: **Jede Funktionalität in eine eigene Komponente.**

Schau dir an, wie eine bestehende Komponente aufgebaut ist:
```bash
ls ~/zephyrproject/rpico-room-sensor/components/component_a/
cat ~/zephyrproject/rpico-room-sensor/components/component_a/CMakeLists.txt
```

---

## 9.2 — Eigene Komponente anlegen

### Aufgabe: Komponente `sensor_bme280` erstellen

1. Erstelle die Verzeichnisstruktur:
   ```bash
   mkdir -p ~/zephyrproject/rpico-room-sensor/components/sensor_bme280/src
   ```

2. Erstelle die `CMakeLists.txt` der Komponente:
   ```bash
   nano ~/zephyrproject/rpico-room-sensor/components/sensor_bme280/CMakeLists.txt
   ```
   Inhalt:
   ```cmake
   target_sources(app PRIVATE
     src/sensor_bme280.c
   )

   target_include_directories(app PRIVATE
     src
   )
   ```

3. Registriere die Komponente im Hauptprojekt. Öffne die Projekt-CMakeLists.txt:
   ```bash
   nano ~/zephyrproject/rpico-room-sensor/CMakeLists.txt
   ```
   Füge am Ende hinzu:
   ```cmake
   add_subdirectory(components/sensor_bme280)
   ```

---

## 9.3 — Den Sensor-Code schreiben

### Aufgabe: Header-Datei erstellen

```bash
nano ~/zephyrproject/rpico-room-sensor/components/sensor_bme280/src/sensor_bme280.h
```

```c
#ifndef SENSOR_BME280_H
#define SENSOR_BME280_H

/**
 * @brief Letzte Messwerte abfragen
 *
 * @param temperature  Zeiger auf Temperatur-Variable (in °C * 100)
 * @param pressure     Zeiger auf Druck-Variable (in kPa * 100)
 * @param humidity     Zeiger auf Feuchte-Variable (in % * 100)
 * @return 0 bei Erfolg, negativer Fehlercode sonst
 */
int sensor_bme280_get_values(int32_t *temperature,
                             int32_t *pressure,
                             int32_t *humidity);

#endif /* SENSOR_BME280_H */
```

### Aufgabe: Implementierung schreiben

```bash
nano ~/zephyrproject/rpico-room-sensor/components/sensor_bme280/src/sensor_bme280.c
```

```c
#include <zephyr/kernel.h>
#include <zephyr/device.h>
#include <zephyr/drivers/sensor.h>
#include <zephyr/logging/log.h>
#include <zephyr/shell/shell.h>

#include "sensor_bme280.h"

LOG_MODULE_REGISTER(sensor_bme280, LOG_LEVEL_INF);

/* Interne Datenspeicherung */
static int32_t last_temp;       /* °C * 100 */
static int32_t last_press;      /* kPa * 100 */
static int32_t last_humidity;   /* % * 100 */

/* Thread-Konfiguration */
#define BME280_THREAD_STACKSIZE 1024
#define BME280_THREAD_PRIORITY  8
#define BME280_READ_INTERVAL_MS 2000

/* Sensor-Thread: liest den BME280 zyklisch aus */
static void bme280_thread_entry(void *p1, void *p2, void *p3)
{
    ARG_UNUSED(p1);
    ARG_UNUSED(p2);
    ARG_UNUSED(p3);

    const struct device *dev = DEVICE_DT_GET_ANY(bosch_bme280);
    struct sensor_value temp, press, humidity;

    if (dev == NULL || !device_is_ready(dev)) {
        LOG_ERR("BME280 nicht verfügbar!");
        return;
    }

    LOG_INF("BME280-Thread gestartet, lese alle %d ms", BME280_READ_INTERVAL_MS);

    while (1) {
        if (sensor_sample_fetch(dev) == 0) {
            sensor_channel_get(dev, SENSOR_CHAN_AMBIENT_TEMP, &temp);
            sensor_channel_get(dev, SENSOR_CHAN_PRESS, &press);
            sensor_channel_get(dev, SENSOR_CHAN_HUMIDITY, &humidity);

            /* Werte als Ganzzahl speichern (x100 für 2 Dezimalstellen) */
            last_temp = temp.val1 * 100 + temp.val2 / 10000;
            last_press = press.val1 * 100 + press.val2 / 10000;
            last_humidity = humidity.val1 * 100 + humidity.val2 / 10000;

            LOG_DBG("T=%d.%02d P=%d.%02d H=%d.%02d",
                    last_temp / 100, last_temp % 100,
                    last_press / 100, last_press % 100,
                    last_humidity / 100, last_humidity % 100);
        } else {
            LOG_WRN("BME280 Lesefehler");
        }

        k_sleep(K_MSEC(BME280_READ_INTERVAL_MS));
    }
}

/* Thread statisch definieren */
K_THREAD_DEFINE(bme280_thread, BME280_THREAD_STACKSIZE,
                bme280_thread_entry, NULL, NULL, NULL,
                BME280_THREAD_PRIORITY, 0, 0);

/* Öffentliche API: Werte abfragen */
int sensor_bme280_get_values(int32_t *temperature,
                             int32_t *pressure,
                             int32_t *humidity)
{
    if (temperature) {
        *temperature = last_temp;
    }
    if (pressure) {
        *pressure = last_press;
    }
    if (humidity) {
        *humidity = last_humidity;
    }
    return 0;
}

/* ---- Shell-Befehl ---- */

static int cmd_sensor_read(const struct shell *sh, size_t argc, char **argv)
{
    ARG_UNUSED(argc);
    ARG_UNUSED(argv);

    int32_t t, p, h;

    sensor_bme280_get_values(&t, &p, &h);

    shell_print(sh, "Temperatur:      %d.%02d °C", t / 100, t % 100);
    shell_print(sh, "Luftdruck:       %d.%02d kPa", p / 100, p % 100);
    shell_print(sh, "Luftfeuchtigkeit: %d.%02d %%", h / 100, h % 100);

    return 0;
}

SHELL_STATIC_SUBCMD_SET_CREATE(sensor_cmds,
    SHELL_CMD(read, NULL, "BME280-Sensorwerte anzeigen", cmd_sensor_read),
    SHELL_SUBCMD_SET_END
);

SHELL_CMD_REGISTER(sensor, &sensor_cmds, "Sensor-Befehle", NULL);
```

---

## 9.4 — main.c aufräumen

Da die Sensor-Logik jetzt in der eigenen Komponente ist, kann `main.c` einfach bleiben:

```bash
nano ~/zephyrproject/rpico-room-sensor/components/main/src/main.c
```

```c
#include <zephyr/kernel.h>
#include <zephyr/logging/log.h>

LOG_MODULE_REGISTER(main, LOG_LEVEL_INF);

int main(void)
{
    LOG_INF("=== RPico Room Sensor gestartet ===");
    LOG_INF("Verwende 'sensor read' in der Shell für Messwerte.");
    return 0;
}
```

---

## 9.5 — Bauen, Flashen, Testen

```bash
cd ~/zephyrproject
source .venv/bin/activate

west build -p auto -b rpi_pico/rp2040/w \
  -d rpico-room-sensor/build \
  rpico-room-sensor \
  -S usb_serial_port

west flash -d rpico-room-sensor/build
```

Verbinde dich per Serial-Monitor und teste deinen neuen Shell-Befehl:

```
uart:~$ sensor read
Temperatur:      22.34 °C
Luftdruck:       100.12 kPa
Luftfeuchtigkeit: 45.67 %
```

Prüfe auch, dass der Thread läuft:
```
uart:~$ kernel threads
```

---

## 9.6 — Committen und Pushen

Jetzt sicherst du deine Arbeit mit Git:

```bash
cd ~/zephyrproject/rpico-room-sensor

git add components/sensor_bme280/
git add CMakeLists.txt
git add components/main/src/main.c
git status   # Prüfe, was committet wird
git commit -m "Sensor-Komponente BME280 mit Shell-Befehl hinzugefügt"
```

Falls du einen Fork auf GitHub erstellt hast:
```bash
git push origin main
```

---

## 9.7 — Bonus-Aufgaben (für Schnelle)

### Bonus A: BMP280 einbinden
Du hast auch einen BMP280 (nur Temperatur + Druck, ohne Feuchtigkeit). Der Zephyr-Treiber `bosch,bme280` unterstützt auch den BMP280. Schließe ihn an und teste, ob er erkannt wird. Wie unterscheiden sich die Werte vom BME280?

### Bonus B: RFID-Leser (SPI)
Schließe das RFID-Kit über SPI an. Erstelle ein Devicetree-Overlay für den SPI-Bus. Recherchiere, ob Zephyr einen Treiber für den MFRC522 (den RFID-Chip auf dem Kit) hat.

> **Tipp:** Suche in der Zephyr-Doku und im Quellcode:
> ```bash
> find ~/zephyrproject/zephyr/drivers -name "*mfrc*" -o -name "*rc522*"
> ```

### Bonus C: Board-Wechsel auf Pico 2W
Falls du in Kapitel 7 den Pico 2W-Build noch nicht getestet hast: Stelle das gesamte Projekt auf den Pico 2W um und prüfe, ob alles funktioniert.

---

## Abschluss-Checkliste: Der gesamte Kurs

- [ ] **Kapitel 1:** WSL2 läuft, Linux-Terminal-Grundlagen beherrscht
- [ ] **Kapitel 2:** Git-Workflow verstanden (clone, add, commit, branch)
- [ ] **Kapitel 3:** C-Grundlagen: Pointer, Structs, Header-Dateien
- [ ] **Kapitel 4:** Zephyr-Umgebung eingerichtet, erstes Build geflasht
- [ ] **Kapitel 5:** Threads erstellt, Scheduling-Konzept verstanden
- [ ] **Kapitel 6:** GPIO angesteuert, Interrupt-Callback implementiert
- [ ] **Kapitel 7:** Devicetree gelesen und Overlay geschrieben
- [ ] **Kapitel 8:** BME280 per I2C angeschlossen und Daten ausgelesen
- [ ] **Kapitel 9:** Eigene Komponente mit Thread und Shell-Befehl gebaut

> **Alles geschafft?** Herzlichen Glückwunsch! Du hast in wenigen Wochen mehr über Embedded-Systeme gelernt als viele Informatik-Studenten im ersten Jahr. Du kannst jetzt C programmieren, weißt was ein RTOS ist, verstehst Devicetrees und kannst Hardware-Treiber integrieren. Das sind echte berufliche Fähigkeiten im Bereich Embedded Systems.

---

## Weiterführende Ressourcen

Wenn du tiefer einsteigen willst:

- **Zephyr Dokumentation (komplett)**  
  <https://docs.zephyrproject.org/latest/>

- **Zephyr RTOS Tutorial (maksimdrachov) — 10 Lektionen mit Übungen**  
  <https://github.com/maksimdrachov/zephyr-rtos-tutorial>

- **Nordic Developer Academy — Zephyr-Kurse (kostenlos)**  
  <https://academy.nordicsemi.com>

- **Hackster.io: Zephyr auf Pico 2**  
  <https://www.hackster.io/cdwilson/zephyr-rtos-on-raspberry-pi-pico-2-part-1-cf39f0>

- **DigiKey: Introduction to Zephyr (Video-Serie, 12 Teile)**  
  <https://www.digikey.com/en/maker/tutorials/2025/introduction-to-zephyr-part-1-getting-started-installation-and-blink>
