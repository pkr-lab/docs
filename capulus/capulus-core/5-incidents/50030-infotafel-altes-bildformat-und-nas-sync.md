# Incident-Report — 22.10.2026: Infotafel zeigt altes Bildformat, zieht keine neuen Bilder, überspringt Slides zu schnell

| | |
|---|---|
| **Betroffen** | Infotafel-Kiosk (Raspberry Pi 3 B+, Rolle `xibo_kiosk`): Bildschirmauflösung, NFS-Bilder-Sync, Anzeigedauer der Slideshow |
| **Ausfall** | Seit dem Umzug der Infotafel von direktem LAN-Zugang auf ausschließliche Tailscale-Anbindung (genauer Zeitpunkt nicht protokolliert, vor dem 22.10.2026) bis zur Behebung am 22.10.2026, ca. 19:15–20:10 CEST |
| **Datenverlust** | Keiner. Der lokale Bild-Cache blieb durchgehend erhalten, es wurden nur veraltete Bilder im falschen Format gezeigt |
| **Ursache** | Vier getrennte Probleme, drei davon direkt oder indirekt durch den Netzwerk-Umzug ausgelöst — siehe [Kurzfassung](#kurzfassung) |
| **Status** | Alle vier behoben und auf der Infotafel verifiziert. Commit [`2ba5432`](https://github.com/pkr-lab/capulus-ansible/commit/2ba5432) auf Branch `feat/xibo-kiosk-edid-fix`, noch nicht nach `main` gemergt |

---

## Kurzfassung

Auslöser war die Meldung, der Bildschirm zeige weiterhin ein altes, falsches Format. Die Untersuchung deckte vier voneinander
unabhängige Probleme auf:

1. **Falsches Bildformat (1024×768 statt 1920×1080).** Das Paket `x11-xserver-utils` enthält unter Debian 13 (trixie) das Tool
   `cvt` nicht mehr (ausgelagert nach `xcvt`). Ohne `cvt` konnte `xibosignage-display-mode.sh` nie einen 1920×1080-Modus
   erzeugen. Zusätzlich liefert der angeschlossene Bildschirm **kein EDID** (0 Bytes trotz `connected`), wodurch der
   vc4-KMS-Treiber jeden nicht per EDID validierten Modus mit `Configure crtc 0 failed` ablehnt — selbst nachdem `cvt`
   installiert war.
2. **Keine neuen Bilder.** Nach dem Netzwerk-Umzug ist das NAS (`192.168.178.97`) für die Infotafel nicht mehr direkt erreichbar.
   Der `homeserver` annonciert die alte LAN-Subnetzroute `192.168.178.0/24` per Tailscale (im Tailnet bereits genehmigt), aber
   die Infotafel akzeptierte Subnetzrouten grundsätzlich nicht (`tailscale_accept_routes: false`, Rollen-Default).
3. **Slides werden manchmal sofort übersprungen statt die volle Anzeigedauer zu laufen.** Race Condition zwischen dem Sync-Job
   (löscht ausgetauschte Bilder sofort aus dem Cache) und dem Browser (hält sein Manifest bis zu 10 s im Speicher) — siehe
   [Problem 3](#problem-3-slides-werden-manchmal-sofort-übersprungen).
4. **Anzeigedauer war 10 s statt der gewünschten 20 s** (`xibo_kiosk_slide_duration_ms`, keine fachliche Ursache, nur falsch
   konfiguriert).

Reihenfolge der Bearbeitung in diesem Vorfall: zuerst 1 (meldeter Auslöser), danach 2 und 3 (vom Nutzer während der laufenden
Behebung gemeldet, als er die Sync-Logs beobachtete).

---

## Problem 1: Falsches Bildformat

### Diagnose

```bash
ssh infotafel "export DISPLAY=:0 XAUTHORITY=/run/xibosignage/Xauthority; xrandr --query"
```
```
Screen 0: minimum 320 x 200, current 1024 x 768, maximum 2048 x 2048
HDMI-1 connected primary 1024x768+0+0 … 0mm x 0mm
   1024x768      60.00*
   800x600       60.32    56.25
   848x480       60.00
   640x480       59.94
```

`0mm x 0mm` und ausschließlich die vier VESA-Notfallmodi (alle ≤ 65 MHz Pixeltakt) sind das klassische Zeichen für **kein
EDID**: Der Treiber kennt die wirklichen Fähigkeiten des Bildschirms nicht und lässt nur diese Sicherheits-Modi zu.

```bash
sudo journalctl -t xibosignage-display -b --no-pager
```
```
Modus 1920x1080 auf HDMI-1 nicht verfuegbar
```

```bash
which cvt   # -> bash: cvt: Kommando nicht gefunden
```

`add_mode()` in `xibosignage-display-mode.sh` erzeugt den Custom-Modus per `cvt`/`xrandr --newmode`. Ohne `cvt` blieb
`wanted` immer leer, der Modus wurde nie gesetzt. Nach Installation von `xcvt` (Paket, das `cvt` seit der Debian-13-Aufteilung
bereitstellt) wurde die Modeline zwar erzeugt und zur Liste hinzugefügt, aber:

```bash
xrandr --output HDMI-1 --mode 1920x1080_60.00
# xrandr: Configure crtc 0 failed
```

```bash
cat /sys/class/drm/card0-HDMI-A-1/edid | wc -c   # -> 0
cat /sys/class/drm/card0-HDMI-A-1/status         # -> connected
```

Der vc4-KMS-Treiber (`dtoverlay=vc4-kms-v3d`) validiert ohne EDID nur gegen seine eingebaute Sicherheits-Modusliste (alle
≤ 65 MHz) und verweigert jede Mode-Konfiguration außerhalb davon — unabhängig davon, ob `cvt` verfügbar ist.

### Zwei verworfene Lösungsversuche

Beide erfordern einen Reboot zum Testen und haben **nicht** funktioniert — hier dokumentiert, damit sie nicht erneut versucht
werden:

| Versuch | Befund |
|---|---|
| Kernel-Parameter `video=HDMI-A-1:1920x1080@60e` in `cmdline.txt` | `dmesg` zeigt `forcing HDMI-A-1 connector on`, aber `/sys/class/drm/card0-HDMI-A-1/modes` blieb bei der VESA-Liste, EDID weiterhin 0 Bytes. Der Parameter erzwingt nur den Connector-Status, keinen gültigen Modus. |
| Firmware-Parameter `hdmi_force_hotplug=1`, `hdmi_group=1`, `hdmi_mode=16`, `hdmi_ignore_edid=0xa5000080` in `config.txt` | `vcgencmd get_config hdmi_mode` / `hdmi_group` meldete `unknown` — bei aktivem `dtoverlay=vc4-kms-v3d` (Full-KMS) werden diese klassischen GPU-Firmware-Variablen nicht mehr ausgewertet. |

### Behebung

Eine synthetische, spezifikationskonforme EDID-Binärdatei täuscht dem Treiber ein echtes 1920×1080@60-fähiges Display vor
und wird per Kernel-Parameter `drm.edid_firmware=` eingespielt — das umgeht die Validierung vollständig, weil der Treiber
dann eine (wenn auch gefälschte) EDID hat.

1. `scripts/generate-edid.py` (neu, Repo-Root) erzeugt die 128-Byte-EDID: CEA-861-VIC-16-Timing für 1920×1080@60 (148,5 MHz
   Pixeltakt, aus der CEA-861-Spezifikation übernommen, nicht neu berechnet), `sRGB`-Chromatizität, Checksumme. Validiert mit
   `edid-decode -c` (`EDID conformity: PASS`), bevor sie aufs Gerät kam.
2. Rolle `xibo_kiosk`, neue Tasks in `tasks/main.yml`: Datei nach `/lib/firmware/edid/<xibo_kiosk_edid_filename>` deployen,
   `drm.edid_firmware=<connector>:edid/<datei>` idempotent in `cmdline.txt` eintragen (`slurp` + `lineinfile` mit
   vorausberechnetem Gesamtinhalt der Zeile, siehe [Lehren](#lehren) Punkt 2).
3. Neue Variablen (`defaults/main.yml`): `xibo_kiosk_edid_force` (Default `false`), `xibo_kiosk_edid_connector`,
   `xibo_kiosk_edid_filename`, `xibo_kiosk_edid_base64`. Für die Infotafel aktiviert in `host_vars/infotafel/vars.yml`.
4. `xibosignage-display-mode.sh` erzwingt seitdem `xibo_kiosk_display_mode` ohne Scan nach der tatsächlichen Auflösung (das
   war bereits vor diesem Vorfall so vereinfacht worden); mit der gefälschten EDID akzeptiert der Treiber diesen Modus jetzt.

Nach Reboot bestätigt:

```
EDID-Bytes: 128 (statt 0)
Kernel-Modi: 1920x1080
xrandr: 1920x1080 60.00*+   (current UND preferred)
```

Chromium-Kiosk lief danach stabil ohne neue Fehler weiter.

---

## Problem 2: Keine neuen Bilder (falsches Routing zum NAS)

### Diagnose

```bash
sudo journalctl -u xibosignage-sync -b --no-pager
```
```
mount fehlgeschlagen: Command '['mount', '/mnt/xibosignage-display']' timed out after 30 seconds
Quelle nicht erreichbar, behalte Cache
```

```bash
ping -c 2 -W 2 192.168.178.97   # 100% packet loss, von der Infotafel aus
sudo tailscale status
```

Die Infotafel hängt nur noch per Tailscale (`100.105.105.92`) im Netz, nicht mehr im LAN `192.168.178.0/24`. Das NAS
(`ugreen-nas`, `192.168.178.97`) hat selbst **keinen** Tailscale-Knoten — es ist in der Peer-Liste nicht vorhanden. Der
einzige Weg dorthin ist die vom `homeserver` annoncierte Subnetzroute:

```bash
sudo tailscale status --json | jq '.Peer[] | select(.HostName=="homeserver") | {PrimaryRoutes, AllowedIPs}'
```
```json
{"PrimaryRoutes": ["192.168.178.0/24"], "AllowedIPs": ["100.74.0.59/32", "…", "192.168.178.0/24"]}
```

Die Route ist im Tailnet bereits genehmigt (sie erscheint bei `homeserver`s `AllowedIPs`). Der eigene Status der Infotafel
zeigte aber nur ihre eigene `/32`-Adresse, keine fremde Subnetzroute:

```json
{"AllowedIPs": ["100.105.105.92/32", "fd7a:…/128"]}
```

— das Zeichen für `--accept-routes=false`, den Rollen-Default von `tailscale_accept_routes`. Die Health-Check-Zeile von
`tailscale status` bestätigte es direkt: *„Some peers are advertising routes but --accept-routes is false“*.

### Behebung

```bash
sudo tailscale set --accept-routes=true   # Sofort-Test
```
```bash
ping -c 3 192.168.178.97   # jetzt erfolgreich, 65–185 ms (Routing ueber homeserver)
```

Dauerhaft in `host_vars/infotafel/vars.yml`: `tailscale_accept_routes: true` (die Variable existierte bereits in der Rolle
`tailscale`, war für diesen Host nur nie gesetzt). Über die Rolle ausgerollt (`tailscale set --accept-routes=true`, Task
„Refresh Tailscale options“), danach lief der nächste Sync sofort erfolgreich und zog acht neue Bilder.

---

## Problem 3: Slides werden manchmal sofort übersprungen

### Diagnose

Beim Beobachten der Sync-Logs während der Behebung von Problem 2 fiel auf, dass Dateinamen bei jedem Sync wechseln (z. B.
`xibo-playlist-005-60.png` → `xibo-playlist-005-69.png` — der Suffix ist eine vom Workflow auf dem NAS vergebene
Versionsnummer, keine stabile ID), und dass alte Namen **sofort** nach dem Kopieren der neuen entfernt werden:

```
kopiert: xibo-playlist-005-69.png
…
entfernt: xibo-playlist-005-60.png
manifest aktualisiert: 8 Bilder
```

`slideshow.js` hält sein Bild-Manifest bis zu `xibo_kiosk_manifest_reload_seconds` (Default 10 s) im Speicher, unabhängig
vom Sync-Takt (`xibo_kiosk_sync_interval_seconds`, 30 s). Zeigt die Rotation in diesem Fenster noch auf einen Dateinamen, der
gerade gelöscht wurde, schlägt das `Image`-Laden mit 404 fehl. `present()` in `slideshow.js` behandelt das als Fehler und
ruft `schedule(250)` auf — das betroffene Bild wird komplett übersprungen, die Rotation springt nach 250 ms zum nächsten
Bild, statt die konfigurierte Anzeigedauer abzuwarten.

### Behebung

`xibosignage-sync.py`: Gelöschte Dateien verschwinden weiterhin **sofort** aus `manifest.js` (verhindert, dass ein frisch
geladenes Manifest je auf sie zeigt), aber die physische Datei bleibt zusätzlich `xibo_kiosk_orphan_grace_seconds` (neu,
Default 30 s — bewusst über dem Manifest-Reload-Intervall) liegen, falls ein Browser noch ein älteres Manifest im Speicher
hat. Umgesetzt über eine neue, in `media/.orphans.json` persistierte Zeitstempel-Liste (`load_orphan_state`/
`save_orphan_state`/`remove_orphans`); `write_manifest()` bekommt jetzt explizit die Namen, die gelistet werden sollen, statt
blind alles im Cache-Verzeichnis aufzulisten.

---

## Zusätzliche Änderung: Anzeigedauer

`xibo_kiosk_slide_duration_ms`: `10000` → `20000`. Keine Fehlerursache, nur ein fachlicher Wunsch, der im selben Zug erledigt
wurde, weil `config.js` ohnehin neu ausgerollt werden musste.

---

## Geänderte Dateien

Alle vier Punkte in einem Commit (`2ba5432`, Branch `feat/xibo-kiosk-edid-fix`):

| Datei | Änderung |
|---|---|
| `roles/xibo_kiosk/defaults/main.yml` | `xibo_kiosk_edid_*`-Variablen, `xibo_kiosk_orphan_grace_seconds`, `xibo_kiosk_slide_duration_ms: 20000` |
| `roles/xibo_kiosk/tasks/main.yml` | `xcvt`-Paket, EDID-Datei-Deployment, `drm.edid_firmware=`-Task in `cmdline.txt` |
| `roles/xibo_kiosk/handlers/main.yml` | Hinweis-Handler „Reboot nötig nach EDID-Erzwingung“ |
| `roles/xibo_kiosk/files/xibosignage-sync.py` | Orphan-Grace-Logik (`.orphans.json`), `write_manifest()` nimmt jetzt eine explizite Namensliste |
| `roles/xibo_kiosk/templates/xibosignage-sync.service.j2` | `--orphan-grace-seconds`-Parameter |
| `roles/xibo_kiosk/templates/xibosignage-display-mode.sh.j2` | (bereits vor diesem Vorfall vereinfacht: erzwingt `xibo_kiosk_display_mode` ohne Scan/Fallback-Timeout) |
| `scripts/generate-edid.py` | neu: CEA-861/DMT-EDID-Generator |
| `host_vars/infotafel/vars.yml` | neu: `xibo_kiosk_edid_force: true`, `xibo_kiosk_edid_connector: "HDMI-A-1"`, `xibo_kiosk_edid_base64`, `tailscale_accept_routes: true` |
| `inventory/hosts.yml` | `infotafel`-`ansible_host` von `192.168.178.98` (LAN) auf `100.105.105.92` (Tailscale) — Spiegelbild desselben Netzwerk-Umzugs |

Rollendoku aktualisiert: [xibo_kiosk](../../capulus-ansible/rollen/xibo_kiosk.md),
[300e8 Infotafel-Kiosk](../../capulus-ansible/3-apps-workloads/300e8-infotafel-kiosk.md),
[60020 Begründungen](../../capulus-ansible/6-hintergruende/60020-ansible-rollen.md#xibo_kiosk-raspberry-pi-3-b-bilder-slideshow).

---

## Lehren

1. **EDID-loses HDMI ist auf vc4-KMS (Raspberry Pi OS, Full-KMS) nur noch per `drm.edid_firmware=` lösbar.** Die früher
   üblichen Wege — `video=`-Kernel-Parameter, `hdmi_group`/`hdmi_mode`/`hdmi_ignore_edid` in `config.txt` — greifen bei
   `dtoverlay=vc4-kms-v3d` nicht mehr. Vor einem erneuten Versuch mit einem anderen Gerät/Kabel zuerst `vcgencmd get_config
   hdmi_mode` prüfen: meldet es `unknown`, ist der Firmware-Weg von vornherein aussichtslos.
2. **Ansible-Tasks, die eine Datei mit nur einer Zeile per `replace` + `lineinfile backrefs` ändern, sind nicht idempotent**,
   wenn der `replace`-Schritt den Wert erst entfernt und `lineinfile` ihn danach wieder anfügt — beide Schritte melden dann
   bei jedem Lauf `changed`, selbst wenn der Endzustand gleich bleibt. Robuster: Inhalt mit `slurp` lesen, die gewünschte
   Gesamtzeile in Jinja vorausberechnen, und sie per `lineinfile` mit `regexp: '^.*$'` **vergleichen lassen** statt sie über
   eine Rückreferenz zusammenzusetzen.
3. **Ein NAS ohne eigenen Tailscale-Knoten ist ein unsichtbarer Single Point of Failure für jeden Host, der auf Subnetzrouting
   angewiesen ist.** Die Infotafel hing am `homeserver` als einzigem Pfad zum NAS, ohne dass das irgendwo dokumentiert war —
   sichtbar wurde es erst, als der direkte LAN-Zugang wegfiel. Siehe [Offene Punkte](#offene-punkte).
4. **Browser-seitige Caches (hier: `manifest.js` für bis zu 10 s im Speicher) und serverseitige Löschvorgänge brauchen eine
   gemeinsame Gnadenfrist**, sonst erzeugt jede Content-Rotation eine Race Condition. Die Lösung (Referenz sofort entfernen,
   Datei verzögert löschen) ist allgemein übertragbar auf jeden Cache mit „Löschen nach Ersetzen“-Semantik.
5. **Reboots der Infotafel brauchen deutlich länger als erwartet, bis Tailscale/SSH wieder erreichbar sind** (beobachtet:
   zwischen 2 und über 9 Minuten, einmal kam das Gerät gar nicht von selbst zurück und musste vor Ort per Stromstecker
   neu gestartet werden). Vor einem geplanten Reboot der Infotafel großzügig Zeit einplanen und nicht nach wenigen Minuten
   von einem Boot-Fehler ausgehen.

---

## Offene Punkte

- [ ] `ugreen-nas` hat keinen eigenen Tailscale-Knoten. Fällt der `homeserver` aus oder verliert er seine Subnetzroute, ist
      das NAS von jedem reinen Tailscale-Host (Infotafel, zukünftige Displays) nicht mehr erreichbar. Tailscale direkt auf dem
      NAS einrichten, statt sich auf die Subnetzroute über den `homeserver` zu verlassen.
- [ ] Branch `feat/xibo-kiosk-edid-fix` noch nicht nach `main` gemergt (Stand dieses Reports).
- [ ] Kein automatischer Check, der ein verschwundenes EDID (`/sys/class/drm/.../edid` → 0 Bytes) oder eine fehlgeschlagene
      Subnetzroute proaktiv meldet (z. B. über `node_exporter`/`ntfy_publisher`, die auf der Infotafel bereits laufen).
- [ ] `xibosignage-sync.service` lief während der Diagnose einmal in einen harten NFS-Mount-Hang (`mount.nfs` im
      D-State, nicht per Signal abbrechbar), nachdem der NAS-Pfad kurzzeitig nicht erreichbar war. Das hat sich von selbst
      aufgelöst, aber ein `soft`-Timeout schützt laut dieser Beobachtung nicht zuverlässig vor einem hängenden Mount-Versuch
      selbst (nur vor hängenden I/O-Operationen nach erfolgreichem Mount).
