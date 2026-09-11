# Nachweis-Tool

Manipulationssicheres Anwesenheitsprotokoll für eCampus-Zeiterfassung —
automatisierte Checks per GitHub Actions, Desktop-App und mobile
Weboberfläche, Hash-Chain-basiert.

## Download

Aktuelle Version unter **[Releases](../../releases/latest)** — dort liegen
`nachweis_tool.py`, `nachweis_app.py`, `app.py`, `kursplan.json` (als
Vorlage) und der komplette restliche Code als Anhang.

Die App selbst zeigt automatisch einen Hinweis an, sobald eine neuere
Version veröffentlicht wurde — du musst also nicht aktiv nachschauen.
Wer zusätzlich eine GitHub-Benachrichtigung bei neuen Releases möchte:
oben auf **"Watch" → "Custom"** klicken, Haken bei **"Releases"** setzen.

## Eigenes Backend einrichten

Diese Anleitung führt dich einmalig durch die Einrichtung deiner eigenen,
unabhängigen Instanz des Nachweis-Tools. Du bekommst dabei zwei eigene
private GitHub-Repos und einen eigenen kostenlosen Render-Service — nichts
davon hat Zugriff auf die Daten von irgendjemand anderem, der dieses Tool
ebenfalls nutzt.

Dauer: ca. 30–45 Minuten beim ersten Mal. Kostenlos.

### Was du am Ende hast

- Repo 1 ("Zeitnachweis" o.ä.): deine Log-Daten, dein Kursplan, die GitHub
  Action für die automatischen eCampus-Checks.
- Repo 2 ("zeitnachweis-backend" o.ä.): der Server-Code.
- Einen Render-Service, der Repo 2 ausführt.
- Eine Backend-URL und einen API-Key, die du am Ende in `nachweis_app.py`
  bzw. im Web-Frontend einträgst.

### Phase 1 — Repo für deine Daten anlegen

1. Auf **github.com** einloggen, oben rechts **"+" → "New repository"**.
2. Namen vergeben (z.B. `Zeitnachweis`), Sichtbarkeit **Private**.
3. Aus dem [aktuellen Release](../../releases/latest) herunterladen und
   hochladen (Add file → Upload files): `nachweis_tool.py`,
   `kursplan.json`, `.gitignore`.
   **Achtung:** GitHub zeigt die `.gitignore` im Release als
   `default.gitignore` an (Eigenheit bei Dateien, die nur aus einem Punkt
   plus Namen bestehen) — nach dem Download zurück in `.gitignore`
   umbenennen, bevor du sie hochlädst.
4. Drei Workflow-Dateien brauchen einen eigenen Unterordner, den "Upload
   files" nicht automatisch anlegt: für jede einzeln **Add file → Create
   new file**, als Dateiname jeweils exakt eintippen (mit den
   Schrägstrichen — GitHub legt die Ordner dabei automatisch an), Inhalt
   einfügen, committen:
   - `.github/workflows/nachweis-log.yml` — der automatische eCampus-Check
   - `.github/workflows/verify-log.yml` — manuelle Hash-Chain-Prüfung
   - `.github/workflows/nachweis-toggle.yml` — die echte Kommen/Gehen-Buchung
5. Passe `kursplan.json` an deinen eigenen Kursplan an (Kursnummern, Titel,
   Zeiträume aus deinem eCampus-Kursplan).

### Phase 2 — Repo für den Server-Code anlegen

1. Neues, **zweites** privates Repo (z.B. `zeitnachweis-backend`).
2. `app.py` und `requirements.txt` aus dem Release per "Upload files"
   hochladen (liegen flach im Root, kein Unterordner nötig).

### Phase 3 — GitHub-Zugangstoken erstellen

1. **github.com → Settings (dein Profil, nicht das Repo) → Developer settings
   → Personal access tokens → Fine-grained tokens → Generate new token.**
2. Token name: frei wählbar, z.B. `nachweis-backend`.
3. Expiration: "No expiration" ist hier vertretbar, weil der Token gleich
   eng eingeschränkt wird.
4. Repository access: **"Only select repositories"** → dein Repo aus
   Phase 1 (die Daten, NICHT das Backend-Repo) auswählen.
5. Permissions → Contents: **"Read and write"**.
6. Zusätzlich Actions: **"Read and write"** — wird für den "Jetzt prüfen"-
   und den "Kommen/Gehen"-Button gebraucht, die beide einen Workflow per
   API anstoßen.
7. **Generate token**, Wert sofort kopieren und sicher speichern (Passwort-
   Manager) — wird nur einmal angezeigt.

### Phase 4 — Backend auf Render deployen

