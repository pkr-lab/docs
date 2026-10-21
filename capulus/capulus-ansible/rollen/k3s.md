# Rolle `k3s`

Kubernetes-Server und Helm, je Cluster eigene CIDRs.

| | |
|---|---|
| Läuft gegen | homeserver, prod-vm, entw-vm |
| Tags | – |
| Quellcode | [roles/k3s](https://github.com/pkr-lab/capulus-ansible/tree/main/roles/k3s) |

## Variablen

Defaults aus `defaults/main.yml`. Überschreiben in `group_vars/` oder `host_vars/`.

| Variable | Default | Bedeutung |
|---|---|---|
| `k3s_installer_url` | `https://raw.githubusercontent.com/k3s-io/k3s/2977c525a2e7a487886107ce4df43630ae9b03b2/i…` | – |
| `k3s_installer_sha256` | `e5cc3b3d9dfc1662c2d9be6da5abc9a4cd317d6abc3a5ffc02e3dd3248207fee` | – |
| `k3s_helm_release` | `v3.22.1` | – |
| `k3s_helm_arch` | `{{ 'arm64' if ansible_architecture == 'aarch64' else 'amd64' }}` | – |

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
| Upgrade ändert nichts | Version unverändert: der Installer läuft nur bei abweichender Version | `k3s --version`, Variable für die Zielversion prüfen |

Allgemeine Fehlersuche: [Host-Troubleshooting](../a-betriebssystem/a0040-troubleshooting.md), [zentrale Fehlersuche](../../capulus-core/2-betrieb-hardware/20070-troubleshooting-zentral.md).

[Alle Rollen](index.md)
