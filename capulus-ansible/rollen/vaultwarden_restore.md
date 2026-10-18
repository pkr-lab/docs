# Rolle `vaultwarden_restore`

Wiederherstellung der Vaultwarden-Daten aus dem NAS-Backup.

| | |
|---|---|
| Läuft gegen | homeserver |
| Tags | `vaultwarden-restore` |
| Quellcode | [roles/vaultwarden_restore](https://github.com/pkr-lab/capulus-ansible/tree/main/roles/vaultwarden_restore) |

## Variablen

Defaults aus `defaults/main.yml`. Überschreiben in `group_vars/` oder `host_vars/`.

| Variable | Default | Bedeutung |
|---|---|---|
| `vaultwarden_namespace` | `vaultwarden` | – |
| `vaultwarden_release` | `vaultwarden` | – |
| `vaultwarden_restore_image` | `alpine:3.20` | – |
| `force_restore` | `false` | – |

## Ablauf

1. Restore ohne force_restore=true ablehnen
2. Vaultwarden-Deployment auf 0 Replicas skalieren
3. Warten bis der Vaultwarden-Pod terminiert ist
4. Eventuellen Restore-Job aus einem vorherigen Lauf entfernen
5. Restore-Job-Manifest rendern
6. Restore-Job anwenden
7. Warten bis der Restore-Job abgeschlossen ist
8. Gerendertes Job-Manifest wieder entfernen
9. Vaultwarden-Deployment wieder auf 1 Replica skalieren
10. Warten bis Vaultwarden wieder Ready ist
11. Erreichbarkeit nach dem Restore pruefen (/alive)

## Templates

- `restore-job.yaml.j2`

[Alle Rollen](index.md)