1. Auf **render.com** einloggen bzw. registrieren.
2. **New → Web Service**, dein Backend-Repo aus Phase 2 auswählen. Falls es
   in der Liste fehlt: "Configure GitHub App" bzw. bei den GitHub-Settings
   unter Installed GitHub Apps → Render → Configure → Repository access
   erweitern.
3. Language: Python 3. Build Command: `pip install -r requirements.txt`.
   Start Command: `gunicorn app:app`.
4. Instance Type: **Free** auswählen (nicht die vorausgewählte kostenpflichtige
   Stufe).
5. Bei Environment Variables drei Einträge anlegen:
   - `GITHUB_TOKEN` → der Token aus Phase 3
   - `GITHUB_REPO` → `DeinGitHubName/DeinDatenRepo` (aus Phase 1)
   - `NACHWEIS_API_KEY` → über den "Generate"-Link rechts daneben einen
     zufälligen Wert erzeugen lassen
6. **Deploy web service.** Dauert beim ersten Mal 1–2 Minuten.
7. Test: `https://dein-service-name.onrender.com/health` im Browser öffnen —
   sollte `{"status": "ok"}` zeigen.

### Phase 5 — Secrets im Daten-Repo hinterlegen

Im Repo aus Phase 1 → **Settings → Secrets and variables → Actions → New
repository secret**, insgesamt fünf Stück:

- `ECAMPUS_URL`, `ECAMPUS_EMAIL`, `ECAMPUS_PASSWORD` — deine eigenen
  eCampus-Zugangsdaten
- `NACHWEIS_BACKEND_URL` — die Render-URL aus Phase 4
- `NACHWEIS_API_KEY` — derselbe Wert wie bei Render in Phase 4

### Phase 6 — Testen

1. Im Daten-Repo → **Actions → "eCampus Nachweis-Logger" → Run workflow.**
2. Läuft er grün durch, sollte kurz danach eine `nachweis_log.jsonl` mit
   einem neuen Eintrag im Repo auftauchen (committet vom Backend, nicht von
   der Action selbst).

### Phase 6b — Automatischen Zeitplan einrichten

**Wichtig:** Dieser Workflow läuft absichtlich NICHT über GitHubs eigenen
`schedule`-Trigger — der garantiert keine pünktliche Ausführung und kann bei
hoher globaler Last um Stunden verzögert sein. Stattdessen stößt ein
kostenloser externer Dienst den Workflow zuverlässig per API an:

1. Auf **cron-job.org** registrieren (kostenlos).
2. Neuen Cronjob anlegen:
   - URL: `https://dein-service-name.onrender.com/trigger-check`
   - Methode: `POST`
   - Custom Header: `X-API-Key: <dein NACHWEIS_API_KEY>`
   - Zeitplan: stündlich, Mo–Fr, im gewünschten Zeitfenster (z.B. 08:30–19:30),
     Zeitzone direkt auf Europe/Berlin stellen — kein Umrechnen auf UTC nötig.
3. Speichern. Ab jetzt läuft der Check zuverlässig zur gewünschten Zeit,
   weil die Anfrage technisch wie ein manueller "Run workflow"-Klick
   behandelt wird, nicht wie ein geplanter GitHub-Workflow.

### Phase 7 — App bzw. Weboberfläche verbinden

- **Desktop-App**: `nachweis_app.py` starten. Bei fehlender Konfiguration
  öffnet sich automatisch die Einrichtung — dort URL aus Phase 4 und Key
  aus Phase 4/5 eintragen.
- **Weboberfläche**: die Render-URL selbst im Browser öffnen (bzw. am
  iPhone über "Zum Home-Bildschirm hinzufügen"), dort fehlt dann nur noch
  der API-Key.

### Falls die eCampus-Selektoren nicht passen

Die in `nachweis_tool.py` hinterlegten Klick-Ziele wurden für die COMCAVE-
eCampus-Oberfläche ermittelt und sollten für jeden COMCAVE-Standort
funktionieren, da alle dieselbe Plattform nutzen. Falls dein Login-Ablauf
abweicht: `pip install playwright`, `python -m playwright install chromium`,
dann `python -m playwright codegen <deine-eCampus-URL>` — zeigt live die
passenden Selektoren für deinen Login-Flow an.

### Optional — Backend wach halten

Render legt kostenlose Services nach ca. 15 Minuten Inaktivität schlafen;
der nächste Aufruf danach braucht ein paar Sekunden länger (Kaltstart).
Kostenlose Abhilfe: bei **uptimerobot.com** registrieren, neuen HTTP(s)-
Monitor auf `https://dein-service-name.onrender.com/health` mit 5-Minuten-
Intervall anlegen. Nicht nötig für die Grundfunktion, macht "Jetzt prüfen"
aber spürbar schneller.
