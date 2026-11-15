# Garage — S3-Speicher für Argo Workflows

Einzelner [Garage](https://garagehq.deuxfleurs.fr)-Server (Single-Node, eine Replika) als S3-kompatibler Artefaktspeicher und Log-Archiv für [Argo Workflows](../f-cicd-automatisierung/f0000-argo-workflows.md). Löst seit 2026-11 MinIO ab (Gründe: [60040](../6-hintergruende/60040-helm-charts-tech.md#argo-workflows-und-garage)). Für einen kleinen Heim-CI-Bedarf ausgelegt, nicht als hochverfügbarer Objektspeicher.

> **Cluster:** TECH · Ordner `argocd/apps/tech/garage/` · S3-API nur cluster-intern `garage.garage.svc.cluster.local:3900`, keine Weboberfläche. `kubectl`-Befehle in diesem Doc gelten für den TECH-Cluster ([Zugriff je Cluster](../../capulus-ansible/a-betriebssystem/a0010-overview.md#kubectl-zugriff-je-cluster)).

---

## Architektur

| Teil | Wert |
|---|---|
| Chart | eigenes Chart (Deployment, ConfigMap, Service, PVC, SealedSecret), kein Upstream-Chart |
| Image | `dxflrs/garage:v2.5.0` (Docker Hub, Renovate bumpt `image.tag`) |
| Start | `garage server --single-node --default-bucket`: Layout, Key und Bucket legt der Server beim Start an |
| Konfiguration | ConfigMap `garage-config` (`/etc/garage.toml`): `replication_factor = 1`, Metadaten in SQLite |
| Speicher | PVC `garage-data`, `10Gi`, StorageClass `local-path`, `Delete=false`; Unterordner `meta/` und `data/` |
| Zugangsdaten | Secret `garage-secrets` (Schlüssel `rpcSecret`, `accessKey`, `secretKey`) aus einem SealedSecret, Werte `secrets.encrypted*` |
| Bucket | `argo-artifacts` (`defaultBucket`), Key hat Lesen, Schreiben, Owner |
| Region | `garage` (`s3Region`), Argo Workflows signiert mit derselben Region |
| Ports | 3900 S3-API (Service `garage`), 3903 Admin-API nur für die Probes (`/health`, ohne Token) |
| Ressourcen | Request 20m und 64 MiB, Limit 1 CPU und 512 MiB |
| Pod | läuft als UID 1000, Root-Dateisystem schreibgeschützt, keine Capabilities |

Verwendung: Argo Workflows schreibt Artefakte und Logs in den Bucket `argo-artifacts`, siehe [f0000](../f-cicd-automatisierung/f0000-argo-workflows.md). Zugriff aus anderen Namespaces sperrt die NetworkPolicy `tier-default-ingress`, freigegeben ist nur `argo-workflows` ([d0030](../d-sicherheit/d0030-network-policies.md)).

---

## Zugangsdaten

Key und RPC-Secret liegen als SealedSecret im Repo (Token-Register-Eintrag `garage-s3-keys`, Versiegeln: [Sealed Secrets](../b-kubernetes-gitops/b0070-sealed-secrets.md)). Den Klartext liest man bei Bedarf aus dem Cluster:

```bash
kubectl -n garage get secret garage-secrets -o jsonpath='{.data.accessKey}' | base64 -d; echo
kubectl -n garage get secret garage-secrets -o jsonpath='{.data.secretKey}' | base64 -d; echo
```

Rotation: `scripts/seal-garage.sh` in capulus-core erzeugt alle drei Werte neu und versiegelt den Key zusätzlich für `argo-workflows/argo-artifacts-s3`; PR, Merge, Garage startet neu und legt den neuen Key mit Bucket-Rechten an. Der alte Key bleibt in Garage bestehen und wird von Hand gelöscht:

```bash
kubectl -n garage exec deploy/garage -- /garage key list
kubectl -n garage exec deploy/garage -- /garage key delete --yes <ALTE_KEY_ID>
```

---

## Fehlersuche

```bash
kubectl -n garage get pods,pvc
kubectl -n garage logs deploy/garage --tail=100
kubectl -n garage exec deploy/garage -- /garage status
kubectl -n garage exec deploy/garage -- /garage bucket info argo-artifacts
```

| Symptom | Ursache | Lösung |
|---|---|---|
| Pod `CrashLoopBackOff`, Log `Access key … is associated with a secret key different than the one given` | Secret-Key zur selben Key-ID geändert | Key-ID ebenfalls neu erzeugen (`seal-garage.sh` macht beides) oder alten Key mit `garage key delete` entfernen |
| Log `Access key … was deleted in the cluster, cannot add it back` | Key mit `garage key delete` gelöscht, aber noch in `garage-secrets` | `seal-garage.sh` laufen lassen, neue Key-ID |
| Log `Refusing to run in single-node mode` | Layout wurde von Hand geändert (Version > 1) | `--single-node` ist nur für den Ein-Knoten-Betrieb; Layout zurücksetzen oder PVC leeren |
| Pod `CreateContainerConfigError` (`secret "garage-secrets" not found`) | SealedSecret nicht entschlüsselt | [Sealed Secrets: Fehlersuche](../b-kubernetes-gitops/b0070-sealed-secrets.md#fehlersuche) |
| Argo Workflows meldet `AuthorizationHeaderMalformed` oder `InvalidRegion` | Region in `artifactRepository.s3.region` passt nicht zu `s3Region` | beide auf `garage` |
| Argo Workflows meldet `Forbidden` beim Artefakt-Upload | Key in `argo-artifacts-s3` passt nicht zu `garage-secrets` | `seal-garage.sh` versiegelt beide aus demselben Lauf |
| Pod `Pending` | `local-path` bindet die PVC an den Node, auf dem sie angelegt wurde | Node prüfen; Daten sind nur Logs und Artefakte, PVC löschen ist vertretbar |
| Speicher voll | 10 GiB erreicht, Artefakte werden nicht aufgeräumt | `garage bucket info argo-artifacts`, alte Objekte löschen oder `persistence.size` erhöhen |

Allgemeine Fehlersuche: [zentrale Fehlersuche](../2-betrieb-hardware/20070-troubleshooting-zentral.md).
