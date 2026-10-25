# Installation

Ein Repo bekommt Releases und Commit-Prüfung mit zwei Aufruferdateien und einem optionalen Hook.

## Voraussetzungen

- Das Repo ist öffentlich oder das Repo `pkr-lab/.github` erlaubt den Zugriff (öffentlich: kein Schritt nötig).
- Der Standardbranch heißt `main` (sonst `branches` setzen).
- Die Commits des Repos folgen Conventional Commits, damit semantic-release eine Version berechnen kann.
- Unter Settings → Actions → General dürfen Workflows `GITHUB_TOKEN` mit Schreibrechten bekommen („Read and write permissions“), und der Branch `main` erlaubt dem Release-Commit das Pushen (kein Ruleset, das Pushes ohne Pull Request verbietet, oder mit Ausnahme für GitHub Actions).

## Setup

`.github/workflows/release.yml`:

```yaml
name: Release

on:
  push:
    branches: [main]
  workflow_dispatch: {}

permissions:
  contents: write
  issues: write
  pull-requests: write
  packages: write

concurrency:
  group: release-${{ github.ref }}
  cancel-in-progress: false

jobs:
  release:
    uses: pkr-lab/.github/.github/workflows/release.yml@<commit-sha> # v1.0.1
```

`.github/workflows/commitlint.yml`:

```yaml
name: Commitlint

on:
  pull_request:
    branches: [main]

permissions:
  contents: read

jobs:
  commitlint:
    uses: pkr-lab/.github/.github/workflows/commitlint.yml@<commit-sha> # v1.0.1
```

Den SHA liefert `git rev-parse v1.0.1^{commit}` im Repo `.github`. Ein vorhandenes `.releaserc.json` im Code-Repo löschen, der Workflow erzeugt die Konfiguration selbst.

## Lokaler Hook

```bash
mkdir -p .githooks
cp /pfad/zu/.github/hooks/commit-msg .githooks/commit-msg
git config core.hooksPath .githooks
```

Bei Repos mit Makefile (capulus-core, capulus-ansible) erledigt `make hooks` den letzten Schritt.


## Fehlersuche

| Symptom | Ursache | Lösung |
|---|---|---|
| Workflow-Aufruf meldet „workflow was not found“ | SHA oder Pfad im `uses:` falsch | [Nutzung: Fehlersuche](nutzung.md#fehlersuche) |
| Kein Release nach dem Merge | Commit-Format, nach dem letzten Tag keine Commits mit `feat`/`fix` | [Nutzung: Fehlersuche](nutzung.md#fehlersuche) |
| `commit-msg`-Hook lehnt ab | Kopfzeile verletzt das Format | `<typ>(<scope>): <beschreibung>` ([Lokaler Hook](#lokaler-hook)) |
| Push auf `main` abgelehnt | Branch-Schutz | PR verwenden, [Branch-Schutz (capulus-core)](../capulus/capulus-core/f-cicd-automatisierung/f0090-branch-schutz-main.md#fehlersuche) |
