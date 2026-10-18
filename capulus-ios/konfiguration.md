# Konfiguration

| Setting | Where | Default |
|---|---|---|
| API base URL | `Utilities/Constants.swift` → `apiBaseURL` | `https://carplay-api.prod.homeserver` |
| WoL agent base URL (vereinsheim-alarmmonitor) | `Constants.swift` → `wolAgentBaseURL` | `https://100.123.214.4:9102` (Tailscale IP) |
| Refresh interval | `Constants.swift` → `refreshInterval` | 30s |
| Bearer token | Settings screen (gear icon) → iOS Keychain | none until set |
| WoL agent bearer token | Settings screen (gear icon) → iOS Keychain | none until set |

Reachability requires Tailscale active on the device (or being on the LAN)
— see `Services/TailscaleConnectivity.swift` for what this app can and
can't actually detect about that (iOS gives no app visibility into another
app's VPN state; it watches the general network path plus whether the last
real request succeeded).

## Transport: HTTPS to carplay-api

`Constants.swift` points at `https://carplay-api.prod.homeserver`. The backend
runs in the **TECH** cluster (`argocd/apps/tech/carplay-api/`), the `prod` in
the hostname is only the URL tier (see
`capulus-core/c-netzwerk-dns/c0040-domain-tiers.md`). Traefik redirects HTTP → HTTPS
globally (`argocd/apps/tech/traefik-config/helmchartconfig.yaml`) and serves
`*.tech.homeserver` / `*.prod.homeserver` with certificates from a private root
CA (`capulus-core/d-sicherheit/d0040-internal-tls.md`,
`Resources/homeserver-root-ca.pem` (Kopie von `pki/homeserver-root-ca.pem` in capulus-core)).

The app has **no App Transport Security exception**: every request to a
hostname must be HTTPS with a certificate iOS trusts. That requires the root CA
to be installed *and* enabled on the device (Settings → General → VPN & Device
Management for the profile, then Settings → General → About → Certificate Trust
Settings). If the app shows a connection error, check this first: open
`https://carplay-api.prod.homeserver/health` in Safari, which must load without
a certificate warning.

The WoL agent on the Pi is HTTPS as well (`https://100.123.214.4:9102`). Its
certificate is issued by the same root CA and carries the Tailscale IP as a
subject alternative name, so no additional trust setup is needed on the device.
There is no plain-HTTP URL left in the app.

**Root-CA pinning.** `MTLSDelegate` (used by the carplay-api and WoL-agent sessions) validates the server certificate of every host ending in `.homeserver` and of `100.123.214.4` against the bundled root CA only (`SecTrustSetAnchorCertificates` plus `SecTrustSetAnchorCertificatesOnly`), not against the device trust store. A certificate from any other CA, even one the user installed on the phone, is rejected; the installed profile is no longer needed for this app. The CA file is `Resources/homeserver-root-ca.pem` (Kopie von `pki/homeserver-root-ca.pem` in capulus-core), `project.yml` adds it to the bundle as a resource (`homeserver-root-ca.pem`). If it is missing from the bundle, all pinned requests fail (fail closed). All other hosts (weather, news, fuel prices) use the system default. When the root CA is replaced (expires 2036-08-10), rebuild the app with the new file.
