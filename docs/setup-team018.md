# VLAOrga lokal starten (Team 018)

Diese Anleitung beschreibt Schritt für Schritt, wie man VLAOrga zusammen mit einer lokalen Kopie der Versuchsdatenbank **Linus** baut und startet.

> **Getestet:** am 17.11.2026 auf macOS (Apple Silicon, M1) mit Docker Desktop und VS Code.
> **Nicht getestet:** Windows und Linux. Die Hinweise dafür sind sorgfältig zusammengestellt, aber noch nicht ausprobiert. Wer es dort durchspielt, ergänzt bitte diese Datei.

## Überblick: was läuft wo?

| Teil | Technik | Läuft als | Adresse |
|---|---|---|---|
| Backend | Java 21, Spring Boot 3.5, Gradle | Docker-Container `dev-app` | Port 8080 |
| Frontend | React 19, TypeScript, Vite | `npm run dev` im Terminal | <http://localhost:5173> |
| VLAOrga-Datenbank | PostgreSQL 16 | Docker-Container `db` | Port 5432 |
| Linus-Datenbank | MariaDB 10.5 (wie Produktion) | Docker-Container `linus-db` | Port 3306 |

- **VLAOrga** ist der Kalender der Vorlesungsassistenz. Eigene Daten (Termine, Notizen, Benutzer) speichert es in PostgreSQL.
- **Linus** ist die Versuchsdatenbank (Versuche, Buchungen). VLAOrga liest daraus **nur**. Lokal ersetzen wir Linus durch eine MariaDB, in die wir einen Dump (eine Sicherungskopie als `.sql`-Datei) einspielen.
- Java, PostgreSQL und MariaDB muss man **nicht** selbst installieren: Das erledigt Docker.
- Solange die VM des IAP noch nicht bereitsteht, läuft alles lokal. Auf der VM werden später dieselben Container gestartet.

## 0. Voraussetzungen installieren

