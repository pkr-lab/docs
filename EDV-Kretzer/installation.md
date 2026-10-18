# Installation

## Lokal ansehen

Die Seite braucht keinen Build. Sie muss über einen Webserver geöffnet werden, weil `js/apps.js` und `js/updates.js` JSON-Dateien per `fetch` laden; per Doppelklick auf eine HTML-Datei funktioniert das nicht.

```bash
git clone https://github.com/pkr-lab/EDV-Kretzer.git
cd EDV-Kretzer
python3 -m http.server 8000
```

Dann <http://localhost:8000/> öffnen.

## Deployment auf netcup

Jeder Push auf `main` startet `.github/workflows/deploy.yml`. Der Workflow verbindet sich per SSH mit dem Webserver, wechselt in das Docroot und führt `git pull --ff-only` aus. Dafür sind im Repo diese GitHub-Secrets nötig:

| Secret | Inhalt |
|---|---|
| `NETCUP_HOST` | Hostname des Servers |
| `NETCUP_USER` | SSH-Benutzer |
| `NETCUP_SSH_KEY` | privater SSH-Schlüssel |
| `NETCUP_PORT` | SSH-Port, Standard `22` |
| `NETCUP_PATH` | Pfad zum Docroot mit dem Git-Checkout |

Auf dem Server muss das Docroot ein Klon des Repos sein. Manuell startet man das Deployment über „Run workflow“ (`workflow_dispatch`).
