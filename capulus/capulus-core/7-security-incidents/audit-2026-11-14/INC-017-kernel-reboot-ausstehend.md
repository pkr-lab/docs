# INC-017 – Neuer Kernel/libc installiert, aber nicht aktiv (Neustart seit Wochen ausstehend)

| Feld | Wert |
|---|---|
| ID | INC-017 |
| Datum | 2026-11-14 |
| Kategorie | schwachstellen |
| Schweregrad | **Mittel** (Einstufung ohne CVE-Abgleich der Pakete) |
| Betroffen | homeserver (.94), control-plane-1 (.95), prod-vm (.99) |

## Beschreibung und Nachweis
```
$ uname -r   -> 7.0.0-38-generic  (alle drei Server)    Arbeitsplatz: 7.0.0-43-generic
$ ls -l /boot/vmlinuz-*   (homeserver) -> -38: 2026-09-04   -43: 2026-10-23   (Datei seit 22 Tagen vorhanden)
$ ls /var/run/reboot-required  -> vorhanden;  reboot-required.pkgs (homeserver): linux-image-7.0.0-43-generic, linux-base, libc6
$ apt list --upgradable
homeserver:      crowdsec-firewall-bouncer-iptables 0.0.38, tailscale 1.104.7
control-plane-1: base-files, containerd.io, docker-buildx/compose-plugin, motd-news-config, snapd, software-properties-*, distro-info, tailscale 1.104.7
prod-vm:         keine ausstehenden Paketupdates (nur Reboot)
```
Zusätzlich läuft auf dem Homeserver seit 9 Tagen derselbe Kernel (letzter Boot 05.11. 22:27). Neue Kernel-Versionen werden erst nach dem Neustart wirksam, `libc6` betrifft alle laufenden Dienste. Ob die Pakete Sicherheitskorrekturen enthalten, wurde nicht geprüft (USN/Changelog nicht abgerufen).

## Abgleich mit dem Incident-Register (2026-11-14)
Incident **040** (Patchstand) beschreibt den Stand vom 18.10. (damals alles aktuell, Reboots erfolgt). Dies ist ein neuer Stand vom 14.11.: Kernel 7.0.0-43 ist seit dem 23.10. verfügbar, die Server laufen seit dem Neustart vom 05.11./12.11. weiter mit -38. 040 nennt als offen den fehlenden Reboot-Alarm (`node_reboot_required` im `node_exporter`); das würde dieses Problem sichtbar machen.

## Risiko
Bekannte Kernel- oder libc-Schwachstellen bleiben ausnutzbar, obwohl der Patch installiert ist. Besonders relevant für Systeme mit Container-Workloads (Kernel-Escape) und Netzwerkdiensten.

## Empfohlener Fix
1. Wartungsfenster wählen: Der Homeserver ist Control-Plane, Kubernetes-Knoten, KVM-Host (prod-vm läuft darin) und Druckserver. Reihenfolge: zuerst `control-plane-1` und `prod-vm`, zuletzt den Homeserver (die prod-vm läuft auf ihm).
2. Pro Knoten: Skript ausführen (installiert nur Updates, **kein** Reboot): `ssh <host> 'bash -s -- --dry-run' < scripts/incident-fixes/INC-017-updates-ohne-reboot.sh`
3. Knoten sauber leeren: `kubectl drain <node> --ignore-daemonsets --delete-emptydir-data`, dann `sudo reboot`, danach `kubectl uncordon <node>`. `cluster-power-manager` und `power-agent` laufen auf dem Homeserver: Auswirkung auf das Wake-/Shutdown-Verhalten vorher klären.
4. Nach dem Reboot `uname -r` prüfen (muss `7.0.0-43-generic` zeigen; sonst GRUB-Standardeintrag prüfen: Boot am 05.11. erfolgte trotz installiertem -43 mit -38).

## Fix-Skript
[`scripts/incident-fixes/INC-017-updates-ohne-reboot.sh`](https://github.com/pkr-lab/capulus-core/blob/main/scripts/incident-scripts/incident-fixes/INC-017-updates-ohne-reboot.sh) – installiert Paket-Updates, sichert die Versionsliste unter `/var/backups/incident-INC-017-*`. Der Neustart bleibt bewusst manuell.

## Rollback
`sudo apt-get install <paket>=<alte Version>` (Liste in `versions-before.txt`); beim Kernel im GRUB-Menü den alten Eintrag wählen (`-38` bleibt installiert).

## Quellen
- https://ubuntu.com/security/notices
- https://docs.k3s.io/upgrades/manual
