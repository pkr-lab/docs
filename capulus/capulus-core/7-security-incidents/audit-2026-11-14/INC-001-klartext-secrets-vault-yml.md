# INC-001 – ZURÜCKGEZOGEN: „Unverschlüsselte `vault.yml`“ war ein Fehlalarm

| Feld | Wert |
|---|---|
| ID | INC-001 |
| Datum | 2026-11-14 (angelegt), 2026-11-14 (zurückgezogen) |
| Kategorie | secrets |
| Schweregrad | **keiner** (Fehlalarm) |
| Betroffen | `capulus-ansible/host_vars/*/vault.yml`, `group_vars/all.yml` |

## Was ich ursprünglich behauptet hatte
Die drei `vault.yml` und `group_vars/all.yml` seien im Git unverschlüsselt, weil `head -n1` kein `$ANSIBLE_VAULT;` zeigte und ein Muster-Scan `password:`-Zeilen mit 8-Zeichen-Werten fand. Ich hatte daraus „Klartext-Secrets im Git“ gemacht.

## Warum das falsch war
Das Repo verschlüsselt Secrets **einzeln als `!vault`-Inline-Werte** (`ansible-vault encrypt_string`), nicht ganze Dateien; das steht in `capulus-ansible/konfiguration.md` der Doku („Secrets stehen als `!vault`-Werte direkt in den YAML-Dateien“). Meine Prüfung suchte `$ANSIBLE_VAULT` nur am Zeilenanfang und übersah die eingerückten Blöcke. Die „identischen 8-Zeichen-Werte“ waren der Text `!vault |` selbst (8 Zeichen).

## Nachprüfung (ohne einen Wert auszugeben)
```
group_vars/all.yml                            !vault-Einträge=13  eingerückte $ANSIBLE_VAULT-Blöcke=13
host_vars/worker-1/vault.yml                  !vault-Einträge=1   Blöcke=1
host_vars/controlplane-1/vault.yml            !vault-Einträge=1   Blöcke=1
host_vars/vereinsheim-alarmmonitor/vault.yml  !vault-Einträge=3   Blöcke=3
Schlüssel mit password/token/secret/key im Namen ohne !vault in all.yml: nur öffentliche Schlüssel (wireguard_backup_peer_public_key,
libvirt_host_admin_ssh_public_key) und Semaphore-Verweisnamen (ssh_key:, vault_key:), keine Secrets.
```
`group_vars/user_inventory_vault.yml` ist zusätzlich komplett verschlüsselt (`$ANSIBLE_VAULT;` in Zeile 1).

## Folgen
- Das Skript `INC-001-vault-yml-verschluesseln.sh` ist **gelöscht** (es hätte ganze Dateien verschlüsselt und die Inline-Konvention gebrochen).
- Die `CLAUDE.md` von capulus-ansible nennt `all.yml` „mit ansible-vault verschlüsselt“. Das stimmt nur für die Werte, nicht für die Datei. Das ist eine kleine Unschärfe in der Doku, kein Sicherheitsproblem.
- Nicht geprüft habe ich, ob die verwendeten Vault-Passwörter stark sind, und nicht, ob früher in der Git-Historie Klartext-Secrets standen (die Doku-Incidents 001, 030 und 032 behandeln solche Fälle).
