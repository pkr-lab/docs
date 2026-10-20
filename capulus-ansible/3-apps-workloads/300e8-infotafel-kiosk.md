# Infotafel-Kiosk (Rolle `xibo_kiosk`)

Raspberry Pi 3 B+ mit Chromium im Kiosk-Modus, der eine Bilder-Slideshow zeigt. Die Bilder legt der n8n-Workflow "xibosignage: Playlist -> Display" auf dem NAS ab (`192.168.178.97:/volume1/k8s-storage/xibosignage-display`). Die Rolle liegt unter [roles/xibo_kiosk](https://github.com/pkr-lab/capulus-ansible/tree/main/roles/xibo_kiosk), das Playbook ist [xibo-kiosks.yml](https://github.com/pkr-lab/capulus-ansible/blob/main/xibo-kiosks.yml), der Rollout läuft mit `make xibo-kiosks` (Dry-Run: `make xibo-kiosks-check`). Die Einbettung (CMS, n8n-Playlist-Sync, NAS-Ordner) steht in [300e0](../../capulus-core/3-apps-workloads/300e0-xibosignage.md), die Rollenreferenz in [rollen/xibo_kiosk.md](../rollen/xibo_kiosk.md), die Begründungen in [60020](../6-hintergruende/60020-ansible-rollen.md#xibo_kiosk-raspberry-pi-3-b-bilder-slideshow).

Ziel der Rolle: Nach dem Booten erscheint so schnell wie möglich ein Bild, und zwar ohne dass Netzwerk, NAS oder ein lokaler Webserver erreichbar sein müssen. Neue Bilder kommen danach im Hintergrund.

---

## Umgebung

| Frage | Antwort |
|---|---|
| Wie startet Chromium? | Aus einer eigenen X11-Sitzung: getty@tty1 (Autologin) → `~/.profile` → `startx` → `/usr/local/bin/xibosignage-kiosk-start.sh`. |
| X11 oder Wayland? | X11 (Xorg, dazu openbox als minimaler Window-Manager). Vorher lief Chromium über `xibosignage-kiosk.service` in der Desktop-Sitzung von Raspberry Pi OS (lightdm). Auf Bookworm ist das labwc oder wayfire, also Wayland. Das Skript setzte aber `DISPLAY=:0` und `xset` voraus, die Umgebung war damit nicht eindeutig. Mit der eigenen X11-Sitzung steht sie fest. |
| Woher kommen die Bilder? | NAS-Share per NFS (read-only, `/mnt/xibosignage-display`). Ein Sync-Dienst kopiert sie in einen lokalen Cache, Chromium liest nur den Cache. |
| Wie werden sie angezeigt? | `file:///var/www/xibosignage-slideshow/index.html`, Crossfade-Slideshow in `slideshow.js`. Es gibt keinen Webserver mehr. |

---

## Startablauf vom Boot bis zum ersten Bild

```
Kernel / systemd
  └─ multi-user.target (lightdm maskiert, kein graphical.target mehr nötig)
       └─ getty@tty1  --autologin <xibo_kiosk_user>
            └─ ~/.profile: exec startx /usr/local/bin/xibosignage-kiosk-start.sh -- :0 vt1 -nolisten tcp -s 0 -dpms
                 └─ Xorg (Bildschirmschoner und DPMS aus)
                      └─ xibosignage-kiosk-start.sh
                           ├─ openbox &
                           ├─ unclutter -idle 3 &
                           └─ Schleife: chromium --kiosk file:///var/www/xibosignage-slideshow/index.html
                                └─ manifest.js (Bildliste) → erstes Bild preload + decode → Anzeige
```

Parallel dazu, ohne dass Chromium darauf wartet:

```
timers.target
  └─ xibosignage-sync.timer (3 s nach Boot, danach alle 30 s)
       └─ xibosignage-sync.service (After=network-online.target, Nice 19, idle-IO)
            └─ mount NFS → neue Bilder in den Cache → manifest.js aktualisieren
```

Was gegenüber vorher entfällt:

- `network-online.target`: Der Kiosk hat keine Abhängigkeit mehr auf das Netzwerk.
- Die lightdm-Desktop-Sitzung (Panel, Dateimanager, Compositor usw.).
- Der Python-Webserver samt `curl`-Warteschleife (`sleep 1` pro Versuch).
- Der NFS-Mount beim Boot (`_netdev` zog `network-online.target` nach sich). Der Eintrag ist jetzt `noauto`, den Mount erledigt der Sync-Dienst.
- Der Warte-Start der ersten Rotation: Das erste Bild erscheint sofort, nicht erst nach 10 Sekunden.

Zeitstempel für die Messung schreibt das Start-Skript ins Journal (Tag `xibosignage-kiosk`, Sekunden seit Kernelstart): `x11 bereit` und `starte chromium`.

---

## Mauszeiger

Zwei unabhängige Ebenen, damit der Zeiger auch verschwindet, wenn eine versagt. Beide nutzen `xibo_kiosk_cursor_hide_seconds` (Default 3).

1. **Systemweit (X11):** `unclutter -idle 3` startet im Kiosk-Start-Skript. Nach 3 Sekunden ohne Mausbewegung wird der Zeiger ausgeblendet, bei Bewegung erscheint er wieder. Das gilt auch außerhalb des Chromium-Fensters.
2. **In der Seite:** `slideshow.css` setzt `cursor: none`. Bei `mousemove` oder `mousedown` setzt `slideshow.js` die Klasse `cursor-visible` auf `<html>` und entfernt sie nach `CURSOR_HIDE_MS` (aus `config.js`) wieder.

Unter Wayland würde `unclutter` nicht greifen. Das ist mit dem dedizierten X11 kein Thema mehr, die Seitenlösung deckt es aber ohnehin ab.

---

## Cache- und Nachladelogik

Verzeichnisse auf dem Pi:

| Pfad | Inhalt |
|---|---|
| `/var/www/xibosignage-slideshow/index.html`, `slideshow.css`, `slideshow.js`, `config.js` | Seite (aus der Rolle) |
| `/var/www/xibosignage-slideshow/default.svg` | Mitgeliefertes Standardbild, erscheint, solange der Cache leer ist |
| `/var/www/xibosignage-slideshow/media/` | Lokaler Bilder-Cache (persistent auf der SD-Karte) |
| `/var/www/xibosignage-slideshow/manifest.js` | Bildliste als `window.XIBO_MANIFEST=[{"src":"media/…","v":<mtime>}]` |

Ablauf in `/usr/local/bin/xibosignage-sync.py`:

1. NFS mounten (Timeout 30 s). Klappt das nicht, bleibt der Cache unverändert, das Skript beendet sich mit Fehlercode und der Timer versucht es nach 30 s erneut.
2. Quelle auflisten (`jpg`, `jpeg`, `png`, `gif`, `webp`, keine Dateien mit führendem Punkt).
3. Pro Datei: überspringen, wenn die mtime im Cache gleich ist. Dateien, die jünger als `xibo_kiosk_sync_settle_seconds` (5 s) sind, werden im nächsten Lauf geholt, weil der n8n-Workflow sie evtl. noch schreibt.
4. Übertragung atomar: Die Datei wird nach `media/.partial/` geschrieben, per `fsync` gesichert, mit der mtime der Quelle versehen und dann mit `os.replace` an ihren Platz verschoben. Hat sich die Quelle währenddessen geändert, wird das Ergebnis verworfen. Halbe Dateien sind so nie sichtbar, auch nicht nach einem Stromausfall.
5. Skalierung: Ist ein Bild (jpg, png, webp) größer als `xibo_kiosk_image_max_width` × `xibo_kiosk_image_max_height` (1920×1080), wird es mit ImageMagick verkleinert (`nice 19`, `-strip`, Qualität 85, Speicherlimit 256 MiB). Kleinere Bilder und GIFs werden unverändert kopiert. Schlägt die Skalierung fehl, wird das Original kopiert. WebP-Konvertierung gibt es bewusst nicht: Das Encoden belastet den Pi 3 B+ stärker, als es nützt, und der n8n-Workflow liefert bereits auf 1920×1080 skalierte Bilder.
6. Aus dem Cache verschwinden nur Dateien, die in der Quelle nicht mehr vorkommen. Ist die Quelle leer, bleibt der Cache stehen (Schutz vor einem leeren Bildschirm bei einem Fehler im Workflow). Wer wirklich alle Bilder entfernen will, löscht `/var/www/xibosignage-slideshow/media/*` auf dem Pi.
7. `manifest.js` wird nur neu geschrieben, wenn sich der Inhalt ändert, ebenfalls atomar (`.tmp` + `os.replace`).

Ablauf in der Seite (`slideshow.js`):

- `manifest.js` wird synchron im `<head>` geladen. Das erste Bild bekommt einen `<link rel="preload" as="image" fetchpriority="high">`, wird mit `loading="eager"` geladen und per `decode()` fertig dekodiert, bevor die Überblendung startet. Es gibt kein Lazy Loading.
- Ist die Liste leer, zeigt die Seite `default.svg`.
- Alle `xibo_kiosk_manifest_reload_seconds` (10 s) lädt die Seite `manifest.js` neu (per `<script>`, weil `fetch` auf `file://` blockiert ist). Neue Bilder werden eingereiht, die Rotation springt nicht zurück. Kommt die Liste leer zurück, während schon Bilder laufen, wird sie ignoriert.
- Die URL eines Bildes enthält `?v=<mtime>` statt eines Zeitstempels bei jedem Aufruf. Der Browser-Cache wird dadurch genutzt und erkennt trotzdem geänderte Dateien.
- Ein Bild, das nicht lädt, wird übersprungen. Schlagen alle fehl, erscheint `default.svg`.

Fehlerfälle:

| Situation | Verhalten |
|---|---|
| Pi bootet ohne Netzwerk | Chromium startet normal, zeigt den Cache. Der Sync wartet auf `network-online.target` und läuft danach. |
| NAS nicht erreichbar | Mount scheitert nach höchstens 30 s, alte Bilder bleiben, neuer Versuch nach 30 s. |
| Erster Start, kein Cache | `default.svg`, danach die ersten Bilder, sobald der Sync sie geholt hat. |
| Stromausfall während des Sync | Es gibt nur vollständige Dateien im Cache, `.partial/` wird beim nächsten Lauf geleert. |
| Stromausfall allgemein | Das Chromium-Profil liegt unter `/dev/shm` und wird bei jedem Start neu angelegt. "Chromium wurde nicht korrekt beendet" kann nicht erscheinen. |
| Chromium stürzt ab | Die Schleife im Start-Skript startet ihn nach 1 s neu. |

---

## Geänderte und neue Systemdateien auf dem Pi

Alle werden von der Rolle `xibo_kiosk` ausgerollt (`make xibo-kiosks`), Quellen in [roles/xibo_kiosk/](https://github.com/pkr-lab/capulus-ansible/tree/main/roles/xibo_kiosk).

| Pfad auf dem Pi | Quelle in der Rolle | Änderung |
|---|---|---|
| `/usr/local/bin/xibosignage-kiosk-start.sh` | `templates/xibosignage-kiosk-start.sh.j2` | neu geschrieben (X-Client mit openbox, unclutter, Chromium-Schleife) |
| `/usr/local/bin/xibosignage-sync.py` | `files/xibosignage-sync.py` | neu |
| `/usr/local/bin/xibosignage-diagnose.sh` | `files/xibosignage-diagnose.sh` | neu |
| `/etc/systemd/system/xibosignage-sync.service` | `templates/xibosignage-sync.service.j2` | neu |
| `/etc/systemd/system/xibosignage-sync.timer` | `templates/xibosignage-sync.timer.j2` | neu |
| `/etc/systemd/system/getty@tty1.service.d/autologin.conf` | `templates/getty-autologin.conf.j2` | neu |
| `/etc/systemd/system/default.target` | Symlink auf `/lib/systemd/system/multi-user.target` | neu (statt graphical.target) |
| `/home/<xibo_kiosk_user>/.profile` | Block `ANSIBLE MANAGED BLOCK (xibo_kiosk role)` | ergänzt (`startx` auf tty1) |
| `/etc/fstab` | Eintrag für `/mnt/xibosignage-display` | Optionen jetzt `ro,soft,timeo=30,retrans=2,noauto,nofail,_netdev` |
| `lightdm.service` | – | gestoppt, deaktiviert, maskiert |
| `ModemManager`, `triggerhappy`, `cups`, `cups-browsed` | `xibo_kiosk_disable_services` | gestoppt, deaktiviert |
| `/var/www/xibosignage-slideshow/*` | `files/`, `templates/xibosignage-config.js.j2` | Seite, `default.svg`, `config.js`, `manifest.js` |

Entfernt werden die alten Units und Skripte: `xibosignage-kiosk.service`, `xibosignage-webserver.service`, `xibosignage-manifest.service`, `xibosignage-manifest.timer`, `/usr/local/bin/xibosignage-manifest.sh`, `manifest.json` und der Symlink `media` auf den NFS-Mount.

Zusätzlich installierte Pakete: `xserver-xorg`, `xinit`, `openbox`, `imagemagick`.

Variablen (Defaults in [roles/xibo_kiosk/defaults/main.yml](https://github.com/pkr-lab/capulus-ansible/blob/main/roles/xibo_kiosk/defaults/main.yml)):

| Variable | Default | Wirkung |
|---|---|---|
| `xibo_kiosk_sync_interval_seconds` | `30` | Abstand der Sync-Läufe |
| `xibo_kiosk_manifest_reload_seconds` | `10` | wie oft die Seite die Bildliste neu liest |
| `xibo_kiosk_slide_duration_ms` | `10000` | Anzeigedauer pro Bild |
| `xibo_kiosk_image_max_width` / `_height` | `1920` / `1080` | Zielauflösung für die Skalierung, an den Bildschirm anpassen |
| `xibo_kiosk_cursor_hide_seconds` | `3` | Idle-Zeit bis der Zeiger verschwindet |
| `xibo_kiosk_chromium_extra_flags` | `[]` | zusätzliche Chromium-Flags, z. B. für die GPU |
| `xibo_kiosk_disable_services` | siehe Defaults | Dienste, die die Rolle deaktiviert |

---

## Messen und prüfen

Nach dem Rollout und einem Neustart auf dem Pi:

```bash
sudo /usr/local/bin/xibosignage-diagnose.sh
```

Das Skript gibt `systemd-analyze`, `critical-chain`, `blame`, die Kiosk-Zeitstempel, Sync-Status, Cache-Inhalt, Session-Typ und GPU-Hinweise aus. Für einen Vorher-Nachher-Vergleich das Skript vor dem Rollout einmal ausführen (die Rolle legt es erst beim Rollout ab, vorher genügt `systemd-analyze` und `systemd-analyze critical-chain graphical.target`).

Hardwarebeschleunigung: Die Rolle ändert hier nichts, weil die richtigen Flags vom Gerät abhängen. Prüfen: In `chrome://gpu` müssen "Compositing" und "Rasterization" auf "Hardware accelerated" stehen, und `/dev/dri/card0` sowie `dtoverlay=vc4-kms-v3d` in `/boot/firmware/config.txt` (bei älteren Images `/boot/config.txt`) müssen vorhanden sein. Falls Chromium im Software-Modus läuft, in `xibo_kiosk_chromium_extra_flags` zuerst `--ignore-gpu-blocklist` und `--enable-gpu-rasterization` probieren und erneut prüfen. Das ist nicht vorab gesetzt, weil sich falsche GPU-Flags auf dem Pi mit schwarzem Bildschirm äußern können.

---

## Weitere Beschleunigungen (nicht umgesetzt)

Diese Punkte sind nicht eindeutig unkritisch oder berühren Boot-Dateien und bleiben deshalb dir überlassen:

| Kandidat | Wirkung | Warum nicht automatisch |
|---|---|---|
| `bluetooth.service`, `hciuart.service` | spart Startzeit und RAM | wäre nur sicher, wenn keine Bluetooth-Peripherie genutzt wird |
| `avahi-daemon.service` | spart einen Dienst | könnte `infotafel.local` für den Zugriff nutzen |
| `apt-daily.timer`, `apt-daily-upgrade.timer`, `man-db.timer` | weniger Last kurz nach dem Boot | betrifft automatische Sicherheitsupdates |
| `NetworkManager-wait-online.service` | verkürzt `systemd-analyze` auf `network-online.target` | verzögert den Kiosk nicht (er hängt nicht daran), aber der Sync-Start hängt davon ab; ohne Wait-Online greift nur der 30-s-Retry |
| `/boot/firmware/config.txt`: `disable_splash=1`, `boot_delay=0` | spart die Regenbogen-Splash-Phase | Boot-Datei, Fehler machen den Pi unbootbar |
| `/boot/firmware/cmdline.txt`: `quiet loglevel=3` | weniger Konsolenausgabe beim Boot | Boot-Datei |
| `dtparam=audio=off` | spart ein Kernelmodul | nur sinnvoll, wenn kein HDMI-Ton gebraucht wird |

Mit `systemd-analyze blame` auf dem Pi lässt sich prüfen, welche davon tatsächlich Zeit kosten.

---

## Rückbau

Zurück zur Desktop-Sitzung: die Rolle aus dem Git-Stand vor diesem Branch ausrollen (sie legt `xibosignage-kiosk.service` wieder an) und auf dem Pi von Hand:

```bash
sudo systemctl unmask lightdm
sudo systemctl enable lightdm
sudo rm /etc/systemd/system/default.target
sudo systemctl set-default graphical.target
sudo rm /etc/systemd/system/getty@tty1.service.d/autologin.conf
```

Zusätzlich den Block `ANSIBLE MANAGED BLOCK (xibo_kiosk role)` aus `~/.profile` löschen und neu starten.
