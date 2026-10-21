# Rolle `dnsmasq`

Split-DNS `*.homeserver` (inkl. `*.dev.homeserver` → ENTW und PROD-Einzelhosts), Weiterleitung an Pi-hole.

| | |
|---|---|
| Läuft gegen | homeserver |
| Tags | `dns`, `dnsmasq` |
| Quellcode | [roles/dnsmasq](https://github.com/pkr-lab/capulus-ansible/tree/main/roles/dnsmasq) |

## Variablen

Die Rolle hat keine Defaults.

## Ablauf

1. Install dnsmasq
2. Deploy dnsmasq configuration
3. dnsmasq - automatischer Neustart bei Fehlern (systemd-Drop-in)
4. Drop-in-Verzeichnis anlegen
5. Drop-in schreiben
6. Allow DNS from local subnet (UFW)
7. Allow DNS from Tailnet (UFW)
8. Enable and start dnsmasq
9. Ensure systemd-resolved drop-in directory exists
10. Point systemd-resolved at dnsmasq (primary) + gateway (fallback)
11. Ensure /etc/resolv.conf is the systemd-resolved stub symlink

## Handler

- systemd nach dnsmasq-Drop-in neu laden
- Restart dnsmasq
- Restart systemd-resolved

## Templates

- `dnsmasq.conf.j2`

---

## Fehlersuche

Allgemeines Vorgehen bei Rollen: `ansible-playbook <playbook>.yml --limit <host> --tags <tag> --check --diff -vvv` zeigt Änderungen ohne sie anzuwenden (`make <ziel>-check` bei den Playbooks mit Dry-Run). Die Rollenübersicht mit Make-Zielen: [Rollen](index.md#fehlersuche-je-rolle).

```bash
sudo systemctl status dnsmasq
sudo journalctl -u dnsmasq -n 50 --no-pager
dig +short carplay-api.prod.homeserver @192.168.178.94
sudo dnsmasq --test
resolvectl status
```

| Symptom | Ursache | Lösung |
|---|---|---|
| `dnsmasq` startet nicht | Konfigurationsfehler oder Port 53 belegt (zum Beispiel `systemd-resolved` oder libvirt-dnsmasq) | `sudo dnsmasq --test`, `sudo ss -ulnp \| grep :53`, [libvirt_host](libvirt_host.md) (Snippet entschärft) |
| `*.homeserver` löst nicht auf | Wildcard-Eintrag (`address=/homeserver/<ip>`) fehlt oder PROD-Host nicht in `dnsmasq_prod_vm_hosts` | Variablen prüfen, `make dnsmasq`, [DNS](../../capulus-core/c-netzwerk-dns/c0000-dns-architecture.md#fehlersuche) |
| Host selbst löst nichts auf | `systemd-resolved` zeigt nicht auf dnsmasq oder UFW blockiert Loopback-DNS | `resolvectl status`, Abschnitt „UFW blockierte Loopback-DNS“ in der DNS-Doku |
| Cluster löst Namen nicht auf | Weiterleitung aus CoreDNS | [CoreDNS-Erweiterung](../../capulus-core/c-netzwerk-dns/c0050-coredns-custom.md#fehlersuche) |

Allgemeine Fehlersuche: [Host-Troubleshooting](../a-betriebssystem/a0040-troubleshooting.md), [zentrale Fehlersuche](../../capulus-core/2-betrieb-hardware/20070-troubleshooting-zentral.md).

[Alle Rollen](index.md)
