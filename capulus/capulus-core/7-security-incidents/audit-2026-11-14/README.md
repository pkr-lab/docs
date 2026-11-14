# Security-Audit capulus – 2026-11-14

Umfang: (1) lokaler Arbeitsplatz `pke-ms-7e59` (Ubuntu 26.04.1, Kernel 7.0.0-43, AMD Ryzen 9 9900X, MSI MAG X870E TOMAHAWK WIFI), (2) die Projekte unter `~/git/github/capulus` (ansible, core, ios, operator, wiki-sync) und (3) die Server **homeserver (.94), control-plane-1 (.95), prod-vm (.99)**, jeweils lesend per SSH ohne `sudo` (Rohdaten liegen lokal beim Betreiber und gehören nicht ins öffentliche Repo). Alle Prüfungen waren **lesend** ohne root. Es wurde nichts geändert und kein Fix-Skript ausgeführt.

**Nicht geprüft:** worker-1 (.96) und entw-vm (.100) sind nicht erreichbar, das NAS (.97) und .98 haben keinen SSH-Zugang, auf den Servern wurde kein `sudo` verwendet – siehe [INC-013](INC-013-remote-hosts-nicht-geprueft.md). Die Adressen 192.157.178.x (öffentlich) wurden nur initial per Login-Versuch berührt (Timeout, nichts erreicht). Ebenfalls nicht geprüft: `rkhunter`, `chkrootkit`, Lynis im Vollumfang (brauchen root; `smartctl`, `tpm2_getcap`, `ufw`, `sshd -T`, `ss` wurden vom Benutzer ausgeführt), CVE-Scans (`osv-scanner`, `trivy`, `govulncheck`, `pip-audit` noch nicht installiert), Hersteller-Webseiten.

## Übersicht der Funde

| ID | Kategorie | Schweregrad | Status | Titel |
|---|---|---|---|---|
| [INC-020](INC-020-schluessel-und-nopasswd-sudo.md) | apt | **Hoch** | Offen, Skript (Opt-in) + manuell | Private SSH-Schlüssel ohne Passphrase, root-äquivalent auf Servern (`NOPASSWD:ALL`) |
| [INC-016](INC-016-etcd-snapshot-kopie-fehlgeschlagen.md) | ransomware | Mittel | Offen, Fix-Skript bereit | etcd-Snapshot-Kopie auf NAS schlägt auf control-plane-1 fehl |
| [INC-017](INC-017-kernel-reboot-ausstehend.md) | schwachstellen | Mittel | Offen, Skript + manueller Reboot | Kernel -43/libc6 installiert, Neustart seit Wochen offen (3 Server) |
| [INC-018](INC-018-server-ohne-festplattenverschluesselung.md) | hardware | Mittel | Offen, manuell | Server ohne LUKS; Secure Boot am Homeserver aus |
| [INC-002](INC-002-secure-boot-deaktiviert.md) | hardware | Mittel | Offen, manuell | Secure Boot deaktiviert (Arbeitsplatz) |
| [INC-003](INC-003-keine-festplattenverschluesselung.md) | hardware | Mittel | Offen, manuell | Systemplatte unverschlüsselt |
| [INC-009](INC-009-keine-lokale-datensicherung.md) | ransomware | Mittel | Offen, manuell | Keine nachweisbare lokale Datensicherung |
| [INC-004](INC-004-ssh-keyfile-rechte.md) | apt | Niedrig | Offen, Fix-Skript bereit | Key-Datei mit Modus 664 |
| [INC-015](INC-015-dev-webserver-offen.md) | apt | Niedrig | Offen, manuell | `http.server` auf 0.0.0.0:8000 liefert Repo samt `.git` |
| [INC-021](INC-021-firmware-und-cpu-stand-server.md) | hardware | Niedrig | Offen, manuell | BIOS control-plane-1 von 2019; Herstellerstand ungeprüft |
| [INC-006](INC-006-sshd-defaults.md) | apt | Niedrig | Offen, Fix-Skript bereit | sshd ohne explizite Härtung |
| [INC-007](INC-007-teamviewer-remote-zugriff.md) | botnet/apt | Niedrig | Rückfrage nötig, Skript bereit | TeamViewer-Dienst aktiv |
| [INC-008](INC-008-ausstehende-updates.md) | schwachstellen | Niedrig | Offen, Fix-Skript bereit | 5 ausstehende Updates, alte Snap-Laufzeiten |
| [INC-010](INC-010-tailscale-verwaiste-geraete.md) | apt | Niedrig | Offen, manuell | Verwaiste Geräte im Tailnet |
| [INC-019](INC-019-server-dienste-und-ports.md) | ddos | Info | Teilweise bewusst (Incident 038), Skript + Ansible | Homeserver: `cups-browsed`/`ModemManager` ohne Bedarf; CUPS/SANE/rpcbind bewusst offen |
| [INC-005](INC-005-rpcbind-exponiert-firewall-unklar.md) | ddos | Info | Entschärft durch UFW, Skript optional | rpcbind/Ollama auf allen Interfaces, UFW sperrt |
| [INC-011](INC-011-docker-gruppe.md) | apt | Info | Zur Kenntnis | Docker-Gruppe = root-äquivalent |
| [INC-012](INC-012-pruefwerkzeuge-fehlen.md) | schwachstellen/hardware | Info | Teilweise erledigt | Prüfwerkzeuge installiert; SMART/TPM/CVE-Scan offen |
| [INC-013](INC-013-remote-hosts-nicht-geprueft.md) | hardware | Info | Teilweise erledigt | Ziel-PCs/VMs: 3 von 7 Adressen geprüft |
| [INC-014](INC-014-lynis-haertungsvorschlaege.md) | apt/schwachstellen | Info | Zur Kenntnis | Lynis: 38 Härtungsvorschläge (Index 66) |

