# Release-Automatisierung — Release bei jedem Merge auf `main`

Jeder Push auf `main` (also jeder gemergte Pull Request) löst automatisch einen [semantic-release](https://semantic-release.gitbook.io/)-Lauf aus, der — falls die Commits seit dem letzten Tag das rechtfertigen — eine neue Versionsnummer vergibt, `CHANGELOG.md` fortschreibt, einen Git-Tag `vX.Y.Z` setzt und ein GitHub-Release mit Release Notes veröffentlicht. Der Ablauf ist für alle Repos von pkr-lab derselbe und liegt zentral im Repo `pkr-lab/.github` ([github-workflows](../../../github-workflows/index.md)); capulus-core ruft ihn nur auf.

---

## Warum das funktioniert, ohne extra Konfiguration pro Commit

Die Commit-Historie folgt [Conventional Commits](https://www.conventionalcommits.org/) (`feat(app): …`, `fix(app): …`). semantic-release liest das Präfix, um die nächste Version zu bestimmen:

| Commit-Präfix | Versions-Bump |
|---|---|
| `fix: ...` | Patch (`1.2.3` → `1.2.4`) |
| `feat: ...` | Minor (`1.2.3` → `1.3.0`) |
| `feat!: ...` oder `BREAKING CHANGE:` im Footer | Major (`1.2.3` → `2.0.0`) |
| alles andere (`docs:`, `chore:`, `refactor:`, `test:`, `ci:`, …) | kein Release |

Ein Check (commitlint) prüft auf Pull Requests, dass alle Commits dieses Format haben. Lokal macht `make hooks` den `commit-msg`-Hook aktiv.

---

## Architektur

```
git push origin main (per Merge)
        │
        ▼
.github/workflows/release.yml           (Aufrufer in capulus-core)
        │  uses: pkr-lab/.github/.github/workflows/release.yml@<sha>
        ▼
semantic-release                        (Konfiguration wird zentral erzeugt)
        │
        ├── commit-analyzer, release-notes-generator → Version und Notes
        ├── changelog                                → CHANGELOG.md
        ├── git                                      → Commit "chore(release): X.Y.Z [skip ci]"
        └── github                                   → Tag vX.Y.Z + GitHub-Release
```

Aufrufer: [`.github/workflows/release.yml`](https://github.com/pkr-lab/capulus-core/blob/main/.github/workflows/release.yml). Eine eigene `.releaserc.json` gibt es in capulus-core nicht mehr.

---

## Was das NICHT tut

- **Kein Einfluss auf ArgoCD/Deployment.** ArgoCD trackt weiterhin `targetRevision: main` direkt (siehe [capulus-ansible/a-betriebssystem/a0010-overview.md](../../capulus-ansible/a-betriebssystem/a0010-overview.md)). Releases sind Versionshistorie, kein Deployment-Gate.
- **Kein `npm publish`.** Dieses Repo ist kein npm-Paket.
- **Keine Release-Dateien und keine Images.** Die Workload-Images baut `build-images.yml` getrennt (`sha-<commit>`).
- **Der CHANGELOG-Commit braucht Schreibrecht auf `main`.** Ist ein Ruleset aktiv, das Pushes ohne Pull Request verbietet, muss GitHub Actions dort eine Ausnahme bekommen.

---

## Troubleshooting

| Symptom | Hinweis |
|---|---|
| Kein Release trotz `feat:`-Commit | Präfix exakt Conventional-Commits-Syntax (`feat: ...`, nicht `Feat:` oder `feature:`); der Commit muss nach dem letzten Tag liegen |
| Workflow schlägt mit 403/permission-Fehler fehl | Der Aufrufer braucht `contents: write`, `issues: write`, `pull-requests: write`, `packages: write`; zusätzlich darf `main` den CHANGELOG-Commit annehmen |
| Mehrere Commits, aber nur ein Release | Erwartetes Verhalten: semantic-release fasst alle Commits seit dem letzten Tag zusammen |
| Fehler bei den Preset-Paketen | siehe [Fehlersuche in github-workflows](../../../github-workflows/nutzung.md#fehlersuche) |
| Workflow-Lauf ansehen | GitHub-Repo → **Actions** → **Release** |
