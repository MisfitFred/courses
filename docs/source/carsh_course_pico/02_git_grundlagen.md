# Kapitel 2: Git — Versionskontrolle

**Zeitaufwand:** ca. 2 Stunden  
**Voraussetzung:** Kapitel 1 abgeschlossen (WSL2 läuft, Terminal-Grundlagen bekannt)  
**Ziel:** Du verstehst, was Git ist, kannst Repos klonen, Änderungen committen und mit GitHub arbeiten.

---

## Warum Git?

Stell dir vor, du arbeitest an einem Programm und änderst etwas – danach funktioniert nichts mehr. Ohne Git musst du dich erinnern, was du geändert hast. Mit Git kannst du **jederzeit zu jedem vorherigen Stand zurückkehren**. Git ist das Standardwerkzeug in der Softwareentwicklung – egal ob bei Google, in Startups oder in der Embedded-Entwicklung.

---

## 2.1 — Git verstehen (Theorie)

### Lies zuerst

- **Atlassian: What is Git?**  
  <https://www.atlassian.com/git/tutorials/what-is-git>  
  Lies nur diese eine Seite (ca. 10 Min). Sie erklärt das Konzept hinter Git.

- **Atlassian: Setting up a repository**  
  <https://www.atlassian.com/git/tutorials/setting-up-a-repository>  
  Erklärt `git init`, `git clone`, `git config` (ca. 10 Min).

### Die Kernidee in einem Satz

> Git speichert **Schnappschüsse** (Snapshots) deines Projekts. Jeder Schnappschuss heißt **Commit** und hat eine eindeutige ID. Du kannst jederzeit zwischen Commits hin- und herspringen.

### Wichtige Begriffe

| Begriff | Bedeutung |
|---------|-----------|
| **Repository (Repo)** | Ein Projektordner, der von Git überwacht wird |
| **Commit** | Ein gespeicherter Snapshot deines Projekts |
| **Branch** | Ein paralleler Entwicklungszweig |
| **Clone** | Eine lokale Kopie eines Repos (z.B. von GitHub) |
| **Push** | Lokale Commits auf den Server (GitHub) hochladen |
| **Pull** | Änderungen vom Server herunterladen |
| **Staging Area** | Der „Wartebereich" – hierhin fügst du Dateien hinzu, bevor du sie commitest |

---

## 2.2 — GitHub-Account erstellen

Bevor du weiterarbeitest, brauchst du einen GitHub-Account.

### Aufgabe

1. Gehe auf <https://github.com> und erstelle einen kostenlosen Account.
2. Merke dir deinen Benutzernamen und dein Passwort.

---

## 2.3 — Git konfigurieren

### Aufgabe: Git einrichten

Öffne dein Ubuntu-Terminal und konfiguriere Git mit deinem Namen und deiner E-Mail (die gleiche wie bei GitHub):

```bash
git config --global user.name "Dein Name"
git config --global user.email "deine.email@beispiel.de"
git config --global init.defaultBranch main
```

Prüfe die Konfiguration:
```bash
git config --list
```

---

## 2.4 — Dein erstes eigenes Repo

### Lies zuerst

- **Atlassian: Saving changes (add & commit)**  
  <https://www.atlassian.com/git/tutorials/saving-changes>  
  Lies die Abschnitte zu `git add` und `git commit` (ca. 10 Min).

### Aufgabe: Ein lokales Repo erstellen und committen

1. Erstelle einen Projektordner:
   ```bash
   mkdir ~/mein-erstes-repo
   cd ~/mein-erstes-repo
   ```

2. Initialisiere ein Git-Repository:
   ```bash
   git init
   ```
   Git überwacht ab jetzt diesen Ordner.

3. Erstelle eine Datei:
   ```bash
   nano README.md
   ```
   Schreibe hinein:
   ```
   # Mein erstes Repo
   Das ist mein erstes Git-Projekt.
   ```
   Speichern: **Strg+O**, **Enter**, **Strg+X**.

4. Schau dir den Status an:
   ```bash
   git status
   ```
   Git zeigt dir, dass `README.md` eine „untracked file" ist – Git kennt die Datei noch nicht.

5. Füge die Datei zur Staging Area hinzu:
   ```bash
   git add README.md
   ```

6. Erstelle deinen ersten Commit:
   ```bash
   git commit -m "Erstes Commit: README hinzugefügt"
   ```

7. Schau dir die Historie an:
   ```bash
   git log --oneline
   ```
   Du siehst deinen Commit mit seiner kurzen ID und der Nachricht.

