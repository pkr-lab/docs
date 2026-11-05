# Rolle `resource_watchdog`

Selbst-Abschaltung bei Dauerlast, auf den k3s-Servern nur Alarm.

| | |
|---|---|
| Läuft gegen | alle Knoten und Kiosks |
| Tags | `resource-watchdog`, `thermal` |
| Quellcode | [roles/resource_watchdog](https://github.com/pkr-lab/capulus-ansible/tree/main/roles/resource_watchdog) |

## Variablen

Defaults aus `defaults/main.yml`. Überschreiben in `group_vars/` oder `host_vars/`.

| Variable | Default | Bedeutung |
|---|---|---|
| `resource_watchdog_enabled` | `true` | – |
| `resource_watchdog_action` | `poweroff` | `poweroff` fährt den Host bei Dauerlast herunter (Worker, Kiosks). `notify` schickt nur einen ntfy-Alarm (Priorität 5) und lässt den Host laufen. `group_vars/k3s_servers.yml` setzt `notify` |
| `resource_watchdog_alert_repeat_seconds` | `3600` | Mindestabstand zwischen zwei Alarmen im Modus `notify` |
| `resource_watchdog_poll_seconds` | `10` | – |
| `resource_watchdog_cpu_threshold` | `90` | – |
| `resource_watchdog_cpu_sustain_seconds` | `300` | – |
| `resource_watchdog_ram_threshold` | `90` | – |
| `resource_watchdog_ram_sustain_seconds` | `300` | – |
| `resource_watchdog_cpu_warn_threshold` | `85` | – |
| `resource_watchdog_cpu_warn_sustain_seconds` | `60` | – |
| `resource_watchdog_ram_warn_threshold` | `80` | – |
| `resource_watchdog_ram_warn_sustain_seconds` | `60` | – |
| `resource_watchdog_ntfy_host` | `ntfy.tech.homeserver` | – |
| `resource_watchdog_ntfy_ip` | `{{ network_static_ip }}` | – |
| `resource_watchdog_ntfy_topic` | `Home-Lab-System` | – |
| `resource_watchdog_metrics` | `[{name: cpu, sustain_seconds: '{{ resource_watchdog_cpu_sustain_seconds }}', threshold:…` | – |

## Ablauf

1. Deploy resource watchdog script
2. Ensure resource watchdog config directory exists
3. Deploy per-metric environment files
4. Deploy resource watchdog systemd template unit
5. Enable and start resource watchdog instances
6. Disable resource watchdog instances if turned off via resource_watchdog_enabled

## Handler

- Restart resource watchdogs

## Templates

- `instance.env.j2`
- `resource-watchdog@.service.j2`

## Dateien

- `resource-watchdog.sh`

---

## Fehlersuche

Allgemeines Vorgehen bei Rollen: `ansible-playbook <playbook>.yml --limit <host> --tags <tag> --check --diff -vvv` zeigt Änderungen ohne sie anzuwenden (`make <ziel>-check` bei den Playbooks mit Dry-Run). Die Rollenübersicht mit Make-Zielen: [Rollen](index.md#fehlersuche-je-rolle).

```bash
systemctl list-units "resource-watchdog@*"
sudo journalctl -u "resource-watchdog@*" -n 100 --no-pager
cat /etc/resource-watchdog/*.env
```

| Symptom | Ursache | Lösung |
|---|---|---|
| Auf controlplane-0 oder controlplane-1 kommt ein Alarm „Dauerhaft hohe CPU/RAM-Auslastung“, der Host bleibt an | `resource_watchdog_action: notify` auf den k3s-Servern (gewollt: ein Poweroff würde ein etcd-Mitglied und bei zwei Mitgliedern die API mitnehmen) | Ursache der Last suchen, Alarm wiederholt sich höchstens alle `resource_watchdog_alert_repeat_seconds` |
| Host schaltet sich ab | Dauerlast über der Schwelle (CPU und RAM 90 % über 300 s) — gewollter Selbstschutz | Ursache der Last suchen ([Monitoring](../../capulus-core/6-hintergruende/60060-monitoring-und-alerting.md)), Schwellen unter `resource_watchdog_*` anpassen |
| Warnmeldungen (85 % CPU, 80 % RAM über 60 s) fehlen | Meldung an ntfy scheitert | [ntfy_publisher](ntfy_publisher.md), Topic `Home-Lab-System` |
| Dienst läuft nicht | Instanz nicht aktiviert (Vorlage `resource-watchdog@.service`) | `systemctl enable --now resource-watchdog@<name>` |

Allgemeine Fehlersuche: [Host-Troubleshooting](../a-betriebssystem/a0040-troubleshooting.md), [zentrale Fehlersuche](../../capulus-core/2-betrieb-hardware/20070-troubleshooting-zentral.md).

[Alle Rollen](index.md)
