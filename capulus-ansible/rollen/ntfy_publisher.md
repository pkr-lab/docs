# Rolle `ntfy_publisher`

Meldungen der Watchdogs und Host-Skripte per HTTPS an ntfy, mit Root-CA und Publisher-Token.

| | |
|---|---|
| Läuft gegen | alle Knoten und Kiosks |
| Tags | `ntfy-publisher`, `thermal` |
| Quellcode | [roles/ntfy_publisher](https://github.com/pkr-lab/capulus-ansible/tree/main/roles/ntfy_publisher) |

## Variablen

Defaults aus `defaults/main.yml`. Überschreiben in `group_vars/` oder `host_vars/`.

| Variable | Default | Bedeutung |
|---|---|---|
| `ntfy_publisher_dir` | `/etc/ntfy-publisher` | – |
| `ntfy_publisher_ca_src` | `{{ playbook_dir }}/files/homeserver-root-ca.pem` | – |

## Ablauf

1. verzeichnis für ntfy-zugangsdaten anlegen
2. ntfy-auth-header-datei schreiben
3. homeserver-root-ca für die ntfy-meldungen ablegen

[Alle Rollen](index.md)
