# App-Umzug mit Daten nach PROD

Runbook und Vorlage fuer den Umzug einer App samt Daten von TECH nach PROD. **Stand 21.09.2026:** die
Apps paperless-ngx, wikijs, nextcloud, immich und xibosignage sind umgezogen, ihre
TECH-Ordner sind entfernt. Die Kopier-Jobs bleiben als Vorlage fuer weitere Umzuege erhalten;
Vaultwarden, Zammad und n8n bleiben bewusst in TECH
([Planung](../4-planung/40080-multi-cluster-entw-prod-tech.md)).

Manuell anzuwendende Hilfsmanifeste, **nicht** von ArgoCD gelesen (weder das
Manifest- noch das Charts-Set schauen in diesen Ordner).

Grundsatz: **Kopie, waehrend TECH weiterlaeuft.** TECH-Daten werden nur
gelesen, das alte NFS-Verzeichnis (StorageClass `nas`, `reclaimPolicy: Retain`)
bleibt unangetastet und ist der Rueckfall. Nie TECH und PROD gleichzeitig auf
dieselben Daten schreiben lassen.

## Weitere Apps

Je App ein eigener Kopier-Job in diesem Ordner, gleicher Ablauf (unten):

| Job | App | Besonderheit |
|---|---|---|
| `paperless-ngx-data-copy-job.yaml` | paperless-ngx | 4 Volumes |
| `wikijs-postgres-copy-job.yaml` | wikijs | Postgres-Datenverzeichnis, eigener uid/Modus 0700 |
| `nextcloud-data-copy-job.yaml` | nextcloud | `html`- und `data`-Volume |
| `immich-data-sync-job.yaml` | immich | rsync grosser Foto-Bestaende, Postgres-Schalter |
| `xibosignage-data-copy-job.yaml` | xibosignage | CMS-Daten, MySQL-Schalter |

Ablauf: Push 1 = TECH und PROD auf `replicaCount: 0`, Endkopie, Push 2 = PROD auf `1` plus Hosts in
`dnsmasq_prod_vm_hosts`.

## Ablauf (Vorlage fuer `nas`-Apps)

Alte Daten: `192.168.178.97:/volume1/k8s-storage/<app>-<pvc>-<uid>`
(`kubectl get pv <pv> -o jsonpath='{.spec.nfs.path}'`).

### A. Vorbereitung (TECH laeuft unveraendert weiter)

1. Committen/pushen, dann `kubectl apply -f argocd/bootstrap-prod/appproject.yaml`
   und beide ApplicationSets refreshen. Es entsteht `prod-<app>` mit
   `replicaCount: 0` (das PVC wird in PROD angelegt, kein Pod).
2. Pruefen (PROD): `kubectl -n <app> get pvc`.
3. Probekopie (TECH laeuft, die DB kann dabei inkonsistent sein, es geht nur
   um den Test des Kopierwegs): Job anwenden, Logs lesen, Job loeschen.
4. Generalprobe: `replicaCount: 1` in `argocd/apps/prod/<app>/values.yaml`
   pushen. PROD-<app> laeuft auf der Probekopie, TECH und DNS bleiben
   unveraendert (kein Nutzer merkt etwas). Testen ueber `--resolve`:
   `curl -skI --resolve <app>.prod.homeserver:443:192.168.178.99 https://<app>.prod.homeserver`
   (muss `200` liefern). Danach vor der Endkopie wieder `replicaCount: 0` pushen.

### B. Umschalten (kurze Pause fuer die Nutzer)

5. TECH-<app> stoppen **per Git**: `replicaCount: 0` in
   `argocd/apps/tech/<app>/values.yaml`, pushen (ein `kubectl scale`
   wuerde von selfHeal zurueckgesetzt). Warten bis der Pod weg ist.
6. Endkopie (PROD-<app> muss dabei auf 0 stehen): Job erneut anwenden (leert das Ziel und kopiert neu). In den
   Logs muessen die erwarteten Dateien stehen (z. B. die SQLite-Datenbank); danach Job loeschen.
