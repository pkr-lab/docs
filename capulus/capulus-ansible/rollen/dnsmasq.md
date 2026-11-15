# Rolle `dnsmasq`

Split-DNS `*.homeserver` (inkl. `*.dev.homeserver` → ENTW und PROD-Einzelhosts), Weiterleitung an Pi-hole.

| | |
|---|---|
| Läuft gegen | controlplane-0 |
| Tags | `dns`, `dnsmasq` |
| Quellcode | [roles/dnsmasq](https://github.com/pkr-lab/capulus-ansible/tree/main/roles/dnsmasq) |

## Variablen

Die Rolle hat keine Defaults.

## Hostnamen

Das Template legt `address=`-Einträge für die Hosts an: `homeserver` und `controlplane-0` (beide auf die Server-IP), `controlplane-1` und `worker-0` (`.95`), `worker-1`, `ugreen-nas`. `homeserver` und `worker-0` sind Übergangs-Alias für das neue Namensschema ([60010](../../capulus-core/6-hintergruende/60010-ansible-hosts-und-playbooks.md#namensschema-inventar-host-gruppen-und-k3s-nodename)). `address=/homeserver/…` gilt zugleich für alle Subdomains `*.homeserver` und bleibt dauerhaft.

## Ablauf

1. dnsmasq installieren
2. dnsmasq-konfiguration ablegen
3. automatischen neustart von dnsmasq bei fehlern einrichten (systemd-drop-in)
4. drop-in-verzeichnis anlegen
5. drop-in schreiben
6. dns aus dem lokalen subnetz erlauben (ufw)
7. dns aus dem tailnet erlauben (ufw)
8. dnsmasq aktivieren und starten
9. sicherstellen dass das drop-in-verzeichnis von systemd-resolved existiert
10. systemd-resolved auf dnsmasq (primär) und gateway (fallback) zeigen lassen
11. sicherstellen dass /etc/resolv.conf der stub-symlink von systemd-resolved ist

## Handler

- systemd nach dnsmasq-drop-in neu laden
- dnsmasq neu starten
- systemd-resolved neu starten

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
