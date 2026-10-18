# Nutzung

## Normaler Ablauf in einem Repo

```bash
git switch -c feat/neue-sache
git commit -m "feat(bereich): kurze beschreibung"
git push -u origin feat/neue-sache
gh pr create
```

Der Pull Request läuft durch commitlint und die CI des Repos. Nach dem Merge entsteht das Release, wenn ein `feat`, `fix` oder `BREAKING CHANGE` seit dem letzten Tag dabei ist. Mehrere Merges zwischen zwei Läufen fasst semantic-release zu einem Release zusammen.

## Artefakt anhängen

1. Skript `.github/release/build.sh` anlegen. Es bekommt die Version als `$1`, liegt im Repo-Root als Arbeitsverzeichnis und legt Dateien unter `dist/` ab:

```bash
#!/usr/bin/env bash
set -euo pipefail

version="$1"
mkdir -p dist
git archive --format=tar.gz --prefix="projekt-v${version}/" -o "dist/projekt-v${version}.tar.gz" HEAD
```

2. Im Aufrufer `with:` ergänzen:

```yaml
    with:
      build-script: .github/release/build.sh
      assets: '["dist/*.tar.gz"]'
```

3. `dist/` in `.gitignore` aufnehmen. Beispiele: capulus-ansible (Tarball), DocFlowEngine (ZIP mit Modulmanifest, Version per `sed` gesetzt), PowerShell (ZIP).

## Container-Image

Im Aufrufer `with: image: <name>` setzen. Voraussetzung ist ein `Dockerfile` im Repo-Root. Das Image bekommt nur den Versionstag (`vX.Y.Z`), kein `latest`. Das Paket auf ghcr.io ist beim ersten Push privat; für anonymen Pull einmal auf öffentlich stellen. Beispiel: capulus-wiki-sync.

## Release manuell auslösen

`workflow_dispatch` am Release-Workflow des Repos (Actions → Release → Run workflow). Es entsteht nur dann ein Release, wenn seit dem letzten Tag releasefähige Commits vorliegen.

## Fehlersuche

| Symptom | Ursache und Abhilfe |
|---|---|
| `Missing helper: conventional-changelog-conventionalcommits requires conventional-changelog-writer@9` | Preset in Version 9 gezogen; im Workflow ist `conventional-changelog-conventionalcommits@8` festgelegt |
| Kein Release trotz `feat`-Commit | Kopfzeile prüfen (`feat:` kleingeschrieben, Doppelpunkt und Leerzeichen); der Commit muss nach dem letzten Tag liegen |
| 403 beim Push des CHANGELOG-Commits | `main` verbietet Pushes ohne Pull Request. Für GitHub Actions eine Ausnahme im Ruleset eintragen oder das Ruleset anders zuschneiden |
| Release-Commit löst weitere Workflows aus | `[skip ci]` steht in der Nachricht; Workflows auf `push` werden dann übersprungen |
| actionlint meldet SC2129 | mehrere `echo >> "$GITHUB_OUTPUT"` in einer `{ …; } >> "$GITHUB_OUTPUT"`-Gruppe zusammenfassen |
| Aufrufer meldet „workflow was not found“ | SHA oder Pfad falsch; `.github/workflows/release.yml@<sha>` aus dem Repo `pkr-lab/.github` verwenden |
