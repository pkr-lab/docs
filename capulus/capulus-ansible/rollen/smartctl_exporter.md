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

---

## Fehlersuche

Allgemeines Vorgehen bei Rollen: `ansible-playbook <playbook>.yml --limit <host> --tags <tag> --check --diff -vvv` zeigt Änderungen ohne sie anzuwenden (`make <ziel>-check` bei den Playbooks mit Dry-Run). Die Rollenübersicht mit Make-Zielen: [Rollen](index.md#fehlersuche-je-rolle).

```bash
sudo systemctl status prometheus-smartctl-exporter
curl -s localhost:9633/metrics | head
sudo smartctl -a /dev/sda
```

| Symptom | Ursache | Lösung |
|---|---|---|
| Keine S.M.A.R.T.-Werte in Grafana | Exporter aus, Platte nicht auslesbar (USB-Bridge, NVMe), Rechte | `systemctl status prometheus-smartctl-exporter`, `smartctl -a <platte>`, [Hardware-Monitoring](../2-betrieb-hardware/20060-hardware-monitoring.md#bekannte-einschränkungen--fehlersuche) |
| Rolle abgeschaltet | `smartctl_exporter_enabled: false` | Wert prüfen |

Allgemeine Fehlersuche: [Host-Troubleshooting](../a-betriebssystem/a0040-troubleshooting.md), [zentrale Fehlersuche](../../capulus-core/2-betrieb-hardware/20070-troubleshooting-zentral.md).

[Alle Rollen](index.md)
