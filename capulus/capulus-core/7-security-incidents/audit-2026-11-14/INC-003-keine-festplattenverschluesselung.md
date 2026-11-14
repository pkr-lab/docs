# INC-003 – Systemplatte unverschlüsselt (lokaler Arbeitsplatz)

| Feld | Wert |
|---|---|
| ID | INC-003 |
| Datum | 2026-11-14 |
| Kategorie | hardware |
| Schweregrad | **Mittel** |
| Betroffen | `/dev/nvme0n1p2` (ext4, 3 TB), Samsung SSD 9100 PRO 4TB |

## Beschreibung und Nachweis
```
$ lsblk -f
nvme0n1
├─nvme0n1p1 vfat  FAT32 E50B-59F4  /boot/efi
└─nvme0n1p2 ext4  1.0   8de17e00-…  3T 12% /
```
Kein `crypto_LUKS`-Eintrag. Auf dem Rechner liegen `~/.ssh`-Schlüssel (u. a. prod_vm_ed25519, netcup-deploy) und die Repos mit Betriebsgeheimnissen (siehe INC-001); `kubectl` ist installiert, ein vorhandenes `~/.kube` wurde nicht geprüft.

## Risiko
Bei Diebstahl, Hardwaredefekt mit Austausch oder Offline-Boot lassen sich alle Daten, inklusive SSH-Keys und Tokens, ohne Anmeldung lesen.

## Empfohlener Fix (manuell)
Nachträgliche Verschlüsselung der Root-Partition im laufenden Betrieb ist riskant. Empfohlen:
1. Vollständiges Backup (siehe INC-009), Wiederherstellung getestet.
2. Neuinstallation mit LUKS2 (Ubuntu-Installer „Festplatte verschlüsseln“, optional TPM-gestützt: `systemd-cryptenroll --tpm2-device=auto`).
3. TPM geprüft (`sudo tpm2_getcap properties-fixed`): **TPM 2.0**, Revision 1.59, Hersteller AMD (Firmware-TPM, fTPM), FIPS-140-2-Modus gemeldet. Damit ist eine TPM-gestützte LUKS-Entsperrung grundsätzlich möglich. Hinweis: fTPM-Firmware hängt am BIOS; BIOS-Updates von MSI beachten (AMD-fTPM-Probleme sind in der Vergangenheit per AGESA-Update behoben worden).
4. Mindestens bis dahin: sensible Daten (`~/.ssh`, Tokens) in verschlüsseltem Container (z. B. `gocryptfs`) ablegen.

## Fix-Skript
Keines (Neuinstallation/Migration ist nicht sicher automatisierbar). Manuelle Schritte siehe oben.

## Rollback
Backup einspielen.

## Quellen
- https://ubuntu.com/core/docs/full-disk-encryption
- https://www.freedesktop.org/software/systemd/man/systemd-cryptenroll.html
