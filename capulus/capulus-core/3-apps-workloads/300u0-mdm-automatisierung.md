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
| `app_catalog` | Katalog der installierbaren Apps. `standard = true`: Soll auf allen iPads. `enabled` leitet Workflow 14 aus dem Soll ab (Standard-App oder Zusatz-App eines iPads) und steuert die Erlaubt-Liste, nicht von Hand ändern |
| `ipad_app` | Soll-Liste je iPad: Seriennummer und App-Store-ID zusätzlicher Apps, die nur auf diesem iPad installiert sein sollen |
| `ipad_app_ist` | Ist-Zustand: je Enrollment die zuletzt gemeldeten Apps (Bundle-ID, Name, Version, Zeitpunkt) aus `InstalledApplicationList` |
| `setting` | `dep_name`, `dep_profile_json` (Vorlage des DEP-Profils), `abgleich_entfernen` (`ja` oder `nein`: ob der Abgleich zusätzliche Apps entfernt) |

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

Dieser Weg ist im Formular `mdm-auftrag` aufgegangen ([Workflow 12](#12-auftrag-per-mail-oder-formular), Feld Push-Nachricht mit Titel und Dringlichkeit). Den früheren Workflow `mdm-nachricht-an-ipads.json` und sein Formular `mdm-nachricht` gibt es nicht mehr.

| | |
|---|---|
| Zweck | Einmalige Mitteilung, die als Push-Benachrichtigung „einfliegt“. MDM kennt keinen Befehl für einen freien Hinweistext, deshalb läuft sie über ntfy |
| Ablauf | ntfy-JSON-Publish (UTF-8) auf das Topic `Home-Lab-iPad`. „Dringend“ umgeht den Nicht-stören-Modus |
| Empfang | Auf jedem iPad die ntfy-App mit dem Benutzer `ipad` (nur Lesen auf `Home-Lab-iPad`), Mitteilungen erlaubt. Einrichtung: [300v0](300v0-mdm-betrieb.md#nachricht-an-die-ipads-einrichten) |

Eine Zustellgarantie gibt es nicht: ntfy meldet nicht zurück, welche iPads die Mitteilung erhalten haben. Ein ausgeschaltetes iPad bekommt sie, sobald es wieder online ist, sofern ntfy sie noch vorhält.

---

## 9. iPads sperren mit Nachricht

Dieser Weg ist im Formular `mdm-auftrag` und im Mail-Auftrag aufgegangen ([Workflow 12](#12-auftrag-per-mail-oder-formular), Feld Sperrbildschirm-Nachricht). Den früheren Workflow `mdm-ipads-sperren.json` und sein Formular `mdm-sperren` gibt es nicht mehr.

| | |
|---|---|
| Zweck | Sperrt iPads sofort und zeigt eine Nachricht (und optional eine Telefonnummer) auf dem Sperrbildschirm. MDM-Befehl `DeviceLock` mit `Message` und `PhoneNumber` (Apple: auch auf nicht beaufsichtigten Geräten). Entsperrt wird mit dem Code des iPads |
| Schutz | Das Formular sperrt nur, wenn im Feld `Sperren bestaetigen` die Auswahl `Ja, jetzt sperren` steht, sonst meldet es einen Fehler und sperrt nichts. Der Mail-Auftrag braucht keine Bestätigung, dort zählt der erlaubte Absender |
| Ergebnis | Scheitert der Push (Gerät offline, APNs-Fehler), liegt der Befehl in der Warteschlange und das iPad sperrt sich beim nächsten Check-in |

**Festtext:** Jede Sperrbildschirm-Nachricht beginnt automatisch mit „Das iPad ist nicht verloren gegangen, es gibt nur eine dringende Mitteilung:“. Er steht als Konstante `lockPrefix` im Node `Befehle bauen` von Workflow 12; die Längenprüfung (300 Zeichen) gilt nur für den eigenen Text.

Unterschied zu den anderen Wegen: Der Lock-Screen-Text (Workflow 2) ist ein dauerhafter, kleiner Hinweis und sperrt nicht. Die Mitteilung (Workflow 8) ist eine einmalige Benachrichtigung über ntfy. Das Sperren sperrt das iPad wirklich.

---

---

## 10. App auf iPads installieren

Datei `mdm-app-installieren.json` · Formular `https://n8n.prod.homeserver/form/mdm-app-installieren` (Basic Auth `MDM Formulare`).

| | |
|---|---|
| Zweck | Pflegt die Soll-Liste der Apps (alle iPads oder einzelne) und lässt sie ohne Apple-ID installieren. Das eigentliche Installieren, die Lizenzen und die Erlaubt-Liste übernimmt [Workflow 14](#14-apps-abgleichen). Auf beaufsichtigten iPads sind nur Apps der Liste sichtbar |
| Katalog | Tabelle `automation.app_catalog` (App-Store-ID, Name, Bundle-ID, `config`). Start: ntfy `1625396347`, DLRG Trainer `1659881569`, Microsoft Teams `1113153706`, Bitwarden `1137397744`, Mattermost `1257222717`. `standard = true` heißt: Soll auf allen iPads (heute ntfy, DLRG Trainer, Bitwarden, Mattermost). Zusätzliche Apps einzelner iPads stehen in `automation.ipad_app`. Teams und Authenticator sind nur im Katalog |
| iPads | Die Gruppen des Dropdowns (`Kleines Becken`, …) und ihre Seriennummern liest der Node `iPads lesen` aus `automation.ipad_group` und `automation.ipad` ([Pflege](300v0-mdm-betrieb.md#ipad-liste-pflegen)). Eine Gruppe ohne iPad lehnt das Formular ab |
| App-Konfiguration | Die Spalte `config` (JSON) wird als Managed App Configuration im `InstallApplication`-Befehl (`Configuration`) mitgesendet. Bitwarden bekommt `baseEnvironmentUrl = https://vault-tech.pkr-lab.de` (Bitwarden-Doku, Abschnitt Mobile-Apps; der Wert braucht das Präfix `https://`). Ändern: `UPDATE automation.app_catalog SET config = … WHERE adam_id = …`, danach im Formular die App erneut installieren. Prüfen: MDM-Befehl `ManagedApplicationConfiguration` mit der Bundle-ID. Passwörter oder Benutzernamen lassen sich so nicht eintragen, nur das, was die App als Konfigurationsschlüssel kennt |
| Auslöser | Formular: App aus der Liste, „Alle Apps der Liste“ oder „Andere App“ (App-Store-ID oder Link), optional Seriennummer |
| Neue App | Der Workflow fragt Name, Bundle-ID und Preis bei der iTunes-Lookup-API ab (öffentlich, ohne Anmeldung), lehnt bezahlte und iPad-untaugliche Apps ab und nimmt die App in den Katalog auf. Eine neue App muss vorher in Apple Business unter „Apps und Bücher“ abgerufen sein |
| Ablauf | Installieren: App in die Soll-Liste eintragen (`Alle iPads` setzt `app_catalog.standard`, sonst eine Zeile je Seriennummer in `ipad_app`), dann Workflow 14 starten. Der Abgleich weist die Lizenz gerätegebunden zu, reiht `InstallApplication` ein und berechnet die Erlaubt-Liste neu. Deinstallieren: App aus der Soll-Liste nehmen, `RemoveApplication` für die gewählten iPads einreihen, danach den Abgleich starten. Eine Standard-App lässt sich nur über `Alle iPads` deinstallieren, für einzelne iPads lehnt das Formular ab. Web-Links gehen als Profil `webclip-*` direkt an die iPads |
| Systemapps | Immer erlaubt: App Store, Einstellungen, Safari, Fotos (ansehen), Dateien, Uhr. Die Kamera-App ist nicht in der Liste. Die Liste steht als `SYSTEM_APPS` in `automation.profile_state` (Zeile `app-allowlist`) |
| Neue Geräte | Das Profil `app-allowlist` geht über Workflow 1 automatisch an jedes neu angemeldete iPad. Die Standard-Apps installiert der Abgleich (Workflow 14), den die Grundeinrichtung startet |
| Ergebnis | Seite im Formular und ntfy-Meldung auf `Home-Lab-MDM` |
| Credentials | `MDM Formulare`, `MDM NanoMDM API`, `MDM Automation DB`, `Apple Apps Token`, `ntfy-publisher` |

Wichtig:
- **Die Liste wirkt nur auf beaufsichtigten Geräten** (Apple: `allowListedAppBundleIDs` verlangt Aufsicht). Auf einem manuell angemeldeten iPad wird sie ignoriert. Apple hat den Schlüssel ab iOS 27 als veraltet markiert (Ersatz: deklaratives Management `com.apple.configuration.app.settings`); er funktioniert weiter, bis Apple ihn entfernt.
- **Einstellungen muss in der Liste bleiben**, sonst kommt auf dem iPad niemand mehr ans WLAN. Wer die Systemapps ändert, ändert `SYSTEM_APPS` in der Datenbank und löst über das Formular eine Aktualisierung aus.
- „Alle Apps der Liste“ trägt alle freigegebenen Katalog-Apps für die gewählten iPads in die Soll-Liste ein. Ein frisch eingerichtetes iPad bekommt die Standard-Apps vom Abgleich, ein manuelles Nachziehen ist nicht mehr nötig.
- Die Aufrufe der Lizenz-API sind nach Dritt-Quellen gebaut und noch nicht mit einem echten Token getestet. Bei Abweichungen im Antwortformat den Node `Lizenz zuweisen` in Workflow 14 anpassen.

Fehlerfälle: Token ungültig oder Lizenz fehlt → Meldung mit HTTP-Status, die Befehle werden trotzdem eingereiht und scheitern am Gerät. Unbekannte oder bezahlte App → Hinweis im Formular, nichts geändert. Kein angemeldetes iPad → „Nichts installiert“.

---

## 11. Hintergrundbild ändern

Dieser Weg ist im Formular `mdm-auftrag` und im Mail-Auftrag aufgegangen ([Workflow 12](#12-auftrag-per-mail-oder-formular), Feld Hintergrundbild). Den früheren Workflow `mdm-hintergrund-aendern.json` und sein Formular `mdm-hintergrund` gibt es nicht mehr.

| | |
|---|---|
| Zweck | Setzt das Hintergrundbild (Sperr- und Home-Bildschirm) und verhindert, dass Nutzer es ändern |
| Startbild | `profiles/ipad-wallpaper.png` (2048 × 2732, DLRG Andernach). Bei leerer Datenbank wird es in `automation.profile_state` (Zeile `wallpaper`) übernommen |
| Neue Geräte | Das Bild und die Sperre (`wallpaper`, `wallpaper-lock`) gehen über Workflow 1 automatisch an jedes neu angemeldete iPad |
| Eingabe | PNG oder JPG, bis 4 MB, empfohlen 2048 × 2732. Bei „Alle iPads“ wird das Bild in `profile_state` gespeichert, die Version erhöht und für alle Geräte beansprucht, bei einzelnen iPads geht der `Settings`-Befehl direkt an diese |
| Reihenfolge | Der Bild-Befehl wird vor der Sperre eingereiht (sortiert nach `kind`), damit die Sperre ihn nicht verhindert. Das Profil `wallpaper-lock` (`allowWallpaperModification = false`) wird mitgesendet |

Wichtig:
- **Nur beaufsichtigte iPads.** `Settings`/`Wallpaper` und `allowWallpaperModification` verlangen Aufsicht (Apple).
- Der `Settings`-Befehl wirkt **einmalig** („the user may still change the settings later“). Erst die Einschränkung macht das Bild unveränderlich. Auf der Home-Seite zählt iPadOS ab 17 beim ersten Setzen beide Bildschirme.
- Der Bild-Befehl liegt als `kind = 'wallpaper'` in `automation.profile_state` (Spalte `kind`: `profile` oder `wallpaper`). Der gemeinsame Befehlsbauer der Workflows 1, 2 und 12 baut für `wallpaper` einen `Settings`-Befehl statt `InstallProfile`.

---

## 12. Auftrag per Mail oder Formular

Datei `mdm-auftrag-per-mail.json` · IMAP-Ordner `Ausbildung-Ipads` und Formular `https://n8n.prod.homeserver/form/mdm-auftrag`.

| | |
|---|---|
| Zweck | Eine Mail oder das Formular ändert das Hintergrundbild, sendet eine Push-Mitteilung, sperrt iPads mit Nachricht und installiert Apps auf einem, mehreren oder allen iPads. Er ersetzt die früheren Einzelformulare für Mitteilung, Sperren, Hintergrundbild und Textdatei (Abschnitte 8, 9 und 11 beschreiben nur noch die Funktionen); das Ziel ist wählbar, die Eingabe kommt aus einer Mail oder dem Formular. Die Installation einer App läuft wie in Workflow 10 über die Soll-Liste |
| Auslöser | Ungelesene Mail im Ordner `Ausbildung-Ipads` von den Absendern aus `sender` (Node `Konfiguration`, zusätzlich im IMAP-Filter des Triggers). Der Betreff spielt keine Rolle |
| Auslöser Formular | Formular `mdm-auftrag` hinter Basic Auth (Credential `MDM Formulare`, wie die anderen Formulare). Felder: iPads (Checkboxen `Alle iPads` und die Gruppen `Kleines Becken`, `Grosses Becken`, `Vorstands-Tisch`), Einzelne iPads (Checkboxen mit den Namen aus `automation.ipad`), Seriennummern (kommagetrennt), Hintergrundbild (Upload), Push-Nachricht, -Titel und -Dringlichkeit, Sperrbildschirm-Nachricht und -Telefon, Sperren bestaetigen (Ja, jetzt sperren), App installieren (Dropdown mit den Katalog-Apps) und Andere App (App-Store-ID oder Link, nicht zusammen mit dem Dropdown). Alles außer der Auswahl ist optional, mindestens eine Aktion nötig. Das Formular erzeugt denselben Plan wie die Mail, ab dem Node `Auftrag lesen` laufen beide Wege gemeinsam. Das Ergebnis erscheint zusätzlich als Antwortseite |
| Eingabe Mail | Genau ein JSON-Anhang, dazu optional ein Bild (PNG oder JPG, bis 4 MB). Felder: `ipads` (Pflicht), `push`, `sperrbildschirm` (Sperren mit Nachricht), `app`, optional `hintergrund: false` zum Ignorieren des Bildes. Mindestens eine Aktion nötig, unbekannte Felder lehnt der Workflow ab, damit sich Tippfehler nicht still verlieren |
| Ziel | `ipads` ist eine Liste aus Gruppenschlüsseln oder -namen (`kleines-becken`, `grosses-becken`, `vorstands-tisch`), iPad-Namen (`automation.ipad.name`), einer Seriennummer oder `"alle"`. Schreibweise (Groß-/Kleinschreibung, Umlaute, Leerzeichen statt Bindestrich) wird normalisiert. Gruppen und Seriennummern stehen in der Datenbank (`automation.ipad_group`, `automation.ipad`, Pflege: [300v0](300v0-mdm-betrieb.md#ipad-liste-pflegen)); der Node `iPads lesen` holt sie bei jedem Lauf. Eine Gruppe ohne iPad lehnt der Workflow ab |
| Hintergrund | Bild als Anhang. Bei `alle` wie Workflow 11: Bild in `profile_state` (Zeile `wallpaper`) speichern, Version erhöhen, für alle Geräte beanspruchen. Bei einzelnen iPads geht der `Settings`-Befehl direkt an diese Geräte, `profile_state` bleibt unverändert |
| Sperren | `sperrbildschirm` als Text oder `{nachricht, telefon}` (Nachricht 1 bis 300 Zeichen, Telefonnummer optional). Sendet wie Workflow 9 den MDM-Befehl `DeviceLock`: Das iPad sperrt sich sofort und zeigt die Nachricht auf dem Sperrbildschirm, entsperrt wird mit dem Code des iPads. Der Befehl geht als letzter ein, nach Bild und App. Der dauerhafte Footer-Text (Workflow 2) lässt sich per Mail nicht ändern |
| Push | `push` als Text oder `{nachricht, titel, dringlichkeit}` (`normal`, `hoch`, `dringend`). Läuft über ntfy-JSON-Publish wie Workflow 8 |
| App | `app` als Katalogname, App-Store-ID oder App-Store-Link. Der Name darf Groß-/Kleinschreibung ignorieren und ein eindeutiger Teil ab vier Zeichen genügt (`Authenticator` findet `Microsoft Authenticator`, `Microsoft` ist mehrdeutig und wird abgelehnt). Die App kommt in die Soll-Liste (bei `alle` als Standard-App, sonst nur für die gewählten iPads) und Workflow 14 installiert sie mit Lizenz und aktualisiert die Erlaubt-Liste; eine neue ID wird über die iTunes-Lookup-API geprüft und in den Katalog aufgenommen. Eine App je Mail. Das Ergebnis der Installation kommt als eigene ntfy-Meldung von Workflow 14 |
| Ergebnis | ntfy-Meldung auf `Home-Lab-MDM` mit Zählung je Aktion. Fehler (Absender, JSON, Ziel, Bild) ebenfalls dort, mit hoher Priorität |
| Credentials | `Netcup Automation`, `MDM Formulare`, `MDM NanoMDM API`, `MDM Automation DB`, `ntfy-publisher` |

Wichtig:
- **ntfy adressiert kein einzelnes iPad.** Ohne `ntfy_topic` an der Gruppe geht die Mitteilung an `Home-Lab-iPad`, also an alle iPads, mit dem Gruppennamen im Titel. Für Mitteilungen nur an eine Gruppe bekommt sie ein eigenes Topic: in der ntfy-App dieser iPads abonnieren, Leserecht für den Benutzer `ipad` und Schreibrecht für `ntfy-publisher` einrichten, dann `UPDATE automation.ipad_group SET ntfy_topic = '…' WHERE key = '…'`.
- **Neue Gruppe oder neuer iPad-Name:** Neben dem Datenbankeintrag braucht das Formular eine Checkbox-Option mit demselben Namen (Node `Formular Auftrag`, Workflow 10 zusätzlich im Dropdown `iPads`). Per Mail genügt der Datenbankeintrag.
- **Die Erlaubt-Liste gilt immer für alle iPads**, auch wenn die App nur auf einem installiert wird (Workflow 10 verhält sich genauso).
- Ein einzelnes iPad mit eigenem Bild oder Text wird beim nächsten Ändern für `alle` (Formular oder Mail) überschrieben.
- Kommen mehrere Mails gleichzeitig an, verarbeitet der Workflow nur die erste. Für die übrigen kommt eine ntfy-Fehlermeldung mit der Bitte, erneut zu senden.
- Wirkt wie die Formulare nur auf beaufsichtigten iPads.

Fehlerfälle (jeweils ohne Änderung): Absender nicht erlaubt, kein oder unlesbarer JSON-Anhang, unbekanntes Feld oder iPad, Gruppe ohne iPad in der Datenbank, iPad-Liste nicht lesbar, Bild kein PNG/JPG oder größer als 4 MB, Text zu lang, keine Aktion. Ein gewähltes iPad ohne Anmeldung wird übersprungen und gemeldet, ist keines angemeldet, passiert nichts. App unbekannt, bezahlt oder in Apple Business nicht abgerufen → Meldung, die übrigen Aktionen laufen weiter.

---

## 13. iPads herunterfahren

Datei `mdm-ipads-herunterfahren.json` · Zeitplan jeden Donnerstag 19:30 Uhr (`30 19 * * 4`, Zeitzone Europe/Berlin).

| | |
|---|---|
| Zweck | Fährt alle angemeldeten iPads automatisch herunter |
| Auslöser | Zeitplan `Donnerstag 19:30`. Kein Formular, kein manueller Weg |
| Ablauf | Alle aktiven Geräte-Enrollments aus der Datenbank lesen (wie „alle iPads“), je Gerät den MDM-Befehl `ShutDownDevice` über `PUT /v1/enqueue/<id>` bei NanoMDM einreihen, Ergebnis auswerten |
| Ergebnis | ntfy auf `Home-Lab-MDM`: „n von m iPad(s) erhalten den Befehl“. Scheitert der Push oder das Einreihen, Meldung mit hoher Priorität. Ist kein iPad angemeldet, kommt ebenfalls eine Meldung |
| Credentials | `MDM Automation DB`, `MDM NanoMDM API`, `ntfy-publisher` |

Wichtig:
- **Internet nötig:** Der Befehl erreicht das iPad nur über APNs und eine Verbindung zum MDM-Server. Das iPad braucht zum Zeitpunkt des Befehls WLAN oder Mobilfunk.
- **Nur beaufsichtigte iPads** führen `ShutDownDevice` aus.
- **Kein Einschalten per MDM:** Ein heruntergefahrenes iPad muss am Gerät mit der Einschalttaste gestartet werden.
- **Offline oder bereits aus:** Der Befehl bleibt in der NanoMDM-Warteschlange und läuft, sobald das iPad wieder online ist. Ein iPad, das am Donnerstag aus war, kann sich deshalb nach dem nächsten Einschalten selbst wieder ausschalten.
- Zeit ändern: Ausdruck im Node `Donnerstag 19:30` anpassen. Abschalten: Workflow in n8n deaktivieren.

---

## 14. Apps abgleichen

Datei `mdm-apps-abgleichen.json` · Zeitplan täglich 18:00 Uhr (Europe/Berlin) · Formular `https://n8n.prod.homeserver/form/mdm-apps-abgleichen` (Basic Auth `MDM Formulare`) · Aufruf durch die Workflows 10, 12 und die Grundeinrichtung.

| | |
|---|---|
| Zweck | Die einzige Stelle, die Apps auf iPads bringt. Sie vergleicht den Ist-Zustand jedes iPads mit der Soll-Liste, installiert Fehlendes und hält die Erlaubt-Liste aktuell |
| Soll | Standard-Apps (`app_catalog.standard`) plus Apps einzelner iPads (`ipad_app`). Die Erlaubt-Liste ist global: Systemapps plus alle Apps, die irgendwo Soll sind |
| Ist | Der Workflow fordert bei allen angemeldeten iPads `InstalledApplicationList` an, wartet 4 Minuten, liest die Antworten aus `public.command_results` und speichert sie in `ipad_app_ist` (Bundle-ID, Name, Version) |
| Vergleich | Je iPad: fehlt eine Soll-App, wird sie installiert. Eine App, die nicht `com.apple.*` ist und nicht im Soll dieses iPads steht, gilt als zusätzlich und wird gemeldet |
| Installieren | Lizenz je App gerätegebunden zuweisen (`POST /mdm/v2/assets/associate` mit den Seriennummern der iPads, denen sie fehlt), danach `InstallApplication` je iPad und App. Eine App, die in Apple Business nicht abgerufen ist, wird gemeldet und übersprungen |
| Entfernen | Zusätzliche Apps werden nur entfernt (`RemoveApplication`), wenn das Formular mit Modus `Installieren und Entfernen` läuft oder `automation.setting` `abgleich_entfernen` auf `ja` steht. Standard ist `nein`: nur melden |
| Erlaubt-Liste | `app_catalog.enabled` wird aus dem Soll abgeleitet, danach das Profil `app-allowlist` neu berechnet und bei Änderung an alle iPads gesendet |
| Nicht geprüft | Ein gesperrtes iPad (`NotNow`) oder ein iPad ohne Antwort zählt als nicht geprüft, nie als „App fehlt“. Der nächste Lauf holt es nach |
| Auslöser | Zeitplan täglich 18:00 (meldet nur bei Abweichungen oder Problemen), Formular mit Modus `Nur melden`, `Installieren` oder `Installieren und Entfernen` (meldet immer), Aufruf durch andere Workflows |
| Ergebnis | ntfy auf `Home-Lab-MDM` mit den Abweichungen je iPad, den eingereihten Installationen und Problemen |
| Credentials | `MDM Formulare`, `MDM NanoMDM API`, `MDM Automation DB`, `Apple Apps Token`, `ntfy-publisher` |

Wichtig:
- **Neue Tabellen in der laufenden Datenbank:** `automation-schema.sql` läuft nur beim ersten Start der Datenbank. Für den bestehenden Cluster legt `scripts/mdm-migrate-abgleich.sh` die Spalte `standard` und die Tabellen `ipad_app` und `ipad_app_ist` an, mehrfaches Ausführen ist unschädlich.
- **Erster Lauf immer mit `Nur melden`**, damit zu sehen ist, welche iPads welche Apps bekommen würden. Das gilt besonders nach dem Hinzufügen einer Standard-App, weil sie auf alle iPads kommt.
- **Katalog und iPad müssen übereinstimmen:** Meldet ein iPad eine andere Bundle-ID als der Katalog (so war Mattermost zuerst als `com.mattermost.rnbeta` eingetragen, installiert war `com.mattermost.rn`), taucht die App im Abgleich als fehlend und zugleich als zusätzlich auf. Dann `bundle_id` im Katalog korrigieren.
- Der Workflow wird von den anderen Workflows über die feste Workflow-ID aufgerufen (Node `Abgleich starten`). Bei einem Neuimport in n8n die ID dort anpassen.
- Wirkt wie die übrigen App-Workflows nur auf beaufsichtigten iPads.
