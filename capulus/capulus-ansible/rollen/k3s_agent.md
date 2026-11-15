# Rolle `k3s_agent`

Cluster-Beitritt (TECH) per Join-Token vom Control-Plane.

| | |
|---|---|
| Läuft gegen | worker-1 (controlplane-1 nutzt nur die Voraussetzungen und die Firewall dieser Rolle, siehe [400a0](../../capulus-core/4-planung/400a0-controlplane-ha-zwei-server.md)) |
| Tags | – |
| Quellcode | [roles/k3s_agent](https://github.com/pkr-lab/capulus-ansible/tree/main/roles/k3s_agent) |

## Variablen

Der Server, bei dem der Agent beitritt, kommt aus `k3s_server_host` (`controlplane-0`, gesetzt in `group_vars/k3s_servers.yml` und `group_vars/k3s_agents.yml`, nicht im Vault). Der k3s-Nodename ist `k3s_agent_hostname` aus den `host_vars`.

Defaults aus `defaults/main.yml`. Überschreiben in `group_vars/` oder `host_vars/`. `k3s_installer_url` und `k3s_installer_sha256` stehen gemeinsam für `k3s` und `k3s_agent` in `group_vars/all.yml`, `k3s_version` und `k3s_server_host` des TECH-Clusters kommen aus `k3s_cluster_version` und `k3s_cluster_server_host` (ebenfalls `group_vars/all.yml`).

| Variable | Default | Bedeutung |
|---|---|---|
| `k3s_agent_ufw_enable` | `false` | – |
| `k3s_agent_ufw_trusted_hosts` | `[]` | – |
| `k3s_agent_ufw_vm_bridge` | `` | – |

## Aufteilung der Tasks

Die Voraussetzungen (`tasks/prerequisites.yml`: nfs-common, Hostname, Swap, Module, sysctl) und die Host-Firewall (`tasks/ufw.yml`, nur mit `k3s_agent_ufw_enable: true`) liegen in eigenen Dateien; Swap, Module und sysctl kommen aus `common/tasks/k8s_prerequisites.yml`. Die Rolle `k3s` nutzt sie im Modus `join` für controlplane-1 per `import_role … tasks_from`, statt sie zu kopieren. Die Variablen `k3s_agent_*` behalten ihren Namen, auch wenn sie auf einem Server gelten (`k3s_agent_hostname` ist dort der OS-Hostname).

## Ablauf

1. nfs-common installieren
2. Set system hostname
3. Update /etc/hosts with new hostname
4. kubernetes-voraussetzungen aus der rolle common übernehmen (`common/tasks/k8s_prerequisites.yml`)
5. Allow kubelet API (port 10250) from k3s server
6. flannel-vxlan (udp 8472) nur aus dem lan erlauben
7. offene regeln für flannel-vxlan und die api (6443) auf dem agent entfernen
8. Allow all traffic from k3s pod CIDR
9. Allow all traffic from k3s service CIDR
10. infrastruktur-hosts vollständig erlauben
11. veraltete offene regeln entfernen
12. ssh aus dem lan erlauben
13. tailscale-wireguard erlauben
14. bridge für virtuelle maschinen weiterleiten
15. ufw aktivieren
16. Read k3s node-token from server
17. Set k3s node-token fact
18. Create /etc/rancher/k3s directory
19. Template k3s agent configuration
20. Check if k3s is already installed
21. Get current k3s version (if installed)
22. Decide whether to (re)run the k3s installer
23. Download k3s install script
24. Install or upgrade k3s agent
25. Remove k3s install script
26. Enable and start k3s-agent service
27. Wait for {{ k3s_agent_hostname }} node to become Ready
28. Display cluster node status
29. Show all cluster nodes

## Handler

- Restart k3s-agent

## Templates

- `k3s-agent-config.yaml.j2`

---

## Fehlersuche

Allgemeines Vorgehen bei Rollen: `ansible-playbook <playbook>.yml --limit <host> --tags <tag> --check --diff -vvv` zeigt Änderungen ohne sie anzuwenden (`make <ziel>-check` bei den Playbooks mit Dry-Run). Die Rollenübersicht mit Make-Zielen: [Rollen](index.md#fehlersuche-je-rolle).

```bash
sudo systemctl status k3s-agent
sudo journalctl -u k3s-agent -n 100 --no-pager
sudo cat /etc/rancher/k3s/config.yaml
kubectl get nodes -o wide              # auf dem Homeserver
```

| Symptom | Ursache | Lösung |
|---|---|---|
| Worker tritt nicht bei (`Unauthorized`, Timeout zur Server-URL) | Join-Token veraltet (Control-Plane neu aufgesetzt) oder Server nicht erreichbar (UFW, Netz, Tailscale) | Rolle erneut ausrollen: sie liest den Token neu von `/var/lib/rancher/k3s/server/node-token` des Servers (`make controlplane-1`/`worker-1`) |
| Pods auf dem Worker starten nicht (NFS) | `nfs-common` fehlt oder NAS nicht erreichbar | Paket installieren (die Rolle tut das), [NAS-Speicher](../../capulus-core/2-betrieb-hardware/20000-nas-storage.md#fehlerbehebung) |
| Kubelet nicht erreichbar vom Server (`logs`, `exec` schlagen fehl) | UFW blockiert Port 10250 vom Server | Regel „kubelet API von k3s-Server“ (die Rolle setzt sie), UFW-Status prüfen |
| Netz zwischen Nodes gestört | VXLAN (UDP 8472) nur aus dem LAN erlaubt | Firewall und Flannel prüfen ([k3s](../../capulus-core/b-kubernetes-gitops/b0000-k3s.md#fehlersuche)) |
| Worker ist nachts aus | gewollt (Cluster Power Manager) | [Cluster Power Manager](cluster_power_manager.md) |

Allgemeine Fehlersuche: [Host-Troubleshooting](../a-betriebssystem/a0040-troubleshooting.md), [zentrale Fehlersuche](../../capulus-core/2-betrieb-hardware/20070-troubleshooting-zentral.md).

[Alle Rollen](index.md)
