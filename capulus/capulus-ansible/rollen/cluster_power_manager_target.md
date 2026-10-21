# Rolle `cluster_power_manager_target`

autorisiert den Shutdown-Schlüssel, beschränkt auf `poweroff`.

| | |
|---|---|
| Läuft gegen | worker-0, worker-1 |
| Tags | `cluster-power-manager-target`, `power-management` |
| Quellcode | [roles/cluster_power_manager_target](https://github.com/pkr-lab/capulus-ansible/tree/main/roles/cluster_power_manager_target) |

## Variablen

Defaults aus `defaults/main.yml`. Überschreiben in `group_vars/` oder `host_vars/`.

| Variable | Default | Bedeutung |
|---|---|---|
| `cluster_power_manager_ssh_user` | `ubuntu` | – |
| `cluster_power_manager_ssh_key_path` | `/etc/cluster-power-manager/id_ed25519` | – |
| `cluster_power_manager_poweroff_bin` | `/usr/sbin/poweroff` | – |

## Ablauf

1. Public Key des cluster-power-manager vom Homeserver lesen
2. cluster-power-manager-Key autorisieren (nur für poweroff, forced command)
3. Passwortlosen sudo für poweroff erlauben ({{ cluster_power_manager_ssh_user }})

---

## Fehlersuche

Allgemeines Vorgehen bei Rollen: `ansible-playbook <playbook>.yml --limit <host> --tags <tag> --check --diff -vvv` zeigt Änderungen ohne sie anzuwenden (`make <ziel>-check` bei den Playbooks mit Dry-Run). Die Rollenübersicht mit Make-Zielen: [Rollen](index.md#fehlersuche-je-rolle).

```bash
sudo cat /root/.ssh/authorized_keys ~ubuntu/.ssh/authorized_keys | grep cluster-power
sudo cat /etc/sudoers.d/* | grep poweroff
ssh -i /etc/cluster-power-manager/id_ed25519 ubuntu@worker-0     # auf dem Homeserver: führt nur poweroff aus
```

| Symptom | Ursache | Lösung |
|---|---|---|
| Shutdown des Workers schlägt fehl (`Permission denied (publickey)`) | Schlüssel des Power-Managers nicht in `authorized_keys` des Workers | Rolle auf dem Worker erneut ausrollen (`make worker-0`/`worker-1`) |
| Befehl wird abgewiesen | der Schlüssel ist per `command=` auf `poweroff` beschränkt, andere Befehle funktionieren nicht | gewollt, nur `poweroff` ausführen |
| `sudo: a password is required` | Sudoers-Eintrag für `poweroff` fehlt | Rolle erneut ausrollen, Datei unter `/etc/sudoers.d/` prüfen |

Allgemeine Fehlersuche: [Host-Troubleshooting](../a-betriebssystem/a0040-troubleshooting.md), [zentrale Fehlersuche](../../capulus-core/2-betrieb-hardware/20070-troubleshooting-zentral.md).

[Alle Rollen](index.md)
