# github-workflows (pkr-lab/.github)

Zentrale, wiederverwendbare GitHub-Workflows für alle Repos von pkr-lab: automatische Releases aus Conventional Commits und eine Prüfung der Commit-Nachrichten auf Pull Requests. Dazu ein lokaler `commit-msg`-Hook.

Repository: [pkr-lab/.github](https://github.com/pkr-lab/.github)

## Überblick

| Datei | Aufgabe |
|---|---|
| `.github/workflows/release.yml` | wiederverwendbar: semantic-release mit CHANGELOG, Tag, GitHub-Release, optionalem Artefakt-Skript und Container-Image |
| `.github/workflows/commitlint.yml` | wiederverwendbar: prüft alle Commits eines Pull Requests |
| `hooks/commit-msg` | lokaler Hook mit denselben Regeln, ohne Abhängigkeiten |
| `.github/workflows/self-release.yml`, `self-commitlint.yml`, `ci.yml` | das Repo nutzt seine Workflows selbst und prüft sie mit actionlint |

## Architektur

```mermaid
flowchart LR
  subgraph repo["Code-Repo (z. B. cubi)"]
    caller["release.yml<br/>(Aufrufer)"]
    clcaller["commitlint.yml<br/>(Aufrufer)"]
    build[".github/release/build.sh<br/>(optional)"]
  end
  subgraph central["pkr-lab/.github"]
    rel["release.yml<br/>(wiederverwendbar)"]
    cl["commitlint.yml<br/>(wiederverwendbar)"]
  end
  caller -->|"uses @SHA"| rel
  clcaller -->|"uses @SHA"| cl
  rel -->|"prepareCmd"| build
  rel --> out["Tag, CHANGELOG.md,<br/>GitHub-Release, Artefakte, Image"]
```

Die Aufrufer in den Repos binden die Workflows mit dem Commit-SHA ein (`uses: pkr-lab/.github/.github/workflows/release.yml@<sha> # v1.0.1`). Renovate hebt die Version an. Die Konfiguration von semantic-release liegt nur zentral; Repos haben keine eigene `.releaserc.json`.

## Wer nutzt sie

capulus-core, capulus-ansible, capulus-ios, capulus-wiki-sync, cubi, ui-terminal-config, DocFlowEngine, PowerShell, ps-data-fixtures, EDV-Kretzer, pkr-lab und pkr-lab-docs.

## Seiten

- [Installation](installation.md)
- [Konfiguration](konfiguration.md)
- [Nutzung](nutzung.md)
- [Releases](releases.md)
