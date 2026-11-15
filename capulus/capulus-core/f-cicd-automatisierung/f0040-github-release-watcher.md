# GitHub Release Watcher → Zammad-Benachrichtigung + App-Update-Liste

Der `github-release-watcher` pollt periodisch die GitHub-Releases-API für
eine Liste konfigurierter Repos (aktuell 8, siehe `values.yaml`) und macht
daraus zwei getrennte Dinge:

1. **Zammad-Ticket** — nur für Repos mit `notifyZammad: true` (aktuell nur
   `DocFlowEngine`); die übrigen 7 (Nextcloud, Immich, Vaultwarden,
   Paperless-ngx, n8n, Wiki.js, Zammad selbst) erzeugen
   bewusst **kein** Ticket.
2. **App-Update-Liste** — bei **jedem** Lauf wird für **alle** Repos
   `currentVersion` (aus `values.yaml`, manuell gepflegt) gegen die gerade
   abgefragte GitHub-Version verglichen und das Ergebnis in eine eigene
   `updates`-ConfigMap geschrieben. `carplay-api` liest genau diese
   ConfigMap (siehe `templates/role.yaml`, RoleBinding in den
   `carplay-api`-Namespace) und zeigt sie der iOS-App als Update-Status pro
   Dienst — unabhängig davon, ob ein Repo Zammad-Tickets erzeugt oder nicht.

Die Deployment-Konfiguration liegt unter `argocd/apps/tech/github-release-watcher/`.

---

## Übersicht

| Komponente     | Technologie                          | Namespace                 |
|----------------|---------------------------------------|----------------------------|
| Watcher        | Python 3.12 stdlib, `CronJob`         | `github-release-watcher`  |
| State          | ConfigMap (letzter gesehener Tag/Repo, nur für `notifyZammad: true`-Repos)| `github-release-watcher`  |
| App-Update-Liste| Separate ConfigMap (`currentVersion` vs. aktuelles GitHub-Release, alle Repos)| `github-release-watcher`, lesbar für `carplay-api` |
| Benachrichtigung| Zammad-Ticket via REST-API (nur `notifyZammad: true`) | —            |
| Secrets        | SealedSecrets                        | `github-release-watcher`  |

**Warum Polling statt Webhook:** Der Cluster hat keinen öffentlichen Ingress
(siehe [capulus-core/f-cicd-automatisierung/f0000-argo-workflows.md](f0000-argo-workflows.md) — Git-Webhooks sind
bewusst außerhalb des Scopes). GitHub kann daher nicht direkt in den Cluster
pushen; stattdessen fragt ein `CronJob` standardmäßig alle 2 Stunden die
öffentliche GitHub-API ab (ausgehende Verbindung, wie beim GHCR-Push aus Argo
Workflows) — Takt über `schedule` in `values.yaml` anpassbar.

**Warum kein Trigger-Setup in Zammad nötig ist:** Zammad benachrichtigt
Agenten standardmäßig per E-Mail über neue Tickets in Gruppen, denen sie
zugeordnet sind — steuerbar über die persönlichen Profil-Einstellungen
(**Profil → Benachrichtigungen**), nicht über einen zusätzlichen Trigger.
Voraussetzung ist ein funktionierender ausgehender E-Mail-Kanal in Zammad
(Admin → Channels → Email), der ohnehin für den Helpdesk-Betrieb benötigt
wird — siehe [capulus-core/3-apps-workloads/30000-zammad.md](../3-apps-workloads/30000-zammad.md).

---

## Voraussetzungen

- ArgoCD läuft, Root-ApplicationSet ist aktiv
  (`argocd/bootstrap/root-applicationset.yaml`)
- Sealed-Secrets Controller ist installiert
- Zammad ist deployt und erreichbar ([capulus-core/3-apps-workloads/30000-zammad.md](../3-apps-workloads/30000-zammad.md)),
  inkl. konfiguriertem ausgehendem E-Mail-Kanal
- `kubeseal` CLI ist lokal installiert

---

## Schritt 1 — Zammad-API-Token erzeugen

