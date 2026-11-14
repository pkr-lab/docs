# INC-011 – Benutzer `pke` ist in den Gruppen `docker` und `sudo`

| Feld | Wert |
|---|---|
| ID | INC-011 |
| Datum | 2026-11-14 |
| Kategorie | apt (Rechteausweitung) |
| Schweregrad | **Info** |
| Betroffen | Gruppen `docker` (GID 116), `sudo`, `adm` |

## Beschreibung und Nachweis
```
$ getent group sudo adm docker
sudo:x:27:pke      adm:x:4:syslog,pke      docker:x:116:pke
$ awk -F: '$3>=1000||$3==0' /etc/passwd  -> root, pke, nobody (keine weiteren Benutzer, kein zusätzlicher UID 0)
$ ls -la /etc/sudoers.d  -> 010-pela-nopasswd (0 Byte, leer), ollama-suspend (53 Byte, nicht lesbar)
```
Docker-Mitgliedschaft entspricht faktisch root (Container mit gemountetem `/`). Das ist für einen Entwickler-Arbeitsplatz üblich, aber relevant: Jedes Programm unter `pke` kann damit root werden. Auffällig: Die leere Datei `010-pela-nopasswd` verweist auf einen Benutzer `pela`, der lokal nicht existiert (er existiert in `~/.ssh/config` für infotafel/alarm-monitor). Leere Datei hat keine Wirkung; Inhalt von `ollama-suspend` wurde nicht geprüft.

## Risiko
Kompromittierung eines Prozesses unter `pke` (Browser-Exploit, bösartiges npm-/pip-Paket) führt über Docker-Socket direkt zu root.

## Empfohlener Fix
Bei Bedarf Rootless-Docker oder Podman verwenden; leere Datei `/etc/sudoers.d/010-pela-nopasswd` entfernen (nur nach Bestätigung); `ollama-suspend` mit `sudo cat` prüfen (lesend, ich frage vorher).

## Fix-Skript
[`scripts/incident-fixes/INC-011-sudoers-leere-datei-entfernen.sh`](https://github.com/pkr-lab/capulus-core/blob/main/scripts/incident-scripts/incident-fixes/INC-011-sudoers-leere-datei-entfernen.sh) – entfernt `/etc/sudoers.d/010-pela-nopasswd`, **nur wenn die Datei leer ist** (sonst Abbruch ohne Änderung), und prüft danach `visudo -c`. Die Docker-Gruppenmitgliedschaft bleibt bewusst unverändert (Architekturentscheidung: Rootless-Docker/Podman).

## Rollback
Leere Datei neu anlegen: `sudo install -m 0440 -o root -g root /dev/null /etc/sudoers.d/010-pela-nopasswd` (ursprünglich 0 Byte, Modus 640 root:root, siehe `~/.incident-backups/INC-011-*/stat.txt`).

## Quellen
- https://docs.docker.com/engine/security/#docker-daemon-attack-surface
- https://docs.docker.com/engine/security/rootless/
