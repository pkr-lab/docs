# carplay-api

Backend for the **Homeserver Dashboard** iOS app (a pure iPhone app — no
CarPlay component; this directory/namespace kept its name from an earlier
CarPlay-oriented cut of the project). Combines VictoriaMetrics (per-host
metrics), ntfy (alerts) and Uptime-Kuma (service status) into one payload,
cached for 30s, served over Gin — plus brightness and Wake-on-LAN/shutdown
endpoints proxied to power-agent (`capulus-ansible/roles/power_agent`) on the
homeserver host.

Full setup/operations guide: [`capulus-core/3-apps-workloads/300d0-carplay-api.md`](../3-apps-workloads/300d0-carplay-api.md).

## Deviations from the original spec

A few things in the original spec don't match how this cluster (or the
upstream services) actually work. Rather than ship something that 404s or
lies about its security model, these were adjusted — details in
capulus-core/3-apps-workloads/300d0-carplay-api.md:

- **Uptime-Kuma**: no Bearer-token `/api/status_page/monitors` REST API
  exists in real Uptime-Kuma. The client uses Kuma's actual (public,
  unauthenticated) status-page JSON API instead — requires a status page
  configured in the Kuma UI.
- **K8s packaging**: this is a Helm chart (`Chart.yaml` + `values.yaml` +
  `templates/`), like every other app in `argocd/apps/`, not raw
  kustomize/kubectl YAML. Secrets are `SealedSecret`s, not plaintext.
- **Namespace**: `carplay-api`, not `homeserver-app` — the root
  `ApplicationSet` derives the namespace from the directory name.
- **mTLS**: not implemented. Traefik in this cluster doesn't terminate
  client-certificate TLS, so a "cert pinning" check here would have had
  nothing behind it. The real boundary is Bearer token + restricting ingress
  to the Tailscale network (`IP_ALLOWLIST_CIDRS`), see below.
- **CORS**: kept, but it's a browser-only mechanism and doesn't restrict the
  iOS app (URLSession ignores it) or non-browser clients. The actual network
  restriction is the IP allowlist / Tailscale-only ingress.

## Endpoints

| Method | Path | Auth | Description |
|---|---|---|---|
| GET | `/health` | none | Liveness/readiness. Always 200 if the process is up; body reports per-dependency reachability for diagnostics. |
| GET | `/metrics` | Bearer token | Prometheus text exposition: request counts/durations, cache hit/miss. |
| GET | `/api/dashboard` | Bearer token | Combined alerts + per-host metrics + status, cached 30s. |
| GET | `/api/brightness` | Bearer token | Current Homeserver screen brightness, proxied to power-agent. |
| PUT | `/api/brightness` | Bearer token | Body `{"percent": 0-100}` — sets Homeserver screen brightness. |
| POST | `/api/power/wake` | Bearer token | Body `{"target": "worker-0"\|"worker-1"}` — sends a WoL magic packet. |
| POST | `/api/power/shutdown` | Bearer token | Body `{"target": "worker-0"\|"worker-1"\|"homeserver", "code"?: "..."}` — `code` required and checked against `SHUTDOWN_CONFIRMATION_CODE` only when `target` is `"homeserver"`. |
| GET | `/api/updates` | Bearer token | Per-repo update status (name, current vs. latest GitHub release) read from the `github-release-watcher`'s `updates` ConfigMap — see [capulus-core/f-cicd-automatisierung/f0040-github-release-watcher.md](../f-cicd-automatisierung/f0040-github-release-watcher.md). |

