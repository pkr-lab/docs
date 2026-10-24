# Infotafel-Kiosk (Rolle `xibo_kiosk`)

Raspberry Pi 3 B+ mit Chromium im Kiosk-Modus, der eine Bilder-Slideshow zeigt. Die Bilder legt der n8n-Workflow "xibosignage: Playlist -> Display" auf dem NAS ab (`192.168.178.97:/volume1/k8s-storage/xibosignage-display`). Die Rolle liegt unter [roles/xibo_kiosk](https://github.com/pkr-lab/capulus-ansible/tree/main/roles/xibo_kiosk), das Playbook ist [xibo-kiosks.yml](https://github.com/pkr-lab/capulus-ansible/blob/main/xibo-kiosks.yml), der Rollout läuft mit `make xibo-kiosks` (Dry-Run: `make xibo-kiosks-check`). Die Einbettung (CMS, n8n-Playlist-Sync, NAS-Ordner) steht in [300e0](../../capulus-core/3-apps-workloads/300e0-xibosignage.md), die Rollenreferenz in [rollen/xibo_kiosk.md](../rollen/xibo_kiosk.md), die Begründungen in [60020](../6-hintergruende/60020-ansible-rollen.md#xibo_kiosk-raspberry-pi-3-b-bilder-slideshow).

Ziel der Rolle: Nach dem Booten erscheint so schnell wie möglich ein Bild, und zwar ohne dass Netzwerk, NAS oder ein lokaler Webserver erreichbar sein müssen. Neue Bilder kommen danach im Hintergrund.

---

## Umgebung

| Frage | Antwort |
|---|---|
| Wie startet Chromium? | Als eigener Dienst `xibosignage-kiosk.service` auf einem eigenen X-Server (`xibosignage-xorg.service`), beide früh im Boot. Rückfallebene: getty@tty1 (Autologin) → `~/.profile` → `startx`. |
| X11 oder Wayland? | X11 (Xorg, dazu openbox als minimaler Window-Manager). Vorher lief Chromium über `xibosignage-kiosk.service` in der Desktop-Sitzung von Raspberry Pi OS (lightdm). Auf Bookworm ist das labwc oder wayfire, also Wayland. Das Skript setzte aber `DISPLAY=:0` und `xset` voraus, die Umgebung war damit nicht eindeutig. Mit dem eigenen X-Server steht sie fest. |
| Woher kommen die Bilder? | NAS-Share per NFS (read-only, `/mnt/xibosignage-display`). Ein Sync-Dienst kopiert sie in einen lokalen Cache, Chromium liest nur den Cache. |
| Wie werden sie angezeigt? | `file:///var/www/xibosignage-slideshow/index.html`, Crossfade-Slideshow in `slideshow.js`. Es gibt keinen Webserver mehr. |

---

## Startablauf vom Boot bis zum ersten Bild

Der Standard ist `xibo_kiosk_start_mode: systemd`: Der X-Server und Chromium starten als eigene Dienste früh im Boot und warten auf keinen Login.

```
Kernel / systemd
  ├─ lightdm maskiert, Standard-Target multi-user.target
  └─ xibosignage-xorg.service (root, nach local-fs, udev und tmpfiles, ohne DefaultDependencies,
       also nicht hinter basic.target; Plymouth wird beendet)
       └─ Xorg :0 vt7 -nolisten tcp -s 0 -dpms, Cookie in /run/xibosignage/Xauthority
            └─ xibosignage-kiosk.service (User=<xibo_kiosk_user>, Gruppen video und render)
                 └─ xibosignage-kiosk-start.sh
                      ├─ wartet, bis X antwortet (xrandr)
                      ├─ xibosignage-display-mode.sh once (Bildschirmmodus erzwingen)
                      ├─ xibosignage-display-mode.sh par (Seitenverhältnis-Korrektur berechnen)
                      ├─ openbox &
                      ├─ unclutter -idle 3 &
                      └─ Schleife: chromium --kiosk file:///var/www/xibosignage-slideshow/index.html
                           └─ manifest.js (Bildliste) → erstes Bild preload + decode → Anzeige
```

Die Konsole bleibt auf tty1 (normaler Login-Prompt), der X-Server läuft auf vt7. Beide Dienste haben `Restart`, fehlt das DRM-Gerät beim ersten Versuch noch, startet der X-Server nach einer Sekunde erneut.

`xibo_kiosk_start_mode: getty` ist der frühere Weg und bleibt als Rückfallebene:

```
multi-user.target → getty@tty1 --autologin <xibo_kiosk_user>
  └─ ~/.profile: exec startx /usr/local/bin/xibosignage-kiosk-start.sh -- :0 vt1 -nolisten tcp -s 0 -dpms
       └─ derselbe Ablauf ab xibosignage-kiosk-start.sh
```

Warum der Wechsel: Gemessen auf der Infotafel (Modus `getty`) war `multi-user.target` erst nach 28,4 s erreicht, weil `getty@tty1` hinter `plymouth-quit-wait`, `systemd-user-sessions` und `remote-fs.target` hängt. `x11 bereit` kam bei Uptime 42,8 s, `starte chromium` bei 43,0 s. Mit den eigenen Diensten hängt der Kiosk nicht mehr an diesem Login-Pfad. Messwerte dieses Modus stehen unter [Messwerte](#messwerte).

Parallel dazu, ohne dass Chromium darauf wartet:

```
timers.target
  └─ xibosignage-sync.timer (20 s nach Boot, danach alle 30 s)
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

## Bildschirmformat

`/usr/local/bin/xibosignage-display-mode.sh` erzwingt beim Start (`once`) immer den in `xibo_kiosk_display_mode` konfigurierten
Modus (Default `1920x1080`) auf dem ersten verbundenen Ausgang — es sucht nicht mehr nach dem nativen/bevorzugten Modus und
pollt auch nicht mehr im Hintergrund. Fehlt der Modus in der von `xrandr` gelisteten Liste, berechnet das Skript ihn per `cvt`
und fügt ihn per `xrandr --newmode`/`--addmode` hinzu. Die erkannte (erzwungene) Auflösung schreibt es nach
`/dev/shm/xibosignage-screen` (`<Breite> <Höhe>`) und protokolliert sie im Journal (Tag `xibosignage-display`).

Das war nicht immer so: Früher suchte das Skript den vom EDID gemeldeten bevorzugten Modus, fiel auf den ersten
Kernel-Modus zurück und hatte eine Timeout-/Fallback-Logik für den Fall, dass gar kein Bildschirm erkannt wurde. Die
Infotafel liefert aber **dauerhaft kein EDID** (`xrandr` zeigt `0mm x 0mm`, nur die vier VESA-Notfallmodi bis 65 MHz
Pixeltakt) — die ganze Erkennung lief dort also ohnehin nie auf etwas anderes als den konfigurierten Fallback hinaus, siehe
[60020](../6-hintergruende/60020-ansible-rollen.md#xibo_kiosk-raspberry-pi-3-b-bilder-slideshow) und den Incident-Report
[50030](../../capulus-core/5-incidents/50030-infotafel-altes-bildformat-und-nas-sync.md#problem-1-falsches-bildformat).

**Ohne EDID reicht `xrandr --addmode` allein nicht.** Der vc4-KMS-Treiber (`dtoverlay=vc4-kms-v3d`) validiert jede
Modus-Konfiguration gegen die Fähigkeiten, die er aus dem EDID kennt. Ohne EDID nimmt er konservativ nur die vier
VESA-Notfallmodi an und lehnt `1920x1080` mit `Configure crtc 0 failed` ab — selbst wenn der Modus per `cvt`/`--addmode`
in der Liste steht. Dafür gibt es `xibo_kiosk_edid_force` (Default `false`, für die Infotafel aktiv): Eine mit
[`scripts/generate-edid.py`](https://github.com/pkr-lab/capulus-ansible/blob/main/scripts/generate-edid.py) erzeugte,
spezifikationskonforme EDID-Binärdatei (validiert mit `edid-decode -c`) wird nach `/lib/firmware/edid/` deployt und per
Kernel-Parameter `drm.edid_firmware=<xibo_kiosk_edid_connector>:edid/<xibo_kiosk_edid_filename>` in `cmdline.txt`
eingebunden. Der Treiber hält das für ein echtes, 1920×1080@60-fähiges EDID und akzeptiert den Modus. **Nach einer
Änderung dieser Variablen ist ein Reboot nötig** (Kernel-cmdline-Parameter), der Rolle-Handler weist darauf hin.

Zwei naheliegende Alternativen wurden geprüft und verworfen (Details im Incident-Report): der Kernel-Parameter
`video=<connector>:<modus>e` erzwingt nur den Connector-Status, keinen gültigen Modus; die klassischen
Firmware-Parameter `hdmi_group`/`hdmi_mode`/`hdmi_ignore_edid` in `config.txt` werden bei aktivem Full-KMS
(`dtoverlay=vc4-kms-v3d`) gar nicht mehr ausgewertet (`vcgencmd get_config hdmi_mode` meldet `unknown`).

### Seitenverhältnis-Korrektur

Das EDID nennt die physische Größe des Panels (`798mm x 334mm` in der `xrandr`-Ausgabe der Infotafel, das ist 21:9, typisch für einen 34-Zoll-Ultrawide). Der Pi 3 B+ gibt maximal 1920×1080 (16:9) aus, größere Modi sind nicht in der Kernel-Liste. Streckt der Monitor dieses Bild auf die ganze Fläche, sind die Pixel nicht quadratisch und jedes Bild erscheint in die Breite gezogen.

`xibosignage-display-mode.sh par` berechnet daraus den Faktor `k = (Panelbreite / Panelhöhe) / (Pixelbreite / Pixelhöhe)`. Liegt er außerhalb 0,92 bis 1,08 (und zwischen 0,5 und 3), startet Chromium mit `index.html?par=<k>`. `slideshow.js` macht den `#stage` dann `k` mal so breit und staucht ihn per `scaleX(1/k)` auf die Bildschirmbreite zurück. Das Bild wird in einem Kasten mit dem Panelverhältnis eingepasst und wirkt nach dem Strecken des Monitors unverzerrt, mit schwarzen Balken wie bei `contain`. Der Faktor steht im Journal (`bildschirm seitenverhaeltnis-korrektur 1.344`).

`xibo_kiosk_pixel_aspect` steuert das: `auto` (Standard), `off` oder eine feste Zahl. Zeigt der Monitor 16:9-Signale mit Balken statt gestreckt (Einstellung "Seitenverhältnis beibehalten" am Monitor), ist die Korrektur falsch und das Bild erscheint gestaucht: dann `off` setzen. Unsinnige EDID-Größen (unter 100 mm) ignoriert das Skript.

Der Sync liest diese Datei:

- Zielgröße für die Skalierung ist die erkannte Auflösung, begrenzt durch `xibo_kiosk_image_max_width` × `_height`. Ein 4K-Bildschirm führt also nicht zu 4K-Bildern auf dem Pi 3 B+, ein kleiner Bildschirm nicht zu unnötig großen.
- Die zuletzt verwendete Zielgröße steht in `media/.target`. Ändert sie sich (anderer Bildschirm, Modus korrigiert), verarbeitet der Sync alle Bilder aus der Quelle im nächsten Lauf neu. Die Originale auf dem NAS bleiben unangetastet, es gibt keinen Qualitätsverlust durch wiederholtes Skalieren.
- Fehlt die Datei oder ist sie unlesbar, gelten die Variablen als Zielgröße.

Die Seite selbst verzerrt nie: `slideshow.css` nutzt `object-fit: contain`, jedes Bild behält sein Seitenverhältnis und bekommt bei Abweichung schwarze Balken. Das gilt für 16:9, 16:10 und 4:3 gleichermaßen. Auch der n8n-Workflow skaliert nur mit `onlyIfLarger` und ändert das Seitenverhältnis nicht.

Ist das Bild trotzdem verzerrt, ist der Bildschirm in einem falschen Modus: `sudo /usr/local/bin/xibosignage-diagnose.sh` zeigt im Abschnitt "Bildschirm" die `xrandr`-Modi (`*` aktuell, `+` nativ) und die erkannte Auflösung. Bleibt der Modus trotz `xibo_kiosk_display_mode` beim VESA-Notfallmodus (`xrandr --output … --mode …` meldet `Configure crtc 0 failed`), liefert der Bildschirm kein EDID — Abschnitt „Bildschirmformat“, `xibo_kiosk_edid_force`.

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
5. Skalierung: Ist ein Bild (jpg, png, webp) größer als die Zielgröße (erkannte Bildschirmauflösung, höchstens `xibo_kiosk_image_max_width` × `xibo_kiosk_image_max_height`, Default 1920×1080), wird es mit ImageMagick verkleinert (`nice 19`, `-strip`, Qualität 85, Speicherlimit 256 MiB). Kleinere Bilder und GIFs werden unverändert kopiert. Schlägt die Skalierung fehl, wird das Original kopiert. WebP-Konvertierung gibt es bewusst nicht: Das Encoden belastet den Pi 3 B+ stärker, als es nützt, und der n8n-Workflow liefert bereits auf 1920×1080 skalierte Bilder.
6. Dateien, die in der Quelle nicht mehr vorkommen, verschwinden **sofort** aus `manifest.js`, werden aber erst nach `xibo_kiosk_orphan_grace_seconds` (Default 30 s, persistiert in `media/.orphans.json`) physisch gelöscht. Grund: Der Browser hält sein Manifest bis zu `xibo_kiosk_manifest_reload_seconds` im Speicher; würde die Datei sofort verschwinden, könnte eine laufende Rotation noch kurz auf sie zeigen und das Bild mit einem 404 überspringen, siehe [50030](../../capulus-core/5-incidents/50030-infotafel-altes-bildformat-und-nas-sync.md#problem-3-slides-werden-manchmal-sofort-übersprungen). Ist die Quelle leer, bleibt der Cache komplett stehen (Schutz vor einem leeren Bildschirm bei einem Fehler im Workflow). Wer wirklich alle Bilder entfernen will, löscht `/var/www/xibosignage-slideshow/media/*` auf dem Pi.
7. `manifest.js` wird nur neu geschrieben, wenn sich der Inhalt ändert, ebenfalls atomar (`.tmp` + `os.replace`).

Ablauf in der Seite (`slideshow.js`):

- `manifest.js` wird synchron im `<head>` geladen. Das erste Bild bekommt einen `<link rel="preload" as="image" fetchpriority="high">`, wird mit `loading="eager"` geladen und per `decode()` fertig dekodiert, bevor die Überblendung startet. Es gibt kein Lazy Loading.
- Ist die Liste leer, zeigt die Seite `default.svg`.
- Alle `xibo_kiosk_manifest_reload_seconds` (10 s) lädt die Seite `manifest.js` neu (per `<script>`, weil `fetch` auf `file://` blockiert ist). Neue Bilder werden eingereiht, die Rotation springt nicht zurück. Kommt die Liste leer zurück, während schon Bilder laufen, wird sie ignoriert.
- Die URL eines Bildes enthält `?v=<mtime>` statt eines Zeitstempels bei jedem Aufruf. Der Browser-Cache wird dadurch genutzt und erkennt trotzdem geänderte Dateien.
- Ein Bild, das nicht lädt, wird komplett übersprungen (0 s Anzeigedauer, nicht nur verkürzt) und die Rotation springt nach 250 ms zum nächsten. Schlagen alle fehl, erscheint `default.svg`. Dank der Löschungs-Gnadenfrist (oben) sollte das im Normalbetrieb praktisch nie mehr vorkommen.

Fehlerfälle:

| Situation | Verhalten |
|---|---|
| Pi bootet ohne Netzwerk | Chromium startet normal, zeigt den Cache. Der Sync wartet auf `network-online.target` und läuft danach. |
| NAS nicht erreichbar | Mount scheitert nach höchstens 30 s, alte Bilder bleiben, neuer Versuch nach 30 s. Hängt die Infotafel nur noch per Tailscale im Netz (kein eigener LAN-Zugang), braucht sie `tailscale_accept_routes: true`, weil das NAS selbst keinen Tailscale-Knoten hat und nur über die vom `homeserver` annoncierte Subnetzroute erreichbar ist, siehe [50030](../../capulus-core/5-incidents/50030-infotafel-altes-bildformat-und-nas-sync.md#problem-2-keine-neuen-bilder-falsches-routing-zum-nas). |
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
| `/etc/systemd/system/prometheus-node-exporter-apt.service.d/low-priority.conf` | Block in `tasks/main.yml` | neu (`Nice=19`, `CPUWeight=20`, Idle-IO) |
| `/usr/local/bin/xibosignage-display-mode.sh` | `templates/xibosignage-display-mode.sh.j2` | neu |
| `/usr/local/bin/xibosignage-sync.py` | `files/xibosignage-sync.py` | neu |
| `/usr/local/bin/xibosignage-diagnose.sh` | `files/xibosignage-diagnose.sh` | neu |
| `/etc/systemd/system/xibosignage-sync.service` | `templates/xibosignage-sync.service.j2` | neu |
| `/etc/systemd/system/xibosignage-sync.timer` | `templates/xibosignage-sync.timer.j2` | neu |
| `/lib/firmware/edid/<xibo_kiosk_edid_filename>` | `xibo_kiosk_edid_base64` (b64decode) | nur bei `xibo_kiosk_edid_force: true` |
| `/boot/firmware/cmdline.txt` | Task in `tasks/main.yml` (`slurp` + `lineinfile`) | nur bei `xibo_kiosk_edid_force: true`: `drm.edid_firmware=<connector>:edid/<datei>` angehängt |
| `/etc/systemd/system/xibosignage-xorg.service` | `templates/xibosignage-xorg.service.j2` | neu (Modus `systemd`) |
| `/etc/systemd/system/xibosignage-kiosk.service` | `templates/xibosignage-kiosk.service.j2` | neu (Modus `systemd`) |
| `/etc/systemd/system/getty@tty1.service.d/autologin.conf` | `templates/getty-autologin.conf.j2` | nur im Modus `getty`, im Modus `systemd` entfernt |
| `/etc/systemd/system/default.target` | Symlink auf `/lib/systemd/system/multi-user.target` | neu (statt graphical.target) |
| `/home/<xibo_kiosk_user>/.profile` | Block `ANSIBLE MANAGED BLOCK (xibo_kiosk role)` | nur im Modus `getty` (`startx` auf tty1), im Modus `systemd` entfernt |
| `/etc/fstab` | Eintrag für `/mnt/xibosignage-display` | Optionen jetzt `ro,soft,timeo=30,retrans=2,noauto,nofail,_netdev` |
| `lightdm.service` | – | gestoppt, deaktiviert, maskiert |
| `ModemManager`, `triggerhappy`, `cups`, `cups-browsed` | `xibo_kiosk_disable_services` | gestoppt, deaktiviert |
| `/var/www/xibosignage-slideshow/*` | `files/`, `templates/xibosignage-config.js.j2` | Seite, `default.svg`, `config.js`, `manifest.js` |

Entfernt werden die alten Units und Skripte: `xibosignage-kiosk.service`, `xibosignage-webserver.service`, `xibosignage-manifest.service`, `xibosignage-manifest.timer`, `/usr/local/bin/xibosignage-manifest.sh`, `manifest.json` und der Symlink `media` auf den NFS-Mount.

Zusätzlich installierte Pakete: `xserver-xorg`, `xinit`, `openbox`, `imagemagick`, `xcvt` (stellt `cvt` bereit, seit Debian 13 nicht mehr in `x11-xserver-utils` enthalten).

Variablen (Defaults in [roles/xibo_kiosk/defaults/main.yml](https://github.com/pkr-lab/capulus-ansible/blob/main/roles/xibo_kiosk/defaults/main.yml)):

| Variable | Default | Wirkung |
|---|---|---|
| `xibo_kiosk_start_mode` | `systemd` | `systemd` (früher Start per Dienst) oder `getty` (Autologin und `startx`) |
| `xibo_kiosk_supplementary_groups` | `video`, `render` | Gruppen des Kiosk-Dienstes für den GPU-Zugriff ohne Login-Sitzung |
| `xibo_kiosk_sync_interval_seconds` | `30` | Abstand der Sync-Läufe |
| `xibo_kiosk_sync_boot_delay_seconds` | `20` | Verzögerung des ersten Sync-Laufs nach dem Boot, damit er den Start nicht bremst |
| `xibo_kiosk_display_mode` | `1920x1080` | erzwungener Bildschirmmodus (immer aktiv) |
| `xibo_kiosk_edid_force` | `false` | erzwingt eine synthetische EDID per `drm.edid_firmware=`, wenn der Bildschirm keine liefert (Reboot nötig) |
| `xibo_kiosk_edid_connector` / `_filename` / `_base64` | leer / `{{ display_mode }}.bin` / leer | Connector, Dateiname und Inhalt der EDID für `xibo_kiosk_edid_force` ([generate-edid.py](https://github.com/pkr-lab/capulus-ansible/blob/main/scripts/generate-edid.py)) |
| `xibo_kiosk_pixel_aspect` | `auto` | Seitenverhältnis-Korrektur: `auto`, `off` oder fester Faktor |
| `xibo_kiosk_manifest_reload_seconds` | `10` | wie oft die Seite die Bildliste neu liest |
| `xibo_kiosk_orphan_grace_seconds` | `30` | Wartezeit vor dem Löschen einer aus der Quelle verschwundenen Bilddatei (Schutz vor der Race Condition in [50030](../../capulus-core/5-incidents/50030-infotafel-altes-bildformat-und-nas-sync.md)) |
| `xibo_kiosk_slide_duration_ms` | `20000` | Anzeigedauer pro Bild |
| `xibo_kiosk_image_max_width` / `_height` | `1920` / `1080` | Obergrenze für die Skalierung, die Zielgröße selbst kommt vom erkannten Bildschirm |
| `xibo_kiosk_display_poll_seconds` | `5` | Abstand der Bildschirmmodus-Prüfung |
| `xibo_kiosk_cursor_hide_seconds` | `3` | Idle-Zeit bis der Zeiger verschwindet |
| `xibo_kiosk_chromium_extra_flags` | `[]` | zusätzliche Chromium-Flags, z. B. für die GPU |
| `xibo_kiosk_disable_services` | ModemManager, triggerhappy, cups, cups-browsed, e2scrub_reap, upower | Dienste, die die Rolle deaktiviert |

---

## Messen und prüfen

Nach dem Rollout und einem Neustart auf dem Pi:

```bash
sudo /usr/local/bin/xibosignage-diagnose.sh
```

Das Skript gibt `systemd-analyze`, `critical-chain`, `blame`, die Kiosk-Zeitstempel, Sync-Status, Cache-Inhalt, Bildschirmmodus, Session-Typ und GPU-Hinweise aus. Für einen Vorher-Nachher-Vergleich das Skript vor dem Rollout einmal ausführen (die Rolle legt es erst beim Rollout ab, vorher genügt `systemd-analyze` und `systemd-analyze critical-chain graphical.target`).

Hardwarebeschleunigung: Die Rolle ändert hier nichts, weil die richtigen Flags vom Gerät abhängen. Prüfen: In `chrome://gpu` müssen "Compositing" und "Rasterization" auf "Hardware accelerated" stehen, und `/dev/dri/card0` sowie `dtoverlay=vc4-kms-v3d` in `/boot/firmware/config.txt` (bei älteren Images `/boot/config.txt`) müssen vorhanden sein. Falls Chromium im Software-Modus läuft, in `xibo_kiosk_chromium_extra_flags` zuerst `--ignore-gpu-blocklist` und `--enable-gpu-rasterization` probieren und erneut prüfen. Das ist nicht vorab gesetzt, weil sich falsche GPU-Flags auf dem Pi mit schwarzem Bildschirm äußern können.

---

## Messwerte

Infotafel, Modus `getty`, vor der Umstellung auf eigene Dienste (Diagnose vom 20.10.2026):

| Messwert | Wert |
|---|---|
| Kernel | 4,6 s |
| Userspace bis `multi-user.target` | 28,4 s (gesamt 31,3 s) |
| `x11 bereit` / `starte chromium` | Uptime 42,8 s / 43,0 s |
| Engpass | `getty@tty1` hängt hinter `plymouth-quit-wait`, `systemd-user-sessions`, `remote-fs.target` |
| Größte Einzelposten | `prometheus-node-exporter-apt` 17,9 s, `e2scrub_reap` 7,6 s, NetworkManager 6,9 s (wait-online 6,1 s), cloud-init zusammen etwa 8 s |

Modus `systemd`, erste Fassung (Dienste hinter `basic.target`):

| Messwert | Wert |
|---|---|
| Kernel / Userspace | 5,4 s / 33,3 s, `multi-user.target` nach 29,6 s |
| Kiosk-Skript gestartet (Log `x11 bereit`, damals noch vor dem Warten) | Uptime 20,2 s |
| `starte chromium` | Uptime 29,8 s (vorher 43,0 s) |
| Engpass | `Xorg` begann erst bei Uptime 21 s, weil die Dienste mit `DefaultDependencies` hinter `basic.target` hingen, danach brauchte X bis zur Bereitschaft etwa 9 s (CPU-Last durch Boot-Dienste) |

Aktuelle Fassung: Die Dienste haben `DefaultDependencies=no` und hängen nur noch an `local-fs.target`, udev und tmpfiles. Der apt-Exporter des Monitorings (19,7 s im Boot) läuft per Drop-in mit `Nice=19`, `CPUWeight=20` und Idle-IO. Die Zeitstempel im Journal heißen jetzt `kiosk-skript gestartet`, `x11 bereit` und `starte chromium`. Werte nach dem nächsten Neustart mit `sudo /usr/local/bin/xibosignage-diagnose.sh` erheben.

---

## Weitere Beschleunigungen (nicht umgesetzt)

Diese Punkte sind nicht eindeutig unkritisch oder berühren Boot-Dateien und bleiben deshalb dir überlassen:

| Kandidat | Wirkung | Warum nicht automatisch |
|---|---|---|
| `cloud-init` (`sudo touch /etc/cloud/cloud-init.disabled`) | spart nach den Messwerten etwa 8 s Userspace | nur sinnvoll, wenn die Imager-Erstkonfiguration abgeschlossen ist |
| `prometheus-node-exporter-apt.service` | lief im Boot 17,9 s und konkurriert um die CPU | gehört zum Monitoring der Rolle `node_exporter`, besser deren Timer verschieben als abschalten |
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

Zurück zum Start über Autologin und `startx`: `xibo_kiosk_start_mode: getty` setzen und die Rolle ausrollen (entfernt die beiden Dienste, legt Autologin und `.profile`-Block wieder an), danach neu starten.

Zurück zur Desktop-Sitzung: die Rolle aus dem Git-Stand vor diesem Branch ausrollen (sie legt `xibosignage-kiosk.service` wieder an) und auf dem Pi von Hand:

```bash
sudo systemctl unmask lightdm
sudo systemctl enable lightdm
sudo rm /etc/systemd/system/default.target
sudo systemctl set-default graphical.target
sudo rm -f /etc/systemd/system/getty@tty1.service.d/autologin.conf
sudo systemctl disable xibosignage-xorg.service xibosignage-kiosk.service
```

Zusätzlich den Block `ANSIBLE MANAGED BLOCK (xibo_kiosk role)` aus `~/.profile` löschen und neu starten.

---

## Fehlerbehebung

Die Meldungen des Sync-Dienstes stehen im Journal (`journalctl -u xibosignage-sync.service -b`), die des Kiosk-Starts unter dem Tag `xibosignage-kiosk` (`journalctl -t xibosignage-kiosk -b`), die der Bildschirmanpassung unter `xibosignage-display`. Der Rundumschlag ist `sudo /usr/local/bin/xibosignage-diagnose.sh` (siehe „Messen und prüfen“). Allgemeine Rollenhinweise: [Rolle xibo_kiosk](../rollen/xibo_kiosk.md#fehlersuche).

| Meldung oder Symptom | Ursache | Lösung |
|---|---|---|
| Schwarzer Bildschirm nach dem Boot | `xibosignage-xorg` oder `xibosignage-kiosk` läuft nicht (DRM-Gerät noch nicht bereit, Dienst gescheitert) | `systemctl status xibosignage-xorg xibosignage-kiosk`, Dienste starten sich bei Fehlern neu (`Restart`) |
| Bild erscheint erst nach etwa 30 Sekunden | Startmodus `getty` statt `systemd` | `xibo_kiosk_start_mode: systemd` ([Startablauf](#startablauf-vom-boot-bis-zum-ersten-bild)) |
| `mount fehlgeschlagen: …` (Sync-Log) | NFS-Server `192.168.178.97` oder Pfad nicht erreichbar, bei reinem Tailscale-Zugang ggf. fehlendes Subnetz-Routing | `showmount -e 192.168.178.97`, NAS prüfen ([NAS-Speicher](../../capulus-core/2-betrieb-hardware/20000-nas-storage.md#fehlerbehebung)), `tailscale status` ([50030](../../capulus-core/5-incidents/50030-infotafel-altes-bildformat-und-nas-sync.md#problem-2-keine-neuen-bilder-falsches-routing-zum-nas)) |
| `xrandr: Configure crtc 0 failed`, Bild bleibt verzerrt/im Notfallmodus | Bildschirm liefert kein EDID, vc4-KMS verweigert den erzwungenen Modus | `xibo_kiosk_edid_force: true` setzen, Abschnitt „Bildschirmformat“ |
| Ein Bild wird nach wenigen hundert ms übersprungen statt die volle Anzeigedauer zu laufen | Bild wurde aus dem Cache gelöscht, während der Browser noch ein älteres Manifest hielt | sollte mit `xibo_kiosk_orphan_grace_seconds` (Default 30 s) nicht mehr vorkommen, [50030](../../capulus-core/5-incidents/50030-infotafel-altes-bildformat-und-nas-sync.md#problem-3-slides-werden-manchmal-sofort-übersprungen) |
| `Quelle nicht erreichbar, behalte Cache` | NAS offline oder Netzproblem | kein Fehler: das Display zeigt den Cache weiter |
| `Quelle nicht lesbar: …` | Rechte oder Mount-Zustand | `findmnt /mnt/xibosignage-display`, Export prüfen |
| `Quelle ohne Bilder, behalte Cache` | n8n-Workflow hat keine Bilder abgelegt | [Xibo Signage](../../capulus-core/3-apps-workloads/300e0-xibosignage.md#fehlerbehebung), n8n-Workflow „xibosignage: Playlist → Display“ |
| `uebersprungen <datei>: Quelle waehrend des Kopierens veraendert` | Datei wurde beim Kopieren geschrieben | erledigt sich beim nächsten Sync (alle 30 s) |
| `uebersprungen <datei>: Zieldatei leer` | Skalierung lieferte keine Daten | Bild prüfen (Format), `skalieren fehlgeschlagen fuer …` davor im Log |
| `skalieren fehlgeschlagen fuer <datei>: …` | Bildformat nicht lesbar | anderes Format oder neu exportieren |
| `manifest schreiben fehlgeschlagen: …` | `/var/www/xibosignage-slideshow` nicht beschreibbar | Rechte und Speicherplatz (`df -h`) prüfen |
| `marker schreiben fehlgeschlagen: …` / `entfernen fehlgeschlagen <datei>: …` | Dateisystemfehler im Cache | SD-Karte prüfen (Defekt wahrscheinlich bei Wiederholung) |
| `Zielaufloesung <mxn>, Bilder werden neu verarbeitet` | Bildschirm gewechselt, Cache wird neu skaliert | kein Fehler |
| Bild verzerrt oder falsch skaliert | Seitenverhältnis-Korrektur / EDID | Abschnitt „Bildschirmformat“, `xibosignage-diagnose.sh` (Abschnitt Bildschirm) |
| Mauszeiger sichtbar | `unclutter` läuft nicht | Abschnitt „Mauszeiger“ |
| Nach Änderung der Rolle nichts anders | Dienste nicht neu gestartet | `sudo systemctl restart xibosignage-kiosk`, Rolle: `make xibo-kiosks` |

Allgemeine Fehlersuche: [zentrale Fehlersuche](../../capulus-core/2-betrieb-hardware/20070-troubleshooting-zentral.md).
