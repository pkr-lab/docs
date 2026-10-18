# Konfiguration

Wo was konfiguriert wird, aufgeteilt nach Repo. Die Hostkonfiguration (Inventory, Variablen, Rollen) liegt in [capulus-ansible](../capulus-ansible/konfiguration.md), die Cluster-Konfiguration hier.

## Konfigurationsdateien in capulus-core

| Datei / Ordner | Zweck |
|---|---|
| `argocd/apps/<cluster>/<app>/values.yaml` | Werte je App; Image-Tags unter `image.tag` (selbst gebaute Images: `sha-<commit>`) |
| `argocd/apps/<cluster>/<app>/Chart.yaml` | Helm-Chart je App, Abhängigkeiten als vendorte `charts/*.tgz` |
| `argocd/bootstrap/` | Hub-ApplicationSet und AppProjects, **generiert** aus capulus-ansible |
| `argocd/bootstrap-prod/` | PROD-AppProject und zwei ApplicationSets (handgeschrieben) |
| `argocd/promotion.yaml` | Gates und Smoke-Checks der Promotion-Kette |
| `ops/token-register.yaml` | Register aller Tokens und Passwörter |
| `pki/homeserver-root-ca.pem` | Root-CA des Homelabs, Quelle der Wahrheit |
| `renovate.json` | Renovate: Zeitplan, Gruppierung, Digest-Pinning, ENTW-Regeln |
| `.github/workflows/release.yml` | Aufrufer des zentralen Release-Workflows aus `pkr-lab/.github` (Branch `main`, Tag `vX.Y.Z`, CHANGELOG.md) |
| `.github/workflows/commitlint.yml`, `.githooks/commit-msg` | Prüfung der Commit-Nachrichten auf PRs und lokal (`make hooks`) |
| `.yamllint`, `.gitleaks.toml`, `.gitleaksignore` | Lint- und Secret-Scan-Konfiguration |
| `.github/CODEOWNERS`, `.github/dependabot.yml` | Zuständigkeiten, Sicherheits-PRs für Go-Module und Actions |

## Promotion (`argocd/promotion.yaml`)

| Schlüssel | Bedeutung | Wert |
|---|---|---|
| `gates.entw_hours` | Stunden, die eine Version auf ENTW gesund laufen muss, bevor sie nach TECH geht | `24` |
| `gates.tech_hours` | Stunden auf TECH, bevor sie nach PROD geht | `2` |
| `exclude` | Apps, die nicht promoviert werden | `[]` |
| `smoke.<app>.entw[]` | HTTP-Smoke-Checks (`url`, `contains`) auf ENTW | `example-whoami`, `demo-app` |

Ablauf und Voraussetzungen: [f00b0 Promotion-Kette](f-cicd-automatisierung/f00b0-promotion-chain.md).

## Token-Register (`ops/token-register.yaml`)

Jeder Eintrag beschreibt ein Secret:

| Feld | Bedeutung |
|---|---|
| `id`, `service`, `kind` | Kennung, zugehöriger Dienst, Art (`password`, `api-token`, …) |
| `stored_in` | Dateien, in denen der Wert liegt (SealedSecrets; Dateien aus capulus-ansible mit Präfix `capulus-ansible:`) |
| `issued`, `expires`, `max_age_days` | Ausgabedatum, Ablauf, höchstes Alter |
| `rotation.mode`, `rotation.runbook`, `rotation.overlap` | Rotationsart, Anleitung, ob alter und neuer Wert zeitweise parallel gelten |
| `owner`, `blast_radius` | Verantwortlicher und Auswirkung bei Kompromittierung |

`python3 scripts/check-token-register.py` prüft das Register in der CI. Jedes SealedSecret im Repo braucht einen Eintrag.

## Secrets

- Cluster-Werte: Sealed Secrets, je Cluster ein eigener Schlüssel. Versiegelt wird mit `printf '%s' "$WERT" | kubeseal --raw …`, nicht mit `echo` oder Here-String, weil ein mitversiegeltes Zeilenende schon Ausfälle verursacht hat.
- Host-Werte: Ansible Vault in capulus-ansible.
- Niemals Klartext-Secrets committen. gitleaks läuft in der CI.

## Service-URLs

Jeder Hostname trägt ein Tier-Label (`tech` = Infrastruktur/Admin, `prod` = Apps mit echtem Nutzerkreis,
`dev` = ENTW) — Details und Begründung:
**[capulus-core/c-netzwerk-dns/c0040-domain-tiers.md](c-netzwerk-dns/c0040-domain-tiers.md)**. Die Spalte
*Cluster* zeigt, wo die App tatsächlich läuft; das Tier im Namen ist nur die URL-Konvention.

