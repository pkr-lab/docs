# Rolle `crowdsec`

Brute-Force-Schutz für SSH und Traefik-Logs.

| | |
|---|---|
| Läuft gegen | homeserver |
| Tags | `crowdsec`, `security` |
| Quellcode | [roles/crowdsec](https://github.com/pkr-lab/capulus-ansible/tree/main/roles/crowdsec) |

## Variablen

Defaults aus `defaults/main.yml`. Überschreiben in `group_vars/` oder `host_vars/`.

| Variable | Default | Bedeutung |
|---|---|---|
| `crowdsec_repo_key_fingerprint` | `<verdeckt>` | – |
| `crowdsec_collections` | `[crowdsecurity/sshd, crowdsecurity/linux, crowdsecurity/traefik]` | – |
| `crowdsec_traefik_log_glob` | `/var/log/containers/*traefik*.log` | – |

## Ablauf

1. Install gnupg for the key handling
2. Create APT keyrings directory
3. Check whether the CrowdSec repository keyring exists
4. Download CrowdSec repository key
5. Create temporary GnuPG home
6. Convert CrowdSec repository key to a keyring
7. Remove downloaded CrowdSec repository key
8. Read CrowdSec keyring fingerprint
9. Remove temporary GnuPG home
10. Verify CrowdSec repository key fingerprint
11. Write CrowdSec APT repository
12. Install crowdsec package
13. Enable and start crowdsec service
14. Wait for the local CrowdSec API to accept connections
15. Install crowdsec-firewall-bouncer package (iptables backend, matches UFW)
16. Enable and start crowdsec-firewall-bouncer service
17. Install CrowdSec collections
18. Ensure acquisition.d directory exists
19. Deploy SSH log acquisition config (journald)
20. Deploy Traefik log acquisition config
21. Deploy CIDR whitelist for LAN + Tailscale
22. Flush handlers so acquisition/whitelist changes are live before verifying
23. Verify firewall bouncer registered with the local API
24. Warn if no bouncer is registered

## Handler

- Restart crowdsec

## Templates

- `acquis-ssh.yaml.j2`
- `acquis-traefik.yaml.j2`
- `whitelist.yaml.j2`

---

## Fehlersuche

Allgemeines Vorgehen bei Rollen: `ansible-playbook <playbook>.yml --limit <host> --tags <tag> --check --diff -vvv` zeigt Änderungen ohne sie anzuwenden (`make <ziel>-check` bei den Playbooks mit Dry-Run). Die Rollenübersicht mit Make-Zielen: [Rollen](index.md#fehlersuche-je-rolle).

```bash
sudo systemctl status crowdsec crowdsec-firewall-bouncer
sudo cscli metrics
sudo cscli decisions list
sudo cscli bouncers list
sudo cscli alerts list
```

| Symptom | Ursache | Lösung |
|---|---|---|
| Playbook bricht ab: `Der Fingerprint von …crowdsec…keyring.gpg weicht ab` | Repository-Schlüssel stimmt nicht mit `crowdsec_repo_key_fingerprint` überein | Schlüssel nicht vertrauen, Ursache klären, Fingerprint nur nach Prüfung anpassen |
| Warnung „no bouncer registered“ | Firewall-Bouncer nicht an der lokalen API registriert | `sudo cscli bouncers list`, Bouncer-Dienst neu starten |
| Eigene IP wird gesperrt | Whitelist (`local-whitelist.yaml`) enthält das Netz nicht | `sudo cscli decisions delete --ip <ip>`, Whitelist anpassen ([CrowdSec](../d-sicherheit/d0020-crowdsec.md#troubleshooting)) |
| Keine Erkennungen für SSH oder Traefik | Acquisition-Dateien (`/etc/crowdsec/acquis.d/ssh.yaml`, `traefik.yaml`) greifen nicht | `sudo cscli metrics`, Log-Quellen prüfen |

Allgemeine Fehlersuche: [Host-Troubleshooting](../a-betriebssystem/a0040-troubleshooting.md), [zentrale Fehlersuche](../../capulus-core/2-betrieb-hardware/20070-troubleshooting-zentral.md).

[Alle Rollen](index.md)
