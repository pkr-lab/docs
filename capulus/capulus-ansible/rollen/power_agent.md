# Rolle `power_agent`

HTTP-API für Helligkeit, Wake und Shutdown aus der iOS-App.

| | |
|---|---|
| Läuft gegen | homeserver |
| Tags | `power-agent`, `power-management` |
| Quellcode | [roles/power_agent](https://github.com/pkr-lab/capulus-ansible/tree/main/roles/power_agent) |

## Variablen

Defaults aus `defaults/main.yml`. Überschreiben in `group_vars/` oder `host_vars/`.

| Variable | Default | Bedeutung |
|---|---|---|
| `power_agent_enabled` | `true` | – |
| `power_agent_port` | `9101` | – |
| `power_agent_bind_address` | `0.0.0.0` | – |
| `power_agent_known_hosts_path` | `/etc/power-agent/known_hosts` | – |
| `power_agent_token_path` | `/etc/power-agent/token` | – |
| `power_agent_backlight_path` | `/sys/class/backlight/intel_backlight/brightness` | – |
| `power_agent_backlight_max_path` | `/sys/class/backlight/intel_backlight/max_brightness` | – |
| `power_agent_bl_power_path` | `/sys/class/backlight/intel_backlight/bl_power` | – |
| `power_agent_poweroff_bin` | `/usr/sbin/poweroff` | – |
| `power_agent_node_name` | `{{ inventory_hostname }}` | – |
| `power_agent_ssh_user` | `{{ cluster_power_manager_ssh_user }}` | – |
| `power_agent_ssh_key` | `{{ cluster_power_manager_ssh_key_path }}` | – |
| `power_agent_kubeconfig` | `{{ cluster_power_manager_kubeconfig }}` | – |
| `power_agent_state_dir` | `{{ cluster_power_manager_state_dir }}` | – |
| `power_agent_targets` | `{{ cluster_power_manager_workers }}` | – |

## Ablauf

1. Python3 installieren (Laufzeit für power-agent.py)
2. Config-Verzeichnis anlegen
3. Bearer-Token für power-agent erzeugen (falls noch nicht vorhanden)
4. Token-Datei-Rechte absichern
5. Erzeugtes Token einlesen
6. bekannte hostschlüssel der worker aus der root-known_hosts lesen
7. hostschlüssel fehlender worker einmalig abrufen
8. known_hosts-zeilen zusammenstellen
9. festes known_hosts für die ssh-aufrufe des power-agent schreiben
10. warnen, wenn kein worker-hostschlüssel gefunden wurde
11. power-agent.py deployen
12. Konfiguration templaten
13. systemd-Unit deployen
14. power-agent aktivieren und starten
15. power-agent deaktivieren, falls power_agent_enabled false
16. Hinweis zum Sealed-Secret-Abgleich ausgeben

## Handler

- Restart power agent

## Templates

- `config.env.j2`
- `power-agent.service.j2`

## Dateien

- `power-agent.py`

---

## Fehlersuche

Allgemeines Vorgehen bei Rollen: `ansible-playbook <playbook>.yml --limit <host> --tags <tag> --check --diff -vvv` zeigt Änderungen ohne sie anzuwenden (`make <ziel>-check` bei den Playbooks mit Dry-Run). Die Rollenübersicht mit Make-Zielen: [Rollen](index.md#fehlersuche-je-rolle).

```bash
sudo systemctl status power-agent
sudo journalctl -u power-agent -f
sudo cat /etc/power-agent/config.env | sed "s/token=.*/token=<TOKEN>/I"
curl -s -H "Authorization: Bearer $(sudo cat /etc/power-agent/token)" http://10.42.2.1:9101/brightness
cat /sys/class/backlight/intel_backlight/brightness
```

| Symptom | Ursache | Lösung |
|---|---|---|
| carplay-api meldet `502 … power-agent unreachable` | Dienst läuft nicht oder lauscht nicht auf der Pod-Bridge-Adresse | `systemctl status power-agent`, `power_agent_bind_address`, [carplay-api](../../capulus-core/komponenten/carplay-api.md#fehlersuche) |
| `502 … power-agent returned 401` | Token im Secret `carplay-api-power-agent-token` weicht von `/etc/power-agent/token` ab | Secret neu versiegeln ([carplay-api: Ersteinrichtung](../../capulus-core/3-apps-workloads/300d0-carplay-api.md#ersteinrichtung)) |
| Helligkeit lässt sich nicht setzen | Backlight-Pfade (`power_agent_backlight_path`) existieren nicht (anderer Treiber) oder Rechte | Pfade unter `/sys/class/backlight/` prüfen |
| Shutdown des Homeservers dauert | `kubectl drain` (bis 180 s) läuft zuerst | `journalctl -u power-agent -f` verfolgen, [iOS: Steuerung](../../capulus-ios/steuerung.md) |
| Wake eines Workers schlägt fehl | WoL-Gerät oder SSH-Zugang | [Rolle wake_on_lan](wake_on_lan.md), [cluster_power_manager](cluster_power_manager.md) |

Allgemeine Fehlersuche: [Host-Troubleshooting](../a-betriebssystem/a0040-troubleshooting.md), [zentrale Fehlersuche](../../capulus-core/2-betrieb-hardware/20070-troubleshooting-zentral.md).

[Alle Rollen](index.md)
