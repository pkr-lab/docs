# Branch-Schutz für `main` — PR-Pflicht + CI als Gate

`main` ist die Quelle von TECH und PROD: ein Commit dort wird von ArgoCD sofort
ausgerollt. Bis September 2026 gingen Änderungen direkt per Push auf `main`, die
[CI](f0070-ci-lint.md) lief erst danach und konnte einen Fehler nur noch melden
(so blieb z. B. ein kaputter Renovate-Bump an pacmans `go.mod` bis zum
Image-Build unbemerkt). Das Ruleset **„main: PR + CI“** macht daraus ein echtes
Gate: Änderungen kommen nur noch per Pull Request, und der PR lässt sich erst
mergen, wenn die CI-Jobs grün sind.

---

## Was das Ruleset erzwingt

| Regel | Wirkung |
|---|---|
| Pull Request | Kein Direkt-Push auf `main`. Kein Pflicht-Review (`0` Freigaben), du arbeitest allein — die Sicherheit kommt aus den Checks. |
| Pflicht-Checks | `make lint (yamllint, helm lint)`, `helm template \| kubeconform`, `go build, vet, test, tidy`, `token register`, `gitleaks` — fünf der sechs Jobs aus [`ci.yml`](https://github.com/pkr-lab/capulus-core/blob/main/.github/workflows/ci.yml) |
| Kein Force-Push, kein Löschen | `main` ist nicht überschreibbar |
| „Up to date“ nicht verlangt | `strict_required_status_checks_policy: false` — sonst müsste jeder PR nach jedem anderen Merge neu gebaut werden |

**Notausgang:** Repo-Admins dürfen einen PR trotz roter oder fehlender Checks
mergen (`bypass_mode: pull_request`), aber weiterhin **nicht direkt pushen**.
Das verhindert einen Deadlock, wenn die CI selbst kaputt ist (der Fix für die CI
müsste sonst durch die kaputte CI).

Die Image-Builds aus [`build-images.yml`](f0060-build-images.md) sind bewusst
**keine** Pflicht-Checks: der Pfadfilter lässt den Workflow bei PRs ohne
Image-Änderung gar nicht laufen, ein Pflicht-Check bliebe dann ewig auf „pending“.
Dasselbe gilt für [CodeQL](../d-sicherheit/d0080-codeql-dependabot.md): der Workflow
überspringt reine Doku-PRs, und der Swift-Job auf macOS soll einen Merge nicht
blockieren, wenn er selbst ein Problem hat. Neue Funde ab `high` sieht man trotzdem
als roten Check „Code scanning results“ im PR.

## Aktivieren

Das Ruleset wird per Skript angelegt (reproduzierbar, im Repo versioniert):

```bash
scripts/setup-main-ruleset.sh --dry-run   # zeigt das JSON
scripts/setup-main-ruleset.sh             # anlegen bzw. aktualisieren (gh als Admin)
```

**Reihenfolge ist wichtig:** erst die neue `ci.yml` (mit allen fünf Jobs) auf
`main` bringen, dann das Skript ausführen. Andernfalls verlangt das Ruleset
Checks, die es nirgends gibt, und der erste PR bleibt auf „pending“. Nach dem
Aktivieren ist auch der eigene Push auf `main` gesperrt.

## Arbeitsablauf danach

1. Branch anlegen, committen, pushen (`git push -u origin <branch>`).
2. PR gegen `main` öffnen (`gh pr create` oder im Browser) — die CI startet.
3. Grün → mergen. Rot → fixen und nachpushen (der Lauf wird ersetzt).

Wer sonst noch PRs aufmacht:

| Quelle | Bemerkung |
|---|---|
| Renovate (App und [Fallback-Workflow](f0020-renovate.md)) | PRs sind jetzt gegated. Ein kaputter Bump (Major-Sprung mit geändertem Import-Pfad, halb aktualisierte `go.mod`) bleibt rot statt auf `main` zu landen. |
| [ENTW-Promotion](f0080-entw-promotion.md) | öffnet PRs mit einem PAT (sonst liefe die CI auf dem PR nicht an) und mergt nie selbst |
| [Dependabot](../d-sicherheit/d0080-codeql-dependabot.md) | nur Sicherheits-PRs (`fix(deps): …`, Labels `dependencies` und `security`). Die CI läuft darauf ohne PAT, Dependabot-PRs lösen Workflows aus. Gemergt wird von Hand. |
| Kleine GitOps-Schritte (Migration: App stoppen, kopieren, starten) | ebenfalls per PR. Ein Lauf dauert ca. 2 Minuten (der `make lint`-Job ist der längste). Nur im Notfall per Admin-Bypass mergen. |

Nicht betroffen ist `mirror-gitlab.yml` (liest `main`). `release.yml` pusht seit der Umstellung auf die zentralen Workflows den Commit
`chore(release): X [skip ci]` mit `CHANGELOG.md` direkt auf `main` und braucht bei aktivem Ruleset eine Ausnahme (siehe oben).

## Vorhandene Rulesets im Repo

Stand 2026-10-19: Alle Rulesets sind **deaktiviert** (`enforcement: disabled`, geändert am 2026-10-18 gegen 15:29 Uhr während der Umstellung auf die
zentralen Release-Workflows). Es gibt auch keinen klassischen Branch-Schutz. Ein Merge nach `main` ist damit nicht durch Checks gesperrt, und der
Release-Commit `chore(release): X [skip ci]` kann direkt auf `main` landen.

| Ruleset | Stand | Bemerkung |
|---|---|---|
| `entw-1` | deaktiviert | `entw`: kein Löschen, kein Force-Push |
| `Protect MAIN` | deaktiviert | Löschen/Force-Push/PR (0 Freigaben) plus `code_quality`, trifft `main` **und** `entw` |
| `main: PR + CI` | deaktiviert | PR-Pflicht, Admin-Bypass (`pull_request`) und Pflicht-Checks. Die gespeicherten Check-Namen sind veraltet (siehe unten) |

Das Ruleset `entw-2` aus früheren Ständen existiert nicht mehr.

**Wiedereinschalten:** Das gespeicherte Ruleset `main: PR + CI` verlangt `make lint (yamllint, ansible-lint, helm lint)` und `docs links`. Beide Jobs
gibt es nicht mehr (der Job heißt `make lint (yamllint, helm lint)`, die Link-Prüfung läuft in pkr-lab-docs), `isc-news-bot (typecheck, unit und
Mock-Tests)` fehlt. Ein aktiviertes Ruleset würde jeden PR auf „pending“ halten. Zuerst `scripts/setup-main-ruleset.sh` ausführen, das die aktuelle Liste
enthält. Es legt das Ruleset als `active` an.

**Ausnahme für den Release-Commit:** Das Ruleset verlangt Pull Requests, der Release-Workflow pusht aber `CHANGELOG.md` direkt auf `main`. Der
`GITHUB_TOKEN` (`github-actions[bot]`) lässt sich nicht als Bypass-Akteur eintragen (nach Community-Berichten; eine offizielle Quelle dazu wurde nicht
gefunden, vor dem Einschalten ausprobieren). Möglichkeiten: ein GitHub-App-Token oder ein Deploy-Key bzw. PAT eines Akteurs mit Bypass für den Schritt
`semantic-release` (Änderung im zentralen Workflow `pkr-lab/.github`), oder Rulesets ausgeschaltet lassen. Entscheidung offen, siehe
[600b0](../6-hintergruende/600b0-nachpruefung-repo-aufspaltung.md#rulesets).

## Stolperfallen

- **Pfadgefilterte Workflows als Pflicht-Check** bleiben auf „pending“, wenn der
  Filter nicht greift. Nur Jobs ohne Pfadfilter (die vier aus `ci.yml`) sind Pflicht.
- **`GITHUB_TOKEN`-PRs lösen keine CI aus.** Automatisch erzeugte PRs brauchen ein
  PAT (siehe Promotion), sonst sind sie nicht mergbar.
- **Check-Namen** im Ruleset müssen den `name:` der Jobs in `ci.yml` entsprechen.
  Wird ein Job umbenannt oder kommt einer dazu (zuletzt `token register`), in
  `scripts/setup-main-ruleset.sh` nachziehen und das Skript erneut ausführen: es
  aktualisiert das vorhandene Ruleset.
- **Lock-out:** Ist das Ruleset aktiv und die CI kaputt, hilft nur der
  Admin-Bypass am PR (oder das Ruleset unter *Settings → Rules* auf `disabled`
  setzen).