---

## 2.5 — Das Kursprojekt klonen

Jetzt klonst du das Projekt, mit dem wir den Rest des Kurses arbeiten werden.

### Aufgabe: Repository klonen

1. Klone das Kurs-Repo:
   ```bash
   cd ~
   git clone https://github.com/MisfitFred/rpico-room-sensor.git
   ```

2. Wechsle in das Repo:
   ```bash
   cd rpico-room-sensor
   ```

3. Schau dir die Struktur an:
   ```bash
   ls -la
   ```

4. Sieh dir die Commit-Historie an:
   ```bash
   git log --oneline
   ```

5. Sieh dir die Branches an:
   ```bash
   git branch -a
   ```

Nimm dir einen Moment, um die Dateien zu erkunden. Öffne z.B. die `README.md`:
```bash
cat README.md
```
Du musst den Inhalt noch nicht verstehen – es geht nur darum, dass du weißt, wie das Repo aufgebaut ist.

---

## 2.6 — Ändern, Committen, Wiederherstellen

### Aufgabe: Den Git-Workflow üben

1. Erstelle im geklonten Repo einen neuen Branch:
   ```bash
   git checkout -b mein-test-branch
   ```

2. Erstelle eine neue Datei:
   ```bash
   nano notizen.txt
   ```
   Schreibe etwas hinein (z.B. „Das ist ein Test") und speichere.

3. Stage und committe die Datei:
   ```bash
   git add notizen.txt
   git commit -m "Notizen-Datei hinzugefügt"
   ```

4. Ändere die Datei nochmal:
   ```bash
   nano notizen.txt
   ```
   Füge eine zweite Zeile hinzu und speichere.

5. Schau dir den Unterschied an:
   ```bash
   git diff
   ```
   Git zeigt dir genau, was sich geändert hat (grün = neu, rot = gelöscht).

6. Committe die Änderung:
   ```bash
   git add notizen.txt
   git commit -m "Notizen erweitert"
   ```

7. Schau dir die Historie an:
   ```bash
   git log --oneline
   ```
   Du siehst jetzt zwei Commits auf deinem Branch.

8. Wechsle zurück zum Haupt-Branch:
   ```bash
   git checkout main
   ```
   Prüfe mit `ls` – die Datei `notizen.txt` ist weg! Sie existiert nur auf deinem Test-Branch.

9. Wechsle zurück:
   ```bash
   git checkout mein-test-branch
   ```
   Die Datei ist wieder da. Das ist die Kraft von Branches.

---

## 2.7 — Interaktiv üben (Bonus)

Wenn du git-Konzepte wie Branching und Merging visuell verstehen willst, gibt es ein hervorragendes interaktives Tool:

- **Learn Git Branching**  
  <https://learngitbranching.js.org/?locale=de_DE>  
  Arbeite die Levels unter **„Main" → „Introduction Sequence"** durch (Levels 1–4).  
  Das sind ca. 15–20 Minuten und du siehst dabei visuell, was bei jedem Befehl passiert.

---

## Selbsttest – Kapitel 2

- [ ] Habe ich einen GitHub-Account?
- [ ] Kann ich ein Repo initialisieren (`git init`)?
- [ ] Verstehe ich den Workflow: `git add` → `git commit`?
- [ ] Habe ich das Kursprojekt geklont und kann die Dateien sehen?
- [ ] Kann ich einen Branch erstellen und zwischen Branches wechseln?
- [ ] Weiß ich, was `git status`, `git log` und `git diff` anzeigen?

> **Alle Haken gesetzt?** Dann weiter zu Kapitel 3: C für Java-Umsteiger!  
> **Probleme?** Schreib eine Mail an [LEHRER-EMAIL] mit Screenshot und Fehlermeldung.

---

## Zusammenfassung: Die wichtigsten Git-Befehle

```
git init                    # Neues Repo erstellen
git clone <url>             # Repo von GitHub klonen
git status                  # Aktuellen Status anzeigen
git add <datei>             # Datei zur Staging Area hinzufügen
git add .                   # Alle Änderungen hinzufügen
git commit -m "Nachricht"   # Snapshot erstellen
git log --oneline           # Commit-Historie anzeigen
git diff                    # Änderungen anzeigen
git branch                  # Branches auflisten
git checkout -b <name>      # Neuen Branch erstellen und wechseln
git checkout <name>         # Zu Branch wechseln
git push                    # Commits auf GitHub hochladen
git pull                    # Änderungen von GitHub herunterladen
```
