# Nutzung

## Inhalte pflegen

| Aufgabe | Vorgehen |
|---|---|
| Neue Anwendung im Katalog | Eintrag in `data/apps.json` ergänzen (Felder siehe [Konfiguration](konfiguration.md)) |
| Neuigkeit veröffentlichen | entfällt, der Verlauf auf `updates.html` kommt automatisch aus den Releases von capulus-core |
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

**Schalter im Cluster (n8n):** Der n8n-Workflow „EDV-Kretzer – Status-Hinweis schalten“ (`automation/n8n-status-schalten.json`) stellt ein Formular bereit: Aktion (Wartung, Störung, Hinweis einschalten oder ausschalten), betroffene Bereiche (Mehrfachauswahl), weitere Bereiche als Freitext (z. B. „Pi-hole, Grafana“, Trenner Komma, Semikolon oder Zeilenumbruch, höchstens 12 Einträge insgesamt), Text und Dauer (offen, 1 Stunde bis 3 Tage). Nach dem Absenden ist der Banner nach etwa einer Minute auf der Website. Die Formular-URL zeigt der Knoten „Formular“ in n8n (Production URL).

Einrichtung einmalig in n8n (https://n8n.prod.homeserver):
1. Workflow aus `automation/n8n-status-schalten.json` importieren.
2. Die Zugangsdaten „GitHub PAT (EDV-Kretzer)“ verwenden (Typ Header Auth, Header `Authorization: Bearer <TOKEN>`). Der Token braucht für das Repo `pkr-lab/EDV-Kretzer` die Berechtigung **Actions: Read and write**. Hat der vorhandene Token die nicht, einen neuen fine-grained Token anlegen und im Zugangsdaten-Eintrag ersetzen.
3. Workflow aktivieren. Die Bereichsliste steht im Knoten „Formular“ und lässt sich dort ändern.

Der Schalter ist nur über n8n erreichbar, also im Heimnetz oder per Tailscale. Schalten Sie den Hinweis ein, bevor Sie am Cluster arbeiten: Fällt n8n selbst aus, geht es nicht mehr. Der Banner liegt bewusst auf dem Webserver bei netcup und nicht im Cluster, damit er auch bei einem Ausfall sichtbar bleibt.

**Ein- und Ausschalten per GitHub:** Im Repo EDV-Kretzer unter Actions den Workflow „Status-Hinweis“ über „Run workflow“ starten (geht auch in der GitHub-App auf dem Handy). Der Workflow schreibt `data/status.json`, committet sie und deployt sofort.

| Eingabe | Bedeutung |
|---|---|
| `aktiv` | Haken gesetzt: Banner an. Ohne Haken: Banner aus |
| `art` | `maintenance` (gelb, „Wartung“), `incident` (rot, „Störung“), `info` (blau, „Hinweis“) |
| `titel` | optional, leer = Standardtitel der Art |
| `bereiche` | optional, durch Komma getrennt, erscheint im Banner als „Betroffen: …“ |
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
