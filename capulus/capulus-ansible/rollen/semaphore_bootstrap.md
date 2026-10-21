# Rolle `semaphore_bootstrap`

Projekte, Inventories, Templates und Zeitpläne in Semaphore per REST-API.

| | |
|---|---|
| Läuft gegen | homeserver |
| Tags | – |
| Quellcode | [roles/semaphore_bootstrap](https://github.com/pkr-lab/capulus-ansible/tree/main/roles/semaphore_bootstrap) |

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

## Ablauf

1. List existing environments in project {{ project_spec.name }}
2. Decide if environment exists for {{ env_spec.name }}
3. Find existing environment id for {{ env_spec.name }}
4. Create environment {{ env_spec.name }}
5. Update environment {{ env_spec.name }}
6. List existing inventories in project {{ project_spec.name }}
7. Decide if inventory exists for {{ inventory_spec.name }}
8. Find existing inventory id for {{ inventory_spec.name }}
9. Create inventory {{ inventory_spec.name }}
10. Update inventory {{ inventory_spec.name }}
11. Fail with HTTP details if inventory update failed
12. List existing keys in project {{ project_spec.name }}
13. Decide if key exists in project {{ project_spec.name }}
14. Read Semaphore SSH private key from disk
15. Create SSH access key in project {{ project_spec.name }}
16. Fail with visible HTTP details if SSH key creation failed
17. Find existing ssh key id for {{ key_spec.name }}
18. Update SSH key {{ key_spec.name }} in project {{ project_spec.name }}
19. Create None access key in project {{ project_spec.name }}
20. Create login_password key in project {{ project_spec.name }}
21. Fail with HTTP details if vault key creation failed
22. Find existing login_password key id for {{ key_spec.name }}
23. Update login_password key {{ key_spec.name }} in project {{ project_spec.name }}
24. Read admin password from disk
25. Set admin password fact
26. Login to Semaphore
27. Fail if Semaphore rejected the admin credentials
28. Capture session cookie
29. Set newline-char fact for PEM body construction
30. Pre-flight — semaphore_vault_password must not be empty
31. Wait for the semaphore Deployment to be created
32. Wait for the semaphore Deployment to roll out
33. Wait for the Semaphore HTTP API to respond
34. Authenticate against Semaphore
35. Provision each Semaphore project
36. List all projects for orphan cleanup
37. Delete orphan projects not in semaphore_projects
38. Display Semaphore bootstrap summary
39. List existing projects for {{ project_spec.name }}
40. Find project id for {{ project_spec.name }}
41. Create project {{ project_spec.name }}
42. Capture new project id for {{ project_spec.name }}
43. Provision default access keys in project {{ project_spec.name }}
44. Re-list keys in project {{ project_spec.name }}
45. Build key name→id map for project {{ project_spec.name }}
46. Provision repository for project {{ project_spec.name }}
47. Provision inventories for project {{ project_spec.name }}
48. Re-list inventories in project {{ project_spec.name }}
49. Build inventory name→id map for project {{ project_spec.name }}
50. Provision environments for project {{ project_spec.name }}
51. Re-list environments in project {{ project_spec.name }}
52. Build environment name→id map for project {{ project_spec.name }}
53. Provision templates for project {{ project_spec.name }}
54. Re-list templates in project {{ project_spec.name }}
55. Build template name→id map for project {{ project_spec.name }}
56. Provision schedules for project {{ project_spec.name }}
57. List existing repositories in project {{ project_spec.name }}
58. Find repository id in project {{ project_spec.name }}
59. Assert git-none key exists in project {{ project_spec.name }}
60. Create repository in project {{ project_spec.name }}
61. Capture new repository id for project {{ project_spec.name }}
62. List existing schedules in project {{ project_spec.name }}
63. Decide if schedule '{{ schedule_spec.name }}' exists
64. Find existing schedule id for '{{ schedule_spec.name }}
65. Assert template '{{ schedule_spec.template }}' exists for schedule '{{ schedule_spec.name }}
66. Create schedule '{{ schedule_spec.name }}
67. Update schedule '{{ schedule_spec.name }}
68. List existing templates in project {{ project_spec.name }}
69. Decide if template exists for {{ template_spec.name }}
70. Find existing template id for {{ template_spec.name }}
71. Assert inventory '{{ template_spec.inventory }}' exists in project {{ project_spec.name }}
72. Create template {{ template_spec.name }}
73. Update template {{ template_spec.name }}

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
