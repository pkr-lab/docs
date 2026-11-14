# INC-006 – sshd ohne explizite Härtung (Passwort-Login/X11 nicht abgeschaltet)

| Feld | Wert |
|---|---|
| ID | INC-006 |
| Datum | 2026-11-14 |
| Kategorie | apt / Zugriffskontrolle |
| Schweregrad | **Niedrig** |
| Betroffen | `/etc/ssh/sshd_config` (Workstation) |

## Beschreibung und Nachweis
```
$ grep -hEv '^\s*(#|$)' /etc/ssh/sshd_config
Include /etc/ssh/sshd_config.d/*.conf
KbdInteractiveAuthentication no
UsePAM yes
X11Forwarding yes
(PrintMotd no, AcceptEnv, Subsystem sftp)
$ ls /etc/ssh/sshd_config.d/   -> Verzeichnis existiert nicht
```
`PasswordAuthentication` und `PermitRootLogin` sind nicht gesetzt; es gelten die Debian/Ubuntu-Defaults. **Update:** `sudo sshd -T` bestätigt die effektiven Werte: `passwordauthentication yes`, `x11forwarding yes`, `permitrootlogin prohibit-password`, `maxauthtries 6`, `logingracetime 120`, `allowtcpforwarding yes`, `compression yes`, `pubkeyauthentication yes`. KEX enthält post-quantum `mlkem768x25519-sha256` (gut); MACs enthalten noch `hmac-sha1`/`umac-64`. Der Port ist durch UFW auf `192.168.178.0/24` beschränkt, das Brute-Force-Risiko ist dadurch begrenzt (Schweregrad bleibt Niedrig). Positiv: `authorized_keys` enthält nur 3 Schlüssel mit sprechenden Kommentaren (homelab, prod-vm-admin, ubuntu@homeserver); die Zuordnung zu bekannten Geräten bitte selbst bestätigen.

## Risiko
Bei aktiviertem Passwort-Login ist Brute-Force/Credential-Stuffing möglich, X11-Forwarding vergrößert die Angriffsfläche unnötig.

## Empfohlener Fix
Drop-in `60-hardening.conf` mit `PasswordAuthentication no`, `PermitRootLogin no`, `X11Forwarding no`, `MaxAuthTries 3`, `LoginGraceTime 30`. Das Skript prüft vorher, dass ein Public Key vorhanden ist, validiert mit `sshd -t` und nimmt bei Fehler zurück. **Vor dem Schließen der aktuellen Sitzung in einem zweiten Terminal einen Key-Login testen.**

## Fix-Skript
[`scripts/incident-fixes/INC-006-sshd-haertung.sh`](https://github.com/pkr-lab/capulus-core/blob/main/scripts/incident-scripts/incident-fixes/INC-006-sshd-haertung.sh) – lokal ausführen: `bash INC-006-sshd-haertung.sh --dry-run`. Per SSH nur mit passwortlosem sudo möglich (`ssh host 'bash -s -- --dry-run' < skript.sh`).

## Rollback
`sudo rm /etc/ssh/sshd_config.d/60-hardening.conf && sudo systemctl reload ssh` (oder Backup aus `~/.incident-backups/INC-006-*`).

## Quellen
- https://man.openbsd.org/sshd_config
- https://www.ssh-audit.com/hardening_guides.html
