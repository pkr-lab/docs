# Helm

Helm-Releases aller Namespaces ansehen: Status, Values und Historie.

Code: [modules/helm.py](https://github.com/pkr-lab/cubi/blob/main/src/cubi/modules/helm.py)

---

## Überblick

`cubi helm` ruft `helm list -A -o json --max 500` auf und zeigt die Releases als Tabelle (alle 30 Sekunden neu), sortiert nach Namespace und Name. Der Bereich liest nur und ändert nichts.

| Spalte | Quelle |
|---|---|
| Release, Namespace | `name`, `namespace` |
| Rev | `revision` |
| Status | `status`, farbig: `deployed` grün, `failed` und `unknown` rot, `pending-*` und `uninstalling` gelb, `superseded` und `uninstalled` grau |
| Chart, App-Version | `chart`, `app_version` |
| Alter | aus `updated` berechnet (`3d`, `5h`) |

Enter auf einem Release öffnet das Menü:

| Aktion | Befehl |
|---|---|
| Status | `helm status <name> -n <ns>` |
| Values | `helm get values <name> -n <ns>` (leer: `Keine eigenen Values`) |
| Historie | `helm history <name> -n <ns>` |

Beachten: Im Homelab rollt ArgoCD die Charts aus. ArgoCD rendert sie mit `helm template` und legt keine Helm-Release-Secrets an, ArgoCD-Apps fehlen daher in dieser Tabelle. Sichtbar sind nur Releases, die Helm selbst installiert hat. Nach dem Code von capulus-ansible ist das mindestens ArgoCD (Rolle [argocd](../capulus/capulus-ansible/rollen/argocd.md), `helm upgrade --install argocd argo/argo-cd`) und, was k3s selbst per Helm-Controller mitbringt (zum Beispiel Traefik, konfiguriert über [`helmchartconfig.yaml`](https://github.com/pkr-lab/capulus-core/blob/main/argocd/apps/tech/traefik-config/helmchartconfig.yaml)). TODO: prüfen, welche Releases `helm list -A` im Cluster tatsächlich zeigt. Für ArgoCD-Apps ist [ArgoCD](argocd.md) die richtige Ansicht. Hintergrund: [Helm-Charts](../capulus/capulus-core/6-hintergruende/60040-helm-charts-tech.md) und [ArgoCD](../capulus/capulus-core/b-kubernetes-gitops/b0010-argocd.md).

---

## Debugging und Fehlersuche

| Meldung | Ursache | Lösung |
|---|---|---|
| `helm ist nicht installiert oder nicht im PATH` | Helm fehlt | installieren |
| `Kubernetes cluster unreachable: …` (Fehlertext von helm) | falscher Kontext oder Cluster nicht erreichbar | `kubectl config current-context`, siehe [Kubernetes](kubernetes.md) |
| `Zeitüberschreitung bei: helm list -A -o json --max 500` | API-Server langsam (Timeout 30 Sekunden) | Verbindung prüfen |
| `Ungültige Ausgabe von: helm list …` | Ausgabe ist kein JSON | Befehl direkt ausführen, `helm`-Version prüfen |
| Tabelle leer (`0 Releases`) | keine Helm-Releases im Cluster, siehe Hinweis oben | normal bei reinem ArgoCD-Betrieb |
| Release auf `failed` oder `pending-upgrade` | abgebrochener Upgrade oder Installation | *Historie* und *Status* lesen, Ursache beheben, Rollback mit `helm rollback` außerhalb von cubi |
| Menü *Values* zeigt `Keine eigenen Values` | Release nutzt nur Chart-Defaults | `helm get values <name> -n <ns> -a` für alle Werte |

Checkliste: `helm list -A` direkt, `helm status`, bei Zugriffsproblemen `kubectl auth can-i list secrets -A` (Helm speichert Release-Daten in Secrets).

Allgemeines: [Debugging und Fehlersuche](debugging.md).
