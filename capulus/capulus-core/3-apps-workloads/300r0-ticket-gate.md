# ticket-gate — E-Mail-Bestätigung vor dem Ticketformular

> **Cluster:** TECH · Ordner `argocd/apps/tech/ticket-gate/` · URL `https://ticket-gate-tech.pke-lab.de` (extern), `ticket-gate.tech.homeserver` (intern). `kubectl`-Befehle in diesem Doc gelten für den TECH-Cluster ([Zugriff je Cluster](../../capulus-ansible/a-betriebssystem/a0010-overview.md#kubectl-zugriff-je-cluster)).

Kleiner Go-Dienst (nur Standardbibliothek, keine Abhängigkeiten) zwischen dem Kontaktformular auf `www.edv-kretzer.de` und [Zammad](30000-zammad.md). Ein Ticket entsteht erst, nachdem die Absenderin oder der Absender die E-Mail-Adresse mit einem 6-stelligen Code bestätigt hat. Der Dienst legt das Ticket selbst über die Zammad-API an, **die Adresse im Ticket stammt aus der Bestätigung, nicht aus dem Formular**.

## Warum ein eigener Dienst

Das frühere Formular war das Zammad-Widget (`form.js`), das direkt an `support-tech.pke-lab.de/api/v1/form_submit` sendet. Dieser Endpunkt nimmt jede Adresse an, eine Prüfung im Browser wäre mit einem direkten Request zu umgehen. Eine Prüfung, die nicht umgehbar ist, geht nur serverseitig vor Zammad. Details und Entscheidungen: [60040, ticket-gate](../6-hintergruende/60040-helm-charts-tech.md#ticket-gate).

---

## Ablauf

```
Browser (www.edv-kretzer.de, js/kontakt-verifizierung.js)
   │ 1. POST /v1/session                   → session_token (nur im Speicher der Seite)
   │ 2. POST /v1/verify/request {email}    → 202, Mail mit Code geht raus
   │ 3. POST /v1/verify/confirm {code}     → verification_token
   │ 4. POST /v1/tickets {token, Inhalt}   → Ticket in Zammad, Nummer zurück
   ▼
ticket-gate (Namespace ticket-gate)
   │  Mail:   SMTP netcup (Port 465, TLS)
   │  Ticket: http://zammad-nginx.zammad.svc.cluster.local:8080/api/v1/tickets
```

| Schritt | Was der Server tut |
|---|---|
| Session | legt eine zufällige Session an (2 Stunden), speichert nur ihren HMAC |
| Code senden | normalisiert die Adresse, prüft Limits, erzeugt den Code, speichert den HMAC, versendet die Mail im Hintergrund, **antwortet immer gleich** |
| Code prüfen | zählt den Versuch, vergleicht zeitkonstant, verwirft den Code bei Erfolg, nach 5 Fehlversuchen oder nach Ablauf, stellt das Token aus |
| Ticket | verbraucht das Token atomar, prüft Session-Bindung, legt das Ticket mit der verifizierten Adresse an |

Ein neuer Code für dieselbe Session ersetzt den alten. Ein neues Token für dieselbe Session ersetzt das alte. Das Token wird auch verbraucht, wenn Zammad den Fehler `502 ticket_failed` meldet: Wer danach erneut senden will, bestätigt die Adresse neu.

## Endpunkte

Alle Endpunkte sind `POST` mit `Content-Type: application/json`, Body maximal 32 KiB, unbekannte JSON-Felder werden abgelehnt. Die Session wird als `Authorization: Bearer <session_token>` geschickt. Es gibt keine Cookies, deshalb ist CSRF ausgeschlossen (kein ambient Credential). Zusätzlich lässt der Dienst nur die Origins aus `ALLOWED_ORIGINS` zu.

| Pfad | Body | Antwort |
|---|---|---|
| `POST /v1/session` | — | `200 {session_token, expires_in}` |
| `POST /v1/verify/request` | `{email}` | `202 {status:"ok", resend_after}` |
| `POST /v1/verify/confirm` | `{code}` | `200 {verification_token, email, expires_in}` |
| `POST /v1/tickets` | `{verification_token, name?, subject, message}` | `201 {ticket_number}` |
| `GET /healthz` | — | `200 ok` (Probes) |

Fehler haben die Form `{"error": "<code>"}`, bei `429` zusätzlich `retry_after` (Sekunden) und den Header `Retry-After`.

| Code | Status | Bedeutung |
|---|---|---|
| `session_invalid` | 401 | Session fehlt, unbekannt oder abgelaufen |
| `email_invalid` | 400 | Adresse syntaktisch ungültig (oder enthält Steuerzeichen) |
| `code_invalid` | 400 | Code falsch oder nicht 6 Ziffern, `attempts_left` gibt die Restversuche an |
| `code_locked` | 400 | 5. Fehlversuch, Code verworfen |
| `code_expired` | 400 | kein gültiger Code (abgelaufen, schon benutzt, nie angefordert) |
| `verification_required` | 403 | Token fehlt, ist falsch, abgelaufen, verbraucht oder gehört zu einer anderen Session |
| `ticket_invalid` | 400 | Betreff oder Nachricht leer oder zu lang |
| `ticket_failed` | 502 | Zammad hat das Ticket nicht angelegt |
| `rate_limited` | 429 | Limit erreicht |
| `request_invalid` | 400 | JSON fehlerhaft oder unbekanntes Feld (z. B. ein mitgeschicktes `email` beim Ticket) |
| `origin_not_allowed` | 403 | Origin nicht in `ALLOWED_ORIGINS` |
| `unavailable` | 503 | zu viele offene Sessions |

## Datenmodell

Alles liegt **im Arbeitsspeicher** des einen Pods (`replicaCount: 1`, Strategie `Recreate`). Ein Neustart verwirft Sessions, Codes, Token und Zähler, Nutzer fordern dann einfach einen neuen Code an. Die Zugriffe laufen über `internal/store`, eine Redis-Anbindung wäre dort nachrüstbar.

| Objekt | Schlüssel | Inhalt | Lebensdauer |
|---|---|---|---|
| Session | `HMAC("session", token)` | Ablaufzeit | 2 h (`SESSION_TTL_SECONDS`) |
| Code | Session-Schlüssel | Adresse, `HMAC("code", session, adresse, code)`, Ablaufzeit, Fehlversuche | 10 min (`CODE_TTL_SECONDS`) |
| Token | `HMAC("token", token)` | Session-Schlüssel, Adresse, Ablaufzeit | 30 min (`TOKEN_TTL_SECONDS`) |
| Zähler | Schlüssel je Regel (IP-Adresse bzw. HMAC der Adresse) | Zeitstempel | bis zum Ende des Fensters |

Klartext-Codes, Klartext-Token und Klartext-Sessions liegen nirgends. Ein Aufräumjob löscht abgelaufene Einträge jede Minute (`Purge`), abgelaufene Einträge werden außerdem beim Zugriff verworfen.

## Konfiguration

Die Werte kommen aus `values.yaml` (`config:` → ConfigMap) und dem SealedSecret `ticket-gate-secrets`. Fehlt eine Pflichtvariable, beendet sich der Dienst beim Start mit einer Fehlermeldung.

| Variable | Standard | Bedeutung |
|---|---|---|
| `ALLOWED_ORIGINS` | — (Pflicht) | erlaubte Origins, kommagetrennt (`https://www.edv-kretzer.de,https://edv-kretzer.de`) |
| `HMAC_SECRET` | — (Pflicht, Secret) | mindestens 32 Zeichen, Schlüssel für alle Hashes |
| `SMTP_HOST` / `SMTP_PORT` / `SMTP_SECURITY` | — / `465` / `tls` | `tls` (impliziertes TLS) oder `starttls` |
| `SMTP_USERNAME` / `SMTP_PASSWORD` | — (Pflicht, Passwort als Secret) | Zugang zum Postfach |
| `SMTP_FROM` / `SMTP_FROM_NAME` | — (Pflicht) / `EDV-Kretzer` | Absender |
| `ZAMMAD_URL` | — (Pflicht) | interne Basis-URL von Zammad |
| `ZAMMAD_TOKEN` | — (Pflicht, Secret) | Zammad-API-Token |
| `ZAMMAD_GROUP` | `Users` | Gruppe der neuen Tickets |
| `CLIENT_IP_HEADER` | leer | Header mit der Client-IP (`CF-Connecting-IP`); leer = Adresse der Verbindung |
| `CODE_TTL_SECONDS` / `TOKEN_TTL_SECONDS` / `SESSION_TTL_SECONDS` | `600` / `1800` / `7200` | Gültigkeiten |
| `MAX_ATTEMPTS` | `5` | Fehlversuche je Code |
| `MAIL_COOLDOWN_SECONDS` | `60` | Mindestabstand je Adresse |
| `MAIL_PER_EMAIL_PER_HOUR` / `MAIL_PER_IP_PER_HOUR` | `5` / `10` | Mails pro Stunde je Adresse bzw. IP |
| `CONFIRM_PER_IP` / `CONFIRM_WINDOW_SECONDS` | `20` / `600` | Prüfversuche je IP im Fenster |
| `SESSION_PER_IP_PER_HOUR` / `TICKET_PER_IP_PER_HOUR` | `30` / `10` | Sessions und Tickets je IP und Stunde |
| `MAX_SESSIONS` | `50000` | Obergrenze offener Sessions |
| `PORT` / `LOG_LEVEL` | `8080` / `info` | |

Secrets (`HMAC_SECRET`, `SMTP_PASSWORD`, `ZAMMAD_TOKEN`) stehen im [Token-Register](https://github.com/pkr-lab/capulus-core/blob/main/ops/token-register.yaml) als `ticket-gate-hmac-secret`, `ticket-gate-smtp-password` und `ticket-gate-zammad-token`.

## Sicherheitsmaßnahmen

| Anforderung | Umsetzung |
|---|---|
| Prüfung nur serverseitig | `POST /v1/tickets` prüft und verbraucht das Token selbst; ohne gültiges Token `403`, ohne Session `401`. Zammad nimmt öffentlich keine Formulare mehr an ([Zammad sperren](#zammad-absichern)) |
| Zufall | `crypto/rand`, `rand.Int` über 1.000.000 Werte (gleichverteilt), formatiert mit führenden Nullen |
| Hashing | HMAC-SHA256 mit Server-Secret, längenpräfixierte Eingaben; Code, Adresse und Session gehen in den Hash ein |
| Kein Klartext | Code nur in der Mail; weder in Antworten noch im Speicher noch im Log. Das Log enthält auch keine Adressen |
| Zeitkonstant | `hmac.Equal` auf den Hashes |
| Ablauf, Versuche, Einmaligkeit | 10 min, 5 Versuche, Code wird bei Erfolg, Sperre und Ablauf gelöscht; neuer Code ersetzt alten |
| Bindung | Code an Adresse und Session, Token an Session und Adresse; Token ist einmalig (atomar) und 30 min gültig |
| Adresse aus dem Token | `/v1/tickets` kennt kein Adressfeld, ein mitgeschicktes `email` führt zu `400` |
| Rate Limits | Mail: 1 pro 60 s und 5 pro Stunde je Adresse, 10 pro Stunde je IP. Prüfen: 20 je 10 min je IP. Session und Ticket je IP. Alle Regeln einer Anfrage werden gemeinsam gebucht („alles oder nichts“) |
| Keine Preisgabe | Antwort auf `verify/request` ist für jede gültige Adresse identisch, Versand läuft im Hintergrund, Zustellfehler gehen nur ins Log. Es gibt kein „Adresse bekannt“ oder „hat Tickets“ |
| Adresse | `trim`, Kleinschreibung, strenge Syntaxprüfung (ASCII, kein Display-Name, keine Steuerzeichen, max. 254 Zeichen) |
| Header-Injection | Adresse wird vorher validiert, `BuildMessage` lehnt CR, LF und NUL in jedem Header ab, Betreff ist konstant (RFC 2047) |
| CSRF | keine Cookies, Session per Header; zusätzlich Origin-Allowlist, `application/json` Pflicht |
| Aufräumen | `Purge` jede Minute, Zähler laufen am Fensterende aus, Obergrenze für Sessions und Zähler |
| Mail-Inhalt | Betreff „Ihr Bestätigungscode“, Code, Gültigkeit, Hinweis zum Ignorieren, kein Link |
| TLS | SMTP mit TLS ≥ 1.2 und Zertifikatsprüfung |

Grenzen: Die Zähler hängen an der Client-IP aus `CF-Connecting-IP`. Cloudflare setzt den Header selbst; über den internen Host `ticket-gate.tech.homeserver` (LAN) könnte jemand ihn setzen und nur sein **eigenes** IP-Limit umgehen, die Limits je Adresse bleiben bestehen. Weil der Zustand im Speicher liegt, setzt ein Neustart alle Zähler zurück.

## Zammad absichern

Damit der Dienst nicht zu umgehen ist, darf Zammad keine anonymen Tickets mehr annehmen.

1. **Traefik:** `argocd/apps/tech/zammad/templates/ingress-form-sperre.yaml` führt `/api/v1/form_submit` und `/api/v1/form_config` auf dem öffentlichen Host durch eine Middleware (`ipAllowList` nur `127.0.0.1/32`, also `403` für alle). Der interne Host `zammad.tech.homeserver` bleibt unberührt. Schalter: `formSperre.enabled`.
2. **Zammad-Admin (von Hand):** *Kanäle → Formular* deaktivieren und *Kanäle → Chat* deaktivieren. Das Chat-Widget läuft über den WebSocket `/ws`, den die Agenten-Oberfläche ebenfalls nutzt, er lässt sich nicht per Pfad sperren. Die Chat-Einbindung (Button, `chat.min.js`, jQuery) ist von allen Seiten der Website entfernt, ebenso der Chat-Abschnitt der Datenschutzerklärung.
3. **API-Token:** In Zammad einen Benutzer mit Rolle *Agent* (Gruppe aus `ZAMMAD_GROUP` mit Vollzugriff) anlegen, unter *Profil → Token-Zugriff* ein Token mit Recht `ticket.agent` erzeugen. Das Token legt Tickets für beliebige Kunden an (`customer_id: "guess:<adresse>"`), behandle es wie ein Admin-Secret.

## Inbetriebnahme

1. **capulus-ansible:** `ticket-gate` in `argocd_workloads_apps` und `argocd_network_policy_refined_namespaces` eintragen, unter `argocd_network_policy_extra_ingress` den Eintrag `zammad: [ticket-gate]` ergänzen, dann dort `make render-bootstrap CORE_DIR=<Pfad zu capulus-core>` ausführen und das generierte `argocd/bootstrap/*` in capulus-core mitnehmen.
2. **Zugangsdaten** in `~/ticket-gate-credentials-2026-10-30.txt` ablegen (Zeilen `smtp_password=…` und `zammad_token=…`), dann `scripts/seal-ticket-gate.sh` ausführen. Das Skript erzeugt das `HMAC_SECRET` selbst (`openssl rand -hex 32`), versiegelt alle drei Werte für den Namespace `ticket-gate` und setzt `secrets.enabled: true`.
3. PR mergen. `build-images.yml` baut `ghcr.io/pkr-lab/ticket-gate` und bumpt `image.tag`; bis dahin steht der Platzhalter `sha-0000000` in `values.yaml`.
4. Die Website (EDV-Kretzer) erst nach dem Rollout deployen, sonst zeigt das Formular Netzwerkfehler.
5. In Zammad Formular und Chat deaktivieren und das Token anlegen (siehe oben, ohne Token schlägt `ticket_failed` an).

## Tests ausführen

```bash
cd argocd/apps/tech/ticket-gate/src
go vet ./...
go test -count=1 ./...
```

Ohne lokales Go: `docker run --rm -v "$PWD":/src -w /src golang:1.27-alpine go test -count=1 ./...`. Der Docker-Build führt die Tests ebenfalls aus (`go test ./...` vor dem `go build`).

| Testdatei | Deckt ab |
|---|---|
| `internal/gate/verify_test.go` | Ticket ohne Session, ohne Verifikation, mit erfundenem Token; Adressfeld im Ticket abgelehnt; Token einmalig, an Session gebunden, läuft nach 30 min ab, neues Token ersetzt altes; falscher, abgelaufener, wiederverwendeter Code; Code einer anderen Adresse (andere und gleiche Session); Code aus anderer Session; neuer Code macht alten ungültig; Sperre nach 5 Fehlversuchen; Format und Verteilung der Codes; kein Klartext in Antworten, Speicher und Log; identische Antworten für beliebige Adressen; ungültige Adressen und Header-Injection; alle Rate Limits; Aufräumen; Origin/CORS, Content-Type und Body-Grenze; gleichzeitige Nutzung eines Tokens erzeugt genau ein Ticket |
| `internal/mail/` | Aufbau der Mail, Ablehnung von Header-Injection, Versand gegen einen Fake-SMTP-Server mit TLS, Ablehnung eines nicht vertrauenswürdigen Zertifikats |
| `internal/zammad/` | Anfrage an Zammad mit verifizierter Adresse, Fehlerstatus |
| `internal/config/`, `internal/limiter/` | Pflichtvariablen, Standardwerte, Limiter-Logik |

## Rotation

| Secret | Vorgehen |
|---|---|
| `HMAC_SECRET` | `scripts/seal-ticket-gate.sh` erneut ausführen (erzeugt ein neues Secret), PR mergen. Offene Sessions, Codes und Token werden ungültig, Nutzer fordern einen neuen Code an |
| `SMTP_PASSWORD` | beim Hoster ändern, in die Credentials-Datei eintragen, Skript ausführen. Dasselbe Postfach nutzt [Vaultwarden](300a0-vaultwarden.md) |
| `ZAMMAD_TOKEN` | in Zammad neues Token erzeugen, in die Credentials-Datei eintragen, Skript ausführen, danach das alte Token in Zammad löschen |

## Fehlersuche

| Symptom | Ursache | Lösung |
|---|---|---|
| Pod startet nicht, Log „invalid configuration“ | Pflichtvariable fehlt, `secrets.enabled: false` | Meldung im Log lesen, Secrets versiegeln |
| Formular meldet „Keine Verbindung“ | Ingress/Tunnel, Pod nicht bereit | `kubectl -n ticket-gate get pods,ingress`, `curl -I https://ticket-gate-tech.pke-lab.de/healthz` |
| `origin_not_allowed` | Seite wird von einer Adresse außerhalb `ALLOWED_ORIGINS` geöffnet (z. B. `localhost`) | Origin ergänzen oder die echte Seite testen |
| Code kommt nicht an | SMTP-Zugang, Spam-Ordner; der Dienst meldet Zustellfehler nur im Log | `kubectl -n ticket-gate logs deploy/ticket-gate`, Meldung „code mail failed“ |
| `ticket_failed` | Zammad nicht erreichbar, Token ohne Rechte, Gruppe fehlt | Log, Token und `ZAMMAD_GROUP` prüfen, NetworkPolicy `zammad: [ticket-gate]` |
| Viele `rate_limited` im Betrieb | alle Besucher erscheinen mit einer IP | `CLIENT_IP_HEADER` gesetzt? Cloudflare-Tunnel liefert `CF-Connecting-IP` |
