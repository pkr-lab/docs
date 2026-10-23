# capulus-operator — Hintergründe

Der Code von [capulus-operator](https://github.com/pkr-lab/capulus-operator) und das Chart `argocd/apps/tech/capulus-operator` sind kommentarfrei. Dieses Doc hält fest, **warum** etwas so gebaut ist. Was der Operator tut und wie man ihn bedient: [capulus-operator](../../capulus-operator/index.md).

---

## Entscheidungen

| Fundstelle | Begründung |
|---|---|
| Eigener Go-Operator statt nur bestehende Operatoren | Backups, Rotation und PKI haben im Homelab eigene Regeln (Token-Register, SealedSecrets, NAS-Squash), die kein fertiger Operator abbildet. Für Datenbanken steuert der Operator **CloudNativePG** an, statt Postgres selbst zu betreiben |
| Eine Instanz pro Cluster | ENTW bleibt als Testumgebung isoliert und folgt der Promotion-Kette ENTW → TECH → PROD. Ausgerollt ist bisher TECH |
| Operator erzeugt einen CronJob, statt selbst zu planen | Zeitplan, Zeitzone, `concurrencyPolicy: Forbid`, Job-Historie und Logs bleiben Kubernetes-Standard und mit `kubectl` sichtbar. Der Operator gleicht nur den Spec ab und liest den Status zurück |
| Spec-Hash als Annotation (`capulus.homeserver/spec-hash`) | Ein Vergleich mit dem vom API-Server um Standardwerte ergänzten CronJob wäre immer ungleich und würde bei jedem Abgleich schreiben. Der Hash vergleicht nur, was der Operator selbst erzeugt |
| Fremder CronJob gleichen Namens wird nicht übernommen | Ohne Owner-Referenz auf das `Backup` verweigert der Operator den Abgleich (`Ready=False`). Ein versehentlich gleich benannter CronJob wird so nie überschrieben |
| Status per `Status().Patch` mit `MergeFrom` | `Update` lieferte „object has been modified“, weil CronJob-Ereignisse und Status-Schreiben konkurrieren (beim Test gegen einen echten Cluster beobachtet) |

---

## Backup

- **Die Skripte der Container bekommen alle Werte als Positionsargumente** (`sh -c SKRIPT name arg …`), nie per String-Interpolation. Dateipfade, Datenbanknamen und `--exclude`-Muster müssen so nicht gequotet werden.
- **`restic forget --group-by host,tags` statt der Voreinstellung `host,paths`.** Beim Umstieg von einem Chart-CronJob auf ein `Backup` ändern sich die gesicherten Pfade (`/source` wird `/sources/<name>`). Mit Gruppierung nach Pfad gäbe es zwei Gruppen, und die alten Snapshots würden nie ausgedünnt. Host und Tag bleiben gleich, wenn das `Backup` wie der alte Job heißt (Vaultwarden: `vaultwarden`).
- **SQLite wird mit `sqlite3 -readonly … .backup` gesichert**, auf einer schreibgeschützt gemounteten PVC. Das entspricht dem früheren Vaultwarden-Job. n8n braucht ein schreibbares Datenverzeichnis, weil ein lesender Zugriff im WAL-Modus `-shm` öffnen können muss ([60040](60040-helm-charts-tech.md#n8n)); für n8n fehlt dem Operator deshalb noch eine Option für schreibbare Quellen.
- **`pg_dump` braucht mindestens die Major-Version des Servers.** Der Standard `postgres:17-alpine` passt nicht zu jedem Cluster, `spec.images.postgres` überschreibt ihn je Backup.
- **Der Operator legt weder PVCs noch Secrets an.** `storageClassName` einer PVC ist unveränderlich, und Passwörter bleiben SealedSecrets im Chart der App.
- **Alerts:** Der bestehende Alert `BackupJobStale` filtert auf CronJobs mit `backup` im Namen und erfasst die vom Operator erzeugten CronJobs nicht (`vaultwarden`). Dafür gibt es `vmrule-capulus-backups` ([60060](60060-monitoring-und-alerting.md)). `CapulusBackupFailed` greift zusätzlich, wenn ein Backup noch nie erfolgreich war: Dann existiert die Zeitreihe `capulus_backup_last_success_timestamp_seconds` nicht.

## Chart und CI

- **Image-Benutzer numerisch** (`USER 65532:65532`). Ein symbolisches `USER nonroot` lehnt die `runAsNonRoot`-Prüfung des kubelet ab ([60040](60040-helm-charts-tech.md#root-container-und-runasnonroot), pacman).
- **Das GHCR-Package ist privat**, der Namespace braucht das Pull-Secret `ghcr-pull` ([60040](60040-helm-charts-tech.md#images-und-build)).
- **CRDs liegen als Kopie in `crds/` des Charts.** Quelle der Wahrheit ist `config/crd/bases/` im Operator-Repo (erzeugt mit `make generate`). Nach einer Änderung an den Typen die Dateien ins Chart kopieren, sonst läuft der Operator gegen ein altes Schema.
- **CEL-Regeln in den kubebuilder-Markern:** Leerstrings nicht mit `''` prüfen. controller-gen macht daraus ungültiges YAML (`”`), das der API-Server beim Anlegen der CRD ablehnt. Stattdessen `has(self.feld)`. Aufgefallen beim Test gegen einen echten Cluster, die Unit-Tests erkennen es nicht.
- **Events:** Der Operator darf `events` erzeugen (ClusterRole). Ohne die Regel verwirft der API-Server die Events der Leader Election (`events is forbidden` im Log), der Operator läuft trotzdem.
- **Namespace-Eintrag in capulus-ansible:** `capulus-operator` steht in `argocd_platform_apps` und in `argocd_network_policy_refined_namespaces` ([60030](60030-argocd-und-bootstrap.md)). Danach `argocd/bootstrap/` neu rendern.

## Reihenfolge der Umstellung

Der Operator läuft zuerst, danach ersetzt ein Chart seinen CronJob durch ein `Backup`. Sonst prunt ArgoCD den alten CronJob, und bis zum ersten Abgleich gibt es keine Sicherung.
