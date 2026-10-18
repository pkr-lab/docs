# Nutzung

Typische Abläufe im Betrieb. Befehle für Hosts (Playbooks, `make …`) laufen im Verzeichnis von [capulus-ansible](../capulus-ansible/nutzung.md).

## Lokale Prüfungen vor dem Commit

```bash
make lint
python3 scripts/check-token-register.py
(cd argocd/apps/tech/pacman/server && go build ./... && go vet ./... && go test ./... && go mod tidy -diff)
(cd argocd/apps/tech/carplay-api/src && go build ./... && go vet ./... && go test ./... && go mod tidy -diff)
(cd argocd/apps/tech/isc-news-bot && npm ci && npm test)
helm template argocd/apps/<cluster>/<app> | kubeconform -strict -ignore-missing-schemas -summary
```

`make lint` führt yamllint über `argocd/` und `helm lint` über alle Charts aus. Nur die betroffenen Teile prüfen, nicht immer alles.

## Application hinzufügen

Erst entscheiden, **in welchen Cluster** die App gehört: **ENTW** (Entwicklung/Test, der übliche Startpunkt), **TECH** (Infrastruktur und Betriebsdienste) oder **PROD** (Apps mit echtem Nutzerkreis). Die Projekte, Ordner und Pflichtschritte je Cluster stehen in [b0020 ArgoCD-Projekte](b-kubernetes-gitops/b0020-argocd-projects.md).

| Cluster | Ordner | Zusätzlich nötig |
|---|---|---|
| **TECH** | `argocd/apps/tech/<app>/` | Namespace in `argocd_platform_apps` oder `argocd_workloads_apps` (`roles/argocd/defaults/main.yml` in capulus-ansible), danach dort `make render-bootstrap CORE_DIR=<Pfad zu capulus-core>` |
| **PROD** | `argocd/apps/prod/<app>/` | Namespace in `argocd/bootstrap-prod/appproject.yaml` (im Hub anwenden), bei reinen Manifest-Ordnern Pfad im ApplicationSet, interner Host in `dnsmasq_prod_vm_hosts` (capulus-ansible) |
| **ENTW** | `argocd/apps/entw/<app>/` auf dem Branch `entw` | nichts |

Beispiel TECH:

```bash
cd capulus-core
git checkout -b feat/my-app
mkdir -p argocd/apps/tech/my-app

cd ../capulus-ansible
$EDITOR roles/argocd/defaults/main.yml
make render-bootstrap CORE_DIR=../capulus-core
```

Danach in beiden Repos je einen Commit und PR (`feat(apps): my-app hinzufügen`). `main` ist geschützt, ein Merge rollt sofort aus. Innerhalb von etwa 3 Minuten erkennt ArgoCD das neue Verzeichnis, legt eine Application `my-app` im Namespace `my-app` an (PROD-Applications heißen `prod-<app>`) und synct sie. Hostnamen immer im Tier-Schema (`<app>.tech.homeserver`, `<app>.prod.homeserver`, `<app>.dev.homeserver`).

Mehr: [b0010 ArgoCD](b-kubernetes-gitops/b0010-argocd.md), [f00b0 Promotion-Kette](f-cicd-automatisierung/f00b0-promotion-chain.md).

## Secret versiegeln

```bash
printf '%s' "$WERT" | kubeseal --raw --namespace <ns> --name <secret> \
  --controller-namespace sealed-secrets --controller-name sealed-secrets-controller \
  --from-file=/dev/stdin
```

Hilfsskripte in `scripts/`: `seal-*.sh` für einzelne Secrets, `reseal-for-prod.sh` für den Umzug eines SealedSecrets in den PROD-Cluster. Neue Tokens werden in `ops/token-register.yaml` eingetragen.

## Promotion ENTW → TECH → PROD

Eine Änderung läuft zuerst über den Branch `entw` in den ENTW-Cluster, nach 24 Stunden Gesundheit als PR nach `main` (TECH), nach weiteren 2 Stunden als PR für PROD. Details und Voraussetzungen: [f0080 ENTW-Promotion](f-cicd-automatisierung/f0080-entw-promotion.md), [f00b0 Promotion-Kette](f-cicd-automatisierung/f00b0-promotion-chain.md).

## Images bauen

`build-images.yml` baut die selbst gehosteten Images (`pacman`, `carplay-api`, `n8n`, `alert-caller`, `isc-news-bot`) bei Änderungen am Dockerfile oder Quellcode und schreibt den Tag `sha-<commit>` nach `image.tag` in der `values.yaml` (`scripts/bump-image-tag.py`). Registry: `ghcr.io/pkr-lab`. Manuell: Workflow `Build Workload Images` mit Auswahl der App starten. Details: [f0060 Images bauen](f-cicd-automatisierung/f0060-build-images.md).

## Monitoring

Ein schlanker VictoriaMetrics-+-Grafana-Stack lebt unter `argocd/apps/tech/monitoring/` und wird automatisch von ArgoCD ausgerollt. Er überwacht **alle drei Cluster**, das NAS und die Kiosk-Geräte; Logs laufen zentral in VictoriaLogs (`argocd/apps/tech/logging/`, [Doku](3-apps-workloads/300j0-logging.md)).

<details>
<summary><strong>Stack-Details</strong></summary>

| Komponente | Detail |
|---|---|
| **TSDB** | VMSingle — 15 Tage Retention, 10 Gi `local-path`-PVC |
| **Scrapers** | VMAgent scrapet alle `VMServiceScrape`/`VMPodScrape` + Prometheus `ServiceMonitor`-CRDs |
| **Host-Metriken** | `prometheus-node-exporter` als DaemonSet auf den TECH-Knoten, `node_exporter` auf `prod-vm`/`entw-vm`, smartctl-Exporter für die Platten |
| **Cluster-Metriken** | kubelet/cAdvisor, kube-apiserver, kube-state-metrics, CoreDNS |
| **Alerts** | Default-kube-prometheus-Rules; ntfy-Alertmanager-Bridge |
| **Dashboards** | Ordner *Hardware* (Übersicht mit Waben, Server-Detail, Speicher & S.M.A.R.T.), *Plattform (TECH)* und *Anwendungen (TECH)*, Node Exporter Full, VictoriaMetrics + Kubernetes Views, siehe [Hardware-Monitoring](../capulus-ansible/2-betrieb-hardware/20060-hardware-monitoring.md) |

</details>

Grafana öffnen unter **https://grafana.tech.homeserver** — Admin-Passwort abfragen (auf dem TECH-Cluster):

```bash
kubectl -n monitoring get secret grafana-admin \
  -o jsonpath='{.data.admin-password}' | base64 -d; echo
```

## Tokens rotieren

`scripts/token-watch.py` meldet täglich ablaufende oder zu alte Tokens. Rotationsanleitungen stehen im Feld `rotation.runbook` des Registers; das Terminal-Werkzeug [cubi](../cubi/index.md) zeigt Alter und Ablauf an und führt durch die Rotation.
