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

[Alle Rollen](index.md)
