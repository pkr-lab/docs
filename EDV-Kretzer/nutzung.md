# Nutzung

## Inhalte pflegen

| Aufgabe | Vorgehen |
|---|---|
| Neue Anwendung im Katalog | Eintrag in `data/apps.json` ergänzen (Felder siehe [Konfiguration](konfiguration.md)) |
| Neuigkeit veröffentlichen | Eintrag in `data/updates.json` ergänzen oder den n8n-Workflow auslösen |
| Text ändern | die jeweilige HTML-Datei bearbeiten; `<title>` und `<meta name="description">` mit anpassen |
| Neue Seite anlegen | HTML-Datei nach dem Muster einer vorhandenen Seite erstellen, in der Navigation und in `sitemap.xml` und `sitemap.html` eintragen |
| Bild einfügen | nach `assets/images/` legen, mit `alt`-Text einbinden |
| Wartungs- oder Störungshinweis | Workflow „Status-Hinweis“ starten, siehe unten |

## Ablauf einer Änderung

```bash
git checkout -b feat/neue-seite
python3 -m http.server 8000
git commit -m "feat(site): neue Seite hinzufügen"
```

Nach dem Merge nach `main` deployt GitHub Actions automatisch. Der Status der Dienste auf `updates.html` kommt live aus dem Browser und braucht kein Deployment.

## Status-Hinweis (Wartung, Störung)

Ein Banner oben auf jeder Seite zeigt, dass gerade am Server oder an der Website gearbeitet wird. Er liest `data/status.json` und erscheint nur, wenn dort `"active": true` steht.

**Ein- und Ausschalten:** Im Repo EDV-Kretzer unter Actions den Workflow „Status-Hinweis“ über „Run workflow“ starten (geht auch in der GitHub-App auf dem Handy). Der Workflow schreibt `data/status.json`, committet sie und deployt sofort.

| Eingabe | Bedeutung |
|---|---|
| `aktiv` | Haken gesetzt: Banner an. Ohne Haken: Banner aus |
| `art` | `maintenance` (gelb, „Wartung“), `incident` (rot, „Störung“), `info` (blau, „Hinweis“) |
| `titel` | optional, leer = Standardtitel der Art |
| `text` | Meldungstext |
| `bis` | optional, Zeitpunkt wie `2026-10-21T18:00:00+02:00`. Danach blendet sich der Banner von selbst aus |

Per Kommandozeile:

```bash
gh workflow run status.yml -R pkr-lab/EDV-Kretzer -f aktiv=true -f art=maintenance -f text="Wartungsarbeiten, kurze Ausfälle möglich." -f bis=2026-10-21T18:00:00+02:00
gh workflow run status.yml -R pkr-lab/EDV-Kretzer -f aktiv=false
```

Der Zeitpunkt „Seit“ setzt der Workflow selbst. Der Text wird als reiner Text angezeigt, HTML darin hat keine Wirkung. Andere Automatisierungen (n8n, Skripte) können denselben Aufruf nutzen. Die Seiten laden die Datei bei jedem Aufruf neu, ein Browser-Cache verzögert den Banner nicht.

## Typische Fehler

| Symptom | Ursache |
|---|---|
| Katalog oder Changelog bleibt leer | Seite per `file://` geöffnet; über `python3 -m http.server` starten |
| Status-Widget zeigt „nicht abrufbar“ | Status-Page nicht erreichbar oder Slug falsch |
| Deployment schlägt fehl | Secret fehlt, der SSH-Schlüssel steht nicht in `authorized_keys` oder der Git-Spiegel `~/git/EDV-Kretzer` ist beschädigt ([installation.md](installation.md#deployment-auf-netcup)) |
