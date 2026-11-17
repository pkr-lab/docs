# ISC-Manager-Bot — Person aus SEWOBE ins ISC übertragen (RPA)

Eigenständiger Dienst, der **eine** Person aus SEWOBE (`https://mv.dlrg.net`, Mitgliederverwaltung) in das DLRG-ISC (`https://dlrg.net`, App „Personen“) überträgt. Er steuert einen headless Chromium über Playwright und baut auf dem [ISC-News-Bot](300k0-isc-news-bot.md) auf (gleiche Struktur, gleiches Secret-Muster, gleicher ISC-Login). n8n ist Auftraggeber: Eine Mail oder ein Test-Formular löst den Workflow aus, n8n ruft den Dienst per HTTP auf.

> **Cluster:** TECH · Ordner `argocd/apps/tech/isc-manager-bot/` · kein Ingress, nur cluster-intern unter `http://isc-manager-bot.isc-manager-bot.svc.cluster.local:8080`. `kubectl`-Befehle in diesem Doc gelten für den TECH-Cluster ([Zugriff je Cluster](../../capulus-ansible/a-betriebssystem/a0010-overview.md#kubectl-zugriff-je-cluster)).

---

## Architektur

```
n8n (Namespace n8n)                                Namespace isc-manager-bot
 ├─ IMAP-Trigger (Ordner NewUebertrag)             ┌──────────────────────────────┐
 ├─ Test-Formular                                  │ isc-manager-bot              │
 └─ Workflow „ISC-Manager: Person aus SEWOBE…“     │ Node.js + Playwright         │
      │  POST /sync  (Bearer-Token)  ────────────▶ │ headless Chromium (uid 1000) │
      │  NetworkPolicy: Ingress nur aus n8n        └───────┬───────────────┬──────┘
      ▼                                                    │ HTTPS 443     │ HTTPS 443
   ntfy-Meldung (Ergebnis / Fehler)                        ▼               ▼
                                                  mv.dlrg.net (SEWOBE)  dlrg.net (ISC)
```

Der Dienst hat keinen Zugang von außen. Die NetworkPolicy erlaubt Eingang nur aus dem Namespace `n8n`, Ausgang nur zu DNS und TCP 443 außerhalb der privaten Netze.

### Ablauf eines Laufs

Jeder Lauf startet mit zwei frischen Browser-Kontexten (ISC und SEWOBE, keine gemeinsamen Cookies), nichts wird zwischengespeichert. Pro Pod läuft immer nur ein Lauf (Mutex), weitere Anfragen warten bis `LOCK_WAIT_MS`, danach `BUSY` (HTTP 429).

| Schritt | Was passiert | Bei Fehler |
|---|---|---|
| 1. ISC-Login | Anmelden wie im News-Bot, „Angemeldet bleiben“ aus | `LOGIN_FAILED`, `LOGIN_UNEXPECTED` |
| 2. Gliederung | Aktive Gliederung prüfen, bei Bedarf auf EDV `1002011` umschalten | `WRONG_GLIEDERUNG` |
| 3. SEWOBE-Login | Benutzer und Passwort, dann zweiter Faktor | `LOGIN_FAILED`, `TOTP_FAILED` |
| 4. SEWOBE-Suche | Suche nach „Vorname Nachname“, exakter Namensvergleich auf „Nachname, Vorname“ | `SEWOBE_NO_MATCH` (kein exakter Treffer), `SEWOBE_AMBIGUOUS` (mehrere aktive Treffer) |
| 5. SEWOBE-Daten | Person öffnen, Felder lesen, Datum von `TT.MM.JJJJ` nach ISO umrechnen | `SEWOBE_DATA_INVALID` (Pflichtfeld fehlt oder ungültig) |
| 6. ISC-Abgleich | Personenliste lesen (im Browser gefiltert), nur nach DLRG-Manager-Id, danach Vorname + Name + Geburtsdatum | `ISC_CONFLICT`, `ISC_AMBIGUOUS` |
| 7. Plan | Vergleich Soll gegen Ist, nur geänderte Felder, leere SEWOBE-Felder überschreiben nichts | `GESCHLECHT_FEHLT`, `LAND_UNBEKANNT` |
| 8. Speichern | Grunddaten, Adresse, Mitgliedschaft, danach Gegenprüfung im ISC. Bei `dryRun` entfällt der Schritt | `SAVE_FAILED`, `SAVE_UNCONFIRMED`, `FORM_INVALID` |
| 9. Abmelden | In beiden Systemen, Kontexte und Browser schließen, auch bei Fehlern | — |

**Zweiter Faktor.** Der Bot erzeugt den 6-stelligen Code selbst aus dem Base32-Secret `SEWOBE_TOTP_SECRET` (Bibliothek `otpauth`, SHA-1, 30 s). Bleiben weniger als 5 Sekunden im Zeitfenster, wartet er auf das nächste. Lehnt SEWOBE den Code ab, versucht er es genau einmal mit dem Code des nächsten Fensters, danach `TOTP_FAILED`. Derselbe Code wird nie zweimal gesendet.

**Abgleich im ISC.**

| Lage | Ergebnis |
|---|---|
| Genau ein Treffer über die Id | aktualisieren, nur geänderte Felder |
| Mehrere Treffer über dieselbe Id | `ISC_AMBIGUOUS`, nichts geändert |
| Kein Id-Treffer, genau ein Treffer über Vorname + Name + Geburtsdatum ohne Id | aktualisieren und Id setzen |
| Kein Id-Treffer, ein Treffer über Namen, aber mit **anderer** Id | `ISC_CONFLICT`, nichts geändert |
| Kein Id-Treffer, mehrere Namenstreffer | `ISC_AMBIGUOUS`, nichts geändert |
| Gar kein Treffer | neu anlegen |

Die Id ist die DLRG-Manager-Id, in SEWOBE die 13-stellige Mitgliedsnummer. Namen werden getrimmt, Leerraum zusammengefasst und ohne Groß-/Kleinschreibung verglichen.

**Idempotenz.** Beim Anlegen wird die Id gleich mitgesetzt. Dieselbe Mail ein zweites Mal findet die Person über die Id und endet mit `unchanged`, ohne Schreibzugriff. Gleichzeitige Anfragen laufen nacheinander.

---

## API-Referenz

Alle Endpunkte außer `/healthz` und `/readyz` verlangen `Authorization: Bearer <API_TOKEN>`.

| Methode und Pfad | Zweck |
|---|---|
| `POST /sync` | Eine Person abgleichen (Body siehe unten) |
| `GET /healthz` | Lebenszeichen, ohne Browser, ohne Token |
| `GET /readyz` | Bereit, wenn der Chromium-Build vorhanden ist, ohne Token |

### Request `POST /sync`

`Content-Type: application/json`, Body höchstens `MAX_BODY_BYTES` (Standard 64 KiB).

| Feld | Pflicht | Bedeutung |
|---|---|---|
| `vorname` | ja | Höchstens 100 Zeichen |
| `nachname` | ja | Höchstens 100 Zeichen |
| `mitgliedsnummer` | nein | 13 Ziffern. Filtert die SEWOBE-Treffer, damit gleichnamige Personen unterscheidbar sind |
| `dryRun` | nein | `true`: alles außer Speichern |

### Antwort

| Feld | Inhalt |
|---|---|
| `ok` | `true` bei `created`, `updated`, `unchanged`, `skipped` |
| `result` | `created`, `updated`, `unchanged`, `skipped` oder `error` |
| `dryRun` | Wurde nur geprüft |
| `changedFields` | Bei `created` und `updated`: Namen der Felder, nie Werte |
| `diff` | **Nur bei `dryRun`:** Liste aus Feld, alter und neuer Wert |
| `warnings` | `GESCHLECHT_NICHT_GESETZT`, wenn die Anrede in SEWOBE kein Geschlecht liefert |
| `reason` | Bei `skipped` und `error`: Grund im Klartext |
| `errorCode`, `step`, `error`, `details` | Bei `error`: Code, Schritt, Meldung, Feldnamen oder Zähler |
| `durationMs` | Laufzeit |

`changedFields` enthält je nach Fall `vorname`, `nachname`, `geburtsdatum`, `geschlecht`, `dlrgManagerId`, `strasse`, `plz`, `ort`, `land`, `mitgliedschaft`.

### Fehlercodes und HTTP-Status

| Status | Codes |
|---|---|
| 200 | `created`, `updated`, `unchanged`, `skipped` |
| 400 | `VALIDATION_FAILED` (Anfrage ungültig, Details in `details`) |
| 422 | `SEWOBE_NO_MATCH`, `SEWOBE_AMBIGUOUS`, `SEWOBE_DATA_INVALID`, `ISC_CONFLICT`, `ISC_AMBIGUOUS`, `GESCHLECHT_FEHLT`, `LAND_UNBEKANNT`, `FORM_INVALID`: fachlich, im ISC wurde nichts verändert (außer `FORM_INVALID` nach einem Teilschritt, dann nur die bereits bestätigten) |
| 429 | `BUSY` |
| 502 | `LOGIN_FAILED`, `LOGIN_UNEXPECTED`, `TOTP_FAILED`, `WRONG_GLIEDERUNG`, `FORM_CHANGED` (Selektor fehlt, Seite hat sich geändert), `SAVE_FAILED`, `SAVE_UNCONFIRMED` |
| 504 | `TIMEOUT` |
| 500 | `INTERNAL` |

---

## Feld-Mapping

| SEWOBE | ISC (Person) | Regel |
|---|---|---|
| Vorname | Vorname | |
| Name | Name | |
| Geburtstag (`TT.MM.JJJJ`) | Geburtsdatum (ISO) | Ungültige Daten (zum Beispiel 31.02.) → `SEWOBE_DATA_INVALID`, es wird nichts geraten |
| Anrede „Herr“ | Geschlecht männlich | |
| Anrede „Frau“ | Geschlecht weiblich | |
| Anrede „Divers“ | Geschlecht divers | Nur gesetzt, weil es in SEWOBE gesetzt ist |
| Anrede „Keine Anrede“ oder leer | — | Bestehende Person: Geschlecht bleibt, Warnung `GESCHLECHT_NICHT_GESETZT`. Neue Person: `GESCHLECHT_FEHLT`, es wird nicht angelegt |
| Mitgliedsnummer (13 Ziffern) | DLRG-Manager-Id | Abgleichsschlüssel |
| Straße | Straße & Nr. | Mehrzeilig wird auf eine Zeile gebracht |
| PLZ, Ort | PLZ, Ort | |
| Land | Land | Auswahl über den Anzeigenamen („Deutschland“), unbekannt → `LAND_UNBEKANNT`, auch im Dry-Run |
| — | DLRG-Mitgliedschaft, Gliederung „Ortsgruppe Andernach e.V.“ | Immer, aber nur wenn noch keine existiert |

Regeln für alle Felder: Leere SEWOBE-Felder überschreiben nie einen ISC-Wert. Gespeichert wird nur, was sich unterscheidet. Die Adresse ist die neueste „Private Anschrift“ der Person, sie wird an Ort und Stelle geändert, nicht dupliziert. Gibt es keine, wird eine angelegt, aber nur wenn SEWOBE Adressdaten liefert.

**Nicht aktive Personen.** Nur Personen mit dem Status „aktiv“ werden übertragen. Alle anderen Status (zum Beispiel ausgetreten, passiv, gekündigt) ergeben `skipped` mit Begründung. Die Standardsuche in SEWOBE blendet ausgeschiedene Personen teils aus, dann endet der Lauf mit `SEWOBE_NO_MATCH`.

---

## Dry-Run

Mit `dryRun: true` läuft alles außer Speichern, Anlegen und Mitgliedschaft: beide Logins, Suche, Abgleich, Plan. Die Antwort enthält `diff` mit Feld, altem und neuem Wert. Der Diff steht **nur** in der Antwort, nie in den Logs. Der Dry-Run erkennt auch `LAND_UNBEKANNT` und `GESCHLECHT_FEHLT`.

---

## Datenschutz

| Maßnahme | Umsetzung |
|---|---|
| Keine Personendaten in Logs | Der Logger verwirft Felder mit Namen wie `name`, `geburt`, `strasse`, `plz`, `land`, `uid`, `diff`, `user`, `pass`, `token`, `totp` und ersetzt registrierte Geheimnisse durch `***` |
| Keine Personendaten in Fehlermeldungen | Fehlertexte nennen Schritt und Code, nie Namen, Nummern oder URLs mit Ids. Playwright-Meldungen enthalten Locator-Texte und werden deshalb durch eine feste Meldung ersetzt |
| Keine Screenshots in Antworten | Anders als beim News-Bot wird kein Screenshot mitgeschickt |
| Datensparsame Personenliste | Der ISC liefert alle rund 5000 Personen. Der Bot filtert im Browser auf die Treffer für Id oder Name + Geburtsdatum, nur diese verlassen die Seite |
| Keine Sitzungsspeicherung | Frischer Kontext pro Lauf, Abmeldung in beiden Systemen im `finally` |
| Geheimnisse | Nur Umgebungsvariablen aus dem SealedSecret, TOTP-Secret nie im Klartext im Repo |

---

## Konfiguration

ConfigMap `isc-manager-bot-config` (Werte in `values.yaml` unter `config`):

| Variable | Standard | Bedeutung |
|---|---|---|
| `ISC_BASE_URL` | `https://dlrg.net` | Muss mit `https://` beginnen |
| `SEWOBE_BASE_URL` | `https://mv.dlrg.net` | Muss mit `https://` beginnen |
| `GLIEDERUNG_EDV` | `1002011` | Ziel-Gliederung in beiden Systemen |
| `MITGLIEDSCHAFT_GLIEDERUNG` | `Ortsgruppe Andernach e.V.` | Gliederung der angelegten Mitgliedschaft |
| `STEP_TIMEOUT_MS` | 30000 | Pro Schritt |
| `SEARCH_TIMEOUT_MS` | 90000 | SEWOBE-Suche, sie braucht rund 20 s |
| `SAVE_TIMEOUT_MS` | 45000 | Bestätigung beim Speichern |
| `SYNC_TIMEOUT_MS` | 360000 | Gesamtlauf |
| `LOCK_WAIT_MS` | 60000 | Wartezeit auf den Mutex |
| `MAX_BODY_BYTES` | 65536 | Größe des Request-Body |

Secret `isc-manager-bot-secrets` (nur Umgebung, nie Datei): `ISC_USERNAME`, `ISC_PASSWORD`, `SEWOBE_USERNAME`, `SEWOBE_PASSWORD`, `SEWOBE_TOTP_SECRET`, `API_TOKEN`. Ein fehlender Wert oder ein ungültiges Base32-Secret beendet den Start mit Konfigurationsfehler.

---

## Secrets anlegen und versiegeln

SealedSecrets sind an Namespace und Name gebunden. Die ISC-Werte des News-Bots lassen sich deshalb nicht kopieren, sie müssen für `isc-manager-bot` neu versiegelt werden. Bis dahin ist `secrets.enabled: false` und `replicaCount: 0`, es läuft kein Pod.

1. **SEWOBE-Benutzer mit 2FA.** Eigener Benutzer mit Lesezugriff auf die Gliederung 1002011. Das Base32-Secret des Authenticators (nicht den 6-stelligen Code) bereithalten.
2. **Versiegeln.** Aus dem Repo-Wurzelverzeichnis im TECH-Kontext:
   ```bash
   export KUBECONFIG=~/.kube/homeserver.yaml
   ISC_USERNAME=… ISC_PASSWORD=… SEWOBE_USERNAME=… SEWOBE_PASSWORD=… SEWOBE_TOTP_SECRET=… scripts/seal-isc-manager-bot.sh
   ```
   Fehlt eine Variable, fragt das Skript sie ohne Echo ab. `API_TOKEN` erzeugt es mit `openssl rand -hex 32`, sofern nicht gesetzt, und gibt ihn einmal aus. Das Skript schreibt die sechs Werte nach `values.yaml`, setzt `secrets.enabled: true` und `replicaCount: 1`.
3. **Committen und pushen.** Danach rollt ArgoCD aus.
4. **Token für n8n.** Den ausgegebenen `API_TOKEN` als Header-Auth-Credential `ISC-Manager-Bot API-Token` in n8n anlegen (Header `Authorization`, Wert `Bearer <Token>`).
5. **Register pflegen.** Die vier Einträge `isc-manager-bot-*` in [`ops/token-register.yaml`](https://github.com/pkr-lab/capulus-core/blob/main/ops/token-register.yaml) tragen nach dem Versiegeln das Ausstellungsdatum (`issued`) ein. `scripts/check-token-register.py` prüft das im CI.

Rotation: Schritte 2 bis 4 wiederholen. Für `API_TOKEN` zuerst die neue Credential in n8n anlegen, dann rollen.

---

## Deployment und Promotion

- Namespace `isc-manager-bot` muss in `argocd_workloads_apps` (`roles/argocd/defaults/main.yml` in capulus-ansible) stehen, danach dort `make render-bootstrap CORE_DIR=<Pfad zu capulus-core>`. Die Dateien unter `argocd/bootstrap/` sind generiert.
- Der Image-Pull-Secret `ghcr-pull` muss im Namespace existieren, wie bei den anderen Apps.
- `build-images.yml` baut `ghcr.io/pkr-lab/isc-manager-bot` bei Änderungen an Dockerfile, `package*.json`, `tsconfig.json`, `src`, `test` und `scripts` und schreibt `sha-<commit>` nach `image.tag`. Das Basisimage `mcr.microsoft.com/playwright` muss zur npm-Version von `playwright` passen (Renovate hebt beide gemeinsam an).
- Die CI führt den Job `isc-manager-bot` (Typecheck, Unit- und Mock-Tests) aus, ohne Zugriff auf die echten Systeme.

---

## n8n-Workflow einrichten

Datei: `argocd/apps/tech/n8n/workflows/isc-manager-bot-sync.json` (importieren, `active` ist `false`).

| Node | Aufgabe |
|---|---|
| E-Mail empfangen | IMAP-Trigger, Credential `Netcup Automation`, Ordner `NewUebertrag`, nur ungelesene Mails |
| Formular Test | Formular mit Vorname, Nachname, DLRG-Manager-ID (optional) und „Testlauf (nichts speichern)“, Standard Ja. Geschützt mit Basic-Auth (Credential `MDM Formulare`), Pfad `isc-manager-test` |
| Konfiguration | Bot-URL und ntfy-Thema `isc-manager` |
| Eingaben pruefen | Vereinheitlicht Mail und Formular zu `{ vorname, nachname, mitgliedsnummer?, dryRun }`, prüft die 13 Ziffern |
| ISC-Manager-Bot abgleichen | `POST /sync`, Timeout 420 s, Credential `ISC-Manager-Bot API-Token` |
| Erfolgreich? | Verzweigt nach `ok` |
| ntfy-Nodes | Meldungen über den Sub-Workflow „Publish: Benachrichtigung“, ohne Namen: Quelle, Ergebnis, Feldnamen, Fehlercode und Schritt |

Nach dem Import: Credentials zuordnen (Header-Auth für den Bot, Basic-Auth für das Formular), Workflow aktivieren.

**Mailformat.** Noch nicht festgelegt. Der Mail-Pfad liest vorläufig den Mailtext als JSON mit `vorname`, `nachname` und optional `mitgliedsnummer`, alles andere endet mit einer ntfy-Meldung „Eingabe ungültig“. Sobald das Format steht, wird der Node „Eingaben pruefen“ angepasst.

**Test mit dem Formular.** Formular aufrufen, Name eintragen, „Testlauf“ auf Ja lassen. Das Ergebnis zeigt die ntfy-Meldung. Den `diff` (alte und neue Werte) sieht man in der Ausführung des Nodes „ISC-Manager-Bot abgleichen“ in n8n. Erst mit „Nein“ wird gespeichert.

---

## Selektoren anpassen bei Änderungen an ISC oder SEWOBE

Alle Selektoren stehen an zwei Stellen: `src/isc/selectors.ts` und `src/sewobe/selectors.ts`. Ändert sich eine Seite, meldet der Bot `FORM_CHANGED` mit dem fehlenden Element. Die Mocks in `test/fixtures/mock-isc.ts` und `mock-sewobe.ts` müssen dieselben Selektoren bedienen.

### Lokal testen

```bash
cd argocd/apps/tech/isc-manager-bot
npm ci
npx playwright install chromium
npm test
```

Gegen die echten Systeme (nur Dry-Run, nichts wird gespeichert):

```bash
ISC_USERNAME=… ISC_PASSWORD=… SEWOBE_USERNAME=… SEWOBE_PASSWORD=… SEWOBE_TOTP_SECRET=… \
SMOKE_VORNAME=… SMOKE_NACHNAME=… npm run smoke
```

---

## Troubleshooting

| Symptom | Ursache | Abhilfe |
|---|---|---|
| Pod startet nicht, `CreateContainerConfigError` | SealedSecret fehlt oder nicht entschlüsselt | `kubectl -n isc-manager-bot get sealedsecret,secret`, Namespace und Name beim Versiegeln prüfen |
| Pod beendet sich mit „Konfigurationsfehler“ | Variable fehlt, URL ohne `https://`, TOTP-Secret nicht Base32 | `kubectl -n isc-manager-bot logs deploy/isc-manager-bot` |
| `TOTP_FAILED` | Secret falsch, Uhr des Nodes weicht ab | Secret neu versiegeln, `chronyc tracking` auf dem Node prüfen |
| `LOGIN_FAILED` mit `step: sewobe-login` | Passwort falsch oder Benutzer gesperrt | Anmeldung von Hand prüfen |
| `SEWOBE_NO_MATCH` | Schreibweise weicht ab, Person ist ausgeschieden oder die Suche findet sie nicht | In SEWOBE von Hand suchen, ggf. `mitgliedsnummer` mitschicken |
| `SEWOBE_AMBIGUOUS` | Mehrere aktive Personen gleichen Namens | `mitgliedsnummer` mitschicken |
| `ISC_CONFLICT` | Gleicher Name und Geburtstag im ISC, aber andere DLRG-Manager-Id | Im ISC klären, der Bot ändert nichts |
| `ISC_AMBIGUOUS` | Mehrere gleiche Personen im ISC | Dubletten im ISC bereinigen |
| `GESCHLECHT_FEHLT` | Neue Person, Anrede „Keine Anrede“ oder leer | In SEWOBE die Anrede pflegen |
| `LAND_UNBEKANNT` | Land in SEWOBE heißt im ISC anders | Land in SEWOBE prüfen |
| `FORM_CHANGED` | Seite im ISC oder SEWOBE hat sich geändert | Selektor anpassen, siehe oben |
| `SAVE_UNCONFIRMED` | ISC hat nicht bestätigt oder zeigt nach dem Speichern andere Werte | Person im ISC prüfen, Lauf wiederholen (idempotent) |
| `BUSY` (429) | Anderer Lauf aktiv | n8n wiederholt später |

---

## Risiken

- **RPA gegen zwei Webseiten.** Ändert eines der Systeme seine Oberfläche, bricht der Lauf mit `FORM_CHANGED`. Er speichert dann nichts Halbes, weil nach jedem Schritt zurückgelesen wird.
- **Der Bot lief bisher nur gegen die Mocks.** Die Selektoren und Abläufe stammen aus Beobachtungen im echten ISC und in SEWOBE (Personenliste, Grunddaten speichern, Person neu anlegen, Adresse anlegen und ändern, Suche, Personenseite). Das Absenden von „Neue Mitgliedschaft“ und der komplette Lauf mit echtem TOTP wurden nicht live ausgeführt. Vor dem ersten Produktivlauf zuerst `npm run smoke` (Dry-Run), dann das Test-Formular mit `Testlauf = Ja` und erst danach mit `Nein` an einer Testperson ausprobieren.
- **Der Bot kann Personen im ISC anlegen und ändern.** Der ISC-Account braucht deshalb Personenrechte. Die Rechte so eng wie möglich halten.
- **Zweiter Faktor liegt im Cluster.** Wer `SEWOBE_TOTP_SECRET` und das Passwort hat, hat vollen SEWOBE-Zugang. Das Secret nur als SealedSecret ablegen und im Token-Register führen.
