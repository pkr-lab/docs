# cubi

Interactive terminal tool for everyday Kubernetes work. It bundles tools you already have installed into one menu that you navigate with the arrow keys.

```
cubi
```

<img src="pics/menu.png" alt="cubi main menu" width="600">

## Sections

| Section | Contents |
| --- | --- |
| Connect | SSH to VMs and hosts, grouped as defined in the config |
| ArgoCD | Table of all apps with sync and health status, filters, sync, diff, refresh, link to the GitHub source |
| Tailscale | Device list with status, ping, SSH, up and down |
| GitHub | Workflow runs, pull requests, issues (incl. editing) and commits of a repo, plus the repo's project Kanban board |
| Kubernetes | Pods with logs and shell, nodes, warnings, switching namespace and context |
| Helm | Releases with status, values and history |
| Tokens | Age, expiry and rotation of all tokens in capulus-core's token register (`ops/token-register.yaml`) |
| Login | Login status for ArgoCD, GitHub and Tailscale, plus the matching login commands |

Every section can be opened directly, for example `cubi argocd` or `cubi login`.

The ArgoCD view lists all apps with sync and health status. Apps with problems are sorted to the top, and the table refreshes automatically.

<img src="pics/argocd.png" alt="ArgoCD app table with sync and health status" width="500">

Repository: [pkr-lab/cubi](https://github.com/pkr-lab/cubi)

## Architektur

cubi ist ein Python-3-Programm ohne Server. `bin/cubi` startet `cubi.main`, das einen Abschnitt (Modul) auswählt; jedes Modul ruft die vorhandenen Werkzeuge (`kubectl`, `argocd`, `gh`, `tailscale`, `helm`, `ssh`) als Unterprozess auf und stellt die Ausgabe mit [rich](https://github.com/Textualize/rich) als Tabelle oder Menü dar. cubi speichert keine Zugangsdaten.

| Modul (`src/cubi/modules/`) | Aufruf | Aufgabe |
|---|---|---|
| `connect.py` | `cubi connect` | SSH-Verbindung zu VMs und Hosts, gruppiert nach `connect` in der Konfiguration |
| `argocd.py` | `cubi argocd` (`argo`) | Tabelle aller Apps mit Sync- und Health-Status, Filter, Sync, Diff, Refresh; Status direkt per `kubectl` |
| `tailscale.py` | `cubi tailscale` (`ts`) | Geräteliste, Ping, SSH, `up` und `down` |
| `github.py` | `cubi github` (`gh`) | Workflow-Läufe, Pull Requests, Issues und Commits eines Repos, Kanban-Board |
| `kube.py` | `cubi kubernetes` (`k8s`, `kubectl`) | Pods mit Logs und Shell, Nodes, Warnungen, Namespace- und Kontextwechsel |
| `helm.py` | `cubi helm` | Releases mit Status, Values und Verlauf |
| `tokens.py` | `cubi tokens` (`token`) | Alter, Ablauf und Rotation der Tokens aus dem Token-Register von capulus-core |
| `auth.py` | `cubi login` (`auth`) | Login-Status für ArgoCD, GitHub und Tailscale samt Login-Befehlen |

Gemeinsame Bausteine: `ui.py` (Tabellen, Menüs, Eingaben), `shell.py` (Unterprozesse), `config.py` (Konfiguration), `token_register.py` (Register lesen und fortschreiben), `argocd_config.py`, `timeutil.py`.

Beziehung zu capulus-core: Das Modul `tokens` liest `ops/token-register.yaml` und führt `scripts/check-token-register.py` im Checkout von [capulus-core](../capulus-core/index.md) aus. Runbooks mit dem Präfix `pkr-lab-docs:` öffnet es aus dem Checkout dieses Docs-Repos.

## Seiten

- [Installation](installation.md)
- [Konfiguration](konfiguration.md)
- [Nutzung](nutzung.md)
- [Releases](releases.md)
