# Installation

Code: [project.yml](https://github.com/pkr-lab/capulus-ios/blob/main/project.yml), [README.md](https://github.com/pkr-lab/capulus-ios/blob/main/README.md)

---

## Voraussetzungen

| Voraussetzung | Wofür | Prüfen |
|---|---|---|
| macOS mit Xcode | Bauen, Signieren | `xcodebuild -version` |
| [XcodeGen](https://github.com/yonaskolb/XcodeGen) | erzeugt `.xcodeproj` und `Info.plist` | `brew install xcodegen`, `xcodegen --version` |
| iOS 15.0 oder neuer | Deployment-Target (`project.yml`) | Gerät oder Simulator |
| Apple-Developer-Team | Installation auf einem echten Gerät | Xcode → Signing & Capabilities |
| laufende `carplay-api` im TECH-Cluster | Daten für den PKR-Lab-Modus | `curl --cacert pki/homeserver-root-ca.pem https://carplay-api.prod.homeserver/health` |
| Tailscale auf dem Gerät oder LAN | Erreichbarkeit der `.homeserver`-Hosts | siehe [Tailscale](../capulus-core/c-netzwerk-dns/c0010-tailscale.md) |
| Tankerkönig-API-Key (optional) | Tankstellen im Alltag-Modus | kostenlos auf creativecommons.tankerkoenig.de |

Der Alltag-Modus braucht weder `carplay-api` noch Tailscale, nur Internet und (für Tankstellen) den API-Key.

---

## Schritt für Schritt

### 1. Xcode-Projekt erzeugen

```bash
git clone https://github.com/pkr-lab/capulus-ios.git
cd capulus-ios
brew install xcodegen
xcodegen generate
open HomeserverDashboard.xcodeproj
```

In Xcode unter *Signing & Capabilities* das eigene Team wählen. Passt der Platzhalter `com.yourname.homeserver-dashboard` aus `project.yml` (`PRODUCT_BUNDLE_IDENTIFIER`) nicht, ändert man ihn **in `project.yml`**, nicht im Xcode-Projekt: Jeder `xcodegen generate` überschreibt das Projekt.

Alles, was die App-Bundle-Datei braucht (Anzeigename, Launch-Screen, Ausrichtung, Standort-Beschreibungstexte), steht in `project.yml` unter `targets.HomeserverDashboard.info.properties`. XcodeGen überschreibt `Info.plist` bei jedem Lauf aus diesen Werten und mischt nichts zusammen. Deshalb ist `Info.plist` per `.gitignore` ausgeschlossen, genau wie das `.xcodeproj`.

Der Keychain-Dienstname ist unabhängig vom Bundle-Identifier fest in `Constants.Keychain.service` eingetragen (`com.yourname.homeserver-dashboard`). TODO: prüfen, ob gespeicherte Tokens nach einem Wechsel von Bundle-Identifier oder Team (andere Keychain-Zugriffsgruppe) lesbar bleiben. Im Zweifel Tokens neu eingeben.

### 2. API-Token hinterlegen (PKR-Lab-Modus)

`carplay-api` muss mit einem versiegelten `CARPLAY_API_TOKEN` im TECH-Cluster laufen, siehe [carplay-api: Ersteinrichtung](../capulus-core/3-apps-workloads/300d0-carplay-api.md#ersteinrichtung). In der App das Zahnrad öffnen (auf den Tabs **Steuerung** oder **Tankstellen**), das Token einfügen und *In Keychain speichern* tippen. Der Wert ist derselbe Klartext, aus dem das SealedSecret erzeugt wurde.

Auf den Tabs Übersicht, Alerts und im Alltag-Dashboard gibt es kein Zahnrad. Wer die App neu installiert, wechselt für die Eingabe zum Tab Steuerung.

### 3. Token für den Vereinsheim-WoL-Agent (optional)

Für den Knopf „Windows-PC (Vereinsheim)“ auf dem Tab Steuerung braucht die App ein eigenes Token vom Banana Pi. Auslesen und Hintergrund: [Vereinsheim-Alarmmonitor: iOS-App](../capulus-ansible/3-apps-workloads/30020-vereinsheim-alarmmonitor.md#ios-app). Eingabe im selben Einstellungs-Dialog im Feld „Vereinsheim-WoL-Agent-Token“.

### 4. Tankerkönig-API-Key (optional)

Im Einstellungs-Dialog im Feld „Tankerkönig API-Key“ eintragen, siehe [Alltag-Modus](alltag-modus.md#tankstellen).

### 5. Starten

⌘R in Xcode, auf einem Simulator oder einem Gerät. Für den Homeserver-Zugriff muss das Gerät im selben Netz oder im Tailnet sein. Der Simulator nutzt das Netz des Macs: Dort muss der Mac die `.homeserver`-Namen auflösen können ([DNS](../capulus-core/c-netzwerk-dns/c0000-dns-architecture.md)).

---

## Prüfen

1. Modus **PKR-Lab**, Tab Übersicht: Nach wenigen Sekunden erscheinen Host-Karten und Dienststatus. Unten steht „Aktualisiert …“.
2. Erscheint oben der gelbe Hinweis „Homeserver nicht erreichbar — zeige letzten bekannten Stand.“, siehe [Debugging und Fehlersuche](debugging.md).
3. Modus **Alltag**, Tab Dashboard: Wetter und Pegel laden ohne weitere Einrichtung.

---

## Häufige Probleme bei der Installation

| Problem | Ursache | Lösung |
|---|---|---|
| `xcodegen: command not found` | XcodeGen fehlt | `brew install xcodegen` |
| Xcode meldet fehlendes `Info.plist` | `xcodegen generate` nicht ausgeführt | `xcodegen generate` ausführen |
| Handänderungen in Xcode sind weg | `xcodegen generate` überschreibt das Projekt | Änderungen in `project.yml` machen |
| `Signing for "HomeserverDashboard" requires a development team` | kein Team gewählt | Team in Signing & Capabilities setzen |
| Bundle-ID bereits vergeben | Platzhalter `com.yourname…` | eigene ID in `project.yml` |
| Archiv wird von Xcode abgelehnt | App-Icon-Datei fehlt oder ist zu klein | `Assets.xcassets/AppIcon.appiconset` prüfen (1024×1024) |
| App startet, aber Daten fehlen | Token fehlt oder Homeserver nicht erreichbar | Abschnitt „Prüfen“ und [Debugging](debugging.md) |

Allgemeines zur Fehlersuche: [Debugging und Fehlersuche](debugging.md).
