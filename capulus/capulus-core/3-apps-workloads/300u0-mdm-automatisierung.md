# MDM — Automatisierung mit n8n

Die zwölf Workflows, die den MDM-Stack betreiben. Sie liegen als JSON in `argocd/apps/tech/n8n/workflows/mdm-*.json` und laufen in der bestehenden n8n-Instanz ([30070](30070-n8n.md)). Überblick: [300r0](300r0-mdm.md).

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
| `Netcup Automation` | IMAP | bereits vorhanden, Zugang zum Postfach mit dem Ordner `Ausbildung-Ipads` (derselbe wie bei „Instagram-Post per Mail“) | 12 |
| `Zammad EDV-Kretzer` | Zammad Basic Auth | bereits vorhanden | 3 |
| `Apple Apps Token` | Header Auth | Name `Authorization`, Wert `Bearer <Server-Token aus Apple Business, Apps und Bücher>`. Von Hand in n8n anlegen, nicht im Repo | 10, 12 |

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
| Weitere Basisprofile | Zeile in `automation.profile_state` einfügen (`enabled = true`): Der Workflow sendet sie ab dem nächsten Check-in jedes Geräts. Vorlage mit Platzhaltern `__NAME__`, Werte in `params`. Das oberste `PayloadVersion` der Vorlage muss immer `1` sein (Apple), die Versionsnummer der Tabelle bleibt intern |
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

---

## 8. Nachricht an iPads

Datei `mdm-nachricht-an-ipads.json` · Formular `https://n8n.prod.homeserver/form/mdm-nachricht` (Basic Auth `MDM Formulare`).

