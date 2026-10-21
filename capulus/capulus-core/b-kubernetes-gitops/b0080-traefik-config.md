# Traefik-Konfiguration — Weiterleitung, Metriken und Standard-Zertifikat

Traefik wird vom k3s selbst mitgebracht ([k3s](b0000-k3s.md#ingress--traefik)). Der Ordner `traefik-config` ergänzt ihn per GitOps um drei Dinge: die globale Weiterleitung von HTTP auf HTTPS, einen Metrik-Endpunkt für das Monitoring und das Standard-Zertifikat. Es sind reine Konfigurationsobjekte ohne eigene Pods.

> **Cluster:** TECH · Ordner `argocd/apps/tech/traefik-config/` · deployt in den Namespace `kube-system` (dort müssen `HelmChartConfig` und `TLSStore` liegen), AppProject-Namespace-Eintrag `traefik-config`. `kubectl`-Befehle in diesem Doc gelten für den TECH-Cluster ([Zugriff je Cluster](../../capulus-ansible/a-betriebssystem/a0010-overview.md#kubectl-zugriff-je-cluster)). PROD hat einen eigenen Ordner `argocd/apps/prod/traefik-config/`.

---

## Bestandteile

| Datei | Objekt | Wirkung |
|---|---|---|
| `helmchartconfig.yaml` | `HelmChartConfig/traefik` (k3s-Helm-Controller) | `web` leitet dauerhaft (`permanent: true`) auf `websecure` (HTTPS) um. Metriken: Prometheus-Endpunkt auf Port `9100` des Entrypoints `metrics` (nur im Cluster, `expose.default: false`), mit Labels für Router und Services |
| `tlsstore.yaml` | `TLSStore/default` (`traefik.io/v1alpha1`) | Standard-Zertifikat für alle Hosts ohne eigenen TLS-Block: Secret `homeserver-wildcard-tls` ([Internes TLS](../d-sicherheit/d0040-internal-tls.md)) |
| `vmservicescrape.yaml` | `VMServiceScrape/traefik-metrics` | VictoriaMetrics scrapt `/metrics` am Port `metrics` des Traefik-Service alle 30 Sekunden (`honorLabels`). Daraus entsteht unter anderem die [Dienst-Aktivität](../../capulus-ios/uebersicht-und-alerts.md) in der iOS-App |

Folge der Weiterleitung: Jeder Aufruf mit `http://` bekommt ein `301`/`308` auf `https://`. Clients, die Weiterleitungen nicht folgen oder Token im ersten Request senden, sind betroffen (Hintergrund: [Security-Incident 018](../7-security-incidents/018-ios-app-klartext-http-mit-bearer-token.md)). Die Bedeutung des Hostnamen-Schemas steht in [Domain-Tiers](../c-netzwerk-dns/c0040-domain-tiers.md), die LAN-Adresse des Service in [MetalLB](b0060-metallb.md).

---

## Fehlersuche

```bash
kubectl -n kube-system get helmchartconfig traefik -o yaml
kubectl -n kube-system get tlsstore default -o yaml
kubectl -n kube-system get pods -l app.kubernetes.io/name=traefik
kubectl -n kube-system logs deploy/traefik --tail=100
kubectl -n kube-system get certificate homeserver-wildcard-tls
```

| Symptom | Ursache | Lösung |
|---|---|---|
| HTTP-Aufrufe werden nicht umgeleitet | `HelmChartConfig` nicht angewendet oder liegt im falschen Namespace | muss `metadata.namespace: kube-system` haben ([60040](../6-hintergruende/60040-helm-charts-tech.md)). Nach einer Änderung startet der k3s-Helm-Controller den Traefik-Redeploy, `kubectl -n kube-system get pods` beobachten |
| Browser zeigt Zertifikatsfehler für `*.homeserver` | `TLSStore` fehlt oder das Secret `homeserver-wildcard-tls` existiert nicht | `kubectl -n kube-system get certificate homeserver-wildcard-tls` muss `READY=True` zeigen, [Internes TLS](../d-sicherheit/d0040-internal-tls.md) |
| Browser zeigt das Traefik-Standardzertifikat (`TRAEFIK DEFAULT CERT`) | Secret fehlt oder Traefik hat es noch nicht geladen | Secret und `TLSStore` prüfen, Traefik-Pod neu starten |
| Keine Traefik-Metriken in VictoriaMetrics | `VMServiceScrape` nicht angewendet (VM-Operator-CRD fehlt) oder Port `metrics` nicht im Service | `kubectl -n kube-system get vmservicescrape`, `kubectl -n kube-system get svc traefik-metrics` oder `get svc`, [Monitoring](../6-hintergruende/60060-monitoring-und-alerting.md) |
| Dienst-Aktivität in der App ist 0 | Metrik-Labels (`addServicesLabels`) fehlen oder der Service-Name passt nicht zum Filter | [carplay-api: Datenquellen](../6-hintergruende/60080-carplay-api-und-ios-app.md#datenquellen-und-ihre-eigenheiten) |
| ArgoCD-App `traefik-config` ist `OutOfSync` | k3s verwaltet `HelmChartConfig` selbst oder CRD fehlt | `kubectl api-resources | grep helmchartconfig`, [ArgoCD](b0010-argocd.md) |
| Nach k3s-Update fehlt die Konfiguration | k3s hat das Traefik-Chart ersetzt, `HelmChartConfig` bleibt aber bestehen | Objekte prüfen, ggf. in ArgoCD einen Sync auslösen |
| Neue Ports (zum Beispiel für TCP-Dienste) erreichbar machen | Entrypoint fehlt | `ports:` in `valuesContent` ergänzen, auch [Port-Übersicht](../c-netzwerk-dns/c0030-port-uebersicht.md) pflegen |
| `404 page not found` von Traefik | kein Router für Host und Pfad | Ingress des Dienstes prüfen (`kubectl get ingress -A`) |

Allgemeine Fehlersuche: [zentrale Fehlersuche](../2-betrieb-hardware/20070-troubleshooting-zentral.md).
