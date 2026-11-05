# Rolle `k3s`

Kubernetes-Server und Helm, je Cluster eigene CIDRs. Seit der Umstellung des TECH-Clusters auf zwei Control-Planes kennt die Rolle drei Modi (`k3s_mode`).

| Modus | Hosts | Bedeutung |
|---|---|---|
| `single` (Standard) | prod-vm, entw-vm | ein Server mit SQLite, wie bisher |
| `init` | controlplane-0 | erster etcd-Server (`cluster-init: true`). Auf einem bestehenden SQLite-Server konvertiert ein Neustart den Datastore zu embedded etcd, **nicht umkehrbar** |
| `join` | controlplane-1 | weiterer Server: `server:` und `token:` (Server-Token wird zur Laufzeit von `k3s_primary` gelesen) |

Im Modus `init` und `join` ist `k3s_version` Pflicht (alle Server brauchen dieselbe Version). Die Rolle schreibt die Config, startet k3s aber nicht neu: Der Neustart, der `cluster-init` oder eine geänderte `tls-san`-Liste wirksam macht, ist ein bewusster manueller Schritt ([Runbook in 400a0](../../capulus-core/4-planung/400a0-controlplane-ha-zwei-server.md#ap3-etcd-und-server-join)).

| | |
|---|---|
| Läuft gegen | controlplane-0, controlplane-1, prod-vm, entw-vm |
| Tags | – |
| Quellcode | [roles/k3s](https://github.com/pkr-lab/capulus-ansible/tree/main/roles/k3s) |

## Variablen

Defaults aus `defaults/main.yml`. Überschreiben in `group_vars/` oder `host_vars/`.

| Variable | Default | Bedeutung |
|---|---|---|
| `k3s_installer_url` | `https://raw.githubusercontent.com/k3s-io/k3s/2977c525a2e7a487886107ce4df43630ae9b03b2/i…` | – |
| `k3s_installer_sha256` | `e5cc3b3d9dfc1662c2d9be6da5abc9a4cd317d6abc3a5ffc02e3dd3248207fee` | – |
| `k3s_node_name` | `{{ inventory_hostname }}` | Nodename in k3s (`node-name`). `controlplane-0` setzt `homeserver` in den `host_vars`, weil ein Nodename nicht änderbar ist |
| `k3s_mode` | `single` | `single`, `init` oder `join`, siehe oben |
| `k3s_primary_host` | erster Host der Gruppe `k3s_primary` | Quelle von Join-Token und `server:`-URL |
| `k3s_resolv_nameservers` | `[]` (auf den k3s-Servern `network_static_ip`, `1.1.1.1`, `8.8.8.8`) | Nameserver für kubelet und CoreDNS. Nicht leer: Die Rolle schreibt `/etc/rancher/k3s/resolv.conf` und setzt `resolv-conf` darauf. Leer: `/run/systemd/resolve/resolv.conf` wie bisher (prod-vm, entw-vm) |
| `k3s_resolv_search` | `[]` (auf den k3s-Servern `tail532fb5.ts.net`, `andernach.dlrg-jugend.de`) | `search`-Zeile der eigenen `resolv.conf`. Ohne sie verlören Pods die Such-Domains des Hosts, die kubelet sonst aus `systemd-resolved` übernimmt |
| `k3s_resolv_conf_path` | abgeleitet | Pfad, der als `resolv-conf` in die Config kommt |
| `k3s_node_taints` | `[]` | Taints, die der Node bei der Registrierung bekommt (`node-taint`). controlplane-1 setzt `capulus.homeserver/protected=true:PreferNoSchedule`. Ein Taint in der Config wirkt nur bei der Erstregistrierung |
| `k3s_tls_san_extra` | `[]` | zusätzliche Einträge für `tls-san` (später die Tailscale-Adresse von ControlPlane-2). Die Liste enthält sonst IP, Inventarname und Nodename aller Server |
| `k3s_etcd_heartbeat_interval_ms` | `500` | etcd-Heartbeat, muss auf allen Mitgliedern gleich sein (Begründung in [60020](../6-hintergruende/60020-ansible-rollen.md#k3s-embedded-etcd)) |
| `k3s_etcd_election_timeout_ms` | `5000` | etcd-Election-Timeout, gleiche Werte auf allen Mitgliedern |
| `k3s_etcd_expose_metrics` | `true` | etcd-Metriken auf Port 2381 |
| `k3s_etcd_snapshot_cron` | `0 */6 * * *` | Zeitplan der lokalen Snapshots |
| `k3s_etcd_snapshot_retention` | `28` | lokale Snapshots **pro Server** (7 Tage) |
| `k3s_etcd_snapshot_compress` | `true` | Snapshots komprimieren |
| `k3s_etcd_min_members` | `2` bei `join`, sonst `1` | Mindestzahl etcd-Mitglieder, auf die die Rolle nach dem Start wartet |
| `k3s_snapshot_copy_enabled` | `false` (auf den k3s-Servern `true`) | richtet Mount und Timer ein, der Snapshots auf die NAS kopiert. Erst aktivieren, wenn das Verzeichnis auf der NAS existiert |
| `k3s_snapshot_nfs_server`, `k3s_snapshot_nfs_export` | leer (auf den k3s-Servern `192.168.178.97`, `/volume1/k8s-storage/k3s-etcd-snapshots`) | NAS und Verzeichnis für die Kopie |
| `k3s_snapshot_mount_point` | `/mnt/k3s-etcd-snapshots` | Mount-Punkt der Freigabe |
| `k3s_snapshot_copy_calendar` | `*-*-* 00,06,12,18:20:00` | Zeitpunkt der Kopie (20 Minuten nach dem Snapshot) |
| `k3s_snapshot_copy_keep_days` | `14` | Aufbewahrung auf der NAS |
| `k3s_ntfy_host`, `k3s_ntfy_ip`, `k3s_ntfy_topic` | `ntfy.tech.homeserver`, `{{ network_static_ip }}`, `Home-Lab-System` | Ziel der Fehlermeldung des Kopier-Skripts (nutzt Zugangsdaten und CA der Rolle `ntfy_publisher`) |
| `k3s_ufw_enable` | `true` | legt die Server-Regeln an (Modi `init` und `join`) |
| `k3s_ufw_api_sources` | LAN und `100.64.0.0/10` | Quellen für Port 6443 |
| `k3s_ufw_etcd_extra_peers` | `[]` | zusätzliche etcd-Peers (Port 2379–2380), später ControlPlane-2 |
| `k3s_helm_release` | `v3.22.1` | – |
| `k3s_helm_arch` | `{{ 'arm64' if ansible_architecture == 'aarch64' else 'amd64' }}` | – |

## Ablauf (Modus `join`, Ergänzungen)

Vor den bekannten Schritten: Version prüfen, abbrechen wenn noch `k3s-agent.service` existiert, Voraussetzungen aus der Rolle `k3s_agent` (nfs-common, Hostname, Swap aus, Module, sysctl), Server-Firewallregeln (6443 aus den freigegebenen Quellen, 2379–2380 nur von den anderen Servern), dann die Host-Firewall aus `k3s_agent` (löscht erst danach die offene 6443-Regel, damit niemand ausgesperrt wird), Join-Token lesen. Nach dem Start prüft die Rolle `/readyz/etcd`, zählt die etcd-Mitglieder und richtet optional die Snapshot-Kopie ein.

## Ablauf

1. Create /etc/rancher/k3s directory
2. Template kube-apiserver audit policy
3. Create /var/log/k3s directory for audit log output
4. Template k3s server configuration
5. Check if k3s is already installed
6. Get current k3s version (if installed)
7. Decide whether to (re)run the k3s installer
8. Download k3s install script
9. Install or upgrade k3s (pinned version or latest in channel)
10. Remove k3s install script
11. Enable and start k3s service
12. Wait for k3s API server to be reachable (port 6443)
13. Wait for k3s node to be in Ready state
14. Display node status
15. Create .kube directory for ubuntu user
16. Copy k3s kubeconfig to ubuntu user
17. Set KUBECONFIG in /etc/environment (system-wide)
18. Set KUBECONFIG in ubuntu user's .bashrc
19. Add kubectl completion and `k` alias to ubuntu user's .bashrc
20. Check if Helm is already installed
21. Get installed Helm version
22. Decide whether to (re)install Helm
23. Download Helm release
24. Install Helm binary
25. Set Helm binary permissions
26. Remove Helm archive
27. Verify Helm installation
28. Display Helm version
29. Wait for core system pods to be ready
30. Get system pod status
31. Show system pod status

## Templates

- `audit-policy.yaml.j2`
- `k3s-config.yaml.j2`
- `k3s-etcd-snapshot-copy.service.j2`, `k3s-etcd-snapshot-copy.timer.j2`

## Dateien

- `k3s-etcd-snapshot-copy.sh`

---

## Fehlersuche

Allgemeines Vorgehen bei Rollen: `ansible-playbook <playbook>.yml --limit <host> --tags <tag> --check --diff -vvv` zeigt Änderungen ohne sie anzuwenden (`make <ziel>-check` bei den Playbooks mit Dry-Run). Die Rollenübersicht mit Make-Zielen: [Rollen](index.md#fehlersuche-je-rolle).

```bash
sudo systemctl status k3s
sudo journalctl -u k3s -n 100 --no-pager
sudo k3s check-config
kubectl get nodes; kubectl get pods -A | grep -v -e Running -e Completed
sudo cat /etc/rancher/k3s/config.yaml
```

| Symptom | Ursache | Lösung |
|---|---|---|
| Task „Wait for k3s API server“ läuft ab | k3s startet nicht (Port 6443) | `journalctl -u k3s`, [k3s: Fehlersuche](../../capulus-core/b-kubernetes-gitops/b0000-k3s.md#fehlersuche), [Troubleshooting](../a-betriebssystem/a0040-troubleshooting.md) |
| Installer-Prüfsumme stimmt nicht | `k3s_installer_sha256` passt nicht zur gepinnten URL (`k3s_installer_url`) | URL und Prüfsumme gemeinsam aktualisieren ([60020](../6-hintergruende/60020-ansible-rollen.md)) |
| Node bleibt nicht `Ready` | CNI, Netzwerk, Kernelmodule | `kubectl describe node`, `k3s check-config` |
| Audit-Log fehlt | `k3s_audit_log_enabled` aus | [Secrets-Encryption und Audit-Log](../d-sicherheit/d0050-secrets-encryption-audit-log.md#fehlersuche) |
| `kubectl` meldet `permission denied` auf `k3s.yaml` | kubeconfig nicht in `~/.kube/config` kopiert oder `KUBECONFIG` nicht gesetzt | Shell neu starten (`/etc/environment`, `.bashrc`) |
| `k3s_version` ist leer (Modus `init`/`join`) | die Rolle verlangt eine festgeschriebene Version | `k3s_version` in `group_vars/k3s_servers.yml` und `group_vars/k3s_agents.yml` setzen |
| Beitritt bricht ab: „k3s-agent noch installiert“ | der Node war vorher Agent | `/usr/local/bin/k3s-agent-uninstall.sh`, `kubectl delete node <name>`, dann erneut |
| Upgrade ändert nichts | Version unverändert: der Installer läuft nur bei abweichender Version | `k3s --version`, Variable für die Zielversion prüfen |

Allgemeine Fehlersuche: [Host-Troubleshooting](../a-betriebssystem/a0040-troubleshooting.md), [zentrale Fehlersuche](../../capulus-core/2-betrieb-hardware/20070-troubleshooting-zentral.md).

[Alle Rollen](index.md)