| | |
|---|---|
| Zweck | Einmalige Mitteilung an alle iPads, die als Push-Benachrichtigung „einfliegt“. MDM kennt keinen Befehl für einen freien Hinweistext, deshalb läuft sie über ntfy |
| Auslöser | Formular: Titel (optional), Nachricht (bis 500 Zeichen), Dringlichkeit (Normal, Hoch, Dringend) |
| Ablauf | Eingabe prüfen, per ntfy-JSON-Publish (UTF-8) auf das Topic `Home-Lab-iPad` senden, Ergebnis im Formular anzeigen. „Dringend“ umgeht den Nicht-stören-Modus |
| Empfang | Auf jedem iPad die ntfy-App mit dem Benutzer `ipad` (nur Lesen auf `Home-Lab-iPad`), Mitteilungen erlaubt. Einrichtung: [300v0](300v0-mdm-betrieb.md#nachricht-an-die-ipads-einrichten) |
| Credentials | `MDM Formulare`, `ntfy-publisher` |

Fehlerfälle: Zu lange Nachricht → Hinweis im Formular. ntfy lehnt ab (zum Beispiel Credential ohne Schreibrecht) → Meldung mit dem HTTP-Status. Eine Zustellgarantie gibt es nicht: ntfy meldet nicht zurück, welche iPads die Mitteilung erhalten haben. Ein ausgeschaltetes iPad bekommt sie, sobald es wieder online ist, sofern ntfy sie noch vorhält.

---

## 9. iPads sperren mit Nachricht

Datei `mdm-ipads-sperren.json` · Formular `https://n8n.prod.homeserver/form/mdm-sperren` (Basic Auth `MDM Formulare`).

| | |
|---|---|
| Zweck | Sperrt ein oder alle iPads sofort und zeigt eine Nachricht (und optional eine Telefonnummer) auf dem Sperrbildschirm. MDM-Befehl `DeviceLock` mit `Message` und `PhoneNumber` (Apple: auch auf nicht beaufsichtigten Geräten). Entsperrt wird mit dem Code des iPads |
| Auslöser | Formular: Nachricht (bis 300 Zeichen), Telefonnummer (optional), Seriennummer (leer = alle angemeldeten iPads), Bestätigung |
| Schutz | Das Formular sperrt nur bei der Auswahl „Ja, jetzt sperren“. Telefonnummer und Seriennummer werden auf erlaubte Zeichen geprüft. Jede Ausführung wird per ntfy auf `Home-Lab-MDM` protokolliert |
| Ablauf | Geräte aus der Datenbank lesen (`enrollments`, `devices`, nur Geräte-Enrollments, aktiv, optional nach Seriennummer), je Gerät `PUT /v1/enqueue/<Gerät>` mit `DeviceLock`, Ergebnis zählen |
| Ergebnis | Seite im Formular: „n von m iPad(s) erhalten den Sperrbefehl“. Scheitert der Push (Gerät offline, APNs-Fehler), liegt der Befehl in der Warteschlange und das iPad sperrt sich beim nächsten Check-in |
| Credentials | `MDM Formulare`, `MDM NanoMDM API`, `MDM Automation DB`, `ntfy-publisher` |

Fehlerfälle: falsche Bestätigung, ungültige Zeichen oder zu lange Nachricht → Hinweis im Formular, nichts gesperrt. Unbekannte Seriennummer oder noch kein angemeldetes iPad → „Nichts gesperrt“.

Unterschied zu den anderen Wegen: Der Lock-Screen-Text (Workflow 2) ist ein dauerhafter, kleiner Hinweis und sperrt nicht. Die Mitteilung (Workflow 8) ist eine einmalige Benachrichtigung über ntfy. Dieser Workflow sperrt das iPad wirklich.

---

## 10. App auf iPads installieren

Datei `mdm-app-installieren.json` · Formular `https://n8n.prod.homeserver/form/mdm-app-installieren` (Basic Auth `MDM Formulare`).

| | |
|---|---|
| Zweck | Installiert freigegebene Apps ohne Apple-ID auf dem iPad und hält die Liste der erlaubten Apps aktuell. Auf beaufsichtigten iPads sind nur diese Apps sichtbar |
| Katalog | Tabelle `automation.app_catalog` (App-Store-ID, Name, Bundle-ID, `config`). Start: ntfy `1625396347`, DLRG Trainer `1659881569`, Microsoft Teams `1113153706`, Bitwarden `1137397744` |
| App-Konfiguration | Die Spalte `config` (JSON) wird als Managed App Configuration im `InstallApplication`-Befehl (`Configuration`) mitgesendet. Bitwarden bekommt `baseEnvironmentUrl = https://vault-tech.pke-lab.de` (Bitwarden-Doku, Abschnitt Mobile-Apps; der Wert braucht das Präfix `https://`). Ändern: `UPDATE automation.app_catalog SET config = … WHERE adam_id = …`, danach im Formular die App erneut installieren. Prüfen: MDM-Befehl `ManagedApplicationConfiguration` mit der Bundle-ID. Passwörter oder Benutzernamen lassen sich so nicht eintragen, nur das, was die App als Konfigurationsschlüssel kennt |
| Auslöser | Formular: App aus der Liste, „Alle Apps der Liste“ oder „Andere App“ (App-Store-ID oder Link), optional Seriennummer |
| Neue App | Der Workflow fragt Name, Bundle-ID und Preis bei der iTunes-Lookup-API ab (öffentlich, ohne Anmeldung), lehnt bezahlte und iPad-untaugliche Apps ab und nimmt die App in den Katalog auf. Eine neue App muss vorher in Apple Business unter „Apps und Bücher“ abgerufen sein |
| Ablauf | Lizenz gerätegebunden zuweisen (`POST https://vpp.itunes.apple.com/mdm/v2/assets/associate`), je Gerät und App `InstallApplication` (`PurchaseMethod 1`) einreihen, danach die Erlaubt-Liste aus Systemapps und Katalog neu berechnen. Hat sie sich geändert, geht das Profil `app-allowlist` (`com.apple.applicationaccess`, `allowListedAppBundleIDs`) an alle iPads |
| Systemapps | Immer erlaubt: App Store, Einstellungen, Safari, Fotos (ansehen), Dateien, Uhr. Die Kamera-App ist nicht in der Liste. Die Liste steht als `SYSTEM_APPS` in `automation.profile_state` (Zeile `app-allowlist`) |
| Neue Geräte | Das Profil `app-allowlist` geht über Workflow 1 automatisch an jedes neu angemeldete iPad |
| Ergebnis | Seite im Formular und ntfy-Meldung auf `Home-Lab-MDM` |
| Credentials | `MDM Formulare`, `MDM NanoMDM API`, `MDM Automation DB`, `Apple Apps Token`, `ntfy-publisher` |

Wichtig:
- **Die Liste wirkt nur auf beaufsichtigten Geräten** (Apple: `allowListedAppBundleIDs` verlangt Aufsicht). Auf einem manuell angemeldeten iPad wird sie ignoriert. Apple hat den Schlüssel ab iOS 27 als veraltet markiert (Ersatz: deklaratives Management `com.apple.configuration.app.settings`); er funktioniert weiter, bis Apple ihn entfernt.
- **Einstellungen muss in der Liste bleiben**, sonst kommt auf dem iPad niemand mehr ans WLAN. Wer die Systemapps ändert, ändert `SYSTEM_APPS` in der Datenbank und löst über das Formular eine Aktualisierung aus.
- Alle Apps auf einem frisch eingerichteten iPad installiert nach dem Enrollment „Alle Apps der Liste“ mit der Seriennummer des Geräts. Eine automatische Installation beim Enrollment ist noch nicht eingerichtet.
- Die Aufrufe der Lizenz-API sind nach Dritt-Quellen gebaut und noch nicht mit einem echten Token getestet. Bei Abweichungen im Antwortformat den Node `Lizenz zuweisen` anpassen.

Fehlerfälle: Token ungültig oder Lizenz fehlt → Meldung mit HTTP-Status, die Befehle werden trotzdem eingereiht und scheitern am Gerät. Unbekannte oder bezahlte App → Hinweis im Formular, nichts geändert. Kein angemeldetes iPad → „Nichts installiert“.

---

## 11. Hintergrundbild ändern

Datei `mdm-hintergrund-aendern.json` · Formular `https://n8n.prod.homeserver/form/mdm-hintergrund` (Basic Auth `MDM Formulare`).

| | |
|---|---|
| Zweck | Setzt das Hintergrundbild (Sperr- und Home-Bildschirm) auf allen iPads und verhindert, dass Nutzer es ändern |
| Startbild | `profiles/ipad-wallpaper.png` (2048 × 2732, DLRG Andernach). Bei leerer Datenbank wird es in `automation.profile_state` (Zeile `wallpaper`) übernommen |
| Neue Geräte | Das Bild und die Sperre (`wallpaper`, `wallpaper-lock`) gehen über Workflow 1 automatisch an jedes neu angemeldete iPad |
| Auslöser | Formular mit Dateiupload (PNG oder JPG, bis 4 MB, empfohlen 2048 × 2732) |
| Ablauf | Bild prüfen (Dateikennung), Base64 in `params.IMAGE_B64` speichern, Version erhöhen, für alle Geräte beanspruchen, `Settings`-Befehl mit `Item Wallpaper`, `Where 3` einreihen. Das Profil `wallpaper-lock` (`com.apple.applicationaccess`, `allowWallpaperModification = false`) wird mitgesendet |
| Reihenfolge | Der Bild-Befehl wird vor der Sperre eingereiht (sortiert nach `kind`), damit die Sperre ihn nicht verhindert |
| Credentials | `MDM Formulare`, `MDM NanoMDM API`, `MDM Automation DB`, `ntfy-publisher` |

Wichtig:
- **Nur beaufsichtigte iPads.** `Settings`/`Wallpaper` und `allowWallpaperModification` verlangen Aufsicht (Apple).
- Der `Settings`-Befehl wirkt **einmalig** („the user may still change the settings later“). Erst die Einschränkung macht das Bild unveränderlich. Auf der Home-Seite zählt iPadOS ab 17 beim ersten Setzen beide Bildschirme.
- Der Bild-Befehl liegt als `kind = 'wallpaper'` in `automation.profile_state` (Spalte `kind`: `profile` oder `wallpaper`). Der gemeinsame Befehlsbauer der Workflows 1, 2, 10 und 11 baut für `wallpaper` einen `Settings`-Befehl statt `InstallProfile`.

---

## 12. Auftrag per Mail

Datei `mdm-auftrag-per-mail.json` · IMAP-Ordner `Ausbildung-Ipads`.

| | |
|---|---|
| Zweck | Eine Mail ändert das Hintergrundbild, sendet eine Push-Mitteilung, sperrt iPads mit Nachricht und installiert Apps auf einem, mehreren oder allen iPads. Die Wege entsprechen den Workflows 8, 9, 10 und 11, nur dass die Eingabe aus einer Mail kommt und das Ziel wählbar ist |
| Auslöser | Ungelesene Mail im Ordner `Ausbildung-Ipads` von den Absendern aus `sender` (Node `Konfiguration`, zusätzlich im IMAP-Filter des Triggers). Der Betreff spielt keine Rolle |
| Eingabe | Genau ein JSON-Anhang, dazu optional ein Bild (PNG oder JPG, bis 4 MB). Felder: `ipads` (Pflicht), `push`, `sperrbildschirm` (Sperren mit Nachricht), `app`, optional `hintergrund: false` zum Ignorieren des Bildes. Mindestens eine Aktion nötig, unbekannte Felder lehnt der Workflow ab, damit sich Tippfehler nicht still verlieren |
| Ziel | `ipads` ist eine Liste aus `kleines-becken`, `grosses-becken`, `vorstands-tisch`, einer Seriennummer oder `"alle"`. Schreibweise (Groß-/Kleinschreibung, Umlaute, Leerzeichen statt Bindestrich) wird normalisiert. Namen und Seriennummern stehen im Node `Konfiguration` (`ipads`), solange dort `CHANGE-ME-…` steht, lehnt der Workflow das iPad ab |
| Hintergrund | Bild als Anhang. Bei `alle` wie Workflow 11: Bild in `profile_state` (Zeile `wallpaper`) speichern, Version erhöhen, für alle Geräte beanspruchen. Bei einzelnen iPads geht der `Settings`-Befehl direkt an diese Geräte, `profile_state` bleibt unverändert |
| Sperren | `sperrbildschirm` als Text oder `{nachricht, telefon}` (Nachricht 1 bis 300 Zeichen, Telefonnummer optional). Sendet wie Workflow 9 den MDM-Befehl `DeviceLock`: Das iPad sperrt sich sofort und zeigt die Nachricht auf dem Sperrbildschirm, entsperrt wird mit dem Code des iPads. Der Befehl geht als letzter ein, nach Bild und App. Der dauerhafte Footer-Text (Workflow 2) lässt sich per Mail nicht ändern |
| Push | `push` als Text oder `{nachricht, titel, dringlichkeit}` (`normal`, `hoch`, `dringend`). Läuft über ntfy-JSON-Publish wie Workflow 8 |
| App | `app` als Katalogname, App-Store-ID oder App-Store-Link. Ablauf wie Workflow 10 (Lizenz zuweisen, `InstallApplication`, Erlaubt-Liste neu berechnen); eine neue ID wird über die iTunes-Lookup-API geprüft und in den Katalog aufgenommen. Eine App je Mail |
| Ergebnis | ntfy-Meldung auf `Home-Lab-MDM` mit Zählung je Aktion. Fehler (Absender, JSON, Ziel, Bild) ebenfalls dort, mit hoher Priorität |
| Credentials | `Netcup Automation`, `MDM NanoMDM API`, `MDM Automation DB`, `Apple Apps Token`, `ntfy-publisher` |

Wichtig:
- **ntfy adressiert kein einzelnes iPad.** Ohne `ntfyTopic` im Eintrag des iPads geht die Mitteilung an `Home-Lab-iPad`, also an alle iPads, mit dem iPad-Namen im Titel. Für Mitteilungen nur an ein iPad bekommt es ein eigenes Topic: in dessen ntfy-App abonnieren, Leserecht für den Benutzer `ipad` und Schreibrecht für `ntfy-publisher` einrichten, dann `ntfyTopic` setzen.
- **Die Erlaubt-Liste gilt immer für alle iPads**, auch wenn die App nur auf einem installiert wird (Workflow 10 verhält sich genauso).
- Ein einzelnes iPad mit eigenem Bild oder Text wird beim nächsten Ändern für `alle` (Formular oder Mail) überschrieben.
- Kommen mehrere Mails gleichzeitig an, verarbeitet der Workflow nur die erste. Für die übrigen kommt eine ntfy-Fehlermeldung mit der Bitte, erneut zu senden.
- Wirkt wie die Formulare nur auf beaufsichtigten iPads.

Fehlerfälle (jeweils ohne Änderung): Absender nicht erlaubt, kein oder unlesbarer JSON-Anhang, unbekanntes Feld oder iPad, fehlende Seriennummer im Workflow, Bild kein PNG/JPG oder größer als 4 MB, Text zu lang, keine Aktion. Ein gewähltes iPad ohne Anmeldung wird übersprungen und gemeldet, ist keines angemeldet, passiert nichts. App unbekannt, bezahlt oder in Apple Business nicht abgerufen → Meldung, die übrigen Aktionen laufen weiter.
