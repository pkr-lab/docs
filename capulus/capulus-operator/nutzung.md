# Nutzung

Code: [Beispiele](https://github.com/pkr-lab/capulus-operator/tree/main/config/samples), [Controller](https://github.com/pkr-lab/capulus-operator/blob/main/internal/controller/backup_controller.go)

---

## Backup anlegen

Eine `Backup`-Ressource neben die App legen, am besten als Template im Chart der App (Beispiel: `argocd/apps/tech/vaultwarden/templates/restic-backup.yaml`). Repository-PVC und Passwort-Secret müssen schon existieren, der Operator legt beides nicht an. Passwörter bleiben SealedSecrets.

Ablauf eines Laufs: Pro SQLite- oder Postgres-Quelle erzeugt ein InitContainer eine konsistente Kopie unter `/stage` (`sqlite3 .backup` mit Integritätsprüfung, bzw. `pg_dump -F c`). Danach sichert restic `/stage` und die PVC-Quellen (nur lesend gemountet), räumt nach `retention` auf (`forget --prune`) und prüft einen Teil der Daten (`check`).

| `type` | Gesichert |
|---|---|
| `Files` | gesamte PVC unter `/sources/<name>` |
| `SQLite` | konsistente Kopie in `/stage/<name>/`, PVC ohne die DB-Datei |
| `Postgres` | Dump in `/stage/<name>/<database>.dump` |

---

## Status

```bash
kubectl get backup -A
kubectl -n vaultwarden describe backup vaultwarden
```

| Phase | Bedeutung |
|---|---|
| `Pending` | noch kein Lauf |
| `Running` | ein Job läuft |
| `Succeeded` | letzter Lauf erfolgreich |
| `Failed` | letzter Lauf fehlgeschlagen |
| `Suspended` | `spec.suspend: true` |

Conditions: `Ready` (CronJob abgeglichen) und `Healthy` (letzter Lauf).

## Lauf von Hand auslösen

```bash
kubectl -n vaultwarden create job --from=cronjob/vaultwarden restic-test
kubectl -n vaultwarden logs job/restic-test -c restic
kubectl -n vaultwarden delete job restic-test
```

Das Log endet mit `fertig:`. Ein von Hand ausgelöster Job zählt für `Healthy`, setzt aber `lastScheduleTime` nicht.

---

## Metriken und Alerts

Der Operator liefert auf Port 8080 Metriken, VictoriaMetrics scrapt sie über die `VMServiceScrape` des Charts.

| Metrik | Bedeutung |
|---|---|
| `capulus_backup_last_success_timestamp_seconds` | letzter erfolgreicher Lauf |
| `capulus_backup_last_schedule_timestamp_seconds` | letzter gestarteter Lauf |
| `capulus_backup_failed` | 1, wenn der letzte Lauf fehlgeschlagen ist |
| `capulus_backup_suspended` | 1, wenn pausiert |

Alerts (`vmrule-capulus-backups` im Chart `monitoring`): `CapulusBackupStale` (über 30 h ohne Erfolg), `CapulusBackupFailed` (letzter Lauf fehlgeschlagen), `CapulusBackupOperatorDown` (keine Metriken). Pausierte Backups lösen keinen Alert aus. Der alte Alert `BackupJobStale` erfasst `Backup`-CronJobs nicht, weil deren Name nicht `backup` enthält.

---

## Wiederherstellung

Die restic-Repositories unterscheiden sich nicht von den früheren CronJob-Repositories. Die Wiederherstellung steht je App in der App-Doku, für Vaultwarden unter [restic für Vaultwarden](../capulus-core/2-betrieb-hardware/20010-nas-backup.md#restic-für-vaultwarden-im-cluster).

---

## Fehlersuche

| Symptom | Ursache und Abhilfe |
|---|---|
| `kubectl get backup` zeigt `Ready=False`, Reason `ReconcileFailed` | Meldung in `describe backup`. Typisch: ungültige Quelle (fehlendes `pvcName`) oder ein CronJob gleichen Namens, der nicht zum `Backup` gehört |
| Pod des Operators `ImagePullBackOff` | Pull-Secret `ghcr-pull` fehlt oder `image.tag` gibt es in GHCR nicht |
| Job schlägt mit `wrong password` fehl | `repository.passwordSecretRef` zeigt auf ein falsches Secret oder Key. Passwort des Repositories im Passwortmanager prüfen |
| `pg_dump: aborting because of server version mismatch` | `images.postgres` auf dieselbe oder eine neuere Major-Version als der Server setzen |
| Alert `CapulusBackupOperatorDown` | `kubectl -n capulus-operator get pods`, Log des Managers ansehen |
