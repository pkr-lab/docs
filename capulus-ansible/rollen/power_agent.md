# Rolle `power_agent`

HTTP-API für Helligkeit, Wake und Shutdown aus der iOS-App.

| | |
|---|---|
| Läuft gegen | homeserver |
| Tags | `power-agent`, `power-management` |
| Quellcode | [roles/power_agent](https://github.com/pkr-lab/capulus-ansible/tree/main/roles/power_agent) |

## Variablen

Defaults aus `defaults/main.yml`. Überschreiben in `group_vars/` oder `host_vars/`.

| Variable | Default | Bedeutung |
|---|---|---|
| `power_agent_enabled` | `true` | – |
| `power_agent_port` | `9101` | – |
| `power_agent_bind_address` | `0.0.0.0` | – |
| `power_agent_known_hosts_path` | `/etc/power-agent/known_hosts` | – |
| `power_agent_token_path` | `/etc/power-agent/token` | – |
| `power_agent_backlight_path` | `/sys/class/backlight/intel_backlight/brightness` | – |
| `power_agent_backlight_max_path` | `/sys/class/backlight/intel_backlight/max_brightness` | – |
| `power_agent_bl_power_path` | `/sys/class/backlight/intel_backlight/bl_power` | – |
| `power_agent_poweroff_bin` | `/usr/sbin/poweroff` | – |
| `power_agent_node_name` | `{{ inventory_hostname }}` | – |
| `power_agent_ssh_user` | `{{ cluster_power_manager_ssh_user }}` | – |
| `power_agent_ssh_key` | `{{ cluster_power_manager_ssh_key_path }}` | – |
| `power_agent_kubeconfig` | `{{ cluster_power_manager_kubeconfig }}` | – |
| `power_agent_state_dir` | `{{ cluster_power_manager_state_dir }}` | – |
| `power_agent_targets` | `{{ cluster_power_manager_workers }}` | – |

## Ablauf

1. Python3 installieren (Laufzeit für power-agent.py)
2. Config-Verzeichnis anlegen
3. Bearer-Token für power-agent erzeugen (falls noch nicht vorhanden)
4. Token-Datei-Rechte absichern
5. Erzeugtes Token einlesen
6. bekannte hostschlüssel der worker aus der root-known_hosts lesen
7. hostschlüssel fehlender worker einmalig abrufen
8. known_hosts-zeilen zusammenstellen
9. festes known_hosts für die ssh-aufrufe des power-agent schreiben
10. warnen, wenn kein worker-hostschlüssel gefunden wurde
11. power-agent.py deployen
12. Konfiguration templaten
13. systemd-Unit deployen
14. power-agent aktivieren und starten
15. power-agent deaktivieren, falls power_agent_enabled false
16. Hinweis zum Sealed-Secret-Abgleich ausgeben

## Handler

- Restart power agent

## Templates

- `config.env.j2`
- `power-agent.service.j2`

## Dateien

- `power-agent.py`

[Alle Rollen](index.md)