| Service | Tier | Cluster | URL |
|---|---|---|---|
| Grafana | tech | TECH | https://grafana.tech.homeserver |
| ArgoCD-Hub | – | TECH | https://argocd.tech.homeserver |
| ArgoCD ENTW | – | ENTW | https://argocd.dev.homeserver |
| Headlamp | tech | TECH | https://headlamp.tech.homeserver |
| Semaphore | tech | TECH | https://semaphore.tech.homeserver |
| lldap (Nutzerverwaltung) | tech | TECH | https://lldap.tech.homeserver |
| ntfy | tech | TECH | https://ntfy.tech.homeserver |
| Pi-hole | tech | TECH | https://pihole.tech.homeserver |
| Argo Workflows | tech | TECH | https://argo-workflows.tech.homeserver |
| MinIO Console | tech | TECH | https://minio.tech.homeserver |
| kubeseal-webgui | tech | TECH | https://kubeseal-webgui.tech.homeserver |
| Vaultwarden | tech (Ausnahme) | TECH | https://vault.tech.homeserver |
| Zammad | tech (Ausnahme) | TECH | https://zammad.tech.homeserver |
| Alarmmonitor (alamos-apager) | prod | TECH | https://alamos-apager.prod.homeserver |
| MediaMTX (Live-Stream-Playback) | prod | TECH | https://stream.prod.homeserver |
| n8n | prod | TECH | https://n8n.prod.homeserver |
| Uptime Kuma | prod | TECH | https://uptime-kuma.prod.homeserver |
| Homeserver-Dashboard-API (carplay-api) | prod | TECH | https://carplay-api.prod.homeserver |
| pacman (Schulungsobjekt) | prod | TECH | https://pacman.prod.homeserver |
| Nextcloud | prod | PROD | https://nextcloud.prod.homeserver |
| Immich | prod | PROD | https://immich.prod.homeserver |
| Paperless-ngx | prod | PROD | https://paperless.prod.homeserver |
| Wiki.js | prod | PROD | https://wiki.prod.homeserver |
| Xibo CMS (xibosignage) | prod | PROD | https://xibo.prod.homeserver |
| Demo-Apps | dev | ENTW | http://whoami.dev.homeserver, http://demo.dev.homeserver |

