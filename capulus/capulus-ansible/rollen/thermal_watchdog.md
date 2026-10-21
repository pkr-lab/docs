# Rolle `thermal_watchdog`

Selbst-Abschaltung bei Übertemperatur.

| | |
|---|---|
| Läuft gegen | alle Knoten und Kiosks |
| Tags | `thermal`, `thermal-watchdog` |
| Quellcode | [roles/thermal_watchdog](https://github.com/pkr-lab/capulus-ansible/tree/main/roles/thermal_watchdog) |

## Variablen

Defaults aus `defaults/main.yml`. Überschreiben in `group_vars/` oder `host_vars/`.

| Variable | Default | Bedeutung |
|---|---|---|
| `thermal_watchdog_enabled` | `true` | – |
| `thermal_watchdog_threshold_c` | `100` | – |
| `thermal_watchdog_sustain_seconds` | `90` | – |
| `thermal_watchdog_poll_seconds` | `10` | – |
| `thermal_watchdog_ntfy_host` | `ntfy.tech.homeserver` | – |
| `thermal_watchdog_ntfy_ip` | `{{ network_static_ip }}` | – |
| `thermal_watchdog_ntfy_topic` | `Home-Lab-System` | – |

## Ablauf

1. Install lm-sensors (hwmon kernel modules / sysfs helpers)
2. Deploy thermal watchdog script
3. Deploy thermal watchdog systemd unit
4. Enable and start thermal watchdog
5. Disable thermal watchdog if turned off via thermal_watchdog_enabled

## Handler

- Restart thermal-watchdog

## Templates

- `thermal-watchdog.service.j2`

## Dateien

- `thermal-watchdog.sh`

---

## Fehlersuche

Allgemeines Vorgehen bei Rollen: `ansible-playbook <playbook>.yml --limit <host> --tags <tag> --check --diff -vvv` zeigt Änderungen ohne sie anzuwenden (`make <ziel>-check` bei den Playbooks mit Dry-Run). Die Rollenübersicht mit Make-Zielen: [Rollen](index.md#fehlersuche-je-rolle).

```bash
sudo systemctl status thermal-watchdog
sudo journalctl -u thermal-watchdog -n 100 --no-pager
cat /sys/class/thermal/thermal_zone*/temp
```

| Symptom | Ursache | Lösung |
|---|---|---|
| Host schaltet sich ab | Temperatur über `thermal_watchdog_threshold_c` (100 °C) für `sustain_seconds` (90 s) — Selbstschutz | Lüfter, Staub, Last prüfen; Hardware-Dashboard ([Hardware-Monitoring](../2-betrieb-hardware/20060-hardware-monitoring.md)) |
| Keine ntfy-Meldung vor dem Abschalten | Publisher-Token oder CA fehlt | [ntfy_publisher](ntfy_publisher.md) |
| Dienst startet nicht | Sensor fehlt (VM oder Gerät ohne Thermalzone) | `ls /sys/class/thermal/`, Rolle für diesen Host abschalten |

Allgemeine Fehlersuche: [Host-Troubleshooting](../a-betriebssystem/a0040-troubleshooting.md), [zentrale Fehlersuche](../../capulus-core/2-betrieb-hardware/20070-troubleshooting-zentral.md).

[Alle Rollen](index.md)
