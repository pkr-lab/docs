# Ansible-Rollen

Übersicht aller Rollen von capulus-ansible. Je Rolle gibt es eine Seite mit Variablen, Ablauf, Handlern, Templates und Dateien. Die Seiten werden aus `defaults/main.yml` und `tasks/` erzeugt und beschreiben den Stand des Repos.

| Rolle | Läuft gegen | Zweck |
|---|---|---|
| [`alamos_kiosk`](alamos_kiosk.md) | Raspberry Pis | Chromium-Kiosk und Heartbeat-Timer für den Alarmmonitor |
| [`alert_caller`](alert_caller.md) | externe VM `alert-caller-vm`, Vereinsheim-Pi | Telefon-Alarm: Webhook-Server, Piper-Ansage, baresip-Anruf |
| [`argocd`](argocd.md) | homeserver (Hub), entw-vm | ArgoCD per Helm, Bootstrap-ApplicationSet und -Projects, Namespace-Labels, NetworkPolicies |
| [`banana_pi_kiosk`](banana_pi_kiosk.md) | Banana Pi | Chromium-Kiosk, Heartbeat-Timer und WoL-Agent |
| [`cluster_power_manager`](cluster_power_manager.md) | homeserver | weckt Worker per WoL, fährt sie per SSH wieder herunter |
| [`cluster_power_manager_target`](cluster_power_manager_target.md) | worker-0, worker-1 | autorisiert den Shutdown-Schlüssel, beschränkt auf `poweroff` |
| [`common`](common.md) | homeserver, prod-vm, entw-vm | Basis-OS, UFW, Pakete, sysctl, chrony, Swap aus, optional statische IP |
| [`crowdsec`](crowdsec.md) | homeserver | Brute-Force-Schutz für SSH und Traefik-Logs |
| [`cups_print_server`](cups_print_server.md) | homeserver | Druckserver, QPDL-Treiber-Build, zweite Warteschlange |
| [`disable_eee`](disable_eee.md) | homeserver, worker-0, worker-1 | Energy Efficient Ethernet dauerhaft deaktivieren (Realtek-NIC-Link-Drops) |
| [`dnsmasq`](dnsmasq.md) | homeserver | Split-DNS `*.homeserver` (inkl. `*.dev.homeserver` → ENTW und PROD-Einzelhosts), Weiterleitung an Pi-hole |
| [`journal_upload`](journal_upload.md) | alle Ansible-Hosts | journald → VictoriaLogs |
| [`k3s`](k3s.md) | homeserver, prod-vm, entw-vm | Kubernetes-Server und Helm, je Cluster eigene CIDRs |
| [`k3s_agent`](k3s_agent.md) | worker-0, worker-1 | Cluster-Beitritt (TECH) per Join-Token vom Control-Plane |
| [`libvirt_host`](libvirt_host.md) | homeserver, worker-1 | KVM/libvirt, Bridge `br0`, legt `prod-vm` bzw. `entw-vm` an |
| [`nightly_worker_wake`](nightly_worker_wake.md) | homeserver | 01:00-Timer: Worker wecken, Update anstoßen, wieder herunterfahren |
| [`node_exporter`](node_exporter.md) | prod-vm, entw-vm, Banana Pi, Kiosks | Host-Metriken für Nicht-k3s-Hosts bzw. VMs |
| [`ntfy_publisher`](ntfy_publisher.md) | alle Knoten und Kiosks | Meldungen der Watchdogs und Host-Skripte per HTTPS an ntfy, mit Root-CA und Publisher-Token |
| [`power_agent`](power_agent.md) | homeserver | HTTP-API für Helligkeit, Wake und Shutdown aus der iOS-App |
| [`resource_watchdog`](resource_watchdog.md) | alle Knoten und Kiosks | Selbst-Abschaltung bei Dauerlast |
| [`semaphore_bootstrap`](semaphore_bootstrap.md) | homeserver | Projekte, Inventories, Templates und Zeitpläne in Semaphore per REST-API |
| [`semaphore_secrets`](semaphore_secrets.md) | homeserver | Bootstrap-Secret für den Semaphore-Pod |
| [`semaphore_targets`](semaphore_targets.md) | alle verwalteten Hosts | SSH-Pubkey von Semaphore in `authorized_keys` |
| [`smartctl_exporter`](smartctl_exporter.md) | homeserver, worker-0/-1 | S.M.A.R.T.-Werte der physischen Platten |
| [`ssh_hardening`](ssh_hardening.md) | alle verwalteten Hosts | Schlüsselbasierter SSH-Zugang (Passwort-Login aus) |
| [`tailscale`](tailscale.md) | homeserver, worker-0/-1, xibosignage-Displays, Banana Pi, alert-caller-VM | WireGuard-Mesh-VPN, Auth-Key aus Ansible Vault; nur der homeserver ist Subnet-Router |
| [`thermal_watchdog`](thermal_watchdog.md) | alle Knoten und Kiosks | Selbst-Abschaltung bei Übertemperatur |
| [`vaultwarden_restore`](vaultwarden_restore.md) | homeserver | Wiederherstellung der Vaultwarden-Daten aus dem NAS-Backup |
| [`vmagent`](vmagent.md) | Tailscale-only-Hosts (Banana Pi) | Metriken-Push per remote-write |
| [`wake_on_lan`](wake_on_lan.md) | worker-0, worker-1 | `ethtool wol g` bei jedem Boot |
| [`wireguard_backup`](wireguard_backup.md) | homeserver | Notfall-Tunnel, falls die Tailscale-Control-Plane ausfällt |
| [`worker_apt_update`](worker_apt_update.md) | worker-0, worker-1 | apt-Update der Worker (haben keine `common`-Rolle) |
| [`xibo_kiosk`](xibo_kiosk.md) | xibosignage-Displays | NFS-Mount, Manifest-Timer, Chromium-Slideshow-Kiosk |

Hintergründe und Fallstricke: [60020 Ansible-Rollen](../6-hintergruende/60020-ansible-rollen.md).
