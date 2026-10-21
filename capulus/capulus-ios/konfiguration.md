# Konfiguration

Code: [Utilities/Constants.swift](https://github.com/pkr-lab/capulus-ios/blob/main/HomeserverDashboard/Utilities/Constants.swift), [Services/KeychainService.swift](https://github.com/pkr-lab/capulus-ios/blob/main/HomeserverDashboard/Services/KeychainService.swift), [project.yml](https://github.com/pkr-lab/capulus-ios/blob/main/project.yml)

Die App hat keine Konfigurationsdatei und keine Feature-Flags. Alle Adressen und festen Werte stehen im Quelltext in `Constants.swift`, geheime Werte im Keychain des Geräts. Eine Änderung der Konstanten erfordert einen neuen Build.

---

## Konstanten (`Constants.swift`)

| Konstante | Wert | Bedeutung |
|---|---|---|
| `apiBaseURL` | `https://carplay-api.prod.homeserver` | Backend `carplay-api` (läuft im TECH-Cluster, `prod` ist nur die URL-Stufe, siehe [Domain-Tiers](../capulus-core/c-netzwerk-dns/c0040-domain-tiers.md)) |
| `wolAgentBaseURL` | `https://100.123.214.4:9102` | WoL-Agent auf dem Banana Pi im Vereinsheim, Tailscale-IP |
| `refreshInterval` | 30 Sekunden | Pause zwischen zwei Abrufen von `GET /api/dashboard` |
| `requestTimeout` | 10 Sekunden | Timeout je Anfrage an `carplay-api` und WoL-Agent |
| `maxAlertTitleLength` | 20 | Alert-Titel werden in der Liste auf 20 Zeichen gekürzt (`…`) |
| `maxAlertSubtitleLength` | 30 | Alert-Nachricht auf 30 Zeichen |
| `Weather.latitude`, `longitude`, `locationLabel` | 50.4205, 7.4061, „Andernach, Deutschland“ | Wetterstandort (fest) |
| `Weather.openMeteoForecastURL` | `https://api.open-meteo.com/v1/forecast` | Wetterquelle |
| `Tankerkoenig.baseURL` | `https://creativecommons.tankerkoenig.de/json` | Preis-API |
| `Tankerkoenig.nearbySearchRadiusKm` | 10 | Umkreis der Suche nach der nächsten Station |
| `Tankerkoenig.excludedBrands` | Shell, Aral, Esso, Agip, Eni | Marken, die bei „Nächste“ übersprungen werden |
| `Tankerkoenig.fixedStations` | Andernach, Plaidt, Mülheim-Kärlich | drei Stationen mit fester Tankerkönig-ID |
| `Pegel.baseURL`, `stationUUID`, `stationLabel` | Pegelonline REST v2, UUID der Station, „Rhein-Pegel Andernach“ | Pegelstand |
| `SelfHostedServices.all` | Nextcloud, Immich, Vaultwarden, Paperless-ngx, n8n, Wiki.js, Zammad | Kacheln „Dienste“ im Alltag-Dashboard, jeweils `https://<host>` |
| `News.tagesschauURL`, `heiseFeedURL`, `weltFeedURL` | Tagesschau-API, Heise-Atom, WELT-RSS | Nachrichtenquellen |
| `News.maxSummaryLength` | 160 | Zusammenfassung wird gekürzt |
| `Keychain.service` | `com.yourname.homeserver-dashboard` | Dienstname aller Keychain-Einträge |

Die Host-Karten der Übersicht (Homeserver, NAS, worker-0, worker-1) werden nicht in der App festgelegt, sondern vom Backend geliefert (`config.hosts` in den Helm-Values von `carplay-api`, siehe [carplay-api: Konfiguration](../capulus-core/3-apps-workloads/300d0-carplay-api.md#konfiguration-valuesyaml)). Die App zeigt nur die Hosts, die online sind.

---

## Keychain-Einträge

Die App speichert drei Werte mit `kSecClassGenericPassword` und `kSecAttrAccessibleWhenUnlockedThisDeviceOnly` (nur bei entsperrtem Gerät lesbar, nicht in Backups oder auf andere Geräte übertragbar):

| Konto (`kSecAttrAccount`) | Inhalt | Eingabe | Verwendet von |
|---|---|---|---|
| `carplay-api-bearer-token` | Bearer-Token für `carplay-api` | Einstellungen → „carplay-api Token“ | `HomeserverAPIClient` (Header `Authorization: Bearer …`) |
| `vereinsheim-wol-agent-bearer-token` | Token des WoL-Agents | Einstellungen → „Vereinsheim-WoL-Agent-Token“ | `RemoteWolAgentClient` |
| `tankerkoenig-api-key` | Tankerkönig-API-Key | Einstellungen → „Tankerkönig API-Key“ | `TankerkoenigAPIClient` |

Das Einstellungs-Sheet (Zahnrad) öffnen nur die Tabs **Steuerung** und **Tankstellen**. Jeder Bereich hat *In Keychain speichern* (aktiv erst bei nicht leerem Feld) und *Gespeichertes … entfernen*. Nach dem Speichern leert die App das Eingabefeld. Beim Öffnen steht „Es ist bereits ein Token/Key gespeichert.“, wenn ein Wert existiert. Der gespeicherte Wert wird nie angezeigt.

---

## Transport

Die App hat keine App-Transport-Security-Ausnahme: Jede Anfrage muss HTTPS sein. Traefik leitet HTTP global auf HTTPS um und liefert `*.tech.homeserver` und `*.prod.homeserver` mit Zertifikaten einer privaten Root-CA aus ([Internes TLS](../capulus-core/d-sicherheit/d0040-internal-tls.md)).

Für alle Hosts mit Endung `.homeserver` und für `100.123.214.4` validiert `MTLSDelegate` das Serverzertifikat **nur** gegen die mitgelieferte Root-CA `homeserver-root-ca.pem` (`SecTrustSetAnchorCertificates` plus `SecTrustSetAnchorCertificatesOnly`), nicht gegen den Trust Store des Geräts. Ein Zertifikat einer anderen CA wird abgelehnt, auch wenn der Nutzer sie auf dem iPhone installiert hat. Ein Konfigurationsprofil mit der Root-CA auf dem Gerät ist für diese App nicht mehr nötig. Fehlt die Datei im Bundle, schlagen alle gepinnten Anfragen fehl (fail closed). Alle anderen Hosts (Wetter, News, Tankpreise) nutzen den Standard.

Die Root-CA läuft 2036-08-10 ab. Bei einem Wechsel muss `Resources/homeserver-root-ca.pem` ersetzt und die App neu gebaut werden. Der CI-Job `ca-checksum` vergleicht die Datei mit `pki/homeserver-root-ca.pem` in capulus-core und schlägt bei Abweichung fehl. Details: [Verbindung und Sicherheit](verbindung-und-sicherheit.md).

---

## Projekteinstellungen (`project.yml`)

| Schlüssel | Wert | Wirkung |
|---|---|---|
| `deploymentTarget.iOS` | `15.0` | Mindestversion |
| `PRODUCT_BUNDLE_IDENTIFIER` | `com.yourname.homeserver-dashboard` | Platzhalter, anpassen |
| `MARKETING_VERSION`, `CURRENT_PROJECT_VERSION` | `1.0.0`, `1` | wird getrennt von den Repo-Releases gepflegt |
| `TARGETED_DEVICE_FAMILY` | `1,2` | iPhone und iPad |
| `UIRequiresFullScreen` | `true` | kein Split View auf dem iPad |
| `UISupportedInterfaceOrientations` | iPhone: Hochformat. iPad: alle vier | |
| `NSLocationWhenInUseUsageDescription` | Text zur Tankstellen-Suche | Standortabfrage „beim Verwenden“ |
| `NSLocationTemporaryUsageDescriptionDictionary.TankstellenSuche` | Text für die temporär genaue Standortfreigabe | wird von `LocationManager` angefragt |
| `SWIFT_VERSION` | `5.0` | kein Swift-6-Strict-Concurrency |
| `scheme.testTargets` | leer | es gibt keine Tests |

Die Datei `HomeserverDashboard.entitlements` ist leer. Es gibt keine Entitlements, kein Keychain-Sharing und keine Network-Extension.

---

## Debugging und Fehlersuche

| Symptom | Ursache | Lösung |
|---|---|---|
| Änderung an `Constants.swift` wirkt nicht | alter Build auf dem Gerät | neu bauen und installieren |
| Alle Anfragen an `*.homeserver` schlagen fehl, Safari lädt die Seite | Root-CA fehlt im Bundle oder ist abgelaufen/ersetzt | `Resources/homeserver-root-ca.pem` prüfen (`openssl x509 -in … -noout -dates`), `project.yml` bindet sie als Ressource ein |
| `ca-checksum` in der CI schlägt fehl | Kopie weicht von capulus-core ab | Datei aus `pki/homeserver-root-ca.pem` neu kopieren |
| Token fehlt nach Neuinstallation oder Identifier-Wechsel | TODO: prüfen, ob iOS die Keychain-Einträge behält | Token neu eingeben, siehe [Installation](installation.md) |
| Zahnrad fehlt | nur auf Steuerung und Tankstellen vorhanden | zu einem dieser Tabs wechseln (im Alltag-Modus: Tankstellen) |
| „Speichern fehlgeschlagen: …“ | Keychain-Fehler (`KeychainError.unhandled(status)`) | Gerät entsperren, Fehlercode (`OSStatus`) nachschlagen |

Mehr: [Debugging und Fehlersuche](debugging.md).
