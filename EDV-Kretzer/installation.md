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

Jeder Push auf `main` startet `.github/workflows/deploy.yml`. Der Workflow verbindet sich per SSH mit dem netcup-Webhosting und führt zwei Schritte aus:

1. `git fetch` im Git-Spiegel `~/git/EDV-Kretzer` (ein Bare-Repository mit `mirror = true`, Quelle `https://github.com/pkr-lab/EDV-Kretzer.git`).
2. `git checkout -f main` mit dem Docroot als Arbeitsverzeichnis. Neue und geänderte Dateien werden geschrieben, gelöschte entfernt.

Das Docroot `/edv-kretzer.de/httpdocs` ist damit kein Klon und enthält kein `.git`. Dasselbe tut das Git-Werkzeug von Plesk. Ein `git pull` im Docroot funktioniert nicht, der frühere Workflow tat genau das und konnte nie laufen. Dafür sind im Repo diese GitHub-Secrets nötig:

| Secret | Inhalt |
|---|---|
| `NETCUP_HOST` | `hosting235033.ae9f6.netcup.net` |
| `NETCUP_USER` | `hosting235033` (SSH-Benutzer des Hostings, nicht die Kundennummer) |
| `NETCUP_SSH_KEY` | privater SSH-Schlüssel |
| `NETCUP_PORT` | SSH-Port, Standard `22` |
| `NETCUP_PATH` | Docroot, `/edv-kretzer.de/httpdocs` |

Der öffentliche Teil des Schlüssels steht in `~/.ssh/authorized_keys` des Hosting-Benutzers (nicht in einer eigenen `.pub`-Datei, SSH liest nur `authorized_keys`). Manuell startet man das Deployment über „Run workflow“ (`workflow_dispatch`).

### Secrets anlegen

Stand 2026-10-19 sind im Repo keine Secrets hinterlegt, der Lauf endete mit `missing server host`. Mit der GitHub-CLI (als Repo-Admin angemeldet) fragt jeder Befehl den Wert interaktiv ab, er landet weder in der Shell-Historie noch im Repo:

```bash
gh secret set NETCUP_HOST -R pkr-lab/EDV-Kretzer
gh secret set NETCUP_USER -R pkr-lab/EDV-Kretzer
gh secret set NETCUP_PATH -R pkr-lab/EDV-Kretzer
gh secret set NETCUP_SSH_KEY -R pkr-lab/EDV-Kretzer < ~/.ssh/netcup-deploy
```

Für `NETCUP_SSH_KEY` ein eigenes Schlüsselpaar nur für diesen Zweck erzeugen (`ssh-keygen -t ed25519 -f ~/.ssh/netcup-deploy -C edv-kretzer-deploy`), die Zeile aus `netcup-deploy.pub` an `~/.ssh/authorized_keys` auf dem Server anhängen (Rechte `600`) und den privaten Teil als Secret speichern. `NETCUP_PORT` entfällt bei Port 22. Vorher testen:

```bash
ssh -i ~/.ssh/netcup-deploy hosting235033@hosting235033.ae9f6.netcup.net
```

Danach den Workflow über „Run workflow“ starten.

### Stolperstellen

- Das Verzeichnis `.github` und `automation` liegen im Docroot und sind damit über die Website abrufbar. Inhalt: Workflows und der n8n-Workflow ohne Zugangsdaten. Bei Bedarf per `.htaccess` sperren.
- Zugangsdaten gehören nur in GitHub-Secrets, nie in den Spiegel oder das Docroot.
