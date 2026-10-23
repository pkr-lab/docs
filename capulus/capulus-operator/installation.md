# Installation

Code: [capulus-operator](https://github.com/pkr-lab/capulus-operator), Chart: [argocd/apps/tech/capulus-operator](https://github.com/pkr-lab/capulus-core/tree/main/argocd/apps/tech/capulus-operator)

---

## Voraussetzungen

| Voraussetzung | Wofür | Prüfen |
|---|---|---|
| Namespace `capulus-operator` in `argocd_platform_apps` (capulus-ansible) | ArgoCD legt die Application und den Namespace an | `argocd/bootstrap/projects.yaml` enthält den Namespace |
| Pull-Secret `ghcr-pull` im Namespace `capulus-operator` | das Image liegt in einem privaten GHCR-Package | `kubectl -n capulus-operator get secret ghcr-pull` |
| Image `ghcr.io/pkr-lab/capulus-operator:sha-<commit>` | Workflow `build-image.yml` baut es bei jeder Änderung an Code oder Dockerfile | Tag in `values.yaml` unter `image.tag` |
| VictoriaMetrics-Operator | `VMServiceScrape` und `VMRule` | läuft im Namespace `monitoring` |

---

## Schritt für Schritt

### 1. Image bauen lassen

Nach dem Merge nach `main` baut `build-image.yml` das Image und pusht `sha-<commit>`. Den Tag in `argocd/apps/tech/capulus-operator/values.yaml` unter `image.tag` eintragen. Ein Tag, den es in GHCR nicht gibt, endet im Pod in `ImagePullBackOff`.

### 2. Pull-Secret anlegen

```bash
kubectl create namespace capulus-operator
kubectl -n capulus-operator create secret docker-registry ghcr-pull \
  --docker-server=ghcr.io --docker-username=<user> --docker-password=<pat>
```

### 3. Operator ausrollen

capulus-ansible: `make render-bootstrap CORE_DIR=<Pfad zu capulus-core>` erzeugt `argocd/bootstrap/projects.yaml` neu. Nach dem Merge legt ArgoCD die Application `capulus-operator` an. Sie bleibt auf `InvalidSpecError` ("do not match any of the allowed destinations in project 'tech'"), bis das AppProject im Cluster angewendet ist, denn `projects.yaml` rollt nicht ArgoCD aus, sondern die Rolle `argocd`:

```bash
ssh homeserver 'sudo kubectl diff -f -' < argocd/bootstrap/projects.yaml
ssh homeserver 'sudo kubectl apply -f -' < argocd/bootstrap/projects.yaml
ssh homeserver 'sudo kubectl -n argocd annotate application capulus-operator argocd.argoproj.io/refresh=hard --overwrite'
```

Das Chart bringt die CRDs aus `crds/` mit.

### 4. Verifizieren

```bash
kubectl -n capulus-operator get pods
kubectl get crd | grep capulus.homeserver
kubectl -n capulus-operator logs deploy/capulus-operator | tail
```

Erwartet: Pod `Running`, vier CRDs, im Log `Starting workers` ohne `error`.

---

## Reihenfolge bei der Umstellung eines Backups

Der Operator muss laufen, **bevor** ein bestehender Backup-CronJob eines Charts durch eine `Backup`-Ressource ersetzt wird. Sonst fehlt zwischen dem Prune des alten CronJobs und dem ersten Abgleich durch den Operator eine Sicherung. Für Vaultwarden heißt das: erst `capulus-operator` mergen und prüfen, dann die Umstellung im Chart `vaultwarden`.
