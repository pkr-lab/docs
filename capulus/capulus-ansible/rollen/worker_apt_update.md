# Rolle `worker_apt_update`

apt-Update der Worker (haben keine `common`-Rolle).

| | |
|---|---|
| Läuft gegen | worker-0, worker-1 |
| Tags | `apt-update`, `updates` |
| Quellcode | [roles/worker_apt_update](https://github.com/pkr-lab/capulus-ansible/tree/main/roles/worker_apt_update) |

## Variablen

Defaults aus `defaults/main.yml`. Überschreiben in `group_vars/` oder `host_vars/`.

| Variable | Default | Bedeutung |
|---|---|---|
| `worker_apt_update_enabled` | `true` | – |
| `worker_apt_update_k3s_node_name` | `{{ k3s_agent_hostname }}` | – |
| `worker_apt_update_kubeconfig` | `/etc/rancher/k3s/k3s.yaml` | – |
| `worker_apt_update_reboot_timeout_seconds` | `300` | – |
| `worker_apt_update_post_reboot_delay_seconds` | `15` | – |
| `worker_apt_update_dns_fallback_enabled` | `false` | – |
| `worker_apt_update_dns_fallback_interface` | `eno1` | – |
| `worker_apt_update_dns_fallback_servers` | `[1.1.1.1, 8.8.8.8]` | – |

## Ablauf

1. Statische Fallback-Nameserver setzen (Netplan-Drop-in)
2. Fallback-Nameserver-Konfiguration deployen
3. Handler sofort ausfuehren (DNS muss vor dem apt-Update unten stehen)
4. Update apt cache
5. Upgrade all packages to latest (dist-upgrade)
6. Check whether a reboot is required
7. Reboot-erforderlich -- Node vor dem Neustart cordon/drain (auf homeserver)
8. kubectl cordon {{ worker_apt_update_k3s_node_name }}
9. kubectl drain {{ worker_apt_update_k3s_node_name }}
10. Reboot-erforderlich -- {{ inventory_hostname }} jetzt automatisch neu starten
11. Reboot-erforderlich -- Abschluss melden (für den nightly_worker_wake-Bericht)

## Handler

- Netplan-Konfiguration anwenden

## Templates

- `dns-fallback.yaml.j2`

---

## Fehlersuche

Allgemeines Vorgehen bei Rollen: `ansible-playbook <playbook>.yml --limit <host> --tags <tag> --check --diff -vvv` zeigt Änderungen ohne sie anzuwenden (`make <ziel>-check` bei den Playbooks mit Dry-Run). Die Rollenübersicht mit Make-Zielen: [Rollen](index.md#fehlersuche-je-rolle).

```bash
sudo journalctl -u unattended-upgrades -n 50 --no-pager
kubectl get nodes
sudo apt update && apt list --upgradable
```

| Symptom | Ursache | Lösung |
|---|---|---|
| Update läuft nachts nicht | Teil des Zyklus von [nightly_worker_wake](nightly_worker_wake.md) | [Nightly Worker Update](../2-betrieb-hardware/20030-nightly-worker-update.md#fehlerbehebung) |
| Worker kommt nach dem Reboot nicht zurück | `reboot_timeout_seconds` (300) überschritten | Konsole am Gerät, [k3s_agent](k3s_agent.md) |
| DNS-Fehler beim apt-Update | Pi-hole/dnsmasq nicht erreichbar | `worker_apt_update_dns_fallback_enabled: true` (Fallback `1.1.1.1`, `8.8.8.8`), [DNS](../../capulus-core/c-netzwerk-dns/c0000-dns-architecture.md#fehlersuche) |
| Node steht danach auf `SchedulingDisabled` | Vor einem nötigen Neustart führt die Rolle auf dem Homeserver `kubectl cordon` und `kubectl drain` (Timeout 120 s, Fehler werden ignoriert) für `worker_apt_update_k3s_node_name` aus | `kubectl get nodes`, bei Bedarf `kubectl uncordon <node>` |

Allgemeine Fehlersuche: [Host-Troubleshooting](../a-betriebssystem/a0040-troubleshooting.md), [zentrale Fehlersuche](../../capulus-core/2-betrieb-hardware/20070-troubleshooting-zentral.md).

[Alle Rollen](index.md)
