# Nutzung

Alle Befehle laufen im Verzeichnis von capulus-ansible. Das Vault-Passwort wird bei jedem Lauf abgefragt (`VAULT_OPTS ?= --ask-vault-pass`); mit `make … VAULT_OPTS="--vault-password-file ~/.vault_pass"` geht es auch ohne Rückfrage.

## Typische Abläufe

| Aufgabe | Befehl |
|---|---|
| Erreichbarkeit prüfen | `make ping` |
| Änderungen ansehen, ohne sie anzuwenden | `make check` bzw. `make <ziel>-check` |
| Eine Rolle gezielt ausrollen | `make <rolle>` (z. B. `make dnsmasq`, `make argocd`, `make power-agent`) |
| Alles ausrollen | `make install` |
| Worker einbinden oder aktualisieren | `make worker-0`, `make worker-1` |
| PROD- oder ENTW-Cluster ausrollen | `make prod`, `make entw` |
| Vault-Variablen bearbeiten | `make vault-edit`, `make user-inventory-edit` |
| ArgoCD-Bootstrap in capulus-core neu erzeugen | `make render-bootstrap CORE_DIR=../capulus-core` |
| Lint | `make lint` |

Direkter Aufruf mit Tags:

```bash
ansible-playbook -i inventory/hosts.yml site.yml --tags k3s --ask-vault-pass
ansible-playbook -i inventory/hosts.yml site.yml --skip-tags tailscale --ask-vault-pass
ansible-playbook -i inventory/hosts.yml entw.yml --limit entw-vm --check --diff
```

Die Tags je Rolle stehen auf den [Rollen-Seiten](rollen/index.md).

## Alle Make-Targets

