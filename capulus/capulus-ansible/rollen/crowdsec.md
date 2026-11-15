# Rolle `crowdsec`

Brute-Force-Schutz für SSH und Traefik-Logs.

| | |
|---|---|
| Läuft gegen | controlplane-0 |
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

1. gnupg für die schlüsselbehandlung installieren
2. verzeichnis für apt-keyrings anlegen
3. prüfen ob der keyring des crowdsec-repositorys existiert
4. schlüssel des crowdsec-repositorys herunterladen
5. temporäres gnupg-home anlegen
6. schlüssel des crowdsec-repositorys in einen keyring umwandeln
7. heruntergeladenen schlüssel des crowdsec-repositorys entfernen
8. fingerprint des crowdsec-keyrings lesen
9. temporäres gnupg-home entfernen
10. fingerprint des crowdsec-repository-schlüssels prüfen
11. apt-repository für crowdsec eintragen
12. paket crowdsec installieren
13. dienst crowdsec aktivieren und starten
14. warten bis die lokale crowdsec-api verbindungen annimmt
15. paket crowdsec-firewall-bouncer installieren (iptables-backend, passend zu ufw)
16. dienst crowdsec-firewall-bouncer aktivieren und starten
17. crowdsec-collections installieren
18. sicherstellen dass das verzeichnis acquisition.d existiert
19. acquisition-konfiguration für ssh-logs ablegen (journald)
20. acquisition-konfiguration für traefik-logs ablegen
21. cidr-whitelist für lan und tailscale ablegen
22. handler sofort ausführen, damit acquisition und whitelist vor der prüfung wirken
23. prüfen dass der firewall-bouncer bei der lokalen api registriert ist
24. warnen wenn kein bouncer registriert ist

## Handler

- crowdsec neu starten

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