| Programm | macOS | Windows | Linux |
|---|---|---|---|
| git | `xcode-select --install` oder <https://git-scm.com> | <https://git-scm.com> (installiert auch **Git Bash**, wichtig, siehe unten) | Paketmanager, z. B. `sudo apt install git` |
| Docker | Docker Desktop | Docker Desktop mit **WSL 2** (bei der Installation vorgeschlagen) | Docker Engine + Compose-Plugin (<https://docs.docker.com/engine/install/>) |
| Node.js ≥ 20 und npm | <https://nodejs.org> (LTS) oder `brew install node` | <https://nodejs.org> (LTS) | <https://nodejs.org> oder Paketmanager |
| VS Code (empfohlen) | <https://code.visualstudio.com> | dto. | dto. |

Prüfen, ob alles da ist (im Terminal, siehe nächster Abschnitt):

```bash
git --version
docker --version
docker compose version
node --version
npm --version
```

Jeder Befehl muss eine Versionsnummer ausgeben. **Docker Desktop muss gestartet sein** (Wal-Symbol in der Menüleiste bzw. Taskleiste), sonst schlagen alle `docker`-Befehle fehl.

Außerdem braucht man den **Linus-Dump** (`dump.sql`). Wir nutzen den Dump vom 11.06.2025 der aktuellen Linus-Version (Django, Tabelle `accounts_user`) aus der Datensicherung des AG. Er enthält **echte personenbezogene Daten**, liegt deshalb **nicht** im Repository und wird nur teamintern über die Hessenbox weitergegeben. Bitte nirgendwo öffentlich ablegen oder weiterverteilen.

## Wo gebe ich die Befehle ein?

Alle Befehle in dieser Anleitung werden in einem **Terminal** eingegeben, am einfachsten im Terminal von VS Code:

1. VS Code öffnen, **File → Open Folder…** und den Ordner `VLAOrga` wählen (nach Schritt 1).
2. Menü **Terminal → New Terminal**. Unten öffnet sich ein Terminal, das schon im richtigen Ordner steht.
3. Befehl aus einem grauen Kasten kopieren, ins Terminal einfügen, **Enter** drücken. Warten, bis wieder eine Eingabezeile erscheint, dann den nächsten Befehl.

Welche Terminal-Art?

- **macOS:** Standard ist `zsh`. Passt, nichts ändern.
- **Linux:** Standard ist `bash`. Passt.
- **Windows:** Bitte **Git Bash** benutzen, **nicht** PowerShell oder CMD. Dann funktionieren alle Befehle genau wie hier geschrieben. In VS Code: rechts im Terminal-Bereich auf den kleinen Pfeil neben dem **+** klicken → **Git Bash**.

Ein zweites Terminal (für das Frontend in Schritt 6) öffnet man mit dem **+** im Terminal-Bereich.

Zeilen, die mit `#` beginnen, sind Kommentare und müssen nicht eingegeben werden. Platzhalter in spitzen Klammern wie `<name>` ersetzt man durch eigene Werte, ohne die Klammern.

## 1. Repository klonen

**Nur Windows, einmalig vorher:** Damit die Shell-Skripte unter `scripts/` nicht durch Windows-Zeilenenden kaputtgehen:

```bash
git config --global core.autocrlf input
```

Dann (alle Systeme), im Terminal in dem Ordner, in dem das Projekt liegen soll, z. B. Desktop:

```bash
git clone https://github.com/tpse-team-018/VLAOrga.git
cd VLAOrga
git checkout team018/lauffaehiger-code
cp .env.template .env
```

- `git clone` lädt das Repository herunter, `git checkout` wechselt auf unseren Team-Branch.
- `.env` enthält Einstellungen und Passwörter für die lokalen Datenbanken. Für die lokale Entwicklung können die Werte aus `.env.template` bleiben. `.env` wird **nicht** committet.
- **Der Ordnername muss `VLAOrga` bzw. `vlaorga` bleiben.** Docker leitet daraus die Containernamen ab (z. B. `vlaorga-db-1`, `vlaorga-linus-db-1`), die die Befehle hier und die Skripte in `scripts/` verwenden.

Alternative in VS Code ohne Terminal: in der Befehlspalette „Git: Clone“ wählen → URL einfügen → Zielordner wählen → „Open“. Danach links unten in der Statusleiste auf den Branch-Namen klicken und `origin/team018/lauffaehiger-code` wählen. `.env` dann trotzdem im Terminal mit `cp .env.template .env` anlegen.

Ab hier werden **alle Befehle im Ordner `VLAOrga`** ausgeführt (VS Code-Terminal nach „Open Folder“).

## 2. Datenbanken starten

```bash
docker compose up -d db linus-db
```

Startet die beiden Datenbank-Container im Hintergrund (`-d`). Beim ersten Mal lädt Docker die Images herunter, das dauert etwas.

Kommt ein Fehler `bind: address already in use` für Port 5432 oder 3306: siehe „Häufige Probleme“.

## 3. Linus-Dump einspielen

1. Ordner `backups` im Projektordner anlegen (falls nicht vorhanden) und den Dump dort ablegen, also `VLAOrga/backups/dump.sql`. Der Ordner steht in `.gitignore` und wird **nie** committet.
   ```bash
   mkdir -p backups
   ```
2. Ist der Dump gepackt (z. B. `dump.sql.xz`), zuerst entpacken.
3. Ca. 20 Sekunden nach Schritt 2 (die Datenbank braucht kurz zum Hochfahren) einspielen:

```bash
docker exec -i vlaorga-linus-db-1 mysql -u db_user -pdb_password db_name < backups/dump.sql
```

Das dauert einige Sekunden und gibt normalerweise nichts aus. Kontrolle:

```bash
docker exec vlaorga-linus-db-1 mysql -u db_user -pdb_password db_name -e "SHOW TABLES;"
```

Es sollten u. a. `accounts_user`, `experiment`, `reservation` und `reservation_experiment` erscheinen. Die Warnung `Using a password on the command line interface can be insecure` kann man ignorieren.

> Windows ohne Git Bash: Das `<` funktioniert in PowerShell nicht. Stattdessen:
> `cmd /c "docker exec -i vlaorga-linus-db-1 mysql -u db_user -pdb_password db_name < backups\dump.sql"`

## 4. Backend bauen und starten

```bash
docker compose up --build -d
```

Baut das Backend (Java) in einem Container und startet alle Container. Der erste Build dauert einige Minuten, weil Gradle alle Abhängigkeiten herunterlädt. Danach ca. 30 Sekunden warten und prüfen:

```bash
docker compose ps
docker compose logs --since 2m dev-app | tail -5
```

Erwartet:
- bei `docker compose ps` drei Container mit Status „Up“ (`db`, `linus-db`, `dev-app`)
- im Log die Zeile `Started VlaBackendApplication`

Steht dort stattdessen ein Fehler, siehe „Häufige Probleme“. Das ganze Log sieht man mit `docker compose logs dev-app`.

## 5. Benutzer anlegen

VLAOrga hat keine Registrierung, Benutzer werden per Skript angelegt (Backend muss laufen):

```bash
./scripts/addUserWithPassword.sh <name> <passwort>
```

Beispiel: `./scripts/addUserWithPassword.sh melda geheim123`

Erwartete Ausgabe am Ende: `INSERT 0 1`.

- `permission denied` (macOS/Linux): einmal `chmod +x scripts/*.sh` ausführen.
- Windows: nur in **Git Bash** ausführen. Erscheint `the input device is not a TTY`, den Befehl mit `winpty` davor ausführen.

## 6. Frontend starten

In einem **zweiten** Terminal (das **+** im VS Code-Terminal-Bereich), das erste darf offen bleiben:

```bash
cd VLA-Frontend
npm install
npm run dev
```

- `npm install` lädt die JavaScript-Abhängigkeiten (nur beim ersten Mal nötig bzw. nach Änderungen an `package.json`).
- `npm run dev` startet den Entwicklungsserver. **Dieses Terminal muss offen bleiben**, solange man VLAOrga benutzt. Beenden mit `Ctrl+C`.

Dann im **normalen Browser** (Safari, Chrome, Firefox; der in VS Code eingebaute Browser funktioniert nicht zuverlässig) öffnen:

**<http://localhost:5173/login.html>**

Mit dem Benutzer aus Schritt 5 anmelden. Unter der Überschrift steht „Entwicklungsinstanz – TPSE Team 018“. Danach erscheint die Wochenansicht des Kalenders.

## 7. Prüfen, ob die Verbindung zu Linus funktioniert

Nach dem Login im selben Browser öffnen:

<http://localhost:5173/api/linusExperiments/1>

Es erscheint ein Versuch aus der Linus-Datenbank als JSON (Text mit Name, Beschreibung usw.). Damit ist gezeigt: VLAOrga liest Daten aus Linus.

Im Kalender öffnet der Knopf **„Linus“** die „Linus Terminzuordnung“. Sie zeigt nur Linus-Buchungen **ab heute**. Da der Dump von Juni 2025 ist, ist die Liste dort zunächst leer. Das ist kein Fehler.

**Optional, nur zum lokalen Testen:** Die Buchungsdaten in der **lokalen Kopie** in die Zukunft verschieben (ändert nichts an der echten Linus-Datenbank):

```bash
docker exec vlaorga-linus-db-1 mysql -u db_user -pdb_password db_name -e "UPDATE reservation SET lecture_date = lecture_date + INTERVAL 75 WEEK;"
```

Danach im Kalender „Linus“ öffnen: Es erscheinen Linus-Buchungen der nächsten Tage. Rückgängig machen: Dump erneut einspielen (Schritt 3).

## Stoppen und wieder starten

```bash
docker compose stop            # anhalten (Daten bleiben erhalten)
docker compose start           # wieder starten
docker compose down            # Container entfernen (Linus-Dump muss danach neu eingespielt werden, Schritt 3)
```

Frontend: im zweiten Terminal `Ctrl+C`, zum Starten wieder `npm run dev` im Ordner `VLA-Frontend`.

Nach einem Neustart des Rechners: Docker Desktop starten, dann `docker compose start` und `npm run dev`.

## Häufige Probleme

| Problem | Lösung |
|---|---|
| `Cannot connect to the Docker daemon` / `docker: command not found` | Docker Desktop ist nicht gestartet bzw. nicht installiert. Starten und ca. 1 Minute warten. |
| `bind: address already in use` für Port 5432 | Ein lokal installiertes PostgreSQL läuft. **macOS:** `sudo lsof -nP -iTCP:5432 -sTCP:LISTEN` zeigt es; bei EDB-Installation `sudo launchctl unload /Library/LaunchDaemons/postgresql-<Version>.plist` (nach einem Neustart des Macs erneut nötig). **Windows:** in der Diensteverwaltung den Dienst „postgresql-…“ beenden. **Linux:** `sudo systemctl stop postgresql`. Bei Port 3306 analog lokales MySQL/MariaDB stoppen. |
| `No such container: vlaorga-linus-db-1` | Ordner heißt nicht `VLAOrga`, oder Schritt 2 fehlt. Mit `docker ps` die tatsächlichen Namen prüfen. |
| Backend beendet sich mit `missing table [accounts_user]` | Linus-Dump fehlt oder hat das alte Schema (PHP-Linus, Tabelle `user`) → aktuellen Dump einspielen (Schritt 3), dann `docker compose restart dev-app`. |
| „Verbindungsfehler“ im Kalender | Nicht angemeldet → <http://localhost:5173/login.html> öffnen und einloggen. |
| `http://localhost:5173` zeigt nichts | Frontend-Terminal geschlossen oder `npm run dev` nicht gestartet (Schritt 6). |
| Skript: `bad interpreter` / `\r: command not found` (Windows) | Windows-Zeilenenden. `git config --global core.autocrlf input` setzen, Repository neu klonen. |
| Linux: `permission denied` bei `docker` | Benutzer zur Gruppe `docker` hinzufügen: `sudo usermod -aG docker $USER`, dann ab- und wieder anmelden. |
| `package-lock.json` oder `.DS_Store` nach `npm install` geändert | Nicht committen: `git restore VLA-Frontend/package-lock.json .DS_Store` |

## Was nie committet werden darf

- `backups/` (Linus-Dump mit echten Daten)
- `.env` (lokale Passwörter)

Beides steht in `.gitignore`. Vor jedem Commit trotzdem `git status` prüfen.

## Unsere Änderung für den Meilenstein

Die Login-Seite (`VLA-Frontend/public/login.html`) zeigt unter der Überschrift den Hinweis „Entwicklungsinstanz – TPSE Team 018“. So ist sofort sichtbar, dass die selbst gebaute Version läuft.
