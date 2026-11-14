# INC-009 – Keine nachweisbare lokale Datensicherung (Ransomware-Resilienz)

| Feld | Wert |
|---|---|
| ID | INC-009 |
| Datum | 2026-11-14 |
| Kategorie | ransomware |
| Schweregrad | **Mittel** (Einschränkung: Nachweis nur „nicht gefunden“) |
| Betroffen | Workstation `pke-ms-7e59`, Home-Verzeichnis und `/` |

## Beschreibung und Nachweis
```
$ command -v restic borg timeshift rsnapshot duplicity   -> nur /usr/bin/restic
$ systemctl list-timers --all   -> kein Backup-/Restic-/Snapshot-Timer
$ lsblk -f   -> eine NVMe, ext4 ohne Snapshots (kein btrfs/ZFS/LVM)
$ crontab -l -> no crontab for pke
```
restic ist installiert, aber weder ein Timer noch ein Cronjob wurde gefunden. Die Cluster-Apps (Vaultwarden, n8n, Mattermost, MDM) haben restic-Secrets im Repo, das betrifft die Server, nicht diesen Rechner. Backups könnten an anderer Stelle laufen (z. B. manuell oder über einen nicht geprüften User-Timer); geprüft wurden System-Timer, Crontab und installierte Tools.

## Risiko
Ransomware oder ein defektes Laufwerk vernichtet Arbeitsstand und lokale Konfiguration ohne Rückweg. Repos sind durch Git-Remotes teilweise geschützt, nicht aber `~/.ssh`, Konfigurationen und Ungepushtes.

## Empfohlener Fix (manuell, Entscheidung zum Ziel nötig)
1. Ziel wählen: externes Laufwerk (nur zeitweise verbunden) oder Objektspeicher mit Object Lock (z. B. MinIO-Bucket mit Retention aus dem Cluster).
2. `restic init` auf dem Ziel, Passwort im Passwortmanager.
3. systemd-User-Timer täglich: `restic backup ~/ --exclude-caches`; `restic forget --keep-daily 7 --keep-weekly 4 --keep-monthly 6 --prune` auf separatem Timer.
4. Einmal pro Quartal `restic restore` testen, `restic check` wöchentlich.
5. Mindestens eine Kopie offline oder unveränderlich (append-only/Object Lock).

## Fix-Skript
In Ansible: neue Rolle `restic_backup` (im Playbook `ollama-pc.yml`, Tag `restic-backup`). Sie ist **aus**, bis `restic_backup_enabled: true`, `restic_backup_repository` und `restic_backup_password` (als `!vault`-Wert) gesetzt sind. Das Skript unten bleibt als Alternative ohne Ansible.

[`scripts/incident-fixes/INC-009-restic-user-backup.sh`](https://github.com/pkr-lab/capulus-core/blob/main/scripts/incident-scripts/incident-fixes/INC-009-restic-user-backup.sh) – richtet ein tägliches `restic backup ~/` (systemd-User-Timer) und eine wöchentliche Aufbewahrung plus `restic check` ein. **Ziel und Passwort bestimmst du** (Opt-in im Runner):
```
RESTIC_REPOSITORY=/media/pke/BACKUP/restic RESTIC_PASSWORD_FILE=$HOME/.config/restic/password \
  bash scripts/incident-fixes/INC-009-restic-user-backup.sh --dry-run
```
Voraussetzungen: Passwortdatei mit Modus 600 (nicht im Repo ablegen, Kopie im Passwortmanager). Für S3/REST-Ziele müssen Zugangsdaten zusätzlich in `~/.config/restic/env` ergänzt werden (das Skript schreibt nur Repository und Passwortdatei). Das Skript initialisiert das Repository, falls es fehlt. Für „offline/unveränderlich“ ein externes, nur zeitweise verbundenes Laufwerk oder ein Bucket mit Object Lock verwenden.

## Rollback
Timer abschalten: `systemctl --user disable --now restic-backup.timer restic-prune.timer`; die Unit-Dateien und `~/.config/restic/` können gelöscht werden (alte Fassungen unter `~/.incident-backups/INC-009-*`). Das Repository selbst bleibt bestehen.

## Quellen
- https://restic.readthedocs.io/en/stable/
- https://www.cisa.gov/stopransomware/ransomware-guide
