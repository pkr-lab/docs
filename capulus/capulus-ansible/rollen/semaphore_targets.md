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

---

## Fehlersuche

Allgemeines Vorgehen bei Rollen: `ansible-playbook <playbook>.yml --limit <host> --tags <tag> --check --diff -vvv` zeigt Änderungen ohne sie anzuwenden (`make <ziel>-check` bei den Playbooks mit Dry-Run). Die Rollenübersicht mit Make-Zielen: [Rollen](index.md#fehlersuche-je-rolle).

```bash
sudo cat ~/.ssh/authorized_keys | grep -i semaphore
ssh -i <semaphore-key> ubuntu@<host> true     # vom Homeserver, Schlüssel unter /etc/semaphore-secrets
```

| Symptom | Ursache | Lösung |
|---|---|---|
| Semaphore kann sich nicht per SSH verbinden | Public Key nicht in `authorized_keys` des Ziels | `make semaphore-targets` ausführen (holt den Schlüssel vom Homeserver und trägt ihn ein) |
| `Fetch Semaphore public key` scheitert | `/etc/semaphore-secrets/id_ed25519.pub` fehlt | Rolle [semaphore_secrets](semaphore_secrets.md) zuerst ausrollen |

Allgemeine Fehlersuche: [Host-Troubleshooting](../a-betriebssystem/a0040-troubleshooting.md), [zentrale Fehlersuche](../../capulus-core/2-betrieb-hardware/20070-troubleshooting-zentral.md).

[Alle Rollen](index.md)
