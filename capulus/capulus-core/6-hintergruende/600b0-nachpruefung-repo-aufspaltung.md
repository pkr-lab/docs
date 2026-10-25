# Nachprüfung nach der Repo-Aufspaltung — Befunde, Fixes und offene Entscheidungen

Stand 2026-10-19. capulus-core wurde am 2026-10-18 in capulus-core, capulus-ansible, capulus-ios und capulus-wiki-sync getrennt, die Doku zog nach pkr-lab-docs, und die Releases laufen über die zentralen Workflows in `pkr-lab/.github` (v1.0.1). Diese Seite hält fest, was dabei kaputtgegangen ist, was repariert wurde, was geprüft wurde und was noch zu entscheiden ist.

> **Stand der Prüfung:** Die erste Runde lief ohne LAN-Zugriff, nur statisch. Am Abend des 2026-10-19 kamen die lesenden Live-Tests dazu (`kubectl`, Ansible-`ping`, Check-Modus auf dem homeserver). Nicht möglich waren Läufe, die das Vault-Passwort brauchen (worker-0/1, PROD-Cluster-Kontext, Semaphore-Anmeldung) und das ArgoCD-CLI (Token abgelaufen). Siehe [Live-Tests](#live-tests).

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
| C | `Deploy to netcup` in EDV-Kretzer schlägt bei jedem Push fehl: `missing server host`. Das Repo hat keine Secrets (`NETCUP_HOST`, `NETCUP_USER`, `NETCUP_SSH_KEY`, optional `NETCUP_PORT`, `NETCUP_PATH`). Besteht seit mindestens 2026-08-14, hängt nicht am Split. | Zusätzlich war der Workflow falsch: Auf dem Hosting ist `httpdocs` kein Klon, sondern der Checkout eines Bare-Spiegels unter `~/git/EDV-Kretzer`, `git pull` im Docroot konnte nie funktionieren. Behoben durch einen Deploy über den Spiegel, Secrets-Werte und Ablauf in [installation.md](../../../EDV-Kretzer/installation.md#deployment-auf-netcup). Der Workflow nutzt außerdem `appleboy/ssh-action@v1.0.3` ohne SHA-Pin. |
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

Alle Repos nutzten `"schedule": ["before 6am on monday"]`. Der Self-hosted-Lauf kann dieses Fenster nicht treffen. Im Log vom 2026-10-19 meldet Renovate für die gruppierten Branches „Skipping branch creation as not within schedule“; Ausnahme sind Patch-Updates (Regel „at any time“). Seit dem 2026-06-05 (PR #13, #14, GitHub-App) gibt es in capulus-core keinen Renovate-PR.

**Entscheidung (2026-10-19):** Zeitplan `on monday` (ganzer Montag) und Renovate nur in den vier capulus-Repos. Umgesetzt in capulus-core (`renovate.json`, `renovate.yml`), capulus-ansible, capulus-ios und capulus-wiki-sync. Hinweis: Beim ersten Lauf in capulus-core entstehen bis zu 5 PRs gleichzeitig (`prConcurrentLimit: 5`), weitere in den Wochen danach. `commitMessagePrefix: "feat(deps):"` löst dort je gemergtem PR ein Minor-Release aus. Die `renovate.json` in pkr-lab-docs und `.github` bleiben liegen, werden vom Fallback-Lauf aber nicht mehr bearbeitet.

### Die sieben Repos ohne `renovate.json`

cubi, ui-terminal-config, DocFlowEngine, PowerShell, ps-data-fixtures, EDV-Kretzer und pkr-lab haben keine Renovate-Konfiguration. Ihr einziger Renovate-relevanter Inhalt ist der SHA-Pin der zentralen Workflows (EDV-Kretzer zusätzlich `appleboy/ssh-action`).

**Entscheidung (2026-10-19):** Renovate läuft nur in den capulus-Repos, für diese sieben Repos wird nichts eingerichtet. Neue Versionen von `pkr-lab/.github` (SHA-Pin in `release.yml` und `commitlint.yml`) werden dort von Hand nachgezogen. Der folgende Vorschlag blieb ungenutzt und gilt als Option, falls sich das ändert.

**Option ohne Onboarding-PRs und ohne neue Dateien in den sieben Repos:** Ein zweiter Schritt in `renovate.yml` von capulus-core ruft Renovate mit einer eigenen globalen Config für diese Repos auf:

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
| Ansible: Erreichbarkeit | `ansible all -m ping`: homeserver, prod-vm, infotafel, vereinsheim-alarmmonitor erreichbar; worker-0/1 brauchen Vault (`become_pass`); entw-vm nicht erreichbar (liegt auf worker-1); ugreen-nas SSH verweigert; alert-caller-vm hat den Platzhalter `alert-caller-vm-ip-eintragen` | ok / offen |
| Ansible: `--check --diff` homeserver | `site.yml --limit homeserver`: 143 Tasks ok, Abbruch an der ersten vault-abhängigen Variable (`semaphore_secrets`); gemeldete Änderungen nur Rauschen (apt-Cache, Download-Tasks, Dateien in `/tmp`). `ssh_hardening` bricht im Check-Modus ab (der Assert liest `sshd -T`, das im Check-Modus nicht läuft) und wurde übersprungen. Auf dem Host wurde nichts verändert, die sshd-Datei besteht weiter. | teilweise ok |
| Ansible: `render-bootstrap` | Vergleich mit `argocd/bootstrap/` in core | Fehler (Drift `metallb-system`), gefixt (Nr. 3) |
| Ansible: `alert_caller` | Tag `v3.49.0` in core vorhanden, alle 3 Dateien und SHA256 stimmen, identisch auch auf `main` | ok |
| Semaphore-Bootstrap | Code gelesen, Repository-Update gegen Mock-API getestet; Pod `semaphore` läuft, die Konfiguration im Programm ist ohne Admin-Anmeldung nicht lesbar | Fehler, gefixt (Nr. 1); Live offen |
| Live-AppProject `tech` | enthält `metallb-system` nicht, entspricht dem Stand in core. Die Drift wird erst beim nächsten `make argocd` (Ansible-Template) im Cluster wirksam. | ok |
| Argo CD: Quellen | alle `repoURL` (core), `path`, `targetRevision` (`main`) in `argocd/bootstrap*/`; Verzeichnisse existieren; jeder App-Ordner hat seinen Namespace im AppProject (TECH und PROD) | ok |
| Argo CD: Quellen, Synced/Healthy | `kubectl -n argocd get applications`: 45 Applications, alle `repoURL` capulus-core, `path` und `targetRevision main` korrekt, alle `Synced`, 44 `Healthy`, `ollama` `Progressing` (Pod `Pending`, weil `worker-1` ausgefallen ist: „Kubelet stopped posting node status“; kein Split-Thema) | ok |
| Argo CD: `argocd app diff` | CLI-Token abgelaufen (`token is expired`), Anmeldung nicht durchgeführt. Da alle Apps `Synced` sind, gibt es keinen Diff. | ok (indirekt) |
| Helm: Rendern | `helm template` aller 41 Charts (34 direkt, 7 nach `helm dependency build` in Scratch-Kopie); 7 Ordner sind reine Manifeste | ok |
| Helm: `helm diff`, `kubectl --dry-run=server` | nicht nötig, da alle Apps `Synced` sind | – |
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

Gelaufen am 2026-10-19 (alles lesend): `kubectl get nodes`, `kubectl -n argocd get applications/appprojects`, `ansible all -m ping`, `ansible-playbook site.yml --check --diff --limit homeserver`, `ssh homeserver ls`. Ergebnisse stehen in der Testmatrix.

Noch offen, jeweils mit Befehl bzw. Handgriff:

| Punkt | Prüfung |
|---|---|
| ArgoCD-CLI | `argocd login argocd.tech.homeserver` (Token abgelaufen), danach `argocd app list`. |
| Semaphore-Repository | In der Oberfläche Projekt `home-server` → Repositories → `home-server-git`: URL muss `https://github.com/pkr-lab/capulus-ansible.git` sein. Sonst nach dem Merge von Fix Nr. 1 `make semaphore-bootstrap` ausführen (**ändert Semaphore-Konfiguration**). Dazu den Verlauf des täglichen 06:00-Templates seit 2026-10-18 ansehen. |
| Ansible mit Vault | `make check` bzw. `make worker-0-check` mit Vault-Passwort. Vorher `ssh_hardening` im Check-Modus beachten (bricht ab). |
| PROD-Cluster | Kubeconfig-Kontext für PROD (VM `prod-vm`) fehlt auf diesem Rechner; wiki-docs-sync dort nach dem Renovate-PR auf `v1.1.0` prüfen: `kubectl --context prod -n wiki-docs-sync get cronjob,job`. |
| worker-1 | Seit mindestens einem Tag ausgefallen (`NotReady`, Tailscale „offline“). Betrifft ollama und die ENTW-VM. Ursache nicht untersucht. |

---

## Offene Entscheidungen

Entschieden am 2026-10-19: Renovate-Zeitplan `on monday`, Renovate nur in den vier capulus-Repos, lange CHANGELOGs bleiben unverändert, Rulesets werden bald vom Betreiber wieder eingeschaltet, Branches und PRs werden geöffnet.

1. Rulesets: Beim Wiedereinschalten Bypass-Mechanismus für den Release-Commit festlegen und das gespeicherte Ruleset korrigieren ([Rulesets](#rulesets)). `CLAUDE.md` von capulus-core entsprechend anpassen.
2. `Deploy to netcup`: Secrets in EDV-Kretzer hinterlegen (Werte in [installation.md](../../../EDV-Kretzer/installation.md#deployment-auf-netcup)) und den Workflow einmal per „Run workflow“ prüfen.
3. Label `org.opencontainers.image.created` im zentralen Release-Workflow: nicht nötig, solange die Regel für das wiki-sync-Image gilt. Nur sinnvoll, wenn weitere versionierte Images dazukommen, für die dieselbe Karenzzeit gelten soll.
4. Authentik-Reste (D) und n8n-Texte (E): bekannt, bleiben liegen.
