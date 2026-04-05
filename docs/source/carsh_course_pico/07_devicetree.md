# Kapitel 7: Devicetree

**Zeitaufwand:** ca. 3 Stunden  
**Voraussetzung:** Kapitel 6 abgeschlossen  
**Ziel:** Du verstehst, was ein Devicetree ist, kannst Overlays schreiben und das Board-Target auf den Pico 2W umstellen.

---

## Warum Devicetree?

In Kapitel 6 hast du `GPIO_DT_SPEC_GET(DT_ALIAS(led0), gpios)` benutzt – aber woher weiß Zephyr, welcher Pin die LED ist? Die Antwort: aus dem **Devicetree**.

Der Devicetree ist eine **Beschreibung der Hardware** in einer strukturierten Textdatei. Statt Hardware-Details im C-Code zu hardcoden, stehen sie im Devicetree. Der Vorteil: Gleicher C-Code läuft auf verschiedener Hardware – nur der Devicetree ändert sich.

Dieses Konzept kommt aus der Linux-Kernel-Welt und ist eines der mächtigsten Features von Zephyr.

---

## 7.1 — Theorie: Devicetree verstehen

### Lies zuerst

- **Zephyr Doku: Devicetree Guide (Introduction)**  
  <https://docs.zephyrproject.org/latest/build/dts/index.html>  
  Lies die Abschnitte „Introduction" und „Syntax and structure" (ca. 20 Min).

- **Zephyr Doku: Devicetree HOWTOs**  
  <https://docs.zephyrproject.org/latest/build/dts/howtos.html>  
  Lies den Abschnitt „Set devicetree overlays" (ca. 10 Min).

### Die Grundstruktur

Ein Devicetree sieht so aus:

```dts
/ {
    aliases {
        led0 = &myled;
    };

    leds {
        compatible = "gpio-leds";
        myled: led_0 {
            gpios = <&gpio0 25 GPIO_ACTIVE_HIGH>;
            label = "Onboard LED";
        };
    };
};
```

**Erklärung:**
- `/` — Der Wurzelknoten (Root)
- `aliases` — Kurznamen für Nodes (damit dein Code `DT_ALIAS(led0)` verwenden kann)
- `compatible` — Sagt Zephyr, welchen Treiber es verwenden soll
- `gpios = <&gpio0 25 ...>` — Pin 25 am GPIO-Controller 0
- `myled:` — Ein **Label**, mit dem man den Node referenzieren kann

### Wichtige Begriffe

| Begriff | Bedeutung |
|---------|-----------|
| **Node** | Ein Eintrag im Devicetree (z.B. ein Sensor, ein Bus, eine LED) |
| **Property** | Eine Eigenschaft eines Nodes (z.B. `gpios`, `reg`, `status`) |
| **Label** | Ein Name zum Referenzieren (z.B. `&gpio0`) |
| **Alias** | Ein Kurzname (z.B. `led0 = &myled`) |
| **Overlay** | Eine Datei, die den bestehenden DT ergänzt oder ändert |
| **Binding** | Eine YAML-Datei, die beschreibt, welche Properties ein Node haben muss |
| **compatible** | Sagt Zephyr, welcher Treiber zum Node passt |

---

## 7.2 — Den Devicetree deines Boards untersuchen

### Aufgabe: Generierten Devicetree anschauen

Beim Bauen generiert Zephyr eine vollständige DT-Datei. Sieh sie dir an:

```bash
cat ~/zephyrproject/rpico-room-sensor/build/zephyr/zephyr.dts | less
```

(Navigiere mit Pfeiltasten, beende mit `q`)

Suche nach folgenden Dingen:
1. Wo ist `led0` definiert?
2. Wo ist `i2c0` definiert?
3. Wo ist `spi0` definiert?

> **Tipp:** Drücke `/` in `less`, tippe den Suchbegriff (z.B. `led0`) und drücke Enter.

### Aufgabe: Board-DTS-Datei finden

Die Basis-Devicetree-Datei deines Boards liegt im Zephyr-Quellcode:

```bash
find ~/zephyrproject/zephyr/boards -name "*.dts" | grep rpi_pico
```

Öffne die gefundene Datei und schau dir an, wie die Hardware des Pico W beschrieben ist.

---

## 7.3 — Overlays: Den Devicetree anpassen

Ein **Overlay** ergänzt oder überschreibt den bestehenden Devicetree, ohne die Originaldatei zu ändern. Das ist der Weg, wie du in deinem Projekt Hardware hinzufügst.

### Aufgabe: Ein einfaches Overlay erstellen

Erstelle eine Overlay-Datei für den Pico W:

