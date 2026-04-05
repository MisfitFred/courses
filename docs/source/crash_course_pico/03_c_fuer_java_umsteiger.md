# Kapitel 3: C für Java-Umsteiger

**Zeitaufwand:** ca. 4 Stunden  
**Voraussetzung:** Kapitel 1 und 2 abgeschlossen  
**Ziel:** Du verstehst die wichtigsten Unterschiede zwischen Java und C, kannst einfache C-Programme schreiben, kompilieren und ausführen.

---

## Warum C?

Java läuft in einer virtuellen Maschine (JVM) – zwischen deinem Code und der Hardware sitzt immer ein Übersetzer. In der Embedded-Welt ist das nicht möglich: Ein Mikrocontroller wie der RP2040 hat nur 264 KB RAM. Da muss dein Programm direkt auf der Hardware laufen, ohne JVM, ohne Garbage Collector, ohne Overhead. Deshalb ist C die Standardsprache für Embedded-Systeme.

Die gute Nachricht: Java wurde aus C entwickelt. Die Syntax ist zu 70 % identisch. Schleifen, if/else, Funktionsaufrufe – alles bekannt. Die wichtigsten Unterschiede lernst du in diesem Kapitel.

---

## 3.1 — Von Java zu C: Der Überblick

### Lies zuerst

- **C for Java Programmers (Kapitel 1–5)** — George Ferguson, University of Rochester  
  <https://www.cs.rochester.edu/u/ferguson/csc/c/c-for-java-programmers.pdf>  
  Dieses PDF ist speziell für Leute geschrieben, die Java können und C lernen wollen. Lies **Kapitel 1 bis 5** (Seiten 1–30). Das dauert ca. 45 Minuten. Kapitel 6+ kannst du überspringen.

### Die wichtigsten Unterschiede auf einen Blick

| Thema | Java | C |
|-------|------|---|
| Objektorientiert | Ja (Klassen, Objekte) | Nein (Funktionen + Structs) |
| Speicherverwaltung | Automatisch (Garbage Collector) | Manuell (malloc/free) |
| Kompilierung | Bytecode → JVM | Maschinencode → direkt auf CPU |
| Strings | String-Klasse | char-Arrays mit `\0` am Ende |
| Boolean | `true` / `false` | 0 = false, alles andere = true |
| Arrays | Wissen ihre Länge | Wissen ihre Länge **nicht** |
| Import | `import java.util.*` | `#include <stdio.h>` |
| Einstiegspunkt | `public static void main(String[] args)` | `int main(void)` |

---

## 3.2 — Erstes C-Programm: Hello World

### Aufgabe: Hello World kompilieren und ausführen

1. Erstelle einen Arbeitsordner:
   ```bash
   mkdir ~/c-uebungen
   cd ~/c-uebungen
   ```

2. Erstelle eine Datei `hello.c`:
   ```bash
   nano hello.c
   ```

3. Schreibe folgenden Code hinein:
   ```c
   #include <stdio.h>

   int main(void)
   {
       printf("Hello, World!\n");
       return 0;
   }
   ```

4. Speichere (Strg+O, Enter, Strg+X) und kompiliere:
   ```bash
   gcc hello.c -o hello
   ```
   - `gcc` = der C-Compiler
   - `hello.c` = deine Quelldatei
   - `-o hello` = Name der ausführbaren Datei

5. Starte das Programm:
   ```bash
   ./hello
   ```

### Was ist hier anders als in Java?

- `#include <stdio.h>` ist wie `import` – es bindet die Standard-Ein-/Ausgabe-Bibliothek ein
- `printf()` statt `System.out.println()` – die Formatierung funktioniert mit Platzhaltern: `%d` für int, `%f` für float, `%s` für Strings
- `return 0;` am Ende von main – das sagt dem Betriebssystem „alles OK"
- Es gibt keine Klasse, kein `public static` – nur eine Funktion namens `main`

---

## 3.3 — Variablen, Funktionen und Kontrollstrukturen

### Aufgabe: Vergleich Java vs. C

Erstelle eine Datei `rechner.c` mit folgendem Inhalt:

```c
#include <stdio.h>

/* Funktion: Berechnet die Summe von a bis b */
/* In Java wäre das eine Methode in einer Klasse */
int summe(int a, int b)
{
    int ergebnis = 0;

    for (int i = a; i <= b; i++) {
        ergebnis += i;
    }

    return ergebnis;
}

int main(void)
{
    int start = 1;
    int ende = 100;

    int resultat = summe(start, ende);

    printf("Summe von %d bis %d = %d\n", start, ende, resultat);

    /* if/else funktioniert genau wie in Java */
    if (resultat > 5000) {
        printf("Das ist eine große Zahl!\n");
    } else {
        printf("Das ist eine kleine Zahl.\n");
    }

    return 0;
}
```

Kompiliere und starte:
```bash
gcc rechner.c -o rechner
./rechner
```

