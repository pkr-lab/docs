# Releases

Releases entstehen automatisch aus den Commit-Nachrichten. Der Prozess ist in allen Repos von pkr-lab gleich und läuft über die wiederverwendbaren Workflows im Repo [pkr-lab/.github](https://github.com/pkr-lab/.github), beschrieben unter [github-workflows](../github-workflows/index.md).

## Ablauf

1. Änderungen kommen per Pull Request nach `main`. Der Check **commitlint** prüft, dass alle Commits des Pull Requests dem Format [Conventional Commits](https://www.conventionalcommits.org/) folgen.
2. Jeder Push auf `main` startet `.github/workflows/release.yml`. Der Workflow ruft den zentralen Release-Workflow auf (semantic-release).
3. semantic-release bestimmt aus den Commits seit dem letzten Tag die nächste Version nach [SemVer](https://semver.org/), schreibt `CHANGELOG.md` fort (Commit `chore(release): X.Y.Z [skip ci]`), setzt den Git-Tag `vX.Y.Z` und veröffentlicht ein GitHub-Release mit Release Notes.

| Commit | Wirkung |
|---|---|
| `fix:` | Patch (`1.2.3` → `1.2.4`) |
| `feat:` | Minor (`1.2.3` → `1.3.0`) |
| `feat!:` oder `BREAKING CHANGE:` im Footer | Major (`1.2.3` → `2.0.0`) |
| `docs:`, `chore:`, `refactor:`, `test:`, `ci:`, `build:`, `perf:`, `style:` | kein Release |

Format: `<typ>(<scope>): <beschreibung>`, Der Typ und der Scope sind kleingeschrieben, die Beschreibung endet ohne Punkt, die Kopfzeile ist höchstens 120 Zeichen lang. Lokal prüft der Hook `.githooks/commit-msg` dasselbe; aktivieren mit `git config core.hooksPath .githooks`.

## Versionen

Die Tags `DocFlowEngine-1.0.0` und `DocFlowEngine-2.0.0` bleiben bestehen. Zusätzlich markiert `v2.0.0` denselben Commit wie `DocFlowEngine-2.0.0`; ab dort zählt semantic-release weiter (`v2.1.0`, …).

## Artefakte

Pro Release hängt ein ZIP (`DocFlowEngine-vX.Y.Z.zip`) am GitHub-Release. Es enthält `scripts/` mit dem Modul (Manifest `DocFlowEngine.psd1`, `ModuleVersion` wird beim Bauen auf die Release-Version gesetzt, `DocFlowEngine.psm1` und Teilmodule), `config/`, `LICENSE` und `README.md`. Gebaut von `.github/release/build.sh`; ohne Veröffentlichung in der PowerShell Gallery.

## Änderungsprotokoll

[CHANGELOG.md](https://github.com/pkr-lab/DocFlowEngine/blob/main/CHANGELOG.md) im Repo und die [GitHub Releases](https://github.com/pkr-lab/DocFlowEngine/releases).
