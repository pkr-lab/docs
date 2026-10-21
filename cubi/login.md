# Login

Login-Status für ArgoCD, GitHub und Tailscale auf einen Blick, plus die passenden Login-Befehle.

Code: [modules/auth.py](https://github.com/pkr-lab/cubi/blob/main/src/cubi/modules/auth.py), [argocd_config.py](https://github.com/pkr-lab/cubi/blob/main/src/cubi/argocd_config.py)

---

## Überblick

`cubi login` (`auth`) zeigt eine Tabelle mit je einer Zeile pro ArgoCD-Kontext aus der Config, einer Zeile für GitHub und einer für Tailscale. GitHub und Tailscale werden parallel abgefragt. Enter auf einer Zeile öffnet das passende Login-Menü. cubi speichert nichts selbst: Zugangsdaten verwalten `argocd`, `gh` und `tailscale`.

| Zeile | Quelle | Statusmeldungen |
|---|---|---|
| ArgoCD · `<kontext>` | `~/.config/argocd/config` (Token wird dekodiert, nicht geprüft) und `argocd.contexts[]` | `Core-Modus, kein Login nötig`, `gültig bis <zeit>`, `eingeloggt` (Token ohne Ablauf), `abgelaufen seit <zeit>`, `Token ungültig`, `nicht eingeloggt` |
| GitHub · github.com | `gh auth status -h github.com` | `eingeloggt als <konto>`, `nicht eingeloggt` |
| Tailscale · `<tailnet>` | `tailscale status --json`, `BackendState` | `verbunden` (Running), `Login nötig` (NeedsLogin), `gestoppt` (Stopped), sonst der Rohwert |

Die ArgoCD-Prüfung ist rein lokal: cubi liest das JWT aus der ArgoCD-Konfiguration und vergleicht das Ablaufdatum mit der Uhrzeit. Ob der Server das Token noch akzeptiert, prüft cubi nicht. Nur der Core-Modus (`core: true`) braucht keinen Login.

---

## Menüs

| Zeile | Einträge |
|---|---|
| ArgoCD (nicht Core) | *Login mit Passwort* (`argocd login <server> [--plaintext] [--insecure] [--name <name>] [--username <name>]`, gemäß Config), *Kontext aktivieren* (`argocd context <name>`) |
| ArgoCD (Core) | nur *Kontext aktivieren* |
| GitHub | *Login (Code im Terminal)* (`gh auth login -h github.com`), *Scopes erneuern* (`gh auth refresh`), *Projekt-Scope aktivieren (für Kanban-Board)* (`gh auth refresh -s project,read:project`) |
| Tailscale | `tailscale up`, `tailscale login`; bei Berechtigungsfehlern mit `sudo` wiederholen |

Die GitHub-Befehle laufen mit `BROWSER=true` und `GH_BROWSER=true`, damit kein Browser aufgeht. Der Code erscheint im Terminal, einzugeben unter `https://github.com/login/device`. SSO-Login für ArgoCD gibt es in cubi nicht (es würde einen Browser öffnen), nur Passwort-Login.

---

## Debugging und Fehlersuche

| Meldung | Ursache | Lösung |
|---|---|---|
| `abgelaufen seit <zeit>` | ArgoCD-Token abgelaufen | Zeile öffnen → *Login mit Passwort* |
| `Token ungültig` | JWT in `~/.config/argocd/config` nicht lesbar | neu einloggen |
| `nicht eingeloggt` (ArgoCD) | kein Kontext mit diesem Namen oder kein `auth-token` | *Login mit Passwort*; `name` und `server` in der Config prüfen: `argocd login` legt den Kontext unter `--name` an (wenn `server` ≠ `name`) |
| `Unbekannter ArgoCD-Kontext: <name>` | Kontext fehlt in `argocd.contexts[]` | Config ergänzen, siehe [Konfiguration](konfiguration.md) |
| `Kontext <name> konnte nicht aktiviert werden` | Kontext in `~/.config/argocd/config` existiert nicht (noch nicht eingeloggt) | erst *Login mit Passwort*, dann *Kontext aktivieren* |
| `Ungültiger Eintrag in argocd.contexts: name ist Pflicht` | Kontext ohne `name` | ergänzen |
| `nicht eingeloggt` (GitHub) | `gh auth status` endet mit Fehler | *Login (Code im Terminal)* |
| `gh ist nicht installiert oder nicht im PATH` | `gh` fehlt | installieren |
| `Login nötig` (Tailscale) | Anmeldung fehlt oder abgelaufen | *tailscale login* |
| `gestoppt` (Tailscale) | `tailscale down` ausgeführt | *tailscale up* |
| `Status nicht lesbar` | `tailscale status --json` lieferte keinen JSON (Fehlertext folgt) | Befehl direkt ausführen, `tailscaled` prüfen |
| `Zeitüberschreitung bei: gh auth status -h github.com` oder `tailscale status --json` | Netzwerk oder Dienst hängt (15 Sekunden) | Verbindung prüfen |
| `argocd app diff` meldet `Hinweis: Login abgelaufen oder fehlt. Im Hauptmenü unter Login erneuern.` | ArgoCD-Fehlertext enthält `unauthenticated`, `token is expired`, `expired`, `please log in` oder `login` | hier neu einloggen |

Checkliste „Login-Bereich zeigt Fehler“:

1. Zeile öffnen und den Login-Befehl ausführen, bis die Zeile grün ist.
2. ArgoCD: `argocd context` in der Shell, stimmt der aktive Kontext mit der Tabelle?
3. GitHub: `gh auth status` zeigt Konto und Scopes.
4. Tailscale: `tailscale status`.

Allgemeines: [Debugging und Fehlersuche](debugging.md).
