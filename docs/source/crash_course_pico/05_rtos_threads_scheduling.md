# Kapitel 5: RTOS — Threads & Scheduling

**Zeitaufwand:** ca. 3 Stunden  
**Voraussetzung:** Kapitel 4 abgeschlossen (Zephyr baut, Pico W funktioniert)  
**Ziel:** Du verstehst, warum man ein RTOS braucht, kannst eigene Threads erstellen und weißt, wie der Scheduler entscheidet, welcher Thread läuft.

---

## Warum ein RTOS?

Stell dir vor, dein Pico soll gleichzeitig:
1. Einen Temperatursensor auslesen (alle 2 Sekunden)
2. Einen Tastendruck erkennen (sofort reagieren)
3. Daten über USB ausgeben (wenn neue Messwerte da sind)

Mit einem einfachen `while(true)`-Programm (wie in Arduino) musst du das alles in einer Schleife nacheinander abarbeiten – wenn das Sensor-Auslesen 500ms dauert, reagiert dein Taster erst mit 500ms Verzögerung.

Ein **RTOS (Real-Time Operating System)** löst dieses Problem: Du schreibst jede Aufgabe als separaten **Thread**, und der **Scheduler** sorgt dafür, dass alle Threads fair und rechtzeitig CPU-Zeit bekommen.

---

## 5.1 — Theorie: Threads und Scheduling

### Lies zuerst

- **Zephyr Doku: Threads**  
  <https://docs.zephyrproject.org/latest/kernel/services/threads/index.html>  
  Lies die Abschnitte „Lifecycle", „Thread States" und „Thread Priorities" (ca. 15 Min).

- **Zephyr-RTOS-Tutorial: Scheduling (mit Diagrammen)**  
  <https://github.com/maksimdrachov/zephyr-rtos-tutorial/blob/main/docs/5-scheduling/introduction.md>  
  Gute visuelle Erklärung des Schedulers (ca. 15 Min).

### Die Kernkonzepte

**Thread-Zustände:**

```
             ┌──────────┐
        ┌───►│  Ready   │◄──────┐
        │    └────┬─────┘       │
        │         │ Scheduler   │ k_sem_give() /
        │         ▼ wählt aus   │ k_sleep() läuft ab
        │    ┌──────────┐       │
        │    │ Running  │───────┤
        │    └────┬─────┘       │
        │         │ k_sleep()   │
        │         │ k_sem_take()│
        │         ▼             │
        │    ┌──────────┐       │
        └────│ Waiting  │───────┘
             └──────────┘
```

**Prioritäten:**
- Niedrigere Zahl = höhere Priorität (Priorität 1 ist wichtiger als Priorität 7)
- Negative Prioritäten = **kooperative** Threads (werden nicht unterbrochen)
- Nicht-negative Prioritäten = **preemptive** Threads (können vom Scheduler unterbrochen werden)

**Der Scheduler wählt immer den Thread mit der höchsten Priorität, der bereit (ready) ist.**

---

## 5.2 — Praxis: Deinen ersten Thread erstellen

Jetzt schreibst du Code, der auf dem Pico läuft. Wir arbeiten im Kursprojekt.

### Aufgabe: Zwei Threads mit unterschiedlichen Prioritäten

1. Öffne die Hauptdatei des Projekts:
   ```bash
   cd ~/zephyrproject
   nano rpico-room-sensor/components/main/src/main.c
   ```

2. Ersetze den Inhalt mit folgendem Code:

   ```c
   #include <zephyr/kernel.h>
   #include <zephyr/logging/log.h>

   LOG_MODULE_REGISTER(main, LOG_LEVEL_INF);

   /* Stack und Thread für Thread A definieren */
   #define THREAD_A_STACKSIZE 512
   #define THREAD_A_PRIORITY  5

   /* Stack und Thread für Thread B definieren */
   #define THREAD_B_STACKSIZE 512
   #define THREAD_B_PRIORITY  7    /* niedrigere Priorität als A */

   void thread_a_entry(void *p1, void *p2, void *p3)
   {
       ARG_UNUSED(p1);
       ARG_UNUSED(p2);
       ARG_UNUSED(p3);

       while (1) {
           LOG_INF("Thread A läuft (Priorität %d)", THREAD_A_PRIORITY);
           k_sleep(K_MSEC(1000));  /* 1 Sekunde schlafen */
       }
   }

   void thread_b_entry(void *p1, void *p2, void *p3)
   {
       ARG_UNUSED(p1);
       ARG_UNUSED(p2);
       ARG_UNUSED(p3);

       while (1) {
           LOG_INF("Thread B läuft (Priorität %d)", THREAD_B_PRIORITY);
           k_sleep(K_MSEC(2000));  /* 2 Sekunden schlafen */
       }
   }

   /* Threads statisch definieren - sie starten automatisch */
   K_THREAD_DEFINE(thread_a, THREAD_A_STACKSIZE,
                   thread_a_entry, NULL, NULL, NULL,
                   THREAD_A_PRIORITY, 0, 0);

   K_THREAD_DEFINE(thread_b, THREAD_B_STACKSIZE,
                   thread_b_entry, NULL, NULL, NULL,
                   THREAD_B_PRIORITY, 0, 0);

   int main(void)
   {
       LOG_INF("=== Threads & Scheduling Demo ===");
       LOG_INF("Thread A: Priorität %d, alle 1s", THREAD_A_PRIORITY);
       LOG_INF("Thread B: Priorität %d, alle 2s", THREAD_B_PRIORITY);

       /* main() kehrt zurück - die Threads laufen weiter! */
       return 0;
   }
   ```

