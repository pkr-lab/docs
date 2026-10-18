# Nutzung

## Inhalte pflegen

| Aufgabe | Vorgehen |
|---|---|
| Neue Anwendung im Katalog | Eintrag in `data/apps.json` ergänzen (Felder siehe [Konfiguration](konfiguration.md)) |
| Neuigkeit veröffentlichen | Eintrag in `data/updates.json` ergänzen oder den n8n-Workflow auslösen |
| Text ändern | die jeweilige HTML-Datei bearbeiten; `<title>` und `<meta name="description">` mit anpassen |
| Neue Seite anlegen | HTML-Datei nach dem Muster einer vorhandenen Seite erstellen, in der Navigation und in `sitemap.xml` und `sitemap.html` eintragen |
| Bild einfügen | nach `assets/images/` legen, mit `alt`-Text einbinden |

## Ablauf einer Änderung

```bash
git checkout -b feat/neue-seite
python3 -m http.server 8000
git commit -m "feat(site): neue Seite hinzufügen"
```

Nach dem Merge nach `main` deployt GitHub Actions automatisch. Der Status der Dienste auf `updates.html` kommt live aus dem Browser und braucht kein Deployment.

## Typische Fehler

| Symptom | Ursache |
|---|---|
| Katalog oder Changelog bleibt leer | Seite per `file://` geöffnet; über `python3 -m http.server` starten |
| Status-Widget zeigt „nicht abrufbar“ | Status-Page nicht erreichbar oder Slug falsch |
| Deployment schlägt fehl | Secret fehlt oder das Docroot auf dem Server hat lokale Änderungen, `git pull --ff-only` bricht ab |