1. In Zammad einloggen (Account, dessen Gruppen-Mitgliedschaft die
   gewünschten E-Mail-Empfänger erhalten soll)
2. **Profil → Token Access → Neuer Token**
3. Berechtigung `ticket.agent` (Tickets lesen/erstellen) zuweisen
4. Token kopieren — wird nur einmal angezeigt

## Schritt 2 — Secret versiegeln

```bash
NS=github-release-watcher
SECRET_NAME=github-release-watcher-zammad-token

echo -n "<ZAMMAD_API_TOKEN>" | kubeseal --raw \
  --namespace $NS --name $SECRET_NAME \
  --controller-namespace sealed-secrets \
  --controller-name sealed-secrets-controller \
  --from-file=/dev/stdin
# → Ausgabe als secrets.encryptedToken in values.yaml eintragen
```

## Schritt 3 — `values.yaml` anpassen

In `argocd/apps/tech/github-release-watcher/values.yaml` ist `github.repos` eine
Liste von Objekten, nicht nur `owner/repo`-Strings — pro Repo lässt sich
`notifyZammad` einzeln steuern:

```yaml
github:
  sourceRepo: "pkr-lab/capulus-core"
  sourceRef: "main"
  repos:
    - id: docflowengine
      name: "DocFlowEngine"
      repo: "pkr-lab/DocFlowEngine"
      currentVersion: ""
      notifyZammad: true
    - id: nextcloud
      name: "Nextcloud"
      repo: "nextcloud/server"
      currentFrom:
        file: "argocd/apps/prod/nextcloud/values.yaml"
        image: "nextcloud"
      notifyZammad: false
    - id: victoria-logs
      name: "VictoriaLogs"
      repo: "VictoriaMetrics/helm-charts"
      tagPrefix: "victoria-logs-single-"
      currentFrom:
        file: "argocd/apps/tech/logging/Chart.lock"
        chart: "victoria-logs-single"
      notifyZammad: false

zammad:
  url: "https://zammad.tech.homeserver"
  group: "Users"
  requesterEmail: "<empfaenger>@example.com"

secrets:
  encryptedToken: "<Ausgabe aus Schritt 2>"
```

| Feld | Bedeutung |
|---|---|
| `repo` | GitHub-Repo, dessen Releases geprüft werden |
| `tagPrefix` | Nur Releases mit diesem Tag-Präfix zählen, das Präfix wird vor dem Vergleich entfernt. Für Repos mit mehreren Produkten (`VictoriaMetrics/helm-charts`, `zammad/zammad-helm` mit `zammad-`, sealed-secrets mit `helm-v`, Headlamp mit `headlamp-helm-`, n8n mit `n8n@`) |
| `currentFrom.file` | Datei in `sourceRepo@sourceRef`, aus der die installierte Version gelesen wird |
| `currentFrom.image` | Wert der `repository:`-Zeile, die Version ist das folgende `tag:` (ohne Digest) |
| `currentFrom.chart` | Name des Dependency-Eintrags (`- name:`) in `Chart.yaml`/`Chart.lock`, die Version ist das folgende `version:` |
| `currentFrom.pattern` | Regulärer Ausdruck, die erste Gruppe ist die Version (n8n: `FROM n8nio/n8n:…` im Dockerfile) |
| `currentVersion` | Feste Version, nur ohne `currentFrom`. Leer heißt „unbekannt“ |
| `notifyZammad` | `true` erzeugt bei einem neuen Release ein Zammad-Ticket (nur DocFlowEngine) |

Die installierte Version liest der Watcher bei jedem Lauf aus dem Repo, nach einem Upgrade muss nichts nachgezogen werden.
Neue App: Eintrag mit `currentFrom` anlegen, dann den Watcher einmal von Hand starten (Schritt 6) und im Log die Zeile
`[OK] <repo>: …, installiert <version>` prüfen. `installierte Version nicht lesbar` heißt: `image`/`chart` passt nicht zur Datei.

