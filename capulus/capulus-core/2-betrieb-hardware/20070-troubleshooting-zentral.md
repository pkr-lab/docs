# Zentrale Fehlersuche — von Symptom zur richtigen Anleitung

Einstiegsseite für Störungen im gesamten Capulus-Umfeld: Cluster, Apps, Hosts, CI/CD, iOS-App, Werkzeuge und Website. Sie führt vom Symptom über eine feste Reihenfolge zur passenden Detailseite. Jede Detailseite hat einen eigenen Abschnitt „Fehlersuche“ (je nach Seite auch „Troubleshooting“ oder „Fehlerbehebung“) mit Meldungen, Ursachen und Lösungen.

---

## Die ersten fünf Minuten

Vor dem Suchen in einzelnen Docs lohnt dieser Ablauf. Er grenzt die Ebene ein, auf der der Fehler liegt (von außen nach innen).

| Schritt | Frage | Befehl oder Ort |
|---|---|---|
| 1 | Läuft der Cluster, und sind die Nodes bereit? | `kubectl get nodes` (Zugriff je Cluster: [a0010](../../capulus-ansible/a-betriebssystem/a0010-overview.md#kubectl-zugriff-je-cluster)), [k3s: Fehlersuche](../b-kubernetes-gitops/b0000-k3s.md#fehlersuche) |
| 2 | Ist ArgoCD grün? Welche App ist rot? | `cubi argocd` ([cubi: ArgoCD](../../../cubi/argocd.md)) oder `kubectl -n argocd get applications` |
| 3 | Läuft der Pod? | `cubi kubernetes` ([cubi: Kubernetes](../../../cubi/kubernetes.md)), `kubectl -n <ns> get pods`, `describe`, `logs` |
| 4 | Ist der Name auflösbar und der Port erreichbar? | `dig +short <host>`, `curl -vk https://<host>/`, [DNS](../c-netzwerk-dns/c0000-dns-architecture.md#fehlersuche), [Ports](../c-netzwerk-dns/c0030-port-uebersicht.md#fehlersuche) |
| 5 | Stimmen Zertifikat und Secrets? | [Internes TLS](../d-sicherheit/d0040-internal-tls.md#fehlersuche), [Sealed Secrets](../b-kubernetes-gitops/b0070-sealed-secrets.md#fehlersuche) |
| 6 | Hat sich vor kurzem etwas geändert? | letzte Merges und Promotions: `cubi github` ([GitHub](../../../cubi/github.md)), [Promotion-Kette](../f-cicd-automatisierung/f00b0-promotion-chain.md#fehlersuche) |
| 7 | Hilft ein Blick in Logs und Alarme? | VictoriaLogs und Grafana ([Logging](../3-apps-workloads/300j0-logging.md#fehlerbehebung)), ntfy-Alarme ([ntfy](../1-benachrichtigungen/10010-ntfy.md)) |

Nützliche Dauerbefehle (im Cluster mit dem Problem):

```bash
kubectl get pods -A | grep -v -e Running -e Completed          # alles, was nicht läuft
kubectl get events -A --sort-by=.lastTimestamp | tail -30       # letzte Ereignisse
kubectl -n <ns> describe pod <pod>                              # Events unten lesen
kubectl -n <ns> logs <pod> --previous --tail=100                # Log des vorherigen Starts
kubectl -n argocd get applications.argoproj.io | grep -v 'Synced.*Healthy'
```

---

## Symptom → Anleitung

### Cluster, GitOps und Netzwerk

| Symptom | Mögliche Ursache | Weiter |
|---|---|---|
| `kubectl` verbindet nicht | k3s aus, falscher Kontext, kubeconfig | [k3s](../b-kubernetes-gitops/b0000-k3s.md#fehlersuche) |
| Worker `NotReady` oder fehlt | Worker ist aus (nachts gewollt), Netz, `k3s-agent` | [k3s](../b-kubernetes-gitops/b0000-k3s.md#fehlersuche), [Cluster Power Manager](../../capulus-ansible/2-betrieb-hardware/20020-cluster-power-manager.md#fehlerbehebung), [Nightly Update](../../capulus-ansible/2-betrieb-hardware/20030-nightly-worker-update.md#fehlerbehebung) |
| Neue App erscheint nicht in ArgoCD | Ordner, ApplicationSet, AppProject | [ArgoCD](../b-kubernetes-gitops/b0010-argocd.md#fehlersuche), [AppProjects](../b-kubernetes-gitops/b0020-argocd-projects.md#fehlersuche) |
| App `OutOfSync`, `Degraded`, `Unknown` | Manifeste, Controller-Änderungen, Pod-Fehler | [ArgoCD](../b-kubernetes-gitops/b0010-argocd.md#fehlersuche), [cubi: ArgoCD](../../../cubi/argocd.md#debugging-und-fehlersuche) |
| Änderung wird zurückgesetzt | Self-Heal, ApplicationSet, übergeordnete App | [cubi: Features](../../../cubi/argocd-features.md#debugging-und-fehlersuche) |
| PROD-Apps fehlen oder Cluster `prod` nicht erreichbar | Cluster-Registrierung im Hub | [PROD im TECH-Hub](../komponenten/bootstrap-prod.md#fehlersuche) |
| App-Umzug nach PROD hakt | Kopier-Job, Daten, DNS | [App-Umzug](../komponenten/bootstrap-prod-migrations.md#fehlersuche) |
| ENTW-App oder ENTW-ArgoCD | eigene Instanz, Branch `entw` | [ENTW-ArgoCD](../b-kubernetes-gitops/b0050-entw-argocd.md#troubleshooting) |
| Pod `Pending` oder PVC `Pending` | Ressourcen, StorageClass, NAS | [k3s](../b-kubernetes-gitops/b0000-k3s.md#fehlersuche), [NAS-Speicher](20000-nas-storage.md#fehlerbehebung) |
| HPA zeigt `<unknown>` oder skaliert nicht | Metrics-Server, Requests | [HPA](../b-kubernetes-gitops/b0040-hpa-autoscaling.md#fehlersuche) |
| `SealedSecret` wird nicht entschlüsselt, Pod `CreateContainerConfigError` | falscher Cluster oder Name, Schlüssel fehlt | [Sealed Secrets](../b-kubernetes-gitops/b0070-sealed-secrets.md#fehlersuche) |
| Host nicht auflösbar (`*.homeserver`) | DNS-Weg, dnsmasq, Tailscale-Split-DNS | [DNS](../c-netzwerk-dns/c0000-dns-architecture.md#fehlersuche), [Tailscale](../c-netzwerk-dns/c0010-tailscale.md#troubleshooting) |
| Pods lösen `*.homeserver` nicht auf | CoreDNS-Erweiterung | [CoreDNS](../c-netzwerk-dns/c0050-coredns-custom.md#fehlersuche) |
| Zertifikatswarnung im Browser | Root-CA nicht installiert, Host nicht in SAN, Zertifikat abgelaufen | [Internes TLS](../d-sicherheit/d0040-internal-tls.md#fehlersuche), [Traefik](../b-kubernetes-gitops/b0080-traefik-config.md#fehlersuche) |
| `404 page not found` oder Redirect-Schleife | Ingress, Traefik-Konfiguration | [Traefik](../b-kubernetes-gitops/b0080-traefik-config.md#fehlersuche) |
| `192.168.178.200` antwortet nicht | MetalLB | [MetalLB](../b-kubernetes-gitops/b0060-metallb.md#fehlersuche) |
| Dienst von außen (Internet) nicht erreichbar | Cloudflare Tunnel | [Tunnel](../e-externe-erreichbarkeit/e0000-cloudflare-tunnel.md#troubleshooting), [Deploy](../e-externe-erreichbarkeit/e0010-cloudflare-deploy.md#troubleshooting) |
| Pod erreicht anderen Namespace nicht (Timeout) | NetworkPolicy | [Network Policies](../d-sicherheit/d0030-network-policies.md#fehlersuche) |
| VPN, Subnet-Routen, Tailscale-Login | Auth-Key, ACL, Dienst | [Tailscale](../c-netzwerk-dns/c0010-tailscale.md#troubleshooting), [WireGuard-Backup](../c-netzwerk-dns/c0011-wireguard-backup.md#troubleshooting) |
| Ports, wer lauscht wo | Port-Übersicht | [Ports](../c-netzwerk-dns/c0030-port-uebersicht.md#fehlersuche) |
| Werbung blockiert oder DNS-Probleme durch Pi-hole | Blocklisten, Update | [Pi-hole](../c-netzwerk-dns/c0020-pihole.md#5-troubleshooting), [Incident 50020](../5-incidents/50020-pihole-update-dns-deadlock.md) |

### Alarme, Monitoring und Logs

| Symptom | Mögliche Ursache | Weiter |
|---|---|---|
| Alarm kommt nicht aufs Telefon | Alertmanager-Route, ntfy-bridge, ntfy-Token, iOS-App | [ntfy-bridge](../1-benachrichtigungen/10030-ntfy-bridge.md#fehlersuche), [ntfy](../1-benachrichtigungen/10010-ntfy.md#troubleshooting) |
| Kein Anruf bei schweren Ausfällen | alert-caller, SIP | [Telefon-Alarm](../1-benachrichtigungen/10020-telefon-alarm.md#fehlersuche) |
| Keine Alarme zu ArgoCD oder Backups | Regeln, Routing | [GitOps- und Backup-Alerts](20050-gitops-und-backup-alerts.md#fehlersuche) |
| Metriken oder Dashboards leer | VictoriaMetrics, Scrapes | [Monitoring](../3-apps-workloads/300q0-monitoring.md#fehlersuche), [Monitoring: Hintergründe](../6-hintergruende/60060-monitoring-und-alerting.md), [Hardware-Monitoring](../../capulus-ansible/2-betrieb-hardware/20060-hardware-monitoring.md#bekannte-einschränkungen--fehlersuche) |
| Logs fehlen in Grafana | `journal_upload`, Collector | [Logging](../3-apps-workloads/300j0-logging.md#fehlerbehebung), [Rolle journal_upload](../../capulus-ansible/rollen/journal_upload.md) |
| Status-Page oder Uptime-Kuma-Monitore falsch | Monitore, Slug, CORS | [Uptime Kuma](../3-apps-workloads/30080-uptime-kuma.md#fehlersuche) |
| NAS-Platte rot, Homeserver nicht erreichbar | RAID, NFS | [NAS-Speicher](20000-nas-storage.md#fehlerbehebung), [Incident 50000](../5-incidents/50000-nas-disk-red-homeserver-unreachable.md) |
| Backup fehlt oder zu alt | CronJob, restic, NAS | [NAS-Backup](20010-nas-backup.md#fehlerbehebung), [GitOps- und Backup-Alerts](20050-gitops-und-backup-alerts.md#fehlersuche) |

### CI/CD, Releases und Secrets

| Symptom | Mögliche Ursache | Weiter |
|---|---|---|
| CI rot (lint, kubeconform, go, token register, gitleaks) | Code, Schema, Register | [CI-Lint](../f-cicd-automatisierung/f0070-ci-lint.md#fehlersuche), [Skripte und Workflows](../f-cicd-automatisierung/f00c0-skripte-und-workflows-referenz.md) |
| Push auf `main` abgelehnt, PR nicht mergebar | Branch-Schutz | [Branch-Schutz](../f-cicd-automatisierung/f0090-branch-schutz-main.md#fehlersuche) |
| Kein PR aus ENTW | Promotion, CI auf `entw`, Konflikt | [ENTW-Promotion](../f-cicd-automatisierung/f0080-entw-promotion.md#fehlersuche) |
| Versionen wandern nicht von ENTW nach TECH und PROD | Gate, ArgoCD-Token, Tailnet | [Promotion-Kette](../f-cicd-automatisierung/f00b0-promotion-chain.md#fehlersuche), [Tailscale-Runner](../f-cicd-automatisierung/f00a0-tailscale-runner-poc.md#troubleshooting) |
| Image wird nicht gebaut oder nicht ausgerollt | Pfadfilter, Registry, Bump-PR | [Images bauen](../f-cicd-automatisierung/f0060-build-images.md#fehlersuche) |
| Kein Release, Changelog fehlt | Commit-Format, Rechte | [Release-Automatisierung](../f-cicd-automatisierung/f0030-release-automation.md#troubleshooting), [github-workflows](../../../github-workflows/index.md) |
| Renovate-PRs fehlen | Zeitplan, Token | [Renovate](../f-cicd-automatisierung/f0020-renovate.md#troubleshooting) |
| GitLab-Spiegel veraltet | Mirror-Token | [GitLab-Mirror](../f-cicd-automatisierung/f0050-gitlab-mirror.md#troubleshooting) |
| Neue Version eines Fremdprojekts wird nicht gemeldet | Release-Watcher | [GitHub Release Watcher](../f-cicd-automatisierung/f0040-github-release-watcher.md#troubleshooting) |
| Argo-Workflow-Lauf scheitert | Artefaktspeicher, Rechte | [Argo Workflows](../f-cicd-automatisierung/f0000-argo-workflows.md#4-troubleshooting), [MinIO](../3-apps-workloads/300p0-minio.md#fehlersuche) |
| Security-Issue oder CodeQL-Befund | Alerts | [CodeQL und Dependabot](../d-sicherheit/d0080-codeql-dependabot.md#troubleshooting) |
| Token läuft ab, Rotation | Token-Register | [Secrets-Rotation](../d-sicherheit/d0060-secrets-rotation.md#fehlersuche), [cubi: Tokens](../../../cubi/tokens.md#debugging-und-fehlersuche) |
| Docs im Wiki nicht aktuell | wiki-docs-sync | [wiki-sync](../wiki-sync.md#fehlersuche) |

### Apps und Dienste

| App | Detailseite (Fehlersuche) |
|---|---|
| Zammad | [30000](../3-apps-workloads/30000-zammad.md#troubleshooting) |
| Alarmmonitor-Kiosks (ALAMOS) | [30010](../3-apps-workloads/30010-alamos-apager.md#fehlerbehebung), [Vereinsheim-Alarmmonitor](../../capulus-ansible/3-apps-workloads/30020-vereinsheim-alarmmonitor.md#fehlerbehebung), [Windows-PC](../3-apps-workloads/30021-vereinsheim-windows-pc-steuerung.md#fehlersuche) |
| Infotafel-Kiosk | [Infotafel](../../capulus-ansible/3-apps-workloads/300e8-infotafel-kiosk.md#fehlerbehebung) |
| Wiki.js | [30030](../3-apps-workloads/30030-wikijs.md#troubleshooting) |
| MediaMTX | [30040](../3-apps-workloads/30040-mediamtx.md#troubleshooting) |
| Paperless-ngx | [30050](../3-apps-workloads/30050-paperless-ngx.md#fehlersuche) |
| n8n | [30070](../3-apps-workloads/30070-n8n.md#fehlerbehebung) |
| Uptime Kuma | [30080](../3-apps-workloads/30080-uptime-kuma.md#fehlersuche) |
| Monitoring (VictoriaMetrics, Grafana, Alertmanager) | [300q0](../3-apps-workloads/300q0-monitoring.md#fehlersuche) |
| Rhein-Dashboard | [30090](../3-apps-workloads/30090-rhein-dashboard.md#fehlerbehebung) |
| Vaultwarden | [300a0](../3-apps-workloads/300a0-vaultwarden.md#9-troubleshooting) |
| Nextcloud | [300b0](../3-apps-workloads/300b0-nextcloud.md#troubleshooting) |
| Immich | [300c0](../3-apps-workloads/300c0-immich.md#troubleshooting) |
| carplay-api | [300d0](../3-apps-workloads/300d0-carplay-api.md#fehlerbehebung), [Komponente](../komponenten/carplay-api.md#fehlersuche) |
| Xibo Signage | [300e0](../3-apps-workloads/300e0-xibosignage.md#fehlerbehebung) |
| pacman | [300f0](../3-apps-workloads/300f0-pacman-visitor-tracking.md#fehlersuche), [Komponente](../komponenten/pacman.md#fehlersuche) |
| Ollama | [300g0](../3-apps-workloads/300g0-ollama.md#troubleshooting) |
| ALAMOS-Einsatz nach Zammad | [300h0](../3-apps-workloads/300h0-alamos-einsatz-zammad.md#fehlerbehebung) |
| alamos-relay | [300i0](../3-apps-workloads/300i0-alamos-relay.md#fehlerbehebung) |
| Logging | [300j0](../3-apps-workloads/300j0-logging.md#fehlerbehebung) |
| isc-news-bot | [300k0](../3-apps-workloads/300k0-isc-news-bot.md#troubleshooting) |
| Instagram-Post per Mail | [300l0](../3-apps-workloads/300l0-instagram-post-per-mail.md#fehlerbehebung) |
| ISC Urkunden melden | [300m0](../3-apps-workloads/300m0-isc-urkunden-melden.md#troubleshooting) |
| Headlamp | [300n0](../3-apps-workloads/300n0-headlamp.md#fehlersuche) |
| kubeseal-webgui | [300o0](../3-apps-workloads/300o0-kubeseal-webgui.md#fehlersuche) |
| MinIO | [300p0](../3-apps-workloads/300p0-minio.md#fehlersuche) |
| ntfy | [10010](../1-benachrichtigungen/10010-ntfy.md#troubleshooting) |
| ntfy-bridge | [10030](../1-benachrichtigungen/10030-ntfy-bridge.md#fehlersuche) |
| lldap | [d0072](../d-sicherheit/d0072-lldap.md#troubleshooting) |
| Beispiel-Apps (demo-app, whoami) | [b0090](../b-kubernetes-gitops/b0090-beispiel-apps.md#fehlersuche) |

### Hosts und Provisionierung (capulus-ansible)

| Symptom | Weiter |
|---|---|
| Playbook scheitert, Erstinstallation | [Troubleshooting](../../capulus-ansible/a-betriebssystem/a0040-troubleshooting.md), [Rollen: Fehlersuche](../../capulus-ansible/rollen/index.md#fehlersuche-je-rolle) |
| Semaphore läuft nicht | [Semaphore](../../capulus-ansible/b-kubernetes-gitops/b0030-semaphore.md#troubleshooting) |
| Brute-Force-Schutz, CrowdSec | [CrowdSec](../../capulus-ansible/d-sicherheit/d0020-crowdsec.md#troubleshooting) |
| Drucker (CUPS) | [Drucker](../../capulus-ansible/2-betrieb-hardware/20040-printer.md#fehlerbehebung) |
| Secrets-at-Rest und Audit-Log | [Secrets-Encryption](../../capulus-ansible/d-sicherheit/d0050-secrets-encryption-audit-log.md#fehlersuche) |
| Kiosk-Geräte (Banana Pi, Xibo, ALAMOS) | [Rollen](../../capulus-ansible/rollen/index.md#fehlersuche-je-rolle), [Vereinsheim-Alarmmonitor](../../capulus-ansible/3-apps-workloads/30020-vereinsheim-alarmmonitor.md#fehlerbehebung) |

### Weitere Projekte

| Projekt | Detailseite |
|---|---|
| [cubi](../../../cubi/index.md) | [Debugging und Fehlersuche](../../../cubi/debugging.md) |
| [capulus-ios](../../capulus-ios/index.md) | [Debugging und Fehlersuche](../../capulus-ios/debugging.md) |
| [EDV-Kretzer (Website)](../../../EDV-Kretzer/index.md) | [Debugging und Fehlersuche](../../../EDV-Kretzer/debugging.md) |
| [ui-terminal-config](../../../ui-terminal-config/index.md) | [Debugging und Fehlersuche](../../../ui-terminal-config/debugging.md) |
| [DocFlowEngine](../../../DocFlowEngine/index.md) | [Debugging und Fehlersuche](../../../DocFlowEngine/debugging.md) |
| [PowerShell-Skripte](../../../PowerShell/index.md) | [Debugging und Fehlersuche](../../../PowerShell/debugging.md), [git-pull-push](../../../PowerShell/bash-git-pull-push.md#debugging-und-fehlersuche) |
| [github-workflows](../../../github-workflows/index.md) | [Nutzung](../../../github-workflows/nutzung.md) |

---

## Vorgehen bei Vorfällen

1. **Eingrenzen**: Welche Ebene? (Netz, DNS, Ingress, Pod, Daten, Secret, Abhängigkeit). Von außen nach innen prüfen.
2. **Nichts zerstören**: erst lesen (`get`, `describe`, `logs`), dann ändern. Bei GitOps-Apps Änderungen über Git machen, weil Self-Heal manuelle Änderungen zurücksetzt. Für Tests Auto-Sync kontrolliert abschalten ([cubi: Auto-Sync](../../../cubi/argocd-features.md)).
3. **Dokumentieren**: bei größeren Vorfällen einen Bericht nach Typ C ([TEMPLATE](../TEMPLATE.md)) unter [5-incidents](../5-incidents/50000-nas-disk-red-homeserver-unreachable.md) bzw. für Sicherheitsvorfälle unter `7-security-incidents/` anlegen.
4. **Regel ableiten**: Erkenntnisse gehören als Begründung in die Hintergründe ([60000](../6-hintergruende/60000-uebersicht.md)).

Beispiele für ausgewertete Vorfälle: [Incident 12.08.2026](../d-sicherheit/d0000-incident-2026-08-12.md), [NAS-Platte rot](../5-incidents/50000-nas-disk-red-homeserver-unreachable.md), [PROD-VM-Basis-Image ersetzt](../5-incidents/50010-prod-vm-basis-image-ersetzt.md), [Pi-hole-Update und DNS-Deadlock](../5-incidents/50020-pihole-update-dns-deadlock.md).
