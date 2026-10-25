# pkr-lab-docs

Zentrale Dokumentation aller Projekte von pkr-lab. Jedes Projekt hat einen eigenen Ordner, dessen Name dem Repo entspricht. Die Code-Repos selbst enthalten nur eine kurze README mit Zweck, Quickstart und einem Link hierher.

## Projekte

| Projekt | Zweck | Repo | Doku |
|---|---|---|---|
| capulus-core | GitOps-Homelab: ArgoCD-Apps, Promotion ENTW → TECH → PROD, Token-Register | [GitHub](https://github.com/pkr-lab/capulus-core) | [capulus-core/](capulus/capulus-core/index.md) |
| capulus-ansible | Ansible-Playbooks und Rollen für Hosts, k3s-Cluster und Kiosks | [GitHub](https://github.com/pkr-lab/capulus-ansible) | [capulus-ansible/](capulus/capulus-ansible/index.md) |
| capulus-ios | SwiftUI-App „Homeserver Dashboard“ | [GitHub](https://github.com/pkr-lab/capulus-ios) | [capulus-ios/](capulus/capulus-ios/index.md) |
| capulus-operator | Kubernetes-Operator: Backups, Datenbanken, Zertifikate, Secret-Rotation | [GitHub](https://github.com/pkr-lab/capulus-operator) | [capulus-operator/](capulus/capulus-operator/index.md) |
| cubi | Terminal-Werkzeug für den Kubernetes-Alltag | [GitHub](https://github.com/pkr-lab/cubi) | [cubi/](cubi/index.md) |
| ui-terminal-config | Starship, Nerd Font und KI-CLIs für Ubuntu Desktop | [GitHub](https://github.com/pkr-lab/ui-terminal-config) | [ui-terminal-config/](ui-terminal-config/index.md) |
| DocFlowEngine | PowerShell-Modul: Dokumente erkennen, umbenennen, sortieren | [GitHub](https://github.com/pkr-lab/DocFlowEngine) | [DocFlowEngine/](DocFlowEngine/index.md) |
| PowerShell | Skriptsammlung für Windows-Administration | [GitHub](https://github.com/pkr-lab/PowerShell) | [PowerShell/](PowerShell/index.md) |
| ps-data-fixtures | Testdaten für PowerShell-Skripte | [GitHub](https://github.com/pkr-lab/ps-data-fixtures) | [ps-data-fixtures/](ps-data-fixtures/index.md) |
| EDV-Kretzer | Statische Website von EDV-Kretzer | [GitHub](https://github.com/pkr-lab/EDV-Kretzer) | [EDV-Kretzer/](EDV-Kretzer/index.md) |
| .github | wiederverwendbare Workflows für Releases und Commitlint | [GitHub](https://github.com/pkr-lab/.github) | [github-workflows/](github-workflows/index.md) |
| pkr-lab | GitHub-Profilseite | [GitHub](https://github.com/pkr-lab/pkr-lab) | [pkr-lab/](pkr-lab/index.md) |

## Fehlersuche

Einstieg bei Störungen: **[Zentrale Fehlersuche](capulus/capulus-core/2-betrieb-hardware/20070-troubleshooting-zentral.md)** (Symptom → richtige Anleitung). Jedes Projekt hat eine eigene Seite „Debugging und Fehlersuche“ oder einen Abschnitt „Fehlersuche“:

| Projekt | Fehlersuche |
|---|---|
| capulus-core | [Zentrale Fehlersuche](capulus/capulus-core/2-betrieb-hardware/20070-troubleshooting-zentral.md), [Skripte und Workflows](capulus/capulus-core/f-cicd-automatisierung/f00c0-skripte-und-workflows-referenz.md) |
| capulus-ansible | [Rollen: Fehlersuche je Rolle](capulus/capulus-ansible/rollen/index.md#fehlersuche-je-rolle), [Troubleshooting](capulus/capulus-ansible/a-betriebssystem/a0040-troubleshooting.md) |
| capulus-ios | [Debugging](capulus/capulus-ios/debugging.md) |
| cubi | [Debugging](cubi/debugging.md) |
| ui-terminal-config | [Debugging](ui-terminal-config/debugging.md) |
| DocFlowEngine | [Debugging](DocFlowEngine/debugging.md) |
| PowerShell | [Debugging](PowerShell/debugging.md), [git-pull-push](PowerShell/bash-git-pull-push.md) |
| EDV-Kretzer | [Debugging](EDV-Kretzer/debugging.md) |

## Aufbau

```
pkr-lab-docs/
├── README.md
├── capulus-core/            Doku in Kategorien (a-… bis f-…, 1-… bis 7-…), Komponenten, Archiv
├── capulus-ansible/         index, installation, konfiguration, nutzung, releases, rollen/ plus Fachdocs
├── capulus-ios/
├── cubi/
├── ui-terminal-config/
├── DocFlowEngine/
├── PowerShell/
├── ps-data-fixtures/
├── EDV-Kretzer/
├── github-workflows/
├── pkr-lab/
└── scripts/check-doc-links.py
```

Jedes Projekt hat mindestens `index.md` (Zweck, Überblick, Architektur), `installation.md`, `konfiguration.md`, `nutzung.md` und `releases.md`.

## Mitarbeit

- Commits folgen [Conventional Commits](https://www.conventionalcommits.org/), z. B. `docs(capulus-ansible): Rolle k3s ergänzen`.
- Links zwischen Seiten sind relativ, Links in Code-Repos sind absolute GitHub-URLs.
- Vor dem PR: `python3 scripts/check-doc-links.py`.
- Konventionen für Dateinamen und Aufbau der capulus-core-Docs: [TEMPLATE.md](capulus/capulus-core/TEMPLATE.md).

## Wiki

Die Seiten werden automatisch in Wiki.js im PROD-Cluster gespiegelt, die Ordnerstruktur bleibt als Seitenbaum erhalten. Änderungen macht man hier, nicht im Wiki. Aufbau und Betrieb: [wiki-sync](capulus/capulus-core/wiki-sync.md).
