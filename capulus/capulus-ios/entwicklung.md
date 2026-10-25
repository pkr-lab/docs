# Entwicklung

Bauen, CI und Erweiterung der iOS-App. Für den Betrieb siehe [Debugging und Fehlersuche](debugging.md).

Code: [pkr-lab/capulus-ios](https://github.com/pkr-lab/capulus-ios)

---

## Lokal bauen

```bash
brew install xcodegen
xcodegen generate                       # erzeugt HomeserverDashboard.xcodeproj und Info.plist
open HomeserverDashboard.xcodeproj      # ⌘R
# oder ohne Xcode-Oberfläche:
xcodebuild build -project HomeserverDashboard.xcodeproj -target HomeserverDashboard CODE_SIGNING_ALLOWED=NO
```

Quelle der Wahrheit ist `project.yml`. Das `.xcodeproj` und `HomeserverDashboard/Info.plist` stehen in `.gitignore` und werden bei jedem `xcodegen generate` überschrieben. Neue Quelldateien unter `HomeserverDashboard/` nimmt XcodeGen automatisch auf (`sources: - path: HomeserverDashboard`), `Info.plist` und die Entitlements sind ausgenommen. Die Root-CA ist als Ressource eingetragen (`Resources/homeserver-root-ca.pem`).

Ein Commit-Hook prüft Conventional Commits (`.githooks/commit-msg`, aktivieren mit `git config core.hooksPath .githooks`).

---

## Architektur

- Reines SwiftUI, keine Drittbibliotheken, kein Swift Package.
- **MVVM light:** Ansichten beobachten `DashboardViewModel.shared` und `PowerViewModel.shared` (Singleton, `@MainActor`, `ObservableObject`). Ansichten des Alltag-Modus halten ihren Zustand selbst in `@State` und rufen die Clients direkt.
- **Clients** sind einfache Klassen mit `URLSession` und `async/await`. `HomeserverAPIClient` und `RemoteWolAgentClient` nutzen `MTLSDelegate` und verpacken `dataTask` in `withCheckedThrowingContinuation`.
- **Modelle** sind `Codable` und decodieren tolerant (Standardwerte für fehlende Listen, Rückfallwerte für unbekannte Enum-Strings).
- Adressen und feste Werte stehen zentral in `Utilities/Constants.swift`.

### Neuen Tab oder neue Datenquelle ergänzen

1. Modell in `Models/` (Codable, tolerant).
2. Client in `Services/` (für `*.homeserver`-Hosts mit `MTLSDelegate`, sonst Standard-Session).
3. Ansicht in `Views/` mit `ScreenBackground`, `SectionCard` aus `Theme.swift`, Fehlertext als rote Zeile.
4. Tab in `RootTabView.swift` dem passenden Modus zuordnen.
5. Neue Hosts oder Schlüssel in `Constants.swift`. Neue Geheimnisse als eigenes Konto in `Constants.Keychain` und in `KeychainService` und `SettingsView`.
6. Standort- oder andere Berechtigungen in `project.yml` unter `info.properties`.
7. Seite hier ergänzen, mit Fehlertabelle.

---

## Tests

Es gibt keine Tests. In `project.yml` ist `scheme.testTargets` leer. Am meisten würde ein Test-Target mit Codable-Tests der Modelle gegen echte Antworten von `carplay-api`, Open-Meteo, Pegelonline und Tankerkönig bringen (Pflichtfelder, tolerante Fallbacks). Der Build in der CI (siehe unten) ist die einzige automatische Prüfung der Übersetzbarkeit.

---

## CI

Workflows unter `.github/workflows/`:

| Workflow | Auslöser | Prüft |
|---|---|---|
| `ci.yml` | PR und Push auf `main` | Job **build** (macOS): `brew install xcodegen`, `xcodegen generate`, `xcodebuild build … CODE_SIGNING_ALLOWED=NO`. Job **ca-checksum**: `Resources/homeserver-root-ca.pem` ist byte-identisch mit `pki/homeserver-root-ca.pem` in capulus-core (`main`). Job **gitleaks**: Secret-Scan über die ganze Historie (`.gitleaks.toml`) |
| `codeql.yml` | PR, Push, wöchentlich Montag 03:41 UTC | CodeQL für `swift` (Autobuild nach `xcodegen generate`) und `actions` |
| `commitlint.yml` | PR | Conventional Commits, [wiederverwendbarer Workflow](../../github-workflows/index.md) |
| `release.yml` | Push auf `main` | semantic-release über den zentralen Workflow, siehe [Releases](releases.md) |

Renovate (`renovate.json`) läuft montags, pinnt GitHub-Actions auf Digests und wartet drei Tage nach Veröffentlichung (`minimumReleaseAge`). Die Rolle von Renovate im Gesamtsystem: [Renovate](../capulus-core/f-cicd-automatisierung/f0020-renovate.md), CodeQL und Dependabot: [CodeQL und Dependabot](../capulus-core/d-sicherheit/d0080-codeql-dependabot.md).

### Fehlerbilder in der CI

| Symptom | Ursache | Lösung |
|---|---|---|
| `ca-checksum` rot | Root-CA in capulus-core geändert oder Kopie veraltet | Datei kopieren: `cp <capulus-core>/pki/homeserver-root-ca.pem Resources/` |
| `build` rot, `xcodegen` meldet Konfigurationsfehler | Syntaxfehler in `project.yml` | lokal `xcodegen generate` ausführen |
| `build` rot mit Swift-Compilerfehler | Code | lokal mit `xcodebuild` nachstellen |
| `gitleaks` rot | Geheimnis im Diff oder in der Historie | entfernen und ggf. Token rotieren ([Secrets-Rotation](../capulus-core/d-sicherheit/d0060-secrets-rotation.md)); echte Platzhalter in `.gitleaks.toml` freigeben |
| `commitlint` rot | Commit-Nachricht nicht im Format | Nachricht ändern (`<typ>(<scope>): <beschreibung>`) |

---

## Release

Die Marketing-Version in `project.yml` (`MARKETING_VERSION`) wird getrennt von den Repo-Releases gepflegt. Eine signierte `.ipa` entsteht nicht in der CI. Der Build für TestFlight läuft lokal in Xcode. Siehe [Releases](releases.md).
