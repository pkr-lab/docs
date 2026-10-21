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

---

## Fehlersuche

Allgemeines Vorgehen bei Rollen: `ansible-playbook <playbook>.yml --limit <host> --tags <tag> --check --diff -vvv` zeigt Änderungen ohne sie anzuwenden (`make <ziel>-check` bei den Playbooks mit Dry-Run). Die Rollenübersicht mit Make-Zielen: [Rollen](index.md#fehlersuche-je-rolle).

```bash
kubectl -n vaultwarden get pods,pvc,job
kubectl -n vaultwarden logs job/vaultwarden-restore
make vaultwarden-restore FORCE_RESTORE=true      # in capulus-ansible, überschreibt Live-Daten
```

| Symptom | Ursache | Lösung |
|---|---|---|
| `Sicherheitsabbruch: dieser Lauf wuerde die aktuellen Daten … ueberschreiben …` | Restore ohne `force_restore=true` | nur bei frischem Redeploy oder Disaster Recovery mit `-e force_restore=true` starten |
| Restore-Job bleibt hängen oder scheitert | NAS-Backup-PVC nicht gebunden, Datei nicht vorhanden | `kubectl describe job`, [NAS-Backup](../../capulus-core/2-betrieb-hardware/20010-nas-backup.md#fehlerbehebung) |
| Vaultwarden antwortet danach nicht (`/alive`) | Daten unvollständig, Pod startet nicht | `kubectl -n vaultwarden logs deploy/vaultwarden`, [Vaultwarden](../../capulus-core/3-apps-workloads/300a0-vaultwarden.md#9-troubleshooting) |

Allgemeine Fehlersuche: [Host-Troubleshooting](../a-betriebssystem/a0040-troubleshooting.md), [zentrale Fehlersuche](../../capulus-core/2-betrieb-hardware/20070-troubleshooting-zentral.md).

[Alle Rollen](index.md)
