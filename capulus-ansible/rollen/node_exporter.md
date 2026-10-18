# Rolle `node_exporter`

Host-Metriken für Nicht-k3s-Hosts bzw. VMs.

| | |
|---|---|
| Läuft gegen | prod-vm, entw-vm, Banana Pi, Kiosks |
| Tags | `monitoring`, `node-exporter` |
| Quellcode | [roles/node_exporter](https://github.com/pkr-lab/capulus-ansible/tree/main/roles/node_exporter) |

## Variablen

Defaults aus `defaults/main.yml`. Überschreiben in `group_vars/` oder `host_vars/`.

| Variable | Default | Bedeutung |
|---|---|---|
| `node_exporter_enabled` | `true` | – |

## Ablauf

1. prometheus-node-exporter installieren
2. node_exporter aktivieren und starten
3. node_exporter deaktivieren (node_exporter_enabled false)

[Alle Rollen](index.md)
