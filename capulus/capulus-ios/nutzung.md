# Nutzung

Code: [Views/RootTabView.swift](https://github.com/pkr-lab/capulus-ios/blob/main/HomeserverDashboard/Views/RootTabView.swift)

---

## Aufbau der Bedienung

Oben sitzt der Schalter **PKR-Lab / Alltag**. Er wählt die Tab-Leiste und bleibt über App-Neustarts gespeichert.

| Modus | Tab | Zweck | Seite |
|---|---|---|---|
| PKR-Lab | Übersicht | Hosts, Dienststatus, Dienst-Aktivität, Updates | [Übersicht und Alerts](uebersicht-und-alerts.md) |
| PKR-Lab | Steuerung | Helligkeit, Wake/Shutdown, Vereinsheim-PC | [Steuerung](steuerung.md) |
| PKR-Lab | Alerts | aktive Alarme | [Übersicht und Alerts](uebersicht-und-alerts.md) |
| Alltag | Dashboard | Wetter, Rhein-Pegel, Dienste-Links | [Alltag-Modus](alltag-modus.md) |
| Alltag | Tankstellen | Preise, nächste Station | [Alltag-Modus](alltag-modus.md) |
| Alltag | News | Top-Meldungen | [Alltag-Modus](alltag-modus.md) |

Gesten und Bedienelemente:

- **Ziehen nach unten** auf jedem Tab lädt die Daten neu. Im Modus PKR-Lab lädt die Übersicht zusätzlich alle 30 Sekunden automatisch.
- **Tippen auf einen Monitor oder Alarm** öffnet ein Detail-Sheet, *Fertig* schließt es.
- **Zahnrad** oben rechts (nur Tabs Steuerung und Tankstellen) öffnet die Einstellungen für Tokens und API-Key, siehe [Konfiguration](konfiguration.md#keychain-einträge).
- **Gelber Hinweis** auf der Übersicht: Homeserver nicht erreichbar, es gilt der letzte bekannte Stand.
- Die App läuft nur im dunklen Farbschema. Auf dem iPhone nur im Hochformat, auf dem iPad in allen Ausrichtungen, ohne Split View.

---

## Typische Abläufe

### Zustand prüfen

Modus PKR-Lab → Übersicht. Grüne Prozentwerte und `UP` heißen unauffällig. Rote Karten oder `DOWN` verweisen auf ein Problem: Tippen auf den Monitor öffnet das Detail (Ping, Uptime, letzter Check). Die Ursachensuche steht in [Debugging](debugging.md).

### Alarme ansehen

Tab Alerts. Rote Alarme sind `critical`, orange `warning`. Alarme der letzten Stunden (Standard 12 h) bleiben sichtbar, danach verschwinden sie.

### Worker aufwecken oder ausschalten

Tab Steuerung → Karte des Workers. *Aufwecken* gilt für offline, *Ausschalten* für online. Der Status zieht nach, sobald der Host Metriken liefert. Details: [Steuerung](steuerung.md).

### Homeserver herunterfahren

Nur in Ausnahmefällen: Das nimmt Kubernetes, ArgoCD und alle Dienste mit, und das Wiedereinschalten geht nur vor Ort. Der Bestätigungscode ist derselbe wie für ArgoCD.

### Tankstellen im Alltag

Modus Alltag → Tankstellen. Beim ersten Mal fragt iOS nach der Standortfreigabe. Ohne API-Key bleibt die Seite leer, siehe [Alltag-Modus](alltag-modus.md#tankstellen).

---

## Bekannte Lücken

- **Keine Tests:** Das Schema `HomeserverDashboard` hat keine Test-Targets (`testTargets: []` in `project.yml`). Am meisten würden Codable-Tests der Modelle gegen echte `carplay-api`-Antworten bringen.
- **Fehler der Übersicht unsichtbar:** `DashboardViewModel.error` wird nie angezeigt, siehe [Übersicht und Alerts](uebersicht-und-alerts.md#randfälle-und-einschränkungen).
- **Keine Push-Benachrichtigungen:** Die App ruft nur ab. Alarme kommen per ntfy aufs Telefon ([Telefon-Alarm](../capulus-core/1-benachrichtigungen/10020-telefon-alarm.md)).
- **Feste Standorte und Stationen:** Wetter, Pegel und drei Tankstellen sind im Code festgelegt.
- **Platzhalter-Bundle-ID:** `com.yourname.homeserver-dashboard` in `project.yml`.
- **Swift-Concurrency:** gebaut mit Swift-Sprachmodus 5 (`SWIFT_VERSION: "5.0"`), nicht im Strict-Modus von Swift 6.
- **App Store:** Datenschutzerklärung und Screenshots für TestFlight-Extern oder App Store sind nicht Teil des Repos. TODO: prüfen, ob die App über TestFlight verteilt wird.

---

## Debugging und Fehlersuche

Kurzdiagnose, wenn „nichts geht“:

1. Gelber Hinweis auf der Übersicht? Dann zuerst Netzwerk, Tailscale und Token prüfen.
2. Zeigt der Alltag-Modus Daten? Dann ist die App in Ordnung, und das Problem liegt bei `carplay-api` oder dem Zugang dorthin.
3. Details und Fehlertabellen: [Debugging und Fehlersuche](debugging.md).
