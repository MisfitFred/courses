# Kapitel 6: GPIO & Interrupts

**Zeitaufwand:** ca. 3 Stunden  
**Voraussetzung:** Kapitel 5 abgeschlossen  
**Ziel:** Du kannst GPIOs (LEDs) ansteuern, Interrupts verstehen und einen Button-Callback implementieren.

---

## 6.1 — Was ist GPIO?

**GPIO** steht für **General Purpose Input/Output**. Das sind die Pins am Pico, die du entweder als **Ausgang** (z.B. LED einschalten) oder als **Eingang** (z.B. Tastendruck lesen) konfigurieren kannst.

### Lies zuerst

- **Zephyr Doku: GPIO**  
  <https://docs.zephyrproject.org/latest/hardware/peripherals/gpio.html>  
  Lies den Abschnitt „Overview" (ca. 10 Min).

- **Zephyr Blinky-Sample (Quellcode)**  
  <https://github.com/zephyrproject-rtos/zephyr/tree/main/samples/basic/blinky>  
  Schau dir die `src/main.c` an – das ist das einfachste Zephyr-Programm.

### Wichtige Zephyr-GPIO-Funktionen

| Funktion | Bedeutung |
|----------|-----------|
| `GPIO_DT_SPEC_GET()` | Holt die GPIO-Konfiguration aus dem Devicetree |
| `gpio_is_ready_dt()` | Prüft, ob der GPIO bereit ist |
| `gpio_pin_configure_dt()` | Konfiguriert den Pin (Eingang/Ausgang) |
| `gpio_pin_set_dt()` | Setzt den Pin auf HIGH (1) oder LOW (0) |
| `gpio_pin_toggle_dt()` | Wechselt den Pin-Zustand |
| `gpio_pin_get_dt()` | Liest den aktuellen Pin-Zustand |

---

## 6.2 — Praxis: LED blinken lassen

Der Pico W hat eine Onboard-LED (verbunden über den WiFi-Chip). Wir verwenden die Zephyr-Abstraktion `led0`, die im Devicetree definiert ist.

### Aufgabe: Blinky auf dem Pico

1. Ersetze den Inhalt von `components/main/src/main.c`:

   ```c
   #include <zephyr/kernel.h>
   #include <zephyr/drivers/gpio.h>
   #include <zephyr/logging/log.h>

   LOG_MODULE_REGISTER(main, LOG_LEVEL_INF);

   /* LED aus dem Devicetree holen (Alias "led0") */
   static const struct gpio_dt_spec led = GPIO_DT_SPEC_GET(DT_ALIAS(led0), gpios);

   int main(void)
   {
       int ret;

       if (!gpio_is_ready_dt(&led)) {
           LOG_ERR("LED GPIO ist nicht bereit!");
           return -1;
       }

       ret = gpio_pin_configure_dt(&led, GPIO_OUTPUT_ACTIVE);
       if (ret < 0) {
           LOG_ERR("Fehler beim Konfigurieren der LED: %d", ret);
           return -1;
       }

       LOG_INF("Blinky gestartet!");

       while (1) {
           gpio_pin_toggle_dt(&led);
           k_sleep(K_MSEC(500));
       }

       return 0;
   }
   ```

2. Baue und flashe:
   ```bash
   cd ~/zephyrproject
   source .venv/bin/activate
   west build -p auto -b rpi_pico/rp2040/w \
     -d rpico-room-sensor/build \
     rpico-room-sensor \
     -S usb_serial_port
   west flash -d rpico-room-sensor/build
   ```

3. Die LED auf dem Pico W sollte jetzt im Halbsekundentakt blinken.

### Was ist hier neu?

- `GPIO_DT_SPEC_GET(DT_ALIAS(led0), gpios)` — Holt die LED-Konfiguration aus dem **Devicetree**. Der Alias `led0` ist dort definiert. Du musst die Pin-Nummer nicht hardcoden!
- `gpio_pin_configure_dt(&led, GPIO_OUTPUT_ACTIVE)` — Konfiguriert den Pin als Ausgang
- `gpio_pin_toggle_dt(&led)` — Wechselt den LED-Zustand (an/aus)
- Die `_dt` am Ende steht für „Devicetree" – diese Varianten verwenden die DT-Konfiguration automatisch

