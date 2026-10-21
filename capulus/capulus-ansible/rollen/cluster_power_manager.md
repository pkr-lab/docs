# Rolle `cluster_power_manager`

weckt Worker per WoL, fährt sie per SSH wieder herunter.

| | |
|---|---|
| Läuft gegen | homeserver |
| Tags | `cluster-power-manager`, `power-management` |
| Quellcode | [roles/cluster_power_manager](https://github.com/pkr-lab/capulus-ansible/tree/main/roles/cluster_power_manager) |

## Variablen

Defaults aus `defaults/main.yml`. Überschreiben in `group_vars/` oder `host_vars/`.

| Variable | Default | Bedeutung |
|---|---|---|
| `cluster_power_manager_enabled` | `true` | – |
| `cluster_power_manager_poll_seconds` | `30` | – |
| `cluster_power_manager_cpu_scale_up_threshold` | `60` | – |
| `cluster_power_manager_ram_scale_up_threshold` | `60` | – |
| `cluster_power_manager_scale_up_sustain_seconds` | `60` | – |
| `cluster_power_manager_scale_up_sustain_seconds_stage2` | `200` | – |
| `cluster_power_manager_cpu_scale_down_threshold` | `30` | – |
| `cluster_power_manager_ram_scale_down_threshold` | `30` | – |
| `cluster_power_manager_scale_down_sustain_seconds` | `900` | – |
| `cluster_power_manager_min_uptime_seconds` | `300` | – |
| `cluster_power_manager_ready_timeout_seconds` | `120` | – |
| `cluster_power_manager_ssh_user` | `ubuntu` | – |
| `cluster_power_manager_ssh_key_path` | `/etc/cluster-power-manager/id_ed25519` | – |
| `cluster_power_manager_state_dir` | `/var/lib/cluster-power-manager` | – |
| `cluster_power_manager_kubeconfig` | `/etc/rancher/k3s/k3s.yaml` | – |
| `cluster_power_manager_ntfy_host` | `ntfy.tech.homeserver` | – |
| `cluster_power_manager_ntfy_ip` | `{{ network_static_ip }}` | – |
| `cluster_power_manager_ntfy_topic` | `Home-Lab-System` | – |
| `cluster_power_manager_workers` | `[{ansible_host: '{{ hostvars.get(''worker-0'', {}).ansible_host \| default('''') }}', m…` | – |

## Ablauf

1. Warnen, falls eine Worker-MAC-Adresse noch nicht gesetzt ist
2. wakeonlan-Paket installieren (zum Senden von Magic Packets)
3. Config-/State-Verzeichnisse anlegen
4. SSH-Schlüsselpaar für Worker-Shutdown erzeugen (falls noch nicht vorhanden)
5. Berechtigungen des privaten Schlüssels absichern
6. Watchdog-Skript deployen
7. Konfiguration templaten
8. systemd-Unit deployen
9. cluster-power-manager aktivieren und starten
10. cluster-power-manager deaktivieren, falls cluster_power_manager_enabled false

## Handler

- Restart cluster power manager

## Templates

- `cluster-power-manager.service.j2`
- `config.env.j2`

## Dateien

- `cluster-power-manager.sh`

---

## Fehlersuche

Allgemeines Vorgehen bei Rollen: `ansible-playbook <playbook>.yml --limit <host> --tags <tag> --check --diff -vvv` zeigt Änderungen ohne sie anzuwenden (`make <ziel>-check` bei den Playbooks mit Dry-Run). Die Rollenübersicht mit Make-Zielen: [Rollen](index.md#fehlersuche-je-rolle).

```bash
sudo systemctl status cluster-power-manager
sudo journalctl -u cluster-power-manager -n 100 --no-pager
ls -l /etc/cluster-power-manager /var/lib/cluster-power-manager
kubectl get nodes
```

| Symptom | Ursache | Lösung |
|---|---|---|
| Worker wird nicht geweckt | Last unter den Schwellen (CPU und RAM je 60 % über `…scale_up_sustain_seconds`) oder WoL nicht aktiv | Log des Dienstes lesen, [Rolle wake_on_lan](wake_on_lan.md), BIOS und Switch prüfen |
| Worker fährt nicht herunter | Last über 30 % (Scale-down erst nach 900 s Dauer), `min_uptime_seconds` (300) nicht erreicht, SSH-Schlüssel nicht autorisiert | Schlüssel `/etc/cluster-power-manager/id_ed25519`, Rolle [cluster_power_manager_target](cluster_power_manager_target.md) |
| Worker bleibt nach dem Wecken `NotReady` | `ready_timeout_seconds` (120) abgelaufen | `journalctl -u k3s-agent` auf dem Worker, [k3s](../../capulus-core/b-kubernetes-gitops/b0000-k3s.md#fehlersuche) |
| Dienst startet nicht neu nach Änderung | `config.env` fehlerhaft | `systemctl cat cluster-power-manager`, `/etc/cluster-power-manager/config.env` |

Betrieb und Schwellen: [Cluster Power Manager](../2-betrieb-hardware/20020-cluster-power-manager.md#fehlerbehebung).

Allgemeine Fehlersuche: [Host-Troubleshooting](../a-betriebssystem/a0040-troubleshooting.md), [zentrale Fehlersuche](../../capulus-core/2-betrieb-hardware/20070-troubleshooting-zentral.md).

[Alle Rollen](index.md)
