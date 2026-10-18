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

[Alle Rollen](index.md)
