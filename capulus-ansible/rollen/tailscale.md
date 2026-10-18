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

[Alle Rollen](index.md)
