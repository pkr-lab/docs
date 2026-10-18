# Rolle `xibo_kiosk`

NFS-Mount, Manifest-Timer, Chromium-Slideshow-Kiosk.

| | |
|---|---|
| Läuft gegen | xibosignage-Displays |
| Tags | `kiosk`, `xibo-kiosk` |
| Quellcode | [roles/xibo_kiosk](https://github.com/pkr-lab/capulus-ansible/tree/main/roles/xibo_kiosk) |

## Variablen

Defaults aus `defaults/main.yml`. Überschreiben in `group_vars/` oder `host_vars/`.

| Variable | Default | Bedeutung |
|---|---|---|
| `xibo_kiosk_enabled` | `true` | – |
| `xibo_kiosk_nfs_server` | `192.168.178.97` | – |
| `xibo_kiosk_nfs_path` | `/volume1/k8s-storage/xibosignage-display` | – |
| `xibo_kiosk_mount_point` | `/mnt/xibosignage-display` | – |
| `xibo_kiosk_http_port` | `8080` | – |
| `xibo_kiosk_web_root` | `/var/www/xibosignage-slideshow` | – |
| `xibo_kiosk_manifest_interval_seconds` | `30` | – |
| `xibo_kiosk_slide_duration_ms` | `10000` | – |
| `xibo_kiosk_user` | `pi` | – |

## Ablauf

1. Pakete für NFS-Mount + Kiosk installieren
2. xibosignage-display vor Attribut-Fix ggf. aushängen
3. Mount-Point für xibosignage-display anlegen
4. xibosignage-display per NFS mounten (read-only)
5. Webroot für Slideshow anlegen
6. Symlink "media" auf den NFS-Mount legen
7. Statische Slideshow-Dateien deployen
8. Slideshow-Konfiguration (config.js) deployen
9. Leere manifest.json anlegen (bis der erste Timer-Lauf sie ersetzt)
10. Manifest-Generator-Skript deployen
11. Manifest-systemd-Service + Timer deployen
12. Webserver-systemd-Service deployen
13. Kiosk-Start-Skript deployen
14. Kiosk-systemd-Service deployen
15. Manifest-Generator einmalig ausführen (erste manifest.json mit echtem Inhalt)
16. Slideshow-Dienste aktivieren
17. Slideshow-Dienste deaktivieren (xibo_kiosk_enabled false)

## Handler

- Restart xibosignage webserver
- Restart xibosignage manifest timer
- Restart xibosignage kiosk

## Templates

- `xibosignage-config.js.j2`
- `xibosignage-kiosk-start.sh.j2`
- `xibosignage-kiosk.service.j2`
- `xibosignage-manifest.service.j2`
- `xibosignage-manifest.sh.j2`
- `xibosignage-manifest.timer.j2`
- `xibosignage-webserver.service.j2`

## Dateien

- `index.html`
- `slideshow.css`
- `slideshow.js`

[Alle Rollen](index.md)
