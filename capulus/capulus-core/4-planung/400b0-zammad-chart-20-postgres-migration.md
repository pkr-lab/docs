# Zammad-Chart 20 — Umstieg auf das neue Postgres-StatefulSet

Plan und Runbook für das Upgrade des Zammad-Helm-Charts in `argocd/apps/tech/zammad/` von 16.4.x auf 20.0.2 (Zammad 7.2.2). Ab Chart 17.0.0 ersetzt Zammad die Bitnami-Subchart `postgresql` (PostgreSQL 17.5, StatefulSet `zammad-postgresql`) durch die CloudPirates-Subchart `postgres` (PostgreSQL 18.4, StatefulSet `zammad-postgres`) mit **neuem, leerem Volume**. Die Daten ziehen per `pg_dump`/`pg_restore` um.

> **Stand (2026-11-15): vorbereitet, nicht ausgeführt.** Chart-Bump und Values liegen im Branch `chore/zammad-chart-20` (capulus-core). Der PR wird erst im Wartungsfenster gemergt, **nachdem** der Auto-Sync pausiert ist (Schritt 2). Nach der Umsetzung wandert der Ist-Zustand nach [30000](../3-apps-workloads/30000-zammad.md) und [60040](../6-hintergruende/60040-helm-charts-tech.md#zammad).

> **Cluster:** TECH · `export KUBECONFIG=~/.kube/homeserver.yaml` · Zugriff je Cluster: [a0010](../../capulus-ansible/a-betriebssystem/a0010-overview.md#kubectl-zugriff-je-cluster).

---

## Warum nicht einfach mergen

Die Application `zammad` läuft mit `automated: {prune: true, selfHeal: true}` aus dem ApplicationSet `home-server-apps-tech`. Ein Merge des Bumps würde sofort:

1. das StatefulSet `zammad-postgres` mit leerer Datenbank anlegen,
2. den Init-Job laufen lassen, der dort eine **frische, leere Zammad-Installation** anlegt,
3. das alte StatefulSet `zammad-postgresql` prunen.

Die Daten wären nicht weg: Das alte PVC `data-zammad-postgresql-0` stammt aus einem `volumeClaimTemplate`, gehört ArgoCD nicht (keine Tracking-ID) und das StatefulSet hat `persistentVolumeClaimRetentionPolicy: Retain`. Zammad liefe aber mit leerer Datenbank, Kunden sähen ein Setup-Formular.

---

## Ausgangslage (gemessen am 2026-11-15)

| Punkt | Wert |
|---|---|
| Chart auf `main` | 16.4.7 (Zammad 7.1.1); PR [#757](https://github.com/pkr-lab/capulus-core/pull/757) hebt auf 16.4.10, offen |
| Postgres | `docker.io/bitnamilegacy/postgresql:17.5.0`, StatefulSet `zammad-postgresql`, PVC `data-zammad-postgresql-0` (50 Gi, `local-path`, Node `homeserver`) |
| Datenbank | `zammad_production`, 98 MB, 440 Tickets, 602 Artikel, Anhänge in der DB (`store_provider_dbs`, 33 MB), Encoding UTF8, Collation `en_US.UTF-8`, Erweiterung nur `plpgsql` |
| Rollen | `zammad` (Eigentümer, kein Superuser), `postgres` (Superuser ohne Passwort-Datei) |
| Elasticsearch | aus (`zammadConfig.elasticsearch.enabled: false`) |
| MinIO / S3 | aus (Chart-Default), Anhänge in der DB, `storageVolume.enabled: false` |
| Backup | `Backup` `zammad` (capulus-operator), täglich 03:00, `pg_dump` aus `postgres:18-alpine` gegen Host `zammad-postgresql` |
| Kernel `homeserver` | `7.0.0-38-generic` (mit `7.0.0-42` startet Postgres nicht, [50040](../5-incidents/50040-kernel-7042-apparmor-signale.md)) |

---

## Was sich zwischen 16.4 und 20.0 ändert

Verglichen mit `helm template zammad argocd/apps/tech/zammad -n zammad` vor und nach dem Bump (gleiche Values bis auf die Anpassungen unten).

| Chart | Änderung | Folge für uns |
|---|---|---|
| 17.0.0 | Bitnami-`postgresql` → CloudPirates-`postgres` (offizielles Image, PG 18.4), Service `zammad-postgres`, PVC `data-zammad-postgres-0` | Datenmigration (dieses Runbook), Values-Block `postgresql:` → `postgres:` |
| 17.0.0 | Der DB-Benutzer aus `postgres.auth.username` wird per `POSTGRES_USER` angelegt und ist damit **Superuser** | Bewusst hingenommen: Instanz gehört nur Zammad. Die Alternative `customUser` bräuchte ein zweites, neu versiegeltes Superuser-Passwort |
| 17.0.0 | `zammadConfig.initJob.enabled`, `replicas: 0` für Scheduler/Websocket möglich | nicht gebraucht, die Phasen laufen hier über ArgoCD (Schritt 5) |
| 18.0.0 | Elasticsearch über ECK-Operator (`eck-elasticsearch`) | keine, ES bleibt aus, kein ECK-Operator nötig |
| 18.0.0 | `initJob.randomName` entfernt, Job heißt `zammad-init-<revision>` (unter ArgoCD immer `zammad-init-1`) | `randomName: false` entfernt, stattdessen Sync-Hook-Annotation (siehe unten) |
| 18.0.0 | `fsGroupChangePolicy: OnRootMismatch` statt `Always` | keine |
| 19.0.0 | MinIO → RustFS (`zammadConfig.rustfs`) | keine, S3 war und bleibt aus |
| 20.0.0 | Values werden gegen `values.schema.json` geprüft, Scheduler/Websocket nur `0` oder `1` | Render mit unseren Values läuft durch (`replicas: null` bei nginx/railsserver ist erlaubt) |
| 20.0.2 | Zammad `7.2.2-0023`, Redis-Subchart 0.37 (Redis 8.10.2, Init-Container, `REDIS_PASSWORD` aus Secret), Memcached 0.14 | keine Values-Änderung nötig, die `ignoreDifferences`-Regel für `zammad-redis` passt weiter |

Gerenderte Ressourcen, die wegfallen bzw. neu entstehen:

| Fällt weg (Prune) | Neu |
|---|---|
| `StatefulSet/zammad-postgresql`, `Service/zammad-postgresql`, `Service/zammad-postgresql-hl`, `ServiceAccount/zammad-postgresql`, `PodDisruptionBudget/zammad-postgresql`, `NetworkPolicy/zammad-postgresql`, `Job/zammad-init` | `StatefulSet/zammad-postgres`, `Service/zammad-postgres`, `Service/zammad-postgres-headless`, `ConfigMap/zammad-postgres`, `Job/zammad-init-1` (Sync-Hook) |

Die Bitnami-NetworkPolicy fällt ersatzlos weg. Eingehend gilt danach `tier-default-ingress` des Namespaces (Pods im Namespace, `kube-system`, `monitoring`, `ticket-gate`, `cloudflared`), das reicht für Zammad und das Backup.

### Values-Anpassungen im PR

| Key | Alt | Neu |
|---|---|---|
| `zammad.postgresql` | Bitnami-Block (`primary.persistence`, `volumePermissions`, `secretKeys.userPasswordKey`) | entfällt |
| `zammad.postgres.auth` | – | `username: zammad`, `database: zammad_production`, `existingSecret: zammad-secrets`, `secretKeys.adminPasswordKey: postgresql-password` (gleiches Passwort wie bisher, kein neues Secret) |
| `zammad.postgres.persistence` | – | `storageClass: local-path`, `size: 50Gi` |
| `zammad.postgres.nodeSelector` | `primary.nodeSelector` | `kubernetes.io/hostname: homeserver` |
| `zammad.postgres.resources` | `primary.resources` | unverändert (50m / 256Mi, Limit 512Mi) |
| `zammad.zammadConfig.initJob` | `randomName: false` | `annotations: {argocd.argoproj.io/hook: Sync}` |
| `resticBackup.postgres.host` | `zammad-postgresql` | `zammad-postgres` |
| `Chart.yaml` | 16.4.7 / 7.1.1 | 20.0.2 / 7.2.2, `charts/zammad-20.0.2.tgz` |

**Init-Job als Sync-Hook.** Heute legt ArgoCD den Init-Job `zammad-init` etwa alle sechs Minuten neu an: `ttlSecondsAfterFinished: 300` löscht ihn, `selfHeal` findet ihn fehlend und erzeugt ihn wieder, jedes Mal laufen Migrationen und `Locale.sync`/`Translation.sync` (Events am 2026-11-15 13:09, 13:15, 13:21 UTC). Als Sync-Hook läuft der Job nur noch bei einem echten Sync, wie im Chart-README unter „Deploying with ArgoCD“ empfohlen.

---

## Ablauf

Zeitbedarf ohne Störungen etwa 30 Minuten, Zammad ist ab Schritt 3 bis Schritt 7 nicht erreichbar (ticket-gate und n8n bekommen in der Zeit Fehler). Nicht um 03:00 (Backup) und nicht sonntags um 04:30 (Scheduler-Neustart) legen.

### 0. Vorbereitung (vor dem Fenster)

- PR [#757](https://github.com/pkr-lab/capulus-core/pull/757) entweder mergen und den Branch `chore/zammad-chart-20` darauf rebasen oder #757 schließen. Das Runbook gilt für 16.4.7 und 16.4.10 gleich.
- PR `chore/zammad-chart-20` mit grüner CI bereitlegen, **nicht mergen**.
- Einen Revert-Branch vorbereiten, damit ein Rückweg nicht auf eine neue Review wartet (Rollback B).
- Kernel prüfen, muss `7.0.0-38-generic` sein:

```bash
export KUBECONFIG=~/.kube/homeserver.yaml
kubectl get node homeserver -o jsonpath='{.status.nodeInfo.kernelVersion}{"\n"}'
```

- Postgres-Image auf `homeserver` vorab ziehen (Docker Hub war bei der Vorbereitung zeitweise langsam):

```bash
ssh homeserver sudo k3s crictl pull docker.io/postgres:18.4@sha256:a02db8cac496f15b094798a38254f14d6e00741f709360e5e00bb6668ea31636
```

- Prüfen, dass das ApplicationSet noch keine `ignoreApplicationDifferences` hat (sonst Schritt 2 mit `cubi autosync` statt `kubectl patch`, damit fremde Regeln bleiben):

```bash
kubectl -n argocd get applicationset home-server-apps-tech -o jsonpath='{.spec.ignoreApplicationDifferences}{"\n"}'
```

### 1. Aktuelles Backup

```bash
kubectl -n zammad create job --from=cronjob/zammad zammad-vor-chart-20
kubectl -n zammad wait --for=condition=complete --timeout=10m job/zammad-vor-chart-20
```

### 2. Auto-Sync pausieren, dann PR mergen

Die Application gehört dem ApplicationSet. Ein `kubectl patch` an `spec.syncPolicy` setzt der ApplicationSet-Controller nach Sekunden zurück, deshalb erst die Regel im ApplicationSet, dann die Application ([cubi: Apps aus einem ApplicationSet](../../../cubi/argocd-features.md#apps-aus-einem-applicationset)). Gleichwertig: `cubi autosync` → *Auto-Sync deaktivieren* → `zammad`.

```bash
kubectl -n argocd patch applicationset home-server-apps-tech --type merge \
  -p '{"spec":{"ignoreApplicationDifferences":[{"name":"zammad","jsonPointers":["/spec/syncPolicy"]}]}}'
kubectl -n argocd patch application zammad --type json \
  -p '[{"op":"remove","path":"/spec/syncPolicy/automated"}]'
sleep 30
kubectl -n argocd get application zammad -o jsonpath='automated={.spec.syncPolicy.automated}{"\n"}'
```

Erwartet: `automated=` (leer). Erst dann den PR mergen. Danach zeigt ArgoCD `OutOfSync` mit der neuen Revision, im Cluster ändert sich nichts:

```bash
kubectl -n argocd annotate application zammad argocd.argoproj.io/refresh=normal --overwrite
kubectl -n argocd get application zammad -o jsonpath='{.status.sync.status} {.status.sync.revision}{"\n"}'
kubectl -n zammad get sts
```

Während der Pause nicht `make argocd` in capulus-ansible ausführen und das ApplicationSet nicht neu einspielen, das würde die Regel löschen und den Sync sofort starten.

### 3. Zammad stoppen

```bash
kubectl -n zammad scale deploy -l app.kubernetes.io/name=zammad --replicas=0
kubectl -n zammad get pods
```

Memcached (`app.kubernetes.io/name=memcached`), Redis und Postgres laufen weiter. Die HPAs von nginx und railsserver skalieren nicht von 0 hoch. Warten, bis keine `zammad-*`-Pods außer Postgres, Redis und Memcached mehr laufen, auch kein `zammad-init`. Dann darf keine Verbindung mehr offen sein:

```bash
kubectl -n zammad exec zammad-postgresql-0 -c postgresql -- bash -c \
  'PGPASSWORD="$(cat "$POSTGRES_PASSWORD_FILE")" psql -U zammad -d zammad_production -Atc "select count(*) from pg_stat_activity where datname = current_database() and pid <> pg_backend_pid()"'
```

Erwartet: `0`.

### 4. Dump aus der alten Datenbank

Der Dump landet außerhalb jedes Repos. Ein fehlgeschlagener `pg_dump` hinterlässt sonst eine leere Datei, deshalb Größe und Inhaltsverzeichnis prüfen.

```bash
B=~/capulus-migration-backups; mkdir -p "$B"; chmod 700 "$B"
F="$B/zammad-pg17-$(date +%Y%m%d-%H%M).dump"
kubectl -n zammad exec zammad-postgresql-0 -c postgresql -- bash -c \
  'PGPASSWORD="$(cat "$POSTGRES_PASSWORD_FILE")" pg_dump -Fc -U zammad -d zammad_production' > "$F"
echo "rc=$?"; ls -l "$F"
docker run --rm -i docker.io/postgres:18.4 pg_restore --list < "$F" | grep -c 'TABLE DATA'
kubectl -n zammad exec zammad-postgresql-0 -c postgresql -- bash -c \
  'PGPASSWORD="$(cat "$POSTGRES_PASSWORD_FILE")" psql -U zammad -d zammad_production -Atc "select count(*) from pg_tables where schemaname = '"'"'public'"'"'"'
```

Erwartet: `rc=0`, Datei um 25 MB, beide Zahlen gleich (2026-11-15: 128). Das Inhaltsverzeichnis lokal prüfen, nicht per `kubectl exec -i … pg_restore --list < "$F"`: `--list` liest nur den Anfang der Datei und beendet sich, `kubectl` hängt dann beim Schreiben des Rests und bricht mit `i/o timeout` ab (die ausgegebene Zahl stimmt trotzdem). Zählwerte der alten Datenbank sichern:

```bash
cat > "$B/zammad-counts.sql" <<'SQL'
select 'tickets', count(*) from tickets
union all select 'ticket_articles', count(*) from ticket_articles
union all select 'users', count(*) from users
union all select 'stores', count(*) from stores
union all select 'store_provider_dbs', count(*) from store_provider_dbs
union all select 'schema_migrations', count(*) from schema_migrations;
SQL
kubectl -n zammad exec -i zammad-postgresql-0 -c postgresql -- bash -c \
  'PGPASSWORD="$(cat "$POSTGRES_PASSWORD_FILE")" psql -U zammad -d zammad_production -At' < "$B/zammad-counts.sql" > "$B/zammad-counts-alt.txt"
cat "$B/zammad-counts-alt.txt"
```

### 5. Nur das neue Postgres ausrollen (selektiver Sync)

Ein selektiver Sync wendet nur die genannten Ressourcen an, **ohne Prune und ohne Hooks**. Der Init-Job läuft also nicht, die neue Datenbank bleibt leer, das alte StatefulSet bleibt stehen.

```bash
kubectl -n argocd patch application zammad --type merge -p '{"operation":{"initiatedBy":{"username":"runbook-zammad-chart-20"},"sync":{"syncOptions":["ServerSideApply=true"],"resources":[
  {"kind":"ConfigMap","name":"zammad-postgres","namespace":"zammad"},
  {"kind":"Service","name":"zammad-postgres","namespace":"zammad"},
  {"kind":"Service","name":"zammad-postgres-headless","namespace":"zammad"},
  {"group":"apps","kind":"StatefulSet","name":"zammad-postgres","namespace":"zammad"}]}}}'
kubectl -n argocd get application zammad -o jsonpath='{.status.operationState.phase} {.status.operationState.message}{"\n"}'
kubectl -n zammad rollout status sts/zammad-postgres --timeout=10m
```

Gleichwertig mit der CLI: `argocd app sync zammad --core --resource :ConfigMap:zammad-postgres --resource :Service:zammad-postgres --resource :Service:zammad-postgres-headless --resource apps:StatefulSet:zammad-postgres` (Kontext-Namespace `argocd`).

Prüfen, dass die neue Datenbank leer ist und `zammad` gehört:

```bash
kubectl -n zammad exec zammad-postgres-0 -- psql -U zammad -d zammad_production -Atc \
  "select version(); select pg_get_userbyid(datdba) from pg_database where datname = current_database(); select count(*) from pg_tables where schemaname = 'public'"
```

Erwartet: `PostgreSQL 18.4`, `zammad`, `0`. Steht dort eine Tabellenzahl über 0, ist schon etwas gegen die neue Datenbank gelaufen: Schritt 6 trotzdem ausführen, `--clean` räumt sie ab.

### 6. Restore in die neue Datenbank

Ohne `-t` (ein TTY verfälscht den Binärstrom). Der Dump geht über stdin und braucht keinen Platz im Container.

```bash
kubectl -n zammad exec -i zammad-postgres-0 -- sh -c \
  'PGPASSWORD="$POSTGRES_PASSWORD" pg_restore -U zammad -d zammad_production --no-owner --clean --if-exists' < "$F"
echo "rc=$?"
kubectl -n zammad exec zammad-postgres-0 -- vacuumdb -U zammad -d zammad_production --analyze-only
kubectl -n zammad exec -i zammad-postgres-0 -- psql -U zammad -d zammad_production -At < "$B/zammad-counts.sql" > "$B/zammad-counts-neu.txt"
diff "$B/zammad-counts-alt.txt" "$B/zammad-counts-neu.txt" && echo "Zählwerte identisch"
```

Erwartet: `rc=0`, keine Zeile `errors ignored on restore`, `Zählwerte identisch`. Probe am 2026-11-15: Dump der laufenden Datenbank mit genau diesem Befehl in ein lokales `postgres:18-alpine` (18.7, Benutzer per `POSTGRES_USER` wie im Chart) eingespielt, `rc=0` ohne Meldungen, alle sechs Zählwerte gleich, alle Tabellen gehören `zammad`. Bei Fehlern Schritt 6 wiederholen (wegen `--clean --if-exists` wiederholbar) oder Rollback A. Bricht die Übertragung über stdin ab (`i/o timeout`), den Dump erst in den Pod kopieren und von dort einspielen:

```bash
kubectl -n zammad cp "$F" zammad-postgres-0:/tmp/zammad.dump
kubectl -n zammad exec zammad-postgres-0 -- sh -c \
  'PGPASSWORD="$POSTGRES_PASSWORD" pg_restore -U zammad -d zammad_production --no-owner --clean --if-exists /tmp/zammad.dump && rm /tmp/zammad.dump'
```

### 7. Auto-Sync wieder einschalten

Die Regel im ApplicationSet entfernen, der Controller setzt `syncPolicy` aus dem Template zurück, der Auto-Sync startet. Er pruned die alten Bitnami-Ressourcen (das PVC bleibt), aktualisiert die Deployments auf 7.2.2 und lässt den Init-Job `zammad-init-1` als Sync-Hook laufen, der die Migrationen von 7.1 auf 7.2 ausführt. Gleichwertig: `cubi autosync` → *Auto-Sync aktivieren*.

```bash
kubectl -n argocd patch applicationset home-server-apps-tech --type json \
  -p '[{"op":"remove","path":"/spec/ignoreApplicationDifferences"}]'
sleep 30
kubectl -n argocd get application zammad -o jsonpath='automated={.spec.syncPolicy.automated}{"\n"}'
kubectl -n zammad get jobs,pods -w
```

`zammad-nginx` und `zammad-railsserver` haben im Manifest keine `replicas` (HPA), und das ApplicationSet ignoriert dort `/spec/replicas`. Sie bleiben deshalb auf 0 und müssen von Hand hoch, Scheduler und Websocket stellt `selfHeal` selbst auf 1:

```bash
kubectl -n zammad scale deploy zammad-nginx zammad-railsserver --replicas=1
```

Bis der Init-Job fertig ist, starten die Rails-Pods wegen `RAILS_CHECK_PENDING_MIGRATIONS` eventuell ein- oder zweimal neu.

### 8. Prüfen

| Prüfung | Befehl / Ort | Erwartet |
|---|---|---|
| Init-Job | `kubectl -n zammad logs job/zammad-init-1 --all-containers` | endet mit `postgresql init complete :)` |
| Pods | `kubectl -n zammad get pods` | alle `Running`, keine Restarts nach dem Init-Job |
| Application | `kubectl -n argocd get application zammad` | `Synced`, `Healthy` |
| Alte Ressourcen | `kubectl -n zammad get sts,pvc,networkpolicy` | kein `zammad-postgresql`-StatefulSet, PVC `data-zammad-postgresql-0` noch da, `data-zammad-postgres-0` neu |
| Kein Init-Job-Kreislauf | `kubectl -n zammad get events --field-selector involvedObject.kind=Job` nach 15 min | keine neuen `SuccessfulCreate` |
| Oberfläche | `https://zammad.tech.homeserver`, Anmeldung, Ticket mit Anhang öffnen, Suche | Tickets und Anhänge da |
| Version | Zammad → Einstellungen → System → Info | 7.2.2 |
| ticket-gate | Testanfrage über das Kontaktformular | Ticket wird angelegt |
| Backup | `kubectl -n zammad create job --from=cronjob/zammad zammad-nach-chart-20`, dann `kubectl -n zammad logs job/zammad-nach-chart-20 --all-containers` | `pg_dump` gegen `zammad-postgres` erfolgreich, restic-Snapshot angelegt |

Danach den Docs-PR (dieses Doc, [30000](../3-apps-workloads/30000-zammad.md), [20010](../2-betrieb-hardware/20010-nas-backup.md), [60040](../6-hintergruende/60040-helm-charts-tech.md#zammad)) mergen.

### 9. Aufräumen (frühestens nach einer Woche Betrieb)

`local-path` hat `reclaimPolicy: Delete`, das Löschen des PVC löscht das alte Datenverzeichnis endgültig. Erst wenn Rollback B sicher nicht mehr gebraucht wird:

```bash
kubectl -n zammad delete pvc data-zammad-postgresql-0
```

Den Dump unter `~/capulus-migration-backups/` erst löschen, wenn das restic-Repository mehrere Snapshots aus der neuen Datenbank hat.

---

## Rollback

### A — vor Schritt 7 (Auto-Sync noch pausiert)

Das alte StatefulSet und die alten Deployments (7.1.1, Host `zammad-postgresql`) stehen noch unverändert. Zammad läuft sofort wieder auf der alten Datenbank:

```bash
kubectl -n zammad scale deploy -l app.kubernetes.io/name=zammad --replicas=1
```

Danach in Git den Merge rückgängig machen (Revert-PR) und erst nach dessen Merge die ApplicationSet-Regel entfernen (Schritt 7, ohne den Scale-Befehl für nginx/railsserver, die laufen schon). Der Sync pruned dann `zammad-postgres`, das PVC `data-zammad-postgres-0` bleibt und wird von Hand gelöscht. Den Auto-Sync **nie** einschalten, solange `main` noch Chart 20 enthält und die neue Datenbank leer ist: der Init-Job legt sonst eine leere Zammad-Installation an. Passiert das doch: Zammad auf 0 skalieren, Schritt 6 wiederholen, dann einen vollständigen Sync anstoßen, der den Init-Job als Hook erneut ausführt (`argocd app sync zammad --core` oder *Sync* in der ArgoCD-Oberfläche).

### B — nach Schritt 7 (altes StatefulSet ist gepruned)

Das PVC `data-zammad-postgresql-0` enthält den Stand vom Beginn des Fensters. Was nach Schritt 7 in Zammad geschrieben wurde, geht beim Zurückgehen verloren; bei Bedarf vorher einen Dump aus `zammad-postgres-0` ziehen (wie Schritt 4, mit `PGPASSWORD="$POSTGRES_PASSWORD"`).

1. Auto-Sync pausieren (Schritt 2, ohne Merge), Zammad stoppen (Schritt 3).
2. Revert-PR mergen (Chart 16.4.x, Bitnami-Values, Backup-Host `zammad-postgresql`).
3. Auto-Sync einschalten (Schritt 7). ArgoCD legt `zammad-postgresql` neu an, das StatefulSet bindet über den gleichen Namen das vorhandene PVC `data-zammad-postgresql-0` (Node `homeserver`, PG 17.5), die Deployments gehen auf 7.1.x zurück. Die neue Datenbank hat das alte Volume nie berührt, die 7.2-Migrationen liefen nur in `zammad-postgres`.
4. `kubectl -n zammad scale deploy zammad-nginx zammad-railsserver --replicas=1`, dann wie Schritt 8 prüfen.

### C — beide Volumes unbrauchbar

Den Dump aus Schritt 4 in eine leere Datenbank der laufenden Chart-Version einspielen (Schritt 6 gegen den jeweiligen Pod), notfalls den restic-Snapshot aus Schritt 1 ([20010](../2-betrieb-hardware/20010-nas-backup.md)).

---

## Fallstricke

| Fallstrick | Hinweis |
|---|---|
| Auto-Sync zu früh an | Init-Job initialisiert die leere neue Datenbank als frische Installation. Siehe Rollback A |
| `kubectl patch` an der Application allein | wird vom ApplicationSet-Controller zurückgesetzt, erst `ignoreApplicationDifferences` setzen |
| nginx/railsserver bleiben nach Schritt 7 auf 0 | HPA skaliert nicht von 0, `/spec/replicas` ist im ApplicationSet ignoriert, von Hand auf 1 |
| Selektiver Sync startet keinen Init-Job | gewollt, Hooks laufen bei selektiven Syncs nicht |
| PVC `data-zammad-postgresql-0` gelöscht | `local-path` löscht die Daten mit, Rollback B ist dann nur noch über den Dump möglich |
| Postgres startet nicht, `could not signal for checkpoint` | Kernel `7.0.0-42` auf `homeserver`, [50040](../5-incidents/50040-kernel-7042-apparmor-signale.md) |
| `zammad` ist im neuen Postgres Superuser | Chart-Verhalten ab 17.0.0, bewusst hingenommen (siehe Tabelle oben) |
| Bitnami-NetworkPolicy fehlt | gewollt, `tier-default-ingress` deckt Ingress ab. [006](../7-security-incidents/006-netzwerkrichtlinien-prod-und-egress-fehlen.md) nennt `zammad-postgresql` noch als Egress-Ausnahme und ist nach der Umsetzung nachzuziehen |