```bash
nano ~/zephyrproject/rpico-room-sensor/boards/rpi_pico_rp2040_w.overlay
```

Inhalt:

```dts
/* Button auf GPIO 15 als Devicetree-Node definieren */
/ {
    buttons {
        compatible = "gpio-keys";
        user_button: button_0 {
            gpios = <&gpio0 15 (GPIO_PULL_UP | GPIO_ACTIVE_LOW)>;
            label = "User Button";
        };
    };

    aliases {
        sw0 = &user_button;
    };
};
```

Jetzt kannst du im C-Code den Button elegant über den Devicetree ansprechen:

```c
static const struct gpio_dt_spec button = GPIO_DT_SPEC_GET(DT_ALIAS(sw0), gpios);
```

Statt die Pin-Nummer im C-Code zu hardcoden! Baue das Projekt neu und prüfe, ob es kompiliert.

---

## 7.4 — DT-Macros im C-Code

Zephyr bietet Makros, um im C-Code auf Devicetree-Informationen zuzugreifen:

| Makro | Bedeutung | Beispiel |
|-------|-----------|---------|
| `DT_ALIAS(name)` | Node über Alias finden | `DT_ALIAS(led0)` |
| `DT_NODELABEL(label)` | Node über Label finden | `DT_NODELABEL(i2c0)` |
| `DT_PROP(node, prop)` | Property lesen | `DT_PROP(node, label)` |
| `GPIO_DT_SPEC_GET(node, prop)` | GPIO-Spezifikation holen | s.o. |
| `DEVICE_DT_GET(node)` | Geräte-Pointer holen | `DEVICE_DT_GET(DT_NODELABEL(i2c0))` |

### Lies zuerst

- **Zephyr Doku: Devicetree API**  
  <https://docs.zephyrproject.org/latest/build/dts/api/api.html>  
  Überfliege die Liste der verfügbaren Makros (ca. 10 Min). Du musst sie nicht alle kennen – nur wissen, dass sie existieren.

---

## 7.5 — Board-Target umstellen: Pico W → Pico 2W

Jetzt kommt eine wichtige Übung: Du stellst das Projekt auf den **Pico 2W (RP2350)** um, den du tatsächlich besitzt.

### Aufgabe: Für den Pico 2W bauen

1. Finde heraus, wie das Board-Target für den Pico 2W in Zephyr heißt:
   ```bash
   find ~/zephyrproject/zephyr/boards -type d | grep rpi_pico2
   ```

2. Baue das Projekt für den Pico 2W:
   ```bash
   west build -p auto -b rpi_pico2/rp2350a/m33/w \
     -d rpico-room-sensor/build \
     rpico-room-sensor \
     -S usb_serial_port
   ```

   > **Hinweis:** Falls der Build fehlschlägt, liegt es möglicherweise daran, dass die Zephyr-Version in deinem Manifest den Pico 2W noch nicht unterstützt. Melde das per Mail an [LEHRER-EMAIL].

3. Flashe die Firmware auf deinen Pico 2W und prüfe, ob die Shell funktioniert.

4. Vergleiche die generierten Devicetrees beider Boards:
   ```bash
   # Vorher (Pico W) hattest du den DT gesehen.
   # Jetzt schau dir den DT des Pico 2W an:
   cat rpico-room-sensor/build/zephyr/zephyr.dts | less
   ```

### Fragen zum Nachdenken

- Welche Unterschiede siehst du im Devicetree zwischen Pico W und Pico 2W?
- Musste der C-Code geändert werden, um auf dem anderen Board zu laufen?
- Was musste angepasst werden (Board-Overlay-Dateiname)?

> **Das ist die Stärke von Zephyr + Devicetree:** Gleicher Code, anderes Board – nur die Konfiguration ändert sich.

---

## Selbsttest – Kapitel 7

- [ ] Kann ich erklären, was ein Devicetree ist und warum er verwendet wird?
- [ ] Kenne ich die Begriffe Node, Property, Label, Alias, Overlay?
- [ ] Kann ich den generierten Devicetree (`zephyr.dts`) lesen?
- [ ] Habe ich ein Overlay erstellt (z.B. für den Button)?
- [ ] Kenne ich die wichtigsten DT-Macros (`DT_ALIAS`, `DT_NODELABEL`, `GPIO_DT_SPEC_GET`)?
- [ ] Habe ich das Projekt erfolgreich für den Pico 2W gebaut?

> **Alle Haken gesetzt?** Weiter zu Kapitel 8: I2C & BME280!  
> **Probleme?** Schreib eine Mail an [LEHRER-EMAIL].
