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

### Secrets anlegen

Stand 2026-10-19 sind im Repo keine Secrets hinterlegt, der Lauf endet mit `missing server host`. Mit der GitHub-CLI (als Repo-Admin angemeldet) fragt jeder Befehl den Wert interaktiv ab, er landet weder in der Shell-Historie noch im Repo:

```bash
gh secret set NETCUP_HOST -R pkr-lab/EDV-Kretzer
gh secret set NETCUP_USER -R pkr-lab/EDV-Kretzer
gh secret set NETCUP_PATH -R pkr-lab/EDV-Kretzer
gh secret set NETCUP_PORT -R pkr-lab/EDV-Kretzer
gh secret set NETCUP_SSH_KEY -R pkr-lab/EDV-Kretzer < ~/.ssh/<deploy-key>
```

Für `NETCUP_SSH_KEY` ein eigenes Schlüsselpaar nur für diesen Zweck erzeugen (`ssh-keygen -t ed25519 -f ~/.ssh/netcup-deploy -C edv-kretzer-deploy`), den öffentlichen Teil in `~/.ssh/authorized_keys` des netcup-Benutzers eintragen und den privaten Teil als Secret speichern. `NETCUP_PORT` kann entfallen, wenn der Port 22 ist. `NETCUP_PATH` ist das Docroot mit dem Git-Checkout, ohne abschließenden Schrägstrich. Danach den Workflow über „Run workflow“ prüfen.
