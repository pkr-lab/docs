# Rolle `semaphore_secrets`

Bootstrap-Secret für den Semaphore-Pod.

| | |
|---|---|
| Läuft gegen | controlplane-0 |
| Tags | `semaphore`, `semaphore-secrets` |
| Quellcode | [roles/semaphore_secrets](https://github.com/pkr-lab/capulus-ansible/tree/main/roles/semaphore_secrets) |

## Variablen

Defaults aus `defaults/main.yml`. Überschreiben in `group_vars/` oder `host_vars/`.

| Variable | Default | Bedeutung |
|---|---|---|
| `semaphore_namespace` | `semaphore` | – |
| `semaphore_bootstrap_secret_name` | `<verdeckt>` | – |
| `semaphore_state_dir` | `/etc/semaphore-secrets` | – |
| `semaphore_admin_username` | `admin` | – |
| `semaphore_admin_email` | `admin@homeserver` | – |
| `semaphore_vault_password` | `` | – |

## Ablauf

1. sicherstellen dass das semaphore-statusverzeichnis existiert
2. prüfen ob die datei mit dem admin-passwort schon existiert
3. zufälliges admin-passwort erzeugen (nur beim ersten lauf)
4. admin-passwort auf der platte ablegen (nur beim ersten lauf)
5. admin-passwort von der platte lesen
6. admin-passwort als fakt bereitstellen
7. schlüssel für die verschlüsselung der zugangsschlüssel erzeugen (nur beim ersten lauf)
8. rechte der datei mit dem verschlüsselungsschlüssel einschränken
9. verschlüsselungsschlüssel für die zugangsschlüssel von der platte lesen
10. verschlüsselungsschlüssel für die zugangsschlüssel als fakt bereitstellen
11. ssh-schlüsselpaar für semaphore erzeugen (nur beim ersten lauf)
12. rechte des privaten ssh-schlüssels einschränken
13. rechte des öffentlichen ssh-schlüssels einschränken
14. privaten ssh-schlüssel von semaphore lesen
15. öffentlichen ssh-schlüssel von semaphore lesen
16. ssh-schlüssel als fakten bereitstellen (hostvars für nachfolgende rollen)
17. sicherstellen dass der namespace semaphore existiert
18. bootstrap-secret im cluster anlegen (`kubectl apply -f -` über stdin, keine Datei in `/tmp`)
19. altes temporäres secret-manifest entfernen
20. zusammenfassung des semaphore-bootstraps anzeigen

---

## Fehlersuche

Allgemeines Vorgehen bei Rollen: `ansible-playbook <playbook>.yml --limit <host> --tags <tag> --check --diff -vvv` zeigt Änderungen ohne sie anzuwenden (`make <ziel>-check` bei den Playbooks mit Dry-Run). Die Rollenübersicht mit Make-Zielen: [Rollen](index.md#fehlersuche-je-rolle).

```bash
sudo ls -l /etc/semaphore-secrets
kubectl -n semaphore get secret
kubectl -n semaphore describe pod -l app.kubernetes.io/name=semaphore
```

| Symptom | Ursache | Lösung |
|---|---|---|
| Semaphore-Pod `CreateContainerConfigError` | Bootstrap-Secret fehlt oder ist unvollständig | Rolle erneut ausrollen (`make semaphore`), Secret prüfen |
| Admin-Passwort unbekannt | Datei `/etc/semaphore-secrets/…` auf dem Homeserver (erste Ausführung erzeugt es) | Datei lesen ([Semaphore](../b-kubernetes-gitops/b0030-semaphore.md)) |
| Schlüssel würden neu erzeugt | State-Verzeichnis gelöscht | Verzeichnis aus dem Backup wiederherstellen, sonst müssen alle Ziele neu autorisiert werden ([semaphore_targets](semaphore_targets.md)) |

Allgemeine Fehlersuche: [Host-Troubleshooting](../a-betriebssystem/a0040-troubleshooting.md), [zentrale Fehlersuche](../../capulus-core/2-betrieb-hardware/20070-troubleshooting-zentral.md).

[Alle Rollen](index.md)
