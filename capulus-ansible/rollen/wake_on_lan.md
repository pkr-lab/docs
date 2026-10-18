# Rolle `wake_on_lan`

`ethtool wol g` bei jedem Boot.

| | |
|---|---|
| Läuft gegen | worker-0, worker-1 |
| Tags | `power-management`, `wake-on-lan` |
| Quellcode | [roles/wake_on_lan](https://github.com/pkr-lab/capulus-ansible/tree/main/roles/wake_on_lan) |

## Variablen

Defaults aus `defaults/main.yml`. Überschreiben in `group_vars/` oder `host_vars/`.

| Variable | Default | Bedeutung |
|---|---|---|
| `wake_on_lan_enabled` | `true` | – |
| `wake_on_lan_interface` | `{{ ansible_default_ipv4.interface }}` | – |

## Ablauf

1. ethtool installieren (für WoL-Konfiguration)
2. systemd-Unit zur WoL-Aktivierung beim Boot deployen
3. enable-wol.service aktivieren und (neu) starten
4. enable-wol.service deaktivieren, falls wake_on_lan_enabled false

## Templates

- `enable-wol.service.j2`

[Alle Rollen](index.md)
