# Rolle `smartctl_exporter`

S.M.A.R.T.-Werte der physischen Platten.

| | |
|---|---|
| Läuft gegen | homeserver, worker-0/-1 |
| Tags | `monitoring`, `smartctl-exporter` |
| Quellcode | [roles/smartctl_exporter](https://github.com/pkr-lab/capulus-ansible/tree/main/roles/smartctl_exporter) |

## Variablen

Defaults aus `defaults/main.yml`. Überschreiben in `group_vars/` oder `host_vars/`.

| Variable | Default | Bedeutung |
|---|---|---|
| `smartctl_exporter_enabled` | `true` | – |

## Ablauf

1. smartmontools und prometheus-smartctl-exporter installieren
2. smartctl_exporter aktivieren und starten
3. smartctl_exporter deaktivieren (smartctl_exporter_enabled false)

[Alle Rollen](index.md)
