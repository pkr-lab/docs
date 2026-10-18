# Rolle `vmagent`

Metriken-Push per remote-write.

| | |
|---|---|
| Läuft gegen | Tailscale-only-Hosts (Banana Pi) |
| Tags | `monitoring`, `vmagent` |
| Quellcode | [roles/vmagent](https://github.com/pkr-lab/capulus-ansible/tree/main/roles/vmagent) |

## Variablen

Defaults aus `defaults/main.yml`. Überschreiben in `group_vars/` oder `host_vars/`.

| Variable | Default | Bedeutung |
|---|---|---|
| `vmagent_enabled` | `true` | – |
| `vmagent_version` | `1.149.0` | – |
| `vmagent_arch` | `arm` | – |
| `vmagent_sha256` | `{arm: b87fdf5bb91978bd183bf284e90930f8f1fd083b8e1151ef8bc46614435b146e}` | – |
| `vmagent_download_url` | `https://github.com/VictoriaMetrics/VictoriaMetrics/releases/download/v{{ vmagent_versio…` | – |
| `vmagent_install_dir` | `/opt/vmagent-{{ vmagent_version }}` | – |
| `vmagent_node_exporter_port` | `9100` | – |
| `vmagent_scrape_interval` | `30s` | – |
| `vmagent_remote_write_queues` | `1` | – |
| `vmagent_remote_write_basic_auth_user` | `{{ vault_vm_write_auth_user \| default('') }}` | – |
| `vmagent_remote_write_basic_auth_password` | `{{ vault_vm_write_auth_password \| default('') }}` | – |
| `vmagent_remote_write_password_file` | `/etc/vmagent/write-auth-password` | – |

## Ablauf

1. Sicherstellen dass vmagent_remote_write_url gesetzt ist
2. System-User fuer vmagent anlegen
3. Verzeichnis fuer lokalen remote_write-Puffer anlegen
4. Konfigurationsverzeichnis anlegen
5. vmutils-Release herunterladen
6. Installationsverzeichnis fuer vmagent anlegen
7. vmagent-Binary entpacken
8. vmagent-Binary ausfuehrbar machen
9. Zugangsdaten fuer den Schreib-Endpunkt ablegen
10. Scrape-Konfiguration deployen
11. vmagent-systemd-Service deployen
12. vmagent aktivieren und starten
13. vmagent deaktivieren (vmagent_enabled false)

## Handler

- Restart vmagent

## Templates

- `vmagent-scrape.yml.j2`
- `vmagent.service.j2`

[Alle Rollen](index.md)
