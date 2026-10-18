# Nutzung

## Die drei Tabs

1. **Übersicht (Home)** — alerts, then a glass card per monitored host
   (Homeserver, NAS, worker-0, worker-1) that's currently **on**; an
   offline host is omitted rather than shown at a misleading 0%. Each
   online host's card shows CPU/RAM/Disk (and temperature, if that host
   reports a sensor) plus uptime. Below that, Uptime-Kuma service status.
2. **Helligkeit (Brightness)** — a single slider controlling the
   Homeserver's own screen brightness (it's a laptop, an HP ProBook 450
   G9 — see `capulus-ansible/a-betriebssystem/a0010-overview.md`), proxied through carplay-api to
   power-agent on the host (`capulus-core/3-apps-workloads/300d0-carplay-api.md#power-agent`), since
   the carplay-api pod itself has no sysfs access.
3. **Steuerung (Power)** — Wake-on-LAN + shutdown buttons for worker-0 and
   worker-1 (each gated behind a plain "are you sure?" confirmation), and
   shutdown-only for the Homeserver (no WoL path — it's the cluster's only
   always-on node). Homeserver shutdown additionally requires typing the
   confirmation code — **the same one used to log into ArgoCD** — checked
   server-side against `SHUTDOWN_CONFIRMATION_CODE`
   (`capulus-core/3-apps-workloads/300d0-carplay-api.md`). Wrong or missing code = request rejected,
   nothing happens. A fourth card wakes the Windows PC at the
   vereinsheim-alarmmonitor site — no online status (that site has no
   dashboard entry) and no confirmation, since it's WoL-only. This one
   bypasses carplay-api entirely and talks straight to a small agent
   running on the Pi itself, over Tailscale — see
   `capulus-core/4-planung/40020-vereinsheim-wol-router-vpn.md` in capulus-core
   for why.

## Deviations from the original file list / spec

- **`MTLSDelegate.swift`**: kept as the requested filename, but implements
  standard TLS handling + a documented extension point for a *future* client
  certificate, not active pinning — see "Transport" above.
- **Entitlements**: `HomeserverDashboard.entitlements` is empty. The
  original spec's "Keychain Sharing" and "Network Extensions" entries never
  applied (single target, rides on Tailscale's already-active system VPN),
  and CarPlay's entitlement is gone entirely along with the CarPlay code.
- **`.xcodeproj` / `Info.plist` not committed**: both are generated from
  `project.yml` (via XcodeGen) instead of hand-maintained, for the same
  reason this repo's k8s apps are Helm charts, not hand-edited YAML — see
  "Setup" above for why `Info.plist` specifically has to work this way.

## Known gaps (not implemented)

- **Unit tests**: none. `Models/` (Codable round-trips against real
  carplay-api responses) is the highest-value target if you add them.
- **App icon**: `Assets.xcassets/AppIcon.appiconset` has a valid 1024×1024
  slot defined but no actual image — add one before archiving for
  TestFlight, Xcode will otherwise reject the archive.
- **Privacy policy / App Store screenshots**: needed for any TestFlight
  *external* testing group or App Store submission, not for internal
  testing or the Simulator. Not produced here — no product copy or Apple
  Developer account access from this environment.
- **Swift Concurrency**: written under default ("minimal") concurrency
  checking, not Swift 6 strict mode.
