# Rolle `common`

Basis-OS, UFW, Pakete, sysctl, chrony, Swap aus, optional statische IP.

| | |
|---|---|
| Läuft gegen | homeserver, prod-vm, entw-vm |
| Tags | `base`, `common` |
| Quellcode | [roles/common](https://github.com/pkr-lab/capulus-ansible/tree/main/roles/common) |

## Variablen

Defaults aus `defaults/main.yml`. Überschreiben in `group_vars/` oder `host_vars/`.

| Variable | Default | Bedeutung |
|---|---|---|
| `common_ufw_lan_allow_all` | `true` | – |
| `common_ufw_tailnet_allow_all` | `true` | – |
| `common_ufw_k3s_api_sources` | `['{{ local_subnet }}']` | – |
| `common_ufw_trusted_hosts` | `[]` | – |
| `common_ufw_lan_services` | `[]` | – |

## Ablauf

1. Set system hostname
2. Update /etc/hosts with hostname
3. Update apt cache
4. Upgrade all packages to latest (dist-upgrade)
5. Install required system packages
6. Enable unattended-upgrades (security patches between Ansible runs)
7. Configure unattended-upgrades to also pull "updates" and clean up
8. Ensure unattended-upgrades service is enabled
9. Set system timezone
10. Ensure chrony is enabled and running
11. Check for active swap devices
12. Disable swap immediately
13. Remove swap entries from /etc/fstab
14. Ensure swap is not re-enabled by systemd
15. Create modules-load.d config for k3s/Kubernetes
16. Load required kernel modules for k3s/Kubernetes
17. Configure sysctl parameters for Kubernetes
18. Allow SSH (port 22) before any other firewall changes
19. Set UFW default incoming policy to deny
20. Set UFW default outgoing policy to allow
21. Allow HTTP (port 80) for Traefik ingress
22. Allow HTTPS (port 443)
23. k3s-api (6443) nur von den freigegebenen quellen erlauben
24. Remove ArgoCD NodePort 30080 rules (NodePort entfällt, ArgoCD läuft über den Ingress)
25. Remove unrestricted ArgoCD NodePort 30080 rule (manuell gesetzt, jetzt auf LAN/Tailnet begrenzt)
26. Remove useless ArgoCD NodePort 30443 rule (Server spricht dort Klartext, kein TLS)
27. Allow Tailscale WireGuard (UDP 41641)
28. flannel-vxlan (udp 8472) und kubelet (10250) nur aus dem lan erlauben
29. offene k3s-regeln für alle quellen entfernen (6443, 10250, 8472)
30. Allow all traffic from k3s pod CIDR
31. Allow all traffic from k3s service CIDR
32. Allow all traffic from local LAN subnet
33. Allow all traffic from Tailscale network
34. infrastruktur-hosts mit vollzugriff erlauben
35. dienste für das lan freigeben
36. pauschale lan-regel entfernen
37. pauschale tailnet-regel entfernen
38. Enable UFW
39. Verify UFW is active
40. Display UFW status
41. Configure static IP via Netplan
42. Find existing Netplan configs
43. Remove existing Netplan DHCP configs
44. Deploy static Netplan configuration
45. Apply Netplan configuration (connection will briefly drop)
46. Wait for server to be reachable at the new static IP
47. Check whether a reboot is required (kernel/libc updates)
48. Reboot the server because /var/run/reboot-required is set
49. Reboot required but auto_reboot_if_required is false — warn the operator
50. Reboot required but Semaphore is running this task — warn instead of self-killing

## Handler

- Restart chrony
- Reload sysctl

## Templates

- `netplan-static.yaml.j2`

[Alle Rollen](index.md)
