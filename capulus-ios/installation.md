# Installation

## Voraussetzungen

- macOS mit Xcode und [XcodeGen](https://github.com/yonaskolb/XcodeGen) (`brew install xcodegen`)
- iOS 15.0 oder neuer als Deployment-Target
- laufende `carplay-api` im TECH-Cluster mit versiegeltem `CARPLAY_API_TOKEN`
- Tailscale auf dem Gerät oder Zugriff aus dem LAN

## Setup

### 1. Generate the Xcode project

```bash
brew install xcodegen
xcodegen generate
open HomeserverDashboard.xcodeproj
```

Set `PRODUCT_BUNDLE_IDENTIFIER` / your Team in Xcode's Signing & Capabilities
if `com.yourname.homeserver-dashboard` (`project.yml`) isn't yours.

**Every field the app bundle needs — display name, launch screen,
supported orientation — lives in
`project.yml`'s `targets.HomeserverDashboard.info.properties`, not in
`Info.plist` itself.** XcodeGen overwrites `Info.plist` from those
properties on every `generate` run rather than merging with whatever's
already at that path; editing the plist file directly gets silently
discarded the next time someone runs `xcodegen generate`. `Info.plist` is
gitignored for exactly this reason — like the `.xcodeproj`, it's build
output, not a source file.

### 2. Get an API token

Requires carplay-api already deployed (TECH cluster) with a sealed `CARPLAY_API_TOKEN` —
see [`capulus-core/3-apps-workloads/300d0-carplay-api.md`](../capulus-core/3-apps-workloads/300d0-carplay-api.md#ersteinrichtung) in
capulus-core. Run the app, tap the gear icon on any tab → paste the same
token you sealed for the backend → **Save to Keychain**.

For the "Windows-PC (Vereinsheim)" wake button on the Steuerung tab, a
second, separate token is needed — see
[`capulus-ansible/3-apps-workloads/30020-vereinsheim-alarmmonitor.md`, "iOS-App"](../capulus-ansible/3-apps-workloads/30020-vereinsheim-alarmmonitor.md#ios-app)
in capulus-core for how to read it off the Pi. Same Settings screen,
second token field.

### 3. Run

⌘R in Xcode, on a Simulator or a real device on the same network/Tailscale
tunnel as the homeserver.
