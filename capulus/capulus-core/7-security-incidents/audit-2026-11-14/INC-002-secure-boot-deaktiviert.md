# INC-002 – Secure Boot ist deaktiviert (lokaler Arbeitsplatz)

| Feld | Wert |
|---|---|
| ID | INC-002 |
| Datum | 2026-11-14 |
| Kategorie | hardware |
| Schweregrad | **Mittel** (kein CVSS; Härtungslücke) |
| Betroffen | MSI MAG X870E TOMAHAWK WIFI (MS-7E59), BIOS 2.A94 vom 01.13.2026 |

## Beschreibung und Nachweis
```
$ mokutil --sb-state
SecureBoot disabled
$ cat /sys/class/dmi/id/{board_name,bios_version,bios_date}
MAG X870E TOMAHAWK WIFI (MS-7E59) / 2.A94 / 01/13/2026
$ fwupdmgr get-updates   -> "System Firmware" ohne verfügbares Update (nach LVFS-Stand)
```
Hinweis: fwupd kennt nur, was MSI an LVFS liefert. Ob MSI auf der Herstellerseite eine neuere BIOS-Version anbietet, wurde nicht geprüft (kein Web-Abgleich).

## Risiko
Ohne Secure Boot kann ein Angreifer mit kurzem physischem oder privilegiertem Zugriff Bootloader/Kernel-Image manipulieren (Bootkit). Zusammen mit INC-003 (keine Plattenverschlüsselung) ist ein Offline-Zugriff ohne Hürde möglich.

## Empfohlener Fix (manuell, nicht automatisierbar)
1. BIOS-Passwort setzen (UEFI Setup, Entf beim Start).
2. Unter *Settings → Security → Secure Boot* auf „Enabled“, Modus „Standard“ (Microsoft-Schlüssel, damit shim/GRUB von Ubuntu bootet). Vorher prüfen, dass VMware-/NVIDIA-/DKMS-Module signiert werden: `mokutil --list-enrolled`, bei DKMS-Modulen MOK-Enrollment per `sudo mokutil --import` nötig. VMware-Module (`vmware-vmx` ist installiert) sind betroffen.
3. Nach Reboot `mokutil --sb-state` prüfen, `dmesg | grep -i 'secure boot'`.
4. Herstellerseite für BIOS-Updates: https://www.msi.com/Motherboard/MAG-X870E-TOMAHAWK-WIFI/support

## Fix-Skript
Kein Änderungsskript (BIOS-Einstellung). Lesendes Vorab-Prüfskript: [`scripts/incident-fixes/INC-002-secureboot-preflight.sh`](https://github.com/pkr-lab/capulus-core/blob/main/scripts/incident-scripts/incident-fixes/INC-002-secureboot-preflight.sh) listet Secure-Boot-Status, MOK-Schlüssel, DKMS- und VMware-Module und markiert **unsignierte** Out-of-Tree-Module, die nach dem Einschalten nicht mehr laden würden. Aufruf: `bash scripts/incident-fixes/INC-002-secureboot-preflight.sh`. Läuft auch im Runner `run-local.sh`.

Ergebnis des ersten Laufs (vom Benutzer): Secure Boot disabled, 1 MOK-Schlüssel eingetragen, `dkms` nicht installiert, geladene VMware-Module sind nur die im Kernel enthaltenen (`vmw_vmci`, `vmw_vsock_vmci_transport`, `vmwgfx`). Die erste Fassung des Skripts prüfte nur **geladene** Module; die VMware-Workstation-Module `vmmon`/`vmnet` (VMware ist installiert) werden erst beim Start von VMware gebaut und geladen und wären dort bei aktivem Secure Boot unsigniert. Das Skript wurde um eine Prüfung der installierten Module außerhalb von `kernel/` und der Signierer von `nvidia`/`vmmon`/`vmnet`/`vbox`/`zfs` erweitert; bitte erneut ausführen.

## Rollback
Im UEFI Secure Boot wieder auf „Disabled“ stellen.

## Quellen
- https://wiki.ubuntu.com/UEFI/SecureBoot
- https://uefi.org/specs
