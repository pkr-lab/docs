# Rolle `infotafel_poweroff`

Erlaubt dem n8n-Workflow „Infotafel: Herunterfahren“, den Infotafel-Pi per SSH herunterzufahren, und sonst nichts.

| | |
|---|---|
| Läuft gegen | Gruppe `xibo_displays` (Infotafel) |
| Tags | `infotafel-poweroff`, `ssh` |
| Quellcode | [roles/infotafel_poweroff](https://github.com/pkr-lab/capulus-ansible/tree/main/roles/infotafel_poweroff) |

## Variablen

Defaults aus `defaults/main.yml`. Überschreiben in `group_vars/` oder `host_vars/`.

| Variable | Default | Bedeutung |
|---|---|---|
| `infotafel_poweroff_user` | `{{ xibo_kiosk_user }}` | Benutzer, in dessen `authorized_keys` der Key landet |
| `infotafel_poweroff_public_key` | leer | Public Key des n8n-Credentials `SSH Infotafel (Herunterfahren)`, gesetzt in `group_vars/xibo_displays.yml` |
| `infotafel_poweroff_command` | `/usr/bin/systemctl poweroff` | einziger Befehl, den der Key ausführen darf |

## Ablauf

1. warnen, falls kein Public Key gesetzt ist
2. Key mit `command="sudo /usr/bin/systemctl poweroff"` und ohne Weiterleitungen, Agent und PTY autorisieren (forced command: egal, was der Client sendet, es läuft nur dieser Befehl)
3. `/etc/sudoers.d/infotafel-poweroff-n8n` mit passwortloser Freigabe genau dieses Befehls schreiben

```bash
ansible-playbook xibo-kiosks.yml --limit infotafel --tags infotafel-poweroff --check --diff
ansible-playbook xibo-kiosks.yml --limit infotafel --tags infotafel-poweroff
```

Der Workflow ist beschrieben in [capulus-core: 30070-n8n](../../capulus-core/3-apps-workloads/30070-n8n.md).

---

## Fehlersuche

| Symptom | Ursache | Lösung |
|---|---|---|
| n8n meldet „Infotafel nicht heruntergefahren“ (Timeout) | Infotafel aus oder nicht im Tailnet, oder der n8n-Pod erreicht 100.105.105.92 nicht | `tailscale status` auf dem Homeserver, aus dem Pod `node -e` mit `net.connect` auf Port 22 |
| `Permission denied (publickey)` | Rolle noch nicht ausgerollt oder Key im n8n-Credential ist ein anderer | Rolle ausrollen, ggf. neuen Key erzeugen und in Credential und `group_vars/xibo_displays.yml` tauschen |
| Verbindung klappt, Pi fährt nicht herunter | sudoers-Datei fehlt oder Befehlspfad weicht ab (`command -v systemctl`) | Rolle erneut ausrollen, `infotafel_poweroff_command` anpassen |

Ein heruntergefahrener Pi startet nur nach Trennen und Wiederherstellen der Stromversorgung (oder per Zeitschaltuhr).