> **`requesterEmail`:** Muss ein **bereits existierender** Zammad-User
> (Login oder E-Mail) sein. Anders als beim E-Mail-Kanal legt die Ticket-API
> (`POST /api/v1/tickets`) bei unbekannter Adresse **keinen** Customer
> automatisch an, sondern schlägt mit `HTTP 422 No lookup value found for
> 'customer'` fehl. Vorhandene User (inkl. exaktem Login/E-Mail-Feld) prüfen:
> `GET /api/v1/groups` bzw. `GET /api/v1/users` mit dem Zammad-Token
> (`curl -s -H "Authorization: Token token=$TOKEN" https://zammad.tech.homeserver/api/v1/users`).
> Existiert noch kein passender User, in **Admin → Users → New** einen
> Customer-Account mit dieser Adresse anlegen. Diese Adresse bestimmt nur
> den Ticket-"Anfrager", **nicht** direkt den E-Mail-Empfänger der
> Benachrichtigung (siehe unten).

> **Untergruppen:** Ist `zammad.group` eine Untergruppe (z. B. unter
> **Admin → Groups** mit einer Elterngruppe angelegt), erwartet der
> Ticket-API-Lookup den vollqualifizierten Namen mit `::` als Trenner, exakt
> in der Schreibweise der Elterngruppe (Groß-/Kleinschreibung zählt), z. B.
> `Capulus-Core::RepoBenachrichtigung`. Der einfache Name allein
> (`RepoBenachrichtigung`) oder eine falsche Schreibweise schlägt mit
> `HTTP 422 No lookup value found for 'group'` fehl — im Zweifel über
> `GET /api/v1/groups` das exakte `name`-Feld prüfen.

## Schritt 4 — Agenten-Benachrichtigung in Zammad prüfen

Damit die E-Mail tatsächlich verschickt wird, muss der/die Ziel-Agent(en) in
der gewählten `zammad.group`:

1. Mitglied der Gruppe sein (**Admin → Groups → <Gruppe> → Agents**)
2. Unter **Profil → Benachrichtigungen** die Gruppe für das Ereignis
   "Neues Ticket" per E-Mail aktiviert haben (Zammad-Standard: aktiviert)

## Schritt 5 — Deployment via ArgoCD

Nach dem Commit erkennt das Root-ApplicationSet den neuen Ordner
`argocd/apps/tech/github-release-watcher/` automatisch.

```
ArgoCD → Home → github-release-watcher
Status: Syncing → Healthy
```

Der `CronJob` liefert mit `suspend: true` aus. Nach Prüfung der Secrets und
der Zammad-Konfiguration bewusst aktivieren:

```bash
kubectl -n github-release-watcher patch cronjob github-release-watcher \
  --type merge -p '{"spec":{"suspend":false}}'
```

Dauerhaft aktivieren: `suspend: false` in `values.yaml` setzen und committen.

## Schritt 6 — Manuell testen

```bash
kubectl -n github-release-watcher create job --from=cronjob/github-release-watcher test-run
kubectl -n github-release-watcher logs -l job-name=test-run -f
```

Beim ersten Lauf wird pro Repo nur die **Baseline gesetzt** (`[INIT] ...`) —
es wird bewusst noch kein Ticket erzeugt, um beim Rollout keine Alt-Releases
als "neu" zu melden. Erst der nächste tatsächlich neue Release löst ein
Ticket aus.

Um den Zammad-Versand isoliert zu testen, den State manuell zurücksetzen:

```bash
kubectl -n github-release-watcher delete configmap github-release-watcher-state
kubectl -n github-release-watcher create job --from=cronjob/github-release-watcher test-run-2
# -> [INIT] setzt Baseline erneut, danach Tag im State manuell auf einen alten Wert patchen:
kubectl -n github-release-watcher patch configmap github-release-watcher-state \
  --type merge -p '{"data":{"state.json":"{\"pkr-lab/DocFlowEngine\":\"v0.0.0\"}"}}'
kubectl -n github-release-watcher create job --from=cronjob/github-release-watcher test-run-3
```

---

## Funktionsweise