---

## 6.3 — GPIO über die Shell

Dein Projekt hat `CONFIG_GPIO_SHELL=y` in der `prj.conf`. Das bedeutet, du kannst GPIOs direkt über die Shell steuern!

### Aufgabe: GPIO-Shell ausprobieren

Verbinde dich per Serial-Monitor und probiere:

```
uart:~$ gpio info gpio0
uart:~$ gpio conf gpio0 25 out
uart:~$ gpio set gpio0 25 1
uart:~$ gpio set gpio0 25 0
uart:~$ gpio toggle gpio0 25
```

Damit kannst du beliebige GPIO-Pins direkt steuern – sehr nützlich zum Testen.

---

## 6.4 — Interrupts verstehen

### Das Problem ohne Interrupts

Wenn du in einer Schleife prüfst, ob ein Button gedrückt ist (**Polling**), verschwendest du CPU-Zeit:

```c
/* SCHLECHT: Polling */
while (1) {
    if (gpio_pin_get_dt(&button) == 1) {
        /* Button gedrückt */
    }
    k_sleep(K_MSEC(10));  /* 100x pro Sekunde prüfen */
}
```

### Die Lösung: Interrupts

Ein **Interrupt** ist ein Hardware-Signal, das dem Prozessor sagt: „Hey, hier ist etwas passiert – kümmere dich darum!" Der Prozessor unterbricht den aktuellen Thread, führt eine **Interrupt Service Routine (ISR)** aus, und kehrt dann zum unterbrochenen Thread zurück.

Vorteile:
- Der Prozessor kann schlafen, bis etwas passiert → spart Strom
- Sofortige Reaktion, kein Warten bis zum nächsten Poll-Durchlauf

### Lies zuerst

- **Zephyr Doku: Interrupts**  
  <https://docs.zephyrproject.org/latest/kernel/services/interrupts.html>  
  Lies den Abschnitt „Overview" (ca. 10 Min).

### Wichtige Regeln für ISRs

1. **Halte ISRs kurz!** Während eine ISR läuft, können keine Threads ausgeführt werden.
2. **Kein `k_sleep()` in einer ISR!** ISRs dürfen nicht blockieren.
3. **Kein Logging in einer ISR!** Verwende stattdessen `printk()` oder signalisiere einem Thread.
4. In der Praxis: Setze in der ISR nur ein Flag oder gib ein Semaphor frei, und erledige die eigentliche Arbeit in einem Thread.

---

## 6.5 — Praxis: Button mit Interrupt-Callback

Da der Pico W keinen eingebauten Button hat, verwenden wir einen externen Taster auf dem Breadboard.

### Hardware-Aufbau

1. Stecke einen Taster auf das Breadboard.
2. Verbinde eine Seite des Tasters mit **GPIO 15** des Pico.
3. Verbinde die andere Seite mit **GND**.
4. Wir nutzen den internen Pull-Up-Widerstand – kein externer Widerstand nötig.

### Aufgabe: Button-Interrupt implementieren

Ersetze den Inhalt von `components/main/src/main.c`:

