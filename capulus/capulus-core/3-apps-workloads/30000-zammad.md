# Zammad Helpdesk / Ticket-System

> **Cluster:** TECH · Ordner `argocd/apps/tech/zammad/` · URL `https://zammad.tech.homeserver` und `https://support-tech.pke-lab.de`. `kubectl`-Befehle in diesem Doc gelten für den TECH-Cluster ([Zugriff je Cluster](../../capulus-ansible/a-betriebssystem/a0010-overview.md#kubectl-zugriff-je-cluster)).

Zammad ist ein Open-Source Helpdesk- und Ticketing-System.  
Die Deployment-Konfiguration liegt unter `argocd/apps/tech/zammad/`.

---

## Übersicht

| Komponente        | Technologie                       | Namespace   |
|-------------------|-----------------------------------|-------------|
| Zammad App        | Rails + Puma                      | `zammad`    |
| Datenbank         | PostgreSQL 18 (CloudPirates Sub-Chart `postgres`, `local-path`) | `zammad`    |
| Cache             | Memcached (CloudPirates OCI)       | `zammad`    |
| Pub/Sub / Sessions| Redis (CloudPirates OCI Sub-Chart) | `zammad`    |
| Volltextsuche     | PostgreSQL-basiert (eingebaut)     | —           |
| Ingress           | Traefik                           | `zammad`    |
| Secrets           | SealedSecrets                     | `zammad`    |
| Persistenz        | StorageClass `nas` (UGREEN NAS, NFS) | —         |
| Auto-Updates      | Renovate (patch + minor)          | —           |

Das Kontaktformular der Website legt Tickets nicht mehr direkt an, sondern über [ticket-gate](300r0-ticket-gate.md) (E-Mail-Bestätigung per Code). Dafür sind `/api/v1/form_submit` und `/api/v1/form_config` auf dem öffentlichen Host gesperrt, die Kanäle *Formular* und *Chat* sind in der Administration deaktiviert, und ein Agent-Benutzer mit API-Token dient ticket-gate zum Anlegen der Tickets.

PostgreSQL läuft als StatefulSet `zammad-postgres` auf `local-path` und ist per `nodeSelector` an `homeserver` gebunden, Redis liegt auf
`storageClassName: nas` (siehe [capulus-core/2-betrieb-hardware/20000-nas-storage.md](../2-betrieb-hardware/20000-nas-storage.md)).
Da Zammad Attachments in der Datenbank speichert (`storageVolume.enabled: false`), wächst die PostgreSQL-PVC mit den Anhängen.
Der Umstieg von der Bitnami-Subchart (`zammad-postgresql`, PG 17) auf das neue StatefulSet mit Chart 20 ist in
[400b0](../4-planung/400b0-zammad-chart-20-postgres-migration.md) beschrieben.

---

## Voraussetzungen

- ArgoCD läuft und das Root-ApplicationSet ist aktiv (`argocd/bootstrap/root-applicationset.yaml`)
- Sealed-Secrets Controller ist installiert (`argocd/apps/tech/sealed-secrets/`)
- **`nas-storage`-App ist deployt** (`argocd/apps/tech/nas-storage/`) und die
  StorageClass `nas` existiert: `kubectl get storageclass nas`
- **NAS ist online und der NFS-Export erreichbar** (siehe
  [capulus-core/2-betrieb-hardware/20000-nas-storage.md](../2-betrieb-hardware/20000-nas-storage.md)) — sonst bleiben die
  PostgreSQL- und Redis-PVCs auf `Pending`
- `kubeseal` CLI ist lokal installiert
- `kubectl` ist mit dem Cluster verbunden

---

## Schritt 1 — Secrets generieren und versiegeln

Vor dem ersten Deployment muss das Datenbank-Passwort versiegelt werden.  
Es wird **zweimal** versiegelt (unterschiedliche Secret-Keys), enthält aber denselben Wert.

### 1.1 Passwort generieren

```bash
# PostgreSQL-Passwort für den Zammad-DB-User
DB_PASS=$(openssl rand -base64 32 | tr -d '=+/' | head -c 32)
echo "DB_PASS: $DB_PASS"
```

Passwort sicher speichern (z. B. in einem Passwort-Manager).

### 1.2 Secrets versiegeln

```bash
NS=zammad
SECRET_NAME=zammad-secrets
CONTROLLER_NS=sealed-secrets
CONTROLLER=sealed-secrets-controller

# Key 1: postgresql-pass (für den Zammad-App-Container)
echo -n "$DB_PASS" | kubeseal --raw \
  --namespace $NS --name $SECRET_NAME \
  --controller-namespace $CONTROLLER_NS \
  --controller-name $CONTROLLER \
  --from-file=/dev/stdin
# → Ausgabe als encryptedPostgresPass eintragen

# Key 2: postgresql-password (für den Postgres-Sub-Chart, postgres.auth.secretKeys.adminPasswordKey)
echo -n "$DB_PASS" | kubeseal --raw \
  --namespace $NS --name $SECRET_NAME \
  --controller-namespace $CONTROLLER_NS \
  --controller-name $CONTROLLER \
  --from-file=/dev/stdin
# → Ausgabe als encryptedPostgresPassword eintragen
```

> **Hinweis:** Beide Werte enthalten dasselbe Passwort, aber unterschiedliche  
> kubeseal-Ciphertexts. Jeder `kubeseal --raw`-Aufruf erzeugt eine andere  
> Ausgabe — das ist korrekt (nichtdeterministische asymmetrische Verschlüsselung).

### 1.3 Werte in values.yaml eintragen

```yaml
secrets:
  enabled: true
  name: zammad-secrets
  encryptedPostgresPass: "<Ausgabe von Aufruf 1>"
  encryptedPostgresPassword: "<Ausgabe von Aufruf 2>"
```

---

## Schritt 2 — Helm-Abhängigkeit laden (lokal, optional)

ArgoCD lädt die Abhängigkeit beim Sync selbst herunter.  
Für lokale Tests (`helm template`, `helm lint`) einmalig ausführen:

```bash
cd argocd/apps/tech/zammad
helm dependency update
```

---

## Schritt 3 — Deployment via ArgoCD

Nach dem Commit der Änderungen erkennt das Root-ApplicationSet den neuen Ordner  
`argocd/apps/tech/zammad/` automatisch und erstellt die ArgoCD-Application.

```
ArgoCD → Home → zammad
Status: Syncing → Healthy
```

Den Sync-Fortschritt überwachen:

```bash
kubectl get pods -n zammad -w
```

Typische Pod-Reihenfolge beim ersten Start:
1. `zammad-postgres-0` (PVC auf `local-path`, nur auf `homeserver`) und `zammad-redis-0` (PVC auf `nas`)
2. `zammad-memcached-*` — startet parallel (kein PVC, läuft überall)
3. `zammad-init-*` — führt Datenbankmigrationen durch
4. `zammad-*` (App-Pod) — startet nach erfolgreichem Init

> Falls `zammad-redis-0` dauerhaft `Pending` bleibt:
> das NAS ist offline oder die `nas`-StorageClass fehlt — siehe
> [capulus-core/2-betrieb-hardware/20000-nas-storage.md](../2-betrieb-hardware/20000-nas-storage.md) → Fehlerbehebung.

---

## Schritt 4 — Ersten Admin-Account anlegen

Nach dem erfolgreichen Start ist Zammad unter https://zammad.tech.homeserver erreichbar.

1. Browser öffnen: `https://zammad.tech.homeserver`
2. Setup-Wizard durchlaufen:
   - **System-URL** eintragen: `https://zammad.tech.homeserver`
   - **Admin-E-Mail** und Passwort festlegen
   - E-Mail-Kanal konfigurieren (optional, kann später gemacht werden)
3. Login mit den im Wizard erstellten Zugangsdaten

---

## Schritt 5 — DNS

Kein Eintrag nötig: Zammad läuft im TECH-Cluster, und `zammad.tech.homeserver` wird von der
`*.homeserver`-Wildcard in dnsmasq auf den Homeserver (`.94`) aufgelöst, siehe
[capulus-core/c-netzwerk-dns/c0040-domain-tiers.md](../c-netzwerk-dns/c0040-domain-tiers.md#dns-tier-und-cluster-sind-zwei-verschiedene-dinge).
Test: `dig +short zammad.tech.homeserver @192.168.178.94`.

---

## Schritt 6 — Öffentlicher Hostname + HTTPS / TLS (optional)

Aktuell ist nur `zammad.tech.homeserver` (intern, HTTP) konfiguriert. Für einen
öffentlichen Zugriff per Domain zuerst einen weiteren Host in
`argocd/apps/tech/zammad/values.yaml` → `zammad.ingress.hosts` ergänzen, dann TLS
aktivieren:

```yaml
zammad:
  ingress:
    annotations:
      traefik.ingress.kubernetes.io/router.entrypoints: websecure
      traefik.ingress.kubernetes.io/router.tls: "true"
    hosts:
      - host: zammad.tech.homeserver
        paths:
          - path: /
            pathType: Prefix
      - host: zammad.eure-domain.de
        paths:
          - path: /
            pathType: Prefix
    tls:
      - hosts:
          - zammad.eure-domain.de
        secretName: zammad-tls  # Cert-Manager oder manuelles TLS-Secret
```

---

## Automatische Updates (Renovate)

Renovate ist in `renovate.json` konfiguriert und aktualisiert das Zammad Helm Chart  
automatisch bei **patch**- und **minor**-Updates:

```json
{
  "matchPackageNames": ["zammad"],
  "matchUpdateTypes": ["patch", "minor"],
  "automerge": true
}
```

- **Patch/Minor:** Renovate öffnet einen PR und merged ihn automatisch nach grünem CI.
- **Major:** Renovate öffnet nur einen PR zur manuellen Prüfung (Breaking Changes).

Renovate prüft das Chart-Repository `https://zammad.github.io/zammad-helm` auf neue Versionen.

---

## Troubleshooting

### Pods starten nicht

```bash
# Events prüfen
kubectl describe pod -n zammad -l app.kubernetes.io/name=zammad

# Logs des Init-Containers prüfen
kubectl logs -n zammad -l app.kubernetes.io/component=zammadInit --previous
```

### SealedSecret wird nicht entschlüsselt

```bash
# Status des SealedSecrets prüfen
kubectl describe sealedsecret -n zammad zammad-secrets

# Sealed-Secrets Controller Logs
kubectl logs -n sealed-secrets -l app.kubernetes.io/name=sealed-secrets
```

### Datenbank-Verbindungsfehler

```bash
# PostgreSQL-Pod Status
kubectl get pod -n zammad -l app.kubernetes.io/name=postgres

# Direkte DB-Verbindung testen
kubectl exec -it -n zammad zammad-postgres-0 -- \
  psql -U zammad -d zammad_production -c '\dt'
```

### Redis-PVC bleibt `Pending`

Redis läuft auf der `nas`-StorageClass (NFS, UGREEN NAS), PostgreSQL auf `local-path` und hängt nicht vom NAS ab:

```bash
kubectl describe pvc -n zammad data-zammad-redis-0
kubectl -n nas-storage get pods
```

Details: [capulus-core/2-betrieb-hardware/20000-nas-storage.md](../2-betrieb-hardware/20000-nas-storage.md) → Fehlerbehebung.

### ArgoCD zeigt OutOfSync

Liegt meist an generischen Ressourcen (z. B. von PostgreSQL), die ArgoCD nicht kennt.  
ServerSideApply ist aktiviert (`syncOptions: ServerSideApply=true`), was die meisten  
Konflikte löst. Bei anhaltenden Problemen manuell syncen:

```bash
argocd app sync zammad --force
```

---

## Ressourcenverbrauch (Richtwerte Home Lab)

| Komponente     | CPU Request | RAM Request | RAM Limit | Storage              |
|----------------|-------------|-------------|-----------|-----------------------|
| Zammad App     | 100m        | 512Mi       | 1Gi       | —                      |
| PostgreSQL     | 50m         | 256Mi       | 512Mi     | 50Gi (`local-path`)    |
| Redis          | —           | —           | —         | 8Gi (`nas`)            |
| Nginx Sidecar  | 25m         | 64Mi        | 128Mi     | —                      |
| **Gesamt**     | ~200m       | ~896Mi      | ~1.8Gi    | 50Gi SSD, 8Gi NAS      |

Redis-Storage liegt auf dem UGREEN NAS, PostgreSQL auf `local-path`, weil der NFS-Export `all_squash` erzwingt
([60040](../6-hintergruende/60040-helm-charts-tech.md#zammad)). `local-path` setzt die 50Gi nicht durch, sie sind nur der Wert im PVC
(`zammad.postgres.persistence.size`).

---

## Autoskalierung (HPA)

`nginx` (1–3 Replicas, CPU 70%) und `railsserver` (1–3 Replicas, CPU 75%
/ RAM 80%) skalieren per hand-geschriebenem HPA (der Upstream-Chart hat
kein natives Autoscaling). `scheduler` und `websocket` bekommen
**bewusst keinen** HPA: der Upstream-Chart kodiert für beide
`replicas: 1` fest ("Not scalable, may only run once per cluster.") —
doppelte Ausführung würde bei `scheduler` Jobs duplizieren, bei
`websocket` vermutlich Echtzeit-Events nur an einen Teil der
verbundenen Browser-Clients ausliefern. PostgreSQL/Redis bleiben
ebenfalls unangetastet. Details für alle Apps: [../b-kubernetes-gitops/b0040-hpa-autoscaling.md](../b-kubernetes-gitops/b0040-hpa-autoscaling.md).

---

## Relevante Links

- [Zammad Helm Chart Repository](https://github.com/zammad/zammad-helm)
- [Zammad Helm Chart values.yaml](https://github.com/zammad/zammad-helm/blob/main/zammad/values.yaml)
- [Zammad Admin-Dokumentation](https://admin-docs.zammad.org)
- [ArgoCD Setup](../b-kubernetes-gitops/b0010-argocd.md)
