# Rolle `semaphore_bootstrap`

Projekte, Inventories, Templates und Zeitpläne in Semaphore per REST-API.

| | |
|---|---|
| Läuft gegen | controlplane-0 |
| Tags | – |
| Quellcode | [roles/semaphore_bootstrap](https://github.com/pkr-lab/capulus-ansible/tree/main/roles/semaphore_bootstrap) |

## Projekte für die Control-Planes und Worker

Das Projekt für controlplane-1 heißt `controlplane-1` (Template `Deploy controlplane-1`, Inventar `controlplane_1_only`). Das frühere Projekt `worker-0` bleibt nach dem Umbenennen in Semaphore als Altlast stehen, `make semaphore-bootstrap` löscht nichts. Es wird in der UI von Hand gelöscht. Der nächtliche Zyklus ([20030](../2-betrieb-hardware/20030-nightly-worker-update.md)) sucht nur noch `worker-1`.

## Hinweis zur Quelle der Projekte

`semaphore_projects` wird aus `defaults/main.yml` gelesen, kann aber in `group_vars/all.yml` überschrieben werden. Nach dem Umbenennen der Gruppen (`k3s_servers`, `k3s_primary`, `k3s_agents` statt `homeservers`) muss eine solche Überschreibung dieselben Inventar-Inhalte nutzen, sonst findet das Playbook in Semaphore keine Hosts. Nach jedem Merge, der Hostnamen oder Gruppen ändert, läuft `make semaphore-bootstrap`.

## Variablen

Defaults aus `defaults/main.yml`. Überschreiben in `group_vars/` oder `host_vars/`.

| Variable | Default | Bedeutung |
|---|---|---|
| `semaphore_api_base` | `https://semaphore-api.tech.homeserver` | – |
| `semaphore_namespace` | `semaphore` | – |
| `semaphore_ansible_repo_url` | `https://github.com/pkr-lab/capulus-ansible.git` | – |
| `semaphore_admin_password_path` | `/etc/semaphore-secrets/admin_password` | – |
| `semaphore_admin_username` | `admin` | – |
| `semaphore_ssh_private_key_path` | `/etc/semaphore-secrets/id_ed25519` | – |
| `semaphore_api_ready_retries` | `60` | – |
| `semaphore_api_ready_delay` | `5` | – |
| `semaphore_vault_password` | `` | – |
| `semaphore_ssh_login` | `ubuntu` | – |
| `semaphore_default_keys` | `[{login: '{{ semaphore_ssh_login }}', name: semaphore-ssh-key, type: ssh}, {name: git-n…` | – |
| `semaphore_default_environment` | `{env: {ANSIBLE_VAULT_PASSWORD_FILE: /var/run/semaphore-secrets/ansible_vault_password},…` | – |
| `semaphore_projects` | `[{description: Deploy / re-provision the home-server itself., environments: ['{{ semaph…` | – |
| `semaphore_bootstrap_delete_orphan_projects` | `true` | löscht Semaphore-Projekte, die nicht in `semaphore_projects` stehen; `false` lässt von Hand angelegte Projekte stehen |

## Ablauf

1. vorhandene environments im projekt {{ project_spec.name }} auflisten
2. feststellen ob environment {{ env_spec.name }} existiert
3. id des vorhandenen environments {{ env_spec.name }} ermitteln
4. environment {{ env_spec.name }} anlegen
5. environment {{ env_spec.name }} aktualisieren
6. vorhandene inventories im projekt {{ project_spec.name }} auflisten
7. feststellen ob inventory {{ inventory_spec.name }} existiert
8. id des vorhandenen inventorys {{ inventory_spec.name }} ermitteln
9. inventory {{ inventory_spec.name }} anlegen
10. inventory {{ inventory_spec.name }} aktualisieren
11. mit http-details abbrechen, wenn das inventory-update fehlschlug
12. vorhandene keys im projekt {{ project_spec.name }} auflisten
13. feststellen ob der key im projekt {{ project_spec.name }} existiert
14. privaten ssh-schlüssel von semaphore von der platte lesen
15. ssh-zugangsschlüssel im projekt {{ project_spec.name }} anlegen
16. mit sichtbaren http-details abbrechen, wenn das anlegen des ssh-schlüssels fehlschlug
17. id des vorhandenen ssh-schlüssels {{ key_spec.name }} ermitteln
18. ssh-schlüssel {{ key_spec.name }} im projekt {{ project_spec.name }} aktualisieren
19. zugangsschlüssel vom typ none im projekt {{ project_spec.name }} anlegen
20. schlüssel vom typ login_password im projekt {{ project_spec.name }} anlegen
21. mit http-details abbrechen, wenn das anlegen des vault-schlüssels fehlschlug
22. id des vorhandenen login_password-schlüssels {{ key_spec.name }} ermitteln
23. login_password-schlüssel {{ key_spec.name }} im projekt {{ project_spec.name }} aktualisieren
24. admin-passwort von der platte lesen
25. admin-passwort als fakt setzen
26. bei semaphore anmelden
27. abbrechen, wenn semaphore die admin-zugangsdaten ablehnt
28. session-cookie übernehmen
29. zeilenumbruch als fakt für den pem-body setzen
30. vorabprüfung — semaphore_vault_password darf nicht leer sein
31. warten bis das semaphore-deployment angelegt ist
32. warten bis das semaphore-deployment ausgerollt ist
33. warten bis die http-api von semaphore antwortet
34. bei semaphore authentifizieren
35. jedes semaphore-projekt einrichten
36. alle projekte für das aufräumen verwaister projekte auflisten
37. verwaiste projekte löschen, die nicht in semaphore_projects stehen (nur mit `semaphore_bootstrap_delete_orphan_projects: true`)
38. zusammenfassung des semaphore-bootstraps anzeigen
39. vorhandene projekte für {{ project_spec.name }} auflisten
40. projekt-id für {{ project_spec.name }} ermitteln
41. projekt {{ project_spec.name }} anlegen
42. neue projekt-id für {{ project_spec.name }} übernehmen
43. standard-zugangsschlüssel im projekt {{ project_spec.name }} einrichten
44. keys im projekt {{ project_spec.name }} erneut auflisten
45. zuordnung key-name→id für das projekt {{ project_spec.name }} bilden
46. repository für das projekt {{ project_spec.name }} einrichten
47. inventories für das projekt {{ project_spec.name }} einrichten
48. inventories im projekt {{ project_spec.name }} erneut auflisten
49. zuordnung inventory-name→id für das projekt {{ project_spec.name }} bilden
50. environments für das projekt {{ project_spec.name }} einrichten
51. environments im projekt {{ project_spec.name }} erneut auflisten
52. zuordnung environment-name→id für das projekt {{ project_spec.name }} bilden
53. templates für das projekt {{ project_spec.name }} einrichten
54. templates im projekt {{ project_spec.name }} erneut auflisten
55. zuordnung template-name→id für das projekt {{ project_spec.name }} bilden
56. schedules für das projekt {{ project_spec.name }} einrichten
57. vorhandene repositories im projekt {{ project_spec.name }} auflisten
58. repository-id im projekt {{ project_spec.name }} ermitteln
59. sicherstellen dass der key git-none im projekt {{ project_spec.name }} existiert
60. repository im projekt {{ project_spec.name }} anlegen
61. neue repository-id für das projekt {{ project_spec.name }} übernehmen
62. vorhandene schedules im projekt {{ project_spec.name }} auflisten
63. feststellen ob schedule '{{ schedule_spec.name }}' existiert
64. id des vorhandenen schedules '{{ schedule_spec.name }}' ermitteln
65. sicherstellen dass template '{{ schedule_spec.template }}' für schedule '{{ schedule_spec.name }}' existiert
66. schedule '{{ schedule_spec.name }}' anlegen
67. schedule '{{ schedule_spec.name }}' aktualisieren
68. vorhandene templates im projekt {{ project_spec.name }} auflisten
69. feststellen ob template {{ template_spec.name }} existiert
70. id des vorhandenen templates {{ template_spec.name }} ermitteln
71. sicherstellen dass inventory '{{ template_spec.inventory }}' im projekt {{ project_spec.name }} existiert
72. template {{ template_spec.name }} anlegen
73. template {{ template_spec.name }} aktualisieren

---

## Fehlersuche

Allgemeines Vorgehen bei Rollen: `ansible-playbook <playbook>.yml --limit <host> --tags <tag> --check --diff -vvv` zeigt Änderungen ohne sie anzuwenden (`make <ziel>-check` bei den Playbooks mit Dry-Run). Die Rollenübersicht mit Make-Zielen: [Rollen](index.md#fehlersuche-je-rolle).

```bash
kubectl -n semaphore get pods
kubectl -n semaphore logs deploy/semaphore --tail=100
ansible-playbook <playbook>.yml --tags semaphore-bootstrap -vvv
```

| Symptom | Ursache | Lösung |
|---|---|---|
| `semaphore_vault_password is empty …` | Vault-Passwort fehlt | in `group_vars/all.yml` (verschlüsselt) setzen |
| `Semaphore returned status … on login …` | Admin-Passwort stimmt nicht mehr (in der UI geändert) oder Pod nicht bereit | Admin-Benutzer im Pod zurücksetzen oder PVC leeren und erneut ausführen |
| `SSH key creation returned HTTP …` / `Vault key creation returned HTTP …` | API-Fehler beim Anlegen der Schlüssel | Antwort im Fehlertext lesen, Semaphore-Version prüfen |
| `Inventory '…' update failed: HTTP …` | API-Fehler beim Aktualisieren des Inventars | Antwort lesen |
| `Key 'git-none' not found …` | `git-none` fehlt in `semaphore_default_keys` | Liste ergänzen |
| `Template '…' references unknown inventory …` | Template verweist auf unbekanntes Inventory | Namen im Projekt abgleichen (Fehler listet die vorhandenen) |
| `Schedule '…' references unknown template …` | Zeitplan verweist auf unbekanntes Template | wie oben |

Betrieb: [Semaphore](../b-kubernetes-gitops/b0030-semaphore.md#troubleshooting).

Allgemeine Fehlersuche: [Host-Troubleshooting](../a-betriebssystem/a0040-troubleshooting.md), [zentrale Fehlersuche](../../capulus-core/2-betrieb-hardware/20070-troubleshooting-zentral.md).

[Alle Rollen](index.md)
