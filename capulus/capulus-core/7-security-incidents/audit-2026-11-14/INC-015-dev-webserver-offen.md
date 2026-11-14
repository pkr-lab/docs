# INC-015 – `python3 -m http.server 8000` auf allen Schnittstellen liefert Repo inklusive `.git`

| Feld | Wert |
|---|---|
| ID | INC-015 |
| Datum | 2026-11-14 |
| Kategorie | apt / Informationsabfluss |
| Schweregrad | **Niedrig** (im LAN durch UFW gesperrt; Tailnet ungeprüft, dort ggf. Mittel) |
| Betroffen | Prozess PID 27602, `0.0.0.0:8000`, Verzeichnis `~/git/github/automations/EDV-Kretzer` |

## Beschreibung und Nachweis
```
$ ps -o pid,user,lstart,args -p 27602
27602  pke  Sa Nov 14 11:54:20 2026  python3 -m http.server 8000      (Elternprozess: systemd --user)
$ readlink /proc/27602/cwd -> /home/pke/git/github/automations/EDV-Kretzer
$ sudo ss -tulpn | grep 8000 -> tcp LISTEN 0.0.0.0:8000 users:(("python3",pid=27602))
$ curl -s -o /dev/null -w '%{http_code}' http://127.0.0.1:8000/ -> HTTP 200
$ find . -maxdepth 3 -name '.git' -> ./.git
```
`http.server` bindet ohne `--bind` an alle Schnittstellen und liefert **jede** Datei im Verzeichnis aus, auch `.git` (Quelltext, Verlauf, ggf. frühere Secrets) und `CLAUDE.md`. Gestartet wurde der Prozess um 11:54 Uhr unter deinem Benutzer; wer ihn gestartet hat (du oder ein Werkzeug in deiner Sitzung), lässt sich aus den Daten nicht ersehen. Die Dateinamen enthalten keine `.env`/`.key`/`.pem` (nur Namen geprüft, keine Inhalte). Ein Login oder eine Authentifizierung gibt es nicht.

## Risiko
Jeder, der Port 8000 erreicht, kann das gesamte Projektverzeichnis samt Git-Historie herunterladen (`git-dumper`-artig). Aktuell verhindert UFW (nur 22/11434 erlaubt) den Zugriff aus dem LAN. Für das **Tailnet ist das unbestätigt** (Tailscale-Regeln `ts-input` können vor UFW greifen, siehe INC-005); dann hätten alle Tailnet-Geräte (u. a. Handy, MacBook, Cluster-Knoten) Zugriff. Bei deaktivierter Firewall, einem Docker-Portmapping-Bypass oder in einem fremden WLAN wäre es offen.

## Empfohlener Fix (manuell)
1. Server beenden, wenn nicht mehr gebraucht: `kill 27602` (Prozess gehört dir; ich habe ihn nicht angefasst, weil ich ihn nicht gestartet habe).
2. Künftig nur Loopback: `python3 -m http.server 8000 --bind 127.0.0.1`; für Zugriff von anderen Geräten besser Tailscale mit gezielter UFW-Regel statt `0.0.0.0`.
3. Statisches Build-Verzeichnis ausliefern, nicht das Repo-Wurzelverzeichnis: `python3 -m http.server -d dist --bind 127.0.0.1 8000`.

## Fix-Skript
[`scripts/incident-fixes/INC-015-dev-webserver-stoppen.sh`](https://github.com/pkr-lab/capulus-core/blob/main/scripts/incident-scripts/incident-fixes/INC-015-dev-webserver-stoppen.sh) – beendet `python3 -m http.server 8000` unter deinem Benutzer, **nur wenn er auf allen Schnittstellen lauscht**, und sichert Verzeichnis und Kommandozeile. Es ist Opt-in im Runner, weil es einen Prozess beendet, den du eventuell gerade nutzt.

## Rollback
Neustart nur lokal: `cd <Verzeichnis aus ~/.incident-backups/INC-015-*/process.txt> && python3 -m http.server 8000 --bind 127.0.0.1`.

## Quellen
- https://docs.python.org/3/library/http.server.html (Hinweis: „not recommended for production“, Option `--bind`)
- https://cwe.mitre.org/data/definitions/538.html
