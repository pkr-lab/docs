# PROD-Cluster im TECH-Hub

Der PROD-Cluster (`prod-vm`, 192.168.178.99) hat **keine eigene ArgoCD-Instanz**: der ArgoCD-Hub auf
TECH (`homeserver`) verwaltet ihn als registrierten Cluster `prod`. Die Dateien in diesem Ordner sind
handgeschrieben, **nicht** vom `argocd`-Ansible-Role generiert (anders als `argocd/bootstrap/`), und
beruehren das ApplicationSet `home-server-apps-tech` nicht. Ueberblick ueber Projekte und
ApplicationSets: [capulus-core/b-kubernetes-gitops/b0020-argocd-projects.md](../b-kubernetes-gitops/b0020-argocd-projects.md).

Inhalt: `appproject.yaml` (Projekt `prod`), `applicationset.yaml` (zwei ApplicationSets) und
`migrations/` (manuelle Hilfsmanifeste fuer den Umzug von Apps samt Daten von TECH nach PROD,
siehe [migrations/README.md](bootstrap-prod-migrations.md)).

## 1. Cluster "prod" im Hub registrieren

Auf **prod-vm** (ServiceAccount + Token, entspricht `argocd cluster add`):

```bash
sudo k3s kubectl apply -f - <<'YAML'
apiVersion: v1
kind: ServiceAccount
metadata: {name: argocd-manager, namespace: kube-system}
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata: {name: argocd-manager}
roleRef: {apiGroup: rbac.authorization.k8s.io, kind: ClusterRole, name: cluster-admin}
subjects: [{kind: ServiceAccount, name: argocd-manager, namespace: kube-system}]
---
apiVersion: v1
kind: Secret
metadata:
  name: argocd-manager-token
  namespace: kube-system
  annotations: {kubernetes.io/service-account.name: argocd-manager}
type: kubernetes.io/service-account-token
YAML
sudo k3s kubectl -n kube-system get secret argocd-manager-token -o jsonpath='{.data.token}' | base64 -d; echo
sudo k3s kubectl -n kube-system get secret argocd-manager-token -o jsonpath='{.data.ca\.crt}'   # base64 belassen
```

Auf **homeserver** (TECH) das Secret anlegen. Token und CA nur lokal
einsetzen, **nicht** ins Repo committen:

```bash
kubectl apply -f - <<YAML
apiVersion: v1
kind: Secret
metadata:
  name: cluster-prod
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: cluster
    cluster-tier: prod
type: Opaque
stringData:
  name: prod
  server: https://192.168.178.99:6443
  config: |
    {"bearerToken": "<TOKEN>", "tlsClientConfig": {"insecure": false, "caData": "<CA_BASE64>"}}
YAML
```

## Zwei ApplicationSets

`applicationset.yaml` enthaelt zwei Sets, weil `helm.releaseName` ArgoCD in den
Helm-Modus zwingt und bei Ordnern ohne `Chart.yaml` scheitert:

- `home-server-apps-prod`: reine Manifest-Ordner, **explizit gelistet**. Ein neuer
  Manifest-Ordner unter `argocd/apps/prod/` braucht dort einen `path`-Eintrag.
- `home-server-apps-prod-charts`: jeder Ordner mit `Chart.yaml`, automatisch;
  Helm-Release-Name = Ordnername (wie in TECH).

Ordner mit `Chart.yaml` nicht im ersten Set listen (doppelte Applications).

## 2. Projekt + ApplicationSet anwenden

```bash
kubectl apply -f argocd/bootstrap-prod/appproject.yaml
kubectl apply -f argocd/bootstrap-prod/applicationset.yaml
```

## 3. Pruefen

```bash
kubectl -n argocd get applications | grep '^prod-'     # auf homeserver (TECH-Hub): prod-sealed-secrets, prod-nextcloud, ...
sudo k3s kubectl get pods -A                            # auf prod-vm
```

## Neue PROD-App hinzufuegen

1. Ordner `argocd/apps/prod/<app>/` anlegen (Ordnername = Namespace = Helm-Release-Name).
2. Namespace in `appproject.yaml` ergaenzen und die Datei im Hub anwenden
   (`kubectl apply -f argocd/bootstrap-prod/appproject.yaml`). Ohne diesen Schritt meldet die
   Application `InvalidSpecError`/`Unknown`; danach `argocd.argoproj.io/refresh=hard` auf die Application setzen.
