# Rolle `alert_caller`

Telefon-Alarm: Webhook-Server, Piper-Ansage, baresip-Anruf.

| | |
|---|---|
| Läuft gegen | externe VM `alert-caller-vm`, Vereinsheim-Pi |
| Tags | `alert-caller`, `alerting` |
| Quellcode | [roles/alert_caller](https://github.com/pkr-lab/capulus-ansible/tree/main/roles/alert_caller) |

## Variablen

Defaults aus `defaults/main.yml`. Überschreiben in `group_vars/` oder `host_vars/`.

| Variable | Default | Bedeutung |
|---|---|---|
| `alert_caller_enabled` | `true` | – |
| `alert_caller_port` | `9310` | – |
| `alert_caller_token` | `{{ vault_alert_caller_token \| default('') }}` | – |
| `alert_caller_auth_mode` | `log` | – |
| `alert_caller_install_dir` | `/opt/alert-caller` | – |
| `alert_caller_config_dir` | `/etc/alert-caller` | – |
| `alert_caller_baresip_dir` | `/etc/alert-caller/baresip` | – |
| `alert_caller_baresip_version` | `4.12.0` | – |
| `alert_caller_baresip_prefix` | `/opt/baresip` | – |
| `alert_caller_baresip_build_dir` | `/usr/local/src/alert-caller` | – |
| `alert_caller_core_ref` | `v3.49.0` | – |
| `alert_caller_core_raw_url` | `https://raw.githubusercontent.com/pkr-lab/capulus-core/{{ alert_caller_core_ref }}/argo…` | – |
| `alert_caller_core_cache_dir` | `{{ lookup('ansible.builtin.env', 'HOME') }}/.cache/capulus-ansible/alert-caller-{{ aler…` | – |
| `alert_caller_core_files` | `[{path: src/caller.py, sha256: f5e9ef8819a3d9ed31c3f2a304b7f3aab0e906678ef2b3ad31429704…` | – |
| `alert_caller_patch_dir` | `{{ alert_caller_core_cache_dir }}/patches` | – |
| `alert_caller_caller_src` | `{{ alert_caller_core_cache_dir }}/src/caller.py` | – |
| `alert_caller_libre_patches` | `[libre-sips-route.patch]` | – |
| `alert_caller_baresip_patches` | `[baresip-srtp-no-gcm.patch]` | – |
| `alert_caller_baresip_modules` | `stdio;g711;opus;aufile;auresamp;menu;ice;srtp;account` | – |
| `alert_caller_build_jobs` | `` | – |
| `alert_caller_static_announcement` | `false` | – |
| `alert_caller_manage_firewall` | `true` | – |
| `alert_caller_piper_dir` | `/opt/piper` | – |
| `alert_caller_piper_version` | `1.8.0` | – |
| `alert_caller_piper_voice` | `de_DE-thorsten-high` | – |
| `alert_caller_sip_user` | `capulus-core` | – |
| `alert_caller_sip_domain` | `sip.linphone.org` | – |
| `alert_caller_sip_password` | `` | – |
| `alert_caller_sip_media_encryption` | `srtp-mand` | – |
| `alert_caller_stun_server` | `stun.cloudflare.com:3478` | – |
| `alert_caller_call_targets` | `[]` | – |
| `alert_caller_watchdog_timeout` | `600` | – |
| `alert_caller_call_cooldown` | `900` | – |
| `alert_caller_call_ring_timeout` | `45` | – |
| `alert_caller_register_timeout` | `15` | – |
| `alert_caller_ice_timeout` | `40` | – |
| `alert_caller_hc_uuid` | `` | – |
| `alert_caller_vm_url` | `` | – |
| `alert_caller_rtp_port_min` | `10000` | – |
| `alert_caller_rtp_port_max` | `10100` | – |
| `alert_caller_webhook_sources` | `[100.64.0.0/10]` | – |

## Ablauf

1. alert-caller einrichten
2. Pflichtvariablen prüfen
3. Zugriffsschutz prüfen
4. Pakete installieren (Sprachausgabe, Build von baresip, Firewall)
5. ufw installieren
6. Cache-Verzeichnisse für alert-caller-Quellen anlegen
7. alert-caller-Quellen aus capulus-core laden
8. Verzeichnisse anlegen
9. Quellen von re und baresip holen
10. libre patchen (sips-Routen, ICE-Takt)
11. baresip patchen (SRTP-Suitenwechsel)
12. re und baresip bauen und installieren
13. Piper in eigener venv installieren
14. Piper-Stimme laden
15. Feste Ansage deployen
16. caller.py deployen
17. baresip-Konfiguration templaten
18. Gruppe für den Lesezugriff auf das SIP-Passwort anlegen
19. baresip-Konto templaten
20. Umgebungsdatei templaten
21. systemd-Unit deployen
22. Firewall einrichten
23. SSH in UFW erlauben
24. UFW Default für eingehenden Verkehr auf deny
25. Webhook-Port nur aus den erlaubten Netzen öffnen
26. RTP-Portbereich öffnen
27. UFW aktivieren
28. alert-caller aktivieren und starten
29. alert-caller deaktivieren, falls alert_caller_enabled false

## Handler

- Restart alert caller

## Templates

- `alert-caller.env.j2`
- `alert-caller.service.j2`
- `baresip-accounts.j2`
- `baresip-config.j2`

## Dateien

- `cluster-down.wav`

---

## Fehlersuche

Allgemeines Vorgehen bei Rollen: `ansible-playbook <playbook>.yml --limit <host> --tags <tag> --check --diff -vvv` zeigt Änderungen ohne sie anzuwenden (`make <ziel>-check` bei den Playbooks mit Dry-Run). Die Rollenübersicht mit Make-Zielen: [Rollen](index.md#fehlersuche-je-rolle).

```bash
sudo systemctl status alert-caller
sudo journalctl -u alert-caller -n 100 --no-pager
ls /etc/alert-caller /opt/alert-caller /etc/alert-caller/baresip
sudo ss -tlnp | grep 9310                   # lauscht der Webhook-Server (Port alert_caller_port)?
```

| Symptom | Ursache | Lösung |
|---|---|---|
| Playbook bricht ab: `alert_caller_sip_password muss im Vault stehen …` | Pflichtvariablen fehlen oder enthalten `;` oder `"` | Vault-Variablen setzen ([Telefon-Alarm](../../capulus-core/1-benachrichtigungen/10020-telefon-alarm.md)) |
| Playbook bricht ab: `alert_caller_auth_mode ist log oder enforce …` | `enforce` ohne Token | `vault_alert_caller_token` setzen oder `alert_caller_auth_mode: log` |
| Kein Anruf bei einem Alarm | baresip-Konto nicht registriert, SIP-Zugang falsch, Piper-Ansage fehlt | Log des Dienstes, [Telefon-Alarm: Fehlersuche](../../capulus-core/1-benachrichtigungen/10020-telefon-alarm.md#fehlersuche) |
| Alertmanager erreicht den Caller nicht | Tailscale auf der externen VM, Port 9310, Token | `tailscale status`, Webhook-URL im Alertmanager |
| baresip-Build scheitert | Build-Abhängigkeiten, Download der gepinnten Version `alert_caller_baresip_version` | Task-Ausgabe lesen, Abhängigkeiten installieren |

Allgemeine Fehlersuche: [Host-Troubleshooting](../a-betriebssystem/a0040-troubleshooting.md), [zentrale Fehlersuche](../../capulus-core/2-betrieb-hardware/20070-troubleshooting-zentral.md).

[Alle Rollen](index.md)
