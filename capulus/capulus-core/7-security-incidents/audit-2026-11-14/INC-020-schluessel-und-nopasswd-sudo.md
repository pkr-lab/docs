# INC-020 – Root-äquivalenter Zugang per SSH-Schlüssel auf allen Servern (`NOPASSWD:ALL`)

| Feld | Wert |
|---|---|
| ID | INC-020 |
| Datum | 2026-11-14 |
| Kategorie | apt (laterale Bewegung) |
| Schweregrad | **Hoch** (vorher Mittel; hochgestuft nach Prüfung der Schlüssel, siehe Update) |
| Betroffen | `/etc/sudoers.d/ubuntu`, `~ubuntu/.ssh/authorized_keys` auf homeserver, control-plane-1 (prod-vm: sudoers-Datei nicht lesbar) |

## Beschreibung und Nachweis
```
$ cat /etc/sudoers.d/ubuntu   (homeserver, control-plane-1)  -> ubuntu ALL=(ALL) NOPASSWD:ALL
$ authorized_keys (nur Optionen/Kommentar, keine Key-Blobs):
homeserver:      homelab (ohne Optionen) · semaphore@homeserver (ohne Optionen) · mac (ohne Optionen)
                 · n8n-nightly-worker-trigger  [command="sudo /usr/bin/systemctl …" – eingeschränkt]
control-plane-1: homelab · semaphore@homeserver (beide ohne Optionen)
prod-vm:         prod-vm-admin · ubuntu@homeserver (ohne Optionen)
$ ssh-keygen -lf (Fingerprint): "homelab" = SHA256:OZSNcvmv… identisch auf Homeserver, control-plane-1 UND in ~/.ssh/authorized_keys des Arbeitsplatzes
$ sshd (alle Server): PasswordAuthentication no, PermitRootLogin no, AllowTcpForwarding no, AllowAgentForwarding no
```
Positiv: Passwort-Login ist abgeschaltet; ein Schlüssel (n8n) ist per `command=` korrekt eingeschränkt.

## Update 2026-11-14: Ergebnis von `INC-020-ssh-key-audit.sh` (vom Benutzer ausgeführt)
```
id_ed25519       modus=600  OHNE Passphrase  fingerprint=SHA256:OZSNcvmv…  (= "homelab")
netcup-deploy    modus=600  OHNE Passphrase
prod_vm_ed25519  modus=600  OHNE Passphrase  fingerprint=SHA256:FVlkIuQb…  (= "prod-vm-admin")
authorized_keys (Arbeitsplatz): homelab, prod-vm-admin, ubuntu@homeserver (alle ohne Optionen)
```
Alle drei privaten Schlüssel liegen **unverschlüsselt** auf der **unverschlüsselten** Platte (INC-003). Der Schlüssel `id_ed25519` („homelab“) ist auf Homeserver und control-plane-1 per `NOPASSWD:ALL` root-äquivalent, `prod_vm_ed25519` auf der prod-vm, `netcup-deploy` gehört zum Hosting-Zugang. Wer eine Kopie der Dateien erhält (Malware unter deinem Benutzer, Backup, gestohlene Platte, die offene `.git`-Auslieferung aus INC-015 betrifft nur das Projektverzeichnis), hat sofort Root auf den Servern, ohne weiteres Geheimnis. Umgekehrt akzeptiert dein Arbeitsplatz den Schlüssel `ubuntu@homeserver` und `prod-vm-admin`: Ein kompromittierter Homeserver oder die prod-vm kann sich als `pke` am Arbeitsplatz anmelden.

## Abgleich mit dem Incident-Register (2026-11-14)
Das Register kennt `NOPASSWD:ALL` für `ubuntu` (Incident 009) und einen passphrasenlosen Banana-Pi-Schlüssel im Git-Verlauf (Incident 001). **Neu** ist, dass die privaten Schlüssel des **Arbeitsplatzes** (`id_ed25519` = „homelab“, `prod_vm_ed25519`, `netcup-deploy`) keine Passphrase haben und der Arbeitsplatz keine Plattenverschlüsselung hat (INC-003). Die Einstufung Hoch bleibt, weil ein einziger Diebstahl/Malware-Lauf damit Root auf Homeserver und Control-Plane bedeutet.

