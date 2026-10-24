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
| `xibo_kiosk_orphan_grace_seconds` | `30` | Wartezeit, bevor eine aus der Quelle verschwundene Bilddatei lokal gelöscht wird (sollte deutlich über `…manifest_reload_seconds` liegen, siehe [50030](../../capulus-core/5-incidents/50030-infotafel-altes-bildformat-und-nas-sync.md#problem-3-slides-werden-manchmal-sofort-übersprungen)) |
| `xibo_kiosk_slide_duration_ms` | `20000` | Anzeigedauer pro Bild |
| `xibo_kiosk_image_max_width` | `1920` | Zielbreite der Skalierung |
| `xibo_kiosk_image_max_height` | `1080` | Zielhöhe der Skalierung |
| `xibo_kiosk_image_jpeg_quality` | `85` | Qualität beim Skalieren |
| `xibo_kiosk_cursor_hide_seconds` | `3` | Idle-Zeit bis der Mauszeiger verschwindet |
| `xibo_kiosk_display_poll_seconds` | `5` | Abstand der Prüfung auf den nativen Bildschirmmodus |
| `xibo_kiosk_display_mode` | `1920x1080` | erzwungener Modus (immer aktiv, kein Scan nach der tatsächlichen Auflösung mehr) |
| `xibo_kiosk_pixel_aspect` | `auto` | Seitenverhältnis-Korrektur der Seite: `auto`, `off` oder fester Faktor |
| `xibo_kiosk_edid_force` | `false` | erzwingt eine synthetische EDID per `drm.edid_firmware=`, wenn der Bildschirm selbst keine liefert (Kernel-Parameter, Reboot nötig) |
| `xibo_kiosk_edid_connector` | leer | Connector-Name für `xibo_kiosk_edid_force`, z. B. `HDMI-A-1` (`cat /sys/class/drm/card*-HDMI-A-*/status`) |
| `xibo_kiosk_edid_filename` | `{{ xibo_kiosk_display_mode }}.bin` | Dateiname unter `/lib/firmware/edid/` |
| `xibo_kiosk_edid_base64` | leer | Base64 der EDID-Binärdatei, erzeugt mit [`scripts/generate-edid.py`](https://github.com/pkr-lab/capulus-ansible/blob/main/scripts/generate-edid.py) |
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
14. `xibo_kiosk_edid_force`: synthetische EDID-Datei nach `/lib/firmware/edid/` deployen, `drm.edid_firmware=` in `cmdline.txt` sicherstellen (Reboot-Hinweis per Handler)
15. Bildschirmmodus-Skript deployen
16. Kiosk-Start-Skript deployen
17. apt-Exporter des Monitorings per Drop-in niedrig priorisieren, eindeutig unkritische Dienste deaktivieren
18. Desktop-Sitzung abschalten: lightdm maskieren, Standard-Target `multi-user`
19. Modus `systemd`: `xibosignage-xorg.service` und `xibosignage-kiosk.service` deployen und für den nächsten Boot aktivieren, Autologin-Drop-in und `.profile`-Block entfernen
20. Modus `getty`: beide Dienste entfernen, getty-Autologin und `startx` in `.profile` eintragen
21. Slideshow-Sync aktivieren
22. Ersten Sync einmalig ausführen
23. Slideshow-Sync deaktivieren (`xibo_kiosk_enabled` false)

## Handler

- Reload systemd
- Restart xibosignage sync timer
- Hinweis auf nötigen Reboot nach EDID-Erzwingung

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

---

## Fehlersuche

Allgemeines Vorgehen bei Rollen: `ansible-playbook <playbook>.yml --limit <host> --tags <tag> --check --diff -vvv` zeigt Änderungen ohne sie anzuwenden (`make <ziel>-check` bei den Playbooks mit Dry-Run). Die Rollenübersicht mit Make-Zielen: [Rollen](index.md#fehlersuche-je-rolle).

```bash
sudo systemctl status xibosignage-xorg xibosignage-kiosk xibosignage-sync.timer
sudo journalctl -u xibosignage-sync -u xibosignage-kiosk -n 100 --no-pager
/usr/local/bin/xibosignage-diagnose.sh
mount | grep xibosignage
```

| Symptom | Ursache | Lösung |
|---|---|---|
| Playbook bricht ab: `xibo_kiosk_start_mode muss systemd oder getty sein` | ungültiger Wert | `systemd` oder `getty` setzen |
| Playbook bricht ab: `xibo_kiosk_edid_force ist aktiv, aber …connector/…base64 ist leer` | `xibo_kiosk_edid_force: true` ohne die beiden Pflichtwerte | [`scripts/generate-edid.py`](https://github.com/pkr-lab/capulus-ansible/blob/main/scripts/generate-edid.py) ausführen, Connector-Namen aus `/sys/class/drm/` übernehmen |
| Schwarzer Bildschirm | X11 (`xibosignage-xorg`) oder Chromium läuft nicht | `systemctl status xibosignage-xorg xibosignage-kiosk`, Diagnose-Skript |
| `xrandr: Configure crtc 0 failed` / Bild bleibt im Notfallmodus (z. B. 1024×768) | Bildschirm liefert kein EDID (`cat /sys/class/drm/card*-HDMI-A-*/edid \| wc -c` → `0`), vc4-KMS verweigert nicht-validierte Modi | `xibo_kiosk_edid_force: true` setzen, siehe [50030](../../capulus-core/5-incidents/50030-infotafel-altes-bildformat-und-nas-sync.md#problem-1-falsches-bildformat) |
| Bilder aktualisieren sich nicht | NFS-Mount (`xibo_kiosk_nfs_server`, `…_nfs_path`), Sync-Timer, oder fehlendes Tailscale-Routing zum NAS | `mount`, `journalctl -u xibosignage-sync`, bei reinen Tailscale-Hosts `tailscale status` prüfen ([50030](../../capulus-core/5-incidents/50030-infotafel-altes-bildformat-und-nas-sync.md#problem-2-keine-neuen-bilder-falsches-routing-zum-nas)), [Xibo Signage](../../capulus-core/3-apps-workloads/300e0-xibosignage.md#fehlerbehebung) |
| Slide wird nach wenigen hundert ms übersprungen statt `…slide_duration_ms` zu laufen | Bild wurde aus dem Cache gelöscht, während der Browser noch ein älteres Manifest hielt (Race Condition) | mit `xibo_kiosk_orphan_grace_seconds ≥ 2× …manifest_reload_seconds` sollte das nicht mehr vorkommen; [50030](../../capulus-core/5-incidents/50030-infotafel-altes-bildformat-und-nas-sync.md#problem-3-slides-werden-manchmal-sofort-übersprungen) |
| Bilder fehlerhaft skaliert | Grenzen (`…image_max_width`, `…image_max_height`, JPEG-Qualität) | Variablen prüfen |
| Tailscale-Zugang des Displays | Auth-Key abgelaufen | [tailscale](tailscale.md) |

Allgemeine Fehlersuche: [Host-Troubleshooting](../a-betriebssystem/a0040-troubleshooting.md), [zentrale Fehlersuche](../../capulus-core/2-betrieb-hardware/20070-troubleshooting-zentral.md).

[Alle Rollen](index.md)
