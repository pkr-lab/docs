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

[Alle Rollen](index.md)
