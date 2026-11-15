# Rolle `homeserver_ca`

Legt die Homeserver-Root-CA in den System-Vertrauensspeicher (`/usr/local/share/ca-certificates/`) und aktualisiert ihn per Handler. Wird von anderen Rollen per `include_role` eingebunden, nicht direkt im Playbook.

| | |
|---|---|
| Läuft gegen | Banana Pi (`banana_pi_kiosk`), Hosts mit `journal_upload` über HTTPS |
| Tags | die der einbindenden Rolle |
| Quellcode | [roles/homeserver_ca](https://github.com/pkr-lab/capulus-ansible/tree/main/roles/homeserver_ca) |

## Variablen

Defaults aus `defaults/main.yml`. Die einbindende Rolle setzt sie über `vars:` des `include_role`.

| Variable | Default | Bedeutung |
|---|---|---|
| `homeserver_ca_src` | `{{ playbook_dir }}/files/homeserver-root-ca.pem` | Quelle der Root-CA im Repo |
| `homeserver_ca_dest` | `/usr/local/share/ca-certificates/homeserver-root-ca.crt` | Ziel im Vertrauensspeicher |

## Ablauf

1. homeserver-root-ca in den system-vertrauensspeicher legen (registriert `homeserver_ca_result`)

## Handler

- `system-vertrauensspeicher aktualisieren` (`update-ca-certificates`)

---

## Fehlersuche

| Symptom | Ursache | Lösung |
|---|---|---|
| Dienst vertraut der CA nach einem Wechsel nicht | Handler lief erst am Ende des Plays, Dienst wurde vorher neu gestartet | `journal_upload` startet seinen Dienst über `homeserver_ca_result` neu; sonst `sudo update-ca-certificates` und Dienst neu starten |

[Alle Rollen](index.md)
