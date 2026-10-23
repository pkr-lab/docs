# Konfiguration

Code: [main.go](https://github.com/pkr-lab/capulus-operator/blob/main/cmd/manager/main.go), [Chart-Values](https://github.com/pkr-lab/capulus-core/blob/main/argocd/apps/tech/capulus-operator/values.yaml), [Typen](https://github.com/pkr-lab/capulus-operator/tree/main/api/v1alpha1)

---

## Flags des Managers

| Flag | Standard | Bedeutung |
|---|---|---|
| `--restic-image` | `restic/restic:0.18.1` | Image des restic-Containers |
| `--sqlite-image` | `alpine:3.24` | Image der SQLite-Dumps (`apk add sqlite` zur Laufzeit) |
| `--postgres-image` | `postgres:17-alpine` | Image für `pg_dump`; Major-Version mindestens die des Servers |
| `--leader-elect` | aus | Leader Election, im Chart an |
| `--leader-election-namespace` | leer | Namespace des Lease, im Chart der Release-Namespace |
| `--metrics-bind-address` | `:8080` | `/metrics` |
| `--health-probe-bind-address` | `:8081` | `/healthz`, `/readyz` |

## Chart-Values

| Key | Bedeutung | Default |
|---|---|---|
| `image.tag` | Image-Tag (`sha-<commit>`) | `sha-917f05b` |
| `imagePullSecrets` | Pull-Secret für GHCR | `ghcr-pull` |
| `backup.resticImage` / `sqliteImage` / `postgresImage` | Werte für die drei Image-Flags | siehe oben |
| `leaderElect` | Leader Election | `true` |
| `resources` | Requests und Limits des Managers | 20m / 48Mi, Limit 128Mi |

---

## Ressource `Backup`

Gruppe `capulus.homeserver/v1alpha1`, Kurzname `bak`.

| Feld | Bedeutung | Default |
|---|---|---|
| `schedule` | Cron-Ausdruck | Pflicht |
| `timeZone` | Zeitzone des Zeitplans | `Europe/Berlin` |
| `suspend` | pausiert den CronJob | `false` |
| `repository.pvcName` | bestehende PVC mit dem restic-Repository | Pflicht |
| `repository.passwordSecretRef` | Secret und Key mit dem Repository-Passwort | Pflicht |
| `sources[]` | Liste der Quellen, `name` ist eindeutig | mindestens eine |
| `sources[].type` | `Files`, `SQLite` oder `Postgres` | Pflicht |
| `sources[].pvcName` | Quell-PVC (`Files`, `SQLite`) | – |
| `sources[].sqlite.file` | Datenbankdatei relativ zur PVC | – |
| `sources[].sqlite.integrityCheck` | `pragma integrity_check` nach dem Dump | `true` |
| `sources[].postgres.*` | `host`, `port` (5432), `user`, `database`, `passwordSecretRef` | – |
| `exclude[]` | `--exclude`-Muster, absolut ab `/sources/<name>/` | – |
| `retention` | `keepDaily` 7, `keepWeekly` 4, `keepMonthly` 6 | siehe Spalte |
| `checkReadDataSubset` | Teil der Daten für `restic check` | `5%` |
| `host` | restic-Host der Snapshots | Name des `Backup` |
| `images.restic` / `sqlite` / `postgres` | überschreibt die Flag-Werte | leer |
| `resources` | Requests und Limits der Job-Container | leer |
| `failedJobsHistoryLimit` / `successfulJobsHistoryLimit` | Job-Historie | `3` |

Der restic-Tag eines Snapshots ist der Name des `Backup`.
