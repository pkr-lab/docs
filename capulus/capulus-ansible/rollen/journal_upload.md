# Rolle `journal_upload`

journald → VictoriaLogs.

| | |
|---|---|
| Läuft gegen | alle Ansible-Hosts |
| Tags | `journal-upload`, `logging` |
| Quellcode | [roles/journal_upload](https://github.com/pkr-lab/capulus-ansible/tree/main/roles/journal_upload) |

## Variablen

Defaults aus `defaults/main.yml`. Überschreiben in `group_vars/` oder `host_vars/`.

| Variable | Default | Bedeutung |
|---|---|---|
| `journal_upload_enabled` | `true` | – |
| `journal_upload_state_dir` | `/var/lib/private/systemd/journal-upload` | – |
| `journal_upload_push_url` | `https://logs-write.tech.homeserver:443/insert/journald` | – |
| `journal_upload_ca_src` | `{{ playbook_dir }}/files/homeserver-root-ca.pem` | – |
| `journal_upload_ca_dest` | `/usr/local/share/ca-certificates/homeserver-root-ca.crt` | – |
| `journal_upload_hosts_entry_enabled` | `true` | – |
| `journal_upload_hosts_entry_name` | `logs-write.tech.homeserver` | – |
| `journal_upload_hosts_entry_ip` | `127.0.0.1` | – |
| `journal_upload_basic_auth_user` | `{{ vault_logs_write_auth_user \| default('') }}` | – |
| `journal_upload_basic_auth_password` | `{{ vault_logs_write_auth_password \| default('') }}` | – |
| `journal_upload_credentials_group` | `jupload-cred` | – |

## Ablauf

1. systemd-journal-remote installieren (liefert systemd-journal-upload)
2. Homeserver-Root-CA-Zertifikat deployen (TLS-Pruefung des Log-Endpunkts)
3. Log-Endpunkt per /etc/hosts aufloesen (unabhaengig von DNS)
4. Drop-in-Verzeichnis fuer systemd-journal-upload anlegen
5. Dauerhaften Neustart fuer systemd-journal-upload konfigurieren
6. Gruppe fuer die Zugangsdaten des Log-Endpunkts anlegen
7. Dienst fuer den Zugriff auf die Zugangsdaten berechtigen
8. journal-upload.conf deployen
9. Startpunkt fuer systemd-journal-upload setzen (kein Upload des Alt-Journals)
10. State-Verzeichnis pruefen
11. State-Datei pruefen
12. Aktuellen Journal-Cursor ermitteln
13. State-Verzeichnis anlegen
14. State-Datei mit Startpunkt anlegen
15. Handler sofort ausfuehren (Start-Limit zuruecksetzen, bevor der Dienst gestartet wird)
16. systemd-journal-upload aktivieren und starten
17. systemd-journal-upload deaktivieren (journal_upload_enabled false)

## Handler

- Homeserver-CA-Vertrauensspeicher aktualisieren
- Start-Limit des Dienstes zuruecksetzen (nach Endlos-Fehlschlaegen)
- Restart systemd-journal-upload

## Templates

- `journal-upload.conf.j2`

---

## Fehlersuche

Allgemeines Vorgehen bei Rollen: `ansible-playbook <playbook>.yml --limit <host> --tags <tag> --check --diff -vvv` zeigt Änderungen ohne sie anzuwenden (`make <ziel>-check` bei den Playbooks mit Dry-Run). Die Rollenübersicht mit Make-Zielen: [Rollen](index.md#fehlersuche-je-rolle).

```bash
sudo systemctl status systemd-journal-upload
sudo journalctl -u systemd-journal-upload -n 100 --no-pager
cat /etc/systemd/journal-upload.conf
getent hosts logs-write.tech.homeserver
sudo cat /var/lib/private/systemd/journal-upload/state
```

| Symptom | Ursache | Lösung |
|---|---|---|
| Dienst `failed` mit Start-Limit | zu viele Neustarts, Endpunkt nicht erreichbar | `systemctl reset-failed systemd-journal-upload`, Endpunkt und Zugangsdaten prüfen |
| `401 Unauthorized` im Log | Basic-Auth-Zugang `logs-writer` stimmt nicht (`vault_logs_write_auth_*`) | Vault-Variablen und Secret `logs-write-auth` prüfen ([Logging](../../capulus-core/3-apps-workloads/300j0-logging.md#fehlerbehebung)) |
| TLS-Fehler (`certificate verify failed`) | Root-CA fehlt (`/usr/local/share/ca-certificates/homeserver-root-ca.crt`) oder `update-ca-certificates` nicht gelaufen | CA ablegen, `sudo update-ca-certificates` |
| Name `logs-write.tech.homeserver` nicht auflösbar | Hosts-Eintrag (`journal_upload_hosts_entry_ip`) zeigt auf falsche Adresse | Eintrag in `/etc/hosts` prüfen |
| Alte Logs fehlen in Grafana | Startpunkt wird auf den aktuellen Cursor gesetzt, es wird kein Alt-Journal hochgeladen | gewollt |
| Dienst hat keinen Zugriff auf das Passwort | Gruppe `jupload-cred` und Dienstberechtigung | Rolle erneut ausrollen |

Allgemeine Fehlersuche: [Host-Troubleshooting](../a-betriebssystem/a0040-troubleshooting.md), [zentrale Fehlersuche](../../capulus-core/2-betrieb-hardware/20070-troubleshooting-zentral.md).

[Alle Rollen](index.md)
