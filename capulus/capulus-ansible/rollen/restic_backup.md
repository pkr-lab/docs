# Rolle `restic_backup`

Tägliches Restic-Backup des Home-Verzeichnisses mit wöchentlicher Aufbewahrung und Integritätsprüfung. Die Rolle ist **aus**, bis Ziel und Passwort gesetzt sind.

| | |
|---|---|
| Läuft gegen | ollama-pc (Playbook `ollama-pc.yml`) |
| Tags | `restic-backup`, `backup` |
| Quellcode | [roles/restic_backup](https://github.com/pkr-lab/capulus-ansible/tree/main/roles/restic_backup) |

## Variablen

Defaults aus `defaults/main.yml`. Überschreiben in `host_vars/ollama-pc/vars.yml`; das Passwort gehört als `!vault`-Wert in den Vault.

| Variable | Default | Bedeutung |
|---|---|---|
| `restic_backup_enabled` | `false` | Rolle aktivieren |
| `restic_backup_repository` | `` | Ziel (Pfad auf einem externen Laufwerk, `s3:…`, `rest:…`) |
| `restic_backup_password` | `` | Repository-Passwort (`!vault`), wird nach `/etc/restic-backup/password` (`0400`, Benutzer) geschrieben |
| `restic_backup_user` | `{{ ansible_user }}` | Benutzer, unter dem Backup und Aufbewahrung laufen |
| `restic_backup_paths` | `[/home/<ansible_user>]` | gesicherte Pfade |
| `restic_backup_excludes` | `[.cache, node_modules, .npm, .cargo/registry, go/pkg/mod, .local/share/Trash, snap/*/common/.cache]` | Ausschlussmuster |
| `restic_backup_config_dir` | `/etc/restic-backup` | Ablage von `env`, `password`, `excludes` |
| `restic_backup_calendar` | `daily` | Zeitplan des Backups |
| `restic_backup_prune_calendar` | `weekly` | Zeitplan von `forget --prune` und `check` |
| `restic_backup_keep_daily`, `_weekly`, `_monthly` | `7`, `4`, `6` | Aufbewahrung |

## Ablauf

1. Pflichtwerte prüfen (Repository und Passwort gesetzt)
2. restic installieren
3. Konfigurationsverzeichnis, Passwortdatei, `env` und Ausschlussliste ablegen
4. systemd-Units und Timer deployen
5. Repository prüfen (`restic cat config`), bei Bedarf `restic init`
6. Timer `restic-backup.timer` und `restic-prune.timer` aktivieren

## Templates

- `env.j2`, `excludes.j2`
- `restic-backup.service.j2`, `restic-backup.timer.j2`
- `restic-prune.service.j2`, `restic-prune.timer.j2`

---

## Fehlersuche

```bash
systemctl list-timers 'restic-*'
journalctl -u restic-backup.service -n 50 --no-pager
```

| Symptom | Ursache | Lösung |
|---|---|---|
| `restic_backup_repository und restic_backup_password (Vault) setzen` | Rolle aktiviert, Werte fehlen | Variablen setzen, Passwort mit `ansible-vault encrypt_string` ablegen |
| Backup meldet `permission denied` | Pfad nicht lesbar für `restic_backup_user` | Benutzer oder `restic_backup_paths` anpassen |
| `Fatal: wrong password or no key found` | Passwort passt nicht zum Repository | richtiges Passwort setzen; das Repository-Passwort lässt sich nicht zurücksetzen |

Allgemeines Vorgehen bei Rollen: [Rollenübersicht](index.md).

[Alle Rollen](index.md)
