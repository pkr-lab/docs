# Releases

Releases entstehen automatisch aus den Commit-Nachrichten. Der Prozess ist in allen Repos von pkr-lab gleich und läuft über einen wiederverwendbaren Workflow im Repo `pkr-lab/.github`.

## Ablauf

1. Änderungen kommen per Pull Request nach `main`. Ein Check (commitlint) prüft, dass die Commit-Nachrichten dem Format [Conventional Commits](https://www.conventionalcommits.org/) folgen.
2. Jeder Push auf `main` startet den Release-Workflow (semantic-release).
3. semantic-release bestimmt aus den Commits seit dem letzten Tag die nächste Version nach [SemVer](https://semver.org/), setzt den Git-Tag `vX.Y.Z`, schreibt `CHANGELOG.md` fort und veröffentlicht ein GitHub-Release mit Release Notes.

| Commit | Wirkung |
|---|---|
| `fix:` | Patch (`1.2.3` → `1.2.4`) |
| `feat:` | Minor (`1.2.3` → `1.3.0`) |
| `feat!:` oder `BREAKING CHANGE:` im Footer | Major (`1.2.3` → `2.0.0`) |
| `docs:`, `chore:`, `refactor:`, `test:`, `ci:` | kein Release |

Das Repo hat noch keine Tags und beginnt bei `v1.0.0`. Der Standardbranch heißt noch `master`; der Release-Workflow ist auf `main` und `master` eingestellt, bis der Branch umbenannt ist.

## Artefakte

Pro Release hängt ein ZIP (`PowerShell-vX.Y.Z.zip`) mit allen Skripten und `Logging.psm1` am GitHub-Release; es ist die Vorlage für die Sync-Quelle.

## Änderungsprotokoll

[CHANGELOG.md](https://github.com/pkr-lab/PowerShell/blob/main/CHANGELOG.md) im Repo und die [GitHub Releases](https://github.com/pkr-lab/PowerShell/releases).
