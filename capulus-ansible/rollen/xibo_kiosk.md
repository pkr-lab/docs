# Rolle `xibo_kiosk`

Sync-Timer mit lokalem Bilder-Cache, Chromium-Slideshow-Kiosk in eigener X11-Sitzung.

| | |
|---|---|
| Läuft gegen | xibosignage-Displays |
| Tags | `kiosk`, `xibo-kiosk` |
| Quellcode | [roles/xibo_kiosk](https://github.com/pkr-lab/capulus-ansible/tree/main/roles/xibo_kiosk) |
| Betrieb und Ablauf | [300e8](../3-apps-workloads/300e8-infotafel-kiosk.md) |

## Variablen

Defaults aus `defaults/main.yml`. Überschreiben in `group_vars/` oder `host_vars/`.

| Variable | Default | Bedeutung |
|---|---|---|
| `xibo_kiosk_enabled` | `true` | Rolle an/aus |
| `xibo_kiosk_nfs_server` | `192.168.178.97` | NAS mit dem Display-Ordner |
| `xibo_kiosk_nfs_path` | `/volume1/k8s-storage/xibosignage-display` | Export auf dem NAS |
| `xibo_kiosk_mount_point` | `/mnt/xibosignage-display` | lokaler Mountpoint (nur der Sync mountet) |
| `xibo_kiosk_web_root` | `/var/www/xibosignage-slideshow` | Seite, `media/`-Cache und `manifest.js` |
| `xibo_kiosk_start_mode` | `systemd` | `systemd` (früher Start per Dienst) oder `getty` (Autologin und `startx`) |
| `xibo_kiosk_supplementary_groups` | `video`, `render` | Gruppen des Kiosk-Dienstes im Modus `systemd` |
| `xibo_kiosk_sync_interval_seconds` | `30` | Abstand der Sync-Läufe |
| `xibo_kiosk_sync_boot_delay_seconds` | `20` | Verzögerung des ersten Sync-Laufs nach dem Boot |
| `xibo_kiosk_sync_settle_seconds` | `5` | Quelldateien, die jünger sind, werden erst im nächsten Lauf geholt |
| `xibo_kiosk_manifest_reload_seconds` | `10` | wie oft die Seite `manifest.js` neu liest |
| `xibo_kiosk_slide_duration_ms` | `10000` | Anzeigedauer pro Bild |
| `xibo_kiosk_image_max_width` | `1920` | Zielbreite der Skalierung |
| `xibo_kiosk_image_max_height` | `1080` | Zielhöhe der Skalierung |
| `xibo_kiosk_image_jpeg_quality` | `85` | Qualität beim Skalieren |
| `xibo_kiosk_cursor_hide_seconds` | `3` | Idle-Zeit bis der Mauszeiger verschwindet |
| `xibo_kiosk_display_poll_seconds` | `5` | Abstand der Prüfung auf den nativen Bildschirmmodus |
| `xibo_kiosk_display_mode` | leer | erzwingt einen Modus, wenn der Bildschirm kein EDID liefert |
| `xibo_kiosk_pixel_aspect` | `auto` | Seitenverhältnis-Korrektur der Seite: `auto`, `off` oder fester Faktor |
| `xibo_kiosk_user` | `pi` | Kiosk-User (Autologin, `startx`) |
| `xibo_kiosk_chromium_extra_flags` | `[]` | zusätzliche Chromium-Flags |
| `xibo_kiosk_disable_services` | `ModemManager`, `triggerhappy`, `cups-browsed`, `cups`, `cups.socket`, `e2scrub_reap`, `upower` | Dienste, die die Rolle deaktiviert |

## Ablauf

1. Pakete für NFS-Mount, Sync und Kiosk installieren
2. Startmodus prüfen, alte Webserver- und Manifest-Units entfernen
3. xibosignage-display ggf. aushängen
4. Mount-Point für xibosignage-display anlegen
5. xibosignage-display in fstab eintragen (read-only, noauto, wird vom Sync gemountet)
6. Webroot für Slideshow anlegen
7. Alten media-Symlink auf den NFS-Mount prüfen und entfernen
8. Lokales Bilder-Cache-Verzeichnis anlegen
9. Statische Slideshow-Dateien deployen
10. Slideshow-Konfiguration (config.js) deployen
11. Leeres manifest.js anlegen (bis der erste Sync-Lauf es ersetzt)
12. Sync- und Diagnose-Skript deployen
13. Sync-systemd-Service und Timer deployen
14. Bildschirmmodus-Skript deployen
15. Kiosk-Start-Skript deployen
16. apt-Exporter des Monitorings per Drop-in niedrig priorisieren, eindeutig unkritische Dienste deaktivieren
17. Desktop-Sitzung abschalten: lightdm maskieren, Standard-Target `multi-user`
18. Modus `systemd`: `xibosignage-xorg.service` und `xibosignage-kiosk.service` deployen und für den nächsten Boot aktivieren, Autologin-Drop-in und `.profile`-Block entfernen
19. Modus `getty`: beide Dienste entfernen, getty-Autologin und `startx` in `.profile` eintragen
20. Slideshow-Sync aktivieren
21. Ersten Sync einmalig ausführen
22. Slideshow-Sync deaktivieren (`xibo_kiosk_enabled` false)

## Handler

- Reload systemd
- Restart xibosignage sync timer

## Templates

- `getty-autologin.conf.j2`
- `xibosignage-kiosk.service.j2`
- `xibosignage-xorg.service.j2`
- `xibosignage-display-mode.sh.j2`
- `xibosignage-config.js.j2`
- `xibosignage-kiosk-start.sh.j2`
- `xibosignage-sync.service.j2`
- `xibosignage-sync.timer.j2`

## Dateien

- `default.svg`
- `index.html`
- `slideshow.css`
- `slideshow.js`
- `xibosignage-diagnose.sh`
- `xibosignage-sync.py`

[Alle Rollen](index.md)
