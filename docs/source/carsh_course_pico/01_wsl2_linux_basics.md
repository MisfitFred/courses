# Kapitel 1: WSL2 & Linux-Grundlagen

**Zeitaufwand:** ca. 2 Stunden  
**Ziel:** Du hast eine funktionierende Linux-Umgebung auf deinem Windows-PC und kannst dich im Terminal bewegen.

---

## Warum Linux?

In der professionellen Embedded-Entwicklung wird fast ausschließlich unter Linux gearbeitet. Compiler, Build-Systeme und Tools wie Zephyr RTOS sind für Linux gebaut. Mit **WSL2** (Windows Subsystem for Linux) bekommst du ein vollwertiges Linux direkt in Windows – ohne Dual-Boot, ohne virtuelle Maschine.

---

## 1.1 — WSL2 installieren

### Lies zuerst

- **Microsoft: WSL installieren (offizielle Doku)**  
  <https://learn.microsoft.com/de-de/windows/wsl/install>  
  Lies den Abschnitt „Installieren von WSL" bis einschließlich „Einrichten deines Linux-Benutzernamens und -Kennworts".

### Aufgabe: WSL2 mit Ubuntu einrichten

1. Öffne **PowerShell als Administrator** (Rechtsklick → „Als Administrator ausführen").
2. Führe folgenden Befehl aus:
   ```powershell
   wsl --install
   ```
3. Starte deinen PC neu, wenn du dazu aufgefordert wirst.
4. Nach dem Neustart öffnet sich automatisch ein Ubuntu-Fenster. Lege einen **Benutzernamen** und ein **Passwort** fest (das Passwort wird beim Tippen nicht angezeigt – das ist normal!).
5. Überprüfe, dass du WSL2 verwendest. Öffne eine neue PowerShell und tippe:
   ```powershell
   wsl -l -v
   ```
   In der Spalte „VERSION" muss **2** stehen.

> **Problem?** Falls bei dir WSL 1 steht, führe aus:  
> `wsl --set-version Ubuntu 2`

### Ergebnis

Du hast jetzt Ubuntu in deinem Windows. Du kannst es jederzeit starten, indem du im Startmenü „Ubuntu" eingibst.

---

## 1.2 — Das Linux-Terminal kennenlernen

### Lies zuerst

- **Ubuntu: The Linux command line for beginners**  
  <https://ubuntu.com/tutorials/command-line-for-beginners>  
  Arbeite die Abschnitte 1–5 durch (ca. 30 Min). Das Tutorial erklärt dir Schritt für Schritt, wie das Terminal funktioniert.

- **Ergänzend / als Nachschlagewerk:**  
  <https://linuxjourney.com/lesson/the-shell> (Abschnitt „Command Line" – kurz und übersichtlich)

### Die wichtigsten Befehle auf einen Blick

| Befehl | Bedeutung | Beispiel |
|--------|-----------|---------|
| `pwd` | Zeigt das aktuelle Verzeichnis | `pwd` → `/home/deinname` |
| `ls` | Listet Dateien und Ordner auf | `ls -la` (zeigt auch versteckte Dateien) |
| `cd` | Wechselt das Verzeichnis | `cd ~/projekte` |
| `mkdir` | Erstellt einen Ordner | `mkdir mein-ordner` |
| `cp` | Kopiert Dateien | `cp datei.txt kopie.txt` |
| `mv` | Verschiebt / benennt um | `mv alt.txt neu.txt` |
| `rm` | Löscht Dateien | `rm datei.txt` |
| `cat` | Zeigt Dateiinhalt an | `cat readme.md` |
| `nano` | Einfacher Texteditor | `nano datei.txt` (Speichern: Strg+O, Beenden: Strg+X) |
| `sudo` | Befehl als Admin ausführen | `sudo apt update` |

### Aufgabe: Terminal-Übungen

Öffne dein Ubuntu-Terminal und führe **alle Schritte nacheinander** aus. Schreib dir die Antworten auf.

1. Finde heraus, in welchem Verzeichnis du dich befindest (`pwd`).
2. Erstelle im Home-Verzeichnis einen Ordner namens `uebung`:
   ```bash
   mkdir ~/uebung
   ```
3. Wechsle in diesen Ordner und erstelle darin eine Datei:
   ```bash
   cd ~/uebung
   nano hallo.txt
   ```
   Schreibe einen beliebigen Text hinein, speichere mit **Strg+O**, **Enter**, dann **Strg+X**.
4. Zeige den Inhalt der Datei an:
   ```bash
   cat hallo.txt
   ```
5. Erstelle eine Kopie der Datei:
   ```bash
   cp hallo.txt kopie.txt
   ```
6. Liste alle Dateien im Ordner auf:
   ```bash
   ls -la
   ```
7. Lösche die Kopie:
   ```bash
   rm kopie.txt
   ```
8. Gehe zurück ins Home-Verzeichnis:
   ```bash
   cd ~
   ```

---

## 1.3 — Pakete installieren

In Linux installiert man Software über den **Paketmanager** (vergleichbar mit einem App Store, aber im Terminal).

### Aufgabe: System aktualisieren und Pakete installieren

1. Aktualisiere die Paketlisten und installiere Updates:
   ```bash
   sudo apt update && sudo apt upgrade -y
   ```
   (`sudo` = „SuperUser Do" – du führst den Befehl mit Admin-Rechten aus)

2. Installiere ein paar Tools, die wir später brauchen werden:
   ```bash
   sudo apt install -y git cmake ninja-build python3-pip python3-venv
   ```
3. Prüfe, ob git installiert wurde:
   ```bash
   git --version
   ```
   Es sollte eine Versionsnummer erscheinen (z.B. `git version 2.43.0`).

---

## Selbsttest – Kapitel 1

Beantworte diese Fragen für dich selbst, bevor du weitergehst:

- [ ] Läuft bei mir WSL2 mit Ubuntu? (`wsl -l -v` zeigt VERSION 2)
- [ ] Kann ich im Terminal navigieren? (`cd`, `ls`, `pwd`)
- [ ] Kann ich Dateien erstellen, kopieren, löschen?
- [ ] Weiß ich, wie ich mit `nano` eine Datei bearbeite?
- [ ] Weiß ich, was `sudo apt install` tut?

> **Alle Haken gesetzt?** Dann weiter zu Kapitel 2!  
> **Probleme?** Schreib eine Mail an [LEHRER-EMAIL] mit einer Beschreibung des Problems und einem Screenshot.
