# Rolle `cluster_power_manager_target`

autorisiert den Shutdown-Schlüssel, beschränkt auf `poweroff`.

| | |
|---|---|
| Läuft gegen | worker-0, worker-1 |
| Tags | `cluster-power-manager-target`, `power-management` |
| Quellcode | [roles/cluster_power_manager_target](https://github.com/pkr-lab/capulus-ansible/tree/main/roles/cluster_power_manager_target) |

## Variablen

Defaults aus `defaults/main.yml`. Überschreiben in `group_vars/` oder `host_vars/`.

| Variable | Default | Bedeutung |
|---|---|---|
| `cluster_power_manager_ssh_user` | `ubuntu` | – |
| `cluster_power_manager_ssh_key_path` | `/etc/cluster-power-manager/id_ed25519` | – |
| `cluster_power_manager_poweroff_bin` | `/usr/sbin/poweroff` | – |

## Ablauf

1. Public Key des cluster-power-manager vom Homeserver lesen
2. cluster-power-manager-Key autorisieren (nur für poweroff, forced command)
3. Passwortlosen sudo für poweroff erlauben ({{ cluster_power_manager_ssh_user }})

[Alle Rollen](index.md)