Brightness and power endpoints are proxied to **power-agent**
(`capulus-ansible/roles/power_agent`), a small privileged daemon on the bare
Homeserver host — this pod runs unprivileged and has no host/sysfs/SSH
access of its own, see [capulus-core/3-apps-workloads/300d0-carplay-api.md](../3-apps-workloads/300d0-carplay-api.md#power-agent).

## Configuration (environment variables)

| Variable | Default | Purpose |
|---|---|---|
| `PORT` | `8080` | HTTP listen port. |
| `LOG_LEVEL` | `info` | `debug`, `info`, `warn`, `error`. |
| `CACHE_TTL` | `30` | Dashboard cache lifetime, seconds. |
| `API_TIMEOUT` | `5` | Overall budget across the 3 upstream calls, seconds. |
| `CARPLAY_API_TOKEN` | *(empty)* | Bearer token required on `/api/*` and `/metrics`. **Pflicht:** ist die Variable leer, bricht der Start mit `CARPLAY_API_TOKEN is not set` ab (fail closed, siehe [60080](../6-hintergruende/60080-carplay-api-und-ios-app.md#absicherung)). |
| `IP_ALLOWLIST_CIDRS` | *(empty)* | Comma-separated CIDRs; empty = allow all. E.g. Tailscale CGNAT range `100.64.0.0/10`. |
| `CORS_ALLOWED_ORIGINS` | *(empty)* | Comma-separated allowed Origins; empty = CORS headers never sent. |
| `RATE_LIMIT_PER_MINUTE` | `100` | Per-client-IP fixed-window limit on `/api/*`. |
| `TRUSTED_PROXIES` | *(empty = trust none)*, chart default `10.42.0.0/16` | CIDRs of proxies allowed to set `X-Forwarded-For` (needed for correct client IPs behind Traefik). |
| `VM_URL` | `http://vmsingle-monitoring-victoria-metrics-k8s-stack.monitoring.svc.cluster.local:8428` | VictoriaMetrics query API base. |
| `HOSTS` | see `values.yaml` `config.hosts` | `id\|name\|instance` triples, comma-separated — one per host card on the app's home screen. `instance` must match VictoriaMetrics' `instance` label exactly. |
| `NTFY_URL` | `http://ntfy.ntfy.svc.cluster.local` | ntfy base URL. |
| `NTFY_TOPIC` | `alerts` | Topic to poll. |
| `NTFY_TOKEN` | *(empty)* | Optional ntfy access token (unneeded while ntfy's `auth-default-access` is `read-write`). |
| `NTFY_SINCE` | `12h` | How far back to poll cached messages. |
| `NTFY_LIMIT` | `10` | Max alerts returned. |
| `UPTIME_KUMA_URL` | `http://uptime-kuma.uptime-kuma.svc.cluster.local` | Uptime-Kuma base URL. |
| `UPTIME_KUMA_SLUG` | `homeserver` | Status-page slug to read. |
| `POWER_AGENT_URL` | `http://192.168.178.94:9101` | Base URL of the power-agent daemon on the Homeserver host. The chart sets `http://10.42.2.1:9101` (the `cni0` address of the Homeserver). |
| `POWER_AGENT_TOKEN` | *(empty)* | Bearer token for power-agent. Unset = every brightness/power request fails with 502. |
| `POWER_AGENT_TIMEOUT` | `8` | Timeout for power-agent calls, seconds — poweroff/WoL can take longer than the 3s VictoriaMetrics budget. |
| `SHUTDOWN_CONFIRMATION_CODE` | *(empty)* | Required to match `code` on `POST /api/power/shutdown` with `target: "homeserver"`. Unset = homeserver shutdown always rejected with 503. |

## Local development

```bash
cd src
go run ./cmd/server
# in another shell
curl localhost:8080/health
```

## Build & push

Automated via [`.github/workflows/build-images.yml`](https://github.com/pkr-lab/capulus-core/blob/main/.github/workflows/build-images.yml)
on every push to `main` touching `Dockerfile`/`src/**` here — builds and
pushes to `ghcr.io/pkr-lab/carplay-api`. See
[pacman/README.md](pacman.md#image-bauen) for the general
mechanics (tag scheme, where the run reports the new tag, manual
`workflow_dispatch` rebuild). The workflow does not touch `values.yaml`;
set `image.tag` to the reported tag and commit that yourself (and add an
`imagePullSecrets` entry if the GHCR package is private).

## Tests

None yet — see capulus-core/3-apps-workloads/300d0-carplay-api.md for suggested coverage
(`internal/clients` PromQL/ntfy/Kuma parsing are the highest-value targets,
since they're the parts most likely to drift from upstream API changes).

## Fehlersuche

Die Anwendung schreibt je Request eine JSON-Zeile (Methode, Pfad, Status, Dauer, Trace-ID, ohne Header und Query-Strings) auf stdout. Das Level steuert `LOG_LEVEL` (`debug`, `info`, `warn`, `error`, Standard `info`). Bei `debug` läuft Gin im Debug-Modus.

```bash
kubectl -n carplay-api logs deploy/carplay-api --tail=100
curl -s https://carplay-api.prod.homeserver/health                      # Erreichbarkeit der Abhängigkeiten
curl -s -H "Authorization: Bearer <TOKEN>" https://carplay-api.prod.homeserver/api/dashboard | jq .
cd argocd/apps/tech/carplay-api/src && LOG_LEVEL=debug CARPLAY_API_TOKEN=test go run ./cmd/server   # lokal
```

| Antwort oder Meldung | Ursache | Lösung |
|---|---|---|
| Start bricht ab: `CARPLAY_API_TOKEN is not set` | Secret fehlt oder ist leer, Pod `CreateContainerConfigError` oder Neustart-Schleife | SealedSecret `carplay-api-token` prüfen ([Ersteinrichtung](../3-apps-workloads/300d0-carplay-api.md#ersteinrichtung)) |
| Start bricht ab: `setting trusted proxies: …` oder `panic: auth: invalid CIDR in allowlist: …` | ungültige CIDR in `TRUSTED_PROXIES` oder `IP_ALLOWLIST_CIDRS` | CIDR korrigieren |
| `401 {"error":"missing bearer token"}` | Header `Authorization: Bearer …` fehlt | Token setzen (in der App: Einstellungen) |
| `401 {"error":"invalid bearer token"}` | Token weicht ab | Token der App mit dem Klartext des SealedSecrets abgleichen |
| `403` | IP-Allowlist blockt die Quelle (hinter Traefik meist die Proxy-IP, wenn `TRUSTED_PROXIES` fehlt) | [Absicherung](../3-apps-workloads/300d0-carplay-api.md#absicherung) |
| `429 {"error":"rate limit exceeded"}` | mehr als `RATE_LIMIT_PER_MINUTE` Anfragen je IP | warten oder Limit erhöhen |
| `400 {"error":"percent must be an integer 0-100"}` | falscher Wert bei `PUT /api/brightness` | Wert 0 bis 100 senden |
| `400 {"error":"target is required"}` | Body ohne `target` | Ziel angeben |
| `400 {"error":"homeserver has no wake action"}` | Wake für `homeserver` angefordert | nur `worker-0` und `worker-1` lassen sich wecken |
| `503 {"error":"shutdown confirmation is not configured on the server"}` | `SHUTDOWN_CONFIRMATION_CODE` leer | Secret `carplay-api-shutdown-code` und `secrets.shutdownConfirmationCode.enabled` |
| `401 {"error":"confirmation code does not match"}` | Code stimmt nicht | Code = ArgoCD-Admin-Passwort, bei Rotation nachziehen |
| `502 {"error":"<aktion> failed: power-agent returned <Status>"}` | power-agent antwortet mit Fehler (zum Beispiel 401 bei falschem Token) | Token und `journalctl -u power-agent` auf dem Homeserver |
| `502 {"error":"<aktion> failed: power-agent unreachable"}` | power-agent nicht erreichbar (Dienst aus, falsche `POWER_AGENT_URL`, NetworkPolicy) | `systemctl status power-agent`, URL `http://10.42.2.1:9101` prüfen ([Rolle power_agent](../../capulus-ansible/rollen/power_agent.md)) |
| Log: `ntfy fetch failed, showing no alerts` | ntfy nicht erreichbar oder Token fehlt | `alerts` bleibt leer, [ntfy](../1-benachrichtigungen/10010-ntfy.md) prüfen |
| Log: `ntfy: skipping malformed line` | ntfy lieferte eine unlesbare Zeile | einzelne Zeile wird übersprungen, meist harmlos |
| Log: `uptime-kuma fetch failed, showing no statuses` | Status-Page nicht erreichbar oder Slug falsch | `UPTIME_KUMA_SLUG`, [Uptime Kuma](../3-apps-workloads/30080-uptime-kuma.md) |
| Log: `uptime-kuma: unrecognized heartbeat time format` | neues Zeitformat der Status-Page | Zeit wird durch „jetzt“ ersetzt, Code anpassen |
| Log: `victoriametrics host query failed` / `traefik query failed` | PromQL-Fehler oder VictoriaMetrics nicht erreichbar | Log enthält die Antwort von VictoriaMetrics, `VM_URL` und `HOSTS` prüfen |
| Log: `k8s in-cluster client unavailable, /api/updates will always be empty` | kein ServiceAccount-Zugriff (Lauf außerhalb des Clusters oder RBAC) | im Cluster prüfen, lokal erwartet |
| Log: `reading updates configmap failed` / `decoding updates configmap failed` | ConfigMap des Release-Watchers fehlt oder ist ungültig | [github-release-watcher](../f-cicd-automatisierung/f0040-github-release-watcher.md) |
| `/api/dashboard` zeigt alte Daten | Cache `CACHE_TTL` (30 s) | warten |
| Teilausfall: einzelne Felder leer, HTTP 200 | die Quellen werden unabhängig abgefragt, ein Ausfall liefert eine leere Liste statt eines Fehlers | Log nach `… failed` durchsuchen |

Die iOS-App zeigt keine Fehlermeldungen der Übersicht, siehe [iOS: Debugging](../../capulus-ios/debugging.md). Tests: `cd argocd/apps/tech/carplay-api/src && go build ./... && go vet ./... && go test ./...`. Allgemeine Fehlersuche: [zentrale Fehlersuche](../2-betrieb-hardware/20070-troubleshooting-zentral.md).
