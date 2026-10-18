# Rolle `wireguard_backup`

Notfall-Tunnel, falls die Tailscale-Control-Plane ausfällt.

| | |
|---|---|
| Läuft gegen | homeserver |
| Tags | `backup`, `vpn`, `wireguard` |
| Quellcode | [roles/wireguard_backup](https://github.com/pkr-lab/capulus-ansible/tree/main/roles/wireguard_backup) |

## Variablen

Defaults aus `defaults/main.yml`. Überschreiben in `group_vars/` oder `host_vars/`.

| Variable | Default | Bedeutung |
|---|---|---|
| `wireguard_backup_interface` | `wg-backup` | – |
| `wireguard_backup_server_tunnel_ip` | `10.99.99.1/24` | – |
| `wireguard_backup_peer_tunnel_ip` | `10.99.99.2/32` | – |
| `wireguard_backup_peer_name` | `admin-backup-device` | – |

## Ablauf

1. Install WireGuard tools
2. Ensure /etc/wireguard exists with restrictive permissions
3. Check whether a server private key already exists
4. Generate server private key (first run only)
5. Read server private key
6. Derive server public key
7. Template WireGuard backup tunnel config
8. Allow WireGuard backup port through UFW
9. Enable and start wg-quick service for the backup tunnel
10. Display server public key and connection info

## Handler

- Restart wireguard backup tunnel

## Templates

- `wg-backup.conf.j2`

[Alle Rollen](index.md)