3. Nur bei einem **reinen Manifest-Ordner** (ohne `Chart.yaml`): `path` in `applicationset.yaml`
   (Set `home-server-apps-prod`) ergaenzen und anwenden. Ordner mit `Chart.yaml` werden vom Set
   `home-server-apps-prod-charts` automatisch gefunden.
4. Fuer einen internen Host unter `*.prod.homeserver`: Name in `dnsmasq_prod_vm_hosts`
   (`capulus-ansible/group_vars/all.yml`) eintragen und `make dnsmasq` ausfuehren, sonst antwortet der
   TECH-Traefik. Ein oeffentlicher Host braucht zusaetzlich einen DNS-Eintrag auf den PROD-Tunnel:
   `cloudflared tunnel route dns homeserver-prod <host>`, siehe
   [capulus-core/e-externe-erreichbarkeit/e0010-cloudflare-deploy.md](../e-externe-erreichbarkeit/e0010-cloudflare-deploy.md).
5. SealedSecrets muessen mit dem Zertifikat des **PROD**-sealed-secrets-Controllers versiegelt werden
   (`scripts/reseal-for-prod.sh`, Kopf des Skripts), die TECH-Versiegelung ist in PROD nicht entschluesselbar.

Rueckbau: `kubectl delete -f argocd/bootstrap-prod/applicationset.yaml`
(entfernt die Applications samt Ressourcen in PROD, TECH bleibt unberuehrt).

---

## Fehlersuche

```bash
kubectl -n argocd get secret -l argocd.argoproj.io/secret-type=cluster          # Cluster prod registriert?
kubectl -n argocd get applications | grep '^prod-'
kubectl -n argocd get applicationset home-server-apps-prod home-server-apps-prod-charts
kubectl -n argocd describe application prod-<app>
curl -k https://192.168.178.99:6443/version                                      # API von prod-vm erreichbar?
```

| Symptom | Ursache | Lösung |
|---|---|---|
| Keine `prod-*`-Applications | Cluster `prod` nicht registriert (Secret `cluster-prod` fehlt) oder ApplicationSets nicht angewendet | Abschnitte 1 und 2 oben |
| Application `Unknown` mit `InvalidSpecError` | Namespace fehlt im Projekt `prod` | Namespace in `appproject.yaml` ergänzen und anwenden, danach `argocd.argoproj.io/refresh=hard` auf die Application setzen (Schritt 2 unter „Neue PROD-App hinzufügen“) |
| `Unable to connect to cluster` / `x509: …` im Hub | Token oder CA im Secret `cluster-prod` falsch, `prod-vm` nicht erreichbar | Token und `caData` neu auslesen (Abschnitt 1), `prod-vm` und `6443` prüfen |
| `Unauthorized` für den Cluster | ServiceAccount-Token ungültig oder gelöscht | Secret `argocd-manager-token` auf `prod-vm` prüfen, Token neu einsetzen |
| Neuer Manifest-Ordner (ohne `Chart.yaml`) erscheint nicht | `path` fehlt im Set `home-server-apps-prod` | Eintrag ergänzen ([Zwei ApplicationSets](#zwei-applicationsets)) |
| Doppelte Application | Ordner mit `Chart.yaml` zusätzlich im ersten Set gelistet | aus `home-server-apps-prod` entfernen |
| `prod-<app>` ist `Healthy`, Host antwortet nicht | DNS-Eintrag (`dnsmasq_prod_vm_hosts`), Tunnel-Route fehlt | [DNS: Fehlersuche](../c-netzwerk-dns/c0000-dns-architecture.md#fehlersuche), [Cloudflare Tunnel](../e-externe-erreichbarkeit/e0010-cloudflare-deploy.md) |
| SealedSecret in PROD nicht entschlüsselbar | mit TECH-Schlüssel versiegelt | `scripts/reseal-for-prod.sh` ([Sealed Secrets](../b-kubernetes-gitops/b0070-sealed-secrets.md#fehlersuche)) |
| Sync-Fehler `resource … is not permitted in project prod` | `clusterResourceWhitelist` kennt den Typ nicht | [AppProjects](../b-kubernetes-gitops/b0020-argocd-projects.md#fehlersuche) |
| Rückbau löscht mehr als gedacht | `kubectl delete -f applicationset.yaml` entfernt die Applications samt Ressourcen in PROD | nur bewusst verwenden |

Allgemeine Fehlersuche: [zentrale Fehlersuche](../2-betrieb-hardware/20070-troubleshooting-zentral.md).
