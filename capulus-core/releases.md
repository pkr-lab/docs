# Releases

capulus-core erzeugt Releases automatisch aus Conventional Commits.

## Ablauf

Jeder Push auf `main` (also jeder gemergte Pull Request) startet `.github/workflows/release.yml`. Der Workflow führt semantic-release mit `.releaserc.json` aus: Branch `main`, Tag-Format `v${version}`, Plugins `commit-analyzer`, `release-notes-generator` und `github`. Reicht ein Commit seit dem letzten Tag, vergibt semantic-release die nächste Version, setzt den Git-Tag und veröffentlicht ein GitHub-Release mit generierten Release Notes.

| Commit | Wirkung |
|---|---|
| `fix:` | Patch (`3.49.0` → `3.49.1`) |
| `feat:` | Minor (`3.49.0` → `3.50.0`) |
| `feat!:` oder `BREAKING CHANGE:` im Footer | Major (`3.49.0` → `4.0.0`) |
| `docs:`, `chore:`, `refactor:`, … | kein Release |

Commits folgen Conventional Commits mit Scope und deutscher Beschreibung, z. B. `feat(isc-news-bot): …`, `fix(monitoring): …`, `chore(<app>): image.tag auf sha-…`.

## Änderungsprotokoll

Die Release Notes stehen bei den [GitHub Releases](https://github.com/pkr-lab/capulus-core/releases). Hintergrund und Architektur der Automatisierung: [f0030 Release-Automatisierung](f-cicd-automatisierung/f0030-release-automation.md).

## Verbindung zu den anderen Repos

capulus-ansible zieht die Quellen des alert-caller aus einem capulus-core-Tag (`alert_caller_core_ref`). Nach einer Änderung an `argocd/apps/tech/alert-caller/` muss dort das Tag und die SHA-256-Prüfsumme angehoben werden.