Zählung: Kritisch 0 · Hoch 1 · Mittel 6 · Niedrig 7 · Info 6 (gesamt 20). **INC-001 wurde zurückgezogen** (Fehlalarm, siehe [INC-001](INC-001-klartext-secrets-vault-yml.md)).

## Positive Befunde (kein Handlungsbedarf)
- **CPU-Mitigations:** Meltdown, MDS, L1TF, Retbleed, TAA, SRBDS, GDS, RFDS usw. „Not affected“; Spectre v1/v2/SSB sowie `spec_rstack_overflow` und `vmscape` mitigiert (`/sys/devices/system/cpu/vulnerabilities/*`). Microcode `0xb404035` früh geladen. Ob es eine neuere Revision gibt, ist nicht geprüft.
- **Malware-Scan:** ClamAV (Signaturen 14.11.2026) über `~/git`, `~/.ssh`, Autostart, Downloads: 2.835 Dateien, 0 Treffer. Kein Vollscan von `/`. `rkhunter`/`chkrootkit` noch nicht gelaufen (brauchen root).
- **SSD:** Samsung 9100 PRO 4TB: SMART PASSED, 0 % Verschleiß, 0 Medienfehler, 364 Betriebsstunden; nur 22 Unsafe Shutdowns auffällig ([INC-012](INC-012-pruefwerkzeuge-fehlen.md)).
- **TPM/Firewall:** TPM 2.0 (AMD-fTPM); UFW aktiv mit `deny incoming`, nur SSH (LAN) und Ollama (nur 192.168.178.94) offen; Tailnet-Erreichbarkeit (Tailscale-Regeln vor UFW) noch ungeprüft ([INC-005](INC-005-rpcbind-exponiert-firewall-unklar.md)). Alle lauschenden Ports sind bekannten Prozessen zugeordnet.
- **Rootkit-Scan (`rkhunter`, vom Benutzer ausgeführt):** 8 Warnungen, alle als **harmlos eingeordnet**: `lwp-request` ist ein Perl-Skript des Pakets `libwww-perl` (dpkg bestätigt), Benutzer/Gruppe `clamav` wurden durch die gerade erfolgte Installation angelegt, `PermitRootLogin` ist nicht explizit gesetzt (wirksam `prohibit-password`, siehe INC-006), `/dev/shm/lttng-ust-wait-8-1000` gehört `pke` und wird von `wireplumber` offen gehalten, `/etc/.java`, `/etc/.updated`, `/etc/.resolv.conf.systemd-resolved.bak` sind root-eigene Systemdateien (April/Mai 2026). Kein Rootkit-Fund. **`chkrootkit` wurde angekündigt, aber es kam keine Ausgabe an: nicht bewertet.**
- **Temperaturen:** CPU 47 °C, NVMe 31 °C, RAM 34 °C, GPU 41 °C (`sensors`).
- **Lynis (ohne root):** Hardening-Index 66, 0 Warnungen ([INC-014](INC-014-lynis-haertungsvorschlaege.md)).
- **Hardware-Fehler:** Im aktuellen Boot nur die beiden Initialisierungsmeldungen (`EDAC MC: Ver 3.0.0`, `MCE: In-kernel MCE decoding enabled`); keine Hardware-Error-/MCE-Ereignisse.
- **Firmware (fwupd):** Keine verfügbaren Updates für System-Firmware, SSD 9100 PRO, USB4-Controller, Bolt-Receiver, MX Master 4; UEFI-CA/KEK-Zertifikate aktuell.
- **Persistenz-Spuren:** keine zusätzlichen Benutzer (nur root, pke), kein zweiter UID 0, keine weltschreibbaren Dateien in `/etc`, keine Prozesse aus `/tmp` oder gelöschte Binaries, keine Crontab, nur erwartbare Autostarts (Bitwarden, Clipboard-Skript, sysmonitor), SUID-Liste enthält nur bekannte Pakete (inkl. VMware), kürzlich geänderte Binaries (`terraform`, `cloudflared`, `tailscale`, `artcraft-launcher`, `wordcraft`) gehören alle zu dpkg-Paketen. Für `artcraft-launcher`/`wordcraft` wurde die Paketherkunft (Repository) nicht geprüft.
- **Logging/Angriffsspuren:** 0 fehlgeschlagene SSH-Logins in 7 Tagen; `journalctl --list-boots` ohne Auffälligkeiten (viele kurze Boots am 13.11., ansonsten lückenlos nicht belegbar).
- **Zugang:** SSH-Verzeichnis-/Schlüsselrechte bis auf INC-004 korrekt (600), `authorized_keys` mit 3 beschrifteten Schlüsseln; `kbd-interactive` aus.
- **Server (ohne root geprüft):** SSH-Passwort-Login aus, Root-Login aus, Forwarding aus (`/etc/ssh/sshd_config.d`); CrowdSec + Firewall-Bouncer auf dem Homeserver; Journal-Upload an externen Host; `unattended-upgrades` aktiv; keine zusätzlichen Benutzer, keine Prozesse aus `/tmp` oder gelöschte Binaries, SUID-Listen unauffällig; nur 4 fehlgeschlagene SSH-Logins in 7 Tagen (Homeserver); SSD des Homeservers 2 % verbraucht, 0 Medienfehler (über den lokalen `smartctl_exporter`); CPU-Mitigations aktiv; keine MCE/EDAC-Fehler; Secure Boot auf control-plane-1 aktiv.
- **Container:** Nicht-root-User, Distroless; Secrets im Cluster-Repo überwiegend als SealedSecrets (`encryptedData`), Dockerfiles ohne `ADD`/`curl|sh`.
- **Updates:** `unattended-upgrades` aktiv, nur 5 Pakete ausstehend, kein Neustart nötig.