```c
#include <zephyr/kernel.h>
#include <zephyr/drivers/gpio.h>
#include <zephyr/logging/log.h>

LOG_MODULE_REGISTER(main, LOG_LEVEL_INF);

/* LED */
static const struct gpio_dt_spec led = GPIO_DT_SPEC_GET(DT_ALIAS(led0), gpios);

/* Button auf GPIO 15 - manuell definiert (nicht aus DT) */
static const struct gpio_dt_spec button = {
    .port = DEVICE_DT_GET(DT_NODELABEL(gpio0)),
    .pin = 15,
    .dt_flags = GPIO_ACTIVE_LOW,  /* Pull-Up: gedrückt = LOW */
};

static struct gpio_callback button_cb_data;

/* Diese Funktion wird aufgerufen, wenn der Button gedrückt wird */
void button_pressed_callback(const struct device *dev,
                             struct gpio_callback *cb,
                             uint32_t pins)
{
    /* ISR: nur kurze Aktionen! */
    gpio_pin_toggle_dt(&led);
    printk("Button gedrückt! LED umgeschaltet.\n");
}

int main(void)
{
    int ret;

    /* LED konfigurieren */
    if (!gpio_is_ready_dt(&led)) {
        LOG_ERR("LED nicht bereit");
        return -1;
    }
    gpio_pin_configure_dt(&led, GPIO_OUTPUT_INACTIVE);

    /* Button konfigurieren */
    if (!device_is_ready(button.port)) {
        LOG_ERR("GPIO Port nicht bereit");
        return -1;
    }

    ret = gpio_pin_configure(button.port, button.pin,
                             GPIO_INPUT | GPIO_PULL_UP);
    if (ret < 0) {
        LOG_ERR("Fehler beim Konfigurieren des Buttons: %d", ret);
        return -1;
    }

    /* Interrupt konfigurieren: auslösen bei fallender Flanke */
    ret = gpio_pin_interrupt_configure(button.port, button.pin,
                                       GPIO_INT_EDGE_TO_ACTIVE);
    if (ret < 0) {
        LOG_ERR("Fehler beim Konfigurieren des Interrupts: %d", ret);
        return -1;
    }

    /* Callback registrieren */
    gpio_init_callback(&button_cb_data, button_pressed_callback,
                       BIT(button.pin));
    gpio_add_callback(button.port, &button_cb_data);

    LOG_INF("Button-Interrupt Demo gestartet. Drücke den Taster!");

    /* main kann schlafen - der Interrupt weckt die Callback-Funktion */
    while (1) {
        k_sleep(K_FOREVER);
    }

    return 0;
}
```

Baue, flashe und teste: Jeder Tastendruck schaltet die LED um.

### Was passiert hier?

1. **`gpio_pin_interrupt_configure()`** — Sagt dem GPIO-Controller: „Löse einen Interrupt aus, wenn sich der Pin ändert"
2. **`gpio_init_callback()` + `gpio_add_callback()`** — Registriert eine Funktion, die beim Interrupt aufgerufen wird
3. **`button_pressed_callback()`** — Die ISR. Sie wird bei jedem Tastendruck aufgerufen und toggelt die LED
4. **`k_sleep(K_FOREVER)`** — main schläft für immer. Die CPU wird nur aufgeweckt, wenn ein Interrupt kommt

---

## 6.6 — Experiment: Interrupt + Thread kombinieren

### Aufgabe: Saubere Architektur

Ändere den Code so, dass die ISR nur ein **Semaphor** freigibt und ein separater Thread die eigentliche Arbeit erledigt. Das ist die professionelle Vorgehensweise.

> **Tipp:** Verwende `K_SEM_DEFINE(button_sem, 0, 1);` und in der ISR `k_sem_give(&button_sem);`. Im Thread wartest du mit `k_sem_take(&button_sem, K_FOREVER);`.

Wenn du nicht weiterkommst, schau in die Zephyr-Doku zu Semaphoren:  
<https://docs.zephyrproject.org/latest/kernel/services/synchronization/semaphores.html>

---

## Selbsttest – Kapitel 6

- [ ] Kann ich eine LED mit `gpio_pin_toggle_dt()` blinken lassen?
- [ ] Verstehe ich, was `GPIO_DT_SPEC_GET()` macht?
- [ ] Kann ich die GPIO-Shell benutzen?
- [ ] Kann ich den Unterschied zwischen Polling und Interrupt erklären?
- [ ] Habe ich einen Button-Interrupt-Callback implementiert und getestet?
- [ ] Verstehe ich, warum ISRs kurz sein müssen?

> **Alle Haken gesetzt?** Weiter zu Kapitel 7: Devicetree!  
> **Probleme?** Schreib eine Mail an [LEHRER-EMAIL].
