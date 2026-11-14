# INC-019 – Homeserver: unnötige Desktop-/Netzdienste und nicht zugeordnete offene Ports

| Feld | Wert |
|---|---|
| ID | INC-019 |
| Datum | 2026-11-14 |
| Kategorie | ddos / Angriffsfläche |
| Schweregrad | **Info** (vorher Niedrig; CUPS, SANE und rpcbind sind laut Incident 038 bewusst, nur `cups-browsed` und `ModemManager` sind ohne Bedarf) |
| Betroffen | homeserver (192.168.178.94), teils control-plane-1 und prod-vm |

## Beschreibung und Nachweis
```
$ ss -tln (homeserver), Auswahl auf 0.0.0.0/*:
22 (sshd), 111 (rpcbind), 631 (cupsd), 6566 (saned über inetd), 9310 (python3 /opt/alert-caller/caller.py, User ubuntu),
2623 (laut Incident 038 der FRR-`mgmtd` des MetalLB-Pods, antwortet nur „Vty password is not set“; auch auf control-plane-1),
6443/10250 (k3s API/kubelet, *), 9100/9120/9140/9141/9633 (Exporter, *)
$ systemctl list-units --state=running  -> u. a. inetd.service, cups.service, cups-browsed.service, ModemManager.service,
  colord, udisks2, upower, avahi-daemon, multipathd, rpcbind, rpc-statd, wpa_supplicant
$ cat /etc/inetd.conf -> "sane-port stream tcp nowait saned:saned /usr/sbin/saned saned"
$ systemctl is-active ufw nftables crowdsec crowdsec-firewall-bouncer -> active inactive active active
```
Der Homeserver hat die Rolle eines Servers, aber viele Desktop-Pakete (hplip, SANE-Netzwerkscanner, ModemManager, colord). `cups-browsed` hatte 2024 eine kritische Schwachstellenkette (CVE-2024-47076/-47175/-47176/-47177); Ubuntu hat sie gepatcht, ein Dienst ohne Bedarf bleibt dennoch Angriffsfläche. Dein Arbeitsplatz hat Drucker-PPDs mit dem Namensbestandteil `homeserver` (Samsung ML-1630/M2026), vermutlich werden Drucker über den Homeserver genutzt, CUPS selbst wird dann gebraucht; `saned`/`inetd`, `cups-browsed` und `ModemManager` vermutlich nicht. Welche der Ports durch UFW im LAN erreichbar sind, ist offen: `ufw status` benötigt `sudo` auf dem Server.

## Abgleich mit dem Incident-Register (2026-11-14)
`pkr-lab-docs/capulus/capulus-core/7-security-incidents/038-unnoetige-netzwerkdienste-auf-den-hosts.md` dokumentiert CUPS (`:631`) und SANE (`:6566`, `saned` über `inetd`) als **bewussten** Druck-/Scanserver (akzeptiertes Restrisiko) und `rpcbind` als nötig für die NFS-Mounts (`vers=3`). Meine ursprüngliche Empfehlung, `inetd` abzuschalten, hätte das Scannen beendet und war falsch; das Skript wurde angepasst. Wer `rpcbind` abschalten will, muss zuerst NFSv4.1 am NAS aktivieren (Incident 038).

## Risiko
Jeder erreichbare unnötige Dienst vergrößert die Angriffsfläche (RCE-Lücken in Druck-/Scannerdiensten, Amplification über rpcbind). `9310/alert-caller` ist ein Eigenbau-Dienst mit SIP-Zugangsdaten laut Repo; Authentifizierung und Erreichbarkeit nicht geprüft.

## Empfohlener Fix
1. Lesend prüfen (benötigt `sudo` auf dem Homeserver; sag Bescheid): `sudo ufw status verbose`, `sudo ss -tulpn`, `sudo cat /etc/cups/cupsd.conf | grep -E '^(Listen|Browsing|<Location|Allow|Order)'`.
2. Skript: deaktiviert `cups-browsed` und `ModemManager` (nur wenn aktiv/aktiviert); `inetd`/SANE bleiben, weil das Scannen über den Homeserver gewollt ist: `ssh control-plane-0 'bash -s -- --dry-run' < scripts/incident-fixes/INC-019-ueberfluessige-dienste.sh`
3. CUPS/SANE auf die nutzenden Geräte beschränken, sobald die Liste feststeht (`ufw allow from <IP> to any port 631,6566`); `rpcbind` bleibt, solange die NAS-Mounts `vers=3` nutzen (Incident 038).
4. `alert-caller` an die LAN-/Cluster-Adresse binden und per UFW nur den notwendigen Quellen erlauben.

## Fix-Skript
[`scripts/incident-fixes/INC-019-ueberfluessige-dienste.sh`](https://github.com/pkr-lab/capulus-core/blob/main/scripts/incident-scripts/incident-fixes/INC-019-ueberfluessige-dienste.sh) (nur `cups-browsed`, `ModemManager`). In Ansible: `common_disabled_services` und `cups_print_server_disabled_services`. Zustand vorher unter `/var/backups/incident-INC-019-*/state.txt`.

## Rollback
`sudo systemctl enable --now <unit>` für die im Backup gelisteten Units.

## Quellen
- https://www.cups.org/ ; https://ubuntu.com/security/CVE-2024-47176
- https://ubuntu.com/server/docs/security-firewall