## Gesamtbewertung: **76 / 100** (Arbeitsplatz und Server gemeinsam; worker-1, entw-vm, NAS ungeprüft)

| Bereich | Ergebnis |
|---|---|
| Schadsoftware/Botnetz/APT | Keine Indizien (sauber), Rest: TeamViewer klären, Tailnet bereinigen (-4) |
| Ransomware | Arbeitsplatz ohne Sicherung (-6); etcd-Kopie auf control-plane-1 seit 12.11. defekt (-4) |
| Secrets | `!vault`-Inline-Werte und SealedSecrets, keine Klartext-Secrets gefunden (0) |
| Hardware/Firmware | Secure Boot aus, keine Plattenverschlüsselung (-6); SSD gesund, TPM 2.0, CPU/Firmware sonst gut |
| Schwachstellen/Updates | Arbeitsplatz gut gepflegt; Server mit seit 22 Tagen nicht aktivem Kernel (-4); CVE-Scan der Abhängigkeiten fehlt (Unsicherheit) |
| Zugangskonzept | Private Schlüssel ohne Passphrase auf unverschlüsselter Platte, root-äquivalent auf Servern (`NOPASSWD:ALL`) (-7) |
| DDoS/Exposition | UFW `deny incoming`, minimale Regeln; Dev-Webserver offen gebunden (-2) |

