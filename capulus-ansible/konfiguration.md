# Konfiguration

## Konfigurationsdateien

| Datei / Ordner | Inhalt |
|---|---|
| `ansible.cfg` | Defaults: Inventory `inventory/hosts.yml`, `roles_path = roles`, Fakt-Cache, SSH-Pipelining, `host_key_checking = True` |
| `requirements.yml` | Galaxy-Collections (`ansible.posix`, `community.general`, `kubernetes.core`, `community.docker`) |
| `inventory/hosts.yml` | Hosts und Gruppen |
| `group_vars/all.yml` | gemeinsame Variablen, Semaphore-Projekte, Vault-Werte |
| `group_vars/<gruppe>.yml` | Werte je Gruppe (`banana_pis`, `xibo_displays`, `alert_callers`, `alert_caller_pi`) |
| `group_vars/user_inventory_vault.yml` | komplett verschlüsselte Nutzerliste (Immich/Vaultwarden) |
| `host_vars/<host>/vars.yml` | Werte je Host; `vault.yml` daneben enthält `!vault`-Werte |
| `files/homeserver-root-ca.pem` | Kopie der Root-CA aus capulus-core (`pki/`) |
| `.yamllint`, `.ansible-lint` | Lint-Konfiguration |
| `.gitleaks.toml` | Secret-Scan; erlaubt die öffentlichen APT-Key-Fingerprints |
| `renovate.json` | Renovate: GitHub-Actions-Digests und die Helm-Version der Rolle `k3s` |

## Wichtige Variablen in `group_vars/all.yml`

| Variable | Bedeutung |
|---|---|
| `hostname`, `timezone` | Hostname und Zeitzone des Homeservers |
| `auto_upgrade`, `auto_reboot_if_required` | Auto-Upgrade-Verhalten, siehe unten |
| `k3s_channel`, `k3s_version` | k3s-Release-Kanal (`stable`) bzw. fester Pin |
| `k3s_pod_cidr`, `k3s_service_cidr` | Netze des TECH-Clusters; PROD und ENTW setzen eigene in `host_vars` |
| `k3s_secrets_encryption_enabled`, `k3s_audit_log_*` | Secrets-Verschlüsselung at rest und Audit-Log ([d0050](d-sicherheit/d0050-secrets-encryption-audit-log.md)) |
| `k3s_kubelet_*` | Speicherschutz und Eviction-Schwellen des Kubelet |
| `helm_version`, `argocd_version` | leer = jeweils aktuelle Version, sonst Pin |
| `argocd_repo_url`, `argocd_repo_revision` | Quell-Repo von ArgoCD (capulus-core) und Branch |
| `semaphore_ansible_repo_url` | Repo, das Semaphore für die Playbooks nutzt (diese Repo); Default in `roles/semaphore_bootstrap/defaults/main.yml` |
| `semaphore_projects` | Semaphore-Projekte, Templates und Zeitpläne |
| `local_subnet`, `prod_vm_ip`, `entw_vm_ip` | LAN-Subnetz und VM-Adressen |
| `dnsmasq_prod_vm_hosts` | interne Hosts, die dnsmasq auf die PROD-VM auflöst |
| `network_configure_static_ip`, `network_*` | optionale statische IP |
| `libvirt_host_enabled`, `libvirt_host_configure_bridge` | KVM und Bridge `br0` |
| `cups_print_server_*` | Druckserver (Gerät, Treiber, Warteschlangen) |
| `wireguard_backup_port` | UDP-Port des Notfall-Tunnels |

Rollenspezifische Variablen stehen auf den [Rollen-Seiten](rollen/index.md).

## Secrets (Ansible Vault)

Secrets stehen als `!vault`-Werte direkt in den YAML-Dateien (`tailscale_auth_key`, `semaphore_vault_password`, `ntfy_publisher_token`, `vault_alert_caller_*`, `vault_n8n_nightly_webhook_token`, `vault_vm_write_auth_*`, `vault_logs_write_auth_*`, SMB-Zugang des Scanners) oder komplett verschlüsselt (`user_inventory_vault.yml`). Neu verschlüsseln:

```bash
ansible-vault encrypt_string 'WERT' --name 'variablenname'
```

Neue Tokens und Passwörter werden zusätzlich in `ops/token-register.yaml` von capulus-core eingetragen (Feld `stored_in` mit Präfix `capulus-ansible:`).

## Abhängigkeiten zu capulus-core

| Variable | Default | Zweck |
|---|---|---|
| `alert_caller_core_ref` | `v3.49.0` | Tag von capulus-core, aus dem die alert-caller-Quellen geladen werden |
| `alert_caller_core_files` | drei Einträge | Pfad und SHA-256 je Datei (`caller.py`, `libre-sips-route.patch`, `baresip-srtp-no-gcm.patch`) |
| `alert_caller_core_cache_dir` | `~/.cache/capulus-ansible/alert-caller-<ref>` | lokaler Cache auf dem Steuerrechner |
| `capulus_core_dir` | – | Zielverzeichnis für `make render-bootstrap` (Parameter `CORE_DIR`) |

Wird `alert_caller_core_ref` angehoben, müssen die Prüfsummen in `alert_caller_core_files` mit `sha256sum` der Dateien aus dem neuen Tag aktualisiert werden.

## Auto-Upgrade

`auto_upgrade: true` (Default) hält bei jedem Playbook-Run den gesamten Stack aktuell:

| Komponente | Mechanismus |
|---|---|
| **APT-Pakete** | `apt dist-upgrade` + `unattended-upgrades` für tägliche Sicherheits-Patches |
| **Tailscale** | `state: latest` für das `tailscale`-Paket |
| **k3s** | Folgt `k3s_channel` (Default `stable`), pin via `k3s_version` |
| **Helm** | Re-Run des offiziellen Installers bei neuem Release |
| **ArgoCD** | `helm upgrade --install` ohne `--version`, pin via `argocd_version` |
| **Reboot** | Auto-Reboot wenn APT `/var/run/reboot-required` setzt (togglebar via `auto_reboot_if_required`) |

Für reproduzierbare Builds: `auto_upgrade: false` in `ansible/group_vars/all.yml`.

</details>
