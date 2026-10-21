# MinIO — S3-Speicher für Argo Workflows

Einzelner MinIO-Server (Standalone, eine Replika) als S3-kompatibler Artefaktspeicher und Log-Archiv für [Argo Workflows](../f-cicd-automatisierung/f0000-argo-workflows.md). Für einen kleinen Heim-CI-Bedarf ausgelegt, nicht als hochverfügbarer Objektspeicher.

> **Cluster:** TECH · Ordner `argocd/apps/tech/minio/` · Konsole `https://minio.tech.homeserver` (nur Konsole, kein öffentlicher S3-Ingress). `kubectl`-Befehle in diesem Doc gelten für den TECH-Cluster ([Zugriff je Cluster](../../capulus-ansible/a-betriebssystem/a0010-overview.md#kubectl-zugriff-je-cluster)).

---

## Architektur

| Teil | Wert |
|---|---|
| Chart | Wrapper um das Chart `minio` von `charts.min.io` (Version `*`, nicht fest gepinnt), `appVersion: latest` |
| Modus | `standalone`, `replicas: 1` |
| Speicher | PVC `10Gi`, StorageClass `nas`, `ReadWriteOnce` ([NAS-Speicher](../2-betrieb-hardware/20000-nas-storage.md)) |
| Zugangsdaten | Secret `minio-root` (Schlüssel `rootUser`, `rootPassword`) aus einem SealedSecret (`templates/root-creds-sealedsecret.yaml`, Werte `rootCreds.encryptedRootUser` und `encryptedRootPassword`), im Chart als `existingSecret: minio-root` verwendet |
| Bucket | `argo-artifacts` (`policy: none`, `purge: false`) |
| Konsole | Ingress Klasse `traefik`, Host `minio.tech.homeserver` |
| Ressourcen | Request 50m und 256 MiB, Limit 1 CPU und 1 GiB |

Verwendung: Argo Workflows schreibt Artefakte und Logs in den Bucket `argo-artifacts`, siehe [f0000](../f-cicd-automatisierung/f0000-argo-workflows.md).

---

## Zugangsdaten

Der Root-Zugang liegt als SealedSecret im Repo (Token-Register-Eintrag `minio-root-creds`, Rotation: [Secrets-Rotation](../d-sicherheit/d0060-secrets-rotation.md), Versiegeln: [Sealed Secrets](../b-kubernetes-gitops/b0070-sealed-secrets.md)). Den Klartext liest man bei Bedarf aus dem Cluster:

```bash
kubectl -n minio get secret minio-root -o jsonpath='{.data.rootUser}' | base64 -d; echo
kubectl -n minio get secret minio-root -o jsonpath='{.data.rootPassword}' | base64 -d; echo
```

---

## Fehlersuche

```bash
kubectl -n minio get pods,pvc,ingress
kubectl -n minio logs deploy/minio --tail=100
kubectl -n minio get job   # Bucket-Anlage: Job des Charts
```

| Symptom | Ursache | Lösung |
|---|---|---|
| Pod `Pending`, PVC `Pending` | StorageClass `nas` nicht bereit, NAS nicht erreichbar | [NAS-Speicher](../2-betrieb-hardware/20000-nas-storage.md), `kubectl -n minio describe pvc` |
| Pod `CreateContainerConfigError` (`secret "minio-root" not found`) | SealedSecret nicht entschlüsselt | [Sealed Secrets: Fehlersuche](../b-kubernetes-gitops/b0070-sealed-secrets.md#fehlersuche) |
| Anmeldung an der Konsole schlägt fehl | Zugangsdaten rotiert, aber Secret nicht aktualisiert | Secret-Inhalt prüfen (siehe oben), Pod neu starten |
| Argo Workflows meldet `NoSuchBucket` oder `Access Denied` beim Artefakt-Upload | Bucket `argo-artifacts` fehlt (Anlage-Job fehlgeschlagen) oder falsche Zugangsdaten im Argo-Secret | Job-Log des Charts, Bucket in der Konsole anlegen, [Argo Workflows](../f-cicd-automatisierung/f0000-argo-workflows.md) |
| Konsole nicht erreichbar | Ingress, DNS, VPN | `nslookup minio.tech.homeserver`, [DNS](../c-netzwerk-dns/c0000-dns-architecture.md) |
| Speicher voll | 10 GiB erreicht, Artefakte werden nicht aufgeräumt | alte Artefakte in der Konsole löschen oder `persistence.size` erhöhen (PVC-Erweiterung beachten) |
| Unerwarteter Versionssprung | `appVersion: latest` und Chart-Version `*` | Chart-Version und Image-Tag in `Chart.yaml` und `values.yaml` pinnen |
| Datenverlust nach PVC-Löschung | `purge: false` bezieht sich nur auf Buckets, nicht auf die PVC | Backup-Konzept: [NAS-Backup](../2-betrieb-hardware/20010-nas-backup.md) |

Allgemeine Fehlersuche: [zentrale Fehlersuche](../2-betrieb-hardware/20070-troubleshooting-zentral.md).
