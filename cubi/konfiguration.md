# Konfiguration

Code: [config.py](https://github.com/pkr-lab/cubi/blob/main/src/cubi/config.py), [config/targets.yaml](https://github.com/pkr-lab/cubi/blob/main/config/targets.yaml)

Hosts, ArgoCD-Kontexte, GitHub-Repos und die Checkouts für das Token-Register stehen in `targets.yaml`. cubi sucht die Datei in dieser Reihenfolge und nimmt die erste, die es als Datei gibt:

1. die Datei aus der Umgebungsvariable `CUBI_CONFIG` (mit `~` erlaubt),
2. `~/.config/cubi/targets.yaml`,
3. `config/targets.yaml` im Repo.

Gibt es keine, bricht cubi mit `Keine targets.yaml gefunden` ab. Die Datei wird einmal je Programmlauf gelesen, Änderungen greifen beim nächsten Start. Fehlt ein Abschnitt, ist er leer: Bereiche, die ihn brauchen, melden das selbst (zum Beispiel `Keine Connect-Ziele in targets.yaml`).

Für eigene Werte empfiehlt sich eine Kopie in `~/.config/cubi/targets.yaml`, damit `git pull` sie nicht überschreibt.

---

## Aufbau von `targets.yaml`

| Schlüssel | Typ | Bedeutung | Standard |
|---|---|---|---|
| `connect.<gruppe>[]` | Liste | Einträge mit `name` und `ssh`. Gruppenname beliebig (`vm`, `hosts`); Namen mit höchstens 3 Zeichen erscheinen großgeschrieben (`vm` → `VM`), sonst nur mit großem Anfangsbuchstaben | – |
| `connect.<gruppe>[].name` | Text | Anzeigename im Menü | Pflicht |
| `connect.<gruppe>[].ssh` | Text | SSH-Alias aus `~/.ssh/config` oder Hostname | Pflicht |
| `argocd.namespace` | Text | Namespace der ArgoCD-Installation, auch für die ConfigMap `cubi-state` | `argocd` |
| `argocd.contexts[]` | Liste | ArgoCD-Zugänge für `cubi login` und die Kontextwahl. Ein Eintrag darf auch nur ein String sein (dann gilt er als `name`) | – |
| `argocd.contexts[].name` | Text | Kontextname | Pflicht |
| `argocd.contexts[].server` | Text | Adresse für `argocd login` | gleich `name` |
| `argocd.contexts[].core` | bool | `true`: Kubernetes-Backend von ArgoCD, kein Login nötig | `false` |
| `argocd.contexts[].plaintext` | bool | `--plaintext` beim Login (HTTP statt HTTPS) | `false` |
| `argocd.contexts[].insecure` | bool | `--insecure` beim Login (Zertifikat nicht prüfen) | `false` |
| `argocd.contexts[].username` | Text | `--username` beim Login | leer |
| `github.default_repo` | Text | Repo, das `cubi github` direkt öffnet (`owner/name`) | leer: Auswahl beim Start |
| `github.repos[]` | Liste | Repos in der Auswahl; `default_repo` wird bei Bedarf vorangestellt | – |
| `github.board.owner`, `github.board.number` | Text, Zahl | festes Kanban-Board; fehlt es, wählt man das Board aus `gh project list` des Repo-Eigentümers | – |
| `tokens.repo` | Pfad | Checkout von capulus-core mit `ops/token-register.yaml` | `../capulus-core` neben dem cubi-Checkout |
| `tokens.docs_repo` | Pfad | Checkout von pkr-lab-docs für Runbooks mit dem Präfix `pkr-lab-docs:` | `../pkr-lab-docs` neben dem cubi-Checkout |

Die Standardwerte für `tokens.repo` und `tokens.docs_repo` beziehen sich auf den Ort des cubi-Checkouts (`config.ROOT.parent`), nicht auf das aktuelle Verzeichnis. Bei einem Symlink-Start aus `~/.local/bin` zählt der Ort, auf den der Symlink zeigt.

Beispiel (Werte sind Platzhalter):

```yaml
connect:
  vm:
    - name: vm-prod
      ssh: prod-vm
  hosts:
    - name: homeserver
      ssh: homeserver
    - name: worker-0
      ssh: worker-0

argocd:
  namespace: argocd
  contexts:
    - name: kubernetes
      core: true
    - name: homeserver:30080
      server: <host>:30080
      plaintext: true
      insecure: true
      username: admin

github:
  default_repo: pkr-lab/capulus-core
  repos:
    - pkr-lab/capulus-core
    - pkr-lab/cubi
  board:
    owner: pkr-lab
    number: 3

tokens:
  repo: ~/gitlab/capulus/capulus-core
  docs_repo: ~/gitlab/pkr-lab-docs
```

---

## SSH-Aliase

cubi verbindet mit `ssh <alias>`. Alias, Benutzer, Schlüssel und Jump-Hosts gehören in `~/.ssh/config`. Im Menü zeigt cubi hinter jedem Eintrag `benutzer@hostname`, aufgelöst mit `ssh -G <alias>`. Ist der Alias nicht definiert, steht dort der Hostname gleich dem Alias, und der Verbindungsaufbau scheitert wie bei einem normalen `ssh`. Siehe [Connect](connect.md).

---

## ArgoCD-Kontexte

ArgoCD-Login-Daten liegen in `~/.config/argocd/config`, die Konfiguration liefert nur die Liste der Zugänge. Für die Statusanzeige der App-Tabelle braucht cubi keinen ArgoCD-Login, es liest Applications direkt über `kubectl`. Der Login betrifft nur `argocd app diff`, `sync`, `history` und `get --refresh`. Siehe [Login](login.md) und [ArgoCD](argocd.md).

Ein Eintrag mit `core: true` nutzt ArgoCDs Core-Modus über den aktuellen `kubectl`-Kontext. Dann ist kein `argocd login` nötig, aber `kubectl` muss auf den Cluster zeigen.

---

## GitHub-Board

Das Kanban-Board nutzt das Feld „Status“ des GitHub-Projekts (oder das erste Single-Select-Feld mit Spalten). Karten verschieben und Labels oder Zuständige von Issues ändern braucht ein Token mit dem Scope `project`. Den ergänzt `cubi login` → GitHub → „Projekt-Scope aktivieren“ (`gh auth refresh -s project,read:project`). Siehe [GitHub](github.md).

---

## Token-Register

`tokens.repo` muss auf einen capulus-core-Checkout zeigen, in dem `ops/token-register.yaml` existiert. Das Register enthält keine Secrets, nur Metadaten und Daten. Aufbau und Felder: [Tokens](tokens.md).

---

## Umgebungsvariablen

| Variable | Wirkung |
|---|---|
| `CUBI_CONFIG` | Pfad zu einer anderen `targets.yaml`, hat Vorrang vor allen anderen Orten |
| `BROWSER=true`, `GH_BROWSER=true` | setzt cubi nur für `gh auth login/refresh` selbst, damit kein Browser aufgeht |
| `GIT_TERMINAL_PROMPT=0`, `GIT_ASKPASS=true`, `SSH_ASKPASS=true` | setzt cubi nur für `git ls-remote` bei der Branch-Prüfung, damit keine Passwortabfrage erscheint |

Alle anderen Variablen (`KUBECONFIG`, `ARGOCD_OPTS`, `GH_TOKEN`, …) erreichen die Werkzeuge unverändert, weil Unterprozesse die Umgebung erben.

---

## Fehlerbilder bei der Konfiguration

| Meldung | Ursache | Lösung |
|---|---|---|
| `Keine targets.yaml gefunden` | weder `CUBI_CONFIG` noch `~/.config/cubi/` noch `config/` enthält eine Datei | Datei anlegen oder `CUBI_CONFIG` auf eine existierende Datei setzen (zeigt `CUBI_CONFIG` ins Leere, wird er übersprungen und die nächste Stelle versucht) |
| `Ungültige Config <pfad>: …` | YAML-Syntaxfehler | Zeile aus der Meldung korrigieren, mit `python3 -c "import yaml,sys; yaml.safe_load(open(sys.argv[1]))" <pfad>` prüfen |
| `Ungültige Config <pfad>: oberste Ebene muss ein Mapping sein` | Datei ist eine Liste oder ein Text | Abschnitte (`connect:`, `argocd:`, …) als Mapping an die oberste Ebene |
| `Keine Connect-Ziele in targets.yaml` | Abschnitt `connect` fehlt oder leer | Gruppen mit Einträgen ergänzen |
| `Ungültiger Eintrag in connect.<gruppe>: name und ssh sind Pflicht` | Eintrag ohne `name` oder `ssh` | beide Schlüssel setzen |
| `Ungültiger Eintrag in argocd.contexts: name ist Pflicht` | Kontext ohne `name` | `name` ergänzen |

Mehr dazu in [Debugging und Fehlersuche](debugging.md).
