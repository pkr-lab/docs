# Konfiguration

## Eingaben von `release.yml`

| Eingabe | Standard | Bedeutung |
|---|---|---|
| `branches` | `["main"]` | JSON-Liste der Release-Branches (`'["main","master"]'`) |
| `build-script` | leer | Skript im Repo, das Artefakte nach `dist/` baut. Es läuft in der Prepare-Phase von semantic-release und bekommt die neue Version als erstes Argument |
| `assets` | `[]` | JSON-Liste von Glob-Mustern, die ans GitHub-Release angehängt werden (`'["dist/*.zip"]'`) |
| `image` | leer | Name eines Container-Images; gesetzt = nach dem Release `docker build` aus dem `Dockerfile` im Repo-Root und Push nach `ghcr.io/<owner>/<image>:vX.Y.Z` |
| `node-version` | `24` | Node.js für semantic-release |

Ausgaben: `released` (`true`/`false`) und `version` (ohne `v`).

Der Aufrufer muss Rechte vergeben, weil ein wiederverwendbarer Workflow nicht mehr Rechte bekommt als sein Aufrufer: `contents: write` (Tag, Release, CHANGELOG-Commit), `issues: write` und `pull-requests: write` (Kommentare des GitHub-Plugins), `packages: write` (Image).

## Was semantic-release macht

Der Workflow schreibt zur Laufzeit eine `.releaserc.json` mit diesen Plugins:

| Reihenfolge | Plugin | Aufgabe |
|---|---|---|
| 1 | `commit-analyzer` (Preset `conventionalcommits`) | nächste Version aus den Commits seit dem letzten Tag |
| 2 | `release-notes-generator` | Release Notes |
| 3 | `changelog` | schreibt `CHANGELOG.md` |
| 4 | `exec` (nur mit `build-script`) | `bash <build-script> <version>` |
| 5 | `git` | committet `CHANGELOG.md` mit `chore(release): X.Y.Z [skip ci]` |
| 6 | `github` | Tag `vX.Y.Z`, GitHub-Release, `assets` |

Der Tag-Aufbau ist `vX.Y.Z`. Ohne vorhandene Tags beginnt die Zählung bei `v1.0.0`, sonst beim höchsten Tag der Form `vX.Y.Z`. Das Preset `conventional-changelog-conventionalcommits` ist auf Version 8 festgelegt, weil Version 9 einen neueren Changelog-Writer verlangt, als semantic-release mitbringt.

## Regeln von `commitlint.yml`

| Regel | Wert |
|---|---|
| Typen | `feat`, `fix`, `docs`, `chore`, `refactor`, `test`, `ci`, `build`, `perf`, `style`, `revert` |
| Typ, Scope | kleingeschrieben, Typ Pflicht |
| Beschreibung | Pflicht, kein Punkt am Ende |
| Kopfzeile | höchstens 120 Zeichen |

Geprüft werden alle Commits zwischen Basis und Kopf des Pull Requests; Merge-Commits ignoriert commitlint. `hooks/commit-msg` prüft lokal Typ, Scope und Doppelpunkt mit einem regulären Ausdruck und lässt Merge-, Revert-, fixup- und squash-Zeilen durch.
