# INC-014 – Lynis-Härtungsvorschläge (Arbeitsplatz)

| Feld | Wert |
|---|---|
| ID | INC-014 |
| Datum | 2026-11-14 |
| Kategorie | apt / schwachstellen (Härtung) |
| Schweregrad | **Info** |
| Betroffen | Workstation `pke-ms-7e59` |

## Beschreibung und Nachweis
```
$ lynis audit system --quick   (ohne root, Umfang eingeschränkt)
Hardening index : 66    Tests performed : 249    Warnings : 0    Suggestions : 38
```
Relevante Vorschläge (Auswahl, nicht alle sind für einen Desktop sinnvoll):
- **ACCT-9628 / FINT-4350:** `auditd` nicht aktiv, kein Integritätswerkzeug (z. B. AIDE, `debsums`) installiert.
- **DEB-0880:** kein `fail2ban` (bei 0 fehlgeschlagenen SSH-Logins in 7 Tagen niedrige Priorität).
- **BOOT-5122:** kein GRUB-Passwort (zusammen mit INC-002/INC-003 relevant).
- **LOGG-2154 / LOGG-2190:** kein externer Log-Host; gelöschte, noch geöffnete Dateien prüfen.
- **AUTH-9286 / AUTH-9230 / AUTH-9328:** Passwortalter, Hash-Rounds und umask in `/etc/login.defs`.
- **PKGS-7346:** 33 entfernte Pakete mit Konfigurationsresten (`dpkg --purge`; nur die Kernelpakete automatisiert, siehe Fix-Skript), **PKGS-7370/7398:** `debsums` bzw. Paket-Audit-Tool installieren.
- **NETW-3200:** Protokolle `dccp`, `sctp`, `rds`, `tipc` per `modprobe`-Blacklist sperren; **USB-1000:** USB-Storage-Treiber (optional).
- **KRNL-6000:** einzelne sysctl-Werte weichen vom Profil ab; **HRDN-7222:** Compiler nicht eingeschränkt; **PRNT-2307:** CUPS-Konfiguration strenger.
Der Index 66 ist für einen Desktop mit Entwicklungswerkzeugen im üblichen Bereich; ein root-Lauf liefert mehr Tests (Index kann abweichen).

## Risiko
Es sind keine akuten Schwachstellen, sondern fehlende Tiefenverteidigung: Ohne `auditd` und Integritätsprüfung wären Manipulationen an Systemdateien schwer nachweisbar (APT-Szenario).

## Empfohlener Fix
Priorität: (1) `auditd` + `debsums`/AIDE, (2) Blacklist ungenutzter Netzwerkprotokolle, (3) `dpkg --purge` der 33 Reste (Liste: `dpkg -l | awk '/^rc/{print $2}'`), (4) übrige nach Bedarf. Installation von Paketen nur nach deiner Freigabe.

## Fix-Skript
[`scripts/incident-fixes/INC-014-haertung-lynis.sh`](https://github.com/pkr-lab/capulus-core/blob/main/scripts/incident-scripts/incident-fixes/INC-014-haertung-lynis.sh) – installiert `auditd` und `debsums`, aktiviert `auditd`, sperrt `dccp`, `sctp`, `rds`, `tipc` per `/etc/modprobe.d/60-incident-unused-net-protocols.conf` und purged entfernte **Kernel-Pakete** mit Konfigurationsresten (`linux-image-*`, `linux-modules-*`, `linux-main-modules-*`, `linux-modules-nvidia-*`; Liste vorher gesichert). Andere `rc`-Pakete werden nur **aufgelistet** und nicht angefasst: Der Trockenlauf zeigte u. a. `tor`, `torbrowser-launcher`, `torsocks`, `github-desktop` und `grub-pc`, bei denen eine selbst angepasste Konfiguration (z. B. `/etc/tor/torrc`) verloren gehen würde. Nicht enthalten sind GRUB-Passwort, `/etc/login.defs`, Compiler-Einschränkung und sysctl-Tuning (Abwägung je Maßnahme). Hinweis: Wer SCTP, TIPC oder RDS tatsächlich nutzt, entfernt die entsprechende Zeile vor dem Lauf.

## Rollback
`sudo rm /etc/modprobe.d/60-incident-unused-net-protocols.conf`; `sudo systemctl disable --now auditd`; `sudo apt-get remove auditd debsums`; entfernte Kernelpakete: Liste unter `/var/backups/incident-INC-014-*/rc-kernel-packages.txt` (Konfigurationsreste lassen sich nicht wiederherstellen, nur Pakete neu installieren; bei Kernel-Residuen ohne Folgen).

## Quellen
- https://cisofy.com/lynis/
- https://cisofy.com/lynis/controls/
