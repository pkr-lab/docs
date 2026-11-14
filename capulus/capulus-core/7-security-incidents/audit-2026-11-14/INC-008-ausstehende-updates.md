# INC-008 – Ausstehende Paket-Updates (5 Pakete)

| Feld | Wert |
|---|---|
| ID | INC-008 |
| Datum | 2026-11-14 |
| Kategorie | schwachstellen |
| Schweregrad | **Niedrig** |
| Betroffen | snapd, software-properties-common, python3-software-properties, distro-info, python3-distro-info |

## Beschreibung und Nachweis
```
$ apt list --upgradable
snapd 2.78+ubuntu26.04 [aktualisierbar von: 2.77.1+ubuntu26.04.2]
software-properties-common / python3-software-properties 0.120.2 [von 0.120.1]
distro-info / python3-distro-info 1.15ubuntu0.1 [von 1.15]
$ ls /var/run/reboot-required  -> nicht vorhanden
$ systemctl is-enabled unattended-upgrades -> enabled
$ uname -r -> 7.0.0-43-generic     Ubuntu 26.04.1 LTS
```
Die Pakete stammen aus `resolute-updates`. Ob darin Sicherheitskorrekturen stecken, wurde nicht geprüft (Changelog/USN nicht abgerufen). Automatische Updates sind aktiv; `snapd` wird häufig nicht automatisch aktualisiert. CVE-Abgleich der installierten Pakete wäre mit einem Scanner möglich (siehe INC-012).

## Risiko
snapd läuft mit root-Rechten und installiert SUID-Sandboxen. Verzögerte Updates verlängern das Zeitfenster für bekannte Lücken.

## Empfohlener Fix
`sudo apt-get install --only-upgrade distro-info python3-distro-info python3-software-properties snapd software-properties-common`, danach Neustart nur nötig, wenn `/var/run/reboot-required` erscheint. Zusätzlich Snaps prüfen: `lsblk` zeigt 46 Loop-Mounts mit jeweils zwei Revisionen (u. a. core18, gnome-3-28-1804); `core18` und `gnome-3-28-1804` sind alte Laufzeiten, deren Weiterverwendung (Support-Ende bzw. nur ESM) geprüft werden sollte.

## Fix-Skript
[`scripts/incident-fixes/INC-008-updates-einspielen.sh`](https://github.com/pkr-lab/capulus-core/blob/main/scripts/incident-scripts/incident-fixes/INC-008-updates-einspielen.sh) – nur die fünf genannten Pakete, Versionsliste als Backup unter `~/.incident-backups/INC-008-*/versions.txt`.

## Rollback
`sudo apt-get install <paket>=<alte Version>` (Versionen in `versions.txt`).

## Quellen
- https://ubuntu.com/security/notices
- https://ubuntu.com/security/cves
