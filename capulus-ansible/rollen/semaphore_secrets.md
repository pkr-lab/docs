# Rolle `semaphore_secrets`

Bootstrap-Secret für den Semaphore-Pod.

| | |
|---|---|
| Läuft gegen | homeserver |
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

1. Ensure semaphore state directory exists
2. Check whether the admin password file already exists
3. Generate a random admin password (first run only)
4. Persist the admin password on disk (first run only)
5. Read admin password from disk
6. Expose admin password as a fact
7. Check whether the access key encryption file already exists
8. Generate the access key encryption key (first run only)
9. Lock down the access key encryption file
10. Read access key encryption key from disk
11. Expose access key encryption key as a fact
12. Generate Semaphore SSH keypair (first run only)
13. Lock down SSH private key permissions
14. Lock down SSH public key permissions
15. Read Semaphore SSH private key
16. Read Semaphore SSH public key
17. Expose SSH keys as facts (hostvars for downstream roles)
18. Ensure semaphore namespace exists
19. Render bootstrap Secret manifest
20. Apply bootstrap Secret to the cluster
21. Remove temporary Secret manifest
22. Display Semaphore bootstrap summary

[Alle Rollen](index.md)
