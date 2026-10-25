# Connect

SSH-Verbindung zu den VMs und Hosts aus der Konfiguration.

Code: [modules/connect.py](https://github.com/pkr-lab/cubi/blob/main/src/cubi/modules/connect.py)

---

## Überblick

`cubi connect` zeigt die Gruppen aus `connect:` in [targets.yaml](konfiguration.md) (zum Beispiel VM und Hosts) und danach die Ziele der gewählten Gruppe. Die Wahl startet `ssh <alias>`. cubi übergibt dabei das Terminal an `ssh` (`os.execvp`), das heißt: cubi ist danach beendet, und nach dem Ende der SSH-Sitzung ist man wieder in der Shell.

Wer das Tailnet nutzt, kommt über [Tailscale](tailscale.md) zum selben Ergebnis, dort aus der Geräteliste.

---

## Ablauf

1. `run()` liest `connect` aus der Config. Ohne Eintrag: `Keine Connect-Ziele in targets.yaml`.
2. Gruppenauswahl, mit Anzahl der Ziele je Gruppe (`3 Ziele`). Esc beendet den Bereich.
3. Zielauswahl: `targets()` prüft jeden Eintrag auf `name` und `ssh`. Zu jedem Eintrag ruft `describe()` `ssh -G <alias>` (5 Sekunden Timeout) und zeigt `benutzer@hostname` als Hinweis. Schlägt das fehl, bleibt der Hinweis leer.
4. Auswahl → `shell.replace(["ssh", alias])`.

Zurück aus der Zielauswahl (Esc) führt zur Gruppenauswahl.

---

## Konfiguration

Siehe [Konfiguration](konfiguration.md), Schlüssel `connect.<gruppe>[]` mit `name` und `ssh`. SSH-Optionen (Benutzer, Port, Schlüssel, ProxyJump) stehen in `~/.ssh/config`, nicht in cubi.

---

## Randfälle

- Die Hinweise `benutzer@hostname` kommen aus der lokalen SSH-Konfiguration, nicht vom Server. cubi prüft nicht, ob der Host erreichbar ist.
- Für jeden Eintrag der gewählten Gruppe läuft ein `ssh -G`, bei vielen Zielen ist die Zielauswahl daher einen Moment verzögert.
- Es gibt keine Auswahl mehrerer Ziele und keinen Befehl auf dem Ziel, nur die interaktive Sitzung.

---

## Debugging und Fehlersuche

| Meldung | Ursache | Lösung |
|---|---|---|
| `Keine Connect-Ziele in targets.yaml` | Abschnitt `connect` fehlt | Gruppen in `targets.yaml` ergänzen, siehe [Konfiguration](konfiguration.md) |
| `Keine Ziele in connect.<gruppe>` | Gruppe ist leer oder keine Liste | Einträge als YAML-Liste (`- name: …`) angeben |
| `Ungültiger Eintrag in connect.<gruppe>: name und ssh sind Pflicht` | Eintrag ohne `name` oder `ssh` | beide Felder setzen |
| `ssh ist nicht installiert oder nicht im PATH` | OpenSSH-Client fehlt | `sudo apt install openssh-client` |
| `ssh: Could not resolve hostname …` (aus ssh, nach dem Wechsel in die Shell) | Alias fehlt in `~/.ssh/config` oder Name nicht auflösbar | `ssh -G <alias>` ausführen, Host-Eintrag ergänzen |
| `Permission denied (publickey)` | Schlüssel nicht hinterlegt | `IdentityFile` im Host-Eintrag prüfen, Schlüssel auf dem Ziel hinterlegen, siehe [SSH-Härtung](../capulus/capulus-ansible/rollen/ssh_hardening.md) |
| Hinweis hinter dem Eintrag fehlt | `ssh -G` schlug fehl oder dauerte über 5 Sekunden | harmlos; `ssh -G <alias>` manuell prüfen |

Checkliste „Connect funktioniert nicht“:

1. `ssh <alias>` direkt in der Shell: klappt es dort nicht, liegt es nicht an cubi.
2. `ssh -G <alias>`: stimmen `hostname`, `user`, `identityfile`?
3. Bei Tailnet-Hosts: `tailscale status`, ist das Gerät online? Siehe [Tailscale](tailscale.md).
4. Host im lokalen Netz: Erreichbarkeit mit `ping`, DNS siehe [DNS-Architektur](../capulus/capulus-core/c-netzwerk-dns/c0000-dns-architecture.md).

Allgemeines: [Debugging und Fehlersuche](debugging.md).
