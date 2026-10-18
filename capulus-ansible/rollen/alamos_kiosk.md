# Rolle `alamos_kiosk`

Chromium-Kiosk und Heartbeat-Timer für den Alarmmonitor.

| | |
|---|---|
| Läuft gegen | Raspberry Pis |
| Tags | `alamos-kiosk`, `kiosk` |
| Quellcode | [roles/alamos_kiosk](https://github.com/pkr-lab/capulus-ansible/tree/main/roles/alamos_kiosk) |

## Variablen

Defaults aus `defaults/main.yml`. Überschreiben in `group_vars/` oder `host_vars/`.

| Variable | Default | Bedeutung |
|---|---|---|
| `alamos_kiosk_enabled` | `true` | – |
| `alamos_kiosk_ca_src` | `{{ playbook_dir }}/files/homeserver-root-ca.pem` | – |
| `alamos_kiosk_base_url` | `https://alamos-apager.prod.homeserver` | – |
| `alamos_kiosk_heartbeat_interval` | `60` | – |
| `alamos_kiosk_user` | `pela` | – |

## Ablauf

1. Sicherstellen dass alamos_kiosk_station gesetzt ist
2. Kiosk-Pakete installieren
3. Homeserver-Root-CA-Zertifikat deployen
4. Kiosk-Start-Skript deployen
5. Kiosk-systemd-Service deployen
6. Heartbeat-Skript deployen
7. Heartbeat-Environment-Datei deployen
8. Heartbeat-systemd-Service + Timer deployen
9. Kiosk-Service und Heartbeat-Timer aktivieren
10. Kiosk-Service und Heartbeat-Timer deaktivieren (alamos_kiosk_enabled false)

## Handler

- Homeserver-CA-Vertrauensspeicher aktualisieren
- Restart alamos kiosk
- Restart alamos heartbeat timer

## Templates

- `alamos-heartbeat.env.j2`
- `alamos-heartbeat.service.j2`
- `alamos-heartbeat.timer.j2`
- `alamos-kiosk-start.sh.j2`
- `alamos-kiosk.service.j2`

## Dateien

- `alamos-heartbeat.sh`

[Alle Rollen](index.md)
