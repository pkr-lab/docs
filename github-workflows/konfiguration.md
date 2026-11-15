# Konfiguration

## Eingaben von `release.yml`

| Eingabe | Standard | Bedeutung |
|---|---|---|
| `branches` | `["main"]` | JSON-Liste der Release-Branches (`'["main","master"]'`) |
| `build-script` | leer | Skript im Repo, das Artefakte nach `dist/` baut. Es läuft in der Prepare-Phase von semantic-release und bekommt die neue Version als erstes Argument |
| `assets` | `[]` | JSON-Liste von Glob-Mustern, die ans GitHub-Release angehängt werden (`'["dist/*.zip"]'`) |
| `image` | leer | Name eines Container-Images; gesetzt = nach dem Release `docker build` aus dem `Dockerfile` im Repo-Root und Push nach `ghcr.io/<owner>/<image>:vX.Y.Z` |
| `changelog` | `true` | `CHANGELOG.md` pflegen und als Release-Commit auf den Branch pushen; `false` bei Branches, die nur per Pull Request änderbar sind (seit v1.2.0) |
| `node-version` | `24` | Node.js für `npx` |

Ausgaben: `released` (`true`/`false`) und `version` (ohne `v`). `released` ist nur `true`, wenn semantic-release **in diesem Lauf** einen Tag `vX.Y.Z` erzeugt hat. Der Workflow merkt sich vor dem Lauf alle vorhandenen Tags; ein Re-Run oder ein `workflow_dispatch` ohne neue Commits meldet deshalb `released=false` und baut kein Image erneut.

Der Aufrufer muss Rechte vergeben, weil ein wiederverwendbarer Workflow nicht mehr Rechte bekommt als sein Aufrufer: `contents: write` (Tag, Release, CHANGELOG-Commit), `issues: write` und `pull-requests: write` (Kommentare des GitHub-Plugins), `packages: write` (Image). `packages: write` ist auch ohne `image` nötig: GitHub prüft die angeforderten Rechte eines aufgerufenen Workflows beim Start, unabhängig davon, ob der Image-Schritt später läuft. Wer darauf verzichten will, braucht einen eigenen Image-Workflow (Bruch der Schnittstelle, `feat!`).

## Was semantic-release macht

Der Workflow schreibt zur Laufzeit eine `.releaserc.json` in den Arbeitsbaum mit diesen Plugins. Sie muss dort liegen, weil semantic-release Plugins aus einer Konfiguration außerhalb des Repos relativ zu deren Ordner sucht und die per `npx` geladenen Plugins dann nicht findet. Vor dem Docker-Build wird die Datei entfernt, damit sie nicht im Build-Kontext landet.

| Reihenfolge | Plugin | Aufgabe |
|---|---|---|
| 1 | `commit-analyzer` (Preset `conventionalcommits`) | nächste Version aus den Commits seit dem letzten Tag; `docs` und `chore` ergeben ein Patch-Release |
| 2 | `release-notes-generator` | Release Notes, Abschnitte für `feat`, `fix`, `perf`, `revert`, `docs`, `chore` |
| 3 | `changelog` (nur mit `changelog: true`) | schreibt `CHANGELOG.md` |
| 4 | `exec` (nur mit `build-script`) | `bash '<build-script>' <version>`; der Pfad steht in einfachen Anführungszeichen, Leerzeichen und Sonderzeichen sind erlaubt, zusätzliche Argumente im Input nicht |
| 5 | `git` (nur mit `changelog: true`) | committet `CHANGELOG.md` mit `chore(release): X.Y.Z [skip ci]` |
| 6 | `github` | Tag `vX.Y.Z`, GitHub-Release, `assets` |

Die npm-Pakete sind exakt gepinnt, Renovate hebt sie über einen Regex-Manager in `renovate.json` an:

| Paket | Workflow |
|---|---|
| `semantic-release`, `@semantic-release/changelog`, `@semantic-release/git`, `@semantic-release/exec` | `release.yml` |
| `conventional-changelog-conventionalcommits` (Hauptversion 8) | `release.yml` |
| `@commitlint/cli` | `commitlint.yml`, `ci.yml` |

Der Tag-Aufbau ist `vX.Y.Z`. Ohne vorhandene Tags beginnt die Zählung bei `v1.0.0`, sonst beim höchsten Tag der Form `vX.Y.Z`. Das Preset `conventional-changelog-conventionalcommits` ist auf Version 8 festgelegt, weil Version 9 einen neueren Changelog-Writer verlangt, als semantic-release mitbringt.

## Regeln von `commitlint.yml`

| Regel | Wert |
|---|---|
| Typen | `feat`, `fix`, `docs`, `chore`, `refactor`, `test`, `ci`, `build`, `perf`, `style`, `revert` |
| Typ, Scope | kleingeschrieben, Typ Pflicht |
| Beschreibung | Pflicht, kein Punkt am Ende |
| Kopfzeile | höchstens 120 Zeichen |
| Breaking Change | `!` vor dem Doppelpunkt erlaubt (`feat!: …`, `fix(app)!: …`) |

Der Workflow bringt einen eigenen Parser für die Kopfzeile mit (`parserPreset.parserOpts.headerPattern`), der `!` kennt. Der Standard-Parser von commitlint (Angular) kennt es nicht und lehnte solche Commits mit `type-empty` und `subject-empty` ab.

Geprüft werden alle Commits zwischen Basis und Kopf des Pull Requests; Merge-Commits ignoriert commitlint. Der Job läuft nur bei `pull_request` und wird bei anderen Ereignissen übersprungen.

`hooks/commit-msg` prüft lokal dieselben Regeln: Typ, Scope (kleingeschrieben), Doppelpunkt, kein Punkt am Ende und höchstens 120 Zeichen. Merge-, Revert-, fixup- und squash-Zeilen lässt er durch. Der Job `commitlint-Regeln` in `ci.yml` schickt dieselben Nachrichten durch Hook und commitlint und schlägt fehl, wenn beide unterschiedlich urteilen.
