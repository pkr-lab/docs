# Rolle `resource_watchdog`

Selbst-Abschaltung bei Dauerlast.

| | |
|---|---|
| Läuft gegen | alle Knoten und Kiosks |
| Tags | `resource-watchdog`, `thermal` |
| Quellcode | [roles/resource_watchdog](https://github.com/pkr-lab/capulus-ansible/tree/main/roles/resource_watchdog) |

## Variablen

Defaults aus `defaults/main.yml`. Überschreiben in `group_vars/` oder `host_vars/`.

| Variable | Default | Bedeutung |
|---|---|---|
| `resource_watchdog_enabled` | `true` | – |
| `resource_watchdog_poll_seconds` | `10` | – |
| `resource_watchdog_cpu_threshold` | `90` | – |
| `resource_watchdog_cpu_sustain_seconds` | `300` | – |
| `resource_watchdog_ram_threshold` | `90` | – |
| `resource_watchdog_ram_sustain_seconds` | `300` | – |
| `resource_watchdog_cpu_warn_threshold` | `85` | – |
| `resource_watchdog_cpu_warn_sustain_seconds` | `60` | – |
| `resource_watchdog_ram_warn_threshold` | `80` | – |
| `resource_watchdog_ram_warn_sustain_seconds` | `60` | – |
| `resource_watchdog_ntfy_host` | `ntfy.tech.homeserver` | – |
| `resource_watchdog_ntfy_ip` | `{{ network_static_ip }}` | – |
| `resource_watchdog_ntfy_topic` | `Home-Lab-System` | – |
| `resource_watchdog_metrics` | `[{name: cpu, sustain_seconds: '{{ resource_watchdog_cpu_sustain_seconds }}', threshold:…` | – |

## Ablauf

1. Deploy resource watchdog script
2. Ensure resource watchdog config directory exists
3. Deploy per-metric environment files
4. Deploy resource watchdog systemd template unit
5. Enable and start resource watchdog instances
6. Disable resource watchdog instances if turned off via resource_watchdog_enabled

## Handler

- Restart resource watchdogs

## Templates

- `instance.env.j2`
- `resource-watchdog@.service.j2`

## Dateien

- `resource-watchdog.sh`

[Alle Rollen](index.md)
