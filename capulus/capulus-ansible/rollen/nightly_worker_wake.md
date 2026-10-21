# Rolle `nightly_worker_wake`

01:00-Timer: Worker wecken, Update anstoßen, wieder herunterfahren.

| | |
|---|---|
| Läuft gegen | homeserver |
| Tags | `nightly-worker-wake`, `power-management` |
| Quellcode | [roles/nightly_worker_wake](https://github.com/pkr-lab/capulus-ansible/tree/main/roles/nightly_worker_wake) |

## Variablen

Defaults aus `defaults/main.yml`. Überschreiben in `group_vars/` oder `host_vars/`.

| Variable | Default | Bedeutung |
|---|---|---|
| `nightly_worker_wake_enabled` | `true` | – |
| `nightly_worker_wake_n8n_trigger_ssh_user` | `ubuntu` | – |
| `nightly_worker_wake_n8n_trigger_public_key` | `` | – |
| `nightly_worker_wake_systemctl_bin` | `/usr/bin/systemctl` | – |
| `nightly_worker_wake_max_runtime_seconds` | `1500` | – |
| `nightly_worker_wake_ready_timeout_seconds` | `180` | – |
| `nightly_worker_wake_stop_grace_seconds` | `30` | – |
| `nightly_worker_wake_poll_seconds` | `15` | – |
| `nightly_worker_wake_ssh_user` | `{{ cluster_power_manager_ssh_user }}` | – |
| `nightly_worker_wake_ssh_key_path` | `{{ cluster_power_manager_ssh_key_path }}` | – |
| `nightly_worker_wake_kubeconfig` | `{{ cluster_power_manager_kubeconfig }}` | – |
| `nightly_worker_wake_state_dir` | `{{ cluster_power_manager_state_dir }}` | – |
| `nightly_worker_wake_semaphore_api_base` | `https://semaphore-api.tech.homeserver` | – |
| `nightly_worker_wake_semaphore_admin_username` | `admin` | – |
| `nightly_worker_wake_semaphore_admin_password_path` | `/etc/semaphore-secrets/admin_password` | – |
| `nightly_worker_wake_ntfy_host` | `{{ cluster_power_manager_ntfy_host }}` | – |
| `nightly_worker_wake_ntfy_ip` | `{{ cluster_power_manager_ntfy_ip }}` | – |
| `nightly_worker_wake_ntfy_topic` | `{{ cluster_power_manager_ntfy_topic }}` | – |
| `nightly_worker_wake_n8n_webhook_url` | `https://n8n.prod.homeserver/webhook/nightly-worker-update` | – |
| `nightly_worker_wake_n8n_token` | `{{ vault_n8n_nightly_webhook_token \| default('') }}` | – |
| `nightly_worker_wake_workers` | `{{ cluster_power_manager_workers }}` | – |

## Ablauf

1. Config-Verzeichnis anlegen
2. nightly-worker-wake.sh deployen
3. Konfiguration templaten
4. n8n-auth-header-datei schreiben
5. systemd-Service (oneshot) deployen
6. Alten nightly-worker-wake.timer deaktivieren (Trigger läuft jetzt über n8n statt systemd-Timer)
7. Alte Timer-Unit-Datei entfernen (Rest einer älteren Rollen-Version)
8. Warnen, falls noch kein n8n-Trigger-Public-Key gesetzt ist
9. n8n-Trigger-Key autorisieren (nur für Service-Start, forced command)
10. Passwortlosen sudo für Service-Start erlauben ({{ nightly_worker_wake_n8n_trigger_ssh_user }})
11. sudoers-Freigabe für n8n-Trigger entfernen, falls nightly_worker_wake_enabled false

## Handler

- Reload systemd for nightly worker wake

## Templates

- `config.env.j2`
- `nightly-worker-wake.service.j2`

## Dateien

- `nightly-worker-wake.sh`

---

## Fehlersuche

Allgemeines Vorgehen bei Rollen: `ansible-playbook <playbook>.yml --limit <host> --tags <tag> --check --diff -vvv` zeigt Änderungen ohne sie anzuwenden (`make <ziel>-check` bei den Playbooks mit Dry-Run). Die Rollenübersicht mit Make-Zielen: [Rollen](index.md#fehlersuche-je-rolle).

```bash
sudo systemctl status nightly-worker-wake.timer nightly-worker-wake.service
sudo journalctl -u nightly-worker-wake -n 150 --no-pager
systemctl list-timers | grep nightly
ls -l /var/lib/cluster-power-manager
```

| Symptom | Ursache | Lösung |
|---|---|---|
| Worker werden nachts nicht geweckt | Timer inaktiv oder Dienst scheitert früh (WoL, SSH) | Log lesen, [Nightly Worker Update](../2-betrieb-hardware/20030-nightly-worker-update.md#fehlerbehebung) |
| Update läuft nicht | Semaphore-API nicht erreichbar (`nightly_worker_wake_semaphore_api_base`), Admin-Zugang | Semaphore prüfen ([Semaphore](../b-kubernetes-gitops/b0030-semaphore.md#troubleshooting)) |
| Worker bleibt an | Laufzeitbegrenzung (`max_runtime_seconds`: 1500) und Herunterfahren schlugen fehl | Log am Ende, [cluster_power_manager_target](cluster_power_manager_target.md) |
| n8n-Trigger löst den Lauf nicht aus | SSH-Schlüssel (`nightly_worker_wake_n8n_trigger_public_key`) nicht hinterlegt | Schlüssel und Benutzer prüfen ([n8n](../../capulus-core/3-apps-workloads/30070-n8n.md#fehlerbehebung)) |

Allgemeine Fehlersuche: [Host-Troubleshooting](../a-betriebssystem/a0040-troubleshooting.md), [zentrale Fehlersuche](../../capulus-core/2-betrieb-hardware/20070-troubleshooting-zentral.md).

[Alle Rollen](index.md)
