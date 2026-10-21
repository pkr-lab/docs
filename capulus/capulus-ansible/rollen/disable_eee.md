# Rolle `disable_eee`

Energy Efficient Ethernet dauerhaft deaktivieren (Realtek-NIC-Link-Drops).

| | |
|---|---|
| Läuft gegen | homeserver, worker-0, worker-1 |
| Tags | `disable-eee`, `network` |
| Quellcode | [roles/disable_eee](https://github.com/pkr-lab/capulus-ansible/tree/main/roles/disable_eee) |

## Variablen

Defaults aus `defaults/main.yml`. Überschreiben in `group_vars/` oder `host_vars/`.

| Variable | Default | Bedeutung |
|---|---|---|
| `disable_eee_enabled` | `true` | – |
| `disable_eee_interface` | `{{ ansible_default_ipv4.interface }}` | – |

## Ablauf

1. ethtool installieren (für EEE-Konfiguration)
2. systemd-Unit zur EEE-Deaktivierung beim Boot deployen
3. disable-eee.service aktivieren und (neu) starten
4. disable-eee.service deaktivieren, falls disable_eee_enabled false

## Templates

- `disable-eee.service.j2`

---

## Fehlersuche

Allgemeines Vorgehen bei Rollen: `ansible-playbook <playbook>.yml --limit <host> --tags <tag> --check --diff -vvv` zeigt Änderungen ohne sie anzuwenden (`make <ziel>-check` bei den Playbooks mit Dry-Run). Die Rollenübersicht mit Make-Zielen: [Rollen](index.md#fehlersuche-je-rolle).

```bash
sudo systemctl status disable-eee
sudo ethtool --show-eee <interface>
ip -br link
```

| Symptom | Ursache | Lösung |
|---|---|---|
| Link bricht trotzdem ab (Realtek-NIC) | Unit lief nicht oder falsche Schnittstelle (`disable_eee_interface` ist die Standard-Route) | `systemctl status disable-eee`, Schnittstellennamen mit `ip -br link` prüfen |
| `ethtool: Operation not supported` | Treiber unterstützt EEE nicht | Rolle für diesen Host mit `disable_eee_enabled: false` abschalten |

Allgemeine Fehlersuche: [Host-Troubleshooting](../a-betriebssystem/a0040-troubleshooting.md), [zentrale Fehlersuche](../../capulus-core/2-betrieb-hardware/20070-troubleshooting-zentral.md).

[Alle Rollen](index.md)
