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

---

## Fehlersuche

Allgemeines Vorgehen bei Rollen: `ansible-playbook <playbook>.yml --limit <host> --tags <tag> --check --diff -vvv` zeigt Änderungen ohne sie anzuwenden (`make <ziel>-check` bei den Playbooks mit Dry-Run). Die Rollenübersicht mit Make-Zielen: [Rollen](index.md#fehlersuche-je-rolle).

```bash
sudo systemctl status alamos-kiosk alamos-heartbeat.timer
sudo journalctl -u alamos-kiosk -u alamos-heartbeat -n 100 --no-pager
cat /etc/alamos-heartbeat.env      # Station und Basis-URL (kein Token)
```

| Symptom | Ursache | Lösung |
|---|---|---|
| Playbook bricht ab: `alamos_kiosk_station ist nicht gesetzt.` | Host-Variable fehlt | `alamos_kiosk_station` je Host in `inventory/hosts.yml` setzen; der Wert muss zu einem Schlüssel im SealedSecret `alamos-apager-stations` passen |
| Bildschirm schwarz oder Browser zeigt Fehlerseite | Kiosk-Dienst läuft nicht, Basis-URL nicht erreichbar, Root-CA fehlt | `systemctl status alamos-kiosk`, `curl -v https://alamos-apager.prod.homeserver` vom Pi, Root-CA (`alamos_kiosk_ca_src`) prüfen |
| Station erscheint im Dashboard als offline | Heartbeat-Timer läuft nicht (alle `alamos_kiosk_heartbeat_interval` Sekunden) | `systemctl list-timers alamos-heartbeat.timer`, `journalctl -u alamos-heartbeat`; Station und Server: [alamos-apager](../../capulus-core/3-apps-workloads/30010-alamos-apager.md#fehlerbehebung) |
| Kiosk startet nach einem Absturz nicht neu | `Restart=always` greift nur, wenn der Dienst aktiv ist | `sudo systemctl restart alamos-kiosk` |

Allgemeine Fehlersuche: [Host-Troubleshooting](../a-betriebssystem/a0040-troubleshooting.md), [zentrale Fehlersuche](../../capulus-core/2-betrieb-hardware/20070-troubleshooting-zentral.md).

[Alle Rollen](index.md)
