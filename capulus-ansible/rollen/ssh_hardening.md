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

[Alle Rollen](index.md)
