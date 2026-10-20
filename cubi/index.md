# cubi

Interactive terminal tool for everyday Kubernetes work. It bundles tools you already have installed into one menu that you navigate with the arrow keys. Everything is drawn inline in the normal terminal, so the history stays visible.

```
cubi
```

<img src="pics/menu.png" alt="cubi main menu" width="600">

## Sections

| Section | Contents |
| --- | --- |
| Connect | SSH to VMs and hosts, grouped as defined in the config |
| ArgoCD | Table of all apps with sync and health status, filters, sync (also for several apps), diff, refresh, link to the GitHub source |
| Auto-Sync | Disable and re-enable ArgoCD auto-sync for several apps at once, remembering prune and selfHeal |
| Branch | Switch the target branch of several ArgoCD apps, or reset them to `main` |
| Pods skalieren | Set the pods of several ArgoCD apps to 0 and restore them later, remembering replicas and auto-sync |
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
| `argocd.py` | `cubi argocd` (`argo`) | Tabelle aller Apps mit Sync- und Health-Status, Filter, Sync (auch mehrere Apps), Diff, Refresh; Status direkt per `kubectl` |
| `autosync.py` | `cubi autosync` (`sync`) | Auto-Sync mehrerer Apps deaktivieren und mit den gemerkten Einstellungen wieder aktivieren, siehe [ArgoCD-Features](argocd-features.md) |
| `scale.py` | `cubi scale` | Deployments und StatefulSets mehrerer Apps auf 0 Replikas stellen und mit den gemerkten Werten wiederherstellen, siehe [ArgoCD-Features](argocd-features.md) |
| `branch.py` | `cubi branch` | `targetRevision` mehrerer Apps umstellen oder auf `main` zurücksetzen, siehe [ArgoCD-Features](argocd-features.md) |
| `tailscale.py` | `cubi tailscale` (`ts`) | Geräteliste, Ping, SSH, `up` und `down` |
| `github.py` | `cubi github` (`gh`) | Workflow-Läufe, Pull Requests, Issues und Commits eines Repos, Kanban-Board |
| `kube.py` | `cubi kubernetes` (`k8s`, `kubectl`) | Pods mit Logs und Shell, Nodes, Warnungen, Namespace- und Kontextwechsel |
| `helm.py` | `cubi helm` | Releases mit Status, Values und Verlauf |
| `tokens.py` | `cubi tokens` (`token`) | Alter, Ablauf und Rotation der Tokens aus dem Token-Register von capulus-core |
| `auth.py` | `cubi login` (`auth`) | Login-Status für ArgoCD, GitHub und Tailscale samt Login-Befehlen |

Gemeinsame Bausteine: `ui.py` (Fassade für Auswahl, Eingaben und Ausgabe), `choice.py` und `term.py` (inline gezeichnete Auswahl, siehe [Select-Komponente](select-komponente.md)), `shell.py` (Unterprozesse), `config.py` (Konfiguration), `token_register.py` (Register lesen und fortschreiben), `argocd_config.py`, `argocd_apps.py` (Apps lesen und patchen), `timeutil.py`.

cubi bleibt immer in der normalen Terminal-Ansicht: kein Vollbild, kein Löschen des Terminals, kein Pager, Editor oder Browser. Details unter [Bedienung](bedienung.md).

Beziehung zu capulus-core: Das Modul `tokens` liest `ops/token-register.yaml` und führt `scripts/check-token-register.py` im Checkout von [capulus-core](../capulus-core/index.md) aus. Runbooks mit dem Präfix `pkr-lab-docs:` öffnet es aus dem Checkout dieses Docs-Repos.

## Seiten

- [Installation](installation.md)
- [Konfiguration](konfiguration.md)
- [Nutzung](nutzung.md)
- [Bedienung](bedienung.md): Tasten, Einfach- und Mehrfachauswahl
- [ArgoCD-Features](argocd-features.md): Auto-Sync, Branch umstellen, Pods auf 0 stellen
- [Select-Komponente](select-komponente.md): Aufbau und Wiederverwendung im Code
- [Releases](releases.md)
