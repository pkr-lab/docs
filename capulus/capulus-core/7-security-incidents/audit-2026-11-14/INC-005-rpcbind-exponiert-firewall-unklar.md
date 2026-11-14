# INC-005 – rpcbind/Ollama auf allen Schnittstellen (durch UFW eingeschränkt)

| Feld | Wert |
|---|---|
| ID | INC-005 |
| Datum | 2026-11-14 |
| Kategorie | ddos / Angriffsfläche |
| Schweregrad | **Info** (vorher Niedrig; Tailnet-Erreichbarkeit noch zu prüfen, ggf. wieder Niedrig) |
| Betroffen | `rpcbind` (TCP/UDP 111 auf 0.0.0.0 und [::]), `sshd` (TCP 22 auf 0.0.0.0), `ufw` |

## Beschreibung und Nachweis
```
$ ss -tulpn
tcp LISTEN 0.0.0.0:111   udp UNCONN 0.0.0.0:111   udp [::]:111
tcp LISTEN 0.0.0.0:22
nur loopback (127.0.0.1): 631 (cups), 902 (vmware), 5939 (teamviewer), 50503/39305/40111 (code/claude), 53 (resolved)
$ systemctl is-active ufw nftables  -> active / inactive
```
`ufw` ist aktiv, die Regeln lassen sich ohne root nicht lesen (`sudo ufw status verbose` nötig, siehe unten). Ob 111 und 22 aus dem LAN (192.168.178.0/24) erreichbar sind, ist daher unbestätigt. Nicht festgestellt, aber auffällig: viele UDP-Hochports auf 0.0.0.0 (z. B. 42845, 46317, 56605, 56627, 33328, 39739, 41641), 41641 gehört zu Tailscale; die übrigen sind vermutlich mDNS/Avahi-Sockets und wurden nicht zugeordnet (Prozessname ohne root nicht auslesbar).

## Update 2026-11-14: Ergebnis von `sudo ufw status verbose` und `sudo ss -tulpn`
```
Status: Aktiv, Protokollierung low, Voreinstellung: deny (eingehend), allow (abgehend), deny (gesendet)
22/tcp     ALLOW IN  192.168.178.0/24     # SSH
11434/tcp  ALLOW IN  192.168.178.94       # Ollama API (Homeserver)
```
- Die UFW-Regeln selbst sind **sinnvoll konfiguriert**: alles eingehende ist gesperrt, bis auf SSH aus dem LAN und die Ollama-API nur vom Homeserver. `rpcbind` (111, `0.0.0.0` und `[::]`), `python3 http.server` (8000, siehe INC-015) und weitere Sockets sind damit aus dem LAN nicht erreichbar. **Unbestätigt: das Tailnet.** Tailscale legt unter Linux üblicherweise eigene Regeln an (Kette `ts-input`, akzeptiert Verkehr auf `tailscale0`), die vor UFW greifen können. Dann wären 22, 111, 8000 und 11434 für Tailnet-Teilnehmer erreichbar, begrenzt nur durch die Tailscale-ACLs (siehe INC-010). Prüfung (lesend, `sudo`): `sudo iptables -S ts-input` bzw. `sudo nft list ruleset | grep -A6 ts-input`, plus ACL-Ansicht in der Tailscale-Admin-Konsole.
- Zuordnung der UDP-Ports: `tailscaled` (41641), `wsdd` (python3, 3702 und hohe Ports: GVfs-Windows-Netzwerkerkennung unter `/run/user/1000/gvfsd`), `avahi-daemon` (5353), `chronyd` (nur Loopback 323), `rpcbind` (111, per systemd-Socket aktiviert). Keine unbekannten Prozesse.
- `ollama` lauscht auf `*:11434` (alle Interfaces, laut Ausgabe läuft es als Benutzer `ollama`), ist aber per UFW auf 192.168.178.94 beschränkt. Die Ollama-API hat keine Authentifizierung: Die Regel ist die einzige Schutzschicht. Ein Docker-Container mit veröffentlichtem Port würde UFW umgehen; aktuell wurde kein laufender Container festgestellt (`docker ps` lieferte keine Zeilen).
- UFW-Blocks (letzte Stunde: 128): fast ausschließlich vom Router 192.168.178.1 bzw. dessen Link-Local-Adresse (Heimnetz-Rauschen), kein Hinweis auf Scans.

## Abgleich mit dem Incident-Register (2026-11-14)
Gilt für den **Arbeitsplatz**: Dort gibt es keine NFS-Einbindung, `rpcbind` ist entbehrlich (Ansible: `ollama_pc_rpcbind_disabled`, greift nur, wenn kein NFS gemountet ist). Auf den **Servern** ist `rpcbind` dagegen nötig, solange die NAS-Mounts `vers=3` nutzen (Incident 038); dort nicht abschalten. Die Server-UFW erlaubt laut Incident 008 das gesamte LAN und Tailnet (`common_ufw_lan_allow_all`, `common_ufw_tailnet_allow_all`).

## Risiko
rpcbind wird für DDoS-Reflection/Amplification (portmap) und zur Diensterkennung missbraucht, wenn die Firewall 111 durchlässt. Port 22 ist normal, aber Ziel von Brute-Force (bisher 0 fehlgeschlagene Logins in 7 Tagen laut `journalctl -u ssh`).

## Empfohlener Fix
1. Erledigt: UFW-Regeln und Prozesszuordnung geprüft (siehe Update oben).
2. Wenn keine NFS-Freigaben genutzt werden: rpcbind abschalten (Skript).
3. UFW-Standard ist bereits `deny incoming` und SSH auf das LAN beschränkt. Optional: `ollama` per `OLLAMA_HOST=192.168.178.66:11434` (oder Loopback) nur an die LAN-Adresse binden, damit der Schutz nicht allein von UFW abhängt.

## Fix-Skript
[`scripts/incident-fixes/INC-005-rpcbind-deaktivieren.sh`](https://github.com/pkr-lab/capulus-core/blob/main/scripts/incident-scripts/incident-fixes/INC-005-rpcbind-deaktivieren.sh) (bricht ab, wenn NFS-Mounts aktiv sind). UFW-Regeln sind bewusst nicht automatisiert, solange die Ist-Regeln unbekannt sind.

## Rollback
`sudo systemctl enable --now rpcbind.socket rpcbind.service` (Ursprungszustand steht in `~/.incident-backups/INC-005-*/state.txt`).

## Quellen
- https://www.cisa.gov/news-events/alerts/2014/01/17/udp-based-amplification-attacks
- https://help.ubuntu.com/community/UFW
