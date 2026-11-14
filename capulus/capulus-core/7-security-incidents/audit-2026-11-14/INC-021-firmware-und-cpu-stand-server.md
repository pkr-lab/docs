# INC-021 – Firmware-/BIOS-Stand und CPU-Mitigations der Server

| Feld | Wert |
|---|---|
| ID | INC-021 |
| Datum | 2026-11-14 |
| Kategorie | hardware |
| Schweregrad | **Niedrig** (BIOS control-plane-1 von 2019; ohne Abgleich mit Herstellerstand) |
| Betroffen | control-plane-1, homeserver, prod-vm |

## Beschreibung und Nachweis
| Host | Hardware | BIOS/UEFI | CPU / Microcode |
|---|---|---|---|
| homeserver (.94) | HP ProBook 450 G9 (Laptop), 61 GiB RAM | U85 Ver. 01.17.00 vom 06.11.2025 | i5-1235U, 0x43b |
| control-plane-1 (.95) | Lenovo ThinkCentre M720q | **M1UKT45A vom 11.07.2019** | Pentium Gold G5400T, 0xfa |
| prod-vm (.99) | KVM-Gast (QEMU Q35, SeaBIOS 1.17.0) auf dem Homeserver | SeaBIOS (VM) | i5-1235U (durchgereicht), 0x43b |

```
$ fwupdmgr get-updates   (homeserver) -> "System Firmware", "Internal SPI Controller (BIOS)", "HP UEFI Secure Boot DB/KEK": keine Updates; "SSD NM710 1TB": keine Updates
$ fwupdmgr get-updates   (control-plane-1/prod-vm) -> nicht ausgewertet
CPU-Vulnerabilities:
 control-plane-1: meltdown (PTI), l1tf, mds, mmio_stale_data, retbleed (IBRS), srbds, itlb_multihit: mitigiert, "SMT vulnerable" (Hyper-Threading an)
 homeserver/prod-vm: nur reg_file_data_sampling (Clear Register File), spectre_v1/v2, ssb, vmscape mitigiert; übrige "Not affected"
$ SMART (homeserver, über den lokalen smartctl_exporter): PASSED, Percentage Used 2 %, Available Spare 100 %, Media Errors 0, 48 °C
$ journalctl -k | grep -iE 'mce|edac|hardware error' -> nur Initialisierungsmeldung "EDAC MC: Ver 3.0.0" (alle drei)
$ IOMMU: DMAR/IOMMU-Initialisierung im Kernel-Log (Homeserver, control-plane-1) vorhanden; Thunderbolt: Controller am Homeserver (00:0d.0), boltctl nicht ausgewertet
$ prod-vm: openipmi.service failed (VM ohne IPMI, harmlos)
```
Die 7 Jahre alte BIOS-Version auf dem ThinkCentre ist auffällig. Ob Lenovo neuere Versionen anbietet und welche CVEs sie schließen (z. B. Intel-Plattform-Security-Advisories), wurde **nicht** verifiziert. Das gilt auch für die HP-BIOS-Version `01.17.00`. Microcode wird über das Betriebssystem aktualisiert (intel-microcode).

## Risiko
Veraltete Firmware kann Plattform-Lücken (SMM, ME, Secure-Boot-Bypass wie LogoFAIL/PKfail) enthalten, die kein Betriebssystem-Update behebt. Offene SMT-Mitigations sind für Mehrmandanten-Szenarien relevant (VMs/Container auf demselben Kern).

## Empfohlener Fix (manuell)
1. Lenovo: Support-Seite für ThinkCentre M720q, BIOS/UEFI prüfen und aktualisieren (Update über Bootmedium oder `fwupdmgr`, falls LVFS das Gerät listet): https://support.lenovo.com/de/de/products/desktops-and-all-in-ones/thinkcentre-m-series-desktops/m720q
2. HP: https://support.hp.com/de-de/drivers/hp-probook-450-15.6-inch-g9-notebook-pc/ , Akku/Netzteil beim Update angesteckt lassen.
3. Updates **nacheinander** und mit Cluster-Drain; BIOS-Einstellungen (Secure Boot, Boot-Reihenfolge, Wake-on-LAN, VT-x) vorher notieren.
4. Auf Servern mit unvertrauenswürdigen Workloads `nosmt` erwägen (Leistung -20–30 %); hier nur eigene Workloads: akzeptieren oder dokumentieren.
5. `intel-microcode` aktuell halten (`apt list --installed | grep intel-microcode`).

## Fix-Skript
Keines (Firmware-Updates nicht sicher automatisierbar). Manuelle Schritte mit Herstellerlinks siehe oben.

## Rollback
BIOS-Downgrade ist bei beiden Herstellern eingeschränkt: vor dem Update Einstellungen sichern, Wiederherstellung per Hersteller-Rescue-Medium.

## Quellen
- https://fwupd.org/ ; https://www.intel.com/content/www/us/en/security-center/default.html
- https://binarly.io/advisories (LogoFAIL, PKfail)
