# Nachprüfung nach der Repo-Aufspaltung — Befunde, Fixes und offene Entscheidungen

Stand 2026-10-19. capulus-core wurde am 2026-10-18 in capulus-core, capulus-ansible, capulus-ios und capulus-wiki-sync getrennt, die Doku zog nach pkr-lab-docs, und die Releases laufen über die zentralen Workflows in `pkr-lab/.github` (v1.0.1). Diese Seite hält fest, was dabei kaputtgegangen ist, was repariert wurde, was geprüft wurde und was noch zu entscheiden ist.

> **Grenze der Prüfung:** Die Nachprüfung lief von einem Arbeitsrechner aus, dessen Shell-Sandbox kein LAN erreicht (auch nicht den Router). `kubectl`, `argocd`, SSH und damit alle Live-Tests gegen die Cluster und Hosts sind **nicht** gelaufen. Was hier „ok“ heißt, ist statisch, lokal oder über GitHub belegt. Die Live-Tests stehen als Befehle in [Live-Tests](#live-tests).

---

## Geprüfter Umfang

Alle Repos lagen sauber und synchron zu `origin` (13 Repos in `~/gitlab`). Erfasst wurden Querverweise in Code, Konfiguration, Workflows, Makefiles, Skripten, App-Quellen, Web-Seiten und Doku: Argo-CD-`repoURL`/`path`/`targetRevision`, Helm-Charts und Image-Referenzen, Ansible-Rollenpfade, `requirements.yml`, Semaphore-Konfiguration, Token-Register, `cubi`-Konfiguration, iOS-Endpunkte und alle absoluten GitHub-Links in allen Repos gegen die lokalen Dateien. Als Vergleichsstand für den Split dient der Branch `backup/pre-split-2026-10-18` in capulus-core (Commit `8a193f8`).

---

## Befunde und Fixes

Alle Fixes liegen als lokale Commits auf eigenen Branches, nichts ist gepusht.

| Nr | Befund | Repo / Datei | Fix (Branch, Commit) |
|---|---|---|---|
| 1 | Semaphore-Repository `home-server-git` behält nach erneutem Bootstrap die alte `git_url` (capulus-core), die Templates zeigen aber auf `site.yml` und `worker-0.yml`, die dort nicht mehr liegen. Der tägliche Lauf 06:00 würde das Playbook nicht finden. Live nicht prüfbar. | capulus-ansible, `roles/semaphore_bootstrap/tasks/repository.yml` | `fix/semaphore-repository-update`, `35474a9`. Test gegen Mock-API: PUT nur bei abweichender URL oder Branch, POST nur bei fehlendem Repo, sonst nichts. |
| 2 | wiki-sync-Chart bleibt auf `v1.0.0`, Renovate öffnet keinen PR (Ursache unten). | capulus-core, `renovate.json` | `fix/renovate-wiki-sync-release-age`, `05ed5fb` |
| 3 | `argocd/bootstrap/projects.yaml` ist gegenüber dem Template der Rolle `argocd` veraltet (`metallb-system` fehlt). Drift stammt aus dem Monorepo (capulus-ansible `36f69f2`), nicht aus dem Split. Kein Live-Effekt: der AppProject kommt per `kubectl apply` aus dem Template, die Datei in core dient nur der CI-Validierung. | capulus-core, `argocd/bootstrap/projects.yaml` | `chore/bootstrap-projects-render`, `6af3bb7` (erzeugt mit `make render-bootstrap`) |
| 4 | Root-CA-Quellverweis `docs/assets/` im Template zeigt ins Leere (Quelle ist `pki/`). Gerenderte Objekte unverändert. | capulus-core, `argocd/apps/tech/github-release-watcher/templates/ca-configmap.yaml` | `chore/ca-configmap-quellpfad`, `5a552e2` |
| 5 | Vier tote Links: `capulus-core/tree/main/docs`, `docs/56-domain-tiers.md` (existierte schon vor dem Split nicht) und `argocd/apps/workloads/{github-release-watcher,alamos-apager}` (seit dem Umbau `platform → tech` unter `apps/tech`). | EDV-Kretzer, `projekt.html`, `data/apps.json` | `fix/broken-repo-links`, `4fe125e` |
| 6 | Planungsseite verlinkt die entfernte `.releaserc.json`. | pkr-lab-docs, `capulus-core/4-planung/40050-…` | dieser Branch `docs/nachpruefung-repo-aufspaltung` |
| 7 | Doku stimmte nicht mit dem Ist-Stand: Fallback-Zeitplan („05:00 UTC vor 6 Uhr“), Rulesets „aktiv“, Pflicht-Checks, „release.yml pusht keine Commits“. | pkr-lab-docs, `f0020`, `f0090`, `60020` | dieser Branch |

### Befunde ohne Fix (Entscheidung nötig)

| Nr | Befund | Hinweis |
|---|---|---|
| A | Renovate öffnet seit Juni 2026 in keinem Repo mehr PRs für Minor-/Major-Updates ([Renovate](#renovate)). | Betrifft auch den SHA-Pin des Release-Workflows. |
| B | Alle Rulesets in capulus-core sind seit 2026-10-18 deaktiviert, `CLAUDE.md` von capulus-core sagt weiterhin „`main` ist geschützt“ ([Rulesets](#rulesets)). | |
| C | `Deploy to netcup` in EDV-Kretzer schlägt bei jedem Push fehl: `missing server host`. Das Repo hat keine Secrets (`NETCUP_HOST`, `NETCUP_USER`, `NETCUP_SSH_KEY`, optional `NETCUP_PORT`, `NETCUP_PATH`). Besteht seit mindestens 2026-08-14, hängt nicht am Split. | Außerdem nutzt der Workflow `appleboy/ssh-action@v1.0.3` ohne SHA-Pin. |
| D | Authentik-Reste in capulus-core: `lldap` führt das Secret-Feld `authentik-bind-password` (`values.yaml`, `templates/sealedsecret.yaml`), `uptime-kuma/templates/ingress-native.yaml` verweist auf Authentiks Login-Seite und ein nicht mehr vorhandenes Doc, Kommentar in `immich/templates/postgres-backup-cronjob.yaml`. | Authentik gilt laut [030](../7-security-incidents/030-minio-oidc-secret-in-git-historie.md) als entfernt. Das Entfernen des Secrets ändert ein SealedSecret im laufenden Cluster, deshalb nicht angefasst. |
| E | Veraltete Pfade in Text und Kommentaren der n8n-Workflows (`docs/…`, `ansible/roles/nightly_worker_wake`) unter `argocd/apps/tech/n8n/workflows/*.json`. | Nur Beschreibungstexte. Eine Änderung ändert die Workflow-Definition im Cluster, deshalb nicht angefasst. |
| F | Die Pflicht-Check-Namen des gespeicherten Rulesets „main: PR + CI“ sind veraltet ([Rulesets](#rulesets)). | |
| G | `cubi`: Ohne Eintrag in `targets.yaml` sucht das Token-Register `pkr-lab-docs` unter `kubernetes/pkr-lab-docs`. Mit der vorhandenen Config (`docs_repo: ~/gitlab/pkr-lab-docs`) funktioniert es. | Kein Bruch, nur eine Falle bei frischer Installation. |
| H | Lokal war `core.hooksPath` in keinem Klon gesetzt. In capulus-core und capulus-ansible wurde `make hooks` ausgeführt. In den übrigen Repos ist der Hook vorhanden (alle zwölf Kopien identisch mit dem zentralen), aber nicht aktiv. | Reine Arbeitsplatz-Einstellung. |

---

## Renovate

### Warum das wiki-sync-Chart auf v1.0.0 stehen blieb

Der Lauf vom 2026-10-19 (Debug-Log) findet den Update `v1.0.0 → v1.1.0`, behandelt ihn aber als **pending**. Zwei Sperren greifen:

1. **Karenzzeit ohne Zeitstempel.** `minimumReleaseAge: 3 days` verlangt einen Release-Zeitpunkt. Renovate liest ihn bei Docker-Images aus dem Label `org.opencontainers.image.created`. Das Image `capulus-wiki-sync:v1.1.0` hat nur `source`, `version` und `revision` (der Build-Schritt in `pkr-lab/.github` setzt `created` nicht). Ohne Zeitstempel bleibt der Update dauerhaft „pending“, auch nach Ablauf der drei Tage.
2. **Zeitplan.** `before 6am on monday` (Europe/Berlin) wird vom Fallback-Workflow nie erreicht: Cron 05:00 UTC ist 06:00 bzw. 07:00 Uhr in Berlin, die Läufe starten real zwischen 09:59 und 12:49 UTC. Renovate meldet „Skipping branch creation as not within schedule“.

Ein Dry-Run der alten und neuen Regel auf einer Kopie des Charts zeigt den Unterschied: alt `pendingChecks: true`, neu keine Sperre. Der Fix (Nr. 2) setzt für dieses Image `minimumReleaseAge: null` und den Zeitplan „at any time“. Beim nächsten Renovate-Lauf (Montag oder „Run workflow“) entsteht der PR auf `v1.1.0`. `v1.1.0` enthält gegenüber `v1.0.0` nur die CI-Umstellung, kein Verhalten ändert sich.

Alternativ oder zusätzlich kann der zentrale Release-Workflow das Label setzen (`--label "org.opencontainers.image.created=$(date -u +%Y-%m-%dT%H:%M:%SZ)"`). Dafür braucht es ein neues Release von `pkr-lab/.github` und einen neuen Pin in den Aufrufern. Für `v1.1.0` hilft das nicht mehr, das Image ist gebaut.

### Wochenfenster ist für den Fallback nicht erreichbar (Befund A)

Alle Repos nutzen `"schedule": ["before 6am on monday"]`. Der Self-hosted-Lauf kann dieses Fenster nicht treffen. Im Log vom 2026-10-19 meldet Renovate für die gruppierten Branches „Skipping branch creation as not within schedule“; Ausnahme sind Patch-Updates (Regel „at any time“). Seit dem 2026-06-05 (PR #13, #14, GitHub-App) gibt es in capulus-core keinen Renovate-PR. Damit wird auch der SHA-Pin der zentralen Workflows nirgends nachgezogen.

Möglichkeiten, jeweils für `renovate.json` in capulus-core, capulus-ansible, capulus-ios, capulus-wiki-sync, pkr-lab-docs und `.github`:

| Variante | Wirkung | Aufwand |
|---|---|---|
| `"schedule": ["on monday"]` | Fenster ganzer Montag Berlin, passt zu Cron 05:00 UTC auch bei Verzögerung | eine Zeile je Repo |
| Cron früher und nicht zur vollen Stunde (`17 2 * * 1`) | trifft das Fenster meistens, bei GitHub-Verzögerung nicht sicher | eine Zeile in `renovate.yml`, unzuverlässig |
| Zeitplan entfernen | PRs jederzeit, jeder Lauf kann PRs öffnen | eine Zeile je Repo |

**Empfehlung:** `on monday`. Achtung: Beim ersten Lauf in capulus-core würden bis zu 5 PRs gleichzeitig entstehen (`prConcurrentLimit: 5`), weitere folgen in den Wochen danach. In capulus-core löst `commitMessagePrefix: "feat(deps):"` je gemergtem PR ein Minor-Release aus.

### Die sieben Repos ohne `renovate.json`

cubi, ui-terminal-config, DocFlowEngine, PowerShell, ps-data-fixtures, EDV-Kretzer und pkr-lab haben keine Renovate-Konfiguration. Ihr einziger Renovate-relevanter Inhalt ist der SHA-Pin der zentralen Workflows (EDV-Kretzer zusätzlich `appleboy/ssh-action`).

**Vorschlag ohne Onboarding-PRs und ohne neue Dateien in den sieben Repos:** Ein zweiter Schritt in `renovate.yml` von capulus-core ruft Renovate mit einer eigenen globalen Config für diese Repos auf:

```json
{
  "onboarding": false,
  "requireConfig": "ignored",
  "enabledManagers": ["github-actions"],
  "extends": ["helpers:pinGitHubActionDigests"],
  "timezone": "Europe/Berlin",
  "schedule": ["at any time"],
  "labels": ["dependencies"],
  "commitMessagePrefix": "ci(deps):",
  "minimumReleaseAge": "3 days",
  "vulnerabilityAlerts": { "enabled": false },
  "packageRules": [
    { "matchPackageNames": ["pkr-lab/.github"], "minimumReleaseAge": null, "groupName": "zentrale Workflows" }
  ]
}
```

Dazu `RENOVATE_REPOSITORIES` mit den sieben Repos und `RENOVATE_CONFIG_FILE` auf diese Datei. Getestet (Dry-Run mit Renovate 44.169.0, Kopie von ps-data-fixtures mit altem Pin `v1.0.0`): Renovate erkennt beide `uses:`-Zeilen, schlägt `v1.0.1` mit dem SHA `e44248cf042c1ec7d3d0aa0f99765c40b5565765` vor und bündelt beide in einem Branch `renovate/zentrale-workflows`.

Eigenschaften:

- Ein PR je Repo und Release von `pkr-lab/.github`, sonst nichts. Kein Onboarding-PR, kein Dependency-Dashboard.
- `ci(deps):` löst in den Zielrepos kein Release aus.
- Der Pin der zentralen Workflows kommt ohne Karenzzeit (eigenes Repo), andere Actions mit drei Tagen.
- Der `RENOVATE_TOKEN` braucht Schreibrechte auf die sieben Repos und die Berechtigung `Workflows`. Laut [f0020](../f-cicd-automatisierung/f0020-renovate.md) war er nur auf capulus-core gescoped, für die schon gelisteten Repos muss er bereits erweitert sein. Das ist nicht prüfbar, ohne den Token zu ändern.
- Die Schrittfolge sollte auf `workflow_dispatch` und Montag laufen und nicht vom Wochenfenster abhängen.

---

## Rulesets

- **Stand:** capulus-core hat drei Rulesets (`Protect MAIN`, `entw-1`, `main: PR + CI`), alle `disabled` seit 2026-10-18 (zuletzt geändert zwischen 15:29:25 und 15:29:41 Uhr). Alle anderen Repos haben weder Rulesets noch klassischen Branch-Schutz.
- **CHANGELOG-Commit:** Er funktioniert, weil nichts aktiv ist. Beleg: `chore(release): X [skip ci]` liegt direkt auf `main` in allen Repos (z. B. capulus-core `ac6fca8`, v3.52.0).
- **Wenn Rulesets wieder aktiv werden:** Der Commit kommt von `github-actions[bot]` mit `GITHUB_TOKEN` und verletzt die Regel „Pull Request“. Der `GITHUB_TOKEN` lässt sich nach Community-Berichten nicht als Bypass-Akteur eintragen; eine offizielle Aussage von GitHub dazu wurde nicht gefunden (Quellen: [Community-Diskussion 136531](https://github.com/orgs/community/discussions/136531), [Xygeni-Detektor](https://detectors.xygeni.io/xydocs/misconfigurations/detectors/github_rulesets.html)). Bewährt ist ein Token eines Akteurs mit Bypass:
  1. GitHub-App (oder Deploy-Key/PAT) anlegen, im Ruleset als Bypass-Akteur mit Modus „Always“ oder „Pull request“ eintragen.
  2. Im zentralen `release.yml` den Schritt `semantic-release` mit diesem Token statt `GITHUB_TOKEN` ausführen (neuer optionaler Secret-Input, neues Release von `pkr-lab/.github`).
  3. Vorher an einem Test-Repo mit aktivem Ruleset ausprobieren.
- **Vor dem Wiedereinschalten** `scripts/setup-main-ruleset.sh` ausführen: Das gespeicherte Ruleset verlangt `make lint (yamllint, ansible-lint, helm lint)` und `docs links`, beide Jobs gibt es nicht mehr. Das Skript enthält die aktuelle Liste (ohne `isc-news-bot (typecheck, unit und Mock-Tests)`; ob dieser Job Pflicht sein soll, ist offen).

---

## Funktionsgleichheit: Testmatrix

Legende: **ok** = belegt, **Fehler** = gefunden, **gefixt** = Fix als Commit vorbereitet, **offen** = nicht ausführbar, siehe [Live-Tests](#live-tests).

| Funktion | Prüfung | Ergebnis |
|---|---|---|
| Ansible: Auflösung | `ansible-playbook --syntax-check` für alle 11 Playbooks (`site`, `worker-0`, `worker-1`, `prod`, `entw`, `alarm-kiosks`, `banana-pi-kiosks`, `xibo-kiosks`, `alert-caller`, `render-bootstrap`, `vaultwarden-restore`) mit ansible-core 2.21.5 und den Collections aus `requirements.yml` | ok |
| Ansible: Inventory | `ansible-inventory --list`, 10 Gruppen, 9 Hosts | ok |
| Ansible: Vault-Werte, Secrets-Referenzen | Vault-Passwort liegt nicht vor | offen |
| Ansible: `--check --diff` gegen Hosts | SSH nicht erreichbar | offen |
| Ansible: `render-bootstrap` | Vergleich mit `argocd/bootstrap/` in core | Fehler (Drift `metallb-system`), gefixt (Nr. 3) |
| Ansible: `alert_caller` | Tag `v3.49.0` in core vorhanden, alle 3 Dateien und SHA256 stimmen, identisch auch auf `main` | ok |
| Semaphore-Bootstrap | Code gelesen, Repository-Update gegen Mock-API getestet | Fehler, gefixt (Nr. 1); Live offen |
| Argo CD: Quellen | alle `repoURL` (core), `path`, `targetRevision` (`main`) in `argocd/bootstrap*/`; Verzeichnisse existieren; jeder App-Ordner hat seinen Namespace im AppProject (TECH und PROD) | ok |
| Argo CD: Synced/Healthy, `argocd app diff` | kein Zugriff | offen |
| Helm: Rendern | `helm template` aller 41 Charts (34 direkt, 7 nach `helm dependency build` in Scratch-Kopie); 7 Ordner sind reine Manifeste | ok |
| Helm: `helm diff`, `kubectl --dry-run=server` | kein Zugriff | offen |
| Images | alle 7 selbstgebauten Referenzen (`alert-caller`, `carplay-api`, `isc-news-bot`, `n8n-custom`, `pacman` je `sha-…`; `capulus-wiki-sync` `v1.0.0` und `v1.1.0`) in GHCR vorhanden | ok |
| Token-Register | `check-token-register.py` (47 Einträge, 0 Probleme); 95 repoübergreifende Verweise (`capulus-ansible:…`, `pkr-lab-docs:…#Anker`) lösen auf | ok |
| iOS: Anbindung | alle 6 Routen der App (`/api/dashboard`, `/api/updates`, `/api/brightness` GET und PUT, `/api/power/wake`, `/api/power/shutdown`) gegen die Routen in `carplay-api/src/cmd/server/main.go`; Host `carplay-api.prod.homeserver` gegen `values.yaml`; WoL-Agent `:9102/wol` gegen die Rolle `banana_pi_kiosk` | ok |
| iOS: Build | `xcodebuild` nur unter macOS; letzter CI-Lauf (`xcodegen + xcodebuild`) auf `main` am 2026-10-18 15:31 grün | ok (über CI) |
| Root-CA | fünf Kopien (core `pki/`, ansible, ios, zwei Charts) byte-identisch | ok |
| wiki-sync | Chart-`env` gegen `config.py`; Image `v1.0.0` und `v1.1.0` vorhanden; Renovate-Ursache reproduziert | ok, Fix Nr. 2 |
| cubi | Token-Register-Lader mit der Config `~/gitlab/…`: 47 Token, 0 fehlende Runbooks | ok |
| Release-Kette | Release- und Commitlint-Workflow in allen 12 Aufrufer-Repos; überall SHA `e44248cf…` = Tag `v1.0.1`; PowerShell auf `main` und `master` (Trigger und `branches`-Input); Build-Skripte vorhanden; Hooks identisch | ok |
| CI-Stand | letzte Läufe aller Repos grün; Ausnahme `Deploy to netcup` (EDV-Kretzer) | Fehler (C) |
| Doku-Links | `scripts/check-doc-links.py`: 269 Dateien, 0 kaputte Links; 313 absolute GitHub-Links gegen lokale Dateien | ok (außer Nr. 5, 6) |

---

## Live-Tests

Die folgenden Schritte sind nicht gelaufen. Alle sind lesend, sofern nicht anders vermerkt. Aus einer Shell mit LAN-Zugriff:

```bash
kubectl get nodes -o wide
argocd app list
argocd app list -o json | jq -r '.[]|[.metadata.name,.spec.source.repoURL,.spec.source.path,.spec.source.targetRevision,.status.sync.status,.status.health.status]|@tsv'
argocd app diff <app>
helm template t argocd/apps/<cluster>/<app> | kubectl diff -f -
kubectl apply --dry-run=server -f argocd/bootstrap/
ansible-playbook site.yml --check --diff --ask-vault-pass
```

Zusätzlich sind diese Punkte nur live prüfbar:

| Punkt | Prüfung |
|---|---|
| Semaphore-Repository | In der Semaphore-Oberfläche Projekt `home-server` → Repositories → `home-server-git`: URL muss `https://github.com/pkr-lab/capulus-ansible.git` sein. Sonst `make semaphore-bootstrap` mit dem Fix ausführen (**ändert Semaphore-Konfiguration**, vorher ankündigen). Danach das Template „Deploy Home Server“ als Testlauf starten (falls die Semaphore-Version „Dry run“ anbietet, damit). |
| Semaphore-Verlauf | Letzte Läufe des täglichen Templates (06:00) seit 2026-10-18 ansehen. |
| Live-AppProject `tech` | `kubectl -n argocd get appproject tech -o yaml`: enthält `metallb-system` nur, wenn `make argocd` seit 2026-10-18 lief. |
| Argo CD Applications | `argocd app list`: alle `Synced` und `Healthy`, `repoURL` überall `…/capulus-core.git`. |
| wiki-docs-sync | Nach Merge und PR auf `v1.1.0`: `kubectl -n wiki-docs-sync get cronjob,job`. |

---

## Offene Entscheidungen

1. Renovate-Zeitplan ändern (`on monday`, siehe oben)? Gilt für sechs Repos.
2. Zweiten Renovate-Schritt für die sieben Repos ohne Config einführen? Token-Scope prüfen.
3. Rulesets: dauerhaft aus lassen oder wieder einschalten (dann Bypass-Mechanismus für den Release-Commit festlegen und das gespeicherte Ruleset korrigieren)? `CLAUDE.md` von capulus-core entsprechend anpassen.
4. Label `org.opencontainers.image.created` im zentralen Release-Workflow ergänzen?
5. Lange CHANGELOGs in pkr-lab-docs, capulus-ansible und capulus-ios: unverändert gelassen. Aktuell 327 Zeilen (pkr-lab-docs), 218 (capulus-ansible) und 30 (capulus-ios). Zu entscheiden, ob sie so bleiben, gekürzt oder ausgelagert werden.
6. `Deploy to netcup`: Secrets setzen oder Workflow deaktivieren?
7. Authentik-Reste (D) und n8n-Texte (E) entfernen? Beides ändert Cluster-Konfiguration.
8. Branches pushen und PRs öffnen: nichts ist gepusht.
