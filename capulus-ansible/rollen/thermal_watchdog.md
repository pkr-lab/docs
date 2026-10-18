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

[Alle Rollen](index.md)
