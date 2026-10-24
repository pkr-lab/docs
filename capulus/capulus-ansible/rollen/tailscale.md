# Rolle `tailscale`

WireGuard-Mesh-VPN, Auth-Key aus Ansible Vault; nur der homeserver ist Subnet-Router.

| | |
|---|---|
| Läuft gegen | homeserver, worker-0/-1, xibosignage-Displays, Banana Pi, alert-caller-VM |
| Tags | `tailscale`, `vpn` |
| Quellcode | [roles/tailscale](https://github.com/pkr-lab/capulus-ansible/tree/main/roles/tailscale) |

## Variablen

Defaults aus `defaults/main.yml`. Überschreiben in `group_vars/` oder `host_vars/`.

| Variable | Default | Bedeutung |
|---|---|---|
| `tailscale_repo_codename` | `{{ ansible_distribution_release }}` | – |
| `tailscale_repo_os` | `{{ 'debian' if ansible_distribution == 'Debian' else 'ubuntu' }}` | – |
| `tailscale_signing_key_fingerprint` | `<verdeckt>` | – |
| `tailscale_accept_routes` | `false` | – |
| `tailscale_advertise_routes` | `{{ local_subnet }}` | – |

## Ablauf

1. Create directory for APT keyrings
2. Download Tailscale GPG signing key
3. Check whether the Tailscale keyring exists
4. Install gnupg for the key fingerprint check
5. Create temporary GnuPG home
6. Read Tailscale signing key fingerprint
7. Remove temporary GnuPG home
8. Verify Tailscale signing key fingerprint
9. Add Tailscale APT repository
10. Install (or upgrade) tailscale package
11. Enable and start tailscaled service
12. Wait for tailscaled to be ready
13. Check current Tailscale connection status
14. Parse Tailscale status
15. Display current Tailscale state
16. Authenticate with Tailscale using auth key
17. Create temporary file for auth key
18. Write auth key to temporary file
19. Connect to Tailscale and advertise the local subnet
20. Remove temporary auth key file
21. Refresh Tailscale options (already connected — update hostname/routes)
22. Get final Tailscale status
23. Get Tailscale IP address
24. Display Tailscale connection info
25. Ensure tailscaled is enabled at boot

---

## Fehlersuche

Allgemeines Vorgehen bei Rollen: `ansible-playbook <playbook>.yml --limit <host> --tags <tag> --check --diff -vvv` zeigt Änderungen ohne sie anzuwenden (`make <ziel>-check` bei den Playbooks mit Dry-Run). Die Rollenübersicht mit Make-Zielen: [Rollen](index.md#fehlersuche-je-rolle).

```bash
sudo systemctl status tailscaled
tailscale status
tailscale netcheck
sudo journalctl -u tailscaled -n 100 --no-pager
```

| Symptom | Ursache | Lösung |
|---|---|---|
| `Der Fingerprint von …tailscale-archive-keyring.gpg weicht ab` | Signaturschlüssel stimmt nicht mit `tailscale_signing_key_fingerprint` überein | Schlüssel nicht vertrauen, Ursache klären |
| `tailscale up` schlägt fehl (`invalid key`, `expired`) | Auth-Key abgelaufen oder verbraucht | neuen Auth-Key erzeugen, mit `ansible-vault encrypt_string` eintragen ([Tailscale](../../capulus-core/c-netzwerk-dns/c0010-tailscale.md#troubleshooting)) |
| Subnetz nicht erreichbar über den Homeserver | Subnet-Route nicht genehmigt (nur der Homeserver ist Subnet-Router) | Admin-Konsole: Route genehmigen |
| Route ist im Tailnet genehmigt, trotzdem nicht erreichbar | Client akzeptiert keine Subnetzrouten (`tailscale_accept_routes: false`, Rollen-Default) | `tailscale status --json` prüfen: `AllowedIPs` des eigenen Hosts zeigt nur die eigene `/32`; `tailscale_accept_routes: true` setzen ([50030](../../capulus-core/5-incidents/50030-infotafel-altes-bildformat-und-nas-sync.md#problem-2-keine-neuen-bilder-falsches-routing-zum-nas)) |
| Gerät `NeedsLogin` | Gerät abgelaufen (zum Beispiel worker-0) | neu anmelden, Ablauf im [Token-Register](https://github.com/pkr-lab/capulus-core/blob/main/ops/token-register.yaml) beachten |
| MagicDNS/Split-DNS wirkt nicht | Einstellung in der Admin-Konsole | [DNS](../../capulus-core/c-netzwerk-dns/c0000-dns-architecture.md#fehlersuche) |

Allgemeine Fehlersuche: [Host-Troubleshooting](../a-betriebssystem/a0040-troubleshooting.md), [zentrale Fehlersuche](../../capulus-core/2-betrieb-hardware/20070-troubleshooting-zentral.md).

[Alle Rollen](index.md)
