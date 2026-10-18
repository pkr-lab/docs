# Rolle `cluster_power_manager`

weckt Worker per WoL, fährt sie per SSH wieder herunter.

| | |
|---|---|
| Läuft gegen | homeserver |
| Tags | `cluster-power-manager`, `power-management` |
| Quellcode | [roles/cluster_power_manager](https://github.com/pkr-lab/capulus-ansible/tree/main/roles/cluster_power_manager) |

## Variablen

Defaults aus `defaults/main.yml`. Überschreiben in `group_vars/` oder `host_vars/`.

| Variable | Default | Bedeutung |
|---|---|---|
| `cluster_power_manager_enabled` | `true` | – |
| `cluster_power_manager_poll_seconds` | `30` | – |
| `cluster_power_manager_cpu_scale_up_threshold` | `60` | – |
| `cluster_power_manager_ram_scale_up_threshold` | `60` | – |
| `cluster_power_manager_scale_up_sustain_seconds` | `60` | – |
| `cluster_power_manager_scale_up_sustain_seconds_stage2` | `200` | – |
| `cluster_power_manager_cpu_scale_down_threshold` | `30` | – |
| `cluster_power_manager_ram_scale_down_threshold` | `30` | – |
| `cluster_power_manager_scale_down_sustain_seconds` | `900` | – |
| `cluster_power_manager_min_uptime_seconds` | `300` | – |
| `cluster_power_manager_ready_timeout_seconds` | `120` | – |
| `cluster_power_manager_ssh_user` | `ubuntu` | – |
| `cluster_power_manager_ssh_key_path` | `/etc/cluster-power-manager/id_ed25519` | – |
| `cluster_power_manager_state_dir` | `/var/lib/cluster-power-manager` | – |
| `cluster_power_manager_kubeconfig` | `/etc/rancher/k3s/k3s.yaml` | – |
| `cluster_power_manager_ntfy_host` | `ntfy.tech.homeserver` | – |
| `cluster_power_manager_ntfy_ip` | `{{ network_static_ip }}` | – |
| `cluster_power_manager_ntfy_topic` | `Home-Lab-System` | – |
| `cluster_power_manager_workers` | `[{ansible_host: '{{ hostvars.get(''worker-0'', {}).ansible_host \| default('''') }}', m…` | – |

## Ablauf

1. Warnen, falls eine Worker-MAC-Adresse noch nicht gesetzt ist
2. wakeonlan-Paket installieren (zum Senden von Magic Packets)
3. Config-/State-Verzeichnisse anlegen
4. SSH-Schlüsselpaar für Worker-Shutdown erzeugen (falls noch nicht vorhanden)
5. Berechtigungen des privaten Schlüssels absichern
6. Watchdog-Skript deployen
7. Konfiguration templaten
8. systemd-Unit deployen
9. cluster-power-manager aktivieren und starten
10. cluster-power-manager deaktivieren, falls cluster_power_manager_enabled false

## Handler

- Restart cluster power manager

## Templates

- `cluster-power-manager.service.j2`
- `config.env.j2`

## Dateien

- `cluster-power-manager.sh`

[Alle Rollen](index.md)
