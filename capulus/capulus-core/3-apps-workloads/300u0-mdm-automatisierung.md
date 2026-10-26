# MDM — Automatisierung mit n8n

Die sieben Workflows, die den MDM-Stack betreiben. Sie liegen als JSON in `argocd/apps/tech/n8n/workflows/mdm-*.json` und laufen in der bestehenden n8n-Instanz ([30070](30070-n8n.md)). Überblick: [300r0](300r0-mdm.md).

> **Cluster:** TECH · n8n im Namespace `n8n` · Zugriff auf NanoMDM, NanoDEP und PostgreSQL im Namespace `mdm` über die Cluster-Dienste (NetworkPolicy und `tier-default-ingress` erlauben genau das).

---

## Import und Credentials

`argocd/apps/tech/mdm/scripts/mdm-n8n-setup.sh` legt fehlende Credentials an, importiert alle `mdm-*.json` (vorhandene Workflows werden nach Namen aktualisiert), ersetzt den Platzhalter `__WEBHOOK_TOKEN__` in den Webhook-Pfaden und ordnet die Credential-IDs zu. Details und Aufruf: [300s0](300s0-mdm-installation.md#erstinstallation). Im Workflow-JSON steht kein Geheimnis, nur Credential-Namen.

| Credential (n8n) | Typ | Inhalt | Verwendet von |
|---|---|---|---|
| `MDM NanoMDM API` | Basic Auth | Benutzer `nanomdm`, Passwort `nanomdm-api-key` | 1, 2, 4 |
| `MDM NanoDEP API` | Basic Auth | Benutzer `depserver`, Passwort `nanodep-api-key` | 3, 4, 5, 6 |
| `MDM Automation DB` | Postgres | Benutzer `mdmauto` auf der Datenbank `nanomdm` (nur Schema `automation` und gesperrte Lesespalten) | 1, 2, 3, 4, 6 |
| `MDM Formulare` | Basic Auth | Benutzer `mdm-admin`, Passwort `form-password` (zufällig, nur lokal in `credentials.env`) | 2, 4 (Formulare) |
| `ntfy-publisher` | Header Auth | bereits vorhanden, schreibt auf Topics `Home-Lab*` ([10010](../1-benachrichtigungen/10010-ntfy.md)) | alle |
| `Zammad EDV-Kretzer` | Zammad Basic Auth | bereits vorhanden | 3 |

Alle Meldungen gehen an das ntfy-Topic **`Home-Lab-MDM`** (in der ntfy-App abonnieren). Die URLs der Dienste stehen je Workflow im Code-Node `Konfiguration`.

Die Code-Nodes nutzen die Node-Module `crypto` und `tls`; sie sind in der n8n-Umgebung freigegeben (`NODE_FUNCTION_ALLOW_BUILTIN: "child_process,crypto,tls"` in `argocd/apps/tech/n8n/values.yaml`).

### Zustand in der Datenbank (Schema `automation`)

| Tabelle | Inhalt |
|---|---|
| `profile_state` | Basisprofile: Name, Version, Vorlage, Parameter (Text, Kennungen). Heute eine Zeile: `lockscreen` |
| `device_profile` | je Gerät und Profil: Version, Status (`claimed`, `queued`, `installed`, `failed`, `error`), Command-UUID, Zeitpunkte. Garantiert die Idempotenz |
| `expiry_notice` | welche Erinnerungsschwellen (30, 14, 7, 1 Tage) für welches Ablaufdatum schon gemeldet wurden |
| `alert_state` | laufende Störungen des Healthchecks (Dedupe und Entwarnung) |
| `setting` | `dep_name`, `dep_profile_json` (Vorlage des DEP-Profils) |

---

## 1. Neues Gerät → Basisprofile

Datei `mdm-neues-geraet-basisprofile.json` · Webhook `POST /webhook/mdm-event-<webhook-token>`.

| | |
|---|---|
| Auslöser | NanoMDM sendet jeden Check-in als Webhook (`NANOMDM_WEBHOOK_URL`). Der Pfad enthält das Token aus dem SealedSecret (`webhook-token`) |
| Ablauf `TokenUpdate` | Nur Gerätekanal (`ids.type = Device`). Ein einziges SQL (`INSERT … ON CONFLICT … RETURNING`) **beansprucht** alle aktiven, noch nicht gesendeten Profile für dieses Gerät. Nur beanspruchte Profile werden gebaut (Platzhalter ersetzen, `PayloadVersion` setzen) und per `PUT /v1/enqueue/<Gerät>` als `InstallProfile` eingereiht. Danach: Status `queued` mit Command-UUID |
| Ablauf `Connect` | Quittung `Acknowledged` → `installed`, `Error`/`CommandFormatError` → `error` |
| Idempotenz | Wiederholte Check-ins finden die Zeile `(Gerät, Profil)` schon vor und beanspruchen nichts. Neu gesendet wird nur bei höherer Profilversion, nach einem Fehlschlag (`failed`) oder wenn ein Anspruch länger als 10 Minuten hängt (abgestürzter Lauf) |
| Weitere Basisprofile | Zeile in `automation.profile_state` einfügen (`enabled = true`): Der Workflow sendet sie ab dem nächsten Check-in jedes Geräts. Vorlage mit Platzhaltern `__NAME__` und `__VERSION__`, Werte in `params` |
| Credentials | `MDM NanoMDM API`, `MDM Automation DB`, `ntfy-publisher` |

Fehlerfälle:

| Fall | Verhalten |
|---|---|
| NanoMDM nicht erreichbar oder lehnt ab | Status `failed`, ntfy-Meldung `MDM: Profil nicht eingereiht`. Der nächste Check-in versucht es erneut |
| Befehl eingereiht, aber APNs-Push schlug fehl | Status bleibt `queued` (der Befehl liegt in der Warteschlange und wird beim nächsten Check-in geholt), ntfy-Meldung mit dem Push-Fehler |
| Gerät lehnt das Profil ab | Status `error`, der Healthcheck (Workflow 6) meldet es |

---

## 2. Lock-Screen-Text ändern

Datei `mdm-lockscreen-text-aendern.json` · Formular `https://n8n.prod.homeserver/form/mdm-lockscreen` (Basic Auth `MDM Formulare`).

| | |
|---|---|
| Auslöser | Formular mit einem Textfeld |
| Ablauf | Text prüfen (1 bis 300 Zeichen), `profile_state.version` um 1 erhöhen und Text speichern, das Profil für **alle** aktiven Geräte beanspruchen (`type = Device`), je Gerät `InstallProfile` einreihen, Status vermerken, Ergebnis aus der Datenbank lesen |
| Ergebnis | Seite im Formular und ntfy-Meldung: „Version N: x von y Geräten eingereiht, z fehlgeschlagen“. Die Quittungen der Geräte vermerkt Workflow 1 |
| Credentials | `MDM Formulare`, `MDM NanoMDM API`, `MDM Automation DB`, `ntfy-publisher` |

Fehlerfälle: leerer oder zu langer Text → Hinweis im Formular, nichts geändert. Geräte, die gerade nicht erreichbar sind, bekommen den Befehl beim nächsten Check-in (er liegt in der Warteschlange).

---

## 3. Ablauf-Überwachung

Datei `mdm-ablauf-ueberwachung.json` · täglich 07:00.

| | |
|---|---|
| Auslöser | Zeitplan |
| Datenquellen | APNs-Push-Zertifikat: Zertifikat aus `public.push_certs` (nur Spalten `topic`, `cert_pem`, `updated_at` lesbar). DEP-Token: `GET /v1/tokens/<name>` (`access_token_expiry`). SCEP-CA: `GET /scep?operation=GetCACert` |
| Schwellen | Erinnerung bei 30, 14, 7 und 1 Tag Restlaufzeit. Wird eine Schwelle verpasst (z. B. erster Lauf bei 5 Tagen), kommt eine Meldung für die kleinste fällige Schwelle |
| Kanäle | ntfy bei jeder Schwelle (ab 7 Tage hohe Priorität, bei 1 Tag dringend). Bei 7 und 1 Tag zusätzlich ein Zammad-Ticket in `Support / Administration` |
| Inhalt | Die fertige Schritt-für-Schritt-Checkliste zur Verlängerung. Für das Push-Zertifikat steht darin ausdrücklich: im Portal **Renew** mit **derselben** Apple Account verwenden, sonst ändert sich das Topic und alle iPads müssen neu enrollt werden |
| Abgelaufen | Täglich eine dringende Meldung mit Hinweis auf den Notfallabschnitt ([300v0](300v0-mdm-betrieb.md#notfall-zertifikat-abgelaufen)) |
| Credentials | `MDM NanoDEP API`, `MDM Automation DB`, `ntfy-publisher`, `Zammad EDV-Kretzer` |

Fehlerfälle: Ein nicht lesbares Ablaufdatum (zum Beispiel noch kein Push-Zertifikat oder DEP-Token hinterlegt) erzeugt täglich die Meldung „nicht prüfbar“ mit der passenden Checkliste. Damit die Meldung nicht vor dem Apple-Setup stört, wird der Workflow erst am Ende von [300t0](300t0-mdm-apple-setup.md#7-restliche-n8n-workflows-aktivieren) aktiviert.

---

## 4. Halbautomatische Verlängerung

Datei `mdm-verlaengerung-einspielen.json` · Formular `https://n8n.prod.homeserver/form/mdm-verlaengerung` (Basic Auth `MDM Formulare`).

| | |
|---|---|
| Auslöser | Formular: Auswahl (Push-Zertifikat erneuern / DEP-Token erneuern / DEP-Token für neuen MDM-Server) und Dateiupload |
| Ablauf Push-Zertifikat | Prüft `push.pem` und `push.key`: gültiges PEM, Schlüssel unverschlüsselt, Schlüssel passt zum Zertifikat, Zertifikat enthält ein Push-Topic, Topic **unverändert** gegenüber dem gespeicherten, Restlaufzeit mindestens 300 Tage. Dann `PUT /v1/pushcert` an NanoMDM, anschließend liest n8n das gespeicherte Zertifikat aus der Datenbank und meldet das neue Ablaufdatum |
| Ablauf DEP-Token | `PUT /v1/tokenpki/<name>` (mit `force=1` bei der dritten Option), dann `GET /v1/tokens/<name>` (Ablaufdatum) und `GET /proxy/<name>/account` (Apple antwortet?). Fehlt der Assigner, legt der Workflow das DEP-Profil aus `automation.setting` an (`POST /proxy/<name>/profile`) und setzt es (`PUT /v1/assigner/<name>`) |
| Ergebnis | Seite im Formular und ntfy-Meldung mit dem neuen Ablaufdatum |
| Credentials | `MDM Formulare`, `MDM NanoMDM API`, `MDM NanoDEP API`, `MDM Automation DB`, `ntfy-publisher` |

Fehlerfälle (jeweils ohne Änderung und mit Klartext im Formular): Schlüssel gehört nicht zum Zertifikat, anderes Topic (falscher Apple-Weg: „Create“ statt „Renew“), altes Zertifikat hochgeladen, Schlüssel passwortgeschützt, Datei fehlt, NanoDEP lehnt den Token ab (bei neuem MDM-Server die dritte Option wählen).

---

## 5. DEP-Sync-Kontrolle

Datei `mdm-dep-sync-kontrolle.json` · Webhook `POST /webhook/mdm-dep-sync-<webhook-token>` und täglich 09:00.

| | |
|---|---|
| Zuweisung | Erledigt der NanoDEP-Assigner (`depsyncer`): Neue Seriennummern bekommen automatisch das DEP-Enrollment-Profil |
| Auslöser Webhook | `depsyncer` meldet jede Synchronisation. Enthält sie neue oder geänderte Geräte, wartet n8n zwei Minuten, fragt die Gerätedetails bei Apple ab (`POST /proxy/<name>/devices`) und prüft, ob ein Profil zugewiesen ist |
| Auslöser täglich | Alle Geräte abrufen (`POST /proxy/<name>/server/devices`, bis 1000) und die ohne Profil melden |
| Meldung | ntfy (hohe Priorität): Seriennummern ohne DEP-Profil, fehlgeschlagener Apple-Abruf, fehlender Assigner |
| Credentials | `MDM NanoDEP API`, `ntfy-publisher` |

Fehlerfälle: Apple-Abruf scheitert (Token abgelaufen) → Meldung mit Hinweis. Geräte, die in Apple Business einem anderen MDM-Server zugeordnet sind, erscheinen nicht im Sync dieses Servers.

---

## 6. Healthcheck

Datei `mdm-healthcheck.json` · alle 10 Minuten.

| Prüfung | Wie |
|---|---|
| Öffentliche Endpunkte | `GET /scep?operation=GetCACaps`, `GET /mdm/enroll`, `GET /mdm` über den öffentlichen Host. Erreichbar heißt: Status 2xx bis 4xx, aber nicht 404 (404 bedeutet, dass die Ingress-Route fehlt) |
| TLS | Verbindung auf Port 443 des öffentlichen Hosts mit Zertifikatsprüfung, Restlaufzeit mindestens 14 Tage |
| Pods | VictoriaMetrics: `kube_deployment_status_replicas_unavailable` im Namespace `mdm`. Der Syncer zählt nur, wenn ein DEP-Token hinterlegt ist |
| Geräteprofile | Anzahl Profile im Status `error`/`failed` sowie `queued` seit über 24 Stunden |

Alarme werden je Prüfung höchstens alle 6 Stunden wiederholt; sobald die Prüfung wieder grün ist, kommt einmal die Entwarnung. Zustand: `automation.alert_state`. Credentials: `MDM NanoDEP API`, `MDM Automation DB`, `ntfy-publisher`.

---

## 7. Backup-Check

Datei `mdm-backup-check.json` · täglich 08:30.

Fragt VictoriaMetrics (kube-state-metrics) ab und meldet per ntfy, wenn eine dieser Bedingungen zutrifft:

- der letzte erfolgreiche Lauf des CronJobs `mdm-backup` ist älter als 24 Stunden oder es gibt keine Metrik,
- Backup-Jobs im Namespace `mdm` sind fehlgeschlagen,
- nicht alle drei Volumes (`mdm-postgres-data`, `mdm-scep-depot`, `mdm-restic`) sind gebunden.

Der CronJob sichert beide Datenbanken per `pg_dump` (damit Push-Zertifikat samt Schlüssel, DEP-Token samt Schlüsselpaar, Geräte und Automatisierungszustand) und das SCEP-Depot (CA-Schlüssel) per restic auf das NAS ([20010](../2-betrieb-hardware/20010-nas-backup.md)). Credentials: `ntfy-publisher`.
