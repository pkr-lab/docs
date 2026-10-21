# Referenz: Skripte und Workflows in capulus-core

Nachschlagewerk für alle 20 Skripte unter [`scripts/`](https://github.com/pkr-lab/capulus-core/tree/main/scripts) und alle 15 Workflows unter [`.github/workflows/`](https://github.com/pkr-lab/capulus-core/tree/main/.github/workflows): Aufruf, Voraussetzungen, Umgebungsvariablen, Exit-Codes, Meldungen und Fehlerbilder. Die Begründungen hinter den Entscheidungen stehen in [600a0 CI-Workflows und Skripte](../6-hintergruende/600a0-ci-workflows-und-skripte.md), die Abläufe in [f0080 ENTW-Promotion](f0080-entw-promotion.md), [f00b0 Promotion-Kette](f00b0-promotion-chain.md) und [f0070 CI-Lint](f0070-ci-lint.md).

---

## Überblick

| Gruppe | Skripte | Workflows |
|---|---|---|
| Secrets versiegeln und Tokens | `seal-github-token.sh`, `seal-ntfy-tokens.sh`, `seal-alert-caller.sh`, `seal-alert-caller-token.sh`, `seal-restic-password.sh`, `seal-write-auth.sh`, `reseal-for-prod.sh`, `create-argocd-ci-token.sh`, `regenerate-grafana-admin.sh`, `show-headlamp-token.sh` | – |
| Token-Register | `check-token-register.py` | `token-watch.yml`, `ci.yml` (Job token-register) |
| Promotion | `promote-entw.sh`, `sync-entw.sh`, `promote-chain.py`, `promotelib.py`, `bump-image-tag.py` | `promote-entw.yml`, `entw-trigger.yml`, `sync-entw.yml`, `promote-chain.yml`, `build-images.yml` |
| Sicherheit | `security-issues.py`, `scan-cluster-images.sh` | `security-issues.yml`, `image-scan.yml`, `codeql.yml` |
| Repo-Verwaltung | `setup-main-ruleset.sh` | `commitlint.yml`, `release.yml`, `renovate.yml`, `mirror-gitlab.yml`, `tailscale-poc.yml`, `ci.yml` |

Gemeinsame Regeln aller Bash-Skripte: `set -euo pipefail`, Fehlermeldungen beginnen mit `Fehler:` und gehen auf stderr, Exit-Code ≠ 0 bei Fehlern. Alle `seal-*`-Skripte lesen den Klartext nie aus dem Repo, schreiben nur versiegelte Werte in `values.yaml` (per `sed`) und committen nichts.

---

## Skripte zum Versiegeln und für Tokens

Voraussetzung für alle `seal-*`-Skripte: `kubeseal` und ein `kubectl`-Kontext auf den Cluster, dessen Controller den Wert entschlüsseln soll (`--controller-namespace sealed-secrets --controller-name sealed-secrets-controller`). Hintergrund: [Secrets-Rotation](../d-sicherheit/d0060-secrets-rotation.md), [Sealed Secrets im Betrieb](../b-kubernetes-gitops/b0070-sealed-secrets.md).

| Skript | Zweck | Aufruf und Eingaben | Schreibt in |
|---|---|---|---|
| `seal-github-token.sh` | GitHub-Lese-Token für `github-release-watcher` (TECH) und `wiki-docs-sync` (PROD) versiegeln, je Cluster mit dessen Zertifikat | Umgebung `TOKEN_FILE` (Standard `~/.github_readonly_token`, Modus 600 oder 400), `PROD_SEALED_CERT` (Standard `~/prod-sealed-secrets.pem`), `OUT_ROOT`. Kontext muss TECH sein, `jq` nötig | `…/github-release-watcher/templates/sealedsecret-github-token.yaml`, `…/wiki-docs-sync/templates/sealedsecret-github-token.yaml`, setzt `github.tokenSecretName` |
| `seal-ntfy-tokens.sh` | ntfy-Zugangstoken (Publisher und Reader) in drei Charts | Umgebung `NTFY_CREDENTIALS` (Datei mit `publisher_token=tk_…` und `reader_token=tk_…`) | `ntfy-bridge`, `alamos-apager` (Publisher) und `carplay-api` (Reader) `values.yaml`, Block `ntfyToken` |
| `seal-alert-caller.sh` | SIP-Passwort des Linphone-Kontos für den `alert-caller` | fragt das Passwort verdeckt ab (kein `;`, `"` oder Leerzeichen) | `…/alert-caller/values.yaml`, Schlüssel `SIP_PASSWORD` |
| `seal-alert-caller-token.sh` | gemeinsames Token zwischen `monitoring` und `alert-caller` | Umgebung `ALERT_CALLER_CREDENTIALS` (Datei mit `alert_caller_token=<64 Hex>`) | `alert-caller/values.yaml` und `monitoring/values.yaml` |
| `seal-restic-password.sh` | Repository-Passwort für das verschlüsselte restic-Backup | `seal-restic-password.sh [vaultwarden\|n8n]`, Passwort abfragen oder leer lassen für ein zufälliges (wird einmal angezeigt) | `<app>/values.yaml` Block `resticBackup` |
| `seal-write-auth.sh` | Basic-Auth-Zugänge `vm-writer` und `logs-writer` (VictoriaMetrics, VictoriaLogs) | fragt das Vault-Passwort ab; Umgebung `WRITE_AUTH_VAULT_OUT` (Standard `~/write-auth-vault.yml`). Braucht `ansible-vault`, `openssl`, optional `htpasswd` | `monitoring/values.yaml` und `logging/values.yaml` (`writeAuth.encryptedUsers`), die Vault-Variablen für capulus-ansible in die Ausgabedatei |
| `reseal-for-prod.sh` | SealedSecrets einer TECH-App für den PROD-Cluster neu versiegeln | `reseal-for-prod.sh [--list] <app>`; Umgebung `PROD_SEALED_CERT`. Kontext muss **TECH** sein (das Skript liest die Secrets aus dem laufenden Cluster) | kopiert `apps/tech/<app>` nach `apps/prod/<app>`, wenn dort nichts liegt, und ersetzt alle SealedSecrets dort |
| `create-argocd-ci-token.sh` | Token für das ArgoCD-Konto `ci` erzeugen und als GitHub-Secret setzen | `create-argocd-ci-token.sh hub\|entw`; Umgebung `ARGOCD_ADMIN_PASSWORD` (sonst aus dem Initial-Secret), `TOKEN_TTL_DAYS` (Standard 365). Braucht `gh`, `jq`, `curl` | GitHub-Secret `ARGOCD_HUB_TOKEN` bzw. `ARGOCD_ENTW_TOKEN` |
| `regenerate-grafana-admin.sh` | Grafana-Admin-Passwort neu erzeugen und Grafana neu starten | `regenerate-grafana-admin.sh [--secret <name>]` (Standard `grafana-admin`). Braucht `kubectl`, `openssl` | Secret im Namespace `monitoring` (per `kubectl apply`), Neustart von `deployment/monitoring-grafana`. Das neue Passwort steht **im Terminal** (und damit in der Shell-Ausgabe) |
| `show-headlamp-token.sh` | kurzlebiges Anmelde-Token für Headlamp (ServiceAccount `headlamp-viewer`, Rolle `view`) | `show-headlamp-token.sh [--duration 1h] [--namespace headlamp] [--kubeconfig <pfad>]` | nichts, gibt das Token aus. Siehe [Headlamp](../3-apps-workloads/300n0-headlamp.md) |

### Meldungen und Fehlerbilder

| Meldung | Skript | Ursache | Lösung |
|---|---|---|---|
| `Fehler: <datei> fehlt` | seal-ntfy-tokens, seal-alert-caller-token, seal-alert-caller, seal-write-auth | Eingabedatei oder `values.yaml` nicht vorhanden | Pfad prüfen, Umgebungsvariable setzen |
| `Fehler: kubeseal nicht gefunden` | alle seal-* | `kubeseal` fehlt | installieren (gleiche Version wie der Controller) |
| `Fehler: Token-Datei <pfad> fehlt oder ist leer` | seal-github-token | `TOKEN_FILE` fehlt | Datei anlegen |
| `Fehler: <datei> muss Modus 600 haben (ist <modus>)` | seal-github-token | Token-Datei zu offen | `chmod 600 <datei>` |
| `Fehler: PROD-Zertifikat <pfad> fehlt` | seal-github-token, reseal-for-prod | PROD-Sealed-Secrets-Zertifikat nicht abgelegt | Zertifikat aus dem PROD-Cluster holen (`kubeseal --fetch-cert` mit PROD-Kontext) |
| `Fehler: TECH-Zertifikat nicht abrufbar (kubectl-Kontext = TECH?)` | seal-github-token | Kontext zeigt nicht auf TECH oder Controller nicht erreichbar | `kubectl config use-context` |
| `Fehler: kubeseal lieferte nichts fuer <name>` / `lieferte kein SealedSecret` | alle | Controller nicht erreichbar, Namespace oder Name falsch | Kontext und Controller prüfen (`kubectl -n sealed-secrets get pods`) |
| `Fehler: alert_caller_token in <datei> fehlt oder hat falsches Format` | seal-alert-caller-token | Schlüssel fehlt oder nicht 64 Hexzeichen | `openssl rand -hex 32` |
| `Fehler: <schlüssel> in <datei> fehlt oder hat falsches Format` | seal-ntfy-tokens | Token fehlt oder beginnt nicht mit `tk_` | Datei korrigieren |
| `Fehler: Schluessel SIP_PASSWORD fehlt in <values>` / `Block alertCallerToken fehlt in <values>` / `Block ntfyToken fehlt` / `resticBackup.encryptedPassword fehlt` / `writeAuth.encryptedUsers fehlt` | alle | Platzhalter in der `values.yaml` fehlt, `sed` würde nichts ersetzen | Struktur der `values.yaml` prüfen, Platzhalter ergänzen |
| `Fehler: das Passwort darf kein ; kein " und kein Leerzeichen enthalten` | seal-alert-caller | Sonderzeichen brechen die SIP-Konfiguration | anderes Passwort wählen |
| `Fehler: leeres Passwort` | seal-alert-caller | Eingabe leer | erneut ausführen |
| `Fehler: $values nicht gefunden (Aufruf: … [vaultwarden\|n8n])` | seal-restic-password | App-Name falsch | `vaultwarden` oder `n8n` angeben |
| `Fehler: $src existiert nicht (TECH-App?)` | reseal-for-prod | App nicht unter `apps/tech` | Namen prüfen |
| `WARNUNG: nicht-strikter Scope im Original, Ergebnis ist strikt (Name+Namespace).` | reseal-for-prod | Original hatte `cluster-wide` oder `namespace-wide` | prüfen, ob die App das braucht |
| `<datei>: (rendert leer, uebersprungen - deaktiviertes Template?)` | reseal-for-prod | `helm template` liefert das SealedSecret nicht (`enabled: false`) | Secret im Chart aktivieren oder ignorieren |
| `Das ArgoCD-Konto 'ci' existiert auf <ziel> noch nicht. …` | create-argocd-ci-token | Rolle in capulus-ansible noch nicht ausgerollt | `argocd_ci_account_enabled: true`, Playbook laufen lassen |
| `curl: (22) …` bei der Session | create-argocd-ci-token | Passwort falsch oder ArgoCD nicht erreichbar | `ARGOCD_ADMIN_PASSWORD` prüfen, Netz und CA |
| `Fehler: kubectl wird benötigt.` / `Fehler: kubeconfig nicht gefunden unter <pfad>` | show-headlamp-token | `kubectl` oder kubeconfig fehlt | installieren, `--kubeconfig` angeben |
| `Error: kubectl command not found.` / `Error: openssl is required …` / `Missing value for --secret` / `Unknown option: …` | regenerate-grafana-admin | fehlendes Werkzeug oder falsche Option | Werkzeug installieren, Optionen prüfen |
| `error: failed to create token … serviceaccounts "headlamp-viewer" not found` (von kubectl) | show-headlamp-token | Headlamp nicht ausgerollt oder falscher Namespace | `--namespace` prüfen |

---

## Token-Register

| Skript | Zweck | Aufruf | Exit-Code |
|---|---|---|---|
| `check-token-register.py` | prüft `ops/token-register.yaml`: Pflichtfelder (`id`, `service`, `kind`, `stored_in`, `rotation`, `owner`, `blast_radius`), erlaubte `kind` (`api-token`, `auth-key`, `password`, `encryption-key`, `ssh-key`, `tunnel-credential`, `certificate`), `rotation.mode` (`auto`, `scripted`, `manual`), Runbook, Datumsformat, `max_age_days`, überfällige Rotation, abgelaufene `expires`, `live_check`, Eindeutigkeit der `id` und dass jedes SealedSecret im Repo in einem `stored_in` vorkommt | im Repo-Wurzelverzeichnis: `python3 scripts/check-token-register.py` | 1 bei jedem Problem, sonst 0. Summenzeile auf stderr |
| `token-watch.py` | prüft Ablaufdaten, legt bei 60/30/14/7/1 Tagen ein Issue mit Label `token-expiry` an oder kommentiert es, schließt es bei Entwarnung | durch `token-watch.yml`; benötigt `gh`; Umgebung für Live-Prüfung (`github-pat`, `tailscale-device`) | 1, sobald ein Token unter 7 Tagen liegt (Lauf wird rot) |

| Meldung | Ursache | Lösung |
|---|---|---|
| `<id>: Pflichtfeld fehlt: <feld>` | Eintrag unvollständig | Feld ergänzen |
| `<id>: unbekannter kind: '<wert>' (erlaubt: …)` | Tippfehler | erlaubten Wert wählen |
| `<id>: rotation.mode fehlt oder unbekannt` / `rotation.runbook fehlt` | Rotationsangaben fehlen | ergänzen |
| `<id>: issued ist kein Datum (YYYY-MM-DD): '<wert>'` | falsches Format | `"JJJJ-MM-TT"` in Anführungszeichen |
| `<id>: max_age_days fehlt` / `muss eine positive Zahl sein` | Pflichtfeld | ergänzen |
| `<id>: Rotation ueberfaellig (issued <d>, max_age_days <n>)` | Frist verstrichen | rotieren, [Secrets-Rotation](../d-sicherheit/d0060-secrets-rotation.md), mit [cubi tokens](../../cubi/tokens.md) eintragen |
| `<id>: abgelaufen am <d>` | `expires` in der Vergangenheit | rotieren und Datum erneuern |
| `<id>: unbekannter live_check` / `live_check github-pat verlangt live_check_env` / `live_check tailscale-device verlangt live_check_ref` | `live_check` falsch konfiguriert | Felder ergänzen |
| `<id>: id mehrfach vergeben` | doppelter Eintrag | umbenennen |
| `SealedSecret ohne Register-Eintrag: <datei>` | neues SealedSecret nicht im Register | Eintrag mit `stored_in` anlegen |
| `Hinweis: issued unbekannt, Alter nicht pruefbar bei N von M Eintraegen` | `issued: null` | kein Fehler |
| `<id>: kein Ablaufdatum bekannt (<quelle>)` / `<id>: <n> Tage (<quelle>, faellig <datum>)` (token-watch) | Statuszeile je Token | – |
| `RuntimeError: gh issue … ` (token-watch) | `gh` ohne Token oder Rechte `issues: write` | `GITHUB_TOKEN` im Workflow prüfen |

---

## Promotion

Der Ablauf (ENTW → `main` → PROD) und die Gates stehen in [f0080](f0080-entw-promotion.md) und [f00b0](f00b0-promotion-chain.md). Hier die Skripte im Einzelnen.

| Skript | Zweck | Aufruf | Umgebung |
|---|---|---|---|
| `promote-entw.sh` | übernimmt neue Commits von `entw` per Cherry-Pick als PR nach `main` | Teilbefehle `detect`, `promote`, `report-ci-failure` | `PROMOTE_HEAD` (Pflicht bei `promote` und `report-ci-failure`), `GH_TOKEN` (Pflicht bei `promote`), `DRY_RUN`, `GITHUB_OUTPUT`, `GITHUB_REPOSITORY` |
| `sync-entw.sh` | hält `entw` auf dem Stand von `main`: PR `main → entw` mit Auto-Merge | ohne Argumente | `DRY_RUN`, `GH_TOKEN` |
| `promote-chain.py` | prüft die Gates (Stunden gesund auf ENTW bzw. TECH, neue Version ausgerollt, Smoke-Checks) und öffnet PRs mit Version-Feldern | `--stage entw\|tech`, `--status entw=<json> hub=<json>` oder `--argocd entw=<url> hub=<url>`, `--plan-only` | `ARGOCD_ENTW_TOKEN`, `ARGOCD_HUB_TOKEN`, `PROMOTE_NOW`, `DRY_RUN`, `AUTOMERGE_TECH`, `SMOKE_IP_ENTW`, `SMOKE_IP_TECH`, `ARGOCD_CA_FILE`, `GITHUB_STEP_SUMMARY`, `GITHUB_OUTPUT`, `GITHUB_REPOSITORY` |
| `promotelib.py` | Bibliothek: findet `image.tag` in `values.yaml` und exakte `dependencies[*].version` in `Chart.yaml` mit Textposition und ersetzt sie, ohne Kommentare oder Anführungszeichen zu verändern | – (Import) | – |
| `bump-image-tag.py` | setzt `image.tag` in einer `values.yaml` (nach dem Image-Build) | `bump-image-tag.py <values.yaml> <tag>` | Exit 0 = geändert, 3 = schon gleich, 1 = Fehler (kein oder mehrere `image.tag`) |

### Meldungen und Fehlerbilder

| Meldung | Skript | Ursache | Lösung |
|---|---|---|---|
| `GH_TOKEN fehlt: Repo-Secret PROMOTE_TOKEN (oder RENOVATE_TOKEN) anlegen. …` | promote-entw (`promote`) | kein PAT: PRs mit `GITHUB_TOKEN` lösen keine CI auf dem PR aus | Secret anlegen, siehe [f0030](f0030-release-automation.md) |
| `PROMOTE_HEAD fehlt` | promote-entw | Umgebung nicht gesetzt (Aufruf von Hand) | `PROMOTE_HEAD=$(git rev-parse origin/entw)` |
| `Fuer <sha> gibt es bereits ein offenes entw-promotion-Issue - kein neuer Versuch.` | promote-entw (`detect`) | ein früherer Versuch ist an CI oder Konflikt gescheitert | Issue lesen, Ursache beheben (neuer Push startet neuen Versuch) |
| `entw=<sha> base=<sha> neue Commits=<n> changed=<bool>` | promote-entw | Statuszeile | – |
| `Konflikt beim Uebernehmen von <sha>` + Issue „ENTW -> main: Konflikt bei …“ | promote-entw (`promote`) | Cherry-Pick-Konflikt in Dateien außerhalb von `argocd/apps/entw/` | Änderung von Hand nach `main` (eigener PR), dann den Tag `entw-promoted` setzen (Befehl steht im Issue) oder die Commit-Nachricht auf `entw` mit `[entw-only]` versehen |
| Issue „ENTW-CI rot bei <sha> - Promotion nach main blockiert“ | promote-entw (`report-ci-failure`) | CI auf `entw` fehlgeschlagen | Fehler auf `entw` beheben und pushen |
| `Nichts zu uebergeben (… x [entw-only], … bereits in main) - setze nur den Tag.` | promote-entw | nur ENTW-Änderungen | kein Fehler |
| PR im Entwurfsmodus mit Warnung „alten Layout“ | promote-entw | Commit legt Dateien unter `argocd/apps/platform\|workloads/` an | nach `tech/` oder `prod/` verschieben |
| `entw enthaelt main bereits, nichts zu tun.` / `entw hat eigene, noch nicht uebergebene Commits - kein Sync:` | sync-entw | Zustandsmeldungen | kein Fehler. Der Sync wartet, bis die eigenen Commits übergeben sind |
| `gh pr merge … --auto` meldet `clean status` | sync-entw | Auto-Merge nicht nötig, PR ist sofort mergebar | das Skript mergt dann direkt |
| `ARGOCD_ENTW_TOKEN fehlt` / `ARGOCD_HUB_TOKEN fehlt` | promote-chain | Token-Secret fehlt | [create-argocd-ci-token.sh](#skripte-zum-versiegeln-und-für-tokens) ausführen |
| `ArgoCD <url>: <fehler>` | promote-chain | ArgoCD nicht erreichbar (Tailnet, CA, Token abgelaufen) | `ARGOCD_CA_FILE`, Tailscale im Runner ([Tailscale-Runner](f00a0-tailscale-runner-poc.md)), Token erneuern |
| `git <befehl>: <stderr>` (RuntimeError) | promote-chain | Git-Aufruf scheiterte (Ref fehlt, Fetch-Tiefe) | Checkout mit `fetch-depth: 0`, Refs prüfen |
| `<values.yaml>: erwartet genau ein `image.tag`, gefunden <n>` | bump-image-tag | Chart hat kein oder mehrere `image.tag` | `values.yaml` anpassen oder die Bump-Logik nur für Charts mit genau einem Image nutzen |
| `unveraendert` (Exit 3) | bump-image-tag | Tag stimmt schon | kein Fehler, der Workflow öffnet keinen PR |

---

## Sicherheit und Repo-Verwaltung

| Skript | Zweck | Aufruf | Umgebung |
|---|---|---|---|
| `security-issues.py` | gleicht Code-Scanning- und Dependabot-Alerts mit Issues (Label `security`) ab | durch `security-issues.yml`; benötigt `gh` | `MIN_SEVERITY` (`low`, `medium`, `high`, `critical`, Standard `medium`), `DRY_RUN=1`, `GITHUB_REPOSITORY` |
| `scan-cluster-images.sh` | Trivy-Scan der Images (Cluster oder Charts) | `scan-cluster-images.sh [--summary] [--charts] [image …]` | `SEVERITY` (Standard `HIGH,CRITICAL`), `KUBECONFIG`, `TRIVY_IMAGE`; braucht `trivy` oder `docker`, bei `--charts` `helm` |
| `setup-main-ruleset.sh` | legt das Ruleset „main: PR + CI“ an oder aktualisiert es | `setup-main-ruleset.sh [--dry-run]`; braucht `gh` mit Admin-Recht, `jq` | – |

| Meldung | Ursache | Lösung |
|---|---|---|
| `::error::MIN_SEVERITY='<x>' ungueltig, erlaubt: low, medium, high, critical` (Exit 2) | falscher Wert | korrigieren |
| `::error title=Security-Alerts nicht abrufbar::<fehler>` (Exit 1) | Alerts ließen sich nicht lesen (Dependabot-Alerts im Repo aus, Rechte `security-events: read`, `vulnerability-alerts: read`) | Einstellungen in GitHub prüfen, [CodeQL und Dependabot](../d-sicherheit/d0080-codeql-dependabot.md) |
| `code-scanning: noch keine Analyse hochgeladen, keine Alerts.` | CodeQL lief noch nie | kein Fehler |
| `<key>: <n> Alert(s), <severity> -> neues Issue` / `-> Issue #<n> wieder geoeffnet` / `Issue #<n> aktualisiert` / `neue Alerts … in Issue #<n> gemeldet` / `keine offenen Alerts ab <severity> mehr -> Issue #<n> geschlossen` | Statuszeilen | – |
| `Fehler: trivy oder docker wird benötigt.` | weder Trivy noch Docker | installieren |
| `Scan abgebrochen: <image>` | Trivy-Fehler (Netz, Datenbank-Download, Image nicht ziehbar) | Zugriff auf die Registry, Trivy-DB prüfen |
| `gh: … Resource not accessible by integration` (setup-main-ruleset) | fehlende Rechte | mit Admin-Konto ausführen |

---

## Workflows

| Workflow | Auslöser | Zweck | Benötigte Secrets | Typische Fehlerursache |
|---|---|---|---|---|
| `ci.yml` | PR und Push auf `main`, manuell, `workflow_call` | Jobs: `lint` (yamllint, helm lint), `manifest-validate` (helm template, kubeconform), `go-check`, `isc-news-bot`, `token-register`, `secret-scan` (gitleaks). Bestandteil des Rulesets „main: PR + CI“ | `GITHUB_TOKEN` | Linter-Fehler, Schemafehler im Chart, `go test` rot, Register-Fehler, Secret im Diff ([f0070](f0070-ci-lint.md)) |
| `commitlint.yml` | PR | Conventional Commits | – | Commit-Nachricht im falschen Format |
| `release.yml` | Push auf `main` | semantic-release über den zentralen Workflow ([f0030](f0030-release-automation.md)) | siehe [github-workflows](../../github-workflows/index.md) | Commit-Typen ohne Release, Rechte |
| `codeql.yml` | Push, PR, wöchentlich Montag 03:41 UTC | CodeQL-Analyse ([d0080](../d-sicherheit/d0080-codeql-dependabot.md)) | – | Build-Fehler im Autobuild |
| `security-issues.yml` | täglich 06:35 UTC | `security-issues.py` | `github.token` | siehe Meldungen oben |
| `image-scan.yml` | wöchentlich Montag 05:30 UTC | Trivy über alle Chart-Images (`scan-cluster-images.sh --charts --summary`) | – | neue Funde mit Fix |
| `token-watch.yml` | täglich 06:05 UTC | `token-watch.py`, rot bei Ablauf in weniger als 7 Tagen | `GITHUB_TOKEN`, `PROMOTE_TOKEN`, `RENOVATE_TOKEN`, `TS_OAUTH_CLIENT_ID`, `TS_OAUTH_SECRET` | Token läuft ab, Rotation nötig |
| `build-images.yml` | Push auf `main` (Pfade der Images), PR, manuell | baut `pacman`, `carplay-api`, `n8n`, `alert-caller`, `isc-news-bot`, pusht nach `ghcr.io/pkr-lab`, öffnet Bump-PR (`bump-image-tag.py`) ([f0060](f0060-build-images.md)) | `GITHUB_TOKEN`, `RENOVATE_TOKEN` | Dockerfile-Fehler, Registry-Rechte |
| `entw-trigger.yml` | Push auf `entw` | startet `promote-entw.yml` auf `main` | – | – |
| `promote-entw.yml` | stündlich (Minute 7), manuell | Jobs: `detect`, `ci`, `ci-failed`, `promote`: übernimmt `entw` nach `main` ([f0080](f0080-entw-promotion.md)) | `RENOVATE_TOKEN` | Konflikt oder rote CI, erzeugt ein Issue |
| `sync-entw.yml` | Push auf `main`, stündlich (Minute 17), manuell | `sync-entw.sh` | `RENOVATE_TOKEN` | `entw` hat eigene, nicht übergebene Commits |
| `promote-chain.yml` | stündlich (Minute 17), manuell (`stage`: all, entw, tech; `plan_only`) | Gates prüfen, PRs ENTW → TECH → PROD ([f00b0](f00b0-promotion-chain.md)) | `ARGOCD_ENTW_TOKEN`, `ARGOCD_HUB_TOKEN`, `RENOVATE_TOKEN`, `TS_AUDIENCE`, `TS_OAUTH_CLIENT_ID` | ArgoCD im Tailnet nicht erreichbar, Token abgelaufen |
| `renovate.yml` | wöchentlich Montag 05:00 UTC, manuell | Self-hosted-Renovate als Rückfall ([f0020](f0020-renovate.md)) | `RENOVATE_TOKEN` | Token abgelaufen |
| `mirror-gitlab.yml` | Push auf `main`, Tags, alle 6 Stunden, manuell | Spiegelung nach GitLab ([f0050](f0050-gitlab-mirror.md)) | `GITLAB_MIRROR_REMOTE`, `GITLAB_MIRROR_USER`, `GITLAB_MIRROR_TOKEN` | Token oder Remote falsch |
| `tailscale-poc.yml` | nur manuell | Test: Runner tritt dem Tailnet bei ([f00a0](f00a0-tailscale-runner-poc.md)) | `TS_OAUTH_CLIENT_ID`, `TS_AUDIENCE` | Federated-Identity falsch konfiguriert |

---

## Checkliste „Workflow oder Skript schlägt fehl“

1. **Lauf öffnen:** GitHub → Actions → Workflow → fehlgeschlagener Schritt. Die Fehlerzeile steht meist am Ende des Schritts.
2. **Welcher Typ?** CI-Job (lint, kubeconform, go, register, gitleaks) → lokal nachstellen (`make lint`, `python3 scripts/check-token-register.py`, siehe [Nutzung](../nutzung.md#lokale-prüfungen-vor-dem-commit)).
3. **Secrets:** Fehlermeldung nennt ein Secret (`… fehlt`)? Unter Settings → Secrets prüfen, ob es existiert und nicht abgelaufen ist (Ablauf im [Token-Register](https://github.com/pkr-lab/capulus-core/blob/main/ops/token-register.yaml), Bedienung mit [cubi tokens](../../cubi/tokens.md)).
4. **Promotion-Kette hängt:** offene Issues mit Label `entw-promotion` oder `token-expiry` ansehen, dann `gh workflow run promote-chain.yml -f plan_only=true` und die Zusammenfassung lesen.
5. **Skript von Hand:** mit `DRY_RUN=1` (soweit unterstützt) und `bash -x scripts/<skript>.sh`. Das richtige `kubectl`-Context prüfen (`kubectl config current-context`), die meisten Skripte setzen TECH voraus.
6. **Stand auf `main` geschützt?** Direkte Pushes sind durch das Ruleset blockiert ([f0090](f0090-branch-schutz-main.md)).

Weitere Fehlerbilder im Cluster: [zentrale Fehlersuche](../2-betrieb-hardware/20070-troubleshooting-zentral.md).
