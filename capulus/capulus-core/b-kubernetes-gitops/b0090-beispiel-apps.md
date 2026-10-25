# Beispiel-Apps — `demo-app` und `example-whoami`

Zwei kleine Apps zum Ausprobieren des GitOps-Ablaufs. Sie haben keinen Nutzen im Betrieb, zeigen aber, wie sich eine Änderung in Git im Cluster auswirkt, und dienen in der Promotion-Kette als harmlose Testfälle ([f00b0](../f-cicd-automatisierung/f00b0-promotion-chain.md)).

> **Cluster:** ENTW (beide, Ordner `argocd/apps/entw/`), PROD (beide, Ordner `argocd/apps/prod/`). Hosts: `demo.dev.homeserver`, `whoami.dev.homeserver`, `demo.prod.homeserver`, `whoami.prod.homeserver`. In TECH gibt es sie nicht. Zugriff je Cluster: [a0010](../../capulus-ansible/a-betriebssystem/a0010-overview.md#kubectl-zugriff-je-cluster).

---

## Überblick

| App | Image | Zweck |
|---|---|---|
| `demo-app` | `hashicorp/http-echo:1.0.0` | zeigt den Text aus der Umgebungsvariablen `MESSAGE` (`message` in `values.yaml`, Standard „Hello from GitOps - Version 1“). Dient der **Self-Heal-Demo**: Ein Eingriff per `kubectl set env` macht die App `OutOfSync`, ArgoCD setzt den Git-Stand zurück |
| `example-whoami` | `traefik/whoami:v1.12.0` | gibt Hostname, IP und Request-Header aus. Dient dem Test von Ingress, Wildcard-DNS, TLS und HPA |

| Eigenschaft | `demo-app` | `example-whoami` |
|---|---|---|
| Service | ClusterIP, Port 80 → 5678 | ClusterIP, Port 80 → 80 |
| Ingress | Klasse `traefik`, Host `demo.<tier>.homeserver` | Klasse `traefik`, Host `whoami.<tier>.homeserver` |
| Replikas | ENTW: 1, **PROD: 0** (bewusst abgeschaltet) | 1 mit HPA (1 bis 3 bei 70 % CPU) |
| Ressourcen | Limit 100m und 64 MiB | Request 10m und 16 MiB, Limit 100m und 64 MiB |

Die PROD-Kopie von `demo-app` ist mit `replicaCount: 0` ausgerollt (Batch 1 der Migration, siehe [40080](../4-planung/40080-multi-cluster-entw-prod-tech.md)): Die Application ist grün, es läuft aber kein Pod, und `demo.prod.homeserver` antwortet nicht.

---

## Self-Heal-Demo mit `demo-app`

```bash
kubectl port-forward svc/demo-app 8080:80 -n demo-app    # http://localhost:8080
kubectl set env deployment/demo-app MESSAGE="Hier stand etwas anderes" -n demo-app
# ArgoCD zeigt OutOfSync, nach einigen Sekunden setzt Self-Heal zurück
```

Danach in ArgoCD (oder per [cubi](../../../cubi/argocd.md)) beobachten, wie die App wieder `Synced` wird. Die Hinweise nach der Installation stehen in `templates/NOTES.txt` des Charts. Auf ENTW greift Self-Heal wie in [b0050](b0050-entw-argocd.md) beschrieben.

---

## Fehlersuche

```bash
kubectl get pods,svc,ingress -n demo-app
kubectl get pods,svc,ingress,hpa -n example-whoami
curl -k https://whoami.dev.homeserver        # bzw. whoami.prod.homeserver
```

| Symptom | Ursache | Lösung |
|---|---|---|
| `demo.prod.homeserver` antwortet nicht, App ist grün | PROD-`replicaCount` ist 0 | Absicht. Zum Testen `replicaCount` in `argocd/apps/prod/demo-app/values.yaml` anheben |
| Hostname nicht auflösbar | DNS-Wildcard fehlt oder Gerät ohne internen DNS | [DNS-Architektur](../c-netzwerk-dns/c0000-dns-architecture.md), [Domain-Tiers](../c-netzwerk-dns/c0040-domain-tiers.md) |
| Zertifikatswarnung | Root-CA nicht installiert | [Internes TLS](../d-sicherheit/d0040-internal-tls.md) |
| `404 page not found` | Ingress fehlt oder Host falsch | `kubectl get ingress -A`, [Traefik-Konfiguration](b0080-traefik-config.md#fehlersuche) |
| `demo-app` ändert den Text nicht nach `values.yaml`-Änderung | Pod nicht neu gestartet oder ArgoCD noch nicht synchron | ArgoCD-Sync abwarten, `kubectl rollout restart deployment/demo-app -n demo-app` |
| App bleibt `OutOfSync` nach manuellem Eingriff | Auto-Sync oder Self-Heal ist aus (zum Beispiel durch `cubi autosync`) | [cubi: ArgoCD-Features](../../../cubi/argocd-features.md), Auto-Sync wieder aktivieren |
| `example-whoami`: HPA zeigt `<unknown>` | Metrics-Server liefert keine Werte | [HPA](b0040-hpa-autoscaling.md#verifizieren) |
| ENTW-App erscheint nicht in ArgoCD | Ordner nicht auf dem Branch `entw` | [b0050](b0050-entw-argocd.md) |
| Promotion überträgt `demo-app`-Änderung nicht | nur Versionsfelder (`image.tag`) werden übertragen, keine `message` | [f00b0](../f-cicd-automatisierung/f00b0-promotion-chain.md) |

Allgemeine Fehlersuche: [zentrale Fehlersuche](../2-betrieb-hardware/20070-troubleshooting-zentral.md).
