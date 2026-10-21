# Rolle `wake_on_lan`

`ethtool wol g` bei jedem Boot.

| | |
|---|---|
| Läuft gegen | worker-0, worker-1 |
| Tags | `power-management`, `wake-on-lan` |
| Quellcode | [roles/wake_on_lan](https://github.com/pkr-lab/capulus-ansible/tree/main/roles/wake_on_lan) |

## Variablen

Defaults aus `defaults/main.yml`. Überschreiben in `group_vars/` oder `host_vars/`.

| Variable | Default | Bedeutung |
|---|---|---|
| `wake_on_lan_enabled` | `true` | – |
| `wake_on_lan_interface` | `{{ ansible_default_ipv4.interface }}` | – |

## Ablauf

1. ethtool installieren (für WoL-Konfiguration)
2. systemd-Unit zur WoL-Aktivierung beim Boot deployen
3. enable-wol.service aktivieren und (neu) starten
4. enable-wol.service deaktivieren, falls wake_on_lan_enabled false

## Templates

- `enable-wol.service.j2`

---

## Fehlersuche

Allgemeines Vorgehen bei Rollen: `ansible-playbook <playbook>.yml --limit <host> --tags <tag> --check --diff -vvv` zeigt Änderungen ohne sie anzuwenden (`make <ziel>-check` bei den Playbooks mit Dry-Run). Die Rollenübersicht mit Make-Zielen: [Rollen](index.md#fehlersuche-je-rolle).

```bash
sudo systemctl status enable-wol
sudo ethtool <interface> | grep -i wake
ip -br link
```

| Symptom | Ursache | Lösung |
|---|---|---|
| Worker lässt sich nicht per WoL wecken | WoL im BIOS oder der Netzwerkkarte aus, `ethtool wol g` nicht gesetzt (Unit läuft bei jedem Boot) | `ethtool <if>` zeigt `Wake-on: g`; BIOS-Einstellung prüfen, [Cluster Power Manager](../2-betrieb-hardware/20020-cluster-power-manager.md#fehlerbehebung) |
| Nach dem Neustart wieder aus | Unit nicht aktiv | `systemctl status enable-wol` |
| Magic Packet kommt nicht an | Switch oder Router leitet Broadcast nicht weiter | Netz prüfen, gleiches Subnetz nötig |

Allgemeine Fehlersuche: [Host-Troubleshooting](../a-betriebssystem/a0040-troubleshooting.md), [zentrale Fehlersuche](../../capulus-core/2-betrieb-hardware/20070-troubleshooting-zentral.md).

[Alle Rollen](index.md)
