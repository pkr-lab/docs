# Rolle `semaphore_targets`

SSH-Pubkey von Semaphore in `authorized_keys`.

| | |
|---|---|
| Läuft gegen | alle verwalteten Hosts |
| Tags | – |
| Quellcode | [roles/semaphore_targets](https://github.com/pkr-lab/capulus-ansible/tree/main/roles/semaphore_targets) |

## Variablen

Defaults aus `defaults/main.yml`. Überschreiben in `group_vars/` oder `host_vars/`.

| Variable | Default | Bedeutung |
|---|---|---|
| `semaphore_public_key_path` | `/etc/semaphore-secrets/id_ed25519.pub` | – |
| `semaphore_authorize_user` | `{{ ansible_user }}` | – |

## Ablauf

1. Fetch Semaphore public key from the home-server
2. Authorize Semaphore key for {{ semaphore_authorize_user }}

[Alle Rollen](index.md)