| Target | Wirkung |
|---|---|
| `make help` | Show this help. |
| `make deps` | Install required Ansible Galaxy collections. |
| `make ping` | Verify Ansible can reach the server. |
| `make check` | Dry-run the full playbook (no changes applied). |
| `make ssh-hardening` | Apply the ssh_hardening role (key-only SSH) on homeserver, workers, prod-vm, entw-vm and the kiosk Pis. |
| `make install` | Provision the home server end-to-end. |
| `make common` | Run only the `common` role (base OS, firewall, packages). |
| `make crowdsec` | Run only the `crowdsec` role (SSH + Traefik brute-force protection). |
| `make dnsmasq` | Run only the `dnsmasq` role (split-DNS for *.homeserver). |
| `make tailscale` | Run only the `tailscale` role (VPN). |
| `make k3s` | Run only the `k3s` role (Kubernetes + Helm) on homeserver. |
| `make k3s-agent` | Join worker-0 as k3s worker node (requires homeserver k3s running). |
| `make argocd` | Run only the `argocd` role (GitOps controller). |
| `make semaphore` | Bootstrap Semaphore Secret on the home-server. |
| `make semaphore-targets` | Push Semaphore SSH key to all managed targets. |
| `make semaphore-bootstrap` | Provision Projects/Inventories/Templates in Semaphore via API. |
| `make semaphore-bootstrap-local` | Bootstrap Semaphore natively on the home server (no SSH). |
| `make worker-0` | Deploy k3s agent + watchdogs on worker-0 (192.168.178.95). |
| `make worker-0-check` | Dry-run the worker-0 playbook (no changes applied). |
| `make worker-1` | Deploy k3s agent + watchdogs on worker-1 (192.168.178.96). |
| `make worker-1-check` | Dry-run the worker-1 playbook (no changes applied). |
| `make prod` | Deploy PROD-Cluster (k3s) in der KVM-VM auf homeserver (192.168.178.99, docs/4-planung/40080 Phase 2.5). VM muss laufen (make libvirt-host). |
| `make prod-check` | Dry-run des PROD-Playbooks. |
| `make entw` | Deploy ENTW-Cluster (k3s + ArgoCD) in der KVM-VM auf worker-1 (192.168.178.100, docs/4-planung/40080 Phase 1.2). VM muss laufen (make worker-1-libvirt-host). |
| `make entw-check` | Dry-run des ENTW-Playbooks. |
| `make cluster-power-manager` | Re-deploy only the cluster_power_manager role on homeserver (WoL scale-up/down). |
| `make power-agent` | Re-deploy only the power_agent role on homeserver (brightness + manual WoL/poweroff HTTP API). |
| `make cups-print-server` | Re-deploy only the cups_print_server role on homeserver (USB-Drucker per IPP/AirPrint). |
| `make libvirt-host` | Re-deploy only the libvirt_host role (PROD-VM, docs/4-planung/40080 Baustein 5). Braucht libvirt_host_enabled: true in group_vars/all.yml. |
| `make libvirt-bridge` | NUR das Bridge-Netzwerk (libvirt_host, riskant - siehe Rollenkommentar). Erst mit Wartungsfenster + Rollback-Plan. |
| `make worker-1-libvirt-host` | Re-deploy only the libvirt_host role auf worker-1 (ENTW-VM, docs/4-planung/40080 Phase 1). Braucht libvirt_host_enabled: true in host_vars/worker-1/vars.yml. |
| `make worker-1-libvirt-bridge` | NUR das Bridge-Netzwerk auf worker-1 (weniger kritisch als homeserver, aber trotzdem bewusster Schritt). |
| `make nightly-worker-wake` | Re-deploy only the nightly_worker_wake role on homeserver (01:00 Uhr WoL+Update+Poweroff-Zyklus). |
| `make worker-apt-update` | Manuelles apt-Update auf worker-0 + worker-1 (läuft normalerweise nachts automatisch über nightly-worker-wake). |
| `make alarm-kiosks` | Alarmmonitor-Kiosks einrichten (Raspberry Pis, ALAMOS AMweb). |
| `make alarm-kiosks-check` | Dry-run des Alarmmonitor-Kiosk-Playbooks (keine Änderungen). |
| `make xibo-kiosks` | xibosignage-Displays einrichten (Raspberry Pi 3B+, Bilder-Slideshow, Tailscale). |
| `make xibo-kiosks-check` | Dry-run des xibosignage-Displays-Playbooks (keine Änderungen). |
| `make banana-pi-kiosks` | Banana-Pi-Alarmmonitore einrichten (Armbian, ALAMOS AMweb + Server-Fallback). |
| `make banana-pi-kiosks-check` | Dry-run des Banana-Pi-Alarmmonitor-Playbooks (keine Änderungen). |
| `make banana-pi-kiosks-restart-session` | tty1-Session auf den Banana-Pi-Alarmmonitoren erzwungen neu starten (laedt bereits deployte Skripte/Configs neu ein, ohne dass ein Playbook-Diff noetig ist). |
| `make alert-caller` | Telefon-Alarmierung einrichten (alert-caller: baresip, Piper, Tailscale auf der externen VM). |
| `make alert-caller-check` | Dry-run des alert-caller-Playbooks (keine Änderungen). |
| `make alert-caller-pi` | Notfall-Caller (feste Ansage, ruft bei ausbleibendem Watchdog an) auf dem Vereinsheim-Pi einrichten. |
| `make alert-caller-pi-check` | Dry-run des Pi-Notfall-Callers (keine Änderungen). |
| `make lint` | Lint YAML and Ansible. |
| `make render-bootstrap` | Regenerate argocd/bootstrap/ in a capulus-core checkout: make render-bootstrap CORE_DIR=/path/to/capulus-core |
| `make vault-edit` | Edit the vault-encrypted vars file. |
| `make user-inventory-edit` | Edit the encrypted Immich/Vaultwarden user inventory (docs/4-planung/40080, Baustein 7). First run: ansible-vault create $(USER_INVENTORY_FILE). |
| `make vaultwarden-restore` | Vaultwarden-Daten aus der NAS-Backup-PVC wiederherstellen (ueberschreibt Live-Daten!). Erst mit FORCE_RESTORE=true bestaetigen. |
| `make clean` | Remove cached collections and temp artifacts. |

## Semaphore

Semaphore läuft im TECH-Cluster und führt dieselben Playbooks per Zeitplan oder Klick aus. Die Rolle `semaphore_bootstrap` legt Projekte, Inventories, Templates und Zeitpläne per API an. Die Projekte lesen dieses Repo (`semaphore_ansible_repo_url`), die Playbook-Pfade sind `site.yml`, `worker-0.yml` und so weiter. Hintergrund: [b0030 Semaphore](b-kubernetes-gitops/b0030-semaphore.md).

## ArgoCD-Bootstrap erzeugen

`argocd/bootstrap/` in capulus-core ist generiert. Änderungen an den App-Listen (`argocd_platform_apps`, `argocd_workloads_apps` in `roles/argocd/defaults/main.yml`) oder an den Templates der Rolle `argocd` werden so übernommen:

```bash
make render-bootstrap CORE_DIR=../capulus-core
```

Ohne `CORE_DIR` bricht das Target mit einer Fehlermeldung ab. Anschließend in capulus-core committen.
