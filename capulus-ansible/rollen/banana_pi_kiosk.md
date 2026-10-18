# Rolle `banana_pi_kiosk`

Chromium-Kiosk, Heartbeat-Timer und WoL-Agent.

| | |
|---|---|
| Läuft gegen | Banana Pi |
| Tags | `banana-pi-kiosk`, `kiosk` |
| Quellcode | [roles/banana_pi_kiosk](https://github.com/pkr-lab/capulus-ansible/tree/main/roles/banana_pi_kiosk) |

## Variablen

Defaults aus `defaults/main.yml`. Überschreiben in `group_vars/` oder `host_vars/`.

| Variable | Default | Bedeutung |
|---|---|---|
| `banana_pi_kiosk_enabled` | `true` | – |
| `banana_pi_kiosk_ca_src` | `{{ playbook_dir }}/files/homeserver-root-ca.pem` | – |
| `banana_pi_kiosk_base_url` | `https://alamos-apager.prod.homeserver` | – |
| `banana_pi_kiosk_user` | `pela` | – |
| `banana_pi_kiosk_heartbeat_interval` | `60` | – |
| `banana_pi_kiosk_failover_poll_seconds` | `30` | – |
| `banana_pi_kiosk_failover_fail_threshold` | `3` | – |
| `banana_pi_kiosk_failover_recover_threshold` | `1` | – |
| `banana_pi_kiosk_periodic_refresh_seconds` | `0` | – |
| `banana_pi_kiosk_daily_reboot_enabled` | `false` | – |
| `banana_pi_kiosk_daily_reboot_time` | `00:00:00` | – |
| `banana_pi_kiosk_wol_enabled` | `true` | – |
| `banana_pi_kiosk_wol_devices` | `{}` | – |
| `banana_pi_kiosk_wol_agent_enabled` | `true` | – |
| `banana_pi_kiosk_wol_agent_port` | `9102` | – |
| `banana_pi_kiosk_wol_agent_bind_address` | `0.0.0.0` | – |
| `banana_pi_kiosk_wol_agent_token_path` | `/etc/banana-pi-wol-agent/token` | – |
| `banana_pi_kiosk_wol_agent_tls_enabled` | `false` | – |
| `banana_pi_kiosk_wol_agent_tls_crt` | `` | – |
| `banana_pi_kiosk_wol_agent_tls_key` | `` | – |
| `banana_pi_kiosk_wol_agent_tls_crt_path` | `/etc/banana-pi-wol-agent/tls.crt` | – |
| `banana_pi_kiosk_wol_agent_tls_key_path` | `/etc/banana-pi-wol-agent/tls.key` | – |

## Ablauf

1. Sicherstellen dass banana_pi_kiosk_station gesetzt ist
2. Sicherstellen dass banana_pi_kiosk_fallback_url gesetzt und entschluesselt ist
3. Kiosk-Pakete installieren (Debian-Paketnamen, nicht Raspbian)
4. Homeserver-Root-CA-Zertifikat deployen
5. NSS-Datenbank-Verzeichnis des Kiosk-Users anlegen
6. Pruefen ob Homeserver-Root-CA bereits in NSS-Datenbank des Kiosk-Users vertraut ist
7. Homeserver-Root-CA in NSS-Datenbank des Kiosk-Users importieren
8. lightdm stoppen und deaktivieren (falls vorhanden)
9. Verzeichnis fuer getty@tty1-Drop-in anlegen
10. Autologin-Drop-in fuer getty@tty1 deployen
11. .bash_profile von {{ banana_pi_kiosk_user }} um automatischen startx-Aufruf ergaenzen
12. .xinitrc deployen
13. Kiosk-Supervisor-Skript deployen
14. Screenshot-Skript deployen
15. wakeonlan installieren
16. Wake-on-LAN-Skript deployen
17. Config-Verzeichnis für WoL-HTTP-Agent anlegen
18. Bearer-Token für WoL-HTTP-Agent erzeugen (falls noch nicht vorhanden)
19. Token-Datei-Rechte absichern
20. Erzeugtes WoL-Agent-Token einlesen
21. TLS-Zertifikat und -Schlüssel für den WoL-HTTP-Agent prüfen
22. TLS-Zertifikat für den WoL-HTTP-Agent ablegen
23. TLS-Schlüssel für den WoL-HTTP-Agent ablegen
24. TLS-Dateien des WoL-HTTP-Agent entfernen, falls TLS aus
25. WoL-HTTP-Agent-Skript deployen
26. WoL-HTTP-Agent-Konfiguration templaten
27. WoL-HTTP-Agent-systemd-Unit deployen
28. WoL-HTTP-Agent aktivieren und starten
29. WoL-HTTP-Agent deaktivieren, falls banana_pi_kiosk_wol_agent_enabled false
30. Hinweis zum WoL-Agent-Token ausgeben
31. Heartbeat-Skript deployen
32. Heartbeat-Environment-Datei deployen
33. Heartbeat-systemd-Service + Timer deployen
34. Heartbeat-Timer aktivieren
35. Heartbeat-Timer deaktivieren (banana_pi_kiosk_enabled false)
36. Daily-Reboot-systemd-Service + Timer deployen
37. Daily-Reboot-Timer aktivieren
38. Daily-Reboot-Timer deaktivieren (banana_pi_kiosk_daily_reboot_enabled false)

## Handler

- Homeserver-CA-Vertrauensspeicher aktualisieren
- Restart banana-pi heartbeat timer
- Reload getty@tty1
- Restart banana-pi daily reboot timer
- Restart banana-pi wol agent

## Templates

- `banana-pi-daily-reboot.service.j2`
- `banana-pi-daily-reboot.timer.j2`
- `banana-pi-heartbeat.env.j2`
- `banana-pi-heartbeat.service.j2`
- `banana-pi-heartbeat.timer.j2`
- `banana-pi-kiosk-supervisor.sh.j2`
- `banana-pi-screenshot.sh.j2`
- `banana-pi-wol-agent.env.j2`
- `banana-pi-wol-agent.service.j2`
- `banana-pi-wol.sh.j2`
- `getty-autologin.conf.j2`
- `xinitrc.j2`

## Dateien

- `banana-pi-heartbeat.sh`
- `banana-pi-wol-agent.py`

[Alle Rollen](index.md)