7. PROD starten: `replicaCount: 1` in `argocd/apps/prod/<app>/values.yaml`,
   pushen. Test wie in Schritt 4:
   `curl -skI --resolve <app>.prod.homeserver:443:192.168.178.99 https://<app>.prod.homeserver`
   muss `200` liefern.
8. DNS intern: `<app>.prod.homeserver` in
   `dnsmasq_prod_vm_hosts` (capulus-ansible/group_vars/all.yml), `make dnsmasq`.
9. Oeffentlich (nur der eine Host der App): `cloudflared tunnel route dns
   homeserver-prod <oeffentlicher-host>`. NIE einen anderen Host als den der
   gerade umgezogenen App eintragen.

### C. Rueckweg

DNS-Eintraege (Schritte 9 und 8) zuruecknehmen (Cloudflare-Dashboard bzw.
`dnsmasq_prod_vm_hosts` + `make dnsmasq`), in TECH `replicaCount: 1`. Aenderungen,
die seit dem Umschalten in PROD entstanden sind, fehlen dann in TECH.

### D. Aufraeumen (erst nach einigen Tagen Betrieb)

TECH-Ordner `argocd/apps/tech/<app>/` entfernen. Die alten PVs bleiben
wegen `Retain` erhalten; das NAS-Verzeichnis erst spaeter von Hand loeschen.

---

## Fehlersuche

```bash
kubectl -n <app> get job,pods,pvc
kubectl -n <app> logs job/<app>-data-copy              # bzw. wikijs-postgres-copy, immich-data-sync, …
kubectl get pv <pv> -o jsonpath='{.spec.nfs.path}'      # alter NFS-Pfad (TECH)
```

| Symptom | Ursache | Lösung |
|---|---|---|
| Job bleibt `Pending`/`ContainerCreating` | NFS-Mount nicht möglich (NAS `192.168.178.97` nicht erreichbar, Pfad falsch, Export nicht freigegeben) | `kubectl -n <app> describe pod`, Pfad mit `kubectl get pv … -o jsonpath` neu bestimmen, [NAS-Speicher](../2-betrieb-hardware/20000-nas-storage.md) |
| Job `Failed`, Log: `ls: /old/…: No such file` | falscher Quellpfad (PV-Name geändert) | Pfad im Job an das PV anpassen |
| `Permission denied` beim Kopieren | falscher Benutzer oder falsche Gruppe im Job (`runAsUser`, `fsGroup`) | Werte an die App anpassen (zum Beispiel wikijs: Gruppe 10, Postgres-Verzeichnis Modus `0700`) |
| Zielvolume `Pending` | PVC in PROD noch nicht angelegt, weil `prod-<app>` fehlt | Schritt A1 ausführen, `kubectl -n <app> get pvc` |
| App in PROD startet nicht nach der Kopie (Datenbank-Fehler) | Datenbank war beim Kopieren in Betrieb (inkonsistent) | Endkopie mit gestoppter TECH-App (Schritt B5) wiederholen, Job leert das Ziel zuerst |
| Daten fehlen in PROD | Job lief gegen laufende PROD-App (Ziel muss auf 0 stehen) | PROD auf `replicaCount: 0`, Job erneut |
| Beide Apps schreiben in dieselben Daten | TECH nicht gestoppt | Absicht: TECH per Git auf `replicaCount: 0` (nicht per `kubectl scale`, `selfHeal` setzt zurück) |
| Rückweg nötig | Probleme nach dem Umschalten | Abschnitt C (DNS zurück, TECH-`replicaCount: 1`), Änderungen seit dem Umschalten fehlen in TECH |
| Job lässt sich nicht erneut anwenden | `ttlSecondsAfterFinished: 3600` hat den alten Job noch nicht gelöscht | `kubectl -n <app> delete job <name>` und erneut anwenden |

Weiter: [PROD im TECH-Hub: Fehlersuche](bootstrap-prod.md#fehlersuche), [Planung Multi-Cluster](../4-planung/40080-multi-cluster-entw-prod-tech.md). Allgemeine Fehlersuche: [zentrale Fehlersuche](../2-betrieb-hardware/20070-troubleshooting-zentral.md).
