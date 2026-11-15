# Rolle `tailscale`

WireGuard-Mesh-VPN, Auth-Key aus Ansible Vault; nur der homeserver ist Subnet-Router.

| | |
|---|---|
| Läuft gegen | controlplane-0, controlplane-1, worker-1, xibosignage-Displays, Banana Pi, alert-caller-VM |
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

1. verzeichnis für apt-keyrings anlegen
2. gpg-signaturschlüssel von tailscale herunterladen
3. prüfen ob der tailscale-keyring existiert
4. gnupg für die fingerprint-prüfung installieren
5. temporäres gnupg-home anlegen
6. fingerprint des tailscale-signaturschlüssels lesen
7. temporäres gnupg-home entfernen
8. fingerprint des tailscale-signaturschlüssels prüfen
9. apt-repository für tailscale eintragen
10. paket tailscale installieren (oder aktualisieren)
11. dienst tailscaled aktivieren und starten
12. warten bis tailscaled bereit ist
13. aktuellen verbindungsstatus von tailscale prüfen
14. tailscale-status auswerten
15. aktuellen tailscale-zustand anzeigen
16. bei tailscale mit auth-key anmelden
17. temporäre datei für den auth-key anlegen
18. auth-key in die temporäre datei schreiben
19. mit tailscale verbinden und das lokale subnetz ankündigen
20. temporäre auth-key-datei entfernen
21. tailscale-optionen auffrischen (schon verbunden — hostname/routen aktualisieren)
22. abschließenden tailscale-status lesen
23. tailscale-ip-adresse lesen
24. tailscale-verbindungsdaten anzeigen

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
| `Download Tailscale GPG signing key` scheitert mit `Connection reset by peer` oder `The read operation timed out` | `pkgs.tailscale.com` setzt Verbindungen zeitweise zurück; die Rolle wiederholt fünfmal, danach bricht sie ab | Lauf erneut starten; bleibt es dauerhaft, `curl -v https://pkgs.tailscale.com/` vom Host prüfen ([Hintergrund](../6-hintergruende/60020-ansible-rollen.md#tailscale-wiederholungen-beim-paket-server)) |
| `tailscale up` schlägt fehl (`invalid key`, `expired`) | Auth-Key abgelaufen oder verbraucht | neuen Auth-Key erzeugen, mit `ansible-vault encrypt_string` eintragen ([Tailscale](../../capulus-core/c-netzwerk-dns/c0010-tailscale.md#troubleshooting)) |
| Subnetz nicht erreichbar über den Homeserver | Subnet-Route nicht genehmigt (nur der Homeserver ist Subnet-Router) | Admin-Konsole: Route genehmigen |
| Route ist im Tailnet genehmigt, trotzdem nicht erreichbar | Client akzeptiert keine Subnetzrouten (`tailscale_accept_routes: false`, Rollen-Default) | `tailscale status --json` prüfen: `AllowedIPs` des eigenen Hosts zeigt nur die eigene `/32`; `tailscale_accept_routes: true` setzen ([50030](../../capulus-core/5-incidents/50030-infotafel-altes-bildformat-und-nas-sync.md#problem-2-keine-neuen-bilder-falsches-routing-zum-nas)) |
| Gerät `NeedsLogin` | Gerät abgelaufen (zum Beispiel worker-0) | neu anmelden, Ablauf im [Token-Register](https://github.com/pkr-lab/capulus-core/blob/main/ops/token-register.yaml) beachten |
| MagicDNS/Split-DNS wirkt nicht | Einstellung in der Admin-Konsole | [DNS](../../capulus-core/c-netzwerk-dns/c0000-dns-architecture.md#fehlersuche) |

Allgemeine Fehlersuche: [Host-Troubleshooting](../a-betriebssystem/a0040-troubleshooting.md), [zentrale Fehlersuche](../../capulus-core/2-betrieb-hardware/20070-troubleshooting-zentral.md).

[Alle Rollen](index.md)
