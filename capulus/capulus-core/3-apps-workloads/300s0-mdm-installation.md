# MDM — Installation und Deployment

Was beim Deployment entsteht, wo es im Repo liegt und in welcher Reihenfolge die Erstinstallation läuft. Überblick und Architektur: [300r0](300r0-mdm.md). Die Apple-Schritte stehen in [300t0](300t0-mdm-apple-setup.md).

> **Cluster:** TECH · Ordner `argocd/apps/tech/mdm/` · Namespace `mdm`. GitOps: Ein Merge nach `main` rollt aus. Es wird nichts von Hand mit `kubectl apply` angewendet.

---

## Was deployt wird

| Ressource | Name | Zweck |
|---|---|---|
| Deployment, Service | `mdm-postgres` | PostgreSQL 17, Daten auf `local-path` (`mdm-postgres-data`, 5 Gi), fest auf `homeserver` |
| Deployment, Service | `mdm-nanomdm` | NanoMDM, Port 9000 |
| Deployment, Service | `mdm-depserver` | NanoDEP-Server, Port 9001 |
| Deployment | `mdm-depsyncer` | NanoDEP-Syncer, wartet im Init-Container, bis ein DEP-Token hinterlegt ist |
| Deployment, Service, PVC | `mdm-scep`, `mdm-scep-depot` | SCEP-Server, Depot (CA, Seriennummern, ausgestellte Zertifikate) auf `local-path` |
| Deployment, Service, ConfigMap | `mdm-enroll`, `mdm-enroll-nginx` | liefert `/mdm/enroll`, entsteht erst, wenn das Enrollment-Profil versiegelt ist |
| Ingress | `mdm` | Hosts `mdm-prod.pke-lab.de` und `mdm.prod.homeserver`, Pfade `/mdm` (exakt), `/mdm/enroll` (exakt), `/scep` |
| CronJob, PVC | `mdm-backup`, `mdm-restic` | täglich 02:40, restic auf das NAS (`nas`, 5 Gi) |
| NetworkPolicies | `default-deny` und je Pfad eine Freigabe | siehe [300w0](300w0-mdm-sicherheit.md#networkpolicies) |
| SealedSecrets | `mdm-secrets`, `mdm-scep-ca`, `mdm-enrollment-profile` | alle Geheimnisse, nichts im Klartext im Repo |
| ConfigMaps | `mdm-postgres-init`, `mdm-postgres-files`, `mdm-scep-ca` | Init-Skript und Schemas, öffentliches SCEP-CA-Zertifikat |

Requests und Limits sind klein gehalten (alle Pods zusammen im Leerlauf wenige hundert MB RAM). Die Pods laufen ohne Root, mit schreibgeschütztem Root-Dateisystem, ohne Capabilities und ohne ServiceAccount-Token. Ausnahme: der Backup-Job läuft als Root, weil er zur Laufzeit Pakete installiert (wie der n8n-Backup-Job).

---

## Skripte

Alle Skripte liegen in `argocd/apps/tech/mdm/scripts/` und sind kommentarfrei. Gemeinsame Hilfsfunktionen: `lib.sh`. Lokale Ablage der Geheimnisse außerhalb von Git:

| Verzeichnis | Inhalt |
|---|---|
| `~/mdm-secrets/credentials.env` | alle erzeugten Passwörter und API-Schlüssel (Klartext, `chmod 600`) |
| `~/mdm-secrets/scep-ca/` | Schlüssel und Zertifikat der SCEP-CA |
| `~/mdm-apple/` | Apple-Dateien: `apns/` (Push-Schlüssel, CSR), `dep-public-cert.pem`, `enroll.mobileconfig` |

Beides in den Passwortmanager legen und sichern (ohne `ca.key` müssen alle iPads neu enrollen, ohne das restic-Passwort sind alle Backups unlesbar).

| Skript | Zweck |
|---|---|
| `mdm-init-secrets.sh` | erzeugt alle Zugangsdaten und die SCEP-CA, versiegelt sie nach `values.yaml` (einmalig) |
| `mdm-apns.sh request <mail>` | Push-Schlüssel und CSR erzeugen, an mdmcert.download senden |
| `mdm-apns.sh decrypt <Anhang>` | die signierte Anfrage aus der mdmcert-Mail entschlüsseln (`push.req`) |
| `mdm-dep-keypair.sh` | erzeugt das NanoDEP-Schlüsselpaar und schreibt das öffentliche Zertifikat für Apple Business |
| `mdm-enrollment-profile.sh <push.pem>` | baut das Enrollment-Profil aus Topic, SCEP-URL und MDM-URL und versiegelt es |
| `mdm-render-profile.sh` | rendert das Sperrbildschirm-Profil lokal (Test, Apple Configurator) |
| `mdm-dep-profile.sh` | definiert das DEP-Enrollment-Profil neu und setzt es als Assigner |
| `mdm-n8n-setup.sh` | legt die n8n-Credentials an und importiert die Workflows per API |

---

## Erstinstallation

1. **Zugangsdaten erzeugen und versiegeln** (einmalig, Zugriff auf den TECH-Cluster nötig; ist im Arbeitsstand dieses PRs schon geschehen, dann überspringen. Das Skript überschreibt nie vorhandene Zugangsdaten):
   ```bash
   argocd/apps/tech/mdm/scripts/mdm-init-secrets.sh
   ```
   Das Skript schreibt `~/mdm-secrets/` und trägt die versiegelten Werte unter `secrets:` in `values.yaml` ein (`mdm-secrets`, `mdm-scep-ca`) sowie das öffentliche CA-Zertifikat unter `scep.caPem`. Es weigert sich, vorhandene Zugangsdaten zu überschreiben.
2. **Sperrbildschirm-Text** in `values.yaml` unter `automation.lockScreenText` setzen. Der Wert ist nur der Startwert: Er wird beim ersten Start der Datenbank übernommen. Spätere Änderungen laufen über das n8n-Formular ([300u0](300u0-mdm-automatisierung.md#2-lock-screen-text-ändern)).
3. **Pull Requests in dieser Reihenfolge mergen:**
   1. `capulus-ansible`: Namespace `mdm` in `argocd_workloads_apps`, `argocd_network_policy_refined_namespaces` und der Freigabe `n8n` ↔ `mdm` in `argocd_network_policy_extra_ingress`.
   2. `capulus-core`: Chart, Workflows, `argocd/bootstrap/projects.yaml` (erlaubt den Namespace im AppProject `tech`, von `make render-bootstrap` erzeugt), `build-images.yml`, Token-Register, n8n-Umgebung.
   3. `pkr-lab-docs`: diese Docs.
4. **Netzwerkrichtlinien aus Ansible anwenden**, damit n8n und `mdm` sich erreichen (`tier-default-ingress` in `mdm`, Freigabe `mdm` → `n8n`):
   ```bash
   make argocd
   ```
   im Repo `capulus-ansible` (fragt das Vault-Passwort).
5. **SCEP-Image** abwarten (siehe unten). Bis der Bump-PR gemergt ist, hängt nur `mdm-scep` in `ImagePullBackOff`, alles andere läuft.
6. **Prüfen:**
   ```bash
   kubectl -n mdm get pods,ingress,networkpolicy,cronjob
   kubectl -n mdm logs deploy/mdm-nanomdm
   ```
   `mdm-depsyncer` bleibt in `Init:0/1`, bis der DEP-Token hochgeladen ist, und `mdm-enroll` fehlt, bis das Enrollment-Profil versiegelt ist. Beides ist so gewollt.
7. **n8n einrichten:**
   ```bash
   N8N_API_KEY=<API-Schlüssel> N8N_ACTIVATE_MATCH='Verlaengerung|Neues Geraet|Lock-Screen|Backup-Check' argocd/apps/tech/mdm/scripts/mdm-n8n-setup.sh --activate
   ```
   Der Schlüssel kommt aus n8n (Einstellungen → n8n API) und wird nirgends gespeichert. Das Skript legt die Credentials `MDM NanoMDM API`, `MDM NanoDEP API`, `MDM Formulare` und `MDM Automation DB` an, importiert alle Workflows (vorhandene werden nach Namen aktualisiert) und aktiviert zunächst nur Formular, Geräte-Workflow, Lock-Screen-Workflow und Backup-Check. Die übrigen (Ablauf-Überwachung, DEP-Sync-Kontrolle, Healthcheck) würden vor dem Apple-Setup nur Alarme für noch nicht eingerichtete Teile senden und werden am Ende von [300t0](300t0-mdm-apple-setup.md#7-restliche-n8n-workflows-aktivieren) aktiviert. Das Passwort für die n8n-Formulare steht danach in `~/mdm-secrets/credentials.env` (`form-password`, Benutzer `mdm-admin`).
8. **Apple-Schritte** nach [300t0](300t0-mdm-apple-setup.md).

### SCEP-Image

`micromdm/scep` veröffentlicht kein Container-Image. `argocd/apps/tech/mdm/image/scep/Dockerfile` baut `scepserver` aus dem Tag `v2.3.0` und läuft über `build-images.yml` (App `mdm-scep`, Image `ghcr.io/pkr-lab/mdm-scep:sha-<commit>`). Nach dem Merge öffnet der Workflow den Bump-PR für `image.tag` in `values.yaml`; **dessen Merge** rollt das Image aus. Bis dahin steht `sha-0000000` als Platzhalter in der `values.yaml`. GHCR-Pakete sind anfangs privat, bei `ImagePullBackOff` die Sichtbarkeit prüfen ([f0060](../f-cicd-automatisierung/f0060-build-images.md)).

---

## Neu deployen und ändern

| Änderung | Vorgehen |
|---|---|
| Ressourcen, Images, Ingress, NetworkPolicies | `values.yaml` oder Template ändern, PR, Merge, ArgoCD synchronisiert |
| Upstream-Version (NanoMDM, NanoDEP) | Tag und Digest in `values.yaml` anheben und die Release-Notes lesen. Bei Schemaänderungen die nummerierten `schema.*.sql`-Dateien der neuen Version von Hand ausführen ([300v0](300v0-mdm-betrieb.md#updates)) |
| Sperrbildschirm-Text | n8n-Formular, nicht `values.yaml` |
| Enrollment-Profil (Topic, URL) | `mdm-enrollment-profile.sh <push.pem>` erneut ausführen und den PR mergen. Alle neuen Enrollments bekommen das neue Profil |
| Workflow ändern | JSON in `argocd/apps/tech/n8n/workflows/` ändern, `mdm-n8n-setup.sh` erneut ausführen (aktualisiert vorhandene Workflows nach Namen) |
| DEP-Enrollment-Profil ändern | `profiles/dep-enrollment-profile.json` ändern, `mdm-dep-profile.sh` ausführen |

Die Datenbank wird nur bei einem **leeren** Volume initialisiert (Rollen, Schemas, Tabellen der Automatisierung, Startwerte). Änderungen an `files/*.sql` oder an `automation.*` in `values.yaml` wirken nicht auf eine vorhandene Datenbank.

---

## Konfiguration (`values.yaml`)

| Schlüssel | Bedeutung | Standard |
|---|---|---|
| `publicHost`, `internalHost` | Hostnamen des Ingress | `mdm-prod.pke-lab.de`, `mdm.prod.homeserver` |
| `image.tag` | Tag des SCEP-Images (Bump-PR) | `sha-…` |
| `nanomdm.image.tag`, `nanodep.image.tag` | gepinnte Upstream-Versionen mit Digest | siehe Datei |
| `nanodep.depName` | Name des DEP-Servers in NanoDEP (Teil der API-URLs) | `capulus` |
| `nanodep.syncer.intervalSeconds` | Abstand der Synchronisation mit Apple Business | `600` |
| `scep.crtValidDays` | Gültigkeit der Geräte-Identitätszertifikate | `3650` |
| `scep.allowRenewDays` | `0`: ein Gerät darf sich jederzeit neu registrieren (Zurücksetzen und erneut einrichten) | `0` |
| `scep.caPem` | öffentliches Zertifikat der SCEP-CA (schreibt `mdm-init-secrets.sh`) | — |
| `automation.lockScreenText` | Startwert der Sperrbildschirm-Nachricht (nur bei leerem Volume) | Platzhaltertext |
| `networkPolicy.enabled` | NetworkPolicies im Namespace | `true` |
| `backup.schedule`, `backup.keep*` | Zeitplan und Aufbewahrung des Backups | `40 2 * * *`, 7 täglich, 4 wöchentlich, 6 monatlich |
| `secrets.*.encryptedData` | versiegelte Werte (`mdm-init-secrets.sh`, `mdm-enrollment-profile.sh`) | — |

---

## Troubleshooting

| Symptom | Hinweis |
|---|---|
| `mdm-scep` in `ImagePullBackOff` | Bump-PR des Images noch offen oder GHCR-Paket privat |
| `mdm-depsyncer` in `Init:0/1` | normal, bis der DEP-Token hochgeladen ist (Log des Init-Containers: `kubectl -n mdm logs deploy/mdm-depsyncer -c wait-for-dep-token`) |
| Pods `CreateContainerConfigError` | SealedSecret fehlt oder wurde für einen anderen Controller versiegelt: `kubectl -n mdm get sealedsecret,secret` |
| ArgoCD: Namespace `mdm` nicht erlaubt | `argocd/bootstrap/projects.yaml` noch nicht synchronisiert, erneut synchronisieren |
| n8n erreicht NanoMDM nicht | `make argocd` (tier-default-ingress mit Freigabe für n8n) noch nicht gelaufen |
| Datenbank-Init schlägt fehl | `kubectl -n mdm logs deploy/mdm-postgres`. Nach dem Beheben das Volume `mdm-postgres-data` leeren lassen (nur bei leerer Installation) |