## Risiko
Wer den Schlüssel `homelab` (oder `semaphore`/`mac`/`prod-vm-admin`) besitzt, erhält sofort Root auf Homeserver und Control-Plane. Dein Arbeitsplatz (unverschlüsselte Platte, Dev-Webserver, Docker-Gruppe, TeamViewer) ist damit das schwächste Glied einer Kette zu allen Servern. Ob die privaten Schlüssel mit einer Passphrase geschützt sind, ist nicht erkennbar (ohne Inhaltsprüfung).

## Empfohlener Fix (manuell, Entscheidung nötig)
1. **Passphrasen setzen** (Prüfung ergab: keine vorhanden): `bash scripts/incident-fixes/INC-020-ssh-key-passphrase-setzen.sh --dry-run`, dann ohne `--dry-run`. Das Skript fragt je Schlüssel interaktiv nach der neuen Passphrase. Komfort bleibt über den `ssh-agent`/GNOME-Keyring (`gcr-ssh-agent` ist aktiv): Passphrase einmal pro Sitzung. Skripte, die den Schlüssel unbeaufsichtigt nutzen, brauchen einen eigenen Automatisierungsschlüssel (Schritt 2).
2. Pro Zweck eigene Schlüssel (Mensch ↔ Automatisierung), Automatisierungsschlüssel mit `from="192.168.178.94"` und ggf. `command=` einschränken.
3. Für den Menschen FIDO2-Schlüssel (`ssh-keygen -t ed25519-sk`) erwägen: Der private Teil verlässt den Hardware-Token nicht.
4. `NOPASSWD:ALL` durch gezielte Regeln ersetzen (z. B. nur `systemctl`-Befehle für Automatisierung), `sudo` für Menschen mit Passwort oder `pam_ssh_agent_auth`.
5. Schlüsselrotation vierteljährlich dokumentieren.

## Fix-Skript
Zwei Skripte: (a) [`scripts/incident-fixes/INC-020-ssh-key-passphrase-setzen.sh`](https://github.com/pkr-lab/capulus-core/blob/main/scripts/incident-scripts/incident-fixes/INC-020-ssh-key-passphrase-setzen.sh) setzt Passphrasen auf `id_ed25519`, `prod_vm_ed25519`, `netcup-deploy` (und `id_rsa`, falls vorhanden), nur wo keine existiert; **interaktiv**, daher nur im Opt-in des Runners. Es legt vorher eine Kopie des ungeschützten Schlüssels unter `~/.incident-backups/INC-020-*` an (nach erfolgreichem Test mit `shred -u` löschen). Das Zugangskonzept (Rollentrennung, `NOPASSWD`, `from=`/`command=`) bleibt manuell, weil ein Fehler dich von allen Servern aussperren würde. (b) Lesendes Prüfskript: [`scripts/incident-fixes/INC-020-ssh-key-audit.sh`](https://github.com/pkr-lab/capulus-core/blob/main/scripts/incident-scripts/incident-fixes/INC-020-ssh-key-audit.sh) meldet je privatem Schlüssel in `~/.ssh`, ob er **mit oder ohne Passphrase** gesichert ist (ohne Ausgabe von Schlüsseldaten), plus Optionen/Kommentar der `authorized_keys`-Einträge. Aufruf: `bash scripts/incident-fixes/INC-020-ssh-key-audit.sh` (läuft auch im Runner `run-local.sh`). Die Umstellung des Zugangskonzepts bleibt manuell (siehe oben).

## Rollback
Passphrase wieder entfernen: `ssh-keygen -p -f <key> -N ''`, oder den Schlüssel aus `~/.incident-backups/INC-020-*` zurückkopieren (Modus 600).

## Quellen
- https://man.openbsd.org/sshd#AUTHORIZED_KEYS_FILE_FORMAT
- https://www.openssh.com/manual.html (FIDO/U2F)
