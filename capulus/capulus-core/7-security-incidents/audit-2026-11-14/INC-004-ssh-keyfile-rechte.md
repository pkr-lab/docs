# INC-004 – Private-Key-Datei mit Gruppen-Schreibrecht (`~/.ssh/client-private.key`)

| Feld | Wert |
|---|---|
| ID | INC-004 |
| Datum | 2026-11-14 |
| Kategorie | malware/apt (Persistenz-/Diebstahlvorbereitung) |
| Schweregrad | **Niedrig** |
| Betroffen | `/home/pke/.ssh/client-private.key`, `client-public.key` |

## Beschreibung und Nachweis
```
$ stat -c '%a %n' ~/.ssh/*
664 /home/pke/.ssh/client-private.key
664 /home/pke/.ssh/client-public.key
600 /home/pke/.ssh/id_ed25519   600 prod_vm_ed25519   600 netcup-deploy   600 config
$ ls -l ~/.ssh/client-private.key   -> -rw-rw-r-- 1 pke pke 45 Sep 13 20:29
```
Die Datei ist 45 Byte groß (vermutlich ein Schlüssel im Base64-Format, z. B. WireGuard; Inhalt nicht gelesen). Sie ist für alle lesbar und für die Gruppe `pke` beschreibbar. Auf diesem Einzelnutzer-System ist die Gruppe nur `pke`, die Lesbarkeit für „others“ (`r`) ist das eigentliche Problem.

## Risiko
Jeder lokale Benutzer oder Prozess (auch ein kompromittierter Dienst mit anderer UID) kann den privaten Schlüssel lesen und sich als dieser Client ausgeben.

## Empfohlener Fix
`chmod 700 ~/.ssh; chmod 600 ~/.ssh/client-private.key; chmod 644 ~/.ssh/client-public.key`. Wurde der Schlüssel schon von anderen UIDs gelesen (unwahrscheinlich, Einzelnutzer-System), neu erzeugen.

## Fix-Skript
[`scripts/incident-fixes/INC-004-ssh-key-rechte.sh`](https://github.com/pkr-lab/capulus-core/blob/main/scripts/incident-scripts/incident-fixes/INC-004-ssh-key-rechte.sh) – `bash INC-004-ssh-key-rechte.sh --dry-run`, speichert alte Modi unter `~/.incident-backups/INC-004-*/modes.txt`.

## Rollback
`chmod <alter Modus> <datei>` gemäß `modes.txt` (alter Modus war 664).

## Quellen
- `man ssh`, Abschnitt FILES
- https://cwe.mitre.org/data/definitions/732.html
