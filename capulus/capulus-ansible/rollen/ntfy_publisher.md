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

---

## Fehlersuche

Allgemeines Vorgehen bei Rollen: `ansible-playbook <playbook>.yml --limit <host> --tags <tag> --check --diff -vvv` zeigt Änderungen ohne sie anzuwenden (`make <ziel>-check` bei den Playbooks mit Dry-Run). Die Rollenübersicht mit Make-Zielen: [Rollen](index.md#fehlersuche-je-rolle).

```bash
ls -l /etc/ntfy-publisher
cat /etc/ntfy-publisher/auth-header | sed "s/Bearer .*/Bearer <TOKEN>/"
curl -sS -H "$(cat /etc/ntfy-publisher/auth-header)" -d "Test" --cacert /etc/ntfy-publisher/homeserver-root-ca.pem https://ntfy.tech.homeserver/Home-Lab-System
```

| Symptom | Ursache | Lösung |
|---|---|---|
| Watchdog-Meldungen kommen nicht an | Token fehlt oder falsch (`/etc/ntfy-publisher/auth-header`), Root-CA fehlt, ntfy nicht erreichbar | Test mit `curl` wie oben, [ntfy: Zugangsdaten](../../capulus-core/1-benachrichtigungen/10010-ntfy.md#authentifizierung-und-zugangsdaten) |
| `403 Forbidden` | Token ohne Schreibrecht auf das Topic | Publisher-Token prüfen ([ntfy](../../capulus-core/1-benachrichtigungen/10010-ntfy.md)) |
| TLS-Fehler | Root-CA nicht abgelegt oder nicht passend | Rolle erneut ausrollen, CA aus `files/homeserver-root-ca.pem` |

Allgemeine Fehlersuche: [Host-Troubleshooting](../a-betriebssystem/a0040-troubleshooting.md), [zentrale Fehlersuche](../../capulus-core/2-betrieb-hardware/20070-troubleshooting-zentral.md).

[Alle Rollen](index.md)
