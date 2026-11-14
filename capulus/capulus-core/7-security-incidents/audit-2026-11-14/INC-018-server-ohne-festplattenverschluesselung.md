# INC-018 – Server ohne Plattenverschlüsselung; Secure Boot nur teilweise aktiv

| Feld | Wert |
|---|---|
| ID | INC-018 |
| Datum | 2026-11-14 |
| Kategorie | hardware |
| Schweregrad | **Mittel** |
| Betroffen | homeserver (HP ProBook 450 G9, Laptop), control-plane-1 (Lenovo ThinkCentre M720q), prod-vm (KVM-Gast) |

## Beschreibung und Nachweis
```
$ lsblk -f | grep -ci crypto_LUKS     -> 0 auf allen drei Servern
homeserver:      mokutil --sb-state -> SecureBoot disabled   (UEFI, TPM /sys/class/tpm/tpm0 vorhanden)
control-plane-1: mokutil --sb-state -> SecureBoot enabled    (UEFI, TPM vorhanden)
prod-vm:         Legacy-BIOS (SeaBIOS), "EFI variables are not supported" -> kein Secure Boot/TPM (VM)
homeserver:      LVM (ubuntu--vg-ubuntu--lv, 928,5 GB ext4) ohne LUKS
```
Der Homeserver ist ein **Laptop** (Chassis „laptop“), der Cluster-Datenspeicher (`local-volume`-PVCs unter `/var/lib/kubelet/pods/…`, u. a. Postgres-, Grafana- und Zammad-Pods in den Mount-Pfaden sichtbar) liegt unverschlüsselt auf dieser Platte. Laut Repo `capulus-core` gibt es im Cluster auch Vaultwarden, Nextcloud, Immich und Mattermost; auf welchem Knoten sie laufen, wurde nicht geprüft.

## Risiko
Bei Diebstahl/Einbruch lässt sich die gesamte Platte (Passwortmanager, Tickets, Kubernetes-Secrets, SSH-Schlüssel) ohne Anmeldung auslesen. Secure Boot aus (homeserver) erlaubt manipulierte Bootloader/Kernel.

## Empfohlener Fix (manuell)
1. **Homeserver:** BIOS-Passwort setzen, Secure Boot aktivieren (HP: F10-Setup, *Advanced → Secure Boot Configuration*). Vorher prüfen, dass `libvirt`/`wireguard`/DKMS-Module signiert sind.
2. **Verschlüsselung:** Neuaufbau mit LUKS2 auf LVM (Wartungsfenster, Datensicherung getestet). Bei Server-Betrieb ist ein automatischer Unlock nötig, z. B. TPM2 (`systemd-cryptenroll --tpm2-device=auto --tpm2-pcrs=7`) oder Clevis/Tang (Netzwerk-gebunden); ein manueller Passwort-Prompt bricht den automatischen Neustart.
3. **VM prod-vm:** Verschlüsselung am Host (LUKS) bzw. VM-Disk-Ebene lösen; für Secure Boot VM mit OVMF (UEFI) neu definieren.
4. Physische Sicherung des Laptops (verschlossener Raum), Gehäuse-/Kensington-Lock.

## Fix-Skript
Keines (BIOS-Einstellungen und Neuaufbau mit LUKS sind nicht sicher automatisierbar). Manuelle Schritte siehe oben.

## Rollback
Backup einspielen; Secure Boot im UEFI wieder auf „Disabled“.

## Quellen
- https://www.freedesktop.org/software/systemd/man/systemd-cryptenroll.html
- https://github.com/latchset/clevis
- https://support.hp.com/de-de/product/setup-user-guides/hp-probook-450-15.6-inch-g9-notebook-pc/
