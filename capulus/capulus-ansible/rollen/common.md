# Rolle `common`

Basis-OS, UFW, Pakete, sysctl, chrony, Swap aus, optional statische IP. Ein automatischer Neustart (`auto_reboot_if_required`) läuft auf k3s-Servern nur, wenn [k3s_reboot_guard](k3s_reboot_guard.md) ihn freigibt.

| | |
|---|---|
| Läuft gegen | controlplane-0, prod-vm, entw-vm |
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
| `common_disabled_services` | `[ModemManager.service]` | Dienste, die gestoppt und deaktiviert werden; nicht vorhandene Einheiten werden übersprungen |

## Ablauf

1. hostnamen setzen
2. hostnamen in /etc/hosts eintragen
3. apt-cache aktualisieren
4. alle pakete aktualisieren (dist-upgrade)
5. benötigte systempakete installieren
6. unattended-upgrades einschalten (sicherheitsupdates zwischen den ansible-läufen)
7. unattended-upgrades auch für "updates" und zum aufräumen einrichten
8. sicherstellen dass der dienst unattended-upgrades aktiviert ist
9. zeitzone setzen
10. chrony aktivieren und starten
11. kubernetes-voraussetzungen herstellen (`tasks/k8s_prerequisites.yml`: Swap aus und maskiert, Kernel-Module `br_netfilter`/`overlay`, sysctl in `/etc/sysctl.d/99-k3s.conf`; auch von `k3s_agent` genutzt)
12. ssh (port 22) vor allen anderen firewall-änderungen erlauben
13. ufw-standardrichtlinie für eingehenden verkehr auf deny setzen
14. ufw-standardrichtlinie für ausgehenden verkehr auf allow setzen
15. http (port 80) für den traefik-ingress erlauben
16. https (port 443) erlauben
17. k3s-api (6443) nur von den freigegebenen quellen erlauben
18. regeln für den argocd-nodeport 30080 entfernen (nodeport entfällt, argocd läuft über den ingress)
19. offene regel für den argocd-nodeport 30080 entfernen (manuell gesetzt, jetzt auf lan/tailnet begrenzt)
20. nutzlose regel für den argocd-nodeport 30443 entfernen (server spricht dort klartext, kein tls)
21. tailscale-wireguard (udp 41641) erlauben
22. flannel-vxlan (udp 8472) und kubelet (10250) nur aus dem lan erlauben
23. offene k3s-regeln für alle quellen entfernen (6443, 10250, 8472)
24. gesamten verkehr aus dem pod-cidr von k3s erlauben
25. gesamten verkehr aus dem service-cidr von k3s erlauben
26. gesamten verkehr aus dem lokalen lan erlauben
27. gesamten verkehr aus dem tailscale-netz erlauben
28. infrastruktur-hosts mit vollzugriff erlauben
29. dienste für das lan freigeben
30. pauschale lan-regel entfernen
31. pauschale tailnet-regel entfernen
32. ufw aktivieren
33. prüfen dass ufw aktiv ist
34. ufw-status anzeigen
35. statische ip per netplan einrichten
36. vorhandene netplan-konfigurationen suchen
37. vorhandene netplan-dhcp-konfigurationen entfernen
38. statische netplan-konfiguration ablegen
39. netplan-konfiguration anwenden (verbindung bricht kurz ab)
40. warten bis der server unter der neuen statischen ip erreichbar ist
41. prüfen ob ein reboot nötig ist (kernel-/libc-updates)
42. server neu starten, weil /var/run/reboot-required gesetzt ist
43. reboot nötig, aber auto_reboot_if_required ist false — betreiber warnen
44. reboot nötig, aber semaphore führt diesen task aus — warnen statt sich selbst abzuschießen
45. nicht benötigte dienste stoppen und deaktivieren

## Handler

- chrony neu starten
- sysctl neu laden

## Templates

- `netplan-static.yaml.j2`

---

## Fehlersuche

Allgemeines Vorgehen bei Rollen: `ansible-playbook <playbook>.yml --limit <host> --tags <tag> --check --diff -vvv` zeigt Änderungen ohne sie anzuwenden (`make <ziel>-check` bei den Playbooks mit Dry-Run). Die Rollenübersicht mit Make-Zielen: [Rollen](index.md#fehlersuche-je-rolle).

```bash
sudo ufw status verbose
sudo systemctl status chrony unattended-upgrades
sudo sysctl net.bridge.bridge-nf-call-iptables net.ipv4.ip_forward
swapon --show; cat /var/run/reboot-required
```

| Symptom | Ursache | Lösung |
|---|---|---|
| Verbindung bricht nach dem Playbook ab | UFW blockiert (SSH-Regel kommt vor allen anderen, aber Quelle/Subnetz falsch) oder Netplan-Umstellung | Konsole am Gerät: `sudo ufw status`, `local_subnet` prüfen ([Troubleshooting](../a-betriebssystem/a0040-troubleshooting.md)) |
| Dienst aus dem LAN nicht erreichbar | Port nicht in `common_ufw_lan_services` oder `common_ufw_lan_allow_all: false` | Variable ergänzen, Rolle ausrollen |
| k3s-API (6443) von außen nicht erreichbar | `common_ufw_k3s_api_sources` beschränkt auf `local_subnet` | Quellen ergänzen |
| Statische IP greift nicht oder Host verschwindet | Netplan-Konfiguration fehlerhaft, `wait_for` auf neue IP läuft ab | Konsole am Gerät, `sudo netplan try`, `/etc/netplan/` |
| `reboot required`-Warnung im Lauf | Kernel- oder libc-Update; `auto_reboot_if_required` ist aus oder Lauf kommt von Semaphore | manuell neu starten (Reihenfolge bei Workern beachten, [Cluster Power Manager](../2-betrieb-hardware/20020-cluster-power-manager.md)) |
| k3s startet wegen Swap nicht | Swap war aktiv | Rolle entfernt Swap aus `/etc/fstab`, `swapon --show` prüfen |

Allgemeine Fehlersuche: [Host-Troubleshooting](../a-betriebssystem/a0040-troubleshooting.md), [zentrale Fehlersuche](../../capulus-core/2-betrieb-hardware/20070-troubleshooting-zentral.md).

[Alle Rollen](index.md)
