# Rolle `libvirt_host`

KVM/libvirt, Bridge `br0`, legt `prod-vm` bzw. `entw-vm` an.

| | |
|---|---|
| Läuft gegen | homeserver, worker-1 |
| Tags | `libvirt-host`, `virtualization` |
| Quellcode | [roles/libvirt_host](https://github.com/pkr-lab/capulus-ansible/tree/main/roles/libvirt_host) |

## Variablen

Defaults aus `defaults/main.yml`. Überschreiben in `group_vars/` oder `host_vars/`.

| Variable | Default | Bedeutung |
|---|---|---|
| `libvirt_host_enabled` | `false` | – |
| `libvirt_host_configure_bridge` | `false` | – |
| `libvirt_host_bridge_name` | `br0` | – |
| `libvirt_host_bridge_interface` | `{{ ansible_default_ipv4.interface }}` | – |
| `libvirt_host_static_ip` | `{{ ansible_default_ipv4.address }}` | – |
| `libvirt_host_gateway` | `{{ ansible_default_ipv4.gateway }}` | – |
| `libvirt_host_prefix_length` | `{{ network_prefix_length \| default(24) }}` | – |
| `libvirt_host_dns` | `{{ network_dns \| default(['1.1.1.1', '8.8.8.8']) }}` | – |
| `libvirt_host_pool_dir` | `/var/lib/libvirt/images` | – |
| `libvirt_host_base_image_url` | `https://cloud-images.ubuntu.com/releases/resolute/release-20260927/ubuntu-26.04-server-…` | – |
| `libvirt_host_base_image_sha256` | `8800651811af9a85465ad1d552add729947bb16488dddb4a9b5305a3d97332b2` | – |
| `libvirt_host_base_image_name` | `ubuntu-26.04-server-cloudimg-amd64.img` | – |
| `libvirt_host_guests_on_boot` | `start` | – |
| `libvirt_host_guests_on_shutdown` | `shutdown` | – |
| `libvirt_host_guests_shutdown_timeout` | `300` | – |
| `libvirt_host_admin_ssh_public_key` | `` | – |
| `libvirt_host_vms` | `[]` | – |

## Ablauf

1. QEMU/KVM + libvirt-Pakete installieren
2. Konfliktenden dnsmasq-Snippet von libvirt-daemon-system entschaerfen
3. Pruefen, ob der Snippet existiert
4. Symlink auf das Paket-Original entfernen
5. Leeren Platzhalter anlegen (blockt die Neuanlage des Symlinks durch Upgrades)
6. Handler sofort ausfuehren (System-dnsmasq moeglichst schnell wieder hochbringen)
7. libvirtd aktivieren und starten
8. libvirt-guests - Verhalten beim Host-Shutdown und -Boot festlegen
9. libvirt-guests - Start nach dnsmasq und Tailscale ordnen (systemd-Drop-in)
10. Drop-in-Verzeichnis anlegen
11. Drop-in schreiben
12. Bridge-Netzwerk einrichten (nur wenn explizit aktiviert)
13. Sicherstellen, dass die Bridge-Parameter plausibel sind
14. Sicherstellen, dass die physische NIC auf dem Host existiert
15. Vorhandene Netplan-Configs finden
16. Alte Netplan-Configs entfernen
17. Bridge-Netplan-Konfiguration deployen
18. Netplan anwenden (Verbindung kann kurz abreissen)
19. Warten, bis der Host wieder erreichbar ist (jetzt ueber die Bridge)
20. SSH-Verbindung zuruecksetzen
21. UFW installiert?
22. UFW - Verkehr innerhalb der Bridge (LAN <-> VMs) erlauben
23. Storage-Pool-Verzeichnis anlegen
24. default-Storage-Pool definieren, falls noch nicht vorhanden
25. default-Storage-Pool starten + autostart aktivieren
26. Pruefen, ob das Basis-Cloud-Image schon vorhanden ist
27. Basis-Cloud-Image herunterladen (nur wenn es fehlt, bestehende Datei wird nie ersetzt)
28. Pro VM aus libvirt_host_vms
29. Pruefen, ob die VM-Domain bereits existiert
30. Eigenstaendige Disk fuer die VM aus dem Basis-Image anlegen (ueberspringt, falls sie schon existiert)
31. Disk auf die konfigurierte Groesse vergroessern
32. Pruefen, ob die Disk einer bestehenden VM noch ein Backing-File hat
33. Warnen, wenn die Disk einer bestehenden VM am Basis-Image haengt
34. cloud-init user-data + meta-data + network-config rendern
35. cloud-init Seed-ISO erzeugen
36. cloud-init-Temp-Dateien aufraeumen
37. VM-Domain per virt-install definieren + starten
38. Warten, bis die VM per SSH erreichbar ist (erster Boot + cloud-init dauert etwas)

## Handler

- System-dnsmasq nach Snippet-Entschaerfung neu starten
- systemd nach libvirt-guests-Drop-in neu laden

## Templates

- `bridge-netplan.yaml.j2`
- `cloud-init-meta-data.yaml.j2`
- `cloud-init-network-config.yaml.j2`
- `cloud-init-user-data.yaml.j2`

[Alle Rollen](index.md)