> Zusätzlich zu den internen `*.homeserver`-URLs können ausgewählte Dienste
> über Cloudflare Tunnel öffentlich unter einer eigenen Domain erreichbar
> gemacht werden (z. B. `https://wiki-prod.deine-domain.de` — Bindestrich
> statt Punkt vor dem Tier, siehe
> [capulus-core/c-netzwerk-dns/c0040-domain-tiers.md](c-netzwerk-dns/c0040-domain-tiers.md#warum-punkt-intern-bindestrich-extern))
> — ohne VPN, ohne offene Ports. TECH und PROD haben je einen eigenen Tunnel. Setup:
> **[capulus-core/e-externe-erreichbarkeit/e0000-cloudflare-tunnel.md](e-externe-erreichbarkeit/e0000-cloudflare-tunnel.md)**.
> Welche Dienste aktuell extern erreichbar sind (u. a. Nextcloud, Immich, Wiki.js, Xibo,
> Vaultwarden, Grafana, ntfy, Status-Seite), steht in der Tabelle unter
> [Externe Erreichbarkeit](c-netzwerk-dns/c0040-domain-tiers.md#externe-erreichbarkeit-wildcard-routing-über-traefik) —
> bei Immich nötig für Handy-Auto-Backup unterwegs.
> Der Live-Stream (`https://stream-prod.pke-lab.de`) ist per mediamtx-eigenem HTTP-Basic-Login
> abgesichert — kein externer Identity-Provider, kein Cloudflare-Zero-Trust-Konto nötig. Details:
> **[capulus-core/3-apps-workloads/30040-mediamtx.md](3-apps-workloads/30040-mediamtx.md)**.

---

## Netzwerk und Sicherheit

<table>
<thead><tr><th>Prinzip</th><th>Umsetzung</th></tr></thead>
<tbody>
<tr><td>Keine öffentlichen Ports</td><td>Zugriff ausschließlich über LAN, Tailscale-VPN oder gezielt per Cloudflare Tunnel (ausgehende Verbindung, kein Port-Forwarding)</td></tr>
<tr><td>Getrennte Cluster</td><td>TECH, PROD und ENTW haben eigene Pod-/Service-Netze und eigene SealedSecrets-Schlüssel; PROD und ENTW laufen als KVM-VMs, ENTW hat keine Route zu TECH/PROD und keinen ArgoCD-Zugriff vom Hub, siehe <a href="4-planung/40080-multi-cluster-entw-prod-tech.md">40080</a></td></tr>
<tr><td>UFW-Firewall</td><td>Erlaubt nur SSH, HTTP/HTTPS, k3s-API, ArgoCD-NodePort (HTTPS-only), Flannel, Tailscale-UDP</td></tr>
<tr><td>Opt-in externe Erreichbarkeit</td><td>Nur Dienste mit einem eigenen <code>*-pke-lab.de</code>-Host in ihrer <code>ingress.hosts</code>-Liste sind öffentlich erreichbar, alles andere bleibt intern</td></tr>
<tr><td>Brute-Force-Schutz</td><td>CrowdSec beobachtet SSH- und Traefik-Logs und lässt einen Firewall-Bouncer auffällige IPs sperren, siehe <a href="../capulus-ansible/d-sicherheit/d0020-crowdsec.md">capulus-ansible/d-sicherheit/d0020-crowdsec.md</a></td></tr>
<tr><td>NetworkPolicies</td><td>Default-Deny je Namespace im TECH-Cluster mit expliziten Ausnahmen, siehe <a href="d-sicherheit/d0030-network-policies.md">capulus-core/d-sicherheit/d0030-network-policies.md</a></td></tr>
<tr><td>Internes TLS</td><td>cert-manager mit eigener Root-CA (PROD: eigene Intermediate-CA), HTTP wird auf HTTPS umgeleitet, siehe <a href="d-sicherheit/d0040-internal-tls.md">capulus-core/d-sicherheit/d0040-internal-tls.md</a></td></tr>
<tr><td>Secrets</td><td>Ansible-Vault für Host-Werte, Sealed Secrets für Cluster-Werte (je Cluster eigener Schlüssel), k3s-Secrets-Encryption at rest + Audit-Log, Rotations-Checkliste in <a href="d-sicherheit/d0060-secrets-rotation.md">d0060</a></td></tr>
<tr><td>ArgoCD Read-only</td><td>Hat ausschließlich Read-Access auf das Git-Repo</td></tr>
<tr><td>ArgoCD-AppProjects</td><td>Je Cluster ein Project (<code>tech</code> / <code>prod</code> / <code>entw</code>) begrenzt Quell-Repo und erlaubte Ziel-Namespaces, siehe <a href="b-kubernetes-gitops/b0020-argocd-projects.md">capulus-core/b-kubernetes-gitops/b0020-argocd-projects.md</a></td></tr>
<tr><td>Code-Scanning &amp; Abhängigkeiten</td><td>CodeQL prüft jeden PR (Go, JS, Python, Swift, Workflows), Dependabot öffnet Fix-PRs für verwundbare Abhängigkeiten, offene Funde werden täglich zu Issues, siehe <a href="d-sicherheit/d0080-codeql-dependabot.md">capulus-core/d-sicherheit/d0080-codeql-dependabot.md</a></td></tr>
<tr><td>Geschützter <code>main</code></td><td>Änderungen nur per Pull Request mit grüner CI (lint, kubeconform, gitleaks), siehe <a href="f-cicd-automatisierung/f0090-branch-schutz-main.md">capulus-core/f-cicd-automatisierung/f0090-branch-schutz-main.md</a></td></tr>
</tbody>
</table>

<details>
<summary><strong>Firewall-Ports</strong></summary>

| Port | Protokoll | Scope | Zweck |
|---|---|---|---|
| 22 | TCP | LAN + Tailnet | SSH |
| 53 | UDP+TCP | LAN + Tailnet | dnsmasq Split-DNS für `*.homeserver` |
| 80 | TCP | LAN + Tailnet | Traefik HTTP (leitet auf HTTPS um) |
| 443 | TCP | LAN + Tailnet | Traefik HTTPS |
| 631 | TCP | LAN + Tailnet | CUPS (IPP/AirPrint) |
| 6443 | TCP | LAN + Tailnet | k3s-API |
| 41641 | UDP | Internet | Tailscale-WireGuard |

Die Regeln der `common`-Rolle gelten auch in `prod-vm` und `entw-vm` (dort jeweils mit den eigenen Pod-/Service-CIDRs).
Zusätzlich lässt UFW jeden Verkehr aus dem LAN-Subnetz und dem Tailnet zu, deshalb ist z. B. die ArgoCD-UI der
ENTW-VM (`https://argocd.dev.homeserver`) aus dem LAN erreichbar. Die vollständige Port-Übersicht je App steht in
[capulus-core/c-netzwerk-dns/c0030-port-uebersicht.md](c-netzwerk-dns/c0030-port-uebersicht.md).

</details>

Vollständige Architektur: **[capulus-ansible/a-betriebssystem/a0010-overview.md](../capulus-ansible/a-betriebssystem/a0010-overview.md)**
