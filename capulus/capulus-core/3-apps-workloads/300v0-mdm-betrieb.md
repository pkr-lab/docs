# MDM — Betrieb

Jährliche Verlängerung, Änderungen im laufenden Betrieb, Backup und Restore, Updates, Fehlersuche und Notfälle. Überblick: [300r0](300r0-mdm.md), Installation: [300s0](300s0-mdm-installation.md).

> **Cluster:** TECH · Namespace `mdm` · `kubectl`-Befehle gelten für den TECH-Cluster ([Zugriff je Cluster](../../capulus-ansible/a-betriebssystem/a0010-overview.md#kubectl-zugriff-je-cluster)).

---

## Jährliche Verlängerung

Zwei Dinge laufen nach einem Jahr ab. n8n erinnert dafür selbst (30, 14, 7, 1 Tag vorher per ntfy, bei 7 und 1 Tag zusätzlich als Zammad-Ticket, [300u0](300u0-mdm-automatisierung.md#3-ablauf-überwachung)) und schickt die Checkliste mit.

### APNs-Push-Zertifikat

1. `MDMCERT_API_KEY=<Schlüssel> argocd/apps/tech/mdm/scripts/mdm-apns.sh request <registrierte-E-Mail>` (neues Schlüsselpaar und neue Anfrage in `~/mdm-apple/apns/`; ein altes Verzeichnis vorher wegsichern).
2. Signierte Anfrage aus der Mail speichern und mit `mdm-apns.sh decrypt <Anhang>` entschlüsseln (`push.req`).
3. `https://identity.apple.com` mit **derselben Apple Account** öffnen, mit der das Zertifikat erstellt wurde. Beim vorhandenen Zertifikat **Renew** wählen (nicht „Create a Certificate“), `push.req` hochladen, `push.pem` herunterladen.
4. Im Formular `https://n8n.prod.homeserver/form/mdm-verlaengerung` **Push-Zertifikat erneuern** wählen und `push.pem` mit dem **neuen** `push.key` hochladen. Das Formular lehnt ab, wenn das Topic vom bisherigen abweicht.

Wichtig: Nur mit **Renew** und derselben Apple Account bleibt das Topic (`com.apple.mgmt.External.…`) gleich. Bei einem neuen Topic müssten alle iPads neu enrollt werden.

### DEP-Token

1. Apple Business: **Einstellungen → MDM-Server → Capulus MDM → Token herunterladen** (`.p7m`).
2. Im Formular **DEP-Token erneuern** wählen und die Datei hochladen. Das Formular zeigt das neue Ablaufdatum.

Gleicher Weg bei Nutzungsbedingungen: Apple verlangt nach geänderten Bedingungen ein erneutes Akzeptieren in Apple Business, bis dahin liefert die DEP-API Fehler.

---

## Sperrbildschirm-Text ändern

Formular `https://n8n.prod.homeserver/form/mdm-lockscreen`. Ablauf und Ergebnis: [300u0](300u0-mdm-automatisierung.md#2-lock-screen-text-ändern). Den aktuellen Stand zeigt:

```bash
kubectl -n mdm exec deploy/mdm-postgres -- psql -U postgres -d nanomdm -c "select name, version, params->>'TEXT' as text from automation.profile_state"
kubectl -n mdm exec deploy/mdm-postgres -- psql -U postgres -d nanomdm -c "select state, count(*) from automation.device_profile group by 1"
```

---

## Backup und Restore

Der CronJob `mdm-backup` läuft täglich um 02:40 und schreibt in ein restic-Repository auf dem NAS (`nas`, PVC `mdm-restic`). Aufbewahrung: 7 täglich, 4 wöchentlich, 6 monatlich. Gesichert wird:

| Was | Inhalt |
|---|---|
| `nanomdm.dump` | Geräte, Befehle, Push-Zertifikat samt Schlüssel (`push_certs`), Automatisierungszustand (`automation.*`) |
| `nanodep.dump` | DEP-Tokens, Schlüsselpaar für Apple Business (`dep_names`), Syncer-Cursor, Assigner |
| `/scep-depot` | CA-Zertifikat und -Schlüssel, Seriennummern, ausgestellte Geräte-Zertifikate |

Der Workflow `MDM: Backup-Check` alarmiert, wenn das letzte erfolgreiche Backup älter als 24 Stunden ist. Das restic-Passwort liegt in `~/mdm-secrets/credentials.env` (`restic-password`) und im Passwortmanager. Allgemeines zu den NAS-Backups: [20010](../2-betrieb-hardware/20010-nas-backup.md).

Manuell auslösen und prüfen:

```bash
kubectl -n mdm create job --from=cronjob/mdm-backup mdm-backup-manuell
kubectl -n mdm logs -f job/mdm-backup-manuell
kubectl -n mdm delete job mdm-backup-manuell
```

### Restore

Gilt für beide Fälle (Datenbank beschädigt, Volume verloren). Bei verlorenem Volume legt der Pod `mdm-postgres` mit leerem Volume zuerst leere Datenbanken und Schemas an, danach wird der Inhalt eingespielt.

1. Wiederherstellungs-Pod starten (mountet das restic-Repository, das SCEP-Depot und die Zugangsdaten):
   ```bash
   kubectl -n mdm apply -f - <<'EOF'
   apiVersion: v1
   kind: Pod
   metadata:
     name: mdm-restore
     labels:
       app.kubernetes.io/component: backup
   spec:
     restartPolicy: Never
     nodeSelector:
       kubernetes.io/hostname: homeserver
     containers:
       - name: restore
         image: alpine:3.24.3
         command: ["sleep", "3600"]
         env:
           - name: RESTIC_REPOSITORY
             value: /repo
           - name: PGHOST
             value: mdm-postgres.mdm.svc.cluster.local
           - name: RESTIC_PASSWORD
             valueFrom: {secretKeyRef: {name: mdm-secrets, key: restic-password}}
           - name: PGPASSWORD
             valueFrom: {secretKeyRef: {name: mdm-secrets, key: postgres-password}}
         volumeMounts:
           - {name: repo, mountPath: /repo}
           - {name: depot, mountPath: /scep-depot}
     volumes:
       - name: repo
         persistentVolumeClaim: {claimName: mdm-restic}
       - name: depot
         persistentVolumeClaim: {claimName: mdm-scep-depot}
   EOF
   ```
   Das Label `app.kubernetes.io/component: backup` ist nötig: Nur damit greifen die NetworkPolicies des Backup-Jobs (Zugriff auf PostgreSQL, Paketquellen).
2. Im Pod: Werkzeuge installieren und den letzten Stand holen:
   ```bash
   kubectl -n mdm exec -it mdm-restore -- sh
   apk add --no-cache postgresql17-client restic
   restic snapshots --host mdm
   restic restore latest --host mdm --target /tmp/restore
   ```
3. Datenbanken einspielen (ersetzt vorhandene Tabellen):
   ```bash
   pg_restore -U postgres --clean --if-exists --no-owner --role=nanomdm -d nanomdm /tmp/restore/stage/nanomdm.dump
   pg_restore -U postgres --clean --if-exists --no-owner --role=nanodep -d nanodep /tmp/restore/stage/nanodep.dump
   ```
4. SCEP-Depot zurückkopieren (nur bei verlorenem Depot):
   ```bash
   cp -a /tmp/restore/scep-depot/. /scep-depot/
   ```
5. Pod löschen und die Dienste neu starten:
   ```bash
   kubectl -n mdm delete pod mdm-restore
   kubectl -n mdm rollout restart deploy/mdm-nanomdm deploy/mdm-depserver deploy/mdm-depsyncer deploy/mdm-scep
   ```
6. Prüfen: `kubectl -n mdm logs deploy/mdm-nanomdm`, im n8n-Formular das Ablaufdatum (Workflow 3 läuft täglich, oder `select topic from push_certs`), ein Test-Check-in.

Der Backup-Pfad wurde mit restic und `pg_restore -l` geprüft. Ein vollständiger Restore in den Cluster ist als Übung im Rahmen des ersten Betriebs durchzuführen.

---

## Nachricht an die iPads einrichten

Eine freie Mitteilung auf den iPads läuft über ntfy ([300u0, Workflow 8](300u0-mdm-automatisierung.md#8-nachricht-an-ipads)). Einmalig vorbereiten:

1. Eigenen ntfy-Benutzer mit minimalen Rechten anlegen. Der bestehende Benutzer `reader` darf alle `Home-Lab*`-Topics lesen und ist für die iPads zu weit gefasst:
   ```bash
   kubectl -n ntfy exec -it deploy/ntfy -- ntfy user add ipad
   kubectl -n ntfy exec deploy/ntfy -- ntfy access ipad Home-Lab-iPad read-only
   ```
   Das Passwort in den Passwortmanager legen. `publisher` darf schon auf `Home-Lab*` schreiben, dafür ist nichts weiter nötig. Zugangsdaten und Rotation allgemein: [10010](../1-benachrichtigungen/10010-ntfy.md#authentifizierung-und-zugangsdaten).
2. Auf jedem iPad die App **ntfy** aus dem App Store installieren.
3. In der App: Einstellungen → Benutzer verwalten → Benutzer `ipad` mit dem Passwort für `https://ntfy-tech.pke-lab.de` hinzufügen.
4. Thema abonnieren: Server `https://ntfy-tech.pke-lab.de`, Thema `Home-Lab-iPad`. Mitteilungen für die App erlauben.
5. Mitteilungen testen: Formular `https://n8n.prod.homeserver/form/mdm-nachricht`.

Die App muss auf dem iPad installiert und das Thema abonniert sein, damit eine Mitteilung ankommt. Eine Verteilung der App per MDM ist nicht eingerichtet (sie bräuchte Lizenzen über Apple Business).

---

## iPad-Liste pflegen

Welche iPads es gibt und zu welcher Gruppe sie gehören, steht in der Datenbank `nanomdm`, nicht in den Workflows: `automation.ipad_group` (Schlüssel, Name, optionales ntfy-Topic) und `automation.ipad` (Seriennummer, Anzeigename, Gruppe, `enabled`). Hat ein iPad einen Namen (`name`, zum Beispiel `Seepferdchen`), ist es zusätzlich einzeln wählbar. Die Workflows 10 und 12 lesen die Tabellen bei jedem Lauf. „Alle iPads“ meint weiter alle in NanoMDM angemeldeten Geräte, unabhängig von dieser Liste.

```bash
kubectl -n mdm exec deploy/mdm-postgres -- psql -U postgres -d nanomdm -c "select g.name as gruppe, i.serial_number, i.name, i.enabled from automation.ipad_group g left join automation.ipad i on i.group_key = g.key order by 1, 2"
kubectl -n mdm exec deploy/mdm-postgres -- psql -U postgres -d nanomdm -c "insert into automation.ipad (serial_number, name, group_key) values ('SERIENNUMMER', 'Anzeigename', 'vorstands-tisch')"
kubectl -n mdm exec deploy/mdm-postgres -- psql -U postgres -d nanomdm -c "update automation.ipad set group_key = 'grosses-becken' where serial_number = 'SERIENNUMMER'"
kubectl -n mdm exec deploy/mdm-postgres -- psql -U postgres -d nanomdm -c "update automation.ipad set enabled = false where serial_number = 'SERIENNUMMER'"
```

Eine neue Gruppe oder ein neuer iPad-Name braucht einen Eintrag in der Datenbank und eine Option mit demselben Namen im Formular ([300u0, Workflow 12](300u0-mdm-automatisierung.md#12-auftrag-per-mail-oder-formular)).

**Einmalig auf einer bestehenden Datenbank:** Das Init-Skript legt die Tabellen nur bei leerem Volume an. Auf der laufenden Datenbank einmal einspielen, bevor die geänderten Workflows importiert werden:

```bash
kubectl -n mdm exec -i deploy/mdm-postgres -- psql -U postgres -d nanomdm -v ON_ERROR_STOP=1 <<'EOSQL'
CREATE TABLE IF NOT EXISTS automation.ipad_group (
    key        text PRIMARY KEY,
    name       text NOT NULL UNIQUE,
    ntfy_topic text NOT NULL DEFAULT ''
);
CREATE TABLE IF NOT EXISTS automation.ipad (
    serial_number text PRIMARY KEY,
    name          text NOT NULL DEFAULT '',
    group_key     text REFERENCES automation.ipad_group (key) ON UPDATE CASCADE ON DELETE SET NULL,
    enabled       boolean NOT NULL DEFAULT true,
    added_at      timestamptz NOT NULL DEFAULT now()
);
GRANT SELECT, INSERT, UPDATE, DELETE ON automation.ipad_group, automation.ipad TO mdmauto;
INSERT INTO automation.ipad_group (key, name)
VALUES ('kleines-becken', 'Kleines Becken'),
       ('grosses-becken', 'Grosses Becken'),
       ('vorstands-tisch', 'Vorstands-Tisch')
ON CONFLICT DO NOTHING;
INSERT INTO automation.ipad (serial_number, name, group_key)
VALUES ('FVQ99FKFPX', 'Kleines Becken Gruppe 1', 'kleines-becken'),
       ('JXF7J61RWP', 'Kleines Becken Gruppe 2', 'kleines-becken'),
       ('DYD6FXG02W', 'Seepferdchen', 'grosses-becken'),
       ('GCFG7XT6XJ', 'Bronze', 'grosses-becken'),
       ('JLXJ6KPYVP', 'Silber', 'grosses-becken'),
       ('DN4LTQ402C', 'Gold', 'grosses-becken'),
       ('D2LTC9DDJW', 'Juniorretter', 'grosses-becken'),
       ('DT7HGM9FPG', 'Vorstands-Tisch', 'vorstands-tisch')
ON CONFLICT DO NOTHING;
EOSQL
```

Das Backup (`nanomdm.dump`) enthält die Tabellen mit, ein Restore bringt die Liste zurück.

---

## Updates

| Was | Vorgehen |
|---|---|
| Upstream-Images (NanoMDM, NanoDEP, nginx, Postgres, Alpine) | Renovate öffnet PRs mit Tag und Digest. Vor dem Merge die Release-Notes lesen. NanoMDM und NanoDEP haben bei einzelnen Versionen **Schemaänderungen** (nummerierte `schema.*.sql` im Repo des Projekts). Diese vor dem Rollout der neuen Version mit `psql` im Pod `mdm-postgres` ausführen und die Dateien in `files/` aktualisieren |
| Postgres-Hauptversion (17 → 18) | nicht per Tag-Wechsel: erst Dump, neuen Pod mit leerem Volume, Restore wie oben |
| SCEP-Server | `Dockerfile` in `image/scep/` anpassen (Version in `go install`), PR, danach Bump-PR für `image.tag`. `micromdm/scep` weist selbst darauf hin, dass der Server einfach gehalten ist und künftig vielleicht nicht mehr gepflegt wird ([600d0](../6-hintergruende/600d0-mdm.md#scep-server)) |
| Workflows | JSON im Repo ändern, `mdm-n8n-setup.sh` ausführen |
| Profile | Sperrbildschirm über das Formular, Vorlagen in `profiles/` |

---

## Geheimnisse rotieren

Gesamtliste mit Auswirkung: [300w0](300w0-mdm-sicherheit.md#secrets) und `ops/token-register.yaml`. Einzelne Werte lassen sich ersetzen, weil jeder Schlüssel eines SealedSecrets unabhängig verschlüsselt ist:

```bash
printf '%s' "$(openssl rand -hex 32)" | kubeseal --raw --namespace mdm --name mdm-secrets \
  --controller-namespace sealed-secrets --controller-name sealed-secrets-controller --from-file=/dev/stdin
```

Den Ausgabewert in `values.yaml` unter `secrets.main.encryptedData.<Schlüssel>` einsetzen und die Klartextdatei `~/mdm-secrets/credentials.env` angleichen.

### API-Schlüssel rotieren

Schlüssel `nanomdm-api-key` bzw. `nanodep-api-key` ersetzen (wie oben), PR mergen, `kubectl -n mdm rollout restart deploy/mdm-nanomdm deploy/mdm-depserver deploy/mdm-depsyncer`. Danach das n8n-Passwort im Credential `MDM NanoMDM API` bzw. `MDM NanoDEP API` auf den neuen Wert setzen (n8n-Oberfläche).

### Webhook-Token rotieren

Schlüssel `webhook-token` ersetzen, PR mergen, `NanoMDM` und `depsyncer` neu starten, dann `mdm-n8n-setup.sh` erneut ausführen (nimmt das Token aus `credentials.env` und aktualisiert die Webhook-Pfade der Workflows 1 und 5).

### SCEP-Challenge rotieren

Schlüssel `scep-challenge` ersetzen, `mdm-enrollment-profile.sh ~/mdm-apple/apns/push.pem` erneut ausführen (das Profil enthält die Challenge), beides in einem PR mergen, `mdm-scep` und `mdm-enroll` starten neu. Bereits enrollte iPads sind nicht betroffen, nur neue Enrollments.

### Datenbank-Passwörter rotieren

Neues Passwort in der Datenbank setzen (`ALTER ROLE nanomdm PASSWORD '…'` im Pod `mdm-postgres`), dann den Schlüssel `nanomdm-db-password` (bzw. `nanodep-db-password`, `automation-db-password`, `postgres-password`) ersetzen, PR mergen und die Pods neu starten. Beim Automatisierungs-Passwort zusätzlich das n8n-Credential `MDM Automation DB` aktualisieren.

---

## SCEP-CA erneuern

Die CA ist 20 Jahre gültig, die Geräte-Identitätszertifikate 10 Jahre (`scep.crtValidDays`). Läuft die CA ab oder ist der Schlüssel kompromittiert, müssen **alle iPads neu enrollt werden**, weil das Geräte-Zertifikat von der alten CA stammt.

1. Neue CA erzeugen und versiegeln (die alte CA wird nach `~/mdm-secrets/scep-ca.alt-<Zeitstempel>` verschoben, alle anderen Zugangsdaten bleiben):
   ```bash
   argocd/apps/tech/mdm/scripts/mdm-init-secrets.sh --ca-only
   ```
2. Enrollment-Profil neu versiegeln: `mdm-enrollment-profile.sh ~/mdm-apple/apns/push.pem`.
3. PR mergen. `mdm-scep` und `mdm-nanomdm` starten mit der neuen CA neu (die Checksumme im Pod-Template erzwingt das).
4. Alle iPads zurücksetzen und neu einrichten (automatisches Enrollment). Das Depot behält die Seriennummern; die alte CA nach erfolgreichem Neu-Enrollment sicher löschen.

---

## Troubleshooting

| Symptom | Ursache und Abhilfe |
|---|---|
| iPad zeigt nach der Einrichtung keine Fernverwaltung | Seriennummer nicht in Apple Business dem Server `Capulus MDM` zugeordnet, oder kein DEP-Profil zugewiesen. Prüfen: `kubectl -n mdm logs deploy/mdm-depsyncer`, Workflow 5, `POST /proxy/capulus/devices` |
| iPad meldet „Remote Management konnte nicht geladen werden“ | `curl -i https://mdm-prod.pke-lab.de/mdm/enroll` prüfen (200 und `application/x-apple-aspen-config`). `kubectl -n mdm logs deploy/mdm-enroll` |
| SCEP-Fehler beim Enrollment | `kubectl -n mdm logs deploy/mdm-scep`. Challenge im Profil stimmt nicht mehr (nach Rotation Profil neu versiegeln), CA-Pfad `scep.caPem` |
| Gerät meldet sich bei `/mdm`, NanoMDM lehnt ab | `kubectl -n mdm logs deploy/mdm-nanomdm`. Zertifikat nicht von der SCEP-CA (CA getauscht, Gerät muss neu enrollen) oder fehlender `Mdm-Signature`-Header (Cloudflare reicht ihn normalerweise durch, [600d0](../6-hintergruende/600d0-mdm.md#authentifizierung-der-geräte-hinter-cloudflare)) |
| Push-Fehler `MissingProviderToken` oder 403 | Kein oder falsches Push-Zertifikat in NanoMDM. Formular erneut ausführen, `select topic from push_certs` |
| Push-Fehler `BadDeviceToken`, `Unregistered` | Gerät hat sich abgemeldet oder wurde zurückgesetzt, Eintrag ist veraltet |
| Gerät bekommt das Profil nicht | Workflow 1 prüfen (n8n → Ausführungen). `select * from automation.device_profile`. Status `claimed` seit Minuten = Lauf abgestürzt (wird nach 10 Minuten neu versucht), `failed` = Einreihen scheiterte, `error` = Gerät lehnte ab |
| Formular fragt Zugangsdaten und lehnt ab | Benutzer `mdm-admin`, Passwort `form-password` aus `~/mdm-secrets/credentials.env`. Das Credential `MDM Formulare` in n8n kontrollieren |
| Webhook von NanoMDM kommt nicht an | Workflow 1 aktiv? `kubectl -n mdm logs deploy/mdm-nanomdm` auf Webhook-Fehler prüfen. NetworkPolicy `egress-webhook-to-n8n` und `tier-default-ingress` in `n8n` (Freigabe für `mdm`) prüfen |
| Alle Workflows melden „Datenbank nicht erreichbar“ | `MDM Automation DB`: Host `mdm-postgres.mdm.svc.cluster.local`, Benutzer `mdmauto`. `tier-default-ingress` in `mdm` erlaubt `n8n` |
| `mdm-depsyncer` bleibt in `Init:0/1` | DEP-Token fehlt oder ist ungültig: Formular, dann `kubectl -n mdm logs deploy/mdm-depsyncer -c wait-for-dep-token` |
| Backup-Check alarmiert | `kubectl -n mdm get cronjob,job`, `kubectl -n mdm logs job/<letzter Job>`. Häufig: NAS nicht erreichbar, PVC `mdm-restic` nicht gebunden |

---

## Notfall: Zertifikat abgelaufen

| Was | Auswirkung | Vorgehen |
|---|---|---|
| **APNs-Push-Zertifikat** | NanoMDM kann keine Push-Benachrichtigungen mehr senden. Die iPads bleiben enrollt, melden sich aber nur noch selten von selbst. Befehle wie ein neuer Sperrbildschirm-Text kommen verzögert oder gar nicht an | Im Push-Portal prüfen, ob für das abgelaufene Zertifikat noch **Renew** angeboten wird (Apple erlaubt das nach eigener Aussage nur begrenzte Zeit, das ist im Ernstfall vor Ort zu klären). Dann wie die normale Verlängerung. Gibt es nur noch „Create“, entsteht ein **neues Topic**: Enrollment-Profil neu versiegeln und alle iPads zurücksetzen und neu einrichten |
| **DEP-Token** | Keine neuen Geräte, keine Zuweisung, keine Synchronisation. Bereits enrollte iPads laufen unverändert weiter | Neuen Token in Apple Business herunterladen und über das Formular einspielen. Mehr ist nicht nötig |
| **SCEP-CA** | Neue Enrollments scheitern, Geräte-Zertifikate laufen erst bei CA-Ablauf aus | Siehe [SCEP-CA erneuern](#scep-ca-erneuern) |
| **TLS des öffentlichen Hosts** | Cloudflare erneuert das Zertifikat selbst. Fällt es aus, schlägt Workflow 6 an | Cloudflare-Status und Tunnel prüfen ([e0000](../e-externe-erreichbarkeit/e0000-cloudflare-tunnel.md)) |

Zusätzlich gilt: Die Verlängerung des Push-Zertifikats erst dann als erledigt betrachten, wenn das Formular das neue Ablaufdatum bestätigt hat.