**Beobachte:** Schleifen, if/else, Variablen – alles (fast) identisch zu Java. Der Hauptunterschied: Funktionen stehen **außerhalb** von Klassen, weil es in C keine Klassen gibt.

---

## 3.4 — Pointer (Zeiger): Das wichtigste neue Konzept

Pointer sind das Konzept, das es in Java nicht gibt. In Java arbeitest du immer mit Referenzen, aber du siehst sie nie direkt. In C hast du volle Kontrolle über Speicheradressen.

### Lies zuerst

- **learn-c.org: Pointers (interaktiv mit Übungen im Browser)**  
  <https://www.learn-c.org/en/Pointers>  
  Arbeite die Lektion durch und löse die Übung am Ende (ca. 15 Min).

- **Programiz: C Pointers (mit Bildern und Beispielen)**  
  <https://www.programiz.com/c-programming/c-pointers>  
  Lies die Seite durch, die Grafiken helfen beim Verständnis (ca. 10 Min).

### Die Grundidee

Jede Variable hat in C zwei Eigenschaften:
1. Einen **Wert** (z.B. `42`)
2. Eine **Adresse** im Speicher (z.B. `0x7ffd5e8a3b2c`)

Ein **Pointer** ist eine Variable, die eine Adresse speichert.

```c
int zahl = 42;       // Eine normale Variable mit Wert 42
int *ptr = &zahl;     // Ein Pointer, der die ADRESSE von 'zahl' speichert

printf("%d\n", zahl);   // Gibt 42 aus (der Wert)
printf("%p\n", &zahl);  // Gibt die Adresse von 'zahl' aus
printf("%p\n", ptr);    // Gibt dasselbe aus (ptr speichert die Adresse)
printf("%d\n", *ptr);   // Gibt 42 aus (Wert AN der Adresse)
```

Die zwei wichtigen Operatoren:
- `&` = „Gib mir die Adresse von" (address-of)
- `*` = „Gib mir den Wert an dieser Adresse" (dereference)

### Aufgabe: Pointer ausprobieren

Erstelle `pointer.c`:

```c
#include <stdio.h>

/* Diese Funktion KANN den Wert nicht ändern (Kopie!) */
void verdopple_falsch(int x)
{
    x = x * 2;
    printf("  Innerhalb der Funktion: x = %d\n", x);
}

/* Diese Funktion KANN den Wert ändern (Pointer!) */
void verdopple_richtig(int *x)
{
    *x = *x * 2;
    printf("  Innerhalb der Funktion: *x = %d\n", *x);
}

int main(void)
{
    int zahl = 10;

    printf("Vorher: zahl = %d\n", zahl);

    printf("Versuch 1 (ohne Pointer):\n");
    verdopple_falsch(zahl);
    printf("Nachher: zahl = %d\n\n", zahl);  /* Immer noch 10! */

    printf("Versuch 2 (mit Pointer):\n");
    verdopple_richtig(&zahl);
    printf("Nachher: zahl = %d\n", zahl);    /* Jetzt 20! */

    return 0;
}
```

Kompiliere und starte:
```bash
gcc pointer.c -o pointer
./pointer
```

**Warum ist das wichtig für Embedded?**  
Auf einem Mikrocontroller greifst du direkt auf Hardware-Register zu – das sind bestimmte Speicheradressen. Wenn du z.B. eine LED einschalten willst, schreibst du einen Wert an eine bestimmte Adresse. Dafür brauchst du Pointer.

---

## 3.5 — Structs: Klassen ohne Methoden

In Java fasst man Daten und Methoden in Klassen zusammen. In C gibt es dafür **Structs** – die haben Daten, aber keine Methoden.

### Aufgabe: Struct erstellen

Erstelle `sensor.c`:

```c
#include <stdio.h>

/* Das ist wie eine Java-Klasse, aber nur mit Attributen */
struct SensorDaten {
    float temperatur;
    float luftfeuchtigkeit;
    int sensor_id;
};

/* Funktionen die mit dem Struct arbeiten, stehen separat */
void sensor_ausgeben(struct SensorDaten *daten)
{
    printf("Sensor #%d:\n", daten->sensor_id);
    printf("  Temperatur:      %.1f °C\n", daten->temperatur);
    printf("  Luftfeuchtigkeit: %.1f %%\n", daten->luftfeuchtigkeit);
}

int main(void)
{
    /* Struct-Variable erstellen und initialisieren */
    struct SensorDaten messung = {
        .temperatur = 22.5,
        .luftfeuchtigkeit = 45.0,
        .sensor_id = 1
    };

    sensor_ausgeben(&messung);

    /* Wert ändern */
    messung.temperatur = 23.1;
    printf("\nNach Änderung:\n");
    sensor_ausgeben(&messung);

    return 0;
}
```

Kompiliere und starte:
```bash
gcc sensor.c -o sensor
./sensor
```