3. Baue und flashe:
   ```bash
   source .venv/bin/activate
   west build -p auto -b rpi_pico/rp2040/w \
     -d rpico-room-sensor/build \
     rpico-room-sensor \
     -S usb_serial_port
   west flash -d rpico-room-sensor/build
   ```

4. Verbinde dich per Serial-Monitor und beobachte die Ausgabe.

### Was passiert hier?

- `K_THREAD_DEFINE()` erstellt einen Thread **zur Compile-Zeit** (statisch). Der Thread startet automatisch beim Booten.
- `k_sleep(K_MSEC(1000))` sagt dem Scheduler: „Ich brauche 1 Sekunde nichts zu tun." Der Scheduler kann in dieser Zeit andere Threads ausführen.
- `LOG_INF()` ist Zephyrs Logging-System – besser als `printf()`, weil es Thread-sicher ist und Zeitstempel mitliefert.
- `main()` ist selbst ein Thread! Er kann auch `return 0;` machen – die anderen Threads laufen trotzdem weiter.

---

## 5.3 — Scheduling beobachten

### Aufgabe: Thread-Info über die Shell abfragen

Wenn die Firmware läuft, öffne die Shell und tippe:

```
uart:~$ kernel threads
```

Dieser Befehl zeigt dir alle laufenden Threads mit ihren Prioritäten, Stack-Nutzung und Zustand. Du siehst dort deine beiden Threads plus System-Threads (z.B. den Shell-Thread, den Idle-Thread).

Probiere auch:
```
uart:~$ kernel stacks
```
Das zeigt dir, wie viel Stack-Speicher jeder Thread verbraucht hat.

### Fragen zum Nachdenken

1. Was passiert, wenn du Thread B die gleiche Priorität wie Thread A gibst?
2. Was passiert, wenn du das `k_sleep()` in Thread A entfernst? (Vorsicht: Der Thread blockiert dann die CPU!)
3. Warum funktioniert die Shell noch, obwohl deine Threads laufen?

---

## 5.4 — Zephyr Logging verstehen

Du hast gerade `LOG_INF()` benutzt. Zephyr hat ein mächtiges Logging-System.

### Die Log-Level

| Makro | Level | Bedeutung |
|-------|-------|-----------|
| `LOG_ERR()` | Error | Kritische Fehler |
| `LOG_WRN()` | Warning | Warnungen |
| `LOG_INF()` | Info | Informationen |
| `LOG_DBG()` | Debug | Debug-Details (standardmäßig aus) |

### Konfiguration in prj.conf

Öffne die `prj.conf` deines Projekts:
```bash
cat ~/zephyrproject/rpico-room-sensor/prj.conf
```

Du siehst dort:
```
CONFIG_LOG=y
CONFIG_LOG_PRINTK=y
CONFIG_LOG_DEFAULT_LEVEL=3
```

- `CONFIG_LOG=y` → Logging ist aktiviert
- `CONFIG_LOG_DEFAULT_LEVEL=3` → Level 3 = Info (alles bis INF wird angezeigt)

### Lies zuerst

- **Zephyr Doku: Logging**  
  <https://docs.zephyrproject.org/latest/services/logging/index.html>  
  Lies den Abschnitt „Overview" und „Usage" (ca. 10 Min).

---

## Selbsttest – Kapitel 5

- [ ] Kann ich erklären, warum man ein RTOS braucht?
- [ ] Verstehe ich die Thread-Zustände: Running, Ready, Waiting?
- [ ] Kann ich mit `K_THREAD_DEFINE()` einen Thread erstellen?
- [ ] Verstehe ich, was `k_sleep()` macht und warum es wichtig ist?
- [ ] Weiß ich, dass niedrigere Prioritätszahl = höhere Priorität bedeutet?
- [ ] Kann ich mit `kernel threads` in der Shell die laufenden Threads sehen?
- [ ] Verstehe ich den Unterschied zwischen `LOG_INF()`, `LOG_WRN()`, `LOG_ERR()`?

> **Alle Haken gesetzt?** Weiter zu Kapitel 6: GPIO & Interrupts!  
> **Probleme?** Schreib eine Mail an [LEHRER-EMAIL].