1. `CronJob` startet im konfigurierten Takt (`schedule`, `timezone` in
   `values.yaml`, Standard: alle 2 Stunden) einen Pod mit
   `python3 /app/watcher.py`
2. Für jedes Repo in `github.repos`: `GET
   https://api.github.com/repos/<owner>/<repo>/releases?per_page=100`, neuestes
   Release ist die höchste Versionsnummer ohne Draft/Pre-Release (mit
   `tagPrefix` nur passende Tags). Die installierte Version kommt aus
   `currentFrom` (Contents-API auf `sourceRepo@sourceRef`).
3. **App-Update-Liste (alle Repos, jeder Lauf):** Die installierte Version
   wird gegen die gerade abgefragte GitHub-Version verglichen (Versionsnummern,
   `v`-Präfix und `release-` spielen keine Rolle),
   das Ergebnis landet in der `updates`-ConfigMap im eigenen Namespace.
   `carplay-api` liest diese ConfigMap über ein eigenes, auf genau diese
   ConfigMap eingeschränktes RoleBinding (`templates/role.yaml`) und zeigt
   sie der iOS-App als Update-Status.
4. **Zammad-Ticket (nur `notifyZammad: true`-Repos):** Der zuletzt gesehene
   Tag pro Repo liegt zusätzlich in einer zweiten, getrennten ConfigMap
   (`state`) im eigenen Namespace. Bei neuem Tag: `POST {zammad.url}/api/v1/
   tickets` legt ein Ticket in der konfigurierten Gruppe an; Zammads
   Agenten-Benachrichtigung übernimmt den E-Mail-Versand. State wird nur bei
   Erfolg aktualisiert — schlägt der Zammad-Call fehl, versucht der nächste
   Lauf es erneut (kein verlorenes Release).

Beide ConfigMaps sind **nicht** Teil der Helm-Templates (werden vom Skript
zur Laufzeit angelegt/geschrieben), damit ArgoCDs `selfHeal`/`prune` sie
nicht mit einem leeren Git-Stand überschreibt — die RBAC-Rules in
`templates/role.yaml` erlauben dem ServiceAccount `create` sowie
`get`/`patch`/`update` ausschließlich auf diese beiden konkreten
ConfigMap-Namen.

---

## Troubleshooting

### CronJob läuft, aber kein Ticket erscheint

```bash
kubectl -n github-release-watcher logs -l app.kubernetes.io/name=github-release-watcher --tail=50
```

- `[ERR] ...: GitHub-Abfrage fehlgeschlagen` — Rate-Limit (60 Req/h ohne
  Token) oder DNS/Egress-Problem; optional `github.tokenSecretName` mit
  einem PAT setzen
- `[ERR] ...: Zammad-Benachrichtigung fehlgeschlagen` — Token ungültig/ohne
  `ticket.agent`-Berechtigung, Gruppe existiert nicht, oder
  `zammad.tech.homeserver` vom Cluster aus nicht erreichbar

### Ticket wird angelegt, aber keine E-Mail kommt an

- Zammad-Agent ist nicht Mitglied der Zielgruppe, oder die persönliche
  E-Mail-Benachrichtigung für "Neues Ticket" ist deaktiviert (Schritt 4)
- Zammads ausgehender E-Mail-Kanal ist nicht konfiguriert oder fehlerhaft
  (Admin → Channels → Email → Testmail versenden)

### SealedSecret wird nicht entschlüsselt

```bash
kubectl describe sealedsecret -n github-release-watcher github-release-watcher-zammad-token
kubectl logs -n sealed-secrets -l app.kubernetes.io/name=sealed-secrets
```

---

## Relevante Links

- [Zammad API-Dokumentation — Tickets](https://docs.zammad.org/en/latest/api/ticket/index.html)
- [GitHub REST API — Releases](https://docs.github.com/en/rest/releases/releases)
- [Zammad Setup](../3-apps-workloads/30000-zammad.md)
- [ArgoCD Setup](../b-kubernetes-gitops/b0010-argocd.md)
