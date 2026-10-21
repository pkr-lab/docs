# CI — Pflicht-Gate für jeden PR

[`.github/workflows/ci.yml`](https://github.com/pkr-lab/capulus-core/blob/main/.github/workflows/ci.yml) prüft jeden
Pull Request gegen `main` (und jeden Push auf `main` selbst), bevor eine
Änderung beim ArgoCD-Sync gegen den Live-Cluster landet. Die sechs Jobs sind
die **Pflicht-Checks** des Rulesets [„main: PR + CI“](f0090-branch-schutz-main.md):
ein PR lässt sich erst mergen, wenn alle grün sind.

Angelegt im Rahmen von
[capulus-core/4-planung/40080-multi-cluster-entw-prod-tech.md](../4-planung/40080-multi-cluster-entw-prod-tech.md),
Baustein 6. Vorher existierte `make lint` nur lokal, und ein kaputtes Manifest
fiel erst beim Sync auf, nicht im PR.

Derselbe Workflow ist zugleich **wiederverwendbar** (`workflow_call` mit dem
Input `ref`): [`promote-entw.yml`](f0080-entw-promotion.md) ruft ihn gegen den
Branch `entw` auf, bevor ein PR nach `main` entsteht.

---

## Sechs parallele Jobs

| Job (= Check-Name) | Prüft | Tool |
|---|---|---|
| `make lint (yamllint, ansible-lint, helm lint)` | YAML-Syntax (`ansible/`, `argocd/`), Ansible-Best-Practices, `helm lint` je Chart | ruft schlicht `make lint` auf — dieselbe Prüfung wie lokal |
| `helm template \| kubeconform` | rendert jeden Chart und validiert die entstandenen Ressourcen gegen echte API-Schemas; prüft zusätzlich **reine Manifest-Ordner** (ohne `Chart.yaml`) und `argocd/bootstrap*` | [kubeconform](https://github.com/yannh/kubeconform) mit CRD-Katalog |
| `go build, vet, test, tidy` | jedes Go-Modul im Repo: `go mod tidy -diff`, `go build`, `go vet`, `go test` | Go (`stable`); Module per `git ls-files` gefunden |
| `docs links` | läuft seit der Verlagerung der Doku nicht mehr in capulus-core, sondern im Repo pkr-lab-docs (zusätzlich mit lychee): relative Links und Überschriften-Anker in allen `*.md` (Slug-Regeln wie auf GitHub) | [`scripts/check-doc-links.py`](https://github.com/pkr-lab/pkr-lab-docs/blob/main/scripts/check-doc-links.py) |
| `token register` | Pflichtfelder, Datumsformat und `max_age_days` in [`ops/token-register.yaml`](https://github.com/pkr-lab/capulus-core/blob/main/ops/token-register.yaml), überfällige Rotation (`issued` plus `max_age_days` in der Vergangenheit), abgelaufene `expires`, und dass jede `SealedSecret`-Datei im Repo dort auftaucht | [`scripts/check-token-register.py`](https://github.com/pkr-lab/capulus-core/blob/main/scripts/check-token-register.py), siehe [40090](../4-planung/40090-standby-cluster-ha-paar.md#token-management) |
| `gitleaks` | Plaintext-Secrets in PRs, bevor sie versiegelt werden | [gitleaks](https://github.com/gitleaks/gitleaks), Konfiguration siehe unten |

### Warum der CRD-Katalog

kubeconform kennt nur Kubernetes-Kernressourcen. Für alles andere
(`Application`, `ApplicationSet`, `AppProject`, `SealedSecret`,
cert-manager, VictoriaMetrics, …) übersprang die erste Fassung die Prüfung
(`-ignore-missing-schemas`) — auch die Manifeste, für die `ci.yml` eigentlich
gedacht war (das `spec.generators[0].template`-Problem aus
[b0020-argocd-projects.md](../b-kubernetes-gitops/b0020-argocd-projects.md)).
Der [CRDs-catalog](https://github.com/datreeio/CRDs-catalog) liefert Schemas für
diese Typen; ohne ihn waren beim Test 10 von 51 Ressourcen in den Manifest-Ordnern
„skipped“, mit ihm 0. `-ignore-missing-schemas` bleibt für CRDs, die auch der
Katalog nicht kennt. Die Schemas kommen bei jedem Lauf per HTTPS von
`raw.githubusercontent.com`; gegen gelegentliche Verbindungsabbrüche
(`connection reset`) wiederholt der Job den Aufruf bis zu dreimal, echte
Validierungsfehler sofort nicht.

### Was `manifest-validate` abdeckt

1. **Helm-Charts** (`argocd/apps/*/*/` mit `Chart.yaml`): `helm dependency build`,
   `helm template`, dann validieren. Ein fehlgeschlagenes `helm template` bricht den
   Job laut ab — früher reichte die Pipe die Fehlermeldung an kubeconform weiter, die
   als „0 Ressourcen“ durchging.
2. **Reine Manifest-Ordner** (`nas-storage`, `traefik-config`, `immich-storage`,
   `coredns-custom`): werden von ArgoCD direkt angewendet und waren bisher ungeprüft.
3. **Bootstrap** (`argocd/bootstrap`, `argocd/bootstrap-prod`, inkl. der
   Migrations-Jobs): Hub- und PROD-ApplicationSets, AppProjects.

### Warum `go-check`

Anlass: Ein Renovate-Merge (`geoip2-golang` v1 → v2, 2026-09-03) hat pacmans
`go.mod` beschädigt — die v1-Zeile wurde überschrieben, der Code importierte
weiter den v1-Pfad. Kein Workflow übersetzte den Go-Code vor dem Merge. Der
[Image-Build](f0060-build-images.md) lief drei Minuten nach dem Merge rot
(`no required module provides package github.com/oschwald/geoip2-golang`), aber
erst *nach* dem Merge und ohne Benachrichtigung — der Fehler blieb 16 Tage
unbemerkt (das laufende Image `v8` war nicht betroffen). Der Job hätte den PR
rot gemacht.
`go mod tidy -diff` fängt zusätzlich halb aktualisierte Abhängigkeiten
(überzählige `go.sum`-Zeilen, ungenutzte `require`-Einträge). Es gibt aktuell
keine `_test.go`-Dateien, `go test` meldet „no test files“ und schlägt bei neuen
Tests automatisch an.

Die Module werden nicht fest verdrahtet: der Job greift dadurch auch auf `entw`
(anderes Ordner-Layout) und für künftige Go-Dienste ohne Anpassung.

### Warum `docs links`

Nach dem Layout-Umzug (`platform`/`workloads` → `tech`) und dem Entfernen einzelner Docs zeigten **26 Links** ins
Leere (z. B. `../../../docs/…` in den READMEs unter `argocd/apps/tech/`, das jetzt eine Ebene tiefer liegt), dazu
Anker auf Überschriften, die es so nicht gibt. Alle sind repariert; der Job verhindert neue. Er nutzt bewusst kein
externes Tool: ein eigenes Skript mit den GitHub-Slug-Regeln (Sonderzeichen wie `/` und `&` entfallen, Umlaute
bleiben) ist ohne Netzzugriff und ohne Action-Version reproduzierbar. Codeblöcke und Inline-Code werden ignoriert.
Lokal: `python3 scripts/check-doc-links.py`.

### gitleaks-Konfiguration

- [`.gitleaks.toml`](https://github.com/pkr-lab/capulus-core/blob/main/.gitleaks.toml) erweitert die Standardregeln um **eine Ausnahme**: den Chiffretext von
  SealedSecrets (Base64, beginnt mit `Ag`, mindestens 200 Zeichen). Ohne sie meldete die Regel `generic-api-key` den
  verschlüsselten Wert von Feldern namens `token:` als Klartext-Token (der Lauf nach Commit `433afb3` wurde rot,
  und mit Pflicht-Checks hätte jeder PR mit einem neuen SealedSecret hier gehangen). Die Ausnahme hängt am **Wert**,
  nicht am Dateipfad: ein versehentlich im Klartext eingetragenes Token in einer `sealedsecret*.yaml` wird weiter
  gemeldet (getestet).
- [`.gitleaksignore`](https://github.com/pkr-lab/capulus-core/blob/main/.gitleaksignore) führt **7 Altfunde in der Git-Historie** auf. Ohne die Liste scheitert
  jeder Lauf, der die ganze Historie scannt (Cron/`workflow_call` der [ENTW-Promotion](f0080-entw-promotion.md)),
  dauerhaft. Fünf sind harmlos (Platzhalter in alten Doku-Beispielen, eine öffentliche UUID im pacman-Manifest).
  **Zwei sind echte Altlasten und gelten als kompromittiert**, weil das Repo öffentlich ist: ein OpenSSH-Private-Key
  der Banana Pis (Commit vom 2026-08-10, in `8625d54` entfernt) und ein `clientSecret` in einer alten MinIO-`values.yaml`
  (Klartext, im aktuellen Stand nicht mehr vorhanden). Zugehörige Zugangsdaten rotieren, sofern noch in Benutzung
  (MinIO-Client in Authentik neu erzeugen). Der Banana-Pi-Schlüssel (Fingerprint `SHA256:G+1ezIam4dN0UiJW3CTXSUKj6L4LPrI89LZH0BFIKqw`)
  war am 2026-10-17 auf keinem erreichbaren Host autorisiert (homeserver, worker-1, prod-vm, entw-vm, vereinsheim-alarmmonitor,
  infotafel), ein Login mit ihm wird auf beiden Pis abgewiesen. Er bleibt im Git-Verlauf und deshalb in der Ignore-Liste; wer
  ihn auf einem neuen oder wieder eingeschalteten Host findet, entfernt den Eintrag aus `authorized_keys`
  (`ssh-keygen -lf ~/.ssh/authorized_keys | grep 'G+1ezIam4dN0UiJW3CTXSUKj6L4LPrI89LZH0BFIKqw'`). Die Datei nennt das bei den
  Einträgen.

---

## Was der Workflow NICHT tut

- **Kein Deploy, kein Zugriff auf den Cluster** — reine Statik-Prüfung,
  `permissions: contents: read`.
- **Kein Ersatz für `helm dependency update`** bei vendorisierten
  Subcharts — `Chart.lock`/`charts/*.tgz` werden weiterhin manuell
  gepflegt (siehe `renovate.json`, dieselbe Ausnahme gilt hier).
- **Kein Ansible-Lauf gegen echte Hosts** — `ansible-lint` prüft nur
  Syntax/Best-Practices der Playbooks/Rollen selbst.
- **Kein Bau der Container-Images** — das macht
  [`build-images.yml`](f0060-build-images.md) (ohne Push auch auf PRs).
- **Kein Code-Scanning und keine Prüfung auf verwundbare Abhängigkeiten** — das
  machen CodeQL und Dependabot, beide ohne Pflicht-Check
  ([d0080](../d-sicherheit/d0080-codeql-dependabot.md)).

## Image-Scan der laufenden Images (lokal, nicht in der CI)

Die CI prüft keine Container-Images auf Schwachstellen. Dafür gibt es [`scripts/scan-cluster-images.sh`](https://github.com/pkr-lab/capulus-core/blob/main/scripts/scan-cluster-images.sh): Es liest alle Images aller Pods per `kubectl` und scannt sie mit Trivy (lokal installiert oder per Docker `aquasec/trivy`), nur behebbare Funde (`--ignore-unfixed`) mit Schwere `HIGH` und `CRITICAL` (`SEVERITY=CRITICAL ./scripts/scan-cluster-images.sh` ändert das).

```bash
./scripts/scan-cluster-images.sh --summary                              # eine Zeile je Image: CRITICAL und HIGH
./scripts/scan-cluster-images.sh louislam/uptime-kuma:2.5.6              # Details für ein Image
```

Die erste Fassung der Tabelle steht im Incident-Audit. Ein voller Lauf braucht je Image rund eine Minute (Datenbank und Image-Layer werden geladen), deshalb eher wöchentlich von Hand als pro PR. Die meisten Funde stecken in den Basis-Paketen der Upstream-Images (das neueste Release von uptime-kuma oder Semaphore trägt sie ebenfalls), nicht in veralteten Tags; Abhilfe ist ein Image-Wechsel oder warten auf ein Rebuild des Herstellers. Zusätzlich läuft wöchentlich (montags 05:30 UTC) und auf Abruf der Workflow [`image-scan.yml`](https://github.com/pkr-lab/capulus-core/blob/main/.github/workflows/image-scan.yml): `./scripts/scan-cluster-images.sh --charts --summary` liest die Images aus `helm template` aller Charts (ohne Cluster-Zugriff), scannt sie mit Trivy im Container (`aquasec/trivy:0.76.0`) und schreibt die Tabelle in die Job-Zusammenfassung. Der Job **blockiert nichts** (kein Pflicht-Check, keine Fehlerschwelle), er zeigt nur, wo sich Funde ansammeln. Die geladenen Trivy-Datenbanken werden nicht gecacht, ein Lauf dauert deshalb lang (Timeout 90 Minuten).

## Unterschiede zwischen lokal und CI

Die Runner nutzen die jeweils neueste Helm-Version (`azure/setup-helm`, aktuell
Helm 4). Lokal mit Helm 3 zeigt kubeconform bei `zammad` einen Fehlalarm:
`updateStrategy.rollingUpdate.maxUnavailable: null` (in den Values absichtlich
gesetzt) rendert Helm 3 als leeres Feld, Helm 4 lässt es weg. Für den Cluster
ist das folgenlos (leer heißt „nicht gesetzt“). Maßgeblich ist der CI-Lauf.

---

## Historie: der erste Lauf (2026-09-18)

`.yamllint` (Repo-Root) existierte vorher nicht, obwohl `make lint` es
referenziert (`yamllint -c .yamllint ...`) — wurde zusammen mit dem Workflow
ergänzt.

Der erste echte CI-Lauf ist prompt fehlgeschlagen — und hat damit genau den
Zweck erfüllt, für den `ci.yml` gebaut wurde: einen vorher nie aufgefallenen
Bug in `make lint` selbst aufgedeckt. `yamllint` versuchte, **Helm-Templates als
rohes YAML zu parsen** — Go-Template-Ausdrücke wie `{{- include "x.labels" . |
nindent 4 }}` sind vor dem Rendern kein gültiges YAML
(`argocd/apps/*/*/templates/**/*.yaml`, ausnahmslos alle Charts). Fix:
`.yamllint` ignoriert `templates/` komplett — die Syntax dort prüfen weiterhin
`helm lint` und `helm template` (Job `manifest-validate`).

Seitdem ist CI auf `main` durchgehend grün (bestätigt an den Läufen vom
2026-09-19). Ausnahme war ein einzelner abgebrochener Lauf durch die
`concurrency`-Regel (`cancel-in-progress`, ein neuerer Push ersetzte ihn).

---

## Fehlersuche

Die Jobs lassen sich lokal nachstellen:

```bash
make lint                                                       # yamllint und helm lint
helm template argocd/apps/<cluster>/<app> | kubeconform -strict -ignore-missing-schemas -summary
python3 scripts/check-token-register.py
python3 scripts/check-doc-links.py                              # Links in pkr-lab-docs
(cd argocd/apps/tech/carplay-api/src && go build ./... && go vet ./... && go test ./...)
```

| Symptom | Ursache | Lösung |
|---|---|---|
| `yamllint`: `line too long` / `wrong indentation` | Formatierung | Zeile anpassen, `.yamllint` im Repo beachten |
| `helm lint`: `chart metadata is missing …` oder Abhängigkeiten fehlen | `Chart.lock` veraltet | `helm dependency update argocd/apps/<cluster>/<app>` und `Chart.lock` committen |
| `kubeconform`: `could not find schema` | CRD ohne Schema im Katalog | `-ignore-missing-schemas` ist gesetzt, ein echter Fehler steht darüber (`Invalid`) |
| `kubeconform`: `Invalid` mit Feldname | Manifest verstößt gegen das Schema | Feld korrigieren |
| `go build, vet, test, tidy` rot | Code oder `go.mod` | `go mod tidy` und committen |
| `token register` rot | siehe [f00c0](f00c0-skripte-und-workflows-referenz.md#token-register) | – |
| `gitleaks` rot | Geheimnis im Diff oder in der Historie | entfernen und rotieren, Platzhalter-Treffer in `.gitleaksignore` freigeben |
| Doku-Linkprüfung rot | relativer Link oder Anker kaputt | `python3 scripts/check-doc-links.py` im Docs-Repo |
| Pflicht-Check bleibt `pending` | Check-Name im Ruleset passt nicht zum Job | [f0090](f0090-branch-schutz-main.md), `scripts/setup-main-ruleset.sh` |

Allgemeine Fehlersuche: [zentrale Fehlersuche](../2-betrieb-hardware/20070-troubleshooting-zentral.md).
