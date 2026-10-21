# Installation

## Voraussetzungen

| Was | Details |
|---|---|
| Zielhost | Ubuntu Server 26.04 LTS mit SSH-Zugang und `sudo`; Erstinstallation: [a0000](a-betriebssystem/a0000-ubuntu-server-install.md) |
| Steuerrechner | Ansible ≥ 2.14, `make`, `git`; für Lint zusätzlich `yamllint` und `ansible-lint` |
| Collections | `ansible.posix`, `community.general`, `kubernetes.core`, `community.docker` (`requirements.yml`) |
| Vault | Vault-Passwort für `group_vars/all.yml`, `user_inventory_vault.yml` und `host_vars/*/vault.yml` |
| Netz | Zugriff auf die Hosts per LAN oder Tailnet; Wake-on-LAN für die Worker |

Vollständige Voraussetzungen und Vorbereitung der Hosts: [a0020](a-betriebssystem/a0020-prerequisites.md), Installation Schritt für Schritt: [a0030](a-betriebssystem/a0030-installation.md).

## Setup

```bash
git clone https://github.com/pkr-lab/capulus-ansible.git
cd capulus-ansible
make deps
$EDITOR inventory/hosts.yml
$EDITOR group_vars/all.yml
make ping
make check
make install
```

| Target | Wirkung |
|---|---|
| `make deps` | installiert die Galaxy-Collections |
| `make ping` | prüft per Ansible-Ping, ob der Homeserver erreichbar ist |
| `make check` | Dry-Run von `site.yml` (`--check --diff`) |
| `make install` | führt `site.yml` aus und provisioniert den Homeserver samt TECH-Cluster |

Tailscale-Auth-Key verschlüsseln und in `group_vars/all.yml` eintragen:

```bash
ansible-vault encrypt_string 'tskey-auth-DEIN_KEY' --name 'tailscale_auth_key'
```

## Reihenfolge beim ersten Rollout

```bash
make install
make worker-0
make worker-1
make libvirt-host
make worker-1-libvirt-host
make prod
make entw
make alarm-kiosks
make xibo-kiosks
```

`make worker-0` und `make worker-1` brauchen den Join-Token, den `make install` erzeugt. `libvirt-host` braucht `libvirt_host_enabled: true`. Nach `make prod` wird der PROD-Cluster im ArgoCD-Hub registriert, siehe [bootstrap-prod](../capulus-core/komponenten/bootstrap-prod.md). Die Kiosk-Playbooks sind optional und laufen nur manuell.

## Lint und Syntax-Check

```bash
make lint
ansible-playbook -i inventory/hosts.yml site.yml --syntax-check
```

Fehlersuche: [a0040 Troubleshooting](a-betriebssystem/a0040-troubleshooting.md).

## Probleme bei der Installation

| Symptom | Ursache | Lösung |
|---|---|---|
| `make ping` schlägt fehl | SSH-Zugang, IP, Schlüssel | [a0020: Fehlersuche](a-betriebssystem/a0020-prerequisites.md#fehlersuche) |
| Playbook bricht bei der Vault-Entschlüsselung ab | Passwort fehlt oder falsch | [Nutzung: Fehlersuche](nutzung.md#fehlersuche) |
| `make worker-0` meldet Token-Fehler | `make install` hat noch keinen Join-Token erzeugt | zuerst `make install` |
| `make libvirt-host` tut nichts | `libvirt_host_enabled: false` | auf `true` setzen |
| `make prod` scheitert | VM läuft nicht | `make libvirt-host`, [libvirt_host](rollen/libvirt_host.md#fehlersuche) |
| Ablauf der Erstinstallation hakt | siehe Schritt-für-Schritt-Anleitung | [a0030: Fehlersuche](a-betriebssystem/a0030-installation.md#fehlersuche), [Troubleshooting](a-betriebssystem/a0040-troubleshooting.md) |
