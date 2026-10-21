# Debugging und Fehlersuche

Zentrale Seite für die iOS-App. Funktionsspezifisches steht auf den Seiten [Übersicht und Alerts](uebersicht-und-alerts.md), [Steuerung](steuerung.md), [Alltag-Modus](alltag-modus.md) und [Verbindung und Sicherheit](verbindung-und-sicherheit.md). Das projektübergreifende Kapitel ist die [zentrale Fehlersuche](../capulus-core/2-betrieb-hardware/20070-troubleshooting-zentral.md). Das Backend hat eine eigene [Fehlerbehebung](../capulus-core/3-apps-workloads/300d0-carplay-api.md#fehlerbehebung).

---

## Wie die App Fehler behandelt

- Es gibt **keine Logs** in der App: kein `print`, kein `os.Logger`, kein Debug-Schalter. Es gibt kein Log-Level, das man hochdrehen könnte.
- Fehler werden je Ansicht als rote Zeile angezeigt (Steuerung, Alltag-Modus, Einstellungen) oder stumm verworfen: Die Übersicht und die Alerts zeigen **keine** Fehlermeldung, nur den gelben Hinweis „Homeserver nicht erreichbar — zeige letzten bekannten Stand.“. Der Abruf der Update-Liste verwirft Fehler ohne Spur.
- Der Fehlertext stammt aus `LocalizedError.errorDescription` der eigenen Typen oder aus dem Systemtext von `URLError`.
- Unbekannte Werte in Antworten werden tolerant behandelt (Alert-Stufe → `info`, Monitor-Status → `down`, fehlende Listen → leer). Fehlen Pflichtfelder (`status`, `updated_at` im Dashboard), schlägt das Decodieren fehl.

---

## Fehlermeldungen aus dem Code

### Verbindung zu `carplay-api` und WoL-Agent

| Meldung | Ursache | Lösung |
|---|---|---|
| `carplay-api returned HTTP <n>` | Statuscode ≠ 200 ohne `error` im Body | Code deuten: 401 Token, 403 Allowlist, 429 Rate-Limit, 502 power-agent, 503 fehlendes Secret ([carplay-api](../capulus-core/3-apps-workloads/300d0-carplay-api.md#fehlerbehebung)) |
| `<Text des Servers>` (aus JSON `{"error": "…"}`) | Backend lehnt die Anfrage ab, zum Beispiel falscher Bestätigungscode | Text lesen, Ursache im Backend ([Steuerung](steuerung.md#debugging-und-fehlersuche)) |
| `Could not parse response: <Detail>` | Antwort ist kein erwartetes JSON (Pflichtfeld fehlt oder anderer Typ) | Antwort mit `curl` prüfen, App und Backend-Version abgleichen |
| `No API token stored in Keychain — open Settings to add one.` | Token des WoL-Agents fehlt (gilt nur dort) | Einstellungen → Vereinsheim-WoL-Agent-Token |
| *(keine App-Meldung)* Statuscode 401 | Token von `carplay-api` fehlt oder falsch. Ohne Token sendet die App die Anfrage ohne Header | Einstellungen → carplay-api Token |
| iOS-Systemtext „cancelled“ / „abgebrochen“ | Pinning lehnt das Zertifikat ab | [Verbindung und Sicherheit](verbindung-und-sicherheit.md#debugging-und-fehlersuche) |
| iOS-Systemtext „The request timed out.“ | kein Antwort in 10 Sekunden | VPN, Netz, Cluster prüfen |
| iOS-Systemtext „A server with the specified hostname could not be found.“ | Name nicht auflösbar | Tailscale, DNS |
| iOS-Systemtext „Could not connect to the server.“ | Host erreichbar, aber Port geschlossen oder Dienst down | Backend-Pod prüfen (`kubectl -n carplay-api get pods`) |

### Alltag-Modus

| Meldung | Ursache | Lösung |
|---|---|---|
| `Wetter konnte nicht geladen werden.` | Open-Meteo antwortete nicht mit 200 | [Alltag-Modus](alltag-modus.md#debugging-und-fehlersuche) |
| `Pegelstand konnte nicht geladen werden.` | Pegelonline nicht erreichbar | wie oben |
| `Kein Tankerkönig-API-Key hinterlegt — in den Einstellungen ergänzen.` | Key fehlt | Einstellungen |
| `Tankerkönig antwortete mit HTTP <n>.` | HTTP-Fehler der API | Key und Nutzungslimits prüfen |
| `Antwort von Tankerkönig konnte nicht gelesen werden.` | Formatänderung | `FuelModels.swift` |
| `Tankpreise konnten nicht geladen werden.`, `Tankstellen in der Nähe konnten nicht geladen werden.` | API meldete `ok: false` ohne Text | Key prüfen |
| `Keine passende Station im Umkreis von 10 km gefunden.` | nur ausgeschlossene Marken | `Constants.swift` |
| `Standortzugriff wurde nicht erlaubt — in den iOS-Einstellungen für diese App aktivieren.` | Standort abgelehnt | iOS-Einstellungen |
| `Nicht verfügbar` | Nachrichtenquelle nicht lesbar | `curl` auf die Quelle |

### Einstellungen

| Meldung | Ursache | Lösung |
|---|---|---|
| `Speichern fehlgeschlagen: <Detail>` | Keychain verweigert (`KeychainError.unhandled(status)`) | Gerät entsperren, `OSStatus` nachschlagen |
| `Es ist bereits ein Token gespeichert.` / `… ein Key …` | Hinweis, kein Fehler | Speichern überschreibt |

---

## Checkliste „Feature funktioniert nicht“

1. **Welcher Modus?** Alltag braucht nur Internet (und den Tankerkönig-Key), PKR-Lab braucht Heimnetz oder Tailscale.
2. **Netz:** Tailscale auf dem Gerät aktiv? `https://carplay-api.prod.homeserver/health` in Safari lädt? (Safari nutzt den System-Trust. Ein Zertifikatsfehler dort ist auch ein Hinweis für die App.)
3. **Token:** Einstellungen → „Es ist bereits ein Token gespeichert.“ Steht dort nichts, Token neu eintragen. Der Wert muss dem Klartext des SealedSecrets entsprechen.
4. **Backend direkt:**
   ```bash
   curl --cacert pki/homeserver-root-ca.pem -H "Authorization: Bearer <TOKEN>" https://carplay-api.prod.homeserver/api/dashboard
   ```
   `200` mit JSON: App-Problem. Fehler: Backend-Problem.
5. **Backend-Logs:** `kubectl -n carplay-api logs deploy/carplay-api --tail=100` (strukturierte JSON-Zeilen mit Methode, Pfad, Status, Dauer, ohne Header).
6. **Zertifikat:** `openssl s_client -connect carplay-api.prod.homeserver:443 -CAfile pki/homeserver-root-ca.pem -verify_return_error </dev/null`. Muss `Verify return code: 0 (ok)` zeigen.
7. **Bundle-CA:** `Resources/homeserver-root-ca.pem` entspricht `pki/homeserver-root-ca.pem` in capulus-core und läuft nicht ab.
8. **power-agent (nur Steuerung):** `systemctl status power-agent` und `journalctl -u power-agent -f` auf dem Homeserver ([Rolle power_agent](../capulus-ansible/rollen/power_agent.md)).
9. **Neu bauen:** Hilft nach Änderungen an `Constants.swift` oder der Root-CA. Das Projekt zuvor mit `xcodegen generate` neu erzeugen.

---

## In Xcode debuggen

- **Fehlerursache der Übersicht lesen:** Haltepunkt in `DashboardViewModel.fetchDashboard()` im `catch`-Block. Dort steht der Fehler (Variable `error`) mit Domäne und Code. Der Wert `self.error` wird sonst nirgends angezeigt.
- **HTTP-Verkehr:** Haltepunkt in `HomeserverAPIClient.request(...)` nach dem `await`, Variable `response` zeigt den Statuscode, `data` den Body mit dem `error`-Text. Mit `po String(data: data, encoding: .utf8)` im Debugger-Fenster lesbar.
- **TLS-Pinning:** Haltepunkt in `MTLSDelegate.urlSession(_:didReceive:completionHandler:)`. Dort sieht man `space.host`, ob `rootCertificate` `nil` ist und das Ergebnis von `SecTrustEvaluateWithError` (Variable `error`).
- **Simulator oder Gerät:** Der Simulator nutzt das Netz des Macs, nicht das Tailnet des iPhones. Tests gegen den Homeserver gehören auf ein Gerät mit Tailscale.
- **Konsole:** Die App schreibt keine eigenen Logzeilen. Systemmeldungen zu Netzwerk und Zertifikaten stehen in der Xcode-Konsole oder in der Konsole-App des Macs (Gerät auswählen, nach dem Prozessnamen filtern).
- **Keychain zurücksetzen:** App vom Gerät löschen und neu installieren. TODO: prüfen, ob Keychain-Einträge nach Deinstallation erhalten bleiben (iOS-Standard ist, dass sie bleiben können). Alternativ in den Einstellungen die Einträge entfernen.

---

## Typische Probleme zwischen den Repos

| Symptom | Ursache | Lösung |
|---|---|---|
| Dashboard leer, App läuft | Backend-Konfiguration (`config.hosts`, Uptime-Kuma-Slug) | [carplay-api: Fehlerbehebung](../capulus-core/3-apps-workloads/300d0-carplay-api.md#fehlerbehebung) |
| Nach Rotation des ArgoCD-Passworts schlägt der Homeserver-Shutdown mit 401 fehl | `secrets.shutdownConfirmationCode` nicht mit erneuert | [Secrets-Rotation](../capulus-core/d-sicherheit/d0060-secrets-rotation.md), [carplay-api](../capulus-core/3-apps-workloads/300d0-carplay-api.md#ersteinrichtung) |
| Root-CA wurde in capulus-core getauscht, App meldet Verbindungsfehler | App enthält alte CA, CI `ca-checksum` ist rot | neue Datei kopieren, App neu bauen, [Internes TLS](../capulus-core/d-sicherheit/d0040-internal-tls.md) |
| Neuer Host soll in der App erscheinen | Host fehlt in `config.hosts` des Backends | Helm-Values von `carplay-api` |
| Updates-Karte ist leer | github-release-watcher hat die ConfigMap nicht geschrieben | [github-release-watcher](../capulus-core/f-cicd-automatisierung/f0040-github-release-watcher.md) |
| WoL für das Vereinsheim scheitert | Agent oder Token auf dem Banana Pi, Router-WoL | [Alarmmonitor](../capulus-ansible/3-apps-workloads/30020-vereinsheim-alarmmonitor.md), [banana_pi_kiosk](../capulus-ansible/rollen/banana_pi_kiosk.md) |
| `Dienste`-Kacheln öffnen nichts | Host im Browser nicht auflösbar | DNS und Root-CA im Browser des Geräts |
| Build schlägt in der CI fehl | `xcodegen` oder `xcodebuild` | siehe [Entwicklung](entwicklung.md#ci) |

Allgemeines Vorgehen, wenn nichts hilft: Backend per `curl` prüfen, dann Zertifikat, dann Token. In dieser Reihenfolge sind die Ursachen von außen nach innen eingegrenzt.
