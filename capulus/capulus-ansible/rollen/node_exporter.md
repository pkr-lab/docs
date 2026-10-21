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

---

## Fehlersuche

Allgemeines Vorgehen bei Rollen: `ansible-playbook <playbook>.yml --limit <host> --tags <tag> --check --diff -vvv` zeigt Änderungen ohne sie anzuwenden (`make <ziel>-check` bei den Playbooks mit Dry-Run). Die Rollenübersicht mit Make-Zielen: [Rollen](index.md#fehlersuche-je-rolle).

```bash
sudo systemctl status prometheus-node-exporter
curl -s localhost:9100/metrics | head
sudo ss -tlnp | grep 9100
```

| Symptom | Ursache | Lösung |
|---|---|---|
| Keine Metriken von prod-vm, entw-vm oder einem Kiosk | Dienst aus, Port 9100 durch UFW blockiert, Scrape-Ziel falsch | `systemctl status prometheus-node-exporter`, [Monitoring](../../capulus-core/6-hintergruende/60060-monitoring-und-alerting.md), [vmagent](vmagent.md) |
| Host in Grafana offline | Scrape erreicht den Host nicht (Tailnet) oder `instance`-Label stimmt nicht | Label in den Scrape-Konfigurationen prüfen |
| Rolle abgeschaltet | `node_exporter_enabled: false` | Wert prüfen |

Allgemeine Fehlersuche: [Host-Troubleshooting](../a-betriebssystem/a0040-troubleshooting.md), [zentrale Fehlersuche](../../capulus-core/2-betrieb-hardware/20070-troubleshooting-zentral.md).

[Alle Rollen](index.md)