Die Bewertung ist vorläufig: Der Score kann nach Freigabe von `sudo ufw status`, Scanner-Installation (INC-012) und Prüfung der Ziel-Hosts (INC-013) fallen oder steigen.

## Bezug zum Incident-Register
Das Register in [`7-security-incidents/`](../README.md) kennt mehrere meiner Funde und Betreiber-Entscheidungen. Überschneidungen: SSH-Härtung/`NOPASSWD` ([009](../009-ssh-haertung-nicht-verwaltet.md)), Tailnet-ACL und alter Runner (036 = meine INC-010), Netzwerkdienste (038: CUPS, SANE/`inetd`, `rpcbind` wegen NFS `vers=3`, FRR-`mgmtd` auf `:2623` sind **bewusst** so; meine INC-019 ist dadurch auf Info herabgestuft), Patchstand (040 für den Stand vom 18.10.; meine INC-017 beschreibt den Stand vom 14.11.), Backups (037, die Server betreffend; der Arbeitsplatz fehlt dort, siehe INC-009). Neu gegenüber dem Register sind u. a. INC-016 (NAS-Mount-Race), INC-003/018 (keine LUKS-Verschlüsselung), INC-020 (Schlüssel ohne Passphrase) und die Arbeitsplatz-Funde.

## Umsetzung in Ansible (Branch `feat/sicherheits-haertung` in capulus-ansible)
Die Rollen sind in [capulus-ansible](../../../capulus-ansible/rollen/index.md) dokumentiert, die Begründungen in [60020](../../../capulus-ansible/6-hintergruende/60020-ansible-rollen.md).

| Incident | Ansible | Standard |
|---|---|---|
| INC-016 | `roles/k3s/files/k3s-etcd-snapshot-copy.sh`: versucht vor dem Fehlschlag `mount` (Selbstheilung nach Boot-Race) | ja, mit `make controlplane-1`/`k3s` |
| INC-019 | `common`: `common_disabled_services` (ModemManager); `cups_print_server`: `cups_print_server_disabled_services` (cups-browsed). SANE/CUPS/rpcbind bleiben (Incident 038) | ja |
| INC-006 | `ssh_hardening` jetzt auch im Playbook `ollama-pc.yml`; Rolle legt das Drop-in-Verzeichnis an, falls es fehlt | ja |
| INC-004, 005, 011, 012, 014 | `ollama_pc`: neue `tasks/hardening.yml` (Schlüsselrechte, `rpcbind` nur ohne NFS, leere sudoers-Datei, Prüfwerkzeuge, `auditd`, Modul-Sperrliste) | ja (`ollama_pc_hardening_enabled`) |
| INC-007 | `ollama_pc_disabled_services` (z. B. `teamviewerd.service`) | leer, Entscheidung bei dir |
| INC-009 | neue Rolle `restic_backup` | aus, bis `restic_backup_enabled`, Repository und Vault-Passwort gesetzt sind |
| INC-008, 017 | keine Codeänderung: bestehende Mechanik (`make k3s-servers-update`, `unattended-upgrades`) | – |
| INC-001, 002, 003, 010, 015, 018, 020, 021 | nicht per Ansible (Fehlalarm / BIOS / Neuinstallation / Konsole / Prozess / interaktiv / Firmware) | – |

