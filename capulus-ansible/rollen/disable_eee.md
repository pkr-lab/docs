# Rolle `disable_eee`

Energy Efficient Ethernet dauerhaft deaktivieren (Realtek-NIC-Link-Drops).

| | |
|---|---|
| Läuft gegen | homeserver, worker-0, worker-1 |
| Tags | `disable-eee`, `network` |
| Quellcode | [roles/disable_eee](https://github.com/pkr-lab/capulus-ansible/tree/main/roles/disable_eee) |

## Variablen

Defaults aus `defaults/main.yml`. Überschreiben in `group_vars/` oder `host_vars/`.

| Variable | Default | Bedeutung |
|---|---|---|
| `disable_eee_enabled` | `true` | – |
| `disable_eee_interface` | `{{ ansible_default_ipv4.interface }}` | – |

## Ablauf

1. ethtool installieren (für EEE-Konfiguration)
2. systemd-Unit zur EEE-Deaktivierung beim Boot deployen
3. disable-eee.service aktivieren und (neu) starten
4. disable-eee.service deaktivieren, falls disable_eee_enabled false

## Templates

- `disable-eee.service.j2`

[Alle Rollen](index.md)
