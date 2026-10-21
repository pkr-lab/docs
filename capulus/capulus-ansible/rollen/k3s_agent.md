# Rolle `k3s_agent`

Cluster-Beitritt (TECH) per Join-Token vom Control-Plane.

| | |
|---|---|
| Läuft gegen | worker-0, worker-1 |
| Tags | – |
| Quellcode | [roles/k3s_agent](https://github.com/pkr-lab/capulus-ansible/tree/main/roles/k3s_agent) |

## Variablen

Defaults aus `defaults/main.yml`. Überschreiben in `group_vars/` oder `host_vars/`.

| Variable | Default | Bedeutung |
|---|---|---|
| `k3s_installer_url` | `https://raw.githubusercontent.com/k3s-io/k3s/2977c525a2e7a487886107ce4df43630ae9b03b2/i…` | – |
| `k3s_installer_sha256` | `e5cc3b3d9dfc1662c2d9be6da5abc9a4cd317d6abc3a5ffc02e3dd3248207fee` | – |
| `k3s_agent_ufw_enable` | `false` | – |
| `k3s_agent_ufw_trusted_hosts` | `[]` | – |
| `k3s_agent_ufw_vm_bridge` | `` | – |

## Ablauf

1. nfs-common installieren
2. Set system hostname
3. Update /etc/hosts with new hostname
4. Disable swap immediately
5. Remove swap entries from /etc/fstab
6. Create modules-load.d config for k3s
7. Load required kernel modules
8. Configure sysctl parameters for k3s agent
9. Allow kubelet API (port 10250) from k3s server
10. flannel-vxlan (udp 8472) nur aus dem lan erlauben
11. offene regeln für flannel-vxlan und die api (6443) auf dem agent entfernen
12. Allow all traffic from k3s pod CIDR
13. Allow all traffic from k3s service CIDR
14. infrastruktur-hosts vollständig erlauben
15. veraltete offene regeln entfernen
16. ssh aus dem lan erlauben
17. tailscale-wireguard erlauben
18. bridge für virtuelle maschinen weiterleiten
19. ufw aktivieren
20. Read k3s node-token from server
21. Set k3s node-token fact
22. Create /etc/rancher/k3s directory
23. Template k3s agent configuration
24. Check if k3s is already installed
25. Get current k3s version (if installed)
26. Decide whether to (re)run the k3s installer
27. Download k3s install script
28. Install or upgrade k3s agent
29. Remove k3s install script
30. Enable and start k3s-agent service
31. Wait for {{ k3s_agent_hostname }} node to become Ready
32. Display cluster node status
33. Show all cluster nodes

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
| Worker tritt nicht bei (`Unauthorized`, Timeout zur Server-URL) | Join-Token veraltet (Control-Plane neu aufgesetzt) oder Server nicht erreichbar (UFW, Netz, Tailscale) | Rolle erneut ausrollen: sie liest den Token neu von `/var/lib/rancher/k3s/server/node-token` des Servers (`make worker-0`/`worker-1`) |
| Pods auf dem Worker starten nicht (NFS) | `nfs-common` fehlt oder NAS nicht erreichbar | Paket installieren (die Rolle tut das), [NAS-Speicher](../../capulus-core/2-betrieb-hardware/20000-nas-storage.md#fehlerbehebung) |
| Kubelet nicht erreichbar vom Server (`logs`, `exec` schlagen fehl) | UFW blockiert Port 10250 vom Server | Regel „kubelet API von k3s-Server“ (die Rolle setzt sie), UFW-Status prüfen |
| Netz zwischen Nodes gestört | VXLAN (UDP 8472) nur aus dem LAN erlaubt | Firewall und Flannel prüfen ([k3s](../../capulus-core/b-kubernetes-gitops/b0000-k3s.md#fehlersuche)) |
| Worker ist nachts aus | gewollt (Cluster Power Manager) | [Cluster Power Manager](cluster_power_manager.md) |

Allgemeine Fehlersuche: [Host-Troubleshooting](../a-betriebssystem/a0040-troubleshooting.md), [zentrale Fehlersuche](../../capulus-core/2-betrieb-hardware/20070-troubleshooting-zentral.md).

[Alle Rollen](index.md)
