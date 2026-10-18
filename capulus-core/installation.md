# Installation

Diese Seite beschreibt, wie der Stack aus capulus-ansible und capulus-core von null aufgebaut wird. Die Voraussetzungen und der Host-Teil stehen ausführlich in der Doku von [capulus-ansible](../capulus-ansible/index.md).

## Voraussetzungen

- Ein Ubuntu-Server (26.04 LTS) mit SSH-Zugang, ≥ 4 GB RAM und ≥ 20 GB Disk für den TECH-Cluster. Erstinstallation: [Ubuntu-Server-Installation](../capulus-ansible/a-betriebssystem/a0000-ubuntu-server-install.md).
- Auf dem Rechner, der Ansible ausführt: Ansible ≥ 2.14, `make`, `git`.
- Vollständige Liste: [a0020 Voraussetzungen](../capulus-ansible/a-betriebssystem/a0020-prerequisites.md).
- Zugriff auf `github.com/pkr-lab/capulus-core` (ArgoCD liest das Repo nur lesend).

## Quickstart

```bash
git clone https://github.com/pkr-lab/capulus-ansible.git
cd capulus-ansible
$EDITOR inventory/hosts.yml
$EDITOR group_vars/all.yml
ansible-vault encrypt_string 'tskey-auth-DEIN_KEY' --name 'tailscale_auth_key'
make install
```

| Schritt | Was |
|---|---|
| Inventory | `ansible_host` (Server-IP) und ggf. `ansible_ssh_private_key_file` anpassen |
| Variablen | Pflicht: `argocd_repo_url` (zeigt auf capulus-core), `local_subnet`, `timezone` |
| Tailscale-Key | mit `ansible-vault encrypt_string` verschlüsseln und den `!vault`-Block über den bestehenden `tailscale_auth_key` einfügen |
| Playbook | `make install` installiert die Collections und führt `site.yml` aus |

Am Ende druckt das Playbook die ArgoCD-URL und das Admin-Passwort:

```
ArgoCD UI:  https://argocd.tech.homeserver   (CLI/CI: argocd login argocd.tech.homeserver --grpc-web)
Username:   admin
Password:   <auto-generiert>
```

ArgoCD liest danach `argocd/bootstrap/` und alle Apps unter `argocd/apps/tech/` aus capulus-core und rollt sie aus.

## Cluster aufbauen

Alle Befehle laufen im Verzeichnis von capulus-ansible, nach `make install` für TECH:

```bash
make worker-0 worker-1        # Worker in den TECH-Cluster aufnehmen
make libvirt-host             # KVM + Bridge auf dem Homeserver, legt prod-vm an
make prod                     # k3s in der prod-vm; danach im Hub registrieren
make worker-1-libvirt-host    # KVM + Bridge auf worker-1, legt entw-vm an
make entw                     # k3s + ArgoCD in der entw-vm
```

Die PROD-Registrierung im Hub steht in [bootstrap-prod](komponenten/bootstrap-prod.md).

## Prüfen

```bash
kubectl -n argocd get applications
```

Alle Applications sollten `Synced` und `Healthy` werden. Fehlersuche: [a0040 Troubleshooting](../capulus-ansible/a-betriebssystem/a0040-troubleshooting.md).
