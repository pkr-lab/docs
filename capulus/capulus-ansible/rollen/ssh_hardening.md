# Rolle `ssh_hardening`

Schlüsselbasierter SSH-Zugang (Passwort-Login aus).

| | |
|---|---|
| Läuft gegen | alle verwalteten Hosts |
| Tags | `security`, `ssh-hardening` |
| Quellcode | [roles/ssh_hardening](https://github.com/pkr-lab/capulus-ansible/tree/main/roles/ssh_hardening) |

## Variablen

Defaults aus `defaults/main.yml`. Überschreiben in `group_vars/` oder `host_vars/`.

| Variable | Default | Bedeutung |
|---|---|---|
| `ssh_hardening_dropin` | `/etc/ssh/sshd_config.d/00-hardening.conf` | – |
| `ssh_hardening_max_auth_tries` | `4` | – |
| `ssh_hardening_allow_tcp_forwarding` | `false` | – |

## Ablauf

1. sshd-härtung ablegen, prüfen und anwenden
2. sshd-drop-in ablegen
3. sshd-konfiguration prüfen
4. wirksame sshd-werte lesen
5. prüfen dass das drop-in die werte aus anderen dateien übersteuert
6. sshd-handler ausführen
7. fehlerhaftes sshd-drop-in entfernen
8. ablauf abbrechen

## Handler

- sshd neu laden

## Templates

- `00-hardening.conf.j2`

---

## Fehlersuche

Allgemeines Vorgehen bei Rollen: `ansible-playbook <playbook>.yml --limit <host> --tags <tag> --check --diff -vvv` zeigt Änderungen ohne sie anzuwenden (`make <ziel>-check` bei den Playbooks mit Dry-Run). Die Rollenübersicht mit Make-Zielen: [Rollen](index.md#fehlersuche-je-rolle).

```bash
sudo sshd -t
sudo sshd -T | grep -E "passwordauthentication|permitrootlogin|pubkeyauthentication"
ls /etc/ssh/sshd_config.d/
sudo journalctl -u ssh -n 50 --no-pager
```

| Symptom | Ursache | Lösung |
|---|---|---|
| `Das Drop-in … wird von einer früher geladenen Datei übersteuert.` | Ein früheres Drop-in (zum Beispiel `50-cloud-init.conf`) setzt dieselben Werte | Der Dateiname der Härtung muss alphabetisch vor `50-cloud-init.conf` liegen (Standard `00-hardening.conf`) |
| `sshd-Härtung abgebrochen, das Drop-in … wurde entfernt. sshd wurde nicht neu geladen.` | `sshd -t` meldete einen Fehler in der Konfiguration | Fehler aus `sshd -t` lesen, Variablen der Rolle prüfen. Die Rolle entfernt das fehlerhafte Drop-in selbst |
| Anmeldung per Passwort nicht mehr möglich | Härtung aktiv (gewollt), Schlüssel fehlt | öffentlichen Schlüssel auf dem Host hinterlegen (Konsole am Gerät) |
| Aussperrung | Schlüssel nicht autorisiert, bevor die Härtung lief | Konsole am Gerät, `sudo rm /etc/ssh/sshd_config.d/00-hardening.conf && sudo systemctl reload ssh` |

Allgemeine Fehlersuche: [Host-Troubleshooting](../a-betriebssystem/a0040-troubleshooting.md), [zentrale Fehlersuche](../../capulus-core/2-betrieb-hardware/20070-troubleshooting-zentral.md).

[Alle Rollen](index.md)