## Fix-Skripte
Die Skripte liegen im Repo [capulus-core](https://github.com/pkr-lab/capulus-core/tree/main/scripts/incident-fixes) unter `scripts/incident-fixes/`.

Alle Änderungsskripte: `set -euo pipefail`, idempotent, `--dry-run`, Backup vor jeder Änderung (lokal `~/.incident-backups/INC-xxx-*`, auf Servern `/var/backups/incident-INC-xxx-*`), keine Kommentare im Code. Nichts davon wurde ausgeführt, nur Syntax geprüft (`bash -n`) und der Übergabe-Mechanismus der Server-Runner mit einem Wegwerf-Skript getestet.

### Sammelaufruf
```
cd capulus-core/scripts/incident-fixes
bash run-local.sh --dry-run            # lokal: nur Standard-Skripte, Trockenlauf
bash run-local.sh                      # lokal: Standard-Skripte ausführen
bash run-servers.sh --dry-run          # Server per SSH: Trockenlauf
bash run-servers.sh                    # Server per SSH ausführen
```
`--include-optional` nimmt die Opt-in-Skripte hinzu. Der lokale Runner fragt einmal `sudo -v` ab. Er schließt mit einer Zusammenfassung (ok/FEHLER je Skript) und führt am Ende die beiden **lesenden** Prüfskripte INC-002 und INC-020 aus. Die Server-Runner nutzen deine SSH-Aliase `control-plane-0` (Homeserver), `control-plane-1`, `prod-vm`; der Neustart der Server bleibt manuell.

### Zuordnung Incident → Skript
| Incident | Skript | Läuft in | Standard im Runner |
|---|---|---|---|
| INC-001 | – (zurückgezogen, Fehlalarm) | – | – |
| INC-002 | `INC-002-secureboot-preflight.sh` | lokal | nur lesend, immer |
| INC-003 | – (Neuinstallation mit LUKS) | manuell | – |
| INC-004 | `INC-004-ssh-key-rechte.sh` | lokal | ja |
| INC-005 | `INC-005-rpcbind-deaktivieren.sh` | lokal | Opt-in |
| INC-006 | `INC-006-sshd-haertung.sh` | lokal | Opt-in (zweite SSH-Sitzung zum Testen offen halten) |
| INC-007 | `INC-007-teamviewer-deaktivieren.sh` | lokal | Opt-in (Entscheidung, ob TeamViewer gebraucht wird) |
| INC-008 | `INC-008-updates-einspielen.sh` | lokal | ja |
| INC-009 | `INC-009-restic-user-backup.sh` | lokal | Opt-in (braucht `RESTIC_REPOSITORY`, `RESTIC_PASSWORD_FILE`) |
| INC-010 | – (Tailscale-Admin-Konsole) | manuell | – |
| INC-011 | `INC-011-sudoers-leere-datei-entfernen.sh` | lokal | ja |
| INC-012 | `INC-012-pruefwerkzeuge-nachinstallieren.sh` | lokal | ja |
| INC-013 | – (Prüfstatus) | – | – |
| INC-014 | `INC-014-haertung-lynis.sh` | lokal | ja |
| INC-015 | `INC-015-dev-webserver-stoppen.sh` | lokal | Opt-in (beendet einen laufenden Prozess) |
| INC-016 | `INC-016-etcd-snapshot-nfs-mount.sh` | control-plane-1 | ja |
| INC-017 | `INC-017-updates-ohne-reboot.sh` | alle drei Server | Opt-in (Wartungsfenster, Reboot manuell) |
| INC-018 | – (BIOS, LUKS-Neuaufbau) | manuell | – |
| INC-019 | `INC-019-ueberfluessige-dienste.sh` | homeserver | ja |
| INC-020 | `INC-020-ssh-key-audit.sh` (nur lesend, immer), `INC-020-ssh-key-passphrase-setzen.sh` (interaktiv) | lokal | Audit immer; Passphrasen setzen: Opt-in |
| INC-021 | – (Firmware-Updates, Herstellerlinks) | manuell | – |

### Reihenfolge-Empfehlung
1. `bash run-local.sh --dry-run` und `bash run-servers.sh --dry-run` lesen, dann ohne `--dry-run` ausführen.
2. **INC-016** zuerst auf den Servern (Backup-Lücke), danach INC-017 in einem Wartungsfenster mit Drain/Reboot (control-plane-1, prod-vm, zuletzt Homeserver).
3. Opt-in lokal einzeln, in dieser Reihenfolge: **INC-020 (Passphrasen setzen)**, INC-009 (Backup-Ziel festlegen), INC-006, INC-005, INC-007, INC-015.
4. Manuell: INC-002/003/018 (Secure Boot, LUKS), INC-010 (Tailscale), INC-020 (Zugangskonzept: Rollentrennung, `NOPASSWD`), INC-021 (Firmware).
