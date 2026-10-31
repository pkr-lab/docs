# Rolle `ollama_pc`

Ollama auf dem Desktop-PC: Installation, systemd-Override, Wake-on-LAN über NetworkManager, Resume-Hook, GPU-Dienste, UFW und Modelle.

| | |
|---|---|
| Läuft gegen | ollama-pc (Gruppe `ollama_pcs`) |
| Playbook | `ollama-pc.yml` (`make ollama-pc-local`, `make ollama-pc-local-check` ohne SSH, `make ollama-pc`, `make ollama-pc-check` per SSH) |
| Tags | `ollama-pc`, `llm` |
| Quellcode | [roles/ollama_pc](https://github.com/pkr-lab/capulus-ansible/tree/main/roles/ollama_pc) |

Bedienung, BIOS und Betrieb: [300g0](../../capulus-core/3-apps-workloads/300g0-ollama.md).

## Variablen

Defaults aus `defaults/main.yml`. Überschreiben in `host_vars/ollama-pc/vars.yml`.

| Variable | Default | Bedeutung |
|---|---|---|
| `ollama_pc_install_script_url` | `https://ollama.com/install.sh` | offizielles Installationsskript |
| `ollama_pc_version` | `` | leer = aktuelle Version, sonst `OLLAMA_VERSION` für das Skript |
| `ollama_pc_listen` | `0.0.0.0:11434` | `OLLAMA_HOST` |
| `ollama_pc_port` | `11434` | Port für UFW und Health-Check |
| `ollama_pc_keep_alive` | `30m` | `OLLAMA_KEEP_ALIVE` |
| `ollama_pc_flash_attention` | `1` | `OLLAMA_FLASH_ATTENTION` |
| `ollama_pc_kv_cache_type` | `q8_0` | `OLLAMA_KV_CACHE_TYPE` |
| `ollama_pc_context_length` | `16384` | `OLLAMA_CONTEXT_LENGTH` |
| `ollama_pc_models` | `[gemma3:27b-it-q8_0]` | Modelle, die fehlende per `ollama pull` geladen werden |
| `ollama_pc_pull_timeout_seconds` | `7200` | Zeitlimit für das Laden eines Modells |
| `ollama_pc_interface` | `{{ ansible_default_ipv4.interface }}` | physische Schnittstelle (nicht die Bridge) |
| `ollama_pc_nm_connection` | `` | NetworkManager-Verbindung, leer = aus der Schnittstelle ermittelt |
| `ollama_pc_gpu_power_limit_watts` | `280` | `nvidia-smi -pl` |
| `ollama_pc_idle_suspend_enabled` | `true` | Auto-Suspend bei Leerlauf einrichten ([300g0](../../capulus-core/3-apps-workloads/300g0-ollama.md#auto-suspend)) |
| `ollama_pc_idle_suspend_minutes` | `15` | Leerlauf in Minuten bis zum Suspend |
| `ollama_pc_idle_suspend_max_load` | `1.0` | Lastwert (5 min), ab dem der PC als beschäftigt gilt |
| `ollama_pc_ufw_enabled` | `true` | UFW einrichten und aktivieren |
| `ollama_pc_ufw_allowed_sources` | `[<Homeserver-IP>]` | Quellen, die Port 11434 erreichen |
| `ollama_pc_ufw_ssh_sources` | `[<local_subnet>]` | Quellen, die SSH erreichen (verhindert den Selbstausschluss) |

`host_vars/ollama-pc/vars.yml` setzt `ollama_pc_interface: enp7s0`, `ollama_pc_nm_connection: netplan-enp7s0` und `wake_on_lan_mac_address` (liest auch die Rolle `power_agent`).

## Ablauf

1. nvidia-treiber prüfen (`nvidia-smi -L`)
2. Paketlisten aktualisieren (ein Fehler bricht nicht ab, nur eine Warnung), benötigte pakete installieren
3. prüfen, ob ollama installiert ist, sonst Skript laden und ausführen
4. systemd-override mit den umgebungsvariablen deployen (Handler: `daemon-reload`, Neustart von `ollama.service`)
5. nvidia-persistenced beim boot einhängen (Symlink in `multi-user.target.wants`) und starten
6. `nvidia-power-limit.service` deployen, aktivieren, anwenden
7. NetworkManager-Verbindung ermitteln, `802-3-ethernet.wake-on-lan magic` setzen, sofort mit `ethtool -s … wol g` aktivieren, `Wake-on: g` prüfen
8. Resume-Hook `/usr/lib/systemd/system-sleep/ollama-resume` deployen
9. Leerlauf-Skript und Timer `ollama-idle-suspend` deployen, aktivieren (oder deaktivieren)
10. UFW-Regeln für SSH und die Ollama-API anlegen, UFW mit `deny incoming` aktivieren
11. ollama.service aktivieren und starten, auf `/api/tags` warten, fehlende Modelle laden

## Handler

- `ollama_pc_daemon_reload`
- `ollama_pc_restart_ollama`

## Templates und Dateien

- `ollama-override.conf.j2`
- `nvidia-power-limit.service.j2`
- `ollama-idle-suspend.service.j2`, `ollama-idle-suspend.timer.j2`
- `files/ollama-resume.sh`, `files/ollama-idle-suspend.sh`

---

## Fehlersuche

Allgemeines Vorgehen bei Rollen: `ansible-playbook ollama-pc.yml --limit ollama-pc --tags ollama-pc --check --diff -vvv` (`make ollama-pc-local-check`). Die Rollenübersicht: [Rollen](index.md#fehlersuche-je-rolle).

```bash
systemctl status ollama nvidia-persistenced nvidia-power-limit ollama-idle-suspend.timer
sudo systemctl cat ollama
sudo ethtool enp7s0 | grep Wake-on
nmcli -g 802-3-ethernet.wake-on-lan connection show netplan-enp7s0
sudo ufw status verbose
journalctl -u ollama -b
```

| Symptom | Ursache | Lösung |
|---|---|---|
| Rolle bricht bei „nvidia-treiber prüfen“ ab | Treiber fehlt oder `nvidia-smi` kennt keine GPU | Treiber installieren (`ubuntu-drivers`), neu starten |
| „Keine NetworkManager-Verbindung für … gefunden“ | Schnittstelle gehört nicht zu NetworkManager | `ollama_pc_nm_connection` setzen oder `nmcli connection show` prüfen |
| „ethtool meldet … nicht 'Wake-on: g'“ | NIC oder Treiber unterstützt Magic Packet nicht, oder falsche Schnittstelle | `ollama_pc_interface` auf die physische NIC setzen (nicht Bridge, nicht `tailscale0`) |
| Nach dem Reboot `Wake-on: d` | Netplan überschreibt die NetworkManager-Einstellung | in der Netplan-Datei der Schnittstelle `wakeonlan: true` setzen, `sudo netplan apply` |
| SSH nach dem Lauf nicht mehr erreichbar | `ollama_pc_ufw_ssh_sources` deckt die Quelle nicht ab | Variable erweitern, Rolle lokal mit `make ollama-pc-local` neu laufen lassen |
| `Timed out waiting for become success or become password prompt` | Ubuntu 26.04 nutzt `sudo-rs`, dessen Passwort-Prompt Ansible nicht erkennt | `host_vars/ollama-pc` setzt `ansible_become_exe: /usr/bin/sudo.ws` (klassisches sudo, Paket `sudo`); fehlt es: `sudo apt install sudo` |
| Warnung „apt update ist fehlgeschlagen“, vorher Abbruch bei „Failed to update apt cache“ | Eine Fremdquelle in `/etc/apt/sources.list.d` ist kaputt (abgelaufener Signaturschlüssel, Zertifikatsfehler). `apt update` zeigt die Quelle. Die Rolle fährt mit dem vorhandenen Cache fort und installiert nur fehlende Pakete daraus | Die Quelle reparieren oder deaktivieren, die Rolle muss dafür nichts ändern |
| Playbook per SSH nicht ausführbar | Auf dem PC läuft kein SSH-Server | `sudo apt install openssh-server`, oder `make ollama-pc-local` |
| `ollama pull` läuft ins Zeitlimit | Download von bis zu 43 GB | `ollama_pc_pull_timeout_seconds` erhöhen |

Allgemeine Fehlersuche: [Host-Troubleshooting](../a-betriebssystem/a0040-troubleshooting.md), [zentrale Fehlersuche](../../capulus-core/2-betrieb-hardware/20070-troubleshooting-zentral.md).

[Alle Rollen](index.md)