**Beachte:** `daten->temperatur` ist die Kurzform von `(*daten).temperatur`. Der Pfeil-Operator `->` wird verwendet, wenn man über einen Pointer auf ein Struct-Member zugreift. Das wirst du in Zephyr ständig sehen.

---

## 3.6 — Header-Dateien und getrennte Kompilierung

In Java hast du pro Klasse eine `.java`-Datei. In C teilt man Code in zwei Dateitypen auf:
- **`.h` (Header):** Enthält die **Deklarationen** (was gibt es?)
- **`.c` (Source):** Enthält die **Implementierung** (wie funktioniert es?)

### Aufgabe: Header und Source trennen

1. Erstelle `mathe.h`:
   ```c
   #ifndef MATHE_H
   #define MATHE_H

   /* Deklaration: "Diese Funktion existiert" */
   int addiere(int a, int b);
   int multipliziere(int a, int b);

   #endif
   ```

2. Erstelle `mathe.c`:
   ```c
   #include "mathe.h"

   /* Implementierung */
   int addiere(int a, int b)
   {
       return a + b;
   }

   int multipliziere(int a, int b)
   {
       return a * b;
   }
   ```

3. Erstelle `main.c`:
   ```c
   #include <stdio.h>
   #include "mathe.h"

   int main(void)
   {
       printf("3 + 4 = %d\n", addiere(3, 4));
       printf("3 * 4 = %d\n", multipliziere(3, 4));
       return 0;
   }
   ```

4. Kompiliere beide `.c`-Dateien zusammen:
   ```bash
   gcc main.c mathe.c -o rechner2
   ./rechner2
   ```

**Warum `#ifndef` / `#define` / `#endif`?**  
Das ist ein sogenannter **Include Guard**. Er verhindert, dass ein Header versehentlich mehrmals eingebunden wird (was zu Fehlern führen würde). Das wirst du in jedem C-Projekt sehen – auch in Zephyr.

---

## 3.7 — CMake: Das Build-System verstehen

In der Übung hast du mit `gcc datei.c` kompiliert. Bei echten Projekten (wie dem Zephyr-Projekt) gibt es hunderte Dateien. Da braucht man ein **Build-System**, das automatisch weiß, was in welcher Reihenfolge kompiliert werden muss.

Zephyr verwendet **CMake**. Du musst CMake nicht meistern, aber verstehen, was eine `CMakeLists.txt` tut.

### Lies zuerst

- **CMake Tutorial – Schritt 1 (offizielle Doku)**  
  <https://cmake.org/cmake/help/latest/guide/tutorial/A%20Basic%20Starting%20Point.html>  
  Lies nur „Step 1: A Basic Starting Point" (ca. 10 Min). Überspringe die Übungen.

### Aufgabe: Die CMakeLists.txt des Kursprojekts lesen

Öffne die CMakeLists.txt des Kurs-Repos:
```bash
cat ~/rpico-room-sensor/CMakeLists.txt
```

Versuche, folgende Fragen zu beantworten:
1. Welchen Namen hat das Projekt?
2. Welche Programmiersprache wird verwendet?
3. Welche Unterordner (Komponenten) werden eingebunden?

> **Tipp:** `add_subdirectory(components/main)` bedeutet: „Schau im Ordner `components/main` nach einer weiteren `CMakeLists.txt` und bau den Code dort auch."

Du musst CMake-Dateien vorerst nicht selbst schreiben – nur lesen und verstehen können.

---

## Selbsttest – Kapitel 3

- [ ] Kann ich ein C-Programm mit `gcc` kompilieren und ausführen?
- [ ] Verstehe ich `printf()` mit Platzhaltern (%d, %f, %s)?
- [ ] Verstehe ich den Unterschied zwischen `&` (Adresse) und `*` (Dereferenzierung)?
- [ ] Kann ich erklären, warum man in C Pointer braucht, um Werte in Funktionen zu ändern?
- [ ] Weiß ich, was ein Struct ist und wie `->` funktioniert?
- [ ] Verstehe ich den Zweck von Header-Dateien (.h) und Include Guards?
- [ ] Kann ich eine einfache CMakeLists.txt grob lesen?

> **Alle Haken gesetzt?** Dann weiter zu Kapitel 4: Toolchain & Hello World auf dem Pico!  
> **Probleme?** Schreib eine Mail an [LEHRER-EMAIL].

---

## Bonus: Weiterführende Quellen

Falls du Pointer oder andere C-Themen vertiefen willst:

- **learn-c.org** — Interaktive C-Übungen im Browser  
  <https://www.learn-c.org/>  
  Empfohlene Lektionen: „Hello World", „Variables and Types", „Arrays", „Pointers", „Structures", „Function arguments by reference"

- **C vs. Java Vergleichstabelle** — Princeton University  
  <https://introcs.cs.princeton.edu/java/faq/c2java.html>  
  Kompakte Gegenüberstellung, gut als Nachschlagewerk
